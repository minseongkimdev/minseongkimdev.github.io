---
title: [작성중] 크롬80 에서 변경된 SameSiteCookie 정책
layout: post
category: Web 
---

## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. SameSite Cookie란?](#1-samesite-cookie란)
- [2. 변경된 크롬의 정책](#2-변경된-크롬의-정책)

- [출처](#출처)

## 1. SameSite Cookie란?

HTTP 응답 헤더의 SameSite속성을 Set-Cookie를 사용하면 쿠키를 first-party로 제한해야 하는지 선언할 수 있다.


CSRF(Cross Site Request Forgery)[^1] 공격을 막기 위해 추가된 정책


### SameSite Values
- `Strict`
	- 이름에서도 알 수 있듯이 엄격한 옵션을 의미한다.
	- 해당 옵션으로 설정되면 도메인이 다를 경우 쿠키를 서버로 전송하지 않는다. 
- `Lax`

- `NONE`
	- 모든경우 (도메인이 다른 경우에도) 쿠키를 전송한다. 	
	- None으로 설정한 경우 Secure 속성을 설정 해야 한다.
	- Secure 속성을 설정하지 않으면 아래와 같은 에러가 발생한다.

> Cookie “myCookie” rejected because it has the “SameSite=None” attribute but is missing the “secure” attribute.
> 
> This Set-Cookie was blocked because it had the "SameSite=None" attribute but did not have the "Secure" attribute, which is required in order to use "SameSite=None".
	

## 2. 변경된 크롬의 정책

Chromium 공식 홈페이지의 업데이트 노트를 보면 아래와 같은 문구가 있다.

> **February 4, 2020**: Chrome 80 Stable released. The enablement of the SameSite-by-default

크롬 80버전에서 정책이 엄격하게 변경되었다.
디폴트 설정이 `SameSite = None` 에서 `SameSite = Lax`로 변경되었다.

이로 인해 다른 도메인에서 쿠키값을 서버로 전달하지 않을 수 있다.

여태까지 CSRF를 방어하는 것이 온전히 개발자 책임이었으나, 




### 가장 최근에 업데이트된 크롬에서 정책 (작성일 2021.05.26 기준)
##### Mar 18, 2021
> The flags `#same-site-by-default-cookies` and `#cookies-without-same-site-must-be-secure` have been removed from chrome://flags as of Chrome 91, as the behavior is now enabled by default. In Chrome 94, the command-line flag `--disable-features=SameSiteByDefaultCookies,CookiesWithoutSameSiteMustBeSecure` will be removed.

크롬 91부터 chrome://flags 에서 `same-site-by-default-cookies`, `cookies-without-same-site-must-be-secure`이 두개의 플래그가 제거된다. 왜냐하면 91부터 디폴트로 활성되기 때문이다.
그리고 크롬 94부터 `-disable-features=SameSiteByDefaultCookies,CookiesWithoutSameSiteMustBeSecure`가 제거될 예정이다.

*이 업데이트로 미루어보았을 때 SameSite Cookie 정책을 더 확고히 하려고하는 점을 엿볼 수 있어 흥미롭다.*

## 3. 글을 마치며





  
## 출처

- [MDN Web Docs SameSite cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [The Chromium Projects SameSite Updates](https://www.chromium.org/updates/same-site)
- [SameSite Cookie란? 변경된 크롬 80 쿠키 정책](https://sevendollars.tistory.com/178#:~:text=%ED%81%AC%EB%A1%AC%20%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80%EC%9D%98%20%EA%B8%B0%EB%B3%B8%20%EC%BF%A0%ED%82%A4,%EC%A0%84%EB%8B%AC%ED%95%98%EC%A7%80%20%EC%95%8A%EC%9D%84%20%EC%88%98%20%EC%9E%88%EC%8A%B5%EB%8B%88%EB%8B%A4.&text=SameSite%EB%9E%80%20%EC%99%B8%EB%B6%80%20%EC%82%AC%EC%9D%B4%ED%8A%B8%EC%97%90,%EB%A5%BC%20%EC%84%A4%EC%A0%95%ED%95%98%EB%8A%94%20%EA%B2%83%20%EC%9E%85%EB%8B%88%EB%8B%A4.)


## 주석

[^1]: CSRF란 사이트간 요청 위조라는 뜻의 웹사이트 취약점 공격 방식 중 하나로 사용자의 의지와는 상관없이 공격자가 의도한 행위를 웹사이트에 요청하는 것을 의미한다. [2008년도에 옥션에 이 공격으로 인한 대규모 해킹사고가 있었다.](https://biz.chosun.com/site/data/html_dir/2008/04/17/2008041700945.html)