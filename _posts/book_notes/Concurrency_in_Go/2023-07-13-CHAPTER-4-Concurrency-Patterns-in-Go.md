---
title: "CHAPTER 4. Concurrency Patterns in Go"
categories:
  - book_notes
tags:
  - Concurrency in Go
---

# Confinement

동시성 코드에서 데이터를 안전하게 사용하기 위한 방법

## Ad hoc confinement

```go
data := make([]int, 4)

// data는 loopData에서만 접근하자고 약속하자.
loopData := func(handleData chan<- int) {
	defer close(handleData)
	for i := range data {
		handleData <- data[i]
	}
}

handleData := make(chan int) 
go loopData(handleData)

for num := range handleData {
	fmt.Println(num)
}
```

- 암묵적인 규칙을 준수하는 방법
- 암묵적인 규칙이기 때문에 안전하지 않다.

## Lexical confinement

```go
chanOwner := func() <-chan int {
	// results의 접근 범위를 chanOwner로 제한한다.
	results := make(chan int, 5) 
	go func() {
		defer close(results)
		for i := 0; i <= 5; i++ {
			results <- i
		}
	}()
	return results
}

consumer := func(results <-chan int) {
	for result := range results {
		fmt.Printf("Received: %d\n", result)
	}
	fmt.Println("Done receiving!")
}

results := chanOwner()
consumer(results)
```

- Lexical scope를 이용하는 방법
- 접근 자체를 막기 때문에 더 안전하다.

# The for-select Loop

```go
for _, s := range []string{"a", "b", "c"} {
	select {
	case <-done:
		return
	case stringStream <- s:
	}
}
```

```go
for {
	select {
	case <-done:
		return
	default:
		// Do non-preemptable work here
	}
	// Or do non-preemptable work here
}
```

# Preventing Goroutine Leaks

고루틴은 garbage collect 대상이 아니다.

```go
doWork := func(strings <-chan string) <-chan interface{} {
	completed := make(chan interface{})
	go func() {
		defer fmt.Println("doWork exited.")
		defer close(completed)
		for s := range strings {
			// Do something interesting
			fmt.Println(s)
		}
	}()
	return completed
}

doWork(nil)
// Perhaps more work is done here
fmt.Println("Done.")
```

- `doWork` 안의 고루틴은 하는 일 없이 메모리만 잡아먹는다.

```go
doWork := func(done <-chan interface{},	strings <-chan string,) <-chan interface{} {
	terminated := make(chan interface{})
	go func() {
		defer fmt.Println("doWork exited.")
		defer close(terminated)
		for {
			select {
			case s := <-strings:
				// Do something interesting
				fmt.Println(s)
			case <-done: 
				return
			}
		}
	}()
	return terminated
}

done := make(chan interface{})
terminated := doWork(done, nil)

go func() { 
	// Cancel the operation after 1 second.
	time.Sleep(1 * time.Second)
	fmt.Println("Canceling doWork goroutine...")
	close(done)
}()

<-terminated
fmt.Println("Done.")

// Canceling doWork goroutine...
// doWork exited.
// Done.
```

- 종료 신호를 보내기 위해 관습적으로 `done`이라는 읽기 전용 채널을 사용한다.
    - 관습적으로 `done`은 첫 번째 인자로 들어간다.
- 만약 어떤 고루틴을 만들었다면, 그 고루틴을 멈추는 부분까지 함께 관리하자.

# The or-channel

여러 채널을 하나로 묶어서 한 채널이 닫히면 모두 닫히게 할 수 있다.

```go
var or func(channels ...<-chan interface{}) <-chan interface{}
or = func(channels ...<-chan interface{}) <-chan interface{} { 
	switch len(channels) {
	case 0: 
		return nil
	case 1: 
		return channels[0]
	}

	orDone := make(chan interface{})
	go func() { 
		defer close(orDone)

		// 채널이 0개 또는 1개인 경우는 위에서 처리되었다.
		switch len(channels) {
		case 2: 
			select {
			case <-channels[0]:
			case <-channels[1]:
			}
		default: 
			select {
			case <-channels[0]:
			case <-channels[1]:
			case <-channels[2]:
			case <-or(append(channels[3:], orDone)...): 
			}
		}
	}()
	return orDone
}

// after 후에 닫히는 채널 생성
sig := func(after time.Duration) <-chan interface{} { 
	c := make(chan interface{})
	go func() {
		defer close(c)
		time.Sleep(after)
	}()
	return c
}

start := time.Now() 
<-or(
	sig(2*time.Hour),
	sig(5*time.Minute),
	sig(1*time.Second),
	sig(1*time.Hour),
	sig(1*time.Minute),
)
fmt.Printf("done after %v", time.Since(start)) 

// done after 1s 
```

- 채널을 두 개씩 기다리기 때문에 이진 트리를 형성한다고 볼 수 있다.

# Error Handling

고루틴에서 에러가 날 수 있다면 그 에러는 더 많은 정보를 가진 바깥으로 보내진 후 처리되는 게 낫다.

```go
type Result struct { 
	Error    error
	Response *http.Response
}
checkStatus := func(done <-chan interface{}, urls ...string) <-chan Result { 
	results := make(chan Result)
	go func() {
		defer close(results)

		for _, url := range urls {
			var result Result
			resp, err := http.Get(url)
			// 여기서 err가 nil이 아니라고 그냥 로그만 찍는 건 도움이 되지 않는다.
			// 더 많은 정보를 가진 밖에서 이를 처리할 수 있도록 하는 게 더 낫다.
			result = Result{Error: err, Response: resp} 
			select {
			case <-done:
				return
			case results <- result: 
			}
		}
	}()
	return results
}

done := make(chan interface{})
defer close(done)

urls := []string{"https://www.google.com", "https://badhost"}
for result := range checkStatus(done, urls...) {
	if result.Error != nil { // <5>
		fmt.Printf("error: %v", result.Error)
		continue
	}
	fmt.Printf("Response: %v\n", result.Response.Status)
}
```

```go
done := make(chan interface{})
defer close(done)

errCount := 0
urls := []string{"a", "https://www.google.com", "b", "c", "d"}
for result := range checkStatus(done, urls...) {
	if result.Error != nil {
		fmt.Printf("error: %v\n", result.Error)
		errCount++
		// 일정 개수의 에러만 처리할 수도 있다.
		if errCount >= 3 {
			fmt.Println("Too many errors, breaking!")
			break
		}
		continue
	}
	fmt.Printf("Response: %v\n", result.Response.Status)
}

// error: Get "a": unsupported protocol scheme ""
// error: Get "https://www.google.com": dial tcp: lookup www.google.com on 169.254.169.254:53: dial udp 169.254.169.254:53: connect: no route to host
// error: Get "b": unsupported protocol scheme ""
// Too many errors, breaking!
```

# Pipelines

- 파이프라인은 여러 스테이지로 구성되고, 각 스테이지에 관심사를 분리해 놓을 수 있다.
- 스테이지는 데이터를 받아 작업을 하고 다시 데이터를 내놓는다.
- 스테이지가 하는 작업의 종류
    - Batch process: 데이터를 받아 한꺼번에 다시 내놓는 작업
        
        ```go
        ints := []int{1, 2, 3, 4}
        for _, v := range multiply(add(multiply(ints, 2), 1), 2) {
        	fmt.Println(v)
        }
        ```
        
    - Stream process: 데이터를 받아 하나씩 다시 내놓는 작업
        
        ```go
        ints := []int{1, 2, 3, 4}
        for _, v := range ints {
        	fmt.Println(multiply(add(multiply(v, 2), 1), 2))
        }
        ```
        
        - `range`가 데이터를 먹여줘야 하고, loop 안에서 계속 함수 호출도 하며 concurrent하게 작성하기도 어렵다.
