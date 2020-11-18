# Lazy in ReasonReact
ReasonML의 `Lazy` 모듈을 이용한 Memoization과 Lazy loading을 위한 실험

## 목적
* OCaml/ReasonML의 `Lazy` 모듈에 대해 알아보고
* Lazy 모듈을 이용해서 `React.useMemo`의 메모이제이션, 혹은 `React.Suspense`를 구현할 수 있을 지에 대한 실험
* Lazy 모듈을 이용해서 조금 더 `Reasonable` 혹은 우아하게 구현할 수 있을지도 모른다는 기대감에 대한 실험

## 출발
OCaml에는 Lazy 모듈이 있습니다.([문서](https://caml.inria.fr/pub/docs/manual-ocaml/libref/Lazy.html)) 이에 대한 구현체로서 ReasonML의 컴파일러인 ReScript(구 BuckleScript)의 [문서](https://rescript-lang.org/docs/manual/v8.0.0/lazy-values)에도 언급되어 있는 것처럼, ReasonML에서도 사용할 수 있습니다. 이 `Lazy` 모듈로 이 둘을 흉내내거나 혹은 더 우아하게 구현할 수 있을 지에 대해 알아보겠습니다.

## Module Lazy에 대해
OCaml의 Lazy 모듈은 deferred computation의 구현체입니다. 사실 여기까지는 아 그런가부다 했는데, ReScript에서는 Lazy Values에 대해 메모이제이션의 구현체라고 합니다. `lazy(expr)`하면 아래와 같은 Js 객체가 생성됩니다.

```reason
let laziness = lazy(expensiveCompute());
```

```js
{
  "LAZY_DONE": false,
  "VAL": function VAL()
}
```

나중에 이것을 사용하려면 `force()`로 꺼내서 쓰면 됩니다.

```reason
let deferredValue = Lazy.force(laziness)
```

## 사례

### 흔한(?) 상황
흔한 것 아닌 흔한 것 같기도 한 흔하지 않은 상황을 가정해보겠습니다. 꽤 긴 시간이 필요한 연산을 하나 집어넣어서 사례를 만들어보겠습니다.

```reason
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

이런 경우, 두번째 input form에 a라는 값을 입력하는 순간 compute함수가 연산을 시작하면서 DOM에는 아무런 반응이 없는 시간이 몇 초 지난 후에 A라고 표시가 됩니다. 이건 이 연산과 아무런 상관이 없는 첫번째 input form에도 영향을 주어서, 첫번째 input form에도 어떤 값을 입력하면 똑같이 수초 후에 DOM에 표시가 되게 됩니다. state가 변하면서 compute 함수가 매번 호출되기 때문이죠.

이럴 때를 위해 `React.useMemo` API를 사용하면 이 문제를 해결할 수 있습니다.

```reason
  // let upperText = Helper.compute(text);
  let memoizedUpperText =
    React.useMemo1(_ => Helper.compute(text), [|text|]);

  // <p> upperText->React.string </p>
  <p> memoizedUpperText->React.string </p>
```

이렇게 바꾸면 최소한 첫번째 input form은 Helper.compute의 연산의 굴레에서 벗어나게 되서 값을 입력하면 DOM에 바로 적용되어 보입니다. 그런데 여전히 두번째 input form은 값을 입력하면 상당한 시간이 지체된 후에 DOM에 그 입력값이 보이게 됩니다.

사실, 개인적으로는 `React.useMemo`를 이용한 memoization에서 기대했던 것은 이것과는 조금 달랐습니다. 두번째 input form에 a를 입력하면 상당한 시간이 걸리는 것은 당연하지만, b를 추가 입력한 후(상당한 시간이 걸린 후)에 다시 b를 지웠을 때(조금 전 입력값인 a와 동일한 인자가 전달 됐으니 메모이제이션 발동!??)도 상당한 시간이 걸리는 것을 확인할 수 있는데요. a라는 입력값에 대한 메모이제이션이 작동한다면 이미 이전에 입력했던 a의 결과인 A를 바로 DOM에 그려줄 거라 기대했습니다만 그렇게 작동하지 않았습니다.

혹시 Lazy로 이 아쉬운 부분을 해결할 수 있을지 시도해보면,

```reason
  // let upperText = Helper.compute(text);
  // let memoizedUpperText =
  //   React.useMemo1(_ => Helper.compute(text), [|text|]);
  let lazyUpperText = lazy(Helper.compute(text));

  // <p> upperText->React.string </p>
  // <p> memoizedUpperText->React.string </p>
  <p> {Lazy.force(lazyUpperText)->React.string} </p>
```

a라는 값을 입력하고(상당한 시간), 추가로 b라는 값을 입력(상당한 시간)한 후에 다시 b를 지우면 DOM이 또 얼어버리고 몇 초 뒤에 A라는 글자가 나타납니다. 즉, cache된 혹은 memoized된 값을 반환해서 빠르게 DOM에 값을 보여주지 않습니다. 오히려 첫번째 input form의 성능까지 떨어뜨려버리는 걸 확인할 수 있습니다.

### 중간 결론

* `React.useMemo`는 진짜 memoization을 구현한게 아니라, 가장 마지막에 연산한 값을 cache했다가 반환해주는 역할만을 한다. 그리고 deps array에 있는 값 외에는 작동하지 않게 격리해준다.
* React 컴포넌트 안에서 `Lazy`는 ReScript에서 언급한 memoize는 잘 작동하지 않는다.(~~memoize the result on the first run, and then return the memoized result on any repeated execution.~~)

### Lazy vs. React.Suspense

이번에는 Lazy 모듈의 deferred computation에 대해 알아보기 위해 `React.Suspense`와 비교를 해보겠습니다. React.Suspense는 아직 experimental 단계로 아직 Production을 위해 사용하지 말라고 React 팀에서는 권고하고 있습니다.

우선, ReasonReact는 Suspense는 binding해두어서 바로 쓸 수 있지만, `React.lazy`는 binding해두지 않아서 직접 binding을 해보겠습니다. Binding하는 코드는 https://github.com/jchavarri/reason-react-lazy-loading 의 소스코드를 참고하였습니다.

```reason
// ExpensiveLazy.re
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

// Expensive.re
[@react.component]
let make = () => {
  let upperText = Helper.compute("i'm lazy");
  <div> upperText->React.string </div>;
};

let default = make;

// LazyExample.re
<React.Suspense fallback={<div> {j|loading...|j}->React.string </div>}>
  <ExpensiveLazy />
</React.Suspense>
```

이렇게 하면 loading...이라는 표시가 뜨고 연산이 다 끝난 `Expensive` 컴포넌트가 화면에 표시 됩니다. 그런데 첫번째 input form의 성능은 여전히 state가 바뀔 때마다, 한 번 DOM에 올라온 Expensive 컴포넌트가 re-rendering되면서 매우 느린 것을 볼 수 있습니다.

사실 이것은 의도한대로 작동하는 것이라고 볼 수 있는데요. React.Suspense는 문서에서도 설명되어있는 것과 같이 code splitting과 같은 컴포넌트 dynamic importing에 그 목적이 있는 것으로 생각됩니다. 그래서 한 번 DOM에 그려지고 나면 일반 컴포넌트와 동일하게 작동하리라 보는 것이 맞겠죠.

이번에는 Lazy 모듈을 사용해서 구현해보겠습니다.

```reason
let handleOnCheck = e => {
    let value = e->ReactEvent.Synthetic.target##checked;
    setChecked(value);
  }

let expensiveComponent = lazy(<Expensive />);

{checked ? Lazy.force(expensiveComponent) : React.null}

<label>
  <input type_="checkbox" checked onChange=handleOnCheck></input>
    {j| show|j}->React.string
</label>
```

show라는 체크박스를 체크하면 지연되어있던 `<Expensive />` 컴포넌트가 연산되어 DOM에 그려지게 됩니다. 하지만 이 경우에도 DOM에 그려지고 난 뒤에 첫번째 input form에 값을 입력하면 컴포넌트의 state가 변하면서 `<Expensive />` 컴포넌트를 다시 그리기 때문에 성능이 느려지게 됩니다.

## Lazy only

이번에는 deferred computation이라는 이 용도만에 집중해보겠습니다.

```reason
let lazyUpperText = lazy(Helper.compute("I'm just lazy"));

{switch(checked){
  |true => Lazy.force(lazyUpperText)->React.string
  |false => React.null
}}

<label>
  <input type_="checkbox" checked onChange=handleOnCheck />
  {j| show|j}->React.string
</label>
```

show 체크박스를 누르면 `Helper.compute`함수의 값비싼 연산이 시작되고, 몇 초가 지난 후에 I'M JUST LAZY라는 string이 DOM에 그려집니다. 하지만, 이번에도 memoization은 작동하지 않습니다. show 체크박스를 껐다가 바로 다시 켜봐도 똑같이 오랫동안 연산을 하는 것을 확인할 수 있습니다. 그리고 첫번째 input form에 값을 입력해서 state가 바뀌면 Helper.compute 함수가 또 실행되면서 성능이 매우 떨어지는 걸 확인할 수 있습니다.

이쯤에서 ReScript Lazy의 구현체인 `camlinternalLazy.js`를 살펴보면,

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

결국 처음에 살펴본 바와 같이 `lazy()`가 반환해주는 건 LAZY_DONE과 VAL을 키로 갖는 Js 객체가 나오고, LAZY_DONE이 false이면 force 함수가 VAL의 expression을 evalutation하여 VAL를 대체하고 LAZY_DONE을 true로 변경해줍니다. 그래서 그 이후에 같은 lazy 객체를 force하면 가장 최근에 evaluation한 cache된 값을 반환해주기 때문에 memoization과 비슷한 구현체라고 ReScript 문서는 설명하는 것으로 생각됩니다. 엄밀히 말하면 memoization은 아니고, 가장 최근의 evalutation 결과를 반환해주는 것이죠.

하지만 이렇게 작동하게 하려면, lazy 선언부를 `LazyExample.re` 컴포넌트 밖에 전역으로 만들어서 사용해야합니다. 그렇지 않고 컴포넌트 안에 담으면 state가 변하면서 lazy가 다시 호출되면서 매번 새로운 lazy 객체를 만들고, 가장 최근에 evaluation한 값은 사라지고 매번 다시 evalutation 해야하는 초기 값이 매번 생성되는 것이죠. 그런데 이렇게 전역으로 선언해서 사용하면, 컴포넌트에서 사용하는 state와 같은 변수를 인자로 넘겨주는 값비싼 연산 같은 곳에는 사용하기 부적합하다는 결론에 다다릅니다.

## 결론

엄밀히 말하면, deferred computation과 memoization은 다른 기능입니다. ReScript의 Lazy Values라는 설명에서 이 둘이 모두 언급되어 있어서 혹시나 하는 마음에 시작한 실험이었는데, 문서에서 설명한 것처럼 두가지를 모두 구현해놓긴 했습니다. 하지만, 정작 React 컴포넌트 안에서 사용하기에는 `Lazy`/`React.useMemo`/`React.Suspense`/`React.lazy` 그 어떤 것도 기대만큼의 memoization을 구현할 수는 없었습니다.

그래서 더욱 아쉬운 건, 애플리케이션을 개발하는 입장에서 deferred computation과 memoization은 둘 중 하나를 포기하기 어려운 것들 입니다. 값비싼 연산 혹은 비동기 작업이 있다면, 이를 제어하고(defered) 한 번 연산한 결과를 재활용(memoization)할 수 있다면 더 우아한 어플리케이션을 만들 수 있을테니까요.