---
title: "nGrinder과 Pinpoint를 통한 부하 테스트 및 힙덤프 분석"
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

## 4. 본격적으로 부하 테스트 해보기 (feat. 힙덤프)

우선 앱의 가장 기본 기능이라고 할 수 있는 로그인 API를 테스트 해보았다.
다음은 부하 테스트 동안의 힙 사용량 추이이다.

푸른색 꺾은선 그래프는 힙의 사용량이고, 붉은 막대 그래프는 Full GC가 발생했을 때 걸리는 시간이다.

![](https://user-images.githubusercontent.com/44136364/139086576-e681beef-db4f-4ff0-a73b-7529c5a26a91.png)

그래프를 분석해보면 부하를 가하고 나서 얼마 동안은 Minor GC만 계속해서 발생하는 것을 확인할 수 있다.

하지만 어떤 이유에서인지 인스턴스들이 전부 회수되지 않고 Young영역에서 Old 영역으로 넘어가게 된다.

![](https://user-images.githubusercontent.com/44136364/139247969-9c056734-3b41-4b70-b746-eca501ceab00.png)

그리고 계속해서 Old영역이 차올라서 6초 가량의 Full GC가 발생하여도 충분히 메모리가 확보되지 않았다.

Full GC가 발생해도 메모리 공간을 확보하지 못하는 이유를 분석하기 위해 힙 사용량이 거의 가득찰 때 힙덤프를 떠서 확인해보았다.

아래의 명령어를 통해 힙덤프 파일을 생성할 수 있다.

`$jmap -dump:format=b, file=<파일명> <PID>`

이렇게 생성된 파일을 [MAT](https://www.eclipse.org/mat/)라는 힙 메모리 분석 툴을 통해 확인할 수 있다.

![](https://user-images.githubusercontent.com/44136364/139091618-8cba46eb-38a6-40cc-a138-6af83f6eef59.png)


`org.apache.catalina.session.StandardManager`가 전체의 88.26%를 차지하는 것을 확인할 수 있다.

**왜 Major GC후에도 세션 인스턴스들이 정리되지 않을까?**

[StandardManager](https://tomcat.apache.org/tomcat-9.0-doc/api/org/apache/catalina/session/StandardManager.html)란 세션이 지속될 수 있도록 관리해주는 클래스이다.


그리고 StandardManager의 부모 클래스인 ManagerBase는 아래와 같이 ConcurrentHashMap을 통해 세션들을 관리한다.

~~~java
protected Map<String, Session> sessions = new ConcurrentHashMap<>();
~~~

아래의 그림처럼 세션들을 담고 있는 ConcurrentHashMap인 sessions 인스턴스가 Reachable[^1] 하기 때문에, **명시적으로 세션 원소를 제거해주지 않는 이상 이 세션 원소들은 Reachable하다.**

![](https://user-images.githubusercontent.com/44136364/139096108-cac8db3b-7b8b-439d-8ef4-4742c5933f39.png)


세션은 Timeout(기본값 60초)을 기준으로 명시적으로 sessions 컬렉션에서 제거된다.

**따라서 Timeout 시간이 지나기 전에는 세션 인스턴스가 GC의 대상이 아니므로, 짧은 시간동안 많은 세션 인스턴스를 생성하면 힙 메모리가 부족해지는 현상이 발생하는 것이다.**


## 마지막으로 주의할 점 두 가지

### 1. 운영중인 서버에서 절대 힙덤프 파일을 생성하지 말것.

다음은 nGrinder에서 확인한 부하 테스트 동안의 애플리케이션의 TPS이다.

![](https://user-images.githubusercontent.com/44136364/139098258-0e784d53-d1cf-4eb8-8046-5d30e28b6153.png)

힙덤프 파일을 생성한 시점에 TPS가 0에 수렴하는 것을 확안헐 수 있다.

서버의 자원이 힙덤프 파일을 생성하는데 쓰이고 이 작업이 꽤 큰 리소스를 점유하므로 실제로 운영중인 서버에서 힙덤프 파일을 생성하지 않도록 주의하자.

### 2. 부하를 가한 후 충분한 시간이 지난 뒤의 결과를 확인할 것.
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
 
**JIT 컴파일러는 바이트 코드를 읽으면서 어떤 메소드가 많이 사용되는지 확인하고 많이 사용되는 메소드가 있다면 이를 컴파일 대상으로 삼는다.**

 여기서 말하는 컴파일이란 단순히 인터프리터가 native 코드로 변환했던 것을 의미하는 게 아니라 최적화된 native 코드를 얻어내는 것을 의미한다. 이렇게 컴파일된 native 코드는 캐싱되어 이후에 번역을 할 필요가 없어진다.

조금 더 구체적으로 알아보면 메소드가 얼마나 자주 호출되는지 판단하기 위해서 메소드 별로 각자 두 개의 카운터를 가지고 캐싱 대상의 우선순위를 결정하게 된다.

수행 카운터 : 메소드 실행시 증가
백에지 카운터 : 높은 바이트 코드 인덱스에서 낮은 바이트 코드 인덱스로 컨트롤 흐름이 변경될 때 증가 (반복문 등)


#### 클래스 로더

클래스 로더는 컴파일 타임이 아닌 런타임에 클래스를 JVM의 Runtime Data Area에 로딩할 수 있게 해주어 컴파일 타임을 줄여주는 기술이다.

런타임 중 클래스가 필요한 시점에 클래스를 로드하기 때문에 JVM을 시작하고 부하테스트를 시작한지 충분한 시간이 지나야 필요한 클래스들이 Runtime Data Area에 적재되어 더 정확한 테스트 결과를 얻을 수 있는 것이다.


## 글을 마치며

실제로 애플리케이션에 부하 테스트를 해보고 힙덤프를 떠서 문제의 원인을 찾는 과정을 진행하였다.

이론적으로만 공부했던 JVM과 Runtime Data Aread의 Heap영역 구조등의 실제로 발생한 문제를 해결하는데 응용해보니 흥미로웠고 CS 지식과 이론의 중요성을 다시금 깨닫는 계기가 되었다.

다음 글에서는 다른 테스트 시나리오를 통해 부하 테스트를 진행해볼 예정이다.


## 참고

-  [NGrinder  - Github Page](https://naver.github.io/ngrinder/)
-  [대규모 분산 시스템 추적 플랫폼, Pinpoint - Naver D2](https://d2.naver.com/helloworld/1194202)
-  [The Garbage First Garbage Collector - Oracle](https://www.oracle.com/java/technologies/javase/hotspot-garbage-collection.html)
-  [Why Many Java Performance Tests are Wrong - DZone](https://dzone.com/articles/why-many-java-performance-test)
-  [Heap Tuning Parameters - Oracle Docs](https://docs.oracle.com/cd/E19900-01/819-4742/abeik/index.html)
-  [Java Garbage Collection](https://d2.naver.com/helloworld/1329)
- [JVM 튜닝](https://imp51.tistory.com/entry/G1-GC-Garbage-First-Garbage-Collector-Tuning)
- [Java Reference와 GC](https://d2.naver.com/helloworld/329631)

## 주석

[^1]: Reachable에 대해 간단히 설명하면, 가비지 컬렉터가 가비지인지 판별하기 위해 사용하는 개념이며 어떤 객체에 유효한 참조가 있으면 Reachable'로, 없으면 Unreachable로 구별하고, Unreachable 객체를 가비지로 간주해 GC를 수행한다. 자세한 내용은 [네이버 기술블로그](https://d2.naver.com/helloworld/329631)에서 확인할 수 있다.
