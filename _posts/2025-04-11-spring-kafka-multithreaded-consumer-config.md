---
title: Spring Kafka 컨슈머 멀티스레드 처리 설정
date: 2025-04-11 22:31:49 +0900
categories: [Kafka]
tags: [kafka, spring, kotlin, concurrency]    # TAG names should always be lowercase
---

## 개요
- Spring Kafka의 @KafkaListener 어노테이션에서 concurrency 속성을 설정하면,  
  리스너 컨테이너가 해당 수만큼의 스레드를 생성하여   
  동일한 리스너 로직을 병렬로 실행하며 메시지를 처리한다.  

## 가정
- 도커로 카프카 컨테이너 한 개를 띄운다.  
- 스프링으로 카프카와 소통한다.  
- 스프링 카프카 의존성 및 스프링 의존성 설치는 이미 된 상태이다.  

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
        enable-auto-commit: true  
        key-deserializer: org.apache.kafka.common.serialization.StringDeserializer  
        value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer  
        properties:  
          spring.json.trusted.packages: "*"  
  ```  
- 토픽 생성  
    - partition 3개짜리 user 토픽을 생성한다.  
      ```bash  
       kafka-topics.sh --create \  
       --bootstrap-server kafka:9092 \  
       --topic user \  
       --partitions 3  
      ```  
- 컨슈머  
  ```kotlin  
  @Component  
  class UserConsumer {  
            
      @KafkaListener(  
          topics = [KafkaTopic.USER],  
          groupId = "user",  
          concurrency = "3", // 리스너를 실행할 스레드 수  
      )  
      fun consume(record: ConsumerRecord<String, String>, acknowledgment: Acknowledgment) {  
          Thread.sleep(2000)  
          println("record++" + record)  
          acknowledgment.acknowledge()  
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

## 실험
- 카프카 메세지 생성 및 소비 명령  
  ```bash  
  curl http://localhost:8080/user  
  ```  
- 카프카 파티션에 할당된 스레드 확인  
    - 명령  
      ```bash  
       kafka-consumer-groups.sh \  
       --bootstrap-server kafka:9092 \  
       --describe \  
       --group user  
      ```  
    - 응답  
      ```text  
      GROUP TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID HOST CLIENT-ID  
      user user 0 8 8 0 consumer-user-10-8ae274db-3b0f-4cd2-bb7a-9dc0c5c5cfc9 /172.18.0.1 consumer-user-10  
      user user 2 4 4 0 consumer-user-9-c4bda59a-ac70-41f4-8925-01497cb6202c /172.18.0.1 consumer-user-9  
      user user 1 0 0 0 consumer-user-8-4e721ce9-b567-4f97-81c6-3f1877cbb76f /172.18.0.1 consumer-user-8  
      ```  
- 결과  
    - 3개의 파티션 모두 다른 CUNSUMER-ID가 할당된 것을 볼 수 있다.  

## 의문1 - 하나의 파티션만 CURRENT-OFFSET이 증가하는 이유?
- 균일한 속도로 카프카 메세지를 생성하다보면  
  하나의 파티션만 CURRENT-OFFSET이 증가하는  것을 볼 수 있다.  
- 이는 기본 파티션 분배 정책이 스티키(sticky) 때문이다.  
- 프로듀서에서 메시지 키를 입력하지 않은 경우,   
  스티키 파티셔너는 새로운 메시지가 이전 메시지를   
  소비했던 컨슈머로 전달되도록 같은 파티션에 넣는다.  
- 이를 통해 동일한 파티션에 메시지를 모아 더 큰 배치를 생성하여   
  전송 횟수와 네트워크 오버헤드를 감소시켜   
  결과적으로 지연 시간을 단축시킨다.  
- 실제로 실험해보면 라운드로빈 설정보다   
  스티키 설정이 전체 처리 시간이 더 짧게 나타난다.  

## 의문2 -스티키 설정 시 나머지 파티션은 사용되지 않는가?
- 아니다. 스티키도 일정 조건이 되면 다른 파티션으로 메세지를 전송한다.  
- 프로듀서의 메시지 버퍼가 꽉 차거나 linger.ms 시간(배치 전송 지연시간)이   
  만료되면 현재 배치를 전송하고 새로운 파티션을 선택한다.  
- 따라서 트래픽이 많아 프로듀서에 메시지가 많이 전달되면   
  메시지 버퍼가 더 빠르게 차게 되고,   
  이 경우 다른 파티션으로 전달하는 일이 잦아져   
  결국 메시지가 파티션에 고루 분배된다.  

## 의문3 - 그럼 특정 파티션으로 메세지를 보내고 싶으면 어떻게 하는가?
- 프로듀서에서 메세지 키를 입력하면 된다.  
- 동일한 메세지 키를 가진 메세지는 해당 파티션으로 전달된다.  
- 반대로 매번 새로운 파티션으로 보내고 싶다면   
  메세지 키를 랜덤으로 설정하면 된다(라운드로빈).   
  ```kotlin  
  // 프로듀서  
  @Service  
  class UserProducer(  
      private val kafkaTemplate: KafkaTemplate<String, String>  
  ) {  
            
      fun send(message: String) {  
          // 랜덤 키 생성  
          val key = UUID.randomUUID().toString()  
                    
          // 랜덤 키 추가  
          kafkaTemplate.send(KafkaTopic.USER, key, message)  
      }  
            
  }  
  ```  

## 의문4 - 파티션 수가 3, 컨슈머 수가 1이라면?
- 3개의 파티션을 1개의 컨슈머가 한 번씩 파티션을 순회하며 처리한다.  
  ```text  
  GROUP TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID HOST CLIENT-ID  
  user user 0 8 8 0 consumer-user-8-540afe8a-734e-439c-a7d4-e8fe84088068 /172.18.0.1 consumer-user-8  
  user user 2 4 4 0 consumer-user-8-540afe8a-734e-439c-a7d4-e8fe84088068 /172.18.0.1 consumer-user-8  
  user user 1 0 0 0 consumer-user-8-540afe8a-734e-439c-a7d4-e8fe84088068 /172.18.0.1 consumer-user-8  
  ```  
- 새로운 메세지가 생성되지 않는 상태에서   
  3개의 파티션에 골고루 메세지가 쌓여있는 상황이라면  
  결과적으로 1개의 컨슈머가 순회하면서 모두 처리한다.  
