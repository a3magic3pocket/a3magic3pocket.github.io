---
title: Support for the experimental syntax 'decorators-legacy' isn't currently enabled
date: 2024-02-19 12:07:53 +0900
categories: [Javascript]
tags: [babel]    # TAG names should always be lowercase
---

## 현상
- next.js에서 api 개발 시 typeorm을 적용하려고 하니  
  에러가 발생한다.  
- 아마 typeorm의 데코레이터 문법이 원인으로 보이며  
  바벨에서 데코레이터 문법으로 지원하게 설정해주면 된다.  

## 해결방안
- @babel/plugin-proposal-decorators 설치  
  ```bash  
            
  npm install @babel/plugin-proposal-decorators --save  
            
  ```  
            
- .babel 파일 수정  
  ```json  
            
  {   
      "presets": ["next/babel"],   
      "plugins": [  
          ["@babel/plugin-proposal-decorators", { "legacy": true }]  
      ]  
  }  
            
  ```  

## 참고
- [Syntax error - Support for the experimental syntax 'decorators-legacy' isn't currently enabled](https://stackoverflow.com/questions/52262084/syntax-error-support-for-the-experimental-syntax-decorators-legacy-isnt-cur){:target="_blank"}  
- [@babel/plugin-proposal-decorators의 legacy: true 옵션](https://simsimjae.tistory.com/446){:target="_blank"}  
