---
title: 여러 개의 카프카 스트림 사용하기
date: 2024-03-27 21:46:18 +0900
categories: [Kafka]
tags: [kafka, kafka-streams]    # TAG names should always be lowercase
---

## 개요
- 토픽마다 다른 json을 받는 카프카 스트림을 생성한다.  
- 카프카 스트림은  
  같은 이름의 StreamsBuilderFactoryBean Bean을   
  오직 하나만 허용한다.  
- 따라서 StreamsBuilderFactoryBean에 이름을 부여하여  
  여러 개 생성할 수 있도록 한다.  
- 이름이 다른 StreamsBuilderFactoryBean은  
  @Qualifier를 이용하여 로드한다[[참고1](https://stackoverflow.com/questions/51699996/how-to-define-two-instances-of-streamsbuilderfactorybean){:target="_blank"}][[참고2](https://stackoverflow.com/questions/51733039/kafka-streams-with-spring-boot){:target="_blank"}].  
- 카프카 스트림 스토어(store)를 직접 조회해야 하는 경우  
  @Autowired로 StreamBuilder를 의존성 주입하여 사용한다[[참고3](https://stackoverflow.com/questions/30360589/does-spring-autowired-inject-beans-by-name-or-by-type){:target="_blank"}].  

## 예시
- 환경  
    - java 21  
    - spring boot 3.2.3  
- 설치  
  ```bash  
  implementation 'org.apache.kafka:kafka-streams'  
  implementation 'org.springframework.kafka:spring-kafka'  
  ```  
-  expt1/Expt1.java  
  ```java  
  @Getter
  @Setter
  public class Expt1 {
      public String name;
      public String value;
  }
  ```  
- expt1/Expt1Serde.java   
  ```java  
  import org.apache.kafka.common.serialization.Serdes.WrapperSerde;  
  import org.springframework.kafka.support.serializer.JsonDeserializer;  
  import org.springframework.kafka.support.serializer.JsonSerializer;  
            
  public class Expt1Serde extends WrapperSerde<Expt1> {  
      public Expt1Serde() {  
          super(new JsonSerializer<>(), new JsonDeserializer<>(Expt1.class));  
      }  
  }  
  ```  
- expt2/Expt2.java  
  ```java  
  @Getter  
  @Setter  
  public class Expt2 {  
      public String name;  
      public String value;  
  }  
  ```  
- expt2/Expt2Serde.java  
  ```java  
  ...  
  public class Expt2Serde extends WrapperSerde<Expt2> {  
      public Expt2Serde() {  
          super(new JsonSerializer<>(), new JsonDeserializer<>(Expt2.class));  
      }  
  }  
  ```  
- global/config/KafkaStreamsConfig.java  
  ```java  
  @Configuration  
  public class KafkaStreamsConfig {  
            
      @Value("${spring.kafka.producer.bootstrap-servers}")  
      private String bootstrapServers;  
            
      KafkaStreamsConfiguration kStreamsConfig(String applicationId, Object valueSerde) {  
          Map<String, Object> props = new HashMap<>();  
          props.put(StreamsConfig.APPLICATION_ID_CONFIG, applicationId);  
          props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);  
          props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());  
          props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, valueSerde);  
            
          return new KafkaStreamsConfiguration(props);  
      }  
            
      @Bean("expt1DSLBuilder")  
      FactoryBean<StreamsBuilder> expt1DSLBuilder() {  
          StreamsBuilderFactoryBean streamsBuilder = new StreamsBuilderFactoryBean(kStreamsConfig("expt1-id", Expt1.class));  
          return streamsBuilder;  
      }  
                
      @Bean("expt1KStream")  
      KStream<String, Expt1> expt1KStream(@Qualifier("expt1DSLBuilder") StreamsBuilder expt1DSLBuilder) {  
          // stream을 불러오는 부분, "expt1" 토픽을 구독한다.  
          KStream<String, Expt1> kStream = expt1DSLBuilder.stream("expt1");  
            
          // 집계를 처리하는 부분  
          // "expt1"는 Materialized view 이름  
          kStream  
              .selectKey((key, value) -> {  
                  return value.getPostId().replace("\"", "");  
              })  
              .groupByKey()  
              .count(Materialized.as("expt1"))  
           // 집계를 완료한 kTable을 "expt1-result" 토픽에 저장한다.  
          .toStream()  
          .to("expt1-result");  
            
          return kStream;  
      }  
            
      @Bean("expt2DSLBuilder")  
      FactoryBean<StreamsBuilder> expt2DSLBuilder() {  
          StreamsBuilderFactoryBean streamsBuilder = new StreamsBuilderFactoryBean(kStreamsConfig("expt2-id", Expt2.class));  
          return streamsBuilder;  
      }  
                
      @Bean("expt2KStream")  
      KStream<String, Expt2> expt2KStream(@Qualifier("expt2DSLBuilder") StreamsBuilder expt2DSLBuilder) {  
          // stream을 불러오는 부분, "expt2" 토픽을 구독한다.  
          KStream<String, Expt2> kStream = expt2DSLBuilder.stream("expt2");  
            
          // 집계를 처리하는 부분  
          // "expt2"는 Materialized view 이름  
          kStream  
              .selectKey((key, value) -> {  
                  return value.getPostId().replace("\"", "");  
              })  
              .groupByKey()  
              .count(Materialized.as("expt2"))  
           // 집계를 완료한 kTable을 "expt2-result" 토픽에 저장한다.  
          .toStream()  
          .to("expt2-result");  
            
          return kStream;  
      }  
  }  
  ```  
- expt1/Expt1Controller.java  
  ```java  
  @RequestMapping(value = "/expt1")  
  @RequiredArgsConstructor  
  @RestController  
  public class Expt1Controller {  
      public final KafkaTemplate<String, Object> kafkaTemplate;  
      @Autowired  
      private @Qualifier("expt1DSLBuilder") FactoryBean<StreamsBuilder> expt1DSLBuilder;  
            
      @PostMapping(value = "/produce")  
      public void produceExpt1(@RequestBody Expt1 expt1) {  
          this.kafkaTemplate.send("expt1", expt1);  
      }  
            
      @GetMapping("/all")  
      public void printAllExpt1() {  
          StreamsBuilderFactoryBean builder = (StreamsBuilderFactoryBean) expt1DSLBuilder;  
          KafkaStreams kafkaStreams = builder.getKafkaStreams();  
            
          ReadOnlyKeyValueStore<String, Expt1> view = kafkaStreams  
                  .store(StoreQueryParameters.fromNameAndType("expt1", QueryableStoreTypes.keyValueStore()));  
            
          KeyValueIterator<String, Expt1> address = view.all();  
          address.forEachRemaining(keyValue -> System.out.println("keyValue.toString()++" + keyValue.toString()));  
      }  
  }  
  ```  

## 참고
- [참고1 - how to define two instances of StreamsBuilderFactoryBean](https://stackoverflow.com/questions/51699996/how-to-define-two-instances-of-streamsbuilderfactorybean){:target="_blank"}  
- [참고2 - Kafka Streams with Spring Boot](https://stackoverflow.com/questions/51733039/kafka-streams-with-spring-boot){:target="_blank"}  
- [참고3 - Does Spring @Autowired inject beans by name or by type?](https://stackoverflow.com/questions/30360589/does-spring-autowired-inject-beans-by-name-or-by-type){:target="_blank"}  
