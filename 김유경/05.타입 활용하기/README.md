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

- 각 API는 계좌, 카드, 앱카드의 결제 수단 정보를 배열로 반환한다.
- 3가지 API의 엔드포인트가 비슷하기 때문에 서버 응답을 처리하는 공통 함수를 생성하고, 해당 함수에 타입을 전달하여 타입별로 처리 로직을 구현할 것이다.

```ts
interface PayMethodBaseFromRes {
  financialCode: string;
  name: string;
}

interface Bank extends PayMethodBaseFromRes {
  fullName: string;
}

interface Card extends PayMethodBaseFromRes {
  appCardType?: string;
}

type PayMethodInfo<T extends Bank | Card> = T & PayMethodInterface;
type PayMethodInterface = {
  companyName: string;
  //...
};
```

```ts
type PayMethodType = PayMethodInfo<Card> | PayMethodInfo<Bank>;
export const useGetRegisteredList = (
  type: "card" | "appcard" | "bank"
): UseQueryResult<PayMethodType[]> => {
  const url = `baeminpay/codes/${type === "appcard" ? "card" : type}`;

  const fetcher = fetcherFactory<PayMethodType[]>({
    const usablePocketList = res?.filter(
      (pocket: PocketInfo<Card> | PocketInfo<Bank>) =>
      pocket?.useType === "USE"
    ) ?? [];
    return usablePocketList;
  });

  const result = useCommonQuery<PayMethodType[]>(url, undefined, fetcher);

  return result;
};
```

- useGetRegisteredList 함수가 반환하는 Data 타입은 PocketInfo<Card> | PocketInfo<Bank> 이다.
- 하지만, useGetRegisteredList가 반환하는 Data 타입은 PayMethodType이기 때문에 사용하는 쪽에서는 PocketInfo일 가능성도 있다.

```ts
const { data: pocketList } = useGetRegisteredList("card");
// const pocketList : PocketInfo<Card> | undefined
```

## 3. extends 조건부 타입을 활용하여 개선하기

- useGetRegisteredList 함수의 반환 Data는 인자 타입에 따라 정해져있다.
  타입으로 "card" 또는 "appcard"를 받으면 카드 결제 수단 정보 타입인 PocketInfo<card>를 반환하고, "bank"를 받는다면 PocketInfo<bank> 를 반환한다.

  - type : "card" | "appcard" => PocketInfo<Card>
  - type : "bank" => PocketInfo<Bank>

```ts
type PayMethodType<T extends "card" | "appcard" | "bank"> = T extends
  | "card"
  | "appcard"
  ? Card
  : Bank;
```

- PayMethodType의 제네릭으로 받은 값이 "card" 또는 "appcard"일 때는 PayMethodInfo<Card> 타입을, 아닐 때는 PayMethodInfo<Bank> 타입을 반환하도록 수정했다.

### 사용 시 장점

- 제네릭과 extends를 함께 사용해 제네릭으로 받는 타입을 제한했다. 따라서 개발자는 잘못된 값을 넘길 수 없기 때문에 휴먼 에러를 방지할 수 있다.
- extends를 활용해 조건부 타입을 설정했다. 조건부 타입을 사용해서 반환 값을 사용자가 원하는 값으로 구체화할 수 있었다. 이에 따라 불필요한 타입 가드, 타입 단언 등을 방지할 수 있다.

## 4. infer를 활용해서 타입 추론하기

- extends 키워드를 사용할 때 infer 키워드를 사용할 수 있다.

```ts
type UnpackPromise<T> = T extends Promise<infer K>[] ? K : any;
```

- UnpackPromise 타입은 제네릭으로 T를 받아서 T가 Promise로 래핑된 경우라면 K를 반환하고, 그렇지 않은 경우에는 any를 반환한다.

```ts
const promises = [Promise.resolve("Mark"), Promise.resolve(38)];
type Expected = UnpackPromise<typeof promises>; // string | number
```

- extends, infer, 제네릭을 활용하면 타입을 조건에 따라 더 세밀하게 사용할 수 있게 된다.

```ts
interface RouteBase {
  name: string;
  path: string;
  component: ComponentType;
}

export interface RouteItem {
  name: string;
  path: string;
  component?: ComponentType;
  pages?: RouteBase[];
}

export const routes: RouteItem[] = [
  {
    name: "기기 내역 관리",
    path: "/device-history",
    component: DeviceHistoryPage,
  },
  {
    name: "헬멧 인증 관리",
    path: "/helmet-certification",
    component: HelmetCertificationPage,
  },
];
```
