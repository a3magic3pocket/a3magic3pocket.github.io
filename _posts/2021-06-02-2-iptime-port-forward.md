---
title: 두 대의 공유기가 연결된 환경에서 포트포워드하기(in windows)
date: 2021-06-02 14:59:00 +0900
categories: [Network]
tags: [portforwarding, router, iptime] # TAG names should always be lowercase
---

## 개요
- 서브 공유기에 랜선을 연결하여 사용 중이신 개발자님께서  
본인 pc에서 개발 서버를 올린 후 개발물을 모바일로 확인해야하는 하는 이슈가 생겼다.
- 서브 공유기에 포트포워드을 해주어 문제를 해결하였다.

## 환경
- ISP와 직접 연결된 메인 공유기가 있고,  
메인 공유기에 랜선을 연결하여 작은 방 내에서 라우팅하는 서브 공유기가 있고,  
우리 개발자님 pc는 서브 공유기와 랜선으로 연결되어있는 구조이다.
- 공유기는 iptime으로 가정한다.
- <a href="/assets/img/2021-06-02-2-iptime-port-forward/00-env.jpg" target="_blank"><img src="/assets/img/2021-06-02-2-iptime-port-forward/00-env.jpg" width="100%"></a> 

## 포트포워드
- 모바일이 메인 공유기의 무선랜에 연결되어 있다고 가정하자.
- 모바일이 메인 공유기 무선랜에 접속되어있기 때문에 랜선 연결 개발서버로 접근하려면,  
서브 공유기로 먼저 접근해야한다.
- 모바일에서 서브 공유기의 내부 ip인 172.12.1.3으로 요청(request)을 하지만  
서브 공유기는 모바일이 서브 공유기에 연결된 디바이스 중 어떤 디바이스에 요청한지 알 수 없다.
- 이를 알려주는 작업이 포트포워드이다.
- 서브 공유기에서 특정포트로 요청을 받았을 때 특정 디바이스의 특정포트로 연결하라는 룰을 지정하는 것이다.
- 여기서는 개발 서버 포트인 8080 포트를 기준으로 처리할 것이다.
- <a href="/assets/img/2021-06-02-2-iptime-port-forward/01-portforward.jpg" target="_blank"><img src="/assets/img/2021-06-02-2-iptime-port-forward/01-portforward.jpg" width="100%"></a> 

## 윈도우 방화벽 인바운드 규칙 생성
- [모바일에서 localhost 접속(in windows)](https://a3magic3pocket.github.io/posts/mobile-localhost/){:target="\_blank"}에서 설명한 방법대로 인바운드 규칙을 생성한다.
- 포트는 8080이라고 가정한다.

## 내부 IP 확인
- 설명
    - 서브 공유기에서 개발 pc에게 할당한 내부 IP를 확인한다.
- 진행방법
    - cmd 창을 켠다.
    - ipconfig를 입력한다.
    - 이더넷 어댑터 이더넷에서 IPv4 주소를 기억한다.

## 공유기 관리 창 접속
- 설명
    - 포트포워드에 필요한 작업을 하기 위해 공유기 관리 창으로 접속한다.
- 진행방법
    - 웹 브라우저를 켠다.
    - 주소검색칸에 192.168.0.1을 입력한다.
    - admin 계정으로 접속한다.

## 외부 IP 주소 획득
- 설명
    - 메인 공유기가 서브 공유기에게 할당한 내부 IP를 알아낸다.
- 진행방법
    - 기본 설정 > 인터넷 정보의 외부 IP 주소를 기억한다.
    - <a href="/assets/img/2021-06-02-2-iptime-port-forward/02-public-ip.jpg" target="_blank"><img src="/assets/img/2021-06-02-2-iptime-port-forward/02-public-ip.jpg" width="70%"></a> 

## 고정 IP 방식 사용
- 설명
    - 서브 공유기가 개발 pc에게 할당한 내부 IP가 변경되지 않도록 IP를 고정한다.
- 진행방법
    - 기본 설정 > 인터넷 설정 정보를 선택한다.
    - 동적 IP 방식에서 > 고정 IP 방식으로 변경한다.
    - 기본 DNS 주소에 8.8.8.8, 보조 DNS 주소에 8.8.4.4를 입력하고(구글 DNS 사용) 나머지는 그대로 둔다.
    - 적용버튼을 눌러 적용한다.
    - <a href="/assets/img/2021-06-02-2-iptime-port-forward/03-static-ip.jpg" target="_blank"><img src="/assets/img/2021-06-02-2-iptime-port-forward/03-static-ip.jpg" width="70%"></a> 

## 포트포워드 설정
- 설명
    - 외부 공유기가 서브 공유기에게 할당한 내부 ip의 특정 포트로 요청을 받으면  
    서브 공유기가 개발 pc에 할당한 내부 ip의 특정 포트로 연결한다.
- 진행방법
    - 고급 설정 > NAT/라우터 관리 > 포트포워드 설정를 선택한다.
    - 새규칙 추가 클릭
    - 세부 항목 작성
        - 규칙이름
            - 원하는 이름 작성
        - 내부 IP 주소
            - 서브 공유기가 개발 pc에 할당한 내부 ip 
            - ipconfig로 확인한 IPv4 주소를 입력한다.
        - 프로토콜
            - http 프로토콜만 처리할 것임으로 TCP 선택
        - 외부포트
            - 서브 공유기에서 입력 받는 포트를 지정하는 곳이다.
            - 8080~8080으로 선택
        - 내부포트
            - 개발 pc에서 입력 받는 포트를 지정하는 곳이다.
            - 8080~8080으로 선택
    - 적용 버튼을 눌러 적용한다.
    - <a href="/assets/img/2021-06-02-2-iptime-port-forward/04-set-portforward.jpg" target="_blank"><img src="/assets/img/2021-06-02-2-iptime-port-forward/04-set-portforward.jpg" width="70%"></a>

## 확인
- 진행방법
    - 모바일을 메인 공유기 무선랜에 접속시킨다.
    - 모바일 브라우저를 켠다.
    - 서브 공유기가 개발pc에게 할당한 내부 ip를 기억한다(그림 예시에서는 172.12.1.3)
    - 포트포워드는 외부포트 내부포트 모두 8080으로 해두었다고 가정한다.
    - 주소입력칸에 172.12.1.3:8080을 입력하면 개발 pc의 개발 서버로 접속되어야 성공이다.
