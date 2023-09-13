---
title: filebeat input에 s3 사용하기(not using AWS SQS)
date: 2022-06-21 19:00:00 +0900
categories: [ELK]
tags: [elk, elasticsearch, filebeat]    # TAG names should always be lowercase
---
## 개요
- filebeat의 input으로 s3를 사용한다.
- s3 input은 filebeat 8.x 버전부터 지원한다.
- AWS SQS(Simple Queue Service)를 사용하는 방법이 있으나  
  여기서는 filebeat이 직접 s3를 탐색하는 방법을 설명한다. 

## filebeat에서 s3를 바라보는 것 외에 다른 방법
- logstash input 에서도 s3를 읽어올 수 있다.
- filebeat는 server 내에 설치되어 동작하는 작은 agent이기 때문에  
  scale out 하는 기능이 없다.
- 만약 로그량이 너무 많아 fileabeat에 부하가 생기는 경우  
  filebeat 서버를 scale up하거나 filebeat 서버를 여러 대 만든 후 
  각 filebeat 마다 바라보는 s3 경로를 다르게 주어  
  운영자가 임의로 설정을 쪼개줘야 한다.
- 원본 로그를 s3에 배치로(약 1초 간격으로) 업로드하고  
  어떤 곳에서 전처리한 후 eleasticsearch로 전송하는 구조라면  
  지금처럼 filebeat에서 s3를 바라보는 구조가 아니라  
  logstash에서 s3를 바라보고, 로그량이 너무 많은 경우 logstash를 scale out하는 구조가 더 적합하다고 생각한다.  

## filebeat 설치 및 실행
- 설명
    - filebeat를 파일로 설치 후 백드라운드로 실행한다.  
      (centos 7에서 전역 filebeat 서비스를 설치하여 실행한 결과  
       awscli가 root 유저로 실행되어 기본 유저로 지정되고   
       지역이 기본값인 미국으로 설정되는 문제가 있어 우선 위 방법으로 진행한다)
- 명령
    - ```bash
        # 아래 사이트에서 filebeat 8.x 버전을 다운로드한다.
        # https://www.elastic.co/kr/downloads/beats/filebeat

        # os에 맞는 파일을 설치한다. (아래 예시는 centos 7 기준)
        wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.2.3-linux-x86_64.tar.gz
        tar xvzf ./filebeat*.tar.gz

        # 실행(백그라운드 실행)
        nohup ./filebeat-8.2.3-linux-x86_64/filebeat -e -c filebeat.yml &
        ```

## awscli 설정
- 설명
    - awscli를 설치한다.
    - [s3 IAM 설정](https://artiiicy.tistory.com/16){:target="_blank"}을 참조하여   
      s3 IAM를 생성하고 액세스 키 ID와 비밀 액세스 키를 획득한다.
    - awscli에서 auth 정보를 설정한다.
- 명령
    - ```bash
        $ aws configure
        AWS Access Key ID [None]: ${위에서 생성한 액세스 키 ID}
        AWS Secret Access Key [None]: ${위에서 생성한 비밀 액세스 키}
        Default region name [None]: ${s3가 위치한 지역}
        Default output format [None]: json
        ```

## filebeat input 설정
- 설명
    - s3를 읽어오도록 filebeat.inputs를 설정한다.
- 설정
    - ```bash
        # ./filebeat-8.2.3-linux-x86_64/filebeat.yml
        filebeat.inputs:
        # Each - is an input. Most options can be set at the input level, so
        # you can use different inputs for various configurations.
        # Below are the input specific configurations.
        - type: aws-s3
          fields:
            index_name: "my-test-log"
          enabled: true
          tags: ["test"]
          bucket_arn: arn:aws:s3:::my-log-bucket
          bucket_list_prefix: WEB/access
          number_of_workers: 5
          bucket_list_interval: 30s
          credential_profile_name: default
          file_selectors:
            - regex: '/log_[0-9]{8}.log$'
              expand_event_list_from_field: Records
        ```
- 주요 설정 설명
    - type
        - aws-s3로 해야 s3 input을 읽을 수 있다.
    - fields > index_name
        - logstash에서 filebeat로부터 받은 데이터로 인덱스를 생성할 때 prefix로 사용한다.
    - tags
        - logstash에서 filebeat로부터 받은 데이터가 어떤 로그인지 판단할 때 사용한다.
    - bucket_arn
        - [s3 버킷의 ARN(리소스 이름)](https://docs.aws.amazon.com/ko_kr/general/latest/gr/aws-arns-and-namespaces.html){:target="_blank"}
        - awscli에서 등록한 auth 정보가 입력한 ARN에 대한 권한이 없는 경우  
          authentication failed 에러가 난다.
    - bucket_list_prefix
        - bucket_arn은 bucket의 최상위 경로까지만 접근한다.
        - 하위 경로로 접근하기 위해서는 bucket_list_prefix에서 설정하면 된다.
    - number_of_workers
        - s3의 변화를 감지하고 데이터를 읽어올때 사용할 workers 수이다.
    - bucket_list_interval
        - s3에 변화가 있는지 확인 간격
    - credential_profile_name
        - awscli에서는 여러 auth 정보를 등록할 수 있다.
        - 이때 profile name을 지정할 수 있는데 해당 profile_name을 설정하는 부분이다.
        - 예시에서는 기본값으로 등록하였으므로 default 로 설정하였다.
    - file_selectors
        - filebeat이 읽고자하는 파일명 패턴을 등록하여 읽을 파일 수를 줄이는 부분이다.    
        - ex) log_20220621.log, log_20220521.log

# filebeat 정상 동작 확인
1. 로그스태시 설치 및 실행
    - 개요
        - 로그스태시를 설치하고 input은 filebeat으로 output은 표준출력으로 지정한다.
    - 로그스태시 설치
        - ```bash
            # 아래 사이트에서 Logstash 7.x 버전을 다운로드한다.
            # (최신 버전은 8이지만 기존에 구축된 로그스태시가 7이라서 예제는 7 기준으로 적는다)
            # https://www.elastic.co/kr/downloads/logstash

            # os에 맞는 파일을 설치한다. (아래 예시는 centos 7 기준)
            wget https://artifacts.elastic.co/downloads/logstash/logstash-7.12.1-linux-x86_64.tar.gz
            tar xvzf ./logstash*.tar.gz

            # 실행
            ./logstash-7.12.1-linux-x86_64/bin/logstash
            ```
    - 로그스태시 설정 변경
        - ```bash
            # ./logstash-7.12.1-linux-x86_64/config/default.conf
            input {
                beats {
                    port => "5044"
                }
            }

            filter { }

            output {
                stdout {
                    codec => rubydebug
                }
            }
            ```
    - 재실행
        - ```bash
            ./logstash-7.12.1-linux-x86_64/bin/logstash
            ```
2. 파일비트 재실행
    - 개요
        - 로그스태시가 떠올랐으면 파일비트를 재실행하여 s3 정보를 로그스태시로 전송한다.
        - 파일비트 설정을 변경하여 output을 logstash로 지정한다.
        - 만약 전송 중 끊겼다면 ./filebeat-8.2.3-linux-x86_64/data/registry 디렉토리를  
        지우고 재실행하면 처음부터 진행할 수 있다.
    - 파일비트 설정 변경
        - ```bash
            # ./filebeat-8.1.2-linux-x86_64/filebeat.yml
            ...
            output.logstash:
                hosts: ["127.0.0.1:5044"]
            ...
            ```
    - 재실행
        - ```bash
            # 새로운 쉘 창을 띄우고 실행
            ./filebeat-8.2.3-linux-x86_64/filebeat -e -c filebeat.yml
            ```
3. 확인
    - 로그스태시 화면
        - 로그스태시 쉘 창에 로그가 막 올라오고 있다면 정상 작동하는 것이다.

## 참고
- [AWS CLI로 인증 정보 (Access Key ID, Secret Access Key) 관리하기](https://www.daleseo.com/aws-cli-configure/){:target="_blank"}
- [elastic 공식홈페이지 AWS S3 input](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-aws-s3.html){:target="_blank"}
