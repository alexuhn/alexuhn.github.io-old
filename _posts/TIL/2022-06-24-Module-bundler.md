---
title: "Module bundler"
excerpt: "Module bundler란 무엇인가"
categories:
  - TIL
tags:
  - JavaScript
---

> 본 글은 [<What is module bundler and how does it work?>](https://lihautan.com/what-is-module-bundler-and-how-does-it-work/)을 바탕으로 작성되었다.

# 모듈 번들러란

모듈 번들러는 자바스크립트 모듈들을 브라우저에서 실행 가능한 하나의 자바스크립트 파일로 묶는 도구이다.

## 왜 필요한가

- 브라우저는 모듈 시스템을 지원하지 않는다.
- 모듈의 의존성을 대신 관리해준다.
- 이미지나 CSS와 같은 자원들을 의존성 순서에 맞게 불러와준다.

```jsx
<html>
  <script src="/src/foo.js"></script>
  <script src="/src/bar.js"></script>
  <script src="/src/baz.js"></script>
  <script src="/src/qux.js"></script>
  <script src="/src/quux.js"></script>
</html>
```

위와 같은 코드를 실행시키기 위해선 5번의 HTTP 요청을 보내야 한다. 따라서 5개의 파일을 아래와 같이 하나의 파일로 모으는 것이 낫다.

```jsx
<html>
  <script src="/dist/bundle.js"></script>
</html>
```

이떄 다음과 같은 문제가 생긴다.

- 5개의 파일 순서를 어떻게 유지하는가? 어떻게 5개의 파일 간 의존성을 해결하는가?
- 이름이 중복되는 오류를 어떻게 해결하는가?
- 아직 번들에 포함되지 않은 파일을 어떻게 구분하는가?

만약 파일간 관계를 알고 있다면 위 문제는 해결될 수 있다.

# Webpack은 어떻게 작동하는가

다음과 같은 세 파일이 있다고 가정한다.

```jsx
// circle.js

const PI = 3.141;
export default function area(radius) {
  return PI * radius * radius;
}
```

```jsx
// square.js

export default function area(side) {
  return side * side;
}
```

```jsx
// app.js

import squareArea from "./square";
import circleArea from "./circle";
console.log("Area of square: ", squareArea(5));
console.log("Area of circle", circleArea(5));
```

webpack은 세 파일을 다음과 같이 묶는다.

```jsx
// webpack-bundle.js

const modules = {
  "circle.js": function (exports, require) {
    const PI = 3.141;
    exports.default = function area(radius) {
      return PI * radius * radius;
    };
  },
  "square.js": function (exports, require) {
    exports.default = function area(side) {
      return side * side;
    };
  },
  "app.js": function (exports, require) {
    const squareArea = require("square.js").default;
    const circleArea = require("circle.js").default;
    console.log("Area of square: ", squareArea(5));
    console.log("Area of circle", circleArea(5));
  },
};

webpackStart({
  modules,
  entry: "app.js",
});
```

특이점은 다음과 같다.

- module map의 사용
  - module map: 모듈의 이름과 모듈 함수를 연결짓는 딕셔너리
- 각 모듈은 함수로 묶여있다.
  - 이 함수를 module factory function이라 한다.
  - 각 함수는 각 모듈만의 scope를 만든다.
  - 함수는 `exports`, `require` 매개변수를 갖는다.
- 애플리케이션은 `webpackStart`를 통해 시작된다.
  - `webpackStart` 함수는 module map을 이용해 모든 것을 하나로 묶는 함수이다.
  - `webpackStart` 함수는 런타임이라 불리기도 한다.

## `webpackStart` 함수

```jsx
function webpackStart({ modules, entry }) {
  const moduleCache = {};
  const require = (moduleName) => {
    // 캐시에 존재한다면 캐시 버전을 반환한다.
    if (moduleCache[moduleName]) {
      return moduleCache[moduleName];
    }
    const exports = {};
    // 상호 참조(circular dependencies)로 인해 발생하는 무한 루프를 막는다.
    moduleCache[moduleName] = exports;

    // 모듈을 require 한다.
    // export되었다면 이는 "exports"로 할당된다.
    modules[moduleName](exports, require);
    return moduleCache[moduleName];
  };

  // 프로그램을 시작한다.
  require(entry);
}
```

`webpackStart`는 두 가지를 정의한다.

- `require` 함수
  - Common JS의 `require`와는 다르다.
  - 모듈 이름을 받아 모듈의 exported interface를 반환한다.
    - `circle.js`의 경우 반환 값은 `{ default: function area(radius){ ... } }`가 될 것이다.
  - expoted interface는 모듈 캐시에 캐싱된다.
    - 같은 모듈에 대하여 `require` 함수를 여러번 실행하더라도 module factory function은 한 번만 실행된다.
- module cache

# 요약

- 모듈 번들러는 자바스크립트 모듈들을 브라우저에서 실행 가능한 하나의 자바스크립트 파일로 묶는 도구이다.
- 모듈 번들러는 모듈간 의존성 문제를 대신 해결해준다.
- webpack은 module map을 사용해 모듈들을 이름에 따른 각 함수로 묶고 `webpackStart` 함수를 사용해 이를 합친다.

# 참고 자료

- [What is module bundler and how does it work?](https://lihautan.com/what-is-module-bundler-and-how-does-it-work/)
