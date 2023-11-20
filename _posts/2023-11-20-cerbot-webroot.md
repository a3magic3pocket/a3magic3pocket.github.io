---
title: certbot, webroot 방식으로 인증서 발급받기
date: 2023-11-20 18:28:45 +0900
categories: [certbot]
tags: [cerbot, nginx]    # TAG names should always be lowercase
---

## 설명
- webroot 방식으로 certbot 인증서를 발급받는다.  
- certbot 도커를 활용하여 인증서를 발급 받는다.  
- 예시에서 인증 받을 도메인은 helloworld.com으로 가정한다.  

## 사전 준비
- nginx를 띄우고 Route 53 DNS에서 서버IP와 도메인을 연결한다.  
  이를 통해 도메인을 입력하여 서버로 http 접속이 가능한 상태가 된다.  
- /var/www/letsencrypt/.well-known/acme-challenge 디렉토리를 생성한다.    
  ```bash  
  mkdir -p /var/www/letsencrypt/.well-known/acme-challenge  
  ```  
- 아래와 같이 nginx 설정을 한다.  
  ```bash  
  server {  
      listen 80 default_server;  
                
      server_name _;  
            
            
      location ^~ /.well-known/acme-challenge/ {  
          root /var/www/letsencrypt;  
      }  
            
  }  
            
  ```  

## 명령
```bash  
docker run -it --rm --name cerbot \  
-v "/etc/letsencrypt:/etc/letsencrypt" \  
-v "/var/www/letsencrypt:/var/www/letsencrypt" \  
certbot/certbot certonly -w /var/www/letsencrypt --webroot -v\  
-d helloworld.com \  
-d www.helloworld.com  
```  

## 주의사항
- certbot 컨테이너의 /var/www/letsencrypt 가   
  호스트서버와 반드시 공유되어야 한다.  

## 참고
- [Certbot Webroot](https://eff-certbot.readthedocs.io/en/stable/using.html#webroot){:target="_blank"}  
- [Set up Letsencrypt/Certbot with Nginx web server with webroot](https://softdiscover.com/linux/set-up-letsencrypt-certbot-with-nginx-web-server-with-webroot/){:target="_blank"}  
