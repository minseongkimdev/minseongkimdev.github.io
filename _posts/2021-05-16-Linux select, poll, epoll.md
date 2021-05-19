---
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

Lorem ipsum[^1] dolor sit amet, consectetur adipiscing elit. Pellentesque vel lacinia neque. Praesent nulla quam, ullamcorper in sollicitudin ac, molestie sed justo. Cras aliquam, sapien id consectetur accumsan, augue magna faucibus ex, ut ultricies turpis tortor vel ante. In at rutrum tellus.

# Sample heading 1
## Sample heading 2
### Sample heading 3
#### Sample heading 4
##### Sample heading 5
###### Sample heading 6

Mauris viverra dictum ultricies. Vestibulum quis ipsum euismod, facilisis metus sed, varius ipsum. Donec scelerisque lacus libero, eu dignissim sem venenatis at. Etiam id nisl ut lorem gravida euismod.

## Lists

Unordered:

- Fusce non velit cursus ligula mattis convallis vel at metus[^2].
- Sed pharetra tellus massa, non elementum eros vulputate non.
- Suspendisse potenti.

Ordered:

1. Quisque arcu felis, laoreet vel accumsan sit amet, fermentum at nunc.
2. Sed massa quam, auctor in eros quis, porttitor tincidunt orci.
3. Nulla convallis id sapien ornare viverra.
4. Nam a est eget ligula pellentesque posuere.

## Blockquote

The following is a blockquote:

> Suspendisse tempus dolor nec risus sodales posuere. Proin dui dui, mollis a consectetur molestie, lobortis vitae tellus.

## Thematic breaks (<hr>)

Mauris viverra dictum ultricies[^3]. Vestibulum quis ipsum euismod, facilisis metus sed, varius ipsum. Donec scelerisque lacus libero, eu dignissim sem venenatis at. Etiam id nisl ut lorem gravida euismod. **You can put some text inside the horizontal rule like so.**

---
{: data-content="hr with text"}

Mauris viverra dictum ultricies. Vestibulum quis ipsum euismod, facilisis metus sed, varius ipsum. Donec scelerisque lacus libero, eu dignissim sem venenatis at. Etiam id nisl ut lorem gravida euismod. **Or you can just have an clean horizontal rule.**

---

Mauris viverra dictum ultricies. Vestibulum quis ipsum euismod, facilisis metus sed, varius ipsum. Donec scelerisque lacus libero, eu dignissim sem venenatis at. Etiam id nisl ut lorem gravida euismod. Or you can just have an clean horizontal rule.

## Code

Now some code:

```
const ultimateTruth = 'this theme is the best!';
console.log(ultimateTruth);
```

And here is some `inline code`!

## Tables

Now a table:

| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |

## Images

![theme logo](https://raw.githubusercontent.com/riggraz/no-style-please/master/logo.png){:.ioda}

Logo of *no style, please!* theme[^4]

---
{: data-content="footnotes"}

[^1]: this is a footnote. It should highlight if you click on the corresponding superscript number.
[^2]: hey there, i'm using no style please!
[^3]: this is another footnote.
[^4]: this is a very very long footnote to test if a very very long footnote brings some problems or not. I strongly hope that there are no problems but you know sometimes problems arise from nowhere.

### 출처

-  [https://reakwon.tistory.com/117](https://reakwon.tistory.com/117)
- https://docs.oracle.com/javase/8/docs/technotes/guides/io/enhancements.html
- https://duksoo.tistory.com/entry/System-call-%EB%93%B1%EB%A1%9D-%EC%88%9C%EC%84%9C
- https://niklasjang.github.io/backend/select-poll-epoll/
- https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/poll.2.html
- https://www.joinc.co.kr/w/Site/system_programing/File/select
- https://ozt88.tistory.com/21﻿