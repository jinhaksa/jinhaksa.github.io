---
layout: post
title: Parameter Sniffing
categories: [dev]
author: 고동하
email: godongha@jinhakapply.com
date: 2021-11-30
tag:
  - parameter sniffing
  - mssql
  - sql server
---

<!-- @format -->

# Parameter Sniffing (feat.SQL Server)

## Parameter Sniffing 이란?

SQL Server에서 파리미터가 사용된 SP 또는 쿼리(이하 parameterized query)를 실행할 때 옵티마이저가 파라미터 값을 전달 받아 효율적인 실행 계획을 생성하는 프로세스

---

간단한 예를 먼저 살펴보자

### 테스트 환경, 데이터 및 ERD

- DBMS : Microsoft SQL Server 2019 (RTM) - 15.0.2000.5 (X64)
- SSMS : v18.6
- DATA : AdventureWorks2019(https://docs.microsoft.com/ko-kr/sql/samples/adventureworks-install-configure?view=sql-server-ver15&tabs=ssms)
- 예제 참고 : SQL Server 2017 Query Performance Tuning(Chapter 17)

  ![](/assets/img/posts/dev/2021-11-30-parameter-sniffing/ps_6.png)

```sql
-- [1] Person.Address.City 컬럼을 'London' 조건으로 검색
USE AdventureWorks2019;

CREATE OR ALTER PROC dbo.AddressByCity_London
AS
    SELECT  a.AddressID,
            a.AddressLine1,
            AddressLine2,
            a.City,
            sp.Name AS StateProvinceName,
            a.PostalCode
    FROM    Person.Address AS a
    JOIN    Person.StateProvince AS sp
        ON a.StateProvinceID = sp.StateProvinceID
    WHERE a.City = 'London';

EXEC AddressByCity_London;
```

![](/assets/img/posts/dev/2021-11-30-parameter-sniffing/ps_1.png)

```sql
-- [2] Person.Address.City 컬럼을 지역 변수 @City 조건으로 검색
CREATE OR ALTER PROC dbo.AddressByCity_Local
AS
DECLARE @City VARCHAR(30) = 'London';
    SELECT  a.AddressID,
            a.AddressLine1,
            AddressLine2,
            a.City,
	          sp.Name AS StateProvinceName,
	          a.PostalCode
    FROM    Person.Address AS a
    JOIN    Person.StateProvince AS sp
        ON a.StateProvinceID = sp.StateProvinceID
    WHERE a.City = @City;

EXEC dbo.AddressByCity_Local;
```

![](/assets/img/posts/dev/2021-11-30-parameter-sniffing/ps_2.png)

```sql
-- [3] Person.Address.City 컬럼을 프로시저 파라미터 @City 조건으로 검색
-- 아래에서 재사용 예정
CREATE OR ALTER PROC dbo.AddressByCity @City NVARCHAR(30)
AS
    SELECT  a.AddressID,
            a.AddressLine1,
            AddressLine2,
            a.City,
	          sp.Name AS StateProvinceName,
	          a.PostalCode
    FROM    Person.Address AS a
    JOIN    Person.StateProvince AS sp
        ON a.StateProvinceID = sp.StateProvinceID
    WHERE a.City = @City;

EXEC AddressByCity 'London';
```

![](/assets/img/posts/dev/2021-11-30-parameter-sniffing/ps_3.png)

위 실행 계획에서 유심히 볼 필요가 있는 부분은 [예상 행 수]에 맞는 [실제 행 수]의 결과이다.  
[1]번 쿼리에서는 'London' 이라는 값을 알고 있기 때문에 실행 계획부터 정확한 실행 계획을 생성할 수 있었다.

[2]번 쿼리에서는 컴파일이 될 때까지 옵티마이저는 @City에 어떤 값이 들어갈지 알 수 없기 때문에, 통계 데이터를 이용하여 실행계획을 생성하였다.

- 실행 계획에서 사용될 수 있는 후보가 되는 인덱스들의 모든 통계 정보를 확인 후 해당 통계 정보의 평균 값(all density)을 이용하여 실행 계획을 생성한다.  
  통계를 이용한 예상 실행 행 수 = Rows \* All density

```sql
-- City 만을 단독으로 하는 NONCLUSTERED INDEX 생성
CREATE NONCLUSTERED INDEX IX_Address_City ON Person.Address (City);

-- IX_Address_City 인덱스의 통계
DBCC SHOW_STATISTICS('Person.Address', 'IX_Address_City');

-- IX_Address_City INDEX 삭제
DROP INDEX IX_Address_City ON Person.Address;
```

![](/assets/img/posts/dev/2021-11-30-parameter-sniffing/ps_5.png)

[3]번 쿼리에서는 프로시저에서 파라미터를 받아 @City로 검색 했을 때, [1]번 쿼리와 같은 실행 결과가 나왔다.  
프로시저가 실행되면서 파라미터의 들어온 값을 가지고 실행계왹을 세운다. 이와 같은 프로세스가 파라미터 스니핑이다.

## Bad Parameter Sniffing

파라미터 스니핑은 parameterized query가 실행 될 때마다 파라미터 값에 따라 실행 계획을 생성하는 것이 아니라, 첫 실행 계획을 캐시에 저장한다.  
때문에, 연이어 실행되는 parameterized query 또한, 같은 실행 계획이 재사용된다.

```sql
-- [4] Person.Address.City 컬럼을 'Mentor' 조건으로 검색
CREATE OR ALTER PROC dbo.AddressByCity_Mentor
AS
    SELECT  a.AddressID,
            a.AddressLine1,
            AddressLine2,
            a.City,
	          sp.Name AS StateProvinceName,
	          a.PostalCode
    FROM    Person.Address AS a
    JOIN    Person.StateProvince AS sp
        ON a.StateProvinceID = sp.StateProvinceID
    WHERE a.City = 'Mentor';

EXEC AddressByCity_Mentor;
```

```sql
-- 캐싱된 실행 계획 삭제
DBCC FREEPROCCACHE;
-- [3] 번 쿼리 실행(변수를 다르게 하여 순차적으로 실행)
EXEC AddressByCity 'Mentor';
EXEC AddressByCity 'London';
```

![](/assets/img/posts/dev/2021-11-30-parameter-sniffing/ps_4.png)

위 쿼리는 파라미터 스니핑의 나쁜 예이다.  
먼저 실행된 'Mentor' 의 실행 계획을 다음 실행한 'London'에서도 똑같이 사용하고 있다.  
규모가 작은 데이터이며 이와 같은 상황은 간헐적으로 나타나기 때문에 체감이 들지 않지만  
운영에서 치명적으로 작용할 수 있는 확률이 있다.

## 해결 방법

위 쿼리에서 'Mentor', 'London' 를 만족시키는 방법은 없다.(결정이 필요하다.)
아래와 같은 방법을 사용할 수는 있으나, 고려할 부분이 많다.

1. 쿼리 실행시 매번 리컴파일
2. 통계 값 사용
3. 파라미터를 지정하여 사용
4. 파라미터 스니핑을 사용하지 않도록 설정
5. 쿼리 스토어 플랜 강제 적용

---

### 1. 쿼리 실행시 매번 리컴파일

매번 리컴파일 방법을 사용하면 Bad Parameter Sniffing은 피할 수 있으나, 매번 리컴파일 비용을 사용하여 심각한 장애로 연결될 수 있다.

```sql
CREATE OR ALTER PROC dbo.AddressByCity @City NVARCHAR(30)
AS
    …
    WHERE a.City = @City
    OPTION (RECOMPILE);

EXEC AddressByCity 'London';

-- 또는 OPTION (RECOMPILE); 을 사용하지 않고
EXEC AddressByCity 'London' WITH RECOMPILE;
```

---

### 2. 통계 값 사용

데이터의 분산도가 일정하며, 특정 값이 지속적으로 호출되지 않는 상황에 적합하다.

이는 위에서 사용한 [2]번 쿼리처럼 지역 변수를 선언하여 사용하는 방법이다.

```sql
CREATE OR ALTER PROC dbo.AddressByCity @City NVARCHAR(30)
AS

DECLARE @LocalCity VARCHAR(30);
SET @LocalCity = @City;

    SELECT  a.AddressID,
            a.AddressLine1,
            AddressLine2,
            a.City,
	sp.Name AS StateProvinceName,
	a.PostalCode
    FROM    Person.Address AS a
    JOIN    Person.StateProvince AS sp
        ON a.StateProvinceID = sp.StateProvinceID
    WHERE a.City = @LocalCity;

EXEC AddressByCity 'London';

```

SQL Server 2008 부터 OPTIMIZE FOR(UNKNOWN) 으로 사용 가능하며, 이와 같은 방법을 권장한다.

- 지역변수를 사용할 경우, 리컴파일 중 가변 스니핑 발생 가능성有

---

### 3. 파라미터를 지정하여 사용

분산도가 한 쪽으로 치우쳐져 있으며, 특정 값이 지속적으로 호출될 때 사용된다.

```sql
CREATE OR ALTER PROC dbo.AddressByCity @City NVARCHAR(30)
AS
    SELECT  a.AddressID,
            a.AddressLine1,
            AddressLine2,
            a.City,
	sp.Name AS StateProvinceName,
	a.PostalCode
    FROM    Person.Address AS a
    JOIN    Person.StateProvince AS sp
        ON a.StateProvinceID = sp.StateProvinceID
    WHERE a.City = @City
    OPTION (OPTIMIZE FOR (@City = 'London'));

EXEC AddressByCity 'London';

```

### 4. 파라미터 스니핑을 사용하지 않도록 설정

이는 SQL Server 2016(13.x)부터 데이터베이스 단위의 설정으로 수행된다.  
[ALTER DATABASE SCOPED CONFIGURATION (Transact-SQL)](https://docs.microsoft.com/ko-kr/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql?view=sql-server-2017) 의 PARAMETER_SNIFFING 옵션 참고

쿼리 수준에서는 OPTIMIZE FOR UNKNOWN 쿼리 힌트를 사용해야 한다.  
SQL Server 2016(13.x) SP1부터 USE HINT 'DISABLE_PARAMETER_SNIFFING' [쿼리힌트](https://docs.microsoft.com/ko-kr/sql/t-sql/queries/hints-transact-sql-query?view=sql-server-2017) 참고

---

#### 이 글을 김현창PD에게 바칩니다.
