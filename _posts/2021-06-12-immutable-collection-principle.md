---
title: "Guava ImmutableList 내부 원리 분석"
layout: post
category: Java
---



## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. ImmutableList 내부 코드 살펴보기](#2-immutablelist-내부-코드-살펴보기)
  - [of()](#of)
  - [Builder](#builder)
  - [RegularImmutableList](#regularimmutablelist)
- [3. ImmutableList iterator](#3-immutablelist-iterator)
- [글을 마치며](#글을-마치며)
- [출처](#출처)
- [주석](#주석)
## 1. 들어가면서

개발자들이 흔히 하는 실수중 하나는,
자바 Collection은 기본적으로 Immutable하지 않은 성질을 고려하지 않고 개발할 때 발생한다.

자바 8(이하) 에서는 Immutable한 Collection을 기본적으로 제공하지 않아, 구글의 Guava[^1]와 같은 서브파티 라이브러리 안에 구현된 `ImmutableCollection`을 이용해야 한다.

`ImmutableCollection`중 가장 많이 쓰이는 컬렉션중 하나인 `ImmutableList`에서 Immutable한 성질을 구현하기 위해 내부적으로 어떤 장치들이 있는지 궁금해졌고, 이를 공부하고 분석한 내용을 공유해보고자 한다.

[Guava ImmutableCollection에 대해 궁금하다면 Github Wiki에 잘 정리되어 있으니 참고해보길 바란다.](https://github.com/google/guava/wiki/ImmutableCollectionsExplained)



## 2. ImmutableList 내부 코드 살펴보기

ImmutableList는 한번 초기화 된 후에 add, set, remove 등과 같이 원소를 추가, 수정, 삭제가 불가능하다.

만약 이를 시도하면 아래 코드에서 확인할 수 있다 싶이 항상 UnsupportedOperationException이 발생한다.

~~~java
@Deprecated
@Override
@DoNotCall("Always throws UnsupportedOperationException")
public final void add(int index, E element) {
  throw new UnsupportedOperationException();
}

@CanIgnoreReturnValue
@Deprecated
@Override
@DoNotCall("Always throws UnsupportedOperationException")
public final E set(int index, E element) {
  throw new UnsupportedOperationException();
}

@CanIgnoreReturnValue
@Deprecated
@Override
@DoNotCall("Always throws UnsupportedOperationException")
public final E remove(int index) {
  throw new UnsupportedOperationException();
}
~~~


ImmutableList를 생성할 때 아래의 두가지 방법을 제공한다.

- of()
- Builder


다음은 ImmutableList를 생성하는 예시이다.

~~~java
public static final ImmutableList<Color> GOOGLE_COLORS = new ImmutableList
  Builder<Color>()
	 .addAll(WEBSAFE_COLORS)
	 .add(new Color(0, 191, 255))
	 .build();

   // of() 예시 추가하기
~~~
### of()

of()에서는 생성할 때 원소의 갯수에 따라 다른 메서드가 존재한다.
2개 이상의 원소를 추가할 때, 공통적으로 construct 메서드를 통해 생성한다.
~~~java

public static <E> ImmutableList<E> of(E e1, E e2) {
	return construct(e1, e2);
}

~~~

흥미로운 사실은 of()는 매개변수의 갯수에 따라 아래와 같이 overrloading 되어 있다.

![](https://blog.kakaocdn.net/dn/5fxIi/btq65Tnng3H/fIA9HanAJwo2jiISVNhQsK/img.png)

그리고 원소의 갯수가 13개 이상일 때, 매개변수로 12개까지는 개별적으로 받지만 그 이상은 varargs 형태로 받게된다.

~~~java
@SafeVarargs
public static <E> ImmutableList<E> of(
  E e1, E e2, E e3, E e4, E e5, E e6, E e7, E e8, E e9, E e10, E e11, E e12, E... others)
~~~

여기서 of()를 overrloading 할필요 없이, of()를 하나만 선언하고 매개변수를 varags형태로 정의하면 간편하지 않을까 라고 생각할 수 있다. 하지만 주석을 통해 아래와 같이 설명하고 있다. 

> These go up to eleven. After that, you just get the varargs form, and
> whatever warnings might come along with it. :(

> @SafeVarargs For Eclipse. For internal javac we have disabled this > pointless type of warning.


해석해보면, varags로 인한 불필요한 warning을 피하기 위해 원소의 갯수가 많아 varags가 꼭 필요한 상황에서만 사용하기 위해 overrloading 한 것으로 추측해 볼 수 있다.

construct()에서는 checkElementNotNull()을 통해 원소가 null인지 체크한 뒤 asImmutableList를 호출한다.

(Guava의 ImmutableCollection에선 null 원소를 허용하지 않는다.)

~~~java
private static <E> ImmutableList<E> construct(Object... elements) {
	return asImmutableList(checkElementsNotNull(elements));
}
~~~

asImmutableList에서는 원소의 갯수에 따라 switch문을 따라 분기처리를 해놓았다.
2개 이상일 때를 살펴보면 RegularImmutableList 객체를 리턴하고 있는걸 확인할 수 있다.

~~~java

static <E> ImmutableList<E> asImmutableList(@Nullable Object[] elements, int length) {
	switch (length) {
		case 0:
			return of();

		case 1:
			@SuppressWarnings("unchecked")
		  E onlyElement = (E) requireNonNull(elements[0]);
			return of(onlyElement);

		default:
			@SuppressWarnings("nullness")
			Object[] elementsWithoutTrailingNulls =
				length < elements.length ? Arrays.copyOf(elements, length) : elements;
				return new RegularImmutableList<E>(elementsWithoutTrailingNulls);
		}
}

~~~

RegularImmutableList은 Builder를 살펴본 뒤 알아보도록 하자.
### Builder

Builder 클래스 내부에 contents 속성을 가지고 있다.

Builder의 build() 통해 ImmutableList를 생성하기 전까지는 Builder의 add()를 통해 원소를 추가 할 수 있으므로 contents가 final로 선언되지 않았다.

(Builder를 통해 ImmutableList를 생성하기 전에 원소들을 임시로 보관하는 배열이다.)

~~~java
public static final class Builder<E> extends ImmutableCollection.Builder<E> 

@VisibleForTesting @Nullable Object[] contents;
~~~

Builder 클래스를 사용하여 add함수는 아래와 같다.

checkNotNull()을 통해 추가하려는 원소가 null임을 검사하고, null이면 NullPointerException을 발생시킨다.

~~~java
@CanIgnoreReturnValue
@Override
public Builder<E> add(E element) {
  checkNotNull(element);
	getReadyToExpandTo(size + 1);
	contents[size++] = element;
	return this;
}
~~~

아래의 getReadyToExpandTo()를 통해, copyOf()메서드를 활용하여, Array의 길이를 minCapacity만큼 증가시킨다.

~~~java
private void getReadyToExpandTo(int minCapacity) {
	if (contents.length < minCapacity) {
		this.contents = Arrays.copyOf(contents, expandedCapacity(contents.length, minCapacity));
		forceCopy = false;
		} else if (forceCopy) {
		contents = Arrays.copyOf(contents, contents.length);
		forceCopy = false;
	}
}
~~~

그 다음 contents Array의 마지막 인덱스에 원소를 추가한다.

그리고 build() 메서드에서 asImmutable()를 호출한다.
of()를 설명할 때, 이미 asImmutable()를 확인했기 때문에 이에 대한 설명은 생략하도록 한다.

~~~java
@Override
public ImmutableList<E> build() {
	forceCopy = true;
	return asImmutableList(contents, size);
}
~~~

### RegularImmutableList

ImmutableList abstract 클래스를 확장한 클래스이다.
~~~java
class RegularImmutableList<E> extends ImmutableList<E>
~~~

이 클래스에서 아래와 같이 final로 선언된 배열을 통해 Builder혹은 of()를 통해 추가한 원소들을 관리한다.

~~~java
@VisibleForTesting final transient Object[] array;
~~~

그리고 기본적인 연산을 위한 메서드들이 정의되어 있다.

~~~java
...
@Override
public int size() {
  return array.length;
}

@Override
@SuppressWarnings("unchecked")
public E get(int index) {
  return (E) array[index];
}
...
~~~

결국 핵심은 final로 선언한 배열을 통해 원소륻릉 관리하고,
add, set, remove과 같이 원소를 추가, 수정, 삭제 할 수 없다는 것이다.

## 3. ImmutableList iterator

ImmutableList에는 아래와 같이 iterator가 선언되어 있다. 


~~~java

@Override
public UnmodifiableListIterator<E> listIterator(int index) {
	return new AbstractIndexedListIterator<E>(size(), index) {
	  @Override
		protected E get(int index) {
			return ImmutableList.this.get(index);
		}
	};
}
~~~

AbstractionIntexedListIterator는 UnmodifiableListIterator를 상속받고 있다.

이름 그대로 수정 불가능한 Iterator임을 유추할 수 있다.

~~~java
abstract class AbstractIndexedListIterator<E> extends UnmodifiableListIterator<E> 
~~~

그리고 UnmodifiableListItersator에서 add(), set()을 아래와 같이 항상 UnsupportedOperationException을 던지도록 구현되어 있다.

~~~java

@Deprecated
@Override
@DoNotCall("Always throws UnsupportedOperationException")
public final void add(E e) {
  throw new UnsupportedOperationException();
}

@Deprecated
@Override
@DoNotCall("Always throws UnsupportedOperationException")
public final void set(E e) {
  throw new UnsupportedOperationException();
}
~~~

요약하면, iterator를 통해 순회는 가능하지만 add나 set와 같이 추가와 수정은 불가능 하다.
(항상 UnsupportedOperationException 예외 발생)

## 글을 마치며

ImmutableList를 확장한 RegularImmutableList에서 내부적으로 final하게 선언된 배열(Array)을 통해 원소를 관리하고 있었다.

원소를 추가, 삭제가 불가하니 크기가 고정된 배열을 통해 관리함을 쉽게 유추할 수 있고 꽤나 합리적인 선택이라고 할 수 있다.

또다른 흥미로운 점은 Kotlin 진영에서는 기본적으로 List를 생성하고 나서 원소를 수정할 수 없다. (read-only)

아예 Mutable한 List는 MutableList 클래스로 따로 정의하고 있다.

이는 Mutable한 List의 성질을 고려하지 못하고 개발했을 때의 실수를 줄여주기 위한
Kotlin 언어 개발자들의 작은 배려이지 않았을까?

## 출처

- [Guava Github Wiki](https://github.com/google/guava/wiki/ImmutableCollectionsExplained)

## 주석

[^1]: Guava : 구글에서 제공하는 자바의 일부 기능을 보완하기 위한 유틸리티 라이브러리.
Immutable  Collection, Graph 라이브러리가 포함되어 있고 I/O, Hashing, Caching, Primitives, String 등을 보완한다.



