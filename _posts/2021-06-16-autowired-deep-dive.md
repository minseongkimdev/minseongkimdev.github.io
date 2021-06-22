---
title: "@Autowired Deep Dive"
layout: post
category: Spring
---

## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. 빈의 생명주기](#2-빈의-생명주기)
- [3. @Autowired](#3-autowired)
- [4. BeanPostProcessor](#4-beanpostprocessor)
- [5. AutowiredAnnotationBeanPostProcessor](#5-autowiredannotationbeanpostprocessor)
  - [Reflection](#reflection)
- [글을 마치며](#글을-마치며)
- [출처](#출처)
    - [공식문서](#공식문서)
    - [블로그](#블로그)
- [각주](#각주)

## 1. 들어가면서


`@Autowired` 어노테이션만으로 정말 쉽게 스프링 빈 객체를 주입받을 수 있다.

이 간결함 속에 내부적으로 어떤 일이 일어나는지 분석해본 내용을 이 글을 통해 공유해보고자 한다.


## 2. 빈의 생명주기

우선 Autowiring이 이뤄지는 과정을 알기 위해서는 빈의 생명주기에 대해 이해해야 한다.

특히 Autowiring이 이뤄지는 빈의 생성 과정에 초점을 두어 설명하자면

대략적인 스프링 빈의 라이프 사이클은 아래와 같다.

![](https://reflectoring.io/assets/img/posts/the-lifecycle-of-spring-beans/spring-bean-lifecycle.png)

위 과정중에서 빈이 사용될 수 있는 상태가 되기 전까지 실행되는 메서드의 흐름은 아래와 같다.

![](https://springframework.guru/wp-content/uploads/2019/05/aware_interfaces_callbacks_in_bean_lifecycle.png)


위 처럼 스프링 빈의 생명주기는 인스턴스화부터 소멸까지 여러 단계로 이루어져 있다.

빈의 생성단계를 5단계로 나눠보면 아래와 같다.

- Instantiation : 
스프링 프레임워크가 빈 객체를 인스턴스화 한다. 
(빈의 Constructor가 실행됨.)

- Populating Properties : 빈을 인스턴스화 한 후 Spring은 `Aware`[^0] 인터페이스를 구현하는 빈을 스캔하고 관련 속성을 설정한다.

- **Pre-Initialization : `BeanPostProcessor`의 `postProcessBeforeInitialization()`메소드가 트리거 되고, 그 직후에 `@PostConstruct`[^1] 어노테이션이 붙은 메서드가 실행된다.**

- AfterPropertiesSet :
InitializingBean을 구현한 빈의 afterPropertiesSet()이 실행된다.

- Custom-Initialization : initMethod은 우리가 @Bean 속성에 정의한 초기화 메서드를 트리거 한다.


- Post-Initializaton : BeanPostProcessor의 postProcessAfterInitialization()를 트리거 한다.


위의 과정중에 Autowiring은 Pre-Initialization 단계의 BeanPostProcessor에 의해 이뤄진다.
## 3. @Autowired


위 단계에서 Autowiring과 관련 있는 부분은 Pre-Initialization의 `BeanPostProcessor` 이다.

[spring.io docs에서 @Autowired에 대해 아래와 같이 기술하고 있다.](https://docs.spring.io/spring-framework/docs/3.0.x/javadoc-api/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.html)

> Note that actual injection is performed through a **BeanPostProcessor**

> 실제로 injection은 **BeanPostProcessor**을 통해 이뤄진다는 사실을 참고하십시오.

> Please consult the javadoc for the AutowiredAnnotationBeanPostProcessor class (which, by default, checks for the presence of this annotation).

> 기본적으로 @Autowired 어노테이션의 존재를 체크하는 **AutowiredAnnotationBeanPostProcessor의** javadoc을 참고하십시오.

문서에 나와있는 대로, @Autowired는 `BeanPostProcessor`와 이 인터페이스의 구현체인 `AutowiredAnnotationBeanPostProcessor`와 연관이 있음을 알 수 있다.

이제 `BeanPostProcessor`와 `AutowiredAnnotationBeanPostProcessor`에 대해 본격적으로 알아보자.

## 4. BeanPostProcessor

스프링의 `BeanPostProcessor`는 **빈의 초기화 라이프 사이클 이전, 이후에 필요한 부가 작업을 할 수 있는 라이프사이클 콜백**이다.

그리고 뒤에서 설명하겠지만, `BeanPostProcessor`의 구현체인 `AutowiredAnnotationBeanPostProcessor`가 빈의 초기화 라이프 사이클 이전, 즉 빈이 생성되기 전에 @Autowired가 달려있으면, 해당 빈을 찾아서 주입하는 작업을 한다.

다시 돌아와서 `BeanPostProcessor`에 대해 부연설명을 하지면 오직 두개의 메서드로만 구성되어 있다.

- `postProcessBeforeInstantiation` : 빈의 초기화할 때 호출되는 `init()` 메서드가 실행되기 전에 호출된다.
- `postProcessAfterInstantiation` : `init()` 메서드가 호출된 후에 호출된다.

~~~java
@Nullable
default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }
~~~

`postProcessAfterInstantiation()` 메서드는 빈 객체가 생성된 후에 호출된다.

~~~java
default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        return true;
    }
~~~

아래와 같이 `BeanPostProcessor` 를 구현하여 얼마든지 Custom `BeanPostProcessor` 을 구현할 수 있다.

~~~java
public class MinseongBeanProcessor implements BeanPostProcessor{
	
		public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("Hello,");
		return bean;
	}
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("World");
		return bean;
	}
}
~~~

그리고 beans 파일에 어래와 같이 Custom `BeanPostProcessor` 클래스 위치를 지정해줘야 한다.
~~~java
<bean class='kim.minseong.MinseongBeanProcess'>
~~~

BeanPostProcessor에 대해 조금 더 첨언하자면 빈을 프록시로 래핑하는데에 사용된다. 즉 Proxy Wrapping 로직을 제공하기 위해 AbstractAdvisingBeanPostProcessor을 활용하기도 한다.

## 5. AutowiredAnnotationBeanPostProcessor

긴 설명 끝에 드디어 실질적으로 Autowiring를 수행하는 `AutowiredAnnotationBeanPostProcessor` 를 설명할 차례이다.

위에서 잠깐 설명했듯이 `@Autowired` 애노테이션은 `BeanPostProcessor` 라는 라이프 사이클 인터페이스의 구현체인 `AutowiredAnnotationBeanPostProcessor` 에 의해 의존성 주입이 이루어진다.

즉 빈이 생성되기 전에 `@Autowired` 가 붙어있으면 해당하는 빈을 찾아서 주입해주는 작업을 하는 것이다.


다음은 이 클래스의 핵심이라고 할 수 있는 `@Autowired` 로 어노테이션된 필드나 메서드에 대해서 객체를 주입해주는 역할을 수행하는 메서드이다.

~~~java
public void processInjection(Object bean) throws BeanCreationException {
  Class<?> clazz = bean.getClass();
  InjectionMetadata metadata = this.findAutowiringMetadata(clazz.getName(), clazz, (PropertyValues)null);

  try {
    metadata.inject(bean, (String)null, (PropertyValues)null);
  } catch (BeanCreationException var5) {
    throw var5;
  } catch (Throwable var6) {
    throw new BeanCreationException("Injection of autowired dependencies failed for class [" + clazz + "]", var6);
  }
}
~~~

`InjectMetadata` 클래스의 `inject()` 메서드를 호출하여 객체를 주입한다.

~~~java
protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs) throws Throwable {
  if (this.isField) {
  Field field = (Field)this.member;
  ReflectionUtils.makeAccessible(field);
  field.set(target, this.getResourceToInject(target, requestingBeanName));
  } else {
    if (this.checkPropertySkipping(pvs)) {
      return;
    }

    try {
      Method method = (Method)this.member;
      ReflectionUtils.makeAccessible(method);
      method.invoke(target, this.getResourceToInject(target, requestingBeanName));
    } catch (InvocationTargetException var5) {
      throw var5.getTargetException();
    }
  }
}
~~~

위 코드에서 마법같은 일이 일어나는데 차례대로 설명하면 `Field`인지 `Method` 인지 if문으로 분기가 된다.

Reflection에 의해 `Field` 와 `Method` 전부 **타입에 상관없이** Injection이 가능하다.

또한 `ReflectionUtils` 의 `makeAccessible()`를 통해 접근제어자가 `private` 임에도 불구하고, Autowiring 되는 객체에 접근이 가능하도록 한다.

 이것은 개발자들이 `private` 필드도 주입받을 수 있게 되어, 캡슐화를 지키면서 Autowiring를 활용할 수 있다.

`ReflectionUtils` 의 `makeAccessible()` 메서드를 계속해서 타고 들어가다보면 아래의 `AccessibleObject` 클래스의 `setAccessible0()` 메서드를 호출하는 것을 확인할 수 있다.

~~~java
private static void setAccessible0(AccessibleObject obj, boolean flag)
        throws SecurityException{
  if (obj instanceof Constructor && flag == true) {
    Constructor<?> c = (Constructor<?>)obj;
    if (c.getDeclaringClass() == Class.class) {
      throw new SecurityException("Cannot make a java.lang.Class" + " constructor accessible");
    }
  }
  obj.override = flag;
}
~~~

AccessibleObject 클래스는 [java docs](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/AccessibleObject.html)에 아래와 같이 기술되어 있다.

> It provides the ability to flag a reflected object as **suppressing default Java language access control checks when it is used.**
> 
> 이 클래스는 리플렉션된 객체에 대해 해당 객체가 사용될 때 **자바의 접근 제어자를 무시할 수 있는 능력을 제공**한다.


위에서 확인했듯이 AutowiredAnnotationBeanPostProcessor 내부에서 꽤나 복잡한 과정을 거친다.

요약하자면 아래와 같다.

1. AutowiredAnnotationBeanPostProcessor의 processInjection() 이 호출된다.


2. InjectMetadata 클래스의 inject()이 호출되는데, 여기서 ReflectionUtils의 makeAccessible()를 통해 private한 필드에도 Autowiring이 가능하도록 해준다.


3. InjectMetadata 클래스의 inject()에서 필드의 경우 Field.set()을 통해, 메서드의 경우 Method.invoke()를 통해  Injection이 이뤄진다.

### Reflection

Relection이란 자바에서 구체적인 클래스 타입을 알지 못해도, 그 클래스의 메소드와 타입, 변수들에 접근할 수 있는 API 이다.

스프링의 @Autowired 같은 경우에 조건을 충족하는 빈들에게 다른 빈을 주입해줘야 한다. 하지만 프레임워크 단에서는 개발자가 어떤 클래스를 만들지 모른다.

그래서 스프링에서는 Reflection을 통해 개발자가 작성한 임의의 클래스에 접근한다.

@Autowried 외에 스프링의 많은 부분에서 Reflection이 활용된다. 모두 설명하기엔 이 글의 범위를 벗어나므로 다른 레퍼런스를 참고하길 바란다.


## 글을 마치며

@Autowired 만으로 정말 쉽게 DI가 가능했다. 하지만, 이것은 스프링 내부적으로 꽤 복잡한 과정을 거쳐 가능한 것이었다.

마치 호수위의 백조가 발을 분주하게 움직여 우아하게 호수위를 떠다닐 수 있었던 것처럼, 스프링은 개발자들을 위해 정말 분주히 많은 과정을 거쳐 개발자에게 심플하고 편리한 추상화를 제공한다.

**스프링(봄)** 이라는 이름이 전통적인 J2EE가 지배적이던 시절을 "겨울" 이라고 보고, 자바에 "봄"이 도래하게 할 것이라는 당찬 포부와 함께 지어진 것이라고 한다.

스프링을 공부하면 할수록 꽤나 적절한 이름이라는 생각이 든다.

## 출처

#### 공식문서

[@Autowired - docs.spring.io](https://docs.spring.io/spring-framework/docs/3.0.x/javadoc-api/org/springframework/beans/factory/annotation/Autowired.html)

[Spring Framework: The Origins of a Project and a Name](https://spring.io/blog/2006/11/09/spring-framework-the-origins-of-a-project-and-a-name#:~:text=Fortunately%20Yann%20stepped%20up%20with,%E2%80%9Cwinter%E2%80%9D%20of%20traditional%20J2EE.)
#### 블로그


[Hooking Into the Spring Bean Lifecycle](https://reflectoring.io/spring-bean-lifecycle/)


[How about the automatic injection process of Spring @Autowired annotation?](https://www.fatalerrors.org/a/0tlx0zg.html)


[자바 리플렉션이란?](https://dublin-java.tistory.com/53)
## 각주


[^0]: 스프링의 Aware 인터페이스를 통해 스프링 프레임워크 내부 작업에 연결할 수 있다.

[^1]: 스프링은 [JSR-250](https://jcp.org/en/jsr/detail?id=250) 스펙에 따라 @PostConstruct을 지원한다.

[^2]: BeanDefinition은 속성값, 생성자 파라미터 값 및 구체적인 구현에서 제공되는 메타데이터를 통해 빈 인스턴스를 설명한다.
주된 목적은 BeanFactoryPostProcessor 속성값과 기타 빈 메타 데이터를 검사하고 수정할 수 있도록 하는 것이다.

