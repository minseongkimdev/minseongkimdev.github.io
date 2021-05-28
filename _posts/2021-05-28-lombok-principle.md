---
title: "Lombok의 동작원리"
layout: post
category: Java
---


## 0. 글의 순서

- [0. 글의 순서](#0-글의-순서)
- [1. 들어가기 전에](#1-들어가기-전에)
- [2. 사용 예제](#2-사용-예제)



## 1. 들어가기 전에



## 2. 사용 예제

자주 사용되는 패턴 중 하나인 Builder 패턴을 직접 작성해보고 마무리 해보자.

`@AllArgsConstructor(access = AccessLevel.PRIVATE)
 @Builder(builderMethodName = "travelCheckListBuilder")
 @ToString
 public class TravelCheckList {

    private Long id;
    private String passport;
    private String flightTicket;
    private String creditCard;
    private String internationalDriverLicense;
    private String travelerInsurance;

    public static TravelCheckListBuilder builder(Long id) {
        if(id == null) {
            throw new IllegalArgumentException("필수 파라미터 누락");
        }
        return travelCheckListBuilder().id(id);
    }
}`