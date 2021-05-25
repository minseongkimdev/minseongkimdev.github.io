---
title: "[작성중] 리눅스 select(), poll(), epoll()이란?"
layout: post
category: CS
---

## 들어가기 전에
알아 보기전에 꼭 알아야 할 키워드 들이 있다.

1. **File Descriptor** : 리눅스 혹은 유닉스 계열의 프로세스가 파일을 다룰 때 사용하는 개념. 프로세스에서 특정 파일에 접근할 때 사용하는 추상적인 값. 프로세스에서 열린 파일의 목록을 관리하는 테이블의 인덱스.

2. **I/O Multiplexing** : 하나의 통신 채널을 통해서 둘 이상의 데이터를 전송하는 기술, 물리적 장치의 효율성을 높이기 위해, 최소한의 물리적 요소만을 이용하여, 최대한의 데이터를 전달하기 위해 사용되는 기술.

![](https://blog.kakaocdn.net/dn/bkmuyL/btq40REKOVZ/TUYBDu7XcGVZSnZ8PQlkk1/img.png)

멀티플렉싱이 필요한 이유는, 각 파일을 처리할 때 각각의 io통로를 통로를 만들어 각각의 프로세스와 스레드를 만들게 되면 아래와 같은 단점이 있다.

- 프로세스간의 통신을 위해 IPC가 필요하다.
- 프로세스 동기화, 스레드를 동기화 해야한다.
- 컨텍스트 스위칭 등의 오버헤드가 있을 수 있다.

![](https://blog.kakaocdn.net/dn/GvHqy/btq4XE0VCfq/EXSWKOLMtWnYAOb5rKlaF0/img.png)

그래서 위와 같은 단점을 보완하기 위해서 하나의 채널을 통해 둘 이상의 데이터를 송수신 하여, 프로세스의 갯수를 최소한으로 유지하면서 여러개의 파일을 처리하는 방법인 `IO Multiplexing`이 등장하였다. 


3. **System Call** 
