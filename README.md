# SPRING BOOT

### 스프링 부트 - 핵심 기능 5가지

- WAS : Tomcat 같은 웹 서버를 내장해서 별도의 웹 서버를 설치하지 않아도 됨
- 라이브버리 관리
    - 손쉬운 빌드 구성을 위한 스타터 종속성 제공
    - 스프링과 외부 라이브러리의 버전을 자동으로 관리
- 자동 구성 : 프로젝트 시작에 필요한 스프링과 외부 라이브러리의 빈을 자동 등록
- 외부 설정 : 환경에 따라 달라져야 하는 외부 설정 공통화
- 프로덕션 준비 : 모니터링을 위한 메트릭, 상태 확인 제공

## 웹 서버와 서블릿 컨테이너

### 서블릿 컨테이너 초기화

- WAS를 실행하는 시점에 필요한 초기화 작업들이 있다. 서비스에 필요한 필터와 서블릿을 등록하고, 여기에 스프링을 사용한다면 스프링 컨테이너를 만들고, 서블릿과 스프링을 연결하는 디스페처 서블릿도 등록해야 한다
- WAS가 제공하는 초기화 기능을 사용하면, WAS 실행 시점에 이러한 초기화 과정을 진행할 수 있다
- 과거에는 `web.xml`을 사용해서 초기화했지만, 지금은 서블릿 스펙에서 자바 코드를 사용한 초기화도 지원한다

**서블릿 컨테이너와 스프링 컨테이너**


**서블릿 컨테이너 초기화 개발**

서블릿은 `ServletContainerInitializer` 라는 초기화 인터페이스를 제공한다. 서블릿 컨테이너는 실행 시점에 초기화 메서드인 `onStartup()`을 호출해준다. 여기서 어플리케이션에 필요한 기능들을 초기화 하거나 등록할 수 있다

```java
public interface ServletContainerInitializer {
	public void onStartup(Set<Class<?>> c, ServletContext ctx) throws
		ServletException;
	}
```

- `Set<Class<?>> c` : 조금 더 유연한 초기화 기능을 제공한다. `@HandlesTypes` 애노테이션과 함께
사용한다
- `ServletContext ctx` : 서블릿 컨테이너 자체의 기능을 제공한다. 이 객체를 통해 필터나 서블릿을 등록할 수 있다

서블릿 컨테이너 초기화 인터페이스를 구현후 WAS에 실행할 초기화 클래스를 알려줘야 한다.

```java
/META-INF/services/jakarta.servlet.ServletContainerInitializer
```

해당 경로에 클래스를 명시해줘야 한다

```java
hello.container.MyContainerInitV1
```


**서블릿을 등록하는 2가지 방법**

- `@WebServlet` 애노테이션
- 프로그래밍 방식

```java
public class HelloServlet extends HttpServlet {

@Override
protected void service(HttpServletRequest req, HttpServletResponse resp)
									throws ServletException, IOException {
		System.out.println("HelloServlet.service");
		resp.getWriter().println("hello servlet!");
		}
	}
```

- 이 서블릿을 등록하고 실행하면 다음과 같은 결과가 나온다.
    - 로그: `HelloServlet.service`
    - HTTP 응답: `hello servlet`

**어플리케이션 초기화**

서블릿 컨테이너는 조금 더 유연한 초기화 기능을 지원한다. 

```java
import jakarta.servlet.ServletContext;

public interface AppInit {
	void onStartup(ServletContext servletContext);
}
```

- 어플리케이션 초기화를 진행하려면 먼저 인터페이스를 만들어야 한다. 내용과 형식은 상관없고, 인터페이스는 꼭 필요하다

```java
/**
* http://localhost:8080/hello-servlet
*/
public class AppInitV1Servlet implements AppInit {

	@Override
	public void onStartup(ServletContext servletContext) {
		System.out.println("AppInitV1Servlet.onStartup");
		
		//순수 서블릿 코드 등록
		ServletRegistration.Dynamic helloServlet =
					servletContext.addServlet("helloServlet", new HelloServlet());
		helloServlet.addMapping("/hello-servlet");
	}
}
```

- 여기서는 프로그래밍 방식으로 `HelloServlet` 서블릿을 서블릿 컨테이너에 직접 등록한다
- HTTP로 `/hello-servlet` 를 호출하면 `HelloServlet` 서블릿이 실행된다

> 참고 - **프로그래밍 방식을 사용하는 이유**
`@WebServlet` 을 사용하면 애노테이션 하나로 서블릿을 편리하게 등록할 수 있다. 하지만 애노테이션 방식을 사용하면 유연하게 변경하는 것이 어렵다. 마치 하드코딩 된 것 처럼 동작한다. 아래 참고 예시를 보면 `/test` 경로를 변경하고 싶으면 코드를 직접 변경해야 바꿀 수 있다.
반면에 프로그래밍 방식은 코딩을 더 많이 해야하고 불편하지만 무한한 유연성을 제공한다.
예를 들어서`/hello-servlet` 경로를 상황에 따라서 바꾸어 외부 설정을 읽어서 등록할 수 있다.
서블릿 자체도 특정 조건에 따라서 `if` 문으로 분기해서 등록하거나 뺄 수 있다. 서블릿을 내가 직접 생성하기 때문에 생성자에 필요한 정보를 넘길 수 있다.
> 

```java
@HandlesTypes(AppInit.class)
	public class MyContainerInitV2 implements ServletContainerInitializer {
	
	@Override
	public void onStartup(Set<Class<?>> c, ServletContext ctx) throws
														ServletException {
		System.out.println("MyContainerInitV2.onStartup");
		System.out.println("MyContainerInitV2 c = " + c);
		System.out.println("MyContainerInitV2 container = " + ctx);
		for (Class<?> appInitClass : c) {
			try {
				//new AppInitV1Servlet()과 같은 코드
				AppInit appInit = (AppInit)
				appInitClass.getDeclaredConstructor().newInstance();
				appInit.onStartup(ctx);
			} catch (Exception e) {
				throw new RuntimeException(e);
			}
		}
	}
}
```

**어플리케이션 초기화 과정**

- `@HandlesTypes` 어노테이션에 어플리케이션 초기화 인터페이스를 지정한다
- 서블릿 컨테이너 초기화( `ServletContainerInitializer` )는 파라미터로 넘어오는 `Set<Class<?>> c`에 어플리케이션 초기화 인터페이스를 구현체들을 모두 찾아서 클래스 정보로 전달한다
- `appInitClass.getDeclaredConstructor().newInstance()`
    - 리플렉션을 사용해서 객체를 생성한다. 참고로 이 코드는 `new AppInitV1Servlet()` 과 같다
- appInit.onStartup(ctx_
    - 어플리케이션 초기화 코드를 직접 실행하면서 서블릿 컨테이너 정보가 담긴 `ctx`도 함께 전달한다

### 스프링 컨테이너 등록

**과정**

- 스프링 컨테이너 만들기
- 스프링 MVC 컨트롤러를 스프링 컨테이너에 빈으로 등록하기
- 스프링 MVC를 사용하는데 필요한 디스패처 서블릿을 서블릿 컨테이너에 등록하기


```java
@RestController
public class HelloController {
	
	@GetMapping("/hello-spring")
	public String hello() {
		System.out.println("HelloController.hello");
		return "hello spring!";
	}
}
```

- 스프링 컨트롤러 등록

```java
@Configuration
public class HelloConfig {
	
	@Bean
	public HelloController helloController() {
		return new HelloController();
	}
}
```

- 빈으로 등록

```java
/**
* http://localhost:8080/spring/hello-spring
*/
public class AppInitV2Spring implements AppInit {
	
	@Override
	public void onStartup(ServletContext servletContext) {
		System.out.println("AppInitV2Spring.onStartup");
		
		//스프링 컨테이너 생성
		AnnotationConfigWebApplicationContext appContext = new
		AnnotationConfigWebApplicationContext();
		appContext.register(HelloConfig.class);
		
		//스프링 MVC 디스패처 서블릿 생성, 스프링 컨테이너 연결
		DispatcherServlet dispatcher = new DispatcherServlet(appContext);
		
		//디스패처 서블릿을 서블릿 컨테이너에 등록 (이름 주의! dispatcherV2)
		ServletRegistration.Dynamic servlet =  servletContext.addServlet("dispatcherV2", dispatcher);
		
		// /spring/* 요청이 디스패처 서블릿을 통하도록 설정
		servlet.addMapping("/spring/*");
	}
}
```

- `AppInitV2Spring`는 `AppInit` 을 구현했다. `AppInit` 을 구현하면 애플리케이션 초기화 코드가 자동으로 실행된다. 앞서 `MyContainerInitV2` 에 관련 작업을 이미 해두었다.
- `AnnotationConfigWebApplicationContext` 가 스프링 컨테이너이다
    - `AnnotationConfigWebApplicationContext` 는 부모를 확인하면 `ApplicationContext` 인터페이스를 확인할 수다
    - 어노테이션 기반 설정과 웹 기능을 지원하는 스프링 컨테이너로 이해하면 된다
- `appContext.register(HelloConfig.class)`
    - 컨테이너에 스프링 설정을 추가한다.

**스프링 MVC 디스패처 서블릿 생성, 스프링 컨테이너 연결**

- `new DispatcherServlet(appContext)`
- 코드를 보면 스프링 MVC가 제공하는 디스패처 서블릿을 생성하고,  생성자에 앞서 만든 스프링 컨테이너를 전달하는 것을 확인할 수 있다.
- 이 디스패처 서블릿에 HTTP 요청이 오면 디스패처 서블릿을 해당 스프링 컨테이너에 들어있는 컨트롤러 빈을 호출한다

**디스패처 서블릿을 서블릿 컨테이너에 등록**

- `servletContext.addServlet("dispatcherV2", dispatcher)`
    - 디스패처 서블릿을 서블릿 컨테이너에 등록한다
- `/spring/*` 요청이 디스패처 서블릿을 통하도록 설정
    - `/spring/*` 이렇게 경로를 지정하면 `/spring` 과 그 하위 요청은 모두 해당 서블릿을 통하게 된다.
    - `/spring/hello-spring`
    - `/spring/hello/go`

**실행 과정 정리**

`/spring/hello-spring`

실행을 `/spring/*` 패턴으로 호출했기 때문에 다음과 같이 동작한다

- `dispatcherV2` 디스패처 서블릿이 실행된다.(/spring)
- `dispatcherV2` 디스패처 서블릿은 스프링 컨트롤러를 찾아서 실행한다.(/hello-spring)
    - 이때 서블릿을 찾아서 호출하는데 사용된 /spring을 제외한 /hello-spring가 매핑된 컨트롤러 HelloController의 메서드를 찾아서 실행한다


### 스프링 MVC 서블릿 컨테이너 초기화 지원

스프링 MVC는 서블릿 컨테이너 초기화 작업을 이미 만들어두었다. 덕분에 개발자는 서블릿 컨테이너 초기화 과정은 생략하고, 애플리케이션 초기화 코드만 작성하면 된다. 스프링이 지원하는 애플리케이션 초기화를 사용하려면 다음 인터페이스를 구현하면 된다.

```java
package org.springframework.web;

public interface WebApplicationInitializer {
	void onStartup(ServletContext servletContext) throws ServletException;
}
```

```java
/**
* http://localhost:8080/hello-spring
*
* 스프링 MVC 제공 WebApplicationInitializer 활용
* spring-web
* META-INF/services/jakarta.servlet.ServletContainerInitializer
* org.springframework.web.SpringServletContainerInitializer
*/
public class AppInitV3SpringMvc implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		System.out.println("AppInitV3SpringMvc.onStartup");
		
		//스프링 컨테이너 생성
		AnnotationConfigWebApplicationContext appContext = new
		AnnotationConfigWebApplicationContext();
		appContext.register(HelloConfig.class);
		
		//스프링 MVC 디스패처 서블릿 생성, 스프링 컨테이너 연결
		DispatcherServlet dispatcher = new DispatcherServlet(appContext);
		
		//디스패처 서블릿을 서블릿 컨테이너에 등록 (이름 주의! dispatcherV3)
		ServletRegistration.Dynamic servlet =
		servletContext.addServlet("dispatcherV3", dispatcher);
		
		//모든 요청이 디스패처 서블릿을 통하도록 설정
		servlet.addMapping("/");
	}
}
```

- `WebApplicationInitializer` 인터페이스를 구현한 부분을 제외하고는 이전의 `AppInitV2Spring`과 거의 같은 코드이다.
    - `WebApplicationInitializer` 는 스프링이 이미 만들어둔 애플리케이션 초기화 인터페이스이
    다.
    - 여기서도 디스패처 서블릿을 새로 만들어서 등록하는데, 이전 코드에서는 `dispatcherV2` 라고 했고, 여기서는 `dispatcherV3` 라고 해주었다. 참고로 이름이 같은 서블릿을 등록하면 오류가 발생한다.
- `servlet.addMapping("/")` 코드를 통해 모든 요청이 해당 서블릿을 타도록 했다.
    - 따라서 다음과 같이 요청하면 해당 디스패처 서블릿을 통해 `/hello-spring` 이 매핑된 컨트롤러 메서드가 호출된다.


스프링은 어떻게 `WebApplicationInitializer` 인터페이스 하나로 애플리케이션 초기화가 가능하게 할까? 스프링도 결국 서블릿 컨테이너에서 요구하는 부분을 모두 구현해야 한다.

`spring-web` 라이브러리를 열어보면 서블릿 컨테이너 초기화를 위한 등록 파일을 확인할 수 있다. 그리고 이곳에 서블릿 컨테이너 초기화 클래스가 등록되어 있다

```java
/META-INF/services/jakarta.servlet.ServletContainerInitializer
org.springframework.web.SpringServletContainerInitializer
```


- 초록색 부분은 스프링에서 제공하는 영역이다
- `WebApplicationInitializer` 인터페이스만 구현하면 편리하게 어플리케이션을 초기화 할 수 있다

### 내장 톰캣

- 스프링 부트는 내부에 톰캣 서버를 두어 별도의 서버 필요 없이 동작할 수 있게 한다
- jar는 jar를 담아서 인식하는 것이 불가능하지만 스프링 부트가 제공하는 jar은 jar안에 jar를 포함할 수 있는 특별한 구조를 제공한다

### **실행 가능 Jar**

- jar 내부에 jar를 포함하기 때문에 어떤 라이브러리가 포함되어 있는지 쉽게 확인할 수 있다
- jar 내부에 jar를 포함하게 때문에 내부에 같은 경로의 파일이 있어도 둘 다 인식할 수 있다
- 실행 가능 jar은 자바 표준이 아니고 스프링 부트에서 새롭게 정의한 것이다

**실행 가능 Jar내부 구조**

- `boot-0.0.1-SNAPSHOT.jar`
    - `META-INF`
        - `MANIFEST.MF`
    - `org/springframework/boot/loader`
        - `JarLauncher.class` : 스프링 부트 `main()` 실행 클래스
    - `BOOT-INF`
        - `classes` : 우리가 개발한 class 파일과 리소스 파일
        - `hello/boot/BootApplication.class`
        - `hello/boot/controller/HelloController.class`
        - …
    - `lib` : 외부 라이브러리
        - `spring-webmvc-6.0.4.jar`
        - `tomcat-embed-core-10.1.5.jar`
        - ...
    - `classpath.idx` : 외부 라이브러리 모음
    - `layers.idx` : 스프링 부트 구조 정보

**Jar 실행 정보**

`java -jar xxx.jar`를 실행하게 되면 우선 `META-INF/MANIFEST.MF` 파일을 찾는다. 그리고 여기서는 `Main-Class`를 읽어서 `main()`를 실행하게 된다

```java
META-INF/MANIFEST.MF
Manifest-Version: 1.0
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: hello.boot.BootApplication
Spring-Boot-Version: 3.0.2
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Spring-Boot-Layers-Index: BOOT-INF/layers.idx
Build-Jdk-Spec: 17
```

- `Main-Class`
    - 우리가 기대한 `main()` 이 있는 `hello.boot.BootApplication` 이 아니라 `JarLauncher` 라
    는 전혀 다른 클래스를 실행하고 있다.
    - `JarLauncher` 는 스프링 부트가 빌드시에 넣어준다. `org/springframework/boot/loader/
    JarLauncher` 에 실제로 포함되어 있다.
    - 스프링 부트는 jar 내부에 jar를 읽어들이는 기능이 필요하다. 또 특별한 구조에 맞게 클래스 정보도 읽어들여야 한다. 바로 `JarLauncher` 가 이런 일을 처리해준다. 이런 작업을 먼저 처리한 다음 `Start-Class:` 에 지정된 `main()` 을 호출한다.
- `Start-Class` : 우리가 기대한 `main()` 이 있는 `hello.boot.BootApplication` 가 적혀있다.
- 기타: 스프링 부트가 내부에서 사용하는 정보들이다.
    - `Spring-Boot-Version` : 스프링 부트 버전
    - `Spring-Boot-Classes` : 개발한 클래스 경로
    - `Spring-Boot-Lib` : 라이브러리 경로
    - `Spring-Boot-Classpath-Index` : 외부 라이브러리 모음
    - `Spring-Boot-Layers-Index` : 스프링 부트 구조 정보
- 참고: `Main-Class` 를 제외한 나머지는 자바 표준이 아니다. 스프링 부트가 임의로 사용하는 정보이다

## 라이브러리 관리

- 스프링 부트는 개발자 대신에 수 많은 라이브러리 버전을 직접 관리해준다
    - 버전 관리 기능을 사용하려면 `io.spring.dependency-management` 플러그인을 사용해야 한다

```java
plugins {
	id 'org.springframework.boot' version '3.0.2'
	id 'io.spring.dependency-management' version '1.1.0' //추가
	id 'java'
}
```

**dependency-management 버전 관리**

- `io.spring.dependency-management` 플러그인을 사용하면 `spring-boot-dependencies` 에 있는 다음 bom 정보를 참고한다.
- 참고로 `spring-boot-dependencies` 는 스프링 부트 gradle 플러그인에서 사용하기 때문에 개발자의 눈에 의존 관계로 보이지는 않는다.
- 스프링 부트가 제공하는 버전 관리는 스프링 자신을 포함해서 수 많은 외부 라이브러리의 버전을 최적화 해서 관리해준다. 또한 호환성 테스트를 했기 때문에 안전하게 사용할 수 있다

### 스프링 부트 스타터

- 수 많은 라이브러리를 직접 명시하는 것인 너무나 복잡한 일이다
- 스프링 부트 스타터는 이런 문제를 해결하기 위해 프로젝트를 시작하는데 필요한 라이브러리를 모아둔 `스프링 부트 스타터`를 제공한다

```java
dependencies {
//3. 스프링 부트 스타터
	implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

- 이것은 사용하기 편리하게 의존성을 모아둔 세트이다.
    - 이것을 하나 포함하면 관련 의존성 세트가 한번에 들어온다.
    - 스타터도 스타터를 가질 수 있다.
- 스프링과 웹을 사용하고 싶으면 `spring-boot-starter-web`
    - 스프링 웹 MVC, 내장 톰캣, JSON 처리, 스프링 부트 관련, LOG, YML 등등
- 스프링과 JPA를 사용하고 싶으면 `spring-boot-starter-data-jpa`
    - 스프링 데이터 JPA, 하이버네이트 등등

**스프링 부트 스타터 - 자주 사용하는 것 위주**

- `spring-boot-starter` : 핵심 스타터, 자동 구성, 로깅, YAML
- `spring-boot-starter-jdbc` : JDBC, HikariCP 커넥션풀
- `spring-boot-starter-data-jpa` : 스프링 데이터 JPA, 하이버네이트
- `spring-boot-starter-data-mongodb` : 스프링 데이터 몽고
- `spring-boot-starter-data-redis` : 스프링 데이터 Redis, Lettuce 클라이언트
- `spring-boot-starter-thymeleaf` : 타임리프 뷰와 웹 MVC
- `spring-boot-starter-web` : 웹 구축을 위한 스타터, RESTful, 스프링 MVC, 내장 톰캣
- `spring-boot-starter-validation` : 자바 빈 검증기(하이버네이트 Validator)
- `spring-boot-starter-batch` : 스프링 배치를 위한 스타터
