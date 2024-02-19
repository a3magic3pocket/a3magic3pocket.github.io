---
title: multiple sheets 사용 시 tableOverflow 활성화하기(jspreadsheet-ce)
date: 2023-11-20 18:01:59 +0900
categories: [Javascript]
tags: [jspreadsheet-ce]    # TAG names should always be lowercase
---

## 설명
- jspreadsheet-ce에서 sheet를 생성할때 tableOverflow를 지정할 수 있다.  
- tableOverflow를  false로 설정하면 사용자 커서가 화면 밖으로 넘어가도  
  화면이 커서를 따라지 않는다.  
- tableOverflow를 true로 설정하면 사용자 커서가 화면 밖으로 넘어가면  
  자동으로 화면이 커서를 따라 이동한다.  
- 문제는 jspreadsheet-ce에서 sheet를 여러개 사용하면  
  tableOverflow: true가 제대로 동작하지 않는다.  
- fullscreen: false, tableHeight: "100px", tableWidth: "100%"를 함께   
  지정해주면 tableOverflow: true가 동작한다.  
- tableHeight는 반드시 명시적으로 지정해야한다(ex. 100px or 1rem)  

## 예시
```typescript  
import React, { useRef, useEffect } from "react";  
import jspreadsheet from "jspreadsheet-ce";  
import type IJspreadsheet from "jspreadsheet-ce";  
import "jspreadsheet-ce/dist/jspreadsheet.css";  
        
interface IJspreadsheetWrapper extends HTMLDivElement {  
// single sheet  
jspreadsheet: IJspreadsheet.JSpreadsheet;  
// multiple sheets  
jexcel: IJspreadsheet.JSpreadsheet[];  
}  
        
         
export default function App() {  
const jRef = useRef<IJspreadsheetWrapper>(null);  
const defaultOption: IJspreadsheet.JSpreadsheetOptions = {  
  data: [[]],  
  minDimensions: [10, 10],  
  tableOverflow: true,  
  tableHeight: `100px`,  
  tableWidth: "100%",  
  fullscreen: false,  
};  
        
const sheetOptions: IJspreadsheet.TabOptions[] = [  
  {  
    ...defaultOption,  
    sheetName: "sheet1",  
  },  
];  
        
useEffect(() => {  
  useEffect(() => {  
    if (jRef.current && !jRef.current.jexcel) {  
      jspreadsheet.tabs(jRef.current, sheetOptions);  
  }  
}, [sheetOptions]);  
         
return (  
  <div>  
    <div ref={jRef} />  
  </div>  
);  
}  
```  

## 참고
- [The Javascript spreadsheet with React](https://bossanova.uk/jspreadsheet/v4/examples/react){:target="_blank"}  
- [jspreadsheet-ce index.js](https://github.com/jspreadsheet/ce/blob/master/src/index.js){:target="_blank"}  
