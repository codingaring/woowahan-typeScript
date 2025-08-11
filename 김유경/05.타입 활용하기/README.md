# 조건부 타입

> Condition ? A : B

- A는 Condition이 true일때 도출되는 타입이고, B는 false일때 도출되는 타입이다.

- 조건부 타입을 활용하면 중복되는 타입 코드를 제거하고 상황에 따라 적절한 타입을 얻을 수 있어 더욱 정확한 타입 추론이 가능하게 된다.

## 1. extends와 제네릭을 활용한 조건부 타입

- extends 키워드는 타입스크립트에서 다양한 상황에서 활용된다.
- 타입을 확장할 때와 타입을 조건부로 설정할 때 사용되며, 제네릭 타입에서는 한정자 역할로도 사용된다.

```ts
    T extends U ? X : Y;
```

- 타입 T를 U에 할당할 수 있으면 X 타입 아니면 Y 타입으로 결정됨을 의미한다.

```ts
interface Bank {
  financialCode: string;
  componyName: string;
  name: string;
  fullName: string;
}

interface Card {
  financialCode: string;
  companyName: string;
  name: string;
  appCardType?: string;


type PayMethod<T> = T extends "card" ? Card : Bank;
type CardPayMethodType = PayMethod<"card">;
type BankPayMethodType = PayMethod<"bank">;
```

## 2. 조건부 타입을 사용하지 않았을 때의 문제점

react-query, 계좌, 카드, 앱 카드 등 3가지 결제 수단 정보를 가져오는 API가 있다.

```md
- 계좌 정보 엔드포인트 : www.baemin.com/baeminpay/.../bank
- 카드 정보 엔드포인트 : www.baemin.com/baeminpay/.../card
- 앱 카드 정보 엔드포인트 : www.baemin.com/baeminpay/.../appcard
```
