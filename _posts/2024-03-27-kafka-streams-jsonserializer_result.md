---
title: 카프카 스트림, JsonSerializer 사용하기
date: 2024-03-27 21:02:48 +0900
categories: [Kafka]
tags: [kafka, kafka-streams]    # TAG names should always be lowercase
---

## 개요
- 생산자(procuder)와 카프카 스트림(kafka streams)이  
  서로 Json으로 메세지를 주고 받도록 한다[[참고1](https://stackoverflow.com/questions/70420373/issue-with-configuring-serdes-for-kafka-streams){:target="_blank"}].  
- 커스텀 Serde를 선언하고   
  카프카 스트림 설정의 value_serde에 등록한다.  

## Serde란?
- Serde란 Serializer<>와 Deserializer<> 담긴 Wrapper 이다[[참고2](https://stackoverflow.com/questions/56292702/what-is-the-difference-between-implementing-deserializer-and-serde-in-kafka-cons){:target="_blank"}].  
- Serializer는 생산자가 메시지를 Serialization 할 때 사용한다.  
- Deserializer는 소비자가 메세지를 Deserialization 할 때 사용한다.  

## 예시
- 환경  
    - java 21  
    - spring boot 3.2.3  
- 설치  
  ```bash  
  implementation 'org.apache.kafka:kafka-streams'  
  implementation 'org.springframework.kafka:spring-kafka'  
  ```  
- 가정  
    - "블로그 포스트 조회 수"를 기록하는 카프카 스트림을 구성한다.  
    - 생산자의 메세지에는 postId와 userId가 포함된다.  
- blog/CountBlogViewMessage.java  
  ```java  
  @Getter  
  @Setter  
  public class CountBlogViewMessage {  
      public String postId;  
      public String userId;  
  }  
  ```  
- blog/CountBlogViewsSerde.java  
  ```java  
  import org.apache.kafka.common.serialization.Serdes.WrapperSerde;  
  import org.springframework.kafka.support.serializer.JsonDeserializer;  
  import org.springframework.kafka.support.serializer.JsonSerializer;  
            
  public class CountBlogViewsSerde extends WrapperSerde<CountBlogViewMessage > {  
      public CountBlogViewsSerde() {  
          super(new JsonSerializer<>(), new JsonDeserializer<>(CountBlogViewMessage .class));  
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
            
      @Bean  
      FactoryBean<StreamsBuilder> countBlogViewsDSLBuilder() {  
          StreamsBuilderFactoryBean streamsBuilder = new StreamsBuilderFactoryBean(kStreamsConfig("count-blog-views-id", CountBlogViewsSerde.class));  
          return streamsBuilder;  
      }  
                
      @Bean  
      KStream<String, CountBlogViewMessage> countBlogViewsKStream(StreamsBuilder countBlogViewsDSLBuilder) {  
          // stream을 불러오는 부분, "count-blog-views" 토픽을 구독한다.  
          KStream<String, CountBlogViewMessage> kStream = countBlogViewsDSLBuilder.stream("count-blog-views");  
            
          // 집계를 처리하는 부분  
          // "countBlogViews"는 Materialized view 이름  
          kStream    
              .filter((key, value)-> value.getPostId() != null || value.getUserId() != null)  
              .selectKey((key, value) -> {    
                  String myKey = value.getPostId() + "|" + value.getUserId();  
                  return myKey.replace("\"", "");  
              })    
              .groupByKey()    
              .count(Materialized.as("countBlogViews"))    
          // 집계를 완료한 kTable을 "count-blog-views-result" 토픽에 저장한다.    
          .toStream()    
          .to("count-blog-views-result");    
            
          return kStream;  
      }  
  }  
  ```  
- blog/BlogController.java  
  ```java  
  @RequestMapping(value = "/count-blog-views")  
  @RequiredArgsConstructor  
  @RestController  
  public class BlogController {  
      public final KafkaTemplate<String, Object> kafkaTemplate;  
      public final FactoryBean<StreamsBuilder> countBlogViewsDSLBuilder;  
            
      // kStream 생성자  
      @GetMapping(value = "/produce")  
      public String produceCountBlogViews(@RequestBody CountBlogViewMessage countBlogViewMessage) {  
          this.kafkaTemplate.send("count-blog-views", countBlogViewMessage);  
            
          return "success";  
      }  
            
      // 집계 결과를 출력하는 함수  
      @GetMapping("/all")  
      public void printAllCountBlogViews() {  
          StreamsBuilderFactoryBean builder = (StreamsBuilderFactoryBean) countBlogViewsDSLBuilder;  
            
          KafkaStreams kafkaStreams = builder.getKafkaStreams();  
            
          // "countBlogViews"는 Materialized view 이름  
          ReadOnlyKeyValueStore<String, CountBlogViewMessage> view = kafkaStreams  
              .store(StoreQueryParameters.fromNameAndType("countBlogViews", QueryableStoreTypes.keyValueStore()));  
            
          KeyValueIterator<String, CountBlogViewMessage> address = view.all();  
          address.forEachRemaining(keyValue -> System.out.println("keyValue.toString()++" + keyValue.toString()));  
      }  
  }  
  ```  

## 참고
- [참고1 - Issue with configuring Serdes for Kafka Streams](https://stackoverflow.com/questions/70420373/issue-with-configuring-serdes-for-kafka-streams){:target="_blank"}  
- [참고2 - What is the difference between implementing Deserializer and Serde in Kafka Consumer API?](https://stackoverflow.com/questions/56292702/what-is-the-difference-between-implementing-deserializer-and-serde-in-kafka-cons){:target="_blank"}  
