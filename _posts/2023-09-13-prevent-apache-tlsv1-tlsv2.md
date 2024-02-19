---
title: Apache에서 TLSv1, TLSv1.1 접근 막기
date: 2023-09-13 20:26:23 +0900
categories: [Webserver]
tags: [apache, tls, ssl]    # TAG names should always be lowercase
---

## 설명
- Apache Webserver에서 보안성이 낮은 프로토콜인  
  TLSv1, TLSv1.1을 허용하지 않게 설정한다.  

## 설정파일 수정
- 가정  
    - conf.d/my.conf에서 SSL 설정을 한다고 가정한다.  
    - conf/http.conf에서 conf.d/my.conf를   
      include 하고 있다고 가정한다.  
- conf/http.conf  
  ```  
  ...  
  IncludeOptional conf.d/my.conf  
  ```  
- conf.d/my.conf  
  ```bash  
  Listen 443 https  
  <VirtualHost *:443>  
  	ServerName my.website.com  
            
  	SSLEngine on  
  	SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1  
          SSLCertificateFile "my.crt"  
          SSLCertificateKeyFile "my.key"  
          SSLCACertificateFile "my.bundle_crt"  
  	....  
  </VirtualHost>  
  ```  
            
- SSLProtocol  
    - 모든 프로토콜 허용  
      ```bash  
      SSLProtocol all  
      ```  
    - SSLv3, TLSv1, TLSv1.1 빼고 나머지 허용  
      ```bash  
      SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1  
      ```  
- 주의사항  
    - 다른 VirtualHost에서 SSLProtocol을 다르게 설정하면  
      먼저 선언된 것으로 일괄 적용된다고 하니 주의해야한다.  
    - [How can I disable TLS 1.0 and 1.1 in apache?](https://serverfault.com/questions/848177/how-can-i-disable-tls-1-0-and-1-1-in-apache){:target="_blank"}  
    - 아래 명령어로 현재 경로에서 SSLProtocol이 설정된   
      모든 파일의 행을 검색한 뒤 동일한 값을 갖도록 수정한다.  
      ([Apache won't disable TLSv1 no matter what I do](https://serverfault.com/questions/992350/apache-wont-disable-tlsv1-no-matter-what-i-do){:target="_blank"})  
      ```  
      find ./ -type f -exec grep -i sslprotocol {} +  
      ```  

## 확인
- openssl을 이용하여 특정 프로토콜로 요청을 보내  
  응답을 보고 확인한다.  
- 명령  
  ```bash  
  # TLSv1으로 연결 시도  
  openssl s_client -connect my.website.com:443 -tls1  
            
  # TLSv1.1으로 연결 시도  
  openssl s_client -connect my.website.com:443 -tls1_1  
  ```  
- 결과  
    - TLSv1과 TLSv1.1로 연결 시도 시 연결이 실패해야 성공이다.  
    - 연결 실패 시 최초 에러가 출력되며,   
      'Secure Renegotiation IS NOT supported' 가 떠야한다.  
