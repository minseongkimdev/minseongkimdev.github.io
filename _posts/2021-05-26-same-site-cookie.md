---
title: "크롬80 에서 변경된 SameSiteCookie 정책"
layout: post
category: CS
---

## 0. 글의 순서

- [1. SameSite Cookie란?](#1-samesite-cookie란)
- [2. First-party cookie, Third-party cookie란?](#2-first-party-cookie,-third-party-cookie란?)
- [3. SameSite Cookie란?](#3-sameSite-values)
- [4. 변경된 크롬의 정책](#4-변경된-크롬의-정책)
- [5. 글을 마치며](#5-글을-마치며)
- [출처](#출처)
- [각주](#각주)


## 1. SameSite Cookie란?

HTTP 응답 헤더의 SameSite속성을 Set-Cookie를 사용하여 브라우저가 사이트 간 요청과 함께 쿠키를 보내지 못하게 하는 정책이다.

특히 CSRF(Cross Site Request Forgery)[^1] 공격을 막기 위해 추가된 정책이다.

이해를 돕기 위해 예를 들면
피해자가 공격자가 제공한 XSRF 링크에 접속하면, 피해자의 개인정보가 담긴 쿠키가 공격자에게 전달될 수 있는 문제가 발생할 수 있다.

여기까지만 읽어보았을 때 이해가 어려울 수 있으니, 우선 fisrt-party, third-party cookie에 대해 알아보자.

## 2. First-party cookie, Third-party cookie란?

한 웹페이지를 로드하는데, 모든 컨텐츠들이 같은 도메인으로부터 오는 것이 아니다.

우리에게 익숙한 네이버 메인화면이다.
![](https://blog.kakaocdn.net/dn/p9shF/btq5Sbu5t0v/Za7xbUOOGr1oekSDYcH3Jk/img.png)
네이버 메인에는 광고, 뉴스 등의 이미지들이 많은데, 이러한 리소스는 모두 네이버 도메인으로 로부터 오는 것이 아니다.

아래의 뉴스기사 썸네일 이미지는
![](https://blog.kakaocdn.net/dn/bGqLod/btq5OTP5eac/kFq14R2SAjyjbBQj1gkfSK/img.png)

아래의 도메인에서 로드 해오고 있는 것이다.
![](https://blog.kakaocdn.net/dn/9LJat/btq5M36KBUB/KGWkku8PkN9p0ZQ7kV7Ij1/img.png)

여기서, 네이버 도메인 관점에서
`htts://naver.com` 에서 발급한 쿠키를 first-party cookie,
`https://s.pstatic.net` 에서 발급한 쿠키를 third-party cookie라고 한다.

정리해보면 아래와 같다.
![](https://web-dev.imgix.net/image/tcFciHGuF3MxnTr1y5ue01OGLBn2/zjXpDz2jAdXMT83Nm3IT.png?auto=format&w=1600)

- fisrt-party cookie : 현재 사이트의 도메인과 일치하는 쿠키. 
- thid-party cookie : 현재 사이트가 아닌 다른 도메인에서 발급한 쿠키.

흥미로운 예시를 알아보면 다른 도메인에서 단순히 이미지를 로드하는데, third-party cookie를 전송하는 것은 불필요한 오버헤드를 발생시키는 것이지만,
Youtube 영상을 로드해온다고 하면 "나중에 보기" 기능을 누르면, third-party cookie를 전송하여, Youtube의 세션을 활용할 수 있다.

## 3. SameSite Values
first-party, third-party cookie 개념에 대해 이해했으니 이와 연결지어, SameSite의 3개의 속성에 대해 쉽게 이해할 수 있다.


![](https://headerbidding.co/wp-content/uploads/2020/01/Samesite-Cookie-Attribute.png)

- `NONE`
	- 모든 경우에 (도메인이 다른 경우에도) 쿠키를 전송한다.
	- 하지만 None으로 설정할 경우 Secure 속성을 설정 해야 한다.
	- Secure 속성을 설정하지 않으면 아래와 같은 에러가 발생한다.


> Cookie “myCookie” rejected because it has the “SameSite=None” attribute but is missing the “secure” attribute.
> 
> This Set-Cookie was blocked because it had the "SameSite=None" attribute but did not have the "Secure" attribute, which is required in order to use "SameSite=None".
	

- `Lax`
	-  동일한 도메인에 속한 모든 사이트는 사용자가 다른 사이트에서 왔는지, 사이트에서 직접 방문했는지에 관계없이 쿠키를 설정하고 엑세스할 수 있다.
	-  **즉, third-party 쿠키 전송까지 허용한다.**

- `Strict`
	- 이름에서도 알 수 있듯이 엄격한 옵션을 의미한다.
	- 오로지 쿠키를 설정한 도메인만 접근할 수 있다.
	-  쿠키를 `mineseongkimdev.github.io` 에서 발급했다면, 이 쿠키에 접근할 수 있는 유일한 도메인은 `mineseongkimdev.github.io` 뿐이다.
	- **즉, 오로지 first-party 쿠키만 전송하고, 다른 도메인에 third-party 쿠키 전송을 허용하지 않는다.**


## 4. 변경된 크롬의 정책

Chromium 공식 홈페이지의 업데이트 노트를 보면 아래와 같은 문구가 있다.

> **February 4, 2020**: Chrome 80 Stable released. The enablement of the SameSite-by-default

크롬 80버전에서 정책이 엄격하게 변경되었다.
디폴트 설정이 `SameSite = None` 에서 `SameSite = Lax`로 변경되었다.

이로 인해 다른 도메인에서 쿠키값을 서버로 전달하지 않을 수 있다.

여태까지 CSRF를 방어하는 것이 온전히 개발자 책임이었으나, 이제는 크롬에서 디폴트 값을 `Lax`로 설정하였다.

왜냐하면 많은 개발자와 사용자들이 적극적으로 SameSite을 `Lax`, `Strict`로 설정하지 않아서 구글에서 크롬80에서 아예 디폴트 값을 `Lax`로 변경하는 특단의 조치를 취한것이다.


### + 가장 최근에 업데이트된 크롬에서 정책 (작성일 2021.05.26 기준)
##### Mar 18(5월 18일), 2021
> The flags `#same-site-by-default-cookies` and `#cookies-without-same-site-must-be-secure` have been removed from chrome://flags as of Chrome 91, as the behavior is now enabled by default. In Chrome 94, the command-line flag `--disable-features=SameSiteByDefaultCookies`,`CookiesWithoutSameSiteMustBeSecure` will be removed.

크롬 91부터 chrome://flags 에서 `same-site-by-default-cookies`, `cookies-without-same-site-must-be-secure`이 두개의 플래그가 제거된다. 왜냐하면 91부터 디폴트로 활성되기 때문이다.
그리고 크롬 94부터 `-disable-features=SameSiteByDefaultCookies`,`CookiesWithoutSameSiteMustBeSecure`가 제거될 예정이다.

*이 업데이트로 미루어보았을 때 SameSite Cookie 정책을 더 확고히 하려고하는 점을 엿볼 수 있어 흥미롭다.*

## 5. 글을 마치며

SameSite Cookie란 무엇이고 SameSite의 속성에는 어떤 것들이 있고, 크롬 업데이트에서 이를 어떻게 반영하고 마지막으로 최근에는 어떻게 업데이트 됐는지까지 알아보았다.

[참고로 2008년도에 옥션에 CSRF로 인한 대규모 해킹사고가 있었다.](https://biz.chosun.com/site/data/html_dir/2008/04/17/2008041700945.html)

약 13년이 지난 지금에도 CSRF를 막을 완벽한 방법은 없고, 구글도 이를 방지하기 위해 크롬 업데이트에 관련 속성을 수년간 계속해서 변경하는것을 보면, 웹이라는 생태계는 끊임없이 해킹과 방어 기술이 발전하고 있다는 점이 정말 흥미롭다는 생각이 든다.

  
## 출처

### 공식문서

- [MDN Web Docs SameSite cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [The Chromium Projects SameSite Updates](https://www.chromium.org/updates/same-site)
- [Samesite Cookies Explained](https://web.dev/samesite-cookies-explained/)

### 블로그

- [SameSite Cookie attribute?](https://medium.com/compass-security/samesite-cookie-attribute-33b3bfeaeb95)
- [Chrome’s SameSite Cookie Update – What You Need to Do?](https://laptrinhx.com/news/chrome-s-samesite-cookie-update-what-you-need-to-do-gQRLN4D/)
- [SameSite Cookie란? 변경된 크롬 80 쿠키 정책](https://sevendollars.tistory.com/178#:~:text=%ED%81%AC%EB%A1%AC%20%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80%EC%9D%98%20%EA%B8%B0%EB%B3%B8%20%EC%BF%A0%ED%82%A4,%EC%A0%84%EB%8B%AC%ED%95%98%EC%A7%80%20%EC%95%8A%EC%9D%84%20%EC%88%98%20%EC%9E%88%EC%8A%B5%EB%8B%88%EB%8B%A4.&text=SameSite%EB%9E%80%20%EC%99%B8%EB%B6%80%20%EC%82%AC%EC%9D%B4%ED%8A%B8%EC%97%90,%EB%A5%BC%20%EC%84%A4%EC%A0%95%ED%95%98%EB%8A%94%20%EA%B2%83%20%EC%9E%85%EB%8B%88%EB%8B%A4.)

## 각주

[^1]: CSRF란 사이트간 요청 위조라는 뜻의 웹사이트 취약점 공격 방식 중 하나로 사용자의 의지와는 상관없이 공격자가 의도한 행위를 웹사이트에 요청하는 것을 의미한다. 