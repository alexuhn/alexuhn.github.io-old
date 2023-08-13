---
title: "CHAPTER 6. Goroutines and the Go Runtime"
categories:
  - book_notes
tags:
  - Concurrency in Go
---

# Work Stealing

Go는 work stealing strategy 알고리즘을 사용하여 여러 작업(고루틴)을 여러 OS thread에 할당한다. Work stealing strategy는 다음과 같다.

공유되는 중앙 자원은 critical section이 되므로 하나의 프로세서에 하나의 큐(deque)를 할당하는 게 낫다. Go의 fork-join 모델은 이 큐를 아래 규칙을 따라 이용한다.

1. Fork point (새 고루틴이 생기는 지점) 에서 현재 thread의 deque tail에 새로운 작업을 추가한다.
2. 만약 현재 thread가 일을 하지 않으면 다른 thread의 deque head에서 작업을 뺏어온다.
3. 만약 join point에서 연관된 고루틴의 작업이 끝나지 않은 등의 이유로 작업을 이어갈 수 없으면 현재 thread의 deque tail에서 작업을 가져온다.
4. 만약 deque가 비었다면 다음 둘 중 하나를 한다.
    1.  Join point에서 그냥 기다린다.
    2. 아무 thread의 deque head에서 작업을 뺏어온다.

새로운 작업을 deque의 tail에 추가함으로써 현재의 thread가 미래에 해당 작업을 처리할 가능성이 커진다. 이로 인해 같은 프로세서의 캐시를 이용할 확률이 높아져 작업 처리가 더 효율적으로 된다. 결과적으로 프로그램의 전체적인 성능이 향상될 수 있다.

## 예시

두 개의 single core 프로세서에서 아래 작업을 실행한다.

```go
var fib func(n int) <-chan int
fib = func(n int) <-chan int {
	result := make(chan int)
	go func() {
		defer close(result)
		if n <= 2 {
			result <- 1
			return
		}
		result <- <-fib(n-1) + <-fib(n-2)
	}()
	return result
}

fmt.Printf("fib(4) = %d", <-fib(4))
```

1. 첫 고루틴인 main goroutine이 할당된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine |  |  |  |
2. `fib(4)`가 호출되고 T1 work deque tail에 고루틴이 할당된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine | `fib(4)` |  |  |
3. T1과 T2 둘 중 하나가 고루틴을 뺏어간다. 여기선 T1이 먼저 뺏었다고 하자.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine<br>(join point) |  |  |  |
    | `fib(4)` |  |  |  |
4. `fib(4)`가 ``fib(3)``과 ``fib(2)``를 순서대로 호출해 고루틴이 deque tail에 할당된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine<br>(join point) | ``fib(3)`` |  |  |
    | `fib(4)` | ``fib(2)`` |  |  |
5. T2는 작업이 없으므로 T1 work deque head에서 작업을 뺏어온다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine<br>(join point) | `fib(2)` | `fib(3)` |  |
    | `fib(4)` |  |  |  |
6. T1의 `fib(4)`는 더 이상 할 수 있는 게 없으므로 T1 work deque tail에서 작업을 가져온다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine<br>(join point) |  | `fib(3)` |  |
    | `fib(4)`<br>(join point) |  |  |  |
    | `fib(2)` |  |  |  |
7. T2의 ``fib(3)``가 ``fib(2)``과 `fib(1)`를 순서대로 호출해 고루틴이 deque tail에 할당된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine<br>(join point) |  | `fib(3)` | `fib(2)` |
    | `fib(4)`<br> (join point) |  |  | fib(1) |
    | `fib(2)` |  |  |  |
8. T1에서 ``fib(2)``는 `1`을 반환하고 T2에서 ``fib(3)``은 진행되지 못하므로 deque tail에서 작업을 가져온다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine<br>(join point) |  | `fib(3)`<br>(join point) | `fib(2)` |
    | `fib(4)`<br>(join point) |  | fib(1) |  |
    | return `1` |  |  |  |
9. T1은 다시 할 게 없으므로 T2 work deque head에서 작업을 뺏어오고 T2의 `fib(1)`은 `1`을 반환한다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine<br>(join point) |  | `fib(3)`<br>(join point) |  |
    | `fib(4)`<br>(join point) |  | return `1` |  |
    | `fib(2)` |  |  |  |
10. T1의 ``fib(2)``는 `1`을 반환한다. T2의 join point에 고루틴이 결과를 가지고 모두 돌아오게 된다. 따라서 ``fib(3)``은 이제 `1+1=2`를 반환할 수 있다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine<br>(join point) |  | return `2` |  |
    | `fib(4)`<br>(join point) |  |  |  |
11. 이번엔 T1의 join point에 고루틴이 결과를 가지고 모두 돌아오게 된다. 따라서 `fib(4)`는 이제 `2+1=3`을 반환할 수 있다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine<br>(join point) |  |  |  |
    | return `3` |  |  |  |
12. 마지막으로 main goroutine의 join point에 고루틴이 결과를 가지고 돌아오며 최종 결과를 출력한다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | print `3` |  |  |  |

## Stealing Tasks or Continuations?

그래서 무엇을 큐에 넣고 뺏어야 하는가? Fork-join 모델에선 다음 두 선택지가 있다.

1. Tasks
    1. 고루틴은 task다.
2. Continuations
    1. 고루틴이 호출된 이후의 모든 것은 continuation이다.

이전 예시에서는 고루틴 즉, task를 큐에 넣고 뺏었지만 사실 Go의 work-stealing 알고리즘은 continuation을 큐에 넣고 뺏는다. 이를 통해 join point에서 기다리는 횟수를 줄일 수 있다. 

위 예시를 continuation을 적용해 다시 한번 살펴보자.

1. 첫 고루틴인 main goroutine이 할당된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | main goroutine |  |  |  |
2. `fib(4)`가 호출되고 T1 work deque tail에 main goroutine의 continuation이 할당된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | `fib(4)` | continuation of main goroutine |  |  |
3. T2는 작업이 없으므로 T1 work deque head에서 continuation을 뺏어온다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | `fib(4)` |  | continuation of main goroutine |  |
4. `fib(4)`가 `fib(3)`를 호출해 바로 실행한다. `fib(4)`의 continuation은 deque tail에 할당된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | `fib(3)` | continuation of `fib(4)` | continuation of main goroutine |  |
5. T2는main goroutine의 continuation을 실행할 수 없으므로 join point에서 기다려야 한다. 할 일이 없으므로 T1 work deque head에서 continuation을 뺏어온다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | `fib(3)` |  | continuation of main goroutine<br>(join point) |  |
    |  |  | continuation of `fib(4)` |  |
6. T1의 `fib(3)`가 `fib(2)`을 호출해 실행시키고, `fib(3)`의 continuation이 deque tail에 할당된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | `fib(2)` | continuation of `fib(3)` | continuation of main goroutine<br>(join point) |  |
    |  |  | continuation of `fib(4)` |  |
7. T2의 `fib(4)`가 `fib(2)`을 호출해 실행시키고, `fib(4)`의 continuation이 deque tail에 할당된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | `fib(2)` | continuation of `fib(3)` | continuation of main goroutine<br>(join point) | continuation of `fib(4)` |
    |  |  | `fib(2)` |  |
8. T1과 T2에서 `fib(2)`는 `1`을 반환한다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | return `1` | continuation of `fib(3)` | continuation of main goroutine<br>(join point) | continuation of `fib(4)` |
    |  |  | return `1` |  |
9. T1은 다시 할 게 없으므로 T1 work deque tail에서 continuation을 뺏어와 `fib(1)`을 실행한다.
이때 task를 뺏을 때와 다르게 T1에서 `fib(3)`, `fib(2)`, `fib(1)`을 연속적으로 처리하게 된다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | `fib(1)` |  | continuation of main goroutine<br>(join point) | continuation of `fib(4)` |
    |  |  | return `1` |  |
10. T2는 T2 work deque에서 continuation을 가져온다. 하지만 T1에서 아직 `fib(3)`이 처리 중이라 여기서 또한 할 게 없으므로 join point에서 기다린다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | `fib(1)` |  | continuation of main goroutine<br>(join point) |  |
    |  |  | `fib(4)`<br>(join point) |  |
11. T1의 continuation of `fib(3)`은 작업이 완료되어 `2`를 반환한다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    | return `2` |  | continuation of main goroutine<br>(join point) |  |
    |  |  | `fib(4)`<br>(join point) |  |
12. T2의 continuation of `fib(4)`은 작업이 완료되어 `3`을 반환한다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    |  |  | continuation of main goroutine<br>(join point) |  |
    |  |  | return `3` |  |
13. T2의 continuation of main goroutine은 작업이 완료되어 `4`를 출력한다.
    
    
    | T1 call stack | T1 work deque | T2 call stack | T2 work deque |
    | --- | --- | --- | --- |
    |  |  | print `3` |  |

물론 이는 실제 Go의 알고리즘을 반영하지 못한다. Go 런타임은 모든 OS thread가 context(예시의 큐)를 다룰 수 있도록 최적화가 되어있다. 예를 들어 OS thread가 input/output 등으로 block 된다면 context를 분리하여 다른 thread가 처리할 수 있도록 하고 이후 unblock 된다면 context를 다시 뺏어갈 수 있도록 한다. 만약 이조차 불가능하다면 context의 고루틴을 global context에 두고 미래에 처리될 수 있도록 한다.
