---
layout: post
title: 마이크로서비스와 SAGA 패턴
categories: [dev]
author: 김동현
email: kdh@jinhakapply.com
date: 2022-01-28
tag:
  - microservice
  - saga
---

### 모놀리식 vs 마이크로서비스

기존의 프로젝트들은 단일 애플리케이션 위에 각 기능이 켜켜히 쌓아 올려지는 모놀리식(Monolithic) 아키텍처가 대부분이었습니다. 초기에 구조를 잡기에도 편리하고 하나의 구조 위에서 설정과 변경이 가능하기 때문에 테스트하기 용이하다는 장점을 가지고 있습니다.

그러나 프로젝트가 점점 비대해지면서 수많은 문제점들이 드러나게 되는데, 추가와 변경이 반복되면서 점점 복잡해지는 로직을 파악하기가 어려워지고, 빌드 시간도 길어지게 됩니다. 코드 한 줄 수정하는 데도 전체를 다시 빌드해야 하기 때문에 비효율적이고, 각 기능별 버전 관리 또한 통합되기 어려워 프로덕션 배포 시 어느 부분을 빼고 어느 부분을 더해야 하는지 확인하는 것 또한 문제가 됩니다.

이런 점들을 보완하기 위해 도입된 것이 애플리케이션을 여러 단위로 쪼개는 마이크로서비스(Microservices) 입니다.

![출처: [https://microservices.io/index.html](https://microservices.io/index.html)](/assets/img/posts/dev/2022-01-28-microservice-saga-pattern/p1.png)

출처: [https://microservices.io/index.html](https://microservices.io/index.html)

쇼핑몰을 구축한다고 하면 위 그림과 같이 프론트, 계정, 보유 상품, 배송 서비스 등으로 각각의 애플리케이션을 구축해서 일정한 양식을 통해(주로 REST API를 통한) 각 애플리케이션 간 통신을 하여 유기적으로 동작하게 됩니다.

개발 조직들은 각각의 서비스를 맡아서 따로 개발하고, 배포도 단위별로 할 수 있으며, 각 애플리케이션 별로 별도의 개발 스택을 사용할 수도 있기 때문에 유연한 구축, 변경, 분리가 가능해집니다.

그러나 마이크로서비스라고 장점만 있는 것은 아닙니다. 마이크로서비스는 모놀리식 아키텍처와 반대의 장단점을 가지고 있다고 볼 수 있습니다. 분리되어 있기 때문에 통합된 테스트를 하기가 어렵고, 배포할 때 각 서비스 간 호환 버전이 맞아야 하기 때문에 조율이 필요합니다. 또한 적절하게 서비스를 분리하기 힘들고, 마이크로서비스 또한 규모가 커지게 되면 각 서비스의 연결 관계와 동작이 어떻게 이루어지는지 파악하기가 힘들어 관리 포인트가 증가할 수 있습니다.

이런 단점에도 불구하고 마이크로서비스가 주는 이점이 크기 때문에 단점을 최소화 하기 위한 여러 가지 아키텍처 패턴들이 고안되어 왔습니다.

오늘은 그 중에서 데이터 정합성(Consistency)을 위한 패턴인 SAGA에 대해 알아보고자 합니다.

### SAGA Pattern

서비스들을 분리하면 각 서비스가 동일한 데이터베이스를 사용할 수도 있고(용도와 권한 관리 필요성에 따라 스키마 분리, 또는 테이블을 따로 사용할 수도 있습니다), 서비스마다 데이터베이스를 각각 사용할 수도 있습니다.

이 때 복수의 데이터베이스를 처리해야 하는 경우 트랜잭션(transaction) 처리에 곤란한 상황이 발생하게 됩니다.

![Untitled](/assets/img/posts/dev/2022-01-28-microservice-saga-pattern/p2.png)

사용자가 물건을 구매했는데 Purchase DB에 반영이 안된 경우 하나의 DB였다면 단순히 트랜잭션 롤백을 통해 없던 일로 만들면 됩니다. 그러나 위처럼 Inventory DB에는 갱신이 되고 Purchase만 안된 경우 구매는 안했으나 물건은 들어와 있는 경우가 발생하게 됩니다. 정합성을 유지하기 위해서는 실패시 다른 한 쪽의 데이터를 복구하고 그 처리 결과를 User에 적용해야 합니다(보상 트랜잭션: Compensating transaction).

이런 경우에 분산 트랜잭션(2단계 커밋, 2PC)을 이용할 수도 있지만, 지원하지 않거나 모든 자원이 항상 가용하지는 않은 경우 사용하기 힘듭니다.

Saga 패턴은 메시지나 이벤트 큐를 이용하여 일련의 로컬 트랜잭션이 연속적으로 이루어질 수 있도록 설계한 패턴입니다.

### Choreography-based saga

Saga 패턴은 크게 두 가지 종류로 구분할 수 있습니다. Choreography saga와 Orchestration sage입니다. 비유하자면 전자는 지방자치형, 후자는 중앙집권형 입니다.

Choreography saga는 필요한 서비스들 끼리 이벤트 리스닝을 통해 정보를 주고받습니다.

![출처: [https://microservices.io/patterns/data/saga.html](https://microservices.io/patterns/data/saga.html)](/assets/img/posts/dev/2022-01-28-microservice-saga-pattern/p3.png)

출처: [https://microservices.io/patterns/data/saga.html](https://microservices.io/patterns/data/saga.html)

Order Service에 주문이 들어오면 Order events channel에 해당 주문건이 push 되고, 이를 구독하고 있는 Customer Service에서 해당 주문에 대해 처리할 수 있는지 여부를 잔고 DB를 조회한 뒤 결제가 가능하다면 해당 금액을 차감하고 Customer events channel을 통해 다시 Order Service로 반환합니다. 상품 금액보다 잔고가 적으면 결제 거부 처리를 할 것입니다(로직은 차이가 있을 수 있습니다).

만약 이 요청이 모종의 이유(서버 장애 등으로 인한)로 실패할 경우에도 Order events channel을 거쳐 차감한 금액을 다시 잔고에 추가하라는 요청을 보내야 됩니다. 이벤트들이 반드시 보존돼야 하기 때문에 이런 events channel은 [Apache Kafka](https://kafka.apache.org/)와 같은 event streaming 도구를 사용합니다.

Choreography saga는 간단하고 느슨하게 연결되기 때문에 작은 규모에서는 유용하지만, 서비스마다 퍼져있는 로직을 이해하고 관리하기 어려워서 중앙에서 제어하는 Orchestration saga 방식이 더 자주 쓰입니다.

### Orchestration-based saga

Orchestration saga는 중앙집권에 비유했듯이 하나의 Saga 관제 매니저에 의해 이벤트 생성 호출 등의 관리가 이루어집니다. 각 로직은 서비스에 존재하지만 로직 간 관계는 중앙에서 관리하기 때문에 한눈에 데이터의 흐름을 파악하기 용이합니다.

![출처: [Microservices Patterns](https://livebook.manning.com/book/microservices-patterns/chapter-4/107)](https://drek4537l1klr.cloudfront.net/richardson3/Figures/04fig06_alt.jpg)

출처: [Microservices Patterns](https://livebook.manning.com/book/microservices-patterns/chapter-4/107)

위의 그림은 식당 서비스의 구성 예시입니다. Order Service에 주문이 들어오면 Create order saga orchestrator가 필요한 각각의 서비스에 처리, 요청 이벤트를 전달하고 받아옵니다. Choreography-base saga는 Order와 Consumer, Consumer와 Kitchen, Order와 Accounting Service가 뒤섞여서 작동하겠지만, Orchestration-base saga는 Order Service 내의 orchestrator가 모든 것을 통제하고 처리하게 됩니다.

주의할 점은 orchestrator가 단순히 데이터 처리 프로세스를 지시하는 역할만 맡아야 한다는 것입니다. 그렇지 않고 비즈니스 로직이 주입되어 독립성이 훼손되면 마이크로서비스의 의미가 퇴색되고 관리가 어려워지게 됩니다.

### 격리성 유지

여느 디자인 패턴이 그렇듯 saga 패턴 또한 만능은 아닙니다.

Saga 패턴은 기본적으로 서비스 로컬에서 커밋이 이루어지기 때문에 이상 현상(Anomaly)이 발생할 수 있습니다. 둘 이상의 서비스에서 동시에 접근하여 다른 saga에서 수정된 값을 읽어들인다든지, 롤백(rollback)이 필요한 데이터를 읽어들여 값에 반영되는 경우도 있을 수 있습니다.

따라서 saga의 실행 순서, 교환적 로직(순서를 바꿔서 실행하여도 무관한), semantic lock 등을 적절히 활용하여 이상 현상이 발생할 여지를 없애는 것이 중요합니다.
