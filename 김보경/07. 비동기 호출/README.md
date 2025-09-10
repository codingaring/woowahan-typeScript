# 7장 비동기 호출

> 비동기 처리를 다룰 때는 다음과같은 사항을 고려해야 합니다.
>
> - 현재 비동기 동작이 어떤 상태인가?
> - 비동기 동작을 위해 필요한 정보가 무엇인가?
> - 요청이 성공했다면 실패에 대한 정보를 어떻게 확인할 것인가?
> - 요청이 실패했다면 실패에 대한 정보를 어떻게 확인할 것인가?
> - 비동기 요청에 대한 코드를 쉽게 유지할 수 있도록 어떻게 구조화하고 관리할 것인가?
>
> 이 장에서는 타입스크립트에서 비동기 요청을 어떻게 처리하고 관리하는지를 다뤄볼 예정입니다.

## 07.1 API 요청

### 1. fetch로 API 요청하기

- fetch를 하드코딩으로 작성하며 수정사항이나 새로운 API요청 정책이 추가될 때마다 하드코딩된 비동기 코드를 변경해야하는 취약점과 번거로움을 설명하고 있음

### 2. 서비스 레이어로 분리하기

- API 요청 정책이 추가되어 코드가 변경될 수 있다는 것을 감안한다면, 비동기 호출 로직은 컴포넌트 영역에서 분리되어 다른 영역(서비스 레이어)에서 처리되어야 합니다.
- 그러나 단순히 함수를 분리하는 것만으로는 추가정책을 해결하거나 쿼리 매개변수 그리고 헤더 추가 또는 쿠키를 읽어 토큰을 집어넣는 등 다양한 API정책을 모두 구현하는 것은 번거로운 일입니다.

### 3. Axios활용하기

- 내장 라이브러리인 fetch를 사용해도 되지만, 기능구현하기엔 번거로우니까 Axios를 사용합시다!

  ```ts
  export const apiRequester: AxiosInstance = axios.create({
    baseURL: "https://api.baemin.com",
    timeout: 5000,
  });

  export const fetchCart = (): AxiosPromise<FetchCartResponse> =>
    apiRequester.get<FetchCartResponse>("cart");

  export const postCart = (
    postCartRequest: PostCartRequest
  ): AxiosPromise<PostCartResponse> =>
    apiRequester.post<PostCartResponse>("cart", postCartRequest);
  ```

  - 각 서버가 담당하는 부분이 다르거나 새로운 프로젝트의 일부로 포함될 대 기존에 사용하는 옵션값과는 다른 새로운 URL로 요청해야 하는 상황이 생길 수 있습니다.
  - 이럴때에는 아래 예시화 같이 각 서버의 기본 URL을 호출하도록 2개 이상의 API요청을 처리하는 인스턴스를 따로 구성해주어야 합니다.

    ```ts
    import axios, { AxiosInstance } from "axios";

    const defaultConfig = {};

    const apiRequester: AxiosInstance = axios.create(defaultConfig);
    const orderApiRequester: AxiosInstance = axios.create({
      baseURL: "https://api.baemin.or/",
      ...defaultConfig,
    });
    const orderCartApiRequester: AxiosInstance = axios.create({
      baseURL: "https://cart.baemin.order/",
      ...defaultConfig,
    });
    ```

### 4. Axios 인터셉터 사용하기

- 각각의 requester 인스턴스는 서로 다른 역할을 담당하는 다른 서버이기 때문에 각 인스턴스별로 다른 헤더값을 설정해줘야 하는 로직이 필요할 수도 있습니다.

  - 이때 인터셉터 기능을 사용하여 requester에 따라 비동기 호출 내용을 추가해서 처리할 수 있습니다.
  - 또 API 에러처리에서 하나의 에러 객체로 묶어서 처리할 수도 있습니다.

    ```ts
    import axios, {
      AxiosInstance,
      AxiosRequestConfig,
      AxiosResponse,
    } from "axios";

    const getUserToken = () => "";
    const getAgent = () => "";
    const getOrderClientToken = () => "";
    const orderApiBaseUrl = "";
    const orderCartApiBaseUrl = "";
    const defaultConfig = {};
    const httpErrorHandler = () => {};

    const apiRequester: AxiosInstance = axios.create({
      baseURL: "https://api.baemin.com",
      timeout: 5000,
    });

    const setRequestDefaultHeader = (requestConfig: AxiosRequestConfig) => {
      const config = requestConfig;
      config.headers = {
        ...config.headers,
        "Content-Type": "application/json;charset=utf-8",
        user: getUserToken(),
        agent: getAgent(),
      };
      return config;
    };

    const setOrderRequestDefaultHeader = (
      requestConfig: AxiosRequestConfig
    ) => {
      const config = requestConfig;
      config.headers = {
        ...config.headers,
        "Content-Type": "application/json;charset=utf-8",
        "order-client": getOrderClientToken(),
      };
      return config;
    };

    // `interceptors` 기능을 사용해 header를 설정하는 기능을 넣거나 에러를 처리할 수 있다
    apiRequester.interceptors.request.use(setRequestDefaultHeader);
    const orderApiRequester: AxiosInstance = axios.create({
      baseURL: orderApiBaseUrl,
      ...defaultConfig,
    });
    // 기본 apiRequester와는 다른 header를 설정하는 `interceptors`
    orderApiRequester.interceptors.request.use(setOrderRequestDefaultHeader);
    // `interceptors`를 사용해 httpError 같은 API 에러를 처리할 수도 있다
    orderApiRequester.interceptors.response.use(
      (response: AxiosResponse) => response,
      httpErrorHandler
    );
    const orderCartApiRequester: AxiosInstance = axios.create({
      baseURL: orderCartApiBaseUrl,
      ...defaultConfig,
    });
    orderCartApiRequester.interceptors.request.use(setRequestDefaultHeader);
    ```

    > 그냥 인스턴스 별로 인터셉터로 알맞는 헤더값 주입해주는 내용이네요

  - 이와 달리 요청 옵션에 따라 다른 인터셉터를 만들기 위해 빌더 패턴을 추가하여 `APIBuilder`같은 클래스 형태로 구성하기도 합니다.

    > 아하모먼트 😇
    > 빌더패턴은 객체 생성을 더 편리하고 가독성 있게 만들기 위한 디자인 패턴중 하나입니다. 주로 복잡한 객체의 생성을 단순화하고, 객체 생성 과정을 분리하여 객체를 조립하는 방법을 제공합니다.

    ```ts
    import axios, { AxiosPromise } from "axios";

    // 임시 타이핑
    export type HTTPMethod = "GET" | "POST" | "PUT" | "DELETE";

    export type HTTPHeaders = any;

    export type HTTPParams = unknown;

    //
    class API {
      readonly method: HTTPMethod;
      readonly url: string;
      baseURL?: string;
      headers?: HTTPHeaders;
      params?: HTTPParams;
      data?: unknown;
      timeout?: number;
      withCredentials?: boolean;

      constructor(method: HTTPMethod, url: string) {
        this.method = method;
        this.url = url;
      }

      call<T>(): AxiosPromise<T> {
        const http = axios.create();

        // 만약 `withCredential`이 설정된 API라면 아래 같이 인터셉터를 추가하고, 아니라면 인터셉터 를 사용하지 않음
        if (this.withCredentials) {
          http.interceptors.response.use(
            (response) => response,
            (error) => {
              if (error.response && error.response.status === 401) {
                /* 에러 처리 진행 */
              }
              return Promise.reject(error);
            }
          );
        }
        return http.request({ ...this });
      }
    }

    export default API;
    ```

    - 이처럼 기본 API 클래스로 실제 호출 부분을 구성하고, 위와 같은 API를 호출하기 위한 Wrapper를 빌더 패턴으로 만듭니다.

    ```ts
    import API, { HTTPHeaders, HTTPMethod, HTTPParams } from "./7.1.4-2";

    const apiHost = "";

    class APIBuilder {
      private _instance: API;

      constructor(method: HTTPMethod, url: string, data?: unknown) {
        this._instance = new API(method, url);
        this._instance.baseURL = apiHost;
        this._instance.data = data;
        this._instance.headers = {
          "Content-Type": "application/json; charset=utf-8",
        };
        this._instance.timeout = 5000;
        this._instance.withCredentials = false;
      }

      static get = (url: string) => new APIBuilder("GET", url);

      static put = (url: string, data: unknown) =>
        new APIBuilder("PUT", url, data);

      static post = (url: string, data: unknown) =>
        new APIBuilder("POST", url, data);

      static delete = (url: string) => new APIBuilder("DELETE", url);

      baseURL(value: string): APIBuilder {
        this._instance.baseURL = value;
        return this;
      }

      headers(value: HTTPHeaders): APIBuilder {
        this._instance.headers = value;
        return this;
      }

      timeout(value: number): APIBuilder {
        this._instance.timeout = value;
        return this;
      }

      params(value: HTTPParams): APIBuilder {
        this._instance.params = value;
        return this;
      }

      data(value: unknown): APIBuilder {
        this._instance.data = value;
        return this;
      }

      withCredentials(value: boolean): APIBuilder {
        this._instance.withCredentials = value;
        return this;
      }

      build(): API {
        return this._instance;
      }
    }

    export default APIBuilder;
    ```

    - 위 APIBuilder를 사용하는 예시코드도 같이 보겠습니다.

    ```ts
    import APIBuilder from "./7.1.4-3";

    // ex
    type Response<T> = { data: T };
    type JobNameListResponse = string[];

    const fetchJobNameList = async (name?: string, size?: number) => {
      const api = APIBuilder.get("/apis/web/jobs")
        .withCredentials(true) // 이제 401 에러가 나는 경우, 자동으로 에러를 탐지하는 인터셉터를 사용하게 된다
        .params({ name, size }) // body가 없는 axios 객체도 빌더 패턴으로 쉽게 만들 수 있다
        .build();
      const { data } = await api.call<Response<JobNameListResponse>>();
      return data;
    };
    ```

    - 해당 패턴은 보일러플레이트 코드가 많다는 단점을 갖고있지만 옵션이 다양한 경우 인터셉터를 설정값에 따라 적용하고, 필요 없는 인터셉터는 선택적으로 사용할 수 있다는 장점도 갖고있습니다.

### 5. API 응답 타입 지정하기

- 같은 서버에서 오는 Response는 대체로 일관된 구조로 되어있으므로 하나의 타입으로 묶을 수 있습니다.

  ```ts
  export interface Response<T> {
    data: T;
    status: string;
    serverDateTime: string;
    errorCode?: string; // FAIL, ERROR
    errorMessage?: string; // FAIL, ERROR
  }

  const fetchCart = (): AxiosPromise<Response<FetchCartResponse>> =>
    apiRequester.get<Response<FetchCartResponse>>("cart");

  const postCart = (
    postCartRequest: PostCartRequest
  ): AxiosPromise<Response<PostCartResponse>> =>
    apiRequester.post<Response<PostCartResponse>>("cart", postCartRequest);
  ```

  - 이와 같이 서버에서 응답을 통일해줄 때 주의할 점이 있습니다. - Response 타입을 apiRequester 내에서 처리하고 싶은 생각이 들 수 있는데, 이렇게 하면 UPDATE나 CREATE같이 응답이 없을 수 있는 API를 처리하기 까다로워 집니다.
    ```ts
    const updateCart = (
      updateCartRequest: unknown
    ): AxiosPromise<Response<FetchCartResponse>> => apiRequester.get("cart");
    ```
    - 따라서 Response 타입은 apiRequester가 모르게 관리되어야 합니다.

### 6. 뷰 모델(View Model) 사용하기

- API 응답은 변할 가능성이 큽니다. 특히 서버 스펙이 자주 바뀔수 있는 환경에서는 뷰모델을 사용하여 API 변경에 따른 범위를 한정해주어야 합니다.
- 흔히 좋은 컴포넌트는 변경될 이유가 하나뿐인 컴포넌트라고 말합니다. API를 여러 컴포넌트에서 사용하는 경우 수정사항이 생긴다면 기존 컴포넌트들도 모두 수정해야 하는 상황이 발생하곤 합니다.
- 이러한 문제를 해결하기 위해 뷰 모델을 도입할 수 있습니다.

  ```ts
  // 기존 ListResponse에 더 자세한 의미를 담기 위한 변화
  interface JobListItemResponse {
    name: string;
  }

  interface JobListResponse {
    jobItems: JobListItemResponse[];
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

  - 뷰 모델을 만들면 API 응답이 변경되어도 UI가 깨지지 않도록 개발할 수 있습니다. 또한 도메인 개념을 추가 구성하거나 백엔드 혹은 UI에서 로직을 추가하여 처리할 필요 없이 간편하게 새로운 필드를 뷰 모델에 추가할 수 있습니다.
  - 그러나 뷰 모델 방식에서도 문제가 발생할 수 있습니다.
    - 추상화 레이어 추가는 결국 코드를 복잡하게 만들며 레이어를 관리하고 개발하는 데도 비용이 듭니다.
      > 저희는 그래서 뷰모델 레이어를 구성하지 않고 serverFn에서 직접 처리해서 프론트로 받는 구조를 취하고 있습니다.
      > 하지만 언젠가는 뷰모델 (custom hook)을 사용하게 될 날이 올거같긴 해요
    - 단순히 API 20개를 사용한다면 20개의 응답이 추가될 것입니다. 이 말은 20개 이상의 뷰 모델이 추가될 수 있다는 뜻입니다.
  - 결국 API 응답이 변경되는 상황에서 클라이언트 코드를 수정하는데 들어가는 비용을 줄이면서도 도메인 일관성을 지킬 수 있는 절충안을 찾아야 합니다.
    - 꼭 필요한 곳에만 뷰 모델을 부분적으로 만들어서 사용하기!!!!
    - 백엔드와 클라이언트 개발자의 의사소통!!!!!!

> 아하모먼트 😇
> React에 가장 적합한 디자인패턴이 MVVM이라고 합니다
> 뷰모델(커스텀훅)이 데이터 상태를 관리하고, view에서 이를 보여주는 방식이라 단방향 데이터 흐름과 역할이 명확히 구분되어있는 패턴이라고 할수있죠.
> 예시에서도 추상레이어를 언급했듯 커스텀훅으로 모듈화하는것이 동일한 개념으로 이해되었습니다.
> [React Architecture 정리](https://daily-programmers-diary.tistory.com/8)

### 7. Superstruct를 사용해 런타임에서 응답 타입 검증하기

- 타입스크립트는 정적 검사 도구로 런타임에서 발생하는 오류는 찾아낼 수 없습니다. 런타임에 API 응답의 오류를 방지하려면 Superstruct 같은 라이브러리를 사용하면 됩니다.
- 런타임 응답 타입 검증을 하기 위해 사용하는 Superstruct 라이브러리를 소개하겠습니다.
  - Superstruct를 사용하여 인터페이스 정의와 자바스크립트 데이터의 유효성 검사를 쉽게 할 수 있습니다.
  - Superstruct는 런타임에서의 데이터 유효성 검사를 통해 개발자와 사용자에게 자세한 런타임 에러를 보여주기 위해 고안되었습니다.
- 간단하게 Superstruct 사용 방법을 살펴보겠습니다.

  ```ts
  import {
    assert,
    is,
    validate,
    object,
    number,
    string,
    array,
  } from "superstruct";

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

  > Superstruct 내용은 zod를 사용한 경험과 동일한것 같아서 가볍게 넘어갔습니다.
  > [whats_your_experience_with_superstruct](https://www.reddit.com/r/typescript/comments/1kaibjf/whats_your_experience_with_superstruct/)
  > 궁금해서 커뮤니티도 잠깐 찾아보고 생각을 해본 결과 저는 앞으로도 zod를 기반으로 작업을 하게될거같아요.
  > zod에서 많은 기능을 제공하고 모든 기능을 사용할 수는 없다고 해도 편의성과 확장성을 고려한다면 zod쪽이 훨씬 우수한 것 같고, superstruct 대비 거대한 모듈사이즈와 zod가 제공하는 모든 기능을 사용하지 않는다고 해도, zod mini라는 선택지 또한 있기에 superstruct를 고려할 요소가 없다고 결론지었습니다.

## 07.2 API 상태 관리하기

- 실제 API를 요청하는 코드는 컴포넌트 내에서 비동기 함수를 직접 호출하지는 않습니다.
- API의 성공 실패에 따른 상태가 관리되어야 하므로 상태 관리 라이브러리의 aciton이나 hook과 같이 재정의된 형태를 사용해야 합니다.

### 1. 상태 관리 라이브러리에서 호출하기

### 2. 훅으로 호출하기

- 앞 내용은 알고있던 내용이라 가볍게 지나갔습니다.

> 번외로 queryClient.invalidQuery 등 return 값이 없는 처리를 진행할때에는 함수 앞에 void를 명시해주는게 좋은것 같다는 개인적인 생각이 듭니다.

> 아마모먼트 😇
> 폴링 (polling)
> 클라이언트가 주기적으로 서버에 요청을 보내 데이터를 업데이트하는 것 입니다.
> 클라이언트는 일정한 시간 간격으로 서버에 요청을 보내고, 서버는 해당 요청에 대해 최신 상태의 데이터를 응답으로 보내주는 방식을 이야기합니다.

> 서버에서 스케쥴러로 클라이언트로 지속적인 값을 보내주는것과 폴링으로 클라이언트에서 지속적으로 서버를 호출하는것 등 다양한 처리방법이 있는것 같네요

```tsx
const JobList: React.FC = () => {
  // 비동기 데이터를 필요한 컴포넌트에서 자체 상태로 저장
  const {
    isLoading,
    isError,
    error,
    refetch,
    data: jobList,
  } = useFetchJobList();

  // 간단한 Polling 로직, 실시간으로 화면이 갱신돼야 하는 요구가 없어서 // 30초 간격으로 갱신한다
  useInterval(() => refetch(), 30000);

  // Loading인 경우에도 화면에 표시해준다
  if (isLoading) return <LoadingSpinner />;

  // Error에 관한 내용은 11.3 API 에러 핸들링에서 더 자세하게 다룬다
  if (isError) return <ErrorAlert error={error} />;

  return (
    <>
      {jobList.map((job) => (
        <Job job={job} />
      ))}
    </>
  );
};
```

- 우아한 형제들에서 이야기하는 react-query 도입 시도 내용이 있는것 같습니다.
  - 최근 사내에서도 Redux나 MobX와 같은 전역 상태 관리 라이브러리를 변경하고자 하는 시도가 이루어지고 있습니다.
  - 앞서 언급했다시피 상태관리 라이브러리에서는 비동기 상태를 변경하는 코드가 점점 추가되면 전역 상태 관리 스토어가 비대해지기 때문입니다.
  - 단순히 상태를 변경하는 액션이 증가하는 것뿐만 아니라 전역 상태 자체도 복잡해집니다.
  - 에러 발생, 로딩중 등과 같은 상태는 전역으로 관리할 필요가 거의 없습니다.
    - **다른 컴포넌트가 에러 상태인지, 성공 상태인지를 구독하는 경우 컴포넌트의 결합도와 복잡도가 높아져 유지보수를 어렵게 만들 수 있습니다.**

## 07.3 API 에러 핸들링

- 비동기 API 호출을 하다 보면 상태 코드에 따라 401(invalidate), 404(not found), 500(internal server errro) 혹은 CORS 에러 등 다양한 에러가 발생할 수 있습니다.
- 타입스크립트에서는 어떻게 이러한 에러를 처리하고 명시할 수 있는지에 대해 알아보겠습니다. 끼얏호우

### 1. 타입 가드 활용하기

- Axios 라이브러리에서는 Axios 에러에 대해 `isAxiosError` 라는 타입 가드를 제공하고 있습니다.

  - `isAxiosError` 타입 가드를 직접 사용할 수도 있지만, 서버 에러임을 명확하게 표시하고 서버에서 내려주는 응답 객체에 대해서도 구체적으로 정의함으로써 에러 객체가 어떤 속성을 가졌는지를 파악할 수 있습니다.
  - 아래와 같이 서버에서 전달하는 공통 에러 객체에 대한 타입을 정의할 수 있습니다.

  ```ts
  interface ErrorResponse {
    status: string;
    serverDateTime: string;
    errorCode: string;
    errorMessage: string;
  }
  ```

  - Axios 에러는 `AxiosError<ErrorResponse>`로 표현할 수 있으며 다음과 같이 타입 가드를 명시적으로 작성할 수 있습니다.

  ```ts
  function isServerError(error: unknown): error is AxiosError<ErrorResponse> {
    return axios.isAxiosError(error);
  }
  ```

  - 사용자 정의 타입 가드를 정의할 때는 타입 가드 함수의 return type으로 **parameterName is Type** 형태의 **타입명제** 를 정의해주는게 좋습니다. 이때 **prameterName**은 타입 가드 함수의 시그니처에 포함된 매개변수여야 합니다.

  ```ts
  const onClickDeleteHistoryButton = async (id: string) => {
    try {
      await axios.post("https://....", { id });

      alert("주문 내역이 삭제되었습니다.");
    } catch (error: unknown) {
      if (isServerError(e) && e.response && e.response.data.errorMessage) {
        // 서버 에러일 때의 처리임을 명시적으로 알 수 있다 setErrorMessage(e.response.data.errorMessage);
        return;
      }
      setErrorMessage(
        "일시적인 에러가 발생했습니다. 잠시 후 다시 시도해주세요"
      );
    }
  };
  ```

  - 타입 가드를 활용하면 위 코드처럼 서버 에러를 명시적으로 확인할 수 있습니다.

### 2. 에러 서브클래싱 하기

- 실제 요청을 처리할 때 단순한 서버 에러도 발생하지만 인증 정보 에러, 네트워크 에러, 타임아웃 에러 같은 다양한 에러가 발생하기도 합니다.
- 이를 더욱 명시적으로 표시하기 위해 서브클래싱을 활용할 수 있습니다.

  > 서브클래싱 Subclassing
  > 기존(상위 또는 부모) 클래스를 확장하여 새로운(하위 또는 자식) 클래스를 만드는 과정을 말합니다.
  > 새로운 클래스는 상위 클래스의 모든 속성과 메서드를 상속받아 사용할 수 있고 추가적인 속성과 메서드를 정의할 수도 있습니다.

- 서브클래싱을 활용하면 에러가 발생했을 때 코드상에서 어떤 에러인지를 바로 확인할 수 있습니다. 또한 에러 인스턴스가 무엇인지에 따라 에러 처리 방식을 다르게 구현할 수 있습니다.

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

  - Axios를 사용하고 있다면 조건에 따라 인터셉터에서 적합한 에러 객체를 전달할 수 있습니다.

  ```ts
  const httpErrorHandler = (
    error: AxiosError<ErrorResponse> | Error
  ): Promise<Error> => {
    let promiseError: Promise<Error>;

    if (axios.isAxiosError(error)) {
      if (Object.is(error.code, "ECONNABORTED")) {
        promiseError = Promise.reject(new TimeoutError());
      } else if (Object.is(error.message, "Network Error")) {
        promiseError = Promise.reject(new NetworkError());
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

  - 사용처에서는 아래와 같이 활용 가능합니다

  ```ts
  const alert = (meesage: string, { onClose }: { onClose?: () => void }) => {};

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
      alert("네트워크 연결이 원활하지 않습니다. 잠시 후 다시 시도해주세요.", {
        onClose: errorCallback?.onNetworkErrorCallback,
      });
    } else if (error instanceof OrderHttpError) {
      alert(error.message, params);
    } else if (error instanceof Error) {
      alert(error.message, params);
    } else {
      alert(defaultHttpErrorMessage, params);
    }

    const getOrderHistory = async (page: number): Promise<History> => {
      try {
        const { data } = await fetchOrderHistory({ page });
        const history = await JSON.parse(data);

        return history;
      } catch (error) {
        onActionError(error);
      }
    };
  };
  ```

  - 이처럼 에러를 서브클래싱해서 표현하면 명시적으로 에러 처리를 할 수 있습니다.
  - `error instanceof OrderHttpError`와 같이 작성된 타입 가드문을 통해 코드상에서 에러 핸들리에 대한 부분을 한눈에 볼 수 있습니다.

### 3. 인터셉터를 활용한 에러 처리

- Axios 같은 페칭 라이브러리는 인터셉터 기능을 제공합니다. 이를 사용하면 HTTP 에러에 일관된 로직을 적용할 수 있습니다.

  ```ts
  const httpErrorHandler = (
    error: AxiosError<ErrorResponse> | Error
  ): Promise<Error> => {
    (error) => {
      // 401 에러인 경우 로그인 페이지로 이동
      if (error.response && error.response.status === 401) {
        window.location.href = `${backOfficeAuthHost}/login?targetUrl=${window.location.href}`;
      }
      return Promise.reject(error);
    };
  };

  orderApiRequester.interceptors.response.use(
    (response: AxiosResponse) => response,
    httpErrorHandler
  );
  ```

### 4. 에러 바운더리를 활용한 에러 처리

### 5. 상태 관리 라이브러리에서의 에러 처리

### 6. react-query를 활용한 에러 처리

### 7. 그 밖의 에러 처리

- 레거시코드 혹은 대응할 수 없는 경우 200번 대의 성공 응답에 대한 에러처리가 필요한 상황이 생길 수 있습니다.

```ts
httpStatus: 200
{
  "status": "C20005", // 성공인 경우 success를 응답
  "message": "장바구니에 품절된 메뉴가 있습니다"
}
```

- 해당 에러를 처리하기 위해 요청 함수 내에서 조건문으로 status를 비교할 수 있습니다.

  ```ts
  const successHandler = (response: CreateOrderResponse) => {
    if (response.status === "SUCCESS") {
      // 성공 시 진행할 로직을 추가한다
      return;
    }
    throw new CustomError(response.status, response.message);
  };
  const createOrder = (data: CreateOrderData) => {
    try {
      const response = apiRequester.post("https://...", data);

      successHandler(response);
    } catch (error) {
      errorHandler(error);
    }
  };
  ```

  - 해당 방법을 사용하면 간단하게 커스텀 에러를 처리할 수 있습니다.
  - 또한 영향 범위가 각 요청에 대한 성공/실패 응답 처리 함수로 한정되어 관리하기 편리해집니다.
    - 그러나 이렇게 처리해야 하는 경우가 많을 때에는 매번 동일한 구문 구조를 추가해야 합니다.
  - 만약 커스텀 에러를 사용하고 있는 요청을 일괄적으로 처리하고 싶다면 Axios 등의 라이브러리 기능을 활용할 수 있습니다.

    - 특정 호스트에 대한 API requester를 별도로 선언하고 상태 코드로 비교 로직을 인터셉터에 추가할 수 있습니다.

      ```ts
      export const apiRequester: AxiosInstance = axios.create({
        baseURL: orderApiBaseUrl,
        ...defaultConfig,
      });

      export const httpSuccessHandler = (response: AxiosResponse) => {
        if (response.data.status !== "SUCCESS") {
          throw new CustomError(response?.data.message, response);
        }

        return response;
      };

      apiRequester.interceptors.response.use(
        httpSuccessHandler,
        httpErrorHandler
      );

      const createOrder = (data: CreateOrderData) => {
        try {
          const response = apiRequester.post("https://...", data);

          successHandler(response);
        } catch (error) {
          // status가 SUCCESS가 아닌 경우 에러로 전달된다
          errorHandler(error);
        }
      };
      ```

## 07.4 API 모킹

- 앞의 내용은 생략했습니다.
- 이슈가 생겼을 때 charles 등의 도구를 활용하면 응답 값을 그대로 복사하여 이슈 발생 상황을 재현하는 데 도움이 됩니다.
- 우아한형제들 프론트엔드에서는 `axios-mock-adapter`, `NextApiHandler` 등을 활용하여 API를 모킹해서 사용하고 있습니다.

> 요즘 테스팅을 간간히 작성중인데, 미들웨어때문에 서비스함수 테스팅이 어려운 상황입니다. 인증인가를 모킹으로 우회하기에는 아직 어렵네요 :(

### 1. JSON 파일 불러오기

- 간단한 조회만 필요한 경우에는 `*.json` 파일을 만들거나 자바스크립트 파일 안에 JSON 형식의 정보를 저장하고 export 해주는 방식을 사용하면 됩니다.
- 이후 GET 요청에 파일 경로를 삽입해주면 조회 응답으로 원하는 값을 받을 수 있습니다.

  ```ts
  // mock/service.ts
  const SERVICES: Service[] = [
    {
      id: 0,
      name: "배달의민족",
    },
    {
      id: 1,
      name: "만화경",
    },
  ];

  export default SERVICES;

  // api.ts
  const getServices = ApiRequester.get("/mock/service.ts");
  ```

### 2. NextApiHandler 활용하기

```ts
// api/mock/brand
import { NextApiHandler } from "next";

const BRANDS: Brand[] = [
  {
    id: 1,
    label: "배민스토어",
  },
  {
    id: 2,
    label: "비마트",
  },
];

const handler: NextApiHandler = (req, res) => {
  // request 유효성 검증
  res.json(BRANDS);
};

export default handler;
```

### 3. API 요청 핸들러에 분기 추가하기

- 요청 경로를 수정하지 않고 평소에 개발할 떄 필요한 경우에만 실제 요청을 보내고 그 외에는 목업을 사용하여 개발하고 싶다면 다음과 같이 처리할 수 있습니다.
- API 요청을 훅 또는 별도 함수로 선언해준 다음 조건에 따라 목업 함수를 내보내거나 실제 요청 함수를 내보낼 수 있습니다.

  ```ts
  const mockFetchBrands = (): Promise<FetchBrandsResponse> =>
    new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          status: "SUCCESS",
          message: null,
          data: [
            {
              id: 1,
              label: "배민스토어",
            },
            {
              id: 2,
              label: "비마트",
            },
          ],
        });
      }, 500);
    });

  const fetchBrands = () => {
    if (useMock) {
      return mockFetchBrands();
    }

    return requester.get("/brands");
  };
  ```

  - 해당 방법을 사용하면 개발이 완료된 이후에도 유지보수할 때 목업 함수를 사용할 수 있습니다.
  - 필요한 경우에만 실제 API에 요청을 보내고 평소에는 서버에 의존하지 않고 개발할 수 있게 됩니다.
  - 그러나 if문이 귀찬....

### 4. axios-mock-adapter로 모킹하기

- 서비스 함수에 분기문이 추가되는 것을 바라지 않는다면 라이브러리를 사용하면 됩니다.
- 해당 라이브러리는 Axios 요청을 가로채서 요청에 대한 응답 값을 대신 반환합니다.

  - 먼저 MockAdapter 객체를 생성하고, 해당 객체를 사용해서 모킹할 수 있습니다.
  - 앞선 두 방법과는 다르게 mock API의 주소가 필요하지 않습니다. 앞의 방법과 비슷하게 조회 요청에 대한 목업을 작성하면 다음과 같습니다.

  ```ts
  // mock/index.ts
  import axios from "axios";
  import MockAdapter from "axios-mock-adapter";
  import fetchOrderListSuccessResponse from "fetchOrderListSuccessResponse.json";

  interface MockResult {
    status?: number;
    delay?: number;
    use?: boolean;
  }

  const mock = new MockAdapter(axios, { onNoMatch: "passthrough" });

  export const fetchOrderListMock = () =>
    mock.onGet(/\/order\/list/).reply(200, fetchOrderListSuccessResponse);

  // fetchOrderListSuccessResponse.json
  {
    "data": [
      {
        "orderNo": "ORDER1234", "orderDate": "2022-02-02", "shop": {
        "shopNo": "SHOP1234",
        "name": "가게이름1234" },
        "deliveryStatus": "DELIVERY"
      },
    ]
  }
  ```

  - 단순히 응답 바디만 모킹할 수도 있지만 상태 코드, 응답 지연 시간 등을 추가로 설정할 수도 있습니다.

    ```ts
    export const lazyData = (
      status: number = Math.floor(Math.random() * 10) > 0 ? 200 : 200,
      successData: unknown = defaultSuccessData,
      failData: unknown = defaultFailData,
      time = Math.floor(Math.random() * 1000)
    ): Promise<any> =>
      new Promise((resolve) => {
        setTimeout(() => {
          resolve([status, status === 200 ? successData : failData]);
        }, time);
      });

    export const fetchOrderListMock = ({
      status = 200,
      time = 100,
      use = true,
    }: MockResult) =>
      use &&
      mock
        .onGet(/\/order\/list/)
        .reply(() =>
          lazyData(status, fetchOrderListSuccessResponse, undefined, time)
        );
    ```

    - 해당 라이브러리를 사용하면 GET뿐만 아니라 다른 HTTP메서드에 대한 목업을 작성할 수 있게 됩니다.
    - 또한 networkError, timeoutError등을 메서드로 제공하기 때문에 다음처럼 임의로 에러를 발생시킬 수도 있습니다.
      ```ts
      export const fetchOrderListMock = () =>
        mock.onPost("/order/list").networkError();
      ```

### 5. 목업 사용 여부 제어하기

```ts
const useMock = Object.is(REACT_APP_MOCK, "true");

const mockFn = ({ status = 200, time = 100, use = true }: MockResult) =>
  use &&
  mock.onGet(/\/order\/list/).reply(
    () =>
      new Promise((resolve) =>
        setTimeout(() => {
          resolve([
            status,
            status === 200 ? fetchOrderListSuccessResponse : undefined,
          ]);
        }, time)
      )
  );

if (useMock) {
  mockFn({ status: 200, time: 100, use: true });
}
```

- 특정 플래그를 통해 mockFn을 제어할 수 있습니다.
- 스크립트 실행 시 구분짓고자 한다면 package.json에서 설정 가능합니다.

  ```json
  // package.json

  {
    "scripts": {
      "start:mock": "REACT_APP_MOCK=true npm run start",
      "start": "REACT_APP_MOCK=false npm run start"
    }
  }
  ```

- 또는 config 파일을 별도로 구성하거나 프록시를 사용할 수도 있습니다.
- API 요청의 흐름을 파악하고 싶다면 react-query-devtools 같은 별도의 도구를 사용하면 됩니다.
- 목업을 사용할 때 네트워크 요청을 확인하고 싶을 때에는 네트워크에 보낸 요청을 변경해주는 Cypress같은 도구의 웹훅을 사용하면 됩니다.

### Axios등 fetch 라이브러리에 대한 우형 이야기를 정리해보았어요

#### 데이터 fetching 라이브러리를 사용하나요? 사용한다면 어떤 기준으로 선택했나요? 또 사용하고 나서 느낀 장단점은 어떤게 있나요?

- A: 관리하는 프로젝트에 비동기 요청이 많지 않아 사용하고 있지 않다. 필요한 경우 Recoil에서 제공하는 useSelector를 활용하여 데이터 페칭을 처리하고 있음.
- B: react-query를 사용중. "상태관리"를 하는 목적을 고려했을때, 서버에서 가져온 데이터 관리 용도로 클라이언트 상태를 관리하는 MobX나 Redux를 사용하는게 맞을까? 라는 의문이 생겨서 고민끝에 Redux를 걷어내게 됨.
