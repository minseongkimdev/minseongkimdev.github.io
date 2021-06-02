---
title: "코드레벨에서 알아보는 Lombok의 동작원리"
layout: post
category: Java
---


## 0. 글의 순서

- [1. 요약](#1-요약)
- [2. 들어가기 전에](#2-들어가기-전에)
- [3. 사용 예제](#3-사용-예제)



## 1. 요약

Lombok[^1]은 **컴파일타임**에 AST트리를 수정하여, 수정된 부분이 바이트코드에 반영된다. 좀 더 구체적으로 알아보자.

1. javac는 소스파일을 파싱하여 AST트리[^2]를 만든다.
2. **Lombok은 AnnotationProcessor에 따라 AST트리를 동적으로 수정**하고, 새 노드(소스코드)를 추가한 뒤 마지막 바이트코드를 분석 및 생성한다.
3. 최종적으로 javac는 Lombok Annotation Processor에 의해 수정된 AST를 기반으로 바이트코드를 생성한다.

![](https://blog.kakaocdn.net/dn/dx57m7/btq4OgEoDDR/k0gmC0fcjePAOQZl9FGj40/img.jpg)
* Lombok이 처리되는 과정

## 2. AnnotationProcessor

AnnotationProcessor은 Lombok의 어노테이션을 분석해서 AST트리를 수정하는 역할을 한다. 직접 코드를 확인하며 분석해보자.
[공식 repo에서 코드를 확인할 수 있다.](https://github.com/projectlombok/lombok/blob/5120abe4741c78d19d7e65404f407cfe57074a47/src/core/lombok/core/AnnotationProcessor.java)


아래의 코드는 `lombok.core.AnnotationProcessor.java` 의 `process` 함수이다.

아래에서 7번째 줄의 while문 에서 **재귀적으로 루트에서부터 순회**를 하는 것을 확인할 수 있다.
특히 RoundEnvironment의 **rootElements()를 통해 자바 컴파일러가 생성한 AST를 참조**한다.



~~~java
@Override public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
	if (!delayedWarnings.isEmpty()) {
    // 루트 어노테이션을 참조한다.
	Set<? extends Element> rootElements = roundEnv.getRootElements();
           
   // 루트 어노테이션에서부터 순회한다.
	if (!rootElements.isEmpty()) {
		Element firstRoot = rootElements.iterator().next();
		for (String warning : delayedWarnings) processingEnv.getMessager().printMessage(Kind.WARNING, warning, firstRoot);
			delayedWarnings.clear();
		}
	}
		
	for (ProcessorDescriptor proc : active) proc.process(annotations, roundEnv);
		
	boolean onlyLombok = true;
	boolean zeroElems = true;
	for (TypeElement elem : annotations) {
		zeroElems = false;
		Name n = elem.getQualifiedName();
		if (n.toString().startsWith("lombok.")) continue;
		onlyLombok = false;
	}
		
	// Normally we rely on the claiming processor to claim away all lombok annotations.
	// One of the many Java9 oversights is that this 'process' API has not been fixed to address the point that 'files I want to look at' and 'annotations I want to claim' must be one and the same,
	// and yet in java9 you can no longer have 2 providers for the same service, thus, if you go by module path, lombok no longer loads the ClaimingProcessor.
	// This doesn't do as good a job, but it'll have to do. The only way to go from here, I think, is either 2 modules, or use reflection hackery to add ClaimingProcessor during our init.
		
	return onlyLombok && !zeroElems;
}
~~~

다음으로  RoundEnvironment[^3] 인터페이스를 확인해보면
getRootElements()가 정의되어 있고, 앞선 round에서 생성된 root 어노테이션 참조를 리턴한다. 

아래와 같이 친절하게 javadoc을 통해 RoundEnvironment의 getRootElements()에 대해 설명해주고 있다.
번역해보면 이와 같다.

>어노테이션 프로세싱이 생성한 루트 원소를 리턴한다.
>그전 라운드에서 생성된 루트 원소를 리턴한다. 만약 그 전에 것이 없다면, 비어있는 set를 리턴한다.


~~~java
...
public interface RoundEnvironment {
...
    /**
     * Returns the {@linkplain Processor root elements} for annotation processing generated
     * by the prior round.
     *
     * @return the root elements for annotation processing generated
     * by the prior round, or an empty set if there were none
     */
    Set<? extends Element> getRootElements();
...

}
~~~

round에 대해 조금 더 부연설명을 하자면

javac에 어노테이션은 여러 단계에 걸쳐서 프로세싱 되는데, 여기서 round를 하나의 단계라고 이해하면 된다.

![](https://blog.kakaocdn.net/dn/bl3IXu/btq4MgljWtI/9e1Cp40zURM8o9o7KKN0yK/img.png)
*여러 round에 걸쳐 annotation이 처리된다.

## 3. Lombok으로 코드를 생성해보자.

자주 사용되는 패턴 중 하나인 Builder[^4] 패턴을 직접 작성해보자.

아래와 같은 User 클래스가 있고 생성자에 Lombok의 Builder패턴을 사용하기 위한 해당 어노테이션을 달아보았다. 
~~~java
public class User {

  private int id;
  private String name;
  private String email;
  private String nickname;

  @Builder
  public User(String name, String email, String nickname) {
      this.name = name;
      this.email = email;
      this.nickname = nickname;
  }

}    
~~~


그 다음 컴파일 후에 Lombok이 생성한 코드를 확인해보자.
*IntelliJ에 바이트코드를 자바소스코드로 변환해서 보여주는 기능이 있다.*
해당 기능을 사용해서 확인해보면 User 클래스에 컴파일 전과 다르게 Builder패턴과 관련된 코드가 추가된 것을 확인할 수 있다. 추가적으로 User 인스턴스 변수를 쉽게 로그를 확인할 수 있₩도록 toString()을 오버라이딩 해준 모습도 확인할 수 있다. 

~~~java

public class User {
    private int id;
    private String name;
    private String email;
    private String nickname;

    private User(User.Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.nickname = builder.nickname;
    }

    public static User.UserBuilder builder() {
        return new User.UserBuilder();
    }

    public User(final int id, final String name, final String email, final String nickname) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.nickname = nickname;
    }

    public static class UserBuilder {
        private int id;
        private String name;
        private String email;
        private String nickname;

        UserBuilder() {
        }

        public User.UserBuilder id(final int id) {
            this.id = id;
            return this;
        }

        public User.UserBuilder name(final String name) {
            this.name = name;
            return this;
        }

        public User.UserBuilder email(final String email) {
            this.email = email;
            return this;
        }

        public User.UserBuilder nickname(final String nickname) {
            this.nickname = nickname;
            return this;
        }

        public User build() {
            return new User(this.id, this.name, this.email, this.nickname);
        }

        public String toString() {
            return "User.UserBuilder(id=" + this.id + ", name=" + this.name + ", email=" + this.email + ", nickname=" + this.nickname + ")";
        }
    }

    public static class Builder {
        private int id;
        private String name;
        private String email;
        private String nickname;

        public Builder() {
            this.name = this.name;
            this.email = this.email;
        }

        public User.Builder nickname(String nickname) {
            this.nickname = nickname;
            return this;
        }

        public User build() {
            return new User(this);
        }
    }
}


~~~


## 출처

- [10 minutes to teach you how to hack the Java compiler](https://www.programmersought.com/article/42205547853/)

- [기계인간님 위키-빌더 패턴(Builder Pattern)](https://johngrib.github.io/wiki/builder-pattern/)

- [Lombok은 어떻게 동작되나? 간단정리](https://free-strings.blogspot.com/2015/12/lombok.html)


## 각주

[^1]: Lombok : 어노테이션 기반으로 코드를 자동완성 해주는 라이브러리이다. 잘 정리된 자료가 많이 때문에 구체적인 설명과 사용법은 생략한다.

[^2]: AST 트리 : Abstract Syntax Tree의 약자. 주로 컴파일러에서 프로그래밍 언어의 문법 및 각 문단의 역할 구조를 표현하기 위해서 사용된다. 특히 문법적으로 틀린 부분이 없는지, 소스코드 검사 단계에서 사용된다.

[^3]: RoundEnvironment인터페이스는 Annotation processing tool 프레임워크이고, 프로세서가 annotation processing의 라운드를 수정할 수 있도록 도와준다.

[^4]: Builder 패턴 : 객체 생성을 깔끔하고 유연하게 하기 위한 기법.
