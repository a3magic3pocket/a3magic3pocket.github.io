---
title: Docker swarm mode을 이용하여 웹 애플리케이션 운영하기
date: 2023-02-08 23:56:00 +0900
categories: [docker]
tags: [docker, docker-swarm, git, git-action, dockerhub, cicd] # TAG names should always be lowercase
---
## docker swarm mode 란
- 컨테이너 오케스트레이션 툴이다.
    - 컨테이너 오케이션 툴
        - 배포, 관리, 확장, 네트워킹을 자동화한다.
        -  [컨테이너 오케스트레이션이란?](https://www.redhat.com/ko/topics/containers/what-is-container-orchestration){:target="_blank"}
- 기본적인 환경 설정
    - docker-compose yml 파일에 배포할 환경의 명세를 적는다.
    - 배포 명령을 실행하면 yml 파일대로 환경을 조성한다.
    - 주로 관리하는 기능은 아래와 같다.
        - 각 도커 이미지를 서비스화하여 컨테이너를 생성
            - 이때 컨테이너 수도 조절할 수 있다.
        - 컨테이너 별로 참조하는 네트워크를 세팅
        - 컨테이너가 참조하는 secret과 config 생성
- 클러스터 구축
    - Swarm을 쓰는 가장 큰 이유는  
      여러 서버로 클러스터를 구축하여 애플리케이션을 운영할 때  
      더 편리하게 관리하기 위함이다.
    - 노드는 서버(컴퓨터)를 의미한다.
    - Swarm 클러스터는 관리자 노드와 워커 노드로 구축할 수 있다.
    - '기본적인 환경 설정'은 관리자 노드에서 진행하며  
      관리자 노드는 docker-compose yml에 따라  
      다른 노드를 제어한다.
- 서비스 작동 방식
    - [서비스 작동 방식](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/){:target="_blank"}
    - Swarm에서 이미지를 배포하면 단일 컨테이너(Standalone container)가 아닌  
      서비스(Service)를 만들어 배포한다.
    - Swarm 명령어가 입력되면 관리자 노드는 서비스를 생성하고  
      docker-compose yml 명세에 따라 각 할당 가능한 노드 별로   
      적절히 컨테이너를 생성한다.
    - 이때 노드 별 할당된 컨테이너를 작업(Task)라고 명명한다.
    - ![3 nginx replicas](https://docs.docker.com/engine/swarm/images/services-diagram.png)


## 실험 환경
- 가정
    - 단일 노드에서 배포하는 것을 가정한다.
    - 노드 os는 ubuntu 22.04 이다.
    - 배포하고자 하는 웹 애플리케이션은 [simple-locker](https://www.coinlocker.link/){:target="_blank"}이다.
- simple-locker 구조
    - <a href="/assets/img/2023-02-08-docker-swarm-mode-deploy-application/00-simple-locker-structure.png" target="_blank"><img src="/assets/img/2023-02-08-docker-swarm-mode-deploy-application/00-simple-locker-structure.png" width="100%"></a> 
    - simple-locker 간단한 설명 [블로그 링크]
- docker swarm 설정 소스코드
    - [GitHub - a3magic3pocket/simple-docker-swarm: simple-docker-swarm](https://github.com/a3magic3pocket/simple-docker-swarm){:target="_blank"}
- docker-compose.yml 파일
    - ```yml
        {% raw %}
        # docker-compose 버전
        version: "3.9"
        
        # 서비스 명세
        services:
        
          # 서비스 이름
          web:
        
            # 이미지
            image: a3magic3pocket/simple-web:${WEB_TAG}
        
            # 환경변수
            # - "key=value"
            environment:
              - "FRONTEND_ORIGIN=coinlocker.link"
              - "BACKEND_ORIGIN=api.coinlocker.link"
        
            # 네트워크
            # - 같은 네트워크에 포함된 컨테이너들은 모든 포트로 통신할 수 있다.
            # - 설정된 네트워크를 참조한다.
            networks:
              - simple-net
        
            # 배포 옵션
            # (swarm 모드 시에만 동작하는 설정)
            deploy:
              mode: replicated
              replicas: 1
        
        
          # 서비스 이름
          api:
        
            # 이미지
            image: a3magic3pocket/simple-api:${API_TAG}
            
            # 환경변수
            environment:
              - "USE_K8S=true"
              - "FRONTEND_URL_LIST=https://www.coinlocker.link,https://coinlocker.link"
              - "GIN_MODE=release"
              - "IDENTITY_KEY=${IDENTITY_KEY}"
              - "AUTH_SECRET_KEY=${AUTH_SECRET_KEY}"
        
            # 네트워크
            # - 설정된 네트워크를 참조한다.
            networks:
              - simple-net
        
            # 볼륨
            # - 관리자 노드의 경로를 다른 서비스에서 마운트함
            volumes:
              - ./sqlite3:/root/api/sqlite3
        
            # 배포옵션
            deploy:
              mode: replicated
              replicas: 1
        
        
          # 서비스 이름
          nginx:
        
            # 이미지 
            image: nginx:1.23.3-alpine
        
            # 포트 
            # - 외부로 노출되는 port: 도커 컨테이너 포트
            ports:
              - 80:80
              - 443:443
        
            # 컨피그
            # - 설정된 컨피그를 참조한다.
            # - nginx-config 파일을 target 경로에 생성한다.
            configs:
              - source: nginx-config
                target: /etc/nginx/conf.d/default.conf
        
            # 네트워크   
            # - 설정된 네트워크를 참조한다.
            networks:
              - simple-net
        
            # 볼륨
            volumes:
              - /etc/letsencrypt/live/coinlocker.link/fullchain.pem:/etc/ssl/live/fullchain1.pem
              - /etc/letsencrypt/live/coinlocker.link/privkey.pem:/etc/ssl/live/privkey1.pem
        
            # 엑스트라 호스트
            # /etc/hosts 파일에 추가되는 호스트.
            # host-gateway	host.docker.internal 가 컨테이너 내부 /etc/hosts 파일에 추가된다.
            # host-gateway는 manager node의 docker private ip
            extra_hosts:
              - host.docker.internal:host-gateway
        
        # 네트워크 설정
        networks:
          simple-net:
        
        # 컨피그 설정
        configs:
          nginx-config:
            file: "./nginx.conf"
        {% endraw %}
      ```
- 간단한 yml 파일 설명
    - [yml 파일 작성법](https://docs.docker.com/compose/compose-file/compose-file-v3/){:target="_blank"}
    - service_name > image
        - 서비스 생성 시 참조하는 이미지이다.
        - 해당 URL로 pull 하여 서비스를 생성한다.
    - service_name > environment
        - 컨테이너 내에서 할당되는 환경변수이다.
        - 아래와 같이 할당한 경우  
          컨테이너 shell에서 echo "$FRONTEND_ORIGIN"이라는 명령을 내리면  
          coinlocker.link가 출력된다.  
          ```yml
            some_service
              environment:
                - "FRONTEND_ORIGIN=coinlocker.link"
          ```
    - service_name > ports
        - 외부로 노출되는 포트와 내부 도커 컨테이너 포트를 연결한다.
        - 아래와 같이 설정한 경우  
          외부에서 80 포트로 요청하면   
          도커 컨테이너에 1234 포트와 연결된 프로그램으로 요청이 전달된다.  
          ```yml
            some_service
              ports:
                  - 80:1234
          ```
    - service_name > networks
        - 동일이름의 network에 포함되는 컨테이너들은  
          모든 포트로 소통할 수 있다.
    - service_name > deploy
        - swarm 모드에서만 작동하는 설정이다.
        - mode
            - global과 replicated 를 고를 수 있따.
            - global은 노드 당 하나의 컨테이너만 쓴다는 의미
            - replicated는 노드 당 여러 개의 컨테이너를 쓸 수 있다는 의미
        - replicas
            - 서비스 내에서 동작해야하는 컨테이너 수를 의미힌다. 
    - service_name > volume
        - 컨테이너가 참조할 볼륨을 마운트 시키는 명령이다.
        - 아래와 같이 설정한 경우  
          manager 노드의 현 위치(명령을 실행한 위치)의 디스크를  
          컨테이너에서 /root/api/sqlite3라는 경로에 마운트하게 된다.
          ```yml
            some_service
              volumes:
                - ./sqlite3:/root/api/sqlite3
          ```
    - service_name > configs
        - 컨피그는 여러 서비스에서 공통으로 참조할만한 config 정보를  
          저장한다.
        - 서비스와 상관 없이 독립적으로 관리된다.
        - 아래와 같이 설정한 경우  
          컨테이너가 생성될 때 nginx-config 파일을   
          some_service의 /etc/nginx/conf.d/default.conf에  
          생성하게 된다.  
          ```yml
            some_service
              configs:
                  - source: nginx-config
                    target: /etc/nginx/conf.d/default.conf

            configs:
              nginx-config:
              file: "./nginx.conf"
          ```
    - service_name > secrets
        - 시크릿은 여러 서비스에서 공통으로 참조할만한 secret 정보를  
          저장한다.
        - 시크릿은 암호화되어 저장된다.
        - 사용 방법은 configs와 비슷하다.
    - service_name > extra_hosts:
        - 생성되는 컨테이너의 /etc/hosts에 호스트를 추가한다.
        - 아래와 같이 설정한 경우  
          host-gateway host.docker.internal 가   
          컨테이너 내부 /etc/hosts 파일에 추가된다.
          ```yml
            some_service
              extra_hosts:
                - host.docker.internal:host-gateway
          ```
    - ${API_TAG}, ${WEB_TAG}
        - docker-compose.yml 파일에서 참조하는 환경변수이다.
        - manager 노드에서 환경변수를 설정하면  
          docker-compse.yml에서 해당 변수를 가져와 반영한다.
          ```bash
            export API_TAG=0.0.7
            export WEB_TAG=0.0.19
          ```
- 기본적인 배포 방법
    - stack 배포
        - yml 파일에 기록된 모든 것을 자동으로 세팅해준다.
          ```bash
            # 환경변수 선언
            export API_TAG=0.0.7
            export WEB_TAG=0.0.19

            # stack 배포
            docker stack deploy -c docker-compose.yml [stack 이름]

            # stack 목록 조회
            docker stack ls

            # stack 삭제
            docker stack rm [stack 이름]
          ```
    - 서비스 이미지만 변경
        - 이미 배포된 서비스에서 이미지만 업데이트하는 것을 말한다.
          ```bash
            # service 목록 조회
            docker service ls

            # service 업데이트
            docker service update --image [이미지 URL] [service 이름]

            # service 삭제
            docker service rm [service 이름]
          ```
- deploy.sh
    - 설명
        - 배포 과정을 단순화하기 위해 모드에 따라 자동으로 처리되도록  
          작성한 스크립트
        - 모드는 stack, service:api, service:web이다.
        - 공통
            - simple-docker-swarm 하위의  
              simple-api-example과 simple-web-example의 git repository를  
              최신화한다.
            - 최신화된 git repository에서 각각 이미지 tag를 뽑은 뒤  
              현재 서비스에 사용하고 있는 이미지 tag와 비교한다.  
              비교한 결과 차이가 없다면 배포를 중단한다.  
        - stack
            - 같은 이름으로 stack 배포를 진행하면 바뀐 부분만 업데이트한다.
            - 그러나 가끔씩 이미 생성된 네트워크나 시크릿이 존재한다며  
              배포가 되지 않을 때가 있다.
            - 이를 막기 위해 여기서는 stack을 삭제하고 새로 배포한다.  
              이때 웹 접속이 잠시동안 되지 않는다.
        - service:api
            - simpel-api-example의 git repository에서  
              simpe-api.yml에 있는 image URL을 획득한다.
            - 해당 URL로 기존에 존재하는 service의 이미지를 업데이트한다.
        - service:web
            - simpel-web-example의 git repository에서  
              simpe-web.yml에 있는 image URL을 획득한다.
            - 해당 URL로 기존에 존재하는 service의 이미지를 업데이트한다.
    - 코드
        - deploy.sh
          ```bash
            #!/usr/bin/env bash
            STACK_NAME=a3
            WORKING_DIR_PATH="$(dirname $0)"
            API_DIR_PATH="${WORKING_DIR_PATH}/simple-api-example"
            WEB_DIR_PATH="${WORKING_DIR_PATH}/simple-web-example"
            export IDENTITY_KEY=`cat ${WORKING_DIR_PATH}/secrets/IDENTITY_KEY`
            export AUTH_SECRET_KEY=`cat ${WORKING_DIR_PATH}/secrets/AUTH_SECRET_KEY`
            
            # 초기화, git 최신화
            function git_init {
                git --git-dir=${API_DIR_PATH}/.git pull origin main
                git --git-dir=${WEB_DIR_PATH}/.git pull origin main
            }
            
            # API_TAG, WEB_TAG 할당
            function set_tags {
                api_image_url=`get_image_url ${API_DIR_PATH}/simple-api.yml api`
                web_image_url=`get_image_url ${WEB_DIR_PATH}/simple-web.yml web`
                export API_TAG=`echo ${api_image_url} | cut -d ":" -f 2`
                export WEB_TAG=`echo ${web_image_url} | cut -d ":" -f 2`
            }
            
            # 새 이미지가 업데이트 되었는지 확인
            function is_image_updated {
                local is_different=0
            
                declare -a arr=("simple-api:${API_TAG}" "simple-web:${WEB_TAG}")
            
                for row in "${arr[@]}"
                do
                    local prefix=${row%%:*}
                    local yml_tag=${row##*:}
            
                    local current_tag=`docker ps | grep ${prefix} | awk -F ' ' '{ print $2 }' | awk -F ':' '{ print $2 }'`
            
                    if [[ ${current_tag} != ${yml_tag} ]];then
                        is_different=1
                        break
                    fi
                done
            
                return ${is_different}
            }
            
            # 초기 세팅
            function init {
                git_init
                set_tags
                is_image_updated
            
                local is_different=$?
                if [[ ${is_different} -eq 0 ]];then
                    echo "image is not updated"
                    exit 1;
                fi
            }
            
            # api, web manifest 파일에서 이미지 URL 추출
            function get_image_url {
                local yml_file_path=$1
                local keyword=$2
            
                line=`cat ${yml_file_path} | grep "image: " | grep ${keyword}`
                prefix=`echo ${line} | cut -d ":" -f 2`
                tag=`echo ${line} | cut -d ":" -f 3`
            
                echo ${prefix}:${tag}
            }
            
            # 스택배포
            function deploy_stack {
                docker stack rm ${STACK_NAME}
                
                while [[ $(docker network ls | grep "${STACK_NAME}_" | wc -c) -ne 0 ]];
                do
                    sleep 1;
                done
                
                docker stack deploy -c docker-compose.yml ${STACK_NAME}
            }
            
            # 서비스배포
            function deploy_service {
                local service_suffix=$1
                local image_url=""
                if [[ $service_suffix = "api" ]];then
                    image_url=`get_image_url ${API_DIR_PATH}/simple-api.yml api`
                elif [[ $service_suffix = "web" ]];then
                    image_url=`get_image_url ${WEB_DIR_PATH}/simple-web.yml web`
                else
                    echo "deploy_service::service_suffix is not allowed. ${service_suffix}"
                    exit 1;
                fi
            
                docker service update --image ${image_url} "${STACK_NAME}_${service_suffix}"
            }
            
            # 도움말
            function help {
                echo "+----- HOW TO RUN -----+"
                echo "ex) bash deploy.sh <mode>"
                echo "    allowed mode = [stack, service:api, service:web]"
                echo "+----------------------+"
            }
            
            # secret 파일 존재 유무 확인
            if [[ ${IDENTITY_KEY} = "" ]]; then
                echo "${WORKING_DIR_PATH}/secrets/IDENTITY_KEY is empty"
                exit 1
            fi
            if [[ ${AUTH_SECRET_KEY} = "" ]]; then
                echo "${WORKING_DIR_PATH}/secrets/AUTH_SECRET_KEY is empty"
                exit 1
            fi
            
            # 실행
            if [[ $1 = "stack" ]];then
                init
                deploy_stack
            elif [[ $1 = "service:api" ]];then
                init
                deploy_service "api"
            elif [[ $1 = "service:web" ]];then 
                init
                deploy_service "web"
            else
                echo "mode is empty"
                help
            fi
          ```
          
- nginx 설정
    - 설명
        - 리버스 프록시 서버
        - server_name에 따라 api와 web, webhook 서버로 각각 전달한다.
        - 여기서 api, web은 docker-compose에서 설정한 host name이다.
        - host.docker.internal은 manger node의 docker private ip로  
          webhook 서버가 manager node에서 실행 중이므로  
          nginx는 host.docker.internal을 통해 webhook 서버로 요청을 넘긴다.
        - 80 포트로 들어온 모든 http 요청은  433포트로 redirect 한다.
    - 코드
        - nginx.conf
          ```bash
            server {
                listen 80 default_server;
            
                server_name _;
            
                return 301 https://$host$request_uri;
            }
            
            server {
                listen 443 ssl;
            
                server_name api.coinlocker.link;
                underscores_in_headers on;
                server_tokens off;
                #ignore_invalid_headers off;
            
                ssl_certificate /etc/ssl/live/fullchain1.pem;
                ssl_certificate_key /etc/ssl/live/privkey1.pem;
            
                location / {
                    proxy_set_header    host    $host;
                    proxy_set_header    X-Real-IP           $remote_addr;
                    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;        
                    proxy_pass_request_headers  on;
                    proxy_pass  http://api:8080/;
                }
            }
            
            server {
                listen 443 ssl;
            
                server_name webhook.coinlocker.link;
                underscores_in_headers on;
                server_tokens off;
                #ignore_invalid_headers off;
            
                ssl_certificate /etc/ssl/live/fullchain1.pem;
                ssl_certificate_key /etc/ssl/live/privkey1.pem;
            
                location / {
                    proxy_set_header    host    $host;
                    proxy_set_header    X-Real-IP           $remote_addr;
                    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;        
                    proxy_pass_request_headers  on;
                    proxy_pass  http://host.docker.internal:9000;
                }
            }
            
            server {
                listen 443 ssl;
            
                server_name www.coinlocker.link coinlocker.link;
                underscores_in_headers on;
                server_tokens off;
                #ignore_invalid_headers off;
            
                ssl_certificate /etc/ssl/live/fullchain1.pem;
                ssl_certificate_key /etc/ssl/live/privkey1.pem;
            
                location / {
                    proxy_set_header    host    $host;
                    proxy_set_header    X-Real-IP           $remote_addr;
                    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;        
                    proxy_pass_request_headers  on;
                    proxy_pass http://web:3000;
                }
            }
          ```
- Webhook 서버 실행
    - [git action과 dockerhub을 이용한 CI/CD 파이프라인 구축](https://a3magic3pocket.github.io/posts/cicd-using-git-action-and-dockerhub/){:target="_blank"}
    - CI / CD 파이프라인 중 한 모듈로  
      dockerhub에서 새 이미지가 push되어 보내는 webhook 명령을  
      받아서 실제 서버에 배포하는 서버이다.
- ssl 갱신
    - ssl 인증서는 certbot을 이용하여 생성 및 갱신하고 있다.
    - cerbot은 dns 인증방식을 통해 ssl을 인증 받았으며  
      갱신은 crontab으로 등록하여 주기적으로 갱신되도록 하였다.
    - cerbot은 manager 노드에서 실행되어 ssl 인증서를 생성 및 갱신한다.  
      nginx 서비스는 ssl이 포함된 경로를 마운트하여 ssl을 지원한다.
    - set_ssh.sh
      ```bash
        function init_ssl {
            # Reference by:
            #   https://lynlab.co.kr/blog/72
            #   https://www.lesstif.com/system-admin/dns-txt-record-let-s-encrypt-ssl-59343172.html
            #   https://oasisfores.com/letsencrypt-wildcard-ssl-certificate/
            docker run -it --rm --name certbot \
              -v '/etc/letsencrypt:/etc/letsencrypt' \
              -v '/var/lib/letsencrypt:/var/lib/letsencrypt' \
              certbot/certbot certonly -d '*.coinlocker.link' -d 'coinlocker.link' --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory
        }

        function renew_ssl {
            docker run -it --rm --name certbot \
              -v '/etc/letsencrypt:/etc/letsencrypt' \
              -v '/var/lib/letsencrypt:/var/lib/letsencrypt' \
              certbot/certbot renew --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory
        }

        if [[ $1 = "init" ]]; then
            init_ssl
        elif [[ $1 = "renew" ]]; then
            renew_ssl
        else
            echo "argument is not allowed"
            echo "allowed argument: [init, renew]"
            echo "ex. bash set_ssl.sh init"
        fi
      ```
      
    - crontab  
      ```bash
        0 0 1 */2 * /usr/bin/env bash /home/ubuntu/simple-docker-swarm/set_ssl.sh renew && docker service update a3_nginx
      ```
## 실험시작
- swarm mode 시작  
  ```bash
    docker swarm init
  ```
- github 소스 내려 받기  
  ```bash
    # git clone
    git clone https://github.com/a3magic3pocket/simple-docker-swarm.git

    # clone 받은 디렉토리로 이동
    cd ./simple-docker-swarm
  ```
- 의존성 패키지 설치  
  ```bash
    sudo apt-get update sudo apt-get upgrade 

    # 도커 설치 
    curl -fsSL https://get.docker.com -o get-docker.sh DRY_RUN=1 sudo sh ./get-docker.sh 

    # 웹훅 서버 설치 
    sudo apt-get install webhook
  ```
- Webhook 서버 실행  
  ```bash
    bash run_webhook.sh
  ```
- docker-compose를 바탕으로 stack 생성  
  ```bash
    bash deploy.sh stack
  ```
- 잘 생성되었나 확인  
  ```bash
    # stack 생성 확인
    docker stack ls

    # service 생성 확인
    # 이때 서비스의 REPLICAS가 0/1 이면 아직 생성되고 있다는 뜻
    docker service ls

    # watch 명령어로 모든 서비스가 1/1이 될때까지 대기 후 확인
    watch docker service ls
  ```
- 다중 노드 사용 시 유의사항
- 설명
    - 다중 노드로 애플리케이션을 배포 및 운영하고 계신  
      동료 프로그래머에게 몇 가지 유의사항을 들어 기록한다.
- replicas 수
    - 다중 노드 사용 시  
      manager 노드가 할당 가능한 노드에 replicas를 임의로 분배한다.
    - max_replicas_per_node를 설정해주면  
      한 노드 당 가질 수 있는 최대 replicas를 설정하여 골고루 분배할 수 있다.
    - 만일 (할당 가능한 노드 * max_replicas_per_node 수 < 서비스 내의   
      모든 replicas 수) 이면 에러가 나게 되니 주의
    - 명령어  
      ```yml
        some_service:
          deploy:
            mode: replicated
            replicas: 6
            placement:
              max_replicas_per_node: 1
      ```
- 노드 status 세팅
    - reachable 상태로 두면  
      만약 manager 노드가 죽었을 경우  
      reachable 노드가 대신 manager가 된다.  
    - worker 노드는 manager 노드가 될 수 없다.
    - 명령어  
      ```bash
        # 노드 상태 보기
        # - 여기서 'MANAGER STATUS'를 참고하면 된다.
        docker node ls

        # node를 worker node로 만들기
        docker node demote [node id]

        # node를 reachable node로 승격시키기
        docker node promote [node id]
      ```
## 참고
- [Docker Swarm의 주요 용어, 활성화 방법 및 노드(Node) 관리법 살펴보기](https://seongjin.me/docker-swarm-introduction-nodes/){:target="_blank"}
-  [컨테이너 오케스트레이션이란?](https://www.redhat.com/ko/topics/containers/what-is-container-orchestration){:target="_blank"}
- [서비스 작동 방식](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/){:target="_blank"}
- [yml 파일 작성법](https://docs.docker.com/compose/compose-file/compose-file-v3/){:target="_blank"}
