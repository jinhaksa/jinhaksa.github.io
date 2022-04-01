---
layout: post
title: 자바 웹API 서버 구성하기 (上)
categories: [pm]
author: 문재범
email: jbm1871@jinhakapply.com
date: 2022-02-04
tag:
  - java
  - spring
  - api
---

<!-- @format -->

# 자바 웹API 서버 구성하기 (上)

진학어플라이 개발부 서비스파트에서 경찰청 원서접수서비스를 개발하는 도중 한국사능력검정시험 대상자정보 유통 서비스를 연동해야하는 일이 생겼습니다. 해당 서비스를 이용하려면 자바기반의 표준API를 사용해야 합니다.

정부기관의 모듈은 대부분 자바기반입니다. 그러나 진학사 서비스파트는 [**ASP.NET**](http://ASP.NET)을 사용합니다. 자바 기반의 모듈을 **ASP.NET**으로 변환하는 것은 어렵기에 자바 기반의 웹API 서버를 구성해서 기존시스템에서 호출하도록 구성해야합니다.

서비스 파트에서 자바 기반의 웹API 서버를 구현하기에 여의치 않은 상황이라, 전략프로젝트팀 소속인 제가 개발을 맡게되었습니다.

전략프로젝트팀에서는 현재 서버 개발을 TypeScript로 하기 때문에 자바를 사용하지는 않지만 그래도 자바 기반의 개발환경이 아닌 곳에서 자바 웹API 서버를 구성하시는분들에게 도움을 드리기 위해 제가 구성한 방법을 설명해드리고자 합니다. (최소한의 설정)

먼저 프로젝트 구성에 필요한 프로그램은 **JDK8, tomcat8.5, Docker, eclipse 2020-06** 입니다. 모두 설치가 되어야 합니다.

간단하게 설명을 드리자면 jdk는 자바 개발 키트입니다. 자바 개발을 하기 위해서 필수적입니다.

tomcat8.5는 자바 서블릿 구현체를 실행시켜주는 WAS입니다. 서블릿 컨테이너에서 사용자가 구현한 자바 서블릿 구현체를 알맞는 request에 맞게 실행시켜서 결과를 response 해주는 역할을 합니다.

이클립스는 자바를 개발할 수 있는 무료 IDE입니다. 다른 IDE도 많지만 이클립스는 무료이기 때문에 이클립스를 사용하도록 하겠습니다. 2020-06까지 JDK8을 지원하고 그 이후부터는 지원하지 않기때문에 2020-06 버젼을 사용했습니다.

마지막으로 도커 이미지로 빌드해서 도커 환경에서 독립적으로 구동이 가능하도록 할 것이기 때문에 도커가 필요합니다.

### 1. 프로젝트 구성 및 설정

우선 이클립스를 실행하고 인코딩 설정을 변경합니다. 기본적으로 EUC-KR 설정되어 이는 독자적인 한글 인코딩 방식이므로, 범용성을 위해서 UTF-8로 변경하는게 좋습니다.

**Window → Preferences → 검색 <Enc>**

![Untitled](/assets/img/posts/dev/2022-02-04-java-api-server/Untitled.png)

1. **General > Content Types > Text > UTF-8**
2. **General > Workspace > MS949→UTF8**
3. **Web > CSS Files > UTF-8**
4. **Web > HTML Files > UTF-8**
5. **Web > JSP Files > UTF-8**
6. **XML Files > UTF-8**

이클립스에 톰캣 설정을 합니다. 인터넷에 자세하게 나와있기 때문에 여기에서는 따로 설명하지 않겠습니다.

> [https://itworldyo.tistory.com/84](https://itworldyo.tistory.com/84)

프로젝트를 구성해봅시다. 외부 라이브러리를 사용해야 되기 때문에 이를 편하게 관리할 수 있는 관리 도구인 **Maven**(.NET 의 NuGet, Node.js의 npm 같은 역할)을 사용하도록 하겠습니다. (Gradle을 사용해도 된다)

![Untitled](/assets/img/posts/dev/2022-02-04-java-api-server/Untitled 1.png)

1. **Project Explorer 영역 우클릭**
2. **New > Project**
3. **Maven > Maven Project or Maven Module**
4. **프로젝트 생성시 Create a simple project 체크한 뒤 Next > 버튼 클릭**
5. **Group Id(회사 도메인을 반대로 적는게 관례 > com.jinhakapply), Artifact Id(프로젝트명 > my-api)을 입력한다.**
6. **Packaging을 클릭하고 모듈 구성시에는 pom으로 프로젝트 구성시에는 war로 설정한다. (만일 프로젝트에 Springboot framework을 사용할 것이라면 jar로 한다. Springboot는 was가 내장되어 있어서 일반적인 자바 배포파일로 동작하기 때문이다.)**
7. **Finish 버튼 클릭**

![Untitled](/assets/img/posts/dev/2022-02-04-java-api-server/Untitled 2.png)

메이븐 모듈로 생성하고 그 안에 프로젝트를 여러개 만들어서 관리할 수도 있고, 그냥 메이븐 프로젝트 단일로 구성해도 상관없습니다. 만일 모듈로 구성할 경우에 모듈의 Packaging 설정을 pom으로하고 모듈안의 프로젝트는 war로 합니다. 모듈로 구성하면 모듈단에서 공통으로 사용되는 라이브러리를 관리와 빌드 설정이 가능합니다.

![Untitled](/assets/img/posts/dev/2022-02-04-java-api-server/Untitled 3.png)

프로젝트를 구성하면 에러가 발생합니다. 이는 서블릿 설정 파일인 web.xml이 존재하지 않기 때문입니다. Spring Framework 에서는 web.xml 없이도 자바 파일로 web.xml 설정을 대신하는 방법을 지원하기 때문에

![Untitled](/assets/img/posts/dev/2022-02-04-java-api-server/Untitled 4.png)

Generate Deloyment Descriptor Stub을 누르면 web.xml 생성됩니다.

혹은 pom.xml의 빌드 설정에서 <failOnMissingWebXml>false</failOnMissingWebXml>로 설정해줘도 됩니다.

메이븐은 기본적으로 이 POM 파일을 기반으로 실행됩니다. pom.xml 을 메이븐의 설정파일이라고 생각하면 됩니다.

### 2. 메이븐 설정

위에서 설명드렸다싶이 pom.xml은 기본적으로 메이븐의 설정파일이고 메이븐은 이 설정파일은 기반으로 동작합니다. pom.xml에 디렉토리 구조나, 메이븐 레포지터리에 있는 외부 라이브러리를 추가할 수 있고 이 정보들을 기반으로 쉽게 빌드할 수 있습니다.

**메이븐 프로젝트 기본 디렉토리**

```yaml
application-core
pom.xml
src
main
java
com.pak.dir
resources
test
java
com.pak.dir
resources
```

기본적으로 메이븐 프로젝트 구성시에 기본적으로 pom.xml은 아래와 같이 생성됩니다.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion> <!-- 폼 버전, 그냥 4.0.0으로 두면 됩니다. -->
  <groupId>com.test</groupId>
<!-- 조직을 식별하는 유니크한 키입니다. 일반적으로 조직의 도메인으로 설정합니다/ -->
  <artifactId>test-api</artifactId>
<!-- 기본적으로 프로젝트의 이름이라고 생각하면 됩니다. 빌드를 하면
설정한 artifactId로 프로젝트가 빌드됩니다. -->
  <version>0.0.1-SNAPSHOT</version>
<!-- 프로젝트 버젼 -->
  <packaging>war</packaging>
<!-- 프로젝트 구성할 때 선택했던 항목, 만약 삭제하면 디폴트값인 jar로 빌드 -->
</project>
```

플러그인과 빌드 디렉토리 설정을 위해서 build 요소를 추가해줍니다.

먼저 디렉토리 설정을 해줘야 합니다.

```xml
<build>
	<finalName>certhistory</finalName> <!-- 배포할 이름 -->
	<sourceDirectory>src/main/java</sourceDirectory> <!-- 빌드할 소스가 있는 디렉토리 -->
	<outputDirectory>target/classes</outputDirectory> <!-- 배포 후 디렉토리 -->
	<resources> <!-- 리소스 파일 디렉토리 -->
	<!-- 소스 코드를 제외한 리소스 모든 파일들 -->
		<resource>
			<directory>src/main/resources</directory>
			<excludes> <!-- 혹시라도 소스코드가 들어가는 것을 방지하기 위해서 *.java 제외 필터링 -->
				<exclude>**/*.java</exclude>
			</excludes>
		</resource>
	</resources>
</build>
```

먼저 디렉토리 설정을 해줘야 합니다.

1. **소스 (자바) 파일 디렉토리**
2. **빌드 후 (클래스) 파일 디렉토리**
3. **리소스 파일 디렉토리**

```xml
<build>
	<!-- 위 디렉토리 설정 생략... -->
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-compiler-plugin</artifactId>
			<version>3.8.0</version>
			<configuration>
				<source>1.8</source> <!-- 자바 소스, 타겟 버젼 -->
				<target>1.8</target>
			</configuration>
		</plugin>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-resources-plugin</artifactId>
			<version>3.2.0</version>
			<configuration>
				<encoding>UTF-8</encoding>
			</configuration>
		</plugin>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-war-plugin</artifactId>
			<version>3.2.1</version>
			<configuration>
				<warSourceDirectory>src/main/webapp</warSourceDirectory>
				<!-- war빌드시 현재 프로젝트 디렉토리에서 복사할 소스 디렉토리 -->
				<!-- 보통 jsp 파일 -->
				<!-- src/main/webapp가 기본 경로 -->
				<failOnMissingWebXml>false</failOnMissingWebXml> <!-- web.xml 경고 무시 -->
				<webResources>
					<resource>
						<!-- 로컬 lib파일 경로 -->
						<!-- 메이븐에 없는 외부 라이브러리 jar파일을 사용시에 수동으로 빌드 할 때 포함시켜야 합니다. -->
						<!-- https://maven.apache.org/plugins/maven-war-plugin/examples/adding-filtering-webresources.html -->
						<directory>${project.basedir}/src/main/webapp/WEB-INF/lib</directory>
						<includes>
							<include>libgpkiapi_jni_1.5.jar</include>
							<include>commons-httpclient-3.1.jar</include>
						</includes>
						<targetPath>WEB-INF/lib</targetPath>
					</resource>
				</webResources>
			</configuration>
		</plugin>
	</plugins>
</build>
```

메이븐으로 war 파일을 빌드하기 위해서 3가지 플러그인이 필요합니다.

각각 **compiler, resources, war** 플러그인 입니다.

각각 플러그인이 필요한 이유와 역할이 궁금하다면, 아래의 공식 문서를 참조하시면 됩니다.

[https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#a-build-lifecycle-is-made-up-of-phases](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#a-build-lifecycle-is-made-up-of-phases)

[https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#default-lifecycle-bindings-packaging-ejb-ejb3-jar-par-rar-war](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#default-lifecycle-bindings-packaging-ejb-ejb3-jar-par-rar-war)

메이븐 빌드를 위한 설정을 완료했습니다. 이제 의존성 설정을 통해서 자바 웹 API를 쉽게 구현하게도와줄 라이브러리와 프레임워크를 다음과 같이 pom.xml에 추가해야합니다.

```xml
<properties>
<!-- 프로퍼티 안에 <key>value</key> 형식으로 설정하면
${key} 문법을 통해서 프로젝트 어디에서나 접근이 가능합니다.-->
	<org.springframework-version>4.3.2.RELEASE</org.springframework-version>
	<jcloverslf4j.version>1.7.6</jcloverslf4j.version>
	<logback.version>1.1.1</logback.version>
	<project.lib.path>${project.basedir}/src/main/webapp/WEB-INF/lib</project.lib.path>
</properties>

<dependencies>
	<!-- Spring Context -->
	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-context</artifactId>
		<version>${org.springframework-version}</version>
		<!-- Spring Framework는 기본적으로 로깅 인터페이스로
		Jakarta Common Logging(commons-logging)를 사용합니다.
		이 인터페이스를 사용하는 구현체로는 log4j가 있습니다.
		그러나 저는 slf4j 인터페이스의 구현체인 logback을 사용하기 위해서 제외시켰습니다. -->
		<exclusions>
			<exclusion>
				<groupId>commons-logging</groupId>
				<artifactId>commons-logging</artifactId>
			</exclusion>
		</exclusions>
	</dependency>

	<!-- 아래부터 스프링 모듈 필요한 모듈만 추가하면 된다. -->

	<!-- Spring Aspect -->
	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-aspects</artifactId>
		<version>${org.springframework-version}</version>
	</dependency>

	<!-- Spring Web -->
	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-web</artifactId>
		<version>${org.springframework-version}</version>
	</dependency>

	<!-- Spring MVC -->
	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-webmvc</artifactId>
		<version>${org.springframework-version}</version>
	</dependency>

	<!-- jackson -->
	<!-- 데이터 직렬화를 위한 라이브러리 -->
	<dependency>
		<groupId>com.fasterxml.jackson.core</groupId>
		<artifactId>jackson-databind</artifactId>
		<version>2.9.8</version>
	</dependency>

	<!-- Logback -->
	<!-- logback 인터페이스인 slf4j와 스프링 로깅 인터페이스인
	jcl간의 브릿지 역할을 위한 라이브러리 -->
	<dependency>
		<groupId>org.slf4j</groupId>
		<artifactId>jcl-over-slf4j</artifactId>
		<version>${jcloverslf4j.version}</version>
	</dependency>
	<!-- logback 구현체 -->
	<dependency>
		<groupId>ch.qos.logback</groupId>
		<artifactId>logback-classic</artifactId>
		<version>${logback.version}</version>
	</dependency>

	<!-- Servlet -->
	<!-- 기본적으로 스프링프레임워크도 서블릿이기 때문에
	당연히 서블릿도 추가해줘야 한다.  -->
	<dependency>
		<groupId>javax.servlet</groupId>
		<artifactId>javax.servlet-api</artifactId>
		<version>3.0.1</version>
		<scope>provided</scope> <!-- scope:빌드 할 때만 사용 -->
	</dependency>

	<!-- libgpkiapi_jni -->
	<dependency>
		<groupId>libgpkiapi</groupId>
		<artifactId>libgpkiapi</artifactId>
		<version>1.5</version>
		<scope>system</scope>
		<!-- 레포지터리가 아닌 별도의 라이브러리일 경우에 경로설정 -->
		<systemPath>${project.basedir}/src/main/webapp/WEB-INF/lib/libgpkiapi_jni_1.5.jar</systemPath>
	</dependency>

	<!-- org.apache.commons.httpclient -->
	<dependency>
		<groupId>httpclient</groupId>
		<artifactId>httpclient</artifactId>
		<version>3.1</version>
		<scope>system</scope>
		<!-- 레포지터리가 아닌 별도의 라이브러리일 경우에 경로설정 -->
		<systemPath>${project.basedir}/src/main/webapp/WEB-INF/lib/commons-httpclient-3.1.jar</systemPath>
	</dependency>
</dependencies>
```

설정을 위해서 <dependencies>와 <properties> 요소를 추가해야 합니다.

기본적으로 [https://mvnrepository.com/](https://mvnrepository.com/) 에서 의존성 정보를 제공합니다.

![Untitled](/assets/img/posts/dev/2022-02-04-java-api-server/Untitled 5.png)

<dependency>요소를 <dependencies>요소 하위에 추가하면 메이븐이 자동으로 해당 라이브러리를 프로젝트에 추가해주고 빌드시에도 같이 패키징됩니다.

下편에서 계속...
