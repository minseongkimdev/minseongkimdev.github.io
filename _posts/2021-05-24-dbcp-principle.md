---
title: DBCP의 등장배경과 필요성
layout: post
category: CS 
---

## 0. 글의 순서

1. [DBCP란?](#1.-dbcp란?)
2. [DBCP의 등장 배경](#2.-dbcp의-등장-배경)
3. [DBCP의 특징](#3.-dbcp의-특징)
4. [DBCP 좀 더 알아보기](#4.-dbcp-좀-더-알아보기)

## 1. DBCP란?
**Database Connection Connection Pool**의 약자.

데이터베이스와 연결된 커넥션을 미리 만들어서 pool속에 저장해 두고 있다가 필요할 때에 커넥션을 **풀에서 가져다** 쓰고
**다시 풀에 반환**하는 기법을 의미한다.

## 2. DBCP의 등장배경

그래서 DBCP가 왜 등장했을까? 우선 JDBC의 문제점에 대해서 알아보자.

JDBC에서 Connection을 자주 열었다 닫으면, 병목현상이 발생할 수 있어 웹 어플리케이션의 성능이 크게 저하될 수 있다.

JDBC에서는 아래와 같은 과정을 거쳐서 Connection을 맺었다.

> 1. Database Driver를 로드한다.
> 2. JDBC를 통해 Database에 연결한다.
> 3. statement를 생성한다.
> 4. SQL statement를 실행한다.
> 5. Connection을 종료한다.

이 과정에서 특히 웹서버에서 물리적으로 DB서버에 최초로 연결되어 Connection 객체를 생성하는 부분에서 비용이 가장 많이 발생하는데

모든 요청에 대해 Driver를 로드하고, Connection 객체를 생성하게 되면 굉장히 비효율적이다.

또한 JDBC 인터페이스는 별도의 리소스 관리 기능을 제공하지 않는다.

이러한 JDBC의 특징으로 인해 아래와 같은 문제가 발생할 수 있다.

1. 데이터베이스에 빈번한 **Connection을 맺을 때마다 시스템의 메모리를 점유**한다.
 특히 대규모 트래픽이 발생하는 서비스에서는 필연적으로 많은 시스템 리소스를 차지하여 **응답속도가 늦어지거나 OOM이 발생**할 수 있다.

2. 데이터베이스 Connection을 **사용한 뒤에 끊어야 한다**. 그렇지 않으면 메모리 leak이 발생할 수 있다.

3. 만약 Connection의 수가 **계속해서 늘어나면**, 메모리 leak 혹은 OOM이 발생할 수 있다.

즉 위와 같은 JDBC의 비효율적인 1~5의 연결과정을 보완하고 위의 문제를 방지하기 위해 DBCP가 등장하였다.


## 3. DBCP의 특징

#### 1. **자원 재사용** : 
풀속에 **미리 커넥션이 생성**되어 있어 커넥션을 생성하는데 드는 연결 시간을 줄일 수 있다.
#### 2. **연결 최대 수 제한** : 
 커넥션을 **계속해서 재사용**하여 생성되는 커넥션 수 계속해서 늘어나지 않고 **일정하게 유지**된다.
#### 3. **더 빠른 성능** : 
기존의 사용가능한 연결을 재활용 하여, **연결 및 해제에 의해 발생하는 오버헤드를 방지**하여 시스템 전체의 응답 시간을 줄일 수 있다.

#### 4. **자원 누수 방지** :
미리 설정된 **timeout**을 통해 일정 시간 후에 문제가 생긴 Connection을 **강제로 끊을 수 있어** 이로 인한 자원의 누수를 방지할 수 있다.


## 4. DBCP 좀 더 알아보기

Connection Pool를 효율적으로 관리하기 위한 값들이 아래와 존재하는데, 이러한 값들은 DBCP 구현체를 제공하는 라이브러리를 통해 설정할 수 있다.

- maxActive :	동시에 사용할 수 있는 최대 커넥션 개수
- maxIdle	 : Connection Pool에 반납할 때 최대로 유지될 수 있는 커넥션 개수
- minIdle	: 최소한으로 유지할 커넥션 개수
- initialSize : 최초로 getConnection() Method를 통해 커넥션 풀에 채워 넣을 커넥션 개수

위의 값들을 어떻게 설정해야하는지는  다음 포스팅에서 알아보자.

### 출처

-  [Principle of Database Connection Pool](https://www.programmersought.com/article/49193880081/)
-  [Understanding of database connection pool](https://www.programmersought.com/article/26384975899/)
-  [DB Connection Pool에 대한 이야기](https://www.holaxprogramming.com/2013/01/10/devops-how-to-manage-dbcp/)


