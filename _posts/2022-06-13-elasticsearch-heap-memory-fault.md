---
title: circuit_breaking_exception, [parent] Data too large 에러
date: 2022-06-13 18:35:00 +0900
categories: [ELK]
tags: [elk, elasticsearch, heap]    # TAG names should always be lowercase
---
## 개요
- kibana가 다운되어 확인해보니, kibana 최초 시작 시 elasticsearch에 요청하는 GET _cat/indices가 실패하고 있었다.
- 에러는 "circuit_breaking_exception", [parent] Data too large
- 원인은 heap 메모리가 부족이고 elasticsearch에서 사용하는 heap 메모리를 늘려서 해결하였다.

## "circuit_breaking_exception", [parent] Data too large 에러의 원인
- GET _cat/indices 요청의 결과가 elasticsearch에서 지정한 heap 메모리보다 커서 발생하는 에러이다.
- elasticsearch 조회 요청을 작은 단위로 한다면 이 에러는 나지 않는다.
- GET _cat/indices은 elasticsearch에 생성된 모든 인덱스 개요를 조회하는 명령이다.
- kibana에서 최초 실행 시 GET _cat/indices 요청을 하므로 반드시 수정해야했다.

## 해결방안 1. heap 메모리 사용률 올리기
- 설명
    - 기본 사용률은 50%인데 이를 올리면 해결할 수 있다.
    - !주의 - 너무 많이 올릴 경우, elasticsearch가 다운된다.
- 명령
    - ```bash
        # heap 메모리 사용률 50% -> 60%
        curl -XPUT \
            "http://localhost:9200/_cluster/settings" \
            -H 'Content-Type: application/json' -d'
            {
                "transient" : {
                    "indices.breaker.total.limit" : "60%"
                }
            }
        ```

## 해결방안 2. elasticsearch heap 메모리 크기를 올리기
- 설명
    - elasticsearch의 jvm heap 메모리를 올린다.
    - 기본 값은 1GB
- 방법
    - /usr/share/elasticsearch/config/jvm.options.d/ 하위에 custom 파일을 하나 만든다.
    - 1GB를 2GB로 올리기 위해 아래와 같이 작성한다.
    - ```bash
        # 최소 2gb 사용
        -Xms2g
        # 최대 2gb 사용
        -Xmx2g
        ```
- 주의
    - heap 메모리는 32GB 미만으로 설정 가능하다.
    - heap의 최소값은 물리적 RAM의 50% 이상으로 설정하지 않는 것이 좋다고 한다.
    - heap 메모리를 크게 잡으면 캐싱하기에는 유리하나, Garbage collect 시간이 늘어나서 시스템 성능 저하에 원인이 될 수 있다고 한다.
    - 참고 - [갓.바.조.아 | ElasticSearch Heap 사이즈 설정](https://springboot.cloud/17){:target="_blank"}

## 참고
- [엘라스틱서치 공식 홈페이지 Set JVM options](https://www.elastic.co/guide/en/elasticsearch/reference/current/advanced-configuration.html){:target="_blank"}
- [갓.바.조.아 | ElasticSearch Heap 사이즈 설정](https://springboot.cloud/17){:target="_blank"}
