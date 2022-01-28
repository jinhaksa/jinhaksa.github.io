---
layout: post
title: asp.net legacy system에 vuejs & nodejs system 적용하기(1탄)
categories: [dev]
author: 노혜진
email: hjroh@jinhakapply.com
date: 2021-10-29 / 2022-01-28
tag:
  - asp.net
  - .net
  - IIS
  - legacy integration with javascript open source frameworks
  - nodejs
  - vue.js
---

## 개요

새로운 서비스를 위해 javascript 기반 Open Source Framework을 사용하여 웹사이트를 구축하기로 하였다.
서비스를 새로 만들게 되면 으례 내부 운영 관리할 BackOffice도 같이 작업을 하게 되는데 여기에 문제가 있었다.

기존 BackOffice 개발 환경이 우리가 사용하려고 하는 개발 및 운영 환경과 매우 상이하다는 것이다.

```
###Legacy 환경:
  - OS: Windows
  - Web Server: IIS
  - develop language: ASP.NET, C#, jquery
  - DB: MSSQL
  - SSR
###신규 환경:
  - OS: LINUX
  - API Server: Node.js Express.js
  - develop language:  Vue.js, javascript
  - DB: Postgresql,
  - Search Engine: ElasticSearch
  - SPA
  - Deployment: docker
```

신규 서비스만 별도 BackOffice로 구성할 수도 있다. 그렇게 되면 로그인, 사용자 권한 관리 및 메뉴관리, 로그관리 등이 통합되지 못하게 된다.
개발 기간 및 사용자의 편의성을 고려하여 아래의 기준에 맞추어 작업을 진행하였다.

```
- 하나의 사이트로 운영
- 사용자는 한번만 로그인(기존의 asp.NET Forms 인증을 그대로 사용)한다.
- 사용자 메뉴별 기능별 권한 확인도 .NET에서 처리
- 웹로그 및 보안 검토를 위한 개인정보 접근 로그도 .NET 에서 기존 위치에 저장
- 화면 메뉴구성은 .NET에서 제어한다.
```

결론은 하나의 사이트에 다양한 개발환경에서 구성한 비지니스 컴포넌트를 조합하는 Microservice Architecture 적용하는 것이다.
우리는 기존 Legacy 시스템에 영향을 주지 않고 공통적인 비지니스 컴포넌트 기능은 그대로 활용하면서 기존의 구성에서 신규 메뉴에 추가하되 실행되는 페이지를 신규 컴포넌트로 작동하기로 하였다.
이렇게 될 경우 사용자는 BackOffice를 사용하면서도 두개의 시스템이 결합되어 있다는 것을 알지 못한다.

기존 legacy 시스템에 새로운 개발과 운영 플랫폼을 적용하려는 경우가 있을 경우 이 자료가 도움이 되었으면 한다.

## 화면 구성

Header와 메뉴 부분은 asp.net 소스로 구성하고 메뉴 선택에 따른 기능 페이지는 Vue로 구현한 javascript로 구성한다.

![deploy_screen.png](/assets/img/posts/dev/2021-10-29-deploy-legacy/deploy_screen.png)

## 구성도

legacy인 aspx페이지는 asp.NET WebForm 으로 http endpoint를 명확하게 지정하여 해당 페이지를 찾아갑니다.

vue.js로 작업해 build된 js 파일은 endpoint 경로에 상관없이 동일한 파일을 다운 받습니다.
이를 위해 새로 vue로 개발한 소스는 배포시 하위 디렉토리(응용프로그램 경로)를 /vue(예시) 와 같은 고정된 경로를 통하는 것으로 결정을 하였습니다.
/vue/~~~ 로 요청된 request는 동일한 하나의 페이지를 다운받습니다.
사용자 url 라우팅에 따라 해당 메뉴 페이지들은 다운받은 VUE 소스 내에서 SPA 방식으로 동적으로 화면을 구성하게 됩니다.

vue로 작업된 소스에서 데이터를 필요로 할 경우 client에서 api를 호출하는데 이 api에 대해서도 asp.NET에서 권한을 체크한 후에 client 정보를 header에 추가하여 request를 node로 구축한 api서버로 포워딩합니다.
api서버에서 처리된 결과는 json으로 데이터를 회신하고 asp.NET은 client에 회신된 결과를 전달합니다.

endpoint가 다른 것들에 대해 하나의 페이지로 회신하는 방식은 기존 .NET Webform 방식으로는 구현을 할 수 없는 방법입니다.
제어를 하기 위해 controller와 View가 필요합니다. ASP.NET MVC Core를 추가 적용해야 가능합니다.

![deploy_architecture.png](/assets/img/posts/dev/2021-10-29-deploy-legacy/deploy_architecture.png)
