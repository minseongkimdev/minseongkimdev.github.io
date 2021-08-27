---
title: "부동 소수점 테스트"
layout: post
category: Testing
---
<!-- 

최근 자바와 JUnit을 활용한 실용주의 테스트 라는 책을 읽고 있는데 흥미로운 대목이 있었다.

바로 부동 소수점을 테스트 할 때 주의점 이다.

이 부분을 읽으면서 '왜 부동소수점을 테스트할 때 조심해야할까?' 라는 궁금점이 생겼고, 부동소수점의 동작원리와 이 부분을 Java에서는 어떻게 보완하고 있을지(BigDecimal) 조사한 내용을 공유해보고자 한다.

다음은 책에서 발췌한 내용이다.

해당 대목은 부동소수점의 원리와 깊은 연관이 있었다.

Junit을 통해 부동 소수점을 테스트할 때 주의할 점이 있다.

Junit에서는 소수를 isEqual를 통해 비교하면 문제가 발생할 수 있다.

위의 테스트는 통과할가 실패할까?

~~~java
assertThat(2.22 * 3, equalTo(6.66));
~~~

아래와 같이 AssertionError과 함께 기대한 값과 다르며 실패한다.

~~~java
java.lang.AssertionError:
Expected: <6.66>
  but: was <6.659999999999999>
~~~

이 테스트를 통과시키려면 아래과 같이 수정할 수 있다.

~~~java

assertTrue(Math.abs( -0.0005 < (2.22 * 3) - 6.96) && (2.22 * 3) - 6.96) < 0.0005);
~~~

Math.abs를 통해 좀 더 단순하게 표현할 수 있다.
~~~java
assertTrue(Math.abs((2.22 * 3) - 6.96) < 0.0005);
~~~

하지만 가독성 측면에서 그다지 좋은 방법은 아닌듯 하다.
그 대신에 isCloseTo 라는 햄크레스트 매처[^1]를 통해 더 직관적으로 표현할 수 있다.

~~~java
assertThat(2.32 * 3, closeTo(6.96, 0.0005));
~~~

여기서 주의할 점은 범위를 지정할 때 너무 큰 범위를 지정하게 되면 무조건 통과하는 의미가 없는 테스트가 되고 너무 좁게 되면, 실제로 통과해야할 테스트가 깨질 수 있다.

왜 이런 불편함을 감수해야할까?

이유는 부동소수점의 원리에 숨겨져 있다.


## 부동소수점

부동은 아래와 같은 뜻의 부동이 아니다.
> 그는 부동의 자세로 움직이지 않고 12시간 동안 개발을 했다

떠다닌다라는 의미의 부동인데
(부동이라는 단어가 개인적으로 중의적의미가 있어 좋지 않은 번역이라고 생각한다.)


그럼 비트코인과 같은 정확한 값을 표현하기 위해


다음은 업비트의 화면이다. 소수점까지 가격이 표시되고 매수/매도가 가능하다.


여기서 가장 비싼 비트코인의 경우를 살펴보자 0.000001 의 오차가 발생하면 어떻게 될까?
내가 매도를 했는데 얼마가 덜 사지고, 매수를 했는데 얼마가 덜 팔아지면 어떻게 될까?


## BigDecimal

이를 위한 해결책은 BigDicimal를 통해 오차없이 실수를 표현할 수 있다.

어떻게 가능할까?

자바에서는 이런 부동소수점의 단점을 보완하기 위한 
BigDecimal 라는 클래스를 제공한다.

어떤식으로 부동소수점의 단점을 보완하고 있을까?

아예 실수형을 사용하지 않고
정수형을 통해 가수 + 지수부를 다 표현하고, 소수점의 위치를 따로 보관하고 있다.

그리고 아예 새로운 이뮤터블한 객체를 만들어서 스레드 세이프하게 구현이 되어 있다.

(두나무 채용공고를 보니 글 작성 시점 기준으로 백엔드가 루비로 구현되어있는거 같은데 어쨌든.)


~~~java

public class BigDecimal extends Number implements Comparable<BigDecimal> {
    /**
     * The unscaled value of this BigDecimal, as returned by {@link
     * #unscaledValue}.
     *
     * @serial
     * @see #unscaledValue
     */
    private final BigInteger intVal;

    /**
     * The scale of this BigDecimal, as returned by {@link #scale}.
     *
     * @serial
     * @see #scale
     */
    private final int scale;  // Note: this may have any value, so
                              // calculations must be done in longs

~~~


## Immutable


## 글을 마치며

자바의 기초적인 부분에서 문제가 발생할 수 있으니 기초를 항상 탄탄히 해야겠다는 생각이 든다.

자바의 아주 기본이라고 할 수 있는 실수형 타입에 대한 


## 각주

[^1]: 햄크레스트 매처 : 


## 참고서적

- 자바와 JUnit을 활용한 실용주의 테스트 -->