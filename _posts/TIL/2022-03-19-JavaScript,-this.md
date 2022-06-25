---
title: "JavaScript, this"
excerpt: "JavaScript의 this에 대하여"
categories:
  - TIL
tags:
  - JavaScript

last_modified_at: 2022-03-19
---

# this

매서드 내부의 객체를 접근할 때 `this` 키워드를 사용한다. 일반적으로 자바스크립트의 `this`는 함수가 호출되는 방식에 따라 값이 달라진다. 따라서 동일한 함수라도 만약 다른 객체에서 호출했다면 `this`가 참조하는 값이 달라질 수 있다.

```javascript
let user = { name: "John" };
let admin = { name: "Admin" };

function sayHi() {
  alert(this.name);
}

user.f = sayHi;
admin.f = sayHi;

user.f(); // John  (this == user)
admin.f(); // Admin  (this == admin)
```

ES5에서는 함수 호출 방식과 무관하게 `this`가 일정한 값을 가질 수 있도록 한 `bind()` 메서드가 도입되었다. ES6에서 도입된 화살표 함수에서는 `this` 값이 lexical context로 묶이게 된다.

> **Lexical Scope**는 변수가 선언된 공간에 기반하여 결정되는 스코프이다.
>
> ```javascript
> // Define a variable in the global scope:
> const myName = "Oluwatobi";
>
> // Call myName variable from a function:
> function getName() {
>   return myName;
> }
> ```
>
> 위 코드에서 `myName`의 lexical scope는 global scope이다.
>
> ```javascript
> function getName() {
>   const myName = "Oluwatobi";
>   return myName;
> }
> ```
>
> 위 코드에서 `myName`의 lexical scope는 `getName()` 함수 내부이다.

<br>

# `bind()` 메서드

`bind()`메서드는 새로운 함수를 생성하며, 이 함수는 호출 시 함께 넘겨진 값을 `this`에 고정시켜 놓는다.

```javascript
const module = {
  x: 42,
  getX: function () {
    return this.x;
  },
};

const unboundGetX = module.getX;
console.log(unboundGetX()); // The function gets invoked at the global scope
// expected output: undefined

const boundGetX = unboundGetX.bind(module);
console.log(boundGetX());
// expected output: 42
```

<br>

# `call()` 메서드

> `func.call(thisArg[, arg1[, arg2[, ...]]])`

한 객체의 함수 또는 메서드를 다른 객체에 할당하거나 호출할 때 `call()` 메서드를 사용한다. `call()` 메서드는 함수 또는 메서드에 새로운 `this` 값을 전달하며 이를 이용해 이미 작성한 메서드를 다른 객체에서 상속받아 손쉽게 재사용 가능하다. 호출 시 첫 번째 인수는 `this`, 나머지 인수는 `func`의 인수가 된다.

```javascript
function Product(name, price) {
  this.name = name;
  this.price = price;
}

function Food(name, price) {
  Product.call(this, name, price);
  this.category = "food";
}

console.log(new Food("cheese", 5).name);
// expected output: "cheese"
```

```javascript
function sayHi() {
  alert(this.name);
}

let user = { name: "John" };
let admin = { name: "Admin" };

sayHi.call(user); // this = John
sayHi.call(admin); // this = Admin
```

<br>

# `apply()` 메서드

> func.apply(thisArg, [argsArray])

`call()` 메서드는 여러 인수를 전달받지만 `apply()` 메서드는 유사 배열 형태의 인수를 전달 받는다는 차이를 빼면 `apply()` 메서드는 `call()` 메서드와 매우 유사하다. 따라서 인수가 이터러블 형태라면 `call()` 메서드를, 유사 배열 형태라면 `apply()` 메서드를 사용한다.

```javascript
const numbers = [5, 6, 2, 3, 7];

const max = Math.max.apply(null, numbers);

console.log(max);
// expected output: 7

const min = Math.min.apply(null, numbers);

console.log(min);
// expected output: 2
```

<br>

# 화살표 함수

화살표 함수에는 `this`가 없다. 따라서 `this`를 호출할 시 화살표 함수를 둘러싸는 lexical scopre의 `this`가 사용된다. 또한 `bind()`, `call()`, `apply()` 메서드를 사용할 수 없다.

```javascript
function Person() {
  this.age = 0;

  setInterval(() => {
    this.age++; // |this|는 Person 객체를 참조
  }, 1000);
}

var p = new Person();
```

<br>

# Reference

- [this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)
- [Function.prototype.bind()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)
- [Lexical Scope in JavaScript – What Exactly Is Scope in JS?](https://www.freecodecamp.org/news/javascript-lexical-scope-tutorial/)
- [메서드와 this](https://ko.javascript.info/object-methods)
- [화살표 함수](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Functions/Arrow_functions)
