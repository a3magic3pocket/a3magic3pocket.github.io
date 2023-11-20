---
title:  juqery library element에 React component 연동하기(react portal)
date: 2023-11-20 17:38:29 +0900
categories: [react]
tags: [react, next, react-portal]    # TAG names should always be lowercase
---

## 설명
- react로 개발하다보면 react를 지원하지 않는 library(비react library)를   
  사용해야할 경우가 생긴다.   
- 비react library element에 react component를 연동해야할 일이 있다면  
  react portal을 사용하여 해결할 수 있다.  

## react portal이란?
- [공식홈페이지 createPortal]( https://react.dev/reference/react-dom/createPortal){:target="_blank"}  
- react component를 특정 HTML DOM element 하위로 이동시켜주는 기능이다.  
- createPortal() 함수로 포탈노드를 만든 뒤 렌더하면  
  인자로 받은  react component가 인자로 받은 HTML DOM element 하위로 이동한다.  
- createPortal() 함수 인자는 react component, HTML DOM element이다.  

## 예시
- 설명  
    - 과일 목록이 표기된 HTML DOM있다고 가정하자.  
    - 각 과일 항목에 삭제 버튼 react component를 붙이고자 한다.  
- 과일 목록 HTML DOM  
  ```typescript  
  export default function Page() {  
    return (  
      <div>  
        <div>사과</div>  
        <div>배</div>  
        <div>참외</div>  
      </div>  
    );  
  }  
  ```  
- react portal 적용  
  ```typescript  
  'use client';  
  import { useState, useEffect, useRef } from 'react';  
  import { createPortal } from 'react-dom';  
            
  export default function Page() {  
    const fruitsRef = useRef<HTMLDivElement>(null);  
    const [closeButtons, setCloseButtons] = useState<React.ReactPortal[]>([]);  
            
    useEffect(() => {  
      if (fruitsRef.current) {  
        // 과일 HTML DOM element 목록  
        const elems = fruitsRef.current.querySelectorAll('div');  
        const result = [];  
            
        for (const elem of elems) {  
          // 각 과일 HTML DOM element 하위에 삭제 버튼을 이동시키는 포탈 생성  
          const closeButtonElem = createPortal(  
                      
            // 삭제 버튼  
            <button  
              onClick={(e) => {  
                // 과일 삭제 로직  
                const parentElem = (e.target as HTMLElement)  
                  .parentElement as HTMLElement;  
                parentElem.remove();  
              }}  
            >  
              X  
            </button>,  
            
            // 과일 HTML DOM element  
            elem  
          );  
            
          result.push(closeButtonElem);  
        }  
            
        setCloseButtons(result);  
      }  
    }, []);  
            
    return (  
      <>  
        <div ref={fruitsRef}>  
          <div>사과</div>  
          <div>배</div>  
          <div>수박</div>  
        </div>  
            
        {/* 과일 삭제 버튼 포탈 렌더 */}  
        {closeButtons}  
      </>  
    );  
  }  
            
  ```  
- react port 적용 전  
    - <a href='/assets/img/2023-11-20-how-to-use-react-portal/00-origin.jpg' target='_blank'><img src='/assets/img/2023-11-20-how-to-use-react-portal/00-origin.jpg' width='50%' height='50%'></a>  
- react port 적용 후  
    - <a href='/assets/img/2023-11-20-how-to-use-react-portal/01-react-portal.jpg' target='_blank'><img src='/assets/img/2023-11-20-how-to-use-react-portal/01-react-portal.jpg' width='50%' height='50%'></a>  

## 참고
- [createPortal – React](https://react.dev/reference/react-dom/createPortal){:target="_blank"}  
