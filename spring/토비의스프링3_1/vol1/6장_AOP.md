# 6장. AOP
## 1. Proxy
### 1.1. Proxy 정의
* 프록시: 실제 오브젝트 대신에, 타겟인 것처럼 위장해서 클라이언트 요청을 처리하는 오브젝트
* 프록시의 목적
    * 클라이언트가 타겟 오브젝트에 접근하는 것을 제어
    * 타겟 오브젝트에 부가적인 기능을 제공

### 1.2. 관련된 디자인 패턴
#### 데코레이트 패턴 vs 프록시 패턴
패턴을 구현하는 코드는 동일, 단 목적이 어떤 것이냐에 따라 패턴이 구분된다.
* 데코레이트 패턴: 타겟 오브젝트에 새로운 기능을 추가하는 것이 목적
* 프록시 패턴: 타겟 오브젝트에 접근하는 것을 제어하는 것이 목적

---
## 2. 수동 프록시 적용하기
> * 상황: 비즈니스 코드가 반드시 트랜잭션과 함께 동작해야 함
> * 문제: 트랜잭션 코드를 비즈니스 코드에 직접 작성하면, 비즈니스 로직과 트랜잭션 로직이 서로 엉켜서 유지보수가 어려워짐
> * 해결책: 프록시 패턴을 적용하여 [트랜잭션 코드를 비즈니스 코드에서 분리 + 비즈니스 코드 호출 시 반드시 트랜잭션 처리가 동작] 목적 달성 

### 2.1. 트랜잭션 프록시 만들기
#### 1) UserService 인터페이스 생성
```java
public interface UserService {
    void add(User user);
    void upgradeLevels();
}
```

#### 2) UserServiceImpl 클래스 생성
```java
public class UserServiceImpl implements UserService {
    @Override
    public void add(User user) {
        // 사용자 추가 로직
    }

    @Override
    public void upgradeLevels() {
        // 사용자 레벨 업그레이드 로직
    }
}
```
* 오직 비즈니스 로직만 포함되어 있음
* 해당 로직은 반드시 트랜잭션 내에서 실행되어야하므로 클라이언트에게 직접 노출되지 않음.

#### 3) UserServiceTx 클래스 생성 (트랜잭션 프록시)
```java
public class UserServiceTx implements UserService {
    private UserService userService;
    private PlatformTransactionManager transactionManager;

    public UserServiceTx(UserService userService, PlatformTransactionManager transactionManager) {
        this.userService = userService;
        this.transactionManager = transactionManager;
    }

    @Override
    public void add(User user) {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            userService.add(user);
            transactionManager.commit(status);
        } catch (RuntimeException e) {
            transactionManager.rollback(status);
            throw e;
        }
    }

    @Override
    public void upgradeLevels() {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            userService.upgradeLevels();
            transactionManager.commit(status);
        } catch (RuntimeException e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```
* UserServiceTx는 UserService 인터페이스를 구현하여 UserServiceImpl을 감싸는 "프록시" 역할.
* 트랜잭션이라는 부가적인 기능을 제공하면서, UserServiceImpl의 메서드를 호출하여 실제 비즈니스 로직을 수행.

#### 4) 클라이언트 코드에서 UserService를 의존하면 UserServiceTx가 주입되도록 설정
```xml
<bean id="userService" class="user.service.UserServiceTx">
 <property name="transactionManager" ref="transactionManager"></property>
 <property name="userService" ref="userServiceImpl"></property>
</bean>

<bean id="userServiceImpl" class="user.service.UserServiceImpl">
 <property name="userDao" ref="userDao"></property>
 <property name="mailSender" ref="mailSender"></property>
</bean>
```
* UserServiceTx의 bean 이름을 userService로 지정하여 클라이언트가 UserService 인터페이스를 의존할 때 UserServiceTx가 주입되도록 설정
* 클라이언트가 UserService 인터페이스를 의존하면 UserServiceImpl이 주입될 것이라 기대하지만, 
  실제로는 UserServiceTx가 이를 가로채서 트랜잭션 처리를 담당하면서 UserServiceImpl의 메서드를 호출.

### 2.2. 수동 프록시의 문제점
1. 인터페이스의 모든 메서드를 구현해 위임하도록 코드를 작성해야하므로, 인터페이스가 변경될 때마다 프록시 클래스도 함께 수정해야하는 유지보수 문제 발생
2. 부가적인 기능이 필요한 경우마다 프록시 클래스를 만들어야하므로, 코드의 중복과 복잡성이 증가

## 3. 다이나믹 프록시로 개선하기
### 3.1. 다이나믹 프록시란?
* 런타임에 Proxy 역할의 클래스를 동적으로 생성하여, 인터페이스의 모든 메서드를 자동으로 위임하도록 만들어주는 "오브젝트"
* Java의 `java.lang.reflect.Proxy`라는 "프록시 팩토리"를 제공하여 "다이나믹 프록시"를 생성할 수 있도록 지원.
    * 단, "인터페이스 기반" 프록시만 지원하므로, 클래스 기반 프록시가 필요한 경우에는 CGLIB와 같은 라이브러리를 사용해야함
> reflection API
> * 자바의 코드 자체를 추상화하여 런타임에 클래스의 정보(메서드, 필드 등)를 동적으로 조회하거나 조작할 수 있게 해주는 API

### 3.2. Java의 다이나믹 프록시 구성 요소
#### 1) Proxy 클래스
```java
public class Proxy {
    public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) {
        // 런타임에 프록시 클래스 생성 및 인스턴스 반환
    }
}
```
* 다이나믹 프록시를 생성하는 프록시 팩토리 역할의 클래스
* `newProxyInstance(...)`: 런타임에 프록시 클래스를 생성하여 인스턴스를 반환하는 정적 메서드
    * `loader`: 프록시 클래스가 사용할 클래스 로더.
    * `interfaces`: 프록시가 구현할 인터페이스 배열
    * `h`: 프록시의 메서드 호출을 처리할 InvocationHandler 인터페이스의 구현체

> 클래스 로더란?
> * 자바에서 클래스 파일을 읽어서 메모리에 로드하는 역할을 하는 컴포넌트
> * 다이나믹 프록시가 생성하는 프록시 클래스도 런타임에 메모리에 로드되어야 하므로, 프록시 클래스가 사용할 클래스 로더를 지정해야함

#### 2) InvocationHandler 인터페이스
```java
public interface InvocationHandler {
    Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```
* `invoke(...)`: 프록시 객체가 호출될 때마다 실행되는 메서드. 
    * 리플렉션 정보를 인자로 받아서, 실제 타겟 오브젝트의 메서드를 호출하거나 부가적인 기능을 수행.
* 프록시의 목적 중 하나인 "부가적인 기능 제공"을 구현하는 핵심 인터페이스

> * 여기서 "부가적인 기능"을 "어드바이스(advice)"라고 한다
> * 나중에 나오지만 Pattern과 어드바이스를 종합하여 InvocationHandler는 어드바이저(Advisor)이 되는 셈

#### 3) 다이나믹 프록시 생성 예시
```java
public class TransactionHandler implements InvocationHandler {
    private Object target;
    private String pattern;
    private PlatformTransactionManager transactionManager;

    public TransactionHandler(Object target, PlatformTransactionManager transactionManager) {
        this.target = target;
        this.transactionManager = transactionManager;
        this.pattern = "*"; // 기본 패턴 설정
    }
    
    public TransactionHandler(Object target, PlatformTransactionManager transactionManager, String pattern) {
        this.target = target;
        this.transactionManager = transactionManager;
        this.pattern = pattern; // 사용자 정의 패턴 설정
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().matches(pattern)) {
            return invokeInTransaction(method, args);
        } else {
            return method.invoke(target, args); // 패턴에 맞지 않는 메서드는 트랜잭션 없이 호출
        }
    }
    
    public Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object result = method.invoke(target, args); // 실제 타겟 메서드 호출
            transactionManager.commit(status);
            return result;
        } catch (InvocationTargetException  e) {
            transactionManager.rollback(status);
            throw e.getTargetException(); // 실제 예외를 던짐
        }
    }
}
```

```java
// UserService 인터페이스를 구현하는 다이나믹 프록시 오브젝트 생성
UserService userService = (UserService) Proxy.newProxyInstance(
        UserService.class.getClassLoader(),
        new Class<?>[]{UserService.class},
        new TransactionHandler(new UserServiceImpl(), transactionManager, "upgradeLevels")
);
```
* `Proxy.newProxyInstance(...)`를 사용하여 UserService 인터페이스를 구현하는 프록시 인스턴스를 생성.
* `InvocationHandler`의 `invoke(...)` 메서드에서 트랜잭션 처리를 구현하면서, 
   실제 타겟 오브젝트인 UserServiceImpl의 메서드를 리플렉션을 통해 호출하여 비즈니스 로직을 수행하도록 구현.
* 생성된 `userService`의 어떤 메서드를 사용하여도 InvocationHandler의 invoke 메서드가 호출되며, 
    reflection을 통해 동일한 이름의 UserServiceImpl의 메서드가 호출되므로, 인터페이스의 모든 메서드에 대해 트랜잭션 처리가 자동으로 적용됨.
* 다이나믹 프록시를 사용하면, 인터페이스의 모든 메서드를 자동으로 위임하도록 구현할 수 있으므로, 
  **인터페이스가 변경될 때마다 프록시 클래스를 수정할 필요가 없어 유지보수성이 향상**됨

## 4. 다이나믹 프록시 객체를 bean으로 등록하기
### 4.1. Spring이 제공하는 FactoryBean 구현
#### 1) FactoryBean 인터페이스
```java
public interface FactoryBean<T> {
    T getObject() throws Exception; //bean 오브젝트 생성
    Class<?> getObjectType(); // 생성하는 bean 오브젝트의 타입 반환
    boolean isSingleton(); // 생성하는 bean 오브젝트가 싱글톤인지 여부 반환
}
```
* 스프링은 내부적으로 reflection API를 사용하여 bean 정의에 나오는 클래스 이름을 읽어서 클래스를 로드하고, 해당 클래스의 인스턴스를 생성하여 bean으로 등록한다.
* 하지만 다이나믹 프록시와 같이 런타임에 생성되는 오브젝트는 일반적인 방법으로는 bean으로 등록할 수 없으므로, 
  스프링은 FactoryBean 인터페이스를 제공하여 **런타임에 생성되는 오브젝트도 bean으로 등록할 수 있도록 지원**함.
* `FactoryBean`을 구현한 클래스를 스프링 빈으로 등록하면 스프링이 해당 FactoryBean을 사용하여 실제 빈 오브젝트를 생성하도록 지원

#### 2) TransactionProxyFactoryBean 클래스 구현
```java
public class TransactionProxyFactoryBean implements FactoryBean<Object> {
    private Object target; // 타겟 오브젝트
    private PlatformTransactionManager transactionManager; // 트랜잭션 매니저
    private String pattern = "*"; // 트랜잭션이 적용될 메서드 패턴
    private Class<?> serviceInterface; // 프록시가 구현할 인터페이스, target의 인터페이스 타입으로 설정
    
    @Override
    public Object getObject() throws Exception {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                serviceInterface,
                new TransactionHandler(target, transactionManager, pattern)
        );
    }

    @Override
    public Class<?> getObjectType() {
        return serviceInterface;
    }

    @Override
    public boolean isSingleton() {
        return true; // 프록시 객체는 싱글톤으로 생성
    }

    // setter 메서드들...
}
```
```xml
<bean id="userService" class="user.service.TransactionProxyFactoryBean">
     <property name="target" ref="userServiceImpl"></property>
     <property name="transactionManager" ref="transactionManager"></property>
     <property name="pattern" value="upgradeLevels"></property>
     <property name="serviceInterface" value="user.service.UserService"></property>
</bean>
```
* 사용 과정:
  1. 클라이언트가 UserService 인터페이스를 의존 
  2. userService 이름을 가진 bean이라면 TransactionProxyFactoryBean이 getObject() 메서드의 반환값이 주입되어야한다고 인식
  3. TransactionProxyFactoryBean의 getObject() 메서드가 호출되어 다이나믹 프록시 객체가 생성되고 반환
  4. 클라이언트는 UserService 인터페이스를 통해 프록시 객체를 사용하여 메서드를 호출하면, InvocationHandler의 invoke() 메서드가 실행
  5. invoke() 메서드에서 트랜잭션 처리가 수행되고, 실제 타겟 오브젝트인 UserServiceImpl의 메서드가 리플렉션을 통해 호출되어 비즈니스 로직이 실행됨

> 즉, 특정 Bean 이름으로 FactoryBean 을 등록하면 getObject()를 실행시켜 반환값을 주입하는 것을 이용하여 proxy 객체를 주입할 수 있다는 것

## 5. [다이나믹 프록시 + 팩토리 빈] 적용의 한계
### 5.1. 장점
* 인터페이스가 변경될 때마다 프록시 클래스를 수정할 필요가 없어 유지보수성이 향상됨
* 트랜잭션과 같은 부가적인 기능을 프록시 객체가 책임지도록 하여 비즈니스 로직에서 완전히 분리가 이루어짐
* 클라이언트가 타겟 오브젝트를 사용할 때 프록시 객체를 통해 접근하도록 강제할 수 있음.

### 5.2. 단점
* 한번에 여러 개의 프록시를 적용하기 까다로움
* 한번에 여러 개의 클래스에 공통적인 부가 기능을 적용하기 어려움 
  * 각 클래스마다 별도의 프록시 팩토리 빈을 만들어야하므로, 코드의 중복과 복잡성이 증가
* 인터페이스 기반 프록시만 지원하므로, 클래스 기반 프록시가 필요한 경우에는 CGLIB와 같은 라이브러리를 사용해야함

## 6. Spring에서 제공하는 ProxyFactoryBean
위에서 보여준 `TransactionProxyFactoryBean`와 동일한 기능을 하는, 스프링의 지원 기술이다.

### 6.1. ProxyFactoryBean
```java
// 많이 생략
public class ProxyFactoryBean implements FactoryBean<Object> {
    List<MethodInterceptor> advice;
    Target target;
    // 등등... 실제랑은 다른데 대충 이런 기능
}
```
* `ProxyFactoryBean`이 Advisor인 셈
  * Advice(타겟의 부가기능) + Target(메소드 선정 알고리즘)
* 차이점은 `MethodInterceptor`이 있어 직접 타겟을 받을 필요가 없다는 것이다
  ```java
  public interface MethodInterceptor {
      public Object invoke(MethodInvocation invocation) throws Throwable;
  }  
  ```
  * MethodInvocation 에 target 정보 등이 포함되어있다.

### 6.2. TransactionAdvice 적용해보기
```java
public class TransactionAdvice implements MethodInterceptor {
    PlatformTransactionManager txManager;
    
    public Object invoke(MethodInvocation invocation) throws Throwable {
      TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
      try {
        Object result = invocation.proceed();
        transactionManager.commit(status);
        return result;
      } catch (InvocationTargetException  e) {
        transactionManager.rollback(status);
        throw e.getTargetException();
      }
    }
}
```
```xml
<bean id="transactionAdvice" class="springbook.user.service.TransactionAdvice">
    <property name="transactionManager" ref="transactionManager"></property>
</bean>
<bean id="transactionPointCut" class="org.springframework.aop.support.NameMatchMethodPointcut">
    <property name="mappedName" value="upgrade*"></property>
</bean>
<bean id="transactionAdvisor"
    class="org.springframework.aop.support.DefaultPointcutAdvisor">
    <property name="advice" ref="transationAdvice"></property>
    <property name="pointcut" ref="transactionPointcut"></property>
</bean>
<bean id="userService" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="userServiceImpl"/>
    <property name="interceptorNames">
        <list> <value>transactionAdvisor</value></list>
    </property>
</bean>
```
### 6.3. 단점
* 매번 타겟 생길때마다 PointCut만들고 설정정보 지정해주기 귀찮다!!
* Advisor 설정도 더 간결하게 하고 싶다!

## 7. Spring이 제공하는 Bean 후처리기를 사용해서 PointCut을 확장하고 설정도 간결하게 하자
### 7.1. 빈 후처리기란?
**`BeanPostProcessor`인터페이스**
* 스프링 빈 오브젝트로 만들어지고 난 후에, 빈오브젝트를 다시 가공할 수 있게 해준다.
* `DefaultAdvisorAutoProxyCreator`는 어드바이저를 이용한 자동 프록시 생성기로 자체적으로 빈등록을 하고 빈오브젝트가 생성될 때마다 빈후처리기에 보내서 후처리 작업을 요청한다

```
DefaultAdvisorAutoProxyCreator 빈후처리기가 bean 등록되어있다면

1. 빈이 생성될 때마다 빈후처리기에 빈 오브젝트를 보낸다.
2. 빈후처리기는 빈 오브젝트가 프록시 적용대상인지 확인한다.
3. 프록시 적용대상이라면 프록시 생성기에게 프록시를 만들게하고, 만들어진 프록시에 Advisor를 연결한다.
4. 빈후처리기는 프록시가 생성되면 원래 컨테이너가 전달해준 빈 오브젝트 대신 프록시 오브젝트를 컨테이너에게 돌려준다.
```
> 즉, 빈후처리기 하나만 Bean으로 등록하면 된다!

### 7.2. PointCut 
```java
public interface pointcut{
  ClassFilter getClassFilter();
  MethodMatcher getMethodMatcher();
}
```
* 클래스로 먼저 거르고 -> 메서드 거른다

### 7.3. DefaultAdvisorAutoProxyCreator 적용
```xml
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"></bean>
```
* 자동으로 Advisor 인터페이스를 구현한 것을 모두 찾는다, 그리고 생성되는 모든 빈에 대해 어드바이저의 포인트컷을 적용해보면서 프록시 적용 대상을 선정한다.

## 8. 포인트컷 표현식
가장 대표적으로 사용하는 표현식은 `execution()` 이고, 그 외는 나중에 다룬다고 한다.

```java
execution([접근제한자] 리턴타입 [패키지명].[클래스명].[메서드명](파라미터))
```

예시:

```java
execution(public int com.example.demo.Target.minus(int, int) throws java.lang.RuntimeException)
```

위 표현식은 아래 조건을 모두 만족하는 메서드를 의미한다.

* 접근제한자가 `public`
* 리턴 타입이 `int`
* 클래스가 `com.example.demo.Target`
* 메서드명이 `minus`
* 파라미터가 `(int, int)`
* `RuntimeException`을 던짐

---

#### 1) `*`

아무 값이나 올 수 있음을 의미한다.

```java
execution(* *(..))
```

의미:

* 모든 리턴 타입
* 모든 메서드명
* 모든 파라미터

즉, **모든 메서드**를 의미한다.

예시:

```java
execution(* hello(..))
```

* 이름이 `hello`인 모든 메서드

```java
execution(* upgrade*(..))
```

* 이름이 `upgrade`로 시작하는 모든 메서드

---

#### 2) `..`

개수가 정해지지 않은 여러 개를 의미한다.
패키지 경로나 파라미터 위치에서 자주 사용된다.

##### 패키지에서 사용

```java
execution(* com.example..*.*(..))
```

의미:

* `com.example` 패키지와 그 하위 패키지 전체
* 그 안의 모든 클래스
* 모든 메서드

##### 파라미터에서 사용

```java
execution(* *(..))
```

의미:

* 파라미터 개수와 타입에 상관없는 모든 메서드

```java
execution(* *(String, ..))
```

의미:

* 첫 번째 파라미터가 `String`
* 뒤에는 어떤 파라미터가 몇 개가 오든 상관없음

---

#### 3) 접근제한자

생략 가능하며, 지정하면 해당 접근제한자만 매칭된다.

```java
execution(public * *(..))
```

* `public` 메서드만 대상

---

#### 4) 리턴 타입

리턴 타입 위치에도 와일드카드를 사용할 수 있다.

```java
execution(* *(..))
```

* 모든 리턴 타입 허용

```java
execution(String *(..))
```

* 리턴 타입이 `String`인 메서드만 대상

---

#### 5) 클래스명 지정

특정 타입이나 이름 패턴을 기준으로 대상 클래스를 지정할 수 있다.

```java
execution(* *..*ServiceImpl.*(..))
```

의미:

* 이름이 `ServiceImpl`로 끝나는 클래스
* 그 안의 모든 메서드

```java
execution(* *..*ServiceImpl.upgrade*(..))
```

의미:

* 이름이 `ServiceImpl`로 끝나는 클래스
* 그중 `upgrade`로 시작하는 메서드

이 표현식은 토비의 스프링 6.5에서 트랜잭션 포인트컷 예제로 자주 나오는 형태다.

---

#### 6) 메서드명 지정

메서드 이름 패턴으로 적용 범위를 줄일 수 있다.

```java
execution(* *(..))
```

* 모든 메서드

```java
execution(* upgrade*(..))
```

* `upgrade`로 시작하는 메서드

```java
execution(* *ServiceImpl.upgrade*(..))
```

* `ServiceImpl` 클래스의 `upgrade`로 시작하는 메서드

---

#### 7) 파라미터 지정

파라미터 타입과 개수까지 지정할 수 있다.

```java
execution(* *(int, int))
```

* 파라미터가 정확히 `(int, int)` 인 메서드

```java
execution(* *())
```

* 파라미터가 없는 메서드

```java
execution(* *(..))
```

* 파라미터와 상관없는 모든 메서드

---



## 9. AOP
### 9.1. 지금까지 해온 것
1. 트랜잭션 서비스 추상화
2. 프록시와 데코레이트 패턴 적용
3. 다이나믹 프록시와 프록시 팩토리 빈 적용
4. 자동 프록시 생성과 포인트컷 적용
> 목적: 관심사가 같은 코드를 분리해 한 데 모으기 위한 것

### 9.2. AOP의 개념
트랜잭션 경계설정 같은 부가기능은 핵심 비즈니스 로직과는 성격이 다르다.
이런 부가기능은 일반적인 객체지향 설계만으로는 깔끔하게 분리하기 어려웠고, 프록시나 빈 후처리기 같은 기술을 사용해서 분리해왔다.

스프링은 이렇게 **핵심 기능과는 별도로 분리되는 부가기능**을 **애스펙트(Aspect)** 라고 부른다.

애스펙트는 애플리케이션의 핵심 비즈니스 로직은 아니지만, 여러 대상에 공통으로 적용되면서 의미를 가지는 모듈이다.
즉, 여러 곳에 흩어져 들어갈 수 있는 공통 관심사를 하나로 모듈화한 개념이다.

애스펙트는 보통 아래 두 요소로 구성된다.

* **어드바이스(Advice)**

  * 무엇을 할 것인가
  * 예: 트랜잭션 시작, 커밋, 롤백

* **포인트컷(Pointcut)**

  * 어디에 적용할 것인가
  * 예: `ServiceImpl` 클래스의 `upgrade*()` 메서드

정리하면 AOP는 부가기능을 핵심 로직에서 분리해서 별도의 모듈로 관리하고, 필요한 지점에 선언적으로 적용하는 방식
