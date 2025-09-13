# 1. API요청

## fetch로 API 요청하기

- 초기 작업에서는 컴포넌트 뱃지 안에서 직접 데이터 패칭을 했으나, 백엔드 API 수정에 대응하기 쉬운 방법으로 변경하기 위해서 api 패칭을 추상화했다.

## 서비스 레이어로 분리하기

- URL 변경을 비롯한 여러 요구사항에 대응할 수 있게 하기 위해, 비동기 호출 코드는 컴포넌트 영역에서 분리되어 다른 영역에서 처리되어야 한다.
- 이때 단순히 fetch 함수를 분리하는 것만으로는 API 요청 정책이 추가되는 것을 해결하기 어렵다.

```ts
async function fetchCart() {
  const controller = new AbortController();

  const timeoutId = setTimeout(() => controller.abort(), 5000);

  const response = await fetch("https://api.baemin.com/cart", {
    signal: controller.signal,
  });

  clearTimeout(timeoutId);

  return response;
}
```

## Axios 활용하기

- fetch는 내장 라이브러리이기 때문에 따로 임포트하거나 설치할 필요 없이 사용할 수 있다. 하지만 많은 기능을 사용하기 위해서는 직접 구현해야하기 때문에 Axios라이브러리를 사용하기도 한다.

### 구현 상황

- 각 서버(주문을 처리하는 서버와 장바구니를 처리하는 서버)가 담당하는 부분이 다르거나 새로운 프로젝트의 일부로 포함될 때 기존에 사용하는 API Entry(Base URL)와는 다른 새로운 URL로 요청해야하는 상황이 생길 수 있다.
- 이렇게 API Entry가 2개 이상일 경우에는 각 서버의 기본 URL을 호출하도록 orderApiRe-quester, orderCartApiRequester 같이 2개 이상의 API 요청을 처리하는 인스턴스를 따로 구성해야 한다.
  - 이후 다른 URL로 서비스 코드를 호출할 때는 각각의 apiRequester를 사용하면 된다.

```ts
const apiRequester: AxiosInstance = axios.create(defaultConfig);
const orderApiRequester: AxiosInstance = axios.create({
    baseURL: "https://api.baemin.or/",
    ..defaultConfig,
})

const cartApiRequester : AxiosInstance = axios.create({
    baseURL : "https://cart.baemin.order/",
    ...defaultConfig
})
```

## Axios 인터셉터 활용하기

- 각 각의 requester는 서로 다른 역할을 담당하는 다른 서버이기 때문에 헤더 설정도 다를 수 있다.

## API응답 타입 지정하기

- 같은 서버에서 오는 응답의 형태는 대체로 통일되어 있어 앞서 소개한 API의 응답 값은 하나의 Response 타입으로 묶을 수 있다.

```ts
interface Response<T> {
  data: T;
  status: string;
  serverDateTime: string;
  errorCode?: string; // FAIL, ERROR
  errorMessage?: string; // FAIL, ERROR
}

const fetchCart = () => AxiosPromise<Response<FetchCartResponse>> => apiRequester.get<Response<FetchCartResponse>> "cart";

const postCart = (postCartRequest:PostCartRequest):AxiosPromise<Response<PostCartResponse>> => apiRequester.post<Response<PostCartResponse>>("cart", postCartRequest);
```

- 이때 UPDATE나 CREATE 같이 응답이 없을 수 있는 API까지 한 번에 apiRequester 내에서 처리하기 까다로워진다.
  - 따라서 Response의 타입은 apiRequester가 모르게 관리되어야 한다.

### 어떻게?

- API 요청 및 응답 값 중에서는 하나의 API 서버에서 다른 API 서버로 넘겨주기만 하는 값도 존재할 수 있다.
- 해당 값에 어떤 응답이 들어있는지 알 수 없거나 값의 형식이 달라지더라도 로직에 영향을 주지 않는 경우에는 unknown 타입을 사용하여 알 수 없는 값임을 표현한다.

```ts
interface response {
  data: {
    cartItems: CartItem[];
    forPass: unknown;
  };
}
```

- 만약 forPass 안에 프론트 로직에서 사용해야하는 값이 있다면, 여전히 어떤 값이 들어올지 모르는 상태이기 때문에 unknown을 유지한다.
- 로그를 위해 단순히 받아서 넘겨주는 값의 타입은 언제든지 변경될 수 있으므로 forPass 내의 값을 사용하지 않아야 한다.
  - 다만, 이미 설계된 프로덕트에서 쓰고 있는 값이라면 프론트 로직에서 써야하는 값에 대해서만 타입을 선언한 다음에 사용하는 게 좋다.

```ts
type ForPass = {
  type: "A" | "B" | "C";
};

const isTargetValue = () => (data.forPass as ForPass).type === "A";
```

## 뷰모델(View Model) 사용하기

- API 응답은 변할 가능성이 크다. 특히 새로운 프로젝트는 서버 스펙이 자주 바뀌기 떄문에 뷰모델을 사용하여 API 변경에 따른 범위를 한정해줘야 한다.
- 특정 객체 리스트를 조회하여 리스트 각각의 내용과 리스트 전체 길이 등을 보여줘야 하는 화면을 떠올려보자. 해당 리스트를 조회하는 fetchList API는 다음처럼 구성될 것이다.

```ts
interface ListResponse {
  items: ListItem[];
}

const fetchList = async (filter?: ListFetchFilter): Promise<ListResponse> => {
  const { data } = await api
    .params({ ...filter })
    .get("/apis/get-list-summaries")
    .call<Response<ListResponse>>();

  return { data };
};
```

- 해당 api를 사용할 때는 다음처럼 사용한다.
- 이 에시에서는 컴포넌트 내부에서 비동기 함수를 호출하고 then으로 처리하고 있지만, 실제 비동기 함수는 컴포넌트 내부에서 직접 호출되지 않는다.

```tsx
const ListPage: React.FC = () => {
  const [totalItemCount, setTotalItemCount] = useState(0);
  const [items, setItems] = useState<ListItem[]>([]);

  useEffect(() => {
    fetchList(filter).then(({ items }) => {
      setCartCount(items.length);
      setItems(items);
    });
  }, []);

  return (
    <div>
      <Chip label={totalItemCount} />
      <Table items={items} />
    </div>
  );
};
```

- 흔히 좋은 컴포넌트는 변경될 이유가 하나뿐인 컴포넌트라고 말한다. API 응답의 items 인자를 좀 더 정확한 개념으로 나타내기 위해 jobItems나 cartItems 같은 이름으로 수정하면 해당 컴포넌트도 수정해야 한다.
  - 이런 이유로 수정해야하는 컴포넌트가 API에 하나라면 좋겠지만, API를 사용하는 기존 컴포넌트도 수정되어야 한다.
- 이런 문제를 해결하기 위해 뷰 모델을 도입할 수 있다.

```ts
// 기존 ListResponse에 더 자세한 의미를 담기 위한 변화
interface JobListItemResponse {
  name: string;
}

interface JobListResponse {
  jobItems: JobListResponse[];
}

class JobList {
  readonly totalItemCount: number;
  readonly items: JobListItemResponse[];

  constructor({ jobItems }: JobListResponse) {
    this.totalItemCount = jobItems.length;
    this.items = jobItems;
  }
}

const fetchJobList = async (
  filter?: ListFetchFilter
): Promise<JobListResponse> => {
  const { data } = await api
    .params({ ...filter })
    .get("/apis/get-list-summaries")
    .call<Response<JobListResponse>>();

  return new JobList(data);
};
```

- 뷰 모델을 만들면 API 응답이 바뀌어도 UI가 깨지지 않게 개발할 수 있다.
- 또한 앞의 예시처럼 API 응답에는 없는 totalItemCount 같은 도메인 개념을 넣을 떄 백엔드가 UI에서 로직을 추가하여 처리할 필요없이 간편하게 새로운 필드를 뷰 모델에 추가할 수 있다.

- 그러나 뷰 모델 방식에서도 문제가 발생할 수 있다. 추상화 레이어 추가는 결국 코드를 복잡하게 만들며 레이어를 관리하고 개발하는 데도 비용이 든다.

- 앞의 코드에서는 JobListItemResponse 타입은 서버에서 지정한 응답 형식이기 때문에 이를 UI에서 활용하려면 다음처럼 더 많은 타입을 선언해야 한다.

  - 꼭 필요한 곳에만 뷰 모델을 부분적으로 만들어서 사용하고, 백엔드와 클라이언트 개발자가 충분히 소통한 다음에 개발하여 API 응답 변화를 최대한 줄이는 것이 좋다.

- 응답 값의 타입이 string이어야 하는데 number로 들어오는 것과 같이 잘못된 타입이 들어오기도 하는데, Superstruct 같은 라이브러리를 사용하면 이 같은 런타임 API 타입 오류를 방지할 수 있다.

## Superstruct를 사용해 런타임에서 응답 타입 검증하기

> - Superstruct를 사용하여 인터페이스 정의와 자바스크립트 데이터의 유효성 검사를 쉽게 할 수 있다.
> - Superstruct는 런타임에서의 데이터 유효성 검사를 통해 개발자와 사용자에게 자세한 런타임 에러를 보여주기 위해 고안되었다.

### 사용법

```ts
const Article = object({
  id: number(),
  title: string(),
  tags: array(string()),
  author: object({
    id: number(),
  }),
});

const data = {
  id: 34,
  title: "Hello World",
  tags: ["news", "features"],
  author: {
    id: 1,
  },
};

assert(data, Article);
is(data, Article);
validate(data, Article);
```

- 먼저 Article이라는 변수는 Superstruct의 object() 모듈의 반환 결과다.
  - 이름에서 알 수 있듯이 object()는 객체를, string()은 string 타입을, number()은 number 타입을 의미한다.
- asset, is, validate 모듈은 모두 데이터의 유효성 검사를 도와주는 모듈이다.

- 세 모듈의 공통점은 데이터 정보를 담은 data 변수와 데이터 명세를 가진 스키마 Article을 인자로 받아 데이터가 스키마와 부합하는지를 검사한다는 것이다.
  - 차이점은 모듈마다 데이터의 유효성을 다르게 접근하고 반환 값 형태가 다르다는 것이다.

#### 모듈의 차이점

- assert는 유효하지 않을 경우 에러를 던진다.
- is는 유효성 검사 결과에 따라 true또는 false, 즉 boolean 값을 반환한다.
- validate는 [error, data] 형식의 튜플을 반환한다.
  - 유효하지 않을 때는 에러 값이 반환되고, 유효한 경우에는 첫 번째 요소로 undefined, 두 번째 요소로 data value가 반환된다.

#### 타입스크립트와 시너지를 내기 위해서는?

```ts
import { Infer, number, object, string } from "superstruct";

const User = object({
  id: number(),
  email: string(),
  name: string(),
});

type User = Infer<typeof User>;
```

```ts
type User = {
  id: number;
  email: string;
  name: string;
};

import { assets } from "superstruct";

function isUser(user: User) {
  assert(user, User);
  console.log("적절한 유저입니다.");
}

const user_A = {
  id: 4,
  email: "test@woowahan.email",
  name: "woowa",
};

const user_B = {
  id: 5,
  email: "wrong@woowahan.email",
  name: 4,
};

isUser(user_A);
isUser(user_B);
//  error TS2345: Argument of type '{id:number; email:string; name:number;}' is no assignable to parameter of type '{id: number; email:string; name:string;}'
```

- 런타임에 데이터가 오염되어 들어왔을 때 이렇게 런타임 에러가 발생하게 된다.
- 컴파일 단계가 아닌 런타임에서도 적절한 데이터인지를 확인하는 검사가 필요할 때 유용하게 사용할 수 있다.

## 실제 API응답 시의 Superstruct 활용 사례

```ts
interface ListItem {
  id: string;
  content: string;
}

interface ListResponse {
  items: ListItem[];
}

const fetchList = async (filter?: ListFetchFilter): Promise<ListResponse> => {
  const { data } = await api
    .params({ ...filter })
    .get("/apis/get-list-summaries")
    .call<Response<ListResponse>>();

  return { data };
};
```

- fetchList 함수를 호출했을 때 id와 content가 담긴 ListItem타입의 배열이 오기를 기대한다.
- 하지만 실제 서버 응답의 형식은 다를 수 있다.
  - 타입스크립트는 컴파일타입에 타입을 검증하는 역할을 한다. 따라서, 타입스크립트 만으로는 실제 서버 응답의 형식과 명시한 타입이 일치하는지를 확인할 수 없다.

```ts
import { assert } from "superstruct";

function isListItem(listItems: ListItem[]) {
  listItems.forEach((listItem) => assert(listItem, ListItem));
}
```

- isListItem은 ListItem의 배열 목록을 받아와 데이터가 ListItem 타입과 동일한지 확인하고 다를 경우에는 에러를 던진다.
- 이제 fetchList 함수에 Superstruct로 작성한 검증 함수를 추가하면 런타임 유효성 검사를 진행할 수 있게 된다.
