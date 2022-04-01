---
layout: post
title: DB 쿼리에서의 인덱스 사용
categories: [dev]
author: 김동현
email: kdh@jinhakapply.com
date: 2022-04-01
tag:
  - DB
  - SQL
  - MySQL
---

# DB 쿼리에서의 인덱스 활용

4차 산업혁명과 함께 빈번하게 등장하는 단어 중 하나가 바로 빅데이터(Big Data) 입니다. 새삼스런 일도 아니지만 인터넷의 등장 이후 계속해서 축적돼 온 방대한 양의 정보, 사용자의 상호작용 및 패턴 분석에 필요한 로그, 두 세 가지 이상의 정보를 조합하여 가공된 정보 등을 저장하기 위해서 기존의 관계형 DB(RDBMS: Relational database management system)에서 벗어나 MongoDB와 같은 NoSQL, 분산 저장 방식의 Hadoop 등의 기술들이 많이 쓰이고 있습니다만, 여전히 기존 관계형 DB의 중요성은 사라지지 않고 있습니다.

이렇게 많이 쌓인 데이터를 효율적이게 처리하기 위해서는 적절한 인덱스(index) 설정과 함께 인덱스를 활용하여 데이터를 조회할 수 있는 쿼리 작성이 필수적이게 되었습니다.

각 DB마다 엔진의 방식 차이가 있지만, 기본적인 방식은 유사하기 때문에 실행 계획을 살펴가면서 확인해보는 것이 좋습니다. 여기서는 시장 점유율의 대부분을 차지하는 Oracle DB, Mysql 기준으로 설명하도록 하겠습니다.

## B-Tree 인덱스

인덱스에 가장 일반적으로 사용되는 알고리즘은 B-Tree(Balanced Tree)입니다. Hash 인덱스도 사용되긴 하지만 일부 혹은 범위 검색이 제한됩니다.

![인덱스 모식도 예시([출처](https://dzone.com/articles/database-btree-indexing-in-sqlite))](/assets/img/posts/dev/2022-04-01-using-index-at-db-query/img1.png)

인덱스 모식도 예시([출처](https://dzone.com/articles/database-btree-indexing-in-sqlite))

최상위의 루트 노드, 말단의 리프 노드, 중간의 브랜치 노드, 그리고 실제 데이터 부분으로 이루어져 있는데, 실제 데이터는 삽입, 삭제, 수정 등이 반복되기 때문에 정렬되어 있지는 않습니다.

각 노드는 인덱스 키와 프라이머리 키 단위인 페이지로 구성되어 있고, 키는 정렬되어 있기 때문에 이를 이용하여 원하는 데이터를 목차 찾듯이 빠르게 찾을 수 있습니다.

그러나 이진 탐색 트리와 마찬가지로 데이터 조회에는 빠르지만 항상 정렬된 상태를 유지하기 위해 삽입, 삭제 시에는 인덱스가 없는 경우보다 비용이 많이 든다는 점 또한 고려해야 합니다. 또한 인덱스를 찾은 상태에서도 데이터 레코드를 읽어오기 위한 Random I/O가 일어나기 때문에 일반적으로 찾고자 하는 데이터가 전체의 1/4 이상이라면 오히려 인덱스 이용이 비효율적이 될 수도 있습니다.

## 인덱스 스캔 종류

### 1. Index Range Scan

찾아야 할 인덱스 범위가 명확하게 정해졌을 때 사용되는 방식입니다. 해당 인덱스 키를 따라서 데이터를 찾은 뒤 데이터 레코드를 순서대로 읽기만 하면 되기 때문에 효율적인 방식입니다.

### 2. Index Full Scan

인덱스 전체를 모두 읽어와서 사용하는 방식입니다. 전체 테이블을 읽는 것보다는 효율적이긴 하지만, 인덱스를 활용하는 의미가 퇴색되기 때문에 효율적으로 사용하지 못하고 있다고 볼 수 있습니다.

### 3. Index Skip Scan

복수의 인덱스 컬럼에서 선행 컬럼을 사용하지 않고도 후행 컬럼의 인덱스를 따라 중간 중간 생략(skip)하면서 읽어들이는 방식입니다.

예를 들어 dept_no(부서), birthday(생일)로 이루어진 인덱스가 생성된 employees 테이블이 있다고 하면,

```sql
SELECT dept_no, birthday FROM employees WHERE dept_no = 3 AND birthday >= '2022-04-01';
SELECT dept_no, birthday FROM employees WHERE birthday >= '2022-04-01';
```

첫 번째 쿼리는 정상적으로 인덱스를 사용하지만 두 번째는 department 컬럼 조건이 없어서 인덱스를 이용하지 못하나, birthday 컬럼을 가지고 중간 부분을 건너뛰기 때문에 full table scan이 아닌 index skip scan이 사용될 수 있습니다.(Oracle 9i, Mysql 8.0 이후 버전. 다만 Mysql의 경우 인덱스에 존재하는 컬럼으로만 구성된 쿼리만 가능)

그 외에도 DB 종류에 따라 여러 스캔 방식이 존재하지만 크게 보면 위 세 가지의 하위로 들어가게 됩니다.

## 인덱스 가용성

1. 인덱스 지정 컬럼의 순서

   다중 컬럼 인덱스의 경우 선행 컬럼의 조건을 먼저 체크하기 때문에 범위를 더 크게 제한하는 쪽의 컬럼을 선행으로 지정해야 성능에 도움이 됩니다.

2. 인덱스 사용이 불가한 조건식 지양

   다음과 같은 경우는 인덱스 사용이 불가하거나 작업 범위를 줄이지 못하므로 쿼리에서 사용을 자제해야 합니다.

   1. NOT IN, NOT BETWEEN, IS NOT NULL, <>
   2. 컬럼 값을 조작하는 함수(SUBSTRING 등)
   3. LIKE ‘%검색어’(LIKE ‘검색어%’의 경우 가능)
   4. 타입 변환이 일어나는 경우
   5. 문자열 colation이 다른 경우

쿼리에서 인덱스가 사용되는 부분에 따라서도 조금씩 차이가 있습니다. WHERE 절의 경우 인덱스 컬럼 순서에 따라 선행 컬럼이 조건에 있어야 후행 컬럼 조건을 사용할 수 있지만(앞서 언급했던 index skip scan의 경우 예외가 있긴 하나), WHERE 절 내에서 조건을 명시하는 순서는 (버전에 따라 차이가 있지만) optimizer가 알아서 처리해주기 때문에 크게 중요하지는 않습니다. 다만 AND가 아닌 OR 연산자가 있다면 스캔 방식이 바뀔 수 있습니다.

WHERE 절과 달리 GROUP BY 절에서는 컬럼 순서 또한 인덱스 컬럼 순서와 정확히 일치해야 합니다. 모든 인덱스 컬럼을 명시해야 할 필요는 없지만 선행 컬럼이 있어야 후행 컬럼이 사용될 수 있습니다.

가령 column1, column2, column3 순서로 인덱스가 생성되었다면,

```sql
GROUP BY column1, column2 // 가능
GROUP BY column2, column1 // 불가능
GROUP BY column1, column3 // 불가능
GROUP BY column1, column2, column3 // 가능
GROUP BY column1, column3, column2 // 불가능
GROUP BY column1, column2, column3, column4 // column4가 인덱스 컬럼이 아니라 불가능
```

위와 같이 인덱스 사용 여부가 달라집니다.

그렇다면 WHERE 절과 GROUP BY 절에 혼용으로 사용될 경우는 어떨까요? 그 경우 WHERE 절에 선행 컬럼이 있다면 GROUP BY 절에 생략이 가능하지만 마찬가지로 GROUP BY 내에서 순서가 일치해야 하고 WHERE 절 해당 컬럼이 동등 비교 조건(=)인 경우만 가능합니다.

```sql
SELECT * FROM t1 WHERE column1 = 'a' AND column2 = 'b' GROUP BY column3, column4;
```

WHERE 절과 ORDER BY 절을 혼용하는 경우에도 마찬가지이지만, 추가로 ORDER BY 정렬 방향이 인덱스 정렬 방향과 같거나 정반대여야 합니다. 예를 들어 인덱스 컬럼이 (column1 ASC, column2 ASC, column3 ASC) 인 경우 ORDER BY 는 (column1 ASC, column2 ASC, column3 ASC) 또는 (column1 DESC, column2 DESC, column3 DESC) 중 하나여야 합니다.

그리고 WHERE + ORDER BY 가 인덱스를 사용하는 방식은 두 절이 같은 인덱스를 이용하거나, WHERE 절만 이용하여 데이터 조회 후 ORDER BY 가 수행되거나, 반대로 ORDER BY로 인덱스를 이용하여 정렬 후 WHERE 조건을 체크하는 방식이 수행될 수 있습니다.

GROUP BY + ORDER BY의 경우에는 두 절의 컬럼 순서와 방향이 모두 같아야 합니다. 따라서 GROUP BY와 ORDER BY는 둘 모두 인덱스를 사용하던지, 못하던지의 두 경우만 존재합니다. 그러므로 WHERE + GROUP BY + ORDER BY의 경우 셋 모두 사용하거나, WHERE만 사용하거나, GROUP BY + ORDER BY만 사용하는 경우로 나뉠 수 있습니다.(모두 사용하지 못하는 경우도 존재)

쿼리 작성에서 인덱스를 고려해야 하는 부분은 데이터 타입, 함수 사용 등 더 많이 있지만, 이번에는 크게 중요한 부분만 살펴보았습니다. 보다 자세한 쿼리 실행 계획과 작동 원리는 각 DB별 문서와 EXPLAIN 명령어를 통하여 확인해 보는 것이 확실합니다.
