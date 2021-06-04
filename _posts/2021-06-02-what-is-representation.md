---
title: "REST API 제대로 이해하기 1편 - Representation이란?"
layout: post
category: Web
---



## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가기 전에](#1-들어가기-전에)
- [2. Represenation이란 어떤 의미일까?](#2-represenation이란-어떤-의미일까)
- [3. 요약](#3-요약)
- [4. 글을 마치며](#4-글을-마치며)
- [출처](#출처)
    - [공식문서](#공식문서)
    - [블로그](#블로그)
- [주석](#주석)


## 1. 들어가기 전에

REST API에 대해서 제대로 이해하려면, REST(Representational State Transfer)에 대해 명확하게 이해해야 한다.

## 2. Represenation이란 어떤 의미일까?

먼저 Representation에 대해 이해하기 전에 Resource가 무엇인지 알아야하고, **Representation과 정확하게 구별하여 이해해야 한다.**

Resource는 URI와 밀첩한 관련이 있다.

[다음은 RFC에 표기된 URI의 정의이다.](https://datatracker.ietf.org/doc/html/rfc3986)

>A Uniform Resource Identifier (URI) is a compact sequence of
   characters that identifies an abstract or physical resource.
   
> (URI는 추상적인 혹은 물리적인 Resouce를 가리키는 구분자이다.)


그럼 이 URI로 다음과 같은 HTTP Request을 보내는 상황을 가정해보자.

> GET /minseong/profile
> 
> Accpet: application:json


위와 같은 요청을 하여 아래와 같은 결과를 얻었다고 하면,


> {
> 
>  "nickname" : "Ras",
> 
>  "job" : "Programmer",
> 
>   "age" : 25,
> 
>   "gender" : "Male",
> 
> }

이 결과는 Resource일까?

**결론부터 얘기하면 아니다.**

Resource가 아니라 **JSON Representation**을 전송받은 것이다.

**즉, URI는 Resource를 가리키는 구분자 이지만, HTTP 통신을 통해 서버로 부터 받은것은 Reource가 아닌 Representation이다.**

그 이유에 대해서 차근차근 알아보자.

GET 메서드의 정의를 살펴보면 RFC 스펙에 아래와 같이 정의되어 있다.

>The GET method requests transfer of a **current selected representation**
   for the target resource.
> GET 메서드는 현재 선택된 Resource의 **현재 선택된 Representation**을 전송해달라는 요청이다.

즉, 위의 예시에서 아래와 같이 요청하면,
> GET /minseong/profile
> 
> Accpet: application:**html**

클라이언트에서 HTML문서 형태의 Representation을 수신받는 것이다.


**서버로부터 받는것이 Resource가 아니고, 현재 Resource의 Representation이라는 사실이 중요하다.**

이 사실에 집중해서 아래의 공식 스펙을 읽어보자.

HTTP/1.1 스펙을 명시하고 있는 RFC[^1]7231에서 [Representation을 아래와 같이 명시하고 있다.](https://datatracker.ietf.org/doc/html/rfc7231#section-3)
핵심적인 내용만 발췌하여 해석해보자.

 
>    **An abstraction is needed to represent ("take the place of") the current
>    or desired state of that thing in our communications.  That
>    abstraction is called a representation [REST]**.
 
HTTP 통신에서 해당 Resource의 **현재 또는 클라이언트가 원하는 상태를 표현 하려면 추상화**가 필요한데,
이 추상화를 Representaion이라고 한다.
 
> that **consists of a set of representation metadata** and a
> potentially unbounded stream of **representation data**.
 
Representation은 **representation metadata**와 **representation data**로 이뤄져 있다.

(위에 예시에서 
representation metadata는 `Accpet: application:json`에 해당하고
representation data는 response에 담긴 json에 해당한다.)

>    An origin server might be provided with, or be capable of generating,
>    **multiple representations that are each intended to reflect the
>    current state of a target resource**.  In such cases, some algorithm is
>    used by the origin server to select one of those representations as
>    m**ost applicable to a given request, usually based on content
>    negotiation**.
> 

서버는 현재 상태를 반영하는 **여러 형태의 Representation을 제공할 수 있다**.
 이러한 경우에, **"Content Negotiation"에 기반**하여, 주어진 요청에 가장 적합한 Representation을 제공하는 알고리즘이 있다.


잠시 Content Negotitaion에 대해 간단히 알아보면, 이름에서 알 수 있다 싶이 가장 적합한 Content에 대해 협상 하는것이다. 그리고 이를 통해 사용자에게 가장 알맞는 컨텐츠를 제공한다. (언어, 이미지 포맷, 인코딩 등)

예를 들면 Content Negotiation을 통해 한국에서 애플 공홈에 접속하면 내용을 한국어로 보여주고, 미국에서 애플 공홈에 접속하면 영어로 보여줄 수 있는 것이다.

누구에게 주도권이 있느냐에 따라 협상 방법은 크게 두가지가 있다.

- 서버 주도 컨텐츠 협상 
- 에이전트 주도 협상

Content Negotiation에 대한 자세한 내용은 이 글의 범위를 벗어나기 때문에 [해당 문서](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation)를 참고하길 바란다.


![](https://mdn.mozillademos.org/files/13789/HTTPNego.png)

- Content Negotiation이 이뤄지는 의 과정 (출처 : MDN)

## 3. 요약

설명이 다소 길어진감이 있지만 핵심은 간단하다.

1. HTTP 통신을 통해 서버로부터 받는것이 Resource가 아니고, **현재 Resource의 Representation**이다.

2. Representation은 해당 Resource의 **현재 상태 또는 클라이언트가 원하는 상태를 표현하는 추상화**이다.

3. Representation은 **Representation Metadata**와 **Representation Data**로 구성되어 있다.

4. Representation이 여러개 일 경우, **Content Negotiation**에 기반하여 선택된다.


## 4. 글을 마치며

REST API에 대해 알아보기 위해 Representaion에 대해 알아보았다.

이 글에서 알수 있다 싶이 GET, URI, HTTP/1.1 을 명세한 RFC문서에 Represntation에 대한 개념이 명확하게 기술되어 있었다.

사실 HTTP는 하나의 프로토콜(약속)이고, HTTP는 RFC 문서에 명시된대로 구현되어 있기 때문에, HTTP와 이에 관련된 내용(REST 등)들은 RFC 문서상에서 검토하여 명확하게 이해하는것이 중요하다.

아무튼, 이 글을 통해 이해한 Representation을 기반으로 다음 글에서 REST API에 대해 알아보도록 하자.

## 출처

#### 공식문서

- [Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content](https://datatracker.ietf.org/doc/html/rfc7231#section-3)
- [MDN Web Docs - Content negotiation](https://developer.mozilla.org/ko/docs/Web/HTTP/Content_negotiation)


#### 블로그
- [REST의 representation이란 무엇인가](https://blog.npcode.com/2017/04/03/rest%ec%9d%98-representation%ec%9d%b4%eb%9e%80-%eb%ac%b4%ec%97%87%ec%9d%b8%ea%b0%80/)

#### StackOverFlow


- [What does Representational State mean in REST?](https://stackoverflow.com/questions/10418105/what-does-representational-state-mean-in-rest)
- [What does “state transfer” in Representational State Transfer (REST) refer to?](https://stackoverflow.com/questions/4603653/what-does-state-transfer-in-representational-state-transfer-rest-refer-to)


## 주석

[^1]: RFC 문서 : Request For Comments의 약자.  비평을 기다리는 문서라는 의미로, 컴퓨터 네트워크 공학 등에서 인터넷 기술에 적용 가능한 새로운 연구, 혁신, 기법 등을 아우르는 아티클을 나타낸다. 인터넷국제표준화기구 IETF는 일부 RFC를 인터넷 표준으로 받아들이기도 한다.