---
title: Spring Kafka 컨슈머 수동 커밋 설정
date: 2025-04-11 22:32:36 +0900
categories: [Kafka]
tags: [kafka, spring, kotlin]    # TAG names should always be lowercase
---

## 개요
- 카프카 기본 설정은 오토 커밋이다.  
- 이 경우, 컨슈머가 메시지를 실제로 처리했는지와 무관하게,   
  일정 시간이 지나면 자동으로 오프셋이 커밋된다.  
- 그 결과 메세지가 유실될 수 있다.  
- 이를 방지하기 위해 수동 커밋을 설정한다.  

## 설정
- application.yml  
  ```yml  
  spring:  
    kafka:  
      bootstrap-servers: "kafka:9092"  
      producer:  
        key-serializer: org.apache.kafka.common.serialization.StringSerializer  
        value-serializer: org.springframework.kafka.support.serializer.JsonSerializer  
      consumer:  
        auto-offset-reset: latest  
        # |--추가 시작--|  
        enable-auto-commit: false  
        # |--추가 끝--|  
        key-deserializer: org.apache.kafka.common.serialization.StringDeserializer  
        value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer  
        properties:  
          spring.json.trusted.packages: "*"  
  ```  
- KafkaConfig  
  ```kotlin  
  package com.example.demo.config  
            
  import org.springframework.context.annotation.Bean  
  import org.springframework.context.annotation.Configuration  
  import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory  
  import org.springframework.kafka.core.ConsumerFactory  
  import org.springframework.kafka.listener.ContainerProperties  
            
  @Configuration  
  class KafkaConfig {  
            
      @Bean(name = ["kafkaListenerContainerFactory"])  
      fun kafkaListenerContainerFactory(  
          consumerFactory: ConsumerFactory<String, String>  
      ): ConcurrentKafkaListenerContainerFactory<String, String> {  
          val factory = ConcurrentKafkaListenerContainerFactory<String, String>()  
          factory.consumerFactory = consumerFactory  
           // AckMode.MANUAL로 설정해야 수동커밋이 활성화된다.  
          factory.containerProperties.ackMode = ContainerProperties.AckMode.MANUAL  
          return factory  
      }  
  }  
  ```  
- 프로듀서  
  ```kotlin  
  package com.example.demo.kafka.producer  
            
  import com.example.demo.constant.KafkaTopic  
  import org.springframework.kafka.core.KafkaTemplate  
  import org.springframework.stereotype.Service  
            
            
  @Service  
  class UserProducer(  
      private val kafkaTemplate: KafkaTemplate<String, String>  
  ) {  
            
      fun send(message: String) {  
          kafkaTemplate.send(KafkaTopic.USER, message)  
      }  
            
  }  
  ```  
- 컨슈머  
  ```kotlin  
  @Component  
  class UserConsumer {  
            
      @KafkaListener(  
          topics = [KafkaTopic.USER],  
          groupId = "user",  
          // 👇 AckMode.MANUAL 로 설정한 카프카 컨테이너 팩토리 추가  
          containerFactory = "kafkaListenerContainerFactory"  
      )  
      fun consume(record: ConsumerRecord<String, String>, acknowledgment: Acknowledgment) {  
          Thread.sleep(2000)  
          println("record++" + record)  
            
          // 수동 커밋  
          acknowledgment.acknowledge()  
      }  
  }  
  ```  
- 컨트롤러  
  ```kotlin  
  package com.example.demo.controller  
            
  import com.example.demo.kafka.producer.UserProducer  
  import io.swagger.v3.oas.annotations.Operation  
  import org.springframework.http.HttpStatus  
  import org.springframework.http.MediaType  
  import org.springframework.http.ResponseEntity  
  import org.springframework.web.bind.annotation.GetMapping  
  import org.springframework.web.bind.annotation.ResponseStatus  
  import org.springframework.web.bind.annotation.RestController  
  import java.io.IOException  
            
  @RestController  
  class UserController(  
      private val userProducer: UserProducer,  
  ) {  
            
      @Operation(  
          summary = "user 메세지 생성 및 처리",  
          description = "user 메세지 생성 및 처리",  
          tags = ["user"]  
      )  
      @ResponseStatus(HttpStatus.OK)  
      @Throws(IOException::class)  
      @GetMapping(value = ["/user"], produces = [MediaType.APPLICATION_JSON_VALUE])  
      fun user(): ResponseEntity<String> {  
            
          userProducer.send("hello world message")  
            
          return ResponseEntity.ok().body(  
              "success"  
          )  
      }  
  }  
            
  ```  
- 카프카 메세지 생성 및 소비 명령  
  ```bash  
  curl http://localhost:8080/user  
  ```  

## 의문1 - 실패 시는 어떻게 되는가?
- acknowledgment.acknowledge() 실행 전에 실패하면  
  오프셋이 갱신되지 않으면서   
  컨슈머 다음 폴링 시에도 같은 데이터를 가져온다.  
- 에러 핸들링을 설정하지 않는다면  
  acknowledgment.acknowledge() 이 실행되기 전까지  
  이 과정이 반복된다.  

## 에러 핸들러 추가
- 설명  
    - 1초 간격으로 3회 재시도 후 실패하면 다음 offset으로 넘어가게 한다.  
- KafkaConfig  
  ```kotlin  
  @Configuration  
  class KafkaConfig {  
                
      // |-- 추가 시작 --|  
      @Bean  
      fun errorHandler(): DefaultErrorHandler {  
          val backOff = FixedBackOff(1000L, 3L) // 1초 간격, 3회 재시도  
          return DefaultErrorHandler(backOff)  
      }  
      // |-- 추가 끝 --|  
            
      @Bean(name = ["kafkaListenerContainerFactory"])  
      fun kafkaListenerContainerFactory(  
          consumerFactory: ConsumerFactory<String, String>,  
          errorHandler: DefaultErrorHandler  
      ): ConcurrentKafkaListenerContainerFactory<String, String> {  
          val factory = ConcurrentKafkaListenerContainerFactory<String, String>()  
          factory.consumerFactory = consumerFactory  
          factory.containerProperties.ackMode = ContainerProperties.AckMode.MANUAL  
            
          // |-- 추가 시작 --|  
          factory.setCommonErrorHandler(errorHandler)  
          // |-- 추가 끝 --|  
            
          return factory  
      }  
  }  
  ```  
- 결과  
    - 컨슈머에서 acknowledgment.acknowledge() 실행 전에 에러 발생시키고  
      메세지를 생성해보면 3회 재시도 후 무시되는 것을 확인할 수 있다.  
    - 컨슈머 그룹 상태 조회  
      ```text  
      GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                           HOST            CLIENT-ID  
      user            user            2          26              26              0               consumer-user-9-09053eeb-9472-40fc-b866-752da9ffa02e  /172.18.0.1     consumer-user-9  
      user            user            1          80              80              0               consumer-user-8-e8a1989e-c7ae-487d-95af-1eddb6ead117  /172.18.0.1     consumer-user-8  
      user            user            0          34              34              0               consumer-user-10-c633f4c2-7af0-4fe9-ac2a-611f9ded2980 /172.18.0.1     consumer-user-10  
      ```  
    - 실험 대상 파티션은 1번으로 CURRENT-OFFSET과 LOG-END-OFFSET이   
      동일해진 것을 확인할 수 있다.  

## Dead Letter Topic(DLT)로 보내기
- 설명  
    - DLT는 소비자가 처리하지 못한 메세지를 따로 모아두는 용도의 토픽이다.  
    - 컨슈머 메세지가 계속 실패하거나, 예외가 발생해도 처리할 수 없는 상황일때  
      해당 메세지를 DLT로 옮겨서 별도로 처리할 수 있도록 할 수 있다.  
    - 예시에서는 1초 마다 3회 재시도 후 실패하면 user.DLT로 보낸다.  
- user.DLT 토픽 생성  
  ```bash  
  kafka-topics.sh --create \  
   --topic user.DLT \  
   --bootstrap-server kafka:9092 \  
   --partitions 3 \  
   --replication-factor 1  
  ```  
- KafkaConfig  
  ```kotlin  
  @Configuration  
  class KafkaConfig {  
            
      @Bean  
      fun errorHandler(kafkaTemplate: KafkaTemplate<String, String>): DefaultErrorHandler {  
          // |-- 추가 시작 --|  
          // DLT로 메시지를 보낼 Recoverer 설정  
          val recoverer = DeadLetterPublishingRecoverer(kafkaTemplate) { record, ex ->  
              // 기본 DLT 토픽 이름: 원래 토픽 이름 + ".DLT"  
              TopicPartition("${record.topic()}.DLT", record.partition())  
          }  
          // |-- 추가 끝 --|  
            
          // 재시도: 1초 간격, 최대 3회  
          val backOff = FixedBackOff(1000L, 3L)  
            
          // |-- 추가 시작 --|  
          val errorHandler = DefaultErrorHandler(recoverer, backOff)  
          // |-- 추가 끝 --|  
          return errorHandler  
      }  
            
      @Bean(name = ["kafkaListenerContainerFactory"])  
      fun kafkaListenerContainerFactory(  
          consumerFactory: ConsumerFactory<String, String>,  
          errorHandler: DefaultErrorHandler  
      ): ConcurrentKafkaListenerContainerFactory<String, String> {  
          val factory = ConcurrentKafkaListenerContainerFactory<String, String>()  
          factory.consumerFactory = consumerFactory  
          factory.containerProperties.ackMode = ContainerProperties.AckMode.MANUAL  
            
          factory.setCommonErrorHandler(errorHandler)  
            
          return factory  
      }  
  ```  
- user.DLT 토픽 cli로 구독  
  ```bash  
  kafka-log-dirs.sh \  
   --bootstrap-server kafka:9092 \  
   --describe \  
   --topic-list user.DLT  
  ```  
- 결과  
    - 컨슈머에서 acknowledgment.acknowledge() 실행 전에 에러 발생시키고  
      메세지를 생성해보면 3회 재시도 후 user.DLT로 전달된 것을 확인할 수 있다.  
    - 컨슈머 그룹 상태 조회  
      ```text  
      GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                           HOST            CLIENT-ID  
      user            user            2          27              27              0               consumer-user-9-8e8f02f8-574b-444e-9dbc-657097764957  /172.18.0.1     consumer-user-9  
      user            user            1          80              80              0               consumer-user-8-6715d66b-a88a-48a8-8709-5cd9a13b28c6  /172.18.0.1     consumer-user-8  
      user            user            0          34              34              0               consumer-user-10-41b4eacf-896b-429c-8d3c-d6e9c4bdec95 /172.18.0.1  
      ```  
    - 실험 대상 파티션은 0번으로 CURRENT-OFFSET과 LOG-END-OFFSET이   
      동일해진 것을 확인할 수 있다.  
    - user.DLT 토픽 구독 cli 화면에 "hello world message"가 출력되는 것을   
      확인할 수 있다.  
