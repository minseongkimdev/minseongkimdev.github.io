---
title: "PreparedStatement의 동작원리로 알아보는 이점"
layout: post
category: DB
---

## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가기 전에](#1-들어가기-전에)
- [2. Statement 처리 방식](#2-statement-처리-방식)
- [3. Prepared Statement의 동작원리](#3-prepared-statement의-동작원리)
- [4. Prepared Statement의 이점](#4-prepared-statement의-이점)
    - [보안적 측면](#보안적-측면)
    - [성능적 측면](#성능적-측면)
    - [기능적 측면](#기능적-측면)
- [5. 글을 마치며](#5-글을-마치며)
- [출처](#출처)
- [각주](#각주)


## 1. 들어가기 전에

Prepared Statement는 SQL문은 반복적으로 실행하는데 사용되는 기능이다.

이를 사용함으로써, 기능적, 보안적, 성능적 측면에서 이점이 있다.

더 구체적으로 알아보기 전에 Prepared Statement와 Statement의 처리 방식에 대해서 알아보자.

## 2. Statement 처리 방식


Prepared Statement의 동작원리를 알아보기 전에 Statement이 어떻게 처리되는지 알아보자
![](https://miro.medium.com/max/700/0*BpxYqbBLM6hyylux.png)

1. **Parsing** : SQL을 각각의 토큰으로 쪼개 분석한다. 또한 SQL의 문법적인 오류를 체크하여 유효성을 검사한다.
2. **Semantics Check** : DBMS가 쿼리의 유효성을 검증한다.(테이블과 컬럼이 존재하는지, 유저가 쿼리를 실행할 권한이 있는지 등)
3. **Binding** : 쿼리는 기계가 이해할 수 있는 바이트코드로 변환된다. 그 다음, 쿼리는 컴파일 되고 그다음 데이터베이스 서버로 전송되어 최적화되고 실행된다.
4. **Query Optimization** :  DBMS는 쿼리를 실행할 최적화된 알고리즘을 고른다.
5. **Cache** : 최적의 알고리즘은 캐시에 저장된다. 그래서 다음에 같은 쿼리가 실행될 때 아래의 과정을 생략한다.
6. **Execution** : 쿼리가 실행되고 그 결과는 유저에게 리턴된다.


## 3. Prepared Statement의 동작원리

Prepared Statement의 처리 과정은 Statement의 처리 과정과 비슷하지만, 몇 가지 다른 점이 있다.

1. Parsing과 Semetics Check까지는 동일하다.
2. Binding할 때, 데이터베이스 엔진은 Placeholders를 체크하고, Placeholder를 포함하여 컴파일 된다.
3. Cache과정은 동일하다.
4. Cache와 Execution사이에 **Placeholder Replacement라는 추가적인 처리 과정**이 있다. 여기에서, Placeholder는 유저의 데이터로 교체된다. (쿼리는 **이미 Binding 되어 있어, 더 이상의 쿼리에 대한 Compilation이 진행되지 않는다.**)
5. 쿼리가 실행되고 결과가 유저에게 리턴된다.
![](https://miro.medium.com/max/700/0*ds-xeCuwyByLlVnT.png)


## 4. Prepared Statement의 이점

위에서 Prepared Statement의 동작원리에 대해 알아봤으니 이제 이를 통해 얻게되는 이점에 대해서 알아보자.
#### 보안적 측면

보안적 측면을 얘기하기 전에 우선 SQL Injection에 대해서 먼저 알아보자.

아래의 그림처럼 악의적인 사용자가 공격을 목적으로 하는 SQL문을 주입하고 실행하여 데이터베이스를 조작하는 행위이다.

![](https://www.researchgate.net/profile/Muhammad-Iqbal-274/publication/322250414/figure/fig3/AS:579066325864448@1515071578177/A-SQL-injection-attack.png)

이 공격은 OWASP[^1] TOP10 중 첫번째에 속해있고, 공격이 비교적 쉬운편이면서 공격에 성공했을 때 큰 피해를 입힐 수 있는 공격이다.

2017년 3월에 SQL Injection으로 인해 여기어때의 대규모 개인정보 유출 사건도 있었다.

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=http%3A%2F%2Fcfile23.uf.tistory.com%2Fimage%2F99D4993C5C8890FB1D148B)
- 여기어때의 해킹사건 당시 사과문

위에서 알아봤듯이 Prepared State가 처리되는 과정에서 Parsing, Sementics Check, Cache과정까지는 동일하지만, 그 다음에 Placeholder Replacement라는 과정이 있다.
이 과정에서 제공된 데이터는 **단순 문자열로 취급되고, 절대 원래의 쿼리의 로직에 영향을 줄 수 없다.**
Injection 방식은 원래의 쿼리를 악의적으로 수정하는 방식으로 공격하는 방식이기 때문에 이를 예방해준다.


#### 성능적 측면

단순 Statement를 사용하면 매번 SQL를 execute할 때 마다, Parsing -> Semetics Check -> Compliation 과정을 거친다.

하지만 Prepared Statement는 한번 Compliation 후에 Placeholder를 활용해 쿼리문을 재사용할 수 있어 위의 과정을 생략하며 Execute된다.
Statement에서 매번 Parsing -> Semetics Check -> Compliation 과정으로 인한 오버헤드를 방지할 수 있다.

#### 기능적 측면

Statement는 컴파일 타임에 쿼리가 결정되어 변경이 불가능 하다. 
하지만 Prepared Statement를 통해 런타임 동안 Placeholder를 활용해 상황에 맞는 인자를 넘겨 유연하게 프로그래밍할 수 있다.


## 5. 글을 마치며

동작원리를 이해하니 추상적으로 느껴졌던 Prepared Statement의 이점들을 확실하게 이해할 수 있었다.

어떤 기술이던 간에 동작원리를 확실히 이해하는 것이 중요하다.

## 출처

- [SQL Injection 이란? (SQL 삽입 공격)](https://noirstar.tistory.com/264)

- [How to prevent SQL Injection vulnerabilities: How Prepared Statements Work](https://www.hackedu.com/blog/how-to-prevent-sql-injection-vulnerabilities-how-prepared-statements-work)


## 각주

[^1]: Open Web Application Security Project의 약자. 즉 오픈소스 웹 어플리케이션 보안 프로젝트. 웹의 개인정보 노출, 악성 파일 및 스크립트, 보안 취약점 등을 연구하며 매년 이중 빈도가 많고 피해가 큰 10가지 취약점을 발표한다.
