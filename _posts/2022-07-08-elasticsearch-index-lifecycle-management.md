---
title: ILM을 사용하여 일정 기간마다 자동으로 인덱스 삭제하기
date: 2022-07-08 19:00:00 +0900
categories: [ELK]
tags: [elk, elasticsearch, ilm, index-lifecycle-management]    # TAG names should always be lowercase
---
## 개요
- 엘라스틱서치에서 발행하는 많은 에러는  
    많은 경우 지정된 노드 수보다 더 많은 양의 문서를 생성해서 생긴 것이었다.  
- 따라서 사용을 많이 하지 않는 문서는 주기적으로 지워주는 것이 필수적이다.
- ILM을 사용하면 조건을 지정하여, 해당 조건 충족 시 자동으로 인덱스를 삭제해준다.
- 여기서는 키바나를 사용한 방법을 기록한다.

## ILM에서 정책 생성
1. 키바나 접속 > (좌상단) 햄버거 메뉴 클릭 > (Management 하위의) Stack Management 클릭 > Index Lifecycle Policies 클릭
    - <a href="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/00-index-lifecycle-policies.jpg" target="_blank"><img src="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/00-index-lifecycle-policies.jpg" width="70%"></a>
2. Create policy 클릭
3. Create policy
    - 정책 이름 입력
    - Hot phase
        - <a href="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/01-hot-phase.jpg" target="_blank"><img src="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/01-hot-phase.jpg" width="70%"></a>
        - "Delete date after this phase" 클릭
        - Advanced settings 클릭 > Use recommended defaults 토글 선택 후 원하는 조건 선택
            - 날짜(days)와 용량(gigabytes)로 조건을 선택할 수 있음
            - 해당 조건이 충족되면 다음 단계로 롤오버 됨
    - Delete phase
        - <a href="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/02-delete-phase.jpg" target="_blank"><img src="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/02-delete-phase.jpg" width="70%"></a>
        - Move data into phase when: 에서 Hot phase에서 롤오버 된 이후 몇 일 후에 삭제하겠다고 지정 가능
    - Save Policy 버튼 클릭

## 인덱스 템플릿에 생성한 ILM 정책 등록
- 개요
    - 인덱스 템플릿을 생성할때 생성한 ILM을 등록해주면된다.
    - 여기서 등록한 템플릿은 기존에 생성된 인덱스가 아닌 새로 생성될 인덱스들에 적용된다.
- 방법
    1. 키바나 접속 > (좌상단) 햄버거 메뉴 클릭 > (Management 하위의) Stack Management 클릭 > Index Management 클릭
        - <a href="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/03-index-management.jpg" target="_blank"><img src="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/03-index-management.jpg" width="70%"></a>
    2. (제목 밑 서브 메뉴에서) Index Templates 클릭
    3. Create template 버튼 클릭
    4. Create template
        - Logistics
            - 인덱스명, 인덱스 패턴 지정
                - 인덱스 패턴을 my-apache-access-*로 지정하였다면  
                  새로 생성될 인덱스 중 인덱스명이 my-apache-access-로 시작한다면 지금 작성 중인 템플릿을 따르게 된다.
                - ex) my-apahce-access-20220708.log, my-apahce-access-20210912.log
            - <a href="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/04-index-pattern.jpg" target="_blank"><img src="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/04-index-pattern.jpg" width="70%"></a>                
        - Component templates
            - 넘긴다
        - Index settings
            - 위에서 생성한 ILM 정책을 등록해준다.
                - ```bash
                        {
                            "index": {
                                "lifecycle": {
                                "name": "test-my-policy"
                                }
                            }
                        }                
                    ```
        - Mappings
            - 인덱스의 필드 타입을 지정  
                - 모든 필드의 기본 타입은 text이다.
                - text는 fulltext search하는 필드이기 때문에   
                  연산 비용이 비싼 필드이다.
                - 따라서 numeric이나 keyword를 상황에 맞게 설정해준다면   
                  더 효율적으로 검색할 수 있게 된다.
                - 지금은 ILM 정책을 등록할 것이기에 넘긴다.
        - Aliases
            - 넘긴다
- 확인
    - 인덱스 템플릿을 클릭해보면 정상적으로 등록되었는지 확인할 수 있다.
    - <a href="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/05-check-index-template.jpg" target="_blank"><img src="/assets/img/2022-07-08-elasticsearch-index-lifecycle-management/05-check-index-template.jpg" width="70%"></a>                

## 덧붙이는 말
- 현재는 키바나에서 설정하는 방법만 적었지만    
  elasticsearch에서 인덱스 생성 시 붙이는 방법,  
  logstash에서 인덱스 템플릿 설정 시 붙여지도록 하는 방법,  
  직접 elasticsearch에 명령을 내려서 설정하는 방법 등  
  여러 방법이 존재한다.
- 아래 참고 문서를 보면 다양한 방법도 함께 기재되어 있으니 참고하면 좋다.

## 참고
- [Configure a lifecycle policy](https://www.elastic.co/guide/en/elasticsearch/reference/current/set-up-lifecycle-policy.html){:target="_blank"}
- [인덱스 수명 주기 관리를 통해 Hot-Warm-Cold 아키텍처 구현](https://www.elastic.co/kr/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management){:target="_blank"}
