<h1 align = 'center'> 매칭 서비스 기술 보고서 </h1>

### 📑 목차

- [0️⃣ 매칭 서비스 한 눈에 이해하기](#0️⃣-매칭-서비스-한-눈에-이해하기)
  - [⑴ 개요](#1-개요)
  - [⑵ USER FLOW](#2-user-flow)
  - [⑶ 서비스 아키텍쳐](#3-서비스-아키텍쳐)
  - [⑷ ERD](#4-erd)
  
- [1️⃣ '근처 일거리 찾기 API' 고도화 과정과 결과](#1️⃣-근처-일거리-찾기-api-고도화-과정과-결과)
  - [⑴ 최초 구현](#1-최초-구현)
  - [⑵ 1차 고도화: 쿼리 전략 개선](#2-1차-고도화-쿼리-전략-개선)
  - [⑶ 2차 고도화: 쿼리 튜닝](#3-2차-고도화-쿼리-튜닝)
  - [⑷ 3차 고도화: mysql 공간 객체 활용](#4-3차-고도화-mysql-공간-객체-활용)
  - [⑸ 4차 고도화: read/write - through 패턴 구현, l1/l2 캐싱](#5-4차-고도화-readwrite---through-패턴-구현-l1l2-캐싱)

- [2️⃣ '매칭 User flow 시나리오 테스트 15회와 그 개선 과정'](#2️⃣-매칭-user-flow-시나리오-테스트-15회와-그-개선-과정)
  - [(0) 테스트 목표 설정](#0-테스트-목표-설정)
  - [⑴ 테스트 001](#1-테스트-001)
  - [⑵ 테스트 002: was, os 튜닝 후](#2-테스트-002-was-os-튜닝-후)
  - [⑶ 테스트 003: 알림 전송 서비스에 retry 로직 추가 후](#3-테스트-003-알림-전송-서비스에-retry-로직-추가-후)
  - [⑷ 테스트 004: 지연 변이 로직 구현 후](#4-테스트-004-지연-변이-로직-구현-후)
  - [⑸ 테스트 005 ~ 010: 톰캣 thread 수와 db connection 의 연관 관계](#5-테스트-005--010-톰캣-thread-수와-db-connection-의-연관-관계)
  - [⑹ 테스트 016: sql 진입점 로깅, 에러 수집, 슬로우 쿼리 확인](#6-테스트-016-sql-진입점-로깅-에러-수집-슬로우-쿼리-확인)
  



# 0️⃣ 매칭 서비스 한 눈에 이해하기

## (1) 개요

- 하기 싫은 **`소일 거리`** 대신 해줄 사람을 매칭 해주는 서비스
- **`당근 알바`, `해주세요`** 같은 앱을 소일거리 매칭으로 특화



## (2) USER FLOW

### A. 일 등록 부터 해결까지

![image-20250412125244678](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412125244678.png)

- `파란색`: 사용자 모두 긍정적인 행동으로, 서비스를 이용 후 종료
- `빨간색`: 사용자 중 하나가 서비스 이용을 종료하는 행동을 취했을 때, 조치

### B.  `#1`번과 `#2`번 사이, 구직자가 일거리를 찾는 과정

![image-20250412133004318](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412133004318.png)



## (3) 서비스 아키텍쳐

![SPOT_JOB_MATCHING_ARCHITECTURE](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/SPOT_JOB_MATCHING_ARCHITECTURE.png)

- Redis 활용 Read/Write-Through Pattern 구현
- 모든 API 요청은 계약 상대방에게 **`FCM 알림`** 전송



## (4) ERD

![spot_job_matching_erd](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/spot_job_matching_erd.png)

### A. Matching 교차테이블에 대한 이해

![image-20250412154814804](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412154814804.png)

- 각 회원은 일거리에 대해서 자신만의 상태 (**`MatchingStatus`**)를 가진다. 가질 수 있는 상태는 위와 같다.



# 1️⃣ '근처 일거리 찾기 API' 고도화 과정과 결과

## (0) 결론

> **(v-user 3000명, 1000 RPS, ramp-up: 300, 10분 지속 기준)**

- **`평균 RPT`**: **2705ms ➜ 91ms 단축** 
- **`평균TPS`: 200 상승** 

## (1) 최초 구현

1. 프론트에서 위도, 경도 전송
2. Haver-sine 거리 공식 활용, table-full-scan으로 모든 일과의 거리 계산 
3. 페이지 네이션 적용 후 반환

```java
   @Query("""
    SELECT j FROM Job j
    WHERE j.startedAt IS NULL
      AND (6371 * acos(
               cos(radians(:lat)) * cos(radians(j.lat)) *
               cos(radians(j.lng) - radians(:lng)) +
               sin(radians(:lat)) * sin(radians(j.lat))
           )) < :dist
    ORDER BY (6371 * acos(
                cos(radians(:lat)) * cos(radians(j.lat)) *
                cos(radians(j.lng) - radians(:lng)) +
                sin(radians(:lat)) * sin(radians(j.lat))
            )) ASC
    """)
    Slice<Job> findNearByJobWithJPQL(
        @Param("lat") double lat,
        @Param("lng") double lng,
        @Param("dist") double dist,
        Pageable pageable
    );
```

<img src ='https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412162105389.png' align='center'/>

## (2) 1차 고도화: 쿼리 전략 개선

### 🛠️ 부하테스트 기준

- `대상 데이터`: 회원 = 5000명, 일거리 = 5000개, 
- `Thread 수` : 500, `ramp-up`: 300, `지속`: 6분 지속, `페이지 머무르는 시간`: 2초

### 🚨 문제 상황

- 첫 구현의 평균 RPT: 3102ms
- 부하 10분 이상 지속 시 응답 시간은 우상향을 유지

### ✅ 개선 사항

**🌟결과**:  **평균  RPT 3102ms ➜ 403ms 단축**

- 󠁯쿼리나 서버 로직을 최적화 하기 전에, 쿼리 전송 전략  최적화가 우선 되어야 한다 판단
-  JPQL, native-query, query-dsl로 구현 후 성능 테스트 재실시

<img src="https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328011632812.png" width = 100%/>
<img src = "https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328011553536.png" width = 100%/>

<img src = "https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328011835923.png" width=100%/>

- 테스트 결과
  - `JPQL`:  평균 RPT = 1040ms, 평균 TPS = 140, 지속 3분 후부터는 평균 RPT가 1.2초를 넘기며 우상향
  - `native-query`: 평균 RPT = 710ms, 평균 TPS = 210, 모든 응답이 1초 이내로 들어옴 
  - `query-dsl`: 평균 RPT = 400ms 이하, 평균 TPS = 240 



## (3) 2차 고도화: 쿼리 튜닝

### 🛠️ 일거리 데이터 3000개 ➜ 20만 개 추가 후 부하테스트 실행

- `대상 데이터`: 회원 = 200000명, 일거리 = 200000개, 
- `Thread 수` : 500, `ramp-up`: 300, `지속`: 6분 지속, `페이지 머무르는 시간`: 2초

### 🚨 문제 상황

- 기존 쿼리가 **‘**table full scan**’** 을 타는터라 20만 데이터 추가 후 Timeout으로 인한 최대 오류율 30% 평균 RPT 4000ms로 급증 
-  위,경도 복합 인덱스를 만들어도, 옵티마이저가 table full scan을 선택해서 쿼리 개선이 안됨. 
- `RDB에서 일 찾기 fullscan - O(N)` ➜ `찾은 일과 사용자 간의 거리계산 O(N)` ➜ `DTO MAPPING O(N)`이라는 **`O(3N)`**의 시간 복잡도를 극복하지 못함.

- 1차 고도화 했던 API는 RDB 데이터 양이 커지는 것과 RPT가 비례하는 모습을 보임.
- querydsl의 경우 오류율은 없었음.

<img src ='https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328021543165.png' width=100%/>

- 나머지 두 개 (1차 고도화 버전 native-query, jpql 활용 API) 또한 RPT가 데이터양과 비례에서 올라감.
- 3분이 지난 시점부터 에러율이 30% 수준으로 급증함.

<p aling = 'center' style="display: flex; break-inside: avoid"><img src='https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412164948464.png' width='50%'/><img src='https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412165104697.png' width = "50%"</p>



### 📊 트러블 슈팅

**🗺️ Geo Hash원리를 쿼리로 구현 - 보조 인덱스 사용 유도 위함**

- 위도, 경도를 복합 인덱스로 설정
- B-tree 인덱스를 활용하기 위해 위도, 경도와 Between문을 활용해 넓이 3KM 짜리 격자를 생성함. 
  (Between 문에서 인덱스 활용한 1차 필터링 유도)

<img src="https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328022358268.png" width="60%"/>  <img src="https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328021805917.png" alt="설명 이미지" width="300">

- 인덱싱을 활용해 O(M^2)이 아닌 O(logN + M) (N ➜ 데이터 접근, M ➜ 데이터 리드)를 바랬으나, 인덱싱을 활용하지 않고 `FULL-SCAN`을 탐.  

```sql
Limit: 100 row(s) (cost=10612 rows=100)(actual time=314..314 rows=100 loops=1)
   Sort: distance, limit input to 100 row(s) per chunk (cost=10612 rows=97706) (actual time=149..149 rows=0 loops=1) 
       Filter: (Haver-sine d 연산) (actual time=149..149 rows=0 loops=1) 
            Table scan on job (cost=10612 rows=97706) (actual time=0.0353..141 rows=100000 loops=1)
```

![image-20250412171455743](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412171455743.png)

🧪 **인덱스 강제 적용과 미적용 실행 계획 비교**

- 왜 복합 인덱스를 타지 않는지 궁금
- 옵티마이저 강제 힌트를 적용해서, 실행 계획 확인

<p align ='center' style="display: flex; break-inside: avoid"> 
<img src="https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328023929214.png" width = 49%/>
<img src = "https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328023331266.png" width = 49%/>
</p>

- 강제 인덱스 적용 결과, 미적용 시보다, COST는 4.5배 높지만, 실행 시간은 10배 단축됨을 확인
- 옵티마이저가 실행계획에서 랜덤 I/O 접근에 대한 비용을 과하게 잡고 있다고 판단 



**🧪 서브 쿼리 재활용, 미활용 실행계획 분석**

<p align ='center' style="display: flex; break-inside: avoid"> 
<img src="https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328023732304.png" width = 50%/>
<img src = "https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328023751395.png" width = 50%/>
</p>

- 서브쿼리로 거리 계산 결과를 2번 재활용 하였더니, 실행 시간이 10배 빨라짐. 

### ✅ 개선 사항

**🌟결과**: 

- **`쿼리 실행 시간`: 1.2초➜ 0.0102초,**
- **`평균 RPT`: 4000ms ➜ 290ms**
-  **`평균 TPS`: 100 ➜ 240**



 **🛠️ 쿼리 튜닝**

- **`동적 옵티마이저 힌트 주입`**: 실행 계획을 미리 읽고, full-scan 건 수가 10만 건 이상이면 옵티마이저에 복합 인덱스 강제 적용 힌트 (미만이면 권장 적용) 
- **`커버링 인덱스 생성`**: 복합 인덱스를 커버링 인덱스로 변경, 클러스터형 인덱스 랜덤 접근 I/O 시간 단축

<img src ='https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412171840198.png' align='center'/>

- 서브 쿼리로 거리 계산문 재활용
- JPA  ➜ JDBC 변환하여 영속성 캐싱 피하기

**결과 사진**

- 기존 1201ms의 쿼리를 10.2초의 쿼리로 단축

<img src = "https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328023751395.png" width = 70% align='center'/>

- **`평균 RPT`: 4000ms ➜ 290ms**, **`평균 TPS`: 100 ➜ 240**

<img src = "https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-03/image-20250328025139102.png" width = 100%/>

## (4) 3차 고도화: MySQL 공간 객체 활용

### 🛠️ 부하테스트 기준 2차 고도화와 동일

### 🚨 문제 상황

현직자 멘토링에서 다음 2가지를 지적 받음

- 강제 힌트 적용은 옵티마이저를 변형 시킬 수 있어 팀원 쿼리에도 영향을 주기에 안 좋은 개선 방향
- 속도를 위해 비즈니스 로직을 전부 쿼리에 넣는 행위는 관심사 분리를 역행하는 행위 

쿼리에 대한 강제 튜닝 없이 2차 고도화 의 퍼포먼스와 동급, 그 이상을 목표로 고도화 진행

### 📊 트러블 슈팅

🧪 **옵티마이저의 자연스러운 보조 인덱스(B+tree) 사용 시점 확인 실험**

- 판교 PDC 건물을 기점으로 위, 경도 데이터 10만개를 밀집, 분산해서 DB에 넣는 행위를 30번 반복 
- 힌트 없는 환경에서 언제 옵티마이저가 자연스럽게 B+tree Index를 선택하는지 확인
- 그 결과 전체의 2.33%인 데이터 (3000개) 에서만 Btween문 전에 인덱스를 활용함을 발견



🧪 **R-tree 공간 객체 인덱스 적용 후 옵티마이저의 실행 계획 재확인**

- 한 좌표에 10만 개의 데이터가 몰려 있는 Edge Case 가 아닌 한 전체 데이터 100%로 인덱스가 적용됨.
- COST는 강제 인덱싱 적용보다 6배 낮음, 실행 시간은 0.05초로 살짝 느렸으나, 서비스 적용엔 문제 없다 판단
- 해당 Edge Case의 경우, 300RPS만 넘겨도 DB가 다운된다.



### ✅ 개선 사항

**🌟결과**: 

- **`평균 RPT`: 290ms ➜ 170ms**
-  **`평균 TPS`: 240 ➜ 210**

🛠️ **서버 로직 개선**

- 쿼리문에 섞여있던 비즈니스 로직(사용자 - 일 거리 계산)을 서버로 분리, MapStruct + 병렬 스트림 조합 구현

![image-20250412174004294](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412174004294.png)

- **결과**
  : RPT가 1초 안 쪽이긴 하지만, 2차 고도화보다 퍼포먼스가 5배 저하됨. 

![parellel_stream](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/parellel_stream.png)

🛠️ **쿼리 튜닝 **

- 거리 계산, 반경 내 일거리 찾기에 mySQL의 공간 객체 함수 활용

- Spatial Index 기반 R-tree 인덱스 적용

![image-20250412174012888](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412174012888.png)

- 결과
- **`RPT`**: 290ms ➜ 170ms
- **`TPS`**: 240 ➜ 210

<img src = 'https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412212854367.png' width = 100%/>



## (5) 4차 고도화: read/write - through 패턴 구현, L1/L2 캐싱

### 🛠️ 부하테스트 기준 2차 고도화와 동일

### 🚨 문제 상황

- R-tree 인덱스는 태생적으로 커버링 인덱스가 안됨. 
  따라서 O(logN)의 랜덤 I/O 접근 + O(M)의 데이터 Read 라는 태생적 시간 복잡도 한계를 벗어나지 못함
- 해당 한계를 넘어서서 RPT, TPS 성능을 높이면서도 오류율 0.0%를 유지할 방법 모색

### ✅ 개선 사항

**🌟결과:** 

- 평균 RPT 170ms ➜ 91ms 
-  평균 TPS 240ms ➜ 290ms 

🛠️ **쿼리 튜닝 **

- AWS 기술 블로그를 보며, CQRS, DAX 등 디자인 패턴을 학습, 하지만 이런 패턴들은 MSA를 염두한 패턴이라 모놀리식에는 과하다 판단, 내장 캐싱 혹은 redis 까지만 써서, 해결 방법 모색
- L1 캐싱 (Spring 내장 Caffeine 활용) 으로 read/write-through 패턴 구현
- L2 캐싱(Redis)으로 read/write 패턴 구현
- L2 다운 시  L1이 백업할 수 있도록 조치 



# 2️⃣ '매칭 User flow 시나리오 테스트 15회와 그 개선 과정'

## (0) 테스트 목표 설정



## (1) 테스트 001 
## (2) 테스트 002: WAS, OS 튜닝 후
## (3) 테스트 003: 알림 전송 서비스에 Retry 로직 추가 후 
## (4) 테스트 004: 지연 변이 로직 구현 후
## (5) 테스트 005 ~ 010: 톰캣 Thread 수와 DB connection 의 연관 관계
## (6) 테스트 016: SQL 진입점 로깅, 에러 수집, 슬로우 쿼리 확인





<div align='center'><h3>𐦂𖨆𐀪𖠋 작성자 𐀪𐀪</h3><img src ='https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/mark_down_%EC%9E%91%EC%84%B1%EC%9E%90_%EB%AA%85%ED%95%A8.png'/></div>

 
