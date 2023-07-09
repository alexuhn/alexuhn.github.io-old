---
title: "CHAPTER 1. An Introduction to Concurrency"
excerpt: "Concurrency의 등장 배경과 문제점, 어떻게 하면 안전하게 동시성 코드를 작성할 수 있는지에 대하여"
categories:
  - book_notes
tags:
  - Concurrency in Go
---


# Moore’s Law, Web Scale, and the Mess We’re In

- 2012년 즈음까지는 Moore's Law가 맞아들었지만, 그 이후부터는 상황이 달라졌다. 많은 회사들이 computing power를 향상시키기 위해 노력하였고, 그 결과 멀티코어 프로세서가 탄생하였다.
    - 문제를 해결했다고 생각했지만 새로운 한계, Amdahl's law를 마주했다.
        - Amdahl's law 참고) https://needjarvis.tistory.com/522
- 2000년대 초반 cloud computing으로 인해 horizontal scaling이 매우 쉬워졌다. 직접 서버를 관리하는 대신 다른 회사의 데이터 센터를 이용할 수 있게 되었고, 저렴한 가격으로 강력한 computing power를 사용할 수 있게 되었다.
- 덕분에 여러 머신을 손쉽게 사용할 수 있게 되었지만, 이는 어떻게 코드를 concurrent 하게 모델링할 수 있을지에 대한 또 다른 고민을 낳았다. 이 문제를 잘 해결한 것이 있으니, 바로 web scale이다.
- 만약 소프트웨어에 web scale을 적용할 수 있다면 인스턴스 수를 늘려 쉽게 문제를 해결할 수 있을 것이다.
- 멀티 코어, cloud conputing, web scale 그리고 여러 문제와 함께하는 이 세상에서 우리는 어떻게 동시성 문제를 해결할 수 있는가.

# Why Is Concurrency Hard?

동시성 코드에서 흔히 발견되는 문제를 살펴보자.

## Race conditions

- Race consitions은 순서가 보장되어야 할 작업의 작동 순서가 그렇지 못한 경우 발생한다.
- 대표적으로 data race에서 발생하는 문제이다.
    - 예) 한 작업이 값을 읽음과 동시에 다른 작업이 같은 값에 쓰기 작업을 하는 경우
- 문제 예시 코드
    
    ```go
    var data int
    go func() {
    	data++
    }()
    if data == 0 {
    	fmt.Printf("the value is %v.\n", data)
    }
    ```
    
    - 위 코드에선 세 가지의 각기 다른 결과가 나올 수 있다.
        - 아무것도 출력하지 않기
        - `the value is 0`
        - `the value is 1`
    - 이를 해결하기 위해 goroutine과 if 문 사이에 `time.Sleep()`을 넣을 생각은 하지 말자. 애초에 논리적으로 틀린 코드다.

## Atomicity

- Atomic 작업은 어떠한 context 안에서 indivisible 하거나 uninterruptible 하다는 것이다.
- Context의 정의는 중요하다. 어떤 process의 context 내에서 atomic한 작업이 OS의 context 내에서는 atomic하지 않을 수 있고, 어떤 OS의 context 내에서는 atomic한 작업이 application의 context 내에서는 atomic하지 않을 수 있다. <br/>따라서 atomicity를 생각할 때는 먼저 어느 context, 어느 scope 내에서 이 작업이 atomic하다는 건지 정의해야 한다.
- Atomic한 작업 여러 개를 묶어놓는다고 더 큰 atomic한 작업이 되지는 않는다.
- 동시성 프로그래밍에서 어떤 작업이 atomic하다면, concurrent context 내에서 해당 작업은 안전해진다.
    - 동시성 작업이 없는 context를 갖는 코드는 그 context 내에서 atomic하다. 만약 `i++` 코드가 작성되어 있지만 `i`가 외부에 노출되지 않은 goroutine이라면, 이 또한 그 context 내에서 atomic하다.

## Memory Access Synchronization

Data race가 발생하는 예시 코드를 통해 알아보도록 한다.

```go
var data int
go func() { data++ }()
if data == 0 {
fmt.Println("the value is 0.")
} else {
fmt.Printf("the value is %v.\n", data)
}
```

- 공유된 자원에 독점적인 접근 권한이 필요한 부분을 critical section이라 한다.
    - 위 코드에는 세 critical section이 존재한다.
        - `data`를 증가시키는 gorountine
        - `data`가 `0`인지 확인하는 if 문
        - `data`를 가져와 출력하는 `fmt.Printf` 문

Critical section을 안전하게 만들기 위해 위 코드를 다음과 같이 수정할 수 있다.

```go
var memoryAccess sync.Mutex 
var value int
go func() {
	memoryAccess.Lock() 
	value++
	memoryAccess.Unlock() 
}()

memoryAccess.Lock()
if value == 0 {
	fmt.Printf("the value is %v.\n", value)
} else {
	fmt.Printf("the value is %v.\n", value)
}
memoryAccess.Unlock() 
```

- Memory access synchronization이 적용되었다.
- Data race는 해결하였지만, 여전히 race condition 문제가 남아있다. Goroutine이 먼저 실행될지 if-else block이 먼저 실행될지 아직도 알 수가 없다.
- 또한 이는 유지보수, 성능 문제 등을 만들 수도 있다. Critical section에 계속 입장/퇴장을 반복해야 하는지, critical section의 크기는 얼만큼이 적당한지에 대한 문제도 있다.

## Deadlocks, Livelocks, and Starvation

### Deadlock

- 모든 concurrent process가 서로를 기다리는 상태
- The Coffman Conditions: Deadlock이 발생하기 위해 필요한 조건
    - 참고) [https://ko.wikipedia.org/wiki/교착_상태](https://ko.wikipedia.org/wiki/%EA%B5%90%EC%B0%A9_%EC%83%81%ED%83%9C)

```go
type value struct {
		mu    sync.Mutex
		value int
}

var wg sync.WaitGroup
printSum := func(v1, v2 *value) {
	defer wg.Done()
	v1.mu.Lock()
	defer v1.mu.Unlock()
	time.Sleep(2 * time.Second)
	v2.mu.Lock()
	defer v2.mu.Unlock()
	fmt.Printf("sum=%v\n", v1.value+v2.value)
}

var a, b value
wg.Add(2)
go printSum(&a, &b)
go printSum(&b, &a)
wg.Wait()
```

- `go printSum(&a, &b)`에서 a가 잠기고 `go printSum(&b, &a)`에서 b가 잠긴 뒤 서로가 서로를 풀어주기를 기다리게 된다.
- The Coffman Conditions를 적용하면 다음과 같다.
    - 상호 배제: `printSum` 함수는 자원 `a`와 `b`의 독점적인 권한을 요구한다.
    - 점유 대기: 첫 번째 `printSum` 프로세스는 자원 `a`를 가진 상태에서 자원 `b`를 기다리고 있다. 두 번째 `printSum` 프로세스는 자원 `b`를 가진 상태에서 자원 `a`를 기다리고 있다.
    - 비선점: 코드의 gorountine은 자원을 뺏을 방법이 없다.
    - 순환 대기: 첫 번째 `printSum` 프로세스는 두 번째 `printSum` 프로세스를 기다리고, 두 번째 `printSum` 프로세스는 첫 번째 `printSum` 프로세스를 기다린다.
- Deadlock은 네 조건 중 하나라도 만족하지 않으면 방지할 수 있지만 실제로는 이 조건을 찾아내기가 어렵다.

### Livelock

- Concurrent 작업을 막 수행 중이지만 실제로 프로그램이 진행되지는 않는 경우
    - 마치 서로가 서로에게 무한히 길을 비켜주는 상황과 같다.
- Deadlock을 피하고자 되는대로 비켜주는 경우 자주 발생하는 문제다.
    
    ```go
    cadence := sync.NewCond(&sync.Mutex{})
    go func() {
    	for range time.Tick(1 * time.Millisecond) {
    		cadence.Broadcast()
    	}
    }()
    
    takeStep := func() {
    	cadence.L.Lock()
    	cadence.Wait()
    	cadence.L.Unlock()
    }
    
    tryDir := func(dirName string, dir *int32, out *bytes.Buffer) bool {
    	fmt.Fprintf(out, " %v", dirName)
    	atomic.AddInt32(dir, 1) 
    	takeStep()              
    	if atomic.LoadInt32(dir) == 1 {
    		fmt.Fprint(out, ". Success!")
    		return true
    	}
    	takeStep()
    	atomic.AddInt32(dir, -1) 
    	return false
    }
    
    var left, right int32
    tryLeft := func(out *bytes.Buffer) bool { return tryDir("left", &left, out) }
    tryRight := func(out *bytes.Buffer) bool { return tryDir("right", &right, out) }
    walk := func(walking *sync.WaitGroup, name string) {
    	var out bytes.Buffer
    	defer func() { fmt.Println(out.String()) }()
    	defer walking.Done()
    	fmt.Fprintf(&out, "%v is trying to scoot:", name)
    	for i := 0; i < 5; i++ { 
    		if tryLeft(&out) || tryRight(&out) { 
    			return
    		}
    	}
    	fmt.Fprintf(&out, "\n%v tosses her hands up in exasperation!", name)
    }
    
    var peopleInHallway sync.WaitGroup 
    peopleInHallway.Add(2)
    go walk(&peopleInHallway, "Alice")
    go walk(&peopleInHallway, "Barbara")
    peopleInHallway.Wait()
    
    // Barbara is trying to scoot: left right left right left right left right left right
    // Barbara tosses her hands up in exasperation!
    // Alice is trying to scoot: left right left right left right left right left right
    // Alice tosses her hands up in exasperation!
    
    // Alice와 Barbara는 먼저 takeStep()을 해도 cadence를 기다려야 한다.
    // 따라서 atomic.LoadInt32(dir) 값은 항상 2가 된다.
    ```
    
- Livelock은 starvation의 하위 집합이다.

### Starvation

- Concurrent 작업이 필요한 자원을 얻지 못하는 상황
- Livelock은 모든 concurrent 작업이 공평하게 필요한 작업을 얻지 못해 결국 어떤 작업도 끝내지 못하는 상황을 의미하지만 보통 starvation은 한 개 이상의 작업이 더 많은 자원을 가져가려 하므로 나머지 작업이 공평하게 최대 효율을 내지 못하는 상황을 의미한다.
    
    ```go
    var wg sync.WaitGroup
    var sharedLock sync.Mutex
    const runtime = 1 * time.Second
    
    greedyWorker := func() {
    	defer wg.Done()
    
    	var count int
    	for begin := time.Now(); time.Since(begin) <= runtime; {
    		sharedLock.Lock()
    		time.Sleep(3 * time.Nanosecond)
    		sharedLock.Unlock()
    		count++
    	}
    
    	fmt.Printf("Greedy worker was able to execute %v work loops\n", count)
    }
    
    politeWorker := func() {
    	defer wg.Done()
    
    	var count int
    	for begin := time.Now(); time.Since(begin) <= runtime; {
    		sharedLock.Lock()
    		time.Sleep(1 * time.Nanosecond)
    		sharedLock.Unlock()
    
    		sharedLock.Lock()
    		time.Sleep(1 * time.Nanosecond)
    		sharedLock.Unlock()
    
    		sharedLock.Lock()
    		time.Sleep(1 * time.Nanosecond)
    		sharedLock.Unlock()
    
    		count++
    	}
    
    	fmt.Printf("Polite worker was able to execute %v work loops.\n", count)
    }
    
    wg.Add(2)
    go greedyWorker()
    go politeWorker()
    
    wg.Wait()
    
    // Polite worker was able to execute 289777 work loops.
    // Greedy worker was able to execute 471287 work loops.
    ```
    
    - 공유된 `sharedLock`을 `greedyWorker`가 더 많이 점유하고 더 높은 성능을 냈다.
- Starvation은 CPU, 메모리, 파일, 데이터베이스 등 공유될 수 있는 자원만 있다면 발생할 수 있다.
- 위 코드의 `politeWorker` 처럼 memory access synchronization을 적용하는 건 비용이 많이 든다. 그렇다고 비용 감소를 위해 `greedyWorker` 처럼 critical section을 확장시키면 starvation이 생긴다. <br/>
따라서 둘 사이에 균형을 잡아야 하며, 일반적으로 작은 critical section을 이후 확장하는 것이 그 반대 작업보다 쉽기 때문에 저자는 critical section에만 memory access synchronization을 적용하는 것을 추천한다.

## Determining Concurrency Safety

```go
// CalculatePi calculates digits of Pi between the begin and end
// place.
func CalculatePi(begin, end int64, pi *Pi)
```

위 코드를 보고 세 가지 질문을 할 수 있다.

- 어떻게 쓰라는 거지?
- 내가 알아서 concurrent하게 이 함수를 써야 하나?
- `Pi`에 대해서 내가 memory access synchronization을 적용해야 하나? 아니면 `Pi` 내부에서 처리되는 건가?

이러한 질문이 들지 않도록 주석을 잘 달아주자.

```go
// CalculatePi calculates digits of Pi between the begin and end
// place.
//
// Internally, CalculatePi will create FLOOR((end-begin)/2) concurrent
// processes which recursively call CalculatePi. Synchronization of
// writes to pi are handled internally by the Pi struct.
func CalculatePi(begin, end int64, pi *Pi)
```

동시성이 포함된 코드를 작성할 때에는 자세하게 주석을 달아 다음과 같은 질문이 나오지 않는 게 좋다.

- 누가 동시성을 담당하는 거지?
- Concurrency primitives가 어떻게 구성된 거지?
- Memory access synchronization은 누가 담당하고 있지?

아니면 함수형 프로그래밍 방식을 채택해 side effect를 제거하여 함수를 더 분명하게 만들 수도 있다.

```go
func CalculatePi(begin, end int64) []uint
```

Memory access synchronization 문제는 해결되었지만 아직 함수가 동시성을 가졌는지는 분명하지 않다.

채널을 사용하여 이 함수가 동시성을 갖는다는 사실을 더 명확하게 할 수 있다.

```go
func CalculatePi(begin, end int64) <-chan uint
```

하지만 함수가 더 명확해질수록 성능은 더 떨어질 수 있다. 둘 사이의 균형을 잘 맞추어야 한다.

# Simplicity in the Face of Complexity

Go의 concurrency primitives를 통해 우리는 좋은 성능과 명확성을 가질 수 있다.
