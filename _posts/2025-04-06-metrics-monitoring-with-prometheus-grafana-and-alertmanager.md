---
title: 메트릭 모니터링(prometheus, grafana, alertmanager)
date: 2025-04-06 08:49:51 +0900
categories: [Monitoring]
tags: [prometheus, grafana, alertmanager, essay]    # TAG names should always be lowercase
---

## 정의
- 시스템 상태와 성능을 지속적으로 관찰하고 분석하는 과정  
  이를 통해 문제를 조기에 발견하는 것을 목표로 한다.  

## 메트릭(metric)이란
- 시스템, 애플리케이션 또는 프로세스의 성능이나 상태를   
  수치적으로 표현한 데이터 포인트  
- 우리는 메트릭을 지속적으로 수집하여  
  문제 발생 시 경고(alert)을 보내거나  
  시각화하여 문제를 빠르게 판단하는 시스템을 구축한다.  

## 사용툴
- 프로메테우스(prometheus)  
    - 메트릭 수집 및 저장  
- 그라파나(grafana)  
    - 메트릭을 시각화  
- 얼럿매니저(alertmanager)  
    - 메트릭 조건식을 지정하여 조건식을 만족하면 메신저로 메세지 전송  

## 프로메테우스 메트릭 익스포터(metric exporter) 종류
- node_exporter: 시스템 메트릭을 수집한다  
- spring_exporter: 스프링 애플리케이션 매트릭을 수집힌다.  

## node_exporter 메트릭
- 주요 기능  
    - CPU, 메모리, 디스크, 네트워크 대역폭, node_exporter의 gc(garbage collection)  
- 주요 메트릭  
    - CPU 사용률  
        - (sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance))  
    - 사용 가능한 메모리 비율  
        - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)  
    - 디스크 사용률  
        - (node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes  
    - 네트워크 평균 수신 속도(최근 1시간)  
        - avg_over_time(rate(node_network_receive_bytes_total[5m])[1h:5m])  
    - node_exporter 평균 gc 시간  
        - avg(rate(go_gc_duration_seconds[5m])) by (instance)  
- 참고  
    - [설치가이드 및 메트릭 설명](https://prometheus.io/docs/guides/node-exporter/){:target="_blank"}  

## spring_exporter 메트릭
- 주요 기능  
    - HTTP 요청, JVM, gc, 활성스레드, 스프링 CPU 사용량, 스프링 디스크 I/O  
    - CPU와 디스크는 node_exporter로 대체 가능  
- 주요 메트릭  
    - HTTP 요청 처리 시간(초)  
        - sum(http_server_requests_seconds_sum) by (instance) / sum(http_server_requests_seconds_count) by (instance)  
    - HTTP 요청 수  
        - (rate(http_server_requests_seconds_count[5m]))  
    - HTTP 오류 요청 비율  
        - (sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (instance) / sum(rate(http_server_requests_seconds_count[5m])) by (instance))  
    - JVM 힙 메모리 사용률  
        - sum(jvm_memory_used_bytes{area="heap"}) by (instance) / sum(jvm_memory_max_bytes{area="heap"}) by (instance)  
    - 활성 스레드 수  
        - jvm_threads_live_threads  
    - 스프링 평균 gc 지연시간(초)  
        - avg(jvm_gc_pause_seconds_sum) by (instance)  
- 참고  
    - [지원하는 메트릭 설명](https://docs.spring.io/spring-boot/reference/actuator/metrics.html#actuator.metrics.supported){:target="_blank"}  

## 구조
- 모니터링 대상에 exporter 설치  
    - docker container 내부에 spring과 node_exporter를 설치한다.  
    - spring에 프로메테우스 메트릭 관련 의존성 패키지를 설치한다.  
- 프로메테우스 환경을 구축한다.  
    - 프로메테우스 설정에서 모니터링 대상 관련 설정을 추가한다.  
- 얼럿매니저 환경을 구축한다.  
    - 프로메테우스 메트릭을 조합해서 얼럿을 보내도록 설정한다.  
    - 얼럿이 발생하면 메신저(예시에서는 슬랙)  

## 노드 메트릭 설정
- 설명  
    - os에서 node_exporter를 설치해야한다.  
    - jdk 이미지 기반 Dockerfile 내에서  node_exporter를 설치 후 실행한다.  
- 도커파일  
  ```text  
  FROM amazoncorretto:21-alpine3.20  
            
  ...   
            
  # node_exporter 설치  
  RUN curl -L -O https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz  
            
  RUN tar xvfz node_exporter-*.*-amd64.tar.gz  
            
  RUN rm node_exporter-*.*-amd64.tar.gz  
            
  ...  
            
  # 스크립트 복사  
  COPY ./docker/web-script.sh .  
            
  RUN chmod +x /web-script.sh  
            
  CMD ["/web-script.sh"]  
  ```  
- web-script.sh  
  ```bash  
  #!/bin/sh  
            
  # Run node_exporter  
  ./node_exporter-*.*-amd64/node_exporter --web.listen-address=0.0.0.0:9100 --log.level=info &  
            
  ...  
  ```  
- 확인  
    - 도커 컨테이너를 올린다.  
    - 브라우저를 켜고 http://localhost:9100/metrics 확인  
    - 성공 시 응답 예시  
      ```text  
      # HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.  
      # TYPE go_gc_duration_seconds summary  
      go_gc_duration_seconds{quantile="0"} 1.2345e-04  
      go_gc_duration_seconds{quantile="0.25"} 1.2345e-04  
      go_gc_duration_seconds{quantile="0.5"} 1.2345e-04  
      go_gc_duration_seconds{quantile="0.75"} 1.2345e-04  
      go_gc_duration_seconds{quantile="1"} 1.2345e-04  
      go_gc_duration_seconds_sum 1.2345e-04  
      go_gc_duration_seconds_count 9  
      ```  

## 스프링 프로메테우스 메트릭 설정
- 가정  
    - 스프링은 설치되어 있다고 가정한다.  
- build.gradle.kts  
  ```kotlin  
  dependencies {  
      // Actuator 기능 추가  
      implementation("org.springframework.boot:spring-boot-starter-actuator")   
      // Prometheus 메트릭 등록  
      implementation("io.micrometer:micrometer-registry-prometheus")  
  }  
  ```  
- resources/application.properties  
  ```text  
  # Prometheus  
  management.endpoints.web.exposure.include: prometheus  
  ```  
- 확인  
    - 도커 컨테이너를 올려 스프링 웹서버를 실행시킨다.  
    - 브라우저를 켜고 http://localhost:8080/actuator/prometheus 확인  
    - 성공 시 응답 예시  
      ```text  
      # HELP application_ready_time_seconds Time taken for the application to be ready to service requests  
      # TYPE application_ready_time_seconds gauge  
      application_ready_time_seconds{main_application_class="com.example.demo.DemoApplicationKt"} 123.456  
                
      # HELP application_started_time_seconds Time taken to start the application  
      # TYPE application_started_time_seconds gauge  
      application_started_time_seconds{main_application_class="com.example.demo.DemoApplicationKt"} 123.123  
                
      # HELP disk_free_bytes Usable space for path  
      # TYPE disk_free_bytes gauge  
      disk_free_bytes{path="/."} 1.23456789E12  
      ...  
      ```  

## 프로메테우스에서 메트릭 수집
- prometheus.yml  
  ```yml  
    scrape_configs:  
      - job_name: 'web'  
        metrics_path: '/actuator/prometheus'  
        scheme: 'http'  
        static_configs:  
          - targets: ['web:8080']  
              
      - job_name: 'web-node_exporter'  
        metrics_path: '/metrics'  
        scheme: 'http'  
        static_configs:  
          - targets: ['web:9100']  
  ```  
- docker-compose.yml  
  ```yml  
    prometheus:  
      image: prom/prometheus:latest  
      container_name: prometheus  
      restart: always  
      volumes:  
        - "./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml"  
      ports:  
        - "9090:9090"  
      depends_on:  
        - web  
      networks:  
        - app-network  
  ```  
- 확인  
    - 스프링 도커 컨테이너와 프로메테우스 도커 컨테이너를 실행한다.  
    - 브라우저를 켜고 http://localhost:9090 접속  
    - node_exporter 변수 입력  
        - 검색창에 node_cpu_seconds_total 입력 후 "Execute" 버튼 클릭  
        - 성공 시 응답 예시  
          ```text  
          node_cpu_seconds_total{cpu="0", instance="web:9100", job="web-node_exporter", mode="idle"} 1234.56  
          node_cpu_seconds_total{cpu="0", instance="web:9100", job="web-node_exporter", mode="iowait"} 12.34  
          node_cpu_seconds_total{cpu="0", instance="web:9100", job="web-node_exporter", mode="irq"} 0.12  
          node_cpu_seconds_total{cpu="0", instance="web:9100", job="web-node_exporter", mode="nice"} 0.01  
          node_cpu_seconds_total{cpu="0", instance="web:9100", job="web-node_exporter", mode="softirq"} 1.23  
          node_cpu_seconds_total{cpu="0", instance="web:9100", job="web-node_exporter", mode="steal"} 0  
          ```  
    - spring_exporter 변수 입력  
        - 검색창에 http_server_requests_seconds_count 입력 후 "Execute" 버튼 클릭  
        - 성공 시 응답 예시  
          ```text  
          http_server_requests_seconds_count{error="none", exception="none", instance="web:8080", job="web", method="GET", outcome="SUCCESS", status="200", uri="/actuator/prometheus"} 12345  
          http_server_requests_seconds_count{error="none", exception="none", instance="web:8080", job="web", method="GET", outcome="CLIENT_ERROR", status="404", uri="/**"} 6789  
          ```  
- 프로메테우스 검색 사용 방법  
    - 메트릭 이름으로 검색  
        - node_cpu_seconds_total  
    - Label로 필터링 해서 검색  
        - node_cpu_seconds_total{cpu="0", mode="user"}  
    - 집계(aggregation) 함수 사용  
        - sum(node_cpu_seconds_total{mode="idle"})  

## 그라파나에서 프로메테우스 메트릭 시각화
- 설명  
    - 프로메테우스에서 매번 검색하기도 어렵고   
      검색 결과를 시각화하여 한 눈에 보고 싶을 경우  
      그라파나와 연동하여 시각화할 수 있다.  
- docker-compose.yml  
  ```yml  
    grafana:  
      image: grafana/grafana:latest  
      container_name: grafana  
      restart: always  
      ports:  
        - "3000:3000"  
      volumes:  
        - ./storage/grafana:/var/lib/grafana  
      depends_on:  
        - prometheus  
      networks:  
        - app-network  
  ```  
- 확인  
    - 스프링, 프로메테우스, 그라파나 도커 컨테이너를 실행한다.  
    - 브라우저를 켜고 http://localhost:3000 접속  
    - 그라파나 로그인 화면이 나오면 성공  
    - 초기 ID/Password 는 admin/admin 이다.  
- 프로메테우스 데이터 소스 생성  
    - (좌상단 로고 클릭) Connetions > Data sources   
    - 검색창에서 Prometheus 검색 후 결과에서 Prometheus 클릭  
    - Connetion에서 http://prometheus:9090 입력  
        - URL의 prometheus는 도커 서비스 명이다.  
    - save & test 버튼 클릭  
- 대시보드 생성  
    - (좌상단 로고 클릭) Dashboards 클릭  
    - (우상단) New 버튼 클릭, (드롭다운 메뉴에서) New dashboard 클릭  
    - + Add visualization 버튼 클릭  
    - (Select data source)에서 prometheus 클릭(이름은 다를 수 있음)  
    - (오른쪽 사이드메뉴 panel options에서) Title 입력  
    - (하단 메뉴 A 항목에서) builder/code 중 code 선택  
    - "Enter a PromQL Query" 에 내가 원하는 프로메테우스 식을 넣으면 된다.  
        - 예를 들어 CPU 사용률  
          sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)  
    - (하단 메뉴 A 항목에서) Run queries 클릭하여 그래프 확인  
    - (우상단) Save dashboard 버튼 클릭  
    - (Save dashboard 왼쪽 사이드 메뉴에서) Title 입력 후 Save 버튼 클릭  
- 대시보드에 판넬 추가  
    - (좌상단 로고 클릭) Dashboards 클릭  
    - (상단) Add 버튼 클릭, (드롭다운 메뉴에서) Visualization 클릭  
    - 위의 대시보드 생성과 같은 과정을 진행하면 판넬이 추가된다.  
- 대시보드 판넬 삭제 및 수정  
    - 판넬 위젯 우상단에 (세로로 ... ) 클릭  
    - (드롭다운 메뉴에서) 판넬 삭제하고 싶으면 Remove 클릭  
    - (드롭다운 메뉴에서) 판넬 수정하고 싶으면 Edit 클릭  

## alertmanager 알람 전송
- 설명  
    - 프로메테우스 메트릭을 이용하여 조건 충족 시 알람을 전송하게 할 수 있다.  
    - 그라파나에도 알람이 존재하나  
      알람을 보내기 위해 무거운 그라파나를 항시 켜둬야 하는 문제와  
      그라파나 다운 시 알람이 동작하지 않는 문제가 발생할 수 있기에  
      alertmanager를 선택하였다.  
- alert_rules.yml  
    - 설명  
        - 구체적인 알림을 설정하는 부분이다.  
    - 기본적인 구조  
      ```yml  
      groups:  
        - name: 그룹_이름  
          rules:  
            - alert: 알람_이름  
              expr: 경고를 발생시킬 조건 (PromQL)  
              for: 경고 지속 시간  
              labels:  
                severity: 경고 심각도  
              annotations:  
                summary: "요약 메시지"  
                description: "알람 상세 설명"  
      ```  
    - 구체적 설명  
        - groups  
            - 알람 규칙을 그룹으로 묶음  
        - rules  
            - 이 그룹 안에서 설정할 개별 알람 목록  
            - 하나의 그룹 안에 여러 개의 알람 추가 가능  
        - alert: 알람_이름  
            - 알람의 이름  
        - expr: 경고를 발생시킬 조건 (PromQL)  
            - PromQL로 알람 조건을 정의  
            - 예제:  메모리 사용률이 80%를 초과하면 경고  
              ```yml  
              expr: node_memory_Active_bytes / node_memory_MemTotal_bytes > 0.8  
              ```  
        - for: 경고 지속 시간  
            - 경고 조건이 특정 시간 동안 지속될 경우 알람을 발생시킴  
            - 예제:  5분 동안 조건이 충족되면 알람 발생  
                - for: 5m  
        - labels  
            - 알람의 속성(라벨)을 지정  
            - 주로 severity(심각도)를 설정  
            - 예제: warning  
              ```yml  
              labels:  
                severity: warning  
              ```  
        - annotations  
            - 알람이 발생했을 때 추가 메시지를 설정  
            - summary: 간단한 알람 설명  
            - description: 상세한 알람 내용  
            - summary와 description은 Go Template이다.  
            - 예제  
              ```yml  
              annotations:  
                summary: "메모리 사용량이 높습니다!"  
{% raw %}                description: "Instance {{ $labels.instance }}의 메모리 사용률이 80%를 초과했습니다."{% endraw %}  
              ```  
- prometheus.yml  
  ```yml  
  alerting:  
    alertmanagers:  
      - static_configs:  
          - targets:  
              - 'alertmanager:9093'  # Alertmanager의 주소  
            
  rule_files:  
    - '/etc/prometheus/alert_rules.yml'  # 알림 규칙 파일  
  ```  
    - 프로메테우스에 alert_rules.yml을 등록한다.  
    - 얼럿매니저를 등록한다.  
    - 이때 alertmanger:9093 중 alertmanger은 도커 서비스 명이다.  
- alertmanager.yml  
    - 설명  
        - global - alertmanager의 전역 설정  
        - route - 알람을 어떻게 라우팅할지 설정  
        - receivers - 알림을 받을 대상 설정  
    - 기본적 구조  
      ```yml  
        global:  
          resolve_timeout: 알람 해제 후 유지 시간  
                  
        route:  
          group_by: [알람 그룹 기준]  
          group_wait: 최초 알람 대기 시간  
          group_interval: 그룹 내 추가 알람 무시 시간  
          repeat_interval: 같은 알람 반복 전송 시간  
          receiver: 기본 수신자  
                  
        receivers:  
          - name: 수신자_이름  
            알람 전송 방식 설정  
      ```  
    - 구체적 설명  
        - global  
            - algermanager 전역 설정  
            - resolve_timeout  
                - 알람이 해결된 후 해제 상태를 유지하는 시간  
                - 알람 조건이 충족되어 알람이 전송된 후  
                  운영자가 조치하여 알람 조건이 해제되었다고 하자.  
                  그 상태가 resolve_timeout 시간만큼 지나면 "해제됨" 상태가 된다.  
                  이때 슬랙에 "정상"이라고 메세지를 보낼 수 있게 된다.  
        - route  
            - 알람을 그룹화하고 전달하는 방식  
            - group_by  
                - 같은 그룹으로 묶을 라벨 지정  
                - alert_rules.yml의 groups > rules > labels에 따라   
                  group by 하게 된다.  
            - group_wait  
                - 첫 번째 알람이 발생한 후 전송하기 전 대기 시간  
                - 처음 한 개의 알람이 발생하면 프로메테우스에서   
                  "혹시 이거랑 같이 묶을 수 있는 다른 알람도 곧 생기지 않을까?"   
                  하고 기다리는 시간  
            - group_interval  
                - 같은 그룹 내에서 추가 알람 발생 시 추가 알람 무시 시간  
            - repeat_interval  
                - 동일 알람이 반복될 경우 무시하는 시간  
            - receiver  
                - 알람을 보낼 기본 수신자 설정  
                - 아래 receivers에서 설정한 name 중 하나를 입력한다.  
        - receivers  
            - 알람을 받을 대상 설정(Slcak, Email 등)  
            - name: 수신자 이름  
            - 알람 전송 방식 설정: Slack, Email 등의 설정 추가  
                - 슬랙 예시  
                  ```yml  
                  {% raw %}
                  receivers:  
                    - name: 'slack-notifications'  
                      slack_configs:  
                        - api_url: '<your slack-webhook URL>'  
                          channel: '#alarm'  
                          text: >
                            {{ range $i, .Alerts }}
                            {{ if ne $i 0 }}\n--\n{{ end }}
                            *Alert:* {{ .Annotations.summary }}\n
                            *Description:* {{ .Annotations.description }}\n
                            *View Alert:* <http://localhost:9093/#/alerts?silenced=false&inhibited=false&active=true&filter=%7Balertname%3D%22{{ .Labels.alertname }}%22%7D|Click here>
                            {{ end }}

                  {% endraw %}
                  ```  
                - apj_url  
                    - 유효한 webhook 주소를 반드시 적어야 함  
                    - [슬랙 웹훅 생성법](https://help.ovice.com/hc/ko/articles/16515667939737--Webhook-%EC%84%A4%EC%A0%95-Slack%EC%97%90%EC%84%9C-%EC%95%8C%EB%A6%BC-%EB%B0%9B%EA%B8%B0){:target="_blank"}  
                      (1~13번까지 따라하고 웹훅 URL 복사하여 api_url에 붙여넣기)  
- docker-compose.yml  
  ```yml  
    prometheus:  
      image: prom/prometheus:latest  
      container_name: prometheus  
      restart: always  
      volumes:  
        - "./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml"  
        # !-- 추가 시작 --! #  
        - "./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml"  
        # !-- 추가 끝 --! #  
      ports:  
        - "9090:9090"  
      depends_on:  
        - web  
      networks:  
        - app-network  
            
    alertmanager:  
      image: prom/alertmanager:latest  
      container_name: alertmanager  
      restart: always  
      ports:  
        - "9093:9093"  
      volumes:  
        - "./prometheus/alertmanager.yml:/etc/alertmanager/config.yml"  
      command:  
        - '--config.file=/etc/alertmanager/config.yml'  
      depends_on:  
        - prometheus  
      networks:  
        - app-network  
  ```  
- 확인  
    - 스프링, 프로메테우스, 얼럿매니저 도커 컨테이너를 실행한다.  
    - 프로메테우스 알람 룰이 잘 등록되었는지 확인  
        - 브라우저를 켜고 http://localhost:9090/alerts로 접속  
        - alert_rules.yml에 등록한 알람이 잘 등록되었는지 확인  
    - 얼럿매니저 잘 동작하는지 확인  
        - 브라우저를 켜고 http://localhost:9093 접속  
        - 정상 접속되면 성공  
        - 알람이 발송되면 slack과 얼럿매니저에서 기록되어야 함  
- 알람 일부러 일으켜 보기  
    - alert_rules.yml  
      ```yml  
      {% raw %}
      groups:  
        - name: CPU  
          rules:  
            - alert: HighCPUUsage  
      #        expr: (sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)) > 0.7  
      #        for: 30s  
              expr: (sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)) > 0.00001  
              for: 1s  
              labels:  
                severity: critical  
              annotations:  
                summary: "CPU 사용량 70% 초과 지속"  
                description: "Instance {{ $labels.instance }} has high CPU usage."
      {% endraw %}  
      ```  
    - 위 설정을 적용하고 일정 시간 기다림  
    - 이후 slack이나 얼럿매니저를 확인하면 알람 확인 가능  

## 예시 repository
- [링크](https://github.com/wjchoi-hh/deployment-sandbox){:target="_blank"}  

## 참고
- [슬랙 웹훅 생성](https://help.ovice.com/hc/ko/articles/16515667939737--Webhook-%EC%84%A4%EC%A0%95-Slack%EC%97%90%EC%84%9C-%EC%95%8C%EB%A6%BC-%EB%B0%9B%EA%B8%B0){:target="_blank"}  
