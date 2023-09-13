---
title: Markdown link 정규식(regular expression)
date: 2023-09-13 19:16:35 +0900
categories: [regex]
tags: [markdown, regex]    # TAG names should always be lowercase
---

## 개요
- 마크다운 문서에서 링크만 추출하여   
  모든 링크에 '{:target="_blank"}'을 추가하는   
  golang 프로그램을 작성하고자 한다.  
- 마크다운 이미지와 마크다운 문서를 구분하기 위해  
  가장 앞에 !가 붙는지 안 붙는지 정규식에서 판단해야한다.  
- 이때 negative lookahead를 사용하려 했으나 (?!pattern)  
  golang에서는 성능 정책 이슈로 지원하지 않는다고 한다.  
  [Negative look-ahead in Go regular expressions](https://stackoverflow.com/questions/26771592/negative-look-ahead-in-go-regular-expressions){:target="_blank"}  
- !가 앞에 붙지 않는 경우와 [가 처음으로 시작하는 경우를  
  정규식으로 만들어 처리하였다.  

## 마크다운 링크
- 현재 창에서 열기  
  ```markdown  
  [링크이름](링크)  
  ```  
- 새 창에서 열기  
  ```  
  [링크이름](링크){:target="_blank"}  
  ```  

## 마크다운 이미지
- 정규식  
  ```markdown  
  ![이미지이름](이미지링크)  
  ```  

## 기본 마크다운 링크 정규식
- 정규식  
  ```markdown  
  \[.*?\]\(.+?\)  
  ```  
- 설명  
    - 마크다운 링크 형식의 글은 모두 추출한다.  
    - 다만 마크다운 이미지와 마크다운 링크를 구분하지 못한다.  

## 앞에 !가 오면 추출하지 않는 마크다운 링크 정규식
- 정규식  
  ```  
  [^!]\[.*?\]\(.+?\)  
  ```  
- 설명  
    - 앞에 !가 붙으면 추출하지 않도록 수정하였다.  
    - 그러나 앞에 문자도 없이 링크부터 시작하는 행의 경우 추출하지 못한다.  

## 앞에 ! 외의 다른 글자가 오거나 [로 시작하는 마크다운 링크 정규식
- 정규식  
  ```  
  [^!]\[.*?\]\(.+?\)|^\[.*?\]\(.+?\)  
  ```  
- 설명  
    - 마크다운 이미지를 제외하고 마크다운 링크만 추출할 수 있다.  

## 모든 마크다운 링크에 {:target="_blank"}를 붙이기
- 설명  
    - 중복으로 blank구분이 붙여지는 것을 방지하기 위하여  
      모든 마크다운 링크의 blank구문을 제거한다.  
    - 이후 다시 링크를 추출한 후 blank구문을 추가한다.  
- {:target="_blank"}(이하 blank구문)가 붙은 마크다운 링크 정규식  
    - 정규식  
      ```  
      `[^!]\[.*?\]\(.+?\)(\s*{:target\s*=\s*["']_blank["']\s*})|^\[.*?\]\(.+?\)(\s*{:target\s*=\s*["']_blank["']\s*})`  
      ```  
    - 설명  
        - 정규식에서 blank구문을  캡쳐한다.  
        - blank구문은 다양한 모양이 추출될 수 있다.  
          ex) {:target = "_blank"}, {:target= '_blank'}  
        - 추출된 blank구문을 strings.replaceAll(행, blank구문, "")을   
          이용하여 제거한다.  
        - 이제 모든 링크에 blank구문이 없어졌다.  
        - 이후 마크다운 링크 정규식을 이용하여 링크를 추출한뒤  
          blank구문을 이어붙이면  
          중복 없이 모든 링크에 추가할 수 있다.  
- golang으로 구현  
  ```go  
  func RefineLinkPhrase(row string) string {  
      pattern := `[^!]\[.*?\]\(.+?\)|^\[.*?\]\(.+?\)`  
      patternWithBlank := `[^!]\[.*?\]\(.+?\)(\s*{:target\s*=\s*["']_blank["']\s*})|^\[.*?\]\(.+?\)(\s*{:target\s*=\s*["']_blank["']\s*})`  
      re := regexp.MustCompile(pattern)  
      re2 := regexp.MustCompile(patternWithBlank)  
      isMatched := re.MatchString(row)  
      isMatchedWithBlank := re2.MatchString(row)  
            
      // Remove blank phrases  
      if isMatchedWithBlank {  
          for _, subMatchInfo := range re2.FindAllStringSubmatch(row, -1) {  
              if len(subMatchInfo) < 3 {  
                  continue  
              }  
            
              blankPhrase := subMatchInfo[1]  
              if blankPhrase == "" {  
                  blankPhrase = subMatchInfo[2]  
              }  
            
              row = strings.ReplaceAll(row, blankPhrase, "")  
          }  
      }  
            
      // Add blank phrases  
      if isMatched {  
          refinedRow := row  
            
          for _, submatchedInfo := range re.FindAllStringSubmatch(row, -1) {  
              if len(submatchedInfo) < 1 {  
                  continue  
              }  
              linkPhrase := submatchedInfo[0]  
              refinedRow = strings.Replace(refinedRow, linkPhrase, AddBlankPhrase(linkPhrase), 1)  
          }  
            
          return refinedRow  
      }  
            
      return row  
            
  ```  

## 참고
- [Negative look-ahead in Go regular expressions](https://stackoverflow.com/questions/26771592/negative-look-ahead-in-go-regular-expressions){:target="_blank"}  
- [Lookahead assertion: (?=...), (?!...)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Lookahead_assertion){:target="_blank"}  
