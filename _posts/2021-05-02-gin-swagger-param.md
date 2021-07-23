---
title: gin-swagger 파라미터 유형에 따른 주석 변경
date: 2021-05-02 23:40:00 +0900
categories: [Golang, gin-swagger]
tags: [gin, swagger, gin-swagger]    # TAG names should always be lowercase
---
## 개요
- gin-swagger로 API 문서 만들때, 파라미터 받는 방식마다  
gin-swagger 주석이 다른데 자주 까먹어서 기록한다.

## 파라미터 종류
- Query Parameter
  - method: GET, URL: /base/url?param=123
- Path Variable
  - method: GET, URL: /base/url/123

## Gin-swagger 문서 주석
- 위치
  - controller 함수 위에 적으면 된다.
  - [controller example](https://github.com/swaggo/swag/blob/master/example/celler/controller/accounts.go){:target="_blank"}
- 파라미터 주석 명세
  - @Param [param name] [param type] [data type] [is mandatory] [comment] [attribute(optional)] 
- Query Parameter
  - param type이 query
    - ex) // @Param id query int true "ID"
- Path Variable
  - param type이 path
    - ex) // @Param id path int true "ID"
  - Router 명세에 path variable을 넣어줘야 한다.
    - ex) // @Router /base/url/{id} [get]

## 참고
- [[번역] Path Variable과 Query Parameter는 언제 사용해야 할까?](https://ryan-han.com/post/translated/pathvariable_queryparam/){:target="_blank"}
- [Declarative Comments Format](https://swaggo.github.io/swaggo.io/declarative_comments_format/api_operation.html){:target="_blank"}
- [controller example](https://github.com/swaggo/swag/blob/master/example/celler/controller/accounts.go){:target="_blank"}