---
title: Jekyll-theme-Chirpy의 안전한 배포
date: 2020-10-23 17:35:00 +0900
categories: [Github blog]
tags: [github blog, jekyll, deployment]    # TAG names should always be lowercase
---
## 개요
- 첫 post를 작성할 때 Front Matter 부분에 categories를 잘못 입력하여 카테고리 링크에 문제가 생겼다.
- 처음에는 원인을 몰라서 계속 헤매다가 Jekyll-theme-Chirpy에 CI/CD가 붙어있다는 것을 알게 되었다. 
- 자세히 보니 내 commits은 모두 테스트를 통과하지 못하고 있었다.
- 이 부분을 자세히 알기 위해 Chirpy의 README.md를 자세히 읽어보니,   
    Github page에 배포하는 경우 따라야하는 안전한 배포 방법이 기술되어 있었다.  
- 이 부분을 참조하면 나와 같이 오류가 있는 웹 사이트 배포를 하는 실수를 하지 않을 수 있다.

## CI / CD
- 개요
    - CI / CD를 명료하게 알지 못하기에 RedHat 사이트에서 내리고 있는 정의를 참고했다. 
    - 이 post는 CI / CD를 설명하기 위한 post가 아님으로 간단한 설명만 하고 넘어간다. 
    - 자세한 사항은 [RedHat에 게시된 CI/CD(지속적 통합/지속적 제공): 개념, 방법, 장점, 구현 과정](https://www.redhat.com/ko/topics/devops/what-is-ci-cd){:target="_blank"}을 참고하라. 
- CI(Countinuous Integration)
    - 어떤 production을 여러 개발자가 개발하여 버전 관리를 할 경우,  
        다른 개발자의 변경 사항이 의도하지 않은 에러를 만들 수 있다.  
    - 따라서 모든 버전이 반드시 통과해야하는 테스트를 두어 버전의 안정성을 높일 필요가 생겼다. 
    - 개발자들은 이를 자동화하기 위해 버전관리툴에 새 버전이 올라갈 경우(예를 들면 github에 commit을 push),  
        자동으로 테스트 코드가 동작하는 파이프라인이 만들게 된다.  
    - 이게 CI
    - 나는 Chirpy forked repository에 commits을 push 할 때 이 부분에서 통과하지 못하여 CD까지 가지도 못했다.
- CD(Countinuous Delivery)
    - CI를 통과하면 코드가 자동으로 릴리스되고 프로덕션에 적합한 빌드 파일을 만들게 된다.
    - 이게 CD
- 또 다른 CD(Countinuous Deployment)
    - 빌드 파일이 생성되면 실제 프로덕션으로 릴리스 하는 과정을 자동화한 것을 말한다.
    - 이게 또 다른 CD

## Chirpy 빌드 시 일어나는 일
- Jekyll 빌드
    - 빌드를 하게 되면 _site 디렉토리가 생성되게 된다.
    - _site 디렉토리를 확인하고 싶다면 로컬 서버에서 bash tools/build 명령을 실행하면 된다.
    - _site는 호스팅 서버에서 실제로 참조하는 정적 파일들이 저장된다.
    - 우리가 로컬에서 commits 후 github에 push하게 되면 github이 자동으로 Jekyll build를 실행한다.  
    - 그리고 _site를 호스팅하게 되는 것이다.
- Chirpy 빌드
    - Chirpy도 Jekyll 테마이기 때문에 위와 같은 과정을 거친다.
    - 그러나 안정성을 고려하여 Github default로 호스팅하는 master branch 대신 
      gh-page branch를 호스팅하라고 권고한다.
    - 블로그 주인이 fork 뜬 repository에 commits을 push 하면 자동으로 CI/CD가 진행된다.  
    - CI/CD가 정상적으로 완료되면 gh-page라는 branch가 새로 생성된다.
    - 이 branch는 테스트를 모두 통과한 코드이므로 안전하다.
    - 따라서 master branch 대신 gh-page branch로 호스팅하는 것이 안전한 것이다.

## gh-page branch 호스팅
- 설명
    - [Chirpy github repository README.md](https://github.com/cotes2020/jekyll-theme-chirpy/tree/master){:target="_blank"}의 DEPLOYMENT 섹션에 잘 나와있다.
    - 더 자세한 사항은 위의 README.md를 참고하면 된다.
- 실행방법
    - _config.yml을 열고 url이 'https://username.github.io' 형태로 등록되었는지 확인한다(개인 계정만 해당)
    - fork 뜬 repository로 가서 settings를 클릭한다.  
      <a href="/assets/img/2020-10-23-chirpy-safe-deployment/00-forked-repo-settings.jpg" target="_blank"><img src="/assets/img/2020-10-23-chirpy-safe-deployment/00-forked-repo-settings.jpg" width="80%" height="80%"></a> 
    - 왼쪽 사이드 바에 options를 선택하고(settings 첫 화면이 options이다.)  
      스크롤을 쭉 내리면 Github Pages가 나온다.
    - gh-page branch를 선택하고 Save를 눌러 저장한다.  
      <a href="/assets/img/2020-10-23-chirpy-safe-deployment/01-save-settings.jpg" target="_blank"><img src="/assets/img/2020-10-23-chirpy-safe-deployment/01-save-settings.jpg" width="80%" height="80%"></a>  

## 참고
- [jekyll-theme-chirpy git repository](https://github.com/cotes2020/jekyll-theme-chirpy/tree/master)
- [RedHat에 게시된 CI/CD(지속적 통합/지속적 제공): 개념, 방법, 장점, 구현 과정](https://www.redhat.com/ko/topics/devops/what-is-ci-cd){:target="_blank"}
