# 리액트 훅

- 리액트에 훅이 추가(리액트 16.8 버전)되기 이전에는 클래스 컴포넌트에서만 상태를 가질 수 있었다. (충격)
- 클래스 컴포넌트에서는 componentDidMount, componentDidUpdated와 같이 하나의 생명주기 함수에서만 상태 업데이트에 따른 로직을 실행시킬 수 있었다.

#### ❗ 문제점

- 상태를 스토어에 연결하거나, 비슷한 로직을 가진 상태 업데이트 및 사이드 이펙트 처리가 불편
- 모든 상태를 하나의 함수 내에서 처리하다보니 관심사가 뒤섞이게 되고, 상태에 따른 테스트나 잘못 발생한 사이드 이펙트의 디버깅이 어려워졌다.

```ts
    componentDidMount(){
        this.props.updateCurrentPage(routeName);
        this.didFocusSubscription = this.props.navigation.addListener('focus', () => {
            // ...
        })
    this.didBlurSubscription = this.props.navigation.addListener("blur", () => {
        // ...
    })
    }

    componentWillUnmount(){
        if(//...)
    }
```

- 이렇게 디버깅도, 상태관리도 불편했던 과거를 지나 훅이 도입되면서 광명을 찾았다고 한다.
- 라고 할 수 없지요.

### 훅의 등장으로 변경된 지점

- 함수 컴포넌트에서도 클래스 컴포넌트와 같이 컴포넌트의 생명주기에 맞춰 로직을 실행할 수 있게 되었다.
- 이에 따라 비즈니스 로직을 재사용하거나, 작은 단위로 코드를 분할하여 테스트하는게 용이해졌다.
- 사이드 이펙트와 상태를 관심사에 맞게 분리하여 구성할 수 있게 되었다.

## useState

```ts
function useState<S>(
  initialState: S | (() => S)
): [S, Dispatch<SetStateAction<S>>];

type Dispatch<A> = (value: A) => void;
type SetStateAction<S> = S | ((prevState: S) => S);
```

- useState에 타입스크립트를 활용하면 잘못된 타입이 들어가는 것을 미연에 방지할 수 있다는 내용

## 의존성 배열을 사용하는 훅

### useEffect와 useLayoutEffect

- 렌더링 이후 리액트 함수 컴포넌트에 어떤 일을 수행해야 하는지 알려주기 위해 useEffect 훅을 활용할 수 있다.
- useEffect의 타입 정의

```ts
function useEffect(effect: EffectCallback, deps?: DependencyList): void;
type DependencyList = ReadonlyArray<any>;
type EffectCallback = () => void | Destructor;
```

- useEffect의 첫 번째 인자이자, effect의 타입인 EffectCallback은 Destructor를 반환하거나 아무것도 반환하지 않는 함수이다.
- Promise 타입은 반환하지 않으므로 useEffect의 콜백 함수에는 비동기 함수가 들어갈 수 없다. useEffect에서 비동기 함수를 호출할 수 있다면 경쟁 상태(Race Condition)를 불러일으킬 수 있기 때문이다.

> **Race Condition**
> 멀티스레딩 환경에서 동시에 여러 프로세스나 스레드가 공유된 자원에 접근하려고 할 때 발생할 수 있는 문제다. 이러한 상황에서 실행 순서나 타이밍을 예측할 수 없게 되어 프로그램 동작이 원하지 않는 방향으로 흐를 수 있다.

- 두 번째 인자인 deps는 옵셔널하게 제공되며 useEffect가 수행되기 위한 조건을 나열한다. 예를 들어 deps 배열의 원소가 변경되면 실행한다는 식으로 사용한다. 다만, deps의 원소로 숫자나 문자열 같은 타입스크립트 기본 자료형이 아닌 객체나 배열을 넣을 때는 주의해야 한다.

```tsx
type SomeObject = {
  name: string;
  id: string;
};

type LabelProps {
    value: SomObject;
}

const Label : React.FC<LabelProps> = ({value}) => {
    useEffect(()=>{
        // value.name 과 value.id를 사용해서 하는 작업
    },[value])
}
```

- useEffect는 deps가 변경되었는지를 얕은 비교로만 판단하기 때문에, 실제 객체 값이 바뀌지 않았더라도 객체의 참조값이 변경되면 콜백 함수가 실행된다. 앞의 예시처럼 부모에서 받은 인자를 직접 deps로 작성한 경우, 원치 않는 렌더링이 반복될 수 있다.
- 이를 방지하기 위해서는 실제로 사용하는 값을 useEffect의 deps에서 사용해야 한다.

```ts
const { name, id } = value;
useEffect(() => {
  // ...
}, [name, id]);
```

=> 이런 특징은 `useMemo` 나 `useCallback` 과 같은 다른 훅에서도 동일하게 적용된다.

- useEffect는 Destructor를 반환하는데, 이것은 컴포넌트가 마운트 해제될 때 실행하는 함수이다.
  - deps가 빈 배열이라면 useEffect의 콜백 함수는 컴포넌트가 처음 렌더링 될 때만 실행되며, 이때의 Destructor(클린업 함수)는 컴포넌트가 마운트 해제될 때 실행된다.
  - 그러나 deps 배열이 존재한다면 배열의 값이 변경될 때마다 Destructor가 실행된다.

#### useLayoutEffect

- useEffect와 비슷한 역할을 하지만, 레이아웃 배치와 화면 렌더링이 모두 완료된 후에 실행된다.
- useLayoutEffect를 사용하면 화면에 해당 컴포넌트가 그려지기 전에 콜백 함수를 실행하기 때문에 첫 번째 렌더링 때 빈 이름이 뜨는 경우를 방지할 수 있다.

### useMemo와 useCallback

- 두 훅 모두 이전에 생성된 값 또는 함수를 기억하며, 동일한 값과 함수를 반복해서 생성하지 않도록 해주는 훅이다.
- 어떤 값을 계산하는데 오랜 시간이 걸릴 때나 렌더링이 자주 발생하는 form에서 useMemo나 useCallback을 유용하게 사용할 수 있다.

```ts
type DependencyList = ReadonlyArray<any>;

function useMemo<T>(factory: () => T, deps: DependencyList | undefined): T;
function useCallback<T extends (...args: any[]) => any>(
  callback: T,
  deps: DependencyList
): T;
```

## useRef

- 리액트 애플리케이션에서 <input /> 요소에 포커스를 설정하거나 특정 컴포넌트의 위치로 스크롤을 하는 등 DOM을 직접 선택해야 하는 경우가 발생할 수 있다.

```tsx
const MyComponent = () => {
  const ref = useRef<HTMLInputElement>(null);

  const onClick = () => {
    ref.current?.focus();
  };

  return (
    <>
        <button onClick={onClick}>ref에 포커스!</button>
        <input ref={ref} />
    <>
  )
};
```

- useRef의 제네릭에 HTMLInputElement | null이 아닌 HTMLInputElement만 넣었는데 어떻게 초기값으로 null이 들어갈 수 있으며, ref에 input 요소를 저장할 수 있을까?
  - useRef는 세 종류의 타입 정의를 가지고 있고, useRef에 넣어주는 인자 타입에 따라 반환되는 타입이 달라진다.

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

- useRef는 MutableRefObject 또는 RefObject를 반환한다.
- MutableRefObject의 current는 값을 변경할 수 있다. 만약 null을 허용하기 위해 useRef의 제네릭에 HTMLInputElement | null 타입을 넣어주었다면, 해당 useRef는 첫 번째 타입 정의를 따를 것이다. 이때 MutableObject의 current는 변경할 수 있는 값이 되어 ref.current의 값이 바뀌는 사이드 이펙트가 발생할 수 있다.

- 반면, RefObject의 current는 readonly 로 값을 변경할 수 없다. 앞의 예시에서는 useRef의 제네릭으로 HTMLInputElement을 넣고, 인자에 null을 넣어 useRef의 두 번째 타입 정의를 따르게 된다. 이러면 RefObject를 반환하여 ref.current값을 임의로 변경할 수 없게 된다.

### 자식 컴포넌트에 ref 전달하기

`<button />`이나 `<input />`과 같은 기본 HTML 요소가 아닌, 리액트 컴포넌트에 ref를 전달할 수도 있다. 그러나 이때 ref를 일반적인 props를 전달하는 방식으로 전달하면 브라우저에서 경고 메시지를 띄운다.

- ref라는 속성의 이름은 리액트에서 DOM요소 접근 이라는 특수한 목적으로 사용되기 때문에 props를 넘겨주는 방식으로 전달할 수 없다.
- 리액트 컴포넌트에서 ref를 prop으로 전달하기 위해서는 forwardRef를 사용해야 한다.
  - ref가 아닌 inputRef 등 다른 이름을 사용한다면 forwardRef를 사용하지 않아도 된다.

```tsx
interface Props {
  name: string;
}

const MyInput = forwardRef<HTMLInputElement, Props>((props, ref) => {
  return (
    <div>
      <label>{props.name}</label>
      <input ref={ref} />
    </div>
  );
});
```

### useImperativeHandle

ForwardRefRenderFunction고 함께 쓸 수 있는 훅이다.

- 이 훅을 활용하면 부모 컴포넌트에서 ref를 통해 자식 컴포넌트에서 정의한 커스터마이징된 메서드를 호출할 수 있게 된다.

### useRef의 여러 가지 특성

- useRef로 관리되는 변수는 값이 바뀌어도 컴포넌트의 리렌더링이 발생하지 않는다. 이런 특성을 활용하면 상태가 변경되더라도 불필요한 리렌더링을 피할 수 있다.
- 리액트 컴포넌트의 상태는 상태 변경 함수를 호출하고 렌더링된 이후에 업데이트된 상태를 조회할 수 있다.
  - 반면 useRef로 관리되는 변수는 값을 설정한 후 즉시 조회할 수 있다.

## 💡 훅의 규칙

- 훅은 항상 최상위 레벨에서 호출되어야 한다.
  - 조건문, 반복문, 중첩 함수, 클래스 내부에서는 훅을 호출하지 않아야 한다.
  - 반환문으로 함수 컴포넌트가 종료되거나, 조건문 또는 변수에 따라 반복문 등으로 훅의 호출 여부(호출되거나 호출되지 않거나)가 결정되어서는 안된다.
    => 이렇게 해야 useState나 usEEffect가 여러 번 호출되더라도 훅의 상태를 올바르게 유지할 수 있게 된다.
- 훅은 항상 함수 컴포넌트나 커스텀 훅 등의 리액트 컴포넌트 내에서만 호출되어야 한다.

=> 이 두 가지 규칙을 지키면 컴포넌트의 모든 상태 관련 로직을 좀 더 명확하게 드러낼 수 있다. 이러한 규칙이 필요한 이유는 리액트에서 훅은 호출 순서에 의존하기 떄문이다. 모든 컴포넌트 렌더링에서 훅의 순서가 항상 동일하게 유지되어야 하며, 이를 통해 항상 동일한 컴포넌트 렌더링이 보장된다.

# 커스텀 훅

## 나만의 훅 만들기

- 현재 쓰고 있는 방법이라 정리 안했습니다 !
