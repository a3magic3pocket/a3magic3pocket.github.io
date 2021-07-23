---
title: Label 태그에 개행(newLine) 넣기
date: 2021-06-18 17:15:00 +0900
categories: [Javascript]
tags: [javascript, golang] # TAG names should always be lowercase
---

## 개요
- Label 태그에 개행을 넣은 경험을 기록한다.
- gonic-gin으로 환경을 만들어 실험환경을 구축한다.
- js를 통해 문제를 해결한다.

## Label 태그
- Input 태그를 설명하는 문자열이 필요한 경우, Label 태그에 넣어 추가한다.
- Label 태그 사용법
    - Label 태그의 for attribute에 Input 태그 id 할당
        - ~~~ xml
            <input type="text" name="testInput" id="test-input-1">
            <label for="test-input-1">항목1</label>
            ~~~
    - Label 태그 안에 Input 태그를 넣어서 사용
        - ~~~ xml
            <label>항목1
              <input type="text" name="testInput">
            </label>
            ~~~
- 특징
    - Label 태그 안에는 inline level elements만 사용해야한다. ([W3C 17.9.1 The LABEL element](https://www.w3.org/TR/html4/interact/forms.html#h-17.9.1){:target="\_blank"})
    - [inline level elements 예시](http://web.simmons.edu/~grovesd/comm244/notes/week4/block-inline){:target="\_blank"}
    - 따라서 문자열에 br 태그를 추가해서 개행을 구현한다.

## 실험 환경 구축
- 개요
    - gonic-gin framework로 간단한 웹 서버 환경 구축
    - html 파일은 view 디렉토리 하위에 저장.
- 디렉토리 구조
    - main.go
    - /view
        - test.html
- main.go
    - ~~~ go
        package main

        import (
            "net/http"

            "github.com/gin-gonic/gin"
        )

        func main() {
            router := gin.Default()

            router.LoadHTMLGlob("view/*")
            router.GET("test", func(c *gin.Context) {
                c.HTML(
                    http.StatusOK, 
                    "test.html", 
                    gin.H{"testText": "line1<br />line2"},
                )
            })

            router.Run(":8888")
        }
        ~~~
- /view/test.html
    - ~~~ xml
        <!DOCTYPE html>
        <html lang="en">
        <head>
            <meta charset="UTF-8" />
            <meta http-equiv="X-UA-Compatible" content="IE=edge" />
            <meta name="viewport" content="width=device-width, initial-scale=1.0" />
            <title>Document</title>
        </head>
        <body>
            <input type="checkbox" id="test-input" />
            <label for="test-input">{{ .testText }}</label>
        </body>
        </html>
        ~~~

## 실험 시작
- 실행 명령
    - ~~~bash
        go run main.go
        ~~~
- test 페이지 확인
    - 브라우저를 켠다.
    - http://localhost:8888/test 에 접속한다.

## 실험 결과
- 실패
    - <a href="/assets/img/2021-06-18-apply-innerHTML-in-label-tag/00-fail-page.jpg" target="_blank"><img src="/assets/img/2021-06-18-apply-innerHTML-in-label-tag/00-fail-page.jpg" width="70%"></a> 
    - br 태그가 텍스트로 그대로 노출된다.
    - gin.H로 받은 값이 input 태그의 innerText로 저장되어서 그런 것으로 추정된다.
    - 해당 텍스트를 innerHTML로 저장하는 스크립트를 추가하면 문제를 해결할 수 있다.

## 조치
- /view/test.html
    - ~~~ xml
        <!DOCTYPE html>
        <html lang="en">
        <head>
            <meta charset="UTF-8" />
            <meta http-equiv="X-UA-Compatible" content="IE=edge" />
            <meta name="viewport" content="width=device-width, initial-scale=1.0" />
            <title>Document</title>
        </head>
        <body>
            <input type="checkbox" id="test-input" />
            <label for="test-input">{{ .testText }}</label>
            <script type="text/javascript">
                document.addEventListener("DOMContentLoaded", function () {
                    reAssignTestInputLabel();
                });

                // reAssignTestInputLabel : testInput의 innerText를 innerHTML로 변경.
                function reAssignTestInputLabel() {
                    var testInputElem = document.querySelector("#test-input + label");
                    testInputElem.innerHTML = testInputElem.innerText;
                }
            </script>
        </body>
        </html>
        ~~~

## 조치 결과
- 성공
    - <a href="/assets/img/2021-06-18-apply-innerHTML-in-label-tag/01-success-page.jpg" target="_blank"><img src="/assets/img/2021-06-18-apply-innerHTML-in-label-tag/01-success-page.jpg" width="70%"></a> 


## 참고
- [Element div not allowed as child of element label in this context.](https://www.sitepoint.com/community/t/element-div-not-allowed-as-child-of-element-label-in-this-context-suppressing-further-errors-from-this-subtree/257008/6){:target="\_blank"}
- [W3C 17.9.1 The LABEL element](https://www.w3.org/TR/html4/interact/forms.html#h-17.9.1){:target="\_blank"}
- [HTML tags inside label](https://stackoverflow.com/questions/4461942/html-tags-inside-label){:target="\_blank"}
- [inline level elements 예시](http://web.simmons.edu/~grovesd/comm244/notes/week4/block-inline){:target="\_blank"}
- [HTML Block and Inline Elements](https://www.w3schools.com/html/html_blocks.asp){:target="\_blank"}
- [Javascript) innerText vs innerHTML](https://hi098123.tistory.com/83){:target="\_blank"}

