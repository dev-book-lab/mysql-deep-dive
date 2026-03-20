# 파티션과 인덱스 — 로컬 인덱스 vs 글로벌 인덱스

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- MySQL이 글로벌 인덱스를 지원하지 않는 이유는?
- 파티션별 로컬 인덱스는 어떻게 구성되는가?
- 파티션 키가 인덱스 컬럼에 포함되어야 하는 이유는?
- 비파티션 키 컬럼으로 조회 시 발생하는 Full Partition Scan 문제는?
- Covering Index가 파티션 환경에서 특히 중요한 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 인덱스를 추가했는데 파티션 테이블에서 ALL Partition Scan이 나온다

```
실제 이슈:
  파티션 테이블 (RANGE by created_at)
  user_id 인덱스 추가
  
  EXPLAIN SELECT * FROM logs WHERE user_id = 999;
  → partitions: p2022,p2023,p2024,pmax (ALL Partition)
  → type: ref (인덱스는 사용)
  → 12개 파티션 모두의 인덱스를 탐색
  
  "인덱스 있는데 왜 12개 파티션 다 보나요?"
  
  이유:
    MySQL의 인덱스는 파티션별 로컬 인덱스
    user_id 인덱스가 각 파티션에 독립적으로 존재
    user_id = 999가 어느 파티션에 있는지 알 수 없음
    → 12개 파티션의 로컬 인덱스 각각 탐색
    
  vs 글로벌 인덱스(가상):
    단일 user_id 인덱스 → user_id=999 위치 즉시 파악
    → 특정 파티션만 접근
    → 하지만 MySQL은 글로벌 인덱스 미지원
```

---

## 😱 흔한 실수 (Before)

### 1. 파티션 키 이외 컬럼으로만 자주 조회

```sql
-- RANGE(created_at) 파티션
-- 실무에서 가장 많은 조회 패턴이 user_id 기반
SELECT * FROM logs WHERE user_id = 999;

-- 문제:
-- user_id 인덱스 있음 → 각 파티션에서 ref 탐색
-- but: 12개 파티션 모두 접근 → ref × 12번
-- 파티션 없는 테이블의 단일 인덱스 탐색보다 느릴 수 있음!
```

### 2. 파티션 키를 인덱스에서 빠뜨림

```sql
-- user_id 인덱스만 (created_at 없음)
CREATE INDEX idx_user ON logs (user_id);

-- WHERE user_id = 999 AND created_at >= '2024-01-01'
-- created_at이 프루닝에 사용되지만
-- 인덱스 (user_id)로 created_at 필터링 추가 비용
-- → index condition pushdown으로 처리되지만 비효율

-- 더 나은 인덱스: (user_id, created_at)
CREATE INDEX idx_user_created ON logs (user_id, created_at);
-- user_id + created_at 범위를 인덱스에서 직접 처리
```

---

## ✨ 올바른 접근 (After)

```
파티션 테이블의 인덱스 구조 이해:

MySQL 파티션 인덱스 = 로컬 인덱스 (Local Index):
  각 파티션이 독립적인 B-Tree 인덱스를 가짐
  인덱스 탐색은 파티션 내부에서만 유효
  파티션 간 인덱스 공유 없음

글로벌 인덱스 (Global Index) = MySQL 미지원:
  Oracle, PostgreSQL 등은 지원
  전체 테이블에 걸친 단일 인덱스 구조
  파티션 키 이외 컬럼으로도 단일 인덱스 탐색 가능
  MySQL에서는 이 기능 없음

결론:
  파티션 키로 프루닝 가능한 조건 포함 필수
  파티션 키 이외 컬럼 조회 시 = 모든 파티션의 로컬 인덱스 탐색
  → 파티션 수 × 인덱스 탐색 비용
  → 파티션이 많을수록 비용 증가
```

---

## 🔬 내부 동작 원리

### 1. 파티션별 로컬 인덱스 구조

```
파티션 테이블 물리 구조:
  logs.ibd 없음, 대신:
  logs#P#p2022.ibd  → p2022 파티션 데이터 + 인덱스
  logs#P#p2023.ibd  → p2023 파티션 데이터 + 인덱스
  logs#P#p2024.ibd  → p2024 파티션 데이터 + 인덱스
  logs#P#pmax.ibd   → pmax 파티션 데이터 + 인덱스

각 .ibd 파일 내부:
  Clustered Index (PK): 해당 파티션의 Row들
  Secondary Index (user_id): 해당 파티션의 user_id → PK 매핑

WHERE user_id = 999 실행 시:
  p2022.ibd의 user_id 인덱스 탐색 → user_id=999 찾기
  p2023.ibd의 user_id 인덱스 탐색 → user_id=999 찾기
  p2024.ibd의 user_id 인덱스 탐색 → user_id=999 찾기
  pmax.ibd의 user_id 인덱스 탐색 → user_id=999 찾기
  → 4번의 독립적 인덱스 탐색
  → 파티션 수가 많을수록 비용 × N
```

### 2. 파티션 키 포함 복합 인덱스가 효과적인 이유

```sql
-- 시나리오: 파티션 RANGE(created_at)
-- 주요 조회: WHERE user_id = ? AND created_at >= ?

-- 인덱스 1: (user_id)만
-- WHERE user_id = 999 AND created_at >= '2024-01-01':
--   프루닝: created_at 조건 → p2024, pmax (2개 파티션)
--   인덱스: 각 파티션에서 user_id=999 탐색
--   but: created_at 필터는 인덱스 후 추가 필터
--   → 2 × (user_id=999 인덱스 탐색 + created_at 필터)

-- 인덱스 2: (user_id, created_at)
-- WHERE user_id = 999 AND created_at >= '2024-01-01':
--   프루닝: created_at 조건 → p2024, pmax (2개 파티션)
--   인덱스: 각 파티션에서 (user_id=999, created_at>=2024) 복합 탐색
--   → 인덱스에서 직접 정확한 Row 범위 확정
--   → 2 × (복합 인덱스 탐색, 더 적은 결과)

-- 핵심:
-- 파티션 키(created_at)를 인덱스에 포함하면
-- 프루닝 범위 내에서 인덱스 효율이 극대화됨
```

### 3. Covering Index가 파티션에서 더 중요한 이유

```sql
-- 파티션 테이블에서 Secondary Index → PK 조회 패턴:

-- Secondary Index (user_id)에서 PK 찾기
-- → Clustered Index(PK)로 이동해 나머지 컬럼 조회
-- = 파티션 내에서도 두 번 B-Tree 탐색 (이중 조회)

-- Covering Index:
-- SELECT 컬럼이 모두 인덱스에 포함되면 Clustered Index 불필요
-- → 인덱스만으로 결과 반환

-- 파티션 환경에서 Covering Index 효과:
CREATE INDEX idx_user_cover ON logs (user_id, created_at, action);

SELECT user_id, created_at, action FROM logs WHERE user_id = 999;
-- 파티션별 로컬 인덱스만 탐색 (Clustered Index 불필요)
-- EXPLAIN Extra: Using index

-- vs 전체 컬럼 SELECT:
SELECT * FROM logs WHERE user_id = 999;
-- 파티션별: Secondary Index(user_id) → Clustered Index(PK) 이동
-- = 2× B-Tree 탐색 × 파티션 수

-- 결론:
-- 파티션 테이블에서 Covering Index = 2차 탐색 제거 + 파티션 수 이점
-- 일반 테이블 Covering Index보다 효과 더 큼
```

### 4. 파티션 키와 UNIQUE 인덱스 관계

```sql
-- MySQL 파티션 테이블의 UNIQUE 인덱스 제약:
-- UNIQUE 인덱스는 반드시 파티션 키 컬럼을 포함해야 함

-- 불가: email만으로 UNIQUE (파티션 키 포함 안 됨)
CREATE TABLE users_p (
    id         BIGINT AUTO_INCREMENT,
    email      VARCHAR(100) NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at),
    UNIQUE KEY uk_email (email)  -- ❌ 오류!
    -- Error 1503: A UNIQUE INDEX must include all columns in the partition function
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);

-- 가능: email + created_at으로 UNIQUE
CREATE TABLE users_p (
    id         BIGINT AUTO_INCREMENT,
    email      VARCHAR(100) NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at),
    UNIQUE KEY uk_email_date (email, created_at)  -- ✅ 파티션 키 포함
)
PARTITION BY RANGE COLUMNS (created_at) (...);

-- 문제:
-- (email, created_at)이 UNIQUE → 같은 email이 다른 날짜에 중복 가능!
-- 진정한 email 유일성 보장 불가
-- → 파티션 테이블에서 email UNIQUE 제약은 실질적으로 불가
-- → 애플리케이션 레벨에서 중복 체크 필요
```

### 5. 비파티션 키 컬럼 조회 최적화 전략

```sql
-- 전략 1: 쿼리에 파티션 키 조건 추가 (가장 효과적)
-- Before:
SELECT * FROM logs WHERE user_id = 999;
-- After: 파티션 키 범위 추가
SELECT * FROM logs WHERE user_id = 999 AND created_at >= '2024-01-01';
-- 프루닝으로 최근 파티션만 접근

-- 전략 2: 복합 인덱스로 파티션 내 탐색 최소화
CREATE INDEX idx_user_created ON logs (user_id, created_at DESC);
SELECT * FROM logs WHERE user_id = 999 AND created_at >= '2024-01-01';
-- 파티션 프루닝 + 인덱스 탐색 조합

-- 전략 3: 파티션 수 최소화
-- 파티션이 4개면 4번 탐색, 12개면 12번 탐색
-- user_id 조회가 많으면 파티션 수를 줄이는 것도 전략

-- 전략 4: 별도 요약 테이블
-- user_id 기반 빠른 조회를 위한 별도 비파티션 테이블
CREATE TABLE user_log_summary (
    user_id    BIGINT PRIMARY KEY,
    last_log   DATETIME,
    total_cnt  BIGINT,
    UPDATED AT DATETIME
) ENGINE=InnoDB;
-- 집계/최근 조회는 summary 테이블에서
```

---

## 💻 실전 실험

### 실험 1: 로컬 인덱스 다중 탐색 확인

```sql
CREATE TABLE local_idx_test (
    id         BIGINT AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    action     VARCHAR(50),
    PRIMARY KEY (id, created_at),
    INDEX idx_user (user_id)
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p2022 VALUES LESS THAN ('2023-01-01'),
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);

-- 파티션 키 없는 조회
EXPLAIN SELECT * FROM local_idx_test WHERE user_id = 999;
-- partitions: p2022,p2023,p2024,pmax (ALL Partition)
-- type: ref (인덱스 사용)
-- key: idx_user
-- → 4개 파티션의 idx_user를 각각 탐색

-- 파티션 키 추가한 조회
EXPLAIN SELECT * FROM local_idx_test WHERE user_id = 999 AND created_at >= '2024-01-01';
-- partitions: p2024,pmax (프루닝!)
-- type: ref
-- → 2개 파티션의 idx_user만 탐색

-- 비교: EXPLAIN ANALYZE로 실제 loops 차인
EXPLAIN ANALYZE SELECT * FROM local_idx_test WHERE user_id = 999\G
EXPLAIN ANALYZE SELECT * FROM local_idx_test WHERE user_id = 999 AND created_at >= '2024-01-01'\G
```

### 실험 2: Covering Index 효과

```sql
-- Covering Index 없음
EXPLAIN SELECT user_id, created_at, action FROM local_idx_test WHERE user_id = 999;
-- Extra: (Covering Index 없음, Clustered Index 접근 발생)

-- Covering Index 추가
CREATE INDEX idx_user_cover ON local_idx_test (user_id, created_at, action);
EXPLAIN SELECT user_id, created_at, action FROM local_idx_test WHERE user_id = 999;
-- Extra: Using index (Covering Index 적용)
-- Clustered Index 접근 없음 → 빠름

DROP TABLE local_idx_test;
```

### 실험 3: UNIQUE 인덱스 파티션 키 포함 여부

```sql
-- 파티션 키 없는 UNIQUE → 오류
CREATE TABLE uk_test (
    id    BIGINT AUTO_INCREMENT,
    email VARCHAR(100),
    ts    DATE NOT NULL,
    PRIMARY KEY (id, ts),
    UNIQUE KEY uk_email (email)  -- 파티션 키 없음
)
PARTITION BY RANGE COLUMNS (ts) (
    PARTITION p1 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
-- Error 1503 확인

-- 파티션 키 포함 UNIQUE → 성공
CREATE TABLE uk_test2 (
    id    BIGINT AUTO_INCREMENT,
    email VARCHAR(100),
    ts    DATE NOT NULL,
    PRIMARY KEY (id, ts),
    UNIQUE KEY uk_email_ts (email, ts)  -- 파티션 키 포함
)
PARTITION BY RANGE COLUMNS (ts) (
    PARTITION p1 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
-- 성공: 하지만 (email, ts)가 UNIQUE → 동일 email 다른 날짜 허용!

DROP TABLE IF EXISTS uk_test, uk_test2;
```

---

## 📊 성능/비용 비교

```
파티션 수에 따른 인덱스 탐색 비용:

파티션 없는 테이블 (인덱스만):
  WHERE user_id = 999:
    단일 Secondary Index 탐색 → O(log n)
    Clustered Index(PK) 이동 → O(log n)
    총: 2 × B-Tree 탐색

파티션 있는 테이블 (4개 파티션):
  WHERE user_id = 999 (프루닝 없음):
    4개 파티션 각각 Secondary Index 탐색
    4개 파티션 각각 Clustered Index 이동
    총: 4 × 2 × B-Tree 탐색 = 8 × (파티션 없는 것보다 4배)

파티션 있는 테이블 (4개 파티션, 프루닝 1개):
  WHERE user_id = 999 AND created_at >= '2024-10-01':
    1개 파티션 탐색
    총: 1 × 2 × B-Tree 탐색 = 2 × (파티션 없는 것과 동일)

결론:
  프루닝 없이 파티션만 있으면 = 파티션 수배 비용 증가
  프루닝 + 파티션 = 단일 파티션 탐색 = 파티션 없는 것과 유사
  프루닝 + Covering Index = 최적 (Clustered Index 이동 없음)
```

---

## ⚖️ 트레이드오프

```
파티션 + 인덱스 설계 트레이드오프:

파티션 키로만 인덱스 (파티션 내 Sequential):
  ✅ 파티션 프루닝 + 인덱스 탐색 최적
  ❌ 비파티션 키 컬럼 조회 시 ALL Partition Scan

파티션 키 이외 컬럼 인덱스 추가:
  ✅ 비파티션 키 컬럼 조회 속도 개선 (파티션 수 × ref)
  ❌ 파티션 수가 많으면 × 횟수 비용 증가
  ❌ 인덱스 유지 비용 파티션 수배 증가

파티션 수 선택:
  파티션 적을수록: 로컬 인덱스 탐색 × 횟수 감소
  파티션 많을수록: 프루닝 효과 증가 but 인덱스 탐색 비용 증가
  → 주요 조회 패턴에 따라 균형점 찾기

Covering Index:
  ✅ Clustered Index 이동 없음 → 파티션 탐색 횟수 절반
  ✅ 파티션 환경에서 특히 효과적
  ❌ 인덱스 크기 증가 → 파티션 수 × 인덱스 크기

비파티션 키 유일성 제약:
  파티션 테이블에서 파티션 키 없는 UNIQUE 불가
  → 애플리케이션 레벨에서 중복 체크 필요
  → 또는 BEFORE INSERT 트리거 (성능 주의)
```

---

## 📌 핵심 정리

```
파티션과 인덱스 핵심:

MySQL 파티션 인덱스 = 로컬 인덱스:
  각 파티션 독립 B-Tree 인덱스
  파티션 간 인덱스 공유 없음
  글로벌 인덱스 미지원

비파티션 키 컬럼 조회 = 파티션 수 × 인덱스 탐색:
  user_id 조회, RANGE(created_at) 파티션
  → 모든 파티션의 user_id 인덱스 각각 탐색
  → 파티션 10개면 10 × 인덱스 탐색

최적화 전략:
  ① 쿼리에 파티션 키 조건 항상 포함 → 프루닝
  ② 복합 인덱스에 파티션 키 포함 → 파티션 내 최적 탐색
  ③ Covering Index → Clustered Index 이동 제거

UNIQUE 인덱스 제약:
  파티션 키 컬럼이 반드시 UNIQUE에 포함
  → 진정한 컬럼 유일성 보장 어려움
  → 애플리케이션 레벨 중복 체크 필요

설계 원칙:
  파티션 키 ≠ 주요 조회 기준이면 파티셔닝 이점 감소
  주요 조회가 파티션 키 기반인 경우에 파티셔닝 효과적
```

---

## 🤔 생각해볼 문제

**Q1.** RANGE(created_at) 파티션 테이블에서 `WHERE user_id = ?` 조회가 느리다. 파티션 구조와 스키마를 변경하지 않고 쿼리 수준에서 개선하는 방법을 3가지 제시하라.

<details>
<summary>해설 보기</summary>

**방법 1: 쿼리에 파티션 키 조건 추가**
```sql
-- Before
SELECT * FROM logs WHERE user_id = 999;

-- After: 최근 N개월 범위 제한
SELECT * FROM logs WHERE user_id = 999 AND created_at >= '2024-01-01';
-- 프루닝으로 2024 이후 파티션만 탐색
```

실무에서 "특정 사용자의 최근 활동"처럼 날짜 범위가 자연스럽게 포함되는 경우가 많습니다.

**방법 2: LIMIT과 함께 파티션 순서 역이용**
```sql
SELECT * FROM logs WHERE user_id = 999
ORDER BY created_at DESC LIMIT 20;
-- 최근 데이터 조회 시 최신 파티션부터 탐색
-- 원하는 건수 채우면 이전 파티션 탐색 불필요 (동적 최적화)
```

**방법 3: 애플리케이션에서 파티션 키 범위 강제 부여**
```java
// 서비스에서 사용자 최근 활동 조회 시
LocalDateTime cutoff = LocalDateTime.now().minusMonths(6); // 6개월 이내로 제한
logRepository.findByUserIdAndCreatedAtAfter(userId, cutoff);
```

사용자에게 의미 있는 범위(최근 6개월, 최근 1년 등)를 도메인 기준으로 정하면 쿼리 성능과 사용자 경험을 동시에 개선할 수 있습니다.

</details>

---

**Q2.** 파티션 테이블에서 `email` 컬럼의 중복을 방지해야 한다. MySQL 파티션의 UNIQUE 제약 한계를 고려한 현실적인 방법을 제시하라.

<details>
<summary>해설 보기</summary>

**상황**: RANGE(created_at) 파티션에서 `email` UNIQUE 불가 (파티션 키 미포함 UNIQUE 제약 오류).

**방법 1: 별도 조회 테이블 (Email Registry)**
```sql
-- 파티션 없는 별도 테이블에 email 관리
CREATE TABLE user_email_registry (
    email   VARCHAR(100) PRIMARY KEY,
    user_id BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    INDEX idx_user (user_id)
) ENGINE=InnoDB;  -- 파티션 없음 → UNIQUE 보장

-- INSERT 시 두 단계:
-- 1. user_email_registry에 email 삽입 (중복 시 오류)
-- 2. 성공하면 파티션 테이블에 INSERT
-- → 트랜잭션으로 묶어 원자성 보장
```

**방법 2: 애플리케이션 레벨 중복 체크 + 낙관적 락**
```java
@Transactional
public void createUser(String email, ...) {
    // 중복 체크 (SELECT)
    if (userRepo.existsByEmail(email)) {
        throw new DuplicateEmailException();
    }
    // INSERT
    userRepo.save(new User(email, ...));
    // 동시성 문제: 두 트랜잭션이 동시에 중복 체크 통과 가능
    // → 낙관적 락 또는 분산 락으로 보완 필요
}
```

**방법 3: DB 레벨 UNIQUE를 포기하고 배치 중복 정리**
파티션의 이점(대용량 TTL, 성능)이 email UNIQUE 보장보다 중요한 경우, 주기적 배치로 중복 email 감지 및 정리. 실시간 UNIQUE는 포기하지만 서비스 수준에서 허용 가능한 경우에 한함.

실무에서는 **방법 1 (별도 레지스트리 테이블)**이 가장 현실적입니다.

</details>

---

<div align="center">

**[⬅️ 파티션 프루닝](./03-partition-pruning.md)** | **[홈으로 🏠](../README.md)** | **[다음: 파티션의 함정 ➡️](./05-partition-pitfalls.md)**

</div>
