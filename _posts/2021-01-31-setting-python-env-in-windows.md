---
title: windows에서 python 환경 세팅
date: 2021-01-31 12:36:00 +0900
categories: [Python]
tags: [miniconda, python, env]    # TAG names should always be lowercase
---
## 개요
- 친구에게 python 환경 세팅 방법을 공유하기 위해 이 글을 남긴다.
- 보편적으로 사용하는 os는 windows이기 때문에 windows 기반으로 작성한다.

## miniconda 설치
- 설명
  - 가상환경 세팅을 쉽게 하기 위해 miniconda를 이용한다.
- 실행방법
  - [miniconda 설치 링크](https://docs.conda.io/en/latest/miniconda.html){:target="_blank"}
  - python 3.x(지금은 3.8)의 windows 64 bit 인스톨러를 선택하여 다운로드 한다.
  - 인스톨러를 실행하여 다운로드 받는다.

## 가상환경 생성
- 설명
  - 한 컴퓨터에서 다양한 python 프로젝트를 만들 수 있다.
  - 각 python 프로젝트를 독립적으로 만들기 위해 각각 가상환경을 구축해줘야한다.
- 실행방법
  - 윈도우 검색 바에서 prompt를 검색하여 miniconda prompt를 실행한다.
  - my_env라는 가상환경 생성  
    ```
    > conda create -n my_env python=3.8
    ```
  - 가상환경 생성 확인  
    ```
    > conda info --env
    ```

## github에서 소스코드 다운로드
- 설명
  - python 코드를 github에서 내려 받는다.
  - 해당 reposiotry로 가서 원하는 위치에 clone을 받으면 된다.
- 실행방법
  - git repository로 간다.
  - code 버튼을 누르고 https를 선택한 뒤 해당 clone 명령어를 복사한다.
  - 윈도우 검색 바에서 bash를 검색하여 git bash를 실행한다.
  - cd [디렉토리] 명령을 이용하여 원하는 위치로 이동한다.
  - 해당 위치에 도착하면 git clone [복사한 명령어]를 실행하여 git clone을 실시한다.

## 의존성 패키지 설치
- 설명
  - 소스코드에서 사용 중인 외부 패키지를 설치해줘야 한다.
- 실행방법
  - 윈도우 검색 바에서 prompt를 입력하여 minconda prompt를 실행한다.
  - 아래 명령어로 가상환경을 활성화시킨다.
    ```
    > conda activate my-env
    ```
  - cd [디렉토리] 명령을 이용하여 원하는 위치로 이동한다.
  - 해당 위치에 도착하면 아래 명령어로 의존성 패키지를 설치한다.
    ```
    > pip install -r requirements.txt
    ```


## 실행
- 설명
  - REAME.md에 나온 방법대로 실행하면 된다.
