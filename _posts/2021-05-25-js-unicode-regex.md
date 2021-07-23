---
title: 자바스크립트 정규식에서 유니코드 다루기
date: 2021-05-25 18:50:00 +0900
categories: [Javascript]
tags: [javascript, regex, regularexpression, unicode] # TAG names should always be lowercase
---

## 개요
- 프론트에서 input에 4byte 이모지 입력을 막기 위해 정규식 적용하여 이를 기록한다.

## 자바스크립트 정규식 플래그
- 자바스크립트에서 정규식을 쓸 때, 정규식 문 마지막 슬래쉬(/) 뒤에 플래그를 붙일 수 있다.
- 플래그
    - g
        - 전역 검색
    - i
        - 대소문자 구분 없는 검색
    - m
        - 다중행(multi-line) 검색
    - s
        - .에 개행 문자도 매칭(ES2018)
    - u
        - 유니코드; 패턴을 유니코드 코드 포인트의 나열로 취급합니다.
    - y
        - "sticky" 검색을 수행. 문자열의 현재 위치부터 검색을 수행합니다. sticky (en-US) 문서를 확인하세요.

## 유니코드 플래그
- u 플래그를 붙여야 정규식 문 내의 "\u" 문법이 인식된다.

## 적용
- input 내에 모든 4byte 이모지를 빈 문자열로 치환한다.
  ```
  var testString = "🥰🥵🥶🥳🥴🥺👨‍🦰👩‍🦰👨‍🦱👩‍🦱👨‍🦲👩‍🦲👨‍🦳👩‍🦳🎨🎦"
  var converted = testString.replace(/[\u{1F004}-\u{1F9E6}]|[\u{1F600}-\u{1F9D0}]/gu, "")
  console.log("converted", converted)
  ```

## 참고
- [MDN 정규 표현식](https://developer.mozilla.org/ko/docs/Web/JavaScript/Guide/Regular_Expressions){:target="\_blank"}
- [Javascript RegExp Replace](https://stackoverflow.com/questions/6230010/javascript-regexp-replace){:target="\_blank"}
