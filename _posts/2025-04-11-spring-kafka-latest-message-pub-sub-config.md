---
title: Spring Kafka에서 최신 메시지만 소비하는 Pub/Sub 방식 설정
date: 2025-04-11 22:34:28 +0900
categories: [Kafka]
tags: [kafka, spring, kotlin, pubsub]    # TAG names should always be lowercase
---

## 설명
- Kafka를 Pub/Sub 방식으로 활용하고 싶을 때,   
  특정 토픽의 가장 최신 메시지만 소비하는 구조가 필요하다.   
  이 글에서는 Spring Kafka에서   
  auto-offset-reset: latest와 seekToEnd() 설정을 조합하여,   
  컨슈머가 파티션에 연결되었을 때 항상 최신 메시지만   
  소비하도록 설정하는 방법을 소개한다.  

## auto-offset-reset: latest
- 컨슈머 그룹이 아직 오프셋을 저장하지 않은 파티션을  
  처음 읽을 때 어디서부터 읽을지를 지정한다.  
- latest로 설정하면, 해당 컨슈머 그룹은   
  파티션의 가장 마지막 메시지 이후부터 소비를 시작한다.  
- 즉, 기존 메시지는 건너뛰고 새로 들어오는 메시지부터 읽는다.  

## seekToEnd() 
- 리밸런싱 등으로 컨슈머가 파티션을 할당받을 때마다   
  seekToEnd를 실행하여  
  이전에 읽던 위치와 상관없이   
  현재 파티션의 마지막 메시지 이후부터  
  새로 도착하는 메시지만 읽게 된다.  

## 정책
- Pub/Sub 대상이 되는 토픽을 구독하는 컨슈머 그룹 ID에  
  컨테이너 ID가 포함되도록 설정한다.  
  (컨슈머 그룹 ID를 동적으로 할당)  
- 이를 통해 새 컨테이너가 생성될 시 자동으로 새 컨슈머 그룹 ID를 갖게 되고  
  latest 설정에 의해 최신 메세지만 받게 된다.  
- 컨테이너가 재시작되어 기존 컨슈머 그룹 ID로 Kafka에 재연결되는 경우에도,   
  seekToEnd() 덕분에 이전 오프셋이 아닌   
  가장 최신 메시지부터 소비를 시작할 수 있다.  

## application.yml
```yml  
spring:  
kafka:  
  bootstrap-servers: "kafka:9092"  
  producer:  
    key-serializer: org.apache.kafka.common.serialization.StringSerializer  
    value-serializer: org.springframework.kafka.support.serializer.StringSerializer  
  consumer:  
    auto-offset-reset: latest  
    enable-auto-commit: false  
    key-deserializer: org.apache.kafka.common.serialization.StringDeserializer  
    value-deserializer: org.springframework.kafka.support.serializer.StringSerializer  
    properties:  
      spring.json.trusted.packages: "*"  
```  
- 기본 카프카 설정이지만 Pub/Sub 용 카프카 컨슈머는  
  MessageListenerContainer에서  
  auto-offset-reset: latest 설정을 직접할 것이다.  

## SeekToEndReblanaceListener
```kotlin  
package com.example.demo.global.config  
        
import org.apache.kafka.clients.consumer.Consumer  
import org.apache.kafka.common.TopicPartition  
import org.springframework.kafka.listener.ConsumerAwareRebalanceListener  
import org.springframework.stereotype.Component  
        
@Component  
class SeekToEndRebalanceListener: ConsumerAwareRebalanceListener {  
  override fun onPartitionsAssigned(consumer: Consumer<*, *>, partitions: MutableCollection<TopicPartition>) {  
      // 파티션 할당 시, 해당 파티션의 오프셋을 끝으로 이동  
      partitions.forEach { partition ->  
          consumer.seekToEnd(listOf(partition))  
      }  
  }  
}  
```  

## KafkaMessageListenerConfig
```kotlin  
package com.example.demo.global.config  
        
import org.apache.kafka.clients.consumer.ConsumerConfig  
import org.apache.kafka.common.serialization.StringDeserializer  
import org.springframework.beans.factory.annotation.Value  
import org.springframework.context.annotation.Configuration  
import org.springframework.kafka.core.DefaultKafkaConsumerFactory  
import org.springframework.kafka.listener.ContainerProperties  
import org.springframework.kafka.listener.KafkaMessageListenerContainer  
import org.springframework.kafka.listener.MessageListener  
        
@Configuration  
class KafkaMessageListenerConfig(  
  private val seekToEndRebalanceListener: SeekToEndRebalanceListener,  
) {  
        
  @Value("\${spring.kafka.bootstrap-servers}")  
  lateinit var bootstrapServers: String  
        
  fun createKafkaMessageListenerContainer(  
      topic: String,  
      groupId: String,  
      messageListener: MessageListener<String, String>  
  ): KafkaMessageListenerContainer<String, String> {  
      val consumerProps = mapOf<String, Any>(  
          ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG to bootstrapServers,  
          ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java,  
          ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java,  
          ConsumerConfig.GROUP_ID_CONFIG to groupId,  
          ConsumerConfig.AUTO_OFFSET_RESET_CONFIG to "latest"  
      )  
        
      val consumerFactory = DefaultKafkaConsumerFactory<String, String>(consumerProps)  
        
      // 구독할 토픽을 동적으로 지정  
      val containerProperties = ContainerProperties(topic)  
        
      // 메시지를 수동으로 처리할 Listener  
      containerProperties.messageListener = messageListener  
        
      // 컨슈머 리밸런싱이 일어날 때, 항상 최신 오프셋에서 읽기 시작  
      containerProperties.setConsumerRebalanceListener(seekToEndRebalanceListener)  
        
      return KafkaMessageListenerContainer(consumerFactory, containerProperties)  
  }  
        
}  
```  

## 프로듀서
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

## 상수 토픽
```kotlin  
package com.example.demo.constant  
        
object KafkaTopic {  
  const val USER = "user"  
}  
```  

## 상수 컨슈머 그룹 ID
```kotlin  
package com.example.demo.constant  
        
object KafkaConsumerGroupId {  
  const val USER = "user"  
}  
```  

## AppConfig
```kotlin  
package com.example.demo.global.config  
        
import jakarta.annotation.PostConstruct  
import org.springframework.context.annotation.Configuration  
import java.io.BufferedReader  
import java.io.InputStreamReader  
import java.util.UUID  
        
@Configuration  
class AppConfig {  
  var containerId = UUID.randomUUID().toString()  
        
  @PostConstruct  
  private fun initContainerId() {  
      try {  
          // 'hostname' 명령어 실행을 위한 ProcessBuilder 사용  
          val processBuilder = ProcessBuilder("hostname")  
          val process = processBuilder.start()  
          val reader = BufferedReader(InputStreamReader(process.inputStream))  
          val containerIdString = reader.readLine()  // hostname 명령어의 결과  
        
          // hostname 값이 비어 있지 않으면 containerId 업데이트  
          if (!containerIdString.isNullOrBlank()) {  
              containerId = containerIdString  
          }  
      } catch (e: Exception) {  
          // 예외 발생 시 아무 일도 일어나지 않음  
          println("Failed to retrieve container ID. Using default value.")  
          e.printStackTrace()  
      }  
  }  
        
}  
```  

## 컨슈머
```kotlin  
package com.example.demo.kafka.consumer  
        
import com.example.constant.KafkaConsumerGroupId  
import com.example.constant.KafkaTopic  
import com.example.global.config.AppConfig  
import com.example.global.config.KafkaMessageListenerConfig  
import org.springframework.context.annotation.Bean  
import org.springframework.kafka.listener.KafkaMessageListenerContainer  
import org.springframework.kafka.listener.MessageListener  
import org.springframework.stereotype.Service  
        
@Service  
class UserConsumer(  
  private val kafkaMessageListenerConfig: KafkaMessageListenerConfig,  
  private val appConfig: AppConfig,  
) {  
        
  @Bean  
  fun userConsume(): KafkaMessageListenerContainer<String, String> {  
      val container = kafkaMessageListenerConfig.createKafkaMessageListenerContainer(  
          topic = KafkaTopic.USER,  
          groupId = "${KafkaConsumerGroupId.USER}-${appConfig.containerId}",  
          messageListener = messageListener()  
      )  
      container.start()  
        
      return container  
  }  
        
  private fun messageListener(): MessageListener<String, String> {  
      return MessageListener { record ->  
                    
          println("record++" + record)  
      }  
  }  
        
}  
```  
- 컨슈머 그룹 ID에 containerId를 포함시켜   
  새 도커 컨테이너가 참여할 때마다 새로운 컨슈머 그룹으로 컨슈머가 생성된다.  
- @KafkaListener 기반이 아닌 직접 컨테이너를 생성하는 방식이므로, container.start()를 명시적으로 호출해 컨슈머를 시작해야 한다.  
- KafkaMessageListenerContainer를 @Bean으로 등록했기 때문에  
  스프링 컨텍스트가 해당 빈의 라이프사이클을 관리하게 된다.  
  따라서  스프링이 종료되면 userConsume 객체도 자동으로 종료된다.  

## 참고
- [Spring.io - Seeking to a Specific Offset](https://docs.spring.io/spring-kafka/reference/kafka/seek.html)
- [Spring.io - Message Listener Conatiners](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/message-listener-container.html)
