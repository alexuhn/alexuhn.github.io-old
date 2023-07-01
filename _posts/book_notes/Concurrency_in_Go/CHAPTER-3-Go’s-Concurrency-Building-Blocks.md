---
title: "[Concurrency in Go] CHAPTER 3. Go’s Concurrency Building Blocks"
excerpt: "Go에서 동시성 코드를 위해 사용하는 요소"
categories:
  - book_notes
tags:
  - Concurrency in Go
---


# Goroutines

- Concurrent하게 동작하는 함수
    - 앞서 말했듯 parallel하게 동작하지 않을 수 있음에 주의
- OS 스레드가 아니다. 그린 스레드도 아니다. Coroutine으로 알려진 높은 레벨의 추상화다.
    - Coroutine: Suspension과 reentry가 가능하지만 interrupt는 불가한 concurrent subroutine
- Go 런타임은 알아서 고루틴의 suspension과 reentry를 관리해 준다.
- Go는 M:N 스케쥴러를 사용해 고루틴을 관리한다.
    - M개의 그린 스레드 : N개의 OS 스레드
    - 고루틴은 그린 스레드에 스케쥴된다.
- Go는 fork-join이라는 동시성 모델을 따른다.
    - fork: 프로그램이 자식 브랜치로 분리되어 부모 브랜치와 concurrent하게 실행
    - join: 분리된 concurrent 브랜치가 다시 돌아와 합체
        - join point: 자식 브랜치가 부모 브랜치로 돌아오는 지점
        
        ```go
        var wg sync.WaitGroup
        sayHello := func() {
        	defer wg.Done()
        	fmt.Println("hello")
        }
        wg.Add(1)
        go sayHello()
        wg.Wait() // join point
        ```
        
- 고루틴은 고루틴이 생성된 같은 주소 공간에서 실행된다. 클로저를 통해 이를 살펴보자.
    
    ```go
    var wg sync.WaitGroup
    for _, salutation := range []string{"hello", "greetings", "good day"} {
    	wg.Add(1)
    	go func() {
    			defer wg.Done()
    			fmt.Println(salutation)
    	}()
    }
    wg.Wait()
    ```
    
    - 보통 고루틴이 실행되기 전에 for loop가 끝나버려서 `good day`만 세 번 찍힌다.
    - 그럼 어떻게 `salutation`에 접근할 수 있냐 하면, 다행히 Go가 알아서 `salutation`을 heap으로 옮겨놔 고루틴이 접근할 수 있게 한다.
- 고루틴은 매우 매우 가볍다.
    - 하지만 garbage collector는 고루틴을 collect 하지 않기 때문에 leak가 발생하지 않도록 해야 한다.
- 매우 많은 동시성 프로세스는 그만큼의 context switching이 필요하므로 OS level에서  CPU를 낭비를 막기 위해 고민해야 하지만 고루틴의 context switching은 매우 빠르기 때문에 이런 고민을 덜 수 있다.

# The `sync` Package

- 저수준 memory access synchronization에 유용

## `WaitGroup`

- 동시성 작업 결과를 신경 쓸 필요 없거나 다른 방법으로 결과를 모으는 경우 유용하다.
    - 아니라면 채널과 `select` 사용을 권한다.
- `Add`
    - 고루틴 내부에서 `Add`를 호출하지 말자. 추적하기 어렵고 race condition이 생길 수 있다.
    - 쉽게 추적할 수 있게 고루틴에 최대한 붙여서 호출하는 게 일반적이다.

## `Mutex` and `RWMutex`

- `Mutex`
    - Critical section을 보호하는 목적으로 사용한다.
- `RWMutex`
    - 모든 프로세스가 메모리를 읽고 쓸 필요가 없을 때 유용하다.
    - 하지만 실제로 `Mutex`에 비해 좋은 성능을 낼지는 잘 고민하고 써야 한다.
