---
title: "About Redis Pipeline"
layout: post
category: Redis
---

## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. TCP의 3-Way-Handshake (feat. RTT)](#2-tcp의-3-way-handshake-feat-rtt)
- [3. RTT만의 문제가 아니다.](#3-rtt만의-문제가-아니다)
- [4. Redis Pipeline with Spring](#4-redis-pipeline-with-spring)
- [5. 출처](#5-출처)


## 1. 들어가면서

Redis는 In-Memory Store로써 기본적으로 빠른 데이터 입출력을 제공한다.

싱글스레드와 이벤트루프 기반의 비동기 방식으로 요청을 처리하기 때문에 고성능이다.

하지만, [Redis 공식문서](https://redis.io/topics/protocol)의 Protocol Spec을 보면 아래와 같이 기술되어 있다.


> A client connects to a Redis server creating a TCP connection to the port 6379.

> While RESP is technically non-TCP specific, in the context of Redis the protocol is only used with TCP connections (or equivalent stream oriented connections like Unix sockets).


위와 같이 TCP 기반의 네트워크 모델을 기반으로 하기 때문에 네트워크 I/O 에서 병목이 생길 수 있는 가능성이 있다.

그 원인을 TCP의 동작원리에서 찾을 수 있고 이는 Redis 파이프라인이 필요한 이유와 연관이 깊다.

## 2. TCP의 3-Way-Handshake (feat. RTT)


TCP 프로토콜은 연결 지향 프로토콜로써(Connection Oriented Protocol)

연결을 확립하기 위해 아래와 같이 서버와 클라이언트 간의 패킷을
연결 확립 요청 (SYN), 연력 확립 응답 + 연결 확립 요청 (SYN-ACK), 연결 확립 응답(ACK)을 주고 받는 3-Hand-Shake 과정을 거치게 된다.

![](https://www.researchgate.net/publication/323353729/figure/fig1/AS:597014421176321@1519350738343/Measuring-round-trip-time-RTT-in-a-three-way-handshake-of-the-Transmission-Control.png)
* TCP의 3-Hand-Shake

여기서 RTT는 클라이언트로부터 SYN을 수신하고 SYN, ACK를 보낸 뒤, ACK를 받게 되는 시간간격이다.

네트워크 환경에 따라 RTT는 길어질 수 있고 한번에 많은 데이터를 전송하면 여러번의 3-Hand-Shake가 발생하게 된다.

이 문제를 해결하기 위해 Redis 파이프라인이 필요하다.

소프트웨어에서의 파이프라인의 기본 의미는 아래와 같다.

> 시스템의 효율을 높이기 위해 명령문을 수행하면서 몇 가지의 특수한 작업들을 병렬 처리하도록 설계된 하드웨어.

좀더 직관적으로 설명하면 클라이언트가 응답을 기다리지 않고 서버에 여러 요청을 보낼 수 있고 한 번에 응답을 받을 수 있다는 의미이다.

![](https://user-images.githubusercontent.com/44136364/136777234-a9dc4c67-ffde-401b-903f-fc46665931ee.png)

이렇게 되면 클라이언트 입장에서는 기다리지 않고 여러 요청을 보내고 서버 입장에서는 한번에 응답을 보낼 수 있다.

이는 [HTTP 1.1](https://developer.mozilla.org/en-US/docs/Web/HTTP/Connection_management_in_HTTP_1.x)에서도 적용되어 있는 기술이니 참고해보길 바란다.

## 3. RTT만의 문제가 아니다.

파이프라이닝은 RTT를 줄여줄 뿐만 아니라 주어진 Redis 서버에서 처리 성능을 향상시켜줄 수 있다.

파이프라이닝을 사용했을 경우와 사용하지 않았을 때 어떤 차이가 있는지 OS의 관점에서 살펴보자.

![](https://miro.medium.com/max/1540/1*J3LbfnG88ysmltH48VhU6w.png)

클라이언트에게 응답을 해줄 때는 OS에서 시스템 콜을 호출해야 한다.

시스템 콜을 처리하는 과정은 다음과 같다.

1. 유저 프로세스가 시스템 콜을 호출하면 사용자 모드에서 커널 모드로 전환 된다.

2. 커널은 내부적으로 각각의 시스템 콜을 구분하기 위해 기능별로 고유 번호를 할당하고 그 번호에 해당하는 제어 루틴을 내부에 정의한다.

3. 커널은 요청받은 시스템 콜에 대응하는 번호를 확인하고 그에 맞는 서비스 루틴을 호출한다.

4. 그 이후에 커널 모드에서 사용자 모드로 다시 전환 하게 된다.

이 유저 모드 -> 커널 모드 -> 유저 모드로 전환 되는 과정에서 인터럽트와 컨텍스트 스위칭이 발생한다.

![](https://user-images.githubusercontent.com/44136364/136745929-f1b7afba-708b-4e65-8273-41d12ef459ff.png)
- 파이프라이닝을 적용하지 않았을 때와 적용했을 때의 차이

물론 파이프라인을 적용하지 않으면 하나의 시스템 콜 처리속도는 빠를 수 있지만.

계속해서 시스템 영역과 유저 영역을 계속 오가는 컨텍스트 스위칭이 발생하고 결론적으로 성능 저하를 일으킬 수 있다.

이에 반해 파이프라인을 적용하면 적용하지 않았을 때에 비해 적은 read 시스템 콜이 호출되고 많은 응답이 write 시스템콜 안에서 처리되게 되어 더 효율적이라고 할 수 있다.


## 4. Redis Pipeline with Spring

스프링에서 RedisTemplate에서 내부적으로 파이프라인닝을 적용할 수 있는 메서드들을 제공한다.

파라미터로 파이프라인을 적용할 것인지에 대한 플래그인 pipeline을 넘겨줄 수 있는 execute 메서드를 사용하거나 명시적으로 excutePipelined 메서드를 사용할 수 있다.


~~~java
public <T> T execute(RedisCallback<T> action, boolean exposeConnection, boolean pipeline)
~~~

~~~java
public List<Object> executePipelined(RedisCallback<?> action)
~~~

아래에서 메서드 내부의 일부분을 살펴보면 execute 메서드를 실행할 때 openPipleline() 메서드를 통해 파이프라인을 적용하는 것을 화인할 수 있다.

~~~java
return execute((RedisCallback<List<Object>>) connection -> {

	connection.openPipeline();
	boolean pipelinedClosed = false;
	try {

		Object result = executeSession(session);
		if (result != null) {
      
			throw new InvalidDataAccessApiUsageException(
				"Callback cannot return a non-null value as it gets overwritten by the pipeline");
					}
			  List<Object> closePipeline = connection.closePipeline();
				pipelinedClosed = true;
				return deserializeMixedResults(closePipeline, resultSerializer, hashKeySerializer, hashValueSerializer);
		} finally {

			if (!pipelinedClosed) {

				connection.closePipeline();
			}
	  }
});
~~~

이전에 [스프링 내부적으로 적용된 템플릿 콜백패턴](https://blog.minseong.kim/spring-template-callback-pattern.html)에 대해 정리하면서 JdbcTemplate, RestTemplate 내부 구조를 학습하였었는데 RedisTemplate도 이들과 크게 다르지 않아 많은 도움이 되었다.

<!-- ## 글을 마치며

파이프라인을 적용했을 때 빠른 이유를 CS 이론을 근거로 알아보았다.

이번 계기를 통해 다시 한번 메서드 하나로 파이프라인을 적용할 수 있는 스프링의 추상화에 다시 한번 감탄하게 되었다.

TCP의 지향점인 신뢰
어떤 기술이던 간에 완벽한 기술은 없고 
어떻게든 이를 해결하기위해 Pipleling을 통해 최대한 RTT를 최소화 하기 위해 최적화 한 부분이 매우 흥미로웠다. -->

## 5. 출처

- [레디스 공식문서 - 파이프라이닝](https://redis.io/topics/pipelining)

- [스프링 공식문서 - 파이프라이닝](https://docs.spring.io/spring-data/redis/docs/current/reference/html/#pipeline)

- [Mozilla Web Docs - HTTP 1.x](https://developer.mozilla.org/en-US/docs/Web/HTTP/Connection_management_in_HTTP_1.x)

- [Redis Pipeline with Spring Data](https://m.blog.naver.com/PostView.nhn?blogId=willygwu2003&logNo=130172698244&proxyReferer=https:%2F%2Fwww.google.com%2F)

