---
title: 코틀린 스프링 ExceptionAdvice 스니펫
date: 2025-03-30 23:15:06 +0900
categories: [Kotlin]
tags: [kotlin, summary]    # TAG names should always be lowercase
---

## 최초 ExceptionAdvice 세팅
```kotlin  
package kotlins.pring.snippet.config  
        
import org.springframework.web.bind.annotation.RestControllerAdvice  
        
@RestControllerAdvice  
class ExceptionAdvice {  
        
}  
```  

## Jackson parsing 실패 에러 - 필수 필드 누락
- 발생 이유  
    - dto의 non-nullable 한 필드를 요청에서 누락한다면  
      jackson parsing 에러가 발생한다.  
    - @field:NotNull 또는 @field:NotEmpty 조건을 위배해서   
      발생하는 에러가 아니다.  
- dto  
  ```kotlin  
  @JsonInclude(JsonInclude.Include.NON_NULL)  
  data class FruitSizeDto(  
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
- Controller  
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
- ExceptionAdvice  
  ```kotlin  
  @ExceptionHandler(HttpMessageNotReadableException::class)  
  @ResponseBody  
  fun handleHttpMessageNotReadableException(  
      httpServletRequest: HttpServletRequest,  
      httpMessageNotReadableException: HttpMessageNotReadableException  
  ): ResponseEntity<ErrorRes> {  
      val unknownFieldName = "unknown"  
      val cause = httpMessageNotReadableException.cause  
      val fieldErrors = mutableListOf<FieldError>()  
      val code = "400"  
      val status = HttpStatus.BAD_REQUEST  
      var message = "some fields are invalid"  
            
      when (cause) {  
          is MismatchedInputException -> {  
              message = "some fields are missing"  
              val fieldName = cause.path.joinToString(" -> ") {  
                  if (it.index < 0) {  
                      it.fieldName ?: unknownFieldName  
                  } else {  
                      it.index.toString()  
                  }  
              }  
              fieldErrors.add(  
                  FieldError(  
                      field = fieldName,  
                      message = "$fieldName is required"  
                  )  
              )  
          }  
            
          else -> {  
              println("다른 예외 발생")  
          }  
      }  
            
      val errorRes = ErrorRes(  
          code = code,  
          timestamp = Date(),  
          path = httpServletRequest.requestURI,  
          message = message,  
          fieldErrors = fieldErrors  
      )  
            
      return ResponseEntity(errorRes, status)  
  }  
  ```  
- 요청  
    - url  
        - http://localhost:8080/v1/fruits/size  
    - method  
        - POST  
    - body  
      ```json  
      {  
          // fruit_name는 필수값  
          // "fruit_name": "바나나",  
          "fruit_origin": "필리핀",  
          "size": 1  
      }  
      ```  
- 응답  
  ```json  
  {  
      "code": "404",  
      "timestamp": "2025-03-17T08:31:45.193+00:00",  
      "path": "/v1/fruits/size",  
      "message": "some fields are missing",  
      "fieldErrors": [  
          {  
              "field": "fruit_name",  
              "message": "fruit_name is required"  
          }  
      ]  
  }  
  ```  

## Jackson parsing 실패 에러 - 필드 타입 오류
- 발생 이유  
    - dto 의 필드 타입과 요청에서 매칭되는 필드 타입이 다른 경우 발생  
- dto, Controller  
    - '필수 필드 누락' 과 동일  
- ExceptionAdvice  
  ```kotlin  
  @ExceptionHandler(HttpMessageNotReadableException::class)  
  @ResponseBody  
  fun handleHttpMessageNotReadableException(  
      httpServletRequest: HttpServletRequest,  
      httpMessageNotReadableException: HttpMessageNotReadableException  
  ): ResponseEntity<ErrorRes> {  
      val unknownFieldName = "unknown"  
      val cause = httpMessageNotReadableException.cause  
      val fieldErrors = mutableListOf<FieldError>()  
      val code = "400"  
      val status = HttpStatus.BAD_REQUEST  
      var message = "some fields are invalid"  
            
      when (cause) {  
          is InvalidFormatException -> {  
              val fieldName = cause.path.joinToString(" -> ") {  
                  if (it.index < 0) {  
                      it.fieldName ?: unknownFieldName  
                  } else {  
                      it.index.toString()  
                  }  
              }  
              val invalidValue = cause.value  
              val expectedType = cause.targetType.simpleName  
            
              fieldErrors.add(  
                  FieldError(  
                      field = fieldName,  
                      message = "The value of '$fieldName' is invalid: '$invalidValue' (expected: $expectedType)"  
                  )  
              )  
          }  
            
          is MismatchedInputException -> {  
              message = "some fields are missing"  
              val fieldName = cause.path.joinToString(" -> ") {  
                  if (it.index < 0) {  
                      it.fieldName ?: unknownFieldName  
                  } else {  
                      it.index.toString()  
                  }  
              }  
              fieldErrors.add(  
                  FieldError(  
                      field = fieldName,  
                      message = "$fieldName is required"  
                  )  
              )  
          }  
            
          else -> {  
              println("다른 예외 발생")  
          }  
      }  
            
      val errorRes = ErrorRes(  
          code = code,  
          timestamp = Date(),  
          path = httpServletRequest.requestURI,  
          message = message,  
          fieldErrors = fieldErrors  
      )  
            
      return ResponseEntity(errorRes, status)  
  }  
  ```  
- 요청  
    - url  
        - http://localhost:8080/v1/fruits/size  
    - method  
        - POST  
    - body  
      ```json  
      {  
          "fruit_name": "바나나",  
          "fruit_origin": "필리핀",  
          // size는 Int 타입만 허용  
          "size": "숫자만허용"  
      }  
      ```  
- 응답  
  ```json  
  {  
      "code": "400",  
      "timestamp": "2025-03-18T00:38:48.311+00:00",  
      "path": "/v1/fruits/size",  
      "message": "some fields are invalid",  
      "fieldErrors": [  
          {  
              "field": "size",  
              "message": "The value of 'size' is invalid: '숫자만허용' (expected: int)"  
          }  
      ]  
  }  
  ```  

## jakarta.validation 에러
- 발생 이유  
    - jakarta.validation 조건 위배 시 발생  
    - 예시로  jakarta.validation.constraints.NotEmpty 조건을  
      위배하는 상황을 기록한다.  
    - 예시 대상 필드  
      ```kotlin  
      @field:NotEmpty(message = "\"\" 허용 안 함")  
      @JsonProperty("fruit_name")  
      val name: String,  
      ```  
- dto, Controller  
    - '필수 필드 누락' 과 동일  
- ExceptionAdvice  
  ```kotlin  
  @ResponseStatus(HttpStatus.BAD_REQUEST)  
  @ExceptionHandler(MethodArgumentNotValidException::class)  
  @ResponseBody  
  fun handleMethodArgumentNotValidException(  
      httpServletRequest: HttpServletRequest,  
      methodArgumentNotValidException: MethodArgumentNotValidException,  
  ): ErrorRes {  
      val code = "400"  
      val message = "some fields are invalid"  
      val fieldErrors = methodArgumentNotValidException.bindingResult.fieldErrors.map { fieldError ->  
          FieldError(  
              field = fieldError.field,  
              message = fieldError.defaultMessage ?: "Invalid value"  
          )  
      }  
            
      return ErrorRes(  
          code = code,  
          timestamp = Date(),  
          path = httpServletRequest.requestURI,  
          message = message,  
          fieldErrors = fieldErrors  
      )  
  }  
  ```  
- 요청  
    - url  
        - http://localhost:8080/v1/fruits/size  
    - method  
        - POST  
    - body  
      ```json  
      {  
          // fruit_name에는 빈 문자열 입력 금지  
          "fruit_name": "",  
          "fruit_origin": "필리핀",  
          "size": 1  
      }  
      ```  
- 응답  
  ```json  
  {  
      "code": "400",  
      "timestamp": "2025-03-18T01:11:54.717+00:00",  
      "path": "/v1/fruits/size",  
      "message": "some fields are invalid",  
      "fieldErrors": [  
          {  
              "field": "name",  
              "message": "\"\" 허용 안 함"  
          }  
      ]  
  }  
  ```  

## 중복 값 입력 에러
- 발생 이유  
    - soft delete를 지원하는 상황에서 DB의 UNIQUE 필드 제약조건으로는  
      중복 값 입력을 방지할 수 없다.  
    - 또한 업데이트 시 중복 값 입력을 판별해야 하는 경우  
      업데이트 대상을을 제외한 나머지 행의 해당 필드 값이 입력한 값과  
      중복되는지 확인해야한다.  
    - 서비스에서 중복 값을 쉽게 확인할 수 있도록  
      DuplicateCheckService를 만들어서 처리한다.  
- Exception  
    - DuplicateFieldException  
      ```kotlin  
      class DuplicateFieldException(  
          val field: String,  
          val value: String,  
          message: String = "The value '$value' for field '$field' is already in use.", // 기본 오류 메시지  
          val code: String = "400",  
      ) : RuntimeException(message)  
      ```  
    - NotFoundException  
      ```kotlin  
      class NotFoundException(  
          message: String = "not found",  
          val code: String = "404",  
      ) : RuntimeException(message)  
      ```  
    - ServiceException  
      ```kotlin  
      class ServiceException(  
          message: String = "server error occurred",  
          cause: Throwable? = null,  
          val code: String = "500",  
      ) : RuntimeException(message, cause)  
      ```  
- DuplicateCheckService  
  ```kotlin  
  @Service  
  class DuplicateCheckService(  
      private val entityManager: EntityManager  
  ) {  
      companion object {  
          const val ID_ENTITY_FIELD_NAME = "id"  
          const val IS_DELETED_ENTITY_FIELD_NAME = "isDeleted"  
      }  
            
      // 중복 체크 메서드  
      fun checkForDuplicate(  
          entityClass: KClass<*>,  
          fieldName: String,  
          fieldValue: Any,  
          idValue: Long? = null, // 업데이트 시 id 제공  
          useSoftDelete: Boolean = true  
      ) {  
          try {  
              val count = executeDuplicateCheckQuery(entityClass, fieldName, fieldValue, idValue, useSoftDelete)  
              if (count > 0) {  
                  throw DuplicateFieldException(  
                      field = fieldName,  
                      value = fieldValue.toString(),  
                  )  
              }  
          } catch (e: DuplicateFieldException) {  
              throw e  
          } catch (e: PersistenceException) {  
              throw ServiceException("duplicate check error - database access failure", e)  
          } catch (e: Exception) {  
              throw ServiceException("duplicate check error - unexpected error occurred", e)  
          }  
      }  
            
      private fun executeDuplicateCheckQuery(  
          entityClass: KClass<*>,  
          fieldName: String,  
          fieldValue: Any,  
          idValue: Long?,  
          useSoftDelete: Boolean  
      ): Long {  
          val criteriaBuilder: CriteriaBuilder = entityManager.criteriaBuilder  
          val criteriaQuery: CriteriaQuery<Long> = criteriaBuilder.createQuery(Long::class.java)  
          val root: Root<*> = criteriaQuery.from(entityClass.java)  
            
          val predicates = ArrayList<Predicate>()  
            
          // 필드 값 조건 설정 (e.$fieldName = :value)  
          predicates.add(criteriaBuilder.equal(root.get<Any>(fieldName), fieldValue))  
            
          // ID 조건 설정 (e.id != idValue)  
          idValue?.let {  
              predicates.add(criteriaBuilder.notEqual(root.get<Any>(ID_ENTITY_FIELD_NAME), it))  
          }  
            
          // 활성화된 데이터 조건 설정 (optional)  
          if (useSoftDelete) {  
              predicates.add(criteriaBuilder.equal(root.get<String>(IS_DELETED_ENTITY_FIELD_NAME), false))  
          }  
            
          // 쿼리 실행  
          criteriaQuery.select(criteriaBuilder.count(root))  
              .where(criteriaBuilder.and(*predicates.toTypedArray()))  
            
          return entityManager.createQuery(criteriaQuery).singleResult  
      }  
  }  
  ```  
- ExceptionAdvice  
  ```kotlin  
  @RestControllerAdvice  
  class ExceptionAdvice {  
            
      @ResponseStatus(HttpStatus.BAD_REQUEST)  
      @ExceptionHandler(DuplicateFieldException::class)  
      @ResponseBody  
      fun handleDuplicateFieldException(  
          httpServletRequest: HttpServletRequest,  
          duplicateFieldException: DuplicateFieldException,  
      ): ErrorRes {  
          return ErrorRes(  
              code = duplicateFieldException.code,  
              timestamp = Date(),  
              path = httpServletRequest.requestURI,  
              message = "there are duplicate fields",  
              fieldErrors = listOf(  
                  FieldError(  
                      field = duplicateFieldException.field,  
                      message = duplicateFieldException.message ?: "${duplicateFieldException.field} is duplicate"  
                  )  
              )  
          )  
      }  
            
      @ResponseStatus(HttpStatus.NOT_FOUND)  
      @ExceptionHandler(NotFoundException::class)  
      @ResponseBody  
      fun handleNotFoundException(  
          httpServletRequest: HttpServletRequest,  
          notFoundException: NotFoundException,  
      ): ErrorRes {  
          return ErrorRes(  
              code = notFoundException.code,  
              timestamp = Date(),  
              path = httpServletRequest.requestURI,  
              message = "not found",  
              fieldErrors = listOf()  
          )  
      }  
            
      @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)  
      @ExceptionHandler(ServiceException::class)  
      @ResponseBody  
      fun handleServiceException(  
          httpServletRequest: HttpServletRequest,  
          serviceException: ServiceException,  
      ): ErrorRes {  
          return ErrorRes(  
              code = serviceException.code,  
              timestamp = Date(),  
              path = httpServletRequest.requestURI,  
              message = serviceException.message ?: "internal server error",  
              fieldErrors = listOf()  
          )  
      }  
            
            
      @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)  
      @ExceptionHandler(Exception::class)  
      @ResponseBody  
      fun handleException(  
          httpServletRequest: HttpServletRequest,  
          exception: Exception,  
      ): ErrorRes {  
          exception.printStackTrace()  
            
          return ErrorRes(  
              code = "500",  
              timestamp = Date(),  
              path = httpServletRequest.requestURI,  
              message = "internal server error",  
              fieldErrors = listOf()  
          )  
      }  
            
  }  
  ```  
- dto  
  ```kotlin  
  @JsonInclude(JsonInclude.Include.NON_NULL)  
  data class UpdateReqDto<T>(  
      val id: Long,  
      val data: T  
  )  
            
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
- Controller  
  ```kotlin  
  @RestController  
  @RequestMapping("/v1/fruits")  
  class FruitController(  
      private val fruitService: FruitService,  
      private val fruitQuerydslRepository: FruitQuerydslRepository  
  ) {  
            
      @PutMapping(value = ["/{$FRUIT_ID:[0-9]+}"], produces = [MediaType.APPLICATION_JSON_VALUE])  
      fun updateFruit(  
          @PathVariable(name = FRUIT_ID) fruitId: Long,  
          @RequestBody fruitReqDto: FruitReqDto  
      ): DataRes<String> {  
          fruitService.updateFruit(  
              UpdateReqDto(  
                  id = fruitId,  
                  data = fruitReqDto  
              )  
          )  
            
          return DataRes(  
              "success"  
          )  
      }  
  }  
  ```  
- Service  
  ```kotlin  
  @Service  
  class FruitService(  
      private val fruitRepository: FruitRepository,  
      private val duplicateCheckService: DuplicateCheckService,  
  ) {  
            
      @Transactional(rollbackFor = [Exception::class])  
      fun updateFruit(fruitUpdateReqDto: UpdateReqDto<FruitReqDto>): Fruit {  
          val savedFruit = fruitRepository.findByIdOrNull(id = fruitUpdateReqDto.id)  
              ?: throw NotFoundException("fruit id ${fruitUpdateReqDto.id} is not found")  
            
          duplicateCheckService.checkForDuplicate(  
              entityClass = Fruit::class,  
              fieldName = Fruit::name.name,  
              fieldValue = fruitUpdateReqDto.data.name,  
              idValue = fruitUpdateReqDto.id,  
              useSoftDelete = true  
          )  
            
          savedFruit.origin = fruitUpdateReqDto.data.origin  
          savedFruit.name = fruitUpdateReqDto.data.name  
            
          return savedFruit  
      }  
  }  
  ```  
- 중복 값 입력 - 요청  
    - url  
        - http://localhost:8080/v1/fruits/2  
    - method  
        - PUT  
    - body  
      ```json  
      {  
          // 바나나는 중복된 값  
          "fruit_name": "바나나",  
          "fruit_origin": "필리핀",  
          "size": 10  
      }  
      ```  
- 중복 값 입력 - 응답  
  ```json  
  {  
      "code": "400",  
      "timestamp": "2025-03-18T04:05:12.125+00:00",  
      "path": "/v1/fruits/2",  
      "message": "there are duplicate fields",  
      "fieldErrors": [  
          {  
              "field": "name",  
              "message": "The value '바나나2' for field 'name' is already in use."  
          }  
      ]  
  }  
  ```  
- 중복 값 대상 필드가 변경되지 않은 경우 - 요청  
    - url  
        - http://localhost:8080/v1/fruits/2  
    - method  
        - PUT  
    - body  
      ```json  
      {  
          // id = 2 행의 fruit_name은 바나나2, 즉 변경 없음  
          "fruit_name": "바나나2",  
          "fruit_origin": "필리핀",  
          "size": 10  
      }  
      ```  
- 중복 값 대상 필드가 변경되지 않은 경우 - 응답  
  ```json  
  {  
   "data": "success"  
  }  
  ```  

## 참고
- entity/Fruit  
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
