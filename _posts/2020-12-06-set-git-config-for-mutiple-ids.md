---
title: 한 pc에 여러 github id ssh 키 등록하기
date: 2020-12-06 12:58:00 +0900
categories: [Git]
tags: [git, github, ssh]    # TAG names should always be lowercase
---
## 개요
- 때때로 여러 github id를 개인 pc에서 사용할 때가 있다.
- 이때 매번 로그인하기 귀찮으므로 각 github id 마다 ssh 키를 등록해보자

## 환경
- os: Ubuntu 18.04.5 LTS

## ssh 키 생성
- 설명
    - id 마다 ssh 키를 생성해준다.
    - 예시에서는 편의상 id1, id2로 지칭한다.
    - 이메일과 rsa 파일명은 상황에 맞게 수정한다.
- 실행방법
    - ssh 디렉토리 절대경로 확인
        ```
        $ cd ~/.ssh && pwd
        ```
    - id1 ssh 키 생성
        ```
        $ ssh-keygen -t rsa -C "id1@gmail.com" -f "[ssh 디렉토리 경로]/rsa-id1"
        ```
    - id2 ssh 키 생성
        ```
        $ ssh-keygen -t rsa -C "id2@gmail.com" -f "[ssh 디렉토리 경로]/rsa-id2"
        ```
    - 생성 확인
        ```
        $ ls ~/.ssh
        ```

## ssh 퍼블릭 키 깃헙에 등록
- 설명
    - github id 마다 생성한 ssh 퍼블릭 키를 각각 github에 추가해준다.
    - 예시는 id1 기준이며, id2도 같은 방법으로 하면 된다.
- 실행방법
    - ssh 퍼블릭 키 복사
        ```
        # 출력 결과 복사
        $ cat ~/.ssh/rsa-id1.pub 
        ```
    - 웹 브라우저를 켜고 github id1 로그인
    - 웹 브라우저에서 https://github.com/settings/keys 복사 붙여넣기 엔터  
      (github 본인 프로필 아이콘 클릭 > Settings > SSH and GPG Keys를 클릭하여 이동할 수도 있다.)  
    - New SSH Key 버튼 클릭 
    - 복사한 퍼블릭키를 붙여넣기하고 등록

## ssh config 파일 생성
- 설명
    - git clone 시 이 repo가 참고하고 있는 ssh 키가 무엇인지 알려주는 config 파일을 생성해야한다.
    - 이미 config 파일이 생성되어 있다면 아래 내용대로 수정한다.
    - 어떤 에디터를 사용해도 되지만 예시에서는 vim을 사용한다.
- 실행방법
    - vim으로 config 파일을 연다
        ```
        $ vim ~/.ssh/config
        ```
    - 아래 내용울 추가한다.
        ```
        # config
        Host github.com
            HostName github.com
            User git
            IdentityFile ~/.ssh/rsa-id1

        Host github.com-id2
            HostName github.com
            User git
            IdentityFile ~/.ssh/rsa-id2
        ```
- config 설명
    - Host
        - git clone 시 이 repo가 어떤 ssh 키를 참고하는지 알 수 있게 해주는 문자열이다.
        - 기본값은 github.com
    - IdentityFile
        - github에서 등록한 ssh 퍼블릭 키에 대응되는 프라이빗 키의 경로다.
    - HostName, User는 수정하지 않는다.

## __[중요]git clone__
- 설명
    - 기본으로 생성되는 ssh clone 명령어에서 반드시 config에서 등록한 github id의 Host로 변경해야한다.
    - git clone 뿐만아니라 git remote add로 원격 repo를 추가할 때에도 마찬가지다.
- 실행방법
    - 웹 브라우저를 켜고 본인이 clone하고자 하는 repo 페이지로 이동한다.
    - Code 버튼을 누르고 Clone 모달에서 ssh를 누른다.
    - ssh 선택 시 노출된 명령어를 복사한다.  
      ex)git@github.com:id1/id1.github.io.git
    - 여기서 github.com을 github id 별로 본인이 ~/.ssh/config에서 등록한 host로 변경해야한다.
    - 그리고 아래 예시와 같이 clone 명령을 실행한다.
- 예시
    -  id1가 id1.github.io repo를 clone 하는 경우
        - id1는 ~/.ssh/confg에서 등록한 hsot가 github.com이므로 별도의 수정이 없다.
            ```
            $ git clone git@github.com:id1/id1.github.io.git
            ```
    - id2가 id2.github.io repo를 clone 하는 경우
        - id2는 ~/.ssh/confg에서 등록한 hsot가 github-id2이므로 github.com을 github-id2로 수정해야한다.
            ```
            $ git clone git@github-id2:id1/id1.github.io.git
            ```
- 결과
    - 정상적으로 clone이 되었다면 성공한 것이다.
    - access denied 에러가 뜨면서 clone이 안 된다면 config를 설정하지 않았거나  
      config에서 설정한 Host를 clone 명령어에 적용하지 않았을 확률이 높다.

## 참고
- [Multiple SSH Keys settings for different github account](https://gist.github.com/jexchan/2351996){:target="_blank"}

- [[Github] SSH 를 활용하여 여러 계정 관리 방법](https://medium.com/@sunnkis/github-ssh-%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%98%EC%97%AC-%EC%97%AC%EB%9F%AC-%EA%B3%84%EC%A0%95-%EA%B4%80%EB%A6%AC-%EB%B0%A9%EB%B2%95-7ec29bd0186d){:target="_blank"}
