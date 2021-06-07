---
title: "내가 해석한 OOP"
layout: post
category: CS
---


## 0. 글의 순서

- [1. 들어가기 전에](#1-들어가기-전에)
- [2. 객체지향의 특성](#2-객체지향의-특성)
  - [Encapsulation - 캡슐화](#encapsulation---캡슐화)
  - [Inheritance - 상속(재사용)](#inheritance---상속재사용)
  - [Abstraction - 추상화](#abstraction---추상화)
  - [Polymorphism - 다형성](#polymorphism---다형성)
- [글을 마치며](#글을-마치며)
- [출처](#출처)


## 1. 들어가기 전에


OOP에 대한 여러 정의가 있지만 구글링 했을 때 가장 흔히 볼 수 있는 정의는 다음과 같다.

> 컴퓨터 프로그램을 명령어의 목록에서 보는 시각에서 벗어나
> 여러개의 독립된 단위, 즉 객체들의 모임으로 파악하고자 하는 것.



하지만 내가 OOP에 대해 공부해보고 내린 정의는 아래와 같다.

> 프로그래밍에서 **현실세계**를 최대한 반영하기 위한 일련의 노력

왜 그렇게 생각하게 되었는지 나의 생각을 공유해보고자 한다.

참고로 이 글에서 객체지향의 특성을 자세하게 설명하진 않는다. 이 부분에 대해 궁금한 독자는 다른 레퍼런스와 함께 참조하는 것을 권장한다.
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

커피 종류별로 물에 커피원두를 몇그램 타야하는지 등의 내부 메커니즘은 전혀 몰라도 된다.

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

만약에 CoffeeMachine 클래스의 beans 속성의 접근제어자가 **private이 아니라 public이었다면 어떤 문제가 발생할까?**

다른 클래스에서 beans Map 컬렉션을 참조하여, 마음대로 원소(CoffeeBean)를 추가, 삭제할 수 있게 되고 원소의 속성인 quantity를 수정할 수 있는 문제가 발생한다.

현실세계에 빗대어 보면 커피머신 사용자가, 커피 종류별로 필요한 원두의 그램수(레시피)를 바꿀 수 있는 상황을 상상해보면 상당히 어색함을 느낄 수 있을 것이다. (사용자가 바리스타가 아닌 이상..)

이렇게 되는 순간 캡슐화는 깨진것이다.

그럼 캡슐화를 지키면서 커피 종류별로 필요한 원두의 그램수(레시피)를 바꿀수 있을까?

아래와 같이 beans 컬렉션에 접근하여 해당 컬렉션의 원소를 꺼내 속성을 변경하는 public 메서드를 통해 구현하면 된다.

~~~java

public setBeanQuntity(CoffeeSelection coffeeSelection, double quantity) {
 // ...
}


~~~

다른 곳에서 beans에 접근할 수 없고, CoffeeMachine 클래스가 의도한대로만 beans에 접근하여 값을 조작해야만 한다.

이럴 경우 캡슐화가 잘 지켜졌다고 말할 수 있다.

### Inheritance - 상속(재사용)

상위 클래스의 속성 또는 메서드를 하위 클래스에서 상속받아 재사용할 수 있고 하위 클래스에서 필요한 속성 또는 메서드를 추가하거나 재정의 할 수 있다.

상속의 본질적인 목적은 재사용이다.

다음은 네스프레소사의 2종의 커피머신이다. 편의상 순서대로 A,B모델 이라고 해보자.

![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQq8KnKqqqEA_ngda_XBua-UIhoIiUNnV2Fuj6OXZWizuMUqTZw0F0IkBDESb4Jthi5CDc05PQ&usqp=CAc)

![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTR8MEtrGpNIYutg6u_nzlAfB3MAc-QlW6xDu0SHSyr3XOXwJIQCwf4-T799khA7ho1SijAzN3C&usqp=CAc)


B모델은 A모델에 없는 에스프레소 추출 기능이 있다. 그 점 외에는 기능상 아무런 차이점이 없다.

그럼에도 불구하고, B모델을 제작할 때, A모델의 설계를 참고하지 않고 완전히 재창조 해야할까?

코드를 통해 설명해보자, A모델에 해당하는 BasicCoffeeMachine클래스와 B모델에 해당하는 PremiumCoffeeMachine가 있다고 해보자.

A모델에는 커피머신에 필요한 기본적인 기능들이 구현되어 있다. B모델은 A모델을 상속받았고, 에스프레소 기능만 추가되었다.
~~~java

public class BasicCoffeeMachine { 

    public BasicCoffeeMachine(Map beans) { 

    } 
 
    public Coffee brewCoffee(CoffeeSelection selection) {
        // ...
    }
 
 
    public final void addBeans(CoffeeSelection sel, CoffeeBean newBeans){
        // ...
    }

    
}

~~~

A모델의 기능이 필요한 부분에서 super 키워드를 통해 그대로 사용하고, 에스프레소 추출 기능을 추가하기 위해 brewEspresso()를 추가 구현 하였다. 

~~~java

public class PremiumCoffeeMachine extends BasicCoffeeMachine { 
    public PremiumCoffeeMachine() { 
        super(beans); 
    }  
 
    private Coffee brewEspresso() { 
        Configuration config = configMap.get(CoffeeSelection.ESPRESSO); 
 
        GroundCoffee groundCoffee = this.grinder.grind(
            this.beans.get(CoffeeSelection.ESPRESSO), config.getQuantityCoffee()); 
        
        return this.brewingUnit.brew(
            CoffeeSelection.ESPRESSO, groundCoffee, config.getQuantityWater()); 
    } 
 
    public Coffee brewCoffee(CoffeeSelection selection) throws CoffeeException { 
        if (selection == CoffeeSelection.ESPRESSO) {
            return brewEspresso(); 
        } else {
            return super.brewCoffee(selection);
        } 
    } 
}

~~~
A 커피머신 클래스에 있는 속성과 기능을 재사용 하여, B모델에서는 에스프레소 추출 기능에만 집중할 수 있다.

**결국 상속의 핵심은 재사용과 확장이다.**

### Abstraction - 추상화

추상화는 결국 구체적인 것에서 관심 영역에 속한 특성만 가지고 재조합 하는것이다.

추상화를 통해 커피머신의 내부 구현이 얼마나 복잡한지 상관 없이 사용자는 쉽게 커피머신의 기능을 사용할 수 있다.

아래의 코드를 통해 구체적으로 알아보자.

추상화를 위해 CoffeeMachine에서 커피를 제조하는 기능외에 불필요한 속성과, 메서드를 제외하고 구현하였다.

~~~java

public class CoffeeMachine {
    private Map<CoffeeSelection, CoffeeBean> beans;

    public CoffeeMachine(Map<CoffeeSelection, CoffeeBean> beans) { 
         this.beans = beans
    }

    public Coffee brewCoffee(CoffeeSelection selection) throws CoffeeException {
        Coffee coffee = new Coffee();
        System.out.println(“Making coffee ...”);
        return coffee;
    }
}

~~~

CoffeeSelection은 커피 종류들을 정의한 enum 클래스이다.

~~~java

public enum CoffeeSelection { 
    FILTER_COFFEE, ESPRESSO, CAPPUCCINO;
}

~~~

CoffeeBean, Coffee는 관련 속성들을 저장하기 위한 POJO 이다.

~~~java
public class CoffeeBean {
     private String name;
     private double quantity;
  	
     public CoffeeBean(String name, double quantity) {
         this.name = name;
        this.quantity;
    }
}

public class Coffee {
    private CoffeeSelection selection;
    private double quantity;
  	
    public Coffee(CoffeeSelection, double quantity) {
        this.selection = selection;
        this. quantity = quantity;
    }
}

~~~

추상화를 통해 CoffeeMachine을 통해 커피를 제조하는 과정(CoffeeApp의 main메서드)이 매우 단순해졌다.

단순히 아래의 CoffeeApp에서 Coffee 클래스의 인스턴스를 생성하여, Map 컬렉션에 CoffeeBean 인스턴스를 추가한다음, 원하는 enum 타입을 파라미터로 넘겨 간편하게 brewCoffee를 호출할 수 있다.


~~~java

public class CoffeeApp {
    public static void main(String[] args) {
        // create a Map of available coffee beans
        Map<CoffeeSelection, CoffeeBean> beans = new HashMap<CoffeeSelection, CoffeeBean>();
        beans.put(CoffeeSelection.ESPRESSO, 
            new CoffeeBean("My favorite espresso bean", 1000));
        beans.put(CoffeeSelection.FILTER_COFFEE, 
            new CoffeeBean("My favorite filter coffee bean", 1000));

        CoffeeMachine machine = new CoffeeMachine(beans);

        try {
	    Coffee espresso = machine.brewCoffee(CoffeeSelection.ESPRESSO);
	} catch(CoffeeException  e) {
	    e.printStackTrace();
        }
    } 
} 

~~~

### Polymorphism - 다형성

커피머신을 통해 커피를 탈 때, 기본 옵션으로만 커피를 탈 수도 있다. 하지만
수량을 선택할 수도 있고, 샷을 추가할 수 있는 기능이 필요할 수 있지 않을까?

즉, 커피를 탄다는 기능은 동일하지만, 내부 메커니즘이 조금씩 달라야 하는 상황이 있을 수 있다.

이럴 경우, 다형성을 통해 하나의 인터페이스를 통해 접근이 가능하다.

위 설명만으로 무슨 말인지 이해하기 어려울 수 있다고 생각한다.

코드를 통해 구체적으로 알아보자.

~~~java

public class BasicCoffeeMachine {
    // ...
    public Coffee brewCoffee(CoffeeSelection selection) throws CoffeeException {
        switch (selection) {
        case FILTER_COFFEE:
            return brewFilterCoffee();
        default:
            throw new CoffeeException(
                "CoffeeSelection ["+selection+"] not supported!");
        }   
    }
  
    public List brewCoffee(CoffeeSelection selection, int number) throws CoffeeException {
        List coffees = new ArrayList(number);
        for (int i=0; i<number; i++) {
            coffees.add(brewCoffee(selection));
        }
        return coffees;
    }
    // ...
}
~~~

아래에서 이름이 동일한 brewCoffee 메서드에 각각 다른 파라미터를 넘겨 상황에 맞게 적절한 메서드를 메커니즘 수행할 수 있다.

~~~java
BasicCoffeeMachine coffeeMachine = createCoffeeMachine();
coffeeMachine.brewCoffee(CoffeeSelection.FILTER_COFFEE);
~~~

~~~java
BasicCoffeeMachine coffeeMachine = createCoffeeMachine();
List coffees = coffeeMachine.brewCoffee(CoffeeSelection.ESPRESSO, 2);
~~~

그리고, 커피머신에 샷 추가 기능이 추가된다고 하면, Overloading을 활용해 매개변수에 샷 추가 관련 파라미터까지 받는 brewCoffee를 추가적으로 구현하면 된다.

~~~java
BasicCoffeeMachine coffeeMachine = createCoffeeMachine();
int shot = 2;
List coffees = coffeeMachine.brewCoffee(CoffeeSelection.ESPRESSO, 2, shot;
~~~

사실 자바에서 다형성을 구현하는 방법에는 Overloading 뿐만 아니라, Overriding, Funcitional Interface등이 더 있다.

그리고 다형성은 동적다형성, 정적 다형성등 더 많은 개념들 이 있지만, 이 글에 다 담기엔 내용이 많아 다른 레퍼런스를 참조하길 바란다.


## 글을 마치며

위의 객체지향의 4가지 특성을 현실세계(커피머신)의 비유를 통해 모두 설명이 가능했다. 

이는 객체지향이 현실세계를 모방하려고 노력했고, 이것들이 특성들에 자연스럽게 녹아있기 때문에 가능했고 객체지향이 직관적일 수 있었던 것도 같은 이유라고 생각한다.

또한 OOP의 클래스, 객체 등의 요소도 현실세계를 반영하려고 했기 때문에 자연스럽게 도입된 개념이라고 생각한다.

처음으로 OOP의 개념을 정립한 앨런 케이의 의도는 독립적인 프로그램(세포)들이 서로 메시지를 보냄으로서 정보를 전달하는 것이었다고 한다.

애초에 엘런 케이가 OOP의 개념을 세포를 통해 구상을 했다는것은

우리가 사는 현실세계를 반영하려는 노력의 첫 출발점이지 않았을까?


## 출처
#### 서적

- 스프링 입문을 위한 자바 객체지향의 원리와 이해


#### 블로그

- [Object-Oriented Programming — The Trillion Dollar Disaster](https://betterprogramming.pub/object-oriented-programming-the-trillion-dollar-disaster-92a4b666c7c7)
- [OOP Concept for Beginners: What is Abstraction?](https://stackify.com/oop-concept-abstraction/)
- [OOP Concept for Beginners: What is Encapsulation](https://stackify.com/oop-concept-for-beginners-what-is-encapsulation/)
- [OOP Concept for Beginners: What is Inheritance?
](https://stackify.com/oop-concept-inheritance/)
- [OOP Concepts for Beginners: What is Polymorphism](https://stackify.com/oop-concept-polymorphism/)
