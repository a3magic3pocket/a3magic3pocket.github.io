---
title: nestjs에서 custom validator 구현
date: 2024-06-26 01:28:51 +0900
categories: [Nest]
tags: [nest, class-validator]    # TAG names should always be lowercase
---

## 개요
- class-validator를 이용하여 custom validator를 생성하고 적용한다.  

## class-validator란?
- 유효성 검사 라이브러리로 클래스 기반의 객체 유효성을   
  간편하게 검사할 수 있다.  
- 클래스의 프로퍼티 위에 데코레이터를 선언하여   
  유효성 검사 규칙을 적용한다.  
- [[참고1 - 데코레이터 목록]](https://github.com/typestack/class-validator?tab=readme-ov-file#validation-decorators){:target="_blank"}  

## 설치
```bash  
npm install --save class-validator  
```  

## class-validator 예시
- 설명  
    - post 목록을 조회하는 Controller를 만들고  
      Query에 담긴 page-num과 page-size의 유효성 검사를 하고 싶다.  
    - 중요 부분만 작성한다.  
- post/dto/post-list-query.dto.ts  
  ```typescript  
  import { IsNumberString, IsOptional } from "class-validator";  
            
  export class PostListQueryDto{  
    // 숫자가 담긴 문자열  
    @IsNumberString()  
    // 필수값이 아님  
    @IsOptional()   
    // Query 필드명이 page-num  
    @Expose({ name: "page-num" })   
    pageNum: number;  
            
    // 숫자가 담긴 문자열  
    @IsNumberString()  
    // 필수값이 아님  
    @IsOptional()  
    // Query 필드명이 page-size  
    @Expose({ name: "page-size" })  
    pageSize: number;  
  }  
  ```  
- post/post.controller.ts  
  ```typescript  
   @Controller()  
  export class PostController {  
    constructor(  
      private postService: PostService  
    ) {}  
            
    @Get("/v1/posts")  
    async listPosts(@Query() query: PostListQueryDto) {  
      const pageNum = query.pageNum ? query.pageNum : 1;  
      const pageSize = query.pageSize ? query.pageSize : 9;  
      const posts = await this.postService.listPosts(pageNum, pageSize);  
            
      return posts;  
    }  
  ```  

## Custom validator
- 설명  
    - 유효성 규칙이 담긴 ValidatorConstraint를 생성하고  
      @Validate 데코레이터로 적용한다.  
    - 자연수인지 확인하는 custom validator를 생성해보자  
- global/validation/is-natural-validation.ts  
  ```typescript  
  import {  
    ValidationArguments,  
    ValidatorConstraint,  
    ValidatorConstraintInterface,  
  } from "class-validator";  
            
  @ValidatorConstraint({ name: "isNaturalNumber", async: false })  
  export class IsNaturalNumber implements ValidatorConstraintInterface {  
            
    // 유효성 검사 규칙, boolean만 리턴할 수 있고 false면 에러가 나게 된다.  
    validate(  
      value: any,  
      validationArguments?: ValidationArguments  
    ): boolean | Promise<boolean> {  
      return /^[1-9][0-9]*/.test(value);  
    }  
            
    // 유효성 검사 통과 실패 시 리턴되는 에러 메세지를 정의한다.  
    defaultMessage(validationArguments?: ValidationArguments): string {  
      const fieldName = validationArguments ? validationArguments.property : "it";  
      return `${fieldName} is not natural value`;  
    }  
  }  
  ```  
- post/dto/post-list-query.dto.ts  
  ```typescript  
  import { IsNumberString, IsOptional, Validate } from "class-validator";  
            
  export class PostListQueryDto{  
   // 자연수 인지 확인  
   @Validate(IsNaturalNumber)  
   @IsNumberString()  
   @IsOptional()   
   @Expose({ name: "page-num" })   
   pageNum: number;  
            
   // 자연수 인지 확인  
   @Validate(IsNaturalNumber)  
   @IsNumberString()  
   @IsOptional()  
   @Expose({ name: "page-size" })  
   pageSize: number;  
  }  
  ```  

## class-validator 전역 오류 메세지 수정하기
- 설명  
    - class-validator 오류 메세지 자체를 수정할 수 있다.  
    - 새 ValidationPipe에서 exceptionFactory를 수정하고  
      이를 globalPipes에 추가하여 적용한다.  
- main.js  
  ```typescript  
  import { BadRequestException, ValidationPipe } from "@nestjs/common";  
  import { ValidationError } from "class-validator";  
            
  async function bootstrap() {  
    const port = 8080;  
    const app = await NestFactory.create(AppModule);  
            
    // 유효성 검증  
    app.useGlobalPipes(  
      new ValidationPipe({  
        transform: true,  
        transformOptions: {  
          enableImplicitConversion: false,  
        },  
        exceptionFactory: (validationErrors: ValidationError[] = []) => {  
          const errorInfos = validationErrors.map((error) => ({  
            field: error.property,  
            error: Object.values(error.constraints),  
          }));  
            
          const errorRespDto = {  
            description: "my-custom-error description",  
            message: errorInfos,  
          };  
            
          return new BadRequestException(errorRespDto);  
        },  
      })  
    await app.listen(port);  
  }  
  bootstrap();  
            
  ```  

## 참고
- [참고1 - 데코레이터 목록](https://github.com/typestack/class-validator?tab=readme-ov-file#validation-decorators){:target="_blank"}  
- [참고2 - Custom validation classes](https://github.com/typestack/class-validator#custom-validation-classes){:target="_blank"}  
