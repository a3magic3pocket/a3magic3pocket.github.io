---
title: 카프카 커넥트(kafka connect), Custom Post Processor 만들기
date: 2024-04-05 14:34:13 +0900
categories: [Kafka]
tags: [kafka, kafka-connect]    # TAG names should always be lowercase
---

## 개요
- 카프카 스트림에서 string 타입으로 저장된 ID 값을  
  mongoDB sink connector에 저장할때 objectID 타입으로 저장하고 싶다.  
- mongoDB sink connector의 Custom Post Processor를 만들어서  
  해결하였다.  
- [[참고1 - How to convert a String field to ObjectId in MongoSinkConnector](https://www.mongodb.com/community/forums/t/how-to-convert-a-string-field-to-objectid-in-mongosinkconnector/108324/6){:target="_blank"}]  

## Post Processor
- 설명  
    - [[참고2 - How to Create a Custom Post Processor](https://www.mongodb.com/docs/kafka-connector/current/sink-connector/fundamentals/post-processors/#how-to-create-a-custom-post-processor){:target="_blank"}]  
    - Custom Post Processor는 sink connector의 plugin으로 처리된다.  
    - [mongoDB Sink Connector PostProcessor.java](https://github.com/mongodb/mongo-kafka/blob/master/src/main/java/com/mongodb/kafka/connect/sink/processor/PostProcessor.java){:target="_blank"}을   
      상속 받는 클래스를 생성한 후  
      process() 메소드에 원하는 로직을 입력하여 만든다.  
      예시에서는 sinkPostProcessor 패키지의  ObjectIdPostProcessor 클래스로 가정한다.  
    - 작성한 ObjectIdPostProcessor .java를  
      class로 빌드 후 jar 파일로 만든다.  
    - 이후 mongoDB sink connector plugin 디렉토리에 jar를 이동시킨다.  
    - Sink Connector 생성 명령에서 ObjectIdPostProcessor를 추가한다.  
      ```json  
      {  
          ...  
          "post.processor.chain": "sinkPostProcessor.ObjectIdPostProcessor",  
          "value.projection.list": "id"  
      }  
      ```  

## 카프카, 카프카 스트림, 카프카 커넥트 설정
- 기본적으로 아래 링크와 동일한 설정이며  
  앞으로의 내용은 변경된 부분을 주로 서술한다.  
- [카프카 커넥트(kafka connect)로 mongoDB에 카프카 스트림 쓰기](https://a3magic3pocket.github.io/posts/write-kafka-stream-to-mongodb-using-kafka-connect/#%EC%B9%B4%ED%94%84%EC%B9%B4-%EC%84%A4%EC%A0%95){:target="_blank"}  

## 트러블 슈팅1 - java version
- 설명  
    - ObjectIdPostProcessor 클래스는 mongoDB sink connector의  
      자바 버전과 동일한 자바 버전에서 컴파일 되어야 한다.  
    - 그렇지 않으면  아래와 같은 아래가 발생한다.  
      2024-04-04 10:42:51 Caused by: java.lang.UnsupportedClassVersionError: com/myapp/streams/ObjectIdPostProcessor has been compiled by a more recent version of the Java Runtime (class file version 65.0), this version of the Java Runtime only recognizes class file versions up to 55.0  
    - 내 작업 환경에서는 jdk 21을 사용 중이었으나   
      mongoDB sink connector가 설치된  
      confluentinc/cp-kafka-connect:7.6.0 이미지는   
      jdk11을 사용 중인 것으로 추정된다[[참고3](https://stackoverflow.com/questions/9170832/list-of-java-class-file-format-major-version-numbers){:target="_blank"}].  
    - OpenJdk 11을 설치하고 "트러블 슈팅2"로 넘어간다.  
      [[참고4](https://jdk.java.net/java-se-ri/11-MR2){:target="_blank"}]  

## 트러블 슈팅2 - 이클립스 그래들 프로젝트 생성
- 설명  
    - [mongoDB Sink Connector PostProcessor.java](https://github.com/mongodb/mongo-kafka/blob/master/src/main/java/com/mongodb/kafka/connect/sink/processor/PostProcessor.java){:target="_blank"}에는   
      의존성 패키지들이 존재한다.  
    - ObjectIdPostProcessor 빌드 시   
      의존성 패키지들도 함께 처리해주기 위해  
      이클립스 그래들 프로젝트를 생성하고  
      기본 compiler를 java 11로 변경한다.  
    - [[참고5](https://jiurinie.tistory.com/123){:target="_blank"}][[참고6](https://chinsun9.github.io/2020/10/05/gradle/){:target="_blank"}]  
- 이클립스에 그래들 설치  
    - (상단 메뉴)Help > Eclipse Marketplace > gradle 검색 후   
      Buildship Gradle Integration 3.0 설치  
    - <a href='/assets/img/2024-04-05-kafka-connect-custom-post-processor/00-marketplace-gradle.jpg' target='_blank'><img src='/assets/img/2024-04-05-kafka-connect-custom-post-processor/00-marketplace-gradle.jpg' width='80%' height='80%'></a>  
- 그래들 프로젝트 생성  
    - (상단 메뉴)File > New > Project...(Ctrl + N)로 New Project 팝업 생성  
    - <a href='/assets/img/2024-04-05-kafka-connect-custom-post-processor/01-new-project.jpg' target='_blank'><img src='/assets/img/2024-04-05-kafka-connect-custom-post-processor/01-new-project.jpg' width='80%' height='80%'></a>  
    - <a href='/assets/img/2024-04-05-kafka-connect-custom-post-processor/02-new-project2.jpg' target='_blank'><img src='/assets/img/2024-04-05-kafka-connect-custom-post-processor/02-new-project2.jpg' width='80%' height='80%'></a>  
    - <a href='/assets/img/2024-04-05-kafka-connect-custom-post-processor/03-new-project3.jpg' target='_blank'><img src='/assets/img/2024-04-05-kafka-connect-custom-post-processor/03-new-project3.jpg' width='80%' height='80%'></a>  
        - gradle 버전: 7.4.2를 선택  
        - Java home: jdk11 디렉토리 선택  
- 컴파일러 버전 변경  
    - (상단 메뉴)Window > Preferences >   
      compiler 검색 후 Java Compiler 선택,  
      Compiler compliance level 11로 변경  
    -   

## ObjectIdPostProcessor 코드
- 설명  
    - mongoDB sink connector 생성 명령에서  
      value.projection.list에 입력된 필드명(쉼표 구분 comma separated)을   
      읽어 들인다.  
    - 카프카 메세지에서 해당 필드의 값(field value)를  
      BsonObjectID 타입으로 변경 후 저장한다.  
- 의존성 패키지  
  ```bash  
  compileOnly 'org.mongodb.kafka:mongo-kafka-connect:1.11.2'  
  compileOnly 'org.apache.kafka:connect-api:3.7.0'  
  compileOnly 'org.mongodb:bson:3.3.0'  
  compileOnly 'org.slf4j:slf4j-api:1.7.25'  
  ```  
- sinkPostProcessor/ObjectIdPostProcessor.java  
  ```java  
  package sinkPostProcessor;  
            
  import org.bson.types.ObjectId;  
            
  import org.apache.kafka.connect.sink.SinkRecord;  
  import org.bson.BsonObjectId;  
            
  import org.slf4j.Logger;  
  import org.slf4j.LoggerFactory;  
            
  import com.mongodb.kafka.connect.sink.MongoSinkTopicConfig;  
  import com.mongodb.kafka.connect.sink.converter.SinkDocument;  
  import com.mongodb.kafka.connect.sink.processor.PostProcessor;  
            
  import static com.mongodb.kafka.connect.sink.MongoSinkTopicConfig.VALUE_PROJECTION_LIST_CONFIG;  
            
  import java.util.ArrayList;  
  import java.util.Arrays;  
  import java.util.List;  
            
  public class ObjectIdPostProcessor extends PostProcessor {  
  	private static final Logger LOGGER = LoggerFactory.getLogger(PostProcessor.class);  
  	private final List<String> fieldNames;  
            
  	public ObjectIdPostProcessor(final MongoSinkTopicConfig config) {  
  		super(config);  
  		String rawFields = config.getString(VALUE_PROJECTION_LIST_CONFIG);  
  		fieldNames = new ArrayList<String>(Arrays.asList(rawFields.split(",")));  
  	}  
            
  	@Override  
  	public void process(final SinkDocument doc, final SinkRecord orig) {  
  		doc.getValueDoc().ifPresent(vd -> {  
  			for (String fieldName : fieldNames) {  
  				if (vd.containsKey(fieldName)) {  
  					String hash = vd.getString(fieldName).getValue();  
  //					LOGGER.warn("+++" + hash);  
  					  
  					vd.put(fieldName, new BsonObjectId(new ObjectId(hash)));  
  				}  
  			}  
  		});  
  	}  
            
  }  
            
  ```  

## 트러블 슈팅3 - ObjectIdPostProcessor.java를 jar 파일로 추출
- ObjectIdPostProcessor.java 우클릭 > Export 선택  
  <a href='/assets/img/2024-04-05-kafka-connect-custom-post-processor/04-extract-jar.jpg' target='_blank'><img src='/assets/img/2024-04-05-kafka-connect-custom-post-processor/04-extract-jar.jpg' width='80%' height='80%'></a>  
- JAR file 선택  
  <a href='/assets/img/2024-04-05-kafka-connect-custom-post-processor/05-extract-jar2.jpg' target='_blank'><img src='/assets/img/2024-04-05-kafka-connect-custom-post-processor/05-extract-jar2.jpg' width='80%' height='80%'></a>  
- 패지키(sinkPostProcessor) 선택 후 ObjectIdPostProcessor.java 선택,  
  jar 파일 경로를 mongoDB  Sink Connector plugin 경로 하위로 선택  
  <a href='/assets/img/2024-04-05-kafka-connect-custom-post-processor/06-extract-jar3.jpg' target='_blank'><img src='/assets/img/2024-04-05-kafka-connect-custom-post-processor/06-extract-jar3.jpg' width='80%' height='80%'></a>  
- mongoDB  Sink Connector plugin 경로란?  
    - docker-compose.yml 예시  
      ```yml  
      # Use root/example as user/password credentials  
      version: '3.8'  
                
      services:  
        kafka:  
          ...  
                
        mongo:  
          ...  
                
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
    - CONNECT_PLUGIN_PATH와 연결된 ./kafka-connect/jars가  
      mongoDB  Sink Connector plugin 경로이다.  
    - [[참고7](https://bioinfoblog.tistory.com/242){:target="_blank"}]  

## ObjectIdPostProcessor 사용하기
- mongoDB Sink Connector 재실행  
    - ObjectIdPostProcessor.jar를 인식시키기 위하여  
      kafka-connect 컨테이너를 재시작한다.  
- sink connector 생성 명령  
    - 설명  
        - post.processor.chain에 삽입할 Custom Post Processor를 입력한다.  
          ex) "post.processor.chain": "sinkPostProcessor.ObjectIdPostProcessor"  
        - value.projection.list에 대상 필드명을 콤마 구분하여 입력한다.  
          ex) "value.projection.list": "id,target-id"  
    - 명령  
      ```bash  
      curl --location 'http://localhost:8083/connectors' \  
      --header 'Content-Type: application/json' \  
      --data-raw '{  
          "name": "mongo-sink-user",  
          "config": {  
              "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",  
              "connection.uri": "mongodb://mongodbuser:mongodbpw@mongo:27017/",  
              "tasks.max": "1",  
              "topics": "user-result",  
              "database": "my-database",  
              "collection": "user",  
              "key.converter": "org.apache.kafka.connect.storage.StringConverter",  
              "value.converter": "org.apache.kafka.connect.json.JsonConverter",  
              "key.converter.schemas.enable": false,  
              "value.converter.schemas.enable": false,  
              "document.id.strategy.overwrite.existing": true,  
              "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.PartialValueStrategy",  
              "document.id.strategy.partial.value.projection.list": "id,targetId",  
              "document.id.strategy.partial.value.projection.type": "AllowList",  
              "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneBusinessKeyStrategy",  
              "post.processor.chain": "sinkPostProcessor.ObjectIdPostProcessor",  
              "value.projection.list": "id,targetId"  
          }  
      }'  
      ```  

## 추가 - Custom Post Processor 에러 확인
- mongoDB sink connector 생성 시  
  PostProcessor에 에러가 있는 경우 ,  
  kafka-connect container 로그에서 확인할 수 있다.  
- mongoDB sink connector는 생성 후  
  처리 중에 에러가 발생했을 경우,  
  GET connectors/{connector-name}/status 명령으로 확인할 수 있다.  
  ```bash  
  curl --location 'http://localhost:8083/connectors/mongo-sink-user/status'  
  ```  

## 추가 - ObjectIdPostProcessor.jar 파일
- [ObjectIdPostProcessor.jar 파일](https://github.com/a3magic3pocket/copystagram-backend-post-processor/blob/main/ObjectIdPostProcessor-0.0.1.jar){:target="_blank"} 링크에서   
  download raw file을 눌러 다운로드 할 수 있다.  
- ObjectIdPostProcessor를 그대로 사용하고 싶은 경우  
  위 jar 파일만 받아서 mongoDB sink connector plugin 디렉토리에 복사하여  
  사용하면 된다.  

## 참고
- [참고1 - How to convert a String field to ObjectId in MongoSinkConnector](https://www.mongodb.com/community/forums/t/how-to-convert-a-string-field-to-objectid-in-mongosinkconnector/108324/6){:target="_blank"}  
- [참고2 - How to Create a Custom Post Processor](https://www.mongodb.com/docs/kafka-connector/current/sink-connector/fundamentals/post-processors/#how-to-create-a-custom-post-processor){:target="_blank"}  
- [참고3 - List of Java class file format major version numbers?](https://stackoverflow.com/questions/9170832/list-of-java-class-file-format-major-version-numbers){:target="_blank"}  
- [참고4 - Java Platform, Standard Edition 11 Reference Implementations](https://jdk.java.net/java-se-ri/11-MR2){:target="_blank"}  
- [참고5 - 이클립스에서 Gralde Project 생성하기(feat. java-library)](https://jiurinie.tistory.com/123){:target="_blank"}  
- [참고6 - 이클립스에서 gradle 프로젝트 생성하기](https://chinsun9.github.io/2020/10/05/gradle/){:target="_blank"}  
- [참고7 - 이클립스에서 실행 가능한 JAR 파일 만들기](https://bioinfoblog.tistory.com/242){:target="_blank"}  
- [참고 - com.mongodb.kafka.connect.sink.processor.PostProcessor.java](https://github.com/mongodb/mongo-kafka/blob/master/src/main/java/com/mongodb/kafka/connect/sink/processor/PostProcessor.java){:target="_blank"}  
- [참고 - com.mongodb.kafka.connect.sink.processor.KafkaMetaAdder.java](https://github.com/mongodb/mongo-kafka/blob/master/src/main/java/com/mongodb/kafka/connect/sink/processor/KafkaMetaAdder.java){:target="_blank"}  
