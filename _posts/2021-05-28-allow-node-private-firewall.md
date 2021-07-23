---
title: 노드 개발 서버 모바일에서 접속하기(in windows)
date: 2021-05-28 19:00:00 +0900
categories: [Network]
tags: [javascript, node, windows, firewall, network] # TAG names should always be lowercase
---

## 개요
- 노드로 웹 서버를 실행하고 같은 네트워크에 있는 모바일 디바이스로 접속하니 접속 실패하였다.
- [모바일에서 localhost 접속(in windows)](https://a3magic3pocket.github.io/posts/mobile-localhost/){:target="\_blank"} 방법대로 했으나 되지 않았다.
- 혹시나 하여 golang으로 웹 서버를 실행하고 똑같이 모바일 디바이스로 접속해보니 된다.
- 원인은 node 자체에서 추가한 방화벽 규칙이 있었고,  
이 방화벽이 개인(private) 접근을 막고 있어 생기는 문제였다.
- 이 방화벽 인바운드 규칙을 차단에서 허용으로 바꾸어 해결하였다.

## 3000번 포트 허용
- [모바일에서 localhost 접속(in windows)](https://a3magic3pocket.github.io/posts/mobile-localhost/){:target="\_blank"}에서 설명한 방법대로 인바운드 규칙을 생성한다.
- 이때 포트는 node 개발 서버에서 자주 쓰는 3000번으로 해준다.

## Node.js:: Server-side JavaScript 인바운드 규칙
- 3000번 포트를 허용하고 나면 "Windows Defender 방화벽 > 인바운드 규칙"이 켜져있을 것이다.
- 여기서 이름순으로 정렬하고 스크롤을 내리다보면 "Node.js:: Server-side JavaScript" 규칙을 발견할 수 있다.
- 프로필이 "개인"인 "Node.js:: Server-side JavaScript" 규칙이 모두 "연결 차단" 인 걸 알 수 있다.  
    <a href="/assets/img/2021-05-28-allow-node-private-firewall/00-before-rule.jpg" target="_blank"><img src="/assets/img/2021-05-28-allow-node-private-firewall/00-before-rule.jpg" width="70%"></a>   
- 더블 클릭하여 속성 창을 열고 "연결 차단"으로 선택되어 있는 것을 "연결 허용"으로 바꿔주고 확인을 누른다.    
    <a href="/assets/img/2021-05-28-allow-node-private-firewall/01-detail.jpg" target="_blank"><img src="/assets/img/2021-05-28-allow-node-private-firewall/01-detail.jpg" width="70%"></a>   
- 연결 차단 되어 있는 두 개 룰 모두 연결 허용으로 바꿔준다.  
    <a href="/assets/img/2021-05-28-allow-node-private-firewall/02-after-rule.jpg" target="_blank"><img src="/assets/img/2021-05-28-allow-node-private-firewall/02-after-rule.jpg" width="70%"></a>   

## 참고
- [How could others, on a local network, access my NodeJS app while it's running on my machine?](https://stackoverflow.com/questions/5489956/how-could-others-on-a-local-network-access-my-nodejs-app-while-its-running-on){:target="\_blank"}
