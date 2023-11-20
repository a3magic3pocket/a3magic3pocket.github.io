---
title: certbot, DNS 인증 시 자동 갱신하게 만들기(AWS Route 53)
date: 2023-11-20 18:29:45 +0900
categories: [certbot]
tags: [certbot, route53, nginx]    # TAG names should always be lowercase
---

## 설명
- certbot에서는 여러 방식으로 웹서버 소유권 인증을 지원한다.  
  [getting-certificates-and-choosing-plugins](https://eff-certbot.readthedocs.io/en/stable/using.html#getting-certificates-and-choosing-plugins){:target="_blank"}  
- 와일드카드 인증서를 발급받기 위해서는 DNS를 사용하여   
  인증을 받아야 한다.  
- 여러 DNS를 지원하지만 여기서는 AWS Route53 기준으로 설명한다.  
- 최초 발급 방법과 자동갱신 방법을 함께 설명한다.  
- 예시에서 인증서를 발급할 도메인은 helloworld.com이라고 가정한다.  

## 기본 방식
- 서버에서 nginx 도커를 띄워 웹을 띄운다.  
- cerbot 도커를 이용하여 인증서를 획득한다.  
- 자동갱신을 위해 nginx 도커 인증서 경로를   
  호스트서버의 cerbot 인증서 경로와 연결한다.  

## certbot DNS 인증
- certbot 인증 전 과정  
    - nginx를 띄우고 Route 53 DNS에서 서버IP와 도메인을 연결한다.  
    - 이를 통해 도메인을 입력하여 서버로 http 접속이 가능한 상태가 된다.  
- certbot DNS 인증 및 인증서 발급  
    - 명령  
      ```bash  
          docker run -it --rm --name certbot \  
            -v '/etc/letsencrypt:/etc/letsencrypt' \  
            -v '/var/lib/letsencrypt:/var/lib/letsencrypt' \  
            certbot/dns-route53 certonly --dns-route53 -d '*.helloworld.com' -d 'helloworld.com' --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory  
      ```  
    - 명령 설명  
        - cerbot/dns-route53  
            - DNS route53 용 이미지를 사용한다.  
              [certbot/dns-route53 이미지](https://hub.docker.com/r/certbot/dns-route53){:target="_blank"}  
        - -v '/etc/letsencrypt:/etc/letsencrypt'   
-v '/var/lib/letsencrypt:/var/lib/letsencrypt'  
            - certbot 컨테이너 인증서 경로와 호스트서버 볼륨을 연결한다.  
        - -d '*.helloworld.com' -d 'helloworld.com'  
            - 인증서를 발급할 도메인 목록을 지정한다.  
        - --dns-route53  
            - dns를 route53을 사용함을 알려준다.  
        - --rm  
            - 컨테이너는 명령 실행 후 자동으로 삭제한다.  
    - 명령 실행 후  
        - 명령 실행 후에는 여러 질문이 나오는데 적당히 대답한다.  
        - 아래 질문이 나왔을때 키를 복사한 뒤 잠시 창을 최초화하고 대기한다.  
          ```bash  
          Please deploy a DNS TXT record under the name  
          _acme-challenge.helloworld.com with the following value:  
                    
          [it-is-my-key]  
                    
          Before continuing, verify the record is deployed.  
          - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -  
          Press Enter to Continue  
          ```  
    - 복사한 키를 Route53 DNS에 TXT record로 등록한다.  
        - AWS 콘솔에서 '검색'에 Route53을 검색하고 클릭한다.  
            - <a href='/assets/img/2023-11-20-cerbot-route53/00-search-route53.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/00-search-route53.jpg' width='80%' height='80%'></a>  
        - (왼쪽 사이드바 메뉴) "대시보드 > 호스팅영역"을 클릭한다.  
        - 만약 "호스팅 영역 이름"에 인증서를 발급하고자 하는   
          도메인(여기서는 helloworld.com)이 있다면 클릭한다.  
            - <a href='/assets/img/2023-11-20-cerbot-route53/01-create-hosting-area.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/01-create-hosting-area.jpg' width='80%' height='80%'></a>  
        - 없다면 "호스팅 영역 생성" 버튼을 눌러 생성한 후  
          도메인을 클릭한다.  
            - <a href='/assets/img/2023-11-20-cerbot-route53/02-select-hosting-area.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/02-select-hosting-area.jpg' width='80%' height='80%'></a>  
        - 레코드 생성을 누른다.  
            - <a href='/assets/img/2023-11-20-cerbot-route53/03-create-record.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/03-create-record.jpg' width='80%' height='80%'></a>  
        - 레코드 이름에 "_acme-challenge"를 입력한다.  
        - 레코드 유형을 "TXT"를 선택한다.  
        - 값에 이전에 복사한 키를 붙여넣는다.  
        - "다른 레코드 추가" 버튼을 누른다.  
            - <a href='/assets/img/2023-11-20-cerbot-route53/04-record-config.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/04-record-config.jpg' width='80%' height='80%'></a>  
    - TXT record 확인  
        - TTL(tiemto live) 300초는 레코드가 적용되기 까지 300초가 걸린다는 뜻  
        - [Google 관리 콘솔도구 상자](https://toolbox.googleapps.com/apps/dig/?lang=ko#A/){:target="_blank"}에서 _acme-challenge.helloworld.com 검색 시  
          키가 노출될때까지 기다린다.  
        - value에 키가 정확히 노출되면 cerbot 명령창으로 되돌아간다.  
            - <a href='/assets/img/2023-11-20-cerbot-route53/05-google-console.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/05-google-console.jpg' width='80%' height='80%'></a>  
    - cerbot 인증서 발급 확인  
        - cerbot 명령창에서 Enter 키를 누르면 발급결과가 나온다.  
          문제가 없다면 인증서가 발급되었을 것이다.  
        - 발급된 인증서는 호스트서버 /etc/letsencrypt/live 하위에 저장된다.  
    - nginx에 인증서 올리기  
        - nginx.conf에서 ssl 설정을 해준다.  
          ```bash  
          # nginx.conf  
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
                  # your setting ...  
              }  
          }
                    
          ```  
        - 이제 호스트서버 인증서 경로와 웹 nginx 서비스 인증서 경로를 연결한다.  
          ```bash  
          # docker-compose.yml  
          version: "3.9"  
                    
          services:  
            nginx:  
              image: nginx:latest  
              ports:  
                - 80:80  
                - 443:443  
              configs:  
                - source: nginx-config  
                  target: /etc/nginx/conf.d/default.conf  
              volumes:  
                - /etc/letsencrypt/live/helloworld.com/fullchain.pem:/etc/ssl/live/fullchain1.pem  
                - /etc/letsencrypt/live/helloworld.com/privkey.pem:/etc/ssl/live/privkey1.pem  
                    
          configs:  
            nginx-config:  
              file: "my-nginx-conf/nginx.conf"  
          ```  
        - 컨테이너 생성  
          ```bash  
          docker compose -f ./docker-compose.yml up  
          ```  
    -  인증서 확인  
        - 인터넷 브라우저를 켜고   
          https://helloworld.com 접속 시 ssl 적용이 되었는지 확인한다.  

## certbot DNS 자동갱신
- 설명  
    - DNS 인증은 TXT레코드에 지정된 키를 등록하는 방식으로 이뤄진다.  
    - 이 과정을 자동으로 처리해야만 인증서 자동갱신을 할 수 있다.  
    - AWS Route53 DNS를 사용 중이므로   
      DNS 사용 권한이 있는 IAM을 생성하고  
      인증서를 자동갱신 서버에 IAM을 등록 후 자동갱신 스크립트를 실행한다.  
- IAM 만들기  
    - [certbot-dns-route53 공식문서](https://certbot-dns-route53.readthedocs.io/en/stable/){:target="_blank"}에 필요한 권한이 적혀있다.  
      ```json  
      {  
          "Version": "2012-10-17",  
          "Id": "certbot-dns-route53 sample policy",  
          "Statement": [  
              {  
                  "Effect": "Allow",  
                  "Action": [  
                      "route53:ListHostedZones",  
                      "route53:GetChange"  
                  ],  
                  "Resource": [  
                      "*"  
                  ]  
              },  
              {  
                  "Effect" : "Allow",  
                  "Action" : [  
                      "route53:ChangeResourceRecordSets"  
                  ],  
                  "Resource" : [  
                      "arn:aws:route53:::hostedzone/YOURHOSTEDZONEID"  
                  ]  
              }  
          ]  
      }  
      ```  
    - AWS 콘솔에서 '검색'에 iam을 검색하고 클릭한다.  
        - <a href='/assets/img/2023-11-20-cerbot-route53/06-search-iam.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/06-search-iam.jpg' width='80%' height='80%'></a>  
    - (왼쪽 사이드바 메뉴) "정책"을 클릭하고 "정책 생성" 버튼을 클릭한다.  
        - <a href='/assets/img/2023-11-20-cerbot-route53/07-create-policy.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/07-create-policy.jpg' width='80%' height='80%'></a>  
    - JSON을 클릭하고 certbot-dns-route53 공식문서 권한을 복사 붙여넣기 한다.  
      "다음" 버튼을 클릭한다.  
        - <a href='/assets/img/2023-11-20-cerbot-route53/08-policy-config.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/08-policy-config.jpg' width='80%' height='80%'></a>  
    - 정책이름(예시에서는 helloworld) 적고, "정책 생성" 버튼을 클릭한다.  
    - (왼쪽 사이드바 메뉴) "사용자"를 클릭하고 '사용자 생성' 버튼을 클릭한다.  
        - <a href='/assets/img/2023-11-20-cerbot-route53/09-create-user.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/09-create-user.jpg' width='80%' height='80%'></a>  
    - 사용자 세부 정보에서 원하는 "사용자 이름"을 입력하고   
      "다음" 버튼을 클릭한다.  
    - "직접 정책 연결"을 클릭하고 이전에 생성한 정책명을 검색 후 선택한다.  
      그리고 "다음" 버튼을 클릭한다.  
        - <a href='/assets/img/2023-11-20-cerbot-route53/10-select-permissions.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/10-select-permissions.jpg' width='80%' height='80%'></a>  
    - "사용자 생성" 버튼을 클릭하여 사용자를 생성한다.  
- IAM 엑세스 키 만들기  
    - (왼쪽 사이드바 메뉴) "사용자"를 클릭하고   
      엑세스 키 1 아래 엑세스 키 만들기 클릭  
        - <a href='/assets/img/2023-11-20-cerbot-route53/11-create-access-key.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/11-create-access-key.jpg' width='80%' height='80%'></a>  
    - Command Line interface(CLI) 선택, 확인 체크박스 선택 후 "다음" 버튼 클릭  
    - 엑세스 키 만들기 버튼 클릭  
        - <a href='/assets/img/2023-11-20-cerbot-route53/12-create-access-key-2.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/12-create-access-key-2.jpg' width='80%' height='80%'></a>  
    - 엑세스 키와 비밀 엑세스키를 메모장에 복사 붙여넣기  
        - <a href='/assets/img/2023-11-20-cerbot-route53/13-create-access-key-3.jpg' target='_blank'><img src='/assets/img/2023-11-20-cerbot-route53/13-create-access-key-3.jpg' width='80%' height='80%'></a>  
- certbot 인증 시 IAM 엑세스 키 사용하기  
    - [certbot-dns-route53 설정예시](https://certbot-dns-route53.readthedocs.io/en/stable/index.html#config-ini){:target="_blank"}를 보면  
      IAM 엑세스 키를 환경변수로 설정하는 것을 볼 수 있다.  
    - AWS_ACCESS_KEY_ID와 AWS_SECRET_ACCESS_KEY를 파일로 저장한다.  
      ```bash  
      echo "your-access_key" >> AWS_ACCESS_KEY_ID  
      echo "your-secret_access_key" >> AWS_SECRET_ACCESS_KEY  
      ```  
    - certbot 인증명령 실행  
      ```bash  
          docker run -it --rm --name certbot \  
            -v '/etc/letsencrypt:/etc/letsencrypt' \  
            -v '/var/lib/letsencrypt:/var/lib/letsencrypt' \  
            -e "AWS_ACCESS_KEY_ID=$(cat ./AWS_ACCESS_KEY_ID)" -e "AWS_SECRET_ACCESS_KEY=$(cat ./AWS_SECRET_ACCESS_KEY)" \  
            certbot/dns-route53 certonly --dns-route53 -d '*.helloworld.com' -d 'helloworld.com' --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory  
                
      ```  
    - 정상적으로 처리되었다면 ENTER를 입력하라는 프롬프트 없이  
      인증서 발급이 완료된다.  
- cron에 등록  
    - certbot 인증명령을 스크립트 파일(예시에서는 set_ssl.sh)로 만든다.  
    - certbot 인증서는 90일 동안 유효하고 30일 전부터 갱신할 수 있으므로  
      2달 마다 실행하도록 cron에 등록한다.  
      ```bash  
      crontab -e  
                
      # 에디터가 실행되면 아래 내용 입력 후 저장  
      0 0 0 1 1/2 ? * bash /home/ubuntu/my_project/set_ssl.sh  
      ```  

## 덧붙이는 말
- AWS Route53 호스팅 영역은 도메인 가격과 별개로  
  월마다 1달러 정도 비용을 요구한다.  
  그렇기에 필요한 호스팅 영역만 생성하는 것이 좋다.  

## 참고
- [certbot-dns-route53 공식문서](https://certbot-dns-route53.readthedocs.io/en/stable/){:target="_blank"}  
- [User Guide — Certbot 2.6.0 documentation eff-certbot.readthedocs.io](https://eff-certbot.readthedocs.io/en/stable/using.html#dns-plugins){:target="_blank"}  
- [certbot/dns-route53 - Docker Image Docker Hub](https://hub.docker.com/r/certbot/dns-route53){:target="_blank"}  
- [Docker로 간단하게 Let's Encrypt 와일드카드 인증서 발급받기](https://lynlab.co.kr/blog/72){:target="_blank"}  
- [DNS TXT Record 로 Let's Encrypt SSL 인증서 발급 받기](https://www.lesstif.com/system-admin/dns-txt-record-let-s-encrypt-ssl-59343172.html){:target="_blank"}  
- [Let’s Encrypt 와일드카드로 여러개의 서브도메인 인증서 한번에 발급받기](https://oasisfores.com/letsencrypt-wildcard-ssl-certificate/){:target="_blank"}  
- [How to Find Hosted Zone ID in Route53 AWS in 3 Clicks arcadian.cloud"](https://arcadian.cloud/aws/2022/03/22/how-to-find-hosted-zone-id-in-route53-aws-in-3-clicks/){:target="_blank"}  
