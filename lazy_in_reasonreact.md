# Lazy in ReasonReact

## 목적
* `Lazy` 모듈을 ReasonML(이하, Reason)에서 살펴보고, ReasonReact에서 어떻게 활용할 수 있을 지 알아보겠습니다.
* `Lazy` 모듈과 비슷한 사용성을 가진 `React.Suspense`와 비교하면 어떤 차이가 있는 지 알아보겠습니다.

## Lazy 모듈에 대해
OCaml에 Lazy 모듈이 있습니다.([문서](https://caml.inria.fr/pub/docs/manual-ocaml/libref/Lazy.html)) 이 Lazy 모듈을 Reason에서도 사용할 수 있고, 컴파일러인 ReScript(구 BuckleScript)의 [문서](https://rescript-lang.org/docs/manual/v8.0.0/lazy-values)에도 언급되어 있는 것처럼, 컴파일러가 Js의 구현체로 transpile 해주는 것이죠.

OCaml의 Lazy 모듈은 deferred computation의 구현체입니다. 이를 Reason에서 아래와 같이 사용할 수 있습니다.

```reason
let expensiveCompute = () => {
  ...
  "I'm very expensive one."
}
let laziness = lazy(expensiveCompute());

Js.log(laziness)
/**
* {
*   "LAZY_DONE": false,
*   "VAL": function VAL()
* }
*/
```

나중에 evalutation하려면 `force()`로 꺼내서 쓰면 됩니다.

```reason
let deferredValue = Lazy.force(laziness)

Js.log(deferredValue)
// I'm very expensive one.
```

`lazy(expr)`의 expr에 담을 수 있는 것에는 exception도 포함됩니다.

```reason
let readFile =
  lazy({
    raise(Not_found)
  });

try (Lazy.force(readFile)) {
| Not_found => Js.log("No file")
};
```

evalutation할 때 try로 pattern matching해서 꺼내면 되는 것이죠.

## 사례

### 흔한(?) 상황
흔한 것 아닌 흔한 것 같기도 한 흔하지 않은 상황을 가정해보겠습니다. 꽤 긴 시간이 필요한 연산을 하나 집어넣어서 사례를 만들어보겠습니다.

```reason
// LazyExample.re

[@react.component]
let make = () => {
  let (requiredText, setRequiredText) = React.useState(_ => "");
  let (text, setText) = React.useState(_ => "");

  let handleAnotherOnChage = e => {
    e->ReactEvent.Synthetic.stopPropagation;
    e->ReactEvent.Synthetic.preventDefault;
    let value = e->ReactEvent.Synthetic.target##value;
    setRequiredText(_ => value);
  };
  let handleOnChage = e => {
    e->ReactEvent.Synthetic.stopPropagation;
    e->ReactEvent.Synthetic.preventDefault;
    let value = e->ReactEvent.Synthetic.target##value;
    setText(_ => value);
  };

  // 값비싼 연산 중...
  let upperText = Helper.compute(text);
  
  <div className=Styles.LazyExample.container>
    <p> requiredText->React.string </p>
    <input
      type_="text"
      onChange=handleAnotherOnChage
      className=Styles.LazyExample.input
      placeholder="another"
    />
    
    <p> upperText->React.string </p>
    
    <input
      type_="text"
      onChange=handleOnChage
      className=Styles.LazyExample.input
      placeholder="memoized"
    />
  </div>;
};
```

사례를 최대한 단순하게 만들어보았습니다만, 이 사례에서 보면 Helper.compute라는 함수는 text라는 input의 입력 값을 받아서 upperCase로 만들어주는 연산을 수행합니다.

```reason
let compute = t =>
  // 입력값이 없을 때는 비싼 연산을 하지 않게
  if (t == "") {
    "";
  } else {
    let result =
      Belt.Array.make(100000000, t)  // 🤑
      ->Belt.Array.reduce("", (acc, item) => {
          acc == item ? acc : item->Js.String2.toUpperCase
        });
    result;
  };
```

이런 경우, 두번째 input form에 값을 입력하면 -> state가 바뀌고 -> re-rendering -> compute함수이 호출되면서 DOM에는 아무런 반응이 없는 시간이 몇 초 지난 후에 A라고 표시가 됩니다. 이건 이 연산과 아무런 상관이 없는 첫번째 input form에도 영향을 주어서, 첫번째 input form에도 어떤 값을 입력하면 똑같이 수초 후에 DOM에 표시가 되게 됩니다. state가 변하면서 compute 함수가 매번 호출되기 때문이죠.

React.API 중에 이럴 경우를 위해 `React.useMemo`가 있어서, 이런 문제를 간단히 해결할 수 있습니다.

```reason
  // let upperText = Helper.compute(text);
  let memoizedUpperText =
    React.useMemo1(_ => Helper.compute(text), [|text|]);

  // <p> upperText->React.string </p>
  <p> memoizedUpperText->React.string </p>
```

이렇게 바꾸면, `Helper.compute`함수는 text에만 dependency를 가지게 되고, text외에 다른 state의 변화가 유발하는 re-rendering에 대해서는 함수를 호출하지 않고 가장 마지막에 연산해뒀던 값을 반환하게 됩니다. 멋지죠! 그래서 첫번째 input form에 값을 입력해도 DOM은 빠르게 반응합니다. 물론 두번째 input form은 text state를 변화시키기 때문에 두번째 input form에 값을 입력하면 `Helper.compute`함수를 호출하고 컴포넌트가 바빠지게 됩니다.

같은 상황을 Lazy를 이용해서 풀어보면,

```reason
  // let upperText = Helper.compute(text);
  // let memoizedUpperText =
  //   React.useMemo1(_ => Helper.compute(text), [|text|]);
  let lazyUpperText = lazy(Helper.compute(text));

  // <p> upperText->React.string </p>
  // <p> memoizedUpperText->React.string </p>
  <p> {Lazy.force(lazyUpperText)->React.string} </p>
```

이상하게도 `React.useMemo`를 사용하기 전과 똑같습니다. 왜냐하면 리액트 함수 컴포넌트 안에서 state가 변할 때마다 `Helper.compute`함수의 지연된 값을 만들지만, 그 아래 JSX 블럭 안에서 `.force` evaluation이 바로 바로 일어나버리는 것이죠.

그렇다면, `let lazyUpperText = lazy(Helper.compute(text))`를 리액트 컴포넌트 밖으로 빼볼까요? 물론 text는 인자로 넘겨줄 수가 없으니 값을 바꾸겠습니다.

```reason
// 리액트 컴포넌트 밖
let lazyUpperText = lazy(Helper.compute("I'm just lazy"));

// 리액트 컴포넌트 안
{switch(checked){
  |true => Lazy.force(lazyUpperText)->React.string
  |false => React.null
}}

<label>
  <input type_="checkbox" checked onChange=handleOnCheck />
  {j| show|j}->React.string
</label>
```

show라는 체크박스를 하나 만들었고, 체크를 하면 몇 초가 지난 후에 값이 DOM에 잘 그려집니다. 그리고 체크를 풀고 다시 켰을 때, 처음과는 다르게 빠르게 그 값이 그려지죠! 컴포넌트 밖에서 lazy value를 만들 때 deferred computation(지연 연산)을 하고 싶은 인자를 전달해서 lazy value를 만들고, 그것을 컴포넌트 내에서 필요할 때 evalutation하면 좋을 것 같습니다.

그래서 ReScript 문서에서도 유용한 활용 예를 아래와 같이 들고 있습니다.

* DOM의 동일한 tree를 탐색하는 경우
* 변화하지 않을 file system 관련 연산
* 항상 같은 값을 반환하는 API call

우리의 예에서 text라는 값이 계속 변하면 지연은 되지만 매번 연산하는데 오래 걸릴테니, 같은 반환값을 기대할 수 있는 곳에서 사용하라는 것으로 생각됩니다.

### Lazy vs. React.Suspense

지금까지 살펴본 Lazy와 비슷한 목적으로 사용하는 React API가 하나 있습니다. 아직 Experimental 단계여서 Production에서는 사용하지 않기를 권고하고 있는 `React.Suspense`가 바로 그것인데요.

우선, ReasonReact는 Suspense는 binding해두어서 바로 쓸 수 있지만, `React.lazy`는 binding해두지 않아서 직접 binding을 해보겠습니다. Binding하는 코드는 https://github.com/jchavarri/reason-react-lazy-loading 의 소스코드를 참고하였습니다.

```reason
module ExpensiveLazy = {
  [@bs.val]
  external importComponent:
    ([@bs.ignore] (Js.t('a) => React.element), string) =>
    Js.Promise.t(Js.t('a) => React.element) =
    "import";

  [@bs.module "react"]
  external lazy_: (unit => Js.Promise.t('a)) => 'a = "lazy";

  module type T = (module type of Expensive);

  let unsafePlaceholder: module T = [%raw {|{}|}];

  module UnsafePlaceholder = (val unsafePlaceholder);

  let makeProps = UnsafePlaceholder.makeProps;
  let make = lazy_(_ => importComponent(UnsafePlaceholder.make, "./Expensive.bs.js"));
};

module Expensive = {
  [@react.component]
  let make = () => {
    let upperText = Helper.compute("i'm lazy");
    <div> upperText->React.string </div>;
  };

  let default = make;
}

module LazyExample = {
  let make = () => {
    <React.Suspense fallback={<div> {j|loading...|j}->React.string </div>}>
      <ExpensiveLazy />
    </React.Suspense>
  }
}
```

이렇게 하면 loading...이라는 표시가 뜨고 몇 초가 지나고 연산이 다 끝난 `Expensive` 컴포넌트가 화면에 표시 됩니다. 그런데 첫번째 input form의 성능은 여전히 state가 바뀔 때마다, 한 번 DOM에 올라온 Expensive 컴포넌트가 re-rendering되면서 매우 느린 것을 볼 수 있습니다.

사실 이것은 의도한대로 작동하는 것이라고 볼 수 있는데요. React.Suspense는 문서에서도 설명되어있는 것과 같이 code splitting과 같은 컴포넌트 dynamic importing에 그 목적이 있는 것으로 생각됩니다. 그래서 한 번 DOM에 그려지고 나면 일반 컴포넌트와 동일하게 작동하리라 보는 것이 맞겠죠.

이번에는 Lazy 모듈을 사용해서 구현해보겠습니다.

```reason
let expensiveComponent = lazy(<Expensive />);

module LazyExample = {
  let make = () => {
    let handleOnCheck = e => {
        let value = e->ReactEvent.Synthetic.target##checked;
        setChecked(value);
      }

    {checked ? Lazy.force(expensiveComponent) : React.null}

    <label>
      <input type_="checkbox" checked onChange=handleOnCheck></input>
        {j| show|j}->React.string
    </label>
  }
}
```

아쉽게도 지연은 잘되지만, 한 번 평가된 `<Expensive />`라는 컴포넌트가 DOM에 올라올 때마다 그려질 때마다 `Helper.compute`함수가 호출되네요.

이렇게 작동하는 이유는 ReScript Lazy의 구현체인 `camlinternalLazy.js`를 살펴보면 그 이유를 알 수 있습니다.

```js
function force(lzv) {
  if (lzv.LAZY_DONE) {
    return lzv.VAL;
  } else {
    var closure = lzv.VAL;
    lzv.VAL = raise_undefined;
    try {
      return forward_with_closure(lzv, closure);
    }
    catch (e){
      lzv.VAL = (function () {
          throw e;
        });
      throw e;
    }
  }
}
```

결국 처음에 살펴본 바와 같이 `lazy(expr)`이 반환해주는 건 LAZY_DONE과 VAL라는 키를 갖는 Js 객체가 나오고, LAZY_DONE이 false이면 force 함수가 VAL에 담긴 expression을 evalutation하여 VAL를 대체하고 LAZY_DONE을 true로 변경해줍니다. 그래서 그 이후에 같은 lazy 객체를 force하면 가장 최근에 evaluation한 VAL 값을 반환해주는 것이죠.

## 결론

`Lazy`를 사용하면 `React.useMemo`나 `React.Suspense`와는 다른 방식으로 구현을 할 수 있다는 점이 매우 흥미롭습니다. 리액트 컴포넌트 밖에서 복잡하고 시간이 오래 걸릴 연산을 해야하는데 그것을 지연시켜서 호출하고 evaluation을 하고 싶다면, 그리고 한 번 evaluation을 한 값을 반복적으로 사용하는 경우라면, React API가 아닌 lazy를 이용해서 구현해보는 것도 좋을 것 같습니다.

### References
* [Module Lazy - OCaml](https://caml.inria.fr/pub/docs/manual-ocaml/libref/Lazy.html)
* [Lazy Values - ReScript](https://rescript-lang.org/docs/manual/v8.0.0/lazy-values)