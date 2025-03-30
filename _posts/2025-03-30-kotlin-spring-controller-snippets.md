---
title: 코틀린 스프링 컨트롤러 스니펫
date: 2025-03-30 23:14:16 +0900
categories: [Kotlin]
tags: [kotlin, summary]    # TAG names should always be lowercase
---

## 최초 RestController 세팅
```kotlin  
package kotlins.pring.snippet.controller  
        
import org.springframework.web.bind.annotation.RequestMapping  
import org.springframework.web.bind.annotation.RestController  
import kotlins.pring.snippet.service.FruitService  
        
@RestController  
@RequestMapping("/v1/fruits")  
class FruitController(  
  private val fruitService: FruitService,  
) {  
  companion object {  
      const val FRUIT_ID = "fruit_id"  
      const val FRUIT_NAME = "fruit_name"  
      const val FRUIT_ORIGIN = "fruit_origin"  
      const val FRUIT_FILE = "file"  
  }  
        
  // 컨트롤러 메서드 추가  
        
}  
```  

## 파라미터
- 패스 파라미터(path parameter)  
  ```kotlin  
  import org.springframework.http.MediaType  
  import org.springframework.http.ResponseEntity  
  import org.springframework.web.bind.annotation.GetMapping  
  import org.springframework.web.bind.annotation.PathVariable  
            
  @GetMapping(value = ["/{$FRUIT_ID:[0-9]+}"], produces = [MediaType.APPLICATION_JSON_VALUE])  
  fun retrieveFruit(  
      @PathVariable(name = FRUIT_ID) fruitId: Long,  
  ): ResponseEntity<DataRes<FruitResDto>> {  
      val savedFruit = fruitService.retrieveFruit(  
          fruitId = fruitId  
      )  
            
      return ResponseEntity.ok().body(  
          DataRes(  
              FruitResDto(  
                  id = savedFruit.id!!,  
                  name = savedFruit.name,  
                  origin = savedFruit.origin  
              )  
          )  
      )  
  }  
  ```  
- 쿼리 파라미터(query parameter)  
  ```kotlin  
  @GetMapping(value = ["/search"], produces = [MediaType.APPLICATION_JSON_VALUE])  
  fun listFruitByName(  
      @RequestParam(name = FRUIT_NAME) name: String,  
  ): ResponseEntity<DataRes<List<FruitResDto>>> {  
      val savedFruits = fruitService.listFruitByName(  
          name = name  
      )  
            
      return ResponseEntity.ok().body(  
          DataRes(  
              savedFruits.map {  
                  FruitResDto(  
                      id = it.id!!,  
                      name = it.name,  
                      origin = it.origin  
                  )  
              }  
          )  
      )  
  }  
  ```  
- 바디(body) - json  
    - dto  
      ```kotlin  
      @JsonInclude(JsonInclude.Include.NON_NULL)  
      data class FruitReqDto (  
          @JsonProperty("fruit_name")  
          val name: String,  
                
          @JsonProperty("fruit_origin")  
          val origin: String,  
      )  
      ```  
    - 컨트롤러  
      ```kotlin  
      @PostMapping(value = [""], produces = [MediaType.APPLICATION_JSON_VALUE])  
      fun createFruit(  
          @RequestBody fruitReqDto: FruitReqDto  
      ): ResponseEntity<DataRes<FruitResDto>> {  
          val savedFruit = fruitService.createFruit(  
              fruitReqDto = fruitReqDto  
          )  
                
          return ResponseEntity.ok().body(  
              DataRes(  
                  FruitResDto(  
                      id = savedFruit.id!!,  
                      name = savedFruit.name,  
                      origin = savedFruit.origin  
                  )  
              )  
          )  
      }  
      ```  
- 바디(body) - form-data 또는 x-www-form-urlencoded  
    - dto  
      ```kotlin  
      // @ModelAttribute 사용 시 form-data 필드명과 dto 필드명이 항상 일치해야 함  
      data class FruitReqFormDataDto (  
          val name: String,  
          val origin: String,  
      )  
      ```  
    - 컨트롤러  
      ```kotlin  
      @PostMapping(value = ["/form-data"], produces = [MediaType.APPLICATION_JSON_VALUE])  
      fun createFruitByFormData(  
          @ModelAttribute fruitReqFormDataDto: FruitReqFormDataDto  
      ): ResponseEntity<DataRes<FruitResDto>> {  
          val savedFruit = fruitService.createFruitByFormData(  
              fruitReqFormDataDto = fruitReqFormDataDto  
          )  
                
          return ResponseEntity.ok().body(  
              DataRes(  
                  FruitResDto(  
                      id = savedFruit.id!!,  
                      name = savedFruit.name,  
                      origin = savedFruit.origin  
                  )  
              )  
          )  
      }  
                
      ```  
    - 혹은 @QueryParam 사용  
      ```kotlin  
      @PostMapping(value = ["/form-data-request-param"], produces = [MediaType.APPLICATION_JSON_VALUE])  
      fun createFruitByFormDataRequestParam(  
          @RequestParam(name = FRUIT_NAME) name: String,  
          @RequestParam(name = FRUIT_ORIGIN) origin: String,  
      ): ResponseEntity<DataRes<FruitResDto>> {  
          val savedFruit = fruitService.createFruitByFormData(  
              fruitReqFormDataDto = FruitReqFormDataDto(  
                  name = name,  
                  origin = origin  
              )  
          )  
                
          return ResponseEntity.ok().body(  
              DataRes(  
                  FruitResDto(  
                      id = savedFruit.id!!,  
                      name = savedFruit.name,  
                      origin = savedFruit.origin  
                  )  
              )  
          )  
      }  
      ```  
- 바디(body) - 파일  
    - dto  
      ```kotlin  
      data class FruitFileDto (  
          val name: String,  
          val fruitName: String,  
          val fruitOrigin: String,  
      )  
      ```  
    - 컨트롤러  
      ```kotlin  
      @PostMapping(value = ["/file"], produces = [MediaType.APPLICATION_JSON_VALUE])  
      fun createFruitWithFile(  
          @RequestPart(FRUIT_FILE) file: MultipartFile,  
          @ModelAttribute fruitFileDto: FruitFileDto,  
      ): ResponseEntity<DataRes<FruitResDto>> {  
          val savedFruit = fruitService.createFruitWithFile(  
              file = file,  
              fruitFileDto = fruitFileDto  
          )  
                
          return ResponseEntity.ok().body(  
              DataRes(  
                  FruitResDto(  
                      id = savedFruit.id!!,  
                      name = savedFruit.name,  
                      origin = savedFruit.origin  
                  )  
              )  
          )  
      }  
      ```  

## 유효성 검사
- javax.validation 의 유효성 검사 애너테이션을 DTO에 적용  
- [javax.validation 애너테이션 목록](https://jakarta.ee/specifications/bean-validation/3.0/apidocs/jakarta/validation/constraints/package-summary){:target="_blank"}  
- dto  
  ```kotlin  
  @JsonInclude(JsonInclude.Include.NON_NULL)  
  data class FruitSizeDto(  
      @field:NotNull(message = "null 허용 안 함")  
      @field:NotEmpty(message = "\"\" 허용 안 함")  
      @JsonProperty("fruit_name")  
      val name: String,  
            
      @JsonProperty("fruit_origin")  
      val origin: String,  
            
      @field:Min(value = 1, message = "1 이상이어야 함")  
      @field:Max(value = 10, message = "10 이하이어야 함")  
      val size: Int,  
  )  
  ```  
- 컨트롤러  
  ```kotlin  
  @PostMapping(value = ["/size"], produces = [MediaType.APPLICATION_JSON_VALUE])  
  fun createFruitWithSize(  
      @RequestBody @Valid fruitSizeDto: FruitSizeDto,  
  ): ResponseEntity<DataRes<FruitResDto>> {  
      val savedFruit = fruitService.createFruitWithSize(  
          fruitSizeDto = fruitSizeDto  
      )  
            
      return ResponseEntity.ok().body(  
          DataRes(  
              FruitResDto(  
                  id = savedFruit.id!!,  
                  name = savedFruit.name,  
                  origin = savedFruit.origin  
              )  
          )  
      )  
  }  
  ```  

## 페이지네이션
- dto  
  ```kotlin  
  data class DataPageableRes<T>(  
      val totalCount: Long,  
      val data: List<T>,  
      val page: PageInfo  
  )  
            
  data class PageInfo(  
      val totalPages: Int,  
      val number: Int,  
      val size: Int,  
  )  
            
  ```  
- 컨트롤러  
  ```kotlin  
  @GetMapping(value = [""], produces = [MediaType.APPLICATION_JSON_VALUE])  
  fun listFruitsPageableByOrigin(  
      @RequestParam(name = FRUIT_ORIGIN) origin: String,  
      pageable: Pageable,  
  ): ResponseEntity<DataPageableRes<Fruit>> {  
      val fruitPage = fruitService.listFruitsPageable(  
          origin = origin,  
          pageable = pageable  
      )  
            
      return ResponseEntity.ok().body(  
          DataPageableRes(  
              totalCount = fruitPage.totalElements,  
              data = fruitPage.content,  
              page = PageInfo(  
                  totalPages = fruitPage.totalPages,  
                  number = fruitPage.number,  
                  size = fruitPage.size  
              )  
          )  
      )  
  }  
  ```  

## 참고
- Fruit 응답 용 DTO  
  ```kotlin  
  data class FruitResDto (  
      val id: Long,  
      val name: String,  
      val origin: String,  
  )  
  ```  
- 응답 용 DTO  
  ```kotlin  
  data class DataRes<T>(  
      val data: T  
  )  
  ```  
