---
title: elasticsearch wildcard를 사용하여 여러 인덱스를 한 번에 삭제하기
date: 2022-06-13 18:55:00 +0900
categories: [ELK]
tags: [elk, elasticsearch, index]    # TAG names should always be lowercase
---
## 개요
- elasticsearch는 기본적으로 와일드카드를 이용한 인덱스 삭제가 금지되어 있다.
- 잠시 해당 설정을 풀고 와일드카드를 이용한 삭제를 한 후 다시 해당 설정으로 복원하도록 한다.

## wildcard를 이용한 삭제
- ```bash
    # wildcard 사용 금지 해제
    curl -XPUT \
        "http://localhost:9200/_cluster/settings?pretty" \
        -H 'Content-Type: application/json' \
        -d '{ "persistent": { "action.destructive_requires_name": false } }'
    
    # test-로 시작하는 인덱스 삭제
    curl -XDELETE "http://localhost:9200/test-*" 

    # wildcard 사용 금지
    curl -XPUT \
        "http://localhost:9200/_cluster/settings?pretty" \
        -H 'Content-Type: application/json' \
        -d '{ "persistent": { "action.destructive_requires_name": true } }'
    ```


## 참고
- [Delete index APIedit](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-delete-index.html){:target="_blank"}
