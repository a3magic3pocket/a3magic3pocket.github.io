---
title: logstash에 grok pattern 적용하기
date: 2021-10-06 17:25:00 +0900
categories: [Elastic]
tags: [logstash, grok, regular expression] # TAG names should always be lowercase
---

## 개요
- logstash filter를 쓸 때 grok pattern을 사용하는 법을 익힌다.
- [logstash grok 관련 문서](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)

## logstash 정형화 방법
- logstash는 비정형 데이터를 input으로 받았을 때 filter를 통해 정형 데이터로 바꿀 수 있다.
- 이때 dissect나 grok을 쓸 수 있는데,  
  dissect는 구분자(delimiter)를 이용하여 정형화하는 방법이고,  
  grok은 정규식을 이용하여 정형화하는 방법이다.
- 여기서는 grok만 알아본다.

## Grok Basic
- grok 문법
    - %{SYNTAX:SEMANTIC}
- 상세
    - SYNTAX
        - logstash input에서 감지해야하는 정규식 패턴이다.
    - SEMANTIC
        - 감지된 패턴 데이터를 할당할 logstash 변수명(identifier, 식별자)이다.
- SYNTAX 사용법
    - 기본 패턴 사전
        - [logstash 기본 패턴 사전](https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns)에 등록된 패턴을 SYNTAX로 쓸 수 있다.
        - 예를 들어 단어를 SYNTAX로 쓰고 싶은 경우, SYNTAX로 DATA를 쓸 수 있다.
        - 기본 패턴 사전에 등록된 DATA의 정규식은 DATA .*?이다.
    - 커스텀 패턴 사전
        - logstash는 /usr/share/logstash/pipeline/(이하 pipeline) 디렉토리에서 설정파일을 둔다.
        - grok 패턴 사전의 경우, pipeline 디렉토리 하위에 patterns 디렉토리를 만들고 그 안에 파일을 넣어 사용한다.
        - 파일은 확장자 없이 생성하며, 모든 파일이 logstash에 import 된다.
- 예시
    - logstash input에서 아래와 같은 문장이 왔다고 하자.
        - 김수미 킹왕짱
    - 아래와 같은 grok 패턴을 사용했다.
        - %{DATA:kimsumi} %{DATA:king}
    - 결과
        - kimsumi 변수에 "김수미" 할당
        - king 변수에 "킹왕짱" 할당

## Test Grok Patterns
- 개요
    - 웹에서 grok pattern을 테스트 할 수 있는 사이트이다.
    - [링크](https://grokconstructor.appspot.com/do/match#result)
- 사용법
    - Some log ~~로 시작하는 input에 input 데이터를 넣는다.
    - The (unquoted!) pattern ~~로 시작하는 input에 grok 패턴을 넣는다.
    - 위의 go 버튼을 클릭한다.
    - <a href="/assets/img/2021-10-06-logstash-grok-pattern/00-grok-pattern-test.jpg" target="_blank"><img src="/assets/img/2021-10-06-logstash-grok-pattern/00-grok-pattern-test.jpg" width="100%"></a> 
- 결과
    - <a href="/assets/img/2021-10-06-logstash-grok-pattern/01-grok-pattern-test-result.jpg" target="_blank"><img src="/assets/img/2021-10-06-logstash-grok-pattern/01-grok-pattern-test-result.jpg" width="100%"></a>

## gin log 파싱해보기
- gin log 예시
    - [GIN] 2021/10/05 - 09:04:47 \| 200 \|            0s \|             ::1 \| GET      "/now"
    - "김수미 킹왕짱" 예시보다 복잡하지만,  
      기본 패턴 사전을 잘 활용하여 커스텀 패턴을 만들어 해결할 수 있다.
    - grok에서 "[", "]", '"', "\|"는 escaping을 해줘야한다.
    - SEMANTIC를 대괄호를 사용하여 지정하면 계층식 변수 선언이 가능하다.
        - 예를 들어 [gin][type]으로 했다면 { gin: { type: somthing } } 식으로 할당된다.
        - !주의 - Test grok pattern에서는 대괄호를 사용한 SEMANTIC 선언이 되지 않는다.
    - 아래의 GINLOG에서 사용된 \s*는 파싱 시 whitespace를 제거하기 위해 사용하였다.
- pattern 예시
    - GIN_TIMESTAMP %{YEAR}/%{MONTHNUM}/%{MONTHDAY} - %{HOUR}:%{MINUTE}:%{SECOND}
    - GINLOG \[%{DATA:[gin][type]}\] %{GIN_TIMESTAMP:[gin][time]} \| %{NUMBER:[gin][response_status]} \| \s*%{DATA:[gin][response_time]}\s* \| \s*%{IPORHOST:[gin][remote_ip]}\s* \| \s*%{DATA:[gin][method]}\s* \"%{DATA:[gin][url]}\"

## logtstash에 적용해보기
- 개요
    - 이제 실제로 logstash.conf에 grok 패턴을 적용시켜보자.
    - 예시에서는 문제를 간단하게 하기 위해 logstash.input을 파일로 지정한다.
- logstash config 디렉토리
    - 아래와 같은 구조로 디렉토리 및 파일을 만든다.
        - ```
         
            - logstash
                - patterns
                    - gin
                - logstash.conf
                 
            ```
    - gin grok 패턴
        - ```
            # gin
            GIN_TIMESTAMP %{YEAR}/%{MONTHNUM}/%{MONTHDAY} - %{HOUR}:%{MINUTE}:%{SECOND}
            GINLOG \[%{DATA:[gin][type]}\] %{GIN_TIMESTAMP:[gin][time]} \| %{NUMBER:[gin][response_status]} \| \s*%{DATA:[gin][response_time]}\s* \| \s*%{IPORHOST:[gin][remote_ip]}\s* \| \s*%{DATA:[gin][method]}\s* \"%{DATA:[gin][url]}\"
            ```
    - logstash 설정 파일
        - ```
            # logstash.conf
            input {
                file {
                    path => ["/usr/share/logstash/gin.log"]
                    sincedb_path => "/dev/null"
                    start_position => "beginning"
                }
            }

            filter {
                grok {
                    patterns_dir => ["/usr/share/logstash/pipeline/patterns"]
                    match => {"message" => ["%{GINLOG}"]}
                    remove_field => "message"
                    remove_field => "[gin][type]"
                }
            }

            output {
                stdout {
                    codec => rubydebug
                }
            }
            ```
- 예시 파일
    - 최상위 디렉토리 하위에 gin.log 파일을 생성한다.
    - ```
        # gin.log
        [GIN] 2021/10/05 - 09:04:37 | 200 |     27.3227ms |             ::1 | GET      "/now"
        [GIN] 2021/10/05 - 09:04:46 | 200 |     26.6912ms |             ::1 | GET      "/now"
        [GIN] 2021/10/05 - 09:04:46 | 200 |            0s |             ::1 | GET      "/now"
        [GIN] 2021/10/05 - 09:04:46 | 200 |            0s |             ::1 | GET      "/now"
        [GIN] 2021/10/05 - 09:04:46 | 200 |            0s |             ::1 | GET      "/now"
        [GIN] 2021/10/05 - 09:04:47 | 200 |            0s |             ::1 | GET      "/now"
        [GIN] 2021/10/05 - 09:04:47 | 200 |            0s |             ::1 | GET      "/now"
        ```
- logstash 컨테이너 생성
    - ```bash
        # windows 명령어이다. 만약 linux나 os X를 사용한다면 %cd%를 .으로 변경
        > docker run \
          --name es-c-logstash \
          --rm \
          -v %cd%/gin.log:/usr/share/logstash/gin.log \
          -v %cd%/logstash/:/usr/share/logstash/pipeline/ \
          docker.elastic.co/logstash/logstash:7.15.0
        ```
- 결과
    - ```
        ...
        {
            "@timestamp" => 2021-10-05T09:04:46.000Z,
            "gin" => {
                "response_status" => "200",
                "remote_ip" => "::1",
                "response_time" => "0s",
                "url" => "/now",
                "method" => "GET"
            },
            "path" => "/usr/share/logstash/gin.log",
            "host" => "8483a7f58c1d",
            "@version" => "1"
        }
        ...
        ```

## 참고
- [logstash grok 관련 문서](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)
- [logstash 기본 패턴 사전](https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns)
- [test grok pattern](https://grokconstructor.appspot.com/do/match#result)