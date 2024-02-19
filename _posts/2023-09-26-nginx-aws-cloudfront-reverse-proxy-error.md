---
title: nginx, AWS cloudfront로 리버스 프록시 설정 에러
date: 2023-09-26 22:03:12 +0900
categories: [Nginx]
tags: [nginx, aws, cloudfront]    # TAG names should always be lowercase
---

## 현상
- 기존 CDN을 AWS cloudfront + s3 CDN(이하 cloudfront)으로 변경하였다.  
- proxy_pass 경로를 기존 CDN URL에서 cloudfront URL로 변경하였지만  
  포워딩 되지 않고 에러가 발생하였다.  

## 에러
- 원문  
  ```  
  SSL: error:14094410:SSL routines:ssl3_read_bytes:sslv3 alert handshake failure:SSL alert number 40) while SSL handshaking to upstream  
  ```  
- 에러 원문 후미에는 CDN 경로가 기록된다.  
- 이상한 점은 분명 CDN 경로를 cloudfront 도메인으로 입력하였는데   
  ip로 시도한 것으로 로깅되고 있었다.  
  ex) https://my.cloudfront.com/my/image/ -> https://123.123.123.123/my/image/  

## 해결
- 방법  
    - nginx 설정에서   
      proxy_ssl_server_name를 on으로 변경하고      
      proxy host header를 CDN 도메인으로 등록해주자 에러가 해결되었다.  
- 원인  
    - nginx는 ip주소로만 연결할 수 있다고 한다.([참고](https://serverfault.com/questions/915918/nginx-reverse-proxy-is-trying-to-access-backend-over-ip-instead-of-domain-name){:target="_blank"})  
    - ip URL로 요청을 보내면 당연하게도   
      cloudfront 도메인 ssl 인증서를 이용한  
      핸드쉐이킹이 실패할 것이다.  
    - cloudfront 도메인으로 Host header를 지정하여  
      ssl 통신이 가능하게 해주면 된다.  
- nginx 설정  
  ```bash  
  # default.conf  
  ...  
   location ^~ /image/ {  
     proxy_pass https://my.cloudfront.com/my/image/;  
     proxy_ssl_server_name on;  
     proxy_set_header Host my.cloudfront.com;  
     proxy_set_header X-Real-IP $remote_addr;  
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
     proxy_set_header X-Forwarded-Proto https;  
   }  
  ...  
  ```  

## 번외
- 현상  
    - cloudfront CDN으로 포워딩은 되는데   
      임시 buffer 파일을 생성했다는 warning 로그가 남는다.  
- 에러  
    - 원문  
      ```  
      *4 an upstream response is buffered to a temporary file /  
      ```  
- 원인  
    - proxy CDN에서 nginx로 응답을 받을 때  
      응답의 용량이 기본 메모리 버퍼보다 크기 때문에  
      디스크에 임시로 파일을 작성한 후 넘겼다는 의미이다.([참고](https://bcdragonfly.tistory.com/70){:target="_blank"})  
    - proxy buffer 용량을 충분히 크게 잡아주면  
      해당 warning 로그가 남지 않는다.  
- nginx 설정  
  ```bash  
  # default.conf  
  ...  
   location ^~ /image/ {  
     ...  
    # 프록시 버퍼를 16M 지정  
     proxy_buffers 16 16M;  
   }  
  ...  
            
  ```  

## nginx 재시작
- 설명  
    - nginx 설정 변경 후에는 재시작을 해줘야 적용이 된다.  
- 명령  
  ```bash  
  # 설정이 올바른지 확인  
  nginx -t   
            
  # nginx 재시작  
  service nginx restart  
  ```  

## 참고
- [Nginx - Reverse Proxy is trying to access backend over IP instead of domain name](https://serverfault.com/questions/915918/nginx-reverse-proxy-is-trying-to-access-backend-over-ip-instead-of-domain-name){:target="_blank"}  
- [Nginx reverse proxy to Heroku fails SSL handshake](https://stackoverflow.com/questions/38375588/nginx-reverse-proxy-to-heroku-fails-ssl-handshake){:target="_blank"}  
- [[Nginx] 1 an upstream response is buffered to a temporary file 프록시 버퍼 문제](https://bcdragonfly.tistory.com/70){:target="_blank"}  
