---
title: realme x stock rom으로 복구
date: 2021-01-05 12:20:00 +0900
categories: [Android]
tags: [realme x, unlock, unbrick, custom rom]    # TAG names should always be lowercase
---
## 개요
- 설명 
    - 벽돌(soft brick)된 realme x를 다시 제조사 롬(stock rom)으로 되돌리는 방법이다.
    - 웹에 있는 정보 중 유일하게 성공한 방법이다.
- 기기 정보
    - 모델
        - realme x
    - 롬
        - realme UI 1.0
        - (os에서 버전 확인해보면 RMX 1901 *** c.17 이었던 것 같다.)
        - (글로벌 롬은 RMX 1901EX로 시작하는데 좀 이상하긴 함)
- pc 정보
    - os
        - Windows 10 pro

## TWRP 설치
- 방법
    - 아래 글을 참고하여 설치한다.
    - [realme x 언락 및 커스텀 롬 설치](https://a3magic3pocket.github.io/posts/realme-x-unlock-and-install-custom-rom/){:target="_blank"}

## 제조사 롬 설치 1
- 방법
    - [software-update](https://www.realme.com/in/support/software-update){:target="_blank"}로 이동하여 realme ozip 파일을 다운로드한다.
    - ozip 파일을 pc에서 휴대폰으로 옮긴다.
    - twrp에서 wipe > format Data를 눌러 포맷한다.
    - 이후 Advanced Wipe를 누르고 system, dalvik, cache를 선택 후 wipe를 누른다.
    - 메인으로 돌아가 install에서 ozip 파일을 설치한다.
- 설명
    - 사실 이 방법으로도 벽돌은 해결되지 않는다.
    - 나의 경우 destory 에러가 계속 됐다.
    - 하지만 신기하게 제조사 롬을 이렇게 설치한 후 다음 방법을 진행하면 원상복구된다(두 번이나 해봄).

## 제조사 롬 설치 2
- 방법
    - 아래 링크대로 진행하면 된다.
    - [realme Flash tool Tutorial for realme X(realme UI based on Android 10)](https://c.realme.com/in/post-details/1280819845490802688){:target="_blank"}
    - realme community 글이고, 작성자가 administrator 이므로 공식적인 것 같다.
- 설명
    - "제조사 롬 설치 1"을 먼저 진행한 후 진행해야 정상 복구가 된다.
    - realme flash tool로 ofp를 설치하다보면 에러가 뜬다. 
    - system.img와 vendor.img를 설치할 때 volume이 부족하다는 에러가 뜨는데 상관없이 정상복구 되었다.

## 기타 참고하면 좋은 사이트
- 링크
    - [Go Back to Stock ROM](https://forum.xda-developers.com/t/guide-go-back-to-stock-rom-stock-recovery.3974607/){:target="_blank"}
    - [How to Unbrick Realme Phones (Fix Bootloop & Bricked Phones)](https://www.ytechb.com/how-to-unbrick-realme-phones-fix-bootloop-bricked-phones/){:target="_blank"}
    - [software-update](https://www.realme.com/in/support/software-update){:target="_blank"}에서 How to update? 클릭
- 설명
    - 결론은 저기 있는 모든 방법을 해도 안 된다.
    - 내가 생각하는 결정적인 이유는 realme x가 sdcard가 없기 때문이다.
    - realme 공홈 how to update? 내용을 보면 sdcard로 ozip파일을 옮긴 후 제조사 recovery에서 install 하라고 되어있다.
    - 하지만 realme x는 sdcard가 없었고 그래서 internal storage에서 ozip 파일을 접근해야하는데, 인식을 하지 못한다.
    - 아마 sdcard가 있는 다른 realme 기종이라면 이 방법을 통해 쉽게 정상복구 할 수 있을 것이라고 생각한다.

## 참고
- [realme x 언락 및 커스텀 롬 설치](https://a3magic3pocket.github.io/posts/realme-x-unlock-and-install-custom-rom/){:target="_blank"}
- [software-update](https://www.realme.com/in/support/software-update){:target="_blank"}
- [realme Flash tool Tutorial for realme X(realme UI based on Android 10)](https://c.realme.com/in/post-details/1280819845490802688){:target="_blank"}
- [Go Back to Stock ROM](https://forum.xda-developers.com/t/guide-go-back-to-stock-rom-stock-recovery.3974607/){:target="_blank"}
- [How to Unbrick Realme Phones (Fix Bootloop & Bricked Phones)](https://www.ytechb.com/how-to-unbrick-realme-phones-fix-bootloop-bricked-phones/){:target="_blank"}
- [software-update](https://www.realme.com/in/support/software-update){:target="_blank"}