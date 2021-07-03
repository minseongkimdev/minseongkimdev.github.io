---
title: "DispatcherServlet은 어떤 기본전략을 사용할까?"
layout: post
category: Spring
---

## 0. 글의 순서


## 1. 들어가면서

스프링 MVC에서 DispatcherServlet는 properties의 정보를 토대로 본인의 기본 전략을 선택한다.

그럼 우리는 왜 DispatcherServlet가 사용하는 기본전략에 대해 알아야할까?
(어떤 기술에 대해 공부를 할때 '왜'라는 질문은 상당히 중요하다.)

DispatcherServlet의 기본전략을 분석한다는 것은 **스프링 MVC의 핵심인 DispatcherServlet의 기본적인 동작원리**를 이해하는 것이라 상당한 의의를 가지기 때문에 이를 파악하고 있는것은 매우 중요하다.

우선 DispatcherServlet의 구성요소부터 짚고 넘어가보자.

## 2. DispatcherServlet의 구성요소

DispatcherServlet은 onRefresh() 메서드를 통해 initStragies()가 실행된다.

~~~java
protected void onRefresh(ApplicationContext context) {
  this.initStrategies(context);
}
~~~

initStragies()에는 총 9개의 인터페이스 초기화 메서드가 존재한다.
**아래의 9가지가 DispatcherServlet을 구성하는 핵심 인터페이스이다.**
~~~java
protected void initStrategies(ApplicationContext context) {
  this.initMultipartResolver(context);
  this.initLocaleResolver(context);
  this.initThemeResolver(context);
  this.initHandlerMappings(context);
  this.initHandlerAdapters(context);
  this.initHandlerExceptionResolvers(context);
  this.initRequestToViewNameTranslator(context);
  this.initViewResolvers(context);
  this.initFlashMapManager(context);
}
~~~


대표로 initHandlerAdapters()를 확인해보자.

~~~java

private void initHandlerAdapters(ApplicationContext context) {

...
  if (this.handlerAdapters == null) {
    this.handlerAdapters = this.getDefaultStrategies(context, HandlerAdapter.class);

    if (this.logger.isTraceEnabled()) {
      this.logger.trace("No HandlerAdapters declared for servlet '" + this.getServletName() + "': using default strategies from DispatcherServlet.properties");
    }
  }
}
~~~

이와 같이 따로 지정한 HandlerAdapter가 없으면, getDefaultStrategies를 통해 기본 전략을 사용한다.

이제 getDefaultStrategies에서 어떤 기본전략들을 사용하는지 알아보자.

(각 인터페이스의 기능에 대해 구체적으로 설명하지 않는다.)

## 3. DispatcherServlet의 기본 전략들

다음은 DispatcherServlet 클래스의 getDefaultStrategies() 메서드의 일부이다.
Properties 타입의 정적 필드인 defaultStrategies를 초기화해준다.

~~~java
protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
  if (defaultStrategies == null) {
    try {
      ClassPathResource resource = new ClassPathResource("DispatcherServlet.properties", DispatcherServlet.class);
      defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    } catch (IOException var15) {
      throw new IllegalStateException("Could not load 'DispatcherServlet.properties': " + var15.getMessage());
    }
  }
...
~~~

아래의 코드에서 `DispatcherServlet.properties`에서 기본전략들을 로드해옴을 알 수 있다. (클래스 로드 타임에 이 파일을 로드한다.)
~~~java
ClassPathResource resource = new ClassPathResource("DispatcherServlet.properties", DispatcherServlet.class);
~~~


`DispatcherServlet.properties` 파일은 아래와 같다.
([스프링 공식 레포지토리에서 확인할 수 있다.](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/resources/org/springframework/web/servlet/DispatcherServlet.properties))



~~~java
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter


org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager

~~~

또한 주석을 통해 다음과 설명하고 있다. 

> **Default implementation classes for DispatcherServlet's strategy interfaces.**
> Used as fallback when no matching beans are found in the DispatcherServlet context.
> Not meant to be customized by application developers.

> DispatcherServlet의 **전략 인터페이스의 디폴트 구현 클래스**이다.
> DispatcherServlet context에서 적절한 빈을 찾지 못했을 대 사용된다.
> 개발자가 이 파일을 수정하는 것을 의도하지 않았다.

자 이제, 이 디폴트 전략들이 어떻게 동작하는지 알아보자.

## 4. 여러개의 기본전략을 가지는 인터페이스

위에서 눈치 챘을 수도 있지만, 특이하게 아래의 세개의 인터페이스들은 각자의 구현체를 리스트로 관리된다.

~~~java
@Nullable
private List<HandlerMapping> handlerMappings;
@Nullable
private List<HandlerAdapter> handlerAdapters;
@Nullable
private List<HandlerExceptionResolver> handlerExceptionResolvers;
~~~

즉, 각 필드는 여러 기본전략 구현체를 가지고 있다. 그럼 여러 구현체중 어떤 구현체를 사용할까?

그리고 어떤 구현체를 사용할지에 대한 정보를 어디서 찾을 수 있을까?

HandlerMapping, HandlerAdapter, HandlerExceptionResolver가 하는 역할을 다시 한번 생각해보자.

해당 요청을 처리할 수 있는 Handler를 찾는 역할, Handler를 실행하는 역할, Handler 매핑과 실행시에 예외가 발생하면 적절한 예외를 정해주는 역할이다.

이러한 역할들은 DispatcherServlet이 **Handler를 Dispatch하는 과정**에서 필요한 것들이다.

따라서 어렵지 않게 DispatcherServlet의 doDispatch()에 비밀이 숨겨져 있음을 유추할 수 있다.


아래는 doDispatch() 메서드이고, 핵심적인 코드만 간추려보았다.

~~~java
// DispatcherServlet class

protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  ...

  mappedHandler = this.getHandler(processedRequest);

  ...

  HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());

  ...
  
  this.processDispatchResult(processedRequest, response, mappedHandler, mv, (Exception)dispatchException);

  ...
} 
~~~

getHandler() 메서드에서 handlerMappings를 iterator를 통해 **차례대로 순회하며** 핸들러를 찾고있다.
만약에 핸들러를 찾으면 해당 핸들러를 리턴한다.

DispatcherServlet.properties에 선언된 순서가 핸들러를 찾는데 사용되는 우선순위임을 알 수 있다.
~~~java
    @Nullable
    protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        if (this.handlerMappings != null) {
            Iterator var2 = this.handlerMappings.iterator();

            while(var2.hasNext()) {
                HandlerMapping mapping = (HandlerMapping)var2.next();
                HandlerExecutionChain handler = mapping.getHandler(request);
                if (handler != null) {
                    return handler;
                }
            }
        }

        return null;
    }

~~~

getHandlerAdapter()도 살펴보자.

~~~java
    protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
        if (this.handlerAdapters != null) {
            Iterator var2 = this.handlerAdapters.iterator();

            while(var2.hasNext()) {
                HandlerAdapter adapter = (HandlerAdapter)var2.next();
                if (adapter.supports(handler)) {
                    return adapter;
                }
            }
        }

        throw new ServletException("No adapter for handler [" + handler + "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
    }
~~~

iterator를 통해 List를 **차례대로 순회하며** 핸들러를 찾고있다.
핸들러를 실행할 수 있는 어댑터를 찾으면 해당 어댑터를 리턴한다.
즉, 위와 마찬가지로 DispatcherServlet.properties에 선언된 순서가 사용되는 우선순위임을 확인할 수 있다.


마지막으로 processDispatchResult()의 내부에서 processHandlerException를 호출하고 있다.

~~~java

    protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) throws Exception {
        request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
        ModelAndView exMv = null;
        if (this.handlerExceptionResolvers != null) {
            Iterator var6 = this.handlerExceptionResolvers.iterator();

            while(var6.hasNext()) {
                HandlerExceptionResolver resolver = (HandlerExceptionResolver)var6.next();
                exMv = resolver.resolveException(request, response, handler, ex);
                if (exMv != null) {
                    break;
                }
            }
        }

        ...

~~~

마찬가지로 iterator를 통해 List를 **차례대로 순회하며** Exception을 결정하고 있는 것을 확인할 수 있다.

이로써 세가지 인터페이스가 DispatcherServlet.properties의 여러 구현체중 어떤것을 사용하는가에 대한 해답을 찾았다.

## 5. DispatcherServlet의 기본 전략들이 동작하는 방식

DispatcherServlet의 핵심 인터페이스라고 할 수 있는 HandlerMapping, HandlerAdapter, HandlerExceptionResolve의 구현체들 부터 알아보자.
(모든 구현체들을 자세히 다루지 않는다. 출처의 레퍼런스를 달아놓았으니 참고하길 바란다.)
### 1) HandlerMapping

##### BeanNameUrlHandlerMapping 

이름에서 알 수 있다싶이 빈의 이름을 활용한다.
요청 URI와 동일한 이름을 가진 Controller 빈을 매핑한다.
예를 들어 ~/minseong으로 요청이 들어왔을 때 빈의 이름이 minseong인 빈이 존재하면 매핑이 가능하다.

##### RequestMappingHandlerMapping

마찬가지로 이름에서 알 수 있다싶이 @Controller 클래스의 @RequestMapping이 달린 메서드를 매핑한다.
 
##### RouterFunctionMapping

RouterFunctions을 지원하는 HandlerMapping의 구현체이다.
Spring WebFlux에서 사용되어 구체적인 설명은 생략한다.
### 2) HandlerAdapter

##### HttpRequestHandlerAdapter

HttpRequestHandler 인터페이스를 구현한 클래스를 컨트롤러로 사용할 때 사용되는 어댑터이다.

##### SimpleControllerHandlerAdapter

Controller 인터페이스를 구현하는 구현체를 다루며, Controller 객체에 요청을 전달하는데 사용된다.

당연한 이야기이지만 만약에 웹 애플리케이션에서 Controller만 사용하면 이 구현체를 기본 전략으로 사용하기 때문에 따로 별도의 HandlerAdpater를 구성할 필요가 없다.

##### RequestMappingHandlerAdapter

위에서 설명한 RequestMappingHandlerMapping와 함께 사용된다.
RequestMappingHandlerMapping를 통해 핸들러를 호출한다.

##### HandlerFunctionAdapter

HandlerFunctions를 지원하는 구현체이다.
Spring WebFlux에서 사용되어 구체적인 설명은 생략한다.


### 3) HandlerExceptionResolver

##### ExceptionHandlerExceptionResolver

@Controller또는 @ControllerAdvice에 선언된 @ExceptionHandler 어노테이션을 통해 예외를 처리함.
아래와 같이 @ExceptionHandler 어노테이션이 정의되어 있다.

~~~java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ExceptionHandler {
    Class<? extends Throwable>[] value() default {};
}
~~~

예를 들어 아래와 같이 어노테이션에 배열을 넘겨 특정 Exception들에 대해서만 지정해줄 수 있다.
~~~java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
~~~
##### ResponseStatusExceptionResolver

@ResponseStatus 어노테이션을 통해 HTTP 상태코드에 따라 Exception을 결정헌다.
##### DefaultHandlerExceptionResolver

스프링 MVC의 Exception을 HTTP 상태코드로 매핑해주는 역할을 한다.

예를 들어 MissingPathVariableException이 발생하면 500 코드로 변환해준다.

[스프링 공식문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.html)에 변환 테이블이 명시되어 있으니 참고하길 바란다. 

### 그 외의 구현체들

##### 1) AcceptHeaderLocaleResolver

org.springframework.web.servlet.i18n에 속하는 클래스이다.
속한 패키지만 보더라도 다국어 지원과 관련된 기능을 담당하는 클래스임을 짐작할 수 있다.

HTTP Request 헤더의 accept-language[^1]에 들어있는 Locale 정보를 사용하는 LocaleResolver의 구현체이다.


##### 2) FixedThemeResolver

고정된 테마를 사용하기 위한 ThemeResolver의 구현체이다. `defaultThemeName` 프로퍼티를 통해 지정할 수 있다.


(다크모드의 유행(?)으로 비교적 최근에 지원을 시작한 줄 알았으나, 스프링 초창기부터 제공되었다는 사실이 놀라웠다.)

##### 3) DefaultRequestToViewNameTranslator

RequestToViewNameTranslator로 들어오는 요청의 URI를 
뷰 이름으로 변환해준다.

RequestToViewNameTranslator란 HttpServletRequest에서 뷰 이름이 명시적으로 지정되지 않았을 때 URL를 통해 뷰의 이름을 추론해주는 전략 인터페이스이다.

##### 4) InternalResourceViewResolver

JSP와 같은 InternalResourceView[^2]를 지원하기 위한 UrlBasedViewResolver의 하위 클래스이다.

여기서 핵심은 UrlBasedViewResolver인데,
명시적으로 매핑을 정의하지 않아도 뷰의 이름을 URL로 사용한다.

그리고 prefix와 suffix 속성을 통해 URL를 생성할 수 있다.

이는 아래와 같이 UrlBasedViewResolver의 buildView() 메서드에서 prefix, 뷰이름, suffix를 이어붙이는 과정이 있어 가능하다.

~~~java
// UrlBasedViewResolver class

protected AbstractUrlBasedView buildView(String viewName) throws Exception {
  ...
  view.setUrl(this.getPrefix() + viewName + this.getSuffix());
  ...
}
~~~

##### 5) SessionFlashMapManager

Session에 FlashMap 인스턴스를 저장하거나 불러오기 위한 구현체이다.

비교적 최근(v3.1.1)에 추가된 구현체이다. 

좀 더 부연설명을 하자면 FlashMap은 POST 요청을 받고 나서 클라이언트 측의 새로고침으로 인한 중복 form submit을 방지하기 위해 GET으로 뷰를 보여주는 URL로 리다이렉션을 하는데, 이때 URL 파라미터로 데이터를 따로 지정하지 않기 위해 사용한다.


## 5. 글을 마치며

DispatcherServlet에서 사용하는 기본전략들을 전부 설명하느라 글이 길어진 감이 있다. 하지만 중요하기 때문에 전부 설명했다.

다시 한번 강조하지만 DispatcherServlet의 기본전략을 파악하는 것은 **스프링 MVC의 핵심인 DispatcherServlet의 기본적인 동작원리**를 이해하는 것이므로 상당히 중요하다.

또한 DispatcherServlet의 코드를 살펴보며, 어떤 기본전략들을 사용하는지에 대해 직접 확인해보았다.

이를 설명하는 여러 글들이 있지만, 모든 글이 100% 정확하다고 말하기 힘들다. 간혹 공식문서에도 잘못된 정보가 있기도 하다.

**그래서 우리는 코드를 직접 확인하는 습관을 들여야 한다. 코드는 절대 거짓말을 하지 않기 때문이다.**

다음 글에서는 @EnableWebMvc를 활성화 했을 때, 어떤 전략을 사용하는지에 대해 알아보자.

## 출처


- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/i18n/AcceptHeaderLocaleResolver.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/i18n/AcceptHeaderLocaleResolver.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/theme/FixedThemeResolver.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/theme/FixedThemeResolver.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/view/DefaultRequestToViewNameTranslator.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/view/DefaultRequestToViewNameTranslator.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/RequestToViewNameTranslator.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/RequestToViewNameTranslator.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/view/InternalResourceView.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/view/InternalResourceView.html)

- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language)
-  [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/annotation/ResponseStatusExceptionResolver.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/annotation/ResponseStatusExceptionResolver.html)

- [https://www.baeldung.com/spring-mvc-handler-adapters](https://www.baeldung.com/spring-mvc-handler-adapters)


## 각주

[^1]: Accept-Language : 클라이언트가 이해할 수 있는 언어이다.  Contents Negotiation을 통해 서버는 여러 제안중 하나를 선택하여 헤더의 Content-Language에 정보를 담아 클라이언트에게 보내준다. 자세한 스펙은 [이곳](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language)을 참고하길 바란다.


[^2]: InternalResourceView : 동일한 웹어플리케이션의 JSP나 다른 리소스의 Wrapper이다. 자세한 스펙은 [이곳](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/view/InternalResourceView.html)을 참고하길 바란다.