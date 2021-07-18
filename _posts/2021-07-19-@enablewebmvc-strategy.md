---
title: "@EnableWebMvc을 선언하면 내부적으로 어떤일이 일어날까?"
layout: post
category: Spring
---

## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. @EnableWebMvc](#2-enablewebmvc)
- [3. DelegatingWebMvcConfiguration](#3-delegatingwebmvcconfiguration)
- [4. WebMvcConfigurationSupport](#4-webmvcconfigurationsupport)
- [5. Composite 패턴](#5-composite-패턴)
- [6. 어노테이션에 대한 팁](#6-어노테이션에-대한-팁)
- [글을 마치며](#글을-마치며)
- [출처](#출처)

## 1. 들어가면서

[지난 글](https://www.minseong.kim/dispatcherservlet-default-strategy.html) 알아봤던 기본전략에 이어 @EnableWebMvc의 역할과 이 어노테이션을 활성화 하면 내부적으로 어떤일이 일어나는지에 대해 알아보자.
## 2. @EnableWebMvc

우선 @EnableWebMvc는 어떤 역할을 할까?

[스프링 공식문서](https://docs.spring.io/spring-framework/docs/3.1.0.RC1/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html)에는 아래와 같이 기술하고 있다.

> Enables **default Spring MVC configuration** and registers Spring MVC infrastructure components expected by the DispatcherServlet. Use this annotation on an @Configuration class. In turn that will import DelegatingWebMvcConfiguration, which provides default Spring MVC configuration.

스프링 MVC의 **기본 설정을 사용가능**하게 하고, DispatcherServlet에서 사용할 
@Configuratiton이 선언된 클래스에서 이 어노테이션을 사용하고, DelegatingWebMvcConfiguration를 import 하여 기본 스프링 MVC 설정들을 제공한다.

즉 @EnableWebMvc 어노테이션은 기본 전략들을 설정 해주고 필요로 하는 전략들을 손쉽게 등록하여 커스텀할 수 있다.

@EnableWebMvc 어노테이션은 아래와 같이 정의되어 있는데 **핵심은 DelegatingWebMvcConfiguration 클래스를 Import 하고 있다는 것이다.**

~~~java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({DelegatingWebMvcConfiguration.class})
public @interface EnableWebMvc {
}
~~~
그럼 과연 DelegatingWebMvcConfiguration는 어떤 역할을 하고 그 내부는 어떻게 구성되어 있을까?

## 3. DelegatingWebMvcConfiguration

스프링 공식문서에 DelegatingWebMvcConfiguration에 대해 다음과 같이 설명하고 있다.

> A subclass of WebMvcConfigurationSupport that detects and delegates to all beans of type WebMvcConfigurer allowing them to customize the configuration provided by WebMvcConfigurationSupport. This is the class actually imported by @EnableWebMvc.

> DelegatingWebMvcConfiguration은 
WebMvcConfigurationSupport가 제공하는 **Configuration을 커스텀 할 수 있도록** WebMvcConfigurer 타입의 모든 빈들을 감지하고 위임하는 WebMvcConfigurer의 서브클래스이다.

즉 DelegatingWebMvcConfiguration을 통해 WebMvcConfigurer 타입의 빈들을 통해 사용할 전략들을 
 커스텀 할 수 있다.


그리고 아래의 WebMvcConfigurerComposite 타입의 필드에 [Composite 패턴](#composite-패턴)을 통해 여러 WebMvcConfigurer타입의 오브젝트들을 관리하고 있다.

~~~java
	private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
~~~

직접 WebMvcConfigurerComposite 클래스 내부를 확인해보면 List 컬렉션을 통해 여러 WebMvcConfigurer 타입의 요소들을 보관하고 있는것을 확인할 수 있다.

~~~java
	private final List<WebMvcConfigurer> delegates = new ArrayList<>();
~~~

정리를 해보면 DelegatingWebMvcConfiguration은 Composite 패턴을 활용해 여러 WebMvcConfigurer 오브젝트들에게 전략을 커스텀하는 것을 위임(Delegate)하는 것이다.

여기까지 DelegatingWebMvcConfiguration의 역할과 내부구조에 대해 알아봤다.

전략들을 커스텀하는 것에 대해 알아보았으니, 커스텀 하기 전에 **기본적으로 어떤 전략을 사용하는지에 대해서도 알아보자.**

## 4. WebMvcConfigurationSupport

WebMvcConfigurationSupport은 @EnableWebMvc 활성화 했을 때 별도의 커스텀 하지 않았을 때 기본 전략들을 가지고 있는 클래스이고
DeletegatingWebMvcConfigution의 상위 클래스이다.


![](https://blog.kakaocdn.net/dn/bckZwJ/btq9MEO7wqx/9M1E1mSdCewQ2zoFsoLOC0/img.png)


[스프링 공식문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurationSupport.html)에서 매우 드문경우(?)이지만 친절하게 WebMvcConfigurationSupport에서 사용하는 기본전략에 대해 기술하고 있다.

WebMvcConfigurationSupport 클래스는 다음과 같은 HandlerMapping 들을 등록한다. (order가 작을수록 우선순위이다.)

* RequestMappingHandlerMapping (order = 0)
* HandlerMapping (URL경로를 뷰의 이름으로 매핑하기 위함) (order = 1)
* BeanNameUrlHandlerMapping (order = 2)
* RouterFunctionMapping (order = 3)
* HandlerMapping (정적 자원 요청을 처리하기 위함) (order = Integer.MAX_VALUE-1)
* HandlerMapping (기본 서블릿으로 요청을 전달하기 위함) (order = Integer.MAX_VALUE)
 
아래의 HandlerAdapter를 등록한다.

* RequestMappingHandlerAdapter
* HttpRequestHandlerAdapter 
* SimpleControllerHandlerAdapter
* HandlerFunctionAdapter

그리고 아래의 HandlerExceptionResolver들을 등록한다.
<!-- Registers a HandlerExceptionResolverComposite with this chain of exception resolvers: -->
* ExceptionHandlerExceptionResolver
* ResponseStatusExceptionResolver
* DefaultHandlerExceptionResolver

글의 주제상 해당 전략들이 어떤 역할을 하는지 구체적으로 다루지 않으니 잘 설명된 다른 글을 참고하길 바란다.
## 5. Composite 패턴

DelegatingWebMvcConfiguration을 설명할 때 Composite 패턴을 언급했었다.

스프링에서 자주 사용되는 패턴이니 이참에 짚고 넘어가도록 하자.

> made up of various parts or elements.

사전적 의미는 '**여러개의 부분이나 요소들로 이뤄져 있는**' 이라는 뜻이다.


사전적 의미에서 유추할 수 있다 싶이 프로그래밍에서는 **여러 오브젝트**들을 트리 구조로 구성하고 이 구조를 마치 하나의 객체처럼 사용하는 것이다.
하나의 객체처럼 처리하니 구체적인 오브젝트에 대해서 신경쓸 필요없는 장점이 있다.


~![](https://refactoring.guru/images/patterns/content/composite/composite-2x.png?id=8847e6f8e2cb892ed222)
* 출처 : https://refactoring.guru/design-patterns/composite


그렇다면 컴포지트 패턴은 어떻게 구현해야 할까?

![](https://refactoring.guru/images/patterns/diagrams/composite/structure-en-indexed-2x.png?id=a5bbb62b1bc218bc5261)

다음과 같은 요소로 이뤄져 있다.

1. Component : Leaf 클래스와 전체에 해당하는 Composite 클래스의 공통 인터페이스.
2. Composite : 여러 Component를 포함하며 Leaf에게 구현을 위임한다. Component 인터페이스의 메서드를 사용할 뿐이고 Component의 구현체의 동작에 대해 알 수 없다.
3. Client : Component 인터페이스의 메서드를 실행하는 주체.
4. Leaf : Component 인터페이스를 구현한 구체 클래스.


자, 그럼 4개의 구성 요소가 위에서 알아본 것들 중에 어떤 것에 해당하는지 알아보자.

DelegatingWebMvcConfiguration 클래스 내부에 아래와 같이 WebMvcConfigurerComposite 타입의 필드를 가지고 있었고,

~~~java
	private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
~~~

또한 WebMvcConfigurerComposite 내부에 List 컬렉션을 통해 WebMvcConfigurer를 관리하고 있었다.

~~~java
	private final List<WebMvcConfigurer> delegates = new ArrayList<>();
~~~

그리고 WebMvcConfigurer는 스프링 MVC 관련 설정들을 커스텀 하기 위한 인터페이스이다.

이제 느낌이 오는가? Component는 WebMvcConfigurer에 해당하고, 여러 Component를 관리하는 Composite는 WebMvcConfigurerComposite에 해당한다. 

그리고 DelegatingWebMvcConfiguration은 WebMvcConfigurerComposite를 사용하는 클라이언트라고 할 수 있다.


혹시 '그냥 WebMvcConfigurer들이 List에 담겨 있는데 트리구조라고 할 수 있나?' 라고 생각할 수 있다. 하지만 **하나의 루트와 나머지는 모두 리프로만 이뤄진 구조도 트리이다.**

## 6. 어노테이션에 대한 팁


추가적으로 어노테이션에 관한 팁을 주자면, 처음 보는 어노테이션이 있거나 그 어노테이션을 선언했을 때 어떤일이 일어나는지 궁금하다면 그 어노테이션이 어떻게 정의되어있는지 직접 타고 들어가보는 것을 추천한다.

어노테이션의 기본 정보인 Retention, Target을 통해 어떤 부분에 적용이 가능하고 이 어노테이션 정보가 언제 까지 유지되겠구나 라는 정보를 알 수 있고
@Import 메타어노테이션을 통해

이 글에서 다룬 @EnableWebMvc로 예를 들면 다음과 같은 추측이 가능하다. 

> "Retetion이 Runtime이니, 런타임까지 해당 정보가 유지되겠고, Target이 Type이니 클래스에 해당 어노테이션을 선언할 수 있겠네. DeletegatingWebMvcConfiguration클래스를 Import해서 사용하고 있네? 이 클래스를 확인해보면, 이 어노테이션의 역할에 대해 명확하게 알 수 있겠군. 한번 확인해보자~"
 
이 추측을 통해 DeletegatingWebMvcConfiguration의 역할을 공부하여 알게되면, 해당 어노테이션을 선언했을 대 일어나는일을 정확하게 파악할 수 있다.

단지 몇글자 밖에 되지 않는 어노테이션이지만, 내부적으로 많은 정보를 내포하고 있고, 선언하는 순간 스프링에서 많은일이 일어난다.

만약 무지성(?)으로 특정 어노테이션에 대한 이해가 없는 상태에서 난발하게 된다면 장애가 발생해도 그 원인을 찾기 어려워질 수 있다.

따라서 사소해 보이는 어노테이션이라 할지라도, 습관적으로 타고 들어가서 내부적으로 포함하고 있는 정보에 대해서 확인하는 습관을 들이도록 하자.

## 글을 마치며


토비의 스프링 가장 첫 장에 이런 문구가 등장한다.

> 스프링은 자바를 기반으로 한 기술이고 자바는 객체지향 프로그래밍을 가장 중요한 가치로 두고 있다. 스프링의 핵심철학은 객체지향 프로그래밍이 제공하는 폭넓은 혜택을 누릴 수 있도록 기본으로 돌아가는 것이다.

처음에는 '@EnableWebMvc을 선언하면 내부적으로 어떤일이 일어날까?' 라는 호기심을 갖고 접근했지만, 내부적으로 Composite 패턴을 통해 개방 폐쇄 원칙을 지키며 유연하게 설계되어 있는 스프링 내부를 발견하여 매우 흥미로웠다.

토비님이 첫장에 위 멘트를 괜히 넣은게 아니구나 라는 생각이 들었고, 내가 모르는 다른 내부는 어떻게 구성되어있을지 더욱 궁금해졌다.

## 출처

- [https://www.logicbig.com/tutorials/spring-framework/spring-web-mvc/spring-enablewebmvc-annotation.html](https://www.logicbig.com/tutorials/spring-framework/spring-web-mvc/spring-enablewebmvc-annotation.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/DelegatingWebMvcConfiguration.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/DelegatingWebMvcConfiguration.html)

- [https://refactoring.guru/design-patterns/composite](https://refactoring.guru/design-patterns/composite)