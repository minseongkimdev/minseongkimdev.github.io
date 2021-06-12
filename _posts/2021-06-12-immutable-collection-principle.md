---
title: "Guava ImmutableList 내부 원리 분석"
layout: post
category: Java
---



## 0. 글의 순서


## 1. 들어가면서

개발자들이 흔히 하는 실수중 하나는,
자바 Collection은 기본적으로 Immutable하지 않은 성질을 고려하지 않고 개발하는 것이다.

자바 8(이하) 에서는 Immutable한 Collection을 기본적으로 제공하지 않아, 구글의 Guava[^1]와 같은 서브파티 라이브러리 안에 구현된 Immutable Collection을 이용해야 한다.

구글 Guava ImmutableList에서 아래와 같이
왜 쓸까?
신뢰할 수없는 라이브러리에서 안전하게 사용할 수 있습니다.
스레드 안전성 : 경쟁 조건의 위험없이 많은 스레드에서 사용할 수 있습니다.
돌연변이를 지원할 필요가 없으며 이러한 가정으로 시간과 공간을 절약 할 수 있습니다. 모든 불변 컬렉션 구현은 가변 형제보다 메모리 효율성이 높습니다. ( 분석 )
고정 된 상태로 유지 될 것으로 예상하면서 상수로 사용할 수 있습니다.

나는 구글의 Gauva에서는 ImmutableCollection을 어떻게 구현했는지 내부 동작원리가 궁금해졌고, 이를 공부하고 분석한 내용을 공유해보고자 한다.

JDK는 Collections.unmodifiableXXX메소드를 제공 하지만 우리의 의견으로는

다루기 힘들고 장황한; 방어적인 사본을 만들고 싶은 모든 곳에서 사용하는 것이 불쾌합니다.
안전하지 않음 : 반환 된 컬렉션은 원본 컬렉션에 대한 참조를 보유한 사람이없는 경우에만 변경 불가능합니다.
비효율적 : 데이터 구조에는 동시 수정 검사, 해시 테이블의 추가 공간 등을 포함하여 변경 가능한 컬렉션의 모든 오버 헤드가 있습니다.
컬렉션을 수정할 것으로 예상하지 않거나 컬렉션이 일정하게 유지 될 것으로 예상하는 경우 방어 적으로 변경 불가능한 컬렉션에 복사하는 것이 좋습니다.

중요 : 각 Guava 불변 컬렉션 구현 은 null 값을 거부합니다. Google의 내부 코드 기반에 대한 철저한 연구를 수행하여 null요소가 약 5 %의 시간 동안 컬렉션에 허용되고 나머지 95 %의 경우는 null로 빠르게 실패하는 것이 가장 좋습니다. null 값을 사용해야하는 경우 null Collections.unmodifiableList을 허용하는 컬렉션 구현에서 및 해당 친구를 사용하는 것이 좋습니다. 더 자세한 제안은 여기 에서 찾을 수 있습니다 .


Immutable Collection중 ImmutableList를 통해 알아보자.


## 2. ImmutableList 생성

다음은 ImmutableList를 생성하는 예시이다.

~~~java

public static final ImmutableList<Color> GOOGLE_COLORS
	 = new ImmutableList.Builder<Color>()
	 .addAll(WEBSAFE_COLORS)
	 .add(new Color(0, 191, 255))
	 .build();

~~~


ImmutableList내부의 Builder를 

ImmutableCollection을 상속 받은 ImmutableList를 예를 들어 설명해보면,

다음은 ImmutableList의 add 메서드이다.
add메서드를 통해 원소를 추가하려고 하면,
무조건 UnsupprtedOperationException을 발생시킨다.

아래의 두가지 방법을 제공한다.
- of
- Builder


### of()

of()에서는 생성할 때 원소의 갯수에 따라 다른 메서드가 존재한다.
2개 이상의 원소를 추가할 때, 공통적으로 construct 메서드를 통해 생성한다.
~~~java

	public static <E> ImmutableList<E> of(E e1, E e2) {
		return construct(e1, e2);
	}

~~~

construct()에서는 asImmutableList를 호출하고 있고, checkElementNotNull()을 통해 원소가 null인지 체크한다.
(Guava의 ImmutableCollection에서 null 원소를 허용하지 않는다.)


~~~java

	/** Views the array as an immutable list. Checks for nulls; does not copy. */
	private static <E> ImmutableList<E> construct(Object... elements) {
		return asImmutableList(checkElementsNotNull(elements));
	}

~~~

asImmutableList에서는 원소의 갯수에 따라 switch문을 따라 분기처리를 해놓았다.
2개 이상일 때를 살펴보면 RegularImmutableList 객체를 리턴하고 있는걸 확인할 수 있다.

~~~java

	/**
	 * Views the array as an immutable list. Copies if the specified range does not cover the complete
	 * array. Does not check for nulls.
	 */
	static <E> ImmutableList<E> asImmutableList(@Nullable Object[] elements, int length) {
		switch (length) {
			case 0:
				return of();
			case 1:
				/*
				 * requireNonNull is safe because the callers promise to put non-null objects in the first
				 * `length` array elements.
				 */
				@SuppressWarnings("unchecked") // our callers put only E instances into the array
						E onlyElement = (E) requireNonNull(elements[0]);
				return of(onlyElement);
			default:
				/*
				 * The suppression is safe because the callers promise to put non-null objects in the first
				 * `length` array elements.
				 */
				@SuppressWarnings("nullness")
				Object[] elementsWithoutTrailingNulls =
						length < elements.length ? Arrays.copyOf(elements, length) : elements;
				return new RegularImmutableList<E>(elementsWithoutTrailingNulls);
		}
	}

~~~

RegularImmutableList는 ImmutableList를 상속받았고, is클래스 내부에 final하게 선언된 배열(Array) 속성을 가지고 있다.

~~~java

  @VisibleForTesting final transient Object[] array;


  @Override
  public int size() {
    return array.length;
  }
~~~


### Builder
of를 통해 생성하는 것보다 더 많은 기능을 제공한다.
또한 빌더패턴을 활용하여 of보다 코드 가독성이 좋다.


Builder 클래스 내부에 contents 속성을 가지고 있다.

Builder의 build() 통해 ImmutableList를 생성하기 전까지는 Builder의 add()를 통해 원소를 추가 할 수 있으므로 contents가 final로 선언되지 않았다.

~~~java

	public static final class Builder<E> extends ImmutableCollection.Builder<E> 

		// The  first `size` elements are non-null.
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

그리고 build() 메서드에서

~~~java

@Override
  public ImmutableList<E> build() {
	forceCopy = true;
	return asImmutableList(contents, size);
}

~~~

## 3. Immutable Collection iterator

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


~~~java

@Deprecated
@Override
@DoNotCall("Always throws UnsupportedOperationException")
public final void add(int index, E element) {
	throw new UnsupportedOperationException();
}
~~~


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

[^1]: Guava : 구글에서 제공하는 자바의 일부 기능을 보완하기 위한 라이브러리.



