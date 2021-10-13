---
title: "Spring with MySQL Replication"
layout: post
category: MySQL
---

## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가면서](#1-들어가면서)
- [2. MySQL Replication(복제)란?](#2-mysql-replication복제란)
	- [MySQL Replication의 동작 원리](#mysql-replication의-동작-원리)
	- [MySQL의 Replication 방식](#mysql의-replication-방식)
		- [Async Replication (비동기 복제)](#async-replication-비동기-복제)
		- [Semi-sync Replication (준동기 복제)](#semi-sync-replication-준동기-복제)
- [3. AbstractRoutingDataSource](#3-abstractroutingdatasource)
- [4. AbstractRoutingDataSource 동작원리 더 알아보기](#4-abstractroutingdatasource-동작원리-더-알아보기)
- [5. 글을 마치며](#5-글을-마치며)
- [출처](#출처)
- [참고 서적](#참고-서적)

## 1. 들어가면서

Spring 프로젝트를 진행하면서 MySQL Replication을 적용하게 되어 이와 관련된 내용을 정리해보고자 한다.

대규모 트래픽이 발생하다는 가정 아래 프로젝트를 진행하고 있다.

특히 조회와 관련된 기능이 다수 존재하여 이에 대한 부하를 줄이기 위해 MySQL을 이중화 하여 Master는 쓰기, 수정, 삭제 연산을 처리하고 Slave에서는 읽기 연산만을 수행할 수 있게 MySQL Replication을 도입하게 되었다.

## 2. MySQL Replication(복제)란?


![](https://www.researchgate.net/profile/Peter-Kieseberg/publication/266750134/figure/fig4/AS:919665962393600@1596276860039/How-MySQL-replication-works.png)


Master DB의 내용을 복제하여 1개 또는 다수의 Slave가 복제를 하여 사용하는 것을 의미한다.

Read-Write 쿼리와 Read-Only 쿼리를 구분하여 DB의 부하를 분산하고 백업을 수행하는 목적으로 사용한다.

### MySQL Replication의 동작 원리

우선 Replication에 필요한 요소들에 대해 알아보자

- Binary Log : Master에서의 변경을 기록하기 위한 로그 
- Master Thread : Binary Log를 읽어 Slave에 전송하기 위한 스레드
- I/O Thread : Slave에서 데이터를 수신하여 Relay Log에 기록하기 위한 스레드
- Relay Log : Slave의 I/O Thread가 Master로부터 Binary Log를 수신하여 Slave측에 저장한 로그
- SQL Thread : Relay Log를 읽어 Slave에 적용하기 위한 스레드

다음과 같은 순서로 Replication이 진행된다.


1. Commit을 수행하기 전에 Binary Log에 변경사항을 기록한다.
2. 스토리지 엔진에서 트랜잭션 Commit을 수행한다.
3. Master Thread는 Binary Log를 읽어 Slave에 전송한다.
4. Slave의 I/O Thread는 Master로부터 수신한 데이터를 Relay Log에 기록한다.
5. Slave의 SQL Thread는 Relay Log에 기록된 변경 데이터를 읽어서 스토리지 엔진에 적용한다.


### MySQL의 Replication 방식

아래의 두 가지 방식이 존재한다.

#### Async Replication (비동기 복제)

가장 오래됐고 기본적으로 사용되는 방식이며 이름에서 알 수 있다 싶이 비동기적으로 Replication를 진행한다.

Master서버에서 Binary Log Dump Thread가 Slave를 위해 가동되며, Slave의 IO Thread와 Master Thread가 연결되어 slave DB의 IO Thread 요청에 의해 Master Dump Thread가 Binary Log를 읽어 Slave에 전달하게 된다.  이러한 과정이 비동기적으로 동작하여 Master에서는 Slave의 Replication 여부를 확인하지 않는다는게 특징이다.


![](https://blogfiles.pstatic.net/MjAxOTA3MjRfMjk4/MDAxNTYzOTM0MTI4MTM0.xDqR5x4rueiMlpPfrx9DdcrYiboIyJ7OiCiw3YUdioIg.tgEJDaF9tjbGmQ1Tv6qrfKeBe3eEVsgF5u09Ukovf2Yg.PNG.parkjy76/replicationnew.png?type=w2)


#### Semi-sync Replication (준동기 복제)

![](https://blogfiles.pstatic.net/MjAxOTA3MjRfMjgy/MDAxNTYzOTM0MTI2MDc3.WWdDtzr0Gvr-X9JRjyT-JsNrJEweL3RzPVgaFRAN7Ksg.6Mv9ZtdwO-r9BEK1I7kyJEgE3a922qMp2wYncCSU1g8g.PNG.parkjy76/replicationseminew.png?type=w2)

Master와 Slave간의 동기화를 보장하기 위해 서로간의 통신을 해서 데이터의 정합성을 보장하는것이 Async Replication과의 차이점이다.


MySQL Replication에 대한 개념은 여기까지 알아보도록 하고 이를 Spring 프로젝트에 적용하는 법에 대해 알아보자.

## 3. AbstractRoutingDataSource

Spring에서 AbstractRoutingDataSource를 통해 동일 DB스키마에 대한 다중 DB 접속 처리를 할 수 있다.

[스프링 공식문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/lookup/AbstractRoutingDataSource.html)에서 AbstractRoutingDataSource를 다음과 같이 기술하고 있다.

> Abstract DataSource implementation that routes getConnection() calls to one of various target DataSources based on a lookup key.

> Lookup Key를 기반으로 타켓 여러 DataSource들 중 하나의 getConnection()를 호출하도록 라우팅하도록 하는 추상 클래스인 DataSource의 구현체이다.

정말 공식문서가 설명하고 있는대로 내부가 구현되어 있는지 직접 코드레벨에서 확인해보자. (생각보다 내부 구현은 매우 단순하다)


커넥션을 가져오는 메서드는 determineTargetDataSource()를 Wrapping하고 있으며
~~~java
@Override
public Connection getConnection() throws SQLException {
	return determineTargetDataSource().getConnection();
}
~~~

determineTargetDataSource() 메서드를 내부를 살펴보면

determineCurrentLookupKey() 메서드를 통해 Key를 Lookup하고 해당 키를 통해 DataSource를 리턴해줌을 알 수 있다.


~~~java
protected DataSource determineTargetDataSource() {

	Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");

	Object lookupKey = determineCurrentLookupKey();

	DataSource dataSource = this.resolvedDataSources.get(lookupKey);

		if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
			dataSource = this.resolvedDefaultDataSource;
		}
		if (dataSource == null) {
			throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
		}
		return dataSource;
	}
~~~
마지막으로 아래의 determineCurrentLookupKey() 추상 메서드를 구현해서 
Read-Only인 트랜잭션은 Slave를 바라보는 DataSource로 라우팅 하도록 분기할 수 있다.

~~~java
protected abstract Object determineCurrentLookupKey();
~~~



## 4. AbstractRoutingDataSource 동작원리 더 알아보기

AbstractRoutingDataSource은 아래와 같이 InitializingBean 인터페이스를 구현하고 있다.

~~~java
public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean
~~~

InitializingBean인터페이스는 아래의 메서드 하나를 가지고 있다.

~~~java
void afterPropertiesSet() throws Exception;
~~~

이는 Spring Bean의 Life Cycle과 관련이 있고 InitializingBean을 구현하면 Spring에서 내부적으로 BeanFactory에 의해 해당 Bean 객체의 프로퍼티가 모두 설정된 후 afterPropertiesSet을 호출하고, 이는 객체에 특별한 프로퍼티를 추가적으로 설정하거나 필수적으로 요구되는 사항들이 모두 충족되었는지 검사하는 등 다양한 목적으로 사용된다.


(내가 이전에 작성한 [@Autowired의 동작원리](https://blog.minseong.kim/autowired-deep-dive.html)에 빈의 생명주기에 대한 설명이 있으니 참고하길 바란다)

AbstractRoutingDataSource가 오버라이딩 한 afterPropertiesSet() 내부를 살펴보면 Lookup Key와 Datasource를 HashMap에 담는 forEach문을 확인할 수 있다. 


~~~java
@Override
public void afterPropertiesSet() {

		if (this.targetDataSources == null) {
			throw new IllegalArgumentException("Property 'targetDataSources' is required");
		}

		this.resolvedDataSources = CollectionUtils.newHashMap(this.targetDataSources.size());
		this.targetDataSources.forEach((key, value) -> {

			Object lookupKey = resolveSpecifiedLookupKey(key);
			DataSource dataSource = resolveSpecifiedDataSource(value);
			this.resolvedDataSources.put(lookupKey, dataSource);

		});
		
		if (this.defaultTargetDataSource != null) {

			this.resolvedDefaultDataSource = resolveSpecifiedDataSource(this.defaultTargetDataSource);
		}
	}
~~~

이와 같은 내부 구현 덕분에 런타임에 동적으로 Lookup Key를 통해 적절한 DataSource로 라우팅을 할 수 있는 것이다.



## 5. 글을 마치며

MySQL의 Replication과 이를 Spring에 적용하기 위해 AbstractRoutingDataSource까지 살펴보았다.

MySQL Replication에 대한 개념을 공부하고 이를 Spring 프로젝트에 적용하는 과정에서 AbstractRoutingDataSource에에 대해 학습하고 Spring의 Bean Cycle에 대한 개념을 복습할 수 있는 뜻깊은 기회였다.


##  출처

- [Replication - MySQL Documentation](https://dev.mysql.com/doc/refman/5.7/en/replication.html)

- [MySQL Replication for High Availability](https://severalnines.com/resources/database-management-tutorials/mysql-replication-high-availability-tutorial)

- [분산 데이터베이스 환경에서 RoutingDataSource 사용 시 JTA를 이용한 트랜잭션 처리 - Naver D2](https://d2.naver.com/helloworld/5812258)

- [SpringFramework AbstractRoutingDataSource](https://kwonnam.pe.kr/wiki/springframework/abstractroutingdatasource)

- [Different Types of MySQL Replication Solutions](https://www.percona.com/blog/2017/02/07/overview-of-different-mysql-replication-solutions/)

- [Dynamic DataSource Routing - Spring.io](https://spring.io/blog/2007/01/23/dynamic-datasource-routing/) 

- [A Guide to Spring AbstractRoutingDatasource](https://www.baeldung.com/spring-abstract-routing-data-source)

- [MySQL – Replication 구조](http://cloudrain21.com/mysql-replication)

## 참고 서적

- [개발자와 DBA를 위한 Real MySQL](http://www.kyobobook.co.kr/product/detailViewKor.laf?mallGb=KOR&ejkGb=KOR&barcode=9788992939003)