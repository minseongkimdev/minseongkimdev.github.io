---
title: "스프링이 사랑한 템플릿 콜백 패턴"
layout: post
category: Spring
---


## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [템플릿 콜백 패턴](#템플릿-콜백-패턴)
- [2. 템플릿 콜백 패턴이 적용된 곳](#2-템플릿-콜백-패턴이-적용된-곳)
	- [RestTemplate](#resttemplate)
	- [RetryTemplate](#retrytemplate)
- [4 템플릿 콜백 패턴에 대한 팁](#4-템플릿-콜백-패턴에-대한-팁)
- [5. 글을 마치며](#5-글을-마치며)
- [참고 서적](#참고-서적)
- [출처](#출처)
- [각주](#각주)

## 1. 들어가면서

회사에 급한 배포가 있어 한 주 동안 블로깅을 못했는데 다시 본격적으로 시작해보려고 한다.

프로젝트를 진행하면서 Hikari Pool을 적용하였는데 JdbcTemplate관련 의존성 문제가 있었다. 이번 기회에 개발하면서 자주 접할 수 있는 JdbcTemplate에 대해 조사하였고, 그 과정에서 새롭게 배운점을 이 글을 통해 공유해보고자 한다.

## 템플릿 콜백 패턴

템플릿 콜백 패턴에서 '템플릿' 이라는 단어의 뜻부터 살펴보자.

> 어떤 특정한 모양을 만들기 위해 만들어진 틀을 말함.

이 사전적 정의만으로는 템플릿을 이해하기 다소 추상적이어서 예를 들어서 설명 해보겠다.

사실 우리 실행활에서 템플릿을 자주 볼 수 있다.

대학 조별과제 발표를 할 때, 인터넷에 떠돌아 다니는 PPT 무료 템플릿을 다운받아 내용만 수정해서 발표자료로 사용해본 경험이 다들 한번쯤은 있을것이다.

우리는 PPT 템플릿을 통해, **미리 디자인된 양식을 재활용** 하면서 **바뀌어야 할 부분**인 내용만 바꿔가면서 쉽고 빠르게 세련된 PPT 자료를 만들 수 있다.

![](https://i.pinimg.com/564x/7a/cf/9c/7acf9cbaed0ea6c6b4f5c9aab6603362.jpg)
- 출처 : Pinterest


여기서 집중해야할 것은 바뀌지 않는 부분은 템플릿의 양식은 재활용 하면서, **바뀌는 부분만 우리 입맛에 맞게 변경할 수 있다는 것이다.**

이제 템플릿 콜백 패턴에 대입해서 생각해보자.

프로그래밍에서는 고정된 틀 안에 바꿀 수 있는 부분을 넣어서 사용하는 경우에 고정된 부분을 템플릿이라고 부른다.

그리고 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트인 콜백이 있다.

파라미터로 전달되지만, 값을 참조하기 위한 것이 아니라 특정 로직을 담은 메소드를 실행시키기 위해 사용한다.

그리고 템플릿 콜백 패턴의 작업 흐름은 아래와 같다.

![](https://user-images.githubusercontent.com/10750614/55551325-7fb4dd00-5715-11e9-96a7-23b8a0a8b103.png)
- 출처 : 토비의 스프링


1. 클라이언트에서 템플릿 안에서 실행될 로직을 담은 콜백 오브젝트를 만들고 콜백이 참조할 정보를 제공한다. (만들어진 콜백은 템플릿 메소드를 호출할 때 파라미터로 전달됨)


2. 템플릿은 정해진 작업 흐름대로 작업을 진행하다가 내부에서 생성한 참조 정보를 가지고 콜백 오브젝트의 메소드를 호출한다.

3. 콜백은 클라이언트 메소드에 있는 정보와 템플릿이 제공한 참조정보를 이용해서 작업을 수행하고 그 결과를 다시 템플릿에 돌려준다.

4. 템플릿은 콜백이 돌려준 정보를 사용해서 작업을 마저 수행한다. 경우에 따라 최종결과를 클라이언트에 다시 돌려주기도 한다.

탬플릿 메소드 패턴에 이쯤 알아보도록 하고 이를 활용한 스프링의 *Template 클래스에는 어떤 것들이 있고, 어떻게 활용하고 있을지 알아보자.

## 2. 템플릿 콜백 패턴이 적용된 곳



### RestTemplate

[스프링 공식문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)에서는 아래와 같이 기술하고 있다.

> Synchronous client to perform HTTP requests, exposing a simple, template method API over underlying HTTP client libraries

RestTemplate는 HTTP 클라이언트 라이브러리르 통해 템플릿 메서드 API를 제공한다.

다음은 RestTemplate을 통해 간단한 GET 요청을 하는 예시이다.

~~~java
RestTemplate restTemplate = new RestTemplate();

String fooResourceUrl
  = "https://half.kim";

ResponseEntity<String> response
  = restTemplate.getForEntity(fooResourceUrl + "2021-08-08-spring-template-callback-pattern", String.class);
~~~

간단하게 URL 문자열과 응답 타입만 파라미터로 넘기면 GET요청을 할 수 있다.


~~~java
@Override
public <T> ResponseEntity<T> getForEntity(URI url, Class<T> responseType) throws RestClientException {

	RequestCallback requestCallback = acceptHeaderRequestCallback(responseType);

	ResponseExtractor<ResponseEntity<T>> responseExtractor = responseEntityExtractor(responseType);

	return nonNull(execute(url, HttpMethod.GET, requestCallback, responseExtractor));

}
~~~


아래와 같이 RestTemplate 내부에서 RequestCallback 인터페이스 구현을 통해 전략을 생성하고 있다.

~~~java
@Override
public void doWithRequest(ClientHttpRequest request) throws IOException {

	if (this.responseType != null) {

		List<MediaType> allSupportedMediaTypes = getMessageConverters().stream()
			.filter(converter -> canReadResponse(this.responseType, converter))
			.flatMap((HttpMessageConverter<?> converter) -> getSupportedMediaTypes(this.responseType, converter))
			.distinct()
			.sorted(MediaType.SPECIFICITY_COMPARATOR)
			.collect(Collectors.toList());

		if (logger.isDebugEnabled()) {

			logger.debug("Accept=" + allSupportedMediaTypes);

		}
			
		request.getHeaders().setAccept(allSupportedMediaTypes);

	}
}
~~~

getForEntity 메서드에서 호출하는 execute 메서드 안에서 doExecute를 호출하고 있다.

doExecute 메서드는 템플릿 콜백 패턴에서 탬플릿에 해당한다.

HTTP 요청에 공통적으로 필요한 작업을 템플릿 메서드로 분리한 것을 확인할 수 있다.

~~~java
@Nullable
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
	@Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

	Assert.notNull(url, "URI is required");
	Assert.notNull(method, "HttpMethod is required");
	ClientHttpResponse response = null;

	try {
		ClientHttpRequest request = createRequest(url, method);
		if (requestCallback != null) {
			requestCallback.doWithRequest(request);
		}

		response = request.execute();
		handleResponse(url, method, response);
		return (responseExtractor != null ? responseExtractor.extractData(response) : null);
	}

	catch (IOException ex) {

		String resource = url.toString();
		String query = url.getRawQuery();
    resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
		throw new ResourceAccessException("I/O error on " + method.name() +
		" request for \"" + resource + "\": " + ex.getMessage(), ex);

	}
	finally {

	if (response != null) {
		response.close();
	}
	}
}
~~~

RestTemplate의 플로우를 도식화 해보면 다음과 같다.

![](https://user-images.githubusercontent.com/44136364/128606580-889c4ec3-82b3-422c-8338-4f0a22c3af95.png)

템플릿 콜백 패턴이 적용된 또다른 클래스인 RetryTemplate도 살펴보도록 하자.

(RestTemplate은 간단하고 직관적이라 템플릿 콜백 패턴의 이해를 돕기 위해 소개한 것이고, Spring Framework 5부터 WebFlux와 함께 Spring은 RestTemplate를 대체하기 위해 WebClient이 도입됐으니 참고하길 바란다.)

### RetryTemplate


개발을 하다보면 네트워크로 인한 문제는 재시도를 하면 해결되는 경우가 있다.

하지만 이러한 재시도 작업에 대한 로직을 일일이 구현하려면 상당히 번거로운 일이다.

그래서 스프링에서 제공하는 RetryTemplate을 통해 손쉽게 재시도 할 수 있는 로직을 구현할 수 있다.

위에서 설명한 RestTemplate의 템플릿을 보고 '에이~ 굳이 저 정도 로직을 템플릿으로 분리해내야 되나?'라고 생각할까봐 이 클래스를 예로 넣어봤다.

템플릿만 해도 약 130 라인이고, 재시도가 필요한 모든 곳에 이 코드를 복붙한다고 상상해보면 템플릿 콜팩 패턴의 필요성을 확실하게 이해할 수 있을것이다.


아래와 같이 클라이언트 쪽에서 콜백을 생성해서 템플릿 쪽에 넘겨줄 수 있다.

~~~java
retryTemplate.execute(new RetryCallback<Integer, RuntimeException>()  {

  @Override
  public Integer doWithRetry(RetryContext context) {

    return myService.retryLogic();

  }  
});
~~~

다음은 템플릿이다. 메소드가 너무 길어 핵심적인 부분만 가져와보았다.
 
핵심은 매개변수로 콜백을 넘겨받아 워크플로우 내에서 콜백을 실행하는 부분이다.

전체 코드가 궁금하다면 [공식 레포](https://github.com/spring-projects/spring-retry/blob/ebc720045ffb130d558ef9b151d5cec07bdcf81d/src/main/java/org/springframework/retry/support/RetryTemplate.java#L271)**를** 확인해보길 바란다.
~~~java

protected <T, E extends Throwable> T doExecute(RetryCallback<T, E> retryCallback,
	RecoveryCallback<T> recoveryCallback, RetryState state) throws E, ExhaustedRetryException {

	while (canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {

  ...

	try {

		... 

		return retryCallback.doWithRetry(context);

		}
    catch (Throwable e) {
      ...
    }
  }
  ...
}
~~~

마찬가지로 RetryTemplate의 플로우를 도식화를 해보면 아래와 같다.

![](https://user-images.githubusercontent.com/44136364/128606548-79f43fae-3271-4c20-83ba-8b5e03120ff8.png)

사실 아래의 흐름을 그릴 때 위의 RestTemplate에서 빨간색에 해당하는 글씨만 바꾸었지만 RetryTemplate과 꼭 들어맞는다.

이 사실이 시사하는 바는 스프링에서는 템플릿 콜백 패턴이라는 일관된 방법으로 내부 기능을 구현하고 있다는 점이다.

(이 외에도 RedisTemplate도 있어 직접 분석해보길 바란다.)

## 4 템플릿 콜백 패턴에 대한 팁

스프링에서 클래스 이름이 ~Template으로 끝나는 클래스를 발견한다면, 템플릿 콜백 패턴이 적용되어 있을 확률이 매우 높다.

이때 위와 같은 방식으로 어떤 부분이 템플릿이고, 어떤 부분이 콜백인지 집중해서 해당 클래스의 구조를 파악하면 해당 클래스의 동작원리를 쉽게 파악할 수 있을 것이다.

이와 같이 스프링은 일관적인 작명 규칙을 통해 개발자들에게 해당 클래스의 역할과 구조에 대한 힌트를 제공한다.


## 5. 글을 마치며

학창시절 수학을 좋아했었다면 프랙탈[^1]에 대해 한번쯤 들어보았을 것이다.

아래는 피타고라스 나무라고 불리는 프랙탈 도형이다.
처음에 봤을 땐 규칙도 보이지 않고 복잡하게만 보인다.
![](https://user-images.githubusercontent.com/44136364/128604076-1077d9dd-7cb5-44c0-afb6-a94b1775f54a.png)


하지만 아래처럼 프랙탈 도형이 **어떤 작은 도형들로 구성되어 있고** **어떤 규칙을 통해 확장되고 있고 구성되어 있는지만 파악**하면 쉽게 전체 구조를 쉽게 파악할 수 있다.

![](https://user-images.githubusercontent.com/44136364/128604119-fdda0cce-6d5b-4f13-a138-628937d461ed.png)

왜 템플릿 콜백 패턴을 설명하다가 갑자기 어려운 수학 얘기를 하는지 의아해 할 수 있겠지만 눈치가 빠른 사람은 무슨 애기를 하고싶은지 알 것이다.

스프링에 도입해서 생각해보면 템플릿 콜백 패턴을 포함한 스프링에서 사용된 디자인 패턴 (어댑터, 프록시, 데코레이터, 싱글턴, 팩터리 메서드, 전략 패턴 등) 들을 활용하여 아주 일관적으로 내부 구조를 확장하고 있다.

또한 위의 팁에서도 설명했듯이 스프링에서는 아주 친절하게도 스프링에서 클래스나 인터페이스의 목적이 무엇이고 어떤 구조로 이뤄졌는지 일관된 작명법을 통해 개발자에게 힌트를 준다. (*Template -> 템플릿 콜백 패턴 적용됨)

그래서 스프링의 일관성 덕분에 자주 사용되는 디자인 패턴과 그 적용 사례를 습득하면, 새로운 부분도 쉽게 파악할 수 있는 것이다.


## 참고 서적

- 토비의 스프링 3.1 1권 3장 템플릿
- 토비의 스프링 3.1 1권 5장 서비스 추상화
- 스프링 입문을 위한 자바 객체지향의 원리와 이해 6장


## 출처

- [The Guide to RestTemplate](https://www.baeldung.com/rest-template)

- [How to use Redis-Template in Java Spring Boot?](https://medium.com/@hulunhao/how-to-use-redis-template-in-java-spring-boot-647a7eb8f8cc)

- [RestTemplate (정의, 특징, URLConnection, HttpClient, 동작원리, 사용법, connection pool 적용](https://sjh836.tistory.com/141)

- [Spring Retry Github Repository](https://github.com/spring-projects/spring-retry) 


## 각주

[^1]: 프랙탈 : 일부 작은 조각이 전체와 비슷한 기하학적 형태를 말한다. 이런 특징을 자기 유사성이라고 하며, 다시 말해 자기 유사성을 갖는 기하학적 구조를 프랙탈 구조이다.
