---
title: "Cookie와 Web Storage"
excerpt: "Cookie와 Web Storage의 기본적인 의미와 그 차이"
categories:
  - TIL
tags:
  - Web

last_modified_at: 2022-03-13

toc: true
toc_sticky: true
toc_label: "목차"
---

# Cookie

쿠키는 웹 브라우저에 의해 방문자의 컴퓨터에 남겨지는 작은 텍스트 파일이다. 쿠키의 최대 크기는 4KB로 제한된다. 쿠키는 사용자의 웹 경험을 기록하기 위해 사용되며 사용자가 설정한 값, 입력값 등이 저장될 수 있다. HTML5 이전에는 쿠키를 이용해 데이터와 서버 요청을 저장했다.

## Session Cookie

세션 쿠키는 만료되는 날이 정해져있지 않으며 브라우저 또는 탭이 열려있는 동안 계속 유지된다. 브라우저 또는 탭이 닫히면 데이터는 사라진다.

## Persistent Cookie

지속 쿠키는 특정 만료 일자가 정해져 있으며 만료 일자 이후 데이터가 사라진다.

<br>

# Web Storage

웹 스토리지는 적용 범위와 수명에 따라 `Session Storage`와 `Local Storage`로 나뉜다. 쿠키의 크기는 4KB로 제한되지만, 웹 스토리지는 최소 5MB의 큰 저장 공간을 갖는다.

## Session Storage

1. 각각의 [출처](https://developer.mozilla.org/ko/docs/Glossary/Origin)에 대해 독립적인 저장 공간을 페이지 세션이 유지되는 동안(브라우저 또는 탭이 열려있는 동안) 제공한다.
2. 세션에 한정해, 즉 브라우저 또는 탭이 닫힐 때까지만 데이터를 저장한다.

## Local Storage

1. 각각의 출처에 대해 데이터가 저장된다.
2. 브라우저를 닫았다 열어도 데이터가 남아있다.
3. 유효기간 없이 데이터를 저장한다. JavaScript를 사용하거나 브라우저 캐시 또는 로컬 저장 데이터를 지워야만 사라진다.

<br>

# Cookie와 Web Storage의 차이점

쿠키의 목적은 서버와의 소통이다. 쿠키는 모든 HTTP 요청에 자동으로 포함되며 서버와 클라이언트 측에서 모두 접근할 수 있다. 웹 스토리지의 데이터는 쿠키와 다르게 서버에 전송되지 않는다. 따라서 쿠키 대신 사용되는 웹 스토리지는 서버와 클라이언트간의 트래픽을 낮출 수 있다는 장점을 가지게 된다.

<br>

# Reference

- [Web Storage API](https://developer.mozilla.org/ko/docs/Web/API/Web_Storage_API)
- [Cookie](https://developer.mozilla.org/en-US/docs/Glossary/Cookie)
- [Web storage](https://en.wikipedia.org/wiki/Web_storage)
- [Cookies vs. LocalStorage: What’s the difference?](https://medium.com/swlh/cookies-vs-localstorage-whats-the-difference-d99f0eb09b44)
- [HTTP cookie](https://en.wikipedia.org/wiki/HTTP_cookie#Persistent_cookie)
