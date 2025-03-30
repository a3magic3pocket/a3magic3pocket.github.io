---
title: 코틀린 스프링 서비스와 레포지토리 스니펫
date: 2025-03-30 23:11:39 +0900
categories: [Kotlin]
tags: [kotlin, summary]    # TAG names should always be lowercase
---

## 최초 Service 세팅
```kotlin  
@Service  
class FruitService(  
  private val fruitRepository: FruitRepository  
) {  
  // 서비스 메서드 추가  
}  
```  

## JPA 레포지토리(Repository) 세팅
```kotlin  
@Repository  
interface FruitRepository : JpaRepository<Fruit, Long> {  
  // 쿼리 메서드 추가  
}  
```  

## JPA 레포지토리(Repository) 이용
- 단일 생성(Create a New Record)  
    - 서비스  
      ```kotlin  
      @Transactional(rollbackFor = [Exception::class])  
      fun createFruit(fruitReqDto: FruitReqDto): Fruit {  
          val fruit = Fruit(  
              name = fruitReqDto.name,  
              origin = fruitReqDto.origin,  
          )  
                
          return fruitRepository.save(fruit)  
      }  
      ```  
- 복수 생성(Create New Records)  
    - application.yml  
      ```yml  
      spring:  
        jpa:  
          hibernate:  
            properties:  
              hibernate.jdbc.batch_size: 50  # 한 번에 처리할 최대 엔티티 수  
              hibernate.order_inserts: true  # update 쿼리 순서 최적화  
              hibernate.batch_versioned_data: true  # 배치에서 버전 관리된 데이터 처리  
      ```  
    - 서비스  
      ```kotlin  
      @Transactional(rollbackFor = [Exception::class])  
      fun createFruits(fruitReqDtos: List<FruitReqDto>): MutableList<Fruit> {  
          val fruits = fruitReqDtos.map {  
              Fruit(  
                  name = it.name,  
                  origin = it.origin,  
              )  
          }  
                
          return fruitRepository.saveAll(fruits)  
      }  
      ```  
- 단일 조회(Retrieve  a Record)  
    - 서비스  
      ```kotlin  
      @Transactional(readOnly = true)  
      fun retrieveFruit(fruitId: Long): Fruit {  
          val fruit = fruitRepository.findByIdOrNull(id = fruitId) ?: throw Exception("not found")  
                
          return fruit  
      }  
      ```  
- 존재여부 조회(Exists)  
  (인자로 받은 name의 가진 행이 존재하면 true, 아니면 false)  
    - 레포지토리  
      ```kotlin  
      @Repository  
      interface FruitRepository : JpaRepository<Fruit, Long> {  
          // 다른 쿼리 메서드  
          fun existsByName(name: String): Boolean  
      }  
      ```  
    - 서비스  
      ```kotlin  
      @Transactional(readOnly = true)  
      fun existsFruitByName(name: String): Boolean {  
          return fruitRepository.existsByName(name = name)  
      }  
      ```  
- 빈도 조회(Count)  
  (인자로 받은 name의 가진 행의 수)  
    - 레포지토리  
      ```kotlin  
      @Repository  
      interface FruitRepository : JpaRepository<Fruit, Long> {  
          // 다른 쿼리 메서드  
          fun countByName(name: String): Long  
      }  
      ```  
    - 서비스  
      ```kotlin  
      @Transactional(readOnly = true)  
      fun countFruitsByName(name: String): Long {  
          return fruitRepository.countByName(name = name)  
      }  
      ```  
- 목록 조회(Retrieve List of Records)  
    - 전체 목록 조회  
        - 서비스  
          ```kotlin  
          @Transactional(readOnly = true)  
          fun listFruits(): List<Fruit> {  
              val fruits = fruitRepository.findAll()  
                    
              return fruits.toList()  
          }  
          ```  
    - equal 조건이 포함된 목록 조회  
      (인자로 받은 name과 동일한 name을 가진 행 조회)  
        - 레포지토리  
          ```kotlin  
          @Repository  
          interface FruitRepository : JpaRepository<Fruit, Long> {  
              // 다른 쿼리 메서드  
              fun findByName(name: String): List<Fruit>  
          }  
          ```  
        - 서비스  
          ```kotlin  
          @Transactional(readOnly = true)  
          fun listFruitByName(name: String): List<Fruit> {  
              val fruits = fruitRepository.findByName(name = name)  
              if (fruits.isEmpty()) {  
                  throw Exception("not found")  
              }  
                    
              return fruits  
          }  
          ```  
    - IN 조건을 활용한 목록 조회  
      (인자로 받은 origin 목록에 속하는 행 목록 조회)  
        - 레포지토리  
          ```kotlin  
          @Repository  
          interface FruitRepository : JpaRepository<Fruit, Long> {  
              // 다른 쿼리 메서드  
              fun findByOriginIn(origins: List<String>): List<Fruit>  
          }  
          ```  
        - 서비스  
          ```kotlin  
          @Transactional(readOnly = true)  
          fun listFruitsInOrigin(origins: List<String>): List<Fruit> {  
              val fruits = fruitRepository.findByOriginIn(origins = origins)  
              if (fruits.isEmpty()) {  
                  throw Exception("not found")  
              }  
                    
              return fruits  
          }  
          ```  
    - LIKE 조건을 활용한 목록 조회 - 부분 일치  
      (%LIKE% 조건이라 인덱스 안 탐)  
        - 레포지토리  
          ```kotlin  
          @Repository  
          interface FruitRepository : JpaRepository<Fruit, Long> {  
              // 다른 쿼리 메서드  
              fun findByNameContaining(name: String): List<Fruit>  
          }  
          ```  
        - 서비스  
          ```kotlin  
          @Transactional(readOnly = true)  
          fun listFruitsLikeName(search: String): List<Fruit> {  
              val fruits = fruitRepository.findByNameContaining(search = search)  
              if (fruits.isEmpty()) {  
                  throw Exception("not found")  
              }  
                    
              return fruits  
          }  
          ```  
    - LIKE 조건을 활용한 목록 조회 - 앞부분 일치  
      (LIKE% 조건이라 인덱스 탐)  
        - 레포지토리  
          ```kotlin  
          @Repository  
          interface FruitRepository : JpaRepository<Fruit, Long> {  
              // 다른 쿼리 메서드  
              fun findByNameStartingWith(name: String): List<Fruit>  
          }  
          ```  
        - 서비스  
          ```kotlin  
          @Transactional(readOnly = true)  
          fun listFruitsNameStartingWith(search: String): List<Fruit> {  
              val fruits = fruitRepository.findByNameStartingWith(name = search)  
              if (fruits.isEmpty()) {  
                  throw Exception("not found")  
              }  
                    
              return fruits  
          }  
          ```  
    - LIKE 조건을 활용한 목록 조회 - 뒷부분 일치  
      (%LIKE 조건이라 인덱스 탐)  
        - 레포지토리  
          ```kotlin  
          @Repository  
          interface FruitRepository : JpaRepository<Fruit, Long> {  
              // 다른 쿼리 메서드  
              fun findByNameEndingWith(name: String): List<Fruit>  
          }  
          ```  
        - 서비스  
          ```kotlin  
          @Transactional(readOnly = true)  
          fun listFruitsNameEndingWith(search: String): List<Fruit> {  
              val fruits = fruitRepository.findByNameEndingWith(name = search)  
              if (fruits.isEmpty()) {  
                  throw Exception("not found")  
              }  
                    
              return fruits  
          }  
          ```  
    - 정렬 추가(Sort)  
      (name으로 오름차순, origin으로 내림차순 정렬)  
        - 레포지토리  
          ```kotlin  
          @Repository  
          interface FruitRepository : JpaRepository<Fruit, Long> {  
              // 다른 쿼리 메서드  
              fun findAllByOrderByNameAscOriginDesc(): List<Fruit>  
          }  
          ```  
        - 서비스  
          ```kotlin  
          @Transactional(readOnly = true)  
          fun listFruitsOrderByNameAscAndOriginDesc(): List<Fruit> {  
              val fruits = fruitRepository.findAllByOrderByNameAscOriginDesc()  
              if (fruits.isEmpty()) {  
                  throw Exception("not found")  
              }  
                    
              return fruits  
          }  
          ```  
- 페이지네이션(Pagination)  
    - 레포지토리  
      ```kotlin  
      import org.springframework.data.domain.Page  
      import org.springframework.data.domain.Pageable  
                
      @Repository  
      interface FruitRepository : JpaRepository<Fruit, Long> {  
          // 다른 쿼리 메서드  
          fun findByOrigin(origin: String, pageable: Pageable): Page<Fruit>  
      }  
      ```  
    - 서비스  
      ```kotlin  
      @Transactional(readOnly = true)  
      fun listFruitsPageable(origin: String, pageable: Pageable): Page<Fruit> {  
          return  fruitRepository.findByOrigin(  
              origin = origin,  
              pageable = pageable  
          )  
      }  
      ```  
- 단일 갱신(Update a New Record)  
    - dto  
      ```kotlin  
      data class UpdateDto<T>(  
          val id: Long,  
          val data: T  
      )  
      ```  
    - 서비스  
      ```kotlin  
      @Transactional(rollbackFor = [Exception::class])  
      fun updateFruit(fruitUpdateDto: UpdateDto<FruitReqDto>): Fruit {  
          val savedFruit = fruitRepository.findByIdOrNull(id = fruitUpdateDto.id) ?: throw Exception("not found")  
                
          savedFruit.origin = fruitUpdateDto.data.origin  
          savedFruit.name = fruitUpdateDto.data.name  
                
          return savedFruit  
      }  
      ```  
- 복수 갱신(Update New Records)  
    - application.yml  
      ```yml  
      spring:  
        jpa:  
          hibernate:  
            properties:  
              hibernate.jdbc.batch_size: 50  # 한 번에 처리할 최대 엔티티 수  
              hibernate.order_updates: true  # update 쿼리 순서 최적화  
              hibernate.batch_versioned_data: true  # 배치에서 버전 관리된 데이터 처리  
      ```  
    - 서비스  
      ```kotlin  
      @Transactional(rollbackFor = [Exception::class])  
      fun updateFruits(fruitUpdateDtos: List<UpdateDto<FruitReqDto>>): List<Fruit> {  
          val fruitReqDtoMap = fruitUpdateDtos.associate { it.id to it.data }  
          val savedFruits = fruitRepository.findByIdIn(ids = fruitReqDtoMap.keys.toList())  
          for (savedFruit in savedFruits) {  
              val fruitReqDto = fruitReqDtoMap[savedFruit.id] ?: throw Exception("not found fruit ${savedFruit.id}")  
                
              savedFruit.origin = fruitReqDto.origin  
              savedFruit.name = fruitReqDto.name  
              savedFruit.updatedAt = ZonedDateTime.now()  
          }  
                
          return savedFruits  
      }  
      ```  
- 하드 삭제(Hard Delete)  
    - 서비스  
      ```kotlin  
      @Transactional(rollbackFor = [Exception::class])  
      fun deleteFruit(id: Long) {  
          fruitRepository.deleteById(id)  
      }  
      ```  
- 소프트 삭제(Soft Delete)  
    - 서비스  
      ```kotlin  
      @Transactional(rollbackFor = [Exception::class])  
      fun softDeleteFruit(id: Long) {  
          val fruit = fruitRepository.findByIdOrNull(id = id) ?: throw Exception("not found")  
                
          fruit.isDeleted = true  
          fruit.updatedAt = ZonedDateTime.now()  
                
          fruitRepository.save(fruit)  
      }  
      ```  

## 참고
- Fruit 엔티티  
  ```kotlin  
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
