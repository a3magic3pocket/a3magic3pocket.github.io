---
title: git push 시 자동으로 pull and push하는 shell script 작성
date: 2021-07-23 18:37:00 +0900
categories: [Shell]
tags: [shell script, git] # TAG names should always be lowercase
---

## 개요
- git push 전 git db를 최신화 시키고 이를 rebase 하여 local에서 conflict 처리를 하고자 한다.
- 이를 위해 해당 기능의 명령어를 shell script로 작성하여 해결하였다.

## 발단(형상관리 방법의 변화)
- 메인페이지의 빈번한 수정이 요청됨에 따라 배포 주기가 엄청 짧아졌다.
- 이에 대응하기 위해 형상관리 방법을 아래와 같이 간결하게 변경하였다.
    - 기존 방식
        1. (팀장) 공용 remote github repo 생성
        2. (팀장) 실서버 배포용으로 \<main\> 브런치, 개발용 브런치로 \<dev\> 브런치 생성
        3. (개발자) remote github repo를 fork 뜸(앞으로 forked repo라 명명) 
        4. (개발자) local에 forked repo를 클론하고 \<dev\> 브런치 생성
        5. (개발자) local \<dev\>에서 수정 후 commit하고 forked repo \<dev\>로 push
        6. (개발자) forked repo \<dev\>에서 변경사항을 remote github repo \<dev\>로 pull request
        7. (팀장) pull request 확인 후 merge 할지 결정
    - 변경된 방식
        1. (팀장) 공용 remote github repo 생성
        2. (팀장) 실서버 배포용으로 main 브런치, 개발용 브런치로 dev 브런치 생성
        3. (개발자) local에 remote github repo를 클론하고 \<dev\> 브런치 생성
        4. (개발자) local \<dev\>에서 수정 후 commit하고 remote github repo \<dev\>로 push
- 발생 가능한 문제
    1. 여러 개발자가 하나의 repo에 직접 push하기 때문에 conflict이 날 가능성이 높다.
    2. 개발물 적용 전 (팀장)의 확인 과정이 없어 불완전한 코드가 merge 될 가능성이 존재한다.  
       이 부분은 실서버 반영전 테스트 서버에 우선 배포해 충분 테스트를 하여 어느 정도 해소한다.
- 발생 가능한 문제 1.의 해결
    1. push 후 conflict이 나면 github에서 어떤 것을 반영할지 정하고 수정해서 merge 해야한다.
    2. 이는 local에서 conflict을 처리하는 것보다 매우 불편함으로 되도록  
        git push 전 git db를 최신화 시키고 이를 rebase 하여 local에서  
        conflict 처리를 하고자 한다.
    3. 이를 위한 shell script를 작성하여 더 편하게 push를 하도록하였다.

## local에서 conflict 처리
1. remote github repo의 최신 git db를 local git에 반영하고 현재 commit에 rebase
    - ```bash
        ## git pull --rebase [remote repo nickname in local] [local branch name]
        # remote repo nickname in local을 여기선 'upstream' 이라고 가정
        # 우리는 현재 dev 브런치만 사용한다고 가정
        git pull --rebase upstream dev
        ```
    - 이 과정에서 conflict이 발생할 수 있다.
    - conflict이 발생하면 적절히 판단하여 조치 후 merge commit을 하면 된다.
2. local의 지금까지 변경사항을 remote github repo에 push
    - ```bash
        ## git push [remote repo nickname in local] [local branch name]
        # remote repo nickname in local을 여기선 'upstream' 이라고 가정
        # 우리는 현재 dev 브런치만 사용한다고 가정
        git push upstream dev
        ```
3. local의 지금까지 변경사항을 forked repo에 push
    - ```bash
        ## git push [forked repo nickname in local] [local branch name]
        # forked repo nickname in local을 여기선 'origin' 이라고 가정
        # 우리는 현재 dev 브런치만 사용한다고 가정
        git push origin dev
        ```
    - 현재는 forked repo를 쓰지 않지만 그래도 최신화 해줌
4. 1~3과정을 순차적으로 실행하되 하나라도 실패하면  
   그 다음 과정을 진행하지 않고 멈춰야한다. 

## Shell script
- ```bash
    #!/usr/bin/env bash
    TRUE=1
    FALSE=0

    function is_succeeded_git_command() {
        local message=$1

        if [[ ${message} == *error* ]]; then
            return ${FALSE}
        fi

        return ${TRUE}
    }

    function sleep_with_msg() {
        local sleep_sec=$1
        echo "+-- sleep ${sleep_sec} sec"
        sleep ${sleep_sec}
    }

    function pull_and_push() {
        echo "+-- pull --rebase from upstream dev"
        result=1

        # Catch stderr of this command
        result=`git pull --rebase upstream dev 2>&1`
        sleep_with_msg 0.5

        is_succeeded_git_command "${result}"
        is_succeeded=$?
        if [[ is_succeeded -eq ${FALSE} ]]; then
            echo "${result}"
            echo "+-- failed"
            return 9001 
        fi
        echo "echo result :: ${result}"

        # Catch stderr of this command
        echo "+-- push from local dev to upstream dev"
        result=`git push upstream dev 2>&1`
        sleep_with_msg 0.5

        is_succeeded_git_command "${result}"
        is_succeeded=$?
        if [[ is_succeeded -eq ${FALSE} ]]; then
            echo "${result}"
            echo "+-- failed"
            return 9001
        fi
        echo "echo result :: ${result}"

        # Catch stderr of this command
        echo "+-- push from local dev to origin dev"
        result=`git push origin dev 2>&1`
        sleep_with_msg 0.5

        is_succeeded_git_command "${result}"
        is_succeeded=$?
        if [[ is_succeeded -eq ${FALSE} ]]; then
            echo "${result}"
            echo "+-- failed"
            return 9001
        fi
        echo "echo result :: ${result}"

        echo "+-- done"
    }

    pull_and_push
    ```

## 주요 shell 문법
- 함수
    - 선언
        - ```bash
            function 함수이름() {
                return 리턴값
            }
            ```
    - 실행
        - ```bash
            # 함수이름만 적으면 실행된다.
            # 인자는 공백으로 구분한다.
            함수이름 인자1 인자2
            ```
    - 받은 인자의 처리
        - ```bash
            # 받은 인자는 순서대로 $순서 변수에 담긴다.
            # ex. 함수이름 인자1 인자2이면 인자1은 $1, 인자2는 $2에 담긴다.
            # shell에서 그냥 변수 선언 및 할당하면 무조건 global이다.
            # 함수 안에서만 동작하는 변수를 쓰고 싶다면 local 명령어를 써야한다.
            function 함수이름() {
                local 인자1=$1 #인자1에 9가 담김
                local 인자2=$2 #인자2에 a가 담김
                return 리턴값
            }

            함수이름 9 "a"
            ```
    - 함수 리턴값 받기
        - ```bash
            # $?라는 특수 변수는 마지막으로 실행된 명령어, 함수 등의 종료값을 받는다.
            function 함수이름() {
                return 9
            }

            함수이름
            echo "$?" # 이 스크립트를 실행하면 9가 출력된다.
            ```
## is_succeeded_git_command 함수 설명
- 깃 명령어 실행결과 문자열을 받아 성공 여부를 체크하는 함수이다.
- bash shell에서는 boolean type으로 true, false을 사용하나  
  나의 경우 boolean type 테스트가 매우 헷갈려서 그냥 숫자로 할당하여 처리하였다.  
  [shell boolean type](https://stackoverflow.com/questions/2953646/how-can-i-declare-and-use-boolean-variables-in-a-shell-script){:target="\_blank"}
- bash shell에서는 equal test도 여러개이다([shell equality operators](https://stackoverflow.com/questions/20449543/shell-equality-operators-eq){:target="\_blank"}).
    - [[ \${a} = \${b} ]]는 POSIX shell 기반으로 문자열 변수 동일여부 체크 시 사용한다.  
      이 방법은 정규 표현식 처리가 불가능하다.
    - [[ \${a} == \${b} ]]는 bash shell 기반으로 문자열 변수 동일여부 체크 시 사용한다.  
      이 방법은 정규 표현식 처리가 가능하다.
    - [[ \${a} -eq \${b} ]]는 bash shell 기반으로 정수형 변수 동일여부 체크 시 사용한다.  
- git 명령어 실패 시 에러메세지 앞에는 error라는 문구가 항상 포함됨으로  
  [[ \${message} == \*error\* ]] 테스트를 통해 git 명령어 에러 여부를 판단한다  
  (표준에러와 표준출력이 섞여서 있어서 좀 모호하긴 함 ^^;;)

## pull_and_push 함수 설명
- 주요 로직이 포함된 함수이다.
- git pull --rebase upstream dev 2>&1
    - [2>&1 이해하기]{https://blogger.pe.kr/369}를 먼저 읽으면 좋다.
    - 2는 파일디스트럭터 중 하나로 표준에러를 의미한다.
    - 1은 파일디스트럭터 중 하나로 표준출력을 의미한다.
    - 2>&1을 쓰면 표준에러를 표준출력으로 보내겠다는 뜻이다.
    - 2>&1이 없으면 에러가 표준출력으로 변환되지 않기 때문에 '|' 등의 문법을 쓸 수 없다.
- 명령어 표준출력을 변수에 할당
    - 변수이름=\`명령\`
    - 따라서 result=\`git pull --rebase upstream dev 2>&1\`는   
      git pull --rebase 명령 실행 결과(표준출력, 표준에러 포함)가 result 변수에 저장된다.
