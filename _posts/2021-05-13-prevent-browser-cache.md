---
title: 브라우저 캐시 막기
date: 2021-05-13 15:49:00 +0900
categories: [Web]
tags: [nocache] # TAG names should always be lowercase
---

## 개요
- 사용자가 브라우저에서 뒤로가기 입력 시 브라우저 캐시가 로드되어 곤란할 때가 있다.
- http header의 값을 변경하여 브라우저가 해당 페이지를 캐싱하지 않도록 할 수 있다.

## 캐싱 막기
- 웹 서버가 해당 페이지를 로드할 때, 아래와 같은 헤더를 추가한다.
  ```
  Cache-Control: no-cache, no-store, must-revalidate
  ```

## 참고
- [Cache-Control](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Cache-Control){:target="\_blank"}
