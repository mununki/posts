# Phantom type in ReasonML

## 목적
* Phantom type이 무엇인지 알아보자
* 예를 통해 Phantom type을 사용하여 얻을 수 있는 장점

## Phantom type이란?
* Parameterized type인데, 그 parameter가 선언의 오른쪽에는 없는 타입
* 하나의 타입이지만, 여러 subtype을 data의 representation을 바꿀 필요 없이 가질 수 있다.

## Example
* formData validated, unvalidated 의 example
* byPass 같은 것도 못하게 만들 수도 있징~
* module type의 t('a)를 string으로 선언하느냐 마느냐에 따른 차이!!! 컴파일러가 t('a)를 string으로 치환해서 생각하게 하느냐 마느냐에 따라 byPass와 같은 구멍을 막을 수 있음. unification이란 뭘까?

## 장점
1. type FormData.t의 generalized
2. 같은 shape(representation)의 data에 대해 다른 subtypes을 적용할 수 있다.

### References
* Phantom Types in ReasonML https://medium.com/reasontraining/phantom-types-in-reasonml-1a4cfc18d99, https://gist.github.com/busypeoples/3a28d039272ec3eb33ca2fc6b32dafc7