---
title: "DispatcherServlet은 어떤 기본전략을 사용할까?"
layout: post
category: Spring
---

## 0. 글의 순서


## 1. 들어가면서

DispatcherServlet는 properties의 정보를 토대로 본인의 기본 전략을 선택한다.

DispatcherServlet의 기본전략을 분석한다는 것은 **스프링 MVC의 핵심인 DispatcherServlet의 기본적인 동작원리**를 파악하는 것이라 상당한 의의를 가진다.

우선 DispatcherServlet의 구성요소부터 짚고 넘어가보자.


## 2. DispatcherServlet의 구성요소
