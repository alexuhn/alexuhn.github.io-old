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

## Best Practices for Constructing Pipelines

스테이지가 block되더라도 채널을 닫아 이를 unblock할 수 있도록 해야 한다.

```go
select {
case <-done:
	return
case ...:
}
```

### 예시

```go
// Date를 채널로 변환
generator := func(done <-chan interface{}, integers ...int) <-chan int {
	intStream := make(chan int)
	go func() {
		defer close(intStream)
		for _, i := range integers {
			select {
			case <-done:
				return
			case intStream <- i:
			}
		}
	}()
	return intStream
}

multiply := func(
	done <-chan interface{},
	intStream <-chan int,
	multiplier int,
) <-chan int {
	multipliedStream := make(chan int)
	go func() {
		defer close(multipliedStream)
		for i := range intStream {
			select {
			case <-done:
				return
			case multipliedStream <- i * multiplier:
			}
		}
	}()
	return multipliedStream
}

add := func(
	done <-chan interface{},
	intStream <-chan int,
	additive int,
) <-chan int {
	addedStream := make(chan int)
	go func() {
		defer close(addedStream)
		for i := range intStream {
			select {
			case <-done:
				return
			case addedStream <- i + additive:
			}
		}
	}()
	return addedStream
}

// 고루틴 leak 방지 & 종료 신호
done := make(chan interface{})
defer close(done)

intStream := generator(done, 1, 2, 3, 4)
pipeline := multiply(done, add(done, multiply(done, intStream, 2), 1), 2)

for v := range pipeline {
	fmt.Println(v)
}
```

## Some Handy Generators

```go
// 주어진 values를 무한히 반복하는 stream 생성하는 스테이지
repeat := func(
	done <-chan interface{},
	values ...interface{},
) <-chan interface{} {
	valueStream := make(chan interface{})
	go func() {
		defer close(valueStream)
		for {
			for _, v := range values {
				select {
				case <-done:
					return
				case valueStream <- v:
				}
			}
		}
	}()
	return valueStream
}

// valueStream에서 num 개 만큼만 가져오는 스테이지
take := func(
	done <-chan interface{},
	valueStream <-chan interface{},
	num int,
) <-chan interface{} {
	takeStream := make(chan interface{})
	go func() {
		defer close(takeStream)
		for i := 0; i < num; i++ {
			select {
			case <-done:
				return
			case takeStream <- <-valueStream:
			}
		}
	}()
	return takeStream
}

done := make(chan interface{})
defer close(done)

for num := range take(done, repeat(done, 1), 10) {
	fmt.Printf("%v ", num)
}

// 1 1 1 1 1 1 1 1 1 1
```

```go
take := func(
	done <-chan interface{},
	valueStream <-chan interface{},
	num int,
) <-chan interface{} {
	takeStream := make(chan interface{})
	go func() {
		defer close(takeStream)
		for i := 0; i < num; i++ {
			select {
			case <-done:
				return
			case takeStream <- <-valueStream:
			}
		}
	}()
	return takeStream
}

repeat := func(
	done <-chan interface{},
	values ...interface{},
) <-chan interface{} {
	valueStream := make(chan interface{})
	go func() {
		defer close(valueStream)
		for {
			for _, v := range values {
				select {
				case <-done:
					return
				case valueStream <- v:
				}
			}
		}
	}()
	return valueStream
}

// valueStream에서 받은 값을 string으로 전환하는 스테이지
toString := func(
	done <-chan interface{},
	valueStream <-chan interface{},
) <-chan string {
	stringStream := make(chan string)
	go func() {
		defer close(stringStream)
		for v := range valueStream {
			select {
			case <-done:
				return
			case stringStream <- v.(string):
			}
		}
	}()
	return stringStream
}

done := make(chan interface{})
defer close(done)

var message string
for token := range toString(done, take(done, repeat(done, "I", "am."), 5)) {
	message += token
}

fmt.Printf("message: %s...", message)

// message: Iam.Iam.I...
```

- Go에서 `interface{}` 사용을 권장하진 않지만 목적에 따라 잘 사용하면 큰 성능 차이는 없다.

# Fan-Out, Fan-In

- Fan-Out, Fan-In이 무엇인가?
    - Fan-Out: 여러 고루틴을 이용해 파이프라인에서 input을 다루는 프로세스
    - Fan-In: 여러 결과를 하나의 채널로 합치는 프로세스
- 어떤 스테이지에 쓰면 좋은가?
    - 이전에 계산된 값에 의존하지 않아도 되는, 즉 순서에 영향받지 않는 경우
    - 오래 걸리는 작업이 있는 경우

## 예시

```go
rand := func() interface{} { return rand.Intn(50000000) }

done := make(chan interface{})
defer close(done)

start := time.Now()

randIntStream := toInt(done, repeatFn(done, rand))
fmt.Println("Primes:")
for prime := range take(done, primeFinder(done, randIntStream), 10) {
	fmt.Printf("\t%d\n", prime)
}

fmt.Printf("Search took: %v", time.Since(start))

// Primes:
//	24941317
//	36122539
//	6410693
//	10128161
//	25511527
//	2107939
//	14004383
// 	7190363
//	45931967
//	2393161
// Search took: 23.437511647s
```

- `primeFinder`
    - 수를 받아 그 수보다 작은 모든 수로 나눠보며 소수를 찾는다. 즉 매우 오래 걸린다.
    - 스테이지 순서에 영향을 받지도 않는다.
    - 따라서 fan-out에 적당하다.

**Fan-Out**

`NumCPU` 수만큼 고루틴을 늘려 속도를 증가시킬 수 있다.

```go
numFinders := runtime.NumCPU()
finders := make([]<-chan int, numFinders)
for i := 0; i < numFinders; i++ {
	finders[i] = primeFinder(done, randIntStream)
}
```

**Fan-In**

늘어난 고루틴에서 오는 결과를 하나로 합쳐야 한다.

```go
fanIn := func(
	done <-chan interface{},
	channels ...<-chan interface{},
) <-chan interface{} { 
	// 모든 채널을 기다리기 위한 WaitGroup
	var wg sync.WaitGroup 
	multiplexedStream := make(chan interface{})

	// 채널에서 값을 받아 multiplexedStream 채널로 전달
	multiplex := func(c <-chan interface{}) { 
		defer wg.Done()
		for i := range c {
			select {
			case <-done:
				return
			case multiplexedStream <- i:
			}
		}
	}

	// Select from all the channels
	wg.Add(len(channels))
	for _, c := range channels {
		go multiplex(c)
	}

	// Wait for all the reads to complete
	go func() {
		wg.Wait()
		close(multiplexedStream)
	}()

	return multiplexedStream
}
```

# The or-done-channel

채널에서 값을 읽다가 중단하려면 아래와 같이 코드가 복잡해질 수 있다.

```go
loop:
for {
	select {
	case <-done:
		break loop
	case maybeVal, ok := <-myChan:
		if ok == false {
			return // or maybe break from for
		}
		// Do something with val
	}
}
```

or-done 채널을 이용해 이를 읽기 쉽게 바꿀 수 있다.

```go
orDone := func(done, c <-chan interface{}) <-chan interface{} {
	valStream := make(chan interface{})
	go func() {
		defer close(valStream)
		for {
			select {
			case <-done:
				return
			case v, ok := <-c:
				if ok == false {
					return
				}
				select {
				case valStream <- v:
				case <-done:
				}
			}
		}
	}()
	return valStream
}

for val := range orDone(done, myChan) {
	// Do something with val
}
```

# The tee-channel

채널에서 받은 값을 두 곳으로 보낼 때가 있다.

```go
tee := func(
	done <-chan interface{},
	in <-chan interface{},
) (_, _ <-chan interface{}) {
	out1 := make(chan interface{})
	out2 := make(chan interface{})
	go func() {
	defer close(out1)
	defer close(out2)
	for val := range orDone(done, in) {
		var out1, out2 = out1, out2
		// 두 채널 중 한 곳에서 값을 받으면 nil이 되므로
		// 무조건 두 채널 모두 값을 받을 수 있게 됨
		for i := 0; i < 2; i++ {
			select {
			case <-done:
			case out1<-val:
				out1 = nil // 더 이상 값 받을 수 없음
			case out2<-val:
				out2 = nil // 더 이상 값 받을 수 없음
			}
		}
	}
	}()
	return out1, out2
}

done := make(chan interface{})
defer close(done)

out1, out2 := tee(done, take(done, repeat(done, 1, 2), 4))

for val1 := range out1 {
	fmt.Printf("out1: %v, out2: %v\n", val1, <-out2)
}

// out1: 1, out2: 1
// out1: 2, out2: 2
// out1: 1, out2: 1
// out1: 2, out2: 2
```
