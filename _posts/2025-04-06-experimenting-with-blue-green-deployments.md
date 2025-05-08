---
title: 블루 - 그린 배포 실험
date: 2025-04-06 08:47:25 +0900
categories: [Deployment]
tags: [blue-green, essay]    # TAG names should always be lowercase
---

## 블루 - 그린 배포란?
- 두 개의 동일한 환경(블루와 그린)을 유지하며,   
  새 버전을 배포한 후 트래픽을 전환하여 새로운 버전을 운영에 적용  

## 왜 하는가?
- 새로운 애플리케이션 버전을 배포하면서 서비스 중단 없이  
  안전하게 전환하기 위해서  

## 작동 방식
1. 블루 환경에서 운영 중  
    - 기존 애플리케이션(블루)이 사용자의 모든 트래픽을 처리하고 있음  
2. 그린 환경에서 새 버전 배포  
    - 새로운 애플리케이션 버전을 그린 환경에 배포한 뒤 테스트 수행  
3. 트래픽 전환  
    - 모든 트래픽을 블루 환경에서 그린 환경으로 전환  
    - 이 작업은 로드밸런서를 재설정하거나 DNS를 업데이트 하는 방식으로 수행  
4. 블루 환경 유지(롤백 대비)  
    - 그린 환경으로의 전환 후에도 블루 환경은 일정 시간 동안 유지하며  
      문제가 발생하면 즉시 롤백할 수 있도록 준비  

## 장점
- 서비스 중단 없이 배포  
    - 기존 환경은 유지되므로, 배포 중에도 사용자는 끊김 없는 서비스 이용 가능  
- 빠른 롤백 가능  
    - 문제가 발생하면 그린 환경을 비활성하고 다시 블루로 트래픽 전환 가능  
- 테스트 환경 제공  
    - 그린 환경에서 운영 환경과 동일한 조건으로 충분히 테스트 진행 가능  
- 배포 리스크 최소화  
    - 새로운 버전 문제로 인해 운영 환경이 중단될 가능성을 크게 줄임  

## 단점
- 추가 리소스 비용  
    - 블루와 그린 두 개 환경을 동시에 유지해야 하므로 비용 증가  
- 데이터 동기화  
    - 배포 시 데이터베이스와 같은 상태 정보를   
      공유하거나 동기화하는 작업이 필요  

## 온프레미스 환경 블루 - 그린 배포를 한다면?
- 가정  
    - 물리 서버 3대가 있다.  
    - 서버1  
        - nginx 로드밸런서  
    - 서버2, 서버3  
        - 스프링 서버  
        - docker compose 로 블루, 그린 환경을 각각 구성한다.  
        - 현재 배포된 도커 이미지는 my-server:0.0.1 이다.  
    - 서버1에서 테스트 용 PC 접근 경로 설정  
        - 테스트용 PC IP만 블루 또는 그린에   
          선택적으로 접근할 수 있도록 nginx 설정  
- (1) 블루 환경에서 운영 중  
- (2) 그린 환경에서 새 버전 배포  
    - 서버2, 서버3에서 새 버전 my-server:0.0.2를 확보한다.  
    - 새 버전 my-server:0.0.2는   
      docker-registry 서버에 pull 하는 방식으로 가져올 수도 있고  
      직접 빌드하여 만드는 방법을 사용할 수도 있다.  
    - 블루 환경 컨테이너는 그대로 둔 채  
      그린 환경 컨테이너를 my-server:0.0.2로 다시 올린다.  
    - 테스트용 PC에서 테스트를 진행한다.  
- (3) 트래픽 전환  
    - nginx 로드밸런서에서 트래픽이 그린을 바라보도록  
      설정을 변경하고 리로드 한다.  
- (4) 블루 환경 유지(롤백 대비)  

## 로컬에서 온프레미스 환경 블루 - 그린 배포 실험
- 가정  
    - 물리 서버 대신 도커 컨테이너를 띄워서 진행한다.  
    - 컨테이너 구성  
        - nginx 로드밸런서 컨테이너  
        - 서버1 블루 컨테이너  
        - 서버1 그린 컨테이너  
        - 서버2 블루 컨테이너  
        - 서버2 그린 컨테이너  
- github repository  
    - [blue-green-deployment-sandbox](https://github.com/a3magic3pocket/blue-green-deployment-sandbox){:target="_blank"}  
    - 앞으로의 모든 설명은 blue-green_manifest 디렉토리 하위를 기준으로 한다.  
- 준비  
    - 서버  
        - 설명  
            - 스프링 서버  
            - / 로 접근 시 application.yml에 설정한 deployment 문자열을 출력한다.  
            - / 로 접근 시 블루면 blue가 표시되고 그린이면 green이 표시된다.  
        - 빌드 파일 위치  
            - spring-0.0.1-SNAPSHOT.jar  
        - server-blue:0.0.1 이미지 생성  
          ```bash  
          docker build -t server-blue:0.0.1 -f ./Dockerfile-server .  
          ```  
        - server-green:0.0.1 이미지 생성  
          ```bash  
          docker build -t server-green:0.0.1 -f ./Dockerfile-server .  
          ```  
    - nginx 로드밸런서 설정  
        - 설명  
            - blue 버전에 트래픽 분배하는 설정파일과  
              green 버전에 트래픽 분배하는 설정파일을 준비한다.  
            - 트래픽 분배 대상을 변경하고 싶다면  
              nginx 설정을 변경한 후 nginx를 재시작한다.  
            - blue 버전에 트래픽 분배하는 설정 위치  
                - blue-green_manifest/loadbalancer-nginx-blue.conf  
            - green 버전에 트래픽 분배하는 설정 위치  
                - blue-green_manifest/loadbalancer-nginx-green.conf  
        - blue 버전에 트래픽 분배하는 설정  
          ```text  
          # loadbalancer-nginx-blue.conf  
          upstream blue-upstream {  
              server server1-blue:8080;  
              server server2-blue:8080;  
          }  
                    
                    
          upstream green-upstream {  
              server server1-green:8080;  
              server server2-green:8080;  
          }  
                    
                    
          server {  
              listen 80;  
                    
                    
              location / {  
                  # 기본적으로 Blue 환경으로 라우팅  
                  proxy_pass http://blue-upstream;  
              }  
                    
                    
              location /blue {  
                  # 특정 IP만 Blue에 접근 가능  
                  # allow test.pc.ip;  # 허용할 IP  
                  # deny all;              # 다른 모든 IP는 거부  
                    
                    
                  rewrite ^/blue/?(.*)?$ /$1 break;  
                  proxy_pass http://blue-upstream/;  # Blue 환경으로 라우팅  
              }  
                    
                    
              location /green {  
                  # 특정 IP만 Green에 접근 가능  
                  # allow test.pc.ip;  # 허용할 IP  
                  # deny all;              # 다른 모든 IP는 거부  
                    
                    
                  rewrite ^/green/?(.*)?$ /$1 break;  
                  proxy_pass http://green-upstream/;  # Green 환경으로 라우팅  
              }  
          }  
          ```  
        - green 버전에 트래픽 분배하는 설정  
          ```text  
          # loadbalancer-nginx-green.conf  
          upstream blue-upstream {  
              server server1-blue:8080;  
              server server2-blue:8080;  
          }  
                    
                    
          upstream green-upstream {  
              server server1-green:8080;  
              server server2-green:8080;  
          }  
                    
                    
          server {  
              listen 80;  
                    
                    
              location / {  
                  # 기본적으로 Green 환경으로 라우팅  
                  proxy_pass http://green-upstream;  
              }  
                    
                    
              location /blue {  
                  # 특정 IP만 Blue에 접근 가능  
                  # allow test.pc.ip;  # 허용할 IP  
                  # deny all;              # 다른 모든 IP는 거부  
                    
                    
                  rewrite ^/blue/?(.*)?$ /$1 break;  
                  proxy_pass http://blue-upstream/;  # Blue 환경으로 라우팅  
              }  
                    
                    
              location /green {  
                  # 특정 IP만 Green에 접근 가능  
                  # allow test.pc.ip;  # 허용할 IP  
                  # deny all;              # 다른 모든 IP는 거부  
                    
                    
                  rewrite ^/green/?(.*)?$ /$1 break;  
                  proxy_pass http://green-upstream/;  # Green 환경으로 라우팅  
              }  
          }  
          ```  
    - docker-compose.yml  
        - 설명  
            - nginx와 server 의 컨테이너를 제어하는 docker-compose.yml이다.  
        - docker-compose.yml  
          ```yml  
          version: '3.8'  
                    
                    
          services:  
            loadbalancer:  
              image: nginx:1.25.3-alpine  
              container_name: loadbalancer    
              ports:  
                - "80:80"  
              volumes:  
                - ./loadbalancer-nginx-blue.conf:/etc/nginx/sites-available/loadbalancer-nginx-blue.conf  
                - ./loadbalancer-nginx-green.conf:/etc/nginx/sites-available/loadbalancer-nginx-green.conf  
              depends_on:  
                - server1-blue  
                - server1-green  
                - server2-blue  
                - server2-green  
              extra_hosts:  
                - "host.docker.internal:host-gateway"  
              networks:  
                - plain-deployment-net  
                    
                    
            server1-blue:  
              image: server-blue:0.0.1  
              container_name: server1-blue  
              command:  
                - "java"  
                - "-jar"  
                - "/spring.jar"  
                - "--deployment=blue"  
              networks:  
                - plain-deployment-net  
                    
                    
            server1-green:  
              image: server-green:0.0.1  
              container_name: server1-green  
              command:  
                - "java"  
                - "-jar"  
                - "/spring.jar"  
                - "--deployment=green"  
              networks:  
                - plain-deployment-net  
                    
                    
            server2-blue:  
              image: server-blue:0.0.1  
              container_name: server2-blue  
              command:  
                - "java"  
                - "-jar"  
                - "/spring.jar"  
                - "--deployment=blue"  
              networks:  
                - plain-deployment-net  
                    
                    
            server2-green:  
              image: server-green:0.0.1  
              container_name: server2-green  
              command:  
                - "java"  
                - "-jar"  
                - "/spring.jar"  
                - "--deployment=green"  
              networks:  
                - plain-deployment-net  
                    
                    
          networks:  
            plain-deployment-net:  
          ```  
        - 명령  
          ```bash  
          # 실행  
          docker compose -f ./docker-compose.yml up  
                    
          # 종료  
          docker compose -f ./docker-compose.yml down  
          ```  
- 로드밸런서 nginx에서 블루로 트래픽 분배  
    - 명령  
      ```bash  
      docker compose -f ./docker-compose.yml exec loadbalancer sh -c "  
        mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak   
        rm -rf /etc/nginx/conf.d/*.conf  
        ln -s /etc/nginx/sites-available/loadbalancer-nginx-blue.conf /etc/nginx/conf.d/  
        nginx -t  
        nginx -s reload  
      "  
      ```  
    - 확인  
        - 브라우저에서 http://localhost 으로 들어가   
          blue 문자열이 나오는지 확인  
- 새로운 그린 이미지를 빌드하고   
  로드밸런서 nginx에서 트래픽을 블루에서 그린으로 변경  
    - 새로운 그린 이미지 빌드 (server-green:0.0.2)  
      ```bash  
      docker build -t server-green:0.0.2 -f ./Dockerfile-server .  
      ```  
    - docker-compose.yml에서 그린 이미지를   
      새로 빌드한 이미지(server-green:0.0.2)로 변경  
      ```yml  
      version: '3.8'  
                
                
      services:  
        loadbalancer:  
          image: nginx:1.25.3-alpine  
          container_name: loadbalancer    
          ports:  
            - "80:80"  
          volumes:  
            - ./loadbalancer-nginx-blue.conf:/etc/nginx/sites-available/loadbalancer-nginx-blue.conf  
            - ./loadbalancer-nginx-green.conf:/etc/nginx/sites-available/loadbalancer-nginx-green.conf  
          depends_on:  
            - server1-blue  
            - server1-green  
            - server2-blue  
            - server2-green  
          extra_hosts:  
            - "host.docker.internal:host-gateway"  
          networks:  
            - plain-deployment-net  
                
                
        server1-blue:  
          image: server-blue:0.0.1  
          container_name: server1-blue  
          command:  
            - "java"  
            - "-jar"  
            - "/spring.jar"  
            - "--deployment=blue"  
          networks:  
            - plain-deployment-net  
                
                
        server1-green:  
          # | -- 수정 시작 -- |
          image: server-green:0.0.2  
          # | -- 수정 끝 -- |
          container_name: server1-green  
          command:  
            - "java"  
            - "-jar"  
            - "/spring.jar"  
            - "--deployment=green"  
          networks:  
            - plain-deployment-net  
                
                
        server2-blue:  
          image: server-blue:0.0.1  
          container_name: server2-blue  
          command:  
            - "java"  
            - "-jar"  
            - "/spring.jar"  
            - "--deployment=blue"  
          networks:  
            - plain-deployment-net  
                
                
        server2-green:  
          # | -- 수정 시작 -- |
          image: server-green:0.0.2  
          # | -- 수정 끝 -- |
          container_name: server2-green  
          command:  
            - "java"  
            - "-jar"  
            - "/spring.jar"  
            - "--deployment=green"  
          networks:  
            - plain-deployment-net  
                
                
      networks:  
        plain-deployment-net:  
      ```  
    - sever1-green에 변경 사항 적용  
      (server1-green 컨테이너 이미지를 server-green:0.0.2 로 변경)  
      ```bash  
      docker-compose up -d --no-deps --force-recreate server1-green  
      ```  
    - server2-green에 변경 사항 적용  
      (server2-green 컨테이너 이미지를 server-green:0.0.2 로 변경)  
      ```bash  
      docker-compose up -d --no-deps --force-recreate server1-green  
      ```  
    - 테스트  
        - sever1-green, sever2-green 컨테이너에 적용된  
          이미지가 잘 교체되었는지 확인  
          ```bash  
          docker ps | grep green  | sed 's/  */ /g' | awk '{ print $2, $12 }'  
                    
          ## 결과가 아래와 같이 나와야 함  
          # server-green:0.0.2 server2-green  
          # server-green:0.0.2 server1-green  
          ```  
        - 브라우저에서 http://localhost/green 으로 들어가   
          green 문자열이 나오는지 확인  
    - 로드밸런서 nginx에서 트래픽을 그린으로 분배하도록 변경  
        - 명령  
          ```bash  
          docker compose -f ./docker-compose.yml exec loadbalancer sh -c "  
            mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak   
            rm -rf /etc/nginx/conf.d/*.conf  
            ln -s /etc/nginx/sites-available/loadbalancer-nginx-green.conf /etc/nginx/conf.d/  
            nginx -t  
            nginx -s reload  
          "  
          ```  
        - 확인  
            - 브라우저에서 http://localhost 으로 들어가   
              green 문자열이 나오는지 확인  

## 주의할 점
- 블루-그린 데이터 동기화를 위해   
  두 버전에 호환되게 스키마를 운영해야한다.  
    - 기존 컬럼 삭제  
        - 버전 배포 전까지 기존 컬럼을 DB에서 바로 삭제하지 않고,   
          애플리케이션에서만 제거  
        - 오랜 시간 운영 후, 해당 컬럼을 참조하는 코드가   
          완전히 제거된 다음 버전이 배포되면 DB에서 컬럼 삭제  
    - 컬럼 수정 (VARCHAR → JSON 예시)  
        - 새로운 컬럼(contents_new) 추가 (기존 contents 컬럼 유지)  
        - 기존 데이터를 contents_new로 마이그레이션  
        - 새 버전 애플리케이션에서 contents_new 컬럼을 사용하도록 수정  
        - 충분한 운영 후 contents 컬럼이 더 이상 사용되지 않으면 삭제  
- 블루-그린 배포에서 DB 사용 방식  
    - 하나의 DB 사용  
        - 트래픽 변경 시 애플리케이션만 전환하면 되므로 편리하다.  
        - 대규모 테이블 변경 시 라이브 서비스에 영향 가능성이 있다.  
    - 별도의 DB 사용  
        - 새 버전 배포 시 기존 서비스(DB)에 영향을 주지 않는다.  
        - 두 버전이 서로 다른 DB를 사용하므로   
          데이터 동기화를 위한 CDC(Kafka Connect 등) 필요하다.  
        - DB가 버전 별로 필요하고 CDC도 사용해야 하므로  
          리소스 소비가 상대적으로 많다.  
        - 동기화 문제로 인해 롤백이 더 복잡하다.  

## 예시 repository
- [blue-green-deployment-sandbox](https://github.com/a3magic3pocket/blue-green-deployment-sandbox){:target="_blank"}  
