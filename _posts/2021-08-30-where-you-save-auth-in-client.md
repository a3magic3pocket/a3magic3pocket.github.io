---
title: React 사용 시 JWT를 어디에 저장해야할까?
date: 2021-08-30 15:31:00 +0900
categories: [Web]
tags: [golang, react, js, jwt] # TAG names should always be lowercase
---

## 개요
- 예전에 React를 처음 사용할 때 backend를 jwt 인증방식으로 구현한 뒤  
  frontend에서는 이 jwt를 어디에 저장할지 고민했었다.
- 그때 [4장. JWT 이해 및 적용](https://backend-intro.vlpt.us/4/)를 보고  
  쿠키에 httpOnly 설정으로 저장하는 방법을 사용하기로 결정했다.
- 당시는 따라하는데 급급하여 무작정 똑같이 하였는데,  
  이번에 [gonic-gin CORS 처리](https://a3magic3pocket.github.io/posts/cors/), [CORS를 이용한 XSS 공격실험](https://a3magic3pocket.github.io/posts/xss-cors-example/)를 통해  
  쿠키에 저장하는 방법에 대한 작은 확신이 가졌으며  
  앞으로도 위와 같은 방법으로 구현해도 괜찮겠다는 생각이 들었다.

## JWT 저장 1. REDUX에 저장
- React에 대한 이해도가 거의 없을 때에는 redux에 JWT를 저장하는 방식을 생각하였다.
- 하지만 redux는 새로고침 시 저장소가 초기화되므로 적합하지 않다.

## JWT 저장 2. redux-persist를 이용하여 저장
- 그럼 redux-persist를 이용하여 redux의 휘발성을 줄이는 방식을 써보는 것은 어떨까?
- 하지만 redux-persist는 localstorage에 저장하므로 다음과 같은 이슈가 발생한다.

## JWT 저장 3. localstorage에 저장
- localstorage는 각각 출처(Origin)에 대해 독립적인 저장 공간을 제공한다([Web Storage 개념과 사용법](https://developer.mozilla.org/ko/docs/Web/API/Web_Storage_API)).
- 즉 브라우저에서 다른 페이지에 접속 시 우리 서비스에서 저장한 JWT를 열람할 수 없다.
- 하지만 결국 javascript code로 localstorage에 저장된 값을 열람할 수 있다는 사실은 변하지 않는다.
- 따라서 XSS 공격을 받을 시 javascript code로 JWT을 열람할 수 있는 이슈가 있다.

## JWT 저장 4. Cookie에 httpOnly 설정을 하고 저장
- Cookie에서 httpOnly 설정을 한 경우 javascript code로 해당 쿠키를 열람할 수 없다.
- 따라서 XSS 공격에 localstorage보다 안전하다.
- 하지만 Cookie에 JWT를 저장하면 매 요청마다 해당 쿠키가 같이 전송된다는 위험이 있다.
- 이는 CSRF 공격에 취약하다는 이야기다.
- CRSF(교차 사이트 요청 위조)([참고](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html))
    - Cross-Site Request Forgery (CSRF) is a type of attack that occurs  
      when a malicious web site, email, blog, instant message, or   
      program causes a user's web browser to perform an unwanted action  
      on a trusted site when the user is authenticated.
    - 브라우저에서 우리 서비스에 로그인 한 상태라면  
      인증정보가 담긴 JWT가 쿠키(httpOnly 설정)로 저장된다.  
    - 이 상태에서 사용자가 피싱 메일 열람 등을 통해 우리 서비스로 아래와 같은 작업을 하면 작동한다.
        - 서비스 탈퇴 요청
        - 사용자 정보 조회
        - ...
    - 왜냐하면 우리 서비스 서버는 이 요청이 사용자가 직접 보낸 요청인지,  
      피싱에 의한 요청인지 알 수 없기 때문이다. 
- 하지만 CRSF는 XSS보다 대응할 수 있다.
    - 참고
        - [Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
        - [CSRF 공격이란? 그리고 CSRF 방어 방법](https://itstory.tk/entry/CSRF-%EA%B3%B5%EA%B2%A9%EC%9D%B4%EB%9E%80-%EA%B7%B8%EB%A6%AC%EA%B3%A0-CSRF-%EB%B0%A9%EC%96%B4-%EB%B0%A9%EB%B2%95)
    - Referer 체크
        - Referer 요청 헤더는 현재 요청을 보낸 페이지의 절대 혹은 부분 주소를 포함한다([참고](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Referer))
        - Referer를 확인하여 해당 요청이 우리 서비스에서 온 것인지, 피싱 요청인지 확인할 수 있다.
        - 하지만 Referer 역시 요청 단에서 조작이 가능하다.
    - Double Submit Cookie 체크
        - 우리 서비스에서는 전송할때마다 임의의 난수 쿠키를 할당하고 요청 시 해당 난수를 따로 헤더에 등록하여 전송한다.
        - [쿠키의 동일출처정책](https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy#%EA%B5%90%EC%B0%A8_%EC%B6%9C%EC%B2%98_%EB%8D%B0%EC%9D%B4%ED%84%B0_%EC%A0%80%EC%9E%A5%EC%86%8C_%EC%A0%91%EA%B7%BC) 때문에 서버에서 요청을 받을 때    
        우리 서비스의 요청에서는 난수 등록 쿠키가 조회되지만  
        피싱 요청의 경우 난수 등록 쿠키가 조회되지 않을 것이다.
        - 따라서 서버 단에서는 요청의 난수 등록 쿠키와 난수 등록 헤더를 불러와 동일 여부만 확인한다.
    
## 결론
- 프론트에서는 JWT를 httpOnly 설정하고 Cookie에 저장하는 방법이 가장 안전한 것 같다.
- CRSF 문제가 생길 수 있으므로 Referer 체크와 Double Submit Cookie 체크를  
  백엔드에서 구현하여 대비한다. 

## 참고
- [4장. JWT 이해 및 적용](https://backend-intro.vlpt.us/4/)
- [동일출처정책](https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy#%EA%B5%90%EC%B0%A8_%EC%B6%9C%EC%B2%98_%EB%8D%B0%EC%9D%B4%ED%84%B0_%EC%A0%80%EC%9E%A5%EC%86%8C_%EC%A0%91%EA%B7%BC)
- [Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [CSRF 공격이란? 그리고 CSRF 방어 방법](https://itstory.tk/entry/CSRF-%EA%B3%B5%EA%B2%A9%EC%9D%B4%EB%9E%80-%EA%B7%B8%EB%A6%AC%EA%B3%A0-CSRF-%EB%B0%A9%EC%96%B4-%EB%B0%A9%EB%B2%95)


