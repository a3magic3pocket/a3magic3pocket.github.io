---
title: dynalist 여러 줄 코드 블럭 사용하기(multi line code block)
date: 2023-02-08 22:42:00 +0900
categories: [Dynalist]
tags: [dynalist, memo] # TAG names should always be lowercase
---
## 기본 코드 블록 사용법
- 방법
    - \`\`\`hello world \`\`\`
- 결과
    - ```hello world```
- 하지만 개행을 하면 아래와 같이 bullet 기호가 박혀버린다.
    - 방법
        - \`\`\`  
            hello  
            world  
            \`\`\`  
    - 결과
        - \`\`\`
        - hello
        - world
        - \`\`\`


## multi line code 블록에서 개행
- multi line code 블록에서 개행을 할 때는  
    (윈도우 기준)Ctrl + Shift + Enter를이용하면 된다.
    - 방법
        - \`\`\`  
            hello  
            world  
            \`\`\`  
    - 결과
        - ```
            hello
            world
            ```

## 설명문구로 전환
- 코드를 외부에서 복붙(복사, 붙여넣기)할 때에는  
    여전히 bullet 기호가 박혀버린다.  
    이때는 설명 문구로 코드를 복붙하면 된다.  
- 설명문구로 전환
    - 방법
        - 새로 생긴 불릿기호에서 Shift + Enter를 누르면  
            불릿기호 없이 설명문구로 전환된다.
    - 결과
        - 아래는 설명문구  
            설명1  
            설명2  
            설명3  
- 설명문구로 전환 후 코드 블록으로 복사
    - 설명문구로 전환  
        ```
        hello
        world
        ```


## 참고
- [talk.dynalist.io](https://talk.dynalist.io/t/multi-line-code-blocks/41/61){:target="_blank"}
- Alan이 Mar '21에 적은 글에 나와있다.
    - a) Use shift-enter to add a note. you can use multiple lines in the note.
    - b) Or, use ctrl+shift+enter to add more lines to the item without advancing to the next bullet
    - c) If there’s a way to paste multiple lines into a single bullet, I don’t know it.
