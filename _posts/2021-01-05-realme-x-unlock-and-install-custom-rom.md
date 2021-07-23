---
title: realme x 언락 및 커스텀 롬 설치
date: 2021-01-05 12:00:00 +0900
categories: [Android]
tags: [realme x, unlock, unbrick, custom rom]    # TAG names should always be lowercase
---
## 개요
- 동기 
    - 갑작스럽게 realme x 공기계를 사용할 일이 생겼다.
    - 아무래도 중국 롬이다 보니 사소한 일을 할 때에서 개인정보 동의 요청을 받는다.
    - 그래서 실 사용을 위해 커스텀롬을 올리는 작업을 2~3일 간 진행하였다.
    - realme 휴대폰에 커스텀 롬을 올리려는 분이 있다면 도움이 되길 바란다.
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
- 설명 방법
    - 개념 정의 보다는 실행방법 위주로 설명한다.
    - 최소한의 개념만 추상적으로 설명한다.

## Bootloader 언락
- 정의
    - Bootloader 언락(이하 언락)은 휴대폰 부팅 시 어떤 이미지를 참조할 지 컨트롤할 수 있게 만드는 방법인 것 같다.
    - 제조사는 기본적으로 언락을 막아 놓고, 특정 소프트웨어를 배포해 필요할 경우 언락을 할 수 있도록 하고 있다.
    - realme x 제조사인 oppo에서도 bootloader 언락 툴을을 배포했다.
    - 이를 이용하여 언락하면 된다.
- 주의사항
    - 언락을 하게 되면 Widevine DRM L1에서 L3으로 떨어진다.
    - 이러면 일부 스트리밍 앱(ex. 넷플릭스)에서 같은 영상을 고화질로 볼 수 없게 된다.
- 방법
    - [언락 방법](https://www.droidthunder.com/unlock-bootloader-of-realme-x/){:target="_blank"}
- 트러블 슈팅
    - In-depth Test 설치 시간
        - 링크에서는 1시간 정도면 설치된다고 나와있는데, 한국에서는 훨씬 오래 걸린다.
        - 10시간 이후 확인해보니 설치 되어있었다.
        - 휴대폰 종료 후 실행하려면 In-depth Test 앱에서 Query verification status를 누르면 된다.
    - 롬 부팅 시에는 pc에서 휴대폰 인식이 되는데 Fastboot에서 인식이 안되는 현상
        - Fastboot에서는 bootloader 용 드라이버를 따로 설치해줘야 인식된다.
        - [Google USB Driver 설치](https://blog.naver.com/PostView.nhn?blogId=resumet&logNo=221836838267){:target="_blank"}
        - 드라이버 설치 확인
            - 메뉴에서 "장치 관리자" 검색 후 클릭
            - 기타 장치에서 느낌표 마크가 붙은 Android 장비가 있다면 아직 드라이버 설치가 제대로 안 된 것이다.
            - 없다면 설치 완료
        - Fastboot에 기기 인식 확인
            - 아래 명령어를 입력했을 때 기기가 노출된다면 인식이 완료된 것이다.  
              (없다면 언락 방법 링크에서 "Minimal ADB and Fastboot"로 설치하거나 안드로이드 스튜디오로 설치하면 된다.)  
              (adb 또는 fastboot 시 반응이 없다면 전역변수 등록이 안된 것이다.  
              전역변수 등록이 어렵다면 platform-tools 디렉토리로 이동하여 명령어를 실행하라. )  
            - ```
                > fastboot devices
                ```
        
## TWRP 설치
- 정의
    - 데이터를 지우고 롬을 설치할 때 쓸 수 있는 도구이다.
    - Bootloader에서 recovery로 부트하면 접속할 수 있다.
    - 전원 종료 상태에서 Bootloader로 접속하려면 (전원 + 볼륨 아래)를 동시에 누르면 된다.
    - twrp에 접속하면 pc와 휴대폰 간에 파일을 전송 받을 수 있다.
- 설치
    - xda에서 realme x 용 unofficial twrp 이미지를 받을 수 있다. [링크](https://forum.xda-developers.com/t/recovery-3-4-0-10-rmx1901-unofficial-twrp-for-realme-x-stable.3959556/){:target="_blank"}
    - 내가 사용한 버전은 __TWRP 3.4.0-10 Unofficial by mauronofrio__ 이다.
    - fastboot에서 twrp를 recovery 이미지로 설정한다.
    - pc에서 twrp-3.4.0-10-RMX1901-mauronofrio.img를 platform-tools 디렉토리로 옮긴다.
    - ```
        > fastboot devices
        > cd platform-tools
        > fastboot flash recovery twrp-3.4.0-10-RMX1901-mauronofrio.img
        ```

## 커스텀롬 설치(중요)
- 설명
    - 여기서 실수하면 바로 벽돌(soft brick) 상태가 된다.
    - RR롬은 내가 성공한 유일한 커스텀 롬이다.
    - 한 번 벽돌이되면 복구하기 매우 힘드니  
      특정 커스텀 롬 설치를 원하는게 아니라면 아래 방법으로 RR 롬을 설치하기를 권장한다.
    - 설치 시 모든 데이터를 삭제되니 미리 pc로 백업한다.
- 방법
    - [[10.0][OFFICIAL]Resurrection Remix v8.x Realme X(RMX1901) 설치 링크](https://forum.xda-developers.com/t/10-0-official-resurrection-remix-v8-x-realme-x-rmx1901.4156257/){:target="_blank"}
    - 기본적으로 설치 링크에 있는 대로 하면 된다.
    - 이해를 돕기 위해 내가 진행한 그대로 방법 상세에서 상세하게 설명한다.
- 방법 상세
    - (전원 + 볼륨 아래)를 bootloader로 접속 후 볼륨 버튼으로 reboot recovery를 선택하여 twrp로 접속한다. 
    - 메인에서 wipe를 누르고 factory reset를 슬라이드한다. 
    - Format Data를 눌러 포맷한다.
    - 이후 Advanced Wipe를 누르고 system, dalvik, cache를 선택 후 wipe를 누른다.
    - RR 설치 링크에서 Global Firmware와 RR rom을 다운로드 한 후 휴대폰으로 옮긴다.
    - twrp에서 install 눌러 나오는 화면에서 옮겨졌는지 확인할 수 있다(위치 /sdcard)
    - install에서 Global Firmware zip 파일(이름 rui-firmware-only.zip)을 선택하고 flash 한다.
    - 다 됐으면 같은 방법으로 RR rom을 flash 한다.
    - 이제 GApps을 설치해야한다. GApps를 설치해야 play store가 생긴다.
    - RR 설치링크에서 GApps 다운로드 링크를 누르면 GApps 페이지로 이동된다.
    - Platform: ARM64, Android: 10, Variant: pico를 선택하고 다운로드한다.
    - 이후 twrp로 옮긴 후 install에서 flash 한다.
    - 설치 완료 후 system으로 reboot 한다.
    - RR 로고가 뜬다면 성공
    - 안 뜨고 무한 부팅 상태가 발생한다면 벽돌 상태이다.


## 기본적인 문제 해결법
- 설명
    - 벽돌 상태에서 문제를 해결하기 위한 다양한 방법을 시도해봤다.
    - 제조사 rom으로 이동하는 방법은 [다음 글](https://a3magic3pocket.github.io/posts/go-to-stock-rom/){:target="_blank"}에서 상세하게 설명할 것이다.
    - 여기서는 기본적인 조치 방법을 설명한다.
- ozip 파일에서 zip 파일 추출
    - [software-update](https://www.realme.com/in/support/software-update){:target="_blank"}에서 제조사에서 제공하는 ozip 파일을 제공한다.
    - ozip에 필요한 boot.img와 vbmeta.img, recovery.img를 획득할 수 있다.
    - 파이썬을 이용한 방법이 있으나 twrp에서 처리하는 방법이 더 쉽다(twrp-3.4.0-10 버전에서는 ozip 처리 기능 있음)
    - 이 방법은 휴대폰 데이터가 소실되므로 벽돌일 때만 하는 것이 좋다.
    - realme x 용 ozip 파일을 다운로드 받은 후 pc에서 휴대폰으로 옮긴다.
    - trwp에서 install 한다. 
    - 설치가 완료되면 zip 파일이 남는다.
    - 해당 zip 파일에서 boot.img와 vbmeta.img, recovery.img를 추출한다.
- 부트 이미지 변경
    - bootloader로 부팅한다.
    - pc에서 boot.img를 platform-tools 디렉토리로 옮긴다.
    - 아래 명령어를 입력한다.
    - ```
        > fastboot devices
        > cd platform-tools
        > fastboot flash boot boot.img
        ```
- 리커버리 이미지 변경
    - 추출한 recovery.img는 제조사 리커버리 이미지이다.
    - sdcard가 존재하는 oppo 폰은 제조사 리커버리 이미지로 벽돌 복구가 가능하므로 twrp 대신 제조사 recovery로 변경해야한다.
    - pc에서 recovery.img를 platform-tools 디렉토리로 옮긴다.
    - 아래 명령어를 입력한다.
    - ```
        > fastboot devices
        > cd platform-tools
        > fastboot flash recovery recovery.img
        ```
    - twrp를 다시 리커버리 모드로 사용하려면 다시 twrp 이미지를 recovery로 flash 하면 된다.
- 부트 / 이미지 오류 에러
    - 오류
        - 메세지 
            - "the current image(boot/recovery) have been destroyed and cannot boot"
        - 관련 xda 링크
            - [링크](https://forum.xda-developers.com/t/the-current-image-boot-recovery-have-been-destroyed-and-cannot-boot.3987005/){:target="_blank"}
    - RR 설치 전 AOSP 롬 설치하다가 뜬 오류이다.
    - AOSP 롬 설치 후 부팅을 하면 위 오류가 계속 뜨면서 무한 재부팅된다.
    - RR 롬에 적힌 해결 방법([참고](https://forum.xda-developers.com/t/10-0-official-resurrection-remix-v8-x-realme-x-rmx1901.4156257/page-2#post-83532029){:target="_blank"})
        - vbmeta.img를 platform-tools 디렉토리로 옮긴다.
        - vbmeta.img를 flash 한다.
        - ```
            > fastboot devices
            > cd platform-tools
            > fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img
            ```
    - 나는 stock rom으로 복구하여 해결하였다. 이 방법은 [다음 글](https://a3magic3pocket.github.io/posts/go-to-stock-rom/){:target="_blank"}에서 작성한다.
- 강제 재부팅
    - twrp에서 edl 모드로 재부팅할 수 있다.
    - 이러면 검은화면으로 부팅되고 어떤 키도 먹지 않고, pc 연결도 되지 않는다.
    - 퀄컴 드라이버를 설치하여 edl 모드에서 pc 접속이 가능하게 만들 수도 있으나, 가장 쉬운 방법은 강제 재부팅하는 것이다.
    - (전원 + 볼륨 아래 + 볼륨 위)를 동시에 누른다.

## 참고
- [언락 방법](https://www.droidthunder.com/unlock-bootloader-of-realme-x/){:target="_blank"}
- [Google USB Driver 설치](https://blog.naver.com/PostView.nhn?blogId=resumet&logNo=221836838267){:target="_blank"}
- [twrp](https://forum.xda-developers.com/t/recovery-3-4-0-10-rmx1901-unofficial-twrp-for-realme-x-stable.3959556/){:target="_blank"}
- [[10.0][OFFICIAL]Resurrection Remix v8.x Realme X(RMX1901) 설치 링크](https://forum.xda-developers.com/t/10-0-official-resurrection-remix-v8-x-realme-x-rmx1901.4156257/){:target="_blank"}
- [software-update](https://www.realme.com/in/support/software-update){:target="_blank"}
- [destroyed 에러 xda](https://forum.xda-developers.com/t/the-current-image-boot-recovery-have-been-destroyed-and-cannot-boot.3987005/){:target="_blank"}
- [RR 롬 vbmeta.img 적용](https://forum.xda-developers.com/t/10-0-official-resurrection-remix-v8-x-realme-x-rmx1901.4156257/page-2#post-83532029){:target="_blank"}