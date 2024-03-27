---
title: 카프카, JsonSerializer 사용하기
date: 2024-03-27 16:35:51 +0900
categories: [kafka]
tags: [kafka, spring]    # TAG names should always be lowercase
---

## 개요
- 카프카 생산자(producer)  value serializer를 JsonSerializer로 설정한다.  
- 카프카 소비자(consumer) value deserializer를 JsonDeserializer로 설정한다.  
- 소비자마다 다른  json을 message로 받을 수 있도록 한다.  

## 환경
- java 21  
- spring boot 3.2.3  

## 의존성 패키지 설치
```bash  
implementation 'org.springframework.kafka:spring-kafka'  
```  

## 하나의 객체를 jsonSerialization하여 주고 받는 예시
- 가정  
    - 토픽 car를 통해서 생산자와 소비자가   
      CarDto 라는 객체를 주고 받게 하고 싶다.  
- CarDto  
  ```java  
  package com.my.app.car;  
            
  ...  
            
  @Getter  
  @Setter  
  public class CarDto{  
  	String model;  
  	String wheel;  
  	String handle;  
  }  
  ```  
- application.properties  
  ```bash  
  # Consumer  
  spring.kafka.consumer.bootstrap-servers=localhost:9092  
  spring.kafka.consumer.group-id=car  
  spring.kafka.consumer.auto-offset-reset=earliest  
  spring.kafka.consumer.properties.allow.auto.create.topics=false  
            
            
  # Producer  
  spring.kafka.producer.bootstrap-servers=localhost:9092  
            
  ```  
- global/config/KafkaConfig.java  
  ```java  
  @Configuration  
  public class KafkaConfig {  
            
            
      @Value("${spring.kafka.producer.bootstrap-servers}")  
      private String producerBootstrapServers;  
            
            
      @Value("${spring.kafka.consumer.bootstrap-servers}")  
      private String consumerBootstrapServers;  
            
            
      ConsumerFactory<String, Object> consumerFactory(String valueDefaultType) {  
          Map<String, Object> configProps = new HashMap<>();  
          configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, consumerBootstrapServers);  
          configProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  
          configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);  
          configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);  
          configProps.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);  
          configProps.put(JsonDeserializer.VALUE_DEFAULT_TYPE, valueDefaultType);  
            
            
          return new DefaultKafkaConsumerFactory<String, Object>(configProps);  
      }  
                
      ProducerFactory<String, Object> producerFactory() {  
          Map<String, Object> configProps = new HashMap<>();  
          configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, producerBootstrapServers);  
          configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);  
          configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);  
            
          return new DefaultKafkaProducerFactory<String, Object>(configProps);  
      }  
            
      @Bean  
      KafkaTemplate<String, Object> kafkaTemplate() {  
          return new KafkaTemplate<>(producerFactory());  
      }  
            
      @Bean  
      ConcurrentKafkaListenerContainerFactory<String, CarDto> carKafkaListener() {  
          ConcurrentKafkaListenerContainerFactory<String, CarDto> factory = new ConcurrentKafkaListenerContainerFactory<>();  
          factory.setConsumerFactory(consumerFactory("com.my.app.car.CarDto"));  
          return factory;  
      }  
  }  
  ```  
- car/CarController.java  
  ```java  
  @RequestMapping(value = "/car")  
  @RequiredArgsConstructor  
  @RestController  
  public class CarController {  
      public final KafkaTemplate<String, Object> kafkaTemplate;  
            
      @PostMapping(value = "/producer-car")  
      public String produceCar() {  
          CarDto carDto = new CarDto();  
          carDto.setModel("my-model");  
          carDto.setWheel("my-wheel");  
          carDto.setHandle("my-handle");  
            
          this.kafkaTemplate.send("car", carDto);  
          return "success";  
      }  
            
      @KafkaListener(topics = "car", groupId = "car", containerFactory = "carKafkaListener")  
      private void consumeCar(CarDto message) {  
          System.out.println("received message");  
          System.out.println("message.getModel()" + message.getModel());  
          System.out.println("message.getWheel()" + message.getWheel());  
          System.out.println("message.getHandle()" + message.getHandle());  
      }  
  }  
  ```  

## 에러: Caused by: java.lang.IllegalArgumentException: The class 'com.my.app.car.CarDto' is not in the trusted packages
- 원인  
    - 생산자에서 소비자로 메세지를 전달할때 type header를 추가해서 전달한다.  
    - 이때 허가되지 않은 type header 일 경우 위 에러가 발생한다.  
    - 해결방법은 다양한데 나의 경우에는   
      value_default_type을 직접 지정하는 방법을 적용했을 때 해결되었다[[참고1](https://stackoverflow.com/questions/51688924/spring-kafka-the-class-is-not-in-the-trusted-packages){:target="_blank"}].  
- 생산자 type header 확인  
    - kafka console을 이용해서 소비자로서 car 토픽을 구독할 수 있다[[참고2](https://devroach.tistory.com/102){:target="_blank"}].  
      ```bash  
      sh kafka-console-consumer.sh --bootstrap-server localhost:9092 \  
        --from-beginning \  
        --topic car \  
        --property print.headers=true  
      ```  
    - 응답 예시  
      ```bash  
      __TypeId__: com.my.app.car.CarDto  {"model": "my-model", "wheel": "my-wheel", "handle": "my-handle"}  
      ```  
- ConsumerFactory에 VALUE_DEFAULT_TYPE 지정  
  ```java  
  @Configuration  
  public class KafkaConfig {  
      ...  
      ConsumerFactory<String, Object> consumerFactory(String valueDefaultType) {  
          Map<String, Object> configProps = new HashMap<>();  
          ...  
          // <<<<< 추가 시작 <<<<<  
          configProps.put(JsonDeserializer.VALUE_DEFAULT_TYPE, "com.my.app.car.CarDto");  
          // >>>>> 추가 끝 >>>>>  
            
          return new DefaultKafkaConsumerFactory<String, Object>(configProps);  
      }  
      ...  
  }  
  ```  
- 해결되지 않았던 방법들  
    - 생산자 메세지에서 type header 추가 하지 않게 하기  
        - application.properties에 아래 문구를 추가했지만 에러가 해결되지 않았다.  
          ```bash  
          spring.kafka.producer.properties.spring.json.add.type.headers=false  
          ```  
    - 소비자에서 모든 메세지 type header 신뢰하기  
        - application.properties에 아래 문구를 추가했지만 에러가 해결되지 않았다.  
          ```bash  
          spring.kafka.consumer.properties.spring.json.trusted.packages=*  
          ```  

## 토픽 별로 다른 객체를 jsonSerialization하여 주고 받는 예시
- 개요  
    - 토픽 별로 Listener를 추가한다.  
    - 토픽 boat를 통해서 생산자와 소비자가   
      BoatDto를 주고 받을 수 있도록 한다.  
- boat/BoatController.java  
  ```java  
  package com.my.app.boat;  
            
  ...  
            
  @Getter  
  @Setter  
  public class BoatDto{  
      String model;  
      String motor;  
      String handle;  
  }  
  ```  
- global/config/KafkaConfig.java  
  ```java  
  @Configuration  
  public class KafkaConfig {  
            
            
      @Value("${spring.kafka.producer.bootstrap-servers}")  
      private String producerBootstrapServers;  
            
            
      @Value("${spring.kafka.consumer.bootstrap-servers}")  
      private String consumerBootstrapServers;  
            
            
      ConsumerFactory<String, Object> consumerFactory(String valueDefaultType) {  
          Map<String, Object> configProps = new HashMap<>();  
          configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, consumerBootstrapServers);  
          configProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  
          configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);  
          configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);  
          configProps.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);  
          configProps.put(JsonDeserializer.VALUE_DEFAULT_TYPE, valueDefaultType);  
            
            
          return new DefaultKafkaConsumerFactory<String, Object>(configProps);  
      }  
                
      ProducerFactory<String, Object> producerFactory() {  
          Map<String, Object> configProps = new HashMap<>();  
          configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, producerBootstrapServers);  
          configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);  
          configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);  
            
          return new DefaultKafkaProducerFactory<String, Object>(configProps);  
      }  
            
      @Bean  
      KafkaTemplate<String, Object> kafkaTemplate() {  
          return new KafkaTemplate<>(producerFactory());  
      }  
            
      @Bean  
      ConcurrentKafkaListenerContainerFactory<String, CarDto> carKafkaListener() {  
          ConcurrentKafkaListenerContainerFactory<String, CarDto> factory = new ConcurrentKafkaListenerContainerFactory<>();  
          factory.setConsumerFactory(consumerFactory("com.my.app.car.CarDto"));  
          return factory;  
      }  
            
      // >>>>> 추가 시작 >>>>>  
      @Bean  
      ConcurrentKafkaListenerContainerFactory<String, BoatDto> boatKafkaListener() {  
          ConcurrentKafkaListenerContainerFactory<String, BoatDto> factory = new ConcurrentKafkaListenerContainerFactory<>();  
          factory.setConsumerFactory(consumerFactory("com.my.app.boat.BoatDto"));  
          return factory;  
      }  
      // <<<<< 추가 끝 <<<<<  
  }  
  ```  
- boat/BoatController.java  
  ```java  
  @RequestMapping(value = "/boat")  
  @RequiredArgsConstructor  
  @RestController  
  public class BoatController {  
      public final KafkaTemplate<String, Object> kafkaTemplate;  
            
      @PostMapping(value = "/produce-boat")  
      public String produceBoat() {  
          BoatDto boatDto = new BoatDto();  
          boatDto.setModel("my-model");  
          boatDto.setMotor("my-motor");  
          boatDto.setHandle("my-handle");  
            
          this.kafkaTemplate.send("boat", boatDto);  
          return "success";  
      }  
            
      @KafkaListener(topics = "boat", groupId = "boat", containerFactory = "boatKafkaListener")  
      private void consumeBaot(BoatDto message) {  
          System.out.println("received message");  
          System.out.println("message.getModel()" + message.getModel());  
          System.out.println("message.setMotor()" + message.setModel());  
          System.out.println("message.getHandle()" + message.getHandle());  
      }  
  }  
            
            
  ```  

## 참고
- [[참고1 - Spring Kafka The class is not in the trusted packages](https://stackoverflow.com/questions/51688924/spring-kafka-the-class-is-not-in-the-trusted-packages){:target="_blank"}].  
- [[참고2 - 오늘자 삽질 - Spring Kafka](https://devroach.tistory.com/102){:target="_blank"}].  
