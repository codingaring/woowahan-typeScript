> 이번 장에서는 리액트에서 제공하는 몇 가지 훅을 사용하여 상태 또는 사이드 이펙트를 다루는 방법을 소개합니다.
> 또한 상태 로직을 재사용할 수 있게 해주고, 컴포넌트의 복잡성을 낮춰주는 커스텀 훅에 대해 알아보겠습니다.

## 09.1 리액트 훅

- 리액트에 훅이 추가되기 이전에는 클래스 컴포넌트에서만 상태를 가질 수 있었습니다. 라뗀말이야~
  - 클래스 컴포넌트에서는 componentDidMount, componentDidUpdate와 같이 하나의 생명주기 함수에서만 상태 업데이트에 따른 로직을 실행시킬 수 있었습니다.
  - 간단한 형태의 컴포넌트에서는 문제가 되지 않았지만, 프로젝트 규모가 커지면서 상태를 스토어에 연결하거나 비슷한 로직을 가진 상태 업데이트 및 사이드 이펙트 처리가 불편해지는 문제가 발생하였습니다.
  - 또한 모든 상태를 하나의 함수 내에서 처리하다 보니 관심사가 뒤섞이게 되었고, 상태에 따른 테스트나 잘못 발생한 사이드 이펙트의 디버깅이 어려워졌습니다.

```js
componentDidMount() {
  this.props.updateCurrentPage(routeName);
  this.didFocusSubscription = this.props.navigation.addListener('focus', () => {/*
add focus handler to navigation */});
  this.didBlurSubscription = this.props.navigation.addListener('blur', () => {/* add
blur handler to navigation */});
}

componentWillUnmount() {
  if (this.didFocusSubscription != null) {
    this.didFocusSubscription();
  }
  if (this.didBlurSubscription != null) {
    this.didBlurSubscription();
  }
  if (this._screenCloseTimer != null) {
    clearTimeout(this._screenCloseTimer);
    this._screenCloseTimer = null;
  }
}

componentDidUpdate(prevProps) {
  if (this.props.currentPage != routeName) return;

  if (this.props.errorResponse != prevProps.errorResponse) {/* handle error response
*/}
  else if (this.props.logoutResponse != prevProps.logoutResponse) {/* handle logout
response */}
  else if (this.props.navigateByType != prevProps.navigateByType) {/* handle
navigateByType change */}

  // Handle other prop changes here
}
```

- componentWillUnmount 에서는 componentDidMount 에서 정의한 컴포넌트가 DOM에서 해제될 때 실행되어야 할 여러 사이드 이펙트 함수를 호출합니다.
  - 만약 componentWillUnmount에서 실행되어야 할 사이드 이펙트가 하나 빠졌다면, componentDidMount와 비교해가며 어떤 함수가 빠졌는지 찾아야 할 것입니다..... 🤢
  - 또한 props 변경에 대한 디버깅을 수행하려면 componentDidUpdate 에서 내가 원하는 props 변경 조건문이 나올 때까지 코드를 찾아보아야 합니다..... 🤢
- 리액트 훅이 도입되면서 함수 컴포넌트에서도 클래스 컴포넌트와 같이 컴포넌트의 생명주기에 맞추어 로직을 실행할 수 있게 되었습니다.

### 1. useState

- 리액트 함수 컴포넌트에서 상태를 과니하기 위해 useState 훅을 사용할 수 있습니다.

  - useState의 타입 정의는 다음과 같습니다.

  ```ts
  function useState<S>(
    initialState: S | (() => S)
  ): [S, Dispatch<SetStateAction<S>>];

  type Dispatch<A> = (value: A) => void;
  type SetStateAction<S> = S | ((prevState: S) => S);
  ```

  - useState가 반환하는 튜플을 살펴보겠습니다.
    - 튜플의 첫 번째 요소는 제네릭으로 지정한 `S`타입이며, 두 번째 요소는 상태를 업데이트할 수 있는 `Dispatch`타입의 함수입니다.
    - Dispatch함수의 제네릭으로 지정한 SetStateAction에는 useState로 관리할 상태 타입인 `S`또는 이전 상태 값을 받아 새로운 상태를 반환하는 함수인 `(prevState: S) => S`가 들어갈 수 있습니다.

- useState에 타입스크립트를 적용하면 다음과 같은 시너지를 얻을 수 있습니다.

  ```ts
  import { useState } from "react";

  const MemberList = () => {
    const [memberList, setMemberList] = useState([
      {
        name: "KingBaedal",
        age: 10,
      },
      {
        name: "MayBaedal",
        age: 9,
      },
    ]);

    // 🚨 addMember 함수를 호출하면 sumAge는 NaN이 된다
    const sumAge = memberList.reduce((sum, member) => sum + member.age, 0);

    const addMember = () => {
      setMemberList([
        ...memberList,
        {
          name: "DokgoBaedal",
          agee: 11,
        },
      ]);
    };
  };
  ```

  - 잘못된 속성이 포함된 객체가 추가되는 경우, 변수가 NaN이 되는 예상치 못한 사이드 이펙트가 발생할 수 있습니다.

    ```tsx
    import { useState } from "react";

    interface Member {
      name: string;
      age: number;
    }

    const MemberList = () => {
      const [memberList, setMemberList] = useState<Member[]>([]);

      // member의 타입이 Member 타입으로 보장된다
      const sumAge = memberList.reduce((sum, member) => sum + member.age, 0);

      const addMember = () => {
      // 🚨 Error: Type ‘Member | { name: string; agee: number; }’
      // is not assignable to type ‘Member’
        setMemberList([
          ...memberList,
          {
            name: "DokgoBaedal",
            agee: 11,
          },
        ]);
      };

      return (
        // ...
      );
    };
    ```

    - useState에 타입스크립트를 적용용하면 짜잔~ Dispatch를 호출하는 부분에서 추가하려는 개 객체의 타입을 확인하여 컴파일타임에 타입 에러를 발견할 수 있게됩니다람쥐

### 2. 의존성 배열을 사용하는 훅

#### useEffect와 useLayoutEffect

- 렌더링 이후 리액트 함수 컴포넌트에 어떤 일을 수행하야 하는지 알려주기 위해 useEffect 훅을 사용할 수 있습니다.
- useEffect의 타입 정의는 다음과 같습니다.

  ```ts
  function useEffect(effect: EffectCallback, deps?: DependencyList): void;
  type DependencyList = ReadonlyArray<any>;
  type EffectCallback = () => void | Destructor;
  ```

  - useEffect의 첫 번째 인자이자 effect의 타입인 EffectCallback은 Destructor를 반환하거나 아무것도 반환하지 않습니다.
    - Promise 타입은 반환하지 않으므로 useEffect의 콜백 함수에는 비동기 함수가 들어갈 수 없습니다.
    - useEffect에서 비동기 함수를 호출할 수 있다면 경쟁 상태(Race Condition)를 불어일으킬 수 있기 때문입니다.

  > ##### 경쟁 상태 (Race Condition) 😮
  >
  > 멀티스레딩 환경에서 동시에 여러 프로세스나 스레드가 공유된 자원에 접근하려고 할 때 발생할 수 있는 문제입니다.
  > 이러한 상황에서 실행 순서나 타이밍을 예측할 수 없게 되어 프로그램 동작이 원하지 않는 방향으로 흐를 수 있습니다.

  - 두번째 인자인 deps는 옵셔널하게 제공되며, effect가 수행되기 위한 조건을 나열합니다.

    - useEffect는 deps가 변경되었는지를 얕은 비교로만 판단하기 때문에, 실제 객체 값이 바뀌지 않았더라도 객체의 참조 값이 변경되면 콜백 함수가 실행됩니다.
    - 앞의 예시처럼(작성하지 않음) 부모에서 받은 인자를 직접 deps로 작성한 경우, 원치 않는 렌더링이 반복될 수 있습니다.
    - 이를 방지하기 위해서는 다음과 같이 실제로 사용하는 값을 useEffect의 deps에서 사용해야 합니다.
      > ##### 얕은 비교 (shallow compare) 😮
      >
      > 객체나 배열과 같은 복합 데이터 타입의 값을 비교할 때 내부의 각 요소나 속성을 재귀적으로 비교하지 않고, 해당 값들의 참조나 기본 타입인 값만 간단하게 비교하는 것을 말합니다.

  - useEffect는 살펴본 것처럼 Destructor를 반환하는데 이것은 컴포넌트가 마운트 해제될때 실행하는 함수입니다.
    - 하지만 이 말은 일부만 옳다고 설명드릴 수 있는데, deps가 빈 배열이라면 useEffect의 콜백 함수는 컴포넌트가 처음 렌더링될 때만 실행되며, 이때 Destructor(클린업 함수라고도 합니다)는 컴포넌트가 마운트 해제될 때 실행됩니다.
    - 그러나 deps 배열이 존재한다면, 배열의 값이 변경될 때마다 Destructor가 실행됩니다.
      > ##### 클린업(Clean up) 함수 😮
      >
      > useEffect나 useLayoutEffect와 같은 리액트 훅에서 사용되며, 컴포넌트가 해제되기 전에 정리(clean up) 작업을 수행하기 위한 함수를 의미합니다.

- useEffect와 비슷한 역할을 하는 훅으로 useLayoutEffect가 있습니다.

  - 이 훅의 타입 정의 역시 useEffect와 동일하며 하는 역할의 차이만 있습니다.

  ```ts
  type DependencyList = ReadonlyArray<any>;

  function useLayoutEffect(effect: EffectCallback, deps?: DependencyList): void;
  ```

  - useEffect는 앞서 살펴본 componentDidUpdate와 같은 기존 생명주기 함수와는 다르게, 레이아웃 배치와 화면 렌더링이 모두 완료된 후에 실행됩니다.

    ```tsx
    const [name, setName] = useState("");

    useEffect(() => {
      // 매우 긴 시간이 흐른 뒤 아래의 setName()을 실행한다고 생각하자
      setName("배달이");
    }, []);

    return <div>{`안녕하세요, ${name}님!`}</div>;
    ```

    - 이와 같은 코드를 실행하면 처음에는 name이 빈칸으로 렌더링된 후, 다시 name값이 변경되면서 렌더링될 것입니다.
    - 만약 setName이 오랜 시간이 걸린 후에 실행된다면 사용자는 빈 이름을 오랫동안 보고 있어야 할 것입니다.

- useLayoutEffect는 이런 상황에서 사용할 수 있습니다.
  - useLayoutEffect를 사용하면 화면에 해당 컴포넌트가 그려지기 전에 콜백 함수를 실행하기 때문에 첫 번째 렌더링 때 빈 이름이 뜨는 경우를 방지할 수 있습니다.

##### useLayoutEffect를 비동기 데이터 fetch에서 사용하면 어떨까 싶어서 조금 디깅해봤습니다.

- useLayoutEffect는 DOM 변경 완료 후, 브라우저가 화면을 그리기 이전(paint)전에 실행됩니다.
- 하지만 fetch는 비동기이므로 useLayoutEffect 내부에서도 즉시 완료되지 않습니다. 따라서 첫 렌더링 시에는 여전히 빈 값이 표기될 수 밖에 없습니다.

  - 따라서 비동기 데이터 fetch에는 useLayoutEffect가 적합하지 않습니다.

- useLayoutEffect를 사용하는 실제 활용사례를 나열해보겠습니다.

  - 툴팁 혹은 팝오버 위치 계산
  - 스크롤 위치 복원
  - FLIP 애니메이션
  - 포커스 관리
  - DOM 크기 측정 후 조건부 렌더링
  - 등등

  핵심 판단 기준
  useLayoutEffect를 사용해야 하는 경우:

  - ✅ DOM 측정이 필요하고
  - ✅ 그 결과로 즉시 다른 DOM을 업데이트해야 하며
  - ✅ 사용자가 중간 상태를 보면 안 되는 경우

#### useMemo와 useCallback

- useMemo와 useCallback 모두 이전에 생성된 값 또는 함수를 기억하며, 동일한 값과 함수를 반복해서 생성하지 않도록 해주는 훅입니다.
- 어떤 값을 계산하는데 오랜 시간이 걸릴 때나 렌더링이 자주 발생하는 form에서 useMemo나 useCallback을 유용하게 사용할 수 있습니다.

  ```ts
  type DependencyList = ReadonlyArray<any>;

  function useMemo<T>(factory: () => T, deps: DependencyList | undefined): T;
  function useCallback<T extends (...args: any[]) => any>(
    callback: T,
    deps: DependencyList
  ): T;
  ```

  - 두 훅 모두 제네릭을 지원하기 위해 T 타입을 선언해주며 useCallback은 함수를 저장하기 위해 제네릭의 기본 타입을 지정하고 있습니다.
  - 두 훅 모두 앝은 비교를 수행하기 때문에 deps 배열이 변경되지 않았는데도 다시 계산되지 않도록 주의해야 합니다.
  - 또한 불필요한 곳에 사용하지 않는것도 중요합니다. 모든 값과 함수를 useMemo와 useCallback을 사용해서 과도하게 메모이제이션하면 컴포넌트의 성능 향상이 보장되지 않을 수 있습니다.

### 3. useRef

- 리액트로 이루어진 앱에서 input 요소에 포커스를 설정하거나 특정 컴포넌트의 위치로 스크롤을 하는 등 DOM을 직접 선택해야 하는 경우가 발생할 수 있습니다. 이럴떄에는 useRef를 사용하면 됩니다 깐따삐야~

  ```tsx
  import { useRef } from "react";

  const MyComponent = () => {
    const ref = useRef<HTMLInputElement>(null);

    const onClick = () => {
      ref.current?.focus();
    };

    return (
      <>
        <button onClick={onClick}>ref에 포커스!</button>
        <input ref={ref} />
      </>
    );
  };

  export default MyComponent;
  ```

  - useRef의 제네릭에 `HTMLInputElement | null`이 아닌 `HTMLInputElement`만 넣어주고 있음에도 어떻게 초기 설정값에 null이 들어갈 수 있으며, ref에 input요소를 저장할 수 있을까요?
    - useRef는 세 종류의 타입 정의를 가지고 있습니다.
    - useRef에 넣어주는 인자 타입에 따라 반환되는 타입이 달라집니다.

  ```ts
  function useRef<T>(initialValue: T): MutableRefObject<T>;
  function useRef<T>(initialValue: T | null): RefObject<T>;
  function useRef<T = undefined>(): MutableRefObject<T | undefined>;

  interface MutableRefObject<T> {
    current: T;
  }

  interface RefObject<T> {
    readonly current: T | null;
  }
  ```

  - useRef는 `MutableRefObject` 또는 `RefObject`를 반환합니다.
    - `MutableRefObject`의 current는 값을 변경할 수 있습니다.
  - 만약 null을 허용하기 위해 useRef의 제네릭에 `HTMLInputElement | null`타입을 넣어주었다면, 해당 useRef는 첫 번째 타입 정의(`function useRef<T>(initialValue: T): MutableRefObject<T>;`)를 따를 것입니다.
    - 이때 `MutableRefObject`의 current는 변경할 수 있는 값이 되어 ref.current의 값이 바뀌는 사이드 이펙트가 발생할 수 있습니다.
  - 반면 `RefObject`의 current는 readonly로 값을 변경할 수 없습니다.
    - 앞의 예시에서는 useRef의 제네릭으로 `HTMLInputElement`을 넣고, 인자에 null을 넣어 useRef의 두 번째 타입 정의(`function useRef<T>(initialValue: T | null): RefObject<T>;`)를 따르도록 합니다.
      - 이렇게 되면 `RefObject`를 반환하여 ref.current 값을 임의로 변경할 수 없게 됩니다.

#### 이해가 잘 안되어서 보충설명 해볼게요

- 왜 useRef<HTMLInputElement>(null)이 안전한가?

  ```tsx
  // ✅ 권장: RefObject 반환 (readonly)
  const inputRef = useRef<HTMLInputElement>(null);
  // 타입: RefObject<HTMLInputElement>
  // inputRef.current는 readonly라서 변경 불가 ✅

  // ❌ 비권장: MutableRefObject 반환 (변경 가능)
  const inputRef = useRef<HTMLInputElement | null>(null);
  // 타입: MutableRefObject<HTMLInputElement | null>
  // inputRef.current = ... 가능 (위험!) ❌
  ```

  - 왜 이렇게 설계되었나?
    - DOM 참조는 React가 관리해야 함
    - `<input ref={inputRef} />`에서 React가 자동으로 DOM을 할당
    - 개발자가 inputRef.current = ...로 덮어쓰면 버그 발생
  - 타입 안전성
    - RefObject의 readonly current로 실수 방지
    - TypeScript가 컴파일 시점에 에러 체크

- 언제 뭘 쓰는게 좋을까요?

  ```ts
  // 1️⃣ DOM 참조 → RefObject
  const divRef = useRef<HTMLDivElement>(null);

  // 2️⃣ 변수 저장 → MutableRefObject
  const countRef = useRef<number>(0);

  // 3️⃣ 나중에 할당 → MutableRefObject<T | undefined>
  const dataRef = useRef<string>();
  ```

#### 자식 컴포넌트에 ref 전달하기

> 최근 forwardRef 개선과 상충하는 내용으로 해당 내용은 패스했습니다.

#### useImperativeHandle

- useImperativeHandle은 ForwardRefRenderFunction과 함께 쓸 수 있는 훅 입니다.

  - 해당 훅을 활용하면 부모 컴포넌트에서 ref를 통해 자식 컴포넌트에서 정의한 커스터마이징된 메서드를 호출할 수 있게 됩니다.
  - 이에 따라 자식 컴포넌트는 내부 상태나 로직을 관리하면서 부모 컴포넌트와의 결합도도 낮출 수 있게 됩니다.

  ```ts
  // <form> 태그의 submit 함수만 따로 뽑아와서 정의한다
  type CreateFormHandle = Pick<HTMLFormElement, "submit">;

  type CreateFormProps = {
    defaultValues?: CreateFormValue;
  };

  const JobCreateForm: React.ForwardRefRenderFunction<
    CreateFormHandle,
    CreateFormProps
  > = (props, ref) => {
    // useImperativeHandle Hook을 사용해서 submit 함수를 커스터마이징한다
    useImperativeHandle(ref, () => ({
      submit: () => {
        /* submit 작업을 진행 */
      },
    }));

    // ...
  };
  ```

  - 위 예시처럼 자식 컴포넌트에서는 ref와 정의된 CreateFormHandle을 통해 부모 컴포넌트에서 호출할 수 있는 함수를 생성하고, 부모 컴포넌트에서는 다음처럼 `current.submit()`를 사용하여 자식 컴포넌트의 특정 메서드를 실행할 수 있게 됩니다.

    ```ts
    const CreatePage: React.FC = () => {
      // `CreateFormHandle` 형태를 가진 자식의 ref를 불러온다
      const refForm = useRef<CreateFormHandle>(null);

      const handleSubmitButtonClick = () => {
        // 불러온 ref의 타입에 따라 자식 컴포넌트에서 정의한 함수에 접근할 수 있다
        refForm.current?.submit();
      };

      // ...
    };
    ```

#### useRef의 여러 가지 특성

- useRef는 자식 컴포넌트를 저장하는 변수로 활용할 수 있을 뿐 아니라 다른 방식으로도 유용하게 사용할 수 있습니다.

  - useRef로 관리되는 변수는 값이 바뀌어도 컴포넌트의 리렌더링이 발생하지 않습니다. 이런 특성을 활용하면 상태가 변경되더라도 불필요한 리렌더링을 피할 수 있습니다.
  - 리액트 컴포넌트의 상태는 상태 변경 함수를 호출하고 렌더링된 이후에 업데이트된 이후에 업데이트된 상태를 조회할 수 있습니다.(useState를 의미하는듯 하네요) 반면 useRef로 관리되는 변수는 값을 설정한 후 즉시 조회할 수 있습니다.

    ```tsx
    type BannerProps = {
      autoplay: boolean;
    };

    const Banner: React.FC<BannerProps> = ({ autoplay }) => {
      const isAutoPlayPause = useRef(false);

      if (autoplay) {
        // keepAutoPlay 같이 isAutoPlay가 변하자마자 사용해야 할 때 쓸 수 있다
        const keepAutoPlay = !touchPoints[0] && !isAutoPlayPause.current;

        // ...
      }
      return (
        <>
          {autoplay && (
            <button
              aria-label="자동 재생 일시 정지"
              // isAutoPlayPause는 사실 렌더링에는 영향을 미치지 않고 로직에만 영향을 주므로 상태로 사용해서 불필요한 렌더링을 유발할 필요가 없다
              onClick={() => {
                isAutoPlayPause.current = true;
              }}
            />
          )}
        </>
      );
    };

    const Label: React.FC<LabelProps> = ({ value }) => {
      useEffect(() => {
        // value.name과 value.id를 사용해서 작업한다
      }, [value]);

      // ...
    };
    ```

    - isAutoPlayPause는 현재 자동 재생이 일시 정지되었는지 확인하는 ref입니다.
      - 이 변수는 렌더링에 영향을 미치지 않으며, 값이 변경되더라도 다시 렌더링을 기다릴 필요 없이 사용할 수 있어야 합니다.
      - 위 예시처럼 isAutoPlayPause는.current에 null아 아닌 값을 할당해서 마치 변수처럼 활용할 수도 있습니다.

  > ##### 훅의 규칙 😮
  >
  > 리액트 훅을 안전하게 사용하기 위해서는 다음 2가지 규칙을 준수해야 합니다.
  >
  > 1. 훅은 항상 최상위 레벨에서 호춣되어야 합니다. 다시 말해 조건문, 반복문, 중첩 함수, 클래스 등의 내부에서는 훅을 호출하지 않아야 합니다.
  >    반환문으로 함수 컴포넌트가 종료되거나, 조건문 또는 변수에 따라 반복문 등으로 훅의 호출 여부(호출되거나 호출되지 않거나)가 결정되어서는 안됩니다.
  >    이렇게 해야 useState 또는 useEffect가 여러 번 호출되더라도 훅의 상태를 올바르게 유지할 수 있습니다.
  > 2. 훅은 항상 컴포넌트나 커스텀 훅 등의 리액트 컴포넌트 내에서만 호출되어야 합니다.
  >
  > 이러한 규칙이 필요한 이유는 리액트에서 훅은 호출 순서에 의존하기 때문입니다.
  > 모든 컴포넌트 렌더링에서 훅의 순서가 항상 동일하게 유지되어야 하며, 이를 통해 항상 동일한 컴포넌트 렌더링이 보장되어야 합니다.
