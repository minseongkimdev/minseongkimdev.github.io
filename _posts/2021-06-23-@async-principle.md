---
title: "@Async의 동작원리로 알아보는 스프링 AOP"
layout: post
category: Spring
---

## 0. 글의 순서
- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. @EnableAsync](#2-enableasync)
- [3. AsyncAnnotationBeanPostProcessor](#3-asyncannotationbeanpostprocessor)
- [4. AsyncAnnotationAdvisor](#4-asyncannotationadvisor)
    - [buildAdvice()](#buildadvice)
    - [buildPointcut()](#buildpointcut)
- [5. AnnotationAsyncExecutionInterceptor](#5-annotationasyncexecutioninterceptor)
- [더 알아보기 - 필요한 스레드풀을 찾는 과정](#더-알아보기---필요한-스레드풀을-찾는-과정)
- [글을 마치며](#글을-마치며)
- [출처](#출처)


## 1. 들어가면서

스프링 프로젝트 중 주문이 들어왔을 때 FCM을 통해 클라이언트 푸쉬 알림을 보내줘야 하는 기능을 @Async를 통해 구현할 예정이다.

그래서 스프링 프레임워크 내부적으로 어떻게 어노테이션을 선언하는것만로 비동기적으로 동작하는 코드를 작성할 수 있는지 호기심이 생겼다.

이를 분석하는 과정에서 결국 **AOP를 통해 이것이 가능함을 알게 되었고** 해당 내용을 이 글을 통해 공유하고자 한다.
(엄밀히 말하면 @Async를 선언하는 것만으로는 동작하지 않고 @EnableAsync 등의 추가적인 어노테이션이 필요하긴 하다.)

## 2. @EnableAsync

스프링 부트에서 여러 @Enable* 이 있다. 이는 특정 기능을 명시적으로 활성화 하기 위한 어노테이션이다.

@EnableAsync는 @Async 어노테이션을 통해 비동기 프로그래밍이 가능하게 해준다.

~~~java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AsyncConfigurationSelector.class})
public @interface EnableAsync {...}
~~~

위는 @EnableAsync의 메타 어노테이션이고, @Import 메타 어노테이션을 통해 AsyncConfigurationSelector 클래스를 사용함을 알 수 있다.

아래는 AsyncConfigurationSelector의 selectImports()이다.
adviceMode에 따라 다른 설정을 불러오는 걸 알 수 있다.
(본 글은 PROXY 모드를 기준으로 설명한다.)

~~~java
// AsyncConfigurationSelector class

@Nullable
public String[] selectImports(AdviceMode adviceMode) {
    switch(adviceMode) {
        case PROXY:
            return new String[]{ProxyAsyncConfiguration.class.getName()};
        case ASPECTJ:
            return new String[]{"org.springframework.scheduling.aspectj.AspectJAsyncConfiguration"};
        default:
            return null;
    }
}
~~~

그리고 아래와 같이 ProxyAsycConfiguration 클래스의 asyncAdvisor()에서 AsyncAnnotationBeanPostProcessor 인스턴스를 생성한 후 **프록시가 적용될 타겟 클래스를 지정**하는 등의 설정을 한 후 해당 인스턴스를 리턴한다.

~~~java
// ProxyAsyncConfiguration class

public AsyncAnnotationBeanPostProcessor asyncAdvisor() {

    Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
    AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
    bpp.configure(this.executor, this.exceptionHandler);
    Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");

    if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {

        bpp.setAsyncAnnotationType(customAsyncAnnotation);

    }

    bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
    bpp.setOrder((Integer)this.enableAsync.getNumber("order"));

    return bpp;
}
~~~

위 코드에서 등장한 AsyncAnnotationBeanPostProcessor이 내부적으로 어떤 역할을 하는지 알아보자.


## 3. AsyncAnnotationBeanPostProcessor

스프링 공식문서에서는 다음과 같이 정의하고 있다.

> Bean post-processor that automatically applies asynchronous invocation behavior to any bean that carries the Async annotation at class or method-level by adding a corresponding **AsyncAnnotationAdvisor** to the exposed proxy (either an existing AOP proxy or a newly generated proxy that implements all of the target's interfaces).

> 프록시에 @Async 어노테이션이 달린 클래스나 메서드에 그에 상응하는 **AsyncAnnotationAdvisor**를 자동으로 적용해주는 BeanPostProcesssor이다.


위를 통해 AsyncAnnotationBeanPostProcessor는 AsyncAnnotationAdvisor를 타겟 클래스 또는 메서드에 위빙 해줌을 알 수 있다.


AsyncAnnotationBeanPostProcessor의 클래스 구조를 살펴보면 아래와 같다. 

![](https://programmersought.com/images/548/cedfdfe6cb9aa17efee5acd2a02914ec.png)


클래스 구조에서 알 수 있듯이 AsyncAnnotationBeanPostProcessor는 **BeanPostProcessor를 구현**하고 있다.

한가지 팁을 주자면, 어떤 클래스의 구조만 보아도 어떤 역할을 수행할 수 있는지 추측할 수 있다.
(다음 부분을 읽기 전에 한번 추측해보길 바란다.)

다 생각해보았다고 가정하고 설명을 이어나가겠다. 

BeanPostProcessor의 여러 구현체중 AbstractAdvisingBeanPostProcessor라는 클래스가 존재한다.

[스프링 공식문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/framework/AbstractAdvisingBeanPostProcessor.html)에서 이 클래스를 심플하게 정의하고 있다.

> Base class for BeanPostProcessor implementations that apply a Spring AOP Advisor to specific beans.

> Spring AOP Advisor를 특저 빈에 적용하는 BeanPostProcessor 구현체의 기본 클래스


즉, 이 구현체는 **Advisor(AsyncAnnotationAdvisor)가 빈에 적용(위빙)될 수 있게 하는 역할**을 한다.

아래는 빈이 초기화된 후에 실행되는 postProcessAfterInitialization의 일부이다.

아래에서 확인할 수 있듯이 Advisor를 빈에 적용하고, 프록시 패턴이 적용된 빈을 리턴하여 위빙이 이뤄진다.

~~~java
if (bean instanceof Advised) {

  Advised advised = (Advised) bean;

  if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {

	// Add our local Advisor to the existing proxy's Advisor chain...
	  if (this.beforeExistingAdvisors) {

		  advised.addAdvisor(0, this.advisor);

	  }

	  else {

        advised.addAdvisor(this.advisor);

	  }

	  return bean;

  }
}

if (isEligible(bean, beanName)) {
	ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);

	if (!proxyFactory.isProxyTargetClass()) {

		evaluateProxyInterfaces(bean.getClass(), proxyFactory);

	}

	proxyFactory.addAdvisor(this.advisor);
	customizeProxyFactory(proxyFactory);

	// Use original ClassLoader if bean class not locally loaded in overriding class loader
	ClassLoader classLoader = getProxyClassLoader();
	if (classLoader instanceof SmartClassLoader && classLoader != bean.getClass().getClassLoader()) {

		classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();

	}

	return proxyFactory.getProxy(classLoader);
}
~~~

마지막으로 위 코드 상에서 등장한 Advisor의 서브타입인 AsyncAnnotationAdvisor에 대해 구체적으로 알아보자.

## 4. AsyncAnnotationAdvisor

드디어 @Async 어노테이션의 Advice와 Pointcut를 담고 있는 AsyncAnnotationAdvisor를 설명할 차례이다.

Spring 공식문서에서 AsyncAnnotationAdvisor를 아래와 같이 정의하고 있다.

> Advisor that activates asynchronous method execution through the Async annotation.

> @Async 어노테이션을 통해 비동기 메서드 실행을 활성화하는 Advisor이다.

Advisor은 Advice와 Pointcut을 속성으로 가지고 있는데, 이는 Advisor의 생성자 안에서 초기화된다.


다음은 AsyncAnnotationAdvisor의 생성자이다.

~~~java
public AsyncAnnotationAdvisor(@Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {
  Set<Class<? extends Annotation>> asyncAnnotationTypes = new LinkedHashSet(2);

  asyncAnnotationTypes.add(Async.class);

  try {

    asyncAnnotationTypes.add(ClassUtils.forName("javax.ejb.Asynchronous", AsyncAnnotationAdvisor.class.getClassLoader()));

  } catch (ClassNotFoundException var5) {}

  this.advice = this.buildAdvice(executor, exceptionHandler);
  this.pointcut = this.buildPointcut(asyncAnnotationTypes);

}
~~~

buildAdvice(), buildPointcut()을 통해 초기화하고 있다.
차례대로 알아보자.


#### buildAdvice()

이 메서드에서 AnnotationAsyncExecutrionInterceptor가 Advice의 역할을 함을 알 수 있다.

~~~java
    protected Advice buildAdvice(@Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {
        AnnotationAsyncExecutionInterceptor interceptor = new AnnotationAsyncExecutionInterceptor((Executor)null);
        interceptor.configure(executor, exceptionHandler);
        return interceptor;
    }
~~~

여기서 Interceptor는 스프링 MVC의 DispatcherServlet이 컨트롤러를 호출하기 전에 호출되는 Interceptor가 아니니 헷갈리지 말길 바란다.

[스프링 공식문서](https://docs.spring.io/spring-framework/docs/1.2.x/reference/aop.html)에 Advice를 아래와 같이 기술하고 있다.
> Advice: Many AOP frameworks, including Spring, model an advice as an interceptor, maintaining a chain of interceptors "around" the joinpoint.

> 스프링을 포함한 많은 AOP 프레임워크에서, Advice를 Interceptor로 모델링 하여, Joinpoint 주변에 Interceptor들을 체이닝한다.

Interceptor는 Joinpoint 주위에 체이닝 되어 Advice로서 역할을 수행한다.

#### buildPointcut()

아래와 같이 @Async 어노테이션이 담긴 Set를 매개변수로 받아, 해당 @Async의 Pointcut을 찾아 리턴해준다.

~~~java
protected Pointcut buildPointcut(Set<Class<? extends Annotation>> asyncAnnotationTypes) {
    ComposablePointcut result = null;

    AnnotationMatchingPointcut mpc;

    for(Iterator var3 = asyncAnnotationTypes.iterator(); var3.hasNext(); result = result.union(mpc)) {

        Class<? extends Annotation> asyncAnnotationType = (Class)var3.next();
        Pointcut cpc = new AnnotationMatchingPointcut(asyncAnnotationType, true);
        mpc = new AnnotationMatchingPointcut((Class)null, asyncAnnotationType, true);

        if (result == null) {

            result = new ComposablePointcut(cpc);

        } else {

            result.union(cpc);

        }
    }

    return (Pointcut)(result != null ? result : Pointcut.TRUE);
}
~~~

이로써 AsyncAnnotationAdvisor은 자신의 Advice와 Pointcut 속성을 초기화 하는 과정에 대한 설명이 끝났다.


## 5. AnnotationAsyncExecutionInterceptor

실질적으로 개발자가 작성한 코드가 비동기적으로 실행되는 로직이 이 Interceptor에 담겨있다.

하지만 생각보다 비동기적으로 실행되는 로직은 단순하다 

~~~java
@Nullable
public Object invoke(MethodInvocation invocation) throws Throwable {
    Class<?> targetClass = invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null;
    Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
    Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
    AsyncTaskExecutor executor = this.determineAsyncExecutor(userDeclaredMethod);

    if (executor == null) {

        throw new IllegalStateException("No executor specified and no default executor set on AsyncExecutionInterceptor either");

    } else {

        Callable<Object> task = () -> {
            
            try {

                Object result = invocation.proceed();
                if (result instanceof Future) {
                    return ((Future)result).get();
                }

            } catch (ExecutionException var4) {

                this.handleError(var4.getCause(), userDeclaredMethod, invocation.getArguments());
                
            } catch (Throwable var5) {

                this.handleError(var5, userDeclaredMethod, invocation.getArguments());

            }

            return null;
        };
        
        return this.doSubmit(task, executor, invocation.getMethod().getReturnType());
    }
}
~~~

위 메서드에서 아래와 같은 과정을 거친다.

1. determineAsyncExecutor() 메서드를 통해 스레드 풀을 찾는다.
2. Callable에 개발자가 작성한 메서드를 포함시킨다
3. doSubmit() 메서드를 통해 Callable task를 실행시킨다.

이로써 @Async 어노테이션이 달린 메서드가 스레드풀에 있는 스레드를 통해 비동기적으로 실행될 수 있는 것이다.

## 더 알아보기 - 필요한 스레드풀을 찾는 과정

지금까지 알아본 과정은 아래와 같다.
1. @EnableAsync에서 AsyncConfigurationSelector에서 AdviceMode를 선택한다.(아래부터 PROXY모드 기준)
2. ProxyAsyncConfiguration의 asyncAdvisor()에서 AsyncAnnotationBeanPostProcessor 인스턴스를 생성한 후 프록시가 적용될 타겟 클래스를 지정한다.
3. AsyncAnnotationBeanPostProcessor는 AsyncAnnotationAdvisor를 타겟 클래스 또는 메서드에 위빙한다.
4. AsyncAnnotationAdvisor의 buildAdvice()에서 AnnotationAsyncExecutrionInterceptor를 Advice로써 사용한다.
5. AnnotationAsyncExecutrionInterceptor의 determineAsyncExecutor() 메서드를 통해 스레드 풀을 찾고 개발자가 작성한 메서드가 포함된 Callable를 doSubmit()를 통해 비동기적으로 실행한다.

그럼 5번 과정에서 스프링은 비동기 작업 수행에 필요한 스레드 풀을 어떻게 찾을까?

~~~java
// AsyncExecutionAspectSupport class

@Nullable
protected AsyncTaskExecutor determineAsyncExecutor(Method method) {
    AsyncTaskExecutor executor = (AsyncTaskExecutor)this.executors.get(method);

    if (executor == null) {

        String qualifier = this.getExecutorQualifier(method);
        Executor targetExecutor;

        if (StringUtils.hasLength(qualifier)) {

            targetExecutor = this.findQualifiedExecutor(this.beanFactory, qualifier);

        } else {

            targetExecutor = (Executor)this.defaultExecutor.get();

        }

        if (targetExecutor == null) {
            return null;
        }

        executor = targetExecutor instanceof AsyncListenableTaskExecutor ? (AsyncListenableTaskExecutor)targetExecutor : new TaskExecutorAdapter(targetExecutor);

        this.executors.put(method, executor);
    }

    return (AsyncTaskExecutor)executor;
}

~~~

위의 getExecutorQualifier()에서 @Async 어노테이션의 value(스레드 풀의 이름)를 얻어온다.

스레드 풀의 이름을 얻어오지 못하면(개발자가 따로 스레드풀을 명시적으로 지정하지 않으면), defaultExecutor를 사용함을 확인할 수 있는데, 그럼 언제 defaultExecutor가 생성될까?

위에서 언급했듯이 configure() 메서드는 AnnotationAsyncExecutionInterceptor가 생성될때 호출된다.

configure() 메서드에서 getDefaultExecutor()를 호출하고

~~~java
public void configure(@Nullable Supplier<Executor> defaultExecutor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {
    this.defaultExecutor = new SingletonSupplier(defaultExecutor, () -> {
        return this.getDefaultExecutor(this.beanFactory);
    });
    this.exceptionHandler = new SingletonSupplier(exceptionHandler, SimpleAsyncUncaughtExceptionHandler::new);
}
~~~

getDefaultExecutor()에서 SimpleAsyncTaskExecutor()를 리턴해준다.

~~~java
@Nullable
protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
    Executor defaultExecutor = super.getDefaultExecutor(beanFactory);
    return (Executor)(defaultExecutor != null ? defaultExecutor : new SimpleAsyncTaskExecutor());
}
~~~

이로써 사용할 스레드풀을 명시적으로 지정하지 않으면 SimpleAsyncTaskExecutor를 기본적으로 사용함을 알 수 있다.

(스레드풀을 지정하는 법은 따로 설명하지 않는다.)


## 글을 마치며

스프링 AOP를 통해 개발자 입장에서 스레드풀을 지정해주고 비동기로 실핼할 비지니스 로직에만 신경 쓸 수 있다. 이 뿐만 아니라 많은 다른 기능들도 일관적으로 AOP를 활용하여 구현되어 있다. 

이 글을 통해 AOP가 어떻게 이뤄지는지 이해했길 바라고, 다른 기능들도 한번 직접 분석해보길 바란다.

@Asnyc와 AOP를 분석하면서 스프링에 대해 더 깊게 이해할 수 있었고, 왜 AOP가 스프링의 3대 핵심요소 중 하나라고 불리는지 수긍할 수 있게 되었다.

## 출처

- [Spring asynchronous implementation principle and actual combat sharing](https://programmersought.com/article/51026239096)
  
- [Programmer Sought - @Async principle](https://www.programmersought.com/article/5564453186/)

- [Spring asynchronous processing @Async use and principle, source code analysis (@EnableAsync)](https://daydaynews.cc/en/technology/242778.html)

- [Java Thread Pool – ThreadPoolExecutor Example](https://howtodoinjava.com/java/multi-threading/java-thread-pool-executor-example/)

- [Spring @Async rest controller example – Spring @EnableAsync](https://howtodoinjava.com/spring-boot2/rest/enableasync-async-controller/)

- [Task Execution and Scheduling](https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/html/scheduling.html)

- [Aspect Oriented Programming with Spring - Proxying mechanisms](https://docs.spring.io/spring-framework/docs/3.0.0.M3/reference/html/ch08s06.html)

- [Spring Boot @Async 어떻게 동작하는가?](https://brunch.co.kr/@springboot/401)
