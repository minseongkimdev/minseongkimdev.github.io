---
title: "@EnableWebMvc는 어떤 전략을 사용할까?"
layout: post
category: Spring
---


## 0. 글의 순서

## 1. 들어가면서

[지난 글](https://www.minseong.kim/dispatcherservlet-default-strategy.html) 알아봤던 기본전략에 이어 @EnableWebMvc를 활성화 하면 어떤 전략을 사용하는지에 대해 알아볼 것이다.

그럼 @EnableWebMvc란 무엇일까?


## 2. @EnableWebMvc

@EnableWebMvc어노테이션은 Bean 자동으로 설정들을 해준다. 또한 필요로 하는 Bean들을 손쉽게 등록하여 커스텀할 수 있다.

이 어노테이션은 아래와 같이 정의되어 있는데, 핵심은 DelegatingWebMvcConfiguration 클래스를 Import 하고 있다는 것이다.

~~~java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({DelegatingWebMvcConfiguration.class})
public @interface EnableWebMvc {
}
~~~

스프링 공식문서에 DelegatingWebMvcConfiguration에 대해 다음과 같이 설명하고 있다.

> A subclass of WebMvcConfigurationSupport that detects and delegates to all beans of type WebMvcConfigurer allowing them to customize the configuration provided by WebMvcConfigurationSupport. This is the class actually imported by @EnableWebMvc.


> DelegatingWebMvcConfiguration은 
WebMvcConfigurationSupport가 제공하는 Configuration을 커스텀 할 수 있도록 WebMvcConfigurer 타입의 모든 빈들을 감지하고 위임하는 WebMvcConfigurer의 서브클래스이다.


이미 DelegatingWebMvcConfiguration은 Configuration을 커스텀 하는 용으로 잘 알고 있을 것이라 생각한다.

하지만 이 글의 논지에 맞게 DelegatingWebMvcConfiguration의 상위 클래스인 WebMvcConfigurationSupport에 초점을 맞출 예정이다.



## WebMvcConfigurationSupport

이 글의 핵심 클래스인 WebMvcConfigurationSupport은 @EnableWebMvc 활성화 했을 때 별도의 커스텀 하지 않았을 때 기본 전략들을 가지고 있는 클래스이다.

개발자는 역시 코드로 이야기 해야한다. 코드를 살펴보자.



## Configuration 커스텀하기

@EnableWebMvc에서 제공하는 Configuration을 커스텀하려면 스프링 5.0 이전에는 WebMvcConfigurerAdapter를 확장하여 커스텀 하고자 하는 메서드(기능)을 오버라이딩 했었다.





## 출처

- [https://www.logicbig.com/tutorials/spring-framework/spring-web-mvc/spring-enablewebmvc-annotation.html](https://www.logicbig.com/tutorials/spring-framework/spring-web-mvc/spring-enablewebmvc-annotation.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/DelegatingWebMvcConfiguration.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/DelegatingWebMvcConfiguration.html)