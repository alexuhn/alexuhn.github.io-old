---
title: "[Concurrency in Go] CHAPTER 2. Modeling Your Code: Communicating Sequential Processes"
excerpt: "CSP를 이용한 동시성 코드 작성"
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
- 그럼 애초에 memory access synchronization primitive는 왜 만들었고 사람들은 왜 이 방법을 더 많이 쓰냐? 정말 혼란스럽다! 무엇을 쓸 지 어떻게 결정을 내려야 할까?
