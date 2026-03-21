# AUTO_INCREMENT의 함정 — Gap, 재사용 불가, 분산 환경 한계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 트랜잭션 롤백 후 AUTO_INCREMENT 값이 재사용되지 않아 Gap이 생기는 이유는?
- 서버 재시작 후 InnoDB가 AUTO_INCREMENT 최대값을 재계산하는 방식(MySQL 8.0 전/후 차이)은?
- 분산 환경에서 AUTO_INCREMENT 채번 충돌이 발생하는 시나리오는?
- Snowflake ID, UUID의 장단점과 선택 기준은?
- `innodb_autoinc_lock_mode` 설정이 성능에 미치는 영향은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### AUTO_INCREMENT는 순차적이지 않고 재사용되지 않는다

```
실제 이슈:
  이슈 1: "우리 서비스가 10만 건인데 ID가 50만이야?"
    트랜잭션 롤백 = AUTO_INCREMENT 증가 후 롤백 = ID 버려짐
    배치 INSERT 실패 = 할당받은 ID 범위 전체 버려짐
    → Gap이 생기는 건 정상, ID 연속성을 가정하면 문제

  이슈 2: 서버 재시작 후 ID 충돌 (MySQL 5.7 이하)
    t 테이블: id 최대값 = 100
    id=90~100인 Row를 DELETE
    서버 재시작 → AUTO_INCREMENT = 90으로 재계산 (메모리에서 계산)
    새 INSERT → id=90 할당 → 이전 데이터 참조 혼선

  이슈 3: 분산 DB에서 채번 충돌
    샤드 A: AUTO_INCREMENT = 1, 3, 5, 7 ...
    샤드 B: AUTO_INCREMENT = 1, 3, 5, 7 ... ← 중복!
    → 샤드 간 동일 ID 충돌 → FK 오류
```

---

## 😱 흔한 실수 (Before)

### 1. ID 연속성 가정

```python
# Before: ID가 연속적이라 가정
last_id = db.query("SELECT MAX(id) FROM orders")
# 다음 ID = last_id + 1 예상
next_id = last_id + 1
# 이미 트랜잭션 롤백으로 next_id가 존재하거나
# 다른 프로세스가 먼저 할당받을 수 있음
# → 잘못된 가정
```

```sql
-- Before: ID 범위로 배치 처리
-- "마지막 처리 ID + 1부터 순서대로 처리"
DECLARE @last_id INT = 0;
WHILE 1=1 DO
    SELECT id, data FROM orders WHERE id > @last_id LIMIT 1000;
    -- Gap이 있으면 일부 범위를 처리 안 하고 넘어갈 수 있음
    SET @last_id = MAX(id from result);
END WHILE;
-- Gap이 크면 비효율적인 범위 스캔
```

### 2. 분산 환경에서 단순 AUTO_INCREMENT

```sql
-- Before: 2개 샤드에서 AUTO_INCREMENT 충돌
-- 샤드 A (server_id=1):
CREATE TABLE orders (id BIGINT AUTO_INCREMENT PRIMARY KEY) AUTO_INCREMENT=1;
-- id: 1, 2, 3, 4 ...

-- 샤드 B (server_id=2):
CREATE TABLE orders (id BIGINT AUTO_INCREMENT PRIMARY KEY) AUTO_INCREMENT=1;
-- id: 1, 2, 3, 4 ... ← 동일!

-- 두 샤드 데이터를 머지 시:
-- id=1이 두 개 → PRIMARY KEY 충돌
```

### 3. UUID를 PK로 사용 후 성능 저하

```sql
-- Before: UUID를 VARCHAR로 PK 사용
CREATE TABLE events (
    id   CHAR(36) NOT NULL PRIMARY KEY,  -- 'xxxxxxxx-xxxx-...'
    data TEXT
);

-- INSERT 시 성능 문제:
-- UUID는 랜덤 → B-Tree 인덱스에 무작위 삽입
-- 페이지 분할 빈번 발생 → 성능 저하
-- 인덱스 크기: CHAR(36) = 36바이트 (BIGINT의 4.5배)
-- 모든 FK도 36바이트 × FK 수 = 큰 인덱스
```

---

## ✨ 올바른 접근 (After)

```
AUTO_INCREMENT 올바른 이해:

Gap은 정상:
  트랜잭션 롤백 → ID 버려짐 (설계상 정상)
  ID 연속성 가정 = 잘못된 가정
  "50만 건인데 ID가 100만" → 기능 문제 없음

서버 재시작 대비:
  MySQL 8.0+: AUTO_INCREMENT가 Redo Log에 기록됨
  → 재시작 후에도 정확한 AUTO_INCREMENT 유지 (MySQL 5.7 문제 해결)

분산 환경 전략:
  방법 1: AUTO_INCREMENT + 시작값 분리
    샤드 A: AUTO_INCREMENT 1, 3, 5 ... (홀수)
    샤드 B: AUTO_INCREMENT 2, 4, 6 ... (짝수)

  방법 2: Snowflake ID (Twitter 오픈소스)
    64비트: 타임스탬프(41) + 데이터센터(5) + 서버(5) + 시퀀스(12)
    전역 유일, 시간 순 정렬, 분산 환경 최적

  방법 3: UUID v7 (시간 기반, MySQL 8.0.35+)
    시간 기반 정렬 → B-Tree 삽입 효율
    전역 유일
```

---

## 🔬 내부 동작 원리

### 1. AUTO_INCREMENT Gap이 발생하는 이유

```
AUTO_INCREMENT 할당 원리:

트랜잭션 롤백:
  BEGIN;
  INSERT INTO orders (user_id) VALUES (1);  -- id=1001 할당
  INSERT INTO orders (user_id) VALUES (2);  -- id=1002 할당
  ROLLBACK;  -- INSERT 취소
  -- 하지만 AUTO_INCREMENT는 1003으로 유지 (1001, 1002 버려짐)
  
  이유:
  AUTO_INCREMENT는 동시성 성능을 위해
  락 범위를 최소화하는 방식으로 설계됨
  롤백을 허용하면 다른 트랜잭션이 같은 ID를 받을 수 있음
  → 롤백 시 ID 재사용 = 데이터 충돌 위험
  → 설계상 재사용하지 않는 것이 올바른 동작

BULK INSERT Gap:
  INSERT INTO t VALUES (NULL,...),(NULL,...),(NULL,...); -- 100건
  → 100개 ID를 한 번에 할당
  쿼리 실패 시 100개 ID 모두 버려짐
  innodb_autoinc_lock_mode에 따라 더 많이 할당될 수 있음
```

### 2. innodb_autoinc_lock_mode 설정

```
innodb_autoinc_lock_mode 설정값:

0 (Traditional):
  모든 INSERT에 테이블 레벨 AUTO_INCREMENT 락
  INSERT 완료까지 다른 INSERT 차단
  정확한 순번 보장, 성능 낮음

1 (Consecutive, MySQL 5.7 기본):
  단순 INSERT: 가벼운 뮤텍스 (짧은 시간만 락)
  BULK INSERT: 전체 완료까지 테이블 락
  ID 범위 연속성 보장

2 (Interleaved, MySQL 8.0 기본):
  모든 INSERT에 가벼운 뮤텍스 (짧은 시간만)
  높은 동시성, ID 순서 불연속 가능
  BULK INSERT에서 ID 비연속 가능

성능 비교:
  모드 0: 낮은 동시성, 엄격한 ID 순서
  모드 1: 중간 동시성
  모드 2: 높은 동시성 (8.0 기본)

주의:
  모드 2 + binlog_format=STATEMENT 조합:
  Replica에서 INSERT... SELECT 실행 시 ID 순서 다를 수 있음
  → MySQL 8.0은 기본 ROW 포맷으로 이 문제 회피
```

### 3. MySQL 5.7 vs 8.0 AUTO_INCREMENT 재시작 차이

```sql
-- 테스트 시나리오:
CREATE TABLE ai_test (id INT AUTO_INCREMENT PRIMARY KEY);
INSERT INTO ai_test VALUES (NULL),(NULL),(NULL); -- id: 1, 2, 3
DELETE FROM ai_test WHERE id > 1;  -- 2, 3 삭제
-- 현재 AUTO_INCREMENT = 4

-- MySQL 5.7 서버 재시작 후:
-- InnoDB: AUTO_INCREMENT 재계산 = SELECT MAX(id) + 1 = 2
-- 다음 INSERT → id=2 (이전에 삭제한 id와 동일!)
-- → 이전 데이터가 이 id를 참조하고 있다면 혼선

-- MySQL 8.0 서버 재시작 후:
-- AUTO_INCREMENT 값이 Redo Log에 기록됨 (DDL 로그)
-- 재시작 후: AUTO_INCREMENT = 4 (재시작 전 값 유지)
-- → 안전

-- 확인:
SHOW CREATE TABLE ai_test\G
-- AUTO_INCREMENT=4 확인

-- MySQL 5.7에서 안전하게 사용하려면:
ALTER TABLE ai_test AUTO_INCREMENT=100000;
-- 충분히 큰 값으로 설정 (DELETE된 범위보다 높게)
```

### 4. Snowflake ID 구조

```
Snowflake ID (64비트):

비트 구조:
  1비트: 부호 (항상 0)
  41비트: 밀리초 타임스탬프 (기준 시각부터 경과 밀리초)
  10비트: 워커 ID (데이터센터 5비트 + 서버 5비트)
  12비트: 시퀀스 번호 (같은 밀리초 내 순번)

생성 가능 ID 수:
  초당: 4,096 × 1,000 = 4,096,000 개/서버
  BIGINT 범위 내 수십 년간 고유 보장

장점:
  전역 유일 (서버 간 충돌 없음)
  시간 기반 정렬 (B-Tree 삽입 효율)
  오프라인 생성 가능 (DB 조회 불필요)
  64비트 = BIGINT와 동일 크기

단점:
  워커 ID 관리 필요
  구현 복잡도 (라이브러리 사용 권장)
  시스템 시간에 의존 (시간 역행 문제)

Java 구현 (Twitter4J 또는 직접):
  SnowflakeIdGenerator generator = new SnowflakeIdGenerator(workerId);
  long id = generator.nextId();  // 64비트 BIGINT
```

### 5. UUID 전략 비교

```sql
-- UUID v4 (완전 랜덤):
SELECT UUID();
-- '6ba7b810-9dad-11d1-80b4-00c04fd430c8'
-- 문제: 완전 랜덤 → B-Tree 삽입 무작위 → 페이지 분할 빈번

-- UUID_SHORT():
SELECT UUID_SHORT();
-- 94558195826753537 (BIGINT 수준)
-- 서버 시작 시간 + server_id 기반
-- 같은 서버에서는 순차적

-- UUID v7 (시간 기반, RFC 9562):
-- MySQL 8.0.35+: UUID() 함수 내부적으로 v7 사용 검토 중
-- 앞 48비트: 밀리초 타임스탬프 → 시간 순 정렬
-- B-Tree 삽입 효율 개선

-- BINARY(16) UUID 저장 (공간 효율):
CREATE TABLE events_uuid (
    id   BINARY(16) PRIMARY KEY DEFAULT (UUID_TO_BIN(UUID(), 1)),
    data TEXT
);
-- UUID_TO_BIN(uuid, 1): 시간 기반 재배치 → B-Tree 효율 개선
-- CHAR(36) 36바이트 → BINARY(16) 16바이트 (절반 이하)

-- 조회:
SELECT BIN_TO_UUID(id, 1), data FROM events_uuid;
```

---

## 💻 실전 실험

### 실험 1: AUTO_INCREMENT Gap 확인

```sql
CREATE TABLE gap_test (
    id  INT AUTO_INCREMENT PRIMARY KEY,
    val INT
) ENGINE=InnoDB;

-- 정상 INSERT
INSERT INTO gap_test (val) VALUES (1),(2),(3);
SELECT * FROM gap_test;  -- id: 1, 2, 3

-- 트랜잭션 롤백
START TRANSACTION;
INSERT INTO gap_test (val) VALUES (10),(11),(12);  -- id: 4, 5, 6 할당
ROLLBACK;  -- INSERT 취소

-- 다음 INSERT
INSERT INTO gap_test (val) VALUES (99);
SELECT * FROM gap_test;
-- id: 1, 2, 3, 7 (4, 5, 6 Gap!)

-- AUTO_INCREMENT 현재 값 확인
SHOW TABLE STATUS LIKE 'gap_test'\G
-- Auto_increment: 8

DROP TABLE gap_test;
```

### 실험 2: 분산 환경 ID 충돌 방지 설정

```sql
-- 2개 서버에서 짝수/홀수로 ID 분리
-- 서버 A (my.cnf):
-- auto_increment_increment = 2
-- auto_increment_offset = 1
-- → id: 1, 3, 5, 7 ...

-- 서버 B (my.cnf):
-- auto_increment_increment = 2
-- auto_increment_offset = 2
-- → id: 2, 4, 6, 8 ...

-- 현재 세션에서 확인
SHOW VARIABLES LIKE 'auto_increment%';
-- auto_increment_increment: 1 (기본)
-- auto_increment_offset: 1 (기본)

-- 세션 변경 후 테스트
SET auto_increment_increment = 2;
SET auto_increment_offset = 1;
CREATE TABLE dist_test (id BIGINT AUTO_INCREMENT PRIMARY KEY, val INT);
INSERT INTO dist_test (val) VALUES (1),(2),(3),(4),(5);
SELECT * FROM dist_test;  -- id: 1, 3, 5, 7, 9 (홀수만)

DROP TABLE dist_test;
SET auto_increment_increment = DEFAULT;
SET auto_increment_offset = DEFAULT;
```

### 실험 3: UUID vs BIGINT 인덱스 삽입 성능

```sql
-- BIGINT AUTO_INCREMENT
CREATE TABLE ai_bigint (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    data CHAR(100)
) ENGINE=InnoDB;

-- CHAR(36) UUID
CREATE TABLE ai_uuid (
    id   CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    data CHAR(100)
) ENGINE=InnoDB;

-- 삽입 성능 비교 (각 10만 건)
SET @t = SYSDATE(6);
INSERT INTO ai_bigint (data)
SELECT REPEAT('x', 100)
FROM (SELECT @r:=@r+1 FROM information_schema.columns a, (SELECT @r:=0) r LIMIT 100000) t;
SELECT CONCAT('BIGINT: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

SET @t = SYSDATE(6);
INSERT INTO ai_uuid (data)
SELECT REPEAT('x', 100)
FROM (SELECT @r:=@r+1 FROM information_schema.columns a, (SELECT @r:=0) r LIMIT 100000) t;
SELECT CONCAT('UUID: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');
-- UUID가 더 느림 (랜덤 삽입으로 페이지 분할)

DROP TABLE ai_bigint, ai_uuid;
```

---

## 📊 성능/비용 비교

```
ID 생성 전략 비교:

AUTO_INCREMENT BIGINT:
  속도: 가장 빠름 (DB가 자동 생성)
  크기: 8바이트
  정렬: 삽입 순 (B-Tree 효율적)
  전역 유일: 단일 서버만 (분산 불가)
  사용: 단일 DB 환경

UUID v4 CHAR(36):
  속도: 느림 (랜덤 삽입, 페이지 분할)
  크기: 36바이트
  정렬: 무작위 (B-Tree 비효율)
  전역 유일: 보장
  사용: 외부 공개 불가 ID (보안)

UUID v4 BINARY(16):
  속도: CHAR보다 빠름 (절반 크기)
  크기: 16바이트
  정렬: 무작위 (여전히 B-Tree 비효율)
  전역 유일: 보장

Snowflake ID BIGINT:
  속도: AUTO_INCREMENT와 유사 (시간 기반 순차)
  크기: 8바이트
  정렬: 대략 시간 순 (B-Tree 효율)
  전역 유일: 워커 ID 관리 시 보장
  사용: 분산 환경 최적

UUID v7 BINARY(16):
  속도: Snowflake 수준 (시간 기반 순차)
  크기: 16바이트
  정렬: 시간 순 (B-Tree 효율)
  전역 유일: 보장
  사용: 표준 UUID + 분산 환경
```

---

## ⚖️ 트레이드오프

```
ID 전략 트레이드오프:

AUTO_INCREMENT:
  ✅ 단순, 빠름, B-Tree 효율
  ✅ 작은 인덱스 (8바이트)
  ❌ 단일 DB만 (분산 불가)
  ❌ Gap 발생 (재사용 없음)
  ❌ MySQL 5.7: 재시작 후 중복 가능
  ❌ 외부에 노출 시 추측 가능 (보안)

UUID v4:
  ✅ 전역 유일
  ✅ 외부 노출 안전 (예측 불가)
  ✅ 앱에서 생성 가능 (DB 불필요)
  ❌ 랜덤 삽입 → 페이지 분할 → 느림
  ❌ 큰 인덱스 (36바이트)

Snowflake ID:
  ✅ 전역 유일 (분산 환경)
  ✅ 시간 순 정렬 (B-Tree 효율)
  ✅ 8바이트 (BIGINT)
  ❌ 워커 ID 관리 필요
  ❌ 시간 역행 시 문제

실무 선택 가이드:
  단일 DB + 빠른 개발: AUTO_INCREMENT BIGINT
  외부 공개 ID (보안): UUID + 내부 BIGINT 함께 사용
  분산 DB + 고성능: Snowflake ID
  표준 준수 + 분산: UUID v7 (MySQL 8.0.35+)
```

---

## 📌 핵심 정리

```
AUTO_INCREMENT 핵심:

Gap은 정상 동작:
  트랜잭션 롤백, BULK INSERT 실패 → ID 버려짐
  ID 연속성 가정은 잘못된 가정

MySQL 8.0 개선:
  AUTO_INCREMENT가 Redo Log에 기록됨
  재시작 후에도 이전 값 유지 (5.7의 재시작 중복 문제 해결)

innodb_autoinc_lock_mode:
  0: 전통적 (엄격, 느림)
  1: 연속 (MySQL 5.7 기본)
  2: 교차 (MySQL 8.0 기본, 빠름)

분산 환경:
  단순 AUTO_INCREMENT → 충돌
  해결: Snowflake ID, UUID v7, auto_increment_offset 분리

ID 선택:
  단일 DB: AUTO_INCREMENT BIGINT
  분산 DB: Snowflake ID 또는 UUID v7
  외부 공개: UUID v4 (예측 불가)
  성능 + 전역 유일: UUID_TO_BIN(UUID(), 1) 또는 Snowflake
```

---

## 🤔 생각해볼 문제

**Q1.** 주문 시스템에서 AUTO_INCREMENT PK를 사용 중에 "같은 PK가 두 개"라는 오류가 발생했다. MySQL 5.7 환경에서 이 현상이 발생하는 시나리오를 설명하고, 즉각 조치 방법을 제시하라.

<details>
<summary>해설 보기</summary>

**발생 시나리오 (MySQL 5.7)**:

1. orders 테이블의 현재 최대 id = 1000
2. id=950~1000 Row들을 DELETE로 삭제
3. 서버 재시작 → MySQL 5.7이 `SELECT MAX(id) + 1`으로 AUTO_INCREMENT 재계산 = 951
4. 새 주문 INSERT → id=951 할당
5. 기존 데이터에서 id=951을 참조하는 FK가 있었다면 → 데이터 혼선

**즉각 조치**:
```sql
-- 1. 현재 최대 id 확인
SELECT MAX(id) FROM orders;  -- 예: 2000

-- 2. AUTO_INCREMENT를 최대값 이상으로 강제 설정
ALTER TABLE orders AUTO_INCREMENT = 2001;

-- 3. MySQL 8.0 업그레이드 계획 수립 (근본 해결)
-- 8.0에서는 AUTO_INCREMENT가 Redo Log에 기록되어 재시작 후에도 유지
```

**근본 해결**: MySQL 8.0으로 업그레이드하거나, 삭제된 ID 범위를 고려한 AUTO_INCREMENT 값을 항상 수동으로 확인하는 배치를 운영합니다.

</details>

---

**Q2.** 마이크로서비스 환경에서 3개의 서비스가 각각 별도의 MySQL 인스턴스를 가진다. 주문 ID를 글로벌하게 유일하게 관리하려고 한다. Snowflake ID와 UUID v4의 구체적인 트레이드오프를 비교하고 선택 근거를 제시하라.

<details>
<summary>해설 보기</summary>

**Snowflake ID**:
- BIGINT (8바이트) → 인덱스 크기 작음, 정렬 효율
- 시간 기반 → B-Tree 순차 삽입
- 워커 ID 관리 필요 (서비스 3개 × 서버 N대에 고유 워커 ID 할당)
- 구현: 각 서비스 인스턴스에 워커 ID 설정 (Consul, Kubernetes ConfigMap)

**UUID v4**:
- 16바이트 (BINARY) 또는 36바이트 (CHAR)
- 외부 공개 시 추측 불가 (보안 이점)
- 워커 ID 관리 불필요 (어디서든 독립 생성)
- B-Tree 삽입 비효율 (랜덤 → 페이지 분할)

**선택 근거**:

주문 ID가 외부에 노출되지 않는다면 → **Snowflake ID**:
- INSERT 성능 중요 (주문량이 많을 때)
- BIGINT FK로 조인 성능 우수
- 워커 ID = 서비스명 해시 또는 환경변수로 관리

주문 ID를 클라이언트에 직접 노출한다면 → **UUID v4** (보안):
- URL에 `/orders/6ba7b810-9dad-11d1-80b4-00c04fd430c8` 형태
- 연속적이지 않아 추측 공격 방어

**실무 패턴**: 내부 PK는 BIGINT AUTO_INCREMENT, 외부 공개용은 UUID v4를 별도 컬럼으로 둬서 두 장점을 모두 취합니다.

```sql
CREATE TABLE orders (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,  -- 내부 PK (빠른 조인)
    public_id  CHAR(36) NOT NULL DEFAULT (UUID()),  -- 외부 공개 ID
    UNIQUE KEY uk_public_id (public_id)
);
```

</details>

---

<div align="center">

**[⬅️ 정규화 vs 비정규화](./03-normalization-vs-denormalization.md)** | **[홈으로 🏠](../README.md)** | **[다음: Online DDL ➡️](./05-online-ddl-schema-change.md)**

</div>
