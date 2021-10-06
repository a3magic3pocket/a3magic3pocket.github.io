---
title: docker container 환경에서 ELK(Elasticsearch, Logstash, Kibana)로 로깅해보기
date: 2021-10-05 17:25:00 +0900
categories: [Elastic]
tags: [docker, container, filebeat, elk, elasticsearch, logstash, kibana] # TAG names should always be lowercase
---

## 개요
- application server 및 ELK를 docker container로 올려두고 로깅해본다.

## 구조
- 개요
  - api, front, logstash, elasticsearch, kibana 컨테이너를 띄워 운영한다.
    - <a href="/assets/img/2021-10-05-elk-container-example/00-diagram.jpg" target="_blank"><img src="/assets/img/2021-10-05-elk-container-example/00-diagram.jpg" width="70%"></a> 
- 상세설명
  - api
    - golang, gonic-gin framework를 이용하여 현재 한국 시간을 리턴하는 api를 배포하는 서버
    - api 서버 내에 filebeat를 설치하여 gin.log를 logstash로 일정 시간마다 전송한다.
  - front
    - nginx 웹서버를 실행시켜 main.html 페이지를 배포하는 서버
    - main.html에서는 현재 한국 시간을 얻어오는 api를 api 서버로부터 호출한다.
    - front 서버 내에 filebeat를 설치하여 access.log와 error.log를 logstash로 일정 시간마다 전송한다.
  - logstash
    - api 및 front의 filebeat로부터 데이터를 전송 받으면 지정된 filter 처리를 한 후  
      elasticsearch로 전송한다.
  - elasticsearch
    - logstash에서 받은 데이터를 재색인한다.
    - 이를 통해 쉽게 로그 검색을 할 수 있게 된다.
  - kibana
    - GUI로 elasticsearch의 많은 기능을 이용할 수 있게 된다.
    - kibana query를 이용하여 간단하게 로그 검색할 수 있다.
- 덧붙이는 말
  - 각 application server에 filebeat를 직접 설치하는 대신  
    filebeat container에서 docker logs를 가져와 logstash로 보내는 방법도 있다. [참고](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-docker.html)

## 예제 소스코드
- [예제 소스코드](https://github.com/a3magic3pocket/es-container-example)

## docker network 생성
- 개요
  - 컨테이너 간 통신을 하려면 docker network를 생성하고  
    해당 컨테이너들을 생성한 docker network에 연결해야한다.
  - docker network에 연결된 컨테이너들은 사설 IP(private IP)를 받게 된다.
  - 컨테이너 간 통신을 할때 사설 IP를 사용하거나 컨테이너 이름을 hostname으로 쓸 수 있다.
- 실행
  - ```bash
    # es-net이라는 docker network 생성
    $ docker network create es-net

    # 연결된 컨테이너 확인(최초 생성 시에는 아무 것도 없다)
    $ docker network inspect es-net
    # 결과
    [
        {
            "Name": "es-net",
            "Id": "2af31f476054b06841ffbb72903bc07793e860d0b677e886990ec24c4512637f",
            "Created": "2021-10-05T06:53:46.1699186Z",
            "Scope": "local",
            "Driver": "bridge",
            "EnableIPv6": false,
            "IPAM": {
                "Driver": "default",
                "Options": {},
                "Config": [
                    {
                        "Subnet": "172.18.0.0/16",
                        "Gateway": "172.18.0.1"
                    }
                ]
            },
            "Internal": false,
            "Attachable": false,
            "Ingress": false,
            "ConfigFrom": {
                "Network": ""
            },
            "ConfigOnly": false,
            "Containers": {},
            "Options": {},
            "Labels": {}
        }
    ]
    ```

## Elasticsearch
- 컨테이너 생성 설명
  - 9200번, 9300번 포트로 컨테이너와 실제환경(도커를 돌리고 있는 윈도우)를 연결한다.
  - docker network, es-net에 연결한다.
  - -e 명령을 통해 컨테이너 내부의 환경변수를 설정할 수 있다.
  - elasticsearch는 환경변수를 통해 몇 가지 설정을 할 수 있다.
    - action.auto_create_index:true
      - 새로운 데이터가 들어왔을 때 인덱스를 자동으로 생성하겠다는 설정
    - discovery.type=single-node
      - 싱글 노드만 쓰겠다는 설정
      - 다중 노드를 쓴다면 slave 노드를 93xx 포트에 연결한다.
  - --rm을 지정하면 컨테이너 종료 시 컨테이너를 삭제한다.
  - elasticsearch은 7.15.0 버전을 사용한다.
  - !주의 - 컨테이너 이름을 정할 때 "\_"나 "."을 사용하지 말자  
    logstash에서 hosts를 지정할 때 "\_"나 "."가 포함되면 에러를 내보닌다.
- 실행
  - ```bash
    > docker run \
     --name es-c-es \
     --network es-net \
     --rm \
     -p 9200:9200 -p 9300:9300 \
     -e "action.auto_create_index:true" \
     -e "discovery.type=single-node" \ 
     docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    ```
- 실행 확인
  - <a href="/assets/img/2021-10-05-elk-container-example/01-elasticsearch.jpg" target="_blank"><img src="/assets/img/2021-10-05-elk-container-example/01-elasticsearch.jpg" width="70%"></a> 

## Logstash
- 설정
  - logstash 설정은 /usr/share/logstash/pipeline/(이하 pipeline) 하위에서 할 수 있다.
    - logstash 도커 컨테이너는 pipeline 하위의 logstash로 시작하는 파일을 설정파일로 인식한다.
    - pipeline 하위의 patterns 디렉토리에 logstash 설정에서 grok 정규식 사전 파일들을 저장한다.
    - 실제 환경에서 설정파일을 생성한 뒤 컨테이너의 pipeline 디렉토리에 mount 하는 방식으로 설정한다.
- 설정파일
  - patterns
    - config에서 log를 파싱할 때 grok 정규식을 사용하는데,  
      이때 사용할 패턴을 미리 파일로 만들어둔다.
    - ```bash
      # nginx
      NGINXACCESS %{IPORHOST:[nginx][access][remote_ip]} - %{DATA:[nginx][access][user_name]} \[%{HTTPDATE:[nginx][access][time]}\] \"%{WORD:[nginx][access][method]} %{DATA:[nginx][access][url]} HTTP/%{NUMBER:[nginx][access][http_version]}\" %{NUMBER:[nginx][access][response_code]} %{NUMBER:[nginx][access][body_sent][bytes]} \"%{DATA:[nginx][access][referrer]}\" \"%{DATA:[nginx][access][agent]}\"
      NGINXERROR %{DATA:[nginx][error][time]} \[%{DATA:[nginx][error][level]}\] %{NUMBER:[nginx][error][pid]}#%{NUMBER:[nginx][error][tid]}: (\*%{NUMBER:[nginx][error][connection_id]} )?%{GREEDYDATA:[nginx][error][message]}

      # gin
      GIN_TIMESTAMP %{YEAR}/%{MONTHNUM}/%{MONTHDAY} - %{HOUR}:%{MINUTE}:%{SECOND}
      GINLOG \[%{DATA:[gin][type]}\] %{GIN_TIMESTAMP:[gin][time]} \| %{NUMBER:[gin][response_status]} \| \s*%{DATA:[gin][response_time]}\s* \| \s*%{IPORHOST:[gin][remote_ip]}\s* \| \s*%{DATA:[gin][method]}\s* \"%{DATA:[gin][url]}\"
      ```
  - logstash.conf
    - ```bash
      # 5044 포트로 filebeat에서 전송하는 데이터를 받겠다는 의미
      input {
          beats {
              port => 5044
          }
      }

      # 전송 받은 비정형 데이터를 filter에서 정형화 시킨다.
      # 사전에서 정의한 grok 패턴을 통해 비정헝 데이터를 파싱하고
      # mutate, date 등의 명령을 통해 정제한다.
      filter {
        if "nginx-access" in [tags] {
          grok {
            patterns_dir => ["/usr/share/logstash/pipeline/patterns"]
            match => {"message" => ["%{NGINXACCESS}"]}
            remove_field => "message"
          }
          mutate {
            add_field => { "read_timestamp" => "%{@timestamp}" }
          }
          date {
            match => [ "[nginx][access][time]", "dd/MMM/YYYY:H:m:s Z" ]
            remove_field => "[nginx][access][time]"
          }
          useragent {
            source => "[nginx][access][agent]"
            target => "[nginx][access][user_agent]"
            remove_field => "[nginx][access][agent]"
          }
        }
        else if "nginx-error" in [tags] {
          grok {
            patterns_dir => ["/usr/share/logstash/pipeline/patterns"]
            match => { "message" => ["%{NGINXERROR}"] }
            remove_field => "message"
          }
          mutate {
            rename => { "@timestamp" => "read_timestamp" }
          }
          date {
            match => [ "[nginx][error][time]", "YYYY/MM/dd H:m:s" ]
            remove_field => "[nginx][error][time]"
          }
        }
        else if "gin" in [tags] {
          grok {
            patterns_dir => ["/usr/share/logstash/pipeline/patterns"]
            match => {"message" => ["%{GINLOG}"]}
            remove_field => "message"
            remove_field => "[gin][type]"
          }
          mutate {
            add_field => { "read_timestamp" => "%{@timestamp}" }
          }
          date {
            match => [ "[gin][time]", "yyyy/MM/dd - HH:mm:ss" ]
            remove_field => "[gin][time]"
          }
        }
      }

      # stdout을 추가함으로써 처리과정을 사용자가 볼 수 있다.
      # ES_IP 환경변수에 담긴 hostname을 통해 elasticsearch로 정제한 데이터를 송부한다.
      # 매일 인덱스를 새로 생성하는 것을 알 수 있다.
      output {
        stdout {
          codec => rubydebug
        }
        elasticsearch {
          action => "index"
          hosts => ["${ES_IP:=localhost}:9200"]
          manage_template => false
          index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        }
      }
      ```
- 컨테이너 생성 설명
  - docker network, es-net에 연결한다.
  - --rm을 지정하면 컨테이너 종료 시 컨테이너를 삭제한다.
  - -e를 통해 elasticsearch 컨테이너 이름을 logstash 컨테이너의 ES_IP 환경변수로 설정한다.
  - -v를 통해 실제환경의 config/logstash 디렉토리를  
    logstash 컨테이너의 /usr/share/logstash/pipeline 디렉토리에 mount한다.
  - logstash 버전은 7.15.0를 사용한다.
- 실행
  - ```bash
    > docker run \
     --name es-c-logstash \
     --rm \
     --network es-net \
     -e ES_IP=es-c-es \
     -v %cd%/config/logstash/:/usr/share/logstash/pipeline/ \
     docker.elastic.co/logstash/logstash:7.15.0
    ```

## API
- 개요
  - golang, gonic-gin framework를 이용하여 간단한 API를 생성한다.
  - 현재 한국 시간을 알려주는 API로 endpoint는 /now이다.
  - gonic-gin 로그는 stdout 대신 파일에 남도록 설정한다.
- api/main.go
  - ```golang
    import (
      "io"
      "log"
      "net/http"
      "os"
      "time"

      "github.com/gin-contrib/cors"
      "github.com/gin-gonic/gin"
    )

    // handleNow: 현재 한국 시간 획득
    func handleNow(c *gin.Context) {
      loc, _ := time.LoadLocation("Asia/Seoul")
      now := time.Now().In(loc)
      c.JSON(http.StatusOK, gin.H{"data": now})
    }

    func main() {
      // gonic-gin 로그가 stdout 대신 파일에 남도록 설정
      gin.DisableConsoleColor()

      logFilePath := "gin.log"
      f, fileError := os.OpenFile(logFilePath, os.O_WRONLY|os.O_CREATE|os.O_APPEND, os.FileMode(0644))
      if fileError != nil {
        log.Fatal(fileError)
      }
      defer f.Close()

      gin.DefaultWriter = io.MultiWriter(f)

      // Gin mode를 release로 설정
      gin.SetMode(gin.ReleaseMode)

      // CORS 설정, front 서버 host는 localhost:8000이다.
      router := gin.Default()
      corsConf := cors.New(cors.Config{
        AllowOrigins: []string{"http://localhost:8000"},
      })
      router.Use(corsConf)

      // 현재 한국 시간 획득하는 API
      router.GET("/now", handleNow)
    ```
- 로컬환경에서 실행
  - 실행
    - ```bash
      $ go run main.go
      ```
  - 결과
    - <a href="/assets/img/2021-10-05-elk-container-example/02-api-local.jpg" target="_blank"><img src="/assets/img/2021-10-05-elk-container-example/02-api-local.jpg" width="70%"></a> 
- Dockerfile 작성
  - 개요
    - api 서버 내에 filebeat도 설치해야하기 때문에    
      기본 golang docker image에서 filebeat를 설치하는 Dockerfile를 작성하여 커스텀 이미지 생성한다.
    - Dockerfile 명령어 중 entrypoint라는 명령어가 있다.
    - entrypoint는 컨테이너를 생성할 때마다 실행되는 명령어를 지정하는 명령어이다.
    - 우리는 gonic-gin 뿐만아니라 filbeat도 함께 실행해야하므로   
      이를 실행하는 스크립트 파일을 생성하여 entrypoint로 실행시킨다.
  - api_script.sh
    - ```bash
      #!/usr/bin/env bash

      # filebeat를 background로 실행하고, stderr는 stdout으로 변경하여 filebeat.log에 기록
      nohup filebeat -e -c /etc/filebeat/filebeat.yml > filebeat.log 2>&1 &
      # gonic-gin 서버 실행
      go run main.go
      ```
  - Dockerfile
    - ```Dockerfile
      FROM golang:1.16

      WORKDIR /root/api

      COPY . /root/api

      # apt 저장소를 카카오미러로 변경
      RUN cp /etc/apt/sources.list sources.list_backup
      RUN sed -i -e 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list
      RUN sed -i -e 's/security.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list

      # filebeat 설치 및 서비스 등록
      # 참고: https://www.elastic.co/guide/en/beats/filebeat/current/setup-repositories.html
      RUN wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
      RUN apt-get install apt-transport-https -y
      RUN echo "deb https://artifacts.elastic.co/packages/oss-7.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-7.x.list
      RUN apt-get update && apt-get install filebeat -y
      COPY ./filebeat.yml /etc/filebeat/filebeat.yml
      RUN update-rc.d filebeat defaults 90 10

      # api_script.sh 실행
      ENTRYPOINT bash api_script.sh
      ```
- Dockerfile 기반으로 docker image 생성
  - ```bash
    > cd api
    > docker build -t es-c-api .
    ```
- docker image를 기반으로 컨테이너 생성
  - 설명
    - docker network es-net에 연결한다.
    - -e를 이용하여 logstash 컨테이너 이름을 api 컨테이너 LOGSTASH_IP 환경변수로 지정한다.
    - 8080번 포트로 컨테이너와 실제환경을 연결한다.
    - -d는 컨테이너를 detached하라는 의미로  
      컨테이너 stdout이 실제환경 쉘에 노출되지 않으며 background 실행처럼 돌아간다.
    - 가장 끝의 es-c-api는 컨테이너의 기반이 되는 이미지의 이름을 의미한다.
  - 실행
    - ```bash
      > docker run \
       --name es-c-api \
       --network es-net 
       -e LOGSTASH_IP=es-c-logstash \
       -p 8080:8080 \
       -d es-c-api
      ```
  - 실행 확인
    - ```bash
      > docker exec -it es-c-api bash
      $ tail -f /root/filebeat.log
      # 결과
      DEBUG   [input.filestream]      filestream/filestream.go:131    End of file reached: /root/api/gin.log; Backoff now.
      {"id": "DD9DC931427F9DD0", "source": "filestream::.global::native::187961-147", "path": "/root/api/gin.log", "state-id": "native::187961-147"}
      INFO    [file_watcher]  filestream/fswatch.go:137       Start next scan
      DEBUG   [file_watcher]  filestream/fswatch.go:204       Found 1 paths
      ...
      DEBUG   [processors]    processing/processors.go:203    Publish event: {
        "@timestamp": "2021-10-06T06:31:50.050Z",
        "@metadata": {
          "beat": "filebeat",
          "type": "_doc",
          "version": "7.15.0"
        },
        "input": {
          "type": "filestream"
        },
        "ecs": {
          "version": "1.11.0"
        },
        "host": {
          "name": "2af34ed1fa1d"
        },
        "agent": {
          "ephemeral_id": "6a3d70c7-da4d-4bda-8662-c1c9728ec367",
          "id": "4798c380-2dd3-4926-899f-d2006bcb4f4b",
          "name": "2af34ed1fa1d",
          "type": "filebeat",
          "version": "7.15.0",
          "hostname": "2af34ed1fa1d"
        },
        "log": {
          "file": {
            "path": "/root/api/gin.log"
          },
          "offset": 435
        },
        "message": "[GIN] 2021/10/06 - 06:31:47 | 200 |          94µs |      172.18.0.1 | GET      \"/now\"",
        "tags": [
          "gin"
        ]
      }
      ```

## Front
- 개요
  - nginx 웹서버를 실행시켜 main.html 페이지를 배포한다.
  - main.html에는 버튼이 하나 있으며 버튼을 클릭하면  
    현재 한국 시간을 얻어오는 api를 api 서버로부터 호출하여 렌더한다.
- main.html
  - ```html
    <!DOCTYPE html>
    <html lang="ko">
      <head>
        <meta charset="UTF-8" />
        <meta http-equiv="X-UA-Compatible" content="IE=edge" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <script src="./js/request.js" defer></script>
        <script src="./js/time.js" defer></script>
        <title>TEST FRONT WEB</title>
      </head>
      <body>
        <div>
          <button onclick="updateKoreanTime()">Update</button>
          <div>Current time(Korea)</div>
          <div id="korea-time-wrapper">
            <div id="korea-time"></div>
            <div id="korea-time-error"></div>
          </div>
        </div>
      </body>
    </html>
    ```
- js/request.js
  - ```javascript
    // CommonRequest : 공용 Request 함수
    function CommonRequest() {
      this.baseURL = "//localhost:8080"
    }

    // requestGetKoreanTime : 한국 시간 요청
    function requestGetKoreanTime(successFunc, failFunc) {
      CommonRequest.call(this)
      const url = `${baseURL}/now`
        fetch(url)
          .then(resp => resp.json())
          .then(respJSON => successFunc(respJSON))
          .catch(error => failFunc(error))
    }
    ```
- js/time.js
  - ```javascript
    // updateKoreanTime : 한국 시간 업데이트
    function updateKoreanTime() {
        const elemMap = {}
        const ids = ["korea-time", "korea-time-error"]

        for (let i=0; i<ids.length; i++) {
            const id = ids[i]
            const elem = document.querySelector(`#${id}`)
            if (elem === null) {
                console.warn(`A element of ${id} is not exists.`)
                return
            }
            elemMap[id] = elem
        }
      
        function successFunc(jespJSON) {
          console.log("successFunc", jespJSON)
          const koreaTime = jespJSON["data"]
          if (jespJSON["data"] !== "undefined") {
            elemMap["korea-time"].innerText = koreaTime
          }
        }
      
        function failFunc(error) {
          console.log("error", error)
          if (typeof error !== "undefined") {
            elemMap["korea-time-error"].innerText = error
          }
        }

        if (typeof requestGetKoreanTime !== "undefined") {
            requestGetKoreanTime(successFunc, failFunc)
        }
      }
    ```
- 로컬환경에서 실행
  - 실행
    - main.html을 더블클릭하면 브라우저에서 불러올 수 있다.
  - 결과
    - <a href="/assets/img/2021-10-05-elk-container-example/03-front-local.jpg" target="_blank"><img src="/assets/img/2021-10-05-elk-container-example/03-front-local.jpg" width="70%"></a> 
- Dockerfile 작성
  - 개요
    - front 서버 내에 filebeat도 설치해야하기 때문에    
      기본 nginx image에서 filebeat를 설치하는 Dockerfile을 작성하여 커스텀 이미지 생성한다.
    - 우리는 nginx 뿐만아니라 filbeat도 함께 실행해야하므로   
      이를 실행하는 스크립트 파일을 생성하여 entrypoint로 실행시킨다.
  - front_script.sh
    - ```bash
      #!/usr/bin/env bash

      # filebeat를 background로 실행하고, stderr는 stdout으로 변경하여 filebeat.log에 기록
      nohup filebeat -e -c /etc/filebeat/filebeat.yml > filebeat.log 2>&1 &

      # docker container를 detached로 실행할 경우, nginx를 daemon으로 실행하면 안된다.
      nginx -g 'daemon off;'
      ```
  - Dockerfile
    - ```Dockerfile
      FROM nginx:1.21.3

      WORKDIR /root

      # front 파일 복사
      RUN mkdir -p /usr/share/nginx/html/js
      COPY ./main.html /usr/share/nginx/html/
      COPY ./js /usr/share/nginx/html/js

      # apt 저장소 카카오미러로 변경
      RUN cp /etc/apt/sources.list sources.list_backup
      RUN sed -i -e 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list
      RUN sed -i -e 's/security.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list
      RUN apt-get update && apt-get upgrade -y
      RUN apt-get install wget gnupg -y

      # nginx 도커 이미지에서는 docker logs에 로깅할 수 있도록
      # access.log는 stdout으로, error.log는 stderr로 symbolic link를 만들어두었다.
      # 우리는 파일 기반으로 처리할 것이므로 symbolic link를 해제한다.
      RUN rm -f /var/log/nginx/access.log
      RUN rm -f /var/log/nginx/error.log

      # filebeat 다운로드 및 실행
      # 참고: https://www.elastic.co/guide/en/beats/filebeat/current/setup-repositories.html
      RUN wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
      RUN apt-get install apt-transport-https -y
      RUN echo "deb https://artifacts.elastic.co/packages/oss-7.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-7.x.list
      RUN apt-get update && apt-get install filebeat -y
      COPY ./filebeat.yml /etc/filebeat/filebeat.yml
      RUN update-rc.d filebeat defaults 90 10

      # front_script.sh 실행
      COPY ./front_script.sh /root/

      ENTRYPOINT bash front_script.sh
      ```
- Dockerfile 기반으로 docker image 생성
  - ```bash
    > cd front
    > docker build -t es-c-front .
    ```
- docker image를 기반으로 컨테이너 생성
  - 설명
    - docker network es-net에 연결한다.
    - -e를 이용하여 logstash 컨테이너 이름을 api 컨테이너 LOGSTASH_IP 환경변수로 지정한다.
    - 컨테이너의 80번 포트를 실제환경의 8000번 포트로 연결한다.  
      이는 nginx의 기본 포트가 80번이기 때문이다.
    - -d는 컨테이너를 detached하라는 의미로  
      컨테이너 stdout이 실제환경 쉘에 노출되지 않으며 background 실행처럼 돌아간다.
    - 가장 끝의 es-c-front는 컨테이너의 기반이 되는 이미지의 이름을 의미한다.
  - 실행
    - ```bash
      > docker run \
       --name es-c-front \
       --network es-net 
       -e LOGSTASH_IP=es-c-logstash \
       -p 8000:80 \
       -d es-c-front
      ```
  - 실행확인
    - ```bash
      > docker exec -it es-c-front bash
      $ tail -f /root/filebeat.log
      # 결과
      DEBUG   [logstash]      logstash/async.go:120   connect
      INFO    [publisher]     pipeline/retry.go:219   retryer: send unwait signal to consumer
      INFO    [publisher]     pipeline/retry.go:223     done
      ...
      DEBUG   [processors]    processing/processors.go:203    Publish event: {
        "@timestamp": "2021-10-06T06:26:05.525Z",
        "@metadata": {
          "beat": "filebeat",
          "type": "_doc",
          "version": "7.15.0"
        },
        "log": {
          "offset": 2225,
          "file": {
            "path": "/var/log/nginx/access.log"
          }
        },
        "message": "172.18.0.1 - - [06/Oct/2021:06:26:05 +0000] \"GET /main.html HTTP/1.1\" 304 0 \"-\" \"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.61 Safari/537.36\" \"-\"",
        "tags": [
          "nginx-access"
        ],
        "input": {
          "type": "filestream"
        },
        "ecs": {
          "version": "1.11.0"
        },
        "host": {
          "name": "562db3883fa7"
        },
        "agent": {
          "version": "7.15.0",
          "hostname": "562db3883fa7",
          "ephemeral_id": "19325ad6-c59c-4013-82c8-33b040c3d187",
          "id": "d97701a2-949e-4610-b98f-f9441f35555a",
          "name": "562db3883fa7",
          "type": "filebeat"
        }
      }
      ```

## Kibana
- 개요
  - kibana로 GUI 이용하여 elasticsearch 검색을 이용할 수 있다.
- 컨테이너 생성 설명
  - --rm을 지정하면 컨테이너 종료 후 컨테이너가 삭제된다.
  - docker network, es-net에 연결한다.
  - 컨테이너 5601번과 실제환경 5601을 연결한다.  
    이를 통해 실제환경에서 localhost:5601로 접속하면 컨테이너의 키바나로 접속할 수 있다.
  - -e를 통해 elasticsearch와 연결한다.
  - kibana는 7.15.0 버전을 사용한다.
- 실행
  - ```bash
    > docker run \
      --rm \
      --name es-c-kibana \
      --net es-net \
      -p 5601:5601 \
      -e "ELASTICSEARCH_HOSTS=http://es-c-es:9200" \
      docker.elastic.co/kibana/kibana:7.15.0
    # 브라우저를 켜고 localhost:5601로 접속
    # 좌상단 햄버거 메뉴 클릭 
    #  > Analytics 하위의 Discover 클릭 
    #  > 인덱스 패턴이 없다면 filebeat*로 인덱스 패턴 생성 
    #  > query 입력창에 원하는 kibana query 입력
    ```
- 검색 예시
  - 설명
    - gin 로그 중 response_status code가 200인 로그를 보고 싶다.
  - 실행
    - kibana query 입력창에 gin.response_status:200을 입력한다.
  - 결과
    - <a href="/assets/img/2021-10-05-elk-container-example/04-kibana.jpg" target="_blank"><img src="/assets/img/2021-10-05-elk-container-example/04-kibana.jpg" width="100%"></a> 

## 실험 소감
- 이 예시에서는 로그를 컨테이너에 저장하기 때문에 컨테이너 삭제 시 로그가 날아간다.
- 실제 환경이었다면 Application server(front, api)에서 남는 로그를 s3에 저장하고,  
  로그 분석이 필요한 경우, filebeat, logstash, elasticsearch, kibana 컨테이너를 생성하고  
  filebeat input을 s3에서 불러와 로그분석을 할 것 같다.
- 실험에는 총 5개의 컨테이너가 생성되었는데,  
  내 pc에서 해당 컨테이너들을 돌리는데 소비하는 램은 ddr3 12gb였다.

## 참고
- [filebeat input docker](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-docker.html)
- [docker -d 옵션](https://docs.docker.com/engine/reference/run/#detached-vs-foreground)
- [도커 환경변수](https://blog.naver.com/PostView.nhn?isHttpsRedirect=true&blogId=complusblog&logNo=220975099502)
- [nginx pattern](https://www.codegrepper.com/code-examples/whatever/nginx+log+grok+pattern)
- [grok filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)
- [logstash grok patterns](https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns)
- [grok whitespace](https://logstash-users.narkive.com/huZm4lfW/how-to-trim-space-using-grok)
- [os.OpenFile(path, 권한, 파일mod) 예시](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=parkjy76&logNo=220737009212)
- [gonic-gin how to write log file](https://github.com/gin-gonic/gin#how-to-write-log-file)
- [logstash에서 hosts는 underscore를 지원하지 않음](https://github.com/logstash-plugins/logstash-output-elasticsearch/issues/400)
- [filebeat 설치 및 실행](https://www.elastic.co/guide/en/beats/filebeat/current/setup-repositories.html)
- [What's the difference between `os.O_APPEND` and `os.ModeAppend`?](https://stackoverflow.com/questions/50205448/whats-the-difference-between-os-o-append-and-os-modeappend)
