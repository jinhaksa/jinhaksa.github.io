---
layout: post
title: 바인드 파라미터
categories: [dev]
author: 이태환
email: thlee@jinhak.com
date: 2022-04-01
tag:
  - sequelize
  - query
  - bind-variable
---

<!-- @format -->

# 바인드 파라미터

## 들어가기 앞서

진학사 개발팀에서 있었을 땐 프로시저를 사용하였는데, 프로시저를 호출하기 전 파라미터값들의 특정문자(SELECT, AND, OR)등을 체크하여 해당 특정문자가 포함돼있으면 프로시저를 호출하기 전에 막아서 SQL Injection을 방어하였습니다.

이러한 환경이 익숙하여 현재 전략프로젝트팀에서 미니 프로젝트를 개발하였을때 보안에 신경을 쓰지않고 개발을 하게됐습니다.

그래서 아이디와 비밀번호를 매칭하는 쿼리를 짜서 로그인을 개발하였는데 SQL Injection에 당해 아이디와 비밀번호가 매칭이 되지않아도 로그인되는 취약점이 발생하였습니다.

## 기존 로그인 방식

```javascript
const user = await pool.query(
  "SELECT * " +
    "FROM app_member " +
    "WHERE member_id = '" +
    userId +
    "' AND pwd = '" +
    crypto.createHash("sha256").update(password).digest("base64") +
    "' "
);
```

위는 기존에 작성한 쿼리인데 userId라는 변수에 ' OR '1'='1' -- 이라는 값이 들어오면
![Untitled](/assets/img/posts/dev/2022-04-01-bind-variable/query1.png)

서버에서 위와 같은 쿼리가 실행되어 OR 로 인하여 어떠한값이든 참이되고 뒤에 쿼리는 주석처리가 되어 해당 쿼리가 실행이 돼서 SQL Injection에 취약한 문제가 생겼습니다. 이러한 문제를 해결하기위해 바인드 변수를 사용하여 다시 개발하였습니다.
시퀄라이즈(Sequelize)라는 DB 작업을 쉽게 할 수 있도록 도와주는 라이브러리를 사용하였습니다.

## 개선 로그인 방식

```javascript
const user = await sequelize.query(
  `SELECT * FROM app_member WHERE member_id = :userId AND pwd = :pwd`,
  {
    bind: {
      userId: userId,
      pwd: crypto.createHash("sha256").update(password).digest("base64"),
    },
  }
);
```

위는 시퀄라이즈를 사용하여 수정한 쿼리입니다.
![Untitled](/assets/img/posts/dev/2022-04-01-bind-variable/query2.png)

서버에서 위와 같은 쿼리가 실행이 되는데 ' OR '1'='1' -- 부분이 ' ' 으로 한번더 감싸져 문자열 취급이 되어 SQL Injection을 막을 수 있게 되었습니다.

## 바인드 변수에 또 다른 장점

<br>
바인드변수에 장점은 SQL Injection을 막을수 있는 방법뿐만 아니라 또 다른 장점이 있습니다.

SQL에서 쿼리를 받으면 처리하는 절차는

1. 구문 오류 체크
2. 공유 영역에서 해당 구문 검색
3. 권한 체크
4. 실행 계획 수립
5. 실행 계획 공유영역에 저장
6. 쿼리 실행

이 순서대로 실행이 됩니다.
만약 2번 단계에서 해당 구문이 동일한 구문을 찾게 되면, 6번 쿼리 실행으로 건너뜁니다. 1~6번까지 모두 실행이 되면 Hard Parsing / 2번에서 6번으로 건너뛰어서 실행이되면 Soft Parsing이라고 합니다. 당연히 절차적으로 적은 Soft Parsing이 Hard Parsing 보다 성능적으로 우세합니다.

만약 맨처음 방식처럼 바인드 변수를 사용하지않고 쿼리를 작성하면

```query
SELECT * FROM app_member WHERE member_id = 'test' AND pwd = 'test'
```

이렇게 변수값에 데이터들이 바인딩 된 상태로 공유 영역에 저장이 되어 똑같은 회원이 로그인 하지 않는 이상 매번 1~6번의 실행 절차를 걸쳐 성능적으로 안좋아 지게됩니다.

하지만 바인드 변수를 사용하여 쿼리를 작성하면

```query
SELECT * FROM app_member WHERE member_id = ? AND pwd = ?
```

이러한 형식으로 공유 영역에 저장이 되어 어떠한 회원이 로그인을 해도 위에 구문이 공유 영역에 저장이 되어 2번에서 6번으로 바로 실행이 되어 성능적으로 우세해집니다.

이상... 기본적이지만 보안, 성능적으로 중요한 바인드변수에 대해 알아봤습니다.
