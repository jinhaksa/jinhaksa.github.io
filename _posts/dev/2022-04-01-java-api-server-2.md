---
layout: post
title: 자바 웹API 서버 구성하기 (下)
categories: [dev]
author: 문재범
email: jbm1871@jinhakapply.com
date: 2022-04-01
tag:
  - java
  - spring
  - api
---

<!-- @format -->

# 자바 웹API 서버 구성하기 (下)

上편에서는 기본적인 스프링 기반의 웹 애플리케이션 개발 환경 구성에 대해서 알아봤습니다. 下편에서는 스프링 설정과 logback 설정, 그리고 docker 배포 구성에 대해서 설명하겠습니다.

먼저 스프링의 설정 방법에는 두 가지 설정 방법이 있습니다. 하나는 xml 파일을 이용한 설정 방법이고 나머지 하나는 Java 파일을 이용한 설정 방법입니다.

두 가지 방식 모두 많이 사용되지만, Java 파일을 이용한 설정 방법은 IDE에서 Java 문법 체크를 해주기 때문에 오타로 인한 실수를 줄일 수 있습니다. 그래서 저는 Java 파일을 이용해서 스프링 설정 방법을 설명해드리겠습니다.

### **스프링 설정**

먼저 스프링 애플리케이션을 서블릿에 등록해야합니다. 아래에 도식을 보면 무슨 의미인지 이해하시기 쉽습니다.

![servlet.png](/assets/img/posts/dev/2022-04-01-java-api-server-2/servlet.png)

톰캣은

1. init 메소드를 통해서 서블릿을 구동하고 초기화합니다.
2. service 메소드를 호출해서 서블릿이 브라우저의 요청을 처리하도록 합니다. service 메소드 안에서 HTTP의 요청이 GET이면 doGet 메서드를 호출하고.. POST라면 doPost 메서드를 호출합니다.
3. 서버는 destory 메소드를 통해서 서블릿을 제거합니다. 주로 서버가 중단되거나 대기 상태로 유지되는 경우에 해당 메소드가 발생하게 됩니다.

결국 톰캣 위에서 작동하는 스프링 프레임워크도 서블릿으로써, 서블릿 컨테이너에 등록이 되어야 1차 적으로 WAS가 사용자의 요청에 맞는 응답을 줄 수 있게 됩니다.

![dispatcher.png](/assets/img/posts/dev/2022-04-01-java-api-server-2/dispatcher.png)

스프링에서는 해당 역할을 하는 서블릿 클래스를 제공합니다.

해당 역할의 클래스를 DispacherServlet 이라고 부릅니다.

톰캣의 서블릿 영역과 스프링 영역의 가교역할을 해준다고 이해하시면 됩니다. 스프링을 구동하기 위해서 꼭 필요합니다.

저는 한국사 인증을 위한 api이기 때문에 certhistory 라는 이름을 지었습니다.

디렉토리 구조는 다음과 같습니다.

일반적으로 패키지명은 회사 도메인과 프로젝트 이름을 사용하는 것이 관례입니다.

설정을 위한 config, 컨트롤러는 controller, 예외 처리를 위한 exception, 기타 util, vo..

저는 아래와 같이 구성했지만 편한 대로 구성하셔도 상관없습니다.

![project-structure.png](/assets/img/posts/dev/2022-04-01-java-api-server-2/project-structure.png)

ApplicationInitializer 클래스를 생성해줍니다.

이 클래스는 AbstractAnnotationConfigDispatcherServletInitializer 를 상속 받아야 합니다.

그리고 아래와 같이 DispatcherServlet을 등록해줍니다.

(xml 기반의 설정에서 ApplicationInitializer 클래스는 web.xml 파일에 해당합니다.)

```java
package com.jinhakapply.certhistory.initializer;

import javax.servlet.Filter;

import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.filter.CharacterEncodingFilter;
import org.springframework.web.servlet.DispatcherServlet;
import org.springframework.web.servlet.FrameworkServlet;
import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;

import com.jinhakapply.certhistory.config.AppConfig;
import com.jinhakapply.certhistory.config.WebConfig;

public class CertHistoryApplicationInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	@Override
	protected String[] getServletMappings() {
		// DispatcherServlet Mapping
		return new String[] { "/" };
	}

	@Override
	protected FrameworkServlet createDispatcherServlet(WebApplicationContext servletAppContext) {
		// create DispatcherServlet
		DispatcherServlet dispatcherServlet = (DispatcherServlet)super.createDispatcherServlet(servletAppContext);

		// Exception Handler가 없으면 Exception을 던지게끔 설정
		dispatcherServlet.setThrowExceptionIfNoHandlerFound(true);

		return dispatcherServlet;
	}

	@Override
	protected Filter[] getServletFilters() {
		return new Filter[] { new CharacterEncodingFilter("UTF-8", true) };
	}
	// 서블릿 필터에 인코딩을 UTF-8 로 하도록 설정
```

createDispatcherServlet 클래스를 override하고 위 코드와 동일하게 DispatcherServlet 인스턴스를 생성해서 return 해줍니다.

그리고 getServletMappings 메소드를 통해서 서블릿의 모든 요청을 스프링에서 처리할 수 있도록 “/” 루트 디렉토리를 DispatcherServlet 에 맵핑합니다.

xml 설정에 경우에 아래와 같이 등록하시면 됩니다.

그러나 앞으로 설명에서는 xml 설정 방법은 다루지 않도록 하겠습니다.

```xml
<servlet>
	<servlet-name>spring</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
</servlet>

<servlet-mapping>
	<servlet-name>spring</servlet-name>
	<url-pattern>/</url-pattern>
</servlet-mapping>
```

이제 서블릿 컨테이너이 DispatcherServlet 이 등록되었고, “/” 로 모든 요청을 DispatcherServlet이 처리하도록 맵핑되었습니다.

이제 ApplicationContext 와 WebApplicationContext 를 등록해야합니다.

먼저 스프링 내부 구조를 살펴보겠습니다.

![spring.png](/assets/img/posts/dev/2022-04-01-java-api-server-2/spring.png)

ApplicationConext와 WebApplicationContext 모두 스프링 컨테이너이고 자바빈을 주입할 수 있다.

위의 도식은 XML 설정 예시인데 applicationContext.xml 를 읽어서 애플리케이션 컨텍스트를 초기화하고 애플리케이션 그 자체와 관련이 있는 Repository, Service 영역의 빈을 주입, 설정 할 수 있다.

마찬가지로 웹 어플리케이션 컨텍스트는 Controller나 Exception 같은 웹과 관련된 영역의 빈을 주입해서 사용하고 웹과 관련된 스프링 설정을 할 수 있다.

서블릿 listener 에 등록되기 때문에 생명주기상 웹 애플리케이션 컨텍스트 보다 먼저 초기화됩니다.

(서블릿과 스프링 구조를 자세하게 기술하는 것은 따로 책으로 나올 만큼 방대한 분량이기 때문에 여기에서는 간단하게 이해하길 바랍니다)

```java
// ... 위 DispatcherServlet 내용 중략
@Override
	protected Class<?>[] getRootConfigClasses() {
		// TODO Auto-generated method stub
		return new Class<?>[] { AppConfig.class };
	}

	@Override
	protected Class<?>[] getServletConfigClasses() {
		// TODO Auto-generated method stub
		return new Class<?>[] { WebConfig.class };
	}
}
```

![Untitled](/assets/img/posts/dev/2022-04-01-java-api-server-2/Untitled.png)

```java
// AppConfig.java
package com.jinhakapply.certhistory.config;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy
@ComponentScan({"com.jinhakapply.certhistory.service", "com.jinhakapply.certhistory.util"})
public class AppConfig {
}
```

```java
// WebConfig.java
package com.jinhakapply.certhistory.config;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.context.annotation.Import;

import com.jinhakapply.certhistory.web.MvcConfig;

@Configuration
@EnableAspectJAutoProxy
@ComponentScan({"com.jinhakapply.certhistory.controller", "com.jinhakapply.certhistory.exception"})
@Import({MvcConfig.class})
public class WebConfig {
}
```

@ConponentScan 어노테이션을 이용해서 해당 패키지의 클래스를

@Configuration, @Repository, @Service, @Controller, @RestController 어노테이션이 선언된 모든 클래스를 자동으로 컨테이너에 등록해준다.

@EnableAspectJAutoProxy 글로벌 예외처리를 위해서 Spring AOP가 필요하다. 활성화하기 위해서 해당 어노테이션을 선언한다. 스프링이 프록시 형태로 사용자의 클래스를 호출할 수 있게 해준다.

@Configuration 스프링 설정파일을 스프링 프레임워크에 인지 할 수 있도록 하는 어노테이션이다.

@Import({}) 어노테이션을 통해서 설정파일을 나누고 Import 할 수 있습니다.

스프링 MVC 설정 관련 Java파일을 만들고 Import 하도록 하겠습니다.

![Untitled](/assets/img/posts/dev/2022-04-01-java-api-server-2/Untitled%201.png)

```java
package com.jinhakapply.certhistory.web;

import java.nio.charset.Charset;
import java.util.Arrays;
import java.util.List;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.converter.StringHttpMessageConverter;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.web.servlet.config.annotation.DefaultServletHandlerConfigurer;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
@EnableWebMvc
public class MvcConfig extends WebMvcConfigurerAdapter {
	private static final Charset UTF8 = Charset.forName("UTF-8");

	@Override
	public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
		// TODO Auto-generated method stub
		super.extendMessageConverters(converters);
	}

	// 정적 파일을 처리하는 핸들러를 설정
	@Override
	public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
		configurer.enable();
	}

	@Bean
	public StringHttpMessageConverter stringHttpMessageConverter() {
		StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
		stringHttpMessageConverter.setXSupportedMediaTypes(Arrays.asList(new MediaType("text", "html", UTF8)));
		return stringHttpMessageConverter;
	}

	@Bean
	public MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter() {
		MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter = new MappingJackson2HttpMessageConverter();
		mappingJackson2HttpMessageConverter.setSupportedMediaTypes(Arrays.asList(new MediaType("application", "json", UTF8)));
		return mappingJackson2HttpMessageConverter;
	}
}
```

앞서 pom.xml 에서 메이븐 dependency 를 설정 할 때 jackson-databind 라는 라이브러리를 등록했었는데요. 이 라이브러리는 json 을 자바 value object 로 직렬화해주는 라이브러리입니다.

위와 같이 작성해줍니다.

@Bean, @EnableWebMvc 에 대한 설명은 생략하겠습니다.

이렇게 모두 설정하면 @RequestMapping이나 @GetMapping @PostMapping, @PutMapping @DeleteMapping 같은 어노테이션을 컨트롤러에 선언함으로써 api 서버를 작성 할 수 있는 기본이 됩니다.

스프링의 기능이 워낙 방대하기 때문에 자세한 어노테이션들의 의미와 기능은 스프링을 별도로 살펴보셔야 합니다.

이상으로 기본적인 자바 웹 API 설정을 마치겠습니다.
