---
title: html-proofer HTML parsing error 해결
date: 2021-06-19 07:00:00 +0900
categories: [Jekyll]
tags: [html, jekyll, chirpy] # TAG names should always be lowercase
---

## 개요
- chirpy 테마 사용 중 kramdown의 html language code block에서   
에러가 발생해 테스트를 통과하지 못하는 문제가 발견되었다. 
- 원인은 메인에서 노출되는 미리보기 항목에서 html 코드 부분이 노출되면서  
parsing 에러가 나는 것이었다.
- 이를 막기 위해 글 서두에 일정량의 글을 써서 html 코드가 미리보기에서 
노출되지 않도록 하면 된다(미리보기에서 escape하는 기능이 필요한듯).
- (추가)chirpy 테마 최신 버전을 사용하면 이 문제가 해결된다.

## Test 용 html
- ~~~ xml
    <div>test지롱</div>
    ~~~