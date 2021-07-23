---
title: AWS RDS에 EC2 접속 허용하기
date: 2021-05-02 23:06:00 +0900
categories: [AWS]
tags: [rds, ec2, security-group]    # TAG names should always be lowercase
---
## 개요
- 기본 설정으로 RDS와 ec2를 팠는데, ec2에서 RDS 접속이 안되는 현상이 발생해서  
이를 해결했다.

## RDS와 ec2 팔 때 주의할 점
- RDS와 ec2 리전을 동일하게 파야한다.  
다르게 파면 안 나온다.(이렇게 쓰려면 기본 세팅 외에 다른 조치를 해야하는 듯)

## 문제 발생 이유
- RDS에서 사용하는 보안그룹 세팅 시 ec2에서 사용 중인 보안그룹 ID를  
인바운드 그룹에 추가해줘야 한다.  
- RDS에서 사용하는 보안그룹, 인바운드 그룹에 ec2 IP를 추가했지만  
이 방법은 정상 작동하지 않았다.

## 해결과정
- ec2 정보 저장
  - 인스턴스 > 세부정보에서 "VPC ID" 기억
    <a href="/assets/img/2021-05-02-aws-rds-allow-ec2/00-ec2-security-group-id.jpg" target="_blank"><img src="/assets/img/2021-05-02-aws-rds-allow-ec2/00-ec2-security-group-id.jpg" width="100%" height="100%"></a>  
  - 인스턴스 > 보안에서 "보안그룹 ID" 기억
    <a href="/assets/img/2021-05-02-aws-rds-allow-ec2/01-vpc-id.jpg" target="_blank"><img src="/assets/img/2021-05-02-aws-rds-allow-ec2/01-vpc-id.jpg" width="100%" height="100%"></a>
- RDS 용 보안그룹 생성
  - 보안그룹 > 보안그룹 생성 클릭
    <a href="/assets/img/2021-05-02-aws-rds-allow-ec2/02-create-security-group.jpg" target="_blank"><img src="/assets/img/2021-05-02-aws-rds-allow-ec2/02-create-security-group.jpg" width="100%" height="100%"></a>
  - 보안그룹 생성
    <a href="/assets/img/2021-05-02-aws-rds-allow-ec2/03-create-security-group-detail.jpg" target="_blank"><img src="/assets/img/2021-05-02-aws-rds-allow-ec2/03-create-security-group-detail.jpg" width="100%" height="100%"></a>
  - 생성한 보안그룹 ID를 기억
  - 상세 설명
    - 보안그룹 이름, 설명
      - 원하는 이름 및 설명을 적어둔다.
    - VPC
      - 위에서 기억한 ec2 VPC ID를 선택
    - 인바운드 규칙
      - 쓰고 있는 DB 설정에 맞게 추가하면 된다.
      - 나의 경우 MariaDB이니 MariaDB 기준으로 설명한다.
      - 유형
        - MYSQL/Aurora
      - 프로토콜
        - TCP
      - 포트범위
        - 3306
        - 만약 DB에서 custom port를 쓰고 있다면 그 port를 넣는다.  
        (수정이 안되면 유형을 "사용자 지정 TCP"로 바꾸고 수정)
      - 소스 유형
        - 사용자 지정
      - 소스
        - 위에서 기억한 ec2 보안그룹 ID를 입력
      - 설명
        - 구분하기 쉽게 간단한 메모를 써준다.
    - 아웃바운드 규칙
      - 퍼블릭 릴리즈라면 그냥 기본 설정대로  
      유형: 모든 트래픽, 대상: 0.0.0.0/0으로 둔다.
- RDS의 보안그룹 변경
  - 기본 설정으로 생성한 DB를 수정한다.
    <a href="/assets/img/2021-05-02-aws-rds-allow-ec2/04-modify-db-security-group.jpg" target="_blank"><img src="/assets/img/2021-05-02-aws-rds-allow-ec2/04-modify-db-security-group.jpg" width="100%" height="100%"></a>
    - 서브넷 그룹
      - ec2에서 사용 중인 vpc와 같은 vpc여야 한다.
      - 기본 설정이라면 이미 ec2와 동일한 vpc 서브넷일 것이다.
      - 만약 그렇지 않다면 Amazon RDS > 서브넷 그룹에서  
      ec2의 vpc와 동일한 서브넷 그룹을 생성한다.
      - 이 부분은 [링크](https://ndb796.tistory.com/226){:target="_blank"}
에 더 상세하게 나와있으니 필요하다면 참고
    - 보안그룹
      - 위에서 RDS 용으로 생성한 보안그룹 ID를 적는다.
    - 나머지는 그대로 두고 수정
- 확인
  - ec2 인스턴스로 접속
  - mysql-client 설치
    ```
    # ubuntu 기준
    $ sudo apt-get install mysql-client
    ```
  - 접속 명령
    ```
    $ mysql -u [db-user] -p -h [db-host] -P 3306
    ```
  - 접속이 성공했거나, 실패 시 실패메세지가 바로 나온다면  
  RDS 보안그룹이 성공적으로 변경된 것
  - 실패 메세지가 바로 안 나오고 time out 될 때까지 대기한다면  
  보안그룹 변경이 실패한 것

## 참고
- [AWS EC2에 AWS RDS 연동하기](https://ndb796.tistory.com/226){:target="_blank"}

