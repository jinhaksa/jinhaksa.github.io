---
layout: post
title: asp.net legacy system에 vuejs & nodejs system 적용하기
categories: [dev]
author: 노혜진
email: hjroh@jinhakapply.com
date: 2021-10-29
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

신규 서비스만 별도 BackOffice로 구성할 수도 있다. 그렇게 되면 사용자 권한 관리 및 메뉴관리, 로그관리 등이 통합되지 못하게 된다.
물론 공통 기능을 사용하는 DB는 동일한 것을 사용할 수 있지만 같은 기능을 위한 소스가 2곳에 있다는 것은 피할수 없는 상황이다.

결론은 하나의 사이트에 다양한 개발환경에서 구성한 비지니스 컴포넌트를 조합하는 Microservice Architecture 적용하는 것이다.
우리는 기존 Legacy 시스템에 영향을 주지 않고 공통적인 기능은 그대로 활용하면서 기존의 구성에서 신규 메뉴에 추가하되 실행되는 페이지 각각은 신규 컴포넌트로 작동하기로 하였다.
이렇게 될 경우 사용자는 BackOffice를 사용하면서도 두개의 시스템이 결합되어 있다는 것을 알지 못한다.

기존 legacy 시스템에 새로운 개발과 운영 플랫폼을 적용하려는 경우가 있을 경우 이 자료가 도움이 되었으면 한다.

## 구성도

운영자의 불편이 관리고민이 생겼독립적인 형태의 사이트 구축하는 경우와 달리 내부 운영 직원들이 사용하는 관리자용 사이트는 기존 운영중인 windows asp.net 레거시 시스템에 중에는 를 확장하는 과정에서
