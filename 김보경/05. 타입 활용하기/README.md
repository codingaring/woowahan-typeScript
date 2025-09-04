> 이 장에서는 우아한형제들의 타입스크립트 활용 사례를 소개합니다.
> 우아한 형제들의 실무 코드 예시를 살펴보면서 정확한 타이핑을 하지 못해 발생하는 문제를 타입스크립트의 다양한 기법과 유틸리티 타입을 활용해 해결해봅니다.

## 05.1 조건부 타입

- 프로그래밍에서는다양한 상황을 다루기 위해 조건문을 많이 활용합니다.
- 타입 또한 조건에 따라 다른 타입을 반환해야 할 때가 있습니다.
- 타입스크립트에서는 조건부 타입을 사용해 조건에 따라 출력 타입을 다르게 도출할 수 있습니다.
  - 타입스크립트의 조건부 타입은 자바스크립트의 삼항 연산자와 동일하게 `Condition ? A : B`형태를 가지고 있습니다.
- 조건부 타입을 활용하면 중복되는 타입코드를 제거하고 상황에 따라 적절한 타입을 얻을 수 있기 때문에 더욱 정확한 타입 추론을 할 수 있게 됩니다.
- 해당 챕터에서는 `extends, infer, never`등을 활용하여 원하는 타입을 만들어보며 어떤 상황에서 조건부 타입이 필요한지 알아보겠습니다.

### extends와 제네릭을 활용한 조건부 타입

- `extends`키워드는 타입스트립트에서 다양한 상황에서 활용됩니다.
  - 타입을 확장할때, 타입을 조건부로 설정할때 사용
  - 제네릭 타입에서는 한정자 역할로도 사용됩니다.
- `extends`를 사용한 조건부 타입의 활용 예시를 보기 전에 간단하게 `extends`가 어떻게 조건부 타입으로 사용되는지 알아보겠습니다.

  ```ts
  T extends U ? X : Y
  ```

  - 조건부 타입에서 `extends`를 사용할 때는 자바스크립트 삼항 연산자와 함께 사용합니다.

  ```ts
  interface Bank {
    financialCode: string;
    companyName: string;
    name: string;
    fullName: string;
  }
  interface Card {
    financialCode: string;
    companyName: string;
    name: string;
    appCardType?: string;
  }
  type PayMethod<T> = T extends "card" ? Card : Bank;
  type CardPayMethodType = PayMethod<"card">;
  type BankPayMethodType = PayMethod<"bank">;
  // extends 키워드는 일반적으로 문자열 리터럴과 함께 사용하지는 않지만, 예시에서는 extends의 활용법을 설명하기 위해 문자열 리터럴에 사용되고 있습니다.
  ```

  - `Bank`는 계좌를 이용한 결제 수단이며 고유 코드인 financialCode, name 등을 가지고 있습니다. `Card`타입과 다른 점은 fullName으로 은행의 전체 이름 속성을 가지고 있다는 것입니다.
  - `Card`는 카드를 이용한 결제 수단 정보입니다. `Bank`와 마찬가지로 유사한 속성을 가지고 있으며, 다른점으로는 appCardType으로 카드사 입을 사용해서 카드 정보를 등록할 수 있는지를 구별해주는 속성이 있다는 것입니다.
  - `PayMethod`타입은 제네릭 타입으로 `extends`를 사용한 조건부 타입입니다.
    - 제네릭 매개변수에 "card"가 들어오면 `Card`타입, 그 외의 값이 들어오면 `Bank`타입으로 결정됩니다.
    - `PayMethod`를 사용해서 두가지 타입을 도출할 수 있습니다.

### 조건부 타입을 사용하지 않았을 때의 문제점

- 조건부 타입을 사용하기 전에 어떤 이슈가 있었는지 살펴보겠습니다.

  - 예시에서는 계좌, 카드, 앱카드 등 3가지 결제 수단 정보를 가져오는 API가 있으며, API의 엔드포인트는 아래와 같습니다.

  ```md
  - 계좌 정보 엔드포인트: www.baemin.com/baeminpay/.../bank
  - 카드 정보 엔드포인트: www.baemin.com/baeminpay/.../card
  - 앱카드 정보 엔드포인트: www.baemin.com/baeminpay/.../appcard
  ```

  - 각 API은 계좌, 카드, 앱카드의 결제 수단 정보를 배열 형태로 반환합니다.
  - 3가지 API의 엔드포인트가 비슷하기 때문에 서버 응답을 처리하는 공통 함수를 생성하고, 해당 함수에 타입을 전달하여 타입별로 처리 로직을 구현할 것입니다.

    - 함수를 구현하기 앞서 이전에 사용되는 타입들을 서버에서 받아오는 타입, UI 관련 타입 그리고 결제 수단별 특성에 따라 구분하여 살펴보겠습니다.
      - 주석으로 추가 작성 했어요!

    ```ts
    /** 서버에서 받아오는 결제 수간 기본 타입으로 은행과 카드에 모두 구성되어있습니다. */
    interface PayMethodBaseFromRes {
      financialCode: string;
      name: string;
    }

    /** 은행과 카드 각각에 맞는 결제 수단 타입입니다. 결제 수단 기본 타입인 PayMethodBaseFormRes를 상속받아 구현합니다. */
    interface Bank extends PayMethodBaseFromRes {
      fullName: string;
    }
    interface Card extends PayMethodBaseFromRes {
      appCardType?: string;
    }

    /**
     * 최종적인 은행, 카드 결제수단 타입입니다.
     * 프론트에서 추가되는 UI 데이터 타입과 제네릭으로 받아오는 Bank 또는 Card를 합성합니다.
     * extends를 제네릭에서 한정자로 사용하여 Bank 또는 Card를 포함하지 않는 타입은 제네릭으로 넘겨주지 못하게 방어합니다.
     * BankPayMethodInfo = PayMethodInterface & Bank 처럼 카드와 은행의 타입을 만들어 줄 수 있지만 제네릭을 활용해서 중복된 코드를 제거합니다.
     */
    type PayMethodInfo<T extends Bank | Card> = T & PayMethodInterface;
    type PayMethodInterface = {
      companyName: string;
      //...
    };
    ```

  - 다음으로는 `useGetRegisteredList`함수를 살펴보겠습니다.

    - `useCommonQuery<T>`는 useQuery를 한 번 래핑해서 사용하고 있는 함수로 useQuery의 반환 data를 T타입으로 반환합니다.
    - `fetcherFactory`는 axios를 래핑해주는 함수이며, 서버에서 데이터를 받아온 후 onSuccess 콜백 함수를 거친 결괏값을 반환합니다.

    ```ts
    type PayMethodType = PayMethodInfo<Card> | PayMethodInfo<Bank>;

    export const useGetRegisteredList = (
      type: "card" | "appcard" | "bank"
    ): UseQueryResult<PayMethodType[]> => {
      const url = `baeminpay/codes/${type === "appcard" ? "card" : type}`;
      const fetcher = fetcherFactory<PayMethodType[]>({
        onSuccess: (res) => {
          const usablePocketList =
            res?.filter(
              (pocket: PocketInfo<Card> | PocketInfo<Bank>) =>
                pocket?.useType === "USE"
            ) ?? [];
          return usablePocketList;
        },
      });

      const result = useCommonQuery<PayMethodType[]>(url, undefined, fetcher);

      return result;
    };
    ```

    - `useGetResisteredList`는 함수 타입으로 "card", "appcard", "bank"를 받아서 해당 결제 수단의 결제 수단 정보 리스트를 반환하는 함수입니다.
      - 함수가 반환하는 Data 타입은 `PoketInfo<Card> | PoketInfo<Bank>`입니다.
      - 사용자가 타입으로 "card"를 넣었을 때 `useGetRegisteredList`가 반환하는 Data 타입은 `PoketInfo`라고 유추할 수 있습니다.
      - 하지만 `useGetRegisteredList`함수가 반환하는 Data 타입은 `PayMethodType`이기 때문에 사용하는 쪽에서는 `PoketInfo`일 가능성도 있습니다.
    - `useGetResisteredList`함수는 타입을 구분해서 넣는 사용자의 의도와는 다르게 정확한 타입을 반환하지 못하는 함수가 되었습니다.
      - 인자로 넣는 타입에 알맞은 타입을 반환하고 싶지만, 타입 설정이 유니온으로만 되어있기 때문에 타입스크립트는 해당 타입에 맞는 Data 타입을 추론할 수 없습니다.

### extends 조건부 타입을 활용하여 개선하기

- `useGetResisteredList`함수의 반환 Data는 인자 타입에 따라 정해져 있습니다.
  - `type: "card" | "appcard" => PoketInfo<Card>`
  - `type: "bank" => PoketInfo<Bank>`
- 게좌와 카드의 API 함수를 각각 만들 수 도 있지만 엔드포인트의 마지막 경로만 다르다고 계좌와 카드가 같은 컴포넌트에서 사용되기 때문에 하나의 함수에서 한 번에 관리해야 하는 상황이라고 가정해보겠습니다.
- 이러한 상황에서 조건부 타입을 활용하면 하나의 API 함수에서 타입에 따른 정확한 반환 타입을 추론하게 만들 수 있습니다.

  - 조건부 타입을 개선해서 `PayMethodType`을 개선해보겠습니다.

    ```ts
    type PayMethodType<T extends "card" | "appcard" | "bank"> = T extends
      | "card"
      | "appcard"
      ? Card
      : Bank;
    ```

    - `PayMethodType`의 제네릭으로 받은 값으로 각각 알맞는 타입을 반환하도록 수정했습니다.
    - 또한 결제 수단 타입에는 "card", "appcard", "bank"만 들어올 수 있기 때문에 `extends`를 한정자로 사용해서 제네릭에 넘겨오는 값을 제한하도록 했습니다.
      - 새롭게 정의한 `PayMethodType`타입에 제네릭 값을 넣어주기 위해서는 `useGetResisteredList`함수의 인자 타입을 넣어주어야 합니다.
    - 인자 타입을 제네릭으로 받으면서 `extends`를 활용하여 "card", "appcard", "bank" 이외에 다른 값이 인자로 들어오는 경우에는 타입 에러를 반환하도록 구현해보겠습니다.

      ```ts
      export const useGetRegisteredList = <
        T extends "card" | "appcard" | "bank"
      >(
        type: T
      ): UseQueryResult<PayMethodType<T>[]> => {
        const url = `baeminpay/codes/${type === "appcard" ? "card" : type}`;

        const fetcher = fetcherFactory<PayMethodType<T>[]>({
          onSuccess: (res) => {
            const usablePocketList =
              res?.filter(
                (pocket: PocketInfo<Card> | PocketInfo<Bank>) =>
                  pocket?.useType === "USE"
              ) ?? [];
            return usablePocketList;
          },
        });

        const result = useCommonQuery<PayMethodType<T>[]>(
          url,
          undefined,
          fetcher
        );

        return result;
      };
      ```

      - 조건부 타입을 활용하여 `PayMethodType`이 사용자가 인자에 넣는 타입 값에 맞는 타입만을 반환하도록 구현했습니다.
      - 이제 사용자는 `useGetResisteredList`함수를 사용할 때 불필요한 타입 가드를 하지 않아도 됩니다.
      - 또한 데이터를 `PoketInfo<Card>`만을 받는 컴포넌트의 props로 넘겨줄 때 불필요한 타입 단언을 하지 않아도 됩니다.

- 현재까지의 `extends`활용 사례를 바탕으로 크게 아래와 같이 정리해볼 수 있습니다.
  1. 제네릭과 `extends`를 함께 사용하여 제네릭으로 받는 타입을 제한했습니다. 따라서 개발자는 잘못된 값을 넘길 수 없기 때문에 휴먼 에러를 방지할 수 있습니다.
  2. `extends`를 활용하여 조건부 타입을 설정했습니다. 조건부 타입을 사용해서 반환 값을 사용자가 원하는 값으로 구체화할 수 있었습니다. 이에 따라 불필요한 타입 가드, 타입 단언 등을 방지할 수 있습니다.

### infer를 활용해서 타입 추론하기

- `extends`를 사용할 때 `infer`키워드를 사용할 수 있습니다.
- `infer`는 **"추론하다"** 라는 의미를 지니고 있습니다. 타입스크립트에서도 단어 의미처럼 타입을 추론하는 역할을 합니다.
- 삼항 연산자를 사용한 조건문의 형태를 가지는데, `extends`로 조건을 서술하고 `infer`로 타입을 추론하는 방식을 취합니다.

  ```ts
  type UnpackPromise<T> = T extends Promise<infer K>[] ? K : any;

  // 사용 예시
  const promises = [Promise.resolve("Mark"), Promise.resolve(38)];
  type Expected = UnpackPromise<typeof promises>; // string | number
  ```

  - `UnpackPromise`타입은 제네릭으로 T를 받아 `T`가 `Promise`로 래핑된 경우라면 `K`를 반환하고, 그렇지 않은 경우에는 `any`를 반환합니다.
    - `Promise<infer K>`는 Promise의 반환 값을 추론해 해당 값의 타입을 `K`로 한다는 의미입니다.
  - 이처럼 `extends`와 `infer`, 제네릭을 활용하면 타입을 조건에 따라 더 세밀하게 사용할 수 있게 됩니다.

- 이번에는 배민 라이더를 관리하는 라이더 어드민 서비스에서 사용하는 타입으로 `infer`를 살펴보겠습니다.

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
    // ...
  ];
  ```

  - `RouteBase`와 `RouteItem`은 라이더 어드민에서 라우팅을 위해 사용하는 타입입니다.
  - `route`같이 배열 형태로 사용되며, 권한 API로 반환된 사용자 권한과 name을 비교하여 인가되지 않은 사용자의 접근을 방지합니다.
  - `RouteItem`의 name은 pages가 있을 때는 단순히 이름의 역할만 하며 그렇지 않을 때는 사용자 권한과 비교합니다.

  ```ts
  export interface SubMenu {
    name: string;
    path: string;
  }

  export interface MainMenu {
    name: string;
    path?: string;
    subMenus?: SubMenu[];
  }

  export type MenuItem = MainMenu | SubMenu;
  export const menuList: MenuItem[] = [
    {
      name: "계정 관리",
      subMenus: [
        {
          name: "기기 내역 관리",
          path: "/device-history",
        },
        {
          name: "헬멧 인증 관리",
          path: "/helmet-certification",
        },
      ],
    },
    {
      name: "운행 관리",
      path: "/operation",
    },
    // ...
  ];
  ```

  - `MainMenu`와 `SubMenu`는 메뉴 리스트에서 사용하는 타입으로 권한 API를 ㅌ통해 반환된 사용자 권한과 name을 비교하여 사용자가 접근할 수 있는 메뉴만 렌더링합니다.
    - `MainMenu`의 name은 `subMenus`를 가지고 있을 때 단순히 이름의 역할만 하며, 그렇지 않을 때에는 권한으로 간주됩니다.
    - `menuList`에는 `MainMenu`와 `SubMenu` 타입이 올 수 있기 때문에 유니온 타입인 `MainItem`를 정의하여 사용하고 있습니다.
    - 따라서 `menuList`에서 `subMenus`가 없는 `MainMenu`의 name과 `subMenus`에서 쓰이는 name, route name에 동일한 문자열만 입력해야 한다는 제약이 존재합니다.
  - 하지만 앞서 말한 바와 같이 name은 string타입으로 정의되어 있기 때문에 routes와 menuList에서 subMenus의 기기 내역 관리처럼 서로 다른 값이 입력되어도 컴파일타임에서 에러가 발생하지 않습니다.
  - 또한 런타임에서도 인가되지 않음을 안내하는 페이지를 보여주거나 메뉴 리스트를 렌더링하지 않는 정도에 그치기 때문에, 존재하지 않는 권한에 대한 문제로 잘못 인지할 수 있습니다.
  - 이때 `infer`와 불변 객체(as const)를 활용해서 `menuList` 또는 `routes`의 값을 추출하여 타입으로 정의하는 식으로 개선할 수 있습니다.

  ```ts
  export interface MainMenu {
    // ...
    subMenus?: ReadonlyArray<SubMenu>;
  }

  export const menuList = [
    // ...
  ] as const;

  interface RouteBase {
    name: PermissionNames;
    path: string;
    component: ComponentType;
  }

  export type RouteItem =
    | {
        name: string;
        path: string;
        component?: ComponentType;
        pages: RouteBase[];
      }
    | {
        name: PermissionNames;
        path: string;
        component?: ComponentType;
      };
  ```

  - 먼저 subMenus의 타입을 `ReadonlyArray`로 변경하고, menuList에 `as const` 키워드를 추가하여 불변 객체로 정의합니다.
  - `Route`관련 타입의 name은 menuList의 name에서 추출한 타입인 `PermissionNames`만 올 수 있도록 변경합니다.

  ```ts
  type UnpackMenuNames<T extends ReadonlyArray<MenuItem>> =
    T extends ReadonlyArray<infer U>
      ? U extends MainMenu
        ? U["subMenus"] extends infer V
          ? V extends ReadonlyArray<SubMenu>
            ? UnpackMenuNames<V>
            : U["name"]
          : never
        : U extends SubMenu
        ? U["name"]
        : never
      : never;
  ```

  - 그다음 조건에 맞는 값을 추출할 `UnpackMenuNames`라는 타입을 추가했습니다.

    - `UnpackMenuNames`는 불변 객체인 `MenuItem` 배열만 입력으로 받을 수 있도록 제한되어 있으며, `infer U`를 사용하여 배열 내부 타입을 추론합니다.
    - 코드를 자세히 살펴보면 다음과 같은 동작을 수행합니다.

      - `U`가 `MainMenu`타입이라면 subMenus를 `infer V`로 추출합니다.
      - subMenus는 옵셔널한 타입이기 때문에 추출한 `V`가 존재한다면(`SubMenu`타입에 할당할 수 있다면) `UnpackMenuNames`에 다시 전달합니다.
      - `V`가 존재하지 않는다면 `MainMenu`의 name은 권한에 해당하므로 `U["name"]`입니다.
      - `U`가 `MainMenu`가 아니라 `SubMenu`에 할당할 수 있다면(U는 `SubMenu` 타입이기 때문에) `U["name"]`은 권한에 해당합니다.

      ```ts
      export type PermissionNames = UnpackMenuNames<typeof menuList>;
      // [기기 내역 관리, 헬멧 인증 관리, 운행 관리]
      ```

  - `PermissionNames`는 menuList에서 권한으로 유효한 값만 추출하여 배열로 반환하는 타입을 확인할 수 있습니다.

#### 너무 이해가 안되어서 다시 풀어서 써보겠습니다

1. 기본 이해:
   - `UnpackMenuNames`는 재귀적 조건부 타입.
   - 재귀를 사용하여 특정 값을 추출하고 있음
2. 목표는 menuList 에서 실제 권한에 해당하는 name들만을 추출하여 타입으로 만들기임!
   - 예시를 다시 보면 카테고리명에 해당하는 "계정 관리"는 권한이 아님을 알 수 있다.
3. `UnpackMenuNames`타입의 단계별 분석

   ```ts
   type UnpackMenuNames<T extends ReadonlyArray<MenuItem>> =
     T extends ReadonlyArray<infer U> // 1단계: 배열에서 요소 타입 추출
       ? U extends MainMenu // 2단계: MainMenu인지 확인
         ? U["subMenus"] extends infer V // 3단계: subMenus 추출
           ? V extends ReadonlyArray<SubMenu> // 4단계: subMenus가 있는지 확인
             ? UnpackMenuNames<V> // 5단계: 재귀 호출
             : U["name"] // 6단계: subMenus 없으면 name 반환
           : never
         : U extends SubMenu // 7단계: SubMenu라면 name 반환
         ? U["name"]
         : never
       : never;
   ```

   1. T extends ReadonlyArray<infer U>
      - menuList의 타입에서 각 요소의 타입을 U로 추출
      - T = [MainMenu, MainMenu, ...]
      - U = MainMenu (배열의 요소 타입)
   2. U extends MainMenu
      - 추출한 요소가 MainMenu 타입인지 확인
      - U가 MainMenu라면 ? subMenus 확인 로직으로 이동 : SubMenu 확인 로직으로 이동
   3. U["subMenus"] extends infer V ? V extends ReadonlyArray<SubMenu>
      - MainMenu의 subMenus 속성을 V로 추출하고, V가 실제로 SubMenu 배열인지 확인
   4. UnpackMenuNames<V> (재귀 호출)
      - subMenus가 있다면, 그 subMenus 배열에 대해 다시 UnpackMenuNames 실행
      - 이때는 SubMenu[] 배열이므로 7단계로 이동하여 각 SubMenu의 name 반환
   5. U["name"] (subMenus 없는 MainMenu)
      - subMenus가 없는 MainMenu라면 그 name이 권한명
      - ex: "운행 관리"
   6. U extends SubMenu ? U["name"]
      - SubMenu라면 그 name이 권한명
      - ex: "기기 내역 관리", "헬멧 인증 관리"

## 05.2 템플릿 리터럴 타입 활용하기

- 타입스크립트에서는 유니온 타입을 사용하여 변수 타입을 특정 문자열로 지정할 수 있습니다.

  ```ts
  type HeaderTag = "h1" | "h2" | "h3" | "h4" | "h5";
  ```

  - 해당 기능을 활용하면 컴파일 타임의 변수에 할당되는 타입을 특정 문자열로 정확하게 검사하여 휴먼 에러를 방지할 수 있고, 자동 완성 기능을 통해 개발 생산성을 높일 수 있습니다.

- 템플릿 리터럴 타입은 자바스크립트의 템플릿 리터럴 문법을 사용해 특정 문자열에 타입을 선언할 수 있는 기능입니다.

  - 앞선 예시의 `HeaderTag`타입은 템플릿 리터럴 타입을 사용하여 다음과 같이 선언할 수 있습니다.

  ```ts
  type HeadingNumber = 1 | 2 | 3 | 4 | 5;
  type HeaderTag = `h${HeadingNumber}`;
  ```

  - 수평 또는 수직 방향을 표현하는 `Direction`타입을 다음과 같이 표현할 수 있습니다.

  ```ts
  type Direction =
    | "top"
    | "topLeft"
    | "topRight"
    | "bottom"
    | "bottomLeft"
    | "bottomRight";
  ```

  - 이 코드의 `Direction`타입은 중복되는 문자열들이 합쳐져있는 문자열로 선언되어 있습니다.
  - 이 코드에 템플릿 리터럴 타입을 적용하면 다음과 같이 좀 더 명확하게 표시할 수 있습니다.

  ```ts
  type Vertical = "top" | "bottom";
  type Horizon = "left" | "right";
  type Direction = Vertical | `${Vertical}${Capitalize<Horizon>}`;
  ```

#### 주의할 점

- 타입스크립트 컴파일러가 유니온을 추론하는 데 시간이 오래 걸리면 비효율적이기 때문에 타입스크립트가 타입을 추론하지 않고 에러를 내밷을 때가 있습니다.
- 따라서 템플릿 리터럴 타입에 삽입된 유니온 조합의 경우의 수가 너무 많지 않게 적절하게 나누어 타입을 정의하는 것이 좋습니다.
- 예를 들어, 10_000개의 경우의 수를 가지는 유니온 타입을 중복하여 사용하는 경우에는 10_000 \* 10_000의 경우의 수를 가지는 유니온 타입이 탄생하게 됩니다.

```ts
type Digit = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
type Chunk = `${Digit}${Digit}${Digit}${Digit}`;
type PhoneNumberType = `010-${Chunk}-${Chunk}`;
```

> 왜이렇게 투머치처럼 느껴지죠..?

## 05.3 커스텀 유틸리티 타입 활용하기

- 가끔 타입스크립트를 사용하다 보면 표현하기 어려운 타입을 마주할 때가 있습니다.
  - 타입스크립트에서 제공하는 유틸리티 타입으로만 표현하기 어려운 경우.
- 이러한 경우에는 유틸리티 타입을 활용한 커스텀 유틸리티 타입을 제작해서 사용하면 됩니다.

### 유틸리티 함수를 활용해 styled-components의 중복 타입 선언 피하기

- styling에 사용되는 props는 해당 타입을 정확하게 작성해주어야 합니다.
- 이런 경우 `Pick`, `Omit`같은 유틸리티 타입을 잘 활용하여 코드를 간결하게 작성할 수 있습니다.

#### Props타입과 styled-components 타입의 중복 선언 및 문제점

- 아래 컴포넌트는 수평선을 그어주는 `Hr`컴포넌트 입니다.

  ```ts
  // HrComponent.tsx
  export type Props = {
    height?: string;
    color?: keyof typeof colors;
    isFull?: boolean;
    className?: string;
    ...
  }

  export const Hr: VFC<Props> = ({ height, color, isFull, className }) => {
    ...

    return <HrComponent height= {height} color= {color} isFull= {isFull} className= {className} />;
  };

  // style.ts
  import { Props } from '...';

  type StyledProps = Pick<Props, "height" | "color" | "isFull">;

  const HrComponent = styled.hr<StyledProps>`
    height: ${({ height }) = > height || "10px"};
    margin: 0;
    background-color: ${({ color }) = > colors[color || "gray7"]};
    border: none;
    ${({ isFull }) => isFull && css`
      margin: 0 -15px;
    `}
  `;
  ```

  - 해당 컴포넌트에 `StyledProps`를 따로 정의하려면 `Props`와 똑같은 타입임에도 새로 작성해야 하무로 불가피하게 중복된 코드가 생기게 됩니다.
    - 그리고 `Props`의 속성이 변경되면 `StyledProps`도 같이 변경되어야 합니다.

- 이런 문제를 `Pick`, `Omit`같은 유틸리티 타입으로 개선할 수 있습니다.

```ts
// 직접 선언한 props 타입
export type Props = {
  height?: string
  color?: key of type of Color
  isFull?: boolean
  className?: string
}

// 유틸리티 타입 Pick을 사용하여 선언한 StyledProps 타입
type StyledProps = Pick<Props, "height" | "color" | "isFull">
```

- 이외에도 상속받는 컴포넌트 혹은 부모 컴포넌트에서 자식 컴포넌트로 넘겨주는 props등에도 `Pick`, `Omit`같은 유틸리티 타입을 활용할 수 있습니다.

### PickOne 유틸리티 함수

- 타입스크립트에는 서로 다른 2개 이상의 객체를 유니온 타입으로 받을 때 타입 검사가 제대로 진행되지 않는 이슈가 존재합니다.

  ```ts
  type Card = {
    card: string
  };

  type Account = {
    account: string
  };

  function withdraw(type: Card | Account) {
    ...
  }

  withdraw({ card: "hyundai", account: "hana" });
  ```

  - 해당 코드에서는 `Card`, `Account`중 하나의 객체만 받고 싶은 상황에서 유니온 타입을 작성하면 의도한대로 타입 검사가 이루어지지 않습니다.
  - 또한 withdraw 함수의 인자로 서로 속성 중 하나만들 받고 싶지만 실제로는 서로 다른 두 속성을 모두 받아도 타입 에러가 발생하지 않습니다.

    - 왜 타입 에러가 발생하지 않을까요?
      - 지난 회차에서 언급한 바와 같이 집합 관점으로 볼 때 유니온은 합집합이 되기 때문입니다.
      - 따라서 두 속성이 모두 포함되어도 합집합의 범주에 들어가기 때문에 타입 에러가 발생하지 않게 됩니다.

  - 이런 문제를 해결하기 위해 타입스크립트에서는 **식별할 수 있는 유니온 기법**을 자주 활용합니다.

#### 식별할 수 있는 유니온으로 객체 타입을 유니온으로 받기

- 식별할 수 있는 유니온은 각 타입에 type 이라는 공통된 속성을 추가하여 구분짓는 방법입니다.

  ```ts
  type Card = {
    type: "card";
    card: string;
  };

  type Account = {
    type: "account";
    account: string;
  };

  function withdraw(type: Card | Account) {
    ...
  }

  withdraw({ type: "card", card: "hyundai" });
  withdraw({ type: "account", account: "hana" });
  ```

  - 식별할 수 있는 유니온으로 문제를 해결할 수 있지만 일일이 type을 다 넣어줘야 하는 불편함이 생깁니다.

#### PickOne 커스텀 유틸리티 타입 구현하기

- 타입스크립트에서 제공하는 유틸리티 타입을 활용해서 커스텀 유틸리티 타입을 만들어보겠습니다.

  - 구현하고자 하는 타입은 account 또는 card 속성 하나만 존재하는 객체를 받는 타입입니다.
  - 각각의 타입에서 서로 다른 속성을 허용하지 않도록 하려면 하나의 속성이 들어왔을때 다른 타입을 옵셔널한 `undefined`값으로 지정하는 방법을 생각해볼 수 있습니다.
  - 옵셔널 + `undefined`로 타입을 지정하면 사용자가 의도적으로 `undefined`값을 넣지 않는 이상, 원치 않는 속성에 값을 넣었을 때 타입 에러가 발생할 것입니다.

    ```ts
    { account: string; card?: undefined } | { account?: undefined; card: string }

    type PayMethod =
    | { account: string; card?: undefined; payMoney?: undefined }
    | { account: undefined; card?: string; payMoney?: undefined }
    | { account: undefined; card?: undefined; payMoney?: string };
    ```

  - 결국 선택하고자 하는 하나의 속성을 제외한 나머지 값을 옵셔털 타입 `undefined`로 설정하면 원하고자 하는 속성만 받도록 구현할 수 있습니다.
  - 이를 커스텀 유틸리티 타입으로 구현해보겠습니다.

    ```ts
    type PickOne<T> = {
      [P in keyof T]: Record<P, T[P]> &
        Partial<Record<Exclude<keyof T, P>, undefined>>;
    }[keyof T];
    ```

#### PickOne 살펴보기

- 앞의 유틸리티 타입을 하나씩 뜯어보자.
- 먼저 `PickOne`타입을 2가지 타입으로 분리해서 생각할 수 있다.

  > 이때 T에는 객체가 들어온다고 가정하자.

  - One<T>

    ```ts
    type One<T> = { [P in keyof T]: Record<P, T[P]> }[keyof T];
    ```

    - `[P in keyof T]`에서 T는 객체로 가정하기 때문에 P는 T 객체의 키값을 말합니다.
    - `Record<P, T[P]>`는 P타입을 키로 가지고, value는 P를 키로 둔 T 객체의 값의 레코드 타입을 말합니다.
    - 따라서 `{ [P in keyof T]: Record<P, T[P]> }`에서 키는 T 객체의 키 모음이고, value는 해당 키의 원본 객체 T를 말합니다.
    - 위의 타입에서 다시 `[keyof T]`의 키값으로 접근하기 때문에 최종 결과는 전달받은 T와 같습니다.

    ```ts
    type Card = { card: string };

    const one: One<Card> = { card: "hyundai" };
    ```

  - ExcludeOne<T>

    ```ts
    type ExcludeOne<T> = {
      [P in keyof T]: Partial<Record<Exclude<keyof T, P>, undefined>>;
    }[keyof T];
    ```

    - `[P in keyof T]`에서 T는 객체로 가정하기 때문에 P는 T 객체의 키값을 말합니다.
    - `Exclude<keyof T, P>`는 T 객체가 가진 키값에서 P타입과 일치하는 키값을 제외합니다. 이 타입을 A로 가정하겠습니다.
    - `Record<A, undefined>`는 키로 A타입을, 값으로 undefined 타입을 갖는 레코드 타입입니다. 즉, 전달받은 객체 타입을 모두 `{ [key]: undefined }` 형태로 만들어줍니다. 이 타입을 B로 가정하겠습니다.
    - `Partial<B>`는 B타입을 옵셔널로 만들어줍니다. 따라서 `{ [key]?: undefined }`와 같습니다.
    - 최종적으로 `[P in keyof T]`로 매핑된 타입에서 동일한 객체의 키값인 `[keyof T]`로 접근하기 때문에 위 타입이 반환됩니다.

  - 결론적으로 얻고자 하는 타입은 속성 하나와 나머지는 `옵셔널 + undefined`인 타입이기 때문에 앞의 속성을 활용해서 `PickOne`타입을 표현할 수 있습니다.
  - PickOne<T>

    ```ts
    type PickOne<T> = One<T> & ExcludeOne<T>;
    ```

    - `One<T> & ExcludeOne<T>`는 `[P in keyof T]`를 공통으로 갖기 때문에 아래 코드와 같이 교차됩니다.

    ```ts
    [P in keyof T]: Record<P, T[P]> & Partial<Record<Exclude<keyof T, P>, undefined>>
    ```

    - 해당 타입을 해석하면 전달된 T타입의 1개의 키는 값을 가지고 있으며, 나머지 키는 옵셔널한 undefined값을 가진 객체를 의미합니다.

    ```ts
    type Card = { card: string };
    type Account = { account: string };

    const pickOne1: PickOne<Card & Account> = { card: "hyundai" }; // (O)
    const pickOne2: PickOne<Card & Account> = { account: "hana" }; // (O)
    const pickOne3: PickOne<Card & Account> = {
      card: "hyundai",
      account: undefined,
    }; // (O)
    const pickOne4: PickOne<Card & Account> = {
      card: undefined,
      account: "hana",
    }; // (O)
    const pickOne5: PickOne<Card & Account> = {
      card: "hyundai",
      account: "hana",
    }; // (X)
    ```

#### PickOne 타입 적용하기

- 이제 `PickOne`을 활용해서 앞의 코드를 수정해보면 `withdraw({ card: "hyundai", account: "hana" })`를 활용할 때 타입 에러가 발생하는 것을 확인할 수 있습니다.

  ```ts
  type Card = { card: string };

  type Account = {
    account: string
  };

  type CardOrAccount = PickOne<Card & Account>;

  function withdraw (type: CardOrAccount) {
    ...
  }

  withdraw({ card: "hyundai", account: "hana" }); // 에러 발생
  ```

  - 지금까지 `PickOne`타입을 구현하기 위해 단계별로 타입을 구현하는 예시를 살펴보았습니다.
  - 유틸리티 타입으로만 원하는 타입을 추출하기 어려울 때 커스텀 유틸리티 타입을 구현합니다.
  - 하지만 한 번에 바로 커스텀 유틸리티 타입 함수를 작성하기란 쉬운일이 아닙니다.
  - 앞에서 본 예시에서처럼 커스텀 유틸리티 타입을 구현할 때는 정확히 어떤 타입을 구현해야 하는지를 파악하고, 필요한 타입을 작은 단위로 쪼개어 생각하여 단계적으로 구현하는것이 좋습니다.

### NonNullable 타입 검사 함수를 사용하여 간편하게 타입 가드하기

- null을 가질 수 있는 nullable한 값의 null처리는 자주 사용되는 타입 가드 패턴중 하나입니다.
- 일반적으로 if문을 사용해서 null처리 타입 가드를 적용하지만, `is`키워드와 `NonNullable`타입으로 타입 검사를 위한 유틸 함수를 만들어서 사용할 수도 있습니다.

#### NonNullable 타입이란

- 타입스크립트에서 제공하는 유틸리티 타입으로 제네릭으로 받는 타입 T가 null또는 undefined일때 never 또는 T를 반환하는 타입입니다.
- `NonNullable`을 사용하면 null이나 undefined가 아닌 경우를 제외할 수 있습니다.

  ```ts
  type NonNullable<T> = T extends null | undefined ? never : T;
  ```

  > 최근 기능구현에서 우연히 써봤던것 같습니다.
  > zod를 통한 유효성검사에서 nullable이 아닌경우 default value를 자동으로 추출해주는 유틸리티를 구성하는 과정에서 사용했던거같아요.
  > 조만간 npm 배포해볼 예정입니다 :)

#### null, undefined를 검사해주는 NonNullable 함수

- `NonNullable` 유틸리티 타입을 사용하여 null또는 undefined를 검사해주는 타입 가드 함수를 만들어 사용할 수 있습니다.

  ```ts
  function NonNullable<T>(value: T): value is NonNullable<T> {
    return value !== null && value !== undefined;
  }
  ```

#### Promise.all을 사용할 때 NonNullable 적용하기

- `Promise.all`을 사용할 때 `NonNullable`를 적용해보겠습니다.
- 여러 상품의 광고를 조회할 때 하나의 API에서 에러가 발생한다고 해서 전체 광고가 보이지 않으면 안됩니다. 따라서 `try-catch`문을 사용하여 에러가 발생할 때는 null을 반환하고 있습니다.

  ```ts
  class AdCampaignAPI {
    static async operating(shopNo: number): Promise<AdCampaign[]> {
      try {
        return await fetch(`/ad/shopNumber=${shopNo}`);
      } catch (error) {
        return null;
      }
    }
  }

  // 사용처
  const shopList = [
    { shopNo: 100, category: "chicken" },
    { shopNo: 101, category: "pizza" },
    { shopNo: 102, category: "noodle" },
  ];

  const shopAdCampaignList = await Promise.all(
    shopList.map((shop) => AdCampaignAPI.operating(shop.shopNo))
  );
  ```

  - 이때 `AdCampaignAPI.operating`함수에서 null을 반환할 수 있기 때문에 shopAdCampaignList의 타입은 `Array<AdCampaign[] | null>`로 추론됩니다.
  - shopAdCampaignList를 `NomNullable` 함수로 필터링 처리하면 원하는 타입인 `Array<AdCampaign[]>`로 추론할 수 있게 됩니다.

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

## 05.4 불변 객체 타입으로 활용하기

#### Atom 컴포넌트에서 theme style 객체 활용하기

- `Atom`단위의 작은 컴포넌트(Button, Header, Input 등)는 폰트 크기, 폰트 색상, 배경 색상 등 다양한 환경에서 유연하게 사용될 수 있도록 구현되어야 하는데, Props로 넘겨받는 경우에는 사용자가 모든 색상 값을 인지하여야 하고, 변경 사항이 생기면 사이드이펙트의 수정이 번거로워지는 어려움이 생깁니다.
- 이런 문제를 해결하기 위해 대부분의 프로젝트에서는 theme 객체를 두고 관리하게 됩니다.

  - 컴포넌트의 Props의 color, fontSize 값의 타입을 정의할 때는 아래 예시처럼 설정할 수도 있습니다.

  ```ts
  interface Props {
    fontSize?: string;
    backgroundColor?: string;
    color?: string;
    onClick: (
      event: React.MouseEvent<HTMLButtonElement>
    ) => void | Promise<void>;
  }

  const Button: FC<Props> = ({
    fontSize,
    backgroundColor,
    color,
    children,
  }) => {
    return (
      <ButtonWrap
        fontSize={fontSize}
        backgroundColor={backgroundColor}
        color={color}
      >
        {children}
      </ButtonWrap>
    );
  };

  const ButtonWrap = styled.button<Omit<Props, "onClick">>`
    color: ${({ color }) => theme.color[color ?? "default"]};
    background-color: ${({ backgroundColor }) =>
      theme.bgColor[backgroundColor ?? "default"]};
    font-size: ${({ fontSize }) => theme.fontSize[fontSize ?? "default"]};
  `;
  ```

  - 위 예시 코드에서는 아직 props로 color, bgColor를 넘겨줄 때 키값이 자동완성되지 않으며, 잘못된 값을 넣어도 에러가 발생하지 않는다.
  - 이러한 문제는 theme 객체로 타입을 구체화해서 해결할 수 있습니다.
  - `keyof typeof`연산자로 theme 객체로 타입을 구체화하는 과정을 살펴보겠습니다.

#### 타입스크립트 keyof 연산자로 객체의 키값을 타입으로 추출하기

```ts
interface ColorType {
  red: string;
  green: string;
  blue: string;
}

type ColorKeyType = keyof ColorType; // 'red' | ‘green' | ‘blue'
```

#### 타입스크립트 typeof 연산자로 값을 타입으로 다루기

- 객체의 키값을 타입으로 다루려면 값 객체를 타입으로 변환해야 합니다.
- 이때 `typeof`연산자를 활용할 수 있습니다.
- 자바스크립트에서는 `typeof`가 타입을 추출하기 위한 연산자로 사용된다면, 타입스크립트에서는 `typeof`가 변수 혹은 속성의 타입을 추론하는 역할을 합니다.
- 타입스크립트의 `typeof`연산자는 단독으로 사용되기보다 주로 `ReturnType`같이 유틸리티 타입이나 `keyof` 연산자와 같이 타입을 받는 연산자와 함께 사용됩니다.
- `typeof`로 color 객체의 타입을 추론한 결과는 아래와 같습니다.

  ```ts
  const colors = {
    red: "#F45452",
    green: "#0C952A",
    blue: "#1A7CFF",
  };

  type ColorsType = typeof colors;
  /**
  {
    red: string;
    green: string;
    blue: string;
  }
  */
  ```

#### 객체의 타입을 활용해서 컴포넌트 구현하기

- `keyof typeof`연산자를 사용해서 theme 객체 타입을 구체화하고, string으로 타입을 설정했던 Button컴포넌트를 개선해보겠습니다.

  ```ts
  import React, { FC } from "react";
  import styled from "styled-components";

  const colors = {
    black: "#000000",
    gray: "#222222",
    white: "#FFFFFF",
    mint: "#2AC1BC",
  };

  const theme = {
    colors: {
      default: colors.gray,
      ...colors,
    },
    backgroundColors: {
      default: colors.white,
      gray: colors.gray,
      mint: colors.mint,
      black: colors.black,
    },
    fontSize: {
      default: "16px",
      small: "14px",
      large: "18px",
    },
  };

  type ColorType = keyof typeof theme.colors;
  type BackgroundColorType = keyof typeof theme.backgroundColors;
  type FontSizeType = keyof typeof theme.fontSize;

  interface Props {
    color?: ColorType;
    backgroundColor?: BackgroundColorType;
    fontSize?: FontSizeType;
    children?: React.ReactNode;
    onClick: (
      event: React.MouseEvent<HTMLButtonElement>
    ) => void | Promise<void>;
  }

  const Button: FC<Props> = ({
    fontSize,
    backgroundColor,
    color,
    children,
  }) => {
    return (
      <ButtonWrap
        fontSize={fontSize}
        backgroundColor={backgroundColor}
        color={color}
      >
        {children}
      </ButtonWrap>
    );
  };

  const ButtonWrap = styled.button<Omit<Props, "onClick">>`
    color: ${({ color }) => theme.colors[color ?? "default"]};
    background-color: ${({ backgroundColor }) =>
      theme.backgroundColors[backgroundColor ?? "default"]};
    font-size: ${({ fontSize }) => theme.fontSize[fontSize ?? "default"]};
  `;
  ```

  - 이제 해당 컴포넌트를 사용할때 bg의 값만 받을 수 있게 되었고, 다른 값을 넣었을 때는 타입 오류가 발생하게 됩니다.

### 05.5 Record 원시 타입 키 개선하기

- 객체 선언 시 키가 어떤 값인지 명확하지 않다면 `Record`의 키를 string이나 number 같은 원시 타입으로 명시하곤 합니다.
- 이때 타입스크립트는 키가 유효하지 않더라도 타입상으로는 문제 없기 때문에 오류를 표시하지 않습니다.
- 이번에는 `Record`를 명시적으로 사용하는 방법에 대해서 알아보겠습니다.

#### 1. 무한한 키를 집합으로 가지는 Record

- 음식 분류를 키로 사용하는 음식 배열이 담긴 객체를 만들어보겠습니다.

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

  - 해당 예시에서 Category의 타입은 string입니다.
  - Category를 `Record`의 키로 사용하는 `foodByCategory` 객체는 무한한 키 집합을 가지게 됩니다.
  - 이때 `foodByCategory`객체에 없는 키값을 사용하더라도 타입스크립트는 오류를 표시하지 않습니다.

  ```ts
  foodByCategory["양식"]; // Food[]로 추론
  foodByCategory["양식"].map((food) => console.log(food.name)); // 오류가 발생하지 않는다
  ```

  - 그러나 런타임에서 undefined가 되어 오류가 발생하는 경우가 존재할 수 있습니다.

  ```ts
  foodByCategory["양식"].map((food) => console.log(food.name)); // Uncaught TypeError: Cannot read properties of undefined (reading ‘map’)
  ```

  - 이때 자바스크립트의 옵셔널 체이닝 등을 사용하여 런타임 에러를 방지할 수 있습니다.
  - 하지만 어떤 값이 undefined인지 매번 판단해야 한다는 번거로움이 생깁니다.
  - 또한 실수로 undefined일 수 있는 값을 인지하지 못하면 예상치 못한 런타임 에러가 발생할 수 있습니다.
  - 하지만!! 타입스크립트의 기능을 활용하여 개발 중에 유효하지 않은 키가 사용되었는지 또는 undefined일 수 있는 값이 있는지 등을 사전에 파악활 수 있습니다.

#### 2. 유닛 타입으로 변경하기

- 키가 유한한 집합이라면 유닛 타입(다른 타입으로 쪼개지지 않고 오직 하나의 정확한 값을 가지는 타입)을 사용할 수 있습니다.

```ts
type Category = "한식" | "일식";
interface Food {
  name: string;
  // ...
}
const foodByCategory: Record<Category, Food[]> = {
  한식: [{ name: "제육덮밥" }, { name: "뚝배기 불고기" }],
  일식: [{ name: "초밥" }, { name: "텐동" }],
};

// Property ‘양식’ does not exist on type ‘Record<Category, Food[]>’.foodByCategory["양식"];
```

- 저흰 이런식으로 활용하고 있습니다. 바로바로 수정하라는 에러가 표시되어서 좋은거같아요.

```ts
export const GRADE = {
  A: "A",
  B: "B",
  C: "C",
  D: "D",
  E: "E",
  F: "F",
} as const;

export type TGrade = (typeof GRADE)[keyof typeof GRADE];

export const GRADE_LABELS: Record<TGrade, string> = {
  [GRADE.A]: "A",
  [GRADE.B]: "B",
  [GRADE.C]: "C",
  [GRADE.D]: "D",
  [GRADE.E]: "E",
  [GRADE.F]: "F",
};
```

#### 3. Partial을 활용하여 정확한 타입 표현하기

- 키가 무한한 상황에서는 `Partial`을 사용하여 해당 값이 undefined일 수 있는 상태임을 표현할 수 있습니다.
- 객체 값이 undefined일 수 있는 경우에 `Partial`을 사용해서 `PartialRecord`타입을 선언하고 객체를 선언할 때 해당 타입을 활용할 수 있습니다.

```ts
type PartialRecord<K extends string, T> = Partial<Record<K, T>>;
type Category = string;

interface Food {
  name: string;
  // ...
}

const foodByCategory: PartialRecord<Category, Food[]> = {
  한식: [{ name: "제육덮밥" }, { name: "뚝배기 불고기" }],
  일식: [{ name: "초밥" }, { name: "텐동" }],
};

foodByCategory["양식"]; // Food[] 또는 undefined 타입으로 추론
foodByCategory["양식"].map((food) => console.log(food.name)); // Object is possibly 'undefined'
foodByCategory["양식"]?.map((food) => console.log(food.name)); // OK
```

- 타입스크립트는 `foodByCategory[key]`를 Food[] 또는 undefined로 추론하고, 개발자에게 이 값은 undefined일 수 있으니 해당 값에 대한 처리가 필요하다고 표시해줍니다.
- 옵셔널체이닝, 조건문 등 사전 조치로 인하여 런타임 에러를 방지할 수 있게 됩니다.
