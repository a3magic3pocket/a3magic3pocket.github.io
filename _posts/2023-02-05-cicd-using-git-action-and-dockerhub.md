---
title: git action과 dockerhub을 이용한 CI/CD 파이프라인 구축
date: 2023-02-05 12:49:00 +0900
categories: [cicd]
tags: [git, git-action, dockerhub, cicd] # TAG names should always be lowercase
---
## git action과 dockerhub을 이용한 CI/CD 파이프라인 구축 글 쓰기
- CI/CD란
    - 상세한 정의
        - Red Hat 문서에 상세하게 잘 적혀있다.
            - [CI/CD(CI CD, 지속적 통합/지속적 배포): 개념, 툴, 구축, 차이 (redhat.com)](https://www.redhat.com/ko/topics/devops/what-is-ci-cd){:target="_blank"}
            - [CI/CD 파이프라인: 개념, 방법, 장점, 구현 과정 (redhat.com)](https://www.redhat.com/ko/topics/devops/what-cicd-pipeline){:target="_blank"}
    - CI
        - 여러 개발 소스를 자동으로 합쳐준다.
        - 임시로 합친 결과물은 테스트(test)를 통과해야하며   
            통과하지 못하면 최종 소스코드에 반영되지 않는다.
        - 예시
            - 가정
                - A와 B 개발자가 함께 사용하는 github 레포지토리 'MyWork'가 있다.
                - 'MyWork'에는 main, develop이라는 두 개의 브랜치가 존재한다.
                - main는 최종 상품에 해당하는 브랜치이다.
                - develop은 개발자의 개발본이 합쳐지는 브랜치이다.
            - CI 정책을 아래와 같이 가져갈 수 있다.
                - A가 local 'MyWork'에서 개발 후 개발 내역을  
                    remote 'MyWork' 브랜치로 push 한다.
                - push된 소스코드가 유효한지 CI서비스에서 테스트를 진행한다.
                - 만약 테스트를 통과하였다면 remote 'MyWork' develop 브랜치에 merge 된다.  
                - 만약 테스트를 통과하지 못하였다면   
                    remote 'MyWork' develop 브랜치에 merge 되지 않는다.
                - 'MyWork'에서 개발하는 모든 개발자는 동일한 과정을 겪는다.
    - CD
        - CI를 통과한 소스코드는 테스트를 통과하였기 때문에   
            검증되었다고 볼 수 있다.
        - 이를 믿고 실제 서버로 최신 소스코드를 자동으로 배포한다.
        - 예시
            - 가정
                - CI 환경과 동일한 환경이다.
                - main는 직접 push가 불가능하고 개발자 A가 develop 브랜치에서 pull request 하여 merge 할 수 있다.
                - main브랜치 역시 소스코드가 업데이트되면 CI 서비스가 테스트를 시작한다.
                - 테스트를 통과하면 배포를 시작한다.
                - 서비스는 docker container 기반으로 운영 중이라고 가정한다.
                - 서비스 배포는 아래와 같이 진행된다.
                    - CI 서비스에서 테스트를 위해 이미지 생성 
                    - CI 서비스에서 docker registry 서버로 이미지 push
                    - docker registry 서버에서 서비스 서버로 최신 이미지가  
                        push 되었다고 알림
                    - 서비스 서버에서 docker registry  서버에 있는 최신 이미지를   
                        가져와서 배포
            - CD 정책을 아래와 같이 가져갈 수 있다.
                - A가 'MyWork' develop 브랜치에서 main 브랜치로 pull request를  
                    진행하고 merge 한다.
                - CI 서비스에서 main 브랜치의 최신 소스코드의 테스트가 진행된다.
                - 테스트 시 빌드한 이미지가 존재할 것이다. 테스트를 통과하였다면 docker registry 서버로 이미지를 push한다.
                - docker registry 서버는 이미지를 받으면 서비스 서버에 이를 알린다.
                - 서비스 서버는 새로 업데이트된 이미지를 docker registry 서버로부터 가져와 배포를 진행한다.
- 실험환경
    - CI
        - 우리는 git remote repository를 github을 사용할 것이다.
        - CI는 git actions을 이용하여 구축할 것이다.
    - Docker registry server
        - Dockerhub를 이용할 것이다.
    - CD
        - 서비스 서버에 Webhook을 설치하여 Dockerhub에서 webhook URL을 입력하여  배포할 것이다.
    - 실험 시나리오
        - github remote repository에서 main, develop 브랜치를 운영한다.
        - 개발자가 main 브랜치에 직접 push 하는 것을 금지한다.
        - develop 브랜치로 새로운 커밋이 push가 되면 테스트를 진행하고 테스트를 
    통과해야만 develop 브랜치로 merge 한다.
        - 팀장이 갱신된 develop 브랜치를 확인하고 문제가 없으면 main 브랜치로 
    pull request 한다.
        - main 브랜치로 merge 요청이 오면 테스트를 진행하고 테스트를 통과해야만 
    main 브랜치로 merge 한다.
        - 테스트를 통과하였다면 새로 빌드된 이미지를 Dockerhub로 push 한다.
        - Dockerhub는 새 이미지를 push 되면 등록된 webhook URL로 GET 요청을 보낸다.
        - 서비스 서버 Webhook은 지정된 URL로 요청을 받으면 배포 스크립트를 실행하여 배포를 진행한다.
    - 실험용 github repository 
        - [simple-web-example](https://github.com/a3magic3pocket/simple-web-example){:target="_blank"}
    - 실험시작
        - git actions 스크립트 설명
            - [github actions 개요](https://docs.github.com/ko/actions/learn-github-actions/understanding-github-actions){:target="_blank"}
            - 워크플로(workflow)
                - 하나의 이벤트 트리거가 발생하였을때 진행해야하는 작업 묶음을 의미한다.
                - github remote repository의 .github/workflows 디렉토리 하위에 yml로 작성하면 된다.
                - 워크플로는 크게 '이벤트'와 '작업'으로 나눌 수 있다.
            - 이벤트(event)
                - 이벤트는 '작업'들을 실행시키는 트리거로 사용할 수 있는 활동들이다.
                - 예를들러 특정 브랜치에 push, pull request 활동들을 트리거로 쓸 수 있다.
            - 작업
                - 트리거로 설정된 이벤트가 발생하면 미리 지정한 작업들이 순차적으로 시작된다.
                - 보통 CI 과정에서는 환경설정 작업, 테스트 작업이 포함된다.
            - 워크플로 기본 구조
                - ```yml
                    name: <워크플로 이름>

                    on: [<이벤트>]
                    jobs:
                      <작업명>:
                        runs-on: <os 이미지명>
                        
                        steps:
                          - name: <스텝명>
                            uses: <액션명>

                          - name: <스텝명>
                            run: <실행할 명령어>           
                    ```
                - on
                    - [워크플로를 트리거하는 이벤트 목록](https://docs.github.com/ko/actions/using-workflows/events-that-trigger-workflows){:target="_blank"}
                    - 트리거로 설정할 이벤트를 정한다.
                - jobs > MyJobID > runs-on
                    - [Choosing GitHub-hosted runners 목록](https://docs.github.com/ko/actions/using-workflows/workflow-syntax-for-github-actions#choosing-github-hosted-runners){:target="_blank"}
                    - CI를 돌릴 머신의 OS 이미지를 선택한다고 생각하면 된다.
                - jobs > MyJobID > steps > uses
                    - 미리 생성된 액션 환경을 이용하는 부분이다.
                    - 예를들어 node 환경을 구축하려고 한다면  
                        ```yml
                            steps:
                            - uses: actions/setup-node@v3
                              with:
                                node-version: 16
                        ```    
                        을 입력하면 된다.
                    - 미리 생성된 액션은 마켓플레이스에서 검색하여 찾을 수 있다.  
                      [마켓플레이스](https://github.com/marketplace?query=setup+node+){:target="_blank"}  
                      [Setup Node.js environment](https://github.com/marketplace/actions/setup-node-js-environment){:target="_blank"}
                - jobs > MyJobID > steps > run
                    - 쉘로 실행하고자하는 명령어를 입력한다.
        - CI 용 git actions 스크립트 작성
            - 이름
                - [github-actions.yml](https://github.com/a3magic3pocket/simple-web-example/blob/main/.github/workflows/github-actions.yml){:target="_blank"}
            - 설명
                - 개발자가 local 브랜치에서 remote repository 브랜치로 커밋을 푸시한 경우, 새로 업데이트된 소스소드로 빌드한 후 테스트를 진행한다.
            - 본문
                - ```yml
                    # 워크 플로 이름
                    name: simple web example CI

                    # push 이벤트를 트리거로 사용
                    # (이 repository의 모든 브랜치에 push 이벤트가 발생할 경우 jobs 실행)
                    on: [push]

                    # 작업
                    jobs:
                      # 작업ID
                      test:
                        # ubuntu 최신 버전을 CI 머신으로 활용
                        runs-on: ubuntu-latest

                        # 각 단계 설정
                        steps:
                          # github에 있는 소스코드를 CI 머신으로 복사하기
                          - name: Checkout
                            uses: actions/checkout@v2

                          # node 환경 구축
                          - name: Setup Node
                            uses: actions/setup-node@v2
                            with:
                              node-version: "16.13.0"

                          # node 의존성 패키지 설치
                          - name: Install dependencies
                            run: npm install

                          # 테스트 시작
                          - name: Test
                            run: npm run test

                          # 빌드 시작
                          - name: Build Test
                            run: npm run build

                          # Dockerhub에 같은 태그를 가진 이미지가 존재하는지 확인
                          - name: Image Tag Test
                            run: chmod 775 ./extract-image-tag_linux_amd64 && ./extract-image-tag_linux_amd64 -f simple-web.yml -tu https://hub.docker.com/v2/namespaces/a3magic3pocket/repositories/simple-web/tags    
                    ```
        - CD용 git actions스크립트 작성
            - 이름
                - [github-build-image.yml](https://github.com/a3magic3pocket/simple-web-example/blob/main/.github/workflows/github-build-image.yml){:target="_blank"}
            - 설명
                - main 브랜치에서 CI워크플로를 성공적으로 통과했다면  
                  Dockerhub repostiory로 빌드한 이미지를 push 한다.
            - remote main 브랜치에 개발자가 직접 푸시 방지
                - github repository로 이동.   
                    ex) [simple-web-example](https://github.com/a3magic3pocket/simple-web-example){:target="_blank"}
                - (상단 탭)Settings 클릭    
                  <a href="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/00-settings.png" target="_blank"><img src="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/00-settings.png" width="100%"></a> 
                - (좌측 사이드메뉴) Branches 클릭
                - Branch protection rules 오른쪽에 위치한 'Add rule' 클릭  
                  <a href="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/01-branches-and-rule.png" target="_blank"><img src="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/01-branches-and-rule.png" width="100%"></a> 
                - Brach name pattern*에 'main' 입력
                - Require status checks to pass before merging 클릭  
                  (main 브랜치에 머지 전 이상이 없는지 아래 항목을 체크하겠다는 뜻)
                    - Require branches to be up to date before merging 클릭  
                      pull request에 새로 추가된 항목이 있어야 main 브랜치에 merge가능
                    - pull request이 새로 추가되었다면  
                      문제가 없는지 Status check를 해야함
                    - Status checks에 사용할 git action의 job을 선택해야 함
                    - github-actions.yml에 있는 job ID "test"를 입력 후 선택
                - Do not allow bypassing the above settings 클릭    
                    <a href="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/03-rule-detail.png" target="_blank"><img src="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/03-rule-detail.png" width="100%"></a> 
                - Save Changes 클릭하여 저장
            - Dockerhub repository 생성
                - Dockerhub에도 미리 image를 받을 repository를 만들어두어야 한다.
                - ex) [a3magic3pocket/simple-web dockerhub](https://hub.docker.com/r/a3magic3pocket/simple-web){:target="_blank"}
            - Dockerhub auth token 발급
                - [Create an access token](https://docs.docker.com/docker-hub/access-tokens/#create-an-access-token){:target="_blank"}
                - github에서 Dockerhub 인증권한을 얻을 수 있도록  
                  Dockerhub access token을 생성한다.
            - github repostiory의 secrets에 Dockerhub access token 등록
                - github repository로 이동.  
                  ex) [simple-web-example](https://github.com/a3magic3pocket/simple-web-example){:target="_blank"}
                - (상단 탭) Settings 클릭   
                    <a href="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/00-settings.png" target="_blank"><img src="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/00-settings.png" width="100%"></a> 
                - (좌측 사이드메뉴) Environments 클릭  
                    <a href="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/04-rule-environment.png" target="_blank"><img src="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/04-rule-environment.png" width="100%"></a> 
                - "New environment"를 누르고 dockerhub을 등록
                - 생성된 "dockerhub" environment를 클릭
                - Add secret 클릭하고  
                  DOCKERHUB_AUTH_TOKEN, DOCKERHUB_USERNAME을 각각 등록    
                    <a href="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/05-add-secret.png" target="_blank"><img src="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/05-add-secret.png" width="100%"></a> 
            - 본문
              ```yml
                # 워크플로 이름
                name: Build docker image
              
              
                # 트리거
                on:
                  # "simple web example CI" 워크플로가 main 브랜치에서 정상적으로 처리되었다면
                  workflow_run:
                    workflows: ["simple web example CI"]
                    branches: [main]
                    types: 
                      - completed
              
              
                # 작업
                jobs:
                  # 작업ID
                  push_to_registry:
                    # 환경변수(docker access token을 불러오기 위함)
                    environment: dockerhub
              
                    # 작업명
                    name: Push Docker image to Docker Hub
              
                    # 머신 OS 이미지 선택
                    runs-on: ubuntu-latest
              
                    # 워크플로 이벤트가 pull_request이고 워크플로가 성공했다면 아래 스텝 진행
                    if: >
                      $\{{ github.event.workflow_run.event == 'pull_request' &&
                      github.event.workflow_run.conclusion == 'success' }}
              
                    # 스탭들
                    steps:
                      # github에 있는 소스코드를 CI머신으로 복사
                      - name: Check out the repo
                        uses: actions/checkout@v2
                      
                      # dockerfile 빌드
                      - uses: docker/setup-buildx-action@v1
              
                      # 도커 로그인 
                      - name: Log in to Docker Hub
                        uses: docker/login-action@v1
                        with:
                          username: ${{ secrets.DOCKERHUB_USERNAME }}
                          password: ${{ secrets.DOCKERHUB_AUTH_TOKEN }}
              
                      # 태그 추출
                      - run: chmod 775 ./extract-image-tag_linux_amd64 
                      - run: echo "TAG=`./extract-image-tag_linux_amd64 -f simple-web.yml -tu https://hub.docker.com/v2/namespaces/a3magic3pocket/repositories/simple-web/tags`" >> $GITHUB_ENV
              
                      # 추출한 태그를 이용하여 만든 도커이미지를 dockerhub으로 푸시
                      - name: Build and push Docker image
                        uses: docker/build-push-action@v2
                        with:
                          context: .
                          push: true
                          tags: a3magic3pocket/simple-web:${{ env.TAG }}
                 ```
        - 서비스 서버에 Webhook 설치
            - 가정
                - 서비스 서버에서 docker container 기반으로 서비스를 배포하고 있다고 가정한다.
            - Webhook의 용도
                - [adnanh/webhook](https://github.com/adnanh/webhook){:target="_blank"}
                - webhook은 역방향 API라고 생각하면 된다.  
                  무슨일이 있을 때 서버가 클라이언트에게 지정된 URL로 요청을 보낸다.
                - Webhook(adnanh/webhook)은   
                  다른 서버의 webhook 요청을 우리 서버에서 받아 
                  처리할 수 있게 하는 작은 웹서버이다.
                - Webhook을 세팅하고 Dockerhub에 webhook URL을 입력하면
                  Dokcerhub에서 새 이미지가 push 되었을 때 해당 URL로  
                  HTTP요청을 보내게 된다.
                - 배포 스크립트는 사용자가 미리 작성해두어야 한다.
                - Webhook이 Dockerhub로부터 요청을 받으면  
                  지정된 배포 스크립트를 실행하여 배포하게 된다.
            - Webhook 세팅
                - [github webhook을 이용한 원격서버 배포 자동화(AWS)](https://a3magic3pocket.github.io/posts/github-webhook/){:target="_blank"}
                - 위와 같이 설정한다.  
                  단, webhook을 보내는 서버가 github이 아닌 dockerhub이므로  
                  본문에서 '[adnanh / webhook](https://a3magic3pocket.github.io/posts/github-webhook/#adnanh--webhook){:target="_blank"}' 부분만 진행하고  
                  '[백그라운드 실행](https://a3magic3pocket.github.io/posts/github-webhook/#%EB%B0%B1%EA%B7%B8%EB%9D%BC%EC%9A%B4%EB%93%9C-%EC%8B%A4%ED%96%89){:target="_blank"}'를 진행하여  
                  Webhook을 실행하고 webhook URL을 얻으면 된다.
            - Webhook URL을 domain으로 접속되게 만들기
                - 안타깝게도 Dockerhub에서는 IP 형태의 URL를 webhook URL로   
                  입력할 수 없다.
                - webhook URL을 domain 형태로 바꿔야 한다.
                - 1. 직접 DNS에 webhook URL 등록하기
                    - Webhook의 디폴트 포트는  9000번이다.
                    - 클라우드 환경이라면 보안그룹에서 9000 포트를  
                      모든 아이피(0.0.0.0/0)에서 접근 할 수 있도록 개방한다.
                    - 서비스에서 사용하고 있는 도메인의 DNS에  
                      9000 포트로 접근할 수 있도록 A record를 추가한다.  
                      ex) webhook.my-domain.com -> 124.121.12.1:9000
                - 2. 서비스 서버의 리버스 프록시에 webhook URL 등록하기  
                    - 만약 서비스 서버에서 nginx와 같은 웹서버를  
                      리버스 프록시로 사용 중이라면  
                      webhook URL을 프록시로 넘겨줄 수 있다.
                    - 이 경우 nginx 포트로 들어온 요청을 넘기는 것이기 떄문에  
                      80, 433 포트로 요청이 들어오게 된다.  
                      그래서 클라우드의 보안그룹을 수정할 필요가 없다.
                    - server_name으로 요청을 구분할 경우  
                        ex) webhook.my-domain.com -> localhost:9000
                    - path로 요청을 구분할 경우  
                        ex) www.my-domain.com/webhook -> localhost:9000
        - Dockerhub에서 webhook URL 등록  
            - 방법
                - Dockerhub에 로그인 후 CI / CD로 관리되는 repository로 이동한다.  
                    ex) [a3magic3pocket/simple-web - Docker Image | Docker Hub](https://hub.docker.com/r/a3magic3pocket/simple-web){:target="_blank"}
                - 'Manage Repository'를 클릭한다.    
                    <a href="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/06-manage-repository.png" target="_blank"><img src="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/06-manage-repository.png" width="100%"></a> 
                - 'Webhooks'를 클릭
                - Webhook name을 적고(아무거나 적어도 된다)  
                    Webhook URL을 입력하고 Create를 누른다.  
                    <a href="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/07-webhooks.png" target="_blank"><img src="/assets/img/2023-02-05-cicd-using-git-action-and-dockerhub/07-webhooks.png" width="100%"></a> 
    - 상업용으로 CI / CD 구축 시 고려사항
        - 보안을 고려하여 Docker registry 서버는 AWS ECR을 사용하거나  
            서버를 따로 만들어 private docker registry를 운영하는 것이 좋을 것 같다.
        - git remote repository로의 의존성을 분리시키고 싶은 경우  
            jenkins에서 CI / CD를 구축하는 방법도 고려할 수 있다.
        - 다만 jenkins는 jenkins 서버를 따로 두어야하므로   
            관리 포인트가 더 늘어난다.  
            회사 규모에 따라 ISMS 심사 같은 것을 대비해야 할 경우  
            고려해 볼 수 있다.
        - CD도 webhook 외에 다양한 선택지가 있다.
            - jenkins를 쓰는 경우, ssh로 jenkins가 접속하여 직접 스크립트를 실행하는 방법
            - AWS를 쓰는 경우, Code deploy 등을 이용하여 배포하는 방법
            - 등등
        - 회사 상황에 맞게 적절한 선택을 하면 좋을 것 같다.
    - 참고
        - [GitHub Actions 이해 - GitHub Docs](https://docs.github.com/ko/actions/learn-github-actions/understanding-github-actions){:target="_blank"}
        - [워크플로를 트리거하는 이벤트 목록](https://docs.github.com/ko/actions/using-workflows/events-that-trigger-workflows){:target="_blank"}
        - [Choosing GitHub-hosted runners 목록](https://docs.github.com/ko/actions/using-workflows/workflow-syntax-for-github-actions#choosing-github-hosted-runners){:target="_blank"}
        - [CI/CD(CI CD, 지속적 통합/지속적 배포): 개념, 툴, 구축, 차이 (redhat.com)](https://www.redhat.com/ko/topics/devops/what-is-ci-cd){:target="_blank"}
        - [CI/CD 파이프라인: 개념, 방법, 장점, 구현 과정 (redhat.com)](https://www.redhat.com/ko/topics/devops/what-cicd-pipeline){:target="_blank"}
        - [adnanh/webhook](https://github.com/adnanh/webhook){:target="_blank"}
        - [마켓플레이스](https://github.com/marketplace?query=setup+node+){:target="_blank"}
        - [Setup Node.js environment](https://github.com/marketplace/actions/setup-node-js-environment){:target="_blank"}
