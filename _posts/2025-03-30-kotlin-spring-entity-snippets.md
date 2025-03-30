---
title: 코틀린 스프링 엔티티 스니펫
date: 2025-03-30 23:10:52 +0900
categories: [Kotlin]
tags: [kotlin, summary]    # TAG names should always be lowercase
---

## 과일(fruit) 엔티티
```kotlin  
package kotlins.pring.snippet.entity  
        
import jakarta.persistence.*  
import kotlins.pring.snippet.config.converter.BooleanYNConverter  
import org.hibernate.annotations.CreationTimestamp  
import org.hibernate.annotations.UpdateTimestamp  
import java.time.ZonedDateTime  
        
/**  
* 과일  
*/  
@Entity  
@Table(name = "fruit")  
data class Fruit(  
  @Id  
  @GeneratedValue(strategy = GenerationType.IDENTITY)  
  @Column(name = "id", nullable = false, insertable = false, updatable = false)  
  var id: Long? = null,  
        
  @Column(name = "name", nullable = false)  
  var name: String,  
        
  @Column(name = "origin", nullable = false)  
  var origin: String,  
        
  @Column(name = "size", nullable = true)  
  var size: Int = 0,  
        
  @Column(name = "freshness", nullable = false)  
  @Enumerated(EnumType.STRING)  
  var freshness: FruitFreshness = FruitFreshness.FRESH,  
        
  @Column(name = "is_deleted", nullable = false)  
  @Convert(converter = BooleanYNConverter::class)  
  var isDeleted: Boolean = false,  
        
  @Column(name = "created_at", nullable = true, insertable = false, updatable = false)  
  @CreationTimestamp  
  var createdAt: ZonedDateTime? = null,  
        
  @Column(name = "updated_at", nullable = true, insertable = true, updatable = true)  
  @UpdateTimestamp  
  var updatedAt: ZonedDateTime? = null  
        
)  
```  

## 과일(fruit) 테이블 생성 SQL
```sql  
CREATE TABLE fruit (  
  id BIGINT AUTO_INCREMENT PRIMARY KEY,          -- id: 자동 증가하는 기본 키  
  name VARCHAR(255) NOT NULL,                     -- name: 이름, 비어 있을 수 없음  
  origin VARCHAR(255) NOT NULL,                   -- origin: 원산지, 비어 있을 수 없음  
  size INT DEFAULT 0,                             -- size: 크기, 기본값은 0  
  freshness VARCHAR(100) NOT NULL,  -- freshness: 신선도, 'FRESH', 'RIPE', 'ROTTEN' 중 하나  
  is_deleted CHAR(1) NOT NULL DEFAULT 'N',  -- is_deleted: 삭제 여부, 'Y' 또는 'N' 값, 기본값은 'N'  
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP, -- created_at: 생성 시 자동 설정  
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, -- updated_at: 수정 시 자동 설정  
);  
        
```  

## 주요 애너테이션(annotation) 설명
- @Id  
    - primary key 필드를 지정  
- @GeneratedValue(strategy = GenerationType.IDENTITY)  
    - primary key 생성 전략 설정  
    - GenerationType.IDENTITY 는 auto-increment 기능 사용을 의미  
- @Column  
    - name  
        - 테이블의 컬럼명 지정  
    - nullable  
        - null 허용 여부  
    - insertable  
        - true 이면 해당 컬럼이 INSERT 쿼리에 값이 포함되도록 설정  
    - updatable  
        - true 이면 해당 컬럼이 UPDATE 쿼리에 값이 포함되도록 설정  
    - length  
        - 문자열 컬럼의 길이 지정  
- @CreationTimestamp  
    - Hibernate에서 제공하는 애너테이션으로 엔티티 객체가 처음 저장될 때  
      자동으로 현재 시간 설정  
    - ZonedDateTime, LocalDateTime, Date와 같은 타입 지정 가능  
    - ZonedDateTime의 경우 현재 스프링이 실행되는 서버 타임존 기준  
- @UpdateTimestamp  
    - Hibernate에서 제공하는 애너테이션으로 엔티티가 업데이트될 때마다  
      해당 필드에 자동으로 현재 시간을 설정  
    - 나머지는 @CreationTimestamp 와 동일  
- @Enumerated  
    - JPA에서 Enum 타입을 데이터베이스에서 저장할 때의 저장 방식 지정  
    - EnumType.ORDINAL(기본값)  
        - Enum 값의 인덱스(순서)를 정수 값으로 저장  
    - EnumType.STRING  
        - Enum 값의 이름(String) 그대로 저장  
    - *Mysql, MariaDB 를 사용하는 경우  
        - columnDefinition 으로 DB ENUM 타입을 사용하게 할 수 있다.  
          ```kotlin  
          @Column(name = "freshness", columnDefinition = "ENUM('FRESH','RIPE','ROTTEN')")  
          var freshness: FruitFreshness = FruitFreshness.FRESH,  
          ```  
        - 운영 중에 변경이 일어날 것으로 예상되면   
          ENUM 대신 VARCHAR로 저장하자.  

## 컨버터(converter)
- Boolean 타입을 DB에 저장할때 true를 'Y'로, false를 'N'으로 변환하고,  
  DB에서 읽어올 때에는 'Y'를 true로, 'N'을 false로 변환  
- booleanYNConverter  
  ```kotlin  
  package kotlins.pring.snippet.config.converter  
            
  import jakarta.persistence.AttributeConverter  
  import jakarta.persistence.Converter  
            
  @Converter(autoApply = false)  
  class BooleanYNConverter: AttributeConverter<Boolean, String> {  
            
      // boolean 값을 Y/N로 변환  
      override fun convertToDatabaseColumn(attribute: Boolean?): String? {  
          return if (attribute == null) null else if (attribute) "Y" else "N"  
      }  
            
      // Y/N 값을 boolean 으로 변환  
      override fun convertToEntityAttribute(dbData: String?): Boolean {  
          return dbData == "Y"  
      }  
  }  
            
  ```  
