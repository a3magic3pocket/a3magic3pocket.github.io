---
title: windows-nvm d drive 설치 시 npm 인식 못하는 이슈
date: 2023-09-26 21:59:21 +0900
categories: [Node]
tags: [windows-nvm, npm]    # TAG names should always be lowercase
---

## 현상
- nvm 설치 시 root 경로를 D drive로 설정한 경우  
  D drive에서 nvm 명령어 입력 시 에러가 발생한다.  

## 에러
- ERROR open \settings.txt: The system cannot find the file specified  

## 해결방법
- nvm root 경로 하위를 보면 settings.txt를 발견할 수 있다.  
- settings.txt를 C drive 하위로 옮기면 위 에러가 해결된다.   

## 참고
- [when install in another partition npm is not working #46](https://github.com/coreybutler/nvm-windows/issues/46){:target="_blank"}  
