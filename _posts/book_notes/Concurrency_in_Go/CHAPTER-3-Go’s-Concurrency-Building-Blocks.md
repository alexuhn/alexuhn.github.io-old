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
 
## `Cond`

- 어떤 이벤트를 기다리거나 알릴 때 사용
- 사용 예시
    - Infinite loop를 사용해 신호를 기다리는 경우
        
        ```go
        for conditionTrue() == false {
        }
        ```
        
        - 이러면 코어에서 계속 이 작업을 반복해야 한다.
    - 다른 작업을 할 시간을 주는 경우
        
        ```go
        for conditionTrue() == false {
        	time.Sleep(1*time.Millisecond)
        }
        ```
        
        - 여전히 비효율적이다.
    - `Cond`를 사용하는 경우
        
        ```go
        c := sync.NewCond(&sync.Mutex{})
        c.L.Lock()
        for conditionTrue() == false {
        	c.Wait()
        }
        c.L.Unlock()
        ```
        
        - `Wait`은 입장 시 `Unlock`을 호출하고 퇴장 시 `Lock`을 호출한다.
        - `Wait`은 고루틴을 대기하게 만들어 다른 고루틴이 실행될 수 있도록 한다.
    - `Cond`를 사용해 신호를 주고받는 경우
        
        ```go
        c := sync.NewCond(&sync.Mutex{})
        queue := make([]interface{}, 0, 10)
        
        removeFromQueue := func(delay time.Duration) {
        	time.Sleep(delay)
        	c.L.Lock()
        	queue = queue[1:]
        	fmt.Println("Removed from queue")
        	c.L.Unlock()
        	c.Signal()
        }
        
        for i := 0; i < 10; i++{
        	c.L.Lock()
        	for len(queue) == 2 {
        		c.Wait()
        	}
        	fmt.Println("Adding to queue")
        	queue = append(queue, struct{}{})
        	go removeFromQueue(1*time.Second)
        	c.L.Unlock()
        }
        ```
        
        - `Signal`을 호출해서 다시 고루틴이 조건을 확인하게 할 수 있다.
- `Signal`은 가장 오래 기다린 고루틴에 신호를 보내지만 `Broadcast`는 모든 고루틴에 신호를 보낸다.
    - 채널을 이용해 모든 고루틴에 호를 보내는 건 어렵기 때문에 이때는 `Broadcast`가 유용하다.

## `Once`

- 서로 다른 고루틴에서 호출되더라도 `Do`는 딱 한 번만 실행된다.
    
    ```go
    var count int
    
    increment := func() {
    	count++
    }
    
    var once sync.Once
    
    var increments sync.WaitGroup
    increments.Add(100)
    for i := 0; i < 100; i++ {
    	go func() {
    		defer increments.Done()
    		once.Do(increment)
    	}()
    }
    
    increments.Wait()
    fmt.Printf("Count is %d\n", count) // Count is 1
    ```
    
- 여러 번 호출되더라도 `Do`는 딱 한 번만 실행된다.
    
    ```go
    var count int
    increment := func() { count++ }
    decrement := func() { count-- }
    
    var once sync.Once
    once.Do(increment)
    once.Do(decrement)
    
    fmt.Printf("Count: %d\n", count) // Count is 1
    ```
    

## `Pool`

- 사용 예시
    
    ```go
    myPool := &sync.Pool{
    	New: func() interface{} {
    		fmt.Println("Creating new instance.")
    		return struct{}{}
    	},
    }
    
    myPool.Get()
    instance := myPool.Get()
    myPool.Put(instance)
    myPool.Get()
    ```
    
    - `Get`: 사용할 수 있는 인스턴스가 있는지 확인하며, 만약 없다면 `New`를 호출하여 새로운 인스턴스를 생성한다.
    - `Put`: 인스턴스를 반환한다.
- `Pool`은 메모리 효율성을 높일 수 있다.
    
    ```go
    var numCalcsCreated int
    calcPool := &sync.Pool {
    	New: func() interface{} {
    		numCalcsCreated += 1
    		mem := make([]byte, 1024)
    		return &mem
    	},
    }
    
    // Seed the pool with 4KB
    calcPool.Put(calcPool.New())
    calcPool.Put(calcPool.New())
    calcPool.Put(calcPool.New())
    calcPool.Put(calcPool.New())
    
    const numWorkers = 1024*1024
    var wg sync.WaitGroup
    wg.Add(numWorkers)
    for i := numWorkers; i > 0; i-- {
    	go func() {
    		defer wg.Done()
    		mem := calcPool.Get().(*[]byte)
    		defer calcPool.Put(mem)
    		// Assume something interesting, but quick is being done with
    		// this memory.
    	}()
    }
    wg.Wait()
    fmt.Printf("%d calculators were created.", numCalcsCreated)
    // 18 calculators were created.
    ```
    
- `Pool`은 시간 효율성을 높일 수 있다.
    
    ```go
    func connectToService() interface{} {
    	time.Sleep(1 * time.Second)
    	return struct{}{}
    }
    
    func warmServiceConnCache() *sync.Pool {
    	p := &sync.Pool{
    		New: connectToService,
    	}
    	for i := 0; i < 10; i++ {
    		p.Put(p.New())
    	}
    	return p
    }
    
    func startNetworkDaemon() *sync.WaitGroup {
    	var wg sync.WaitGroup
    	wg.Add(1)
    	go func() {
    		connPool := warmServiceConnCache()
    
    		server, err := net.Listen("tcp", "localhost:8080")
    		if err != nil {
    			log.Fatalf("cannot listen: %v", err)
    		}
    		defer server.Close()
    
    		wg.Done()
    
    		for {
    			conn, err := server.Accept()
    			if err != nil {
    				log.Printf("cannot accept connection: %v", err)
    				continue
    			}
    			svcConn := connPool.Get()
    			fmt.Fprintln(conn, "")
    			connPool.Put(svcConn)
    			conn.Close()
    		}
    	}()
    	return &wg
    }
    ```
    
    - 미리 `Pool`을 이용해 connection을 생성해 놓았기 때문에(warming a cache) 더 빠르다.
        
        <div class="notice--primary" markdown="1">
        🤔 미리 connection을 생성해 놓아서 시간을 절약할 수 있는 거라면 굳이 `Pool`을 쓰지 않아도 되는 거 아닌가?
        </div>
        <div class="notice--primary" markdown="1">
        💡 ChatGPT 답변
        가능하지만 `Pool`을 사용했을 때의 장점이 있다.
        
        - `Pool`을 사용하면 자동으로 connection을 재사용할 수 있다.
            - `Put`만 호출하면 이후 재사용할 수 있도록 알아서 나머지를 관리한다.
        - 동적으로 자원을 관리할 수 있다.
            - 자원이 더 필요하면 자동으로 `Pool`이 새 connection을 생성한다.
        - 만약 connection이 오래 사용되지 않으면 자동으로 garbage collection 대상이 된다.
        </div>
        

# Channels

- 값을 읽거나 쓸 수 있는 양방향 채널 선언
    
    ```go
    var dataStream chan interface{}
    dataStream = make(chan interface{})
    ```
    
- 값을 읽기만 할 수 있는 단방향 채널 선언
    
    ```go
    var dataStream <-chan interface{}
    dataStream := make(<-chan interface{})
    ```
    
    - 값을 쓰면 에러 발생
        
        ```go
        readStream := make(<-chan interface{})
        readStream <- struct{}{}
        
        // invalid operation: cannot send to receive-only channel readStream (variable of type <-chan interface{})
        ```
        
- 값을 쓸 수만 있는 단방향 채널 선언
    
    ```go
    var dataStream chan<- interface{}
    dataStream := make(chan<- interface{})
    ```
    
    - 값을 읽으면 에러 발생
        
        ```go
        writeStream := make(chan<- interface{})
        <-writeStream
        
        // invalid operation: cannot receive from send-only channel writeStream (variable of type chan<- interface{})
        ```
        
- 양방향 채널을 단방향 채널로 변환할 수 있다.
    
    ```go
    var receiveChan <-chan interface{}
    var sendChan chan<- interface{}
    dataStream := make(chan interface{})
    
    // Valid statements:
    receiveChan = dataStream
    sendChan = dataStream
    ```
    
- 채널에서 값을 보내고 받기
    
    ```go
    stringStream := make(chan string)
    go func() {
    	stringStream <- "Hello channels!" // 채널 <- 보내는 값
    }()
    fmt.Println(<-stringStream) // <- 채널
    ```
    
- 채널은 blocking 작업이다.
    - 꽉 찬 채널에 값을 쓰려면 빌 때까지 기다려야 한다.
    - 빈 채널에서 값을 읽으려면 값이 들어올 때까지 기다려야 한다.
    - 따라서 잘못 쓰면 deadlock이 발생한다.
        
        ```go
        stringStream := make(chan string)
        go func() {
        	if 0 != 1 {
        		return
        	}
        	stringStream <- "Hello channels!" // 도달 불가능
        }()
        fmt.Println(<-stringStream)
        
        // fatal error: all goroutines are asleep - deadlock!
        ```
        
- 채널 닫기
    
    ```go
    valueStream := make(chan interface{})
    close(valueStream)
    ```
    
    - 흥미롭게도 닫힌 채널에서 값을 읽을 수 있다. 
    값을 읽을 때 두 번째 반환 값을 통해 채널이 닫혔는지 알 수 있다.
        
        ```go
        intStream := make(chan int)
        close(intStream)
        integer, ok := <- intStream
        fmt.Printf("(%v): %v", ok, integer) // (false): 0
        ```
        
- for loop에서 채널을 읽을 수 있다. 
채널이 닫힐 때까지 for loop가 돌아간다.
    
    ```go
    intStream := make(chan int)
    go func() {
    	defer close(intStream)
    	for i := 1; i <= 5; i++ {
    		intStream <- i
    	}
    }()
    
    for integer := range intStream {
    	fmt.Printf("%v ", integer) // 1 2 3 4 5
    }
    ```
    
- 채널을 닫아 여러 고루틴에 동시에 신호를 줄 수 있다.
    
    ```go
    begin := make(chan interface{})
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
    	wg.Add(1)
    	go func(i int) {
    		defer wg.Done()
    		<-begin
    		fmt.Printf("%v has begun\n", i)
    	}(i)
    }
    
    fmt.Println("Unblocking goroutines...")
    close(begin)
    wg.Wait()
    ```
    
    - 고루틴은 `<-begin`에서 기다리다가 `close(begin)`가 호출되면 다음으로 넘어간다.
