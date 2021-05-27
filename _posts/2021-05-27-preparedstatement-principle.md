---
title: "PreparedStatement의 동작원리 "
layout: post
category: DB
---

## 0. 글의 순서

- [1. 들어가기 전에 - SQL Injection이란?](#1-들어가기-전에---sql-injection이란)
- [2. 일반적인 쿼리의 처리 방식](#2-일반적인-쿼리의-처리-방식)
- [3. Prepared Statement의 처리과정](#3-prepared-statement의-처리과정)
- [출처](#출처)
- [각주](#각주)


## 1. 들어가기 전에 - SQL Injection이란?

preparedStatement는 SQL Injection을 방어하기 위한 기술이다.
SQL Injection에 대해서 먼저 알아보자.


아래의 그림처럼 악의적인 사용자가 공격을 목적으로 하는 SQL문을 주입하고 실행하여 데이터베이스를 조작하는 행위이다.

![](https://www.researchgate.net/profile/Muhammad-Iqbal-274/publication/322250414/figure/fig3/AS:579066325864448@1515071578177/A-SQL-injection-attack.png)

이 공격은 OWASP[^1] TOP10 중 첫번째에 속해있고, 공격이 비교적 쉬운편이면서 공격에 성공했을 때 큰 피해를 입힐 수 있는 공격이다.

2017년 3월에 SQL Injection으로 인해 여기어때의 대규모 개인정보 유출 사건도 있었다.

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=http%3A%2F%2Fcfile23.uf.tistory.com%2Fimage%2F99D4993C5C8890FB1D148B)
- 여기어때의 해킹사건 당시 사과문


## 2. 일반적인 쿼리의 처리 방식


Prepared Statement의 동작원리를 알아보기 전에 일반적인 SQL이 어떻게 처리되는지 알아보자
![](https://miro.medium.com/max/700/0*BpxYqbBLM6hyylux.png)

1. **Parsing** : SQL을 각각의 토큰으로 쪼개 분석한다. 또한 SQL의 문법적인 오류를 체크하여 유효성을 검사한다.
2. **Semantics Check** : DBMS가 쿼리의 유효성을 검증한다.(테이블과 컬럼이 존재하는지, 유저가 쿼리를 실행할 권한이 있는지 등)
3. **Binding** : 쿼리는 기계가 이해할 수 있는 바이트코드로 변환된다. 그 다음, 쿼리는 컴파일 되고 그 다음 데이터베이스 서버로 전송되어 최적화 되고 실행된다.
4. **Query Optimization** :  DBMS는 쿼리를 실행할 최적화된 알고리즘을 고른다.
5. **Cache** : 최적의 알고리즘은 캐시에 저장된다. 그래서 다음에 같은 쿼리가 실행될 때 아래의 과정을 생략한다.
6. **Execution** : 쿼리가 실행되고 그 결과는 유저에게 리턴된다.


## 3. Prepared Statement의 처리과정

Prepared Statement는 위 처리과정과 비슷하지만, 몇가지 다른 점이 있다.

1. Parsing과 Semetics Check까지는 동일하다.
2. Binding할 때, 데이터베이스 엔진은 Placeholders를 체크하고, Placeholder를 포함하여 컴파일 된다.
3. Cache과정은 동일하다.
4. Cache와 Execution사이에 Placeholder Replacement라는 추가적인 처리 과정이 있다. 이 포인트에서, Placeholder는 유저의 데이터로 교체된다. 하지만 쿼리는 **이미 Binding 되어 있어, 더 이상의 쿼리에 대한 Compilation이 진행되지 않는다. 이러한 이유로 인해 유저에게 제공되는 데이터는 단순 문자열로 취급되고, 절대 원래의 쿼리의 로직에 영향을 줄 수 없다.**  원래의 쿼리를 수정하는 방식으로 공격하는 Injection을 위와 같은 방식으로 차단한다.

![](https://miro.medium.com/max/700/0*ds-xeCuwyByLlVnT.png)

## 출처

- [SQL Injection 이란? (SQL 삽입 공격)](https://noirstar.tistory.com/264)

- [How to prevent SQL Injection vulnerabilities: How Prepared Statements Work](https://www.hackedu.com/blog/how-to-prevent-sql-injection-vulnerabilities-how-prepared-statements-work)

- [How to prevent SQL Injection vulnerabilities: How Prepared Statements Work](https://www.hackedu.com/blog/how-to-prevent-sql-injection-vulnerabilities-how-prepared-statements-work)

## 각주

[^1]: Open Web Application Security Project의 약자. 즉 오픈소스 웹 어플리케이션 보안 프로젝트. 웹의 개인정보 노출, 악성 파일 및 스크립트, 보안 취약점 등을 연구하며 매년 이중 빈도가 많고 피해가 큰 10가지 취약점을 발표한다.
