---
title: solana DApp 인증 구현
date: 2024-02-19 10:48:24 +0900
categories: [React]
tags: [solana, next, react]    # TAG names should always be lowercase
---

## 개요
- solana DAPP에서 지갑인증을 구현해본다.  
- 지갑으로 인증된 유저만 특정 페이지를   
  조회할 수 있도록 처리할 수 있다.  
- [How to Integrate Sign-in Authentication with a Solana Wallet](https://www.quicknode.com/guides/solana-development/dapps/how-to-authenticate-users-with-a-solana-wallet){:target="_blank"}을 그대로 따라 진행한다.  
- 중요한 부분만 설명을 추가한다.  

## 환경
- next.js로 DApp을 구현하였다.  
- 인증 시 사용하는 api는  next.js의 API 기능을 사용한다.  
- NextAuth.js를 이용하여 인증을 구현한다.  

## 인증 원리
- 지갑의  전자서명(signMessage(data)) 기능을 이용한다.  
- 지갑에는 유저의 공개키와 비밀키가 저장되어있다.  
- 지갑에 저장되어 있는 비밀키로 메세지를 암호화한다.  
  이를 서명(signature)이라고 한다.  
- 유저는 (메세지, 서명, 공개키)를 인증 서버에 전달한다.  
- 인증서버는 서명을 공개키로 복호화한다.  
- 복호화된 메세지가 전달 받은 메세지와 일치하면  
  인증을 요청한 지갑이 전달 받은 공개키 지갑이라는 것을 확인할 수 있다.  

## SigninMessage.ts
- 설명  
    - 인증 시 필요한 정보를 담고 있는 클래스이다.  
- 프로퍼티 설명  
    - domain  
        - 인증 요청 URL, 허가된 곳에서 요청했는지 확인할때 사용한다.  
    - publicKey  
        - 지갑의 공개키  
    - nonce:  
        - 랜덤 문자열로 메세지가 중복되는 것을 막는다.  
    - statement  
        - 인증 시 사용하는 문자열  
          아무 문자나 적어도 된다.  
- prepare 메서드 설명  
    - statement와 nonce를 이어붙여 메세지를 만든다.  
- validate 메서드 설명  
    - 메세지, 서명, 공개키를 이용하여 공개키 주인이 보낸 메세지인지 확인한다.  
    - [nacl.sign.detached.verify 함수 설명](https://github.com/dchest/tweetnacl-js/blob/master/README.md#naclsigndetachedverifymessage-signature-publickey){:target="_blank"}  
- 코드  
  ```typescript  
            
  /* Reference by : https://www.quicknode.com/guides/solana-development/dapps/how-to-authenticate-users-with-a-solana-wallet#create-a-sign-in-message-class */  
            
  import bs58 from "bs58";  
  import nacl from "tweetnacl";  
  type SignMessage = {  
    domain: string;  
    publicKey: string;  
    nonce: string;  
    statement: string;  
  };  
            
  export class SigninMessage {  
    domain: any;  
    publicKey: any;  
    nonce: any;  
    statement: any;  
            
    constructor({ domain, publicKey, nonce, statement }: SignMessage) {  
      this.domain = domain;  
      this.publicKey = publicKey;  
      this.nonce = nonce;  
      this.statement = statement;  
    }  
            
    prepare() {  
      return `${this.statement}${this.nonce}`;  
    }  
            
    async validate(signature: string) {  
      const msg = this.prepare();  
      const signatureUint8 = bs58.decode(signature);  
      const msgUint8 = new TextEncoder().encode(msg);  
      const pubKeyUint8 = bs58.decode(this.publicKey);  
            
      return nacl.sign.detached.verify(msgUint8, signatureUint8, pubKeyUint8);  
    }  
            
  }  
            
  ```  

## Solana Wallet Adapter 추가
- 설명  
    - next.js 앱에서 SessionProvider를 추가한다.  
    - 나머지는 solana 지갑관련된 것으로  
      기호에 따라 변경할 수 있다.  
      (예시에서는 PhantomWallet만 지원하고 있다.)  
- 코드  
  ```typescript   
            
  /* Reference by: https://www.quicknode.com/guides/solana-development/dapps/how-to-authenticate-users-with-a-solana-wallet#add-the-solana-wallet-adapter */  
            
  export default function App({ Component, pageProps }: AppProps) {  
    const network = WalletAdapterNetwork.Devnet;  
    const endpoint = useMemo(() => clusterApiUrl(network), [network]);  
              
    const wallets = useMemo(  
      () => [  
        new PhantomWalletAdapter(),  
      ],  
      []  
    );  
            
    return (  
      <ConnectionProvider endpoint={endpoint}>  
        <WalletProvider wallets={wallets} autoConnect>  
          <WalletModalProvider>  
            // 추가 시작  
            <SessionProvider session={pageProps.session} refetchInterval={0}>  
            // 추가 끝  
              <Component {...pageProps} />  
            // 추가 시작  
            </SessionProvider>  
            // 추가 끝  
          </WalletModalProvider>  
        </WalletProvider>  
      </ConnectionProvider>  
    );  
  }  
            
  ```  

## 프론트엔드
- 설명  
    - 인증 요청 버튼을 클릭하면  
      domain, publicKey, statement, nonce를 이용하여  
      SigninMessage 인스턴스를 생성한다.  
    - nonce는 csrf 토큰으로 랜덤하다.  
    - SingingMessage 인스턴스에서 prepare 메서드를 실행하여  
      메시지를 만든다.  
    - wallet.signMessage(메세지)를 실행하여 서명을 만든다.  
    - SigningMessage 인스턴스를 json으로 바꾸고, 서명과 함께  
      인증 서버로 요청한다.  
- 코드  
  ```typescript  
            
  /* Reference by: https://www.quicknode.com/guides/solana-development/dapps/how-to-authenticate-users-with-a-solana-wallet#update-front-end */  
            
  import Link from "next/link";  
  import { getCsrfToken, signIn, signOut, useSession } from "next-auth/react";  
  import styles from "./header.module.css";  
  import { useWalletModal } from "@solana/wallet-adapter-react-ui";  
  import { useWallet } from "@solana/wallet-adapter-react";  
  import { SigninMessage } from "../utils/SigninMessage";  
  import bs58 from "bs58";  
  import { useEffect } from "react";  
            
  export default function Header() {  
    const { data: session, status } = useSession();  
    const loading = status === "loading";  
            
    const wallet = useWallet();  
    const walletModal = useWalletModal();  
            
    const handleSignIn = async () => {  
      try {  
        if (!wallet.connected) {  
          walletModal.setVisible(true);  
        }  
            
        // csrf 토큰 생성  
        const csrf = await getCsrfToken();  
        if (!wallet.publicKey || !csrf || !wallet.signMessage) return;  
            
        // SigninMessage 인스턴스 생성  
        const message = new SigninMessage({  
          domain: window.location.host,  
          publicKey: wallet.publicKey?.toBase58(),  
          statement: `Sign this message to sign in to the app.`,  
          nonce: csrf,  
        });  
            
        // 서명 생성  
        const data = new TextEncoder().encode(message.prepare());  
        const signature = await wallet.signMessage(data);  
        const serializedSignature = bs58.encode(signature);  
            
        // 인증서버로 요청  
        signIn("credentials", {  
          message: JSON.stringify(message),  
          redirect: false,  
          signature: serializedSignature,  
        });  
      } catch (error) {  
        console.log(error);  
      }  
    };  
            
    useEffect(() => {  
      if (wallet.connected && status === "unauthenticated") {  
        handleSignIn();  
      }  
    }, [wallet.connected]);  
            
    return (  
      <header>  
        <noscript>  
          <style>{`.nojs-show { opacity: 1; top: 0; }`}</style>  
        </noscript>  
        <div className={styles.signedInStatus}>  
          <p  
            className={`nojs-show ${  
              !session && loading ? styles.loading : styles.loaded  
            }`}  
          >  
            {!session && (  
              <>  
                <span className={styles.notSignedInText}>  
                  You are not signed in  
                </span>  
                <span className={styles.buttonPrimary} onClick={handleSignIn}>  
                  Sign in  
                </span>  
              </>  
            )}  
            {session?.user && (  
              <>  
                {session.user.image && (  
                  <span  
                    {% raw %}style={{ backgroundImage: `url('${session.user.image}')`}}{% endraw %}  
                    className={styles.avatar}  
                  />  
                )}  
                <span className={styles.signedInText}>  
                  <small>Signed in as</small>  
                  <br />  
                  <strong>{session.user.email ?? session.user.name}</strong>  
                </span>  
                <a  
                  href={`/api/auth/signout`}  
                  className={styles.button}  
                  onClick={(e) => {  
                    e.preventDefault();  
                    signOut();  
                  }}  
                >  
                  Sign out  
                </a>  
              </>  
            )}  
          </p>  
        </div>  
        <nav>  
          <ul className={styles.navItems}>  
            <li className={styles.navItem}>  
              <Link legacyBehavior href="/">  
                <a>Home</a>  
              </Link>  
            </li>  
            <li className={styles.navItem}>  
              <Link legacyBehavior href="/api/examples/protected">  
                <a>Protected API Route</a>  
              </Link>  
            </li>  
            <li className={styles.navItem}>  
              <Link legacyBehavior href="/me">  
                <a>Me</a>  
              </Link>  
            </li>  
          </ul>  
        </nav>  
      </header>  
    );  
  }  
            
  ```  

## 백엔드
- 설명  
    - 프론트엔드에서 인증요청을 받아서  
      인증여부를 처리하는 부분이다.  
    - 유저가 전달한 (메세지, 서명, 공개키)를 이용하여  
      인증 요청자 지갑 소유 여부를 확인한다.  
    - 인증에 성공하면 jwt를 세션에 담아 응답을 보낸다.  
      [세션 생성 함수 NextAuth(req, res, configuration) 문서](https://next-auth.js.org/configuration/options#options){:target="_blank"}  
- 코드  
  ```typescript  
            
  /* Reference by: https://www.quicknode.com/guides/solana-development/dapps/how-to-authenticate-users-with-a-solana-wallet#implement-backend-api */  
            
  import { NextApiRequest, NextApiResponse } from "next";  
  import NextAuth from "next-auth";  
  import CredentialsProvider from "next-auth/providers/credentials";  
  import { getCsrfToken } from "next-auth/react";  
  import { SigninMessage } from "../../../utils/SigninMessage";  
            
  export default async function auth(req: NextApiRequest, res: NextApiResponse) {  
    const providers = [  
      CredentialsProvider({  
        name: "Solana",  
        credentials: {  
          message: {  
            label: "Message",  
            type: "text",  
          },  
          signature: {  
            label: "Signature",  
            type: "text",  
          },  
        },  
        async authorize(credentials, req) {  
          try {  
            const signinMessage = new SigninMessage(  
              JSON.parse(credentials?.message || "{}")  
            );  
            
            // domain 유효성 확인  
            const nextAuthUrl = new URL(process.env.NEXTAUTH_URL);  
            if (signinMessage.domain !== nextAuthUrl.host) {  
              return null;  
            }  
            
            // csrf 토큰 유효성 확인  
            const csrfToken = await getCsrfToken({ req: { ...req, body: null } });  
            
            if (signinMessage.nonce !== csrfToken) {  
              return null;  
            }  
            
            // 서명 유효성 확인  
            const validationResult = await signinMessage.validate(  
              credentials?.signature || ""  
            );  
            
            if (!validationResult)  
              throw new Error("Could not validate the signed message");  
            
            return {  
              id: signinMessage.publicKey,  
            };  
          } catch (e) {  
            return null;  
          }  
        },  
      }),  
    ];  
            
    const isDefaultSigninPage =  
      req.method === "GET" && req.query.nextauth?.includes("signin");  
            
    // Hides Sign-In with Solana from the default sign page  
    if (isDefaultSigninPage) {  
      providers.pop();  
    }  
            
    // 세션 생성  
    return await NextAuth(req, res, {  
      providers,  
      session: {  
        strategy: "jwt",  
      },  
      secret: process.env.NEXTAUTH_SECRET,  
      callbacks: {  
        async session({ session, token }) {  
          // @ts-ignore  
          session.publicKey = token.sub;  
          if (session.user) {  
            session.user.name = token.sub;  
            session.user.image = `https://ui-avatars.com/api/?name=${token.sub}&background=random`;  
          }  
          return session;  
        },  
      },  
    });  
  }  
            
  ```  

## 참고
- [How to Integrate Sign-in Authentication with a Solana Wallet](https://www.quicknode.com/guides/solana-development/dapps/how-to-authenticate-users-with-a-solana-wallet#overview){:target="_blank"}  
- [tweetnacl 공식문서](https://github.com/dchest/tweetnacl-js/blob/master/README.md#naclsigndetachedverifymessage-signature-publickey){:target="_blank"}  
- [NextAuth.js 공식문서](https://next-auth.js.org/configuration/options#options){:target="_blank"}  
- [전자서명 위키백과](https://ko.wikipedia.org/wiki/%EC%A0%84%EC%9E%90%EC%84%9C%EB%AA%85){:target="_blank"}  
