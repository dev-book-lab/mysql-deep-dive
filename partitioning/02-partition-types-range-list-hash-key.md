# 파티션 종류 — RANGE, LIST, HASH, KEY

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- RANGE 파티션이 날짜/ID 범위 쿼리에 적합한 이유는?
- LIST 파티션은 RANGE와 어떻게 다르며 언제 써야 하는가?
- HASH vs KEY 파티션의 분산 알고리즘 차이는?
- 각 파티션 종류별로 Partition Pruning이 동작하는 조건은?
- RANGE COLUMNS, LIST COLUMNS가 기존 RANGE/LIST와 다른 점은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 파티션 종류를 잘못 선택하면 프루닝이 안 된다

```
실제 이슈:
  "날짜 기반 로그 테이블에 HASH 파티션 적용"
  → 8개 파티션으로 균등 분산
  → 날짜 범위 쿼리: WHERE created_at >= '2024-01-01'
  → EXPLAIN: ALL 8 Partition Scan
  → 파티셔닝 전보다 오히려 느림!

원인:
  HASH 파티션은 날짜 범위 프루닝 불가
  HASH 파티션은 등치(=) 조건에서만 단일 파티션 접근
  날짜 범위 쿼리에는 RANGE 파티션이 맞음

파티션 종류 선택은 조회 패턴이 결정한다
```

---

## 😱 흔한 실수 (Before)

### 1. 날짜 범위 테이블에 HASH 파티션 적용

```sql
-- Before: 잘못된 파티션 종류 선택
CREATE TABLE events (
    id         BIGINT AUTO_INCREMENT,
    event_type VARCHAR(50),
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
)
PARTITION BY HASH(MONTH(created_at))
PARTITIONS 12;

-- 기대: 월별로 12개 파티션으로 분산 → 빠른 월별 조회
-- 실제: HASH 파티션에서 범위 쿼리 프루닝 안 됨
SELECT * FROM events WHERE created_at >= '2024-01-01';
-- EXPLAIN: ALL 12 Partition Scan!

-- RANGE 파티션이었다면:
-- → 2024 이후 파티션만 접근
```

### 2. RANGE 파티션에서 MAXVALUE 파티션 빠뜨림

```sql
-- Before: MAXVALUE 없음
CREATE TABLE logs (
    id BIGINT AUTO_INCREMENT,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- 2025년 데이터 INSERT 시:
INSERT INTO logs (created_at) VALUES ('2025-01-15');
-- Error 1526: Table has no partition for value 2025
-- 새 파티션 추가 전까지 INSERT 불가!
```

---

## ✨ 올바른 접근 (After)

```
파티션 종류 선택 기준:

조회 패턴 → 파티션 종류:

범위 조회 (>, <, BETWEEN):
  RANGE 파티션
  날짜, ID 범위, 숫자 범위
  프루닝: 범위 조건에서 동작 ✅

이산적인 값으로 분류:
  LIST 파티션
  지역(KR/US/JP), 상태(PAID/PENDING), Enum 값
  프루닝: 등치/IN 조건에서 동작 ✅

균등 분산 (범위 조회 없음):
  HASH 파티션 (사용자 정의 해시) 또는 KEY 파티션
  user_id 기반 균등 분산
  프루닝: 등치(=)에서만 동작, 범위 불가 ❌

문자열/날짜 컬럼 직접 파티션 키:
  RANGE COLUMNS / LIST COLUMNS
  YEAR() 변환 없이 DATETIME 컬럼 직접 사용 가능
```

---

## 🔬 내부 동작 원리

### 1. RANGE 파티션

```sql
-- RANGE 파티션 (정수/표현식 기반)
CREATE TABLE orders (
    id         BIGINT AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    amount     DECIMAL(10,2),
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION pmax  VALUES LESS THAN MAXVALUE  -- 반드시 포함!
);

-- RANGE COLUMNS (날짜/문자열 직접 사용 가능, MySQL 5.5+)
CREATE TABLE orders_rc (
    id         BIGINT AUTO_INCREMENT,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p2022 VALUES LESS THAN ('2023-01-01'),
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);

-- 프루닝 동작:
-- WHERE created_at >= '2024-01-01' → p2024, pmax만 접근
-- WHERE YEAR(created_at) = 2024 → (RANGE COLUMNS는 함수 미지원)
-- WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01' → p2024만
```

### 2. LIST 파티션

```sql
-- LIST 파티션 (이산 값 기반)
CREATE TABLE sales (
    id      BIGINT AUTO_INCREMENT,
    country VARCHAR(5) NOT NULL,
    amount  DECIMAL(10,2),
    PRIMARY KEY (id, country)
)
PARTITION BY LIST COLUMNS (country) (
    PARTITION p_asia   VALUES IN ('KR', 'JP', 'CN'),
    PARTITION p_europe VALUES IN ('DE', 'FR', 'GB'),
    PARTITION p_us     VALUES IN ('US', 'CA')
);

-- 주의: LIST COLUMNS는 MAXVALUE 없음
-- 등록 안 된 값 INSERT 시 오류
-- → 디폴트 파티션이 없으면 INSERT 실패
-- MySQL 8.0.19+: DEFAULT 파티션 지원 (LIST 한정)

-- LIST 파티션 (정수 기반)
CREATE TABLE reports (
    id     BIGINT AUTO_INCREMENT,
    status INT NOT NULL,         -- 1=DRAFT, 2=REVIEW, 3=APPROVED, 4=REJECTED
    PRIMARY KEY (id, status)
)
PARTITION BY LIST (status) (
    PARTITION p_active   VALUES IN (1, 2),
    PARTITION p_resolved VALUES IN (3, 4)
);

-- 프루닝 동작:
-- WHERE country = 'KR' → p_asia만 접근
-- WHERE country IN ('KR', 'US') → p_asia, p_us 접근
-- WHERE country != 'KR' → 모든 파티션 접근 (NOT IN은 프루닝 어려움)
```

### 3. HASH vs KEY 파티션

```sql
-- HASH 파티션 (사용자 정의 표현식)
CREATE TABLE users_h (
    id     BIGINT AUTO_INCREMENT,
    email  VARCHAR(100),
    PRIMARY KEY (id)
)
PARTITION BY HASH(id)
PARTITIONS 8;

-- 파티션 결정: partition_number = id % 8
-- id=1 → 파티션 1
-- id=9 → 파티션 1 (9 % 8 = 1)

-- KEY 파티션 (MySQL 내장 해시 함수, 문자열/날짜 지원)
CREATE TABLE users_k (
    id    BIGINT AUTO_INCREMENT,
    email VARCHAR(100),
    PRIMARY KEY (id)
)
PARTITION BY KEY(id)
PARTITIONS 8;

-- 차이점:
-- HASH: 사용자 정의 정수 표현식만 가능
-- KEY: MySQL 내장 해시 함수 → 문자열, 날짜 등 다양한 타입 지원
-- KEY(email) → 이메일로 파티셔닝 가능

-- 프루닝 동작:
-- WHERE id = 1 → partition_number = 1 % 8 = 1 → 1개 파티션만 접근
-- WHERE id > 100 → 어느 파티션에 해당하는지 계산 불가 → ALL Scan

-- 프루닝 조건: HASH/KEY는 등치(=)에서만 단일 파티션 접근
-- 범위 조건에서는 모든 파티션 스캔
```

### 4. 서브파티션 (복합 파티션)

```sql
-- RANGE + HASH 서브파티션
-- 월별 RANGE + 각 월 내에서 8개 HASH 분산
CREATE TABLE big_logs (
    id         BIGINT AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE (YEAR(created_at))
SUBPARTITION BY HASH(user_id)
SUBPARTITIONS 8 (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);

-- 파티션 수: 3 × 8 = 24개
-- 사용 사례: 연도별 범위 접근 + 연도 내에서 user_id 분산
-- 주의: 복잡도 증가, 실제로 잘 사용하지 않음
```

### 5. 각 파티션 종류별 프루닝 정리

```
파티션 종류별 프루닝 가능 조건:

RANGE 파티션:
  col = 값     → 단일 파티션 ✅
  col > 값     → 조건 이후 파티션 ✅
  col < 값     → 조건 이전 파티션 ✅
  col BETWEEN  → 범위 파티션 ✅
  YEAR(col)에 함수 적용 → 프루닝 제한됨

LIST 파티션:
  col = 값     → 해당 파티션 ✅
  col IN (v1, v2) → 해당 파티션들 ✅
  col != 값    → ALL Scan ❌ (어느 파티션인지 모름)

HASH 파티션:
  col = 값     → 단일 파티션 ✅ (MOD 계산 가능)
  col > 값     → ALL Scan ❌
  col BETWEEN  → ALL Scan ❌

KEY 파티션:
  col = 값     → 단일 파티션 ✅
  범위 조건    → ALL Scan ❌

확인 방법:
  EXPLAIN SELECT ... → partitions 컬럼에 접근 파티션 목록
  파티션이 1~2개면 프루닝 동작
  파티션이 전체면 ALL Scan
```

---

## 💻 실전 실험

### 실험 1: RANGE vs HASH 프루닝 비교

```sql
-- RANGE 파티션
CREATE TABLE range_test (
    id         BIGINT AUTO_INCREMENT,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- HASH 파티션
CREATE TABLE hash_test (
    id         BIGINT AUTO_INCREMENT,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id)
)
PARTITION BY HASH(id)
PARTITIONS 4;

-- 범위 조건으로 프루닝 비교
EXPLAIN SELECT * FROM range_test WHERE created_at >= '2024-01-01';
-- partitions: p2024,pmax (프루닝 동작!)

EXPLAIN SELECT * FROM hash_test WHERE id > 100;
-- partitions: p0,p1,p2,p3 (ALL Scan, 프루닝 안 됨)

-- 등치 조건으로 비교
EXPLAIN SELECT * FROM range_test WHERE id = 100;
-- (RANGE는 id로 파티션 결정 안 됨 → ALL Scan)
EXPLAIN SELECT * FROM hash_test WHERE id = 100;
-- partitions: p0 (100 % 4 = 0, 단일 파티션!)

DROP TABLE range_test, hash_test;
```

### 실험 2: LIST 파티션 프루닝

```sql
CREATE TABLE orders_list (
    id      BIGINT AUTO_INCREMENT,
    country CHAR(2) NOT NULL,
    amount  DECIMAL(10,2),
    PRIMARY KEY (id, country)
)
PARTITION BY LIST COLUMNS (country) (
    PARTITION p_kr VALUES IN ('KR'),
    PARTITION p_us VALUES IN ('US'),
    PARTITION p_jp VALUES IN ('JP'),
    PARTITION p_others VALUES IN ('DE', 'FR', 'GB', 'CN')
);

-- 등치 조건 프루닝
EXPLAIN SELECT * FROM orders_list WHERE country = 'KR';
-- partitions: p_kr (단일 파티션)

-- IN 조건 프루닝
EXPLAIN SELECT * FROM orders_list WHERE country IN ('KR', 'US');
-- partitions: p_kr,p_us (두 파티션)

-- NOT IN = ALL Scan
EXPLAIN SELECT * FROM orders_list WHERE country != 'KR';
-- partitions: p_kr,p_us,p_jp,p_others (모두)

DROP TABLE orders_list;
```

### 실험 3: RANGE COLUMNS vs RANGE(YEAR())

```sql
-- RANGE COLUMNS: DATETIME 직접 사용
CREATE TABLE rc_test (
    id BIGINT AUTO_INCREMENT,
    ts DATETIME NOT NULL,
    PRIMARY KEY (id, ts)
)
PARTITION BY RANGE COLUMNS (ts) (
    PARTITION p2023 VALUES LESS THAN ('2024-01-01 00:00:00'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01 00:00:00'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);

-- 정확한 날짜 프루닝 확인
EXPLAIN SELECT * FROM rc_test WHERE ts >= '2024-06-01';
-- partitions: p2024,pmax (정확)

EXPLAIN SELECT * FROM rc_test WHERE ts >= '2024-06-01' AND ts < '2025-01-01';
-- partitions: p2024 (1개 파티션!)

DROP TABLE rc_test;
```

---

## 📊 성능/비교 비교

```
파티션 종류별 사용 사례와 프루닝 효과:

시나리오: 1억 건 로그 테이블, 4개 파티션

RANGE (날짜별):
  범위 조회 (최근 1개월): 1/12 파티션만 스캔
  효과: 92% 데이터 스킵 → 성능 극적 개선
  TTL 삭제: DROP PARTITION → 밀리초

HASH (8 파티션):
  등치 조회 (id = ?): 1/8 파티션 → 약 87.5% 스킵
  범위 조회 (최근 1개월): 8/8 파티션 → 스킵 없음
  날짜 범위 쿼리가 많으면 HASH는 부적합

LIST (국가별):
  WHERE country = 'KR': 1개 파티션 → 나머지 스킵
  WHERE country IN ('KR', 'US'): 2개 파티션
  NOT IN 또는 범위: 전체 스캔

KEY (user_id, 8 파티션):
  WHERE user_id = 123: 1/8 파티션 (해시 계산)
  WHERE user_id > 1000: ALL Scan
  사용자별 데이터 균등 분산에 적합
```

---

## ⚖️ 트레이드오프

```
RANGE 파티션:
  ✅ 날짜/숫자 범위 쿼리 프루닝 최적
  ✅ TTL 삭제 DROP PARTITION 사용 가능
  ✅ 범위 관련 프루닝 가장 강력
  ❌ 파티션 불균형 가능 (특정 시기에 데이터 집중)
  ❌ 새 파티션 추가 DDL 필요 (MAXVALUE 없으면 INSERT 실패)

LIST 파티션:
  ✅ 이산 값 분류에 명확
  ✅ IN 조건 프루닝 동작
  ❌ 정의되지 않은 값 INSERT 실패 (DEFAULT 필요)
  ❌ 새 enum 값 추가 시 파티션 재구성 필요

HASH 파티션:
  ✅ 균등한 데이터 분산 보장
  ✅ Hot Partition 문제 없음
  ❌ 범위 조회 프루닝 없음 → 범위 조회가 많으면 부적합
  ❌ 파티션 수 변경 시 재분배 비용 큼

KEY 파티션:
  ✅ 문자열/날짜 등 다양한 타입 지원
  ✅ MySQL 내장 해시 → 분산 품질 보장
  ❌ HASH와 동일하게 범위 프루닝 불가
```

---

## 📌 핵심 정리

```
파티션 종류 선택 핵심:

RANGE: 날짜/숫자 범위 쿼리가 주 패턴
  MAXVALUE 파티션 반드시 포함
  RANGE COLUMNS로 DATETIME 직접 사용 권장

LIST: 이산적인 고정 값(국가, 상태코드)
  LIST COLUMNS로 문자열 직접 사용 가능
  디폴트 파티션 고려 (MySQL 8.0.19+)

HASH: 균등 분산이 목적, 등치 조회 주 패턴
  파티션 수는 2의 거듭제곱 권장
  범위 조회 많으면 RANGE로 변경 고려

KEY: 문자열/날짜 컬럼으로 균등 분산
  HASH의 더 유연한 버전

프루닝 조건 요약:
  RANGE: 범위 조건(=, >, <, BETWEEN) 모두 프루닝
  LIST: 등치(=), IN 프루닝 / NOT IN은 ALL
  HASH/KEY: 등치(=)만 프루닝 / 범위는 ALL
```

---

## 🤔 생각해볼 문제

**Q1.** 주문 테이블을 파티셔닝하려 한다. 다음 두 가지 주요 조회 패턴이 있을 때 어떤 파티션 종류를 선택하고 파티션 키를 무엇으로 설정하는가?

```
패턴 A: WHERE user_id = ? AND order_date >= ? (80%)
패턴 B: WHERE order_date >= ? AND order_date < ? (배치 집계, 20%)
```

<details>
<summary>해설 보기</summary>

**RANGE COLUMNS 파티션, 파티션 키: order_date (월별 또는 분기별)**

**이유**:
- 패턴 B(20%)가 명확하게 날짜 범위 → RANGE 파티션에서 프루닝 효과
- 패턴 A(80%)는 user_id + order_date 복합 조건 → 파티션 프루닝(날짜) + 파티션 내 user_id 인덱스 조합

**설계 예시**:
```sql
CREATE TABLE orders (
    id         BIGINT AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    order_date DATE NOT NULL,
    amount     DECIMAL(10,2),
    PRIMARY KEY (id, order_date),
    INDEX idx_user_date (user_id, order_date)
)
PARTITION BY RANGE COLUMNS (order_date) (
    PARTITION p2024q1 VALUES LESS THAN ('2024-04-01'),
    PARTITION p2024q2 VALUES LESS THAN ('2024-07-01'),
    PARTITION p2024q3 VALUES LESS THAN ('2024-10-01'),
    PARTITION p2024q4 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
```

패턴 A: `order_date >= '2024-07-01'` 조건으로 프루닝 후 `user_id` 인덱스 탐색
패턴 B: 날짜 범위로 프루닝 → 해당 파티션 집계

HASH를 선택하면 패턴 B의 날짜 범위 조회에서 ALL Partition Scan이 발생합니다.

</details>

---

<div align="center">

**[⬅️ 파티셔닝 판단 기준](./01-when-to-use-partitioning.md)** | **[홈으로 🏠](../README.md)** | **[다음: 파티션 프루닝 ➡️](./03-partition-pruning.md)**

</div>
