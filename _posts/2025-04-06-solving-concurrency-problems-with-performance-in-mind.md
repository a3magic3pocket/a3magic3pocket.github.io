---
title: 동시성 문제 해결 방안 성능 실험
date: 2025-04-06 08:48:05 +0900
categories: [Concurrency]
tags: [k6, lock, essay]    # TAG names should always be lowercase
---

## 동시성 문제란?
- 여러 프로세스나 스레드가 동시에 실행되는 상황에서 자원(데이터 등)을   
  제대로 공유하거나 수정하지 못해 발생하는 문제  

## 구체적인 상황
- id, numLikes 필드로 이뤄진 POST 테이블이 있다.  
- likePost 함수는 postId를 인자로 받고,   
  post을 조회한 뒤 기존 numLikes를 읽고 그 값에 +1을 한 값을 저장한다.  
- 이때 두 트랜잭션이 동시에 likePost 함수를 호출하면  
  둘 중 하나의 트랜잭션은 덮어쓰기 되어 갱신이 소실된다.  

## 해결 방법
- 비관적 락(pessimistic lock)  
    - DB에서 대상이 되는 POST.id 행을 배타적 잠금한다.  
    - 락을 확보한 트랜잭션이 작업을 완료할때까지  
      다른 트랜잭션들은 대기한다.  
    - 대기가 길어서 트랜잭션 타임아웃 시간을 초과하면   
      트랜잭션은 실패하고 롤백 처리된다.  
    - 배타적 잠금이기 때문에 조회도 대기하게 된다.  
- 낙천적 락(optimistic lock)  
    - version 필드를 두어서 이를 애플리케이션에서 처리하여  
      동시성 문제를 해결한다.  
    - likePost 함수의 경우 어떻게 동작하는지 설명한다.  
    - 조회 단계  
        - `SELECT id, numLikes, version FROM POST WHERE id = 1`  
          실행한 결과 `numLikes = 1`, `version = 1`을 받는다.  
    - 수정 작업  
        - numLikes를  2로 업데이트 한다.  
          version도 2로 업데이트 한다.  
          이때 where 절에 `version = 1`을 추가한다.  
        - `UPDATE POST   
          SET numLikes = 2, version = 2  
          WHERE id = 1 AND version = 1`  
    - Optimistic Locking 동작 (버전 비교)  
        - 정상적인 경우  
            - `version = 1`로 읽어온 객체가 수정되어 저장될때,  
              `version = 1`이 `UPDATE` 쿼리의 조건에 포함되므로,  
              `version = 1`인 데이터가 존재하면 `UPDATE`가   
              정상적으로 실행된다.  
        - 경쟁 상태  
            - 만약 첫 번째 스레드가 `version = 1`로 numLikes를 읽은 후,  
              두 번째 스레드에서 numLikes를 수정하여 `version = 2`로 갱신했다면,  
              첫 번째 스레드는 더 이상 `version = 1`을 가진 데이터를   
              수정할 수 없다.   
            - 이 경우, `UPDATE` 쿼리가 영향을 미치지 않게 되며  
              애플리케이션(예를 들면 스프링)에서   
              예외(OptimisticLockingFailureException)를 발생시킨다.  
- 단일 스레드, 비동기 배치 처리  
    - 동시성 문제는 여러 프로세스나 스레드에서 동시에   
      특정 자원을 점유할때 발생한다.  
    - 단일스레드만 사용하여 자원을 점유하게 하면  
      동시성 문제를 막을 수 있다.  
    - 배치 작업을 위하여 요청과 처리를 분리한다.  
    - 요청  
        - 레디스와 같은 단일스레드 메세지 큐를 두고  
          요청은 레디스 메세지 큐에 쌓는다.  
    - 처리  
        - 메세지 큐에서 꺼내어 작업뭉치를 꺼낸뒤   
          비동기로 배치 작업을 처리하는 프로그램을 단일 스레드로 운영한다.  
        - 예를들면 큐에서 3개의 작업을 꺼낸다.  
            - likePost(postId = 1)  
            - likePost(postId = 2)  
            - likePost(postId = 3)  
        - autoCommit = false로 전환한뒤  
          3개의 작업을 진행한다.  
        - 문제가 없다면 commit하고 문제가 있으면 롤백한다.  
          이때 롤백이 되면 3개 작업 모두 롤백된다.  

## 실험 환경
- 웹 프레임워크: 스프링부트, 코틀린  
- DB: 마리아디비  
- 메세지 큐: 레디스 리스트  
- 테스트 툴: k6  

## 테스트 환경
- k6 설정  
    - 가상사용자 수(vus): 100개  
    - 총 요청 수 (iterations): 3000개 또는 6000개  
    - 시간 (duration): 5분  
- k6 설정 설명  
    - 100 명의 유저가 3000개 요청을 5분 동안 동시에 진행한다.  
    - 5분이 되기 전에 총 요청 수가 3000에 도달하면 중지한다.  
- pessimistic lock 설정  
    - timeout: 5초  
    - 격리수준(isolation level): REPEATABLE_READ  
- optimistic lock 설정  
    - timeout: 20초  
    - 격리수준: REPEATABLE_READ  
    - OptimisticLockingFailureException 발생 시 최대 허용 재시도 수: 30  
    - OptimisticLockingFailureException 발생 시 대기시간: 0.1초  
- 단일 스레드, 비동기 배치 처리 설정  
    - DB timeout: 5초  
    - DB 격리수준: REPEATABLE_READ  
    - 비동기 배치 함수 배치 크기: 20개  
    - 비동기 배치 함수가 담긴 스프링 스케쥴러 fixedDelay: 0.1초  
        - fixedDelay는 이전 작업이 완료된 후 재시도 하기 전까지 대기 시간  

## 테스트 시나리오
- 1번 시나리오  
    - 단순 문자열만 리턴하는 컨트롤러 조회  
    - 실험의 대조군으로 사용하기 위해 조회  
    - naive 요청 3000번 조회   
- 2번 시나리오  
    - 한 요청에서 'likePost(postId = someId)'만 실행  
      즉, 한 트랜잭션에서 조회 및 갱신 요청만 실행  
    - 요청 3000번 실행  
- 3번 시나리오  
    - 한 요청에서 '모든 post 목록 조회'와    
      'likePost(postId = someId)'를 묶어서 실행  
      즉, 한 트랜잭션에서 조회 한 번, 한 트랜잭션에서 조회 및 갱신 요청 한 번  
    - 모든 post 목록 조회 3000번,   
      likePost(postId = someId) 3000번 실행  
- 4번 시나리오  
    - 한 요청에서 '모든 post 목록 조회' 또는 'likePost(postId = someId)'를 실행  
      이때 '모든 post 목록 조회'를 전체 요청 수의 90%를 진행하고 'likePost(postId = someId)'는 전체 요청 수의 10%만 진행  
    - 모든 post 목록 조회 5400번,  
      likePost(postId = someId) 600번 실행  

## 결과
- 1번 시나리오 - naive 요청  
    - 총 요청 수: 3000  
    - 총 실패 수: 0  
    - 테스트 완료 시간(ms): 3363.41  
- 2번 시나리오 - likePost 만 진행  
    - pessimistic lock  
        - 총 요청 수: 3000  
        - 총 실패 수: 0  
        - 테스트 완료 시간(ms): 20111.89  
    - optimistic lock  
        - 총 요청 수: 3000  
        - 총 실패 수: 0  
        - 테스트 완료 시간(ms): 31873.77  
    - 단일 스레드, 비동기 배치 처리  
        - 총 요청 수: 3000  
        - 총 실패 수: 0  
        - 테스트 완료 시간(ms): 17954.14  
- 3번 시나리오 - 모든 post 조회와 likePost  1대1 처리  
    - pessimistic lock  
        - 총 요청 수: 6000  
        - 총 실패 수: 0  
        - 테스트 완료 시간(ms): 27453.60  
    - optimistic lock  
        - 총 요청 수: 6000  
        - 총 실패 수: 0  
        - 테스트 완료 시간(ms): 37709.28  
    - 단일 스레드, 비동기 배치 처리  
        - 총 요청 수: 6000  
        - 총 실패 수: 0  
        - 테스트 완료 시간(ms): 23220.97  
- 4번 시나리오 - 모든 post 조회와 likePost  1대9 처리  
    - pessimistic lock  
        - 총 요청 수: 6000  
        - 총 실패 수: 0  
        - 테스트 완료 시간(ms): 16909.48  
    - optimistic lock  
        - 총 요청 수: 6000  
        - 총 실패 수: 0  
        - 테스트 완료 시간(ms): 24357.44  
    - 단일 스레드, 비동기 배치 처리  
        - 총 요청 수: 6000  
        - 총 실패 수: 0  
        - 테스트 완료 시간(ms): 20432.77  

## 의의
- 모든 경우에서 optimistic lock 보다 pessimistic lock의 처리속도가 더 빨랐다.  
- optimistic lock의 경우, 실패 후 재시도 시간을 너무 길게 잡으면  
  처리 속도가 엄청나게 느려지며, 재시도 시간이 너무 긴 경우   
  DB 타임아웃이 발생하여 트랜잭션이 실패한다.  
- pessimistic lock과 optimistic lock 모두 시나리오 3번보다 4번이 테스트 종료 시간이 더 짧았다.  
- optimistic lock의 (시나리오 3번 테스트 종료 시간 - 시나리오 4번 테스트 종료 시간)이 pessimistic lock보다 더 크다.  
- `단일 스레드, 비동기 배치 처리`는 모든 시나리오에서 상대적으로 일정한 처리 속도를 보인다.  
- `단일 스레드, 비동기 배치 처리`는 처리가 비동기로 이뤄지기 때문에  
  테스트가 종료된 후 일정 시간이 지나야 처리가 완료된다.  
- `단일 스레드, 비동기 배치 처리`는 스케쥴러 대기 시간을 길게 설정하거나 배치 크기를 줄일 경우, 처리가 완료되는 시간이 엄청나게 늘어난다.  

## 결론
- 여러 스레드에서 동시에 WRITE가 발생할 상황이 조금이라도 생긴다면  
  pessimistic lock을 쓰는게 더 유리한 것으로 보인다.  
- optimistic lock의 경우 timeout 시간과 재시도 대기 시간을 잘 설정해야만  
  성능이 좋아진다.  
- 예를 들어 현재 테스트 시나리오에서 optimistic lock의 timeout을   
  5로 했을때보다 20으로 했을때가 처리시간이 더 짧다.  
  아마도 timeout 시간이 짧으면 재시도 대기를 하는 요청도  
  늘어나기 때문에 전반적인 처리 시간이 늘어나는 것으로 추정한다.  
-  3번 시나리오에 비해 4번 시나리오의 처리 속도는  
   optimistic lock이 pessimistic lock 보다 더 많이 향상되었다.  
   이를 통해 충돌이 거의 없는 환경에서는 optimistic lock이   
   pessimistic lock보다 더 처리속도가 빠를 것으로 추정한다.  
- `단일 스레드, 비동기 배치 처리` 는 다른 방법에 비해   
  상대적으로 균일한 처리 속도를 보여준다.  
- 하지만 비동기이기 때문에 실시간으로 조회가 이뤄져야 하는 경우  
  사용하기 어렵다.  
- 비동기 스케줄러 시간을 줄이면 전반적인 처리 속도는 상승한다.  

## 예시 github repository
- [concurrency-control-sandbox' github repository](https://github.com/a3magic3pocket/concurrency-control-sandbox){:target="_blank"}  
