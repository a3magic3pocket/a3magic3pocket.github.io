---
title: 카프카 커넥트(kafka connect)로 mongoDB에 카프카 스트림 쓰기
date: 2024-03-27 23:23:11 +0900
categories: [Kafka]
tags: [kafka, kafka-streams, kafka-connect]    # TAG names should always be lowercase
---

## 개요
- 카프카 스트림의 취합된 결과를   
  외부 스토리지에 저장하고 싶을 때가 있다.  
- 이때 결과가 담긴 토픽 데이터를  
  카프카 커넥트(kafka connect)를 이용하여 다른 스토리지에 저장한다.  
- 카프카, mongodb, 카프카 커넥트를 도커 컴포즈로 구축한다.  

## 카프카 커넥트
- 카프카 커넥트는   
  카프카 스트림 데이터를 다른 스토리지로 옮기거나(Sink Connector)  
  다른 스토리지 데이터를 카프카로 옮길 수 있다(Source Connector).  
- 예시에서는 카프카 스트림 데이터를 mongoDB로 옮기는  
  Sink Connector를 다룬다.  

## mongoDB Sink Connector driver
- Sink Connector driver는 스토리지마다 별도로 존재한다.  
  mongoDB는 confluent에서 Sink Connector를 제공한다.  
- [mongoDB Sink Connector driver 다운로드](https://www.confluent.io/hub/mongodb/kafka-connect-mongodb){:target="_blank"}  
- [압축 결과 폴더]/lib/mongo-kafka-connect-x.x.x-confluent.jar 가   
  드라이버 파일이다.  
- 이 파일을 ./kafka/jar로 옮긴다.  

## 카프카 커넥트 도커 이미지(docker image)
- confluentinc/cp-kafka-connect 도커 이미지를 사용한다.  
- [카프카 커넥트 도커 허브 링크](https://hub.docker.com/r/confluentinc/cp-kafka-connect/tags){:target="_blank"}  

## 카프카 설정
- 단일 노드를 사용한다고 가정한다.  
- [아파치 카프카 단일 노드 도커 허브 예시](https://github.com/apache/kafka/blob/trunk/docker/examples/jvm/single-node/file-input/docker-compose.yml){:target="_blank"}  
- 카프카 설정 파일 생성  
    - kafka/config/file-inputs/server.properties  
      ```bash  
      # Licensed to the Apache Software Foundation (ASF) under one or more  
      # contributor license agreements.  See the NOTICE file distributed with  
      # this work for additional information regarding copyright ownership.  
      # The ASF licenses this file to You under the Apache License, Version 2.0  
      # (the "License"); you may not use this file except in compliance with  
      # the License.  You may obtain a copy of the License at  
      #  
      #    http://www.apache.org/licenses/LICENSE-2.0  
      #  
      # Unless required by applicable law or agreed to in writing, software  
      # distributed under the License is distributed on an "AS IS" BASIS,  
      # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  
      # See the License for the specific language governing permissions and  
      # limitations under the License.  
                
      advertised.listeners=PLAINTEXT_HOST://localhost:9092,SSL://localhost:9093,PLAINTEXT://broker:19092  
      controller.listener.names=CONTROLLER  
      group.initial.rebalance.delay.ms=0  
      inter.broker.listener.name=PLAINTEXT  
      listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,CONTROLLER:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT  
      log.dirs=/tmp/kraft-combined-logs  
      offsets.topic.replication.factor=1  
      process.roles=broker  
      ssl.client.auth=required  
      ssl.key.password=abcdefgh  
      ssl.keystore.location=/etc/kafka/secrets/kafka01.keystore.jks  
      ssl.keystore.password=abcdefgh  
      ssl.truststore.location=/etc/kafka/secrets/kafka.truststore.jks  
      ssl.truststore.password=abcdefgh  
      transaction.state.log.min.isr=1  
      transaction.state.log.replication.factor=1  
      ```  
    - kafka/config/secrets/kafka_keystore_creds  
      ```bash  
      abcdefgh  
      ```  
    - kafka/config/secrets/kafka_ssl_key_creds  
      ```bash  
      abcdefgh  
      ```  
    - kafka/config/secrets/kafka_truststore_creds  
      ```bash  
      abcdefgh  
      ```  
    - kafka/config/secrets/kafka.truststore.jks  
    - kafka/config/secrets/kafka01.keystore.jks  

## docker-compose.yml 파일 작성
- yml 파일  
  ```yml  
  # Use root/example as user/password credentials  
  version: '3.8'  
            
  services:  
    kafka:  
      image: apache/kafka:3.7.0  
      restart: unless-stopped  
      hostname: broker  
      container_name: broker  
      ports:  
        - 9092:9092  
        - 9093:9093  
      volumes:  
        - ./kafka/config/secrets:/etc/kafka/secrets  
        - ./kafka/config/file-input:/mnt/shared/config  
      environment:  
        # Environment variables used by kafka scripts will be needed in case of File input.  
        CLUSTER_ID: '4L6g3nShT-eMCtK--X86sw'  
        # Set properties not provided in the file input  
        KAFKA_NODE_ID: 1  
        KAFKA_CONTROLLER_QUORUM_VOTERS: '1@broker:29093'  
        KAFKA_LISTENERS: 'PLAINTEXT_HOST://:9092,SSL://:9093,CONTROLLER://:29093,PLAINTEXT://:19092'  
        # Override an existing property  
        KAFKA_PROCESS_ROLES: 'broker,controller'  
            
            
    mongo:  
      image: mongo:jammy  
      restart: always  
      ports:  
        - 27017:27017  
      volumes:  
        - ./mongodb:/data/db  
      environment:  
        MONGO_INITDB_ROOT_USERNAME: mongodbuser  
        MONGO_INITDB_ROOT_PASSWORD: mongodbpw  
            
    kafka-connect:  
        image: confluentinc/cp-kafka-connect:7.6.0  
        ports:  
          - 8083:8083  
        depends_on:  
          - kafka  
          - mongo  
        environment:  
          CONNECT_BOOTSTRAP_SERVERS: broker:19092  
          CONNECT_REST_PORT: 8083  
          CONNECT_GROUP_ID: "quickstart-avro"  
          CONNECT_CONFIG_STORAGE_TOPIC: "quickstart-avro-config"  
          CONNECT_OFFSET_STORAGE_TOPIC: "quickstart-avro-offsets"  
          CONNECT_STATUS_STORAGE_TOPIC: "quickstart-avro-status"  
          CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1  
          CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1  
          CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1  
          CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"  
          CONNECT_VALUE_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"  
          CONNECT_INTERNAL_KEY_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"  
          CONNECT_INTERNAL_VALUE_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"  
          CONNECT_REST_ADVERTISED_HOST_NAME: "localhost"  
          CONNECT_LOG4J_ROOT_LOGLEVEL: WARN  
          CONNECT_PLUGIN_PATH: "/usr/share/java,/etc/kafka-connect/jars"  
        volumes:  
          - ./kafka-connect/jars:/etc/kafka-connect/jars  
            
            
  ```  
- 주의사항  
    - kafka-connect.environment.CONNECT_BOOTSTRAP_SERVERS는  
      카프카 KAFKA_LISTENERS의 PLAINTEXT 포트로 설정해야 한다.  
    - kafka-connect.environment.CONNECT_PLUGIN_PATH는  
      mongodb Sink Connector driver 파일 위치이다.  

## 카프카 스트림 설정
- 개요  
    - mongoDB는 json 형태로 문서를 삽입할 수 있기 때문에  
      카프카 스트림의 value Serde를 JsonSerde 사용해도 된다.  
    - RDBMS 같은 경우 필드 별 타입이 엄격하기 때문에  
      json schema나 avro 같은 Serde를 반드시 사용해야 하는 것 같다.  
      [[참고1 - 사용 가능한 Serdes 목록](https://docs.confluent.io/platform/7.6/streams/developer-guide/datatypes.html#available-serdes){:target="_blank"}]  
      (실험 하지 못 함)  
    - aggregate를 이용하여 카프카 스트림 value를 원하는 값으로 변경한다.  
      [[참고2 - 사용 가능한 Aggregating Transformation 목록](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#aggregating){:target="_blank"}]  
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
      public int count;  
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
                  String myKey = value.getName() + "|" + value.getValue();  
                  return myKey.replace("\"", "");  
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
    - kStream 집계 부분 설명  
        - .filter  
            - 최초 key, value에서 key는 null이다.  
            - value에서 name과 value가 모두 존재하는지 확인한다.  
            - 하나라도 존재하지 않으면 집계에서 제외한다.  
        - .selectKey  
            - key를 할당한다.  
            - value에서 "${name}|${value}" 문자열을 key로 할당한다.  
        - .groupByKey  
            - key를 기준으로 groupBy 한다.  
        - .aggregate  
            - groupBy 연산을 어떻게 할지 정의한다[[참고3](https://stackoverflow.com/questions/45992888/using-kafka-streams-to-create-a-new-kstream-containing-multiple-aggregations){:target="_blank"}].  

## 카프카 커넥트 설정
- 개요  
    - Rest API로 카프카 커넥트를 제어할 수 있다.  
- 동작 확인  
    - 설명  
        - 정상적으로 컨테이너가 실행 중이면 응답이 오고  
          실행에 실패하면 timeout 에러가 난다.  
    - 명령  
      ```bash  
      curl --location 'http://localhost:8083/connectors'  
      ```  
- 설정 생성   
    - 설명  
        - name, value 필드가 같은 행은 갱신되도록 설정한다.  
          [[참고4 - Sink Connector Configuration Properties](https://www.mongodb.com/docs/kafka-connector/current/sink-connector/configuration-properties/){:target="_blank"}]  
        - 주요 설정 설명  
            - "topics": "expt1-result"  
                - 카프카 커넥트가 mongodb로 데이터를 가져올 토픽 명이다.  
            - "value.converter": "org.apache.kafka.connect.json.JsonConverter"  
                - 카프카 스트림에서 Json 형태로 저장하고 있으므로  
                  JsonConverter를 사용한다.  
            - "document.id.strategy.overwrite.existing": true  
                - 같은 키를 가진 행이 있으면 덮어쓰기 한다.  
            -  "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.PartialValueStrategy"  
                - 여러 필드를 묶어서 키로 인식한다.  
            - "document.id.strategy.partial.value.projection.list": "name,value"  
              "document.id.strategy.partial.value.projection.type": "AllowList",  
                - name과 value 필드를 키로 인식한다.  
            -  "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneBusinessKeyStrategy"  
                - 쓰기 전략을 같은 키가 행이 있으면 교체  
                  없으면 삽입하도록 설정한다.  
    - 명령  
      ```bash  
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
              "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.PartialValueStrategy",  
              "document.id.strategy.partial.value.projection.list": "name,value",  
              "document.id.strategy.partial.value.projection.type": "AllowList",  
              "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneBusinessKeyStrategy"  
          }  
      }'  
      ```  
- 설정 조회  
    - 설명  
        - 정상적으로 카프카 스트림을 mongodb로 쓰고 있는지 확인할 수 있다.  
        - 에러가 발생할 경우 tasks 별 에러 로그를 확인할 수 있다.  
    - 명령  
      ```bash  
      curl --location 'http://localhost:8083/connectors/mongo-sink-expt1/status'  
      ```  
- 설정 삭제  
    - 명령  
      ```bash  
      curl --location --request DELETE 'http://localhost:8083/connectors/mongo-sink-expt1'  
      ```  
- 성공했을 경우  
    - 카프카 스트림이 갱신되면   
      mongodb의 expt database의 expt1 collection에  
      문서가 작성된다.  
    - name과 value가 같은 행이 존재하면 갱신한다.  
    - mongoDB에서는 키로 사용하고 있는   
      name과 value를 unique 키로 생성해서   
      사용하는 것을 권장한다.  

## 참고
- [참고1 - 사용 가능한 Serdes 목록](https://docs.confluent.io/platform/7.6/streams/developer-guide/datatypes.html#available-serdes){:target="_blank"}  
- [참고2 - 사용 가능한 Aggregating Transformation 목록](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#aggregating){:target="_blank"}  
- [참고3 - using kafka-streams to create a new KStream containing multiple aggregations](https://stackoverflow.com/questions/45992888/using-kafka-streams-to-create-a-new-kstream-containing-multiple-aggregations){:target="_blank"}  
- [참고4 - Sink Connector Configuration Properties](https://www.mongodb.com/docs/kafka-connector/current/sink-connector/configuration-properties/){:target="_blank"}  
- [참고 - mongodb SinkConnector docker-compose 환경 예시](https://kimsouce.tistory.com/101){:target="_blank"}  
- [참고 - SinkConnector docker-compose 환경 예시1](https://velog.io/@ksh9409255/%EC%B9%B4%ED%94%84%EC%B9%B4-%EC%BB%A4%EB%84%A5%ED%8A%B8#sink-connector){:target="_blank"}  
- [참고 - SinkConnector docker-compose 환경 예시2](https://velog.io/@ililil9482/kafka-db-%EC%97%B0%EB%8F%99-feat.-mysql){:target="_blank"}  
- [참고 - mongodb SinkConnector 생성 예시](https://github.com/mongodb-university/kafka-edu/blob/main/docs-examples/mongodb-kafka-base/sink_connector/simple_sink.json){:target="_blank"}  
- [참고 - 리스너 포트 설정 오류로 broker 인식 못하는 에러 수정](https://stackoverflow.com/questions/73071862/kafka-connect-finds-0-brokerssolved){:target="_blank"}  
- [참고 - mongodb SinkConnector, 키를 설정하고 키가 중복된 행은 replace 하기](https://stackoverflow.com/questions/68375845/update-partial-field-on-kafka-sink-mongo){:target="_blank"}  
