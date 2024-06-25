---
title: nestjs에서 Google OAuth2.0 인증 구현
date: 2024-06-26 01:25:42 +0900
categories: [Nest]
tags: [nest, passport, oauth2]    # TAG names should always be lowercase
---

## 개요
- nest.js에서 passport-google-oauth20를 이용하여   
  Google OAuth2.0 인증을 구현한다.  
- express-session을 이용하여 session 인증을 구현한다.  

## 의존성 패키지 설치
```bash  
npm install --save passport  
npm install --save passport-google-oauth20  
npm install --save express-session  
npm install -D @types/express-session  
npm install -D @types/passport-google-oauth20  
```  

## Google OAuth 2.0 인증 과정
- 내 앱 로그인 페이지에서 구글 로그인 페이지로 리다이렉트 한다.  
- 구글 로그인 페이지에서 로그인에 성공하면  
  사전에 지정한 "내 앱 로그인 후처리" 페이지로 리다이렉트 한다.  
- 내 앱 로그인 후처리 페이지에서   
  구글 계정의 access_token과 refresh_token을 전달 받고  
  이를 이용하여 요청자의 구글 계정 정보를 얻는다.  
- 구글 계정 정보를 바탕으로 내 앱 DB에서  
  유저 생성 또는 유저 정보 조회를 진행한다.  
- 유저 정보 이용하여 session을 생성한다.  
- 로그인 완료 후 다른 페이지로 리다이렉트한다.  

## passport-google-oauth20
- 구글 로그인 후 access_token과 refresh_token을 발급 받고  
  구글 계정 정보 획득까지 위 라이브러리가 처리해준다.  

## Google Console에서 OAuth 설정하기
- [[참고1 - [NestJS] - 20. 인증과 인가 - OAuth2 Google 소셜 로그인]](https://sjh9708.tistory.com/49){:target="_blank"}에서  
  구글 설정 부분을 참고한다.  
- 구글 로그인 성공 후 리다이렉트 될 URL을 설정한다.  
- clientID와 clientSecret을 얻을 수 있다.  

## Passport와 Guard
- Passport는 인증 미들웨어로 요청의 인증 및 인가를 처리한다.  
- Gaurd는 요청을 핸들링하고, 특정 조건을 만족하지 않는 경우   
  막는 역할을 한다.  
- Passport 전략을 Guard에 연결하여 인증 처리를 할 수 있다.  
- passport-google-oauth20의 Strategy를 이용하여 GoogleStrategy를 만든다.  

## GoogleStrategy
- oauth/oauth-google.guard.ts  
  ```typescript  
  import { Injectable } from "@nestjs/common";  
  import { PassportStrategy } from "@nestjs/passport";  
  import { Profile, Strategy } from "passport-google-oauth20";  
  import { UserRepository } from "@src/user/user.repository";  
  import { IUserCreateDto } from "@src/user/interface/user-create-dto.interface";  
  import { IOAuthUser } from "@src/oauth/interface/oauth-user.interface";  
            
  @Injectable()  
  export class GoogleStrategy extends PassportStrategy(Strategy, "google") {  
    constructor(private userRepository: UserRepository) {  
      // 구글 clientId, clientSecret, callbackURL을 설정한다.  
      super({  
        clientID: process.env.GOOGLE_AUTH_CLIENT_ID,  
        clientSecret: process.env.GOOGLE_AUTH_CLIENT_SECRET,  
        callbackURL: process.env.GOOGLE_AUTH_CALLBACK_URL,  
        scope: ["profile", "email"]  
      });  
    }  
            
    //구글 로그인에 성공하면 실행되는 함수이다.  
    //이미 등록된 회원이면 내 앱 DB에서 조회해서 유저정보를 담고  
    //profile에 구글 계정 정보에 담긴다.  
    //기존 회원이면 내 앱 DB에서 유저 정보 조회 후 유저 정보를 리턴한다.  
    //새로운 회원이면 내 앱 DB에 등록 후 유저 정보를 리턴한다.  
    async validate(accessToken: string, refreshToken: string, profile: Profile) {  
      const email = profile.emails[0].value;  
      const name = email.split("@")[0];  
      let user = await this.userRepository.findByEmail(email);  
      if (user === null) {  
        const defaultDescription = "hello world:)";  
            
        const newUser: IUserCreateDto = {  
          email: email,  
          openId: profile.id,  
          name: name,  
          provider: profile.provider,  
          isActive: true,  
          locale: locale,  
          description: defaultDescription  
        };  
            
        user = await this.userRepository.create(newUser);  
      }  
            
      const oAuthUser: IOAuthUser = {  
        sub: user._id.toString(),  
        openId: user.openId,  
        email: user.email,  
        isActive: user.isActive,  
        name: user.name,  
      };  
            
      return oAuthUser;  
    }  
  }  
  ```  

## GoogleOAuthGuard
- 설명  
    - GoogleStrategy를 활용하는 Guard를 만든다.  
- oauth/oauth-google.strategy.ts  
  ```typescript  
  import { Injectable } from "@nestjs/common";  
  import { ConfigService } from "@nestjs/config";  
  import { AuthGuard } from "@nestjs/passport";  
            
  @Injectable()  
  export class GoogleOAuthGuard extends AuthGuard("google") {  
    constructor(private configService: ConfigService) {  
      super({  
        accessType: "offline",  
      });  
    }  
  }  
  ```  

## OAuthController
- 설명  
    - loginGoogle  
        - 구글 로그인 페이지로 리다이렉트 시키는 컨트롤러  
    - loginGoogleRedirect  
        - 구글 로그인 성공 후 유저 정보를 session에 담는 컨트롤러  
          작업이 끝나면 로그인 성공 페이지로 리다이렉트 한다.  
- login/interface/auth-session.interface.ts  
  ```typescript  
  import { Session } from "express-session";  
  import { IOAuthUser } from "@src/oauth/interface/oauth-user.interface";  
            
  export interface IAuthSession extends Session {  
    user: IOAuthUser;  
    data: Record<string, string | Record<string, string>>;  
  }  
  ```  
- oauth/oauth.controller.ts  
  ```typescript  
  import {  
    Controller,  
    Get,  
    Req,  
    Res,  
    Session,  
    UseGuards,  
  } from "@nestjs/common";  
  import { GoogleOAuthGuard } from "./oauth-google.guard";  
  import { IAuthUserRequest } from "@src/login/interface/auth-user-request-interface";  
  import { IAuthSession } from "@src/login/interface/auth-session.interface";  
  import { Response } from "express";  
            
  @Controller()  
  export class OAuthController {  
            
    // 구글 로그인 페이지로 리다이렉트 시킨다.  
    @Get("/oauth2/authorization/google")  
    @UseGuards(GoogleOAuthGuard)  
    async loginGoogle(@Req() req: Request) {}  
            
    // 구글 로그인 성공 후 유저 정보를 session에 담는다.  
    // 이후 로그인 성공 페이지로 리다이렉트 한다.  
    @Get("/login/oauth2/code/google")  
    @UseGuards(GoogleOAuthGuard)  
    async loginGoogleRedirect(  
      @Req() req: IAuthUserRequest,  
      @Session() session: IAuthSession,  
      @Res() res: Response  
    ) {  
      session.user = req.user;  
      session.data = {};  
            
      res.redirect("/v1/auth/success");  
      return;  
    }  
  }  
  ```  

## LoginGuard
- 설명  
    - session.user 유무를 확인하는 가드이다.  
    - 로그인 하지 않은 경우, 로그인이 필요한 컨트롤러로의 접근을 막는다.  
- login/login.guard.ts  
  ```typescript  
  import {  
    CanActivate,  
    ExecutionContext,  
    Injectable,  
    UnauthorizedException,  
  } from "@nestjs/common";  
  import { Observable } from "rxjs";  
            
  @Injectable()  
  export class LoginGuard implements CanActivate {  
    constructor() {}  
    canActivate(  
      context: ExecutionContext  
    ): boolean | Promise<boolean> | Observable<boolean> {  
      const req = context.switchToHttp().getRequest();  
      const session = req.session;  
            
      if (session.user) {  
        return true;  
      }  
            
      throw new UnauthorizedException("authorization required");  
    }  
  }  
  ```  

## LoginController
- 설명  
    - Google OAuth 로그인에 성공한 경우  
- login/login.controller.ts  
  ```typescript  
  import {  
    Controller,  
    Get,  
    Res,  
    Session,  
    UseGuards,  
  } from "@nestjs/common";  
  import { LoginGuard } from "./login.guard";  
  import { Session as esSession } from "express-session";  
  import { IAuthSession } from "./interface/auth-session.interface";  
  import { Response } from "express";  
            
  @Controller()  
  export class LoginController {  
    @Get("/v1/auth/success")  
    @UseGuards(LoginGuard)  
    async loginSuccess(@Session() session: IAuthSession, @Res() res: Response) {  
      // 로그인 후 작업 ...  
            
      return res.redirect('/main/page');  
    }  
            
    @Get("/v1/auth/logout")  
    @UseGuards(LoginGuard)  
    async logout(@Session() session: esSession, @Res() res: Response) {  
      session.destroy(() => {  
        // 로그아웃 후 작업 ...  
            
        return res.redirect('/main/page');  
      });  
    }  
  }  
  ```  

## OAuthModule
- 설명  
    - OAuthController와 LoginController를 AppModule에 추가하기 위하여  
      OAuthModule을 만든다.  
- oauth/oauth.module.ts  
  ```typescript  
  import { Module } from "@nestjs/common";  
  import { OAuthController } from "./oauth.controller";  
  import { LoginController } from "@src/login/login.controller";  
            
  @Module({  
    controllers: [OAuthController, LoginController],  
  })  
  export class OAuthModule {}  
  ```  

## AppModule
- 설명  
    - OAuthModule을 등록한다.  
- app.module.ts  
  ```typescript  
  import { Module } from "@nestjs/common";  
  import { AppController } from "./app.controller";  
  import { AppService } from "./app.service";  
  import { OAuthModule } from "./oauth/oauth.module";  
  import { GoogleStrategy } from "./oauth/oauth-google.strategy";  
            
  @Module({  
    imports: [  
      OAuthModule,  
    ],  
    controllers: [AppController],  
    providers: [AppService, GoogleStrategy],  
  })  
  export class AppModule {}  
  ```  

## main.js
- 설명  
    - app에 session 설정을 추가한다.  
    - 아래 예시에서는 mongodb에 session을 저장한다.  
- main.js  
  ```typescript  
            
  async function bootstrap() {  
    const port = 8080;  
    const app = await NestFactory.create(AppModule);  
            
    // 세션 설정  
    app.use(  
      session({  
        secret: process.env.AUTH_COOKIE_SECRET,  
        store: MongoStore.create({  
          mongoUrl: process.env.MONGODB_URI,  
          dbName: process.env.MONGDB_DATABASE_NAME,  
          collectionName: process.env.MONGODB_SESSION_COLLECTION_NAME,  
        }),  
        resave: false,  
        saveUninitialized: false,  
        cookie: {  
          maxAge: 6000,  
          secure: process.env.NODE_ENV === "production",  
          httpOnly: true,  
          sameSite: "lax",  
        },  
      })  
    );  
    app.use(passport.initialize());  
    app.use(passport.session());  
    await app.listen(port);  
  }  
  bootstrap();  
  ```  

## 참고
- [참고1 - [NestJS] - 20. 인증과 인가 - OAuth2 Google 소셜 로그인](https://sjh9708.tistory.com/49){:target="_blank"}  
- [참고2 - [NestJS] Authentication using Google OAuth (+ Session)](https://velog.io/@from_numpy/NestJS-Authentication-using-Google-OAuth-Session#-base-model-userentity--socialprovider){:target="_blank"}  
- [참고3 - 올바르게 @nestjs/passport GoogleStrategy 구현하기](https://orangebrother.dev/blog/%08nestjs-google-oauth-passport){:target="_blank"}  
- [참고4 - Nest.js 로 Google 로그인 구현](https://velog.io/@leeseunghee00/Nest-Google-%EB%A1%9C%EA%B7%B8%EC%9D%B8-%EA%B5%AC%ED%98%84){:target="_blank"}  
- [참고5 - nest.js 공식홈페이지 Guards](https://docs.nestjs.com/guards){:target="_blank"}  
- [참고6 - passportjs, passport-google-oauth20](https://www.passportjs.org/packages/passport-google-oauth20/){:target="_blank"}  
