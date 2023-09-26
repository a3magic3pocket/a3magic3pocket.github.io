---
title: javascript, 개별 instance를 전역에서 접근할 수 있도록 변경
date: 2023-09-26 22:54:58 +0900
categories: [javascript]
tags: [javascript]    # TAG names should always be lowercase
---

## 설명
- javascript legacy 소스코드 중   
  class(이하 OldClass)로 처리하고 있는 부분을 발견.  
- OldClass의 특정 method를     
  다른 js파일(이하 NewFile)에서 호출해야하는 상황 발생.  
- 문제는 OldClass에는 contructor 문이 작성되어 있어  
  NewFile 내에서 OldClass instance를 생성하기 어려운 상황.  
- OldClass의 해당 method를 뽑아내어   
  공용 js 파일로 분리시키는 것이 가장 좋으나  
  의존성 파악이 쉽지 않고 긴급하게 처리해야 하는 경우  
  OldClass instance를 window 객체에 할당하여  
  전역으로 접근할 수 있도록 조치하여 해결할 수 있다.  

## 방법
- OldClass instance를 window 객체에 할당한다.  
  ```javascript  
  // old_class.js  
  ...  
  if (window) {  
    window.addEventListener('DOMContentLoaded', function () {  
      window['OldClass'] = new OldClass();  
    });  
  ```  
- window['OldClass']로 할당하면 OldClass를 로드하고 있는 html이나   
  같은 html 내의 다른 js 파일에서 바로 OldClass instance로 접근할 수 있다.  
    - OldClass를 로드하고 있는 html 예시  
      ```html  
      <button onclick="OldClass.SomeClickHandler()"></button>  
      ```  
    - 같은 html 내의 다른 js 예시  
      ```javascript  
      class NewFile {  
        constructor() {  
          if (window && window['OldClass'] && window['OldClass'].SomeMethod) {  
            window['OldClass'].SomeMethod();  
          }  
        }  
      }  
      ```  

## 예시
- 설명  
    - 글씨색: green, 배경색: pink인 버튼이 있다.  
    - OldClass에서는 글씨색을 변경하는 method가 있다.  
    - NewFile에서는 배경색을 변경하는 method가 있다.  
    - NewFile에서 버튼의 클릭 이벤트 함수를 지정한다.  
    - 버튼 클릭 시, 버튼을 글씨색:red, 배경색: yellow로 변경할떄  
      OldClass의 method를 호출하여 글씨색:red를 처리하고자 한다.  
- 파일  
    - main.html  
      ```html  
      <!DOCTYPE html>  
      <html lang="ko">  
        <head>  
          <meta charset="UTF-8" />  
          <meta name="viewport" content="width=device-width, initial-scale=1.0" />  
          <link rel="stylesheet" href="main.css" />  
          <title>hello world</title>  
        </head>  
        <body>  
          <button id="my-btn" class="color-green bg-color-pink">  
            <strong>버튼</strong>  
          </button>  
                
          <script src="old_class.js"></script>  
          <script src="new_file.js"></script>  
        </body>  
      </html>  
      ```  
    - main.css  
      ```css  
      .color-green {  
        color: green;  
      }  
                
      .color-red {  
        color: red;  
      }  
                
      .bg-color-pink {  
        background-color: pink;  
      }  
                
      .bg-color-yellow {  
        background-color: yellow;  
      }  
      ```  
    - old_class.js  
      ```javascript  
      class OldClass {  
        constructor() {  
          console.log('Created OldClass instance');  
        }  
                
        /**  
         * 버튼 글씨색을 빨강으로 변경  
         */  
        changeBtnColorRed() {  
          const btnElem = document.querySelector('#my-btn');  
          const colorClassName = 'color-red';  
          this.#changeColor(btnElem, colorClassName);  
        }  
                
        /**  
         * className이 버튼 글씨색 class 인지 확인  
         * @param {string} className   
         * @returns   
         */  
        #checkColorClass(className) {  
          return className.substring(0, 6) === 'color-';  
        }  
                
        /**  
         * elem의 글씨색을 className으로 변경  
         * @param {Element} elem   
         * @param {string} className   
         * @returns {boolean} 변경여부  
         */  
        #changeColor(elem, className) {  
          if (elem && className) {  
            const isColorClassName = this.#checkColorClass(className);  
            if (!isColorClassName) {  
              return false;  
            }  
                
            const oldClassNames = elem.classList;  
            for (const oldClassName of oldClassNames) {  
              if (this.checkColorClass(oldClassName)) {  
                elem.classList.remove(oldClassName);  
              }  
            }  
                
            elem.classList.add(className);  
                
            return true;  
          }  
                
          return false;  
        }  
      }  
                
      if (window) {  
        window.addEventListener('DOMContentLoaded', function () {  
          window['OldClass'] = new OldClass();  
        });  
      }  
      ```  
    - new_file.js  
      ```javascript  
      class NewFile {  
        constructor() {  
          this.handleBtnClick();  
        }  
                
        /**  
         * 버튼이 눌렸을 때 동작을 지정  
         */  
        handleBtnClick() {  
          const btnElem = document.querySelector('#my-btn');  
          btnElem.addEventListener('click', () => {  
            this.changeBtnColorRedAndBgColorYellow();  
          });  
        }  
                
        /**  
         * 버튼의 글씨색을 빨강으로  
         * 배경색을 노랑으로 변경  
         */  
        changeBtnColorRedAndBgColorYellow() {  
          const btnElem = document.querySelector('#my-btn');  
          const bgColorClassName = 'bg-color-yellow';  
          this.#changeBgColor(btnElem, bgColorClassName);  
          if (window && window['First'] && window['First'].changeColor) {  
            window['First'].changeBtnColorRed();  
          }  
        }  
                
        /**  
         * className이 배경색을 변경하는 class인지 확인  
         * @param {*} className   
         * @returns   
         */  
        #checkBgColorClass(className) {  
          return className.substring(0, 9) === 'bg-color-';  
        }  
                
        /**  
         * elem의 배경색을 className으로 변경  
         * @param {Element} elem   
         * @param {string} className   
         * @returns {boolean} 변경여부  
         */  
        #changeBgColor(elem, className) {  
          if (elem && className) {  
            const isBgColorClassName = this.#checkBgColorClass(className);  
            if (!isBgColorClassName) {  
              return false;  
            }  
                
            const oldClassNames = elem.classList;  
            for (const oldClassName of oldClassNames) {  
              if (this.#checkBgColorClass(oldClassName)) {  
                elem.classList.remove(oldClassName);  
              }  
            }  
                
            elem.classList.add(className);  
                
            return true;  
          }  
                
          return false;  
        }  
      }  
                
      if (window) {  
        window.addEventListener('DOMContentLoaded', function () {  
          window['NewFile'] = new NewFile();  
        });  
      }  
      ```  
- 최초화면  
    - <a href='/assets/img/2023-09-26-javascript-instance-inject-window/00-initial.jpg' target='_blank'><img src='/assets/img/2023-09-26-javascript-instance-inject-window/00-initial.jpg' width='80%' height='80%'></a>  
- 클릭 후 결과  
    - <a href='/assets/img/2023-09-26-javascript-instance-inject-window/01-after.jpg' target='_blank'><img src='/assets/img/2023-09-26-javascript-instance-inject-window/01-after.jpg' width='80%' height='80%'></a>  
