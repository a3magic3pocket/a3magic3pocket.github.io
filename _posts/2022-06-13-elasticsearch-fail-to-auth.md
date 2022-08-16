---
title: security_exception, failed to authenticate user [x-pack-user] 에러
date: 2022-06-13 18:40:00 +0900
categories: [ELK]
tags: [elk, elasticsearch, x-pack, auth]    # TAG names should always be lowercase
---
## 개요
- elasticsearch heap 메모리 부족 에러 발생 시 x-pack으로 생성한 유저의 로그인이 무조건 실패하기 시작했다.
- elasticsearch 개별 서버에 슈퍼 권한을 가진 로컬 유저를 생성하여 해결할 수 있다.
- 슈퍼 권한 로컬 유저로 x-pack-user의 비밀번호를 변경하면 x-pack-user 역시 로그인이 가능해진다.
- 다만 elasticsearch heap 메모리 부족 에러가 해결되면 x-pack 유저 로그인 에러도 자동으로 해결된다.

## elasticsearch 로컬 유저 생성
- 설명
    - 슈퍼권한을 가진 로컬 유저를 생성한다.
- 명령
    - ```bash
        $ sudo /usr/share/elasticsearch/bin/elasticsearch-users useradd tmp-local-user -p 슈퍼관리자비번 -r superuser
        ```

## 기존 x-pack-user 비밀번호 변경
- 설명
    - 로컬 유저로 기존 x-pack-user의 비밀번호를 변경한다.
- 명령
    - ```bash
        curl -XPOST \
            -u tmp-local-user \
            "http://localhost:9200/_security/user/x-pack-user/_password" \
            -H "Content-Type: application/json" \
            -d '{"password" : "바꿀비번"}'
        ```

## 모든 인덱스 삭제 시 비밀번호 정보 제거
- 설명
    - elasticsearch에 등록된 모든 인덱스를 삭제할 경우  
      비밀번호 정보가 담긴 인덱스도 삭제된다.
    - 이 경우에도 위 방법으로 비밀번호를 세팅해주면 해결된다.
    - kibana_system, apm_system, beats_system 유저들도 같은 명령어로 비밀번호를 바꿔준다.

## 참고
- [엘라스틱서치 공식 홈페이지 elasticsearch-users](https://www.elastic.co/guide/en/elasticsearch/reference/current/users-command.html){:target="_blank"}
- [Help! I don't have the password for the elastic user!](https://discuss.elastic.co/t/x-pack-authentication-issue/121632/6){:target="_blank"}
