---
title: 이모지(emoji) 삽입 실패 문제 해결
date: 2021-05-05 07:58:00 +0900
categories: [MariaDB]
tags: [golang, unicode, collation, emoji, regexp]    # TAG names should always be lowercase
---
## 개요
- DB 관련 에러를 처리하여 이를 기록한다.
- DB에러는 Error 1366: Incorrect string value 로 입력된 string에  
특정 emoji가 포함되어 발생하는 것으로 파악되었다.
- 여러 방법 중 문제의 emoji를 대체 텍스트로 변경하는 방법으로 해결하였다.

## 문제 발생 이유
- MariaDB collation
    - MariaDB에서 table을 생성할 때 charset과 collation을 지정한다.
    - charset은 문자(기호)와 인코딩의 집합이고,  
    collation은 charset안의 문자들의 비교를 위한 규칙들이다([참고](https://dev.mysql.com/doc/refman/8.0/en/charset-general.html){:target="_blank"}).
    - 보통 한국어가 포함되는 테이블의 charset은 utf8, collation은 utf8_general_ci를 사용한다.
- utf8_general_ci는 3byte만 허용
    - utf8은 유니코드 문자 인코딩 방식 중 하나이다.
    - utf8은 4byte까지 사용하나 전세계 언어를 표현하는데에는 3byte면 충분하다.
    - 그래서 Mysql 및 MariaDB에는 효율성을 위해 utf8_general_ci를 3byte로 설계했다.
    - 그러나 emoji 중 4byte짜리가 등장하면서  
    utf8_general_ci에 저장하지 못하는 문제가 발생한 것이다.

## 해결방법 1: collation 변경
- MariaDB collation을 utf8_general_ci를 utf8mb4_unicode_ci로 바꾼다.
    - utf8mb4_unicode_ci collation은 4byte utf8 인코딩도 저장할 수 있다.
    - 그러나 운영 중인 DB의 설계를 변경하는 것을 꽤나 큰 리스크이기 때문에  
    우리는 4byte emoji를 필터링 하는 방법을 선택했다.
- 유의사항
    - utf8은 utf8mb4의 서브셋이라 중간에 변경해도 큰 문제는 없는 것 같다.
    - 다만 기존에 index가 걸려있다면 index 길이 때문에 문제가 생길 수 있는 것 같다.
    - 자세한 사항은 ["MySQL utf8에서 utf8mb4로 마이그레이션 하기"](https://www.letmecompile.com/mysql-utf8-utf8mb4-migration/){:target="_blank"} 참고

## 해결방법 2: 4byte emoji를 대체 텍스트로 치환
- Golang에서 정규식(Regular expression)으로 emoji 처리하기
    - utf8 인코딩은 16진수로 표현한다. ex. U+0800
    - 정규식에서의 표현은 \u0800 와 같이 표현할 수 있는데    
    Golang에서는 이 표현 대신 \x{0800}으로 표현한다([참고](https://www.regular-expressions.info/unicode.html){:target="_blank"})
- 4byte emoji의 16진수 값 확인
    - Ref1. [Emoji](https://www.w3schools.com/charsets/ref_emoji.asp){:target="_blank"} 에서 (1F004~1F9E6).
	- Ref2. [Emoji Smileys](https://www.w3schools.com/charsets/ref_emoji_smileys.asp){:target="_blank"} 는 전부 해당. (1F600~1F9D0)
- 4byte emoji를 대체 텍스트로 치환하는 코드
    - [직접 실행](https://play.golang.org/p/-Tvvz0v30Tc){:target="_blank"}  

    ```
    package main

    import (
        "fmt"
        "regexp"
    )

    func main() {
        var emojiRx = regexp.MustCompile(`[\x{1F004}-\x{1F9E6}]|[\x{1F600}-\x{1F9D0}]`)
        var s = emojiRx.ReplaceAllString("Thats a nice joke 🥰 🥵 🥶 🥳 🥴 🥺 👨‍🦰 👩‍🦰 👨‍🦱 👩‍🦱 👨‍🦲 👩‍🦲 👨‍🦳 👩‍🦳 🎨 🎦", `[e]`)
        fmt.Println(s)
    }
    ```

## 참고
- [Character Sets and Collations in General](https://dev.mysql.com/doc/refman/8.0/en/charset-general.html){:target="_blank"}
- [MySQL utf8에서 utf8mb4로 마이그레이션 하기](https://www.letmecompile.com/mysql-utf8-utf8mb4-migration/){:target="_blank"}
- [Matching a Specific Code Point](https://www.regular-expressions.info/unicode.html){:target="_blank"}
- [Emoji](https://www.w3schools.com/charsets/ref_emoji.asp){:target="_blank"}
- [Emoji Smileys](https://www.w3schools.com/charsets/ref_emoji_smileys.asp){:target="_blank"}
- [golang regexp example](https://stackoverflow.com/questions/39575798/how-to-replace-emoji-characters-in-string-using-regex-in-golang/39577651){:target="_blank"}