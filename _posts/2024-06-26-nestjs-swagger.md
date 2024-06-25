---
title:  nestjs에서 swagger 이용하기
date: 2024-06-26 01:33:12 +0900
categories: [Nest]
tags: [nest, swagger]    # TAG names should always be lowercase
---

## 개요
- nestjs에서 swagger를 이용한다.  

## 설치
```bash  
npm install -D @nestjs/swagger swagger-ui-express  
```  

## 설정
- 설명  
    - 개발 서버로 실행할 경우  
      http://localhost:8080/api로 접속할 수 있다.  
    - 아래 코드에서 swaggerEndpoint 값을 바꾸면   
      swagger 문서 진입 경로를 바꿀 수 있다.  
- main.js  
  ```typescript  
  import { DocumentBuilder, SwaggerModule } from "@nestjs/swagger";  
            
  async function bootstrap() {  
    const port = 8080;  
    const app = await NestFactory.create(AppModule);  
            
    // swagger 설정  
    const swaggerConfig = new DocumentBuilder()  
      .setTitle("swagger 문서 타이틀")  
      .setDescription("swagger 문서 설명<br /> 설정1")  
      .setVersion("1.0")  
      .build();  
            
    const document = SwaggerModule.createDocument(app, swaggerConfig);  
    const swaggerEndpoint = "api";  
    SwaggerModule.setup(swaggerEndpoint, app, document);  
            
    await app.startAllMicroservices();  
    await app.listen(port);  
  }  
  bootstrap();  
  ```  

## 주요 데코레이터 예시
- @ApiTags("post")  
    - 문서에서 소속될 태그를 설정한다.  
- @ApiOperation({ summary: "포스트 생성" })  
    - API의 summary를 작성한다.  
- @ApiResponse({status: HttpStatus.CREATED, description: "생성 성공"})  
    - API 호출 후 Response를 기록한다.  
- @ApiQuery({name: "hook-post-id", type: "string", description: "관련 포스트 id"})  
    - swagger 문서에서 API 호출 시 Query를 입력 할 수 있게 한다.  
- @ApiParam({ name: "name", type: "string", description: "유저 명" })  
    - swagger 문서에서 API 호출 시 Param을 입력 할 수 있게 한다.  
- @ApiBody({description: "파일 정보 업로드",type: PostCreateBodyForSwaggerDto})  
    - swagger 문서에서 API 호출 시 Body를 입력 할 수 있게 한다.  

## 예시1 - 유저 정보 조회
- 설명  
    - user 조회 컨트롤러의 swagger 문서를 작성해본다.  
    - path param으로 name 인자를 받을 수 있도록 한다.  
- user/user.controller.ts  
  ```typescript  
  @ApiTags("user")  
  @Controller()  
  export class UserController {  
    constructor(private userRepository: UserRepository) {}  
            
    @Get("/v1/my-user-info/:name")  
    @ApiOperation({ summary: "유저 정보 조회" })  
    @ApiParam({ name: "name", type: "string", description: "유저 명" })  
    @ApiResponse({  
      status: HttpStatus.OK,  
      description: "조회 성공",  
    })  
    @UseGuards(LoginGuard)  
    async retrieve(@Param() params: Record<string, string>) {  
      // 유저 정보 조회 로직  
    }  
  }  
  ```  

## 예시2 - 파일 업로드
- 설명  
    - post 생성 컨트롤러의 swagger 문서를 작성해본다.  
    - content-type: multipart/form-data로 파일을 업로드 할 수 있게 한다.  
- post/dto/post-create-body.dto.ts  
  ```typescript  
  import { ApiProperty } from "@nestjs/swagger";  
  import { Expose } from "class-transformer";  
  import { IsNotEmpty, Length } from "class-validator";  
            
  export class PostCreateBodyDto {  
    @ApiProperty({ type: "string", format: "string" })  
    @Length(0, 1000)  
    @IsNotEmpty()  
    @Expose({ name: "description" })  
    description: string;  
  }  
  ```  
- post/dto/post-create-body-for-swagger.dto.ts  
  ```typescript  
  import { ApiProperty } from "@nestjs/swagger";  
  import { IsNotEmpty } from "class-validator";  
  import { PostCreateBodyDto } from "./post-create-body.dto";  
            
  export class PostCreateBodyForSwaggerDto extends PostCreateBodyDto {  
    @ApiProperty({ type: "string", format: "binary" })  
    @IsNotEmpty()  
    file: any;  
  }  
  ```  
- post/post.controller.ts  
  ```typescript  
  @ApiTags("post")  
  @Controller()  
  export class PostController {  
    constructor(  
      private postRepository: PostRepository,  
      private postService: PostService  
    ) {}  
            
            
    @Post("/post")  
    @ApiOperation({ summary: "포스트 생성" })  
    @ApiConsumes("multipart/form-data")  
    @ApiBody({  
      description: "파일 정보 업로드",  
      type: PostCreateBodyForSwaggerDto,  
    })  
    @ApiResponse({  
      status: HttpStatus.CREATED,  
      description: "생성 성공",  
    })  
    @UseGuards(LoginGuard)  
    @UseInterceptors(FilesInterceptor("image"))  
    async createPost(  
      @Session() session: IAuthSession,  
      @UploadedFiles() imageFiles: Array<Express.Multer.File>,  
      @Body() body: PostCreateBodyDto  
    ) {  
      // 포스트 생성 로직  
    }  
  }  
  ```  

## 참고
- [참고1 - nestjs openapi](https://docs.nestjs.com/openapi/introduction){:target="_blank"}  
