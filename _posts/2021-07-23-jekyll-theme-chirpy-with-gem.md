---
title: gem 기반으로 jekyll 블로그 만들기(chirpy 테마, windows)
date: 2021-07-23 19:19:00 +0900
categories: [Github blog]
tags: [github blog, jekyll, deployment]    # TAG names should always be lowercase
---

## 개요
- 기존의 fork 방식으로 chirpy 테마 jekyll 블로그를 운영하다가 여러 문제가 발생했다.
- [html-proofer HTML parsing error](https://a3magic3pocket.github.io/posts/html-parser-error/){:target="\_blank"}가 발생하더니,  
  이제는 아예 배포 후 테스트를 통과하지 못한다.
- 이와 같은 문제는 버전업만으로도 쉽게 해결된다는 것을 알아내었다.
- 그래서 원본 repo를 로컬에 등록한 후 pull을 했더니  
  삭제한 예제를 포함하여 190개에 달하는 변경사항과 50여개의 conflict가 발생하였다.
- 이를 해결하기 위해 gem 기반으로 블로그를 다시 구축하였다.

## 환경
- os
    - 윈도우 10 pro
- ruby
    - 3.0.0

## gem 방식 jekyll 블로그 구축의 원리
- 먼저 ruby와 jekyll, bundle 등 jekyll 블로그를 만드는 환경을 로컬에 구축한다.
- 그리고 jeykll 라이브러리를 통해 임의의 jekyll 블로그를 만든다.
- 이제 Gemfile에 정해진 테마를 입력하고 bundle install을 하면   
    시스템 ruby 하위에 해당 테마가 설치된다.
- 내가 만든 jekyll 블로그를 빌드 후 배포해보면 자동으로 해당 테마가 입혀져 나오게 된다.
- chirpy 테마는 매 업데이트마다 변경되는 크리티컬 디렉토리 및 파일이 존재한다.
- 최신 버전의 크리티컬 디렉토리를 내가 만든 jekyll 블로그에 덮어쓰기 하면  
    내 블로그는 포스트와 상관없이 최신버전으로 유지시킬 수 있다.  
    (일부 파일은 커스텀 설정이 들어가서 직접 수정해야한다.)  

## 실행
- 루비 인스톨러 설치
    - 윈도우이기에 ruby installer를 이용하여 설치한다.
        - [ruby install 설치 링크](https://rubyinstaller.org/downloads/){:target="\_blank"}
    - 2021.07.23 기준 최신 버전 3.0.0을 설치하였다.
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/00-ruby-installer.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/00-ruby-installer.jpg" width="70%"></a> 
- 루비 설치
    - 인스톨러를 다운로드 받은 후 실행한다.
    - jekyll을 사용할 때 선택항목을 모두 사용하기 때문에 둘 다 다운로드 한다.
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/01-ruby-installer-additional.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/01-ruby-installer-additional.jpg" width="70%"></a> 
- jekyll, bundle 설치
    - [jekyll 공식문서 빠른 시작](https://jekyllrb-ko.github.io/docs/){:target="\_blank"}란을 보고 그대로 하면 된다.
    - ruby prompt 실행
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/02-search.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/02-search.jpg" width="70%"></a> 
    - ruby prompt에서 원하는 위치로 간다.
        - ```bash
            cd [원하는 경로]
            ```
    - jekyll과 bundle 설치
        - ```bash
            # 인스톨러에서 선택항목 설치 안 했으면 에러남
            gem install jekyll bundler

            # 루비 3.0.0은 webrick을 설치해줘야 에러 안 남
            gem install webrick
            ```
    - 블로그 생성
        - ```bash
            # 명령어의 myBlog가 블로그 디렉토리명이다.
            # [github ID].github.io로 하면 remote repo와 이름을 맞출 수 있다.
            jekyll new myBlog
            ```
    - 이동 후 실행
        - ```bash
            # 이동 후 실행
            cd myBlog 
            bundle exec jekyll serve
            ```
    - 확인
        - 웹 브라우저를 켜고 주소 입력 창에 localhost:4000을 입력하여 접속한다.
        - 에러가 뜨지 않고 페이지가 나온다면 성공
- chirpy 테마 입히기
    - 이제부터 [chirpy README.md](https://github.com/cotes2020/jekyll-theme-chirpy){:target="\_blank"}에 나온대로만 하면 잘 작동한다.  
    - myBlog/Gemfile을 텍스트 에디터로 열고 아래 문구를 추가한다.
        - ```bash
            gem "jekyll-theme-chirpy"
            ```
    - myBlog/_config.yml을 열고 theme: 항목에 아래와 같이 적는다.
        - ```bash
            theme: jekyll-theme-chirpy
            ```
    - bundle을 실행하여 theme를 다운로드한다.
        - ```bash
            bundle
            ```
    - 다운로드된 테마의 위치를 알아내고 거기로 이동
        - ```bash
            # 실행하면 jekyll-theme-chirpy가 설치된 경로가 나옴
            bundle info --path jekyll-theme-chirpy

            # jekyll-theme-chirpy 경로로 이동
            cd [jekyll-theme-chirpy 경로]
            ```
    - conflict 요소가 있는 파일 제거
        - 최상단에 위치한 index.markdown과 about.markdown 파일을 제거한다.
- 크리티컬 디렉토리 알아내기
    - [chirpy starter](https://github.com/cotes2020/chirpy-starter){:target="\_blank"} repo로 가보면 크리티컬 디렉토리가 명시되어있다. 
    - 현재 기준으로는 아래와 같다.
        - ```
            ├── _data
            ├── _plugins
            ├── _tabs
            ├── _config.yml
            └──  index.html
            ``` 
    - 위의 jekyll-theme-chirpy에서 해당 디렉토리 및 파일을 복사하여  
      myBlog 디렉토리로 덮어쓰기 한다.
- _config.yml
    - 이제 자신에게 맞게 _config.yml을 수정한다.
    - _config.yml의 주석과 [chirpy demo](https://chirpy.cotes.info/){:target="\_blank"}의 내용을 참고하면 쉽게 수정할 수 있다.
    - _config.yml을 잘못 수정하면 빌드에 실패하니 잘 확인하고 수정한다.
- 컨텐츠 작성
    - _posts 하위에 마크다운 문서를 작성한다.
    - 이미지는 assets/img 하위에 저장한다.
    - 자세한 내용은 [chirpy demo](https://chirpy.cotes.info/){:target="\_blank"}에 나와있다.
- remote repo 생성 및 연결
    - github에 접속한 뒤 \[github ID].github.io를 이름으로 하여 새 repo를 생성한다.
    - 생성 시 README.md를 선택하여 자동으로 remote repo에 브런치가 생성되도록 한다.
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/03-create-remote-repo.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/03-create-remote-repo.jpg" width="70%"></a> 
    - 기본 브런치 명은 main일 것이다.
    - ssh 인증 remote repo 주소를 복사한다  
      (ssh 설정이 완료되었다고 가정한다, 안 되었다면 [링크 참고](https://a3magic3pocket.github.io/posts/set-git-config-for-mutiple-ids/){:target="\_blank"}).  
      - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/04-copy-ssh-address.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/04-copy-ssh-address.jpg" width="100%"></a> 
    - 다시 로컬에서 myBlog로 이동한 다음 아래와 같이 git 명령을 실행한다.
        - ```bash
            # git DB 생성
            git init

            # remote repo와 연결, remote repo 닉네임은 origin
            git remote add origin [ssh 인증 remote repo 주소]

            # local repo를 remote repo와 동기화
            git pull origin main

            # 지금까지 변경사항을 모두 commit하고 origin에 push
            git add .
            git commit -m "initial commit"
            git push origin main
            ```
    - remote repo로 가서 확인해보면 빌드에 실패했을 것이다.
    - 배포 설정이 안되어서 그렇다.
- 배포
    - 가장 중요한 배포다.
    - 너무 중요해서 다음 항목에서 자세히 다룬다.

## 배포 원리
- 일반적인 jekyll의 블로그 내용은 _site 디렉토리 하위에 위치
    - bundle exec jekyll serve 명령으로 빌드 및 배포를 하게 되면  
      자동으로 _site 디렉토리가 생기는 걸 확인할 수 있다.
    - 이 _site 디렉토리가 jekyll 빌드의 결과물로 실질적인 정적 사이트 파일이다.
    - 그런데 .gitignore 파일을 확인해보면 _site 디렉토리가 있다.
    - 즉 git에 올라가지 않는다.
    - github은 jekyll 블로그의 경우 알아서 빌드해서 _site를 생성하고 배포하기 때문이다.
- github에서는 gh-pages라는 브랜치로 jekyll 블로그를 배포해준다.
    - 따라서 보통의 jekyll 블로그는 gh-pages와 관련된 라이브러리를 gem으로 설치한다.
- chirpy는 ci / cd가 구축되어 있어 해당 로직이 돌아야 배포 가능
    - chirpy의 기본 설정은 ci / cd를 통과해야 자동으로 gh-pages를 생성해준다.  
      (안하게 하는 옵션도 있었으나 fork 방법에 적혀있었기에 고려하지 않는다.)
    - 그래서 지정된 test를 모두 통과해야 최종 배포 본을 gh-pages 브런치에 생성한다.
    - ci / cd는 리눅스 인스턴스에서 진행된다.  
      따라서 myBlog에 이 부분의 세팅을 추가해야한다.

## 진행
- pages-deploy.yml 확보
    - 위에서 gem으로 다운로드 한 jekyll-theme-chirpy 디렉토리에서  
      ./github/worflows/pages-deploy.yml.hook 파일을 복사하여  
      myBlog/github/worflows/pages-deploy.yml로 붙여넣기 한다.
    - !!!\[중요]이때 파일명에서 .hook을 제거해주는 것이 매우 중요하다!!!
    - 중간 중간 없는 디렉토리는 새로 만들어준다.
- pages-deploy.yml 수정
    - pages-deploy.yml을 열어보면 아래와 같다.
        - ```yml
            name: 'Automatic build'
            on:
            push:
                branches:
                - master
            ...
            ```
        - 우리의 remote repo의 default 브런치는 main이므로  
          master를 main으로 바꿔준다.
- tools/test.sh, tools/deploy.sh 확보
    - 마찬가지로 jekyll-them-chirpy 디렉토리에서  
      tools/test.sh, tools/deploy.sh를 복사하여  
      myBlog에 붙여넣기한다.
    - 중간 중간 없는 디렉토리는 새로 만들어준다.

## 테스트 리눅스 환경에 필요한 라이브러리 설치
- 번들 락에 linux 플랫폼 추가
    - ```bash
        bundle lock --add-platform x86_64-linux
        ```
    - 미설치 시 에러
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/05-error-0.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/05-error-0.jpg" width="100%"></a> 
- htmlproofer 설치
    - myBlog/Gemfile을 열어 아래 명령을 추가한다.
        - ```
            group :test do
                gem "html-proofer", "~> 3.18"
            end
            ```
    - 그리고 설치
        - ```bash
            bundle install
            ```
    - 미설치 시 에러
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/06-error-1.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/06-error-1.jpg" width="70%"></a> 
- 하고 싶은 말
    - 나는 에러를 보고 없는 라이브러리를 설치하는 방법으로 처리하였지만  
      애초에 jekyll-theme-chirpy/Gemfile 항목을 복사한 뒤   
      myBlog/Gemfile에 추가로 붙여넣기 하고 bundle install 했으면  
      에러 안 났을거 같다.

## 배포 확인
- 이제 아무 commit이나 생성하고 remote repo에 push하면 배포가 되어야한다.
    - ```bash
        git add . 
        git commit -m "second commit for testing deployment"
        git push oriign main
        ```
- github workflow 확인
    - remote repo 메인 화면으로 접속 뒤 아래의 아이콘을 클릭한다.
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/07-workflow-icon.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/07-workflow-icon.jpg" width="100%"></a> 
    - 아마 최초 업로드 시에는 gh-pages 브런치가 없기 때문에 무조건 x 표시일 것이다.
- build failed
    - 최초 업로드 시에는 gh-pages가 없기 때문에 실패라고 뜰 것이다.
    - 해당 항목을 눌러보면 jekyll-theme-chirpy는 등록된 블로그가 아니라면서 에러가 나온다.
    - CI/CD만 성공하면 이런 에러를 볼 일은 없다.
    - 중요한 것은 아래 사진과 같이 왼쪽에 continuous-delivery가 실행되어야 한다.
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/08-workflow-detail.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/08-workflow-detail.jpg" width="100%"></a> 
    - continuous-delivery가 생기지 않는다면 ./github/worflows/pages-deploy.yml 파일이 없는 것이다.  
    - continuous-delivery가 생겼고 실행 중에 에러가 난다면  
      tools/test.sh, tools/deploy.sh에 문제가 있거나  
      테스트 리눅스 환경에 필요한 라이브러리 설치가 안 된 것이다.  
      (정확히는 테스트 환경에서도 작동하도록 Gemfile이 잘 구성되어야 한다.)
- build success
    - ci/cd가 통과하면 gh-pages 브런치가 생겼을 것이다.
    - remote repo의 메인 페이지로 와서 브런치를 눌러보면 gh-pages가 생긴것을 확인할 수 있다.
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/09-new-branch.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/09-new-branch.jpg" width="100%"></a> 
- gihub이 gh-pages로 publish 하도록 설정
    - remote repo 메인의 상단 메뉴에서 settings를 누른다.
    - 왼쪽 사이드 메뉴에서 pages를 클릭한다.
        - <a href="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/10-pages.jpg" target="_blank"><img src="/assets/img/2021-07-23-jekyll-theme-chirpy-with-gem/10-pages.jpg" width="100%"></a> 
    - Source 에서 Branch를 눌러 main 대신 gh-pages를 선택하고 save를 누른다.
    - 새 브라우저를 켜고 주소입력창에 [github ID].github.io를 입력하고 접속해본다.
    - 캐시가 있을 수도 있으니 새로고침을 오래 눌러 캐시를 삭제하고 열어본다.

## \[보너스] 새 버전 업데이트
- Gemfile에 jekyll-theme-chirpy의 원하는 버전을 정확히 적는다.
    - 예시
        - ```
            gem "jekyll-theme-chirpy", "~> 4.0", ">= 4.0.1"
            ```
- 아래 명령 실행
    - ```bash
        bundle update jekyll-theme-chirpy
        ```
- 최선 jekyll-theme-chirpy를 확보했으면 위에서처럼 크리티컬 디렉토리를  
  myBlog로 복사해서 적용하면 된다.

## 참고
-  [chirpy README.md](https://github.com/cotes2020/jekyll-theme-chirpy){:target="\_blank"}
- [ruby install 설치 링크](https://rubyinstaller.org/downloads/){:target="\_blank"}
- [jekyll 공식문서 빠른 시작](https://jekyllrb-ko.github.io/docs/){:target="\_blank"}
- [chirpy starter](https://github.com/cotes2020/chirpy-starter)
- [Bundle only supports platforms x64-mingw32 but local is x86_64-linux](https://stackoverflow.com/questions/42730023/bundle-only-supports-platforms-x64-mingw32-but-local-is-x86-64-linux){:target="\_blank"}
