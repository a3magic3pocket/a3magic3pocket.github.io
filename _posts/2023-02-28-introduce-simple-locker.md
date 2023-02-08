---
title: SIMPLE-LOCKER 간단한 설명
date: 2023-02-08 23:01:00 +0900
categories: [prototype]
tags: [prototype] # TAG names should always be lowercase
---
## 소개
- CI / CD 파이프라인을 구축할때 사용하기 위해 만든  
    간단한 웹 애플리케이션이다.
- 자신만의 '구역'을 생성하고  
    '구역'에 코인로커를 생성 및 할당, 삭제하는 간단한 앱이다.

## 이용방법 예시
- 시작하기
    - 메인
        - <a href="/assets/img/2023-02-28-introduce-simple-locker/00-main.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/00-main.png" width="100%"></a> 
        - 메인에 접속한다
    - 회원가입 및 로그인
        - <a href="/assets/img/2023-02-28-introduce-simple-locker/01-signup.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/01-signup.png" width="100%"></a>
        - <a href="/assets/img/2023-02-28-introduce-simple-locker/02-login.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/02-login.png" width="100%"></a>
    - 회원가입 및 로그인을 한다.
        - 시작하기를 누른다.
            - <a href="/assets/img/2023-02-28-introduce-simple-locker/03-start.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/03-start.png" width="100%"></a>
    - 구역 추가
        - 새 구역에 로커 추가를 누른다.
            - <a href="/assets/img/2023-02-28-introduce-simple-locker/04-create-new-sector.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/04-create-new-sector.png" width="100%"></a>
        - 새 구역에 로커를 추가한다.  
            (강남역 구역 추가)
            - <a href="/assets/img/2023-02-28-introduce-simple-locker/05-create-gangnam-sector.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/05-create-gangnam-sector.png" width="100%"></a> 
        - 새 구역에 또 다른 로커를 추가한다.  
            (사당역 구역 추가)
            - <a href="/assets/img/2023-02-28-introduce-simple-locker/06-create-sadang-sector.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/06-create-sadang-sector.png" width="100%"></a> 
    - 로커 추가
        - 강남역 구역에서 '+' 버튼을 3번 클릭하여 3개의 로커를 추가한다.
            - <a href="/assets/img/2023-02-28-introduce-simple-locker/07-create-boxes.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/07-create-boxes.png" width="100%"></a>
    - 로커 구역 변경
        - 24, 25번 로커를 한 번씩 클릭한다.
            - <a href="/assets/img/2023-02-28-introduce-simple-locker/08-select-boxes.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/08-select-boxes.png" width="100%"></a>
        - 구역 변경을 클릭하고 사당역을 클릭한다.
            - <a href="/assets/img/2023-02-28-introduce-simple-locker/09-change-sector.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/09-change-sector.png" width="100%"></a>
    - 로커 삭제
        - 사당역으로 옮겨진 25번 로커를 클릭하고  
            삭제를 클릭한다.
            - <a href="/assets/img/2023-02-28-introduce-simple-locker/10-remove-box.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/10-remove-box.png" width="100%"></a>
## 구조
- 구성도
    - <a href="/assets/img/2023-02-28-introduce-simple-locker/11-structure.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/11-structure.png" width="100%"></a>
- API
    - [simple-api-example](https://github.com/a3magic3pocket/simple-api-example){:target="_blank"}
    - 개발에 사용한 도구
        - Golang gin framework
- Front
    - [simpe-web-example](https://github.com/a3magic3pocket/simple-web-example){:target="_blank"}
    - 개발에 사용한 도구
        - next.js react.js, redux.js
- Webhook
    - [adnanh/webhook: webhook is a lightweight incoming webhook server to run shell commands (github.com)](https://github.com/adnanh/webhook){:target="_blank"}
    - 개발에 사용한 도구
        - bash shell script
- Reverse proxy
    - 개발에 사용한 도구
        - nginx
## 인증
- 설명
    - 팝업 방식 카카오 소셜로그인하고 비슷한 구조로 인증을 한다.
- 개발에 사용한 도구
    - [appleboy/gin-jwt: JWT Middleware for Gin framework (github.com)](https://github.com/appleboy/gin-jwt){:target="_blank"}
- 구조
    - 클라이언트에서 API 서버로 로그인 페이지를 요청한다.
        - ex) GET https://api.coinlocker.link/login?redirect-url=coinlocker.link/check
        - 파라미터
            - redirect-url
                - 로그인 성공 시 리다이렉트 할 URL
    - API 서버에서 로그인 페이지를 응답으로 리턴한다.
    - 클라이언트에서 로그인 페이지에 ID와 PASSWORD를 담아  
        API 서버로 인증을 요청한다.
        - 클라이언트에서 로그인 버튼을 누르면  
            ```
            POST   
            https://api.coinlocker.link/loing  
            --data {'UserName": "MyUserName", "Password": "MyPassword"}  
            ```
            요청이 API로 전송된다.
        - 관련 소스
            -  [simple-api-example/login.js at main · a3magic3pocket/simple-api-example (github.com)](https://github.com/a3magic3pocket/simple-api-example/blob/main/public/js/login.js){:target="_blank"}
    - API 서버에서는 UserName과 Password를 가지고 인증을 시작한다.
        - 인증(Authentication) 실패 또는 인가(Authorization) 실패 시  
            적절한 에러코드와 에러메시지를 json에 담아 리턴한다.
        - 로그인 성공 시 인자로 받은 redirect-url로 리다이렉트 한다.  
            이때 클라이언트 브라우저에 인증정보가 담긴 JWT 쿠키가 저장된다.
        - JWT 쿠키는 아래와 같은 속성을 갖는다.  
            <a href="/assets/img/2023-02-28-introduce-simple-locker/12-jwt.png" target="_blank"><img src="/assets/img/2023-02-28-introduce-simple-locker/12-jwt.png" width="100%"></a> 
            - domain
                - api.coinlocker.link
            - Path
                - /
            - HttpOnly
                - true
            - Secure
                - true
        - **주의
            - Secure 속성으로 인하여 https에서만 쿠키가 전달된다.
            - 만약 ssl 인증서가 만료되었다면 로그인을 기능을 사용할 수 없다.
        - 관련 소스
            - [simple-api-example/auth.go at main · a3magic3pocket/simple-api-example (github.com)](https://github.com/a3magic3pocket/simple-api-example/blob/main/middleware/auth.go){:target="_blank"}
    - redirect-url로 리다이렉트되면  
        클라이언트에서는 로그인 성공과 관련된 정보를 로컬 스토리지에 저장한다.
        - JWT가 HttpOnly 이기 때문에 클라이언트에서는   
            javascript로 JWT 쿠키를 조회할 수 없다.
        - 더욱이 [동일 출처 정책](https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy){:target="_blank"}에 의해, 클라이언트 origin과 JWT 쿠키에   
            저장된 domain이 다르기 때문에 클라이언트에서는   
            브라우저에서도 조회 할 수 없다.
            - ex) 클라이언트 origin: www.coinlocker.link 또는 coinlocker.link 
            - ex) 쿠키에 저장된 domain: api.coinlocker.link
        - 때문에 redirect-url로 접속되면 클라이언트에서는 임의로   
            로그인에 성공하였다고 판단하고  
            javascript를 이용하여 클라이언트 cookie에   
            key: front-login, value: success라는 값을 저장한다.
            - 이때 클라이언트 cookie 도메인은   
                .(leading dot)을 포함한  도메인으로 설정한다.   
                (ex. www.conlocker.link -> .coinlocker.link  
                coinlocker.link -> .coinlocker.link)  
            - 사용자가 브라우저에서는 로그인 성공으로 표시되지만  
                JWT 쿠키가 만료된 채   
                인증 요구 페이지로 접속한 경우,  
                API 인증에 실패하여 자동으로 로그인 페이지로   
                리다이렉트 시킨다.  
        - (2023.02.05일 기준) cookie의 onchange 이벤트는   
            모든 브라우저에서 지원하지 않기 때문에   
            이벤트 트리거 용으로 사용하기 위해    
                key: trigger, value: Math.random()을 로컬스토리지에 저장한다.   
            - 이때 redirect-url은 반드시 클라이언트 location.host로 시작해야한다.  
                만약 redirect-url의 domain이 클라이언트 domain과 다르다면  
                로컬스토리지에 저장한 값을 프론트에서 불러올 수 없다.  
                (ex. redirect-url의 domain: www.coinlocker.link    
                클라이언트 domain: coinlocker.link   
                이면 안 됨)
            - [cookies.onChanged](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/cookies/onChanged){:target="_blank"}
            - [Window: storage event](https://developer.mozilla.org/en-US/docs/Web/API/Window/storage_event){:target="_blank"}
    - 인증 만료 상태에서 클라이언트가 로그인이 필요한 페이지로 접속한 경우
        - 사용자가 로그인이 필요한 페이지로 접속하여 API를 요청할 경우  
            API에서 인증실패 오류를 내뱉기 때문에  
            로그인 페이지로 리다이렉트하게 된다.  
    - 클라이언트에서 로그아웃 버튼을 누른 경우  
        - 클라이언트에서 API로 로그아웃을 요청한다.  
        - 요청 결과가 성공하였다면 클라이언트에서   
            front-login 쿠키와 username 쿠키를 삭제한다.
