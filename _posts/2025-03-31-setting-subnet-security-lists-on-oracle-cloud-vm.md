---
title: 오라클 클라우드 VM에 서브넷 보안목록 설정하기
date: 2025-03-31 23:12:57 +0900
categories: [Oracle-cloud]
tags: [oracle-cloud, subnet, security-list, summary]    # TAG names should always be lowercase
---

## 설명
- VCN(가상 클라우드 네트워크)에서 보안목록을 생성한다.  
- VCN(가상 클라우드 네트워크)에서 서브넷을 생성한다.  
- VM 생성 시 서브넷을 선택하여 생성한다.  
  보안목록을 수정할 수 있지만   
  이미 생성한 VM에서 다른 서브넷으로 변경할 수는 없다.  

## VCN(가상 클라우드 네트워크)
- 실제 클라우드 내에서의 네트워크를 분리하고 격리하는 역할  

## 보안목록
- 서브넷에 속한 리소스들의 트래픽을 제어하는 규칙  
- 인바운드 규칙  
    - 외부에서 들어오는 트래픽을 제어한다.  
    - IP CIDR, 프로토콜,  포트를 설정한다.  
- 아웃바운드 규칙  
    - 내부에서 외부로 나가는 트래픽을 제어한다.  
    - IP CIDR, 프로토콜,  포트를 설정한다.  

## 서브넷
- VCN 내에서 더 작은 네트워크 단위로 IP 주소 범위를 나눈 것  
- 퍼블릭과 프라이빗 서브넷으로 나뉨  
- 퍼블릭은 인터넷 게이트웨이를 통해 외부와 연결된 서브넷  
- 프라이빗은 인터넷과 연결되지 않은 서브넷  
- 보안목록은 서브넷 단위로 설정할 수 있다.  
- 오라클 클라우드에서 VCN 내의 서브넷은   
  보안목록에 위배되지 않는다면 별도의 설정 없이 통신할 수 있다.  

## 예제 - 보안그룹 생성 및 서브넷 생성
- 보안목록 생성  
    - (검색창에서) "가상 클라우드 네트워크" 검색 후 선택  
    - (가상 클라우드 네트워크 본문에서) 서브넷과 같은 VCN 선택  
    - (왼쪽 사이드 메뉴에서) "보안목록" 클릭  
    - (보안 목록 본문에서) "보안목록 생성" 버튼 클릭  
    - (보안 목록 생성 본문에서)  
        - 이름  
            - 원하는 이름 입력  
        - 수신에 대한 규칙 허용  
            - "+ 다른 수신 규칙" 버튼 클릭  
            - 수신규칙 1  
                - 소스 CIDR  
                    - 적용할 IP 범위를 선택하는 곳  
                    - [IPv4 CIDR 블록](https://namu.wiki/w/CIDR#s-3.1){:target="_blank"}  
                    - 10.0.0.0~10.0.255.255 범위 허용  
                        - 10.0.0.0/16  
                    - 10.0.0.0~10.0.0.255 범위 허용  
                        - 10.0.0.0/24  
                    - 10.0.1.12만 허용  
                        - 10.0.1.12/32  
                    - 모든 IP 허용  
                        - 0.0.0.0/0  
                - IP 프로토콜  
                    - 적용할 프로토콜 선택  
                    - HTTP, HTTPS를 열어야하는 경우 TCP 선택  
                - 소스 포트 범위  
                    - 모두  
                - 대상 포트 범위  
                    - 적용할 포트 선택  
                    - HTTP를 허용하고자 한다면 80 포트 선택  
                - "보안목록 생성" 버튼 클릭  
        - 송신에 대한 규칙 허용  
            - "+ 다른 송신 규칙" 버튼 클릭  
            - 송신규칙 1  
                - 수신 규칙을 참고하여 작성  
                - 모든 트래픽 송신 허용  
                    - 대상 CIDR  
                        - 0.0.0.0/.  
                    - IP 프로토콜  
                        - 모두  
                    - 대상 포트 범위  
                        - 모두  
                - 내부 트래픽만 송신 허용  
                    - 대상 CIDR  
                        - 10.0.0.0/16  
                    - IP 프로토콜  
                        - 모두  
                    - 대상 포트 범위  
                        - 모두  
- 서브넷 생성  
    - (검색창에서) "가상 클라우드 네트워크" 검색 후 선택  
    - (가상 클라우드 네트워크 본문에서) 서브넷을 생성할 VCN 선택  
    - (서브넷 본문에서) "서브넷 생성" 버튼 클릭  
    - (서브넷 생성 본문에서)   
        - 이름  
            - 원하는 이름 입력  
        - 서브넷 유형  
            - 지역별(권장) 클릭  
        - IPv4 CIDR  
            - 적절히 선택  
            - 예시에서는 10.0.1.0/24로 입력했다고 가정  
        - 서브넷 엑세스  
            - 퍼블릭 서브넷 선택  
        - 보안 목록  
            - 서브넷에 적용하고자 하는 보안목록 선택  
            - 선택 안하면 기본 보안목록이 적용됨  
        - "서브넷 생성" 버튼 클릭  
