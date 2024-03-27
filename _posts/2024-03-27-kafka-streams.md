---
title: 카프카 스트림이란?(kafka stream)
date: 2024-03-27 19:59:41 +0900
categories: [kafka]
tags: [kafka, kafka-streams]    # TAG names should always be lowercase
---

## 스트림이란?
- 한 방향으로 흐르는 연속된 데이터 흐름[[참고1](https://ko.wikipedia.org/wiki/%EC%8A%A4%ED%8A%B8%EB%A6%BC_(%EC%BB%B4%ED%93%A8%ED%8C%85){:target="_blank"})[[참고2](https://blog.naver.com/PostView.naver?blogId=harang8069&logNo=222425947051&categoryNo=57&parentCategoryNo=0&viewDate=&currentPage=1&postListTopCurrentPage=1&from=postView){:target="_blank"}]  

## 카프카 스트림이란?
- 카프카 스트림은 스트림을 처리하는 라이브러리이다.  
- 카프카 스트림은 소비자(consumer)에서 동작한다.  
- 예를 들면, '블로그 포스트 조회 수' 같은 이벤트와 같이  
  실시간으로 갱신되는 데이터를 처리할 수 있다.  

## 카프카 스트림 특징
- 카프카 스트림은 정확히 한 번(exactly once)만 데이터를 저장한다[[참고3](https://www.confluent.io/ko-kr/blog/enabling-exactly-once-kafka-streams/){:target="_blank"}].  
- 하나의 카프카 스트림에도   
  멀티 스레딩과 리커버리를 지원하기 때문에  
  쉽게 대용량 처리가 가능하다[[참고4](https://kafka.apache.org/37/documentation/streams/architecture#streams_architecture_threads){:target="_blank"}]  

## 카프카 스트림 구조
- [[참고5: streams_concepts_duality](https://kafka.apache.org/37/documentation/streams/core-concepts#streams_concepts_duality){:target="_blank"}]에 자세히 설명되어 있다.  
- 크게  kStream과 kTable로 구성된다.  
- kStream  
    - 각 시간 별 스트림 데이터이다.  
    - 예를 들어 블로그 포스트 조회 수라면  
      {"post_id": 1}  
- kTable  
    - 각 시간 별 스트림 데이터의 변경을 기록한다.  
    - 예를 들어 변경을 합산(sum)이라고 생각하면 아래와 같이 동작한다.  
        - time=1   
            - kStream={"post_id":1} -> kTable= {"post_id":1, "summed": 1}  
        - time=2  
            - kStream={"post_id":1} -> kTable= {"post_id":1, "summed": 2}  
        - time=3  
            - kStream={"post_id":2} -> kTable= [{"post_id":1, "summed": 2}, {"post_id":2, "summed": 1}]  
- 집계(aggregation)  
    - kTable에서 '변경을 합산(sum)'이라고 설명한 부분이 집계이다.  
    - kStream이 들어왔을 때 key를 기준으로  
      원하는 집계 로직을 구축하면  
      kTable에 반영된다.  
- kStream과 kTable의 join  
    - kStream과 kTable 모두 join이 가능하다.  

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
    - 생산자에서 전송하는 메세지는 문자열이라고 가정한다.  
    - 편의를 위하여 문자열은 "블로그 포스트 ID"라고 가정한다.  
    - [[참고6](https://www.baeldung.com/spring-boot-kafka-streams){:target="_blank"}][[참고7](https://blog.voidmainvoid.net/442){:target="_blank"}]  
- global/config/KafkaStreamsConfig.java  
  ```java  
  @Configuration  
  public class KafkaStreamsConfig {  
            
      @Value("${spring.kafka.producer.bootstrap-servers}")  
      private String bootstrapServers;  
            
      KafkaStreamsConfiguration kStreamsConfig(String applicationId, Object valueSerde) {  
          Map<String, Object> props = new HashMap<>();  
          props.put(StreamsConfig.APPLICATION_ID_CONFIG, "my-streams-app");  
          props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);  
          props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());  
          props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());  
            
          return new KafkaStreamsConfiguration(props);  
      }  
            
      @Bean  
      FactoryBean<StreamsBuilder> countBlogViewsDSLBuilder() {  
          StreamsBuilderFactoryBean streamsBuilder = new StreamsBuilderFactoryBean();  
          return streamsBuilder;  
      }  
                
      @Bean  
      KStream<String, String> countBlogViewsKStream(StreamsBuilder countBlogViewsDSLBuilder) {  
          // stream을 불러오는 부분, "count-blog-views" 토픽을 구독한다.  
          KStream<String, String> kStream = countBlogViewsDSLBuilder.stream("count-blog-views");  
            
          // 집계를 처리하는 부분  
          // "countBlogViews"는 Materialized view 이름  
          kStream  
              .selectKey((key, value) -> {  
                  return value.getPostId().replace("\"", "");  
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
  @RequiredArgsConstructor  
  @RestController  
  public class BlogController {  
      public final FactoryBean<StreamsBuilder> countBlogViewsDSLBuilder;  
            
      // 집계 결과를 출력하는 함수  
      @GetMapping("/count")  
      public void printAllCountBlogViews() {  
          StreamsBuilderFactoryBean builder = (StreamsBuilderFactoryBean) countBlogViewsDSLBuilder;  
            
          KafkaStreams kafkaStreams = builder.getKafkaStreams();  
            
          // "countBlogViews"는 Materialized view 이름  
          ReadOnlyKeyValueStore<String, Expt1Result> view = kafkaStreams  
              .store(StoreQueryParameters.fromNameAndType("countBlogViews", QueryableStoreTypes.keyValueStore()));  
            
          KeyValueIterator<String, Expt1Result> address = view.all();  
          address.forEachRemaining(keyValue -> System.out.println("keyValue.toString()++" + keyValue.toString()));  
      }  
  }  
            
            
  ```  

## 참고
- [참고1 - 스트림 (컴퓨팅)](https://ko.wikipedia.org/wiki/%EC%8A%A4%ED%8A%B8%EB%A6%BC_(%EC%BB%B4%ED%93%A8%ED%8C%85){:target="_blank"})  
- [참고2 - 스트림이란? 스트림(stream)의 의미, 뜻](https://blog.naver.com/PostView.naver?blogId=harang8069&logNo=222425947051&categoryNo=57&parentCategoryNo=0&viewDate=&currentPage=1&postListTopCurrentPage=1&from=postView){:target="_blank"}  
- [참고3 - Enabling Exactly-Once in Kafka Streams](https://www.confluent.io/ko-kr/blog/enabling-exactly-once-kafka-streams/){:target="_blank"}  
- [참고4 - Threading Model](https://kafka.apache.org/37/documentation/streams/architecture#streams_architecture_threads){:target="_blank"}  
- [참고5 - Duality of Streams and Tables](https://kafka.apache.org/37/documentation/streams/core-concepts#streams_concepts_duality){:target="_blank"}]에 자세히 설명되어 있  
- [참고6 - Kafka Streams With Spring Boot](https://www.baeldung.com/spring-boot-kafka-streams){:target="_blank"}]  
- [참고7 - 카프카 스트림즈 KTable로 선언한 토픽을 key-value 테이블로 사용하기](https://blog.voidmainvoid.net/442){:target="_blank"}  
