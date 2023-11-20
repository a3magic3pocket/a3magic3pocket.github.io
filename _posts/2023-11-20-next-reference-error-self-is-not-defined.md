---
title: "next.js 빌드 시 ReferenceError: self is not defined 해결"
date: 2023-11-20 18:15:03 +0900
categories: [react]
tags: [next, react]    # TAG names should always be lowercase
---

## 설명
- 컴포넌트를 import 할 때 생길 수 있는 오류이다.  
- import 대상 컴포넌트에 오직 클라이언트에서만 사용할 수 있는 기능이  
  포함된 경우, 사전 렌더링에 실패할때 이 오류가 발생한다.  
- next/dynamic을 사용하여 ssr를 사용하지 않고 import를 하면 된다.  

## 예시
```typescript  
import dynamic from "next/dynamic";  
        
const YourClientOnly= dynamic(  
() =>  
  import("@/component/your-client-only").then(  
    (module) => module.ClientOnly  
  ),  
{ ssr: false }  
);  
```  

## 참고
- [Optimizing: Lazy Loading | Next.js (nextjs.org)](https://nextjs.org/docs/app/building-your-application/optimizing/lazy-loading#skipping-ssr){:target="_blank"}  
- [javascript - Getting errors when rendering a component with dynamic in Next.js - Stack Overflow](https://stackoverflow.com/questions/75370064/getting-errors-when-rendering-a-component-with-dynamic-in-next-js){:target="_blank"}  
