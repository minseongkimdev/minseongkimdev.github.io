---
title: "@Autowired의 동작원리 분석 (feat. BeanPostProcessor)"
layout: post
category: Spring
---

## 0. 글의 순서



## 1. 들어가면서


@Autowired 어노테이션만으로 정말 쉽게 스프링 빈 객체를 주입받을 수 있다.

이 간결함 속에 내부적으로 어떤 일이 일어나는지 분석해본 내용을 이 글을 통해 공유해보고자 한다.


## 2. @Autowired

[spring.io docs에서 @Autowired에 대해 아래와 같이 기술하고 있다.](https://docs.spring.io/spring-framework/docs/3.0.x/javadoc-api/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.html)

> Note that actual injection is performed through a **BeanPostProcessor**

> 실제로 injection은 **BeanPostProcessor**을 통해 이뤄진다는 사실을 참고하십시오.

> Please consult the javadoc for the AutowiredAnnotationBeanPostProcessor class (which, by default, checks for the presence of this annotation).

> 기본적으로 @Autowired 어노테이션의 존재를 체크하는 **AutowiredAnnotationBeanPostProcessor의** javadoc을 참고하십시오.

문서에 나와있는 대로, @Autowired는 BeanPostProcessor와 관련이 있고, 특히 AutowiredAnnotationBeanPostProcessor와 연관이 있음을 알 수 있다.

## 3. AutowiredAnnotationBeanPostProcessor

![](https://www.fatalerrors.org/images/blog/50c7ef3ab9a0d3f342a87a9b1805230c.jpg)
- 출처 : https://www.fatalerrors.org/a/0tlx0zg.html

AutowiredAnnotationBeanPostProcessor의 상속 구조는 위와 같다.

최상위 클래스가 BeanPostProcessor임을 알 수 있다.

그리고 그 사이에 InstantiationAwareBeanPostProcessor를 상속받는 점도 확인할 수 있다.


### InstantiationAwareBeanPostProcessor

클래스의 이름에서 알 수 있다 싶이

Instantitaion: 초기화를

Aware: 알아차리는

BeanPostProcessor 이다.

InstantiationAwareBeanPostProcessor의 postProcessBeforeInstantiation 메서드는 빈이 초기화 될때, 빈 객체를 리턴받을 수 있다.
(default 메서드로 기본적으로 null를 리턴하도록 구현되어 있다.)
~~~java
@Nullable
default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }
~~~

postProcessAfterInstantiation메서드는 빈 객체가 생성된 후에 호출된다.

~~~java
default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        return true;
    }
~~~

### MergedBeanDefinitionPostProcessor

런타임에 merged빈의 정의를 위한 PostProcessor 콜백 인터페이스이다.

(이 인터페이스 또한 BeanPostProcessor를 상속받고 있다.)

오직 아래의 두 개의 메서드만 정의되어 있다.

- postProcessMergedBeanDefinition
- resetBeanDefinition





1. 

AutowiredAnnotationBeanPostProcessor는 BeanPostProcessor 인터페이스를 구현한 클래스이다.

 

결론부터 말하자면 @Autowired 애노테이션은 BeanPostProcessor라는 라이프 사이클 인터페이스의 구현체인 AutowiredAnnotationBeanPostProcessor에 의해 의존성 주입이 이루어진다.

 

BeanPostProcessor는 빈의 initializing(초기화) 라이프 사이클 이전, 이후에 필요한 부가 작업을 할 수 있는 라이프 사이클 콜백이다.

그리고 BeanPostProcessor의 구현체인 AutowiredAnnotationBeanPostProcessor가 빈의 초기화 라이프 사이클 이전, 즉 빈이 생성되기 전에 @Autowired가 붙어있으면 해당하는 빈을 찾아서 주입해주는 작업을 하는 것이다.


BeanFactory와 BeanPostProcessor와의 관계? 



## 출처

#### 공식문서
[@Autowired-docs.spring.io ](https://docs.spring.io/spring-framework/docs/3.0.x/javadoc-api/org/springframework/beans/factory/annotation/Autowired.html)

#### 블로그

[How about the automatic injection process of Spring @Autowired annotation?](https://www.fatalerrors.org/a/0tlx0zg.html)

[@Autowired 동작 원리 - BeanPostProcessor](https://atoz-develop.tistory.com/entry/Spring-Autowired-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC-BeanPostProcessor)

