---
layout: post
title: "[Daily morning study] JavaScript 프로토타입(Prototype)과 상속 체계"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 프로토타입 기반 언어

JavaScript는 클래스 기반 언어가 아닌 **프로토타입 기반 언어**다. ES6에서 `class` 문법이 추가됐지만 내부적으로는 여전히 프로토타입으로 동작한다.

## 프로토타입(Prototype)이란

모든 객체는 자신의 부모 역할을 하는 객체를 참조하는 `[[Prototype]]`이라는 내부 슬롯을 가진다.

- `__proto__` 프로퍼티로 접근할 수 있지만 비표준이므로 사용 비권장
- `Object.getPrototypeOf(obj)` 사용 권장

```js
const obj = {};
console.log(Object.getPrototypeOf(obj) === Object.prototype); // true
```

## 프로토타입 체인

프로퍼티나 메서드를 탐색할 때 현재 객체부터 시작해 프로토타입 체인을 따라 올라가며 찾는다. 체인의 끝은 `Object.prototype`이고 그다음은 `null`이다.

```
arr -> Array.prototype -> Object.prototype -> null
```

```js
const arr = [1, 2, 3];
// arr.push는 Array.prototype.push
// arr.hasOwnProperty는 Object.prototype.hasOwnProperty
// 직접 정의하지 않았지만 체인을 통해 접근 가능
```

## 함수의 prototype 프로퍼티

함수는 `prototype` 프로퍼티를 가진다. 이 함수를 `new`로 호출하면 생성된 객체의 `[[Prototype]]`이 해당 함수의 `prototype` 객체가 된다.

```js
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function () {
  return `${this.name}이 소리를 냅니다.`;
};

const dog = new Animal('강아지');
console.log(dog.speak()); // 강아지이 소리를 냅니다.
console.log(Object.getPrototypeOf(dog) === Animal.prototype); // true
```

`speak`는 `dog` 자체가 아닌 `Animal.prototype`에 정의되어 있지만, 체인을 통해 접근 가능하다. 모든 인스턴스가 메서드를 공유하므로 메모리 효율적이다.

## 프로토타입 체인으로 상속 구현

```js
function Dog(name, breed) {
  Animal.call(this, name); // 부모 생성자 호출로 name 초기화
  this.breed = breed;
}

// Dog.prototype의 [[Prototype]]을 Animal.prototype으로 설정
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog; // constructor 참조 복원

Dog.prototype.bark = function () {
  return '멍멍!';
};

const myDog = new Dog('바둑이', '진돗개');
console.log(myDog.speak()); // 바둑이이 소리를 냅니다. (Animal.prototype의 메서드)
console.log(myDog.bark());  // 멍멍!
```

`Object.create(Animal.prototype)`을 쓰는 이유: `new Animal()`을 쓰면 Animal 생성자가 실행되어 의도치 않은 부작용이 생길 수 있기 때문이다.

## ES6 class — 문법적 설탕(Syntactic Sugar)

ES6 `class`는 프로토타입 기반 상속을 더 직관적인 문법으로 표현한 것이다. 내부 동작은 동일하다.

```js
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name}이 소리를 냅니다.`;
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // 부모 생성자 호출 필수
    this.breed = breed;
  }

  bark() {
    return '멍멍!';
  }
}

const myDog = new Dog('바둑이', '진돗개');
console.log(myDog instanceof Dog);    // true
console.log(myDog instanceof Animal); // true
```

`extends`를 쓰면 내부적으로 `Dog.prototype`의 `[[Prototype]]`이 `Animal.prototype`을 참조하도록 연결된다.

## instanceof

오른쪽 피연산자의 `prototype`이 왼쪽 객체의 프로토타입 체인에 존재하는지 확인한다.

```js
console.log(myDog instanceof Dog);    // true
console.log(myDog instanceof Animal); // true
console.log(myDog instanceof Object); // true — 모든 객체는 Object의 인스턴스
```

## hasOwnProperty

프로토타입 체인이 아닌 객체 자신이 직접 가진 프로퍼티인지 확인한다.

```js
console.log(myDog.hasOwnProperty('name'));  // true — 자신의 프로퍼티
console.log(myDog.hasOwnProperty('speak')); // false — Animal.prototype의 프로퍼티
```

`for...in` 루프는 프로토타입 체인의 열거 가능한 프로퍼티까지 순회하므로, 자신의 프로퍼티만 처리하려면 `hasOwnProperty`로 필터링해야 한다.

## Object.create()

특정 객체를 프로토타입으로 하는 새 객체를 만든다. 생성자 없이 순수하게 프로토타입 체인만 설정할 때 유용하다.

```js
const proto = {
  greet() {
    return `안녕, ${this.name}`;
  },
};

const obj = Object.create(proto);
obj.name = '철수';
console.log(obj.greet()); // 안녕, 철수
```

`Object.create(null)`을 사용하면 프로토타입이 없는 순수 딕셔너리 객체를 만들 수 있다. `toString` 같은 내장 메서드가 없으므로 순수 키-값 저장소로 쓸 때 적합하다.

## 프로토타입 오염(Prototype Pollution) 주의

```js
// 절대 하면 안 됨
Object.prototype.hello = function () { return 'hello'; };
// 이후 모든 객체에 hello 메서드가 생겨 예측 불가능한 동작 발생
```

외부 입력으로 `__proto__`, `constructor`, `prototype` 같은 키를 그대로 객체에 할당하면 프로토타입 오염이 발생할 수 있다. JSON 파싱 결과를 그대로 객체에 병합할 때 특히 주의해야 한다.

## 정리

| 개념 | 설명 |
|------|------|
| `[[Prototype]]` | 객체의 내부 슬롯, 부모 프로토타입 참조 |
| `prototype` 프로퍼티 | 함수만 가지는 프로퍼티, `new` 인스턴스의 프로토타입이 됨 |
| 프로토타입 체인 | 프로퍼티 탐색 시 체인을 따라 올라가는 메커니즘 |
| `Object.create(proto)` | 지정한 객체를 프로토타입으로 하는 새 객체 생성 |
| `class` / `extends` | 프로토타입 기반 상속의 문법적 설탕 |
| `instanceof` | 프로토타입 체인에서 특정 prototype 존재 여부 확인 |
| `hasOwnProperty` | 체인이 아닌 자신의 프로퍼티인지 확인 |
