---
title: mongoDB Sink Connector, FullKeyStrategy id-strategy 사용하기
date: 2024-04-08 18:22:46 +0900
categories: [Kafka]
tags: [kafka, kafka-connect, mongodb-sink-connector]    # TAG names should always be lowercase
---

## 개요
- PartialValueStrategy 사용할 때 Custom Post Processor를 사용하는 경우,  
  키로 사용되는 필드가 Post Processing 대상이 되면   
  ReplaceOneBusinessKeyStrategy이 동작하지 않는다.  
- id-strategy와 상관없이 Post Processing을 적용하기 쉽도록  
  메세지 키를 이용하여 문서 _id를 만드는 전략인 FullKeyStrategy를 사용한다.  

## FullKeyStrategy
- 메세지의 키에서 _id 필드만 추출하여 그 값을 키로 사용하는 전략이다.  
- 메세지 예시  
  ```  
  SinkRecord{  
      kafkaOffset=2,   
      timestampType=CreateTime,   
      originalTopic=user-result,   
      originalKafkaPartition=0,   
      originalKafkaOffset=2  
  }   
            
  ConnectRecord{  
      topic='expt1-result',   
      kafkaPartition=0,   
      key={'_id':"my-key"},   
      keySchema=Schema{STRING},   
      value={  
          value=my-value,   
          name=my-name,  
          count=17  
      },   
      valueSchema=null,   
      timestamp=1712566487462,   
      headers=ConnectHeaders(  
          headers=[  
              ConnectHeader(  
                  key=__TypeId__,   
                  value=com.myapp.api.user.User,   
                  schema=Schema{STRING}  
              )  
          ]  
      )  
  }  
  ```  
- [[참고](https://www.mongodb.com/ko-kr/docs/kafka-connector/current/sink-connector/fundamentals/post-processors/#configure-the-document-id-adder-post-processor){:target="_blank"}]  

## 카프카, 카프카 스트림, 카프카 커넥트 설정
- 기본적으로 아래 링크와 동일한 설정이며  
  앞으로의 내용은 변경된 부분을 주로 서술한다.  
- [카프카 커넥트(kafka connect)로 mongoDB에 카프카 스트림 쓰기](https://a3magic3pocket.github.io/posts/write-kafka-stream-to-mongodb-using-kafka-connect/#%EC%B9%B4%ED%94%84%EC%B9%B4-%EC%84%A4%EC%A0%95){:target="_blank"}  

## 카프카 스트림 설정
- 설명  
    - selectKey에서 {"_id": "my-key"}와 같이   
      json 문자열을 키로 설정해야한다.  
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
              .filter((key, value)-> value.getName() != null || value.getValue() != null)    
              .selectKey((key, value) -> {      
                  return "{'_id':\"" + value.getName() + "|" + value.getValue() + "\"}";    
              })      
              .groupByKey()    
              .aggregate(    
                  new Initializer<Expt1>() {    
                      public Expt1 apply() {    
                          return new Expt1();    
                      }    
                  },    
                  new Aggregator<String, Expt1, Expt1>() {    
                      public Expt1 apply(String key, Expt1 value, Expt1 aggregate) {    
                          aggregate.setName(value.getName());    
                        
                          aggregate.setValue(value.getValue());    
                                                
                          aggregate.setCount(aggregate.getCount() + 1);    
                                                
                          return aggregate;    
                      }    
                  },    
                  Materialized.as("expt1")    
              )    
           // 집계를 완료한 kTable을 "expt1-result" 토픽에 저장한다.    
          .toStream()    
          .to("expt1-result");    
                        
          return kStream;    
      }    
  }    
  ```  

## 카프카 커넥트 설정
- 설명  
    - document.id.strategy를 FullKeyStrategy로 설정한다.  
    - document.id.strategy.overwrite.existing를 true로 설정한다.  
- 설정 생성 명령  
  ```json  
  curl --location 'http://localhost:8083/connectors' \    
  --header 'Content-Type: application/json' \    
  --data-raw '{    
      "name": "mongo-sink-expt1",    
      "config": {    
          "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",    
          "connection.uri": "mongodb://mongodbuser:mongodbpw@mongo:27017/",    
          "tasks.max": "2",    
          "topics": "expt1-result",    
          "database": "expt",    
          "collection": "expt1",    
          "key.converter": "org.apache.kafka.connect.storage.StringConverter",    
          "value.converter": "org.apache.kafka.connect.json.JsonConverter",    
          "key.converter.schemas.enable": false,    
          "value.converter.schemas.enable": false,    
          "document.id.strategy.overwrite.existing": true,    
          "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.FullKeyStrategy",    
          "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneBusinessKeyStrategy"    
      }    
  }'    
  ```  

## 참고
- [참고 - 문서 ID 추가 포스트 프로세서 구성](https://www.mongodb.com/ko-kr/docs/kafka-connector/current/sink-connector/fundamentals/post-processors/#configure-the-document-id-adder-post-processor){:target="_blank"}  
