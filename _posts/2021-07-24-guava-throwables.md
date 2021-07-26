---
title: "Guava Throwables 알아보기"
layout: post
category: Java
---


## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. 예외처리 전략](#2-예외처리-전략)
- [3. Propagation](#3-propagation)
  - [내부 구현](#내부-구현)
- [4. Cause Chaning](#4-cause-chaning)
  - [내부 구현](#내부-구현-1)
- [5. 글을 마치며](#5-글을-마치며)
- [참고서적](#참고서적)
- [출처](#출처)

## 1. 들어가면서

구글에서 자바의 다양한 확장 기능을 제공하는 [Guava](https://github.com/google/guava) 라는 라이브러리에는 Throwable과 관련된 유틸 기능을 제공하는 [Throwables](https://guava.dev/releases/23.0/api/docs/com/google/common/base/Throwables.html) 클래스가 있다.

과연 이 유틸 클래스는 **어떤 상황에서 필요**하고 **어떤 기능을 수행**할까?

## 2. 예외처리 전략


스프링에서 예외를 처리하는 전략에는 여러가지가 있다.

> 1. 예외 복구 : 예외 상황을 파악하고 문제를 해결하여 정상 상태로 돌려놓는 방법
> 2. 예외 회피 : 자신을 호출한 쪽으로 예외를 던져버리는 방법
> 3. 예외 전환 : 좀 더 의미있는 예외로 변경하거나, 런타임 예외로 포장하는 방법

(이 전략들 외에 catch를 통해 예외를 잡고 아무런 조치를 취하지 않는것은 대표적인 안티패턴이니 주의하도록 하자.)

특히 3번에서 런타임 예외로 포장하는 이유에 대해 좀 더 알아보자.

체크 예외를 다룰 때 불필요한 catch/throws를 난발하게 될 수 있는 단점을 보완하기 위한 이유도 있지만 서버환경에서 런타임 예외로 포장하는게 더 유리하기 때문이다.

서버 환경에서는 조금 특수하게도 수많은 사용자가 동시에 요청을 보내고, 각 요청은 독립적인 작업으로 취급해야 한다. 왜냐하면 하나의 요청을 처리하는 중에 예외가 발생했다고 해서 다른 요청까지 영향을 끼쳐서는 안되기 때문이다.

즉 문제가 생겼을 때 해당 요청의 작업을 취소하고 런타임 예외를 통해 개발자에게 통보해주는게 서버환경에서는 더 유리하다.


아래와 같이 특히 예외 복구와 예외 전환에 필요한 유용한 기능들을 Guava의 Throwables에서 제공하고 있다. 아래의 기능들은 특히 예외 전환에서 빛을 발하게 된다.

- Propagation
- Cause Chaining

Propagation 부터 알아보도록 하자.

## 3. Propagation

아래와 같이 Throwable 클래스를 통해 예외를 처리하는 상황을 가정해보자.

위에서 설명했듯이 서버환경에서는 RuntimeException으로 전환하는 것이 유리하다.

다음은 SQLException, IOException 같이 RuntimeException의 특정 하위 클래스인 경우를 제외한 나머지 예외(체크 예외 포함)들을 IllegalStateException로 전환하는 예이다.

하지만 조건이 많아지면 if-else 문이 길어져 가독성이 떨어진다.

~~~java
...
catch (ExecutionException e) {
  Throwable cause = e.getCause();

  if (cause instanceof SQLException) {

    throw (SQLException) cause;

  } else if (cause instanceof RuntimeException) {

    throw (RuntimeException) cause;

  } else if (cause instanceof Error) {

    throw (Error) cause;

  } else {

    throw new IllegalStateException(e);

  }
}
~~~

그래서 Guava의 Throwables에서 위와 같은 기능을 아래와 같이 깔끔하게 작성할 수 있는 Propatation(전파) 기능을 제공한다.

쉽게 설명하면, 간결한 코드로 상위 타입의 예외로 캐스팅 해서 throws하는 것을 돕는 기능이다.
~~~java
catch (UncheckedExecutionException e) {

  Throwables.propagateIfPossible(e.getCause(), RuntimeException);
  throw new IllegalStateException(e);

}
~~~

### 내부 구현

propagateIfPossible 메서드는 내부적으로 throwIfInstanceOf()와 throwIfUnchecked() 메서드를 호출한다.

throwIfInstanceOf() 에서 예외가 매개변수로 넘긴 declaredType의 하위 타입인지 검사한 후 맞다면 declaredType로 캐스팅 해서 throw 한다.

~~~java
  public static <X extends Throwable> void throwIfInstanceOf(
      Throwable throwable, Class<X> declaredType) throws X {
    checkNotNull(throwable);
    if (declaredType.isInstance(throwable)) {
      throw declaredType.cast(throwable);
    }
  }

~~~

throwIfUnchecked() 에서는 RuntimeException 혹은 Error의 하위 클래스인 경우 RuntimeException과 Error로 캐스팅 해서 throws 한다.

~~~java
  public static void throwIfUnchecked(Throwable throwable) {
    checkNotNull(throwable);
    if (throwable instanceof RuntimeException) {
      throw (RuntimeException) throwable;
    }
    if (throwable instanceof Error) {
      throw (Error) throwable;
    }
  }
~~~

## 4. Cause Chaning

예외가 RuntimeException으로 전환된 경우, 원래 예외를 발생시킨 근본적인 예외를 알아야하거나 어떤 예외들로 체이닝 되었는지 파악해야 하는 경우가 있다.


Gauva는 또한 throw된 예외을 유발한 근본적인 예와 및 그 예외에 체이닝 된 예외들을 쉽게 확인할 수 있는 기능을 제공한다.

getRootCause() 메서드를 통해 가장 안쪽(가장 근본이 되는 예외)의 예외를 얻을 수 있고

~~~java
Throwable getRootCause(Throwable)
~~~

List 컬렉션에 담긴 체이닝 된 여러 예외들을 얻을 수도 있다.

~~~java
List<Throwable> getCausalChain(Throwable)
~~~


### 내부 구현

getRootCause()와 getCausalChain()의 내부구현은 매우 단순하고 직관적이다.

getRootCause(), getCausalChain() 모두 내부적으로 투 포인터 알고리즘을 사용하며 내부 구조가 거의 흡사하다.

유일한 차이점은 getRootCause()는 가장 근본이 되는 예외를 리턴하고, getCausalChain()는 체인을 순회하면서 List 컬렉션에 예외를 담아 리턴한다는 점이다. 아래의 코드에서 확인해보길 바란다.

~~~java
public static Throwable getRootCause(Throwable throwable) {

    Throwable slowPointer = throwable;
    boolean advanceSlowPointer = false;

    Throwable cause;

    while ((cause = throwable.getCause()) != null) {

      throwable = cause;

      if (throwable == slowPointer) {

        throw new IllegalArgumentException("Loop in causal chain detected.", throwable);
      }

      if (advanceSlowPointer) {

        slowPointer = slowPointer.getCause();
      }

      advanceSlowPointer = !advanceSlowPointer; // only advance every other iteration
    }

    return throwable;
  }
~~~


~~~java
public static List<Throwable> getCausalChain(Throwable throwable) {

  checkNotNull(throwable);
  List<Throwable> causes = new ArrayList<>(4);
  causes.add(throwable);

  Throwable slowPointer = throwable;
  boolean advanceSlowPointer = false;

  Throwable cause;
  while ((cause = throwable.getCause()) != null) {
    throwable = cause;
    causes.add(throwable);

    if (throwable == slowPointer) {

      throw new IllegalArgumentException("Loop in causal chain detected.", throwable);
    }

    if (advanceSlowPointer) {

      slowPointer = slowPointer.getCause();
    }

      advanceSlowPointer = !advanceSlowPointer; // only advance every other iteration
    }

    return Collections.unmodifiableList(causes);
  }
~~~

투 포인터에 대한 자세한 설명은 [이 글](https://algodaily.com/lessons/using-the-two-pointer-technique)에 알기 쉽게 설명되었으니 참고하길 바란다.

## 5. 글을 마치며

위와 같이 무언가 굉장한 마법이 숨겨져 있는것 같은 클래스나 메서드도 위와 같이 코드를 직접 확인해보면 매우 간단하거나 직관적인 경우가 있다.

하지만 직접 내부 구현 코드를 확인해보는 것이 익숙하지 않으면 막연히 두렵거나 확인하는데 시간이 오래 걸릴 수 있다.

그러나 이 과정이 익숙해지면 처음 보는 클래스, 메서드, 어노테이션 등의 동작 원리를 정확히 파악할 수 있으니 적극 권장한다.


## 참고서적

토비의 스프링 3.1 1권 4장 예외

## 출처

- [Introduction to Guava Throwables](https://www.baeldung.com/guava-throwables)