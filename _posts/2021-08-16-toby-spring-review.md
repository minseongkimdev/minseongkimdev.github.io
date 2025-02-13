---
title: "토비의 스프링 서평 (feat. 내가 생각하는 스프링 프레임워크)"
layout: post
category: Review
---

## 0. 프롤로그

블로그를 개설한 이후 처음으로 서평을 쓰는 책이다. 그만큼 나에게 많은 깨달음을 주었고 스프링에 대한 이해도를 높여주었다.

그래서 이 책을 읽으면서 느낀점과 스프링 프레임워크에 대한 나의 생각을 기록하기로 결심했다.


## 1. 토비의 스프링에 대한 오해와 진실

> 토비의 스프링과 다른 책과의 가장 큰 차이점은 **전개 방식**이라고 생각한다.
일부러 냄새나는 코드를 보여주고 철저히 **객체지향의 원칙**을 지키기 위해 서서히 리팩토링 해가며
**스프링 왜 이런 구조를 띄고 있고** **DI, AOP, PSA 같은 개념이 왜 도입될 수 밖에 없었는지**에 이해시켜 준다.


사실 스프링 개발 경험이 없어도 워낙 유명한 책이다 보니 이 책의 존재는 알고 있었다.

책을 구매해서 실물을 영접하니 두께에 압도되어 한번 놀라고, 이 두꺼운 책이 두 권이라는 사실에 한번 더 놀랐다.

하지만 1장을 읽자마자 **단순히 긴 책이 아니라는 생각이 들었다.**

일부러 냄새가 나는 코드를 보여주고, **객체지향 원칙에 입각**해 서서히 리팩토링해가는 과정을 보여주며 독자를 이해시켜주었고, 친절하게 설명해주기 위해 길다는 것을 이해하게 되었다. 

내가 가장 인상깊게 읽은 두 부분을 소개하면서 좀 더 구체적으로 설명해보겠다.

## 2. 결과보다는 과정에 집중한다

<!-- > '스프링 내부가 이렇게 구성되어 있다면 어떨까? 엄청 불편하겠지? 코드에서 냄새가 진동하겠지?' -->

토비의 스프링이 좋은 책이라고 평가받는 이유중 하나는 **결과론적으로 단순 지식을 전달하지 않기 때문**이라고 생각한다.

특히 JdbcTemplate에 **트랜잭션 동기화**를 적용하는 과정을 **서비스 추상화**를 통해 스프링이 왜 이런 구조를 취할수 밖에 없었는지 설명하는게 인상적이었다.

단순히 아래와 같이 결과론적으로 설명할 수 있었을 것이다.

> 스프링에서 트랜잭션 추상화를 제공한다.

하지만 책에서는 아래와 같은 플로우로 설명한다.

![](https://user-images.githubusercontent.com/44136364/131277003-8fbaee91-5896-45d9-967e-b7abfe4f0894.png)

<!-- 1. DB에 있는 정보를 업데이트 하다가 중간에 예외가 발생하거나 중단되면 어떻게 될까?
1. 여러개의 SQL이 사용되는 작업을 하나의 트랜잭션으로 취급되어야 하는 경우가 있다. -->
<!-- 3. 트랜잭션의 시작과 종료는 Connection 오브젝트를 통해 이뤄지고, commit() 또는 rollback()으로 트랜잭션을 종료하는 작업을 트랜잭션 경계설정이라고 한다. -->
1. 어떻게 여러개의 SQL을 같은 트랜잭션으로 묶을 수 있을까?
<!-- 4. JdbcTemplate안에서 Connection을 관리하기 때문에 매번 새로운 DB 커넥션과 트랜잭션을 만들어 사용한다.  -->
<!-- 이렇게 되면 Service 내에서 진행되는 여러 작업을 하나의 트랜잭션으로 묶는것이 불가능해 보인다. -->
2. DAO안에서 JDBC API를 직접 이용해서 경계설정을 하면, **비지니스로직과 데이터 엑세스 로직을 묶어버리는 결과를 초래한다.** 
<!--  -->
3. Service에서 만든 **Connection 오브젝트를 트랜잭션 경계 설정을 위해 DAO의 메소드로 일일이 넘겨주는 것도 상당히 불편하다.** 또한 Connection 파라미터가 DAO 인터페이스 메소드에 추가되면 **더이상 데이터 엑세스 기술에 독립적일 수 없다.**
<!--  -->
4. Service에서 생성한 Connetion 오브젝트를 특별한 저장소에(TransactionSynchronizationManager) 보관하고 DAO에서 이를 가져와서 사용하게 하면 DAO 인터페이스 메소드의 파라미터를 제거할 수 있다.
하지만 JDBC에 종속적인 Connection을 이용한 트랜잭션 코드가 Service에 등장하면서 **Service가 트랜잭션 방법에 의존적**이게 되었다.
5. DIP(의존성 역전 원칙)에 입각하여 PlatformTransactionManager를 통해 추상화 계층을 둬서 **애플리케이션 비지니스 로직과 그 하위에서 동작하는 로우레벨의 트랜잭션 기술로 아예 다른 계층**으로 나눈다.
6. 아래와 같이 다양한 트랜잭션 서비스 추상화 기법을 이용해 다양한 트랜잭션 기술을 일관된 방식으로 제어할 수 있게 되었다.
<!-- 7.  -->

![](https://user-images.githubusercontent.com/44136364/131058873-ded30e85-07ca-4f91-9b73-e7fb16470ea7.png)
<!-- 애플리케이션의 비지니스 로직과 그 하위에서 동작하는 로우레벨의 트랜잭션 기술이라는 아예 다른 계층의 특징을 갖는 코드를 분리한 것이다. -->


중요한 점은 위와 같은 리팩토링 과정은 임의로 진행되는 것이 아니라는 것이다.
철저히 SRP(단일책임원칙), DIP(의존성 역전원칙), OCP(개방폐왜원칙) 등의 객체지향 설계의 기본원칙을 근거로 삼는다.


이와 같은 전개는 읽는 사람 입장에서는 추상화 계층이 도입되어야만 했던 이유에 대해 차근차근 이해하기 쉬운 형태이지만, 저자 입장에서는 이러한 스텝을 설계하는 것이 결코 쉬운 일은 아니었을 것이다.


## 3. The Most Important Word, Why?

책을 읽기 전에 @Autowired를 공부하고 [블로그](https://www.half.kim/autowired-deep-dive.html)에 정리했었을 때는 단순히 스프링 PostBeanProsessor가 Advisor를 통해 Proxy를 만들어준다라고 결과론적으로만 알고 있었다.

하지만, 6장(AOP)에서 왜 이런 구조를 띄고 있는지 이해시켜주었고, 이러한 형태를 띄어야 하는 이유를 설득시켜주었다.

(설명하기 앞서 눈치 챘을수도 있지만 위의 트랜잭션 추상화에서 아직 Service에 트랜잭션 코드가 남아있다.
사실 이는 6장 AOP에 대한 복선이고 아래와 같이 AOP를 통해 다른 모듈로 환상적(?)으로 분리해낸다.)

![](https://user-images.githubusercontent.com/44136364/131298845-329e9c4a-98a9-4348-bdb7-85be12039aa9.png)

1. 트랜잭션 추상화를 적용했음 에도 불구하고 Service에 아직 트랜잭션 경계 코드가 남아있다.
<!-- 2. 메소드로 분리 402P -->
2. Service 인터페이스를 둬서 분리하고 ServiceTx는 Service를 구현한 다른 오브젝트(ServiceImpl)를 DI받아 기능 수행을 위임한다.
<!-- 5. 이렇게 되면 의존관계는 Client -> UserServiceTx -> UserServiceImpl -->
3. 프록시 패턴을 통해 부가기능은 마치 자신이 핵심 기능을 가진 클래스인것 처럼 꾸며서 클라이언트가 자신을 거쳐 핵심기능을 사용하도록 만든다.
<!--  -->
4. 하지만 프록시를 만드는 과정은 매우 번거롭다. 따라서 프록시 클래스 없이도 프록시 오브젝트를 런타임에 만들어주는 JDK 다이나믹 프록시 기술을 적용하여 프록시 클래스 코드 작성에 대한 부담을 덜고, 부가 기능 부여 코드가 여기저기 중복돼서 나타나는 문제도 일부 해결된다.
5. 동일한 기능의 프록시를 여러 오브젝트에 적용할 경우, 오브젝트 단위로 중복이 일어나는 문제 발생함. JDK 다이나믹 프록시와 같은 프록시 기술을 추상화한 스프링의 프록시 팩토리 빈을 이용해서 다이나믹 프록시 생성 방법에 DI를 도입하여 해결한다. 


6. 내부적으로 템플릿/콜백 패턴을 활용하는 스프링의 프록시 팩토리 빈 덕분에 부가기능을 담은 어드바이스와 부가기능 선정 알고리즘을 담은 포인트컷은 프록시에서 분리될 수 있고 프록시에서 공유해서 사용할 수 있게 된다.


7. 마지막으로 트랜잭션 적용 대상이 되는 빈마다 일일이 프록시 팩토리 빈을 설정해줘야 하는 불편함 발생한다.


8. 이를 해결하기 위해 스프링 컨테이너의 빈 포스트 프로세서를 통해 컨테이너 초기화 시점에서 자동으로 프록시를 만들어주는 방법을 도입한다.

이 과정을 통해 프록시를 적용할 대상을 일일이 지정하지 않고 패턴을 이용해 자동으로 선정할 수 있도록, 

클래스를 선정하는 기능을 담은 확장된 포인트컷을 사용했고 **결국 트랜잭션 부가기능을 어디에 적용 하는지에 대한 정보를 포인트컷이라는 독립적인 정보로 완전히 분리할 수 있다.**

위에서 설명한 것들을 모두 요약하면 **OOP를 통해 서비스 추상화를 통해 트랜잭션 기술에 종속적이지 않게 리팩토링 하고**.
그 이후에 **AOP를 통해 애플리케이션 로직에 남아있는 트랜잭션 코드를 완전히 분리**해냈다.

이 전개방식에서 알 수 있다 싶이 AOP는 OOP를 도와 핵심기능을 설계하고 구현할 때 객체지향적인 가치를 지킬 수 있도록 해준다는 것을 설명해주고 더불어 **포인트컷, 어드바이스** 등의 개념들이 **왜 필요할 수 밖에 없는지** 이해시켜준다.


## 4. 토비의 스프링이라고 쓰고 토비의 객체지향 프로그래밍이라고 읽는다.

<!-- > **리팩토링은 한 순간에 되는 것이 아니기에** 여러 스텝에 걸쳐 리팩토링 과정을 보여준다. -->


이 책을 통해 스프링에 대학 깊이 있는 학습을 할 수 있지만 OOP에 대한 공부도 많이 되었다.

부끄럽게도 객체지향 설계의 많은 부분을 암기에 의존하여 이해했었다. (캡상추다, SOLID ~)

하지만 스프링에 부분부분 적용된 OOP의 실제 케이스들을 계속해서 접하다보니 "아, 이런 경우에는 OOP의 이런 원칙을 적용하여 이렇게 개선할 수 있겠구나" 라는 생각을 할 수 있게 한 차원 높은 이해를 하게 해주었다.

사실 스프링 3번대 버전을 다룬 10년 더 전에 나온 책이 지금까지 베스트셀러인 이유는

스프링의 핵심 철학이 OOP에 있고 그 근간은 현재까지 유지되고 있으며, 이 책이 OOP를 기반으로 스프링 다루기 때문이라고 생각한다. 

<!-- 갑자기 마술을 부린게 아니다. 필연적으로 객체지향 원칙을 지키다보니 현재의 모습을 띄고 있는 것이다. -->

<!-- 스프링도 스프링이지만 객체지향 공부도 된다 -->

<!-- 객체지향 설계의 장점과 그 적용 과정을 보여주면서 -->

<!-- 인터페이스로 분리했는데 특정 기술에 종속됐네? 어떻게하지? 다음장에서 설명해줄게 ~  -->
<!-- 서비스 추상화로 기술과도 독립시켜버렸다. -->
<!-- 이런점에서책의 구성이 맘에 든다. 길어질수 밖에 없어 호불호가 갈릴수 있겠지만 하지만 개인적으로는 좋았다. -->

<!-- 현재는 스프링 5번대 까지 나왔지만 3.1 버전을 다루고 있다. -->
<!-- 하지만 스프링의 핵심원리는 3.1부터 그대로 이어지고 있다. -->


<!-- 이렇게 트레이드 오프 비교를 통한 설명은 주니어 개발자 입장에서 아래와 같은 깨달음도 주었다. -->

<!-- > 와 이런식으로 트레이드 오프를 비교 해가며 OOP에 입각하여 설계를 할 수 있구나.. -->

<!-- 
## 4. 그 외

거의 매 장마다 코드들이 자주 등장하는데 그 코드에 대한 테스트코드들도 함께 포함되어 있어 좋았다.

스프링 자체 뿐만 아니라 데이터 엑세스 기술과 같은 스프링과 연관 있는 것들까지 꽤 깊게 다룬다. -->



## 토비의 스프링을 읽는 팁

막간을 이용해 팁을 주자면 설명의 호흡이 길어 흐름을 놓치게 될 수 있으니

구글링을 통해 잘 정리된 요약본을 먼저 읽거나 매장 마지막에 있는 요약을 먼저 읽는것을 추천하고 싶다.

<!-- (오히려 스포일러가 된다고 생각하면 이 책이 인도하는대로 순서에 따라 읽으면 된다.) -->

왜냐하면 이 책의 진입장벽은 책의 분량과 방대하고 깊이있는 설명이 가장 큰 요인 중 하나라고 생각하기 때문이다.

또한 스프링을 처음 접하거나 초심자에겐 **스프링 입문을 위한 자바 객체지향의 원리와 이해**라는 책을 먼저 읽는 것을 추천하고 싶다.

책 제목 그대로 스프링 입문자들을 위한 책이며 특히 객체지향과 스프링에서 자주 사용되는 디자인 패턴에 초점을 둔 책이다.

간단한 예시 코드를 통해 쉽게 디자인 패턴에 대해 쉽게 이해할 수 있어 토비의 스프링을 읽기전에 읽으면 한결 수월할 것이다.

예를들어 이 책을 읽은 뒤 토비의 스프링을 읽으면 '템플릿 콜백 패턴이 무엇인지는 알겠는데 스프링에서 **왜 쓰이고** **어떤 부분에 쓰일까?**' 라는 궁금증을 해결해줄 것이다.

<!-- 좋은 설계을 워한 필수조건인데 어떤 상황에서 어떤 이점이 있는지 모르고 암기로 해결? ㄴㄴ -->

<!-- 결론적으로 스프링의 현재 모습을 보여주지만  -->


## 5. 글을 마치며

<!-- 어떻게 보면 스프링 개발을 시작하기 전에 이론적으로 공부를 먼저 할 수 있었던건 행운이었다. -->

<!-- 책을 읽으면서 직간접적으로 도움을 받은 글들이다. -->

<!-- 2장 733 페이지에서 스프링은 매우 일관된 방식으로 작성된 프레임워크이기 때문에 한번 원리를 알고 나면 어떤 세로운 기능이라도 어렵지 않게 이해하고 사용할 수 있다. -->

<!-- 사실 안드로이드에 Dagger2(Hilt) 라는 DI 라이브러리가 있는데 부끄럽지만 그냥 남들이 쓰는가보다 하고 썼었다. -->


<!-- 소프트웨어는 변한다. 비지니스적인 요구사항은 끊임없이 변화한다. 하지만 일관된 원칙으로 확장할 수 있다는게 얼마나 아름다운것인지 여러분들도 느껴봤으면 하는 바이다. -->


<!-- 내가 며칠전에 썼던 템플릿 관련 글에서도 말했듯이 ~Template이 나오면 아 템플릿 콜백 패턴이겠구나 스프링의 일관성을 믿어서 가능한 일이었다. -->

[스프링 공식문서](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/overview.html)에서는 스프링을 아래와 같이 정의하고 있다.

> Spring Framework is a Java platform that provides comprehensive infrastructure support for developing Java applications

> Java 애플리케이션 개발을 위한 포괄적인 인프라 지원을 제공하는 Java 플랫폼

나는 솔직히 이 정의가 맘에 들지 않는다. 스프링이 주는 핵심 가치들을 너무 추상적으로 설명하고 있는것 같다.
(스프링의 정의 마저 추상화 해버린것일까..?)

내가 생각하는 스프링은 다음과 같다.

> 미친듯이 **확장에 대해 유연**하고 지구 끝까지 **추상화**를 도입한 **객체지향** 끝판왕 자바 엔터프라이즈용 프레임워크

<!-- 심지어 웹으로부터도 독립적으로 만들기 위해 서블릿 컨텍스트를 따루 두는 노력 두손 두발 다 들게 했다. -->

또한 **어떠한 기술적 환경에서도 스프링 프레임워크를 도입이 가능하도록** 전세계 엔터프라이즈 애플리케이션 시장을 다 먹어버리겠다는 로드 존슨의 야심도 느꼈다.


<!-- 언젠간 내 블로그 글들을 취합해 개발자들에게 도움이 될 수 있는 책을 한 권 쓰고 싶다는 목표가 생겼다. -->

다소 글이 길어졌다. 이 책을 읽는 2달동안 많은 영감과 생각거리를 준 이 책을 소개해주시고 내용을 이해하는데 큰 도움을 주신 스터디장님과 이 책의 저자분인 토비님께 감사함을 표한다.

