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

# Heartbeats

외부 관찰자에게 심장박동을 들려주는 것과 같이 동시성 작업에서 외부로 신호를 보내는 걸 heartbeats라 한다.

Heartbeats에는 두 가지 종류가 있다.

- 일정 시간 간격으로 작동하는 heartbeats
    - 어떤 이벤트가 발생하길 기다리는 경우 유용하다.
- 어떤 작업의 시작에서 작동하는 heartbeats
    - 테스트에서 타임아웃 대신 사용하기 유용하다.
    
    ```go
    func DoWork(
    	done <-chan interface{},
    	nums ...int,
    ) (<-chan interface{}, <-chan int) {
    	heartbeat := make(chan interface{}, 1)
    	intStream := make(chan int)
    	go func() {
    		defer close(heartbeat)
    		defer close(intStream)
    
    		time.Sleep(2 * time.Second) 
    
    		for _, n := range nums {
    			select {
    			case heartbeat <- struct{}{}:
    			default:
    			}
    
    			select {
    			case <-done:
    				return
    			case intStream <- n:
    			}
    		}
    	}()
    
    	return heartbeat, intStream
    }
    
    // Bad
    func TestDoWork_GeneratesAllNumbers(t *testing.T) {
    	done := make(chan interface{})
    	defer close(done)
    
    	intSlice := []int{0, 1, 2, 3, 5}
    	_, results := DoWork(done, intSlice...)
    
    	for i, expected := range intSlice {
    		select {
    		case r := <-results:
    			if r != expected {
    				t.Errorf(
    					"index %v: expected %v, but received %v,",
    					i,
    					expected,
    					r,
    				)
    			}
    		case <-time.After(1 * time.Second): // Nondeterministic!
    			t.Fatal("test timed out")
    		}
    	}
    }
    
    // Good
    func TestDoWork_GeneratesAllNumbers(t *testing.T) {
    	done := make(chan interface{})
    	defer close(done)
    
    	intSlice := []int{0, 1, 2, 3, 5}
    	heartbeat, results := DoWork(done, intSlice...)
    
    	<-heartbeat // 작업 시작 신호
    
    	i := 0
    	for r := range results {
    		if expected := intSlice[i]; r != expected {
    			t.Errorf("index %v: expected %v, but received %v,", i, expected, r)
    		}
    		i++
    	}
    }
    ```
    

# Replicated Requests

응답을 빠르게 받는 게 매우 중요한 어플리케이션에선 요청을 여러 개 만들어 가장 빨리 오는 응답을 바로 보내줄 수 있다.

```go
doWork := func(
	done <-chan interface{},
	id int,
	wg *sync.WaitGroup,
	result chan<- int,
) {
	started := time.Now()
	defer wg.Done()

	// Simulate random load
	simulatedLoadTime := time.Duration(1+rand.Intn(5)) * time.Second
	select {
	case <-done:
	case <-time.After(simulatedLoadTime):
	}

	select {
	case <-done:
	case result <- id:
	}

	took := time.Since(started)
	// Display how long handlers would have taken
	if took < simulatedLoadTime {
		took = simulatedLoadTime
	}
	fmt.Printf("%v took %v\n", id, took)
}

done := make(chan interface{})
result := make(chan int)

var wg sync.WaitGroup
wg.Add(10)

for i := 0; i < 10; i++ {
	go doWork(done, i, &wg, result)
}

firstReturned := <-result
close(done)
wg.Wait()

fmt.Printf("Received an answer from #%v\n", firstReturned)
// 4 took 5s
// 5 took 1s
// 2 took 3s
// 1 took 1s
// 0 took 4s
// 8 took 4s
// 7 took 5s
// 9 took 3s
// 6 took 2s
// 3 took 1s
// Received an answer from #3
```

만약 모든 요청이 사용하는 자원 등의 이유로 동일한 조건에서 작동하지 못하거나, 너무 비슷해서 차이가 매우 작다면 이 방법은 그다지 유용하지 않다.

# Rate Limiting

많은 rate limiting은 token bucket이라는 알고리즘을 사용한다. 이 알고리즘은 bucket에서 token을 빼내서 요청을 보내는 데 사용하는 과정으로 비유될 수 있다. 

- bucket이 `d`만큼의 깊이를 가지고 있다면 사용자는 최대 `d`번 요청을 보낼 수 있다.
- Bucket에 token이 다시 차는 rate를 `r`이라 하면, 이 값이 곧 rate limit이 된다.
- Burstiness는 bucket이 가득 찼을 때 보낼 수 있는 요청의 수를 의미한다.

자주 들어오는 요청과 그렇지 않은 요청에 동시에 대응하기 위해 multi-rate limiter를 이용할 수도 있다.

```go
type RateLimiter interface {
	Wait(context.Context) error
	Limit() rate.Limit
}

func MultiLimiter(limiters ...RateLimiter) *multiLimiter {
	byLimit := func(i, j int) bool {
		return limiters[i].Limit() < limiters[j].Limit()
	}
	sort.Slice(limiters, byLimit)
	return &multiLimiter{limiters: limiters}
}

type multiLimiter struct {
	limiters []RateLimiter
}

func (l *multiLimiter) Wait(ctx context.Context) error {
	for _, l := range l.limiters {
		if err := l.Wait(ctx); err != nil {
			return err
		}
	}
	return nil
}

func (l *multiLimiter) Limit() rate.Limit {
	return l.limiters[0].Limit()
}

func Open() *APIConnection {
	secondLimit := rate.NewLimiter(Per(2, time.Second), 1)
	minuteLimit := rate.NewLimiter(Per(10, time.Minute), 10)
	return &APIConnection{
		rateLimiter: MultiLimiter(secondLimit, minuteLimit),
	}
}

type APIConnection struct {
	rateLimiter RateLimiter
}

func (a *APIConnection) ReadFile(ctx context.Context) error {
	if err := a.rateLimiter.Wait(ctx); err != nil {
		return err
	}
	// Pretend we do work here
	return nil
}
func (a *APIConnection) ResolveAddress(ctx context.Context) error {
	if err := a.rateLimiter.Wait(ctx); err != nil {
		return err
	}
	// Pretend we do work here
	return nil
}

// 22:46:10 ResolveAddress
// 22:46:10 ReadFile <- 먼저 1 sec token 다 사용
// 22:46:11 ReadFile
// 22:46:11 ReadFile
// 22:46:12 ReadFile
// 22:46:12 ReadFile
// 22:46:13 ReadFile
// 22:46:13 ReadFile
// 22:46:14 ReadFile
// 22:46:14 ReadFile
// 22:46:16 ResolveAddress <- 1 min token과 채워지는 1 sec token 사용
// 22:46:22 ResolveAddress
// 22:46:28 ReadFile
// 22:46:34 ResolveAddress
// 22:46:40 ResolveAddress
// 22:46:46 ResolveAddress
// 22:46:52 ResolveAddress
// 22:46:58 ResolveAddress
// 22:47:04 ResolveAddress
// 22:47:10 ResolveAddress
// 22:47:10 Done.
```

여러 시스템에 접근하는 상황에서 또한 유용하다.

```go
func Open() *APIConnection {
	return &APIConnection{
		apiLimit: MultiLimiter(
			rate.NewLimiter(Per(2, time.Second), 2),
			rate.NewLimiter(Per(10, time.Minute), 10),
		),
		diskLimit: MultiLimiter(
			rate.NewLimiter(rate.Limit(1), 1),
		),
		networkLimit: MultiLimiter(
			rate.NewLimiter(Per(3, time.Second), 3),
		),
	}
}

type APIConnection struct {
	networkLimit,
	diskLimit,
	apiLimit RateLimiter
}

func (a *APIConnection) ReadFile(ctx context.Context) error {
	err := MultiLimiter(a.apiLimit, a.diskLimit).Wait(ctx)
	if err != nil {
		return err
	}
	// Pretend we do work here
	return nil
}

func (a *APIConnection) ResolveAddress(ctx context.Context) error {
	err := MultiLimiter(a.apiLimit, a.networkLimit).Wait(ctx)
	if err != nil {
		return err
	}
	// Pretend we do work here
	return nil
}
```
