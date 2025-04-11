---
title: Spring Kafka 컨슈머 배치 설정
date: 2025-04-11 22:33:19 +0900
categories: [Kafka]
tags: [kafka, spring, kotlin, batch]    # TAG names should always be lowercase
---

## 개요
- 컨슈머에서 폴링 시 배치 단위로 가져오게 할 수 있다.  
- 이를 통하여 처리 성능을 높일 수 있다.  

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
        # | -- 추가 시작 -- |  
        max-poll-records: 20  
        # | -- 추가 끝 -- |  
        properties:  
          spring.json.trusted.packages: "*"  
  ```  
- KafkaConfig  
  ```kotlin  
  package com.example.demo.config  
            
  import org.apache.kafka.common.TopicPartition  
  import org.springframework.context.annotation.Bean  
  import org.springframework.context.annotation.Configuration  
  import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory  
  import org.springframework.kafka.core.ConsumerFactory  
  import org.springframework.kafka.core.KafkaTemplate  
  import org.springframework.kafka.listener.ContainerProperties  
  import org.springframework.kafka.listener.DeadLetterPublishingRecoverer  
  import org.springframework.kafka.listener.DefaultErrorHandler  
  import org.springframework.util.backoff.FixedBackOff  
            
  @Configuration  
  class KafkaConfig {  
            
      @Bean(name = ["batchKafkaListenerContainerFactory"])  
      fun batchKafkaListenerContainerFactory(  
          consumerFactory: ConsumerFactory<String, String>  
      ): ConcurrentKafkaListenerContainerFactory<String, String> {  
          val factory = ConcurrentKafkaListenerContainerFactory<String, String>()  
          factory.consumerFactory = consumerFactory  
          factory.isBatchListener = true  
            
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
  package com.example.demo.kafka.consumer  
            
  import com.example.demo.constant.KafkaTopic  
  import org.springframework.kafka.annotation.KafkaListener  
  import org.springframework.kafka.support.Acknowledgment  
  import org.springframework.stereotype.Component  
            
  @Component  
  class UserConsumer {  
            
      @KafkaListener(  
          topics = [KafkaTopic.USER],  
          groupId = "user",  
          containerFactory = "batchKafkaListenerContainerFactory"  
      )  
      fun consume(record: List<String>) {  
          Thread.sleep(2000)  
          println("record++" + record)  
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

## 결과
- 배치 단위로 잘 넘어온다.  
  ```text  
  record++[hello world message]  
  record++[hello world message, hello world message]  
  record++[hello world message, hello world message, hello world message, hello world message, hello world message, hello world message, hello world message]  
  record++[hello world message, hello world message, hello world message, hello world message, hello world message, hello world message, hello world message, hello world message]  
  record++[hello world message, hello world message, hello world message, hello world message, hello world message, hello world message, hello world message, hello world message]  
  record++[hello world message, hello world message, hello world message, hello world message, hello world message, hello world message, hello world message, hello world message, hello world message]  
  record++[hello world message, hello world message, hello world message, hello world message]  
            
  ```  
- max.poll.records=20은 최대값일 뿐이므로   
  적절한 규모로 배치가 넘어오는 것을 확인할 수 있다.  
