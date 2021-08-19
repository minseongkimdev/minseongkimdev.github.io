---
title: "ParameterResolver를 통한 테스트 코드 리팩토링 (feat. JUnit5의 핵심철학)"
layout: post
category: Testing
---

## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 고민의 시작](#1-고민의-시작)
- [1. Static 오브젝트를 모아놓은 유틸 클래스로 분리하기](#1-static-오브젝트를-모아놓은-유틸-클래스로-분리하기)
- [2. Mock 오브젝트를 제공하는 클래스를 상속받아 사용하는 방법](#2-mock-오브젝트를-제공하는-클래스를-상속받아-사용하는-방법)
  - [불필요하게 많은 클래스](#불필요하게-많은-클래스)
  - [다중상속 불가](#다중상속-불가)
  - [리스코프 치환원칙(LSP) 위반](#리스코프-치환원칙lsp-위반)
- [3. 조합을 통한 방법](#3-조합을-통한-방법)
- [4. ParameterResolver](#4-parameterresolver)
- [5. Junit5의 핵심 철학](#5-junit5의-핵심-철학)
- [참고 서적](#참고-서적)
- [출처](#출처)

## 1. 고민의 시작

현재 진행하고 있는 프로젝트에서 동일한 Mock 오브젝트가 여러 테스트 클래스에 사용되는 상황이 발생했다. 그래서 다음과 같은 고민을 해보았다.

> 여러 테스트 클래스에서 반복되는 Mock 오브젝트의 생성을 한 곳으로 모아 리팩토링할 순 없을까?

![](https://user-images.githubusercontent.com/44136364/130061720-e59b8385-6336-4ea8-9363-b5e552aa137f.png)

생각한 방법은 총 4가지고, 각 방식의 장단점을 비교 끝에 네번째 방법을 선택하게 되었다.

1. Static 오브젝트를 모아놓은 유틸 클래스로 분리하기
2. Mock 오브젝트를 제공하는 클래스를 상속받아 사용하는 방법
3. Composite을 활용하는 방법
4. ParameterResolver를 활용하는 방법

다른 방법들 대신 ParameterResolver 방식을 선택하게된 이유를 공유해보고자 한다.

## 1. Static 오브젝트를 모아놓은 유틸 클래스로 분리하기

해당 방법의 아이디어는 다음과 같다.

> Mock 객체를 Static으로 선언해서 이를 보관하는 유틸 클래스로 분리해내면 되지 않을까? 

구현이 비교적 단순하고 직관적인 장점이 있지만

**이 방식의 문제점은 한 테스트 메서드에서 해당 스태틱 Mock 객체를 수정하게 되면, 이 객체를 사용하는 모든 메서드에 상태가 반영된다는 것이다.**

테스트는 기본적으로 **독립적으로 이루어져야 하고** 스태틱 Mock 객체의 상태가 변경되어 **다른 테스트 메서드의 통과/실패 여부에 영향을 주면 안된다.**

위와 같은 테스트의 기본적인 원칙을 위반할 잠재적인 가능성이 있다.

비록 지금 프로젝트가 구현단계이고 테스트 코드가 적을 때는 문제가 없을수도 있지만 프로젝트가 점점 거대해지고 테스트 코드가 많아짐에 따라 문제가 생길 수 있어 이 방법은 적절치 않다.

## 2. Mock 오브젝트를 제공하는 클래스를 상속받아 사용하는 방법

![](https://user-images.githubusercontent.com/44136364/130064215-58d26333-db72-4ec7-864c-4c8d8d5b1147.png)

공통적으로 **Mock 오브젝트를 생성하기 위한 부모 클래스**로 두고 해당 Mock 오브젝트가 필요한 클래스는 이 클래스를 상속받는 자식 클래스로 두는 방식이다.

하지만 상속을 잘 사용하면 굉장히 강력한 무기가 될 수 있지만 잘못 사용하면 오히려 독이 될 수 있기 때문에 신중하게 사용해야 한다.

상속을 통해 Mock 오브젝트를 위한 부모클래스를 뒀을 때 어떤 단점이 있을까?
### 불필요하게 많은 클래스

공통적으로 Mock 오브젝트가 생길때마다 해당 오브젝트를 관리하는 부모 클래스를 만들어야 한다.

클래스가 많이 생긴다는 것은 그 만큼의 많은 유지보수 비용이 발생을 암시한다.

### 다중상속 불가

만약에 한 테스트 클래스에서 공통적으로 쓰이는 픽스쳐가 여러 종류면 어떻게 해야할까?

자바에서는 다중상속이 불가능하기 때문에 픽스쳐의 타입에 따라 클래스를 분리해서 상속받을 수 없다.

따라서 한 부모 클래스에 여러 픽스쳐들을 둬야하고 그 결과 클래스 자체가 불필요하게 비대해질 수 있다.

또한 객체지향 관점에서 다른 타입의 픽스쳐들을 관리하게되면 단일 책임 원칙에 어긋나게 된다.

(이는 다음에 설명할 조합을 통한 방법과 관련이 있다.)

### 리스코프 치환원칙(LSP) 위반

객체지향 원칙중 하나인 리스코프 치환원칙은 아래와 같다.

> “객체는 프로그램의 정확성을 깨뜨리지 않으면서 하위타입의 인스턴스로 치환할 수 있어야 한다”

좀 더 쉽게 설명하면 아래와 같다.

> “자식클래스는 부모클래스의 역할을 완벽히 수행할 수 있어야한다”

테스트를 수행하는 자식 클래스가 테스트 클래스들에 픽스쳐를 제공하는 부모 클래스의 역할을 수행할 수 있다고 볼 수 있을까?

리스코프 치환원칙에 위반되어 객체지향 관점에서 좋은 방법이라 보기 어렵다.
## 3. 조합을 통한 방법

위에서 설명했듯이 자바에서는 다중상속이 불가능하다.

이에 대한 해결책으로 공통적으로 사용되는 여러개의 픽스쳐가 있다면 

각각 픽스쳐를 제공하는 클래스들을 만들어 테스트 클래스에서 조합을 통해 

필요한 픽스쳐를 제공받을 수 있다.

하지만 테스트 클래스에서 직접 의존성을 맺어 필요한 픽스쳐를 제공받아야 한다.

직접 의존성을 맺게되면 어떤 문제가 있을까?

(여기서 문제점 ㄱㄱ)

잠시 멈춰서 생각해보면 스프링에서 이를 어떻게 슬기롭게 해결했는지 기억해보자.

스프링에서 IoC 컨테이너를 통해 필요한 곳에 빈들을 주입해줬던 사실이 기억나는가?

Junit5에서도 DI를 활용하여 테스트 메서드의 파라미터에 필요한 오브젝트를 주입해주는 기능을 제공한다.
## 4. ParameterResolver

JUnit5에서 ParameterResolver 라는 인터페이스를 제공한다.

이 인터페이스를 구현한 클래스에서 **테스트 메서드에 주입하고 싶은 오브젝트의 의존성을 맺어 주입해준다.**

ParameterResolver의 내부를 살펴보면 단 2개의 메서드만 정의되어 있다.

~~~java
@API(status = STABLE, since = "5.0")
public interface ParameterResolver extends Extension {

	boolean supportsParameter(ParameterContext parameterContext, ExtensionContext extensionContext)
			throws ParameterResolutionException;

	Object resolveParameter(ParameterContext parameterContext, ExtensionContext extensionContext)
			throws ParameterResolutionException;
}

~~~

- supportsParameter() : 파라미터의 타입이 지원가능하면 true를 리턴한다.
- resolveParamater() : 실제로 테스트 메서드에 주입할 오브젝트의 인스턴스를 생성하여 리턴한다.


다음은 진행하고 있는 프로젝트에 실제로 적용된 단위 테스트 코드이다.

UserDTOFixtureProvider에 UserDTO 픽스쳐를 생성하고

정상적으로 회원가입 할 수 있는 UserDTO 픽스쳐를 주입받아 테스트를 수행할 수 있다.

~~~java
@ExtendWith({SpringExtension.class, UserDTOFixtureProvider.class})
@SpringBootTest
class UserServiceTest {

  ...

  @Test
  @DisplayName("이메일, 비밀번호, 이름, 휴대폰번호, 주소, 유저타입이 입력된 경우 회원가입 성공")
  public void signUpTestSuccess(UserDTO user) {

    String originalPassword = user.getPassword();

    boolean isSuccess = userService.signUp(user);

    assertEquals(
        HashingUtil.sha256Hashing(originalPassword),
        userMapper.selectUserByEmail(user.getEmail()).getPassword()
    );

    assertTrue(isSuccess);
  }

  ...
}
~~~

ParamterResolver는 JUnit5를 개발한 JUit-Team이 발표한 **'기능보다는 확장성'** 원칙에 부합하는 기능이다.

위에서 확인했듯이 테스트 클래스에 @ExtendWith를 선언하는 것만으로 손쉽게 테스트 클래스 내의 메서드에 필요한 오브젝트를 주입받을 수 있었다.

~~~java
@ExtendWith(UserDTOFixtureProvider.class)
~~~

그리고 ExtendWith 어노테이션 내부를 살펴보면 
value 타입이 배열이고, Extension을 구현한 타입이면 모두 담을 수 있다.

~~~java
Class<? extends Extension>[] value();
~~~

다른 타입의 픽스쳐가 필요하다면 얼마든지 ParamterResolver를 구현한 클래스를 @ExtendWith로 지정해주면 손쉽게 확장할 수 있다.

## 5. Junit5의 핵심 철학

[Junit5의 공식 레포](https://github.com/junit-team/junit5/wiki/Core-Principles)에서 직접 확인할 수 있다. 


**'기능보다는 확장성'** 외에도 아래와 같은 원칙들이 있다.

> Complementarily, an extension point should be good at one thing

> It should be hard to write tests that behave differently based on how they are run

> Tests should be easy to understand

> Minimize dependencies (especially third-party)

이 원칙을 세운 이유까지 잘 설명되어있고 원문이 길지 않으니 관심있으면 한번 읽어보길 권한다.

한 가지 첨언하고 싶은 것은 Junit5 뿐만 아니라 모든 프로그래밍 언어, 라이브러리, 프레임워크 갑자기 뚝딱 등장한게 아니다.

**각자의 탄생 배경, 설계된 목적, 만든이가 담은 고유한 철학이 존재한다.
**
Java만 해도 CPU 아키텍쳐 마다 다른 코드를 짜야 했던 C언어의 단점을 보안하기 위해
WORA(Write Once Run Anywhere) 철학을 품고 하드웨어에 상관없이 JVM을 통해 동일한 실행환경을 제공한 것처럼 말이다.

이를 알고 사용하는것과 단순히 사용법만 익혀서 사용하는것은 천지 차이라고 생각한다.
<!-- ## 글을 마치며

테스트 코드는 개발자로 하여금 방대한 시스템에 기능을 추가하거나 수정할 때 자신감을 주고 이는 곧 개발 생산성으로 이어진다.

물론 테스트 코드의 작성은 지금 당장의 비용이 들 수 있겠지만, 나중에 발생하는 더 큰 비용의 발생을 막아줄 수 있다.

그리고 테스트 코드의 리팩토링 함으로써 테스트 코드의 가독성이 향상되고, ㅇㅂ -->

<!-- 또한 잘 작성된 테스트 코드 만으로 해당 기능에 대한 명세서와 같은 역할을 할 수도 있다.

이처럼 테스트 코드가 주는 효과는 막대하고 이것의 중요성은 아무리 강조해도 부족하다. -->

<!-- **이처럼 애플리케이션을 구동하는 코드의 리팩토링이 중요한 것처럼 테스트 코드에 대한 리팩토링 또한 중요하다.**

테스트 코드 리팩토링을 통해 유

Junit-Team에서도 **'기능보다는 확장성' 이라는 구호를 외친게 테스트 코드의 리팩토링의 중요성을 강조하기 위해서가 아닐까?** -->

## 참고 서적

* 자바와 JUnit을 활용한 실용주의 단위 테스트

##  출처

- [Junit5 Core Principles](https://github.com/junit-team/junit5/wiki/Core-Principles)
- [Inject Parameters into JUnit Jupiter Unit Tests](https://www.baeldung.com/junit-5-parameters)
