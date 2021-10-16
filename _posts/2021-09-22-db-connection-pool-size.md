---
title: "How to choose DB Connection Pool Size?"
layout: post
category: DB
---

## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. Hikari CP의 동작원리](#2-hikari-cp의-동작원리)
- [3. Connection Pool Size가 무조건 크다고 좋을까?](#3-connection-pool-size가-무조건-크다고-좋을까)
  - [Disk Contention (디스크 경합)](#disk-contention-디스크-경합)
  - [Context Switching](#context-switching)
- [4. 그렇다면 적절한 Size는 어느정도 일까?](#4-그렇다면-적절한-size는-어느정도-일까)
  - [core_count * 2](#core_count--2)
  - [effective_spindle_count](#effective_spindle_count)
- [5. 부하테스트](#5-부하테스트)
- [출처](#출처)
- [참고서적](#참고서적)

## 1. 들어가면서

본 내용은 [Database Connection Pool의 등장배경과 필요성](https://blog.minseong.kim/dbcp-principle.html)의 후속글이며 진행하고 있는 Spring 프로젝트에서 적절한 DB Connection Pool 사이즈를 찾는 과정를 이 글을 통해 공유해보고자 한다.

DB와 WAS 사이에서 TCP를 기반으로 Connection을 맺고 이전에 작성한 [Redis Pipleline](https://blog.minseong.kim/about-redis-pipelining.html)에 대한 글에서도 언급했듯이 TCP의 3-Way-Handshake으로 Connection을 맺기 때문에 네트워크 병목이 발생할 수 있다.

이뿐만 아니라 [MySQL 8.0 공식문서](https://dev.mysql.com/doc/refman/8.0/en/insert-optimization.html)에서 INSERT문을 기준으로 Connection을 맺는데 비용이 많이 든다고 명시하고 있다.

그리고 유저가 갑자기 몰리게 된다면 병목으로 인해 Connection 객체를 얻기 위한 시간이 엄청나게 소요될 것이다.


따라서 대규모 트래픽 환경에서는 Connection Pool은 중요하다.

현재 진행하고 있는 Spring 프로젝트에는 Connection Pool중 Hikari Connection Pool을 적용하였으며 실제 부하테스트를 통해 최적의 사이즈를 찾아가는 과정에 대해 다뤄보고자 한다.

## 2. Hikari CP의 동작원리

Hikari CP에서는 내부적으로 ConcurrentBag이라는 구조체를 이용해서 Connection을 관리하고, ConcurrentBag.borrow() 라는 메서드를 통해 사용 가능한 Connection을 리턴하도록 되어 있다.

Thread가 Connection Pool에 Connection을 요청하는 추상화 해서 도식화 해보면 아래와 같다.

![](https://user-images.githubusercontent.com/44136364/137590034-e0f84530-c017-4d8d-8ee3-372f68d27100.png)


특히 HikariCP은 현재 Thread가 이전에 사용한 Connection 정보가 있는지 확인하고, 이전에 사용했던 Connection List를 확인하고 이를 우선적으로 리턴하는 특징이 있다.

![](https://user-images.githubusercontent.com/44136364/137590886-d88c0648-6039-4d63-9e69-50660ed9f000.png)

그리고 Thread가 HikariCP로부터 Connection을 얻을 수 없으면 HandOff Queue에서 Connection를 얻기 까지 기다린다.
지정한 Timeout 시간이 넘어가면 Exception이 발생한다.

(HandOff 넘기다, 위임하다라는 사전적 의미를 가지고 있다. Connection을 기다리고 있는 Thread에게 Connection을 넘겨주기 위한 Pool이라는 의미에서 Handoff Queue 라고 추측해본다)

Timeout은 `connectionTimeout` 옵션을 통해 조정할 수 있고 최소 250ms로 짧게 할 수 있으며 디폴트 값은 30초이다.
(다른 옵션들은 [공식문서](https://github.com/brettwooldridge/HikariCP)에서 확인할 수 있다.)

![](https://user-images.githubusercontent.com/44136364/137591059-44d39d58-839a-4eed-925e-d9d4324f3028.png)

그리고 정상적으로 트랜잭션이 commit되거나 문제가 발생하여 rollback되면 connection.close()가 호출되어 Connection을 Pool에 반납한다.

이때 HandOff Queue에서 Connection을 기다리는 Thread가 있다면 반납된 Connection을 HandOff Queue에 넣어준다.

[ConcurrentBag.borrow()](https://github.com/brettwooldridge/HikariCP/blob/b5f5700e2dfdb23be9c7d01722df26eff134b1ef/src/main/java/com/zaxxer/hikari/util/ConcurrentBag.java#L120)와[ConcurrentBag.requite()](https://github.com/brettwooldridge/HikariCP/blob/b5f5700e2dfdb23be9c7d01722df26eff134b1ef/src/main/java/com/zaxxer/hikari/util/ConcurrentBag.java#L175)
에서 직접 코드를 확인할 수 있디.


## 3. Connection Pool Size가 무조건 크다고 좋을까?

지금까지 살펴본바로 이런 궁금증을 가져볼 수 있다.

> Connection Pool Size와 Thread의 수를 늘리면 점점 성능이 좋아지지 않을까?

하지만 아래와 같은 이유로 Pool Size와 Thread수에 비례해서 성능 개선에는 한계가 있다.

### Disk Contention (디스크 경합)

RAM에 캐시되어 있지 않은 데이터를 Disk에서 Random Access를 통해 찾아야 하는 경우, 많은 Connection들이 동시에 테이블과 인덱스에 엑세스 해야하므로 경합이 발생해 성능 향상에 분명한 한계가 있다.

### Context Switching

Thread의 수가 코어 수보다 많게 되면 컨텍스트 스위칭이 발생하면서 Thread의 Stack영역의 데이터를 로드하는 등의 오버헤드가 발생하여 Thread의 수를 무작정 늘려도 성능 향상에 분명한 한계가 있다.


## 4. 그렇다면 적절한 Size는 어느정도 일까?

[HikariCP의 공식레포](https://github.com/brettwooldridge/HikariCP)에서는 다음과 같이 소개하고 있다.

>  connections = (core_count * 2) + effective_spindle_count

(PostgreSQL을 기반으로 검증된 공식이지만 러프하게 여러 데이터베이스에 적용 가능할것이라고 기술하고 있다.)

위 공식 (core_count * 2)와 effective_spindle_count을 더한 값인데 왜 이렇게 도출었는지 이유를 생각해보자.

### core_count * 2

Context Switching 및 Disk IO와 관련이 있다.

CPU의 처리속도는 Disk IO보다 훨씬 빠르기 때문에, Disk IO 작업에서 Blocking된 동안 다른 Thread의 작업을 처리할 수 있는 여유가 생긴다.

여유 정도에 따라 멀티스레드 작업을 수행할 수 있게 된다. HikariCP가 선택한 계수는 2인 것이다.

### effective_spindle_count

이는 하드디스크와 관련이 있다.

![](https://www.open.edu/openlearn/ocw/pluginfile.php/1467533/mod_oucontent/oucontent/80661/555c16f3/0c264d1b/tm112_1_ol_s5_f1_4.tif.jpg)
- 출처 : OpenLearn

하드디스크의 원리를 간단히 알아보면 0,1의 디지털 신호를 원반 형태의 Platter에 기록하고 Spindle 모터를 통해 이를 회전시키면서 데이터를 읽어야 하는 부분으로 Head가 이동하면서 저장된 데이터를 읽는 원리이다.

따라서 Spindle 모터의 수는 DB 서버가 관리할 수 있는 동시 I/O의 요청 수를 뜻한다. Disk가 N개 이면, N개의 IO요청을 처리할 수 있는 것이다.


결론적으로 CPU의 처리 효율과 Disk IO의 처리 효율을 고려하여 
아래와 같은 공식이 도출되었음을 추측해볼 수 있다.
>  connections = (core_count * 2) + effective_spindle_count


하지만 아래와 같이 중요한 주의사항이 기술되어 있다.

> However you choose a starting point for a connection pool size, you should probably try incremental adjustments with your production system to find the actual "sweet spot" for your hardware and workload.

> 프로덕션 시스템의 Connection Pool 사이즈를 조절하면서 가장 적합의 사이즈인 "sweet spot"을 찾아야 한다.

안내되어 있는대로 부하 테스트를 통해 현재 진행하고 있는 Spring 프로젝트의 "sweet spot"을 찾아보도록 하자.


## 5. 부하테스트

진행하고 있는 프로젝트에서 부하 테스트를 진행하기 위해 AWS환경에 Naver사의 APM 오픈소스인 Pinpoint와 부하테스트 오픈소스 nGrinder를 EC2 인스턴스에 세팅하였다.

![](https://user-images.githubusercontent.com/44136364/136798988-d9b91621-5471-4ca1-92e4-e063eb2aff05.png)

프로젝트의 RDS의 vCPU 2개와 SSD 1개로 구성되어 있다. SSD이기 때문에 공식에 완벽하게 들어맞진 않지만 러프하게 계산해보면 (2*2) + 1 = 5개의 Connection Pool Size를 도출해낼 수 있다.

<!-- > 부하 테스트 시나리오 : 점심시간 이후 티타임에 로그인 후 카페 리스트를 조회하는 사용자가 급증하는 경우 -->


<!-- ## 글을 마치며

Connection Pool Size 공식은 공식일 뿐이다. 오히려 세상에 수많은 서버 환경에서 적절한 커넥션 풀을 하나의 공식으로 완벽하게 표현한다는 것 자체가 역설이 아닐까라고 생각한다.

처음에 공식대로 Connection Pool 사이즈를 잡아보고, 상황에 맞게 실제 테스트를 해보면서 조절 하는게 합리적이지 않을까 라고 생각한다.

학습한 CS지식을 기반으로 -->

## 출처


- [HikariCP Dead lock에서 벗어나기 (이론편) - 우아한형제들 기술블로그](https://techblog.woowahan.com/2664/)
- [Number Of Database Connections](https://wiki.postgresql.org/wiki/Number_Of_Database_Connections)
- [About Pool Sizing - HikariCP Github Repository](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

- [What is effective spindle count](https://dba.stackexchange.com/questions/228663/what-is-effective-spindle-count)

## 참고서적 
- [자바 성능 튜닝 이야기](http://www.kyobobook.co.kr/product/detailViewKor.laf?mallGb=KOR&ejkGb=KOR&barcode=9788966260928)
