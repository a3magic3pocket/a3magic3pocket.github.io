---
title: search_after를 이용한 pagination(10000개 이상 문서에서 조회)
date: 2022-07-08 19:00:00 +0900
categories: [ELK]
tags: [elk, elasticsearch, search, search-after] # TAG names should always be lowercase
---

## 개요

- elasticsearch에서는 pagination을 위해 search 명령 시 from과 size로 한 번에 볼 수 있는 문서양을 조절할 수 있다.
- 문제는 from과 size 문서는 10,000개 이하 문서까지만 조회가 가능한다는 것이다.
- 10,001번째 문서는 from과 size로 조회하면 빈 값만 리턴하게 된다.
- 이때는 정렬할 필드를 지정 후 정렬하고, search_after 인자를 이용해서 그 다음 문서를 조회할 수 있다.

## 예시 데이터

- | id  | 부서명 |
  | --- | ------ |
  | 1   | A0001  |
  | 2   | A0002  |
  | 3   | A0003  |
  ...

## search_after 인자를 이용한 조회 예시

- ```bash
    GET /_search
    {
        "size": 10,
        "query": {
            "match_all": {}
        },
        "sort": [
            {"id": "desc"}
        ],
        "search_after": [
            "id": 4
        ]
    }
  ```

## 인자 설명
- size
    - 한 번에 조회할 행의 수를 의미한다.
- query
    - 조회 조건을 입력하는 부분이다.
    - 예시에서는 모든 행을 조회한다.
- sort
    - 정렬기준을 지정하는 컬럼이다.
    - 여러 개의 정렬기준을 지정할 수 있다.
    - 정렬기준에 수 만큼 search_after도 지정해야한다.
    - 정렬은 기본적으로 numeric 인자를 허용한다.  
      만약 text type이면 keyword로 변경해서 지정해주면 처리된다.  
      ex) id => id.keyword
- search_after
    - 조회할 기준이 되는 행의 정렬기준 필드 값을 입력한다.
    - search_after 값이 포함된 행부터 size 수 만큼의 행을 search 명령 결과로 리턴하게된다.
    - 예시는 id가 4인 행으로부터 이후 10개의 행을 결과로 리턴한다.

## pit 인자
- 공식문서에는 pit을 생성하여 사용하도록 권고하고 있다.
- search_after 사용 시 정렬이 들어가기 때문에 비용이 비쌀 것으로 추정된다.
- pit은 이를 캐싱하는 역할을 하는 것으로 보인다.
- 그래서 검색 성능이 좀 더 향상시킬 수 있다.
- 그러나 multi index 검색 시에는 사용할 수 없어서 여기서는 pit 인자를 제외시켰다.

## 참고

- [elastic 공식홈페이지 Paginate search results](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html){:target="\_blank"}
- [elastic 공식홈페이지 Point in time](https://www.elastic.co/guide/en/elasticsearch/reference/current/point-in-time-api.html){:target="\_blank"}
- [[ElasticSearch] search_after 사용하기](https://minu0807.tistory.com/76){:target="\_blank"}
- [[elasticsearch] 깊은(deep) 페이지네이션](https://velog.io/@nmrhtn7898/elasticsearch-%EA%B9%8A%EC%9D%80deep-%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98){:target="\_blank"}
- [[elastic search] document 10,000개 이상 검색 하는 방법](https://jaimemin.tistory.com/1543){:target="\_blank"}
