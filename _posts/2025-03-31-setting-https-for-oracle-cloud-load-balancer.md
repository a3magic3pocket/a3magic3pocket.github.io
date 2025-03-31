---
title: 오라클 클라우드 로드밸런서 HTTPS 설정하기
date: 2025-03-31 23:15:02 +0900
categories: [Oracle-cloud]
tags: [oracle-cloud, loadbalancer, summary]    # TAG names should always be lowercase
---

## 설명
- SSL 인증서를 로드밸런서에 등록한다.  
- HTTPS 용 리스너를 생성한다.  
- SSL 인증서는 let's encrypt 인증서를 기준으로 설명한다.  

## Let's Encrypt SSL 인증서 파일 설명
- Certificate (인증서) 파일 (fullchain.pem)  
    - 로드 밸런서에 등록되는 서버 인증서  
    - 서버 Public key 도 포함되어 있음  
- Private Key(비밀키) 파일(private.pem)  
    - 웹서버가 SSL/TLS 암호화 할때 사용  

## SSL 인증서를 로드밸런서에 등록
- (검색창에서)로드 밸런서 검색 후 선택  
- (왼쪽 사이드메뉴에서) "로드 밸런서 인증서" 클릭  
- (로드 밸런서 관리 인증서에서)  "인증서 추가" 버튼 클릭  
- (인증서 추가에서)  
    - 인증서 이름  
        - 원하는 이름 작성  
    - SSL 인증서 지정 > SSL 인증서 붙여넣기  
        - 파일 업로드 시 적용이 안 되는 문제가 발생한 적이 있다.  
        - fullchain.pem 문자열을 읽어 복사 붙여넣기 한다.  
    - 프라이빗 키 지정  
        - private.pem 문자열을 읽어 복사 붙여넣기 한다.  

## HTTPS 용 리스너 생성
- (왼쪽 사이드메뉴에서) "리스너" 클릭  
- (리스너 본문에서) "리스너 생성" 버튼 클릭  
- (리스너 생성)  
    - 이름  
        - 원하는 이름 작성  
    - 프로토콜  
        - HTTPS  
    - 포트  
        - 443  
    - SSL 사용  
        - 예  
    - 인증서 리소스  
        - 로드 밸런서 관리 인증서  
    - 인증서 이름  
        - 위에서 생성한 인증서 이름 선택  
    - 백엔드 집합  
        - 이전에 생성한 80 포트로 전달하는 백엔드 집합 선택  
    - "변경사항 저장" 버튼 클릭  
