# 타입 확장하기

- 타입 확장은 기존 타입을 사용해서 새로운 타입을 정의하는 것을 말한다.
- 기본적으로 `interface`, `type`키워드를 사용해 타입을 정의하고, `extends`, 교차타입, 유니온 타입을 사용하여 타입을 확장한다.

## 타입 확장의 장점

- 타입 확장의 가장 큰 장점은 코드 중복을 줄일 수 있다는 것이다.
- 타입 확장은 중복 제거, 명시적인 코드 작성 외에도 확장성이란 장점을 가지고 있다.

```ts
/**
 * 메뉴 요소 타입
 * 메뉴 이름, 이미지, 할인율, 재고 정보를 담고 있다.
 */
interface BaseMenuItem {
  itemName: string | null;
  itemImageUrl: string | null;
  itemDiscountAmount: number;
  stock: number | null;
}

/**
 * 장바구니 요소 타입
 * 메뉴 타입에 수량 정보가 추가되었다.
 */
interface BaseCartItem extends BaseMenuItem {
  quantity: number;
}

type BaseCartItem = {
  quantity: number;
} & BaseMenuItem;
```

## 유니온 타입

> 유니온 타입은 2개 이상의 타입을 조합하여 사용하는 방법이다.

- 집합 관점으로 봤을 때 합집합의 개념과 같다.
  - 속성의 집합이 아닌 값의 집합이라고 생각해야 유니온 타입이 합집합이라는 개념을 이해할 수 있다.

```ts
type MyUnion = A | B;
```

### 유니온 타입을 사용할 때 주의할 점

- 유니온 타입으로 선언된 값은 유니온 타입에 포함된 모든 타입이 공통으로 갖고 있는 속성에만 접근할 수 있다.

```ts
interface CookingStep {
  orderId: string;
  price: number;
}

interface DeliveryStep {
  orderId: string;
  time: number;
  distance: string;
}

function getDeliveryDistance(step: CookingStep | DeliveryStep) {
  return step.distance;
  // Property 'distance' does not exist on type 'CookingStep | DeliverStep'
  // Property 'distance' does not exist on type 'CookingStep'
}
```

- 즉, step이라는 유니온 타입은 CookingStep 또는 DeliveryStep 타입에 해당할 뿐이지 CookingStep이면서 DeliveryStep인 것은 아니다.

## 교차 타입

> 기존 타입을 합쳐 필요한 모든 기능을 가진 하나의 타입을 만드는 것

```ts
interface CookingStep {
  orderId: string;
  time: number;
  price: number;
}

interface DeliveryStep {
  orderId: string;
  time: number;
  distance: string;
}

type BaedalProgress = CookingStep & DeliveryStep;
```

- 여기서 유니온 타입과 다르게, BaedalProgress는 CookingStep과 deliveryStep 타입을 합쳐 모든 속성을 가진 단일 타입이 된다.

#### 교차 타입을 사용할 때 타입이 서로 호환되지 않는 경우

```ts
type IdType = string | number;
type Numeric = number | boolean;

type Universal = IdType & Numeric;
```

1. string이면서 number인 경우
2. string이면서 boolean인 경우
3. number이면서 number인 경우
4. number이면서 boolean인 경우

= Universal은 IdType과 Numeric의 교차 타입이므로 두 타입을 모두 만족하는 경우에만 유지된다.

=> 따라서 1,2,4번은 성립되지 않고 3번만 유효하기 때문에 Universal의 타입은 number가 된다.

## extends와 교차 타입

```ts
interface BaseMenuItem {
  itemName: string | null;
  itemImageUrl: string | null;
  itemDiscountAmount: number;
  stock: number | null;
}

interface BaseCartItem extends BaseMenuItem {
  quantity: number;
}
```

> 💡**type만 사용할 수 있는 경우** <br />
> 유니온 타입과 교차 타입을 사용한 새로운 타입은 오직 type 키워드로만 선언할 수 있다.

#### 주의할 점

- `extends` 키워드를 사용한 타입이 교차 타입과 100% 상응하지는 않는다.

```ts
interface DeliveryTip {
  tip: number;
}

interface Filter extends DeliveryTip {
  tip: string;
  // Interface 'Filter' incorrectly extends interface 'DeliveryTip'
  // Types of property 'tip' are incompatible
  // Type 'string' is not assignable to type 'number'
}

type DeliveryTip = {
  tip: number;
};

type Filter = DeliveryTip & {
  tip: string;
};
```

- `extends`를 `&`로 바꿨을 뿐인데 에러가 발생하지 않는다.

  - 이때 tip의 타입은 `never`이다.

- type 키워드는 교차 타입으로 선언되었을 때 새롭게 추가되는 속성에 대해 미리 알 수 없기 때문에 선언 시 에러가 발생하지 않는다.
- 하지만 tip이라는 같은 속성에 대해 서로 호환되지 않는 타입이 선언되어 결국 never 타입이 된 것이다.

## 배달의 민족 메뉴 시스템에 타입 확장 적용하기

- 1인분
- 족발/보쌈
- 찜/탕 찌개
- 돈까스/회/일식
- 피자

> 이 메뉴들을 바탕으로 `Menu`라는 인터페이스를 표현한다면?

```tsx
/**
 * 메뉴에 대한 타입
 * 메뉴 이름과 메뉴 이미지에 대한 정보를 담고 있다.
 */
interface Menu {
  name: string;
  image: string;
}

function MainMenu(){
  // Menu 타입을 원소로 갖는 배열
  const menuList: Menu[] = [{name : "1인분", image : "1인분.png"}, ...]

  return (
    <ul>
      {menuList.map((menu) => (
        <li>
          <img src={menu.image} />
          <span>{menu.name}</span>
        </li>
      ))}
    </ul>
  )
}
```

여기서 특정 메뉴의 중요도를 다르게 주기 위한 요구 사항이 추가되었다고 가정해보자.

1. 특정 메뉴를 길게 누르면 gif 파일이 재생되어야 한다.
2. 특정 메뉴는 이미지 대신 별도의 텍스트만 노출되어야 한다.

```ts
/**
 * 방법1 타입 내에서 속성 추가
 * 기존 Menu 인터페이스에 추가된 정보를 전부 추가
 */
interface Menu {
  name: string;
  image: string;
  gif?: string; // 요구사항 1. 특정 메뉴를 길게 누르면 gif 파일이 재생되어야 한다.
  text?: string; // 요구사항 2. 특정 메뉴는 이미지 대신 별도의 텍스트만 노출되어야 한다.
}

/**
 * 방법2 타입 확장 활용
 * 기존 Menu 인터페이스는 유지한 채, 각 요구 사항에 따른 별도 타입을 만들어 확장 시키는 구조
 */
interface Menu {
  name: string;
  image: string;
}

/**
 * gif를 활용한 메뉴 타입
 * Menu 인터페이스를 확장해서 반드시 gif값을 갖도록 만든 타입
 */
interface SpecialMenu extends Menu {
  gif: string; // 요구사항 1. 특정 메뉴를 길게 누르면 gif 파일이 재생되어야 한다.
}

/**
 * 별도의 텍스트를 활용한 메뉴 타입
 * Menu 인터페이스를 확장해서 반드시 text 값을 갖도록 만든 타입
 */
interface PackageMenu extends Menu {
  text: string; // 요구사항 2. 특정 메뉴는 이미지 대신 별도의 텍스트만 노출되어야 한다.
}
```

각 방법을 적용해보자.

```ts
/**
 * 각 배열은 서버에서 받아온 응답 값이라고 가정
 */
const menuList = [
  {
    name: "찜",
    image: "찜.png",
  },
  {
    name: "찌개",
    image: "찌개.png",
  },
  {
    name: "회",
    image: "회.png",
  },
];

const specialMenuList = [
  { name: "돈까스", image: "돈까스.png", gif: "돈까스.gif" },
  { name: "피자", image: "피자.png", gif: "피자.gif" },
];

const packageMenuList = [
  { name: "1인분", image: "1인분.png", text: "1인 가구 맞춤형" },
  { name: "족발", image: "족발.png", text: "오늘은 족발로 결정" },
];
```

### 방법 1 하나의 타입에 여러 속성을 추가할 때

```ts
  menuList : Menu[]; // OK
  specialMenuList : Menu[]; // OK
  packageMenuList : Menu[]; // OK
```

#### 발생할 수 있는 문제

- specialMenuList 배열의 원소가 각 속성에 접근한다고 했을 때 다음과 같은 문제가 발생할 수 있다.

```ts
specialMenuList.map((menu) => menu.text); // TypeError: Cannot read properties of undefined
```

- 배열의 모든 원소는 text 라는 속성을 가지고 있지 않으므로 에러가 발생한다.

### 방법2 타입을 확장하는 방식

- 위의 방식과는 다르게 확장할 타입에 맞게 명확히 규정할 수 있어 위와 같은 문제가 발생하지 않고, 접근하기 전부터 타입이 잘못되었음을 알 수 있다.

#### 결론

주어진 타입에 무분별하게 속성을 추가하여 사용하는 것보다 타입을 확장해서 사용하는 것이 좋다.

- 적절한 네이밍을 사용해서 타입의 의도를 명확히 표현할 수도 있고, 코드 작성 단계에서 예기치 못한 버그도 예방할 수 있기 때문이다.

# 타입 좁히기 - 타입 가드

## 타입 가드에 따라 분기 처리하기

- 분기 처리는 조건문과 타입 가드를 활용하여 변수나 표현식의 타입 번위를 좁혀 다양한 상황에 따라 다른 도작을 수행하는 것을 말한다.
- 타입 가드는 런타임에 조건문을 사용하여 타입을 검사하고, 타입 범위를 좁혀주는 기능을 말한다.

### 사용 상황을 가정해보자 !

- 여러 타입을 할당할 수 있는 스코프에서 특정 타입을 조건으로 만들어 분기처리하고 싶을 때가 있다.
- 여러 타입을 할당할 수 있다는 것은 변수가 유니온 타입 또는 any 타입 등 여러 가지 타입을 받을 수 있다는 것을 말하는데 조건으로 검사하려는 타입보다 넓은 범위를 갖고 있다.

> **스코프(scope)** <br />
> 타입스크립트에서 스코프는 변수와 함수 등의 식별자가 유효한 범위를 나타낸다. 즉, 변수와 함수를 선언하거나 사용할 수 있는 영역을 말한다.

- 특정 문맥 안에서 타입스크립트가 해당 변수를 특정 타입을 추론하도록 유도하면서 런타임도 유효한 방법이 필요하다.
  - **런타임에도 유효**하다는 것은 타입스크립트 뿐만 아니라 자바스크립트에서도 사용할 수 잇는 문법이어야 한다는 의미이다.
  - typeof, instanceof, in 과 같은 연산자를 사용해서 제어문으로 특정 타입 값을 가질 수 밖에 없는 상황을 유도하여 자연스럽게 타입을 좁히는 방식이다.

## 원시 타입을 추론할 때 : typeof 연산자 활용하기

- 자바스크립트의 동작 방식으로 인해 null과 배열 타입 등이 object 타입으로 판별되는 등 복잡한 타입을 검증하기에는 한계가 있기 때문에 원시 타입 검증에만 typeof를 사용하는 것이 좋다.
- string
- number
- boolean
- undefined
- object
- function
- bigint
- symbol

## 인스턴스화된 객체 타입을 판별할 때 instanceof 연산자 활용하기

> 예시: 특정 매개변수가 Date인지를 검사한 후에 Range 타입의 객체를 반환할 수 있도록 분기처리하기

```ts
interface Range {
  start: Date;
  end: Date;
}

interface DatePickerProps {
  selectedDates?: Date | Range;
}

const DatePicker = ({ selectedDates }: DatePickerProps) => {
  const [selected, setSelected] = useState(convertToRange(selectedDates));
  //...
};

export function convertToRange(selected?: Date | Range): Range | undefined {
  return selected instanceof Date
    ? { start: selected, end: selected }
    : selected;
}
```

- typeof는 주로 원시 타입을 판별하는데 사용한다면, instanceof 연산자는 인스턴스화된 객체 타입을 판별하는 타입 가드로 사용할 수 있다.
- A instanceof B 형태로 사용하며 A에는 타입을 검사할 대상 변수, B에는 특정 객체의 생성자가 들어간다.

instanceof는 A의 프로토타입 체인에 생성자 B가 존재하는지 검사해서 존재한다면 true, 그렇지 않다면 false를 반환한다.

#### 유의할 점

- A의 프로토타입 속성 변화에 따라 instanceof 연산자의 결과가 달라질 수도 있다.

```ts
const onKeyDown = (event: React.KeyboardEvent) => {
  if (event.target instanceof HTMLInputElement && event.key === "Enter") {
    // 이 분기에서는 event.target의 타입이 HTMLInputElement이며
    // event.key가 'Enter'이다.
    event.target.blur();
    onCTAButtonClick(event);
  }
};
```

## 객체의 속성이 있는지 없는지에 따른 구분: in 연산자 활용하기

> 객체에 속성이 있는지 확인한 다음에 true 또는 false를 반환한다.

- A in B의 형태로 사용하는데, 이름 그대로 A라는 속성이 B 객체에 존재하는지를 검사한다.
- 프로토탕비 체인으로 접근할 수 있는 속성이면 전부 true를 반환한다.
- B 객체에 존재하는 A 속성에 undefined를 할당한다고 해서 false를 반환하는 것은 아니다. delete 연산자를 사용하여 객체 내부에서 해당 속성을 제거해야만 false를 반환한다.

#### 자바스크립트의 in 연산자와의 차이점

- 자바스크립트의 in 연산자는 런타임의 값만을 검사하지만, 타입스크립트에서는 객체 타입에 속성이 존재하는지를 검사한다.

## is 연산자로 사용자 정의 타입 가드 만들어 활용하기

> 직접 타입 가드 함수를 만들 수도 있다.

- 반환 타입이 타입 명제(type predicates)인 함수를 정의하여 사용할 수 있다.
- 타입 명제는 A is B 형식으로 작성하면 되는데 여기서 A는 매개변수 이름이고, B는 타입이다.

> **타입 명제(type predicates)** <br/>
> 타입 명제는 함수의 반환 타입에 대한 타입 가드를 수행하기 위해 사용되는 특별한 형태의 함수이다.

```ts
const isDestinationCode = (x: string): x is DestinationCode =>
  destinationCodeList.includes(x);
```
