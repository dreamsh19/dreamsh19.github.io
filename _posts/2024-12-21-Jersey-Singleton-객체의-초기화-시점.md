---
title: Jersey Singleton 객체의 초기화 시점
categories:
  - Jersey
tags:
  - Jersey
  - Java
published: true
created: 2024-12-22 (일) 03:17:37
modified: 2024-12-22 (일) 23:37:52
---

## 이슈
- 현재 운영 중인 Jersey 기반의 서비스 중에 애플리케이션 구동 시점에 응답시간이 지연되는 이슈가 있었다.
- 구동 시점에만 이슈가 발생했기 때문에 무엇인가 초기화가 늦어지는 것으로 추측하고, 살펴보기 시작했다.

## 클래스의 초기화 시점

초기화 지연을 의심하여 애플리케이션 내에서 사용하는 Singleton 객체의 초기화 시점에 로그를 찍도록 했다.
예상은 애플리케이션 구동 시점에 모두 초기화될 것으로 예상하였으나, 결과는 놀랍게도 실제 호출 시점에 초기화가 이루어졌다. 그래서 이러한 현상이 발생하는 이유를 찾아보기 시작했다.

초기화가 늦게 이루어지는 클래스는 아래와 같이 Singleton 형태로 구현되어 있었고, 사용하는 쪽에서는 ServiceA.getInstance() 형태로 호출하고 있었다.
```java
public class ServiceA {  
	
	private static final ServiceA INSTANCE = new ServiceA();  
  
    private ServiceA() {}
  
    public static ServiceA getInstance() {  
        return INSTANCE;  
    }
}
```

코드에서 보면 명확하듯이, 클래스가 로딩되는 시점에 static 변수인 INSTANCE 객체가 초기화될 것이라고 생각했고, 그래서 클래스는 애플리케이션 구동 시점에 로딩될테니, 애플리케이션 구동 시점에 Singleton 객체인 INSTANCE가 초기화된다고 생각하고 있었다.

그러나 관련해서 좀 더 찾아보다보니, 여기서 한가지 놓치고 있었던 전제가 있었다.
클래스가 로딩되는 시점에 초기화가 되는 것은 맞지만, 애플리케이션 구동 시점에 클래스가 로딩되는가? 에 대한 전제이다.

그리고 결론부터 이야기하면, 자바의 스펙상, 필요하지 않는 클래스는 미리 로딩하지 않는다.

### JLS 12.4.1에 명시된 스펙

JLS(Java Language Spec)에는 클래스의 초기화 시점을 아래와 같이 정의하고 있다.

> A class or interface type T will be initialized immediately before the first occurrence of any one of the following:
>
>  * T is a class and an instance of T is created.
>  * A `static` method declared by T is invoked.
>  * A `static` field declared by T is assigned.
>  * A `static` field declared by T is used and the field is not a constant variable ([§4.12.4](https://docs.oracle.com/javase/specs/jls/se8/html/jls-4.html#jls-4.12.4 "4.12.4. final Variables")).

위와 같이 초기화 시점에 대한 구체적인 케이스에 대해 표현되어있지만, 요약하자면 실제로 "필요한 시점에 초기화한다"가 핵심이며, 이때 필요한 시점이라는 것은 클래스에 정의된 필드, 메소드에 접근하는 최초 시점을 의미한다. 결국, 클래스 로딩 역시 최대한 lazy하게 수행하는 것이 자바의 스펙이다.
(클래스 초기화에 대한 참고용으로 위 JLS 문서의 클래스 초기화 예제들을 보면, 필요하지 않는 클래스는 절대 초기화하지 않도록 하는, 극단적으로 lazy하게 수행하도록 강제하는 것을 확인할 수 있다.)

---
그럼 위와 같은 스펙에 대한 이해를 바탕으로 처음 ServiceA의 코드를 살펴보자.
결국 ServiceA의 코드는 애플리케이션 로딩 시점에 클래스 로딩을 강제할 요소가 전혀 없다. 그렇기 때문에 `ServiceA.getInstance()`함수가 호출되는 시점에서야 ServiceA의 로딩이 일어나게 된다.

## 해결방안

처음으로 돌아가, 결국 문제의 원인은 ServiceA의 초기화 시점이 애플리케이션 구동 시점이 아닌 실제 사용 시점이기 때문이다.
그렇다면 해결 방법은 ServiceA를 애플리케이션 구동 시점에 초기화하도록 하면 된다.
이를 위한 구체적인 방법은 아래와 같다.

아래 해결방안에 대해 부연설명을 추가하자면, ServiceA.getInstance() 를 Resource 레벨(스프링에서의 Controller)에서의 생성자에서 호출하고 있었는데, 결국 근본적으로 Resource가 지연 초기화 방식으로 동작하고 있었고, 그것으로 인해 발생한 문제라고 볼 수 있다.
Resource가 즉시 초기화가 된다면, Resource 초기화 시점에 ServiceA도 초기화가 될 것이기 때문에 결국 Resource를 즉시 초기화할 수 있는 방법에 대해 검토하였다.

### 1. Spring과의 통합

Spring은 필요한 빈을 애플리케이션 구동 시점에 eager하게 모두 로딩하는 것이 디폴트 설정이다.
그리고 아래와 같이 jersey-spring 의존성을 추가하면 Spring beans와 Jersey를 함께 사용할 수 있다.

```java
<dependency>
	<groupId>org.glassfish.jersey.ext</groupId>
	<artifactId>jersey-spring3</artifactId>
	<version>${project.version}</version>
</dependency>
```

예제 코드는 아래를 참고하면 된다.
![image](https://github.com/user-attachments/assets/db10426a-9d20-4f95-a46b-ae246dc9f95d)
- [jersey/examples/helloworld-spring-webapp/src/main/java/org/glassfish/jersey/examples/helloworld/spring/SpringRequestResource.java at 2.28 · eclipse-ee4j/jersey · GitHub](https://github.com/eclipse-ee4j/jersey/blob/2.28/examples/helloworld-spring-webapp/src/main/java/org/glassfish/jersey/examples/helloworld/spring/SpringRequestResource.java)

그러나 이 방법은 애플리케이션의 복잡도를 증가시키기 때문에 기각하였다.

### 2. @Immediate를 통한 해결

위처럼 새로운 의존성을 추가하지 않고, 기존의 Jersey의 기능을 활용하여 해결할 수 있는 방법을 찾고자했다.

Jersey는 의존성 주입(DI) 구현체로서 내부적으로 [hk2](https://eclipse-ee4j.github.io/glassfish-hk2/)를 사용한다.
그리고 hk2에서도 `@Immediate` annotation을 통해 즉시 초기화를 지원한다.

참고로, hk2는 디폴트 초기화 전략으로 지연 초기화를 채택하고 있다.
![image](https://github.com/user-attachments/assets/5f0e4377-1b37-4658-881f-3d00754a6969)
- [glassfish-hk2/hk2-api/src/main/java/org/glassfish/hk2/internal/ImmediateHelper.java at f80f98503c51ca9fd2f2b57def329b05dacab73a · eclipse-ee4j/glassfish-hk2 · GitHub](https://github.com/eclipse-ee4j/glassfish-hk2/blob/f80f98503c51ca9fd2f2b57def329b05dacab73a/hk2-api/src/main/java/org/glassfish/hk2/internal/ImmediateHelper.java#L93)
- ![image](https://github.com/user-attachments/assets/fb5f702d-788c-4d27-8301-d89b8040a14f)

그렇기 때문에 hk2에서 즉시 초기화를 하기 위해서는 `@Immediate` annotation을 통해 명시적으로 지정을 해주어야한다.

```java
@Path("/monitor/l7check.html")  
@Immediate  
public class L7Resource {
	// 아래 세부 구현
}
```

그러나 이렇게만 설정하면 여전히 즉시 초기화가 일어나지 않는다. 그 이유는 위에서의 전역적인 초기화 전략이 여전히 지연 초기화로 설정되어 있기 때문이다.
따라서 아래와 같은 설정을 통해 전역 설정을 Immediate을 활성화하도록 변경해주어야한다.
```java
@ApplicationPath("/")  
public class ApplicationConfig extends ResourceConfig {  
  
    @Inject  
    public ApplicationConfig(ServiceLocator serviceLocator) {  
       ServiceLocatorUtilities.enableImmediateScope(serviceLocator);  
       register(L7Resource.class);
       // 후략
    }
```

위와 같이 설정하면, 애플리케이션 구동 시점에 싱글턴 객체가 모두 애플리케이션 구동 시점에 즉시 초기화되는 것을 확인할 수 있다.

## 번외1. Singleton의 구현에 따른 초기화 시점 차이

아래 두가지 코드는 Java에서 싱글톤의 구현으로 흔하게 볼 수 있는 구현이다.
```java
public class ServiceA {  
	
	private static final ServiceA INSTANCE = new ServiceA();  
  
    private ServiceA() {
	    System.out.println("ServiceA instantiated");
    }
  
    public static ServiceA getInstance() {  
        return INSTANCE;  
    }
    
    public static void doNothing() {}
}
```

```java
public class ServiceB {  
  
    private ServiceB() {
	    System.out.println("ServiceB instantiated");
    }
  
    private static class Singleton {  
        private static final ServiceB INSTANCE = new ServiceB();  
    }  
  
    public static ServiceB getInstance() {  
        return Singleton.INSTANCE;  
    }  
  
    public static void doNothing() {}
}
```

실제로 서비스 중인 코드에도 두가지 구현이 혼재되어 있었다.
사실 이 이슈를 살펴보기 전까지는 A와 B의 차이가 없다고 생각해서 무엇을 쓰든 상관없다고 생각했다. 왜냐하면, 애플리케이션 구동 시점에 모든 클래스가 항상 로딩된다고 생각했고, 그런 관점에서는 두 코드의 동작에 차이가 없기 때문이다.
그런데, 위에서 살펴본 것처럼 모든 클래스는 항상 로딩되는 것이 아니고 필요한 시점에서야 lazy하게 로딩됨을 알았다. 이 사실을 알고나면, 두 가지 구현은 실제 동작에서 분명한 차이가 발생한다.

실제로 아래와 같은 코드를 수행해보면 그 차이를 명확하게 확인할 수 있다.
```java
import org.junit.jupiter.api.Test;  
  
public class TestRunner {  
    
    @Test
    public void testA(){  
        ServiceA.doNothing();  
    }  
  
    @Test  
    public void testB(){  
        ServiceB.doNothing();  
    }
}
```

테스트 수행 결과 `ServiceA instantiated`만 출력된다.

ServiceA는 doNothing() static 메소드를 호출하면서 ServiceA 클래스가 로딩되고, 로딩될때 static 변수인 INSTANCE가 초기화된다.
그러나, ServiceB는 doNothing() static 메소드를 호출할때 ServiceB 클래스가 마찬가지로 로딩되지만, 이것이 ServiceB.Singleton 클래스의 로딩을 강제하지 않기 때문에, INSTANCE는 초기화되지 않는다. 그리고 실제로 ServiceB.getInstance()를 호출하는 시점에서야 ServiceB.Singleton 클래스가 로딩되면서 INSTANCE 객체가 초기화된다.
그리고 ServiceB의 구현 방식을 Lazy Holder 방식이라고 부르는 것 같고, 실제로 가장 많이 쓰인다고 한다.

## 번외2. Spring과 지연 초기화

스프링 공식 문서에 따르면, 지연 초기화의 단점에 대해 아래와 같이 설명하면서, 디폴트로 지연 초기화가 아닌 즉시 초기화를 채택한 이유에 대해 설명하고 있다.
>  There are a few downsides to lazy initialization that mean we believe it's better to opt-in once you have decided it makes sense to do so. Due to classes no longer being loaded and beans no longer being created until they're needed, it's possible for lazy initialization to mask a problem that previously would have been identified at startup. Such problems can include no class def found errors, out of memory errors, and failures due to misconfiguration.

결국 fail-fast 관점에서 지연 초기화는 별로 바람직하지 않고, 빈 초기화에 실패한다면 애플리케이션 구동 시점에 빠르게 실패하는 것이 대부분의 경우 바람직하다.
(그외에 지연 초기화가 의미 있는 경우에 대해 궁금하다면 [Lazy Initialization in Spring Boot 2.2](https://spring.io/blog/2019/03/14/lazy-initialization-in-spring-boot-2-2) 문서를 참고하면 좋을듯 하다)

참고로 Spring에서도 지연 초기화를 활성화하고 싶다면 빈에 `@Lazy` annotation을 지정함으로써 활성화할 수 있으며, Spring boot 2.2 부터는 `spring.main.lazy-initialization=true` 설정을 통해 지연 초기화를 전역적으로 활성화할 수 있다.

## References

- https://docs.oracle.com/javase/specs/jls/se8/html/jls-12.html#jls-12.4.1
- [web services - How to initialize injected value before REST call? - Stack Overflow](https://stackoverflow.com/questions/35344767/how-to-initialize-injected-value-before-rest-call)
- [java - How do I get my Jersey 2 Endpoints to eagerly initialize on startup? - Stack Overflow](https://stackoverflow.com/questions/28114602/how-do-i-get-my-jersey-2-endpoints-to-eagerly-initialize-on-startup)
- [rest - Initialize singleton in Java Jersey 2 JAX-RS - Stack Overflow](https://stackoverflow.com/questions/36186102/initialize-singleton-in-java-jersey-2-jax-rs)
- [GlassFish HK2 - Dependency Injection Kernel](https://eclipse-ee4j.github.io/glassfish-hk2/)
- [GitHub - eclipse-ee4j/glassfish-hk2: Dynamic dependency injection framework](https://github.com/eclipse-ee4j/glassfish-hk2)
- [Spring Boot Features](https://docs.spring.io/spring-boot/docs/2.2.3.RELEASE/reference/html/spring-boot-features.html#boot-features-lazy-initialization)
- [\[Spring\] Bean Lazy Initialization 사용법](https://effortguy.tistory.com/266)
- [Lazy Initialization in Spring Boot 2.2](https://spring.io/blog/2019/03/14/lazy-initialization-in-spring-boot-2-2)
- [단일체 패턴 (Singleton Pattern)](https://robin00q.tistory.com/87)
