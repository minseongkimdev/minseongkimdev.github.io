---
title: "[작성중]내가 해석한 OOP"
layout: post
category: CS
---


## 0. 글의 순서

## 1. 들어가면서


OOP에 대한 여러 정의가 있지만 구글링 했을 때 가장 흔히 볼 수 있는 정의는 다음과 같다.

> 컴퓨터 프로그램을 명령어의 목록에서 보는 시각에서 벗어나
> 여러개의 독립된 단위, 즉 객체들의 모임으로 파악하고자 하는 것.



하지만 내가 OOP에 대해 공부해보고 내린 정의는 아래와 같다.

> 프로그래밍에서 **현실세계**를 최대한 반영하기 위한 일련의 노력

이 글을 통해 왜 그렇게 생각하게 되었는지 나의 생각을 공유해보고자 한다.




## 2. 객체지향의 특성

특히 객체지향의 특성을 공부하면서, OOP가 현실세계를 반영하기 위한 노력이라는 확신이 들었다.

OOP에는 다음과 같은 특성이 있다.

- Encapsulation - 캡슐화

- Inheritance - 상속

- Abstraction - 추상화 

- Polymorphism - 다형성

차례대로 알아보도록 하자.

### Encapsulation - 캡슐화

캡슐화란 필수적인 정보만 공개하고, 불필요한 정보를 숨기는 개념이다.

커피는 개발자와 뗄래야 뗄 수 없는 존재이니 커피머신을 예로 들어보자.

우리는 커피머신을 사용할 때, 버튼 클릭 한번으로 커피를 탈 수 있다.

커피 종류별로 물에 커피원두를 몇 그램 타야하는 등의 내부 메커니즘은 전혀 몰라도 된다.

왜 커피머신은 이러한 내부 메커니즘을 사용자로부터 숨겼을까?

거꾸로 생각해보면 쉽다. 사용자가 커피머신의 내부 메커니즘을 쉽게 확인할 수 있고 변경할 수 있으면 어떤 문제점이 발생할까?

자바 소스코드를 통해 확인해보자.

다음은 CoffeeMachine 클래스, CoffeeSelection Enum 클래스, CoffeeBean 클래스이다.


(짧고 쉬운 코드라 자세한 설명은 생략한다.)

~~~java

public class CoffeeMachine {

	private Map<CoffeeSelection, CoffeeBean> beans;

  ...

}

public enum CoffeeSelection {
	FILTER_COFFEE, ESPRESSO, CAPPUCCINO;
}

public class CoffeeBean {

	private double quantity;

public CoffeeBean(String name, double quantity) {
		this.quantity = quantity;
	}
...
// getter, setter 생략
}


~~~

만약에 CoffeeMachine 클래스의 beans 속성의 접근제어자가 private이 아니라 public이었다면 어떤 문제가 발생할까?

다른 클래스에서 beans Map 컬렉션을 참조하여, 마음대로 원소(CoffeeBean)를 추가, 삭제할 수 있게 되고 원소의 속성인 quantity를 수정할 수 있게 된다.

커피머신 사용자가, 커피 종류별로 필요한 원두의 그램수를 바꿀 수 있는 상황을 상상해보면 상당히 어색함을 느낄 수 있을 것이다. (사용자가 바리스타가 아닌 이상..)

이렇게 되는 순간 캡슐화는 깨진것이다.

예시를 통해 알아봤듯이 캡슐화는 현실세계를 매우 닮아있다.

### Inheritance - 상속(재사용)

상위 클래스의 특성을 하위 클래스에서 상속하고 거기에 필요한 특성을 추가, 즉 확장해서 사용할 수 있다.

본질적인 목적은 재사용이다.

다음은 네스프레소사의 2종의 커피머신이다. 편의상 순서대로 A,B모델 이라고 해보자.

![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQq8KnKqqqEA_ngda_XBua-UIhoIiUNnV2Fuj6OXZWizuMUqTZw0F0IkBDESb4Jthi5CDc05PQ&usqp=CAc)

![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTR8MEtrGpNIYutg6u_nzlAfB3MAc-QlW6xDu0SHSyr3XOXwJIQCwf4-T799khA7ho1SijAzN3C&usqp=CAc)


B모델은 A모델에 없는 에스프레소 추출 기능이 있다. 그 점 외에는 기능상 아무런 차이점이 없다.

그럼에도 불구하고, B모델을 제작할 때, A모델의 설계를 참조하지 않고 완전히 재창조 해야할까?

~~~java

public class BasicCoffeeMachine { 
    protected Map configMap; 
    protected Map beans; 
    protected Grinder grinder; 
    protected BrewingUnit brewingUnit; 
 
    public BasicCoffeeMachine(Map beans) { 
        this.beans = beans; 
        this.grinder = new Grinder(); 
        this.brewingUnit = new BrewingUnit(); 
 
        this.configMap = new HashMap(); 
        this.configMap.put(CoffeeSelection.FILTER_COFFEE, new Configuration(30, 480)); 
    } 
 
    public Coffee brewCoffee(CoffeeSelection selection) throws CoffeeException { 
        switch (selection) { 
            case FILTER_COFFEE: 
                return brewFilterCoffee(); 
            default: 
                throw new CoffeeException("CoffeeSelection [" + selection + "] not supported!"); 
        } 
    } 
 
    private Coffee brewFilterCoffee() { 
        Configuration config = configMap.get(CoffeeSelection.FILTER_COFFEE); 
 
        // grind the coffee beans 
        GroundCoffee groundCoffee = this.grinder.grind(
            this.beans.get(CoffeeSelection.FILTER_COFFEE), config.getQuantityCoffee()); 
 
        // brew a filter coffee 
        return this.brewingUnit.brew(
            CoffeeSelection.FILTER_COFFEE, groundCoffee, config.getQuantityWater()); 
    } 
 
    public final void addBeans(CoffeeSelection sel, CoffeeBean newBeans)
        throws CoffeeException {
        CoffeeBean existingBeans = this.beans.get(sel);

        if (existingBeans != null) { 
            if (existingBeans.getName().equals(newBeans.getName())) { 
                existingBeans.setQuantity(existingBeans.getQuantity() + newBeans.getQuantity()); 
            } else { 
                throw new CoffeeException(
                    "Only one kind of beans supported for each CoffeeSelection."); 
            } 
        } else { 
            this.beans.put(sel, newBeans); 
        } 
    } 
}

~~~




#### Abstraction - 추상화 

추상화는 구체적인 것을 분해해서 관심 영역에 있는 특성만 가지고 재조합 하는것을 말한다.

추상이라는 단어의 의미는 무엇일까?

커피머신에는 정말 다양한 속성들이 있을것이다.
(크기, 색깔, 가격..)


![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQq8KnKqqqEA_ngda_XBua-UIhoIiUNnV2Fuj6OXZWizuMUqTZw0F0IkBDESb4Jthi5CDc05PQ&usqp=CAc)

추상화와 캡술화가 비슷하다고 생각할 수 있을것 같아. 두 개의 차이점을 표로 정리해보았다.

추출	캡슐화
객체 지향 프로그래밍의 추상화는 디자인 수준에서 문제를 해결합니다.	캡슐화는 구현 수준을 해결합니다.
프로그래밍의 추상화는 가장 필수적인 정보를 표시하면서 원하지 않는 세부 사항을 숨기는 것입니다.	캡슐화는 코드와 데이터를 단일 단위로 바인딩하는 것을 의미합니다.
Java의 데이터 추상화를 통해 정보 객체에 포함되어야하는 항목에 초점을 맞출 수 있습니다.	캡슐화는 보안상의 이유로 개체가 어떤 작업을 수행하는지에 대한 내부 세부 정보 또는 메커니즘을 숨기는 것을 의미합니다.

#### Polymorphism - 다형성

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

## 3. 그 외

앨런 케이는 “객체 지향 프로그래밍" 이라는 용어를 1960년대에 만들었습니다. 그는 생물학 지식을 갖추고 있었고, 살아있는 세포들이 서로 통신하는 것과 같은 방법으로 작동 하는 컴퓨터 프로그램을 만들려고 했었습니다.


## 글을 마치며

위의 객체지향의 4가지 특성을 현실세계(커피머신)의 비유를 통해 모두 설명이 가능했다. 

이는 객체지향이 현실세계를 모방하려고 노력했고, 이것들이 특성들에 자연스럽게 녹아있기 때문에 가능했다고 생각한다.

그리고 OOP가 직관적인 이유도 현실세계와 닮아있는 점들이 있기 때문이다.

그리고 OOP의 클래스, 객체 등의 요소도 현실세계를 반영하려고 했기 때문에 자연스럽게 도입된 개념이라고 생각한다.

앨런 케이의 의도는 독립적인 프로그램(세포)들이 서로 메시지를 보냄으로서 정보를 전달하는 것 이었습니다. 독립된 프로그램들의 상태(state)는 결코 외부로 공개되지 않습니다(캡슐화).

엘런케이는 이러한 의도로 OOP를 맨 처음에 구상했다고 한다. 애초에 독립적인 세포에서 구상을 했다는것이 우리의 현실세계를 반영하려는 노력의 출발점이지 않았을까?


## 참고


#### 공식문서

- [https://docs.oracle.com/javase/tutorial/java/concepts/inheritance.html](https://docs.oracle.com/javase/tutorial/java/concepts/inheritance.html)



#### 서적

- 스프링 입문을 위한 자바 객체지향의 원리와 이해


#### 블로그

- [https://betterprogramming.pub/object-oriented-programming-the-trillion-dollar-disaster-92a4b666c7c7]()
- [https://betterprogramming.pub/object-oriented-programming-the-trillion-dollar-disaster-92a4b666c7c7]()
- [https://stackify.com/oop-concept-abstraction/](https://stackify.com/oop-concept-abstraction/)