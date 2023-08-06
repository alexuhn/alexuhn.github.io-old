---
title: "CHAPTER 5. Concurrency at Scale"
categories:
  - book_notes
tags:
  - Concurrency in Go
---

# Error Propagation

에러는 다음의 중요한 정보를 포함해야 한다.

- 무슨 일이 발생했는지
- 언제, 어디서 발생했는지
- 사용자에게 제공되는 적절한 메세지
- 사용자가 더 많은 정보를 얻을 방법

이러한 정보를 포함하지 않은 날것의 에러를 버그라 할 수 있다.

# Timeouts and Cancellation

타임아웃을 도입하기 좋은 경우는 다음과 같다고 볼 수 있다.

- 포화상태인 시스템
    - 큐에 요청이 꽉 찬 경우
    - 요청이 타임아웃이 선언된 뒤 다시 반복되지 않는 경우
    - 요청을 저장해 둘 수 있는 자원이 없는 경우
- 더 이상 사용 못 하는 데이터
    - 차례가 너무 늦게 돌아와 요청을 처리할 때 이미 데이터는 유효하지 않은 경우
        - `context.WithDeadline`을 사용해 처리하기 좋다.
        - 데드라인을 미리 알지 못하면 `context.WithCancel`을 사용해 취소하면 좋다.
- Deadlock 방지
    - 시스템이 커질수록 모든 흐름을 제어하기 어렵기 때문에 deadlock을 방지하고자 하는 경우
    - Deadlock을 방지한다고 타임아웃을 잘 못 설정하면 livelock이 될 수 있다. 하지만 deadlock이 발생하여 시스템을 재시작하느니 livelock을 고치는 게 낫다.

동시성 작업을 취소하기 위해선 해당 작업은 preemptable 해야 한다. 

```go
// Preemptable 하지 않은 작업
reallyLongCalculation := func(
	done <-chan interface{},
	value interface{},
) interface{} {
	intermediateResult := longCalculation(value)
	select {
	case <-done:
		return nil
	default:
	}
	return longCaluclation(intermediateResult)
}

// Preemptable 한 작업
reallyLongCalculation := func(
	done <-chan interface{},
	value interface{},
) interface{} {
	intermediateResult := longCalculation(done, value)
	return longCaluclation(done, intermediateResult)
}
```

또한 데이터베이스와 같은 shared state를 다루는 작업은 쉽게 취소 또는 롤백할 수 있도록 해야 한다.

```go
// wrong
result := add(1, 2, 3)
writeTallyToState(result)
result = add(result, 4, 5, 6)
writeTallyToState(result)
result = add(result, 7, 8, 9)
writeTallyToState(result)

// good
result := add(1, 2, 3, 4, 5, 6, 7, 8, 9)
writeTallyToState(result)
```
