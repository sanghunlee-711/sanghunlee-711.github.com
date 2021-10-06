---
title: Next JS를 공부해보자[2편-Auth]
author: Sanghun lee
date: 2021-09-28 11:33:00 +0800
categories: [Blogging, Next JS]
tags: [Next JS]
math: true
mermaid: true
image:
  src: https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcZDedj%2FbtraJzj53sn%2FHdUj1jQOUihHUy0oday6kK%2Fimg.png
  width: 850
  height: 585
---

# Next JS

# 사용하는 이유

1편을 통한 간단한 개념잡기 이후에 Next js, Apollo-client 조합으로 쿼리를 이용할 수 있는 훅(useQuery 등)을 활용하여 로그인 로그아웃을 구현해보자!

# 1. 세팅

일단 기본적으로 next-apollo 의 샘플 깃헙 코드를 통해 대부분의 세팅을 진행하였고 추가적으로 link를 설정해서 헤더만 잡아주었다.

물론 이걸 지금 활용을 거의 못하고 있긴한데.. 수정사항이나 더 좋은 것이 생기면 포스팅을 수정해봐야겠다.

일단 현재 진행중인 개인프로젝트에서 사용한 것이므로 간단한 앱의 구조부터 설명을 하면 좋을 것 같아 `\_app`의 코드 먼저보고 시작하겠다

```tsx
const App = ({ Component, pageProps }) => {
  const apolloClient = useApollo(pageProps);

  return (
    <ApolloProvider client={apolloClient}>
      <Wrapper>
        <Layout>
          <Component {...pageProps} />
        </Layout>
      </Wrapper>
    </ApolloProvider>
  );
};

export default wrapper.withRedux(App);
```

별도로 생성한 useApollo를 통해 생성한 클라이언트를 ApolloProvider로 넣어주고 Wrapper를 통해 전역 스타일을 넣어주었다

LayOut을 통해 Header, Footer를 모든 컴포넌트 내에서 동일하게 보여주는 방식을 취한다

리액트로 진행할때는 loggedInRouter, loggedOutRouter를 나눠서 진행했던 라우팅방시에서 조금 차이를 둬야하는 부분이다.
그래서 useMe라는 별도의 훅을 만들어 쿼리를 리턴시키는 용도로 사용하고있다.

```tsx
import { useLazyQuery, useQuery } from "@apollo/client";
import { ME_QUERY } from "../graphql/queries";
import { meQuery } from "../src/__generated__/meQuery";

export const useLazyMe = () => {
  return useLazyQuery<meQuery>(ME_QUERY, {
    context: {
      headers: {
        "folks-token":
          typeof window !== "undefined"
            ? localStorage.getItem("folks-token")
            : "",
      },
    },
    nextFetchPolicy: "network-only",
    fetchPolicy: "network-only",
  });
};

export const useMe = () => {
  return useQuery<meQuery>(ME_QUERY);
};
```

사실 코드만 보면 진짜 별 것 없이 간단한 코드인데 next js에 별도로 apollo가 얹어있는 세팅을 진행하다보니 몇가지 문제가 발생했었다.
아폴로서버가 최상단에 존재하므로 next js 기반에서 앱이 최초 빌드될때 또는 refresh가 일어나서 처음부터 렌더가 되는 시점을 제외하고는
별도로 apollo를 위해 세팅해준 값이 변경되지 않는 문제가 있었다.

아래는 세팅을 위한 일부 설정이었다.

```ts
const token =
  typeof window !== "undefined" ? localStorage.getItem("folks-token") : "";

export const isLoggedInVar = makeVar(Boolean(token));
export const authTokenVar = makeVar(token); //요주의 인물

const httpLink = createHttpLink({
  uri: "http://localhost:4000/graphql",
  headers: {
    "folks-token": authTokenVar() || "",
  },
});

function createApolloClient() {
  return new ApolloClient({
    ssrMode: typeof window === "undefined",
    link: httpLink,
    cache: new InMemoryCache({
      typePolicies: {
        Query: {
          fields: {
            token: {
              read() {
                return authTokenVar();
              },
            },
            isLoggedIn: {
              read() {
                return isLoggedInVar();
              },
            },
            allPosts: concatPagination(),
          },
        },
      },
    }),
  });
}
```

세팅을 조금 더 설명하자면 useReactiveVar는 typePolicies에 미리 설정해준 fields의 값에 해당하는 값이 들어간 query들을 다시 한번 refresh할 용도로 사용하려했다.
일단 여기까지는 작동을하였고 헤더에서 로그인이 되지않은 경우 로그인 버튼을, 로그인 된경우 로그인 된 유저의 프로필사진을 보여주는 기능을 구현하는 것에 문제가 없었다.
문제는 유저의 프로필 사진이 나타나는 시점에 localStorage에 존재하는(이미 로그인 한 경우)

1. 로그아웃 후 다른 아이디로 로그인시 토큰은 변경되었으나 Header의 프로필은 그전의 아이디의 값이 존재했음
2. 그래서 next js의 캐시 인지 apollo의 캐시인지 확인이 필요했음
3. Next js는 애초에 SSR이라 캐시가 크게 의미가 없음(확실?)
4. 중간에 redux toolkit을 다른 이유로 설치하고 편하게 관리하기 위해 user를 별도로 글로벌 State로 활용한것이 설마 문제가 있을까 하여 다 걷어냄
5. 걷어내고 보니 확실히 캐시문제인게 개발자도구를 보면 혼자서 업데이트가 안되 어있음
6. client.resetStore()를 하라는 공식문서 따라 해보려했으나 next js에서는 아폴로를 인스턴스로 가져와서 사용해야하기 때문에 쿼리가 그냥 다깨짐
   그래서 작동이 안되서 멍때림
7. 애초에 clearStore => resetStore로직이 편하게 먹질 못함
8. readQeury를 사용해 로그아웃일때 writeQuery를 사용해서 해당 쿼리의 유저값을 비워주려했음 -> 적절하게 사용할 곳이 아님 -> 공식문서 더 봐야함
   찾아보니 wrtieFragment와 readQuery의 조합으로 쓰면 될 것 같음 -> 진행중 ..
9. 일단 캐시의 데이터를 건드는건 왜인지 작동이 안됨
10. 조금 더 살펴보니 로그인 직 후 profile에서 변경된 makeVar로 만든 토큰값은 변경되어있지만 apollo client는 이미 세팅된 직후라 변경되지 않음

## 1.2. Static Generation vs Server Side Rendering

Next js에서는 아래 두 컨셉을 페이지별로 필요에따라 마음대로 선택하여 사용할 수 있다.

## 1.2.1 Static Generation

- HTML을 빌드타임에 미리 렌더링시키는것(pre-rendering)
- 미리 렌더된 HTML은 각 요청마다 재사용되게 된다.
- 개발모드에서는 Static Generation이라 하더라도 Server-side Rendering와 같이 각 요청(Request)마다 새롭게 생성
- 유저의 요청전에 필요한 페이지인 경우

  - 마케팅 페이지
  - 블로그 게시물(이래서 쓴다)
  - E-commerce 제품 리스트
  - 문서나 문서

- 데이터를 미리 받아오는 등의 전처리가 필요한 경우 `getStaticProps` 라는 함수 안에서 필요한 로직을 실행하면 된다.

## 1.2.2 Server-side Rendering

- 각 요청마다 새로운 HTML을 생성해준다
- 별도 설정없이은 CDN캐시가 되지 않고 매 요청마다 서버에서 새로 계산을 해야하므로 시간이 `getStaticProps`에 비해 더 걸리게 된다.
- 매 요청에(request time)에 데이터를 가져와야 하는경우(빌드타임이아닌고 렌더 되기 전에 이미 완성이 되어있어야 하는 경우) `getServerSideProps` 를 사용하면 된다.
  > When you need to fetch data at request time instead of build time

> You should use getServerSideProps only if you need to pre-render a page whose data must be fetched at request time. Time to first byte (TTFB) will be slower than getStaticProps because the server must compute the result on every request, and the result cannot be cached by a CDN without extra configuration.

## 1.2.3 Client-side Rendering

- 데이터를 pre-rendering할 필요가 없고 정적으로 미리 만들어진 사이트가 외부데이터를 필요로 하지 않을때
- 정적페이지가 로딩 된 다음 외부 데이터를 client의 js사용으로 나머지 부분을 채우게 만드는 방법

- 게시판 페이지등에 활용
- dashboard는 유저의 개인 공간이기 때문에 SEO(검색엔진 최적화)가 필요가 없으므로 pre-render가 필요하지 않음
- 데이터가 수시로 업데이트되는 경우 활용하기

## 1.3 SWR

- Client side에서 데이터를 받아오기 위해 사용하는 Hook임
  > It handles caching, revalidation, focus tracking, refetching on interval, and more.

# 2. 활용

## 2.1 Static Generation

Static Generation의 샘플이다.

getStaticProps를 적용하고 간단하게 만든 REST API를 통해 데이터를 미리 받아온다.
HTML을 빌드타임에 렌더링시킨 뒤 보내게 된다.

재미있는 부분하나 utilStyle.headingMd등 약간 모듈같이 css를 활용할 수 있게 된 점이다.

```tsx
import axios, { AxiosResponse } from "axios";
import Head from "next/head";
import Link from "next/link";
import Layout, { siteTitle } from "../components/layout";
import utilStyles from "../styles/utils.module.css";
// 예시로 필요한 데이터를 보내기 위해 간단한 RESTAPI를 만들어 사용
interface DataTest {
  date: string;
  title: string;
  content: string;
}
interface TestData {
  returnData: DataTest[];
}

const Home = ({ returnData }: TestData) => {
  return (
    <Layout home={true}>
      <Head>
        <title>{siteTitle}</title>
      </Head>
      <Link href="/posts/first-post">
        <a>To go first Post</a>
      </Link>
      <section className={utilStyles.headingMd}>
        <p>Hello There</p>
        <p>This is Sample Website</p>
        <p>
          <a href="https://nextjs.org/learn">our Next.js tutorial</a>.
        </p>
      </section>
      <section className={utilStyles.headingMd}>
        <ul>
          {returnData?.map((el) => (
            <li key={el.date}>
              {el.date} <br />
              {el.title} <br />
              {el.content}
            </li>
          ))}
        </ul>
      </section>
    </Layout>
  );
};

//https://nextjs.org/learn/basics/data-fetching/getstaticprops-details
export async function getStaticProps() {
  const { data }: AxiosResponse<{ data: TestData }> = await axios.get(
    "http://localhost:4000/images/test"
  );

  const returnData = data.data;

  return {
    props: {
      returnData,
    },
  };
}

export default Home;
```

> getStaticProps 예시

## 2.2 Server-side Rendering

```tsx
import axios, { AxiosResponse } from "axios";
import { GetServerSideProps, InferGetServerSidePropsType } from "next";
import Head from "next/head";
import Link from "next/link";
import Layout from "../../components/layout";
import utilStyles from "../../styles/utils.module.css";

interface DataTest {
  date: string;
  title: string;
  content: string;
}
interface TestData {
  data: DataTest[];
}

const FirstPost = ({
  data,
}: InferGetServerSidePropsType<typeof getServerSideProps>) => {
  return (
    <Layout home={false}>
      <Head>
        <title>First Post!</title>
      </Head>
      <h1>First Post!</h1>
      <h2>
        <Link href="/">
          <a> Back to HOme</a>
        </Link>
      </h2>
      <section className={utilStyles.headingMd}>
        <ul>
          {data?.map((el: DataTest) => (
            <li key={el.date}>
              {el.date} <br />
              {el.title} <br />
              {el.content}
            </li>
          ))}
        </ul>
      </section>
    </Layout>
  );
};

export const getServerSideProps: GetServerSideProps = async (context) => {
  const res: AxiosResponse<{ data: TestData }> = await axios.get(
    "http://localhost:4000/images/test"
  );
  const data = res.data.data;

  if (!data) {
    return {
      redirect: {
        destination: "/",
        permanent: false,
      },
    };
  }

  return {
    props: { data },
  };
};

export default FirstPost;
```

# 3. 결론

기본적으로 필요한 CSS, REST-API 받는방식 및 관련 개념을 정리해보았다.

SWR같은경우는 세팅을하고 컴포넌트를 작성하면서 다시한번 샘플을 해당 포스팅에 남겨보아야겠다.

이제 다음편에 해야할 것은 [이 링크](https://github.com/vercel/next.js/tree/canary/examples)를 통해 apollo-client를 연결하여

graphql/ apollo/ apollo-client을 세팅하는 방법을 공부하며 포스팅해보겠다

## 참고

- [Fetching Data at Request Time](https://nextjs.org/learn/basics/data-fetching/request-time)
- [getStaticProps Details](https://nextjs.org/learn/basics/data-fetching/getstaticprops-details)
