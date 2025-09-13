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

# 2. API 상태 관리하기

- 실제 API를 요청하는 코드는 컴포넌트 내에서 비동기 함수를 직접 호출하지는 않는다.
  비동기 API를 호출하기 위해서는 API의 성공/실패에 따른 상태가 관리되어야 하므로 상태 관리 라이브러리의 액션이나 훅과 같이 재정의된 형태를 사용해야 한다.

## 상태 관리 라이브러리 호출하기

- 서비스 코드를 사용해서 비동기 상태를 변화시킬 수 있는 함수를 제공한다.

  - 컴포넌트는 이러한 함수를 사용하여 상태를 구독하며, 상태가 변경될 때 컴포넌트를 다시 렌더링 하는 방식으로 동작한다.

- Redux는 비교적 초기에 나온 상태 관리 라이브러리이다.

```ts
import { useEffect } from "react";
import { useDispatch, useSelector } from "react-redux";

export function useMonitoringHistory() {
  const dispatch = useDispatch();

  // 전역 Store 상태(RootState)에서 필요한 데이터만 가져온다.
  const serchState = useSelector(
    (state: RootState) => state.monitoringHistory.searchState
  );

  // history 내역을 검사하는 함수, 검색 조건이 바뀌면 상태를 갱신하고 API를 호출한다.
  const getHistoryList = async (
    newState : Partial<MonitoringHistorySearchState> => {
        const newSearchState = {...searchState, ...newState};
        dispatch(monitoringHistorySlice.actions.changeSearchState(newSearchState));
        const response = await getHistories(newSearchState); // 비동기 API 호출하기
        dispatch(monitoringHistorySlice.actions.fetchData(response));
    }
  )

  return {
    searchState,
    getHistoryList,
  }
}
```

- 스토어에서 getHistories API만 호출하고, 그 결과를 받아와서 상태를 업데이트하는 일반적인 방식으로 사용할 수 있다.
- 하지만 getHistoryList 함수 안에는 API 상태 저장을 위한 함수를 별도로 호출해서 업데이트해줘야 한다.
- Redux는 비동기 상태가 아닌 전역 상태를 위해 만들어진 라이브러리이기 때문에 미들웨어 라고 불리는 여러 도구를 도입하여 비동기 상태를 관리한다.
  - 따라서 보일러플레이트가 많아지는 등 간편하게 비동기 상태를 관리하기 어려운 상황도 발생한다.

### MobX

- Redux의 불편함을 개선하기 위해 비동기 콜백 함수를 분리하여 액션으로 만들거나 runInAction과 같은 메서드를 사용하여 상태 변경을 처리한다.
- async/ await 나 flow 같은 비동기 상태 관리를 위한 기능도 있어서 이전보다 간편하게 사용할 수 있다.

```ts
import { runInAction, makeAutoObservable } from "mobx";
import type Job from "models/Job";

class JobStore {
  job: Job[] = [];

  constructor() {
    makeAutoObservable(this);
  }
}

type LoadingState = "PENDING" | "DONE" | "ERROR";

class Store {
  job: Job[] = [];
  state: LoadingState = "PENDING";
  errorMsg = "";

  constructor() {
    makeAutoObservable(this);
  }

  async fetchJobList() {
    this.job = [];
    this.state = "PENDING";
    this.errorMsg = "";

    try {
      const projects = await fetchJobList();
      runInAction(() => {
        this.projects = projects;
        this.state = "DONE";
      });
    } catch (e) {
      runInAction(() => {
        this.state = "ERROR";
        this.errorMsg = e.message;
      });
    }
  }
}
```

- 모든 상태 관리 라이브러리에서 비동기 처리 함수를 호출하기 위해 액션이 추가될 때마다 관련된 스토어나 상태가 계속 늘어난다.
- 이로 인한 가장 큰 문제점은 전역 상태 관리자가 모든 비동기 상태에 접근하고 변경할 수 있다는 것이다.
  - 만약 2개 이상의 컴포넌트가 구독하고 있는 비동기 상태가 있다면 쓸데없는 비동기 통신이 발생하거나 의도치 않은 상태 변경이 발생할 수 있다.

## 훅으로 호출하기

- react-query나 useSwr 같은 훅을 사용한 방법은 상태 변경 라이브러리를 사용한 방식보다 훨씬 간단하다.
- 이러한 훅은 캐시를 사용하여 비동기 함수를 호출하며, 상태 관리 라이브러리에서 발생했던 의도치 않은 상태 변경을 방지하는 데 도움을 준다.

```ts
// 사용법은 익히 알고 있는 방식이라서 생략했습니다.
```

# 3. API 에러 핸들링

- 비동기 API 호출을 하다보면 상태 코드에 따라 401 (인증되지 않은 사용자), 404(존재하지 않는 리소스), 500(서버 내부 에러) 혹은 CORS 에러 등 다양한 에러가 발생할 수 있다.
- 타입스크립트에서 이러한 에러를 어떻게 명시하고 처리할 수 있는지 알아보자.

## 타입 가드 활용하기

- Axios 라이브러리에서는 Axios 에러에 대해 isAxiosError라는 타입 가드를 제공하고 있다.
- 이 타입 가드를 직접 활용할 수도 있지만, 서버 에러임을 명확하게 표시하고 서버에서 내려주는 에러 객체에 대해서도 구체적으로 정의함으로써 에러 객체가 어떤 속성을 가졌는지를 파악할 수 있다.

```ts
interface ErrorResponse {
  status: string;
  serverDateTime: string;
  errorCode: string;
  errorMessage: string;
}
```

- ErrorResponse 인터페이스를 사용하여 처리해야 할 Axios 에러 형태는 AxiosError<ErrorResponse> 로 표현할 수 있으며 다음과 같이 타입 가드를 명시적으로 작성할 수 있다.

```ts
function isServerError(error: unknown): error is AxiosError<ErrorResponse> {
  return axios.isAxiosError(error);
}
```

> 사용자 정의 타입 가드를 정의할 때는 타입 가드 함수의 반환타입으로 parameterName is Type 형태의 타입 명제를 정의해주는게 좋다. 이때 parameterName은 타입 가드 함수의 시그니처에 포함된 매개변수여야 한다.

```ts
const onClickDeleteHistoryButton = async (id: string) => {
  try {
    await axios.post("l....", { id });

    alert("주문 내역이 삭제되었습니다.");
  } catch (error) {
    if (isServerError(e) && e.response && e.response.data.errorMessage) {
      // 서버에러 일 때의 처리임을 명시적으로 알 수 있다.
      setErrorMessage(e.response.data.errorMessage);
      return;
    }

    setErrorMessage(
      "일시적인 에러가 발생했습니다. 잠시 후 다시 시도해 주세요."
    );
  }
};
```

## 에러 서브클래싱하기

- 실제 요청을 처리할 때 단순한 서버 에러도 발생하지만 인증 정보 에러, 네트워크 에러, 타임 아웃 에러 같은 다양한 에러가 발생하기도 한다.
- 이를 더욱 명시적으로 표시하기 위해 서브클래싱을 활용할 수 있다.

> **서브 클래싱(Subclassing)** <br />
> 기존 (상위 또는 부모) 클래스를 확장하여 새로운 (하위 또는 자식) 클래스를 만드는 과정을 말한다. <br />
> 새로운 클래스는 상위 클래스의 모든 속성과 클래스를 상속받아 사용할 수 있고 추가적인 속성과 메서드를 정의할 수도 있다.

- 사용자에게 주문 내역을 보여주기 위해 서버에 주문 내역을 요청할 때는 다음과 같은 코드를 작성할 수 있다.

```ts
const getOrderHistory = async (page: number): Promise<History> => {
  try {
    const { data } = await axios.get("");
    const history = await JSON.parse(data);

    return history;
  } catch (error) {
    alert(error.message);
  }
};
```

- 이렇게 작성하면 사용자에게는 어떤 에러가 발생한 것인지 확인시킬 수 있지만, 개발자는 구분할 수 없다.

```ts
class OrderHttpError extends Error {
  private readonly privateResponse: AxiosResponse<ErrorResponse> | undefined;

  constructor(message?: string, response?: AxiosResponse<ErrorResponse>) {
    super(message);
    this.name = "OrderHttpError";
    this.privateResponse = response;
  }

  get response(): AxiosResponse<ErrorResponse> | undefined {
    return this.privateResponse;
  }
}

class NetworkError extends Error {
  constructor(message = "") {
    super(message);
    this.name = "NetworkError";
  }
}

class UnauthorizedError extends Error {
  constructor(message: string, response?: AxiosResponse<ErrorResponse>) {
    super(message, response);
    this.name = "UnauthorizedError";
  }
}
```

- 에러와 같이 에러 객체를 상속한 OrderHttpError, NetworkError, UnauthorizedError를 정의한다.
- Axios를 사용하고 있다면 조건에 따라 인터셉터에서 적합한 에러 객체를 전달할 수 있다.

```ts
const httpErrorHandler = (
  error: AxiosError<ErrorResponse> | Error
): Promise<Error> => {
  let promiseError: Promise<Error>;

  if (axios.isAxiosError(error)) {
    if (Object.is(error.code, "ECONNABORTED")) {
      promiseError = Promise.reject(new TimeoutError());
    } else if (Object.is(error.message, "Network Error")) {
      promiseError = Promise.reject(new NetworkError(""));
    } else {
      const { response } = error as AxiosError<ErrorResponse>;

      switch (response?.status) {
        case HttpStatusCode.UNAUTHORIZED:
          promiseError = Promise.reject(
            new UnauthorizedError(response?.data.message, response)
          );
          break;
        default:
          promiseError = Promise.reject(
            new OrderHttpError(response?.data.message, response)
          );
      }
    }
  } else {
    promiseError = Promise.reject(error);
  }

  return promiseError;
};
```

- 이렇게 정의해둔 후, 다시 요청 코드로 돌아와서 이렇게 사용해줄 수 있다.

```ts
const onActionError = (
  error: unknown,
  params?: Omit<AlertPopup, "type" | "message">
) => {
  if (error instanceof UnauthorizedError) {
    onUnauthorizedError(
      error.message,
      errorCallback?.onUnauthorizedErrorCallback
    );
  } else if (error instanceof NetworkError) {
    alert("네트워크 연결이 원활하지 않습니다. 잠시 후 다시 시도해 주세요.", {
      onClose: errorCallback?.onNetworkErrorCallback,
    });
  } else if (error instanceof OrderHttpError) {
    alert(error.message.params);
  } else if (error instanceof Error) {
    alert(error.message.params);
  } else {
    alert(defaultHttpErrorMessage, params);
  }
};

const getOrderHistory = async (page: number): Promise<History> => {
  try {
    const { data } = await fetchOrderHistory({ page });
    const history = await JSON.parse(data);

    return history;
  } catch (error) {
    onActionError(error);
  }
};
```

## 인터셉터를 활용한 에러 처리

## 에러 바운더리를 활용한 에러 처리

## 상태 관리 라이브러리에서의 에러 처리

## react-query를 활용한 에러 처리

## 그 밖의 에러 처리

# 4. API 모킹

## JSON 파일 불러오기

- 간단한 조회만 필요한 경우에는 \*.json 파일을 만들거나 자바스크립트 파일 안에 JSON 형식의 정보를 저장하고 익스포트 해주는 방식을 사용하면 된다.
- 이후 GET 요청에 파일 경로를 삽입해주면, 조회 응답으로 원하는 값을 받을 수 있다.

```ts
const SERVICES: Service[] = [
  {
    id: 0,
    name: "배달의 민족",
  },
];

export default SERVICES;

// api
const getServices = ApiRequester.get("/mock/services.ts");
```

## NextApiHandler 활용하기

- 프로젝트에서 Next.js를 사용하고 있다면 NextApiHandler를 활용할 수 있다.
- 응답하고자 하는 값을 정의하고 핸들러 안에서 요청에 대한 응답을 정의한다.

## API 요청 핸들러에 분기 추가하기

- 요청 경로를 수정하지 않고 평소에 개발할 때 필요한 경우에만 실제 요청을 보내고 그 외에는 목업을 사용하여 개발하고 싶다면 다음과 같이 처리할 수 있다.

```ts
const mockFetchBrands = (): Promise<FetchBrandsResponse> = new Promise((resolve)=>{
    setTimeout(()=>{
        resolve({
            status: "SUCCESS",
            message: null,
            data [{
                id : 1,
                label :"배민스토어"
            },
            {
                id : 2,
                label :"비마트"
            }]
        })
    },500)
})

const fetchBrands = () => {
    if(useMock){
        return mockFetchBrands();
    }

    return requester.get("/brands")
}
```

- 이 방법을 사용하면 갭잘이 완료된 이후에도 유지보수할 때 목업 함수를 사용할 수 있다. 필요한 경우에만 실제 API에 요청을 보내고 평소에는 서버에 의존하지 않고 개발할 수 있게 된다.

## axios-mock-adapter로 모킹하기

## 목업 사용 여부 제어하기
