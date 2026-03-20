# 파티션의 함정 — PRIMARY KEY 제약, 외래키 불가, 파티션 키 실수

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PRIMARY KEY에 파티션 키를 반드시 포함해야 하는 이유는?
- 파티션 테이블에서 FOREIGN KEY가 지원되지 않는 근본적 이유는?
- 파티션 키를 잘못 선택했을 때 Hot Partition 현상이란?
- NULL 값이 파티션 테이블에서 어떻게 처리되는가?
- 파티션 키로 수정할 수 없는 업데이트 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 파티션 테이블 도입 후 예상치 못한 제약들

```
실제 이슈들:
  이슈 1:
    "파티션 테이블에 FOREIGN KEY 걸려고 했는데 오류가 납니다"
    → 파티션 테이블은 FOREIGN KEY 지원 안 됨
    → 이미 파티셔닝 결정 후 알게 됨 → 롤백 비용

  이슈 2:
    "Auto Increment PK가 있는데 파티션 키를 PK에 추가하라고 하네요"
    → 기존 PK (id) → 새 PK (id, created_at)
    → 기존 FK가 id를 참조하고 있었음 → 모든 FK 수정 필요
    → 대규모 스키마 변경

  이슈 3:
    "파티션 걸었는데 특정 파티션만 느립니다"
    → 파티션 키가 단조증가(Auto Increment) → 최신 파티션에 모든 INSERT 집중
    → Hot Partition 현상

미리 알았다면 피할 수 있는 함정들
```

---

## 😱 흔한 실수 (Before)

### 1. FOREIGN KEY 있는 테이블에 파티셔닝 시도

```sql
-- Before: FK 참조하는 테이블에 파티셔닝
CREATE TABLE orders (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)  -- FK 있음
) ENGINE=InnoDB;

-- 파티션 추가 시도
ALTER TABLE orders PARTITION BY RANGE COLUMNS (created_at) (...);
-- Error 1217: Cannot delete or update a parent row: a foreign key constraint fails
-- 또는
-- Error: Foreign keys are not supported for partitioned tables

-- FK가 있으면 파티셔닝 불가
-- FK를 제거해야 파티셔닝 가능 → 데이터 무결성은 애플리케이션이 담당
```

### 2. PK에 파티션 키 없이 CREATE

```sql
-- Before: 잘못된 파티션 테이블 생성
CREATE TABLE logs (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,  -- id만 PK
    user_id    BIGINT NOT NULL,
    created_at DATETIME NOT NULL
)
PARTITION BY RANGE COLUMNS (created_at) (...);
-- Error 1503: A PRIMARY KEY must include all columns in the table's partitioning function
-- created_at이 PK에 없음 → 오류
```

### 3. 단조증가 ID를 파티션 키로 사용

```sql
-- Before: id(Auto Increment)로 HASH 파티셔닝
CREATE TABLE orders (
    id         BIGINT AUTO_INCREMENT,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id)
)
PARTITION BY HASH(id)
PARTITIONS 8;

-- 이론상 8개 파티션에 균등 분산
-- 하지만:
-- 최신 INSERT는 가장 큰 id → id % 8 패턴이 반복적
-- 실제로는 균등 분산되지만, 범위 조회 프루닝 없음
-- id > 1000000 → ALL Partition Scan
-- 날짜 범위 조회: 역시 ALL Partition Scan
```

---

## ✨ 올바른 접근 (After)

```
파티션 도입 전 체크리스트:

① FOREIGN KEY 확인:
  참조하거나 참조되는 FK 존재? → 제거 후 파티셔닝
  FK 제거 시 무결성 보장 방법은?

② PRIMARY KEY 재설계:
  기존 PK (id) → 새 PK (id, 파티션 키)
  PK가 복합 키가 되면 다른 테이블의 FK 참조 어떻게?

③ NULL 처리 방법:
  파티션 키 컬럼이 NULL 허용? → NULL이 어느 파티션에 들어가는가?

④ 파티션 키 선택:
  주요 쿼리가 파티션 키를 포함하는가?
  데이터가 특정 파티션에 집중되지 않는가? (Hot Partition)

⑤ 파티션 키 업데이트:
  파티션 키 컬럼을 UPDATE하는 패턴이 있는가?
  → DELETE + INSERT로 파티션 이동 비용 발생
```

---

## 🔬 내부 동작 원리

### 1. PRIMARY KEY에 파티션 키 포함 이유

```
MySQL의 파티션 테이블 요구사항:
  모든 UNIQUE 인덱스 (PRIMARY KEY 포함)는
  파티션 함수에 사용된 컬럼을 포함해야 함

이유:
  파티션 테이블에서 Row의 위치 = 파티션 번호 + 파티션 내 PK
  PK가 파티션 키를 포함하지 않으면:
    PK로 Row를 찾으려면 모든 파티션을 검색해야 함
    → PK의 O(1) 접근 보장 불가
  
  PK에 파티션 키 포함 시:
    PK 값에서 파티션 번호 계산 가능 → 1개 파티션만 접근
    해당 파티션 내에서 나머지 PK로 Row 위치 확정

실제 제약:
  PRIMARY KEY (id) + PARTITION BY RANGE (created_at)
  → id만으로 어느 파티션에 있는지 모름 → 불허

  PRIMARY KEY (id, created_at) + PARTITION BY RANGE COLUMNS (created_at)
  → id + created_at으로 파티션 결정 가능 → 허용
  → PK 탐색: created_at으로 파티션 결정 + id로 Row 위치 확정
```

### 2. FOREIGN KEY 미지원 이유

```
파티션 테이블에서 FK 동작 시 문제:

시나리오:
  users(id PK, ...)
  logs(id BIGINT, user_id BIGINT, created_at DATETIME)
  logs PARTITION BY RANGE(created_at)
  logs.user_id REFERENCES users(id)

FK 검사 시 필요한 것:
  logs에 user_id = 999를 INSERT할 때
  users 테이블에 id = 999가 존재하는지 확인
  → 문제 없음

  users에서 id = 999를 DELETE할 때
  logs 테이블의 모든 파티션에서 user_id = 999가 있는지 확인
  → 모든 파티션의 FK 검사 필요 (글로벌 인덱스 없음)
  → 각 파티션의 user_id 인덱스 모두 탐색 = N × 파티션 수

  파티션이 많아질수록 FK 검사 비용 폭발
  + 파티션 DROP 시 FK 참조 무결성 유지 어려움
  → MySQL은 이 복잡성을 피하기 위해 FK 자체를 미지원

대안:
  FK 제거 → 애플리케이션 레벨에서 참조 무결성 보장
  서비스 레이어에서 users 존재 확인 후 logs INSERT
  또는 Soft Delete (users를 DELETE하지 않고 status 변경)
```

### 3. Hot Partition 현상

```
Hot Partition 발생 조건:
  파티션 키가 단조증가 패턴일 때

RANGE(created_at) 파티션:
  created_at이 시간 순서대로 증가
  → 현재 시간 = 현재 파티션에만 INSERT
  → 2024년 3월에는 p2024만 INSERT 대상

문제:
  p2024만 Write Heavy → I/O, Lock 집중
  다른 파티션 (p2022, p2023)은 Read만 → 상대적으로 여유

실제 시나리오:
  12개 파티션 → 11개는 거의 유휴 상태
  1개 파티션에 모든 INSERT/UPDATE 집중
  Hot Partition의 디스크 I/O 포화 → 전체 서비스 느려짐

해결:
  수용 가능: 로그 테이블은 항상 최신 파티션에 INSERT → 어쩔 수 없음
  Hot Partition을 위한 파티션 크기 조정 (더 세분화)
  또는 HASH로 분산 (단, 범위 프루닝 포기)

실시간 집계 테이블에서 Hot Partition 회피:
  HASH(user_id) → 다양한 user_id 고르게 분산
  → INSERT가 8개 파티션에 균등 분배 → Hot Partition 없음
  단, 날짜 범위 조회 프루닝 포기
```

### 4. NULL 처리

```sql
-- RANGE 파티션에서 NULL:
-- NULL은 가장 낮은 값으로 취급 → 첫 번째 파티션에 저장

CREATE TABLE null_test (
    id  BIGINT AUTO_INCREMENT,
    val INT,
    PRIMARY KEY (id, val)
)
PARTITION BY RANGE (val) (
    PARTITION p_neg VALUES LESS THAN (0),
    PARTITION p_low VALUES LESS THAN (100),
    PARTITION p_high VALUES LESS THAN MAXVALUE
);

INSERT INTO null_test (val) VALUES (NULL);
-- NULL은 가장 작은 값 취급 → p_neg 파티션에 저장

EXPLAIN SELECT * FROM null_test WHERE val IS NULL;
-- partitions: p_neg (NULL이 p_neg에 있음)

-- LIST 파티션에서 NULL:
-- NULL이 LIST에 포함되지 않으면 INSERT 실패
-- 명시적으로 NULL을 처리하는 파티션 필요

-- HASH/KEY 파티션에서 NULL:
-- NULL은 0으로 취급 → 파티션 0에 저장

-- 실무 권고:
-- 파티션 키 컬럼은 NOT NULL로 선언 (NULL 처리 복잡성 제거)
```

### 5. 파티션 키 UPDATE의 함정

```sql
-- 파티션 키 컬럼 UPDATE:
-- 파티션 변경이 필요할 수 있음 → DELETE + INSERT 발생

-- RANGE(created_at) 파티션
UPDATE logs SET created_at = '2024-02-01'  -- 다른 파티션으로 이동
WHERE id = 100 AND created_at = '2023-12-15';

-- 내부 처리:
-- 1. p2023에서 id=100 Row DELETE
-- 2. p2024에 새 Row INSERT
-- = DELETE + INSERT 비용 (단순 UPDATE보다 비쌈)
-- 또한: 이 UPDATE는 명시적으로 두 파티션에 접근

-- 파티션 키 컬럼은 UPDATE하지 않는 것이 좋음
-- 업데이트가 필요한 컬럼을 파티션 키로 선택하지 말 것
-- 예: status 컬럼은 자주 변경됨 → 파티션 키로 부적합
```

---

## 💻 실전 실험

### 실험 1: PK 제약 오류 재현

```sql
-- PK에 파티션 키 없을 때 오류
CREATE TABLE pk_test (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    created_at DATETIME NOT NULL
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p1 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
-- Error 1503 확인

-- PK에 파티션 키 포함 (올바른 방법)
CREATE TABLE pk_test_ok (
    id         BIGINT AUTO_INCREMENT,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)  -- 파티션 키 포함
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p1 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
-- 성공

DROP TABLE IF EXISTS pk_test, pk_test_ok;
```

### 실험 2: FK 미지원 확인

```sql
CREATE TABLE ref_table (
    id INT PRIMARY KEY,
    name VARCHAR(50)
) ENGINE=InnoDB;

CREATE TABLE fk_part_test (
    id        BIGINT AUTO_INCREMENT,
    ref_id    INT NOT NULL,
    ts        DATE NOT NULL,
    PRIMARY KEY (id, ts),
    FOREIGN KEY (ref_id) REFERENCES ref_table(id)  -- FK 추가 시도
)
PARTITION BY RANGE COLUMNS (ts) (
    PARTITION p1 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
-- Error: Foreign keys are not yet supported in conjunction with partitioning

DROP TABLE IF EXISTS fk_part_test, ref_table;
```

### 실험 3: NULL 파티션 배치 확인

```sql
CREATE TABLE null_part (
    id  BIGINT AUTO_INCREMENT,
    val INT,
    PRIMARY KEY (id, val)
)
PARTITION BY RANGE (val) (
    PARTITION p_neg  VALUES LESS THAN (0),
    PARTITION p_zero VALUES LESS THAN (1),
    PARTITION p_pos  VALUES LESS THAN MAXVALUE
);

-- NULL 삽입 불가 (PK에 NULL 허용 안 됨)
-- val만 별도로 테스트:
CREATE TABLE null_part2 (
    id  BIGINT AUTO_INCREMENT,
    val INT,
    PRIMARY KEY (id)
)
PARTITION BY HASH(COALESCE(val, 0))  -- NULL 처리 명시
PARTITIONS 4;

INSERT INTO null_part2 (val) VALUES (NULL), (1), (2), (3);
EXPLAIN SELECT * FROM null_part2 WHERE val IS NULL;

DROP TABLE IF EXISTS null_part, null_part2;
```

---

## 📊 성능/비용 비교

```
파티션 함정별 영향도:

PK 재설계 비용:
  기존 PK (id) → 새 PK (id, created_at)
  단순 ALTER: 테이블 재구성 필요 (수억 건이면 수 시간)
  FK 참조 테이블 모두 수정 필요
  → 사전 설계 단계에서 결정해야 할 사항

FK 제거 비용:
  FK 제거 → 무결성 보장 코드 추가 필요
  애플리케이션 변경 비용
  데이터 정합성 사고 위험 증가

Hot Partition 영향:
  파티션당 초당 INSERT가 균등 분배되지 않음
  1개 파티션: 모든 INSERT 처리
  → 디스크 I/O 포화 → 전체 응답 시간 증가
  → IOPS 한계 도달 시 장애

파티션 키 UPDATE 비용:
  단순 UPDATE vs DELETE + INSERT
  단순 UPDATE: 하나의 파티션 내 Row 수정
  파티션 이동 UPDATE: 두 파티션 접근 + 인덱스 수정 2번
  대량 파티션 키 UPDATE: 매우 느림
```

---

## ⚖️ 트레이드오프

```
파티셔닝 도입의 현실적 트레이드오프:

얻는 것:
  대용량 범위 조회 성능 개선
  TTL 삭제 극적 개선 (DROP PARTITION)
  파티션 단위 아카이브 가능

잃는 것:
  FOREIGN KEY → 무결성 보장이 애플리케이션 책임
  UNIQUE 인덱스 유연성 감소 (파티션 키 포함 필수)
  PK 설계 제약 → 복합 PK로 변경 필요
  스키마 복잡도 증가
  파티션 키 UPDATE 비용 증가
  파티션 키로 NULL 처리 주의 필요

도입 판단:
  위의 '잃는 것'을 감당할 수 있는 경우에만 도입
  특히 FK는 파티션 테이블 도입의 가장 큰 장벽
  FK를 제거하기 어려운 레거시 시스템에는 파티셔닝 도입 어려움

대안 검토:
  파티셔닝 대신 테이블 수동 분리 (manual sharding)
  아카이브 테이블 분리 (TTL 목적만이라면)
  TiDB, CockroachDB 등 분산 DB 검토 (글로벌 인덱스 지원)
```

---

## 📌 핵심 정리

```
파티션 함정 핵심:

PRIMARY KEY 제약:
  파티션 함수 컬럼을 PK에 반드시 포함
  기존 AUTO_INCREMENT PK → (id, 파티션 키) 복합 PK로 변경
  → 다른 테이블의 FK 참조 방식 변경 필요

FOREIGN KEY 미지원:
  파티션 테이블은 FK 불가
  → 애플리케이션에서 참조 무결성 보장 필요
  → 파티셔닝 도입 전 FK 제거 계획 필수

Hot Partition:
  파티션 키가 단조증가(시간, Auto Increment) → 최신 파티션 집중
  로그 테이블은 어쩔 수 없지만 인지하고 있어야 함
  → 파티션 세분화 또는 HASH 병행 고려

파티션 키 선택:
  자주 UPDATE되는 컬럼은 파티션 키 부적합
  NULL 허용 컬럼은 복잡성 추가 → NOT NULL 권장

도입 전 확인:
  FK 제거 가능한가?
  PK 재설계 비용은?
  Hot Partition 허용 가능한가?
  파티션 키 UPDATE 패턴이 있는가?
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 테이블에 RANGE(created_at) 파티셔닝을 적용하려 한다. 발생하는 문제와 해결 방법을 설명하라.

```sql
CREATE TABLE orders (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    UNIQUE KEY uk_user_product_date (user_id, product_id, created_at)
) ENGINE=InnoDB;
```

<details>
<summary>해설 보기</summary>

**문제 3가지**:

1. **FOREIGN KEY 오류**: `FOREIGN KEY (user_id) REFERENCES users(id)` → 파티션 테이블에서 FK 불가
2. **PRIMARY KEY 오류**: `id` 단독 PK에 파티션 키(`created_at`) 미포함 → Error 1503
3. **UNIQUE KEY 문제**: `uk_user_product_date`가 파티션 키(`created_at`)를 포함하나 확인 필요 (포함하고 있으므로 이 UNIQUE는 가능)

**해결 방법**:

```sql
-- FK 제거 + PK 재설계 + 파티션 적용
CREATE TABLE orders_new (
    id         BIGINT AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at),          -- 파티션 키 포함
    -- FK 제거 (user_id, product_id 무결성은 애플리케이션에서 보장)
    INDEX idx_user (user_id),
    INDEX idx_product (product_id),
    UNIQUE KEY uk_user_product_date (user_id, product_id, created_at)  -- created_at 포함 OK
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);
```

**FK 제거 후 무결성 보장**:
- 서비스 레이어에서 `users.id`와 `products.id` 존재 확인 후 INSERT
- 삭제 시 연관 orders 처리 로직 명시 (Soft Delete 또는 CASCADE 수동 처리)

</details>

---

<div align="center">

**[⬅️ 파티션과 인덱스](./04-partition-and-index.md)** | **[홈으로 🏠](../README.md)** | **[다음: 파티션 운영 ➡️](./06-partition-operations.md)**

</div>
