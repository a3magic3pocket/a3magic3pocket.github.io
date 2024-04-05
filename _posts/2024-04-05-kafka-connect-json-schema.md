---
title: 카프카 커넥트(kafka connect), json 스키마 사용하기
date: 2024-04-05 11:36:01 +0900
categories: [Kafka]
tags: [kafka, kafka-streams, kafka-connect, json-schema]    # TAG names should always be lowercase
---

## 개요
- 카프카 스트림  데이터를 mongoDB로 보낼때  
  타입을 명시하고 싶을 때가 있다.  
- 이때 json 스키마를 사용할 수 있다.  
- 정해진 포맷에 맞춰 작성 후 데이터를 담아 보내면  
  mongodb가 데이터를 지정된 타입으로 인식하고 저장한다.  
- json 스키마는 매번 스키마를 같이 보내야하기 때문에  
  전송되는 데이터 크기가 그냥 json만 보냈을 때보다 크다.  
- 이때는 스키마 레지스트리(schema registry)를   
  사용하여 스키마를 저장하고,   
  mongoDB에서 이를 참조할 수 있게 하면 크기를 줄일 수 있다.  
- 본 문에서는 json 스키마 사용법만을 기록한다.  

## 예시
- 보내고자 하는 데이터  
    - id 컬럼명  
        - 타입: string  
        - 데이터: my-id  
    - name 컬럼명  
        - 타입: string  
        - 데이터: 민수  
    - age 컬럼명  
        - 타입: int32  
        - 데이터: 17  
- json 스키마  
    - 설명  
        - 크게 schema와 payload 컬럼로 이뤄진다.  
        - schema는 스키마가 담기는 구간으로  
          type, optional, field와 같은 데이터가 저장된다.  
        - payload는 원본 json 데이터가 저장된다.  
        - [[참고1](https://www.confluent.io/ko-kr/blog/kafka-connect-deep-dive-converters-serialization-explained/){:target="_blank"}][[참고2](https://cwiki.apache.org/confluence/display/KAFKA/KIP-301%3A+Schema+Inferencing+for+JsonConverter){:target="_blank"}]  
    - json 스키마 데이터  
      ```json  
      {  
        "schema": {  
          "type": "struct",  
          "fields": [  
            {  
              "type": "string",  
              "optional": false,  
              "field": "id"  
            },  
            {  
              "type": "string",  
              "optional": false,  
              "field": "name"  
            },  
            {  
              "type": "int32",  
              "optional": false,  
              "field": "age"  
            }  
          ],  
          "optional": false,  
          "name": "my-database.my-collection"  
        },  
        "payload": {  
          "id": "my-id",  
          "name": "민수",  
          "age": 17  
        }  
      }  
      ```  

## 카프카, 카프카 스트림, 카프카 커넥트 설정
- 기본적으로 아래 링크와 동일한 설정이며  
  앞으로의 내용은 변경된 부분을 주로 서술한다.  
- [카프카 커넥트(kafka connect)로 mongoDB에 카프카 스트림 쓰기 | 의사줌치 (a3magic3pocket.github.io)](https://a3magic3pocket.github.io/posts/write-kafka-stream-to-mongodb-using-kafka-connect/#%EC%B9%B4%ED%94%84%EC%B9%B4-%EC%84%A4%EC%A0%95){:target="_blank"}  

## 카프카 스트림 설정
- 개요  
    - json 스키마 사용 시, aggregation 후에 Serde를 설정할때  
      deserializer로 JsonSchemaSerde를 설정해야 한다.  
    - JsonSchema.schema.name으로 materialized view가 저장되에  
      기존에 Materialized.as('뷰 이름') 부분은 생략한다.  
- 환경  
    - java 21  
    - spring boot 3.2.3  
- 설치  
  ```bash  
  implementation 'org.apache.kafka:kafka-streams'  
  implementation 'org.springframework.kafka:spring-kafka'  
  ```  
- user/User.java  
  ```java  
  @Setter  
  @Getter  
  public class User {  
      public String id;  
      public String name;  
      public int age;  
  }  
  ```  
- user/UserSerde.java  
  ```java  
  import org.apache.kafka.common.serialization.Serdes.WrapperSerde;  
  import org.springframework.kafka.support.serializer.JsonDeserializer;  
  import org.springframework.kafka.support.serializer.JsonSerializer;  
            
  public class UserSerde extends WrapperSerde<User> {  
      public UserSerde() {  
          super(new JsonSerializer<>(), new JsonDeserializer<>(User.class));  
      }  
  }  
  ```  
- streams/JsonSchema.java  
  ```java  
  @Setter  
  @Getter  
  public class JsonSchema {  
  	public SchemaDto schema;  
  	public Map<String, Object> payload;  
  }  
  ```  
- streams/SchemaDto.java  
  ```java  
  @Getter  
  @Setter  
  public class SchemaDto {  
  	public String type;  
  	public List<FieldsDto> fields;  
  	public boolean optional;  
  	public String name;  
  }  
  ```  
- streams/FieldsDto.java  
  ```java  
  @Getter  
  @Setter  
  public class FieldsDto {  
  	public String type;  
  	public boolean optional;  
  	public String field;  
  }  
  ```  
- streams/JsonSchemaSerde.java  
  ```java  
  import org.apache.kafka.common.serialization.Serdes.WrapperSerde;  
  import org.springframework.kafka.support.serializer.JsonDeserializer;  
  import org.springframework.kafka.support.serializer.JsonSerializer;  
            
  public class JsonSchemaSerde extends WrapperSerde<JsonSchema> {  
  	public JsonSchemaSerde() {  
  		super(new JsonSerializer<>(), new JsonDeserializer<>(JsonSchema.class));  
  	}  
  }  
            
  ```  
- streams/JsonSchemaService.java  
  ```java  
  public class JsonSchemaService {  
      public JsonSchema getDefaultJsonSchema(String schemaName, Map<String, String> fields) {  
          JsonSchema newJsonSchema = new JsonSchema();  
            
          SchemaDto schemaDto = new SchemaDto();  
            
          List<FieldsDto> fieldsDtos = new ArrayList<FieldsDto>();  
          for (String fieldName : fields.keySet()) {  
              FieldsDto fieldsDto = new FieldsDto();  
              fieldsDto.setField(fieldName);  
              fieldsDto.setType(fields.get(fieldName));  
              fieldsDto.setOptional(false);  
              fieldsDtos.add(fieldsDto);  
          }  
            
          // schema name is like "database.collection"  
          schemaDto.setType("struct");  
          schemaDto.setFields(fieldsDtos);  
          schemaDto.setName(schemaName);  
          schemaDto.setOptional(false);  
            
          newJsonSchema.setSchema(schemaDto);  
            
          return newJsonSchema;  
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
            
      @Bean("userDSLBuilder")  
      FactoryBean<StreamsBuilder> userDSLBuilder() {  
          StreamsBuilderFactoryBean streamsBuilder = new StreamsBuilderFactoryBean(  
                  kStreamsConfig("user", UserSerde.class));  
          return streamsBuilder;  
      }  
                
            
      @Bean("userKStream")  
      KStream<String, User> userKStream(@Qualifier("userDSLBuilder") StreamsBuilder userDSLBuilder) {  
          KStream<String, User> kStream = userDSLBuilder.stream("user");  
            
          // @formatter:off  
          kStream  
              .selectKey((key, value) -> {  
                  return value.getId().replace("\"", "");  
              })  
              .groupByKey()  
              .aggregate(  
                  new Initializer<JsonSchema>() {  
                      public JsonSchema apply() {  
                          return new JsonSchema();  
                      }  
                  },  
                  new Aggregator<String, User, JsonSchema>() {  
                      public JsonSchema apply(String key, User value, JsonSchema aggregate) {  
                          JsonSchema jsonSchema = new JsonSchema();  
                          Map<String, String> fields = new HashMap<String, String>();  
                          fields.put("id", "string");  
                          fields.put("name", "string");  
                          fields.put("age", "int32");  
                                    
                          String schemaName = "my-database.user";  
                          JsonSchemaService jsonSchemaService = new JsonSchemaService();  
                                    
                          JsonSchema userJsonSchema = jsonSchemaService.getDefaultJsonSchema(schemaName, fields);  
                                    
                          Map<String, Object> payload = new HashMap<String, Object>();  
                          payload.put("id",value.getId());  
                          payload.put("name", value.getName());  
                          payload.put("age", value.getAge());  
                          userJsonSchema.setPayload(payload);  
                                    
                          aggregate = userJsonSchema;  
                                    
                          return aggregate;  
                      }  
                  },  
                  Materialized.with(Serdes.String(), new JsonSchemaSerde())  
                  // Materialized.as("User")  
              )  
          .toStream()  
          .to("user-result");  
          return kStream;  
      }  
  }  
  ```  

## 카프카 커넥트 설정
- 개요  
    - json schema를 사용하는 옵션을 설정한다.  
      ```json  
      {  
          "value.converter": "org.apache.kafka.connect.json.JsonConverter",   
          "value.converter.schemas.enable": true  
      }  
      ```  
- 생성 명령  
  ```json  
  {  
      "name": "mongo-sink-user",  
      "config": {  
          "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",  
          "connection.uri": "mongodbuser://mongodbpw:mongopw@mongo:27017/",  
          "tasks.max": "1",  
          "topics": "user-result",  
          "database": "my-database",  
          "collection": "user",  
          "key.converter": "org.apache.kafka.connect.storage.StringConverter",  
          "value.converter": "org.apache.kafka.connect.json.JsonConverter",  
          "key.converter.schemas.enable": false,  
          "value.converter.schemas.enable": true,  
          "document.id.strategy.overwrite.existing": true,  
          "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.PartialValueStrategy",  
          "document.id.strategy.partial.value.projection.list": "id",  
          "document.id.strategy.partial.value.projection.type": "AllowList",  
          "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneBusinessKeyStrategy"  
      }  
  ```  

## 참고
- [참고1 - JSON and schemas](https://www.confluent.io/ko-kr/blog/kafka-connect-deep-dive-converters-serialization-explained/){:target="_blank"}  
- [참고2 - KIP-301: Schema Inferencing for JsonConverter](https://cwiki.apache.org/confluence/display/KAFKA/KIP-301%3A+Schema+Inferencing+for+JsonConverter){:target="_blank"}  
