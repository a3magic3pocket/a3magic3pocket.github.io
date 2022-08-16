---
title: elasticsearch depreciation log 안 보기
date: 2022-06-13 18:55:00 +0900
categories: [ELK]
tags: [elk, elasticsearch, index]    # TAG names should always be lowercase
---
## 개요
- elasticsearch의 depreciation log가 너무 많이 기록되어  
  kibana에서 조회 시 정작 필요한 로그 조회가 힘들게 되었다.
- depreciation log를 보지 않도록 설정한다.

## logger.org.elasticsearch.deprecation를 OFF
- ```bash
    curl -XPUT \
        "http://localhost:9200/_cluster/settings?pretty" \
        -H 'Content-Type: application/json' \
        -d '{ "persistent": { "logger.org.elasticsearch.deprecation": "OFF" } }'
    ```

## 참고
- [Deprecation logging](https://www.elastic.co/guide/en/elasticsearch/reference/current/logging.html){:target="_blank"}
