---
title: 모바일에서 localhost 접속(in windows)
date: 2021-05-05 07:58:00 +0900
categories: [Network]
tags: [windows, mobile, localhost]    # TAG names should always be lowercase
---
## 개요
- 모바일에서 개발물에 문제가 있나 확인하기 위해 모바일에서 개발 서버로 접속하고 싶어졌다.
- 윈도우 방화벽에서 인바운드 포트를 개발 서버 포트로 지정하여 특정 포트를 허용한다.
- 윈도우에서 모바일과 연결된 무선랜의 private ip를 고정시킨다.
- 이제 모바일에서 브라우저를 켜고 [위에서 고정한 ip]:[해당 포트]로 접근하면 
성공이다.

## 개발 서버 실행
- 개발 서버를 실행하여 localhost:[지정포트]로 접속할 수 있게 한다.
- 나의 경우 localhost:8080으로 접근할 수 있도록 하였다.

## 윈도우 방화벽에서 개발 서버 포트 허용하기
- 방화벽 검색 > 고급 보안이 포함된 Windows Defender 방화벽 클릭  
  <a href="/assets/img/2021-05-05-mobile-localhost/00-search-firewall.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/00-search-firewall.jpg" width="70%"></a> 
- 인바운드 규칙 클릭 > 새 규칙 클릭  
  <a href="/assets/img/2021-05-05-mobile-localhost/01-make-new-rule.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/01-make-new-rule.jpg" width="100%" height="100%"></a> 
- 포트 선택 후 다음 클릭  
  <a href="/assets/img/2021-05-05-mobile-localhost/02-rule-type.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/02-rule-type.jpg" width="100%" height="100%"></a> 
- TCP 선택, 특정 포트 선택 후 원하는 포트 입력, 다음 클릭  
  <a href="/assets/img/2021-05-05-mobile-localhost/03-protocol-and-port.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/03-protocol-and-port.jpg" width="100%" height="100%"></a> 
- 연결 허용 선택 후 다음 클릭  
  <a href="/assets/img/2021-05-05-mobile-localhost/04-connection-range.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/04-connection-range.jpg" width="100%" height="100%"></a> 
- 도메인, 개인, 공용 선택 후 다음 클릭  
  <a href="/assets/img/2021-05-05-mobile-localhost/05-connection-timing.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/05-connection-timing.jpg" width="100%" height="100%"></a> 
- 이름 및 설명 옵션 입력 후 마침 클릭  
  <a href="/assets/img/2021-05-05-mobile-localhost/06-name-an-description.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/06-name-an-description.jpg" width="100%" height="100%"></a>
- 새로 추가된 규칙 더블 클릭하여 확인  
  <a href="/assets/img/2021-05-05-mobile-localhost/07-check-new-rule-1.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/07-check-new-rule-1.jpg" width="100%" height="100%"></a>
- TCP 및 로컬 포트 확인  
  <a href="/assets/img/2021-05-05-mobile-localhost/08-check-new-rule-2.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/08-check-new-rule-2.jpg" width="70%"></a>

## 내 pc 무선랜 private ip 확인
- cmd 검색  
  <a href="/assets/img/2021-05-05-mobile-localhost/09-search-cmd.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/09-search-cmd.jpg" width="70%"></a>
- cmd 창에 ipconfig 입력 후 엔터  
- 결과에서 "무선 LAN 어댑터 Wi-Fi"의 내용을 모두 기억  
  <a href="/assets/img/2021-05-05-mobile-localhost/10-ipconfig.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/10-ipconfig.jpg" width="100%" height="100%"></a>
- 그 중 IPv4를 복사
- 브라우저를 켜고 주소입력칸에 [복사한 IPv4]:[개발서버 포트]를 입력하여 접속되는지 확인
- 모바일에서도 동일하게 진행(이때 모바일도 동일 무선랜에 접속 중이어야한다.)

## private IP 수동으로 고정시키기
- 개요
    - 무선랜의 private IP는 기본 설정이 자동할당이다.  
    때문에 새로 연결할 때마다 바뀐다.
    - 모바일에서 접속 시 매번 private IP가 바뀌면 귀찮으므로 수동으로 고정시킨다.
- 하단의 무선랜 아이콘 클릭하고 접속 중인 무선랜의 속성 클릭  
  <a href="/assets/img/2021-05-05-mobile-localhost/11-find-wifi.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/11-find-wifi.jpg" width="50%"></a>
- 네트워크 프로필을 개인으로 변경(이 것이 영향을 미쳤는지는 확실하지 않음)  
  <a href="/assets/img/2021-05-05-mobile-localhost/12-network-profile.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/12-network-profile.jpg" width="100%" height="100%"></a>
- IP 설정에서 편집 클릭  
  <a href="/assets/img/2021-05-05-mobile-localhost/13-set-ip.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/13-set-ip.jpg" width="100%" height="100%"></a>
- 수동 할당(위에서 기억한 무선 LAN 어댑터 Wi-Fi 내용 이용)  
    <a href="/assets/img/2021-05-05-mobile-localhost/14-set-ip-detail.jpg" target="_blank"><img src="/assets/img/2021-05-05-mobile-localhost/14-set-ip-detail.jpg" width="50%"></a>
    - 수동 선택
    - IPv4 켬
    - IP 주소
        - 위에서 기억한 IPv4
    - 서브넷 접두사 길이
        - 위에서 기억한 서브넷 마스크를 보고 [링크](https://ko.wikipedia.org/wiki/%EB%B6%80%EB%B6%84%EB%A7%9D){:target="_blank"}에 접속하여 참고하여 입력
    - 게이트 웨이
        - 위에서 기록한 기본 게이트웨이
    - 기본 DNS
        - 8.8.8.8
    - 대체 DNS
        - 8.8.4.4
    - 기본 DNS, 대체 DNS 추가 설명
        - 위의 DNS 서버는 구글 것임
        - 다른 DNS 서버를 사용하고 싶으면 [DNS 서버 주소 목록](https://m.blog.naver.com/kangyh5/221701387941){:target="_blank"} 참고
- 확인
    - 브라우저를 켜고 주소입력칸에 [복사한 IPv4]:[개발서버 포트]를  
    입력하여 접속되는지 확인
    - 모바일에서도 동일하게 진행(이때 모바일도 동일 무선랜에 접속 중이어야한다.)

## 참고
- [localhost 웹 프로젝트화면 모바일로 확인해보기](https://iri-kang.tistory.com/8){:target="_blank"}
- [윈도우 10 와이파이 수동 IP 입력 불가](https://answers.microsoft.com/ko-kr/windows/forum/all/%EC%9C%88%EB%8F%84%EC%9A%B0-10/b2f17b24-54a9-41f9-9b78-cd5a1d1cd65b){:target="_blank"}
- [[Android] 안드로이드 핸드폰과 localhost 서버 연결이 안될 때](https://yujin-dev.tistory.com/49){:target="_blank"}
- [부분망 위키백과](https://ko.wikipedia.org/wiki/%EB%B6%80%EB%B6%84%EB%A7%9D){:target="_blank"}
- [DNS 서버 주소 목록](https://m.blog.naver.com/kangyh5/221701387941){:target="_blank"}
