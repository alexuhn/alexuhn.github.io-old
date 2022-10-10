---
title: "[모던 자바스크립트 Deep Dive]49장 Babel과 Webpack을 이용한 ES6+/ES.NEXT 개발 환경 구축"
excerpt: "Babel과 Webpack을 이용한 ES6+/ES.NEXT 개발 환경 구축 대하여"
categories:
  - book_notes
tags:
  - 모던 자바스크립트 Deep Dive
---

- 구형 브라우저에서도 ES6+와 ES.NEXT의 최신 ECMAScript 사양으로 작성된 코드가 잘 동작하는 개발 환경 구축이 필요하다.
    - ES.NEXT: 제안 단계에 있는 ES 제안 사양
- 대부분의 프로젝트는 모듈을 사용하므로 모듈 로더가 필요하다.
    - 아직은 ES6 모듈(ESM)보다는 별도의 모듈 로더를 사용
        - IE를 포함한 구형 브라우저는 ESM을 지원하지 않기 때문
        - ESM을 사용해도 여전히 트랜스파일링이나 번들링이 필요하기 때문
        - bare import 등 ESM이 아직 지원하지 않는 기능이 있고, 해결되지 않은 몇 가지 이슈가 존재하기 때문
- 따라서 트랜스파일러인 Babel과 모듈 번들러인 Webpack을 이용해 ES6+/ES.NEXT 개발 환경 구축할 수 있다.

# 49.1 Bebel

```jsx
// ES6의 화살표 함수 + ES7의 지수 연산자
[1, 2, 3].map(n => n ** n);
```

```jsx
// Babel이 변환한 ES5 사양의 코드
"use strict";

[1, 2, 3].map(function (n) {
  return Math.pow(n, n);
});
```

- Babel은 ES6+/ES.NEXT로 구현된 최신 사양의 소스 코드를 구형 브라우저에서도 동작하는 ES5 사양의 소스코드로 변환(트랜스파일링)할 수 있다.

## 49.1.1 Babel 설치

```bash
$ npm install --save-dev @babel/core @babel/cli
```

## 49.1.2 Babel 프리셋 설치와 `babel.config.json` 설정 파일 작성

- Babel 프리셋
    - `@babel/preset-env`
    - 함께 사용되어야 하는 Babel 플러그인을 모아둔 것
    - 필요한 플러그인을 프로젝트 지원 환경에 맞춰 동적으로 결정
        - 프로젝트 지원 환경은 [Browserslist](https://github.com/browserslist/browserslist) 형식으로 [`.browserslistrc` 파일에 상세히 설정 가능](https://babeljs.io/docs/en/babel-preset-env#browserslist-integration)
        - 프로젝트 지원 환경 설정 작업 생략 시 기본값으로 설정
- Babel이 제공하는 공식 Babel 프리셋
    - `@babel/preset-env`
    - `@babel/preset-flow`
    - `@babel/preset-react`
    - `@babel/preset-typescript`
1. 기본 설정으로 Babel 프리셋 설치
    - 기본 설정은 모든 ES6+/ES.NEXT 사양의 소스코드 변환
    
    ```bash
    $ npm install --save-dev @babel/preset-env
    ```
    
2. `babel.config.json` 설정 파일 작성
    
    ```json
    {
      "presets": ["@babel/preset-env"]
    }
    ```
    
    - `@babel/preset-env`를 사용한다는 의미

## 49.1.3 트랜스파일링

1. `npm scripts`에 Babel CLI 명령어 등록
    
    ```json
    {
      "name": "esnext-project",
      "version": "1.0.0",
      "scripts": {
        "build": "babel src/js -w -d dist/js"
      },
      "devDependencies": {
        "@babel/cli": "^7.19.3",
        "@babel/core": "^7.19.3",
        "@babel/preset-env": "^7.19.3"
      }
    }
    ```
    
    - 타깃 폴더 `scr/js`에 있는 모든 자바스크립트 파일을 트랜스파일링한 후 그 결과물을 `dist/js`에 저장
    - `-w`(`--watch` 축약형): 타깃 폴더에 있는 모든 자바스크립트 파일의 변경을 감지하여 자동으로 트랜스파일
    - `-d`(`--out-dir` 축약형): 트랜스파일링된 결과물이 저장될 폴더를 지정, 지정된 폴더가 없으면 자동으로 생성
2. `lib.js`와 `main.js` 추가
    
    ```jsx
    // src/js/lib.js
    
    export const pi = Math.PI; // ES6 모듈
    
    export function power(x, y) {
      return x ** y; // ES7: 지수 연산자
    }
    
    // ES6 클래스
    export class Foo {
      #private = 10; // stage 3: 클래스 필드 정의 제안
    
      foo() {
        // stage 4: 객체 Rest/Spread 프로퍼티 제안
        const { a, b, ...x } = { ...{ a: 1, b: 2 }, c: 3, d: 4 };
        return { a, b, x };
      }
    
      bar() {
        return this.#private;
      }
    }
    ```
    
    ```jsx
    // src/js/main.js
    
    import { Foo, pi, power } from "./lib";
    
    console.log(pi);
    console.log(power(pi, pi));
    
    const f = new Foo();
    console.log(f.foo());
    console.log(f.bar());
    ```
    
3. 트랜스파일링 실행
    
    ```bash
    $ npm run build
    ```
    
4. 트랜스파일링된 `main.js` 실행
    
    ```bash
    $ node dist/js/main
    3.141592653589793
    36.4621596072079
    { a: 1, b: 2, x: { c: 3, d: 4 } }
    10
    ```
    

## 49.1.4 Babel 플러그인 설치

- 설치가 필요한 Babel 플러그인은 Babel 홈페이지에서 검색 가능
- 설치한 플러그인은 `babel.config.json` 설정 파일에 추가해야 한다.
    
    ```json
    {
      "presets": ["@babel/preset-env"],
      "plugins": ["@babel/plugin-proposal-class-properties"]
    }
    ```
    

## 49.1.5 브라우저에서 모듈 로딩 테스트

- 위 예제의 모듈 기능은 Node.js 환경에서 동작한다.
- Babel이 모듈을 트랜스파일링한 것도 Node.js가 기본 지원하는 CommonJS 방식의 모듈 로딩 시스템을 따른 것이다.
    
    ```jsx
    // dist/js/main.js
    
    "use strict";
    
    var _lib = require("./lib");
    
    // src/js/main.js
    console.log(_lib.pi);
    console.log((0, _lib.power)(_lib.pi, _lib.pi));
    var f = new _lib.Foo();
    console.log(f.foo());
    console.log(f.bar());
    ```
    
    - 브라우저는 CommonJS 방식의 `require` 함수를 지원하지 않으므로 트랜스파일링된 결과를 실행하면 에러가 발생한다.
        
        ```jsx
        <!DOCTYPE html>
        <html>
        <body>
          <script src="dist/js/lib.js"></script>
          <script src="dist/js/main.js"></script>
        </body>
        </html>
        ```
        
        ![html1]({{ site.url }}{{ site.baseurl }}/assets/images/posts/book_notes/모던_자바스크립트_Deep_Dive/2022-10-10-49장-Babel과-Webpack을-이용한-ES6+-ESNEXT-개발-환경-구축/1.png){: .align-center}
        
- 브라우더의 ES6 모듈(ESM)을 사용하도록 Babel 설정을 할 수 있으나 ESM은 앞서 살펴본 문제가 있다. 따라서 Webpack을 통해 이를 해결하도록 한다.

# 49.2 Webpack

- 의존 관계에 있는 자바스크립트, CSS, 이미지 등의 리소스를 하나 또는 여러 개의 파일로 번들링하는 모듈 번들러
- 의존 모듈이 하나의 파일로 번들링되므로 별도의 모듈 로더가 필요 없다.
- 자바스크립트 파일을 하나로 번들링하므로 HTML 파일에서 자바스크립트 로드를 위해 여러 개의 `script` 태그를 사용할 필요 없다.

## 49.2.1 Webpack 설치

```bash
$ npm install --save-dev webpack webpack-cli
```

## 49.2.2 babel-loader 설치

- Webpack이 모듈을 번들링 할 때 Babel을 사용해 ES6+/ES.NEXT 사양의 소스코드를 ES5 사양의 소스코드로 트랜스파일링하게 한다.
    
    ```bash
    $ npm install --save-dev babel-loader
    ```
    
- `npm scripts`를 수정해 Webpack을 실행하도록 한다.
    
    ```json
    {
      "name": "esnext-project",
      "version": "1.0.0",
      "scripts": {
        "build": "webpack -w"
      },
      "devDependencies": {
        "@babel/cli": "^7.19.3",
        "@babel/core": "^7.19.3",
        "@babel/preset-env": "^7.19.3",
        "babel-loader": "^8.2.5",
        "webpack": "^5.74.0",
        "webpack-cli": "^4.10.0"
      }
    }
    ```
    

## 49.2.3 `webpack.config.js` 설정 파일 작성

- Webpack은 실행될 때 `webpack.config.js` 파일을 참조한다.
    
    ```jsx
    // webpack.config.js
    
    const path = require('path');
    
    module.exports = {
      // entry file
      // https://webpack.js.org/configuration/entry-context/#entry
      entry: './src/js/main.js',
      // 번들링된 js 파일의 이름(filename)과 저장될 경로(path)를 지정
      // https://webpack.js.org/configuration/output/#outputpath
      // https://webpack.js.org/configuration/output/#outputfilename
      output: {
        path: path.resolve(__dirname, 'dist'),
        filename: 'js/bundle.js'
      },
      // https://webpack.js.org/configuration/module
      module: {
        rules: [
          {
            test: /\.js$/,
            include: [
              path.resolve(__dirname, 'src/js')
            ],
            exclude: /node_modules/,
            use: {
              loader: 'babel-loader',
              options: {
                presets: ['@babel/preset-env'],
                plugins: ['@babel/plugin-proposal-class-properties']
              }
            }
          }
        ]
      },
      devtool: 'source-map',
      // https://webpack.js.org/configuration/mode
      mode: 'development'
    };
    ```
    
1. Webpack 실행
    
    ```jsx
    $ npm run build
    ```
    
2. 브라우저에서 번들링된 결과물 실행
    
    ```jsx
    <!DOCTYPE html>
    <html>
    <body>
      <script src="./dist/js/bundle.js"></script>
    </body>
    </html>
    ```
    
    ![html2]({{ site.url }}{{ site.baseurl }}/assets/images/posts/book_notes/모던_자바스크립트_Deep_Dive/2022-10-10-49장-Babel과-Webpack을-이용한-ES6+-ESNEXT-개발-환경-구축/2.png){: .align-center}
    

## 49.2.4 babel-polyfill 설치

- Babel에서 ES5 사양으로 트랜스파일링해도 브라우저가 지원하지 않는 코드가 남아있을 수 있다.
    - ES6에서 추가된 `Promise`, `Object.assign`, `Array.from` 등은 ES5에 대체할 기능이 없어 트랜스파일링되지 못하고 그대로 남는다.
    
    ```jsx
    // src/js/main.js
    
    import { pi, power, Foo } from './lib';
    
    console.log(pi);
    console.log(power(pi, pi));
    
    const f = new Foo();
    console.log(f.foo());
    console.log(f.bar());
    
    // polyfill이 필요한 코드
    console.log(new Promise((resolve, reject) => {
      setTimeout(() => resolve(1), 100);
    }));
    
    // polyfill이 필요한 코드
    console.log(Object.assign({}, { x: 1 }, { y: 2 }));
    
    // polyfill이 필요한 코드
    console.log(Array.from([1, 2, 3], v => v + v));
    ```
    
    ```jsx
    // dist/js/bundle.js
    
    ...
    console.log(new Promise(function (resolve, reject) {
      setTimeout(function () {
        return resolve(1);
      }, 100);
    })); // polyfill이 필요한 코드
    
    console.log(Object.assign({}, {
      x: 1
    }, {
      y: 2
    })); // polyfill이 필요한 코드
    
    console.log(Array.from([1, 2, 3], function (v) {
      return v + v;
    }));
    ...
    ```
    
- 구형 브라우저에서 이를 지원하기 위해서 `@babel/polyfill`을 설치
    
    ```bash
    $ npm install @babel/polyfill
    ```
    
    - 실제 운영 환경에서도 사용되어야 하므로 개발용 의존성 설치 옵션(`--save-dev`)을 지정하지 않는다.
- ES6의 `import`를 사용하는 경우 진입점의 선두에서 폴리필 로드
    
    ```jsx
    // src/js/main.js
    
    import "@babel/polyfill";
    import { pi, power, Foo } from './lib';
    ...
    ```
    
- Webpack을 사용하는 경우 `webpack.config.js` 파일의 `entry` 배열에 폴리필을 추가
    
    ```jsx
    // webpack.config.js
    
    const path = require('path');
    
    module.exports = {
      // entry file
      // https://webpack.js.org/configuration/entry-context/#entry
      entry: ['@babel/polyfill', './src/js/main.js'],
    ...
    ```
    
    ```jsx
    // dist/js/bundle.js
    
    /******/ (() => { // webpackBootstrap
    /******/ 	var __webpack_modules__ = ({
    
    /***/ "./node_modules/@babel/polyfill/lib/noConflict.js":
    /*!********************************************************!*\
      !*** ./node_modules/@babel/polyfill/lib/noConflict.js ***!
      \********************************************************/
    /***/ ((__unused_webpack_module, __unused_webpack_exports, __webpack_require__) => {
    ...
    ```