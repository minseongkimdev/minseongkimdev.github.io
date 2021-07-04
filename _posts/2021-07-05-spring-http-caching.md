---
title: "스프링 HTTP Caching 설정"
layout: post
category: Spring
---


## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. HTTP Caching이란?](#2-http-caching이란)
    - [Cache-Control](#cache-control)
    - [ETag](#etag)
    - [Last-Modified](#last-modified)
- [3. 스프링 HTTP Caching 관련 설정](#3-스프링-http-caching-관련-설정)
    - [CacheControl](#cachecontrol)
      - [WebContentInterceptor](#webcontentinterceptor)
      - [WebContentGenerator](#webcontentgenerator)
    - [Controllers](#controllers)
    - [Static Resources](#static-resources)
    - [ETag Filter](#etag-filter)
      - [ShallowEtagHeaderFilter](#shallowetagheaderfilter)
- [글을 마치며](#글을-마치며)
- [출처](#출처)
- [각주](#각주)


## 1. 들어가면서

HTTP Caching은 불필요한 네트워크 요청을 방지하여 웹 애플리케이션의 성능을 향상시키는데 거의 필수적인 요소중 하나라고 볼 수 있다.

HTTP Caching이 필요한 이유는 아래와 같이 축약할 수 있다.

- 네트워크를 통해 리소스를 가져오는 것 자체가 느리고 비용이 많이 든다.

- 한번의 요청으로 끝나지 않고 클라이언트와 서버간의 여러 왕복이 필요한 응답이 있을 수 있다.

- [렌더링 하는데 반드시 필요한 자원](https://developers.google.com/web/fundamentals/performance/critical-rendering-path)을 가져오기 전까지 웹페이지가 로드되지 않는다.

- 특히 모바일 유저들의 네트워크 사용량이 한정되어 있을 수 있다.

그럼 HTTP Caching에 대해 좀 더 구체적으로 알아보고, 스프링 MVC에서는 어떻게 HTTP Caching과 관련된 설정을 할 수 있을지 확인해보자.

## 2. HTTP Caching이란?

브라우저는 HTTP 요청을 하기 전에 로컬에서 사용할 수 있는 캐시를 먼저 찾고, 만약에 찾으면 캐시를 통해 해결할 수 있는 부분들을 요청과정에서 제거한다.

**그리고 브라우저의 HTTP Caching 동작은 요청헤더와 응답헤더의 조합으로 제어된다.**

HTTP Cahcing을 위해 응답헤더에 설정할 수 있는 세가지 옵션들이 있다.

* Cache-Control
* ETag
* Last-Modified

차례대로 알아보도록 하자.


#### Cache-Control

어떻게 캐싱하고, 얼마나 오랜동안 캐싱할 것인지에 대한 지시자를 담을 수 있는 헤더이다.

이를 통해 클라이언트가 동일한 웹사이트를 다시 방문하면 해당 사용자가 로컬에 존재하는 캐시를 재활용 할지, 다시 서버로 요청할지 등의 행동을 정의할 수 있다.

[RFC 7234](https://tools.ietf.org/html/rfc7234#section-5.2.2)에서 모든 Cache-Control 응답 헤더의 지시자를 명세하고 있으니 궁금하다면 참고하길 바란다. 자주 사용되는 옵션만 살펴보자.

* no-cache : 캐시를 사용하지 않겠다라고 오해하기 쉬우나, 캐시를 쓰기 전에 서버에 정말 사용해도 되는지 한번더 물어보라는 의미이다.
* no-store : 캐시를 사용하지 않겠다는 의미이다.
* must-revalidate : 만료된 캐시만 서버에 확인을 받도록 하는 의미이다.
* public/private  : public은 공유캐시 (중개 서버)에 저장해도 되는지에 대한 여부를 의미하고 private은 브라우저와 같이 특정 사용자 환경에만 저장하라는 의미이다.

참고로 응답 헤더 뿐만아니라, 요청헤더로도 사용할 수 있다. 중개 서버가 존재하는 경우, 중개 서버에 있는 캐시를 가져오지 않도록 하기 위해 Cache-Control를 요청 헤더에 넣어주기도 한다.


캐시를 클라이언트단에 오랫동안 저장하고 있으면 보안상의 문제가 발생할수 있으므로, 서버에서 Cache-Control 헤더를 통해 만료 기간을 설정한다.
#### ETag

쉽게 말하면, 컨텐츠가 바뀌었는지 검사할 수 있는 태그이다.
서버에서 ETag 헤더를 통해 유효성 검사 토큰을 전달하고, 이 토큰이 동일하다면 캐시를 사용하지 않고, 변경되었다면 새 응답을 가져온다.

아래의 그림과 같이 서버로 요청할 때 If-None-Match 헤더에 ETag 토큰을 전송하여, 해당 값이 동일하면 상태코드를 304로 설정하고 캐시를 사용하게 한다.

![](https://web-dev.imgix.net/image/tcFciHGuF3MxnTr1y5ue01OGLBn2/e2bN6glWoVbWIcwUF1uh.png?auto=format&w=948)
- 출처 : https://web.dev/http-cache/

#### Last-Modified

서버가 자원이 가장 마지막에 수정된 시각 정보를 응답 헤더에 넣어주는 정보이다.

~~~java
Last-Modified: <day-name>, <day> <month> <year> <hour>:<minute>:<second> GMT
~~~

HTTP Cahcing에 대해 이쯤 알아보도록 하고 스프링에서는 어떻게 HTTP Caching 관련 응답에 대한 설정을 할 수 있는 방법에 대해 알아보자.


## 3. 스프링 HTTP Caching 관련 설정
#### CacheControl

CacheControl 클래스를 통해 Cache-Control 헤더와 관련된 설정을 할 수 있다.

HTTP Caching과 관련된 디렉티브를 필드로 가지고 있어 해당 디렉티브를 설정할 수 있다.

~~~java
// CacheControl class
    @Nullable
    private Duration maxAge;
    private boolean noCache = false;
    private boolean noStore = false;
    private boolean mustRevalidate = false;
    private boolean noTransform = false;
    private boolean cachePublic = false;
    private boolean cachePrivate = false;
    private boolean proxyRevalidate = false;
    @Nullable
    private Duration staleWhileRevalidate;
    @Nullable
    private Duration staleIfError;
    @Nullable
    private Duration sMaxAge;
~~~

그리고 WebContentInterceptor와 WebContentGenerator 클래스에서 CacheControl 클래스를 사용한다.

##### WebContentInterceptor

Caching에 대한 설정을 응답에 적용하는 핸들러 인터셉터이다.

아래는 addCacheMapping() 메서드이다.

~~~java
// WebContentInterceptor Class

private Map<PathPattern, CacheControl> cacheControlMappings;

public void addCacheMapping(CacheControl cacheControl, String... paths) {
  
  String[] var3 = paths;
  int var4 = paths.length;

  for(int var5 = 0; var5 < var4; ++var5) {
    String path = var3[var5];
    this.cacheControlMappings.put(this.patternParser.parse(path), cacheControl);
  }

}

~~~

위 메서드에서 파라미터로 CacheControl를 전달받아 PathPattern에 CacheControl를 매핑해준다.


##### WebContentGenerator

AbstractController와 WebContentInterceptor과 같이 웹 컨텐츠를 생성해주는 클래스를 위한 상위 클래스이다.
HTTP Cache Control 관련 옵션들을 지원한다.

아래는 Cache Control 관련 설정을 Response에 적용해주는 applyCacheControl() 메서드이다.

~~~java

    protected final void applyCacheControl(HttpServletResponse response, CacheControl cacheControl) {
        String ccValue = cacheControl.getHeaderValue();
        if (ccValue != null) {
            response.setHeader("Cache-Control", ccValue);
            if (response.containsHeader("Pragma")) {
                response.setHeader("Pragma", "");
            }

            if (response.containsHeader("Expires")) {
                response.setHeader("Expires", "");
            }
        }

    }


~~~

#### Controllers

컨트롤러에서 명시적으로 HTTP Caching을 활성화 할 수 있다. 스프링 공식문서에서도 이 방식을 권장한다. 왜냐하면 조건부 요청 헤더[^1]와 비교하기 전에 리소스의 lastModified나 ETag 값을 서버에서 미리 계산해야 하기 때문이다.

아래와 예와 같이 ETag 헤더와 Cache-Control 설정을 빌더패턴을 통해 ResponseEntity에 추가할 수 있다.

~~~java
@GetMapping("/book/{id}")
public ResponseEntity<Book> showBook(@PathVariable Long id) {

    Book book = findBook(id);
    String version = book.getVersion();

    return ResponseEntity
            .ok()
            .cacheControl(CacheControl.maxAge(30, TimeUnit.DAYS))
            .eTag(version) // lastModified is also available
            .body(book);
}
~~~

그리고 다음은 리소스가 변경되지 않았을 때 비어있는 body와 304 NOT_MODIFIED 상태코드를 가진 응답을 보내주는 예제이다.
우리는 조건부 요청 헤더를 컨트롤러 내에서 비교할 수 있다.

~~~java
@RequestMapping
public String myHandleMethod(WebRequest request, Model model) {

    long eTag = ... 

    if (request.checkNotModified(eTag)) {
        return null; 
    }

    model.addAttribute(...); 
    return "myViewName";
}
~~~

eTag를 통해 리소스가 변경 되었는지 검사하고, 변경되지 않았다면 null를 리턴하고 더이상 처리하지 않는다.

(GET, HEAD메서드에서는 304 NOT_MODIFIED를 리턴해줄 수 있고 POST, PUT, DELETE 메서드에서 동시 수정을 막기 위해 412 PRECONDITION_FAILED의 상태코드를 리턴해줄 수 있다.)

#### Static Resources

성능을 위해서 정적 리소스(Javascript, CSS 등)는 Cache-Control를 통해 제공해야한다.

정적 리소스를 Caching 하기 위해서는, 이에 처리하는 리소스 핸들러에 설정을 추가해줘야 한다.
다음에 예는 max-age-31536000 (1년)으로 설정해주는 예이다. 
참고로 이렇게 오랜기간 Caching하는 이유는, 파일이 변경되었을 때만 유저가 캐시를 사용하지 않고 이 자원을 사용하도록 하기 위함이다.
(1년은 [RFC2616](https://www.ietf.org/rfc/rfc2616.txt)에 명시된 최대 기간이다.)
~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
            .addResourceLocations("/public", "classpath:/static/")
            .setCacheControl(CacheControl.maxAge(Duration.ofDays(365)));
    }
}
~~~


#### ETag Filter

ShallowEtagHeaderFilter를 통해 응답의 내용을 기반으로 Shallow ETag 값을 생성한다.
(Shallow ETag는 128 비트 해시값을 생성하는 해시함수인 MD5를 통해 생성된 값이다.)

이와 상반되는 Deep ETag라는 개념도 있지만 더이상 깊이 다루지 않도록 한다.

##### ShallowEtagHeaderFilter

응답을 기반으로 하여 ETag 값을 생성하는 필터이다. 

이 ETag는 요청의 If-None-Match의 값과 비교하고, 만약 이 값이 같으면 응답을 보내지 않고 304 Not Modified 상태코드를 리턴한다.

아래의 generateETagHeaderValue()메서드를 통해 ETag값을 생성하고, updateResponse() 메서드에서 헤더에 생성된 ETag값을 헤더에 세팅해준다.

~~~java
// ShallowEtagHeaderFilter Class
    protected String generateETagHeaderValue(InputStream inputStream, boolean isWeak) throws IOException {
        StringBuilder builder = new StringBuilder(37);
        if (isWeak) {
            builder.append("W/");
        }

        builder.append("\"0");
        DigestUtils.appendMd5DigestAsHex(inputStream, builder);
        builder.append('"');
        return builder.toString();
    }

    private void updateResponse(HttpServletRequest request, HttpServletResponse response) throws IOException {

        ...

        if (this.isEligibleForEtag(request, wrapper, wrapper.getStatus(), wrapper.getContentInputStream())) {
            String eTag = wrapper.getHeader("ETag");
            if (!StringUtils.hasText(eTag)) {
                eTag = this.generateETagHeaderValue(wrapper.getContentInputStream(), this.writeWeakETag);
                rawResponse.setHeader("ETag", eTag);
            }

            if ((new ServletWebRequest(request, rawResponse)).checkNotModified(eTag)) {
                return;
            }
        }

        wrapper.copyBodyToResponse();
    }
~~~

generateETagHeaderValue() 에서 MD5 알고리즘을 통해 ETag를 생성함을 확인할 수 있다.


## 글을 마치며

팀 버너스리가 고안한 HTTP는 맨 처음에 문서만 교환할 수 있는 정말 단순한 형태의 프로토콜이었다.
하지만 웹이 발전함에 따라 HTTP도 같이 발전하며 무수히 많은 기능들이 추가되었다.

 이 글에서 다룬 Caching과 관련되 부분 말고도 무수히 많인 헤더들이 있다. 기회가 된다면 다른 헤더들도 공부해보고, HTTP의 동작원리 등과 같은 더 깊은 공부를 해보고 싶다.

## 출처

- [https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-static-resources](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-static-resources)

- [https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/web/servlet/mvc/WebContentInterceptor.html](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/web/servlet/mvc/WebContentInterceptor.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/support/WebContentGenerator.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/support/WebContentGenerator.html)

- [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/ShallowEtagHeaderFilter.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/ShallowEtagHeaderFilter.html)

- [MDN Web Docs - Last-Modified](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Last-Modified)
 
- [Prevent unnecessary network requests with the HTTP Cache](https://web.dev/http-cache/)


- [Cache Headers in Spring MVC](https://www.baeldung.com/spring-mvc-cache-headers)

- [Cachable Static Assets with Spring MVC](https://www.baeldung.com/cachable-static-assets-with-spring-mvc)

- [What is Cache Control?](https://limelightkr.co.kr/%EC%BA%90%EC%8B%9C-%EC%A0%9C%EC%96%B4%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9E%85%EB%8B%88%EA%B9%8C/#:~:text=%EC%BA%90%EC%8B%9C%20%EC%A0%9C%EC%96%B4%EB%8A%94%20%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80%20%EC%BA%90%EC%8B%B1,%ED%95%98%EB%8A%94%20%EC%A0%80%EC%9E%A5%EC%86%8C%EC%97%90%20%EC%A0%80%EC%9E%A5%ED%95%A9%EB%8B%88%EB%8B%A4.)

- [알아둬야 할 HTTP 쿠키 & 캐시 헤더](https://www.zerocho.com/category/HTTP/post/5b594dd3c06fa2001b89feb9)



## 각주


[^1]: Conditional Request Headers : HTTP는 조건에 따라 다른 응답을 받을 수 있는 기능을 지원한다. HTTP 요청 메세지에서 Validator 헤더를 통해 조건을 걸 수 있고 Validator 헤더에는 Last-Modified와 ETag등이 있다.