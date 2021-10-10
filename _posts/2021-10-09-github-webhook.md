---
title: github webhook을 이용한 원격서버 배포 자동화(AWS)
date: 2021-10-09 23:25:00 +0900
categories: [Git]
tags: [github, webhook, deployment, aws] # TAG names should always be lowercase
---

## 개요
- 코드를 수정하고 테스트 서버에 반영하는데 매번 똑같은 pull 명령만 날리는 것 같아 이를 자동화했다.
- github webhook을 이용하면 git remote repo에서 push가 발생했을 때 지정한 주소로 post 명령을 날려준다.
- 테스트 서버에서는 github webhook을 바라 보고 있는 작은 웹서버를 구동하다가,  
  요청이 들어오면 미리 작성한 script를 실행하여 배포한다.

## 환경
- os
    - ubuntu 20.04 lts
- cloud 
    - AWS
- 구조
    - remote repo
        - https://github.com/a3magic3pocket/github-webhook-test
    - local
        - 로컬에 remote repo를 미리 clone 해둔다.
    - test server
        - 테스트 서버에 remote repo를 미리 clone 해둔다.

## Webhook 이란?
- 간단하게 생각하면 서버에서 클라이언트로 요청을 보내는 것이다(보통의 API와 반대로 동작)
- [webhook 정의](https://en.wikipedia.org/wiki/Webhook)
- 이 말은 클라이언트에서도 서버에서 보낸 요청을 받을 수 있는 작은 웹서버가 필요하다는 이야기가 된다.

## adnanh / webhook
- 개요
    - 테스트 서버에서 github webhook 요청을 받을 수 있는 작은 웹서버를 짜야하나 고민하다가  
      왠지 미리 구현된 패키지가 있을 것 같아 찾아보니 역시 있었다.
    - "adnanh / webhook"은 golang 기반이며 apt-get 패키지로도 배포하고 있다.
    - 설치 후 간단한 설정파일만 작성하고 실행하면 바로 사용할 수 있다.
- 하고자 하는 것
    1. local에서 코드를 수정하고, add, commit 후 remote repo로 push 한다.
    2. remote repo에서는 설정한 webhook 대로 test server로 post 요청을 보낸다.
    3. test server의 "adnanh / webhook"는 remote repo에서 요청을 받으면 지정된 script를 실행시켜 배포한다.
- adnanh / webhook 설치
    - ```bash

        $ sudo apt-get update && sudo apt-get upgrade
        $ sudo apt-get install webhook

        $ cd
        $ mkdir webhook & cd webhook
        $ mkdir scripts

        $ touch scripts/github-webhook-test-deploy.sh
        # 프로그램이 실행할 수 있도록 권한 변경
        $ chmod 755 scripts/github-webhook-test-deploy.sh
        
        ```
- 스크립트 작성
    - ```bash

        # webhook/scripts/github-webhook-test-deploy.sh
        # 문제를 간단하게 하기 위해 우선 branch는 main 하나만 운영한다고 가정한다.
        #!/usr/bin/env bash
        git --git-dir=/home/ubuntu/github-webhook-test/.git pull origin main

        ```
- 설정
    - ```bash

        # webhook/hook.json
        # command-working-directory는 test server에서 git clone한 디렉토리로 지정해준다.
        [
            {
                "id": "github-webhook-test-deploy",
                "execute-command": "/home/ubuntu/webhook/scripts/github-webhook-test-deploy.sh",
                "command-working-directory": "/home/ubuntu/github-webhook-test"
            }
        ]

        ```
- 실행
    - ```bash
        # 실행하면 webhook URL을 얻을 수 있다.
        $ webhook -hooks /home/webhook/hooks.json -verbose
        ```

## Github webhook 설정
- 설명
    - 웹 브라우저를 켜고 remote repo 페이지로 접속
    - Settings > (왼쪽)Webhook > add webhook 버튼 클릭
    - 테스트 서버 웹 URL을 등록해준다.
- 실행
    - <a href="/assets/img/2021-10-09-github-webhook/00-add-webhook.jpg" target="_blank"><img src="/assets/img/2021-10-09-github-webhook/00-add-webhook.jpg" width="100%"></a>

## AWS 보안그룹 추가
- 개요
    - Github webhook 요청을 테스트 서버에서 받을 수 있도록 보안그룹을 추가해줘야 한다.
- git webhook ip 확인
    - https://api.github.com/meta로 GET 요청을 하면 확인할 수 있다.
    - <a href="/assets/img/2021-10-09-github-webhook/01-github-api-ip-list.jpg" target="_blank"><img src="/assets/img/2021-10-09-github-webhook/01-github-api-ip-list.jpg" width="70%"></a>
    - 보여지는 IP는 CIDR 표기법이다. [CIDR 설명](https://kim-dragon.tistory.com/9)  
      등록된 IP만 허용되는게 아니고 CIDR 표기로 허용한 하위 서브넷 IP까지 모두 허용된다.
    - AWS 보안그룹에 github webhook ip 규칙을 추가해준다.
- AWS 보안그룹 생성
    - AWS에 접속 > ec2 인스턴스 > (왼쪽 메뉴) 네트워크 및 보안 / 보안그룹 > 보안 그룹 생성 클릭
    - <a href="/assets/img/2021-10-09-github-webhook/02-make-security-group.jpg" target="_blank"><img src="/assets/img/2021-10-09-github-webhook/02-make-security-group.jpg" width="100%"></a>
    - 테스트 서버 vpc와 같은 vpc를 선택한다.
    - 인바운드 규칙의 포트는 webhook 포트로 지정한다.
- AWS 보안그룹 추가
    - 테스트 서버 ec2 인스턴스에 새로 생성한 github webhook 보안그룹을 추가해준다.
    - ec2 인스턴스 > (우클릭) > 보안 > 보안그룹 변경 클릭
    - <a href="/assets/img/2021-10-09-github-webhook/03-add-security-group-1.jpg" target="_blank"><img src="/assets/img/2021-10-09-github-webhook/03-add-security-group-1.jpg" width="70%"></a>
    - github webhook 보안그룹을 테스트 서버 ec2 인스턴스에 추가한다.
    - <a href="/assets/img/2021-10-09-github-webhook/04-add-security-group-2.jpg" target="_blank"><img src="/assets/img/2021-10-09-github-webhook/04-add-security-group-2.jpg" width="100%"></a>
    - !주의 - 기존 ec2 보안그룹에 github webhook 보안그룹을 추가하는 방법과 위의 방법은 서로 다르다.
        - 기존 ec2 보안그룹의 하위에 github webhook 보안그룹을 추가하는 것은  
          github webhook 보안그룹을 가진 ec2-2에서 기존 ec2 보안그룹을  
          가진 ec2-1에 요청하는 것을 허용한다는 의미이다.  
          예를들어, rds에서 특정 ec2 접근을 허가하기 위해서는   
          rds 보안그룹에 해당 ec2 보안그룹을 추가해야 한다.  
          [AWS RDS에 EC2 접속 허용하기](https://a3magic3pocket.github.io/posts/aws-rds-allow-ec2/)
- webhook 받기 성공
    - 이제 local에서 코드를 수정한 후 add, commit, push를 하면 webhook이 작동됨을 확인할 수 있다.
    - <a href="/assets/img/2021-10-09-github-webhook/05-success-1.jpg" target="_blank"><img src="/assets/img/2021-10-09-github-webhook/05-success-1.jpg" width="70%"></a>

## 암호 설정
- 설명
    - 사실 테스트서버의 webhook URL만 알아낸다면 github에서 보낸 요청이 아니라더라도  
      POST 명령을 날려서 지정된 스크립트를 실행시킬 수 있다.
    - SECRET 키를 이용하여 github에서 보낸 요청에만 스크립트가 실행되도록 할 수 있다.
- 실행
    - 20~30 글자의 랜덤 문자열을 생성한다.
    - github webhook 설정에서 위의 문자열을 SECRET 키로 지정한다.
    - webhook 설정에 SECRET키가 적용되도록 수정한다.
    - ```bash

        # webhook/hook.json
        # !주의 - webhook 2.8.0(latest release) 버전에서는 type이 payload-hash-sha256이지만  
        #        github master branch를 보면 payload-hmac-sha256로 되어있다.
        #        현재 apt-get으로 받은 경우 payload-hash-sha256으로 지정해야 실행된다.
        [
            {
                "id": "github-webhook-test-deploy",
                "execute-command": "/home/ubuntu/webhook/scripts/github-webhook-test-deploy.sh",
                "command-working-directory": "/home/ubuntu/github-webhook-test",
                "trigger-rule":
                {
                    "match":
                    {
                        "type": "payload-hash-sha256",
                        "secret": "my-secret-key",
                        "parameter":
                        {
                            "source": "header",
                            "name": "X-Hub-Signature-256"
                        }
                    }
                }

            }
        ]

        ```

## 브랜치를 구분하여 배포
- 설명
    - github webhook에서 보낸 요청을 보면 body에 branch가 기록되어 있다.
    - "adnanh / webhook"에서 body를 parsing하여 branch name을 얻어 스크립트에 적용한다.
- github webhook에서 최근 보낸 메세지 확인
    - <a href="/assets/img/2021-10-09-github-webhook/06-recent-deliveries.jpg" target="_blank"><img src="/assets/img/2021-10-09-github-webhook/06-recent-deliveries.jpg" width="100%"></a>
- 설정
    - ``` bash

        # webhook/hook.json
        # parse-parameters-as-json으로 github webhook 요청 body를 parsing
        # pass-environment-to-command로 parsing된 json에서 key가 ref인 것만 추출하여  
        # 스크립트 실행 시 ref라는 환경변수 할당
        [
            {
                "id": "github-webhook-test-deploy",
                "execute-command": "/home/ubuntu/webhook/scripts/github-webhook-test-deploy.sh",
                "command-working-directory": "/home/ubuntu/github-webhook-test",
                "parse-parameters-as-json":
                [
                    {
                        "source": "payload",
                        "name": "payload"
                    }
                ],
                "pass-environment-to-command":
                [
                    {
                        "source": "payload",
                        "name": "payload.ref",
                        "envname": "ref"

                    }
                ],
                "trigger-rule":
                {
                    "match":
                    {
                        "type": "payload-hash-sha256",
                        "secret": "my-secret-key",
                        "parameter":
                        {
                            "source": "header",
                            "name": "X-Hub-Signature-256"
                        }
                    }
                }

            }
        ]

        ```
- 스크립트 수정
    - ```bash

        # webhook/scripts/github-webhook-test-deploy.sh
        #!/usr/bin/env bash

        branch=${ref##refs/heads/}
        git --git-dir=/home/ubuntu/github-webhook-test/.git pull origin ${branch}

        ```
- 결과
    - <a href="/assets/img/2021-10-09-github-webhook/07-success-2.jpg" target="_blank"><img src="/assets/img/2021-10-09-github-webhook/07-success-2.jpg" width="70%"></a>

## 백그라운드 실행
- 설명
    - 세션을 종료해도 계속 실행되도록 백그라운드로 실행한다.
- 실행
    - ```bash
        $ nohup webhook -hooks /home/webhook/hooks.json -verbose &
        ```

## 참고
- [webhook 정의](https://en.wikipedia.org/wiki/Webhook)
- [CIDR 설명](https://kim-dragon.tistory.com/9)  
- [Referencing request values](https://github.com/adnanh/webhook/blob/master/docs/Referencing-Request-Values.md)
- [Hook Examples](https://github.com/adnanh/webhook/blob/master/docs/Hook-Examples.md#incoming-github-webhook)
- [Hook definition](https://github.com/adnanh/webhook/blob/master/docs/Hook-Definition.md)
- [About webhooks](https://docs.github.com/en/developers/webhooks-and-events/webhooks/about-webhooks)