> 지금까지는 타입스크립트로 원하는 타입을 정의하고 활용하는 법에 대해 알아보았습니다.
> 타입스크립트에서는 기존에 정의한 타입을 확장하여 새로운 타입을 정의하는 기능을 제공합니다. 타입 확장이라고 부르는데, 일르 사용해서 조금 더 복잡한 타입을 정의할 수 있습니다.
> 한편 타입이 복잡해질수록 어떤 대상에 대한 타입 추론이 더 넓은 범위에서 이루어질 가능성이 있습니다. 이럴 때는 타입을 더 작은 범위로 좁히는 타입 좁히기를 사용해서 정확한 타입 추론을 할 수 있도록 해야 합니다.
> 이번 장에서는 타입 확장과 타입 좁히기의 개념을 살펴보며 더욱 확장성 있고 명시적인 코드 작성법에 대해 알아보겠습니다.

## 04.1 타입 확장하기

- 타입 확장은 기존 타입을 사용해서 새로운 타입을 정의하는 것을 뜻합니다.

### 타입 확장의 장점

- 타입 확장의 가장 큰 장점은 코드 중복을 줄일 수 있다는 것입니다.

```ts
/**
 * 메뉴 요소 타입
 * 메뉴 이름, 이미지, 할인율, 재고 정보를 담고 있다
 * */
interface BaseMenuItem {
  itemName: string | null;
  itemImageUrl: string | null;
  itemDiscountAmount: number;
  stock: number | null;
}

/**
 * 장바구니 요소 타입
 * 메뉴 타입에 수량 정보가 추가되었다
 */
interface BaseCartItem extends BaseMenuItem {
  quantity: number;
}
```

- `interface`키워드 대신 `type`을 사용한다면 아래와 같이 작성하면 된다.

```ts
type BaseMenuItem = {
  itemName: string | null;
  itemImageUrl: string | null;
  itemDiscountAmount: number;
  stock: number | null;
};

type BaseCartItem = {
  quantity: number;
} & BaseMenuItem;
```

- 타입 확장은 중복 제거, 명시적인 코드 작성 외에도 확장성이란 장점을 가지고 있습니다.

```ts
/**
 * 수정할 수 있는 장바구니 요소 타입
 * 품절 여부, 수정할 수 있는 옵션 배열 정보가 추가되었다
 */
interface EditableCartItem extends BaseCartItem {
  isSoldOut: boolean;
  optionGroups: SelectableOptionGroup[];
}

/**
 * 이벤트 장바구니 요소 타입
 * 주문 가능 여부에 대한 정보가 추가되었다
 */
interface EventCartItem extends BaseCartItem {
  orderable: boolean;
}
```

- 이렇게 타입 확장을 활용하면 장바구니와 관련된 요구 사항이 생길 때마다 필요한 타입을 손쉽게 만들 수 있게 됩니다.
- 또한, 기존 장바구에 요소에 대한 요구사항이 변경되어도 `BaseCartItem` 타입만 수정하면 되기 때문에 효율적입니다.

### 유니온 타입

- 유니온 타입은 2개 이상의 타입을 조합하여 사용하는 방법입니다.
- 집합 관점으로 보면 유니온 타입을 합집합으로 해석할 수 있습니다.
- 여기서 주의해야 할 점은, 유니온 타입으로 선언된 값은 유니온 타입에 포함된 모든 타입이 공통으로 갖고 있는 속성에만 접근할 수 있습니다.

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
    // Property ‘distance’ does not exist on type ‘CookingStep | DeliveryStep’
    // Property ‘distance’ does not exist on type ‘CookingStep’
  }
  ```

  - `step.distance`를 호출시 `step`의 타입이 `CookingStep`이라면 `distance`속성을 찾을 수 없기 때문에 에러가 발생합니다.
  - 즉, `step`이라는 유니온 타입은 `CookingStep` 또는 `DeliveryStep` 타입에 해당할 뿐이지 `CookingStep`이면서 `DeliveryStep`인 것은 아닙니다.

> 타입스크립트의 타입을 속성의 집합이 아니라 값의 집합이라고 생각해야 유니온 타입이 합집합이라는 개념을 이해할 수 있습니다.

### 교차 타입

- 교차 타입도 기존 타입을 합쳐 필요한 모든 기능을 가진 하나의 타입을 만드는 것으로 이해할 수 있습니다.

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

  - 해당 예시에서는 두가지 타입을 합쳐 모든 속성을 가진 단일 타입을 만들고 있습니다.
  - `BaedalProgress`교차 타입은 `CookingStep`이 가진 속성(orderId, time, price)과 `DeliveryStep`이 가진 속성 (orderId, time, distance)을 모두 만족(교집합)하는 값의 타입(집합)이라고 해석할 수 있습니다.

- 앞서 유니온 타입은 합집합의 개념이라고 설명했습니다.
- 교차 타입은 교집합의 개념과 비슷합니다.

> 다시 언급하지만 타입스크립트의 타입을 속성의 집합이 아니라 값의 집합으로 이해해야 합니다.

- 다른 예시를 살펴보아용

  ```ts
  /* 배달 팁 */
  interface DeliveryTip {
    tip: string;
    }
  /* 별점 */
  interface StarRating {
    rate: number;
  }
  /* 주문 필터 */
  type Filter = DeliveryTip & StarRating;

  const filter: Filter = {
    tip: “1000원 이하”,
    rate: 4,
  };
  ```

  - 교차 타입은 두 타입의 교집합을 의미한다고 언급했습니다.
  - `DeliveryTip`과 `StarRating`은 공통된 속성이 없는데도 `Filter`의 타입은 공집합(never 타입)이 아닌 두 타입의 속성을 모두 포함한 타입이 됩니다.
    - **왜냐하면 타입이 속성이 아닌 값의 집합으로 해석되기 때문입니다.**
    - 즉, 교차 타입 `Filter`는 `DeliveryTip`의 tip 속성과 `StarRating`의 rate 속성을 모두 만족하는 값이 됩니다.

- 교차 타입을 사용할 때 타입이 서로 호환되지 않는 경우도 있습니다.

  ```ts
  type IdType = string | number;
  type Numeric = number | boolean;

  type Universal = IdType & Numeric;
  ```

  - 먼저 `Universal`타입을 다음과 같이 4가지로 생각해볼 수 있습니다.
    1.  string이면서 number인 경우
    2.  string이면서 boolean인 경우
    3.  number이면서 number인 경우
    4.  number이면서 boolean인 경우
  - `Universal`은 `IdType`과 `Numeric`의 교차 타입이므로 두 타입을 모두 만족하는 경우에만 유지됩니다.
  - 따라서 1, 2, 4번은 성립되지 않고 3번만 유효하기 때문에 `Universal`의 타입은 number가 됩니다.

### extends와 교차 타입

- 유니온 타입과 교차 타입을 사용한 새로운 타입은 오직 `type`키워드로만 선언할 수 있습니다.
- 주의할 점은 `extends`키워드를 사용한 타입이 교차 타입과 100% 상응하지는 않는다은 것입니다.

  ```ts
  interface DeliveryTip {
    tip: number;
  }

  interface Filter extends DeliveryTip {
    tip: string;
    // Interface ‘Filter’ incorrectly extends interface ‘DeliveryTip’
    // Types of property ‘tip’ are incompatible
    // Type ‘string’ is not assignable to type ‘number’
  }
  ```

  - `DeliveryTip`타입은 number 타입의 tip속성을 가지고 있습니다.
  - 이때 `DeliveryTip`을 `extends`로 확장한 `Filter` 타입에 string 타입의 속성 tip을 선언하면 tip의 타입이 호환되지 않는다는 에러가 발생합니다.

- 같은 예시를 교차 타입으로 작성해보겠습니다.

  ```ts
  type DeliveryTip = {
    tip: number;
  };

  type Filter = DeliveryTip & {
    tip: string;
  };
  ```

  - `extends`를 `&`으로 바꿨을 뿐인데 에러가 발생하지 않습니다.
  - 이때 tip 속성의 타입은 `never`로 추론됩니다.

- `type`키워드는 교차 타입으로 선언되었을 때 새롭게 추가되는 속성에 대해 미리 알 수 없기 때문에 선언 시 에러가 발생하지 않습니다.
- 하지만 tip이라는 같은 속성에 대해 서로 호환되지 않는 타입이 선언되어 결국 `never`타입이 된 것입니다.

### 배달의민적 메뉴 시스템에 타입 확장 적용하기

- 먼저 서비스의 메뉴 목록을 바탕으로 `Menu`라는 이름을 갖는 인터페이스를 다음과 같이 표현할 수 있습니다.

  ```ts
  /**
   * 메뉴에 대한 타입
   * 메뉴 이름과 메뉴 이미지에 대한 정보를 담고 있다
   */
  interface Menu {
    name: string;
    image: string;
  }
  ```

  - 개발자에게 메뉴 목록을 주면 `Menu` 인터페이스를 기반으로 사용자에게 앞의 그림 같은 화면을 보여줄 수 있습니다.

  ```tsx
  function MainMenu() {
    // Menu 타입을 원소로 갖는 배열
    const menuList: Menu[] = [{name: “1인분”, image: “1인분.png”}, ...]

    return (
      <ul>
        {menuList.map((menu) => (
          <li>
            <img src= {menu.image} />
            <span>{menu.name}</span>
          </li>
        ))}
      </ul>
    )
  }
  ```

  - 이때 특정 요구사항이 추가되는 상황을 가정해보겠습니다.
    1. 특정 메뉴를 길게 누르면 gif 파일이 재생되어야 한다.
    2. 특정 메뉴는 이미지 대신 별도의 텍스트만 노출되어야 한다.
  - 요구사항을 만족하는 타입의 작성 방법을 2가지로 생각해볼 수 있습니다.

  ```ts
  /**
   * 방법1 타입 내에서 속성 추가
   * 기존 Menu 인터페이스에 추가된 정보를 전부 추가
   */
  interface Menu {
    name: string;
    image: string;
    gif?: string; // 요구 사항 1. 특정 메뉴를 길게 누르면 gif 파일이 재생되어야 한다
    text?: string; // 요구 사항 2. 특정 메뉴는 이미지 대신 별도의 텍스트만 노출되어야 한다
  }

  /**
   * 방법2 타입 확장 활용
   * 기존 Menu 인터페이스는 유지한 채, 각 요구 사항에 따른 별도 타입을 만들어 확장시키는 구조
   */
  interface Menu {
    name: string;
    image: string;
  }

  /**
   * gif를 활용한 메뉴 타입
   * Menu 인터페이스를 확장해서 반드시 gif 값을 갖도록 만든 타입
   */
  interface SpecialMenu extends Menu {
    gif: string; // 요구 사항 1. 특정 메뉴를 길게 누르면 gif 파일이 재생되어야 한다
  }

  /**
   * 별도의 텍스트를 활용한 메뉴 타입
   * Menu 인터페이스를 확장해서 반드시 text 값을 갖도록 만든 타입
   */
  interface PackageMenu extends Menu {
    text: string; // 요구 사항 2. 특정 메뉴는 이미지 대신 별도의 텍스트만 노출되어야 한다
  }
  ```

- 주어진 타입에 무분별하게 속성을 추가하여 사용하는 것보다 타입을 확장해서 사용하는 것이 좋습니다.
- 적절한 네이밍을 사용해서 타입의 의도를 명확히 표현할 수도 있고, 코드 작성 단계에서 예기치 못한 버그도 예방할 수 있기 때문입니다.

> 솔직히 많이 찔리네요 optional 속성을 많이 추가해서 사용했는데 반성해야겠습니다.

### 타입 좁히기 - 타입 가드

- 타입스크립트에서 타입 좁히기는 변수 또는 표현식의 타입 범위를 더 작은 범위로 좁혀나가는 과정을 이야기합니다.
- 타입 좁히기를 통해 더 정확하고 명시적인 타입 추론을 할 수 있게 되고, 복잡한 타입을 작은 범위로 축소하여 타입 안정성을 높일 수 있습니다.

#### 타입 가드에 따라 분기 처리하기

- 타입스크립트에서의 분기 처리는 조건문과 타입 가드를 활용하여 변수나 표현식의 타입 범위를 좁혀 다양한 상황에 따라 다른 동작을 수행하는것을 말합니다.
- 타입 가드는 런타임에 조건문을 사용하여 타입을 검사하고 타입 범위를 좁혀주는 기능을 말합니다.
- 예를 들어 어떤 함수가 `A | B` 타입의 매개변수를 받는다고 가정한다면 인자 타입이 `A`또는 `B`일 때를 구분해서 로직을 처리하고 싶다면 어떻게 하는게 좋을까요?
  - if문을 사용해서 처리하면 될 것 같지만 컴파일 시 타입 정보는 모두 제거되어 런타임에 존재하지 않기 때문에 타입을 사용하여 조건을 만들 수는 없습니다.
    - **즉, 컴파일해도 타입 정보가 사라지지 않는 방법을 사용해야 합니다.**
  - 특정 문맥 안에서 타입스크립트가 해당 변수를 `A`타입으로 추론하도록 유도하면서 런타임에서도 유효한 방법이 필요한데, 이때 타입 가드를 사용하면 됩니다.
  - 타입 가드는 크게 자바스크립트 연산자를 사용한 타입 가드와 사용자 정의 타입 가드로 구분할 수 있습니다.
    - 자바스크립트 연산자를 활용한 타입 가드는 `typeof`, `instanceof`, `in`과 같은 연산자를 사용해서 제어문으로 특정 타입 값을 가질 수밖에 없는 상황을 유도하여 자연스럽게 타입을 좁히는 방식입니다.
      - 자바스크립트 연산자를 사용하는 이유는 런타임에 유효한 타입 가드를 만들기 위해서 입니다.
        > 가장 보편적으로 사용해왔던 방법들인것 같아요 `Object.prototype.toString.call(...)` 포함

#### 원시 타입을 추론할 때: typeof 연산자 활용하기

- `typeof`연산자를 활용하면 원시 타입에 대해 추론할 수 있습니다.

  - `typeof A === B`를 조건으로 분기처리하면, 해당 분기 내에서는 `A`의 타입이 `B`로 추론됩니다.
  - 다만 `typeof`는 자바스크립트 타입 시스템만 대응할 수 있습니다.
  - 자바스크립트의 동작 방식으로 인해 `null`과 배열 타입 등이 `object` 타입으로 판별되는 등 복잡한 타입을 검증하기에는 한계가 있습니다.
  - 이러한 이유로 `typeof`연산자는 주로 원시 타입을 좁히는 용도로만 사용할 것을 권장합니다.

  ```ts
  const replaceHyphen: (date: string | Date) => string | Date = (date) => {
    if (typeof date === “string”) {
    // 이 분기에서는 date의 타입이 string으로 추론된다
    return date.replace(/-/g, “/”);
    }

    return date;
  };
  ```

#### 인스턴스화된 객체 타입을 판별할 때: instanceof 연산자 활용하기

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
  // ...
};

export function convertToRange(selected?: Date | Range): Range | undefined {
  return selected instanceof Date
    ? { start: selected, end: selected }
    : selected;
}
```

- 위 예시 코드에서는 `selected`매개변수가 `Date`인지를 검사한 후에 `Range` 타입의 객체를 반환할 수 있도록 분기처리하고 있습니다.
- `typeof`연산자를 주로 원시 타입을 판별하는 데 사용한다면, `instanceof`연산자는 인스턴스화된 객체 타입을 판별하는 타입 가드로 사용할 수 있습니다.
- `A instanceof B`
  - `instanceof`는 `A`의 프로토타입 체인에 생성자 `B`가 존재하는지를 검사해서 존재한다면 true, 그렇지 않다면 false를 반환합니다.
  - 이러한 동작 방식으로 인해 `A`의 프로토타입 속성 변화에 따라 `instanceof` 연산자의 결과가 달라질 수 있다는 점은 유의해야 합니다.
  - 아래 예시는 `HTMLInputElement`에 존재하는 `blur`메서드를 사용하기 위해서, `event.target`이 `HTMLInputElement`의 인스턴스인지를 검사한 후 분기 처리하는 로직입니다.
  ```ts
  const onKeyDown = (event: React.KeyboardEvent) => {
    if (event.target instanceof HTMLInputElement && event.key === “Enter”) {
    // 이 분기에서는 event.target의 타입이 HTMLInputElement이며
    // event.key가 ‘Enter’이다
    event.target.blur();
    onCTAButtonClick(event);
    }
  };
  ```

#### 객체의 속성이 있는지 없는지에 따른 구분: in 연산자 활용하기

- `in`연산자를 사용하면 속성이 있는지 없는지에 따라 객체 타입을 구분할 수 있습니다.
- `in`연산자는 `A in B`의 형태로 사용하는데 이름 그대로 `A`라는 속성이 `B`객체에 존재하는지를 검사합니다.
  - `in`연산자는 객체에 속성이 있는지 확인한 다음에 true 또는 false를 반환합니다.
  - 프로토타입 체인으로 접근할 수 있는 속성이면 전부 true를 반환합니다.
  - `in`연산자는 `B`객체 내부에 `A`속성이 있는지 없는지를 검사하는 것이기 때문에 `B` 객체에 존재하는 `A`속성에 `undefined`를 할당한다고 해서 false를 반환하는 것은 아닙니다.

```ts
interface BasicNoticeDialogProps {
  noticeTitle: string;
  noticeBody: string;
}

interface NoticeDialogWithCookieProps extends BasicNoticeDialogProps {
  cookieKey: string;
  noForADay?: boolean;
  neverAgain?: boolean;
}

export type NoticeDialogProps =
  | BasicNoticeDialogProps
  | NoticeDialogWithCookieProps;
```

- 해당 예시를 사용하여 props 타입에 따라 렌더링하는 컴포넌트를 분기처리해보겠습니다.
- `NoticeDialogWithCookieProps`는 `BasicNoticeDialogProps`를 상속받고 `cookieKey`속성을 가집니다. 따라서 두 객체 타입을 `cookieKey`속성을 기준으로 `in`연산자로 조건을 만들 수 있습니다.

```ts
const NoticeDialog: React.FC<NoticeDialogProps> = (props) => {
  if (“cookieKey” in props) return <NoticeDialogWithCookie {...props} />;
  return <NoticeDialogBase {...props} />;
};
```

- if문 스코프에서 타입스크립트는 props 객체를 `cookieKey`속성을갖는 객체 타입인 `NoticeDialogWithCookie`로 해석하게 됩니다.
- 자바스크립트의 `in`연산자는 런타임의 값만을 검사하지만 타입스크립트에서는 객체 타입에 속성이 존재하는지를 검사합니다.

#### is 연산자로 사용자 정의 타입 가드 만들어 활용하기

- 타입 가드 함수를 직접 만들수도 있습니다.
- 해당 방식의 타입 가드는 반환 타입이 타입 명제(type predicates)인 함수를 정의하여 사용할 수 있습니다.
  - 타입 명제는 `A is B`형식으로 작성하면 되는데 여기서 `A`는 매개변수 이름이고 `B`는 타입입니다.
  - 참/거짓의 진릿값을 반환하면서 반환 타입을 타입 명제로 지정하게 되면 반환 값이 참일 때 `A`매개변수의 타입을 `B`타입으로 취급하게 됩니다.
    > ##### 타입 명제 (type predicates)
    >
    > 타입 명제는 함수의 반환 타입에 대한 타입 가드를 수행하기 위해 사용되는 특별한 형태의 함수입니다.

```ts
const isDestinationCode = (x: string): x is DestinationCode =>
  destinationCodeList.includes(x);
```

- 해당 예시 코드는 `string`타입의 매개변수가 `destinationCodeList`배열의 원소 중 하나인지 검사하여 boolean을 반환하는 함수입니다.
  - 함수의 반환 값을 boolean이 아닌 `x is DestinationCode`로 타이핑하여 타입스크립트에게 이 함수가 사용되는 곳의 타입을 추론할 때 해당 조건을 타입 가드로 사용하도록 알려줍니다.
- `isDestinationCode`함수의 반환값의 타입이 boolean인 것과 `is`를 활용한 것과의 차이를 알아보겠습니다.
  ```ts
  const getAvailableDestinationNameList = async (): Promise<DestinationName[]> => {
    const data = await AxiosRequest<string[]>(“get”, “.../destinations”);
    const destinationNames: DestinationName[] = [];
    data?.forEach((str) => {
    if (isDestinationCode(str)) {
      destinationNames.push(DestinationNameSet[str]);
      /*
      isDestinationCode의 반환 값에 is를 사용하지 않고 boolean이라고 한다면 다음 에러가 발생한다
      - Element implicitly has an ‘any’ type because expression of type ‘string’
      can’t be used to index type ‘Record<”MESSAGE_PLATFORM” | “COUPON_PLATFORM” | “BRAZE”,
      “통합메시지플랫폼” | “쿠폰대장간” | “braze”>’
      */
      }
    });
    return destinationNames;
  };
  ```
  - if문 내 `isDestinationCode` 함수로 data의 str이 `destinationCodeList`의 문자열 원소인지 체크하고, 맞는다면 `destinationNames`배열에 push를 진행합니다.
  - 만약 `isDestinationCode`의 반환값 타이핑이 boolean으로 되어있다면 타입스크립트는 어떻게 추론을 할까요?
    - 개발자는 if문 내부에서 str 타입이 `DestinationCode`라는 것을 알 수 있습니다. `Array.includes`메서드를 보고 해석할 수 있기 때문입니다.
    - 하지만 타입스크립트는 `isDestinationCode`함수 내부에 있는 `includes`함수를 해석하여 타입을 추론하지 못합니다.
    - 타입스크립트는 if문의 str 타입을 좁히지 못하고 `string`으로만 추론하게 됩니다.
    - `destinationNames`의 타입은 `DestinationName[]`이기 때문에 `string`타입의 str을 push 할 수 없다는 에러가 발생하게 됩니다.

> 저도 최근 타입가드를 구성할 일이 있었습니다. is 키워드를 사용했던 기억이 있네요
>
> ```ts
> /** 웹훅 타입 가드 */
> export const isTransactionWebhook = (
>   webhook: Webhook
> ): webhook is WebhookTransaction => {
>   if (typeof webhook.type === "string") {
>     return Boolean(webhook.type.startsWith("Transaction."));
>   }
>   return false;
> };
>
> export const isBillingKeyWebhook = (
>   webhook: Webhook
> ): webhook is WebhookBillingKey => {
>   if (typeof webhook.type === "string") {
>     return Boolean(webhook.type.startsWith("BillingKey."));
>   }
>   return false;
> };
> ```
>
> 지금 시점에서 다시 작성한다면 더 간단하게 구성할 수 있을거같아요

## 04.3 타입좁히기 - 식별할 수 있는 유니온 (Discriminated Unions)

- 종종 태그된 유니온(Tagged Union)으로도 불리는 식별할 수 있는 유니온은 타입 좁히기에 널리 사용되는 방식입니다.

### 에러 정의하기

- 사용 예시
  - 배민 선물하기 서비스에서는 선물을 보낼 때 필요한 값을 사용자가 올바르게 입력했는지를 확인하는 유효성 검사를 진행합니다.
  - 유효성 에러가 발생하면 사용자에게 다양한 방식으로 에러를 보여주게 됩니다.
  - 우아한형제들에서는 이 에러를 크게 `텍스트에러, 토스트에러, 얼럿에러`로 분류합니다.
  - 이들 모두 유효성 에러로 `에러코드`와 `에러 메세지`를 가지고 있으며, 에러 노출 방식에 따른 추가적인 정보를 필요로 하는 경우가 있을 수 있습니다.
    - 예를 들어 토스트에러는 토스트를 얼마 동안 띄울 것인지에 대한 정보가 필요합니다.
  ```ts
  type TextError = {
    errorCode: string;
    errorMessage: string;
  };
  type ToastError = {
    errorCode: string;
    errorMessage: string;
    toastShowDuration: number; // 토스트를 띄워줄 시간
  };
  type AlertError = {
    errorCode: string;
    errorMessage: string;
    onConfirm: () => void; // 얼럿 창의 확인 버튼을 누른 뒤 액션
  };
  ```
  - 해당 에러 타입의 유니온 타입을 원소로 하는 배열을 정의해보겠습니다.
  ```ts
  type ErrorFeedbackType = TextError | ToastError | AlertError;
  const errorArr: ErrorFeedbackType[] = [
    { errorCode: “100”, errorMessage: “텍스트 에러” },
    { errorCode: “200”, errorMessage: “토스트 에러”, toastShowDuration: 3000 },
    { errorCode: “300”, errorMessage: “얼럿 에러”, onConfirm: () => {} },
  ];
  ```
  - 이때, 해당 배열에 에러 타입별로 정의한 필드를 가지는 에러 객체가 포함되는 경우를 가정해보겠습니다.
    - 예를들어, `ToastError`의 `toastShowDuration` 필드와 `AlertError`의 `onConfirm`필드를 모두 가지는 객체에 대해서는 의도하지 않았으므로 타입 에러를 뱉어야 할 것입니다.
    - 하지만 자바스크립트는 **덕 타이핑** 언어이기 때문에 해당 상황에서 별도의 타입 에러를 뱉지 않습니다.
    - 이러한 의도하지 않은 상황에서 타입 에러가 발생하지 않는다면 앞으로의 개발 과정에서 의미를 알 수 없는 무수한 에러 객체가 생겨날 위험성이 커지게 됩니다.

### 식별할 수 있는 유니온

- 따라서 에러 타입을 구분할 방법이 필요합니다.
- 각 타입이 비슷한 구조를 가지지만 서로 호환되지 않도록 만들어주기 위해서는 타입들이 서로 포함관계를 가지지 않도록 정의해야 합니다.
- 이때 적용할 수 있는 방식이 식별할 수 있는 유니온을 활용하는 것입니다.
  - 식별할 수 있는 유니온이란 타입 간의 구조 호환을 막기 위해 타입마다 구분할 수 있는 판별자를 달아주어 포함 관계를 제거하는 것입니다.
- 판별자의 개념으로 `errorType`이라는 필드를 새로 정의해보겠습니다.

  - 각 에러 타입마다 이 필드에 대해 다른 값을 가지도록 하여 판별자를 달아주면 이들은 포함 관계를 벗어나게 됩니다.

  ```ts
  type TextError = {
    errorType: “TEXT”;
    errorCode: string;
    errorMessage: string;
  };
  type ToastError = {
    errorType: “TOAST”;
    errorCode: string;
    errorMessage: string;
    toastShowDuration: number;
  }
  type AlertError = {
    errorType: “ALERT”;
    errorCode: string;
    errorMessage: string;
    onConfirm: () = > void;
  };
  ```

  - 해당 에러 객체를 활용하여 `errorArr`을 새로 정의해보겠습니다.

  ```ts
  type ErrorFeedbackType = TextError | ToastError | AlertError;

  const errorArr: ErrorFeedbackType[] = [
    { errorType: “TEXT”, errorCode: “100”, errorMessage: “텍스트 에러” },
    {
      errorType: “TOAST”,
      errorCode: “200”,
      errorMessage: “토스트 에러”,
      toastShowDuration: 3000,
    },
    {
      errorType: “ALERT”,
      errorCode: “300”,
      errorMessage: “얼럿 에러”,
      onConfirm: () => {},
    },
    {
      errorType: “TEXT”,
      errorCode: “999”,
      errorMessage: “잘못된 에러”,
      toastShowDuration: 3000, // Object literal may only specify known properties, and ‘toastShowDuration’ does not exist in type ‘TextError’
      onConfirm: () => {},
    },
    {
      errorType: “TOAST”,
      errorCode: “210”,
      errorMessage: “토스트 에러”,
      onConfirm: () => {}, // Object literal may only specify known properties, and ‘onConfirm’ does not exist in type ‘ToastError’
    },
    {
      errorType: “ALERT”,
      errorCode: “310”,
      errorMessage: “얼럿 에러”,
      toastShowDuration: 5000, // Object literal may only specify known properties, and ‘toastShowDuration’ does not exist in type ‘AlertError’
    },
  ];
  ```

  - 이제는 의도한대로 정확하지 않은 에러 객체에 대하여 타입 에러가 발생하는 것을 확인할 수 있습니다.

### 식별할 수 있는 유니온 판별자 선정

- 식별할 수 있는 유니온을 사용할 때 주의할 점이 있습니다.
  - 식별할 수 있는 유니온 판별자는 유닛 타입(unit type)으로 선언되어야 정상적으로 동작합니다.
    - 유닛 타입이란 다른 타입으로 쪼개지지 않고 오직 하나의 정확한 값을 가지는 타입을 말합니다.
    - `null`, `undefined`, 리터럴 타입을 비롯하여 `true`, `1` 등 정확한 값을 나타내는 타입이 유닛 타입에 해당합니다.
    - 반면 다양한 타입을 할당할 수 있는 `void`, `string`, `number`와 같은 타입은 유닛 타입으로 적용되지 않습니다.
  - 공식 깃허브의 이슈 탭을 살펴보면 식별할 수 있는 유니온의 판별자로 사용할 수 있는 타입을 다음과 같이 정의하고 있습니다.
    - 리터럴 타입이어야 한다.
    - 판별자로 선정한 값에 적어도 하나 이상의 유닛 타입이 포함되어야 하며, 인스턴스화할 수 있는 타입(instantiable type)은 포함하지 않아야 한다.

## 04.4 Exhaustiveness Checking으로 정확한 타입 분기 유지하기

- Exhaustiveness는 사전적으로 철저함, 완전함을 의미합니다.
- 따라서 Exhaustiveness Checking은 모든 케이스에 대해 철저하게 타입을 검사하는 것을 말하며 타입 좁히기에 사용되는 패러타임 중 하나 입니다.
- 타입 가드를 사용해서 필요하다고 생각되는 부분만 타입에 대한 분기 처리를 하여 요구 사항에 맞출수도 있지만, 때로는 모든 케이스에 대해 분기 처리를 해야만 유지보수 측면에서 안전하다고 생각되는 상황이 생깁니다.
  - 이때 Exhaustiveness Checking을 통해 모든 케이스에 대한 타입 검사를 강제할 수 있습니다.

### 상품권

- 배민 선물하기 서비스에는 다양한 상품권이 존재합니다. 상품권 가격에 따라 상품권 이름을 반환해주는 함수를 작성하면 다음과 같습니다.

  ```ts
  type ProductPrice = “10000” | “20000”;

  const getProductName = (productPrice: ProductPrice): string => {
    if (productPrice === “10000”) return “배민상품권 1만 원”;
    if (productPrice === “20000”) return “배민상품권 2만 원”;
    else {
      return “배민상품권”;
    }
  };
  ```

  - 여기까지는 큰 문제가 없다고 느껴질 수 있습니다. 하지만 새로운 상품권이 생겨서 `ProductPrice`타입이 업데이트 되어야 한다고 가정해보겠습니다.

  ```ts
  type ProductPrice = “10000” | “20000” | “5000”;

  const getProductName = (productPrice: ProductPrice): string => {
    if (productPrice === “10000”) return “배민상품권 1만 원”;
    if (productPrice === “20000”) return “배민상품권 2만 원”;
    if (productPrice === “5000”) return “배민상품권 5천 원”; // 조건 추가 필요
    else {
      return “배민상품권”;
    }
  };
  ```

  - `ProductPrice`타입이 업데이트 되었을 때 `getProductName`함수도 함께 업데이트 되어야 합니다.
  - 그러나 `getProductName`함수를 수정하지 않아도 별도의 에러가 발생하는 것이 아니기 때문에 실수할 여지가 존재합니다.
  - 이와 같이 모든 타입에 대한 타입 검사를 강제하고 싶다면 다음과 같이 개선하면 됩니다.

  ```ts
  type ProductPrice = “10000” | “20000” | “5000”;

  const getProductName = (productPrice: ProductPrice): string => {
    if (productPrice === “10000”) return “배민상품권 1만 원”;
    if (productPrice === “20000”) return “배민상품권 2만 원”;
    // if (productPrice === “5000”) return “배민상품권 5천 원”;
    else {
      exhaustiveCheck(productPrice); // Error: Argument of type ‘string’ is not assignable to parameter of type ‘never’
      return “배민상품권”;
    }
  };

  const exhaustiveCheck = (param: never) => {
    throw new Error(“type error!”);
  };
  ```

  - `exhaustiveCheck`에서 에러를 뱉고있는데 `ProductPrice`타입 중 5000 의 값에 대한 분기 처리를 하지 않아서 발생하는 것 입니다.
    - 자세히 살펴보면 `exhaustiveCheck`함수의 매개변수를 `never`타입으로 선언하고 있습니다.
    - 즉, 매개변수로 그 어떤 값도 받을 수 없으며 만일 값이 들어온다면 에러를 내밷습니다.
    - 해당 함수를 타입 처리 조건문의 마지막 else문에 사용하면 앞의 조건문에서 모든 타입에 대한 분기 처리를 강제할 수 있습니다.
  - 이렇게 모든 케이스에 대한 타입 분기 처리를 해주지 않았을 때, 컴파일타임 에러를 유도하는 것을 Exhaustiveness Checking이라고 합니다.

- Exhaustiveness Checking을 활용하면 예상치 못한 런타임 에러를 방지하거나 요구사항이 변경되었을때 생길 수 있는 위험성을 줄일 수 있습니다.
- 철저한 분기 처리가 필요하다면 Exhaustiveness Checking패턴을 활용해보시길 바랍니다.

### 우형 이야기

- 우형 개발자분들의 대화를 정리해봤어요
  - `never`타입이 참 신기한 것 같아요. "어떤 값이든 never 그 자체를 제외하고는 never로 할당할 수 없다"라는 개념 하나로 다양한 패턴을 만들어 타입을 잡아주는 경우가 있어요.
  - 코드를 보면서 드는 생각은 런타임 코드에 `exhaustiveCheck`가 포함되어 버려서 뭔가 프로덕션 코드와 테스트 코드가 같이 섞여 있는 듯한 느낌(프로덕션 코드 중간중간에 assert 구문을 넣는듯한 느낌)이 조금 들었어요.
  - 프로덕션 코드에 Assertion을 추가하는 것도 하나의 패턴이에요. 단위 테스트의 Assertion은 특정 단위의 결과를 확인하는 느낌이라면, 코드상의 Assertion은 코드 중간중간에 무조건 특정 값을 가지는 상황을 확인하기 위한 디버깅 또는 주석 같은 느낌으로 사용되는것 같아요.
    - `리펙터링 2판`에서는 "테스트코드가 있다면 어서션이 디버깅 용도로 사용될 때의 효용은 줄어든다. 단위 테스트를 꾸준히 추가하여 사각을 좁히면 어서션보다 나을 때가 많다. 하지만 소통 측면에서는 어서션이 여전히 매력적이다" 라고 나와있긴 합니다.
    - 개인적으로 어서션 자체가 성능에 영향을 주는게 거의 없다보니까 있어도 나쁘지 않은 것 같아요.
    - 아무 주석을 단다거나 문서화가 안 되어 있는 것보다는 낫지 않을까 싶어요.
