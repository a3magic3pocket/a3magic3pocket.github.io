---
title: DB를 활용한 리더선출락 구현
date: 2025-04-11 22:35:03 +0900
categories: [Concurrency]
tags: [lock, spring, kotlin]    # TAG names should always be lowercase
---

## 개요
- 여러 컨테이너가 동시에 실행되는 환경에서는  
  특정 작업(이하 리더 작업)을 오직 하나의 컨테이너만   
  수행해야 할 때가 있다.  
- 리더 작업은 리더가 주기적으로 실행하고  
  리더가 아닌 다른 컨테이너는 대기해야한다.  
- 만약 리더가 다운되어 리더 작업이 불가능한 경우  
  최대한 빨리 대기하고 있는 다른 컨테이너 중 하나를   
  리더로 선출한 뒤 리더 작업을 이행해야한다.  
- DB를 이용하여 리더선출락을 구현하고  
  리더선출 과정을 설명한다.  

## 리더선출과정
- 3개의 컨테이너가 리더 작업에 참여한다.  
- 이 중 하나의 컨테이너를 리더로 선택하여 리더작업을 진행하고  
  나머지는 대기한다.  
- 리더 작업은 1초에 한 번씩 진행되며  
  문제가 없다면 1초 내에 끝난다.  
- 최초 3개의 컨테이너가 부팅되면  
  동시에 리더선출락 확보를 시도한다.  
- 3개 중 리더선출락을 확보한 컨테이너가 리더 컨테이너가 된다.  
- 리더선출락의 유효 기간(Time To Live, TTL)은   
  락의 timestamp 기준으로 1.2초이다.  
- 3개의 컨테이너 모두 1초에 한 번씩 리더선출과정에 참여한다.  
- 리더선출과정  
    - 리더라면?  
        - 락의 유효기간 연장한다.  
          (락의 timestamp를 현재시간으로 변경 후 갱신)  
    - 리더가 아니라면?  
        - 락이 유효하다면?(현재 시각 > 락의 timestamp + TTL)  
            - 다음 리더선출과정까지 대기한다.  
        - 락이 유효하지 않다면?  
            - 내가 리더가 된다.  
              (락 행의 container_id를 내 container_id로 갱신)  

## 리더선출락 DDL
```sql  
CREATE TABLE IF NOT EXISTS `service_leader_lock` (  
`id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,  
`name` VARCHAR(100) NOT NULL UNIQUE,   
`container_id` VARCHAR(100) DEFAULT NULL,  
`timestamp` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,  
INDEX `idx_service_leader_lock_name` (`name`)  
);  
```  

## ServiceLeaderLock
```kotlin  
package com.example.demo.entity  
        
import jakarta.persistence.Column  
import jakarta.persistence.Entity  
import jakarta.persistence.GeneratedValue  
import jakarta.persistence.GenerationType  
import jakarta.persistence.Id  
import jakarta.persistence.Table  
import java.time.ZoneOffset  
import java.time.ZonedDateTime  
        
@Entity  
@Table(name = "service_leader_lock")  
data class ServiceLeaderLock(  
        
  @Id  
  @GeneratedValue(strategy = GenerationType.IDENTITY)  
  val id: Long? = null,  
        
  @Column(name = "name", nullable = false, length = 100, unique= true)  
  val name: String,  
        
  @Column(name = "container_id", nullable = true, length = 100)  
  var containerId: String? = null,  
        
  @Column(name = "timestamp", nullable = false)  
  var timestamp: ZonedDateTime = ZonedDateTime.now(ZoneOffset.UTC)  
        
)  
```  

## ServiceLeaderLockService
```kotlin  
package com.exmaple.demo.service  
        
import com.exmaple.demo.entity.ServiceLeaderLock  
import com.exmaple.demo.constant.ServiceLeaderLock as ServiceLeaderLockConst  
import com.exmaple.demo.repository.ServiceLeaderLockRepository  
import com.exmaple.demo.global.config.AppConfig  
import org.springframework.stereotype.Service  
import org.springframework.transaction.annotation.Transactional  
import java.time.Duration  
import java.time.ZoneOffset  
import java.time.ZonedDateTime  
        
@Service  
class ServiceLeaderLockService(  
  private val appConfig: AppConfig,  
  private val serviceLeaderLockRepository: ServiceLeaderLockRepository,  
) {  
  companion object {  
      // time-to-live for service leader lock  
      val ttl: Duration = Duration.ofSeconds(1).plusMillis(2)  
  }  
        
  @Transactional(rollbackFor = [Exception::class])  
  fun tryToAcquireLock(): Boolean {  
      val lock = serviceLeaderLockRepository.findByName(  
          name = ServiceLeaderLockConst.NAME,  
      )  
        
      // 최초 락 점유  
      if (lock == null) {  
          val serviceLeaderLock = ServiceLeaderLock(  
              name = ServiceLeaderLockConst.NAME,  
              containerId = appConfig.containerId,  
              timestamp = ZonedDateTime.now(ZoneOffset.UTC),  
          )  
        
          serviceLeaderLockRepository.save(serviceLeaderLock)  
        
          return true  
      }  
        
      // 현재 컨테이너가 락을 점유하고 있는 상황  
      // - timestamp 갱신  
      if (lock.containerId == appConfig.containerId) {  
          lock.timestamp = ZonedDateTime.now(ZoneOffset.UTC)  
        
          serviceLeaderLockRepository.save(lock)  
        
          return true  
      }  
        
      // TTL 만료 시  
      val utcNow = ZonedDateTime.now(ZoneOffset.UTC)  
      val expirationTime = lock.timestamp.plus(ttl)  
      if (utcNow.isAfter(expirationTime)) {  
          lock.containerId = appConfig.containerId  
          lock.timestamp = ZonedDateTime.now(ZoneOffset.UTC)  
        
          serviceLeaderLockRepository.save(lock)  
        
          return true  
      }  
        
      return false  
  }  
}  
```  

## AppConfig
```kotlin  
package com.example.demo.global.config  
        
import jakarta.annotation.PostConstruct  
import org.springframework.context.annotation.Configuration  
import java.io.BufferedReader  
import java.io.InputStreamReader  
import java.util.UUID  
        
@Configuration  
class AppConfig {  
  var containerId = UUID.randomUUID().toString()  
        
  @PostConstruct  
  private fun initContainerId() {  
      try {  
          // 'hostname' 명령어 실행을 위한 ProcessBuilder 사용  
          val processBuilder = ProcessBuilder("hostname")  
          val process = processBuilder.start()  
          val reader = BufferedReader(InputStreamReader(process.inputStream))  
          val containerIdString = reader.readLine()  // hostname 명령어의 결과  
        
          // hostname 값이 비어 있지 않으면 containerId 업데이트  
          if (!containerIdString.isNullOrBlank()) {  
              containerId = containerIdString  
          }  
      } catch (e: Exception) {  
          // 예외 발생 시 아무 일도 일어나지 않음  
          println("Failed to retrieve container ID. Using default value.")  
          e.printStackTrace()  
      }  
  }  
        
}  
```  

## ServiceLeaderLockRepository
```kotlin  
package com.example.demo.repository  
        
import com.example.demo.entity.ServiceLeaderLock  
import jakarta.persistence.LockModeType  
import org.springframework.data.jpa.repository.JpaRepository  
import org.springframework.data.jpa.repository.Lock  
        
interface ServiceLeaderLockRepository : JpaRepository<ServiceLeaderLock, Int> {  
        
  @Lock(LockModeType.PESSIMISTIC_WRITE)  
  fun findByName(name: String): ServiceLeaderLock?  
        
}  
```  

## SomeService(리더선출과정 진행 및 리더일 경우, 리더 작업 실행)
```kotlin  
package com.example.demo.service  
        
import org.springframework.scheduling.annotation.Scheduled  
import org.springframework.stereotype.Service  
        
@Service  
class SomeService(  
  private val serviceLeaderLockService: ServiceLeaderLockService,  
) {  
  @Scheduled(fixedRate = 1000)  
  fun runLeaderTask() {  
      val isAcquired = serviceLeaderLockService.tryToAcquireLock()  
      if (isAcquired) {  
          println("run some tasks")  
      }  
  }  
}  
        
```  

## 정리
- DB 기반의 리더 선출 락을 구현하면,   
  다중 컨테이너 환경에서도 안정적으로   
  리더 역할을 선출하고 유지할 수 있다.  
- 트래픽이 높지 않거나 Redis 등 외부 락 시스템을   
  도입하기 어려운 상황에서   
  간단하면서도 효과적인 방식으로 활용이 가능하다.  
