---
title: "스프링 MVC + CORS"
layout: post
category: Spring
---

## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. CORS(Cross-Origin Resource Sharing)란?](#2-corscross-origin-resource-sharing란)
  - [Same-Origin Policy (SOP)](#same-origin-policy-sop)
  - [CORS의 동작 방식](#cors의-동작-방식)
    - [Preflight Request](#preflight-request)
    - [Simple Request](#simple-request)
    - [Credentialed Request](#credentialed-request)
- [3. 스프링 MVC with CORS](#3-스프링-mvc-with-cors)
  - [CORS 요청과 응답은 어떻게 처리될까?](#cors-요청과-응답은-어떻게-처리될까)
  - [스프링 CORS 관련 설정](#스프링-cors-관련-설정)
    - [Local Configuration - @CrossOrigin](#local-configuration---crossorigin)
      - [@RequestMapping이 달린 핸들러 메서드](#requestmapping이-달린-핸들러-메서드)
      - [Controller 클래스](#controller-클래스)
      - [Controller 클래스와 Handler 메서드](#controller-클래스와-handler-메서드)
    - [Global Configuration](#global-configuration)
      - [Java Configuration](#java-configuration)
      - [XML Configuration](#xml-configuration)
- [글을 마치며](#글을-마치며)
- [출처](#출처)
- [각주](#각주)

## 1. 들어가면서

CORS가 무엇인지 부터, 스프링에서는 어떻게 CORS 관련 설정을 할 수 있는지까지 알아보자.

## 2. CORS(Cross-Origin Resource Sharing)란?

![](https://blog.4d.com/wp-content/uploads/2020/05/cors-1.jpg)
- 출처 : https://blog.4d.com
  
Cross-Origin Resource Sharing(이하 CORS)은 HTTP 헤더정보를 통해 애플리케이션이 다른 도메인(Origin)의 리소스에 접근할 수 있도록 하는 메커니즘을 말한다.

Cross-Origin을 한국어로 직역하면, 교차 출처이지만 직관적이지 않다. **다른**(Cross) **출처**(Origin)라고 이해하면 쉽다.

즉, 한 웹페이지 내에서 해당 웹페이지를 가져온 도메인과 다른 도메인(Cross-Origin)에서 자원을 가져오는 것이다.

N사의 홈 화면만 해도 웹페이지 내에 있는 이미지는 https://naver.com이 아닌 N사의 정적리소스를 위한 도메인으로 추측되는 https://s.pstatic.net 에서 가져온다. 

그렇다면 CORS는 왜 등장했을까? 이를 알기 위해선 Same-Origin Policy에 대한 이해가 있어야 한다.
### Same-Origin Policy (SOP)

SOP[^1]은 [RFC 6454](https://datatracker.ietf.org/doc/html/rfc6454#page-5)에서 소개되었고, 이름에서 알 수 있다 싶이 같은 출처(Same-Origin)에서만 리소스를 공유할 수 있다라는 규칙을 정의하고 있는 정책이다.

하지만 위의 N사의 홈화면의 사례와 같이, 다른 출처에서 리소스를 가져와서 사용하는 것은 매우 흔한 일이라 일부 예외를 둔게 CORS 정책이다.

많이들 헷갈려하는것 같아 다시 한번 명확하게 하면, **SOP**는 **다른 출처의 리소스 요청을 제한**하는 정책이고, **CORS**는 **다른 출처의 리소스를 요청할 수 있게** 하는 일종의 예외 정책이다.

(위에서 편의상 Origin을 도메인으로 설명하였지만 , 엄밀히 말하면 Origin은 도메인이 아니다. **Protocol, Host, Port를 합친 정보**를 Origin 이라고 한다.)


### CORS의 동작 방식

CORS가 동작하는 방식에는 아래의 세가지가 있다.

* Preflight Request
* Simple Request
* Credentialed Request

#### Preflight Request

Preflight라는 용어는 인쇄 전에 최종파일을 넘길때 문서를 **사전에 교열**하는 것을 의미한다.

비슷한 맥락으로 HTTP에서 Preflight는 본 요청이 **유효한지 검사하기 위해 보내는 예비 요청**이다.

아래의 그림처럼 본 요청인 GET 요청전에 OPTIONS 메서드를 통해 Origin 헤더에 검사 하고자 하는 출처를 담아서 보낸다.

서버가 보내준 응답의 Access-Control-Allow-Origin 값과 비교해본 후 이 응답이 유효한 응답인지 아닌지를 결정한다.

![](https://blog.kakaocdn.net/dn/bCIxDd/btq8Tst3gqA/JD6D9undrLdIw9zsafJxT0/img.png)
- 출처 : https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS


#### Simple Request

Simple Request은 위의 Preflight Request와 다르게, 예비 요청을 따로 보내지 않고, 
브라우저는 본 요청을 보내고 서버가 보낸 응답의 Access-Control-Allow-Origin 값을 통해 CORS 정책 위반 여부를 검사하는 방식이다.

Preflight Request와의 차이점은 예비요청을 보내는지 여부뿐이다.

![](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/simple-req-updated.png)
- 출처 : https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS


한가지 주의할 점은 아래의 세가지 조건을 모두 충족해야만 Simple Request를 사용할 수 있다.


* 요청 메서드는 다음중 하나여야만 한다.
  * GET
  * HEAD
  * POST

* Accept, Accept-Language, Content-Language, Content-Type, DPR, Downlink, Save-Data, Viewport-Width, Width 헤더만을 사용해야 한다.
  * Content-Type 헤더를 사용한다면, application/x-www-form-urlencoded, multipart/form-data, text/plain만 허용된다.

#### Credentialed Request

이름 그대로 인증된 요청을 사용하는 방법이다. 

기본적으로 브라우저가 제공하는 XMLHttpRequest 객체나 fetch API는 다른 사이트에 요청을 할 때, 별도의 옵션 없이 브라우저의 쿠키나 인증과 관련된 헤더를 함부로 요청에 담지 않는다.

하지만 아래의 조건을 지키면 CORS 정책을 위반하지 않고 쿠키나 인증과 관련된 헤더를 요청에 담는것이 가능하다.

* 브라우저단에서 XMLHttpRequest객체를 생성할 때 withCredentials 플래그를 true로 설정한다.
* 서버단에서 응답헤더에 Access-Control-Allow-Credentials : true로 설정해주면 된다.

만약 응답헤더에 Access-Control-Allow-Credentials : true 가 포함되어 있지 않다면 브라우저에서 해당 응답을 사용할 수 없다.

(참고로 아래와 같이 만약 요청의 Origin과 응답의 Access-Control-Allow-Origin이 같으면, withCredentials 플래그를 따로 지정하지 않아도 된다.)

![](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/cred-req-updated.png)
- 출처 : https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS



## 3. 스프링 MVC with CORS

CORS에 대한 개념은 어느정도 설명하였으니, 본격적으로 스프링에서 CORS 요청이 어떻게 처리되고, CORS 관련 설정하는 법에 대해 알아보자.

### CORS 요청과 응답은 어떻게 처리될까?

스프링 MVC의 HandlerMapping 구현체들은 기본적으로 CORS에 관련된 기능을 제공한다.

HandlerMapping은 Preflight 요청을 직접적으로 다루지만, Simple Request와 Credentialed Request는 CorsFilter에 의해 인터셉트되고 CorsProcessor 구현체에 의해 검증된 후에 CORS 관련 헤더(Access-Control-Allow-Origin 등)를 추가한다. 코드로 직접 살펴보자.

CorsFilter 코드를 살펴보면 DefaultCorsProcessor 오브젝트를 필드로 가지고 있고, doFilterInternal() 내에서 processRequest()를 통해 파라미터로 CorsConfiguration을 전달받아 응답을 처리하는 것을 확인할 수 있다.
~~~java
public class CorsFilter extends OncePerRequestFilter {

    ...

    private CorsProcessor processor = new DefaultCorsProcessor();

    ...

    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        CorsConfiguration corsConfiguration = this.configSource.getCorsConfiguration(request);
        boolean isValid = this.processor.processRequest(corsConfiguration, request, response);
        if (isValid && !CorsUtils.isPreFlightRequest(request)) {
            filterChain.doFilter(request, response);
        }
    }
}
~~~

CorsProcessor 인터페이스 CORS 요청 처리에 중추적인 역할을 하지만 구성은 매우 단순하다. 아래와 같이 CorsProcessor은 단 하나의 메서드만 가지고 있고, 기본적으로 사용되는 구현체인 DefaultCorsProcessor는 [CORS W3C recommendation](https://fetch.spec.whatwg.org/)스펙에 맞게 processRequest() 메서드를 구현하고 있다.

~~~java
public interface CorsProcessor {
    boolean processRequest(@Nullable CorsConfiguration var1, HttpServletRequest var2, HttpServletResponse var3) throws IOException;
}
~~~

CORS 관련 요청과 응답이 처리되는 과정을 살펴보았으니, 스프링에서 CORS 관련 설정하는 법에 대해 알아보자.

### 스프링 CORS 관련 설정

스프링에선 CORS관련 설정을 **지역적**으로 선언하는 법과 **전역적**으로 설정하는 방법이 있다. 차례대로 알아보자.

#### Local Configuration - @CrossOrigin

@CrossOrigin 어노테이션은 아래와 같이 @CrossOrigin 어노테이션이 달린 컨트롤러가 Cross-Origin 요청할 수 있게 해준다.

@CrossOrigin 어노테이션은 여러 곳에 배치될 수 있다. 
핸들러 메서드나 컨트롤러 클래스에 지정할 수도 있고, 이를 혼합하여 사용할 수 있다.
예시와 함께 자세히 알아보자.

##### @RequestMapping이 달린 핸들러 메서드

~~~java
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
~~~

만약에 어떤 설정도 하지 않으면 @CrossOrigin 어노테이션은 기본적으로 
* 모든 Origin을 허용한다.
* @RequestMapping 어노테이션에 지정된 HTTP 메서드만 허용된다.
* Preflight 응답은 30분간 Caching된다.

##### Controller 클래스

Controller 클래스에 지정하면 Controller 클래스 내의 모든 핸들러 메서드에 적용된다.
value값을 설정하여 configuration을 커스텀할 수 있다.

~~~java

@CrossOrigin(origins = "https://minseong.kim", maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @RequestMapping(method = RequestMethod.GET, path = "/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @RequestMapping(method = RequestMethod.DELETE, path = "/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
~~~

그리고 아래와 같이 @CrossOrigin이 구성되어 있어 아래와 같은 속성들을 지정할 수 있다.

~~~java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CrossOrigin {

    @AliasFor("origins")
    String[] value() default {};

    @AliasFor("value")
    String[] origins() default {};

    String[] originPatterns() default {};

    String[] allowedHeaders() default {};

    String[] exposedHeaders() default {};

    RequestMethod[] methods() default {};

    String allowCredentials() default "";

    long maxAge() default -1L;

}
~~~

##### Controller 클래스와 Handler 메서드

아래와 같이 클래스레벨과 메서드레벨에 동시에 지정하면 스프링에서 CORS 설정을 합쳐준다.

두 개의 핸들러 메서드엔 maxAge = 3600이 적용된다.
그리고 retrieve() 핸들러는 https://minseong.kim만 허용하고, remove() 핸들러는 모든 origin을 허용한다.

~~~java
@CrossOrigin(maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin("https://minseong.kim")
    @RequestMapping(method = RequestMethod.GET, "/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @RequestMapping(method = RequestMethod.DELETE, path = "/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
~~~

#### Global Configuration

Controller 메서드 레벨에서도 CORS Configuration를 정의할 수 있지만 애플리케이션 전역에 CORS Configuration을 정의할 수 있다.

기본적으로 GET, HEAD, POST 메서드가 허용된다.


##### Java Configuration

아래는 와일드카드를 활용해 모든 CORS 요청을 허용한 예이다.

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**");
    }
}
~~~

registry.addMapping 메서드에서 추가적으로 설정을 할 수 있는 CorsRegistration 객체를 리턴한다.

추가적으로 allowedMethods, allowedHeaders, exposedHeaders, maxAge, allowCredentials과 같은 응답 헤더들을 커스텀할 수 있는 옵션들이 있다.


##### XML Configuration

아래는 자바로 설정을 추가한 것과 같은 효과를 가진 XML를 통해 설정하는 방법이다.
~~~java
<mvc:cors>
    <mvc:mapping path="/**" />
</mvc:cors>
~~~

그리고 아래와 같이 여러 Properties와 함께 CORS 매핑을 선언할 수 있다.

~~~java
<mvc:cors>

    <mvc:mapping path="/api/**"
        allowed-origins="http://domain1.com, http://domain2.com"
        allowed-methods="GET, PUT"
        allowed-headers="header1, header2, header3"
        exposed-headers="header1, header2" allow-credentials="false"
        max-age="123" />

    <mvc:mapping path="/resources/**"
        allowed-origins="http://domain1.com" />

</mvc:cors>
~~~



## 글을 마치며

웹이 처음 고안됐을때만 해도 웹페이지는 매우 단순한 형태였다.
웹은 단순히 문서 공유를 위해서 고안되었으며, 웹이 진화함에 따라 점점더 복잡해졌고,

## 출처

- [Spring Docs - CorsFilter](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/web/filter/CorsFilter.html)

- [Spring Docs - CorsProcessor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/cors/CorsProcessor.html)

- [Mozilla Web Docs - Cross-Origin Resource Sharing (CORS)
](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)


- [What is ‘CORS’? What is it used for?](https://medium.com/@electra_chong/what-is-cors-what-is-it-used-for-308cafa4df1a)

- [CORS with Spring](https://www.baeldung.com/spring-cors)

- [CORS는 왜 이렇게 우리를 힘들게 하는걸까?](https://evan-moon.github.io/2020/05/21/about-cors/)


## 각주

[^1]: Preflight 요청 : Preflight Request는 actual 요청 전에 인증 헤더를 전송하여 서버의 허용 여부를 미리 체크하는 테스트 요청이다.