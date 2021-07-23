---
title: local git repository에서 git user 세팅
date: 2020-12-06 12:58:00 +0900
categories: [Git]
tags: [git, ssh]    # TAG names should always be lowercase
---
## 개요
- 가끔씩 clone 한 git repo에 user setting이 되지 않아서  
  원격 repo의 commit 목록에 global user가 노출되는 경우가 있다.  
- 화사 프로젝트 repo와 같이 여러 사람이 같이 쓰는 private repo에서 이런 일이 발생하면  
  관리자의 따가운 시선을 받을 수 있다.

## local git repo에서 git user 세팅
- 실행방법
    ```
    $ git config --local user.name "id1"
    $ git config --local user.email "id1@gmail.com"
    ```
- 실행방법2
    - 직접 local repo 디렉토리에서 .gitconfig 파일을 수정해도 된다.
    ```
    $ vim [local repo 디렉토리 경로]/.gitconfig 
    # .gitconfig
    [user]
        name = id1
        email = id1@gmail.com
    ```
        
## 참고
- [깃(git) - 프로젝트/저장소마다 다른 계정 정보 사용하기](https://gist.github.com/jexchan/2351996){:target="_blank"}