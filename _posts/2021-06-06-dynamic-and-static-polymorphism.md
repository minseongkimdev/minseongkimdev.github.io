---
title: "[작성중]JVM의 내부원리로 이해하는 동적, 정적 다형성"
layout: post
category: Java
---



두가지 유형의 다형성이 존재한다.
- 정적 다형성
- 동적 다형성

정적 다형성은 컴파일 타임에 성립되고 ,동적 다형성은 런타임에 인식된다.

자바에서 대표적으로 Override와 Overloading을 통해 다형성을 구현할 수 있다.

각각의 키워드는 어떤 다형성에 속할까?
다음은 @Override 어노테이션의 선언부이다.

~~~java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
~~~
Retention이 SOURCE임을 확인할 수 있고, 이는 어노테이션 정보는 런타임에서 사용할 수 없다.
따라서 Overriding은 정적 다형성에 해당함을 알 수 있다.

동적 다형성은 컴파일러가 아닌 JVM이 어떤 메서드가 실행될지 결정 하는 것이다.

Overloading은 