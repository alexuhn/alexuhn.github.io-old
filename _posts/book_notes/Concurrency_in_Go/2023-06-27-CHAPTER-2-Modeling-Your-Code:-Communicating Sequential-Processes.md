---
title: "CHAPTER 2. Modeling Your Code: Communicating Sequential Processes"
excerpt: "CSP와 memory access synchronization"
categories:
  - book_notes
tags:
  - Concurrency in Go
---

# The Difference Between Concurrency and Parallelism

- Concurrency
    - 코드의 속성
    - Concurrent하게 작성한 코드가 parallel하게 동작하지 않을 수 있다.
- Parallelism
    - 동작하는 프로그램의 속성
    - 프로그램이 parallel하게 동작하는지는 context에 달렸다.
        - 예를 들어 만약 5초의 시간 공간이 존재하는 context에서 2초씩 걸리는 두 개의 작업을 실행한다면 두 작업은 parallel하게 동작하는 것이다.
        - 물론 context는 시간에 국한되지 않는다.

# What Is CSP?

- Communicating Sequential Processes
    - 동명의 논문에서 등장한 개념
    - 논문 저자 Hoare는 동시성 코드를 위한 input과 output primitives를 제안했다.
- 이후 process calculus로 발전하고 Go의 동시성 모델에 영향을 주었다.
    - process calculus: 동시성 시스템을 수학적으로 모델링하고 efficiency와 correctness와 같은 속성을 계산하기 위한 방법
- Hoare의 CSP programming 언어에는 process의 communication 즉 input과 output을 위한 primitives가 존재한다.
    - `!`
        - 프로세스 `!` 프로세스로 보낼 input
        - `lineprinter!lineimage`
    - `?`
        - 프로세스 `?` 프로세스에서 내보낸 output을 담을 변수
        - `cardreader?cardimage`
    - `west?c → east!c`
        - `west` 프로세스의 output이 `c`로 보내지고, `east` 프로세스로 보낼 input도 `c`에서 받는다.
        - 이렇게 프로세스에서 값을 읽는 부분이 프로세스에 값을 보내는 부분과 동일한 경우 correspond 라고 지칭하고. 따라서 두 프로세스는 correspond 하다.
    - `→`
        - Dijkstra의 논문에서 소개된 guarded command.
        - 좌측 항은 조건 또는 guard라고 부르며, 만약 좌측 항이 거짓이거나 거짓을 반환하거나 아니면 exited된다면 우측 항은 절대 실행되지 않는다.
- Go 언어는 CSP를 성공적으로 접목하였다.

# How This Helps You

- 일반적으로 다른 언어는 동시성 문제에서 OS thread와 memory access synchronization을 다뤄야 하지만 Go는 이를 gorountine과 channel로 대체했다.
- Go 언어는 parallelism을 대신 처리해 주기 때문에 우리가 concurrency만 고민할 수 있도록 해준다.

# Go’s Philosophy on Concurrency

- Go에선 memory access synchronization 방식을 따르는 코드 작성 또한 가능하다.
    - `sync`와 같은 패키지를 사용해 lock이나 pool을 구현할 수 있다.
- 하지만 Go는 `sync.Mutex`와 같은 primitive 대신 CSP 스타일을 더 권장한다.
- 그럼 애초에 memory access synchronization primitive는 왜 만들었고 사람들은 왜 이 방법을 더 많이 쓰냐? 정말 혼란스럽다! 무엇을 쓸지 어떻게 결정을 내려야 할까?

![Decisiong tree]({{ site.url }}{{ site.baseurl }}/assets/images/posts/Concurrency_in_Go/1.png){: .align-center}
이렇게 결정해 보자.

1. Is it a performance-critical section?
    1. 성능때문에 무조건 mutex 쓰란 소리가 아니다.
    2. 프로그램에서 한 부분이 유난히 느려 bottleneck이 발생한다면 memory access synchronization primitives를 쓰는 게 도움이 될 수 있다. 
    3. 채널도 내부적으로 memory access synchronization primitives를 쓰기 때문에 더 느릴 수 있다. 그런데 애초에 프로그램을 리팩토링해야하는 건 아닐까?
2. Are you trying to transfer ownership of data?
    1. 만약 어떤 producer 코드가 결과를 생성하고 이를 다른 코드로 넘긴다면 이는 데이터 소유권을 넘기는 과정이다.
    2. 데이터는 한 concurrent context만 소유권을 갖고 있어야 안전하다. 
    3. 채널을 사용하여 데이터 소유권을 넘길 수 있다.
3. Are you trying to guard internal state of a struct?
    1. Memory access synchronization primitives를 사용하면 critical section을 호출자로부터 숨길 수 있다.
        
        ```go
        type Counter struct {
        		mu sync.Mutex
        		value int
        }
        func (c *Counter) Increment() {
        		c.mu.Lock()
        		defer c.mu.Unlock()
        		c.value++
        }
        ```
        
4. Are you trying to coordinate multiple pieces of logic?
    1. 여기저기 lock이 흩뿌려져 있는 건 악몽이지만 채널은 그래도 된다.
    2. 만약 primitives를 사용 중인데 deadlock이나 race가 계속 발생한다면 채널로 교체하는 것도 좋다.

일반적으로 동시성 프로그램에 사용되는 패턴은 접어두자. Go에서는 그다지 유용하지 않다.

Go에서 동시성은 단순하게, 가능하면 채널을 써서, goroutine을 마음껏 이용하며 다루자.

