---
title: 코틀린 스프링 querydsl 스니펫
date: 2025-03-30 23:13:25 +0900
categories: [Kotlin]
tags: [kotlin, querydsl, summary]    # TAG names should always be lowercase
---

## 최초 세팅
- build.gradle.kts  
  ```kotlin  
  plugins {  
      val kotlinVersion = "2.1.10"  
            
      kotlin("jvm") version kotlinVersion  
      kotlin("plugin.spring") version kotlinVersion  
      kotlin("plugin.jpa") version kotlinVersion  
            
      id("org.springframework.boot") version "3.4.3"  
      id("io.spring.dependency-management") version "1.1.7"  
            
      // KSP 플러그인 추가  
      kotlin("kapt") version kotlinVersion  
      id("com.google.devtools.ksp") version "$kotlinVersion-1.0.31"  
  }  
            
  ...  
            
  dependencies {  
      // querydsl  
      val querydslVersion = "6.10.1"  
      implementation("io.github.openfeign.querydsl:querydsl-jpa:$querydslVersion")  
      ksp("io.github.openfeign.querydsl:querydsl-ksp-codegen:$querydslVersion")  
      compileOnly("io.github.openfeign.querydsl:querydsl-apt:$querydslVersion:jpa")  
  }  
            
  ...  
            
  ```  
- QuerydslConfig  
  ```kotlin  
  @Configuration  
  class QuerydslConfig(  
      private val entityManager: EntityManager  
  ) {  
            
      @Bean  
      fun jpaQueryFactory(entityManager: EntityManager): JPAQueryFactory {  
          return JPAQueryFactory(entityManager)  
      }  
            
  }  
  ```  
- QuerydslRepository   
  ```kotlin  
  @Repository  
  class FruitQuerydslRepository(  
  private val jpaQueryFactory: JPAQueryFactory  
  ) {  
      // querydsl 레포지토리 메서드 추가  
  }  
  ```  

## 조인(Join)
(과일 주문 별 과일 정보(이름, 원산지 등) 획득)  
- dto  
  ```kotlin  
  data class FruitOrderDto @QueryProjection constructor(  
      val fruit: Fruit,  
      val fruitOrder: FruitOrder,  
  )  
  ```  
- Querydsl 레포지토리  
  ```kotlin  
  @Repository  
  class FruitQuerydslRepository(  
      private val jpaQueryFactory: JPAQueryFactory  
  ) {  
      fun getFruitsWithQuantity(): MutableList<FruitOrderDto> {  
          val fruit = QFruit.fruit  
          val fruitOrder = QFruitOrder.fruitOrder  
            
          return jpaQueryFactory.select(  
              QFruitOrderDto(  
                  fruit,  
                  fruitOrder  
              )  
          ).from(fruit)  
              .innerJoin(fruitOrder).on(fruit.id.eq(fruitOrder.fruitId))  
              .where(  
                  fruit.isDeleted.eq(false),  
                  fruitOrder.isDeleted.eq(false)  
              ).fetch()  
      }  
  }  
  ```  
- SQL 쿼리  
  ```sql  
  SELECT `f`.*, `fo`.*  
  FROM `FRUIT` AS `f`  
  INNER JOIN `FRUIT_ORDER` AS `fo` ON `f`.`id` = `fo`.`fruit_id`  
  WHERE `f`.`is_deleted` = 'N' AND `fo`.`is_deleted` = 'N'  
  ;  
  ```  

## 페이지네이션(Pagination)
(과일 주문 별 과일 정보(이름, 원산지 등) 획득)  
- Querydsl 레포지토리  
  ```kotlin  
  @Repository  
  class FruitQuerydslRepository(  
      private val jpaQueryFactory: JPAQueryFactory  
  ) {  
            
      fun getFruitsWithQuantityPageable(pageable: Pageable): Page<FruitOrderDto> {  
          val fruit = QFruit.fruit  
          val fruitOrder = QFruitOrder.fruitOrder  
            
          val builder = BooleanBuilder()  
          builder.and(fruit.isDeleted.eq(false))  
          builder.and(fruitOrder.isDeleted.eq(false))  
            
          val data = jpaQueryFactory.select(  
              QFruitOrderDto(  
                  fruit,  
                  fruitOrder  
              )  
          ).from(fruit)  
              .innerJoin(fruitOrder).on(fruit.id.eq(fruitOrder.fruitId))  
              .where(builder)  
              .offset((pageable.pageNumber * pageable.pageSize).toLong())  
              .limit(pageable.pageSize.toLong())  
              .fetch()  
            
          val count = jpaQueryFactory.select(  
              fruit.count()  
          ).from(fruit)  
              .innerJoin(fruitOrder).on(fruit.id.eq(fruitOrder.fruitId))  
              .where(builder)  
              .fetchOne()  
            
          return PageableExecutionUtils.getPage(data, pageable) { count ?: 0L }  
      }  
            
  }  
  ```  
- SQL 쿼리  
  ```sql  
  -- 첫 페이지에서 10개 행 조회  
  SELECT `f`.*, `fo`.*  
  FROM `FRUIT` AS `f`  
  INNER JOIN `FRUIT_ORDER` AS `fo` ON `f`.`id` = `fo`.`fruit_id`  
  WHERE `f`.`is_deleted` = 'N' AND `fo`.`is_deleted` = 'N'  
  OFFSET 0   
  FETCH FIRST 10 ROWS ONLY  
  ;  
            
  --   
            
  SELECT count(`f`.`id`)  
  FROM `FRUIT` AS `f`  
  INNER JOIN `FRUIT_ORDER` AS `fo` ON `f`.`id` = `fo`.`fruit_id`  
  WHERE `f`.`is_deleted` = 'N' AND `fo`.`is_deleted` = 'N'  
  ;  
  ```  

## 테이블 별명(table alias)
(기준 과일 주문과 이전 과일 주문의 수량 차이 계산)  
- dto  
  ```kotlin  
  data class FruitOrderLatestAndPreviousDiff @QueryProjection constructor(  
      val fruitId: Long,  
      val baseOrderId: Long,  
      val previousOrderId: Long,  
      val baseOrderQuantity: Int,  
      val previousOrderQuantity: Int,  
      val diff: Int,  
  )  
  ```  
- Querydsl 레포지토리  
  ```kotlin  
  @Repository  
  class FruitQuerydslRepository(  
      private val jpaQueryFactory: JPAQueryFactory  
  ) {  
            
      fun getLatestAndPreviousOrderDiff(): MutableList<FruitOrderLatestAndPreviousDiff>? {  
          val fruitOrder1 = QFruitOrder("fo1")  
          val fruitOrder2 = QFruitOrder("fo2")  
            
          return jpaQueryFactory.select(  
              QFruitOrderLatestAndPreviousDiff(  
                  fruitOrder1.fruitId,  
                  fruitOrder1.id,  
                  fruitOrder2.id,  
                  fruitOrder1.quantity,  
                  fruitOrder2.quantity,  
                  fruitOrder1.quantity.subtract(fruitOrder2.quantity)// 수량 차이 계산  
              )  
          )  
              .from(fruitOrder1)  
              .innerJoin(fruitOrder2)  
              .on(  
                  fruitOrder1.fruitId.eq(fruitOrder2.fruitId)  
                      .and(fruitOrder1.updatedAt.gt(fruitOrder2.updatedAt))  
              )  
              .where(  
                  fruitOrder1.isDeleted.eq(false),  
                  fruitOrder2.isDeleted.eq(false),  
                  fruitOrder1.id.ne(fruitOrder2.id)  
              )  
              .orderBy(  
                  fruitOrder1.fruitId.asc(), fruitOrder1.updatedAt.desc()  
              )  
              .fetch()  
      }  
            
  }  
  ```  
- SQL 쿼리  
  ```sql  
  SELECT `fo1`.`fruit_id`, `fo1`.`id`, `fo2`.`id`, `fo1`.`quantity`, `fo2`.`quantity`, (`fo1`.`quantity` - `fo2`.`quantity`) as `diff`  
  FROM `FRUIT_ORDER` AS `fo1`  
  INNER JOIN `FRUIT_ORDER` AS `fo2` ON `fo1`.`fruit_id` = `fo2`.`fruit_id`  
  WHERE   
  	 `fo1`.`is_deleted` = 'N' AND   
  	 `fo2`.`is_deleted` = 'N' AND   
  	 `fo1`.`id` != `fo2`.`id`  
  ORDER BY `fo1`.`fruit_id` ASC, `fo1`.`updated_at` DESC  
  ;  
  ```  

## 서브쿼리
(특정 주문 수(quantity)를 갖는 과일 목록 획득)  
- Querydsl 레포지토리  
  ```kotlin  
  @Repository  
  class FruitQuerydslRepository(  
      private val jpaQueryFactory: JPAQueryFactory  
  ) {  
      fun getFruitsOrderedByQuantity(quantity: Int): MutableList<Fruit> {  
          val fruit = QFruit.fruit  
          val fruitOrder = QFruitOrder.fruitOrder  
            
          return jpaQueryFactory.select(  
              fruit  
          )  
              .from(fruit)  
              .where(  
                  fruit.id.`in`(  
                      JPAExpressions.selectDistinct(  
                          fruitOrder.fruitId  
                      ).from(fruitOrder)  
                          .where(fruitOrder.quantity.eq(quantity))  
                  )  
              )  
              .fetch()  
      }  
            
  }  
  ```  
- SQL 쿼리  
  ```sql  
  -- quantity 2인 과일 목록 획득  
  SELECT * FROM `FRUIT`  
  WHERE `id` IN (  
  	SELECT DISTINCT `fruit_id` FROM `FRUIT_ORDER`   
  	WHERE `quantity` = 2 AND `is_deleted` =  'N'  
  ) AND `is_deleted` = 'N'  
  ;  
  ```  

## CTE(Common Table Expression)
(과일 별 수량 합 및 주문 수 획득)  
- 설명  
    - querydsl CTE 지원 안 함 따라서 nativeQuery로 생성  
- dto  
  ```kotlin  
  data class FruitOrderSummaryDto(  
      val id: Long,  
      val name: String,  
      val sumQuantities: Long,  
      val countOrders: Long,  
  )  
  ```  
- EntityExtension  
  ```kotlin  
  // 캐시를 위한 맵을 추가하여 성능 최적화  
  val tableNameCache = mutableMapOf<KClass<*>, String?>()  
  val columnNameCache = mutableMapOf<Pair<KClass<*>, String>, String?>()  
            
  // 엔티티 테이블의 실제 테이블명  
  fun getTableName(clazz: KClass<*>): String? {  
      // 캐시에서 먼저 확인  
      return tableNameCache.getOrPut(clazz) {  
          clazz.java.getAnnotation(Table::class.java)?.name  
      }  
  }  
            
  // 엔티티 컬럼의 실제 컬럼명  
  fun getColumnName(clazz: KClass<*>, entityPropertyName: String): String? {  
      // 캐시에서 먼저 확인  
      return columnNameCache.getOrPut(clazz to entityPropertyName) {  
          clazz.java.declaredFields.find { it.name == entityPropertyName }  
              ?.getAnnotation(Column::class.java)?.name  
      }  
  }  
  ```  
- ArrayExtension  
  ```kotlin  
  inline fun <reified T> Array<*>.safeGet(index: Int): T? {  
      val value = this.getOrNull(index)  
      return if (value is T) value else null  
  }  
  ```  
- Querydsl 레포지토리  
  ```kotlin  
  @Repository  
  class FruitQuerydslRepository(  
      private val jpaQueryFactory: JPAQueryFactory,  
      private val entityManager: EntityManager,  
  ) {  
               
      fun getFruitsWithOrderSummary(): List<FruitOrderSummaryDto> {  
          val cteT = "cte"  
          val fruitOrderT = getTableName(FruitOrder::class)  
          val foFruitIdC = getColumnName(FruitOrder::class, FruitOrder::fruitId.name)  
          val foQuantityC = getColumnName(FruitOrder::class, FruitOrder::quantity.name)  
          val foIdC = getColumnName(FruitOrder::class, FruitOrder::id.name)  
          val foIsDeletedC = getColumnName(FruitOrder::class, FruitOrder::isDeleted.name)  
          val cteQuantityAlias = "sum_quantities"  
          val cteCountAlias = "count_orders"  
          val fruitTAlias = "f"  
          val fruitT = getTableName(Fruit::class)  
          val fIdC = getColumnName(Fruit::class, Fruit::id.name)  
          val fNameC = getColumnName(Fruit::class, Fruit::name.name)  
          val fIsDeletedC = getColumnName(Fruit::class, Fruit::isDeleted.name)  
          val foIsDeletedParam = "fo_is_deleted"  
          val fIsDeletedParam = "f_is_deleted"  
            
          val sql = """  
                  WITH `$cteT` AS (  
                    SELECT `$foFruitIdC`, sum(`$foQuantityC`) AS `$cteQuantityAlias`, count(`$foIdC`) AS `$cteCountAlias`  
                    FROM `$fruitOrderT`  
                    WHERE `$foIsDeletedC` = :$foIsDeletedParam  
                    GROUP BY `$foFruitIdC`  
                  )  
                  SELECT `$fruitTAlias`.`$fIdC`, `$fruitTAlias`.`$fNameC`, `$cteT`.`$cteQuantityAlias`, `$cteT`.`$cteCountAlias`    
                  FROM `$fruitT` AS `$fruitTAlias`  
                  INNER JOIN `$cteT` ON `$fruitTAlias`.`$fIdC` = `$cteT`.`$foFruitIdC`  
                  WHERE `$fruitTAlias`.`$fIsDeletedC` = :$fIsDeletedParam  
                  ;  
          """.trimIndent()  
            
          // 쿼리 생성  
          val query = entityManager.createNativeQuery(sql)  
            
          // 파라미터 할당  
          query.setParameter(foIsDeletedParam, 'N')  
          query.setParameter(fIsDeletedParam, 'N')  
            
          return query.resultList.mapNotNull { row ->  
              FruitOrderSummaryDto(  
                  id = (row as? Array<*>)?.safeGet<Long>(0) ?: return@mapNotNull null,  
                  name = row.safeGet<String>(1) ?: return@mapNotNull null,  
                  sumQuantities = row.safeGet<Long>(2) ?: return@mapNotNull null,  
                  countOrders = row.safeGet<Long>(3) ?: return@mapNotNull null,  
              )  
          }  
      }  
            
  }  
  ```  

## 서브쿼리 결과를 대상으로 조인
- 설명  
    - querydsl은 서브쿼리 결과를 대상으로 조인하는 기능을 지원 안 함  
- dto  
  ```kotlin  
  data class FruitOrderSubqueryJoinDto(  
      val id: Long,  
      val fruitId: Long,  
  )  
  ```  
- Querydsl 레포지토리  
  ```kotlin  
  @Repository  
  class FruitQuerydslRepository(  
      private val jpaQueryFactory: JPAQueryFactory,  
      private val entityManager: EntityManager,  
  ) {  
            
      fun joinByFruitName(name: String): List<Any> {  
          val fruitOrderT = getTableName(FruitOrder::class)  
          val foFruitIdC = getColumnName(FruitOrder::class, FruitOrder::fruitId.name)  
          val foIdC = getColumnName(FruitOrder::class, FruitOrder::id.name)  
          val foIsDeletedC = getColumnName(FruitOrder::class, FruitOrder::isDeleted.name)  
          val fruitOrderTAlias = "fo"  
          val fruitTAlias = "refined_f"  
          val fruitT = getTableName(Fruit::class)  
          val fIdC = getColumnName(Fruit::class, Fruit::id.name)  
          val fNameC = getColumnName(Fruit::class, Fruit::name.name)  
          val fIsDeletedC = getColumnName(Fruit::class, Fruit::isDeleted.name)  
          val foIsDeletedParam = "fo_is_deleted"  
          val fIsDeletedParam = "f_is_deleted"  
          val fNameParam = "f_name"  
            
          val sql = """  
              SELECT `$fruitOrderTAlias`.`$foIdC`, `$fruitOrderTAlias`.`$foFruitIdC`   
              FROM `$fruitOrderT` AS `$fruitOrderTAlias`  
              INNER JOIN (  
                SELECT `$fIdC`  
                FROM `$fruitT`  
                WHERE `$fNameC` = :$fNameParam AND `$fIsDeletedC` = :$fIsDeletedParam  
              ) AS `$fruitTAlias`  
              ON `$fruitOrderTAlias`.`$foFruitIdC` = `$fruitTAlias`.`$fIdC`  
              WHERE `$fruitOrderTAlias`.`$foIsDeletedC` = :$foIsDeletedParam  
              ;  
          """.trimIndent()  
            
          // 쿼리 생성  
          val query = entityManager.createNativeQuery(sql)  
            
          // 파라미터 할당  
          query.setParameter(fNameParam, name)  
          query.setParameter(foIsDeletedParam, 'N')  
          query.setParameter(fIsDeletedParam, 'N')  
            
          return query.resultList.mapNotNull { row ->  
              FruitOrderSubqueryJoinDto(  
                  id = (row as? Array<*>)?.safeGet<Long>(0) ?: return@mapNotNull null,  
                  fruitId = row.safeGet<Long>(1) ?: return@mapNotNull null,  
              )  
          }  
      }  
  }  
  ```  
