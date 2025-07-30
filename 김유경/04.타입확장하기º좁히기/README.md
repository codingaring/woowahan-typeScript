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
