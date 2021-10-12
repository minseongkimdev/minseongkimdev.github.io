---
title: "nGrinder과 Pinpoint를 통한 부하 테스트 - (1) 테스트 및 모니터링 환경 구축 이야기"
layout: post
category: Spring
---

## 1. 들어가면서

대규모 트래픽 환경에서 안정적인 서비스 운영을 위해 신규 API에 대한 부하 테스트는 필수적이라고 할 수 있다.

진행하고 있는 프로젝트에서 부하 테스트를 진행하기 위해 AWS환경에 Naver사의 APM 오픈소스인 Pinpoint와 부하테스트 오픈소스 nGrinder를 EC2 인스턴스에 세팅하였다.

![](https://user-images.githubusercontent.com/44136364/136798988-d9b91621-5471-4ca1-92e4-e063eb2aff05.png)

이 글을 통해 실험 환경을 세팅을 하면서 습득한 지식과 경험에 대한 이야기를 해보려고 한다.


## 2. nGrinder란?

nGrinder에 대해 먼저 알아보자.

![](http://jmlim.github.io/public/post/ngrinder/ngrinder-system-architecture.png)

nGrinder의 구조는 위와 같지만 이해하기 쉽게 추상화를 해보면 아래와 계층 구조를 띈다.

![](https://user-images.githubusercontent.com/44136364/136798659-d43dd4ae-e00e-49fa-929a-7167d4d3fc10.png)

그리고 아래와 같은 구성요소들로 이뤄져있다.

- Controller는 부하 테스트를 제어하고 모니터링
- Agent : 부하를 발생하는 Virtual User를 생성
- Vitual User : 가상 유저 수를 의미 (Agent * Process * Thread)

러프하게 회사로 비유를 하자면 아래와 같다.

Controller : 경영진, Agent : 중간관리자, Virtual : 실무자


기능적인 부분을 설명하면 Gradle에도 사용되는 Groovy를 통해 테스트 시나리오 스크립트를 쉽게 작성할 수 있다.

![](https://user-images.githubusercontent.com/44136364/136924045-8b437855-66c1-4a0a-950a-14a490fe34bd.png)
또한 위와 같이 부하테스트에 대한 직관적인 인터페이스를 지원하며 부하테스트 현황을 쉽게 모니터링 할 수 있다는 장점이 있다.

## 3. Pinpoint 좀 더 알아보기

Naver사에서 개발한 Pinpoint는 **대규모 분산 시스템의 성능을 분석하고 문제를 진단 처리하는 플랫폼**이라고 소개하고 있다.

특히 마이크로서비스와 같이 시스템의 복잡도가 높으면 장애나 성능 문제가 발생했을 때 해결이 어려워 이를 추적할 수 있는 새로운 플랫폼의 필요성을 느껴 개발하였다고 소개하고 있다.

![](https://d2.naver.com/content/images/2015/06/helloworld-1194202-1.png)
- 출처 : Naver D2

특히 마이크로서비스에서 Node1 에서 보낸 N개의 메세지와 Node에서 수신한 N`개의 메세지 간의 관계를 추적하기 어려웠고 이를 [Google Dapper](https://research.google/pubs/pub36356/) 추적 방법에서 힌트를 얻어 HTTP 헤더에 메세지 태그 정보를 넣고 이를 연결고리로 활용해 추적했다고 한다. 자세한 내용은 해당 [기술블로그](https://d2.naver.com/helloworld/1194202)에서 확인할 수 있다.

## 3. 시행착오

Major GC가 발생할 때 어느정도의 STW가 발생하는지 확인하고자 하였으나, 아래와 같이 Minor GC가 짧은 주기로 계속해서 발생하면서 CPU 사용률이 100%에 가까웠고 Baseline이 유지되었다.

![](https://user-images.githubusercontent.com/44136364/136896020-ad34425f-7b9c-4c46-b029-4255f6dde1f5.png)

이는 Heap의 Young영역의 크기가 작아 Minor GC가 자주 발생하기 때문이라고 추측 하였다.

이전의 GC에 대해 학습한 지식을 토대로 이러한 추측을 해보았는데 간단하게 GC의 동작원리를 살펴보자.

#### G1 GC

Java 11버전으로 프로젝트를 진행하고 있기 때문에 기본적으로 G1 GC를 사용한다. (자바 9부터 디폴트 GC로 채택되었으며, 9 이전 버전은 JVM 옵션에 `-XX:+UseG1GC` 명령어를 통해 명시적으로 추가할 수도 있다.)

아래는 고전적인 GC방식에서 Heap 영역을 아래와 같이 나눠 관리하였다. (CMS GC이전)

![](https://databricks.com/wp-content/uploads/2015/05/Screen-Shot-2015-05-26-at-11.35.50-AM-1024x302.png)

하지만 G1 GC에서는 아래와 같이 Heap 영역을 동일한 크기의 격자 모양으로 나누고 각 영역에 인스턴스들을 할당하는 형태이다.

![](https://databricks.com/wp-content/uploads/2015/05/Screen-Shot-2015-05-26-at-11.38.37-AM.png)

이러한 형태를 띄는 이유는 사용자가 기대하는 Stop The World[^1] 시간을 달성하기 위함이다.

그렇다면 STW 시간을 사용자가 원하는 만큼 조정하는 것과 Heap 영역을 격자모양으로 나눈것과 무슨 연관이 있는것일까?

G1 GC의 동작원리를 살펴보면 해답을 찾을 수 있다.

설명하자면 G1 GC는 Heap 전역에서 인스턴스가 GC의 대상인지 아닌지 식별한다.

그렇게 식별이 끝나면 각 격자의 비어있는 영역의 크기를 알 수 있게 되고, GC를 진행했을 때 많은 양의 공간을 확보할수 있는 부분에 집중해서 GC를 진행하게 된다. (그래서 G1 Garbage-First GC라는 명칭이 붙여졌다)

여기서 유저가 정의한 STW시간을 충족하기 위해서 GC를 진행할 격자의 수를 계산하여 목표를 달성할 수 있는 것이다. 

다시 돌아와서 `-XX:NewRatio` 옵션을 통해 Young 영역의 사이즈를 늘려 다시 테스트를 하였더니 아래와 같이, Young영역에 대해 Minor GC가 적절히 발생하면서 Old영역으로 넘어가는 인스턴스들이 증가하여 Baseline의 높이가 점점 높아지는 것을 확인할 수 있다.

![](https://user-images.githubusercontent.com/44136364/136910621-5e9583cd-1c1b-4611-a902-acaaa553c1de.png)


## 테스트 시에 주의할 점

![](https://dz2cdn1.dzone.com/storage/temp/13984218-warmup_time.png) 
- 출처 : DZone


HotSpot VM을 기준으로 테스트를 진행할 때 부하를 발생시키고 나서 어느 정도의 시간동안 성능이 점점 개선되는 양상을 보인다.

테스트는 실제로 서버를 운영할 때와 가장 비슷한 환경에서 해야 유의미하기 때문에 JVM이 구동되고 부하를 가한후 어느정도의 시간이 지나야 유의미한 결과를 얻을 수 있다.

이는 HotSpot VM의 JIT 컴파일러와 클래스 로더와 연관이 있다.

#### JIT 컴파일러

소스 코드(.java)를 바이트 코드(.class)로 변환하고 이 바이트 코드는 JVM 에 있는 인터프리터를 통해서 읽혀지면서 JVM 이 동작하고 있는 머신에서 실행 가능한 native 코드로 변환되어 수행된다. 

이 때 특정 영역의 코드가 자주 읽히게 되면 그 영역을 hot code 영역이라고 부른다.

**JIT 컴파일러는 이 hot code 영역을 좀 더 효율적으로 다루기 위해 등장한 컴파일러다.** 인터프리터가 바이트 코드를 읽던 도중 어떤 영역을 hot code 영역이라고 판단하면 JIT 컴파일러에게 컴파일 요청을한다.

JIT 컴파일러는 이 요청을 받고 바이트 코드를 최적화해서 native 코드로 컴파일 한다. 이렇게 만들어진 native 코드는 이후 같은 hot code 영역을 읽으려 할 때 인터프리터를 사용하지 않고 최적화된 native 코드를 수행할 수 있도록 한다.
 
**JIT 컴파일러는 바이트 코드를 읽으면서 어떤 메소드가 많이 사용되는지 확인하고 많이 사용되는 메소드가 있다면 이를 컴파일 대상으로 삼는다.** 여기서 말하는 컴파일이란 단순히 인터프리터가 native 코드로 변환했던 것을 의미하는 게 아니라 최적화된 native 코드를 얻어내는 것을 의미한다. 이렇게 컴파일된 native 코드는 캐싱되어 이후에 번역을 할 필요가 없어진다.

조금 더 구체적으로 알아보면 메소드가 얼마나 자주 호출되는지 판단하기 위해서 메소드 별로 각자 두 개의 카운터를 가지고 캐싱 대상의 우선순위를 결정하게 된다.

수행 카운터 : 메소드 실행시 증가
백에지 카운터 : 높은 바이트 코드 인덱스에서 낮은 바이트 코드 인덱스로 컨트롤 흐름이 변경될 때 증가 (반복문 등)


#### 클래스 로더

클래스 로더는 컴파일 타임이 아닌 런타임에 클래스를 JVM의 Runtime Data Area에 로딩할 수 있게 해주어 컴파일 타임을 줄여주는 기술이다.

런타임 중 클래스가 필요한 시점에 클래스를 로드하기 때문에 JVM을 시작하고 부하테스트를 시작한지 충분한 시간이 지나야 필요한 클래스들이 Runtime Data Area에 적재되어 더 정확한 테스트 결과를 얻을 수 있는 것이다.


## 글을 마치며

테스트 및 모니터링 환경을 구축하면서 발생한 현상들에 대해 이전에 공부했던 JVM과 Runtime Data Aread의 Heap영역 구조 및 GC의 동작원리 등의 이론을 적용 해서 이해해보니 흥미로웠고 CS 지식과 이론의 중요성을 다시금 깨닫는 계기가 되었다.

다음 글엔 이러한 실험 환경에서의 스레드덤프, 힙덤프를 통한 병목 진단과 Redis를 통한 조회 API 응답 속도 개선 대해 알아볼 예정이다.


## 참고

-  [NGrinder  - Github Page](https://naver.github.io/ngrinder/)
-  [대규모 분산 시스템 추적 플랫폼, Pinpoint - Naver D2](https://d2.naver.com/helloworld/1194202)
-  [The Garbage First Garbage Collector - Oracle](https://www.oracle.com/java/technologies/javase/hotspot-garbage-collection.html)
-  [Why Many Java Performance Tests are Wrong - DZone](https://dzone.com/articles/why-many-java-performance-test)
-  [Heap Tuning Parameters - Oracle Docs](https://docs.oracle.com/cd/E19900-01/819-4742/abeik/index.html)
-  [Java Garbage Collection](https://d2.naver.com/helloworld/1329)
- [JVM 튜닝](https://imp51.tistory.com/entry/G1-GC-Garbage-First-Garbage-Collector-Tuning)
## 주석

[^1]: STW란 GC을 실행하기 위해 JVM이 애플리케이션 실행을 멈추는 것이다. STW가 발생하면 GC를 실행하는 쓰레드를 제외한 나머지 쓰레드는 모두 작업을 멈추고 GC 작업을 완료한 이후에야 중단했던 작업을 다시 시작한다. 어떤 GC 알고리즘을 사용하더라도 STW는 발생하며. (대개의 경우 GC 튜닝이란 이 STW 시간을 줄이는 것이다.)

