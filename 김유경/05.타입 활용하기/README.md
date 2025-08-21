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

- 권한으로 유효한 값만 추출하여 배열로 반환하는 타입 정의

```ts
type UnpackMenunames<T extends ReadonlyArray<MenuItem>> = T extends
ReadonlyArray<infer U>
   ? I extends MainMenu
     ? U["subMenus"] extends infer V
       ? V extends ReadonlyArray<SubMenu>
         ? UnpackMenuNames<V>
         : U['name']
       : never
     : U extends SubMenu
```

- U가 MainMenu 타입이라면 subMenus를 infer V로 추출한다
- subMenus는 옵셔널한 타입이기 때문에 추출한 V가 존재한다면 (SubMenu 타입에 할당할 수 있다면 UnpackMenuNames에 다시 전달한다)
- V가 존재하지 않는다면 MainMenu의 name은 권한에 해당하므로 U["name"]이다.
- U가 MainMenu가 아니라 SubMenu에 할당할 수 있다면 (U는 SubMenu타입이기 때문에) U['name']은 권한에 해당한다.

# 템플릿 리터럴 타입 활용하기

- 타입 스크립트에서는 유니온 타입을 사용하여 변수 타입을 특정 문자열로 지정할 수 있다.

```ts
type HeaderTag = "h1" | "h2";
```

```ts
type HeadingNumber = 1 | 2 | 3 | 4 | 5;
type HeaderTag = `h${HeadingNumber}`;
```

### 주의할 점

- 타입스크립트 컴파일러가 유니온을 추론하는 데 시간이 오래 걸리면 비효율적이기 때문에 타입스크립트가 타입을 추론하지 않고 에러를 내뱉을 때가 많다.
- 따라서 템플릿 리터럴 타입에 삽입된 유니온 조합의 경우의 수가 너무 많지 않게 적절히 나누어 타입을 정의하는 것이 좋다.
- 예시) 전화번호 조합

# 커스텀 유틸리티 활용하기

## 1. 유틸리티 함수를 활용해 styled-components의 중복 타입 선언 피하기

### 활용하지 않는 경우

```ts
export type Props = {
  height?: string;
  color?: keyof typeof colors;
  isFull?: boolean;
  className?: string;
};

export const Hr:VFC<Props> = ({height, color, isFull, className}) => {
  ...
  return <HrComponent height={height} color={color} isFull={isFull} className={className} />
}

// style.ts
import {Props} from '...';
type StyledProps = Pick<<Props, "height" | "color" | "isFull">>;
const HrComponent = styled.hr<StyleProps>`
  height :...
`
```

## 2. PickOne 유틸리티 함수

- 서로 다른 2개 이상의 객체를 유니온 타입으로 받을 때 타입 검사가 제대로 진행되지 않는 이슈가 있다. 이런 문제를 해결하기 위해 PickOne이라는 이름의 유틸리티 함수를 구현해보자.

### 식별할 수 있는 유니온으로 객체 타입을 유니온으로 받기

```ts
type Card = {
  type: "card";
  card: string;
};

type Account = {
  type: "account";
  account: string;
};

function withdraw(type : Card | Account){
  ...
}

withdraw({type : "card", card : 'hyundai"});
withdraw({type : 'account", card : 'hana"});
```

- 이 방법도 유용하지만 일일히 type을 넣어줘야하는 불편함이 생긴다.

```ts

  {account : string; card?: undefined} | {account?:undefined; card:string}
```

- 옵셔널 + undefined

```ts
type PayMethod =
  | { account: string; card?: undefined; payMoney?: undefined }
  | { account: undefined; card?: string; payMoney?: undefined }
  | { account: undefined; card?: undefined; payMoney?: string };
```

```ts
type PickOne<T> = {
  [P in keyof T]: Record<P, T[P]> &
    Partial<Record<Exclude<keyof T, P>, undefined>>;
}[keyof T];
```

### 뜯어보기

- T에는 객체가 들어온다고 가정한다.
  > One<T>

```ts
type One<T> = { [P in keyof T]: Record<P, T[P]> }[keyof T];
```

- [P in keyof T]에서 T는 객체로 가정하고 있기 때문에 P는 T객체의 키 값을 말한다.
- Record<P, T[P]>는 P 타입을 키로 가지고, value는 P를 키로 둔 T객체의 값의 레코드 타입을 말한다.
- 따라서 { [P in keyof T] : Record<P, T[P]>}에서 키는 T 객체의 키 모음이고, value는 해당 키의 원본 객체 T를 말한다.
- 3번의 타입에서 다시 [keyof T]의 키 값으로 접근하기 때문에 최종 결과는 전달받은 T와 같다.

```ts
type Card = { card: string };
const one: One<Card> = { card: "hyundai" };
```

> ExcludeOne<T>

```ts
type ExcludeOne<T> = {
  [P in keyof T]: Partial<Record<Exclude<keyof T, P>, undefined>>;
};
```

- [P in keyof T]에서 T는 객체로 가정하기 때문에 P는 T객체의 키 값을 말한다.
- Exclude<keyof T, P>는 T 객체가 가진 키값에서 P타입과 일치하는 키 값을 제외한다. 이 타입을 A라고 하자.
- Record<A, undefined>는 키로 A타입을, 값으로 undefined 타입을 갖는 레코드 타입이다. 즉, 전달 받은 객체 타입을 모두 { [key]: undefined} 형태로 만든다. 이 타입을 B라고 하자.
- Partial<B>는 B타입을 옵셔널로 만든다. 따라서 { [key] ?: undefined}와 같다.
- 최종적으로 [P in keyof T]로 매핑된 타입에서 동일한 객체의 키 값인 [keyof T]로 접근하기 때문에 4번 타입이 반환된다.

결론적으로 얻고자 하는 타입은 속성 하나와 나머지는 옵셔널 + undefined인 타입이기 때문에 앞의 속성을 활용해서 PickOne 타입을 표현할 수 있다.

> PickOne<T>

1. One<T> & ExcludeOne<T>는 [P in keyof T]를 공통으로 갖기 때문에 아래 같이 교차된다.

> [P in keyof T] : record<P, T[P]> & Partial<Record<Exclude<keyof T, P>, undefined>>

2. 이 타입을 해석하면 전달된 T타입의 1개의 키는 값을 가지고 있으며, 나머지 키는 옵셔널한 undefined값을 가진 객체를 의미한다.

```ts
type Card = { card: string };
type Account = { account: string };

const pickOne1: PickOne<Card & Account> = { card: "hyundai" }; // O
const pickOne2: PickOne<Card & Account> = { account: "hana" }; // O
const pickOne3: PickOne<Card & Account> = {
  card: "hyundai",
  account: undefined,
}; // O
const pickOne4: PickOne<Card & Account> = { card: undefined, account: "hana" }; // O
const pickOne5: PickOne<Card & Account> = { card: "hyundai", account: "hana" }; // X
```

## 3. NonNullable 타입 검사 함수를 사용하여 간편하게 타입 가드하기

- null을 가질 수 있는 값의 null 처리는 자주 사용되는 타입 가드 패턴이다.

### NonNullable 타입이란

- 타입스크립트에서 제공하는 유틸리티 타입으로, 제네릭으로 받는 T가 null 또는 undefined일 때 never 또는 T를 반환하는 타입이다. NonNullable을 사용하면 null이나 undefined가 아닌 경우를 제외할 수 있다.

```ts
type NonNullable<T> = T extends null | undefined ? never : T;
```

### null, undefined를 검사해주는 NonNullable 함수

- NonNullable 함수는 매개변수인 value가 null 또는 undefined라면 false를 반환한다. is 키워드가 쓰였기 때문에 NonNullable 함수를 사용하는 쪽에서 true가 반환된다면 넘겨준 인자는 null이나 undefined가 아닌 타입으로 타입 가드가 된다.

```ts
function NonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}
```

### Promise.all을 사용할 때 NonNullable 적용하기

- 여러 상품의 광고를 조회할 때 하나의 API에서 에러가 발생한다고 해서, 전체 광고가 보이지 않으면 안된다.

```ts
class AdCampaignAPI {
  static async operation(shopNo: number): Promise<AdCampaign[]> {
    try {
      return await fetch(`/ad/shopNumber=${shopNo}`);
    } catch (error) {
      return null;
    }
  }
}

const shopList = [
  { shopNo: 100, category: "chicken" },
  { shopNo: 101, category: "pizza" },
  { shopNo: 102, category: "noodle" },
];

const shopAdCampaignList = await Promise.all(shopList.map((shop)) => AdCampaignAPI.operating(shop.shopNo))
```

- 이때 NonNullable 함수로 필터링하지 않으면 shopAdCampaignList를 순회할 때마다 고차 함수 내 콜백 함수에서 if문을 활용한 타입 가드를 반복해야 한다.

```ts
const shopList = [
  { shopNo: 100, category: "chicken" },
  { shopNo: 101, category: "pizza" },
  { shopNo: 102, category: "noodle" },
];

const shopAdCampaignList = await Promise.all(
  shopList.map((shop) => AdCampaignAPI.operating(shop.shopNo))
);

const shopAds = shopAdCampaignList.filter(NonNullable);
```

# 불변 객체 타입으로 활용하기

- 상숫값을 관리할 때 객체를 사용한다.

- theme 객체, 애니메이션 객체, 상숫값 담은 객체 등

## 1. Atom 컴포넌트에서 theme style 객체 활용하기

```ts
interface Props {
  fontSize?: string;
  backgroundColor?: string;
  color?: string;
  onClick: (event: React.MouseEvent<HTMLButtonElement>) => void | Promise<void>;
}

const Button: FC<Props> = ({ fontSize, backgroundColor, color, children }) => {
  return;
  <ButtonWrap fontSize={fontSize} backgroundColor={fontSize} color={color}>
    {children}
  </ButtonWrap>;
};

const ButtonWrap = styled.button<Omit<Props, "onClick">>`
  color: ${({ color }) => theme.color[color ?? "default"]};
  backgroundcolor: ${({ backgroundColor }) =>
    theme.backgroundColor[backgroundColor ?? "default"]};
  fontsize: ${({ fontSize }) => theme.fontSize[fontSize ?? "default"]};
`;
```

- theme 객체로 타입을 구체화해서 올바르지 않은 값이 입력될 여지를 줄일 수 있다.

### 타입스크립트 keyof 연산자로 객체의 키값을 타입으로 추출하기

- 객체 타입을 받아 해당 객체의 키값을 string 또는 number의 리터럴 유니온 타입으로 반환한다.
- 객체 타입으로 인덱스 시그니처가 사용되었다면 keyof는 인덱스 시그니처의 키 타입을 반환한다.

### 타입스크립트 typeof 연산자로 값을 타입으로 다루기

- keyof 연산자는 객체타입을 받는다. 따라서 객체의 키값을 타입으로 다루려면 값 객체를 타입으로 변환해야한다.
- 이때 사용하는 것이 typeof 연산자이다.

# Record 원시 타입 키 개선하기

## 1. 무한한 키를 집합으로 가지는 Record

```ts
type Category = string;
interface Food {
  name: string;
  // ...
}

const foodByCategory: Record<Category, Food[]> = {
  한식: [{ name: "제육덮밥" }, { name: "뚝배기 불고기" }],
  일식: [{ name: "초밥" }, { name: "텐동" }],
};
```

- 여기에서 Category는 string이다 Category를 Record 키로 사용하는 foodByCategory 객체는 무한한 키 집합을 가지게 된다. 이때 foodByCategory 객체에 없는 키 값을 사용하더라도 타입스크립트는 오류를 표시하지 않는다.
- 이때 자바스크립트의 옵셔널 체이닝 등을 사용해 런타임 에러를 방지할 수 있다.

```ts
foodByCategory["양식"]?.map((food) => console.log(food.name));
```

- 그러나 어떤 값이 undefined인지 매번 판단해야 하는 번거로움이 생기고, 실수로 undefined일 지도 모르는 값을 처리해주지 못하면 에러가 발생할 수 있다.

## 2. 유닛 타입으로 변경하기

- 키가 유한한 집합이라면 유닛 타입(다른 타입으로 쪼개지지 않고 오직 하나의 정확한 값을 가지는 타입)을 사용할 수 있다.

```ts
type Category = "한식" | "일식";

interface Food {
  name: string;
  //...
}

const foodByCategory: Record<Category, Food[]> = {
  한식: [{ name: "제육덮밥" }, { name: "뚝배기 불고기" }],
  일식: [{ name: "초밥" }, { name: "텐동" }],
};

// Property '양식' does not exist on type 'Record<Category, Food[]>'.
foodByCategory["양식"];
```

## 3. Partial을 활용하여 정확한 타입 표현하기

- 키가 유한한 상황에서는 Partial을 사용하여 해당 값이 undefined일 수 있는 상태임을 표현할 수 있다.
- 객체 값이 undefined일 수 있는 경우에 Partial을 사용해서 PartialRecord 타입을 선언하고 객체를 선언할 때 이것을 활용할 수 있다.

```ts
type PartialRecord<K extends string, T> = Partial<Record<K, T>>;
type Category = string;

interface Food {
  name: string;
  //...
}

const foodByCategory: PartialRecord<Category, Food[]> = {
  한식: [{ name: "제육덮밥" }, { name: "뚝배기 불고기" }],
  일식: [{ name: "초밥" }, { name: "텐동" }],
};

foodByCategory["양식"]; // Food[] 또는 undefined 타입으로 추론
foodCategory["양식"].map((food) => console.log(food.name)); // Object is possibly 'undefined'
foodByCategory["양식"]?.map((food) => console.log(food.name)); // OK
```

- foodByCategory[key]를 Food[] 또는 undefined로 추론하고, 개발자에게 이에 대한 처리가 필요함을 안내해주어 런타임 에러를 방지할 수 있다.
