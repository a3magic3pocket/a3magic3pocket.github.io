---
title: unassigned shards 발생 원인 및 해결방법
date: 2022-06-13 19:00:00 +0900
categories: [ELK]
tags: [elk, elasticsearch, unassigned-shards]    # TAG names should always be lowercase
---
## 개요
- kibana에서 elasticsearch status를 확인해보니 red가 되었음을 확인하였다.
- status red란 primary shards 중 unassigned 된 것이 있다는 의미이다.
- 원인은 node 수에 비해 지나치게 많은 인덱스가 생성되어 unassigned 된 것이었다.
- node 수를 늘리거나 불필요한 인덱스를 제거하여 해결할 수 있다.
- replica 수 변경으로 샤드 할당에 실패한 것일 수도 있으니 replica 수도 조정한다.

## elasticsearch 상태 확인
- ```bash
    # 결과에서 unassinged_shards를 확인한다.
    curl -XGET "localhost:9200/_cluster/health?pretty"

    # unassigned 샤드 상태 조회(assigned shards 들을 확인할 수 있다.)
    curl "https://localhost:9200/_cat/shards" | grep "UNASSIGNED"
    ```

## 문제 샤드 발생 원인 탐색
- ```bash
    curl -XGET "localhost:9200/_cluster/allocation/explain?pretty"
    ```

## 대표적인 문제 샤드 발생 원인
- thorttle
    - 에러 원문
        - "explanation" : "reached the limit of ongoing initial primary recoveries [4], cluster setting [cluster.routing.allocation.node_initial_primaries_recoveries=4]"
    - 현상
        - 노드 수 대비 너무 많은 샤드가 생성된 것
        - elasticsearch 재시작 시 일시적으로 걸리는 경우가 있다.
    - 해결방법
        - 노드 수를 늘리거나 인덱스 수를 줄인다.
- same shard
    - 에러 원문
        - "explanation" : "the shard cannot be allocated to the same node on which a copy of the shard already exists [[my-index-2022.06.13][4], node[OnXt_GFVQzeLNXSDFJKLSDNF], [P], s[STARTED], a[id=4daKbROAJDKFHSDJKFBKJS]]"
    - 현상
        - replica 수가 node 수보다 많아 같은 node에 같은 샤드가 할당된 경우
    - 해결방법
        - replica 수를 0으로 바꾸고, 모든 샤드를 재할당한 후 다시 replicas 수를 조정한다.
        - 명령
            - ```bash
                # replica 수 0으로 변경
                curl -XPUT \
                    "http://localhost:9200/_settings" \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "index" : {
                            "number_of_replicas" : 0
                        }
                    }'
                
                # replica 수 변경되었는지 확인
                curl "localhost:9200/_cluster/stats?pretty" | grep "replication"
                ```
                
## 조치 후 샤드 재할당
- ```bash
    # 자동 할당 설정 활성화
    curl -XPUT 'http://localhost:9200/_cluster/settings' \
        -H 'Content-Type: application/json' \
        -d '{
            "transient" : {
                "cluster.routing.allocation.enable" : "all"
            }
        }'

    # 샤드 재할당
    curl -XPOST "http://localhost:9200/_cluster/reroute?retry_failed"

    # unassigned shards가 존재하는지 확인
    curl "https://localhost:9200/_cat/shards" | grep "UNASSIGNED"
    ```

## 참고
- [How to resolve unassigned shards in Elasticsearch](https://www.datadoghq.com/blog/elasticsearch-unassigned-shards/){:target="_blank"}
- [인덱스와 샤드](https://esbook.kimjmin.net/03-cluster/3.2-index-and-shards){:target="_blank"}
- [stackoverflow - ElasticSearch: Unassigned Shards, how to fix?](https://stackoverflow.com/questions/19967472/elasticsearch-unassigned-shards-how-to-fix){:target="_blank"}
