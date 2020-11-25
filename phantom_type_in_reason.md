---
title: Phantom type in ReasonML
date: 2020-11-25
description: 팬텀 타입에 대해
tags:
  - ReasonML
  - Phantom type
---

# Phantom type in ReasonML
안녕하세요. 그린랩스 웹개발팀의 문운기라고 합니다. ReasonML은 훌륭한 타입시스템을 가지고 있습니다. 이 타입시스템 안에서 사용할 수 있는 한 가지 멋지 기교(?)가 있는데, 그 중 Phantom type에 대해 알아보고 그 사용 예를 한 번 살펴보겠습니다.
## 목적
* Phantom type란 무엇인지 알아보자
* Phantom type의 사용 예를 살펴보자

## Phantom type이란?
팬텀타입이란 매개변수가 타입 선언부의 왼쪽에만 존재하는 타입을 말합니다. 이렇게 말하면 언뜻 이해가 쉽게 안될텐데요. 코드로 표현해보겠습니다.

```reason
type t('a) = string;
```

이렇게 하면 우리는 t('a)는 string타입이라고 컴파일러와 약속했습니다. 그렇기 때문에,

```reason
type t('a) = string;
type dog;
type cat;

t(dog) // string
t(cat) // string
```

`t(dog)`과 `t(cat)` 모두 string이라고 컴파일러는 처리하지만, 동시에 컴파일러는 `t(dog)`과 `t(cat)`은 엄연히 다른 타입이라고 구분해서 쓰라고 말할 것 입니다.

```reason
module type Animal = {
  type t('a);
  type dog;
  type cat;
  let makeDog: string => t(dog);
  let makeCat: string => t(cat);
  let mate: (t('a), t('a)) => string;
}

module Animal:Animal = {
  type t('a) = string;
  type dog;
  type cat;
  let makeDog = a => a;
  let makeCat = a => a;
  let mate = (a, b) => {j|$a와 $b는 이제 친구|j}
}

let mike = Animal.makeDog("Mike");
let marla = Animal.makeCat("Marla");
Js.log(Animal.mate(mike, marla)) // Error
/**
Error: This expression has type Animal.t(Animal.cat)
       but an expression was expected of type Animal.t(Animal.dog)
       Type Animal.cat is not compatible with type Animal.dog
*/
```

여기서 mate라는 함수는 같은 t('a)를 2개의 인자로 받아서 string을 반환하는 함수로 module type Animal에서 선언되었기 때문에, 다른 t(dog)과 t(cat)에 대해 타입이 틀렸다고 컴파일이 안되는 것 입니다.

그렇다면 새로운 함수를 하나 추가해볼까요?

```reason
module type Animal = {
  ...
  let interMate: (t('a), t('b)) => string;
};

module Animal = {
  let interMate = (a, b) => {j|$a와 $b도 이제 친구!|j};
}

Js.log(Animal.interMate(mike, marla)) // Ok
```

`interMate`라는 함수는 다른 2개의 타입(t('a), t('b))을 가진 인자를 받아서 string을 반환하는 함수로 타입이 정의되었기 때문에, 각각 t(dog)과 t(cat)의 타입의 인자를 받아도 컴파일이 잘 되는 것 입니다.

이렇게 팬텀 타입은 'a와 같은 매개변수를 선언부의 왼쪽에만 사용하여서, 하나의 타입이지만 똑같은 string이라는 타입의 값("mike", "marla")을 갖더라도 다른 종류의 subtype을 가질 수 있는 효과를 얻을 수 있습니다.

요약해서 말하자면 이렇습니다.

* Parameterized type, 하지만 그 parameter가 선언의 오른쪽에는 없는 타입
* Data의 representation(여기서는 string인 "mike", "marla")을 바꿀 필요 없이 subtype을 가질 수 있는 효과를 얻을 수 있다.

## 활용 예
위의 Animal의 예는 팬텀 타입을 설명하는 레퍼런스들에서 흔히 볼 수 있는 Example입니다. 흔하지만 조금 더 구체적인 예를 하나 더 들어보겠습니다. 어플리케이션을 만들 때 Form 데이터를 많이 사용하게 되는데요. 이 Form 데이터의 validation을 처리하는 부분에 대한 예 입니다.

```reason
open Belt;

module type FormData = {
  type t('a) = string;
  type validated;
  type unvalidated;
  let make: string => t(unvalidated);
  let toUpperCase: t('a) => t('a);
  let validate: t(unvalidated) => Result.t(t(validated), string);
  let isValidated: t(validated) => bool;
};

module FormData: FormData = {
  type t('a) = string;
  type validated;
  type unvalidated;
  let make = a => a;
  let toUpperCase = a => Js.String.toUpperCase(a);
  let validate = a =>
    if (String.length(a) > 3) {
      Result.Ok(a);
    } else {
      Result.Error("validation failed");
    };
  let isValidated = (_) => true;
};

let shouldBeOkay = FormData.make("should be okay"); // FormData.t(FormData.unvalidated)

switch (FormData.validate(shouldBeOkay)) {
| Ok(s) => Js.log2("trying to call API with ", s)
| Error(err) => Js.log2("error! ", err)
};
/** 출력
trying to call API with  should be okay
*/
```

통과하지 못하는 케이스를 만들어보겠습니다.

```reason
let cantBePassed = FormData.make("ok?");

switch (FormData.validate(cantBePassed)) {
| Ok(s) => Js.log2("trying to call API with ", s)
| Error(err) => Js.log2("error! ", err)
}
/** 출력
error!  validation failed
*/
```

길이가 3인 "ok?"는 통과하지 못했네요. 잘 작동합니다.

여기서 조금 더 나아가서, 만약 이 모듈을 form data를 검증하는 라이브러리로 만든다고 가정해보겠습니다. 우리가 만든 라이브러리를 사용하는 유저가 의도했건, 의도하지 않았던 간에 우회하게 하는 함수를 실수(?)로 만들었다고 상상해보겠습니다.

```reason
let byPass:string => FormData.t(FormData.validated) = a => a;
Js.log2("isValidated? ", FormData.isValidated(cantBePassed)); // 컴파일 passed 😱
```

왜 이런 일이 발생했을까요? 이걸 막을 수 있는 방법이 있습니다.

```reason
module type FormData = {
  // type t('a) = string;
  type t('a);
  ...
}
```

위와 같이 모듈 타입의 t('a)를 선언한 부분에 string을 지워버리면, `byPass`라는 함수가 컴파일이 되질 않습니다.

이렇게 하면, t('a)와 string은 엄연히 다른 타입이라고 컴파일러에게 알려주는 것이고, 그래서 `byPass`라는 함수는 string => FormData.t(FormData.validated)라는 선언을 해놓고, 왜 string => string를 구현하느냐고 컴파일 에러를 내는 것이죠. `isValidated` 함수의 오류도 같은 이유로 바로잡을 수 있습니다.

```reason
let byPass: string => FormData.t(FormData.validated) = a => a;
// Error: This expression has type string but an expression was expected of type FormData.t(FormData.validated)

Js.log2("isValidated? ", FormData.isValidated(cantBePassed));
/** Error: This expression has type FormData.t(FormData.unvalidated)
       but an expression was expected of type FormData.t(FormData.validated)
       Type FormData.unvalidated is not compatible with type
         FormData.validated
*/
```

지금 다시 돌아가서 보시면, 첫번째 예였던 module type Animal의 type t('a)는 string 없이 선언되어있는 걸 다시 확인하실 수 있을 겁니다. 그것이 mate와 interMate의 차이를 만든 것이죠.

## 마무리
팬텀 타입에 대해 두 가지 예를 가지고 살펴보았습니다. 팬텀 타입은 ReasonML에만 있는 것이 아니라 Haskell, Rust, Swift 등 다른 언어에서도 활용할 수 있다고 합니다. 사실 팬텀 타입은 타입을 이용한 기교에 속하는 것이라 생각되어서, 꼭 알아야만 하는 것은 아니지만, 타입 시스템 안에서 조금 더 재미있는 코드를 만들 수 있을 것 같습니다.

### References
* Phantom Types in ReasonML https://medium.com/reasontraining/phantom-types-in-reasonml-1a4cfc18d99, https://gist.github.com/busypeoples/3a28d039272ec3eb33ca2fc6b32dafc7
* Phantom Type https://wiki.haskell.org/Phantom_type