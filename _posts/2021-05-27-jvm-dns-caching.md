---
title: "JVM의 DNS Caching"
layout: post
category: Java
---



## 0. 글의 순서

- [1. JVM DSN Caching이란?](#1-jvm-dns-caching이란?)
- [2. JVM TTL 설정](#2-jvm-ttl-설정)
- [출처](#출처)
- [각주](#각주)


## 1. JVM DNS Caching이란?

JVM에서 DNS LookUp[^1]시에 Time-To-Live[^2] 전략을 사용하지 않고 한번 LookUp한 도메인 이름은 JVM이 종료되지 않는한 **영구적으로 캐싱**하고 있다.

왜냐하면 DNS Spoofing[^3]공격을 막기 위한 전략이다.
이러한 특징 때문에 외부의 IP가 변경되어도, DNS Caching으로 인해, 예전의 IP로 접속하는 문제가 생길 수 있다.

이를 방지하는 방법은 두가지가 있다.

1. JVM 재실행
2.  Java SercurityManager의 Policy를 수정

## 2. JVM TTL 설정

DNS Name Lookups에 대한 JVM TTL을 설정하면, 도메인에 대한 정보가 일정 시간 동안만 캐싱된다.

JVM의 TTL을 수정하려면, [networkaddress.cache.ttl](https://docs.oracle.com/javase/7/docs/technotes/guides/net/properties.html) 프로퍼티값을 세팅하면 된다. (-1로 설정하면 영구적으로 캐싱한다.)

설정하는 방법은 두가지가 있다.

- 전역적으로 모든 JVM의 애플리케이션에 대해 설정
	- `$JAVA_HOME/jre/lib/security/java.security` 파일에서 `networkaddress.cache.ttl`의 값을 수정하면 된다.

- 특정 애플리케이션에 대해 설정
	- ttl을 설정하려는 애플리케이션 소스상에서 `java.security.Security.setProperty("networkaddress.cache.ttl" , "60");` 와 같이 설정할 수 있다.


## 출처

#### 공식문서

- [Oracle Docs - Networking Properties
](https://docs.oracle.com/javase/7/docs/technotes/guides/net/properties.html#nct)

- [Amazon Docs - Setting the JVM TTL for DNS Name Lookups
](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/java-dg-jvm-ttl.html)


## 각주

[^1]: DNS Lookup : DNS에서 IP에 해당하는 도메인을 찾는 과정을 말함.


[^2]: Time To Live : 네트워크상에서 데이터의 유효 기간을 나타내기 위한 방법. 정해진 유효기간이 지나면 데이터는 폐기됨. 네트워크 상에서 데이터가 무한히 순환하는 것을 방지하는 역할을 함. DNS TTL은 DNS에 대한 캐시를 얼마동안 유지시킬지에 대한 정보를 뜻함.

[^3]: DNS Spoofing : DNS Cache Poisoning이라고도 하며, DNS 캐시에 악의적으로 다른 정보를 입력하여, 피해자가 다른 사이트에 연결되도록 하는 행위.