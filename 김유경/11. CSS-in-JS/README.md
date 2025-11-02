# CSS-in-JS란

## 1. CSS-in-JS와 인라인 스타일의 차이

- CSS-in-JS는 CSS-in-CSS보다 더 강력한 추상화 수준을 제공한다.
- CSS-in-JS를 활용하면 자바스크립트로 스타일을 선언적이고 유지보수할 수 있는 방식으로 표현할 수 있다.

```tsx
const textStyles = {
    color:white;
    backgroundColor:black;
}

const SomeComponent = () => {
    return (
        <p style={textStyles}>inline style!</p>
    )
}
```

위 코드는 브라우저에서 DOM 노드를 다음과 같이 연결한다.

```html
<p style="color:white; background-color:black;">inline Style!</p>
```

```tsx
import styled from "styled-components";

const Text = styled.div`
    color : white,
    background : black
`;

// 다음처럼 사용
const Example = () => <Text>{Hello CSS-in-JS}</Text>;
```

위 코드는 다음과 같이 브라우저에서 DOM 노드를 연결한다.

```html
<style>
  .hash136s21 {
    background-color: black;
    color: white;
  }
</style>

<p class="hash136s21">Hello CSS-in-JS</p>
```

인라인 스타일은 DOM 노드에 속성으로 스타일을 추가한 반면에 CSS-in-JS는 DOM 상단에 `<style>` 태그를 추가했다.

- CSS-in-JS를 사용하면 실제로 CSS가 생성되기 때문에 미디어 쿼리, 수도 선택자 등과 같은 CSS 기능을 손쉽게 누릴 수 있따.
- styled-components 같은 라이브러리는 SASS의 기능까지 지원한다.

  > SASS 기능이란?

  > CSS를 더 프로그래밍스럽게 작성할 수 있게 해주는 것으로 아래의 기능을 제공하는 것이다. CSS 컴파일러가 해줌

  - nesting (계층적으로 스타일 작성)
  - 변수 $color-primary
  - mixin (반복 스타일 함수화)
  - 조건, 반복문
  - partial + import 로 구조화

### CSS-in-JS의 장점

- 컴포넌트로 생각할 수 있다
  - CSS-in-JS는 스타일을 컴포넌트 단위로 추상화하여 생각할 수 있게 해준다. 따라서 별도의 스타일시트를 유지보수할 필요 없이 각 컴포넌트의 스타일을 관리할 수 있다.
- 부모와 분리할 수 있다.
  - CSS에는 명시적으로 정의하지 않은 경우 부모 요소에서 자동으로 상속되는 속성이 있다. 하지만 CSS-in-JS는 이러한 상속을 받지 않는다. 따라서 각 컴포넌트의 스타일은 부모와 독립되어 독립적으로 동작한다.
- 스코프를 가진다
  - CSS는 하나의 전역 네임 스페이스를 가지기 때문에 선택자 충돌을 피하기 어렵다. 하나의 프로젝트 내에서는 BEM같은 네이밍 컨벤션이 도움을 줄 수 있지만, 서드파티 코드를 통합할 때는 도움이 되지 않는다. CSS-in-JS는 **CSS로 컴파일될 때 고유한 이름을 생성하여 스코프를 만들어준다. 따라서 선택자 충돌을 방지할 수 있다.**
- 자동으로 벤더 프리픽스가 붙는다.
  - CSS-in-JS 라이브러리들은 자동으로 벤더 프리픽스를 추가하여 브라우저 호환성을 향상해준다.
- 자바스크립트와 CSS 사이에 상수와 함수를 쉽게 공유할 수 있다.
  - CSS-in-JS를 활용하면 자바스크립트 변수, 상수, 함수를 스타일 코드 내에서 쉽게 사용할 수 있다. 이를 통해 스타일과 관련된 로직을 함께 관리할 수 있다.

#### BEM(Block Element Modifier)

CSS 클래스 네이밍 컨벤션의 한 형식을 의미한다. 선택자 충돌과 유지보수 문제를 해결하기 위해 개발된 방법론으로 다음과 같은 구조로 클래스를 작명하고 구성하게 된다.

- Block(블록): 컴포넌트나 모듈의 최상위 레벨 요소를 나타낸다. 클래스 명은 중복되지 않아야 하며, 컴포넌트의 기본 스타일을 정의한다.
- Element(요소): 블록 내부의 하위 요소를 나타낸다. 클래스 명은 블록의 클래스 명을 접두어로 가지며 블록 내부에서만 의미가 있다.
- Modifier(수정자): 블록이나 요소의 상태나 특성을 나타내는 클래스를 추가한다. 이로써 특정 상황에 스타일을 변경하거나 동작을 제어할 수 있다.

#### 벤더 프리픽스(Vendor Prefix)

웹 브라우저마다 지원되는 CSS 속성이나 기능이 다를 때 특정 브라우저에서 정상적으로 동작하도록 하기 위해 추가되는 접두사를 말한다.

> `-webkit-`, `-moz-`, `-ms-`, `-o-`

# CSS-in-JS 등장배경

CSS는 단순히 웹 문서를 꾸미는 역할에 그치지 않고, 웹 애플리케이션의 UI를 설계하는 데에도 사용된다.

- 웹 환경이 변하면서 요구 사항이 다양해지고 복잡도도 계속 증가했다.
- 특히 선택자의 복잡도가 높아짐에 따라 CSS 전처리기 방식이 등장하게 되었다.

### 크리스토퍼 쉬도가 제시한 CSS의 문제점 7가지

1. Global Namespace : 모든 스타일이 전역 공간을 공유하므로 중복되지 않는 CSS 클래스 네임을 고민해야 한다.
2. Dependencies (의존성) : CSS의 의존성과 자바스크립트의 의존성이 달라서 사용하지 않는 스타일이 포함되거나 꼭 필요한 스타일이 누락되는 문제가 발생한다. (현재는 번들러의 발전으로 거의 해결)
3. Dead Code Elimination(불필요한 코드 제거) : 기능 추가/수정/삭제 과정에서 불필요한 CSS를 삭제하기 어렵다.
4. Minification(최소화) : 클래스 이름을 최소화하기 어렵다.
5. Sharing Constants (상수 공유) : 자바스크립트와 상태 값을 공유할 수 없다. (이에 대한 해결책으로 현재는 CSS Variable이 도입되어 CSS의 공식 기능으로 제공)
6. Non-deterministic Resolution (비결정적 해결) : CSS 로드 순서에 따라 스타일 우선순위가 달라진다.
7. Isolation (고립) : CSS의 외부 수정을 관리하기 어렵다. (캡슐화)

- 이때를 기점으로 CSS-in-JS를 구현하기 위한 여러 움직임이 있었고, 그 결과 styled-components, emotion과 같은 CSS-in-JS 라이브러리가 등장하게 되었다.

=> 스타일이라는 요소를 컴포넌트의 일부로 간주할 수 있게 되었으며, HTML, CSS, 자바스크립트 같은 단위로 웹 애플리케이션을 분리하는 대신 스타일을 포함한 컴포넌트 단위로 분리할 수 있게 되었다.

# CSS-in-JS 사용하기

- 생략하겠어요 !

## 유틸리티 타입을 활용하여 스타일 컴포넌트에 prop 전달하기
