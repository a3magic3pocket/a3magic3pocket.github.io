---
title: nestjs에서 mongodb-memory-server 이용하여 테스트 작성하기
date: 2024-06-26 01:31:32 +0900
categories: [Nest]
tags: [nest, mongodb-memory-server]    # TAG names should always be lowercase
---

## 개요
- nestjs에서mongodb-memory-server를 이용하여   
  메모리용 mongodb 인스턴스를 생성하고  
  유닛테스트를 작성한다.  

## 설치
```bash  
npm install -D mongodb-memory-server  
```  

## test 용 mongodb 설정
- test/momory-mongodb-setup.ts  
  ```typescript  
  import { MongoMemoryServer } from "mongodb-memory-server";  
            
  let mongo: MongoMemoryServer;  
            
  export const initTestMongodb = async () => {  
    if (!mongo) {  
      mongo = await MongoMemoryServer.create();  
    }  
  };  
            
  export const getTestMongodbUri = () => {  
    return mongo.getUri();  
  };  
            
  export const disconnectTestMongodb = async () => {  
    if (mongo) {  
      await mongo.stop();  
    }  
  };  
  ```  

## 절대경로 alias 추가
- 설명  
    - .spec.ts 파일에서 /test와 /src에 접근하기 쉽도록  
      절대경로 alias를 추가해준다.  
- tsconfig.json  
  ```json  
  {  
    "compilerOptions": {  
      "baseUrl": "./",  
      "paths": {  
        "@test/*": ["test/*"],  
        "@src/*": ["src/*"]  
      }  
      ...  
    }  
  }  
  ```  
- package.json  
  ```json  
  {  
    ...  
    "jset": {  
      ...  
      "rootDir": "./",  
      "moduleNameMapper": {  
        "@test/(.+)$": "<rootDir>/test/$1",  
        "@src/(.+)$": "<rootDir>/src/$1"  
      }  
    }  
  }  
  ```  

## 테스트 코드 작성
- .spec.ts 파일을 생성하고 테스트 코드를 작성하면 된다.  
  ex) post.repository.ts -> post.repository.spec.ts  
- post/post.repository.spec.ts  
  ```typescript  
  import { PostRepository } from "@src/post/post.repository";  
            
  describe("PostRepository", () => {  
    let postRepository: PostRepository;  
    let newPost: Post;  
            
    // 최초 유닛 테스트 시작 전 한 번만 실행  
    beforeAll(async () => {  
      await initTestMongodb();  
      const testMongodbUri = getTestMongodbUri();  
            
      const module: TestingModule = await Test.createTestingModule({  
        imports: [  
          MongooseModule.forRoot(testMongodbUri),  
          MongooseModule.forFeature([  
            { name: POST_COLLECTION_NAME, schema: PostSchema },  
          ]),  
        ],  
        providers: [PostRepository],  
      }).compile();  
            
      postRepository = module.get<PostRepository>(PostRepository);  
            
      // 새 포스트 생성  
      const userId = "kim";  
      const post = new Post(  
        userId,  
        "hello my-description",  
      );  
      newPost = await postRepository.createPost(post);  
    });  
            
    // 모든 유닛 테스트 종료 후 한 번만 실행  
    afterAll(async () => {  
      await disconnectTestMongodb();  
    });  
            
    it("should be defined", () => {  
      expect(postRepository).toBeDefined();  
    });  
            
    describe("findOne", () => {  
      it("existsById", async () => {  
        expect(newPost).not.toEqual(undefined);  
        const exists = await postRepository.existsById(newPost._id.toString());  
        expect(exists).toEqual(true);  
      });  
    });  
  });  
  ```  

## 테스트 실행
```bash  
# 한 번 테스트 실행  
npm run test  
        
# 저장 시 마다 테스트 실행  
npm run test:watch  
```  

## 참고
- [참고1 - mongodb-memory-server doument](https://nodkz.github.io/mongodb-memory-server/docs/api/classes/mongo-memory-server){:target="_blank"}  
- [참고2 - jest document](https://jestjs.io/docs/configuration){:target="_blank"}  
- [참고3 - nestjs mongodb](https://docs.nestjs.com/techniques/mongodb){:target="_blank"}  
