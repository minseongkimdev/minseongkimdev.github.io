---
title: "[작성중]REST API 제대로 이해하기 2편 - 아마도 그건 REST API가 아닐것이다."
layout: post
category: Web
---



## 0. 글의 순서


## 1. 들어가면서

이 글의 제목을 위와 같이 지은건, 오늘날 **REST API**라고 불리는 아키텍쳐들이 로이필딩이 그의 논문에서 제시한 6가지 제약을 대부분 모두 지키고 있지 않고 있기 때문이다.

엄밀히 따지면 그의 논문에서 정의한 제약을 지키지 않는 아키텍쳐는 REST API라고 부르면 안된다.

하지만 오늘날 관용적으로 일부 조건을 충족하면 REST API라고 불려지고 있다.

**REST API에 제약 조건이 있고 그것들을 지켜야 REST API라고 할 수 있는 사실**을 

아는 것과 모르는 것에는큰 차이가 있으므로 한번 이 조건에 대해 알아보도록 하자.

또한 이 글을 통해 REST API에 대해 잘못 알려진 오해를 풀어보고, REST API에 대해 정확하게 이해해보자.


## 2. REST

REpresentational State Transfer의 약자이다.
정의 그대로 리소스의 Representation들 전송한다는 의미이다.
이 말이 무슨말인지 모르는 독자는[내가 이전에 Represenstaion에 쓴 글](https://minseongkimdev.github.io/what-is-representation.html)을 참고하길 바란다.


## 3. REST API에 대한 오해

많은 개발자들이 REST API를 하나의 표준으로 오해하고 있다.

REST API는 표준이 아니라 **일련의 아키텍쳐의 제약 조건**이다.

로이 필딩이 논문에서, 이러한 제약 조건을 지키는 아키텍쳐를 REST API라고 명명한 것이다.
(널리 쓰이다 보니 표준이라는 오해가 있는것 같다.)

엄밀히 말하자면, 로이필딩이 제한한 조건에 부합하지 않으면 REST API라고 말하기 어렵다.

오늘날 관용적으로 일부 조건을 충족하면 REST API라고 불려지고 있지만, REST API에 조건이 있다는을 아는것과 모르는 것에는 큰차이가 있으므로 한번 이 조건에 대해 알아보도록 하자.


REST는 HTTP로 구현되어야만 한다..?

아니다. REST는 프로토콜에 독립적이다.

즉, HTTP가 널리 사용되는 것일뿐, HTTP로 구현되어야만 REST의 조건을 충족하는것이 아니다.
다른 프로토콜을 통해서도 REST API를 구현할 수 있다.


## 4. REST API의 원칙 (제약조건)

로이필딩은 [본인의 논문](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)에서 REST API의 제약조건을 정의하였다.

핵심적인 내용만 발췌해서 정리해보자.
#### Client-server

> **Separation of concerns** is the principle behind the client-server constraints
> Perhaps most significant to the Web, however, is that the separation allows **the components to evolve independently**, thus supporting the Internet-scale requirement of multiple organizational domains.

> **관심사의 분리**는 이 제약의 원칙이다.
> 아마도 웹에서 가장 중요한 점은 이 분리를 통해 클**라이언트와 서버 애플리케이션은 독립적으로 발전**할 수 있다는 것이다.

 데이터 스토리지 문제에서 사용자 인터페이스 문제를 분리하여 여러 플랫폼에서 사용자 인터페이스의 이식성을 개선하고 서버 구성 요소를 단순화하여 확장 성을 개선합니다.
#### Stateless

> This constraint induces the properties of visibility, reliability, and scalability
> 
> The disadvantage is that it may decrease network performance by increasing the repetitive data (per-interaction overhead) sent in a series of requests, since that data cannot be left on the server in a shared context.

> 이 제약은 visibility, reliablity, scalability를 향상시켜준다.
> visibility는
> reliability는 부분적인 실패에 대한 복구를 쉽게하기 때문에 향상된다.

작업을 위한 상태 정보를 따로 저장하지도 않고 관리하지도 않는다. 세션과 쿠키를 통해 별도로 관리한다. 이 때문에 서비스의 불필요한 정보를 관리하지 않아서 구현이 단순해진다.


#### Cacheable

 – 캐시 제약 조건에 따라 요청에 대한 응답 내의 데이터가 캐시 가능 또는 캐시 불가능으로 암시 적 또는 명시 적으로 레이블이 지정되어야합니다. 응답을 캐시 할 수있는 경우 클라이언트 캐시에 해당 응답 데이터를 나중에 동등한 요청에 재사용 할 수있는 권한이 부여됩니다.
#### Uniform Interface


#### Layered System

#### Code-On-Demand


 – 소프트웨어 엔지니어링 원칙의 일반성을 구성 요소 인터페이스에 적용함으로써 전체 시스템 아키텍처가 단순화되고 상호 작용의 가시성이 향상됩니다. 균일 한 인터페이스를 얻으려면 구성 요소의 동작을 안내하는 여러 아키텍처 제약 조건이 필요합니다. REST는 네 가지 인터페이스 제약으로 정의됩니다. 자원 식별; 표현을 통한 자원 조작; 자기 설명 메시지; 그리고 애플리케이션 상태의 엔진으로서의 하이퍼 미디어.
계층 형 시스템 – 계층 형 시스템 스타일을 사용하면 각 구성 요소가 상호 작용하는 직계 계층 너머를 "볼"수 없도록 구성 요소 동작을 제한하여 아키텍처를 계층 적 계층으로 구성 할 수 있습니다.
요청시 코드 (선택 사항) – REST를 사용하면 애플릿 또는 스크립트 형식으로 코드를 다운로드하고 실행하여 클라이언트 기능을 확장 할 수 있습니다. 이는 사전 구현에 필요한 기능의 수를 줄여 클라이언트를 단순화합니다.




## 출처

- [Roy Fielding. “Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation_2up.pdf)

- [Red Hat topics - What is a REST API?](https://www.redhat.com/en/topics/api/what-is-a-rest-api)

- [What is REST](https://restfulapi.net/)
- [](https://spoqa.github.io/2012/02/27/rest-introduction.html)
- [](https://medium.com/@trevorhreed/you-re-api-isn-t-restful-and-that-s-good-b2662079cf0e)