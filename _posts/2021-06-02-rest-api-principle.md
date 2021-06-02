---
title: "[작성중]REST API 제대로 이해하기"
layout: post
category: Web
---



## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가기 전에](#1-들어가기-전에)
- [2. Represenation이란 어떤 의미일까?](#2-represenation이란-어떤-의미일까)
- [3.](#3)
- [4. 글을 마치며](#4-글을-마치며)
- [출처](#출처)
    - [공식문서](#공식문서)
    - [블로그](#블로그)
- [주석](#주석)


## 1. 들어가기 전에

REST API에 대해서 제대로 이해하려면, REST(REepresentation State Transfer)에 대해 명확하게 이해해야 한다.

## 2. Represenation이란 어떤 의미일까?

Representation의 사전적 정의는 다음과 같다.

> (특정한 방식으로의) 묘사[표현]; (어떤 것을) 나타낸[묘사한] 것

그러면 어떤것을 나타내고 있는걸까?

GET은 아래와 같이 정의되어 있다.

>The GET method requests transfer of a current selected representation
   for the target resource.
> GET 메서드는 현재 선택된 **target 리소스의 representation**을 전송해달라는 요청이다.
  

HTTP/1.1 스펙을 명시하고 있는 RFC[^1]7231에서 [Representation을 아래와 같이 명시하고 있다.](https://datatracker.ietf.org/doc/html/rfc7231#section-3)

> Considering that a resource could be anything, and that the uniform
>    interface provided by HTTP is similar to a window through which one
>    can observe and act upon such a thing only through the communication
>    of messages to some independent actor on the other side, an
>    abstraction is needed to represent ("take the place of") the current
>    or desired state of that thing in our communications.  That
>    abstraction is called a representation [REST].
> 
>    For the purposes of HTTP, a "representation" is information that is
>    intended to reflect a past, current, or desired state of a given
>    resource, in a format that can be readily communicated via the
>    protocol, and that consists of a set of representation metadata and a
>    potentially unbounded stream of representation data.
> 
>    An origin server might be provided with, or be capable of generating,
>    multiple representations that are each intended to reflect the
>    current state of a target resource.  In such cases, some algorithm is
>    used by the origin server to select one of those representations as
>    most applicable to a given request, usually based on content
>    negotiation.  This "selected representation" is used to provide the
>    data and metadata for evaluating conditional requests [RFC7232] and
>    constructing the payload for 200 (OK) and 304 (Not Modified)
>    responses to GET (Section 4.3.1).


커뮤니케이션에서 해당 사물의 현재 또는 원하는 상태를 표현 ( "대신")하려면 추상화가 필요합니다.
이 추상화를 Representaion이라고 한다.

![](https://lostechies.com/content/jimmybogard/uploads/2016/05/image_thumb.png)

## 3. REST의 6가지 조건


## 4. 글을 마치며

(HTTP은 하나의 규약이고, 이 규약은 **RFC 스펙**에 명시되어 있다.
(*_HTTP은  RFC 스펙에 명시되어 있는대로 구현되어 있다._*)

공식문서 검토를 통해 추상적으로 이해하고 있었던 REST API의 개념이 명확해졌다.

또한 **REST**라는 용어의 의미를 정확하게 이해하니, REST API를 한층 더 뚜렷하게 이해할 수 있었다.

기술적인 용어에 대한 명확한 이해가 정말 중요함을 다시금 느낀다.

## 출처

#### 공식문서

- [Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content](https://datatracker.ietf.org/doc/html/rfc7231#section-3)

#### 블로그
- [REST의 representation이란 무엇인가](https://blog.npcode.com/2017/04/03/rest%ec%9d%98-representation%ec%9d%b4%eb%9e%80-%eb%ac%b4%ec%97%87%ec%9d%b8%ea%b0%80/)
- [바쁜 개발자들을 위한 REST 논문 요약
](https://blog.npcode.com/2017/03/02/%eb%b0%94%ec%81%9c-%ea%b0%9c%eb%b0%9c%ec%9e%90%eb%93%a4%ec%9d%84-%ec%9c%84%ed%95%9c-rest-%eb%85%bc%eb%ac%b8-%ec%9a%94%ec%95%bd/)

#### StackOverFlow

- [What is “representation”, “state” and “transfer” in Representational State Transfer (REST)?](https://stackoverflow.com/questions/48116321/what-is-representation-state-and-transfer-in-representational-state-trans)
- [What does Representational State mean in REST?](https://stackoverflow.com/questions/10418105/what-does-representational-state-mean-in-rest)
- [What does “state transfer” in Representational State Transfer (REST) refer to?](https://stackoverflow.com/questions/4603653/what-does-state-transfer-in-representational-state-transfer-rest-refer-to)
- [Representational state transfer (REST) and Simple Object Access Protocol (SOAP)](https://stackoverflow.com/questions/209905/representational-state-transfer-rest-and-simple-object-access-protocol-soap)


## 주석

[^1]: RFC 문서 : Request For Comments의 약자.  비평을 기다리는 문서라는 의미로, 컴퓨터 네트워크 공학 등에서 인터넷 기술에 적용 가능한 새로운 연구, 혁신, 기법 등을 아우르는 아티클을 나타낸다. 인터넷국제표준화기구 IETF는 일부 RFC를 인터넷 표준으로 받아들이기도 한다.