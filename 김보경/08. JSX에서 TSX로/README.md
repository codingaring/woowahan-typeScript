# 8장 JSX에서 TSX로

> 리액트의 등작으로 더욱 편리하게 웹 애플리케이션을 개발할 수 있게 되었습니다.
> 이번 장에서는 리액트에서 사용하는 JSX 문법을 타입스크립트에 어떻게 적용하는지를 소개해 보겠습니다.

## 08.1 리액트 컴포넌트 타입

- 리액트 + 타입스크립트 사용시 `@types/react` 패키지에 정의된 리액트 내장 타입을 사용해본 경험은 있을거라 생각합니다.
  - 리액트 내장 타입 중에는 역할이 명확한 것도 있지만, 역할이 비슷해 보이는 타입 또한 있기 때문에 혼동이 있을 수 있습니다.
- 이번 챕터에서는 헷갈릴 수 있는 대표적인 리액트 컴포넌트 타입을 보면서 상황에 따라 어떤 것을 사용하면 좋을지 그리고 사용할 때 유의점은 무엇인지 알아보겠습니다.

### 1. 클래스 컴포넌트 타입

```ts
interface Component<P = {}, S = {}, SS = any>
  extends ComponentLifecycle<P, S, SS> {}

class Component<P, S> {
  /* ... 생략 */
}

class PureComponent<P = {}, S = {}, SS = any> extends Component<P, S, SS> {}
```

    - 클래스 컴포넌트가 상속받는 `React.Component`와 `React.PureComponent`의 타입 정의는 위 예시와 같으며, P와 S는 각각 props와 상태를 의미합니다.
    - 위 예시를 보면 props와 상태 타입을 제네릭으로 받고 있습니다.
    - Welcome 컴포넌트의 props타입을 지정해보면 아래와 코드와 같이 상태가 있는 컴포넌트일 때는 제네릭의 두 번째 인자로 타입을 넘겨주면 상태에 대한 타입을 지정해줄 수 있습니다.

    ```ts
    interface WelcomeProps {
      name: string;
    }

    class Welcome extends React.Component<WelcomeProps> {
      /* ... 생략 */
    }
    ```

### 2. 함수 컴포넌트 타입

```ts
// 함수 선언을 사용한 방식
function Welcome(props: WelcomeProps): JSX.Element {}

// 함수 표현식을 사용한 방식 - React.FC 사용
const Welcome: React.FC<WelcomeProps> = ({ name }) => {};

// 함수 표현식을 사용한 방식 - React.VFC 사용
const Welcome: React.VFC<WelcomeProps> = ({ name }) => {};

// 함수 표현식을 사용한 방식 - JSX.Element를 반환 타입으로 지정
const Welcome = ({ name }: WelcomeProps): JSX.Element => {};

type FC<P = {}> = FunctionComponent<P>;

interface FunctionComponent<P = {}> {
  // props에 children을 추가
  (props: PropsWithChildren<P>, context?: any): ReactElement<any, any> | null;
  propTypes?: WeakValidationMap<P> | undefined;
  contextTypes?: ValidationMap<any> | undefined;
  defaultProps?: Partial<P> | undefined;
  displayName?: string | undefined;
}

type VFC<P = {}> = VoidFunctionComponent<P>;

interface VoidFunctionComponent<P = {}> {
  // children 없음
  (props: P, context?: any): ReactElement<any, any> | null;
  propTypes?: WeakValidationMap<P> | undefined;
  contextTypes?: ValidationMap<any> | undefined;
  defaultProps?: Partial<P> | undefined;
  displayName?: string | undefined;
}
```

> void 타입을 이해한 이후로 VFC의 동작도 함께 이해되는것 같습니다.

- 함수 표현식을 사용하여 함수 컴포넌트를 선언할 때 가장 많이 볼 수 있는 형태는 `React.FC` 혹은 `React.VFC`로 타입을 지정하는 것입니다.
  - 하지만 리액트 18v로 넘어오면서 `React.VFC`가 삭제되고 `React.FC`에서 chidren이 사라지게 되었습니다.
  - 앞으로는 `React.VFC` 대신 `React.FC`또는 props 타입 - 반환 타입을 직접 지정하는 형태로 타이핑을 해주어야 합니다.

#### 번외. 왜 Function Component를 지양해야 하나요? feat.GPT punching

##### 1. 암시적 children props의 문제점 (React 18 이전)

- React 18 이전 버전에서 React.FC는 자동으로 children prop을 포함했습니다

  ```tsx
  // React 18 이전 - 문제가 있는 코드
  interface ButtonProps {
    onClick: () => void;
    variant: "primary" | "secondary";
  }

  const Button: React.FC<ButtonProps> = ({ onClick, variant }) => {
    // children을 받을 의도가 없었는데도 자동으로 포함됨
    return (
      <button onClick={onClick} className={variant}>
        Click me
      </button>
    );
  };

  // 이렇게 사용해도 타입 에러가 발생하지 않음 (의도하지 않은 동작)
  <Button onClick={handleClick} variant="primary">
    <span>Unexpected children</span> {/* 렌더링되지 않지만 타입 에러 없음 */}
  </Button>;
  ```

##### 2. 제네릭 컴포넌트에서의 제약

- 함수 컴포넌트 타입은 제네릭 컴포넌트를 만들 때 복잡성을 증가시킵니다.

  ```tsx
  // 권장하지 않음 - 복잡하고 가독성이 떨어짐
  const GenericList: React.FC<{
    items: T[];
    renderItem: (item: T) => JSX.Element;
  }> = <T,>({ items, renderItem }) => {
    return <ul>{items.map(renderItem)}</ul>;
  };

  // 권장 - 더 깔끔하고 직관적
  function GenericList<T>({
    items,
    renderItem,
  }: {
    items: T[];
    renderItem: (item: T) => JSX.Element;
  }): JSX.Element {
    return <ul>{items.map(renderItem)}</ul>;
  }
  ```

##### 3. defaultProps와의 호환성 문제

```tsx
interface Props {
  name: string;
  age?: number;
}

// 문제가 있는 방식
const Component: React.FC<Props> = ({ name, age = 25 }) => {
  return (
    <div>
      {name} is {age} years old
    </div>
  );
};

// 권장 방식 - defaultProps가 타입 시스템과 잘 동작
function Component({ name, age = 25 }: Props): JSX.Element {
  return (
    <div>
      {name} is {age} years old
    </div>
  );
}
```

##### 4. 반환 타입의 제한

- `React.FC`는 항상 `ReactElement | null`을 반환해야 하므로, 다른 유효한 React return type들을 제한하게 됩니다.

```tsx
// React.FC 사용 시 - 제한적
const StringComponent: React.FC = () => "Hello"; // 타입 에러!

// 권장 방식 - 유연함
function StringComponent(): string {
  return "Hello"; // 정상 작동
}

function NumberComponent(): number {
  return 42; // 정상 작동
}
```

##### React 18에서의 변화

- React 18부터는 `React.FC`에서 자동 children prop이 제거되었지만, 여전히 위에서 언급한 다른 문제들(제네릭, defaultProps, 반환 타입 제한)은 남아있습니다.

### 3. Children props 타입 지정

```ts
type PropsWithChildren<P> = P & { children?: ReactNode | undefined };
```

- 가장 보편적인 children 타입은 `ReactNode | undefined`가 됩니다.
- `ReactNode`는 ReactElement 외에도 boolean, number 등 여러 타입을 포함하고 있는 타입으로, 더 구체적으로 타이핑하는 용도에는 적합하지 않습니다.

  ```ts
  // example 1
  type WelcomeProps = {
    children: "천생연분" | "더 귀한 분" | "귀한 분" | "고마운 분";
  };

  // example 2
  type WelcomeProps = { children: string };

  // example 3
  type WelcomeProps = { children: ReactElement };
  ```

  - 예를 들어, 특정 문자열만 허용하고 싶을 때는 children에 대해 추가로 타이핑을 해주어야 합니다.
  - children에 대한 타입 지정은 다른 prop 타입 지정과 동일한 방식으로 지정할 수 있습니다.

### 4. render 메서드와 함수 컴포넌트의 반환 타입 - `React.ReactElement` vs `JSX.Element` vs `React.ReactNode`

- `React.ReactElement`, `JSX`, `React.ReactNode` 타입은 쉽게 헷갈릴 수 있기 때문에 자세히 살펴보겠습니다.

#### ReactElement

- 먼저 함수 컴포넌트의 반환 타입인 `ReactElement`는 아래와 같이 정의됩니다.
  ```ts
  interface ReactElement<
    P = any,
    T extends string | JSXElementConstructor<any> =
      | string
      | JSXElementConstructor<any>
  > {
    type: T;
    props: P;
    key: Key | null;
  }
  ```
  - `React.createElement`를 호출하는 형태의 구문으로 변환하면 `React.createElement`의 반환 타입은 `ReactElement`입니다.
  - 리액트는 실제 DOM이 아닌 Virtual DOM을 반으로 렌더링하는데 가상 DOM의 엘리먼트는 `ReactElement` 형태로 저장됩니다.
  - 즉, `ReactElement`타입은 리액트 컴포넌트를 객체 형태로 저장하기 위한 포멧입니다.

#### JSX.Element

```ts
declare global {
  namespace JSX {
    interface Element extends React.ReactElement<any, any> {}
  }
}
```

- 함수 컴포넌트 예시에서 `JSX.Element`타입을 사용한 것을 보고 의아했을 수도 있습니다. (몰랐어요 끼얏호우)
- `JSX.Element`타입은 앞의 코드를 보면 알 수 있다시피 리액트의 `ReactElement`를 확장하고 있는 타입이며, 글로벌 네임스페이스에 정의되어 있어 외부 라이브러리에서 컴포넌트 타입을 재정의 할 수 있는 유연성을 제공합니다.
- 이러한 특성으로 인해 컴포넌트 타입을 재정의하거나 변경하는 것이 용이해집니다.

> 아하모먼트 😇
> 글로벌 네임스페이스 (Global Namespace)
> 프로그래밍에서 식별자(변수, 함수, 타입 등)가 정의되는 전역적인 범위를 이야기합니다.
> 자바스크립트나 타입스크립트에서는 기본적인 전역(글로벌) 스코프에서 선언된 변수나 함수 등은 글로벌 네임스페이스에 속합니다.
> 즉, 어떠한 파일이든지 해당 스코프에서 선언된 식별자는 모든 곳에서 접근할 수 있습니다.

#### ReactNode

- ReactNode는 다음과 같이 정의되어 있습니다.

  ```ts
  type ReactText = string | number;
  type ReactChild = ReactElement | ReactText;
  type ReactFragment = {} | Iterable<ReactNode>;

  type ReactNode =
    | ReactChild
    | ReactFragment
    | ReactPortal
    | boolean
    | null
    | undefined;
  ```

  - 단순히 `ReactElement`외에도 boolean, string, number 등의 여러 타입을 포함하고 있습니다.

- `ReactNode`, `JSX.Element`, `ReactElement` 사이의 포함 관계를 정리하면 아래 그림과 같습니다.
  ![Reactnode, JSX. Element, ReactElement 간의 포함 관계](./Reactnode,%20JSX.%20Element,%20ReactElement%20간의%20포함%20관계.png)
  - 레퍼런스 - [ReactNode, ReactElement JSX.Element의 차이](https://medium.com/@gogosky1175/reactnode-reactelement-jsx-element%EC%9D%98-%EC%B0%A8%EC%9D%B4-30bfbd4ef4b7)

### 5. ReactElement, ReactNode, JSX.Element 활용하기

- ReactElement, ReactNode, JSX.Element 는 모두 리액트의 요소를 나타내는 타입입니다.
- 리액트의 요소를 나타내는데 왜 이렇게 많은 타입이 존재할까요?
- 그건 이제 살펴볼겁니다 끼약호우!

```ts
declare namespace React {
  // ReactElement
  interface ReactElement<
    P = any,
    T extends string | JSXElementConstructor<any> =
      | string
      | JSXElementConstructor<any>
  > {
    type: T;
    props: P;
    key: Key | null;
  }

  // ReactNode
  type ReactText = string | number;
  type ReactChild = ReactElement | ReactText;
  type ReactFragment = {} | Iterable<ReactNode>;

  type ReactNode =
    | ReactChild
    | ReactFragment
    | ReactPortal
    | boolean
    | null
    | undefined;
  type ComponentType<P = {}> = ComponentClass<P> | FunctionComponent<P>;
}

// JSX.Element
declare global {
  namespace JSX {
    interface Element extends React.ReactElement<any, any> {
      // ...
    }
    // ...
  }
}
```

- 위 타입을 조금 더 자세히 살펴보겠습니다.

#### ReactElement

- createElement 메서드는 리액트 엘리먼트를 생성하는 메서드 입니다.
- 리액트를 사용하면서 JSX라는 자바스크립트를 확장한 문법을 자주 접했을 텐데 JSX가 createElement 메서드를 호출하기 위한 문법입니다.
- 즉, JSX는 리액트 엘리먼트를 생성하기 위한 문법이며 트랜스파일러는 JSX 문법을 createElement 메서드 호출문으로 변환하여 아 코드와 같이 리액트 엘리먼트를 생성합니다.

  ```ts
  const element = React.createElement(
    "h1",
    { className: "greeting" },
    "Hello, world!"
  );

  // 주의: 다음 구조는 단순화되었다
  const element = {
    type: "h1",
    props: {
      className: "greeting",
      children: "Hello, world!",
    },
  };

  declare global {
    namespace JSX {
      interface Element extends React.ReactElement<any, any> {
        // ...
      }
      // ...
    }
  }
  ```

  - 리액트는 이런 식으로 만들어진 리액트 엘리먼트 객체를 읽어서 DOM을 구성합니다.
  - 리액트에는 여러 개의 createElement 오버라이딩 메서드가 존재하는데, 이 메서드들이 반환하는 타입은 `ReactElement`타입을 기반으로 합니다.
  - 정리하면 ReactElement 타입은 JSX의 createElement 메서드 호출로 생성된 리액트 엘리먼트를 나타내는 타입이라고 볼 수 있습니다.

##### 정확한 변환 과정을 눈에보기 쉽게 정리해볼게요

- 실제 흐름: JSX → ReactElement ← JSX.Element

  ```tsx
  // 1. JSX 코드 (우리가 작성하는 코드)
  const element = <h1 className="greeting">Hello, world!</h1>;

  // 2. 트랜스파일러가 변환 (Babel, TypeScript 등)
  const element = React.createElement(
    "h1",
    { className: "greeting" },
    "Hello, world!"
  );

  // 3. createElement 메서드의 실제 반환값 (런타임)
  const element = {
    type: "h1",
    props: {
      className: "greeting",
      children: "Hello, world!",
    },
    key: null,
    // 기타 React 내부 속성들...
  };
  ```

> JSX 문법 (컴파일 타임) => createElement 호출 (런타임) => 메서드 실행 => ReactElement 객체 생성 (런타임)

#### ReactNode

- ReactNode 타입을 알아보기 이전에 먼저 ReactChild 타입을 알아보겠습니다.
  ```ts
  type ReactText = string | number;
  type ReactChild = ReactElement | ReactText;
  ```
  - ReactChild 타입은 `ReactElement | string | number`로 정의되어 있는 ReactElement보다는 좀 더 넓은 볌위를 갖고 있습니다.
- ReactNode 타입을 살펴보겠습니다.
  ```ts
  type ReactFragment = {} | Iterable<ReactNode>; // ReactNode의 배열 형태
  type ReactNode =
    | ReactChild
    | ReactFragment
    | ReactPortal
    | boolean
    | null
    | undefined;
  ```
  - ReactNode는 앞에서 설명한 ReactChild 외에도 boolean, null, undefined 등 훨씬 넓은 범주의 타입을 포함합니다.
  - 즉, ReactNode는 리액트의 render함수가 반환할 수 있는 모든 형태를 담고 있다고 볼 수 있습니다.

#### JSX.Element

- 앞서 언급했던 JSX.Element 타입의 정의 코드를 다시 살펴보겠습니다.
  ```ts
  declare global {
    namespace JSX {
      interface Element extends React.ReactElement<any, any> {
        // ...
      }
      // ...
    }
  }
  ```
  - JSX.Element는 ReactElement의 제네릭으로 props와 타입 필드에 대해 any 타입을 가지도록 확장하고 있습니다.
  - 즉, JSX.Element는 **ReactElement의 특정 타입으로 props와 타입 필드를 any로 가지는 타입**이라는 것을 알 수 있습니다.
    > JSX.Element는 ReactElement의 특정 타입이다!! props와 타입 필드를 any로 가진다!!

### 6. 사용 예시

- 이들 세가지 타입의 공통점은 모두 리액트에서 제공하는 컴포넌트를 나타낸다는 것입니다.
- 어떤 상황에 어떤 타입을 사용하는게 좋을까용가리?

#### ReactNode

- 리액트노드 타입은 앞서 언급한대로 리액트의 render 함수가 반환할 수 있는 모든 형태를 담고 있기 때문에 리액트 컴포넌트가 가질 수 있는 모든 타입을 의미합니다.
- JSX 형태의 문법을 때로는 string, number, null. undefined 같이 어떤 타입이든 children prop으로 지정할 수 있게 하고 싶다면 ReactNode 타입으로 선언하면 됩니다.
  > 리액트의 내장 타입인 `PropsWithChildren`타입도 ReactNode 타입으로 children을 선언하고 있습니다.
- ReactNode는 prop으로 리액트 컴포넌트가 다양한 형태를 가질 수 있게 하고 싶을때 유용하게 사용됩니다.

#### JSX.Element

- JSX.Element는 앞서 언급한 대로 props와 타입 필드가 any 타입인 리액트 엘리먼트를 나타냅니다.
- 이러한 특성으로 인해 리액트 엘리먼트를 prop으로 전달받아 render props 패턴으로 컴포넌트를 구현할 때 유용하게 활용할 수 있습니다.

  ```tsx
  interface Props {
    icon: JSX.Element;
  }

  const Item = ({ icon }: Props) => {
    // prop으로 받은 컴포넌트의 props에 접근할 수 있다
    const iconSize = icon.props.size;

    return <li>{icon}</li>;
  };

  // icon prop에는 JSX.Element 타입을 가진 요소만 할당할 수 있다
  const App = () => {
    return <Item icon={<Icon size={14} />} />;
  };
  ```

  - icon prop을 JSX.Element 타입으로 선언함으로써 해당 prop에는 JSX 문법만 삽입할 수 있습니다.
  - 또한 icon.props에 접근하여 prop으로 넘겨받은 컴포넌트의 상세한 데이터를 가져올 수 있습니다.

#### ReactElement

- 앞서 살펴본 JSX.Element 예시를 확장하여 추론 관점에서 더 유용하게 활용할 수 있는 방법은 JSX.Element 대신에 ReactElement를 사용하는 것입니다.
- 이때 원하는 컴포넌트의 props를 ReactElement의 제네릭으로 지정해줄 수 있습니다.
- 만약 JSX.Element가 ReactElement의 props타입으로 any가 지정되었다면, ReactElement 타입을 활용하여 제네릭에 직접 해당 컴포넌트의 props타입을 명시해줍니다.

  ```tsx
  interface IconProps {
    size: number;
  }

  interface Props {
    // ReactElement의 props 타입으로 IconProps 타입 지정
    icon: React.ReactElement<IconProps>;
  }

  const Item = ({ icon }: Props) => {
    // icon prop으로 받은 컴포넌트의 props에 접근하면, props의 목록이 추론된다
    const iconSize = icon.props.size;

    return <li>{icon}</li>;
  };
  ```

  - 이처럼 코드를 작성하면 icon.props에 접근할 때 어떤 props가 있는지가 추론되어 IDE에 표시되는 것을 확인할 수 있습니다.

### 7. 리액트에서 기본 HTML 요소 타입 활용하기

- 리액트를 사용하면서 HTML button 태그를 확장한 Button 컴포넌트를 만들어본 경험이 있을 것입니다.

  ```tsx
  const BasicButton = () => <button>정사각형 버튼</button>;
  ```

  - 이렇게 새롭게 만든 버튼 컴포넌트는 기존 HTML button과 같은 역할을 하면서도 새로운 기능이나 UI가 추가된 형태입니다.
  - 기존dml button 태그가 클릭 이벤트를 등록하기 위한 onClick 이벤트 핸들러를 지원하는 것처럼, 새롭게 만든 Button 컴포넌트도 onClick 이벤트 핸들러를 지원해야만 일관성과 편의성을 모두 챙길 수 있습니다.

#### DetailedHTMLProps와 ComponentWithoutRef

- HTML 태그의 속성 타입을 활용하는 대표적인 2가지 방법은 리액트의 DetailedHTMLProps와 ComponentPropsWithoutRef 타입을 활용하는 것입니다.

  - 먼저 React.DetailedHTMLProps를 활용하는 경우에는 아래와 같이 쉽게 HTML 태그 속성과 호환되는 타입을 선언할 수 있습니다.

    ```ts
    type NativeButtonProps = React.DetailedHTMLProps<
      React.ButtonHTMLAttributes<HTMLButtonElement>,
      HTMLButtonElement
    >;

    type ButtonProps = {
      onClick?: NativeButtonProps["onClick"];
    };
    ```

    > ButtonProps의 onClick 타입은 실제 HTML button 태그의 onClick 이벤트 핸들러 타입과 동일하게 할당되는 것을 확인할 수 있습니다.

  - 그리고 React.ComponentPropsWithoutRef 타입은 아래와 같이 활용할 수 있습니다.
    ```ts
    type NativeButtonType = React.ComponentPropsWithoutRef<"button">;
    type ButtonProps = {
      onClick?: NativeButtonType["onClick"];
    };
    ```
    > 마찬가지로 리액트의 button onClick 이벤트 핸들러에 대한 타입이 할당된 것을 볼 수 있습니다.

> 좀 신기하네요 아래는 제가 사용하는 방식입니다.

##### 컴포넌트를 직접 작성할때

```ts
interface BasicButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  // ... restProps
}
```

##### 라이브러리를 확장할때

```ts
interface SomeDomainButtonProps extends ComponentProps<typeof LibButton> {
  // ... restProps
}
```

##### 접근 방식 비교

1. 생산성 측면

```ts
// 📚 책 방식 - 매번 개별 속성을 명시적으로 추가해야 함
type ButtonProps = {
  onClick?: NativeButtonProps["onClick"];
  disabled?: NativeButtonProps["disabled"];
  className?: NativeButtonProps["className"];
  style?: NativeButtonProps["style"];
  id?: NativeButtonProps["id"];
  // ... 필요할 때마다 계속 추가해야 함
};

// ✅ 여러분 방식 - 모든 HTML 속성이 자동으로 포함
interface BasicButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary"; // 추가 속성만 정의
}
```

2. 타입 안정성 측면

- 생략

3. 유지보수성 측면

```ts
// 📚 책 방식 - 새로운 HTML 속성이 필요할 때마다 수정
type ButtonProps = {
  onClick?: NativeButtonProps["onClick"];
  disabled?: NativeButtonProps["disabled"];
  // 🔴 새로운 속성이 필요하면 여기에 추가해야 함
  "aria-label"?: NativeButtonProps["aria-label"]; // 접근성을 위해 추가
  "data-testid"?: NativeButtonProps["data-testid"]; // 테스트를 위해 추가
};

// ✅ 여러분 방식 - 수정 없이 모든 속성 사용 가능
interface BasicButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary";
  // 새로운 HTML 속성은 자동으로 지원됨
}
```

4. 라이브러리 확장에서의 우수성

```tsx
// 라이브러리 컴포넌트 확장 시
import { Button as AntdButton } from "antd";

// ✅ 여러분 방식 - 원본 컴포넌트의 모든 props를 자동 상속
interface CustomButtonProps extends ComponentProps<typeof AntdButton> {
  variant?: "success" | "warning";
  isLoading?: boolean;
}

function CustomButton({ variant, isLoading, ...restProps }: CustomButtonProps) {
  return (
    <AntdButton
      {...restProps} // Antd Button의 모든 props가 타입 안전하게 전달
      loading={isLoading}
      className={`custom-button ${variant} ${restProps.className || ""}`}
    />
  );
}

// 모든 Antd Button의 기능을 그대로 사용 가능
<CustomButton
  variant="success"
  isLoading={loading}
  type="primary" // Antd의 type prop
  size="large" // Antd의 size prop
  icon={<SaveOutlined />} // Antd의 icon prop
  onClick={handleSave}
/>;
```

##### 책의 방식이 적합한 경우

- 매우 제한적은 API를 만들고 싶을때
- 특정 속성만 노출하고 싶을때
- 엄격한 타입 제어가 필요할때

##### 대부분의 경우 (제 방식)

- 기존 컴포넌트를 확장할때
- 유연한 API를 제공하고 싶을때
- 기타 등등

#### 언제 ComponentPropsWithoutRef를 사용하면 좋을까

- 해당 예시에서는 컴포넌트의 props로서 HTML 태그 속성을 확장하고 싶을 때의 상황에 초점을 맞추어 진행해보겠습니다.
- HTML button 태그와 동일한 역할을 하지만 커스텀한 UI를 적용하여 재사용성을 높이기 위한 Button 컴포넌트를 만든다고 가정해보겠습니다.

  - 먼저 HTML button 태그를 태체하는 역할이므로 아래와 같이 기존 button 태그의 HTML 속성을 props로 받을 수 있게 지원해야 합니다.

  ```tsx
  type NativeButtonProps = React.DetailedHTMLProps<
    React.ButtonHTMLAttributes<HTMLButtonElement>,
    HTMLButtonElement
  >;

  const Button = (props: NativeButtonProps) => {
    return <button {...props}>버튼</button>;
  };
  ```

  > NativeButtonProps로 타입을 따로 선언해주고 Button의 props를 따로 확장해서 extends해주는 구조를 가져간다면 깔끔해질 것 같습니다.

  - 해당 예시는 문제가 없는 것 처럼 보이지만 ref를 props로 받을 경우 고려해야 할 사항이 있습니다.
  - 컴포넌트 내에서 ref를 활용해 생성된 DOM 노드에 접근하는 것과 마찬가지로, 재사용할 수 있는 Button 컴포넌트 역시 props로 전달된 ref를 통해 button 태그에 접근하여 DOM 노드를 조작할 수 있을 것으로 예상됩니다.
  - 하지만 여기에서 유의해야 할 점은 **class 컴포넌트와 function 컴포넌트에서 ref를 props로 받아 전달하는 방식에 차이가 있다는 점** 입니다.

    ```tsx
    // 클래스 컴포넌트
    class Button extends React.Component {
      constructor(props) {
        super(props);
        this.buttonRef = React.createRef();
      }

      render() {
        return <button ref={this.buttonRef}>버튼</button>;
      }
    }

    // 함수 컴포넌트
    function Button(props) {
      const buttonRef = useRef(null);

      return <button ref={buttonRef}>버튼</button>;
    }
    ```

    - 클래스 컴포넌트로 만들어진 Button은 props로 전달된 ref가 Button 컴포넌트의 button 태그를 그대로 바라보게 됩니다.
    - 하지만 함수 컴포넌트의 경우 기대와 달리 전달받은 ref가 Button 컴포넌트의 button 태그를 바라보지 않습니다.
    - 클래스 컴포넌트의 ref객체는 마운트된 컴포넌트의 인스턴스를 current 속성값으로 가지지만, 함수 컴포넌트에서는 생성된 인스턴스가 없기 때문에 ref가 기대한 값이 할당되지 않는 것 입니다.
    - 이러한 제약을 극복하고 함수 컴포넌트에서도 ref를 전달받을 수 있도록 도와주는 것이 React.forwardRef 메서드 입니다.

      ```tsx
      // forwardRef를 사용해 ref를 전달받을 수 있도록 구현
      const Button = forwardRef((props, ref) => {
        return (
          <button ref={ref} {...props}>
            버튼
          </button>
        );
      });

      // buttonRef가 Button 컴포넌트의 button 태그를 바라볼 수 있다
      const WrappedButton = () => {
        const buttonRef = useRef();

        return (
          <div>
            <Button ref={buttonRef} />
          </div>
        );
      };
      ```

      - forwardRef는 2개의 제네릭 인자를 받을 수 있는데, 첫 번째는 ref에 대한 타입 정보이며 두번째는 props에 대한 타입 정보입니다. 그렇다면 forwardRef의 타입 선언은 어떻게 이루어질까용가리?

        ```tsx
        type NativeButtonType = React.ComponentPropsWithoutRef<"button">;

        // forwardRef의 제네릭 인자를 통해 ref에 대한 타입으로 HTMLButtonElement를, props에 대한 타입으로 NativeButtonType을 정의했다
        const Button = forwardRef<HTMLButtonElement, NativeButtonType>(
          (props, ref) => {
            return (
              <button ref={ref} {...props}>
                버튼
              </button>
            );
          }
        );
        ```

        - 위 예시코드와 같이 타입을 `React.ComponentPropsWithoutRef<"button">`으로 작성하면 button 태그에 대한 HTML 속성을 모두 포함하지만, ref속성은 제외됩니다.
        - 이러한 특징으로 인해 `DetailedHTMLProps, HTMLProps, ComponentPropsWithRef`와 같이 ref 속성을 포함하는 타입과는 다르게 됩니다.

      - 함수 컴포넌트의 props로 `DetailedHTMLProps`와 같이 ref를 포함하는 타입을 사용하게 되면, 실제로는 동작하지 않는 ref를 받도록 타입이 지정되어 예기치 않은 에러가 발생할 수 있습니다.
      - **따라서 HTML 속성을 확장하는 props를 설계활 때는 ComponentPropsWithoutRef 타입을 사용하여 ref가 실제로 forwardRef와 함께 사용될 때만 props로 전달되도록 타입을 정의하는 것이 안전합니다.**

### 추가 조사

#### React 19에서 업데이트된 forwardRef관련 이슈를 알아보겠습니다.

- [forwardRef관련 react 공식문서 링크](https://ko.react.dev/reference/react/forwardRef)

> AI 를 써서 조금 정리해봤어요

- React 19에서는 ref를 일반 prop으로 취급하도록 변경되었습니다.

##### 1.2 이전 방식 (React 18 이하)

문제점:

- 함수형 컴포넌트는 인스턴스가 없어 ref를 직접 받을 수 없었음
- 반드시 forwardRef로 감싸야 했음
- 추가 보일러플레이트 코드 필요

##### 1.3 새로운 방식 (React 19)

개선사항:

- forwardRef 완전 제거
- ref를 다른 props와 동일하게 취급
- 더 직관적이고 간결한 코드

##### 1.4 TypeScript 타입 처리

React 19에서는 ref 타입을 다음과 같이 처리합니다:

```tsx
import { ComponentPropsWithRef, Ref } from "react";

// 방법 1: 명시적 타입 지정
interface MyInputProps {
  ref?: Ref<HTMLInputElement>;
  placeholder: string;
}

const MyInput: React.FC<MyInputProps> = ({ ref, placeholder }) => {
  return <input ref={ref} placeholder={placeholder} />;
};

// 방법 2: ComponentPropsWithRef 활용
type MyButtonProps = ComponentPropsWithRef<"button"> & {
  variant?: "primary" | "secondary";
};

const MyButton: React.FC<MyButtonProps> = ({ ref, variant, ...props }) => {
  return <button ref={ref} className={variant} {...props} />;
};
```

주의사항:

- `ComponentPropsWithoutRef`는 여전히 유효하지만, React 19에서는 ref가 일반 prop이므로 `ComponentPropsWithRef`를 사용하는 것이 더 자연스러움
- 클래스 컴포넌트에서는 여전히 ref가 컴포넌트 인스턴스를 참조하므로 prop으로 전달되지 않음

#### UI 라이브러리별 대응 예시

##### 2.1 shadcn/ui의 대응

- Before (React 18)

  ```tsx
  import { forwardRef } from "react";
  import * as DialogPrimitive from "@radix-ui/react-dialog";

  const DialogContent = forwardRef<
    React.ElementRef<typeof DialogPrimitive.Content>,
    React.ComponentPropsWithoutRef<typeof DialogPrimitive.Content>
  >(({ className, children, ...props }, ref) => (
    <DialogPrimitive.Content
      ref={ref}
      className={cn("fixed...", className)}
      {...props}
    >
      {children}
    </DialogPrimitive.Content>
  ));

  DialogContent.displayName = DialogPrimitive.Content.displayName;
  ```

- After (React 19)

  ```tsx
  import * as DialogPrimitive from "@radix-ui/react-dialog";

  type DialogContentProps = React.ComponentProps<
    typeof DialogPrimitive.Content
  >;

  const DialogContent: React.FC<DialogContentProps> = ({
    ref,
    className,
    children,
    ...props
  }) => (
    <DialogPrimitive.Content
      ref={ref}
      className={cn("fixed...", className)}
      {...props}
    >
      {children}
    </DialogPrimitive.Content>
  );
  ```

핵심 변경사항:

- forwardRef 제거
- React.ComponentPropsWithoutRef → React.ComponentProps 변경
- ref를 첫 번째 매개변수에서 props의 일부로 처리
- displayName 설정 불필요 (함수명이 자동으로 사용됨)

##### 2.2 Material UI (MUI)의 대응

MUI는 가장 복잡한 마이그레이션 과정을 거쳤으며, 2단계 접근 방식을 사용했습니다:

- Phase 1: 하위 호환성 유지 (forwardRef shim)

  ```tsx
  // MUI의 내부 forwardRef 래퍼
  import * as React from "react";

  const reactMajor = parseInt(React.version.split(".")[0], 10);

  export const forwardRef = <T, P = {}>(
    render: React.ForwardRefRenderFunction<T, P & { ref: React.Ref<T> }>
  ) => {
    if (reactMajor >= 19) {
      // React 19: ref를 prop으로 처리
      const Component = (props: any) => render(props, props.ref ?? null);
      Component.displayName = render.displayName ?? render.name;
      return Component as React.ForwardRefExoticComponent<P>;
    }

    // React 18 이하: 기존 forwardRef 사용
    return React.forwardRef(
      render as React.ForwardRefRenderFunction<T, React.PropsWithoutRef<P>>
    );
  };
  ```

  이 방식의 장점:

  - React 18과 19 모두 지원
  - 점진적 마이그레이션 가능
  - 하나의 코드베이스로 여러 React 버전 지원

- Phase 2: 완전 마이그레이션

  ```tsx
  // Before
  const GridRoot = React.forwardRef((props, ref) => {
    const state = useGridState();
    return <div ref={ref} {...props} {...state} />;
  });

  // After (MUI의 forwardRef shim 사용)
  import { forwardRef } from "@mui/internal-utils";

  const GridRoot = forwardRef((props, ref) => {
    const state = useGridState();
    // 중요: props를 먼저 spread하고 ref는 마지막에
    return <div {...props} {...state} ref={ref} />;
  });
  ```

  주의사항:

  - React 19에서는 ref가 props에 포함되므로, props를 spread한 후 ref를 덮어쓰면 안 됨
  - ref는 항상 마지막에 위치시켜야 함

> 핵심 변경사항으로는 React 19에서는 forwardRef가 더 이상 필요하지 않으며, ref를 일반 prop처럼 직접 전달할 수 있게 되었습니다.

React의 공식 디자인 원칙 문서에서는 다음과 같이 명시합니다:

- "일반적으로 우리는 사용자 영역(userland)에서 구현할 수 있는 기능을 추가하는 것을 거부합니다. 우리는 여러분의 앱을 쓸모없는 라이브러리 코드로 부풀리고 싶지 않습니다."
- forwardRef는 함수형 컴포넌트의 한계를 우회하기 위한 "추가 코드"였으며, React 19에서는 이를 제거함으로써 **불필요한 래퍼 함수 제거**, **더 직관적인 코드 작성 가능**를 개선하였습니다.

## 08.2 타입스크립트로 리액트 컴포넌트 만들기

### 1. JSX로 구현된 Select 컴포넌트

```tsx
const Select = ({ onChange, options, selectedOption }) => {
  const handleChange = (e) => {
    const selected = Object.entries(options).find(
      ([_, value]) => value === e.target.value
    )?.[0];
    onChange?.(selected);
  };

  return (
    <select
      onChange={handleChange}
      value={selectedOption && options[selectedOption]}
    >
      {Object.entries(options).map(([key, value]) => (
        <option key={key} value={value}>
          {value}
        </option>
      ))}
    </select>
  );
};
```

- 예시에서는 JSX로 구현된 Select 컴포넌트를 보여주고 있습니다.
- option, onChange 등을 받도록 구현되어 있지만, 추가적인 설명이 없다면 컴포넌트를 사용하는 입장에서 각 속성에 어떤 타입의 값을 전달해야 하는지 알기 어렵습니다.
  - 따라서 컴포넌트를 사용하는 개발자가 각 속성에 어떤 타입의 값을 전달해야 할지 명확히 알 수 있도록 추가적인 설명이 필요할 수 있습니다.

### 2. JSDocs로 일부 타입 지정하기

- JSDoc을 활용하면 컴포넌트에 대한 설명과 각 속성이 어떤 역할을 하는지 간단하게 알려줄 수 있습니다.

> 이하 생략!

### 3. props 인터페이스 적용하기

> 이하 생략!

### 4. 리액트 이벤트

- 리액트는 Virtual DOM을 다루면서 이벤트도 별도로 관리합니다.
- onclick, onchange와 같이 DOM Element에 등록되어 처리하는 이벤트 리스너와 달리, 리액트 컴포넌트에 등록되는 이벤트 리스너는 onClick, onChange 처럼 카멜케이스로 표기합니다.

  - 따라서 리액트 이벤트는 브라우저의 고유한 이벤트와 완전히 동일하게 동작하지는 않습니다.
  - 예를 들어, 리액트 이벤트 핸들러는 이벤트 버블링 단계에서 호출됩니다?
  - 이벤트 캡처 단계에서 이벤트 핸들러를 등록하기 위해서는 onClickCapture, onChangeCapture와 같이 일반 이벤트 리스너 이름 뒤에 Capture를 붙여야 합니다.
  - 또한 리액트는 브라우저 이벤트를 합성한 합성 이벤트인 `SyntheticEvent`를 제공합니다.

  ```ts
  type EventHandler<Event extends React.SyntheticEvent> = (
    e: Event
  ) => void | null;
  type ChangeEventHandler = EventHandler<ChangeEvent<HTMLSelectElement>>;

  const eventHandler1: GlobalEventHandlers["onchange"] = (e) => {
    e.target; // 일반 Event는 target이 없음
  };

  const eventHandler2: ChangeEventHandler = (e) => {
    e.target; // 리액트 이벤트(합성 이벤트)는 target이 있음
  };
  ```

  - 리액트에서 제공하는 기본 컴포넌트도 SelectProps 처럼 각각 props에 대한 타입을 명시해 주고 있으므로 리액트 컴포넌트에 연결할 이벤트 핸들러도 해당 타입을 일치시켜주어야 합니다.

    - 앞의 예시에서 `React.ChangeEventHandler<HTMLSelectElement>` 타입은 `React.EventHandler<ChangeEvent<HTMLSelectElement>>`와 동일한 타입 입니다.
    - onChange는 HTML의 select 엘리먼트에서 발생하는 change 이벤트에 대한 핸들러로 선언되었습니다.
    - 이제 우리는 `ChangeEvent<HTMLSelectElement>`타입의 이벤트를 매개변수로 받아 해당 이벤트를 처리 하는 핸들러를 작성할 수 있습니다.

    ```tsx
    const Select = ({ onChange, options, selectedOption }: SelectProps) => {
      const handleChange: React.ChangeEventHandler<HTMLSelectElement> = (e) => {
        const selected = Object.entries(options).find(
          ([_, value]) => value === e.target.value
        )?.[0];
        onChange?.(selected);
      };

      return <select onChange={handleChange}>{/* ... */}</select>;
    };
    ```

### 5. 훅에 타입 추가하기

```tsx
type Fruit = keyof typeof fruits; // 'apple' | 'banana' | 'blueberry';
const [fruit, changeFruit] = useState<Fruit | undefined>("apple");

// 에러 발생
const func = () => {
  changeFruit("orange");
};
```

- 타입 매개변수로 좀 더 명확한 타입을 지정함으로써, 다른 개발자가 해당 state나, changeState를 한정된 타입으로만 다룰 수 있게 강제할 수 있습니다.
- `keyof typeof obj`는 해당 객체의 키값을 유니온 타입으로 추출하는 패턴으로 자주 사용됩니다.
  - 앞의 예시에서는 `keyof typeof fruits` 키값만 추출해서 Fruit라는 타입을 새로 만들었습니다.
  - 정의된 타입을 useState의 제네릭으로 활용하여 changeFruit에 정해진 값이 아닌 경우 에러가 발생하도록 설정하였습니다.

### 6. 제네릭 컴포넌트 만들기

- Select를 사용하는 입장에서 제한된 key와 value 만을 가지도록 하려면 어떻게 해야할까요?
- 함수 컴포넌트 역시 함수이므로 제네릭을 사용한 컴포넌트를 만들어낼 수 있습니다.

  ```tsx
  interface SelectProps<OptionType extends Record<string, string>> {
    options: OptionType;
    selectedOption?: keyof OptionType;
    onChange?: (selected?: keyof OptionType) => void;
  }

  const Select = <OptionType extends Record<string, string>>({
    options,
    selectedOption,
    onChange,
  }: SelectProps<OptionType>) => {
    // Select component implementation
  };
  ```

### 7. HTMLAttributes, ReactProps 적용하기

- className, id와 같은 리액트 컴포넌트의 기본 props를 추가하려면 SelectProps에 직접 `className?: string;, id?; string;`을 넣어도 되지만 아래처럼 리액트에서 제공하는 타입을 사용하면 더 정확한 타입을 설정할 수 있게 됩니다.

```ts
type ReactSelectProps = React.ComponentPropsWithoutRef<"select">;

interface SelectProps<OptionType extends Record<string, string>> {
  id?: ReactSelectProps["id"];
  className?: ReactSelectProps["className"];
  // ...
}
```

- `ComponentPropsWithoutRef`는 리액트 컴포넌트의 prop 타입을 반환해주는 타입입니다.
  - `Type["key"]`를 활용하면 객체 형식의 타입 내부 속성값을 가져올 수 있습니다.
- ReactProps에서 여러 개의 타입을 가져와야 한다면 Pick 키워드를 활용하여 아래처럼 사용할 수도 있습니다.
  ```ts
  interface SelectProps<OptionType extends Record<string, string>>
    extends Pick<ReactSelectProps, "id" | "key" | /* ... */> {
    // ...
  }
  ```

9. 공변성과 반공변성

- 앞 예시의 onChange처럼 객체의 메서드 타입을 정의하는 상황은 종종 발생하곤 합니다.
- 객체의 메서드 타입을 정의하는 방법은 2가지가 있습니다. 두 방법은 얼핏 비슷해 보이지만 미묘한 차이를 가지고 있습니다.

  ```tsx
  interface Props<T extends string> {
    onChangeA?: (selected: T) => void;
    onChangeB?(selected: T): void;
  }

  const Component = () => {
    const changeToPineApple = (selectedApple: "apple") => {
      console.log("this is pine" + selectedApple);
    };

    return (
      <Select
        // Error
        // onChangeA={changeToPineApple}
        // OK
        onChangeB={changeToPineApple}
      />
    );
  };
  ```

  - 부모 컴포넌트에서 매개변수가 apple일 때 실행되는 메서드를 생성했다고 가정해보겠습니다.
    - onChangeA일 때는 타입 에러가 발생하지만, onChangeB일 때는 타입 에러가 발생하지 않습니다.

- 아래 예시는 모든 User가 id를 가지고 있으며 회원(Member)은 회원 가입 시 등록한 별명(nickName)을 추가로 갖고 있습니다.

  - Member는 User를 상속하고 있는데 User보다 더 좁은 타입이자 User의 서브 타입입니다.
  - 타입 A가 B의 서브타입일 때, `T<A>`가 `T<B>`의 서브타입이 된다면 공변성을 띄고 있다고 이야기합니다.

    ```ts
    // 모든 유저(회원, 비회원)은 id를 갖고 있음
    interface User {
      id: string;
    }

    interface Member extends User {
      nickName: string;
    }

    let users: Array<User> = [];
    let members: Array<Member> = [];

    users = members; // OK
    members = users; // Error
    ```

  - 일반적인 타입들은 공변성을 가지고 있어서 좁으 타입에서 넓은 타입으로 할당이 가능합니다.
  - 하지만 제네릭 타입을 지닌 함수는 반공변성을 가집니다.

    - 즉, `T<B>`가 `T<A>`의 서브타입이 되어, 좁은 타입 `T<A>`의 함수를 넓은 타입 `T<B>`의 함수에 적용할 수 없다는 것을 의미합니다.

      ```ts
      type PrintUserInfo<U extends User> = (user: U) => void;

      let printUser: PrintUserInfo<User> = (user) => console.log(user.id);

      let printMember: PrintUserInfo<Member> = (user) => {
        console.log(user.id, user.nickName);
      };

      printMember = printUser; // OK.
      printUser = printMember; // Error - Property 'nickName' is missing in type 'User' but required in type 'Member'.
      ```

      - 해당 예시에서 볼 수 있듯이 printMember 타입을 가진 함수는 Member 타입의 객체로부터 nickName까지 함께 출력하는 역할을 합니다.
      - printUser는 `PrintUserInfo<User>`타입으로 정의되어 있어서 Member 타입을 매개변수로 받을 수 없는 상황입니다.
      - 따라서 printMember 함수를 printUser 변수에 할당할 수 없습니다.

  ```ts
  interface Props<T extends string> {
    onChangeA?: (selected: T) => void;
    onChangeB?(selected: T): void;
  }
  ```

  - 다시 기존의 예시로 돌아가 onChangeA같이 함수 타입을 화살표 표기법으로 작성한다면 반공변성을 띄가 됩니다.
  - 또한 onChangeB와 같이 함수 타입을 지정하면 공변성과 반공변성을 모두 가지는 이변성을 띄게 됩니다.
  - 안전한 타입 가드를 위해서는 특수한 경우를 제외하고는 일반적으로 반공변적인 함수 타입을 설정하는 것이 권장됩니다.
