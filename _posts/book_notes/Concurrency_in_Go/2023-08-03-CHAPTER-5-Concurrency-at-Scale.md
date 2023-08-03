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
