---
title: nestjs에서 kafkajs 사용하기
date: 2024-06-26 01:30:17 +0900
categories: [Nest]
tags: [nest, kafka]    # TAG names should always be lowercase
---

## 개요
- kafkajs를 사용해서 nestjs에서 kafka를 제어한다.  
- nest.js에서 제안하는 방식은 microservice를 이용하는 것인데  
  kafkajs 공식문서 사용법을 이용하는 방식이 더 직관적이어서  
  이 방법을 선택하여 구현하였다.  
  [[참고1]](https://kafka.js.org/docs/getting-started){:target="_blank"}[[참고2]](https://engschool.tistory.com/214){:target="_blank"}  
- kafkajs는 kafka streams는 지원하지 않는다.  

## kafkajs 구성 개요
- kafka 인스턴스를 생성한다.  
- kafka producer 인스턴스를 생성한다.  
- kafka consumer 인스턴스를 생성하는 factory를 생성한다.  
- factory는 consumerGroupId 마다   
  새 kafka consumer 인스턴스를 생성한다.  

## KafkaConfigService
- 설명  
    - kafka 인스턴스를 생성한다.  
- global/kafka/kafka.config.ts  
  ```typescript  
  import { Injectable } from "@nestjs/common";  
  import { Kafka, logLevel } from "kafkajs";  
            
  @Injectable()  
  export class KafkaConfigService {  
    private kafka: Kafka;  
            
    constructor() {  
      this.kafka = new Kafka({  
        clientId: process.env.CLIENT_ID,  
        brokers: [process.env.KAFKA_CLIENT_BOOTSTRAP_SERVER],  
        logLevel: logLevel.INFO,  
      });  
    }  
            
    getClient(): Kafka {  
      return this.kafka;  
    }  
  }  
  ```  

## KafkaProducerService
- 설명  
    - kafka producer 인스턴스를 생성한다.  
    - acks를 -1로 선언하여 메세지 손실 가능성을 최소화한다.  
- global/kafka/kafka.producer.service.ts  
  ```typescript  
  import { Injectable } from "@nestjs/common";  
  import { KafkaConfigService } from "./kafka.config";  
  import { CompressionTypes } from "kafkajs";  
            
  @Injectable()  
  export class KafkaProducerService {  
    private producer;  
            
    constructor(private kafkaConfigService: KafkaConfigService) {  
      this.producer = this.kafkaConfigService.getClient().producer();  
      this.producer.connect();  
    }  
            
    async sendMessage(  
      topic: string,  
      key: string,  
      message: string,  
      headers: Record<string, string> = {}  
    ) {  
      try {  
        await this.producer.send({  
          topic,  
          messages: [{ key, value: message, headers }],  
          compression: CompressionTypes.GZIP,  
          acks: -1,  
        });  
        console.log(`Message sent to topic ${topic}: ${message}`);  
      } catch (error) {  
        console.error(`Error sending message to Kafka: ${error.message}`);  
      }  
    }  
            
    async disconnect() {  
      await this.producer.disconnect();  
    }  
  }  
  ```  

## kafkaConsumerService
- 설명  
    - kafka consumer 인스턴스를 생성한다.   
    - 최초 인스턴스 초기화 시 consumerGroupId를 인자로 받는다.  
    - autoCommit은 false로 설정한다.  
    - subscribe 함수 실행 후   
      true를 리턴하면 commit을 하고  
      false를 리턴하면 commit 하지 않는다.  
      commit을 하지 않으면 consumer가 다시 해당 메세지를 처리하게 된다.  
    - subscribe 함수의 dependencies 인자에  
      subscribe 함수 실행에 필요한 객체들을 담는다.  
- global/kafka/interface/k-consumer-message.interface.ts  
  ```typescript  
  export interface IKConsumerMessage {  
    key: string;  
    value: Buffer;  
    headers: Record<string, string>;  
    partition: number;  
    offset: number;  
  }  
  ```  
- global/kafka/kafka.consumer.service.ts  
  ```typescript  
  import { Injectable } from "@nestjs/common";  
  import { KafkaConfigService } from "./kafka.config";  
  import { IKConsumerMessage } from "./interface/k-consumer-message.interface";  
            
  @Injectable()  
  export class KafkaConsumerService {  
    private consumer;  
            
    constructor(  
      private kafkaConfigService: KafkaConfigService,  
      private consumerGroupId: string  
    ) {  
      this.consumer = this.kafkaConfigService  
        .getClient()  
        .consumer({ groupId: this.consumerGroupId });  
      this.consumer.connect();  
    }  
            
    async subscribe(  
      topic: string,  
      callback: (  
        message: IKConsumerMessage,  
        dependencies: Record<string, any>  
      ) => Promise<boolean>,  
      dependencies: Record<string, any>  
    ) {  
      await this.consumer.subscribe({ topic, fromBeginning: true });  
      await this.consumer.run({  
        eachMessage: async ({ topic, partition, message }) => {  
          const callbackMessage = {  
            key: message.key ? message.key.toString() : "",  
            value: message.value.toString(),  
            headers: message.headers,  
            partition: partition,  
            offset: message.offset,  
          };  
          const isOk = await callback(callbackMessage, dependencies);  
            
          if (isOk) {  
            this.commitManually({ topic, partition, offset: message.offset });  
          }  
        },  
        autoCommit: false,  
      });  
    }  
            
    private async commitManually({ topic, partition, offset }) {  
      await this.consumer.commitOffsets([{ topic, partition, offset }]);  
    }  
            
    async disconnect() {  
      await this.consumer.disconnect();  
    }  
  }  
  ```  

## KafkaConsumerFactory 
- 설명  
    - consumerGroupId를 입력 받아   
      새 KafkaConsumerService 인스턴스를 생성한다.  
- global/kafka/kafka-consumer.factory.ts  
  ```typescript  
  import { Injectable } from "@nestjs/common";  
  import { KafkaConsumerService } from "./kafka.consumer.service";  
  import { KafkaConfigService } from "./kafka.config";  
  import { CONSUMER_GROUP_ID } from "./kafka-info";  
            
  @Injectable()  
  export class KafkaConsumerFactory {  
    constructor(private kafkaConfigService: KafkaConfigService) {}  
    createKafkaConsumerService(consumerGroupId: string): KafkaConsumerService {  
      if (!Object.values(CONSUMER_GROUP_ID).includes(consumerGroupId)) {  
        throw new Error(`${consumerGroupId} is unsupported consumerGroupId`);  
      }  
            
      return new KafkaConsumerService(this.kafkaConfigService, consumerGroupId);  
    }  
  }  
  ```  

## KafkaModule
- 설명  
    - 선언한 kafka 객체들을 module에 담는다.  
    - 이때 생성하고자 하는 KafkaConsumerService 인스턴스도 선언한다.  
- global/kafka/kafka-info.ts  
  ```typescript  
  export const CONSUMER_GROUP_ID = {  
    NEWS_INFO: "new-info",  
  };  
            
  export const KAFKA_TOPIC = {  
    NEWS_INFO: "new-info",  
  };  
  ```  
- global/kafka.module.ts  
  ```typescript  
  import { Module } from "@nestjs/common";  
  import { KafkaConfigService } from "./kafka.config";  
  import { KafkaProducerService } from "./kafka.producer.service";  
  import { KafkaConsumerFactory } from "./kafka-consumer.factory";  
  import { CONSUMER_GROUP_ID } from "./kafka-info";  
            
  @Module({  
    providers: [  
      KafkaConfigService,  
      KafkaProducerService,  
      KafkaConsumerFactory,  
      {  
        provide: CONSUMER_GROUP_ID.NEWS_INFO,  
        useFactory: (factory: KafkaConsumerFactory) =>  
          factory.createKafkaConsumerService(CONSUMER_GROUP_ID.NEWS_INFO),  
        inject: [KafkaConsumerFactory],  
      },  
    ],  
    exports: [  
      KafkaProducerService,  
      `${CONSUMER_GROUP_ID.NEWS_INFO}`,  
    ], // Export services for injection  
  })  
  export class KafkaModule {}  
  ```  

## NewInfoService
- 설명  
    - KafkaConsumerService를 주입하여 사용한다.  
    - 뉴스 정보를 담아 kafka producer가   
      news-info 토픽에 메세지를 보내면  
      kafka consumer가 news-info 토픽의 메세지를 읽어 저장한다.  
    - 중요 부분만 작성한다.  
- news-info/news-info.service.ts  
  ```typescript  
  @Injectable()  
  export class NewsInfoService {  
    constructor(  
      @Inject(CONSUMER_GROUP_ID.NEWS_INFO)  
      private kafkaConsumerService: KafkaConsumerService,  
      private newsInfoRepository: NewsInfoRepository  
    ) {  
      const kafkaConsumerDependencies = {  
        newsInfoRepository: this.newsInfoRepository,  
      };  
      this.kafkaConsumerService.subscribe(  
        KAFKA_TOPIC.NEWS_INFO,  
        this.consumeNewsInfo,  
        kafkaConsumerDependencies  
      );  
    }  
            
    async createNewsInfo(newsInfoCreateDto: INewsInfoCreateDto) {  
      this.produceNewsInfoCreation(newsInfoCreateDto).catch((e) => {  
        console.error(e);  
      });  
    }  
            
    private async produceNewsInfoCreation(newsInfoCreateDto: INewsInfoCreateDto) {  
      const key = crypto.randomUUID();  
      this.kafkaProducerService.sendMessage(  
        KAFKA_TOPIC.NEWS_INFO,  
        key,  
        JSON.stringify(newsInfoCreateDto)  
      );  
    }  
            
    private async consumeNewsInfoCreation(  
      message: IKConsumerMessage,  
      dependencies: IKNewsInfoCreationConsumerDependency  
    ) {  
      console.log("receive message");  
      const newsInfoCreationDto: INewsInfoCreateDto = JSON.parse(  
        message.value.toString()  
      );  
            
      try {  
        const source = Object.values(newsInfoCreationDto).join("|");  
        const docHash = await getSha256Buffer(source);  
            
        // DB 저장  
        const newNewsInfo = new NewsInfoSchema(  
          new Types.ObjectId(newsInfoCreationDto.ownerId),  
          newsInfoCreationDto.description,  
          Buffer.from(docHash),  
          newsInfoCreationDto.createdAt  
        );  
            
        const newsInfoRepository = dependencies.newsInfoRepository;  
        const newNewsInfoResult: NewsInfoSchema =  
          await newsInfoRepository.createNewsInfo(newNewsInfo);  
            
        return true;  
      } catch (error) {  
        console.error(error);  
            
        // 실패 시 처리  
        // ex) 실패 알람 전송  
            
        return true;  
      }  
    }  
  }  
  ```  
- news/news.module.ts  
  ```typescript  
  @Module({  
    imports: [  
      MongooseModule.forFeature([  
        { name: NEWS_INFO_COLLECTION_NAME, schema: NewsInfoSchema },  
      ]),  
      KafkaModule,  
    ],  
    controllers: [NewsInfoController],  
    providers: [NewsInfoService, NewsInfoRepository],  
    exports: [NewsInfoRepository],  
  })  
  export class NewsInfoModule {}  
            
  ```  

## 참고
- [참고1 - kafkajs Getting Started](https://kafka.js.org/docs/getting-started){:target="_blank"}  
- [참고2 - [Nest.js] Kafkajs](https://engschool.tistory.com/214){:target="_blank"}  
