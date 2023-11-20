---
title: useEffect cleanUp 함수를 이용하여 react state를 순차적으로 제어하기(zustand)
date: 2023-11-20 17:54:37 +0900
categories: [react]
tags: [react, useeffect, cleanup, zustand]    # TAG names should always be lowercase
---

## 설명
- 어떤 react state를 순차적으로 제어하고 싶을때   
  trigger 용 react state를 생성하여 제어할 수 있다.  
  ```typescript  
  const [target, setTarget] = useState("")  
  const [trigger, setTrigger] = useState("wait")  
            
  useEffect(() => {  
      setTrigger("first")  
  }, [])  
            
  useEffect(() => {  
    if (tigger === "first") {  
      setTarget("In first step")  
      setTrigger("second")  
    }  
  }, [target, trigger])  
            
  useEffect(() => {  
    if (tigger === "second") {  
      setTarget(target + "| executed second")  
      setTrigger("wait")  
    }  
  }, [target, trigger])  
  ```  
- 문제는 zustand state를 trigger로 사용할때 발생한다.  
  zustand state를 trigger로 사용하면 두 개의 useEffect가 동시에 실행되기 때문이다.  
- 이로 인하여 두번째 useEffect에서 참조하는 target은   
  첫번째 useEffect에서 변경한 target이 아닌 최초 target이다.  
- 첫 useEffect의 cleanUp 함수에서 setTrigger를 실행시키면  
  이 문제를 해결할 수 있다.  
  ```typescript  
  const [target, setTarget] = useState("")  
  const [trigger, setTrigger] = useState("wait")  
            
  useEffect(() => {  
      setTrigger("first")  
  }, [])  
            
  useEffect(() => {  
    if (tigger === "first") {  
      setTarget("In first step")  
      // #### 변경 시작 ####  
      return () => {  
        setTrigger("second")  
      }  
      // #### 변경 끝 ####  
    }  
  }, [target, trigger])  
            
  useEffect(() => {  
    if (tigger === "second") {  
      setTarget(target + "| executed second")  
      setTrigger("wait")  
    }  
  }, [target, trigger])  
  ```  
- cleanUp 함수는 useEffect에서 변경한 사항을 렌더 후 실행되기 때문에  
  두번째 useEffect에서 첫번째 useEffect가 변경한 target을 참조하게 된다.  

## 예시
- 문제  
    - 간단한 채팅을 구현하고자 한다.  
    - 채팅에는 질문, 답변 타입이 있고   
      타입에 따라 채팅메세지 뒤에 다른 접미어(suffix)가 붙는다.  
    - 최초 질문이 주어져 있고, 사용자가 답변 버튼을 누르면  
      지정된 답변이 표시된다.  
    - 지정된 답변을 표시할 때에 두 개의 useEffect가 사용된다.  
    - 첫번째 useEffect는 채팅메세지를 표시하고  
      두번째 useEffect는 첫번째 useEffect에서   
      표시한 채팅메세지에 접미어를 붙인다.  
- 문제 이미지  
    - <a href='/assets/img/2023-11-20-render-react-state-using-useEffect-cleanup-function/00-init.jpg' target='_blank'><img src='/assets/img/2023-11-20-render-react-state-using-useEffect-cleanup-function/00-init.jpg' width='80%' height='80%'></a>  
- react state를 trigger로 사용하는 경우  
    - 설명  
        - step이 trigger이다.  
        - step을 useState를 이용하여 구현하면  
          첫 useEffect에서 변경한 chats 값을  
          다음 useEffect에서 불러올 수 있다.  
    - 소스코드  
        -  src/app/expt/page.tsx  
          ```typescript  
          "use client";  
                    
          import { useEffect, useState } from "react";  
                    
          type TChatType = "question" | "answer";  
          interface IChat {  
            chatType: TChatType;  
            content: string;  
          }  
          type TChatSuffix = "|질문" | "|답변";  
          type TStep = "wait" | "first" | "second";  
                    
          export default function Page() {  
            const [chats, setChats] = useState<IChat[]>([  
              {  
                chatType: "question",  
                content: "철수야 뭐해|질문",  
              },  
            ]);  
            const [step, setStep] = useState<TStep>("wait");  
                    
            const questionStyle = {  
              backgroundColor: "cyan",  
              border: "1px solid black",  
              padding: "2px",  
            };  
                    
            const answerStyle = {  
              backgroundColor: "yellow",  
              border: "1px solid black",  
              padding: "2px",  
            };  
                    
            useEffect(() => {  
              if (step === "first") {  
                const newChat: IChat = {  
                  chatType: "answer",  
                  content: "응 짱구야 나 밥먹고 있어",  
                };  
                setChats([...chats, newChat]);  
                setStep("second");  
              }  
            }, [step, chats]);  
                    
            useEffect(() => {  
              if (step === "second") {  
                console.log("chats IN second", chats);  
                setChats(  
                  chats.map((chat, i) => {  
                    const suffix = getSuffix(chat.chatType);  
                    if (i === chats.length - 1) {  
                      chat.content = chat.content + suffix;  
                    }  
                    
                    return chat;  
                  })  
                );  
                setStep("wait");  
              }  
            }, [step, chats]);  
                    
            const getSuffix = (chatType: TChatType): TChatSuffix => {  
              if (chatType === "question") {  
                return "|질문";  
              }  
                    
              return "|답변";  
            };  
                    
            return (  
              <>  
                <button  
                  onClick={() => {  
                    setStep("first");  
                  }}  
                >  
                  답변  
                </button>  
                <div>  
                  {chats.map((chat, i) => {  
                    if (chat.chatType === "answer") {  
                      return (  
                        <div style={answerStyle} key={i}>  
                          {chat.content}  
                        </div>  
                      );  
                    }  
                    
                    return (  
                      <div style={questionStyle} key={i}>  
                        {chat.content}  
                      </div>  
                    );  
                  })}  
                </div>  
              </>  
            );  
          }  
                    
                    
          ```  
                    
    - 결과  
        - <a href='/assets/img/2023-11-20-render-react-state-using-useEffect-cleanup-function/01-success.jpg' target='_blank'><img src='/assets/img/2023-11-20-render-react-state-using-useEffect-cleanup-function/01-success.jpg' width='80%' height='80%'></a>  
- zustand state를 trigger로 사용하고 useEffect cleanUp 함수를 사용하지 않은 경우  
    - 설명  
        - step이 trigger이다.  
        - step을 zustand state를 사용하였다.  
        - 첫 useEffect에서 chats 값을 변경하였지만  
          다음 useEffect에서는 여전히 변경 전 chats를 참조하고 있다.  
    - 소스코드  
        - store/expt/expt.tsx  
          ```typescript  
          import { StateCreator } from "zustand";  
                    
          interface IExptState {  
              step: TStep;  
              setStep: (step: TStep) => void;  
            }  
                    
          type TStep = "wait" | "first" | "second";  
                    
          export const createExptSlice: StateCreator<IExptState, [], [], IExptState> = (  
            set  
          ) => ({  
            step: "wait",  
            setStep: (step: TStep) => set((state) => ({ ...state, step: step })),  
          });  
          ```  
        - store/index.tsx  
          ```typescript  
          import { create } from "zustand";  
          import { devtools } from "zustand/middleware";  
          import { createExptSlice } from "./expt/expt";  
                    
          export const useBoundStore = create<IExptState>()(  
            devtools((...args) => ({  
              ...createExptSlice(...args),  
            }))  
          );  
          ```  
        - app/expt/page.tsx  
          ```typescript  
          "use client";  
                    
          import { useBoundStore } from "@/store";  
          import { useEffect, useState } from "react";  
                    
          type TChatType = "question" | "answer";  
          interface IChat {  
            chatType: TChatType;  
            content: string;  
          }  
          type TChatSuffix = "|질문" | "|답변";  
                    
          export default function Page() {  
            const [chats, setChats] = useState<IChat[]>([  
              {  
                chatType: "question",  
                content: "철수야 뭐해|질문",  
              },  
            ]);  
            // ### 변경 시작 ###   
            const { step, setStep } = useBoundStore((state) => ({  
              step: state.step,  
              setStep: state.setStep,  
            }));  
            // ### 변경 끝 ###   
                    
            const questionStyle = {  
              backgroundColor: "cyan",  
              border: "1px solid black",  
              padding: "2px",  
            };  
                    
            const answerStyle = {  
              backgroundColor: "yellow",  
              border: "1px solid black",  
              padding: "2px",  
            };  
                    
            useEffect(() => {  
              if (step === "first") {  
                console.log("chats IN first", chats);  
                const newChat: IChat = {  
                  chatType: "answer",  
                  content: "응 짱구야",  
                };  
                setChats([...chats, newChat]);  
                setStep("second");  
              }  
            }, [step, chats]);  
                    
            useEffect(() => {  
              if (step === "second") {  
                console.log("chats IN second", chats);  
                setChats(  
                  chats.map((chat, i) => {  
                    const suffix = getSuffix(chat.chatType);  
                    if (i === chats.length - 1) {  
                      chat.content = chat.content + suffix;  
                    }  
                    
                    return chat;  
                  })  
                );  
                setStep("wait");  
              }  
            }, [step, chats]);  
                    
            const getSuffix = (chatType: TChatType): TChatSuffix => {  
              if (chatType === "question") {  
                return "|질문";  
              }  
                    
              return "|답변";  
            };  
                    
            return (  
              <>  
                <button  
                  onClick={() => {  
                    setStep("first");  
                  }}  
                >  
                  답변  
                </button>  
                <div>  
                  {chats.map((chat, i) => {  
                    if (chat.chatType === "answer") {  
                      return (  
                        <div style={answerStyle} key={i}>  
                          {chat.content}  
                        </div>  
                      );  
                    }  
                    
                    return (  
                      <div style={questionStyle} key={i}>  
                        {chat.content}  
                      </div>  
                    );  
                  })}  
                </div>  
              </>  
            );  
          }  
                    
                    
          ```  
    - 결과  
        - <a href='/assets/img/2023-11-20-render-react-state-using-useEffect-cleanup-function/02-fail.jpg' target='_blank'><img src='/assets/img/2023-11-20-render-react-state-using-useEffect-cleanup-function/02-fail.jpg' width='80%' height='80%'></a>  
- zustand state를 trigger로 사용하고 useEffect cleanUp 함수 사용한 경우  
    - 설명  
        - step이 trigger이다.  
        - step을 zustand state를 사용하였다.  
        - 첫 useEffect 실행 후 cleanUp 함수에서 step을 second으로 변경한다.  
        - 첫 useEffect에서 변경한 chats 값을  
          다음 useEffect에서 불러올 수 있다.  
    - 소스코드  
        - store/expt/expt.tsx  
            - 위 예시와 동일  
        - store/index.tsx  
            - 위 예시와 동일  
        - app/expt/page.tsx  
          ```typescript  
          "use client";  
                    
          import { useBoundStore } from "@/store";  
          import { useEffect, useState } from "react";  
                    
          type TChatType = "question" | "answer";  
          interface IChat {  
            chatType: TChatType;  
            content: string;  
          }  
          type TChatSuffix = "|질문" | "|답변";  
                    
          export default function Page() {  
            const [chats, setChats] = useState<IChat[]>([  
              {  
                chatType: "question",  
                content: "철수야 뭐해|질문",  
              },  
            ]);  
            const { step, setStep } = useBoundStore((state) => ({  
              step: state.step,  
              setStep: state.setStep,  
            }));  
                    
            const questionStyle = {  
              backgroundColor: "cyan",  
              border: "1px solid black",  
              padding: "2px",  
            };  
                    
            const answerStyle = {  
              backgroundColor: "yellow",  
              border: "1px solid black",  
              padding: "2px",  
            };  
                    
            useEffect(() => {  
              if (step === "first") {  
                console.log("chats IN first", chats);  
                const newChat: IChat = {  
                  chatType: "answer",  
                  content: "응 짱구야",  
                };  
                setChats([...chats, newChat]);  
                // ### 변경 시작 ###   
                return () => {  
                  setStep("second");  
                };  
                // ### 변경 끝 ###   
              }  
            }, [step, chats]);  
                    
            useEffect(() => {  
              if (step === "second") {  
                console.log("chats IN second", chats);  
                setChats(  
                  chats.map((chat, i) => {  
                    const suffix = getSuffix(chat.chatType);  
                    if (i === chats.length - 1) {  
                      chat.content = chat.content + suffix;  
                    }  
                    
                    return chat;  
                  })  
                );  
                setStep("wait");  
              }  
            }, [step, chats]);  
                    
            const getSuffix = (chatType: TChatType): TChatSuffix => {  
              if (chatType === "question") {  
                return "|질문";  
              }  
                    
              return "|답변";  
            };  
                    
            return (  
              <>  
                <button  
                  onClick={() => {  
                    setStep("first");  
                  }}  
                >  
                  답변  
                </button>  
                <div>  
                  {chats.map((chat, i) => {  
                    if (chat.chatType === "answer") {  
                      return (  
                        <div style={answerStyle} key={i}>  
                          {chat.content}  
                        </div>  
                      );  
                    }  
                    
                    return (  
                      <div style={questionStyle} key={i}>  
                        {chat.content}  
                      </div>  
                    );  
                  })}  
                </div>  
              </>  
            );  
          }  
                    
          ```  
    - 결과  
        - <a href='/assets/img/2023-11-20-render-react-state-using-useEffect-cleanup-function/01-success.jpg' target='_blank'><img src='/assets/img/2023-11-20-render-react-state-using-useEffect-cleanup-function/01-success.jpg' width='80%' height='80%'></a>  

## 덧붙이는 말
- 첫 useEffect 실행결과를 렌더시킨 후   
  다음 useEffect를 실행시키면 해결되기 때문에  
  useEffect cleanUp 대신 setTimeout을 사용하여 문제를 해결할 수도 있다.  

## 참고
- [useEffect – React](https://react.dev/reference/react/useEffect){:target="_blank"}  
- [Zustand Documentation (pmnd.rs)](https://docs.pmnd.rs/zustand/guides/slices-pattern){:target="_blank"}  
