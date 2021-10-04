---
title: 우분투에 ELK(Elasticsearch, Logstash, Kibana) 구축해보기
date: 2021-10-04 17:25:00 +0900
categories: [Elastic]
tags: [elk, elasticsearch, logstash, kibana] # TAG names should always be lowercase
---

## 개요
- 우분투 환경에 ELK를 구축하여 후암동의 평균 공동주택 공시가격을 구해본다.

## 환경 설명
- Windows 10 pro(실제환경)에서 docker를 이용하여 우분투로 가상환경을 구축한다.
- 실제환경의 ~/Document/elasticsearch 디렉토리와 가상환경 /home/elasticsearch를 연결한다.
- Elasticsearch, Logstash, Kibana는 하나의 가상환경에 설치 및 실행한다.

## 우분투 환경 구축
- ```bash
  # 실제환경의 ~/Document/elasticsearch에서 실행
  > docker run --name realty_lab -v %cd%:/home/elasticsearch -p 5601:5601 --memory=4g  -it -d ubuntu:20.04

  # 컨테이너 쉘로 접속
  > docker exec -it realty_lab bash

  # 가상환경 안에서 apt 저장소를 카카오 미러 서버로 변경
  $ cp /etc/apt/sources.list sources.list_backup
  $ sed -i -e 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list
  $ sed -i -e 's/security.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list

  # 의존성 패키지 업데이트
  $ apt-get update && apt-get upgrade -y

  # wget, curl, vim, net-tools 설치
  $ apt-get install wget curl vim net-tools -y

  # elasticsearch라는 이름의 유저 생성
  $ adduser elasticsearch

  # elasticsearch로 로그인
  $ su elasticsearch
  ```

## 실험 데이터 다운로드
- 개요
  - 국토교통부 공동주택 공시가격 csv 파일을 받아 실험한다([다운로드 링크](https://www.data.go.kr/data/3073746/fileData.do)).  
  - 단순 wget으로는 받기 어려우므로 브라우저를 켜고 직접 다운로드한다.
  - 다운로드 한 zip 파일을 ~/Document/elasticsearch에 압축풀기한다.
  - 파일 인코딩이 euc-kr이므로 utf-8로 변환 후 사용한다.
  - 1GB가 넘는 큰 텍스트 파일이다. 따라서 후암동 주소만 분리하여 실험한다.
- 데이터 정제
  - ```bash
    # 파일명을 MULTI_OWNER_HSE_PRICE.csv로 변환
    $ mv [공동주택 공시가격 unzip 파일명] MULTI_OWNER_HSE_PRICES.csv

    # 파일 인코딩을 utf8로 변환
    $ iconv -f cp949 utf-8 MULTI_OWNER_HSE_PRICE.csv MULTI_OWNER_HSE_PRICE_iconv.csv

    # 확인
    $ head MULTI_OWNER_HSE_PRICE_iconv.csv

    # 이상 없으면 원본 지우고 utf8만 남김
    $ rm MULTI_OWNER_HSE_PRICE.csv
    $ mv MULTI_OWNER_HSE_PRICE_iconv.csv MULTI_OWNER_HSE_PRICE.csv
    ```
- 시스템 언어 설정
  - ```bash
    # 유저가 elasticsearch이라면 관리자 권한 획득
    $ exit

    # 참고(https://ti.bqbro.com/21)
    # 기본 시스템 언어로는 쉘에서 한글 입력을 할 수 없다.
    # 따라서 기본 시스템 언어를 utf8로 변경해준다.
    $ apt-get install language-pack-ko
    $ apt-get install language-pack-ko-base

    $ echo "LANG=\"ko_KR.UTF-8\"" >> /etc/environment
    $ echo "LANG=\"ko_KR.EUC-KR\"" >> /etc/environment
    $ echo "LANGUAGE=\"ko_KR:ko:en_GB:en\"" >> /etc/environment

    $ echo "LANG=\"ko_KR.UTF-8\"" >> /etc/default/locale
    $ echo "LANG=\"ko_KR.EUC-KR\"" >> /etc/default/locale
    $ echo "LANGUAGE=\"ko_KR:ko:en_GB:en\"" >> /etc/default/locale

    $ echo "LANG=\"ko_KR.UTF-8\"" >> /etc/profile

    # 우분투 쉘에서 이탈하여 컨테이너 재시작
    $ exit
    > docker stop realty_lab
    > docker start realty_lab

    # 우분투 쉘에 재접근
    > docker exec -it realty_lab bash
    ```
- 후암동 주소만 분리
  - ```bash
    # elasticsearch 로그인
    $ su elasticsearch
    $ cd

    # 후암동 주소만 분리
    $ cat MULTI_OWNER_HSE_PRICES.csv | grep "후암동" > test_data.csv
    ```

## 엘라스틱서치 환경 구축
- ```bash
  # 엘라스틱서치 다운로드(최신버전 링크: https://www.elastic.co/kr/downloads/elasticsearch)
  # 여기서는 7.14.1 버전 사용
  $ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.14.1-linux-x86_64.tar.gz

  # 압축 해제
  $ tar -xvzf elasticsearch*.gz

  # 압축 파일 삭제
  $ rm elastic*.gz

  # nori_tokenizer 설치
  $ ./elasticsearch-7.14.1/bin/elasticsearch-plugin install analysis-nori

  # elasticsearch 데몬으로 실행
  $ ./elasticsearch-7.14.1/bin/elasticsearch -d

  # 실행확인(한 5분 후에 로드가 완료된다.)
  $ curl localhost:9200
  # 결과
  {
    "name" : "b2ca57c11320",
    "cluster_name" : "elasticsearch",
    "cluster_uuid" : "b7508aRxQFSGTG_kRN2AwA",
    "version" : {
      "number" : "7.14.1",
      "build_flavor" : "default",
      "build_type" : "tar",
      "build_hash" : "66b55ebfa59c92c15db3f69a335d500018b3331e",
      "build_date" : "2021-08-26T09:01:05.390870785Z",
      "build_snapshot" : false,
      "lucene_version" : "8.9.0",
      "minimum_wire_compatibility_version" : "6.8.0",
      "minimum_index_compatibility_version" : "6.0.0-beta1"
    },
    "tagline" : "You Know, for Search"
  }
  ```

## 로그스태시 환경 구축
- ```bash
  # 로그스태시 OSS Only 다운로드(최신버전 링크: https://www.elastic.co/kr/downloads/logstash-oss)
  # 여기서는 7.14.1 버전 사용
  $ wget https://artifacts.elastic.co/downloads/logstash/logstash-oss-7.14.1-linux-x86_64.tar.gz

  # 압축 해제
  $ tar -xvzf logstash*.gz

  # 압축 파일 삭제
  $ rm logstash*.gz

  # jdk를 받아야한다. 아래는 elasticsearch와 logstash에서 지원하는 jdk 버전 목록.
  # https://www.elastic.co/kr/support/matrix#matrix_jvm
  # elasticsearch와 logstash를 같은 환경에 설치하였다면 두 버전 모두를 고려한 jdk를 받아야한다.
  # openjdk 11을 받으면 에러 없이 처리된다.
  
  # 유저가 elasticsearch이라면 관리자 권한 획득
  $ exit

  # openjdk 설치
  $ apt-get install openjdk-11-jdk -y

  # openjdk 설치 확인
  $ java -version
  $ javac -version

  # ~/.bashrc 마지막 줄에 자바홈 추가
  echo "# JAVA HOME directory setup" >> ~/.bashrc
  echo "export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64" >> ~/.bashrc
  echo "export PATH=\"$PATH:$JAVA_HOME/bin\"" >> ~/.bashrc
  
  # 환경 변수 변경 내역 반영(Reload ~/.bashrc)
  $ source ~/.bashrc

  # Logstash input plugin 설치
  $ ./logstash-7.14.1/bin/logstash-plugin install logstash-input-jdbc
  ```

## 인덱스 설정 파일 작성 및 인덱스 생성
- 개요
  - 공동주택 공시가격 구조에 맞게 인덱스를 설정해준다.
  - 한국어 파싱을 위해 tokenizer로 nori_tokenizer를 사용한다.
  - fulltext search를 해야하는 컬럼만 type을 text로 지정해준다.
  - 타입에 대한 자세한 내용은 아래 링크를 참고한다.  
    - https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html	
- 인덱스 파일 작성
  - ```bash
    # multi_owner_hse_price_index.json
    {
      "settings": {
        "analysis": {
          "analyzer": {
            "my_analyzer": {
              "type": "custom",
              "tokenizer": "nori_tokenizer",
              "filter": "lowercase"
            }
          }
        }
      },
      "mappings": {
        "properties": {
          "기준연도": {
            "type": "short"
          },
          "기준월": {
            "type": "byte"
          },
          "법정동코드": {
            "type": "integer"
          },
          "도로명주소": {
            "type": "text"
          },
          "시도": {
            "type": "text"
          },
          "시군구": {
            "type": "keyword"
          },
          "읍면": {
            "type": "keyword"
          },
          "동리": {
            "type": "keyword"
          },
          "특수지코드": {
            "type": "interger"
          },
          "본번": {
            "type": "integer"
          },
          "부번": {
            "type": "integer"
          },
          "특수지명": {
            "type": "keyword"
          },
          "단지명": {
            "type": "text"
          },
          "동명": {
            "type": "text"
          },
          "호명": {
            "type": "text"
          },
          "전용면적": {
            "type": "double"
          },
          "공시가격": {
            "type": "double"
          },
          "단지코드": {
            "type": "keyword"
          },
          "동코드": {
            "type": "keyword"
          },
          "호코드": {
            "type": "keyword"
          }
        }
      }
    }
    ```
- elasticsearch에 realty라는 이름의 인덱스 생성
  - ```bash
    $ curl -XPUT http://localhost:9200/realty -d @multi_owner_hse_price_index.json -H 'Content-Type: application/json'
    ```
- 생성 확인
  - ```bash
    $ curl localhost:9200/realty
    # 결과
    {"realty":{"aliases":{},"mappings":{"properties":{"@timestamp":{"type":"date"},"@version":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"host":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"message":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"path":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"공시가격":{"type":"double"},"기준연도":{"type":"short"},"기준월":{"type":"byte"},"단지 명":{"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}},"단 지명":{"type":"text"},"단지코드":{"type":"keyword"},"도로명주소":{"type":"text"},"동리":{"type":"keyword"},"동명":{"type":"text"},"동코드":{"type":"keyword"},"법정동코드":{"type":"integer"},"본번":{"type":"integer"},"부번":{"type":"integer"},"시군구":{"type":"keyword"},"시도":{"type":"text"},"읍면":{"type":"keyword"},"전용면적":{"type":"double"},"특수지명":{"type":"keyword"},"특수지코드":{"type":"integer"},"호명":{"type":"text"},"호코드":{"type":"keyword"}}},"settings":{"index":{"routing":{"allocation":{"include":{"_tier_preference":"data_content"}}},"number_of_shards":"1","provided_name":"realty","creation_date":"1631061617189","analysis":{"analyzer":{"my_analyzer":{"filter":"lowercase","type":"custom","tokenizer":"nori_tokenizer"}}},"number_of_replicas":"1","uuid":"dyj6mpZIRmqxdgqsIz6QvQ","version":{"created":"7140199"}}}}}
    ```

## 로그스태시 실행
- 로그스태시 config 파일 작성
  - ```bash
    # logstash.conf
    input {
      file {
        path => "/home/elasticsearch/test_data.csv"
        start_position => beginning
        sincedb_path => "/dev/null"
      }

    }

    filter {
      csv {
        columns => [
          "기준연도",
          "기준월",
          "법정동코드",
          "도로명주소",
          "시도",
          "시군구",
          "읍면",
          "동리",
          "특수지코드",
          "본번",
          "부번",
          "특수지명",
          "단지명",
          "동명",
          "호명",
          "전용면적",
          "공시가격",
          "단지코드",
          "동코드",
          "호코드"
        ]
        separator => ","
      }
    }

    output {
      stdout {
        codec => rubydebug
      }

      elasticsearch {
        action => "index"
        hosts => ["localhost:9200"]
        index => "realty"
      }
    }
    ```
- logstash 실행
  - ```bash 
    $ ./logstash-7.14.1/bin/logstash -f logstash.conf
    # 결과
    ...
    {
           "시군구" => "용산구",
            "시도" => "서울특별시",
            "부번" => "0",
          "공시가격" => "910000000",
           "동코드" => "3",
          "path" => "/home/elasticsearch/test_data.csv",
            "읍면" => "",
          "host" => "b2ca57c11320",
          "기준연도" => "2020",
      "@version" => "1",
          "단지코드" => "256703",
            "동명" => "103",
           "호코드" => "9",
            "동리" => "후암동",
            "본번" => "458",
    "@timestamp" => 2021-10-04T07:43:54.413Z,
           "단지명" => "브라운스톤남산",
          "전용면적" => "166.56",
         "법정동코드" => "1117010100",
           "기준월" => "1",
         "특수지코드" => "0",
       "message" => "\"2020\",\"1\",\"1117010100\",\"서울특별시 용산구 후암로 65\",\"서울특별시\",\"용산구\",\"\",\" 후암동\",\"0\",\"458\",\"0\",\"\",\"브라운스톤남산\",\"103\",\"301\",\"166.56\",\"910000000\",\"256703\",\"3\",\"9\"",
          "특수지명" => "",
         "도로명주소" => "서울특별시 용산구 후암로 65",
            "호명" => "301"
    }
    ...
    ```
## Elasticsearch로 후암동의 평균 공동주택 공시가격 계산
- ```bash
  $ curl http://localhost:9200/realty/_search?pretty -d '{
    "size": 0,
    "aggs": {
      "avg_price": {
        "avg": {
          "field": "공시가격"
        }
      }
    },
    "query": {
      "bool": {
        "must": [
          { 
            "match": {
              "시도": "서울특별시"
            }
          },
          { 
            "match": {
              "시군구": "용산구"
            }
          },
          {  
            "match": {
              "동리": "후암동"
            }
          }
        ]
      }
    }
  }' -H 'Content-Type: application/json'
  # 결과
  {
    "took" : 156,
    "timed_out" : false,
    "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
    },
    "hits" : {
      "total" : {
        "value" : 10000,
        "relation" : "gte"
      },
      "max_score" : null,
      "hits" : [ ]
    },
    "aggregations" : {
      "avg_price" : {
        "value" : 2.801156181619256E8
      }
    }
  }
  ```
- 평균 공시가격은 약 280,115,618원

## 키바나 설치 및 실행
- 개요
  - 위의 예시처럼 직접 elasticsearch에 쿼리를 날려 검색할 수도 있지만  
    키바나를 통해 GUI로 쉽게 검색 할 수도 있다.
- 키바나 설치 및 실행
  - ```bash
    # 새로운 쉘 연결 생성
    > docker exec -it realty_lab bash

    # elasticsearch로 로그인
    $ su elasticsearch
    $ cd

    # 키바나 설치(링크: https://www.elastic.co/kr/downloads/kibana)
    # 여기서는 7.14.1 버전
    $ wget https://artifacts.elastic.co/downloads/kibana/kibana-7.14.1-linux-x86_64.tar.gz

    # 압축 해제
    $ tar -xvzf kibana*.gz

    # 압축 파일 삭제
    $ rm kibana*.gz

    # 키바나 실행(한 7분 후 로드 완료)
    $ ./kibana-7.14.1-linux-x86_64/bin/kibana -H 0.0.0.0 -p 5601 -e http://localhost:9200
    # 결과
    log   [07:53:37.182] [info][plugins-service] Plugin \"metricsEntities\" is disabled.
    log   [07:53:37.353] [warning][config][deprecation] plugins.scanDirs is deprecated and is no longer used
    log   [07:53:37.355] [warning][config][deprecation] Config key [monitoring.cluster_alerts.email_notifications.email_address] will be required for email notifications to work in 8.0.\"
    log   [07:53:37.359] [warning][config][deprecation] \"xpack.reporting.roles\" is deprecated. Granting reporting privilege through a \"reporting_user\" role will not be supported starting in 8.0. Please set \"xpack.reporting.roles.enabled\" to \"false\" and grant reporting privileges to users using Kibana application privileges **Management > Security > Roles**.
    log   [07:53:37.534] [info][server][NotReady][http] http server running at http://0.0.0.0:5601

    # 브라우저를 켜고 localhost:5601로 접속
    # 좌상단 햄버거 메뉴 클릭 
    #  > Analytics 하위의 Discover 클릭 
    #  > 인덱스 패턴이 없다면 realty*로 인덱스 패턴 생성 
    #  > 좌측 Field fiters 하위 Available fields에서 공시가격 클릭 후 Visualize 클릭
    #  > 우측 Vertical axis의 Median of 공시가격을 클릭하여 Average로 변경함
    ```
- 키바나 검색 결과
    - <a href="/assets/img/2021-10-04-elk-experiment/00-kibana-avg.jpg" target="_blank"><img src="/assets/img/2021-10-04-elk-experiment/00-kibana-avg.jpg" width="100%"></a> 

## 실험 소감
- 생각보다 ELK 전체를 구축했을때 요구 성능이 높아 당황했다(메모리 최소 4gb 필요).
- 만약 전체 데이터로 실험했다면 훨씬 많은 메모리가 필요했을 것이라고 추측한다.
- 실제로는 엘라스틱서치가 죽는 것을 방지하기 위해 3개 이상의 노드를 홀수 개로 생성한다.  
  이 경우 필요 메모리는 더욱 늘어날 것이다.
- 엘라스틱서치는 다양한 aggregation function을 제공한다.  
  검색에 대하여 더 공부하면 지금보다 신기한 계산 결과도 얻을 수 있을 것 같다.

## 참고
- [도커 컨테이너 메모리 설정](https://simpleisit.tistory.com/143)
- [키바나 사용 예시](https://epicarts.tistory.com/75)
- [우분투 한글 설정](https://ti.bqbro.com/21)

