<h1 align = 'center'> 매칭 서비스 기술 보고서 </h1>

### 📑 목차

- [0️⃣ 매칭 서비스 한 눈에 이해하기](#0️⃣-매칭-서비스-한-눈에-이해하기)
  - [⑴ 개요](#(1)-개요)
  - [⑵ USER FLOW](#⑵-user-flow)
  - [⑶ 서비스 아키텍쳐](⑶-서비스-아키텍쳐)
  - [⑷ ERD](#⑷-erd)
  
- [1️⃣ '근처 일거리 찾기 API' 고도화 과정과 결과](#1️⃣-근처-일거리-찾기-api-고도화-과정과-결과)
  - [⑴ 최초 구현](#⑴-최초-구현)
  - [⑵ 1차 고도화: 쿼리 전략 개선](#⑵-1차-고도화-쿼리-전략-개선)
  - [⑶ 2차 고도화: 쿼리 튜닝](#⑶-2차-고도화-쿼리-튜닝)
  - [⑷ 3차 고도화: mysql 공간 객체 활용](#⑷-3차-고도화-mysql-공간-객체-활용)
  - [⑸ 4차 고도화: read/write - through 패턴 구현, l1/l2 캐싱](#⑸-4차-고도화-readwrite---through-패턴-구현-l1l2-캐싱)

- [2️⃣ '매칭 User flow 시나리오 테스트 15회와 그 개선 과정'](#2️⃣-매칭-user-flow-시나리오-테스트-15회와-그-개선-과정)
  - [⑴ 테스트 001](#⑴-테스트-001)
  - [⑵ 테스트 002: was, os 튜닝 후](#⑵-테스트-002-was-os-튜닝-후)
  - [⑶ 테스트 003: 알림 전송 서비스에 retry 로직 추가 후](#⑶-테스트-003-알림-전송-서비스에-retry-로직-추가-후)
  - [⑷ 테스트 004: 지연 변이 로직 구현 후](#⑷-테스트-004-지연-변이-로직-구현-후)
  - [⑸ 테스트 005 ~ 010: 톰캣 thread 수와 db connection 의 연관 관계](#⑸-테스트-005--010-톰캣-thread-수와-db-connection-의-연관-관계)
  - [⑹ 테스트 016: sql 진입점 로깅, 에러 수집, 슬로우 쿼리 확인](#⑹-테스트-016-sql-진입점-로깅-에러-수집-슬로우-쿼리-확인)



# 0️⃣ 매칭 서비스 한 눈에 이해하기

## ⑴ 개요

- 하기 싫은 **`소일 거리`** 대신 해줄 사람을 매칭 해주는 서비스
- **`당근 알바`, `해주세요`** 같은 앱을 소일거리 매칭으로 특화

## ⑵ USER FLOW

### A. 일 등록 부터 해결까지

![image-20250412125244678](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412125244678.png)

- `파란색`: 사용자 모두 긍정적인 행동으로, 서비스를 이용 후 종료
- `빨간색`: 사용자 중 하나가 서비스 이용을 종료하는 행동을 취했을 때, 조치

### B.  `#1`번과 `#2`번 사이, 구직자가 일거리를 찾는 과정

![image-20250412133004318](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412133004318.png)

## ⑶ 서비스 아키텍쳐

![SPOT_JOB_MATCHING_ARCHITECTURE](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/SPOT_JOB_MATCHING_ARCHITECTURE.png)

- Redis 활용 Read/Write-Through Pattern 구현
- 모든 API 요청은 계약 상대방에게 **`FCM 알림`** 전송

## ⑷ ERD

![spot_job_matching_erd](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/spot_job_matching_erd.png)

### A. Matching 교차테이블에 대한 이해

![image-20250412154814804](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/image-20250412154814804.png)

- 각 회원은 일거리에 대해서 자신만의 상태 (**`MatchingStatus`**)를 가진다. 가질 수 있는 상태는 위와 같다.

# 1️⃣ '근처 일거리 찾기 API' 고도화 과정과 결과
## ⑴ 최초 구현
## ⑵ 1차 고도화: 쿼리 전략 개선
## ⑶ 2차 고도화: 쿼리 튜닝
## ⑷ 3차 고도화: MySQL 공간 객체 활용
## ⑸ 4차 고도화: read/write - through 패턴 구현, L1/L2 캐싱

# 2️⃣ '매칭 User flow 시나리오 테스트 15회와 그 개선 과정'
## ⑴ 테스트 001 
## ⑵ 테스트 002: WAS, OS 튜닝 후
## ⑶ 테스트 003: 알림 전송 서비스에 Retry 로직 추가 후 
## ⑷ 테스트 004: 지연 변이 로직 구현 후
## ⑸ 테스트 005 ~ 010: 톰캣 Thread 수와 DB connection 의 연관 관계
## ⑹ 테스트 016: SQL 진입점 로깅, 에러 수집, 슬로우 쿼리 확인





<div align='center'><h3>𐦂𖨆𐀪𖠋 작성자 𐀪𐀪</h3><img src ='https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-04/mark_down_%EC%9E%91%EC%84%B1%EC%9E%90_%EB%AA%85%ED%95%A8.png'/></div>

 
