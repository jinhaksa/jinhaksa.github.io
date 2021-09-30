---
layout: post
title: NodeJS로 Reverse Proxy 구현하기
categories: [dev]
tags:
  - reverse_proxy
  - nodejs
  - micro_service
author: 심동근
email: sdg@jinhakapply.com
date: 2021-09-30
---

# 왜 직접 구현하지?

API Gateway를 NodeJS로 구현하려고 한다. NodeJS에서 찾아보니 [`express-gateway`](https://github.com/ExpressGateway/express-gateway)란 라이브러리가 가장 완성도 있었다. 그러나 Document에 정보가 부족하고 업데이트가 안된지 오래된 것으로 확인됬다. ~~Oh.. sit!~~ 그래서 직접만들기로 맘을 먹었다. 그리고 일단 Reverse Proxy 기능이 필요하기에 찾아보기로 한다.

# 사용되는 라이브러리

웹 서버를 위한 [`express`](https://github.com/expressjs/express)와 프록시를 위한 [`http-proxy-middleware`](https://github.com/chimurai/http-proxy-middleware)를 이용하기로 했다. 두 라이브러리 전부 사용자가 많고 최근에도 계속 업데이트 중이다. 이제 기본적인 기능은 갖춰진 상태로 바로 코드를 작성할 수 있다.

# API

## createProxyMiddleware([context,] config)

```javascript
const { createProxyMiddleware } = require("http-proxy-middleware");

const apiProxy = createProxyMiddleware("/api", {
  target: "http://www.example.org",
});
```

- **context**: Determine which requests should be proxied to the target host.
- **options.target**: target host to proxy to. _(protocol + host)_

## createProxyMiddleware(uri [, config])

```javascript
// shorthand syntax for the example above:
const apiProxy = createProxyMiddleware("http://www.example.org/api");
```

위 함수로 Middleware를 만들고 Router에 등록만 시키면 된다.

# 프록시 구현

localhost에서 네이버 뉴스 페이지를 띄우려 한다. 네이버 뉴스 기본 사이트의 링크는 `https://news.naver.com/` 이다.

```typescript
import express from "express";
import { createProxyMiddleware } from "http-proxy-middleware";

const app = express();

app.use(
  "/news",
  createProxyMiddleware({
    target: "https://news.naver.com",
    changeOrigin: true,
  })
);
app.listen(3000);
```

위 소스코드를 실행하고 `http://localhost:3000/news` 로 접속하면 에러 페이지가 나타난다.

![](/assets/img/posts/dev/2021-09-30-reverse-proxy-nodejs/1.png)

서버에서 프록시 요청을 날린 주소가 `https://news.naver.com`가 아니고 `https://news.naver.com/news`로 요청을 해서 오류 페이지를 띄우는 것이다.

> 이 부분을 통해 이 라이브러리는 메인 경로 이외의 경로를 proxy경로로 넘기는 것을 알 수 이다.

이 문제를 해결하기 위해서는? 라이브러리에 `pathRewrite`옵션이 있다. 이 옵션을 활용하면 해결할 수 있다.

```typescript
import express from "express";
import { createProxyMiddleware } from "http-proxy-middleware";

const app = express();

app.use(
  "/news",
  createProxyMiddleware({
    target: "https://news.naver.com",
    changeOrigin: true,
    // 옵션 추가
    pathRewrite: {
      "^/news": "/",
    },
  })
);

app.listen(3000);
```

`pathRewrite`옵션에 `/news` 경로를 `/`로 바꾸기만 하면 간단히 문제를 해결할 수 있다. 이제 실행하면 아래와 같이 훌륭한 결과를 얻을 수 있다 ^^

![](/assets/img/posts/dev/2021-09-30-reverse-proxy-nodejs/2.png)

# 결과적으로

결과적으로 기본적인 기능에 충실한 `http-proxy-middleware`를 알게 됬고 이제 API Gateway를 구현할 수 있게 되었다... 나중에는 TCP도 proxy가 필요할지 모른다고 생각된다. 목적지 IP 주소만 바꿔주면 되서 더 간단할 거 같긴하다. 아무튼 끝.
