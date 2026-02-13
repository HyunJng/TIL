## 1. Spring Framework
* IoC/DI 를 해주는 ApplicationContext가 핵심이고 @Service/@Repository/@Controller 애너테이션을 통해 빈을 등록하고 관리

## 2. Spring MVC
* 웹 애플리케이션 개발을 위한 프레임워크. Spring의 웹 모듈.
* DispatcherServlet이 모든 요청을 받아서 처리하는 프론트 컨트롤러 역할
* spring-webmvc 의존성에 포함되어있다.
* 라이프사이클
    1. 서블릿 컨테이너가 web.xml을 읽음
    2. <listener>로 등록된 ContextLoaderListener를 초기화
    3. ContextLoaderListener가 내부적으로 ContextLoader를 이용해:
       * contextConfigLocation(보통 <context-param>)을 읽고
       * 그 설정을 기반으로 Root WebApplicationContext를 생성
       * 그리고 ServletContext에 **속성(attribute)**으로 저장해둠(그래서 나중에 다른 구성요소들이 “루트 컨텍스트”를 찾아 쓸 수 있음)
    4. <servlet>로 등록된 DispatcherServlet을 초기화
    5. DispatcherServlet이 내부적으로:
       * contextConfigLocation(보통 <servlet>의 <init-param>)을 읽고
       * 그 설정을 기반으로 WebApplicationContext를 생성

## 3. 레거시 파일
### 3.1. webapp/WEB-INF/web.xml
```xml
<web-app>
  <!-- Root ApplicationContext 설정 -->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
      /WEB-INF/applicationContext.xml
      /WEB-INF/datasource-context.xml
    </param-value>
  </context-param>

  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>

  <!-- WebApplicationContext(DispatcherServlet) 설정 -->
  <servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
</web-app>
```
* WAS가 시작될 때 읽는 파일.
* 정의하는 것:
    * ContextLoaderListener(Root ApplicationContext를 생성)
    * DispatcherServlet(WebApplicationContext를 생성)

### 3.2. applicationContext.xml, datasource-context.xml
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">
    <context:component-scan base-package="com.example.app"/>
</beans>
```
* Root ApplicationContext 설정 파일
* applicationContext.xml: 서비스, 레포지토리 빈 설정
* datasource-context.xml: 데이터소스 설정

### 3.3. dispatcher-servlet.xml
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns context="http://www.springframework.org/schema/context"
    xmlns mvc="http://www.springframework.org/schema/mvc"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">
    <context:component-scan base-package="com.example.app.controller"/>
    <mvc:annotation-driven/>
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```
* WebApplicationContext 설정 파일
* 컨트롤러 빈 설정, 뷰 리졸버 설정

## 4. ContextLoaderListener
* 서블릿 컨테이너(Tomcat 등)가 웹앱을 시작/종료할 때 호출해주는 부트스트랩 리스너
* 역할: Root ApplicationContext(= WebApplicationContext의 부모 컨텍스트)를 만들고, 웹앱 생명주기에 맞춰 시작/종료를 관리하는 것
* 웹과 무관한 Root ApplicationContext를 정의하지만, ContextLoaderListener 자체는 웹(서블릿) 환경에 강하게 의존하는 클래스라서 org.springframework.web 패키지에 있다.
*

## 5. ApplicationContext
### 5.1. Root ApplicationContext
* ContextLoaderListener가 생성
* 웹과 무관한 공통 빈(서비스, 레포지토리 등) 관리

### 5.2 WebApplicationContext
* DispatcherServlet이 생성
* 컨트롤러 빈 관리
* Root ApplicationContext를 부모로 참조

## 6. Spring Boot에서의 ApplicationContext
> classpath에 뭐가 있느냐로 앱 타입을 판단하여 컨텍스트 종류를 고른다
### 6.1. 서블릿 웹 앱(= MVC)

* 조건: `spring-webmvc` 같은 **서블릿 웹 스택**이 클래스패스에 있음 (보통 `spring-boot-starter-web`이 넣어줌)
* 결과: **`ServletWebServerApplicationContext` 계열**(= WebApplicationContext 성격) 생성

    * 내장 톰캣/서블릿 컨테이너 띄움 + `DispatcherServlet` 자동 등록

### 6.2. 리액티브 웹 앱(= WebFlux)

* 조건: `spring-webflux` 같은 **리액티브 스택**이 있음
* 결과: **ReactiveWebServerApplicationContext 계열** 생성 + Netty 등 띄움

### 6.3. 비웹 앱(NONE)

* 조건: 위 둘 다 없음 (즉, 웹 관련 핵심 의존성이 없음)
* 결과: **일반 `ApplicationContext`** 생성 (대표적으로 `AnnotationConfigApplicationContext` 계열)

    * 웹서버 안 뜸, `DispatcherServlet`도 없음 → **WebApplicationContext 아님**

## 7. SpringBoot와 레거시의 서블릿 웹 스택의 컨텍스트 구조
레거시(web.xml) 방식에서는 보통 부모-자식 컨텍스트 2단 구조
* ContextLoaderListener가 Root ApplicationContext
* DispatcherServlet이 자기 WebApplicationContext(자식)

하지만 Spring Boot 기본 구성은 보통 “컨텍스트 1개”로 동작
* Boot가 만든 그 하나의 컨텍스트가 웹 컨텍스트(WebApplicationContext 성격) 이고,
* DispatcherServlet은 그 컨텍스트를 그대로 사용하는 형태가 기본값

