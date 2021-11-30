---
layout: post
title: Babel 7 transpiling
categories: [dev]
author: 김동현
email: kdh@jinhakapply.com
date: 2021-11-30
tag:
  - javascript
  - babel
  - frontend
---

웹 프론트엔드 개발에서 중요하게 고려해야 할 점으로 디자인, 성능, 사용자 편의성, 입력 폼 검증 등 여러가지가 있지만, 사용자의 환경이 제각각이라는 점을 간과하기 쉽습니다. W3C에서 웹 표준을 제정하고 모던 브라우저들은 이를 최대한 지키려 하고 있지만 구형 브라우저, OS에 따라 특수하게 사용되는 브라우저, 그리고 브라우저 버전별로 지원되는 자바스크립트 기능에 한계가 있습니다.

![Promise 브라우저 목록(출처:[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise#browser_compatibility))](/assets/img/posts/dev/2021-11-30-babel7-transpiling/babel1.jpg)

Promise 브라우저 목록(출처:[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise#browser_compatibility))

이를 해결하기 위한 도구가 바로 Babel입니다. Babel은 개발자가 작성한 자바스크립트 코드를 특정 스펙이나 프리셋에 맞게 변환시켜주는 transpiler입니다(Babel 공식 홈페이지에는 javascript compiler라 소개하고 있습니다). 브라우저간 호환성을 위해 bundler인 Webpack과 함께 널리 사용되고 있습니다.

```jsx
// Babel Input: ES2015 arrow function
[1, 2, 3].map((n) => n + 1);

// Babel Output: ES5 equivalent
[1, 2, 3].map(function (n) {
  return n + 1;
});
```

코드가 변환되기 위해서는 특정 문법을 레거시 문법으로 변환시켜줄 **플러그인**이 필요합니다. 많은 플러그인을 일일이 지정해주기는 번거롭기도 하고 지저분하기 때문에 Babel은 자주 사용하는 플러그인과 설정들을 묶어 **preset** 형태로 제공하고 있습니다. 일반적으로 사용되는 preset은 [@babel/preset-env](https://babeljs.io/docs/en/babel-preset-env)가 있습니다.

플러그인이 있어도 변환 타겟 브라우저에 빠져있는 기능을 추가시켜줄 **polyfill**이란 것이 없으면 제대로 작동하지 않습니다. Promise, Array.prototype.includes 등이 존재하지 않을 경우 이를 추가해주는 것이 바로 polyfill입니다. Polyfill은 Babel에서 제공하는 것이 아닌 third-party 라이브러리인 [core-js](https://github.com/zloirock/core-js)가 사용됩니다.

Babel 6에서 버전 7으로 넘어오면서 몇가지 변화가 있었습니다. 주목해야 할 점은 config 파일이 .babelrc에서 babel.config.json으로 바뀌면서 .babelrc에서 발생하던 문제(node_modules과의 충돌)가 해결되었고, @babel/polyfill preset이 deprecated 되어 @babel/preset-env와 core-js가 이를 대체하게 되었습니다.

앞서 언급한 @babel/polyfill에는 기본적으로 core-js와 regenerator-runtime이 포함되어 있었습니다. Regenerator-runtime은 페이스북에서 만든, generator function에 대한 polyfill입니다. 그러나 전역 변수가 오염되고 deprecated 되었기 때문에 대신 core-js와 함께 @babel/preset-env를 사용해야 합니다(필요시 regenerator-runtime의 대체재인 @babel/plugin-transform-regenerator을 사용).

Core-js 사용시 전역 스코프를 오염시키지 않으려면 @babel/runtime과 core-js-pure를 사용해야 합니다. 예제에서는 preset-env만 적용하도록 하겠습니다.

npm init으로 프로젝트를 생성하고, babel과 core-js를 설치해줍니다. 컴파일러이기 때문에 실행 전 빌드되므로 production 모드에 쓸데없이 포함되지 않도록 --save-dev 옵션으로 설치합니다.

```bash
npm install --save-dev @babel/core @babel/cli @babel/preset-env
npm install --save core-js@3.19.2
```

루트 폴더에 babel.config.json 설정 파일을 생성해줍니다(7.8.0 이상 버전. 그 이하는 js 파일 생성).

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": {
          "edge": "17",
          "firefox": "60",
          "chrome": "67",
          "safari": "11.1",
          "ie": "11"
        },
        "useBuiltIns": "usage",
        "corejs": "3.19.2"
      }
    ]
  ]
}
```

presets은 array 타입으로 선언하고, 각 preset에 옵션을 주고 싶으면 중첩 배열 두 번째 인자에 옵션 object를 명시합니다.

여기서는 @babel/preset-env에 targets, useBuiltIns, corejs 옵션을 주었습니다.

target은 컴파일된 코드가 어느 브라우저까지 지원할지 설정하는 부분입니다. 아래와 같이 [browserslist 방식](https://github.com/browserslist/browserslist)으로 선언하거나 직접 브라우저별 버전을 명시할 수 있습니다.

```json
{
  "targets": {
    "chrome": "58",
    "ie": "11"
  }
}
```

corejs는 사용할 core-js 버전을 명시하는 부분입니다. corejs: 3과 같이 선언할 수도 있지만, 그럴 경우 3.0으로 해석되어 마이너 버전 업데이트는 반영이 되지 않습니다.

useBuiltIns 옵션은 false, "usage", "entry" 세 종류가 있습니다. "entry"는 targets에서 지정한 환경에 맞는 모듈들만 import 시키고, "usage"는 코드에서 직접 사용된 부분들에 대해서만 import 합니다.

```jsx
const es6Promise = new Promise((resolve, reject) => {
  try {
    setTimeout(() => {
      alert("world!");
      resolve();
    }, 2000);
  } catch (e) {
    reject(e);
  }
});

alert("Hello....");
es6Promise();
```

위의 샘플 파일은 IE11에서 작동하지 않는 es6 문법인 Promise, 그리고 arrow function을 가지고 있습니다.

```bash
babel src --out-dir dist
```

명령어를 실행하면 dist 폴더에 동일한 이름으로 컴파일된 코드가 생성됩니다.

```bash
"use strict";

require("core-js/modules/es.object.to-string.js");

require("core-js/modules/es.promise.js");

var es6Promise = new Promise(function (resolve, reject) {
  try {
    setTimeout(function () {
      alert("world!");
      resolve();
    }, 2000);
  } catch (e) {
    reject(e);
  }
});
alert("Hello....");
es6Promise();
```

core-js polyfill을 import하고 arrow function, const가 변환된 것을 볼 수 있습니다.

require는 브라우저에서 동작하지 않기 때문에 ES module로 변환해야 하지만, 이번 예제에서는 Webpack을 사용하지 않기 때문에 간단히 [Browserify](https://browserify.org/)와 [http-server](https://www.npmjs.com/package/http-server)를 이용하여 실행하겠습니다.

![컴파일 후 http-server에 적재](/assets/img/posts/dev/2021-11-30-babel7-transpiling/babel2.jpg)

컴파일 후 http-server에 적재

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Test</title>
  </head>
  <body>
    <script src="bundle.js"></script>
  </body>
</html>
```

npm run babel 명령어 실행과 함께 http://localhost:8080으로 접속하면 IE11에서 정상적으로 경고창이 시간차를 두고 뜨는 것을 확인할 수 있습니다.

![babel3.jpg](/assets/img/posts/dev/2021-11-30-babel7-transpiling/babel3.jpg)
