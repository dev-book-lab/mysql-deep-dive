# 파티셔닝이 필요한 상황과 오해 — 인덱스와 파티셔닝은 다르다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 파티셔닝이 인덱스를 대체하지 않는 이유는?
- 어떤 상황에서 파티셔닝이 진짜 효과를 내는가?
- 파티셔닝이 오히려 성능을 떨어뜨리는 시나리오는?
- 데이터 TTL 삭제에 파티셔닝이 왜 더 효과적인가?
- 파티셔닝을 도입하기 전에 확인해야 할 전제 조건은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 파티셔닝을 걸었는데 왜 안 빨라지지?

```
실제 운영 이슈:
  "로그 테이블이 5억 건이 됐습니다. 파티셔닝 걸면 빨라지나요?"
  → RANGE 파티션(월별) 적용
  → 쿼리 시간 변화 없음

  원인:
    쿼리 조건: WHERE user_id = 12345 AND created_at >= '2024-01-01'
    파티션 키: created_at
    → 파티션 프루닝으로 2024년 파티션만 접근 (좋음)
    → 하지만 2024년 파티션 내 user_id 인덱스가 파티션별로 분산
    → 여전히 해당 파티션의 인덱스 탐색 = 인덱스 있는 것과 비슷한 비용

  진짜 효과가 있는 케이스:
    WHERE created_at >= '2024-01-01'  (user_id 조건 없음)
    → 파티션 프루닝으로 2024년 파티션만 스캔
    → 2024년 데이터만 B-Tree 탐색 → 소규모 탐색 범위

핵심:
  파티셔닝 ≠ 인덱스 대체
  파티셔닝 = "탐색 대상 파티션 수를 줄이는 것"
  파티션 내에서는 여전히 인덱스가 필요
```

---

## 😱 흔한 실수 (Before)

### 1. 파티셔닝으로 인덱스를 대체하려 함

```sql
-- 잘못된 기대:
-- "파티셔닝 걸면 인덱스 없어도 빠르다"

CREATE TABLE logs (
    id         BIGINT AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    action     VARCHAR(50),
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)  -- 파티션 키 포함 필수
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- 파티션 걸었지만 user_id 인덱스 없음
SELECT * FROM logs WHERE user_id = 999 AND created_at >= '2024-01-01';
-- 2024 파티션만 접근하지만
-- user_id 조건에 인덱스 없음 → 2024 파티션 전체 Full Scan
-- 여전히 느림!
```

### 2. 소규모 테이블에 파티셔닝 적용

```sql
-- Before: 100만 건짜리 테이블에 파티셔닝
-- "데이터가 나중에 많아질 것 같아서"
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- 문제:
-- 100만 건에서 인덱스가 충분히 빠름 (0.001초)
-- 파티셔닝은 오버헤드만 추가 (파티션 결정 비용, FOREIGN KEY 불가)
-- 외래키 제약 불가로 데이터 무결성 손실
```

### 3. 파티션 키가 아닌 컬럼으로만 조회

```sql
-- 파티션: RANGE(created_at)
-- 하지만 실제 조회 패턴은 user_id로만 조회
SELECT * FROM logs WHERE user_id = 999;
-- 파티션 프루닝 불가 (created_at 조건 없음)
-- 모든 파티션 Full Scan → 파티셔닝 없는 것보다 느릴 수 있음!
```

---

## ✨ 올바른 접근 (After)

```
파티셔닝이 효과적인 3가지 상황:

상황 1: 수억 건 이상의 대용량 테이블
  인덱스 B-Tree 높이가 너무 높아짐
  → 디스크 I/O 비용 증가
  파티션으로 각 파티션이 독립 B-Tree → 높이 감소

상황 2: 파티션 키로 조회 비율이 높을 때
  WHERE created_at BETWEEN '2024-01-01' AND '2024-03-31'
  → 1개 파티션만 접근 (나머지 무시)
  = 데이터를 4분의 1만 스캔

상황 3: 주기적 데이터 삭제 (TTL)
  DELETE FROM logs WHERE created_at < '2022-01-01';
  → 수백만 건 삭제, 오래 걸림, 언두 로그 폭발

  ALTER TABLE logs DROP PARTITION p2022;
  → 파티션 파일 자체를 삭제 (밀리초 단위)
  → 락 범위 최소화, 언두 로그 없음
```

---

## 🔬 내부 동작 원리

### 1. 파티션별 독립 B-Tree 구조

```
일반 테이블 (5억 건):
  단일 B-Tree
  높이: log_B(500,000,000) ≈ 6~8 레벨
  루트→리프: 6~8번의 디스크 I/O
  범위 스캔: 광범위한 리프 페이지 탐색

파티션 테이블 (월별 12 파티션, 5억 건):
  각 파티션: ~4,200만 건
  파티션별 독립 B-Tree
  파티션 내 높이: log_B(42,000,000) ≈ 5 레벨
  → 파티션 1개 접근 시 높이 감소
  → 같은 데이터인데 탐색 레벨 감소
  → 범위 스캔 시 해당 파티션 리프만 스캔

중요:
  파티션은 테이블을 물리적으로 분리 (파일 수준)
  각 파티션은 독립적인 .ibd 파일 (innodb_file_per_table 기준)
  → 특정 파티션 삭제 = 파일 삭제 (거의 즉각)
  → 파티션 프루닝 = 다른 파일은 아예 열지 않음
```

### 2. 파티셔닝 vs 인덱스의 역할 구분

```
인덱스의 역할:
  테이블 내에서 특정 Row를 빠르게 찾는 것
  B-Tree 구조로 O(log n) 탐색
  WHERE user_id = 1 → idx_user_id → 해당 Row 위치 → 데이터 접근

파티셔닝의 역할:
  탐색 대상 파티션 수 줄이기 (파티션 프루닝)
  파티션 내 탐색은 인덱스가 담당
  WHERE created_at >= '2024-01-01' → p2024 파티션만 열기
  → 그 안에서 인덱스 또는 Full Scan

둘의 협력:
  파티셔닝 + 인덱스 = 최적
    파티션 프루닝으로 대상 파티션 1개
    인덱스로 파티션 내 특정 Row 빠르게 찾기

  파티셔닝만 + 인덱스 없음:
    파티션 프루닝은 되지만 파티션 내 Full Scan
    → 파티션이 크면 여전히 느림

  인덱스만:
    전체 B-Tree 탐색 (파티션 프루닝 없음)
    수억 건 범위 스캔 시 많은 리프 페이지 접근
```

### 3. 파티셔닝이 느려지는 시나리오

```
시나리오 1: 파티션 키 없는 조회 = ALL Partition Scan
  파티션: RANGE(created_at, 12 파티션)
  조회: WHERE user_id = 999 (created_at 조건 없음)
  → 12개 파티션 모두 접근
  → 파티션 없는 테이블보다 오히려 느릴 수 있음
     (파티션 결정 오버헤드 × 12)

시나리오 2: 파티션 수가 너무 많을 때
  1일 단위 파티션 × 3년 = 1,095개 파티션
  파티션 메타데이터 조회 비용 × 1,095
  → 파티션 결정 비용 자체가 큰 오버헤드

시나리오 3: 파티션 키에 함수 적용
  파티션: RANGE(YEAR(created_at))
  조회: WHERE created_at = '2024-01-15' → 프루닝 OK
  조회: WHERE DATE_FORMAT(created_at, '%Y') = '2024' → 프루닝 안 됨!
  → 함수 적용 시 프루닝 불가 → ALL Partition Scan

시나리오 4: 소규모 테이블
  파티션 오버헤드 (메타데이터, 파일 관리) > 파티션 이득
  수백만 건 이하 → 인덱스만으로 충분
```

### 4. TTL 삭제에서 파티셔닝이 유리한 이유

```
DELETE 방식:
  DELETE FROM logs WHERE created_at < '2022-01-01';
  
  내부 처리:
  1. created_at < '2022-01-01'인 Row 하나씩 찾아 삭제
  2. 각 Row에 대해 언두 로그 기록 (MVCC 롤백 대비)
  3. 각 Row에 대해 X Lock 획득
  4. 인덱스 엔트리도 각각 제거
  5. COMMIT 후 공간 해제 (즉각 아님)
  
  비용:
  - 수천만 건이면 수십 분 ~ 수 시간
  - 언두 로그 폭발 → 언두 테이블스페이스 증가
  - X Lock으로 인한 다른 DML 차단

DROP PARTITION 방식:
  ALTER TABLE logs DROP PARTITION p2021;
  
  내부 처리:
  1. p2021에 해당하는 .ibd 파일(또는 파티션 세그먼트) 삭제
  2. 메타데이터 업데이트
  
  비용:
  - 수천만 건이어도 밀리초 ~ 수 초 (파일 삭제)
  - 언두 로그 없음
  - 다른 파티션에 대한 DML 영향 없음
  
제약:
  DROP PARTITION은 해당 파티션의 모든 데이터 삭제
  일부만 삭제 불가
  → 삭제 주기를 파티션 단위(월/분기)로 맞춰야 함
```

---

## 💻 실전 실험

### 실험 1: 파티셔닝 vs 인덱스 단독 성능 비교

```sql
-- 파티션 없는 테이블 (인덱스만)
CREATE TABLE logs_normal (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    action     VARCHAR(50),
    created_at DATETIME NOT NULL,
    INDEX idx_user (user_id),
    INDEX idx_created (created_at)
) ENGINE=InnoDB;

-- 파티션 있는 테이블
CREATE TABLE logs_partitioned (
    id         BIGINT AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    action     VARCHAR(50),
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at),
    INDEX idx_user (user_id)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);

-- 1000만 건 데이터 삽입 (실제 실험 환경에서)

-- 케이스 A: 파티션 키로 범위 조회
-- Normal
EXPLAIN ANALYZE SELECT COUNT(*) FROM logs_normal
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- Partitioned (프루닝 동작)
EXPLAIN ANALYZE SELECT COUNT(*) FROM logs_partitioned
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- 케이스 B: 파티션 키 없이 조회 (차이 확인)
EXPLAIN ANALYZE SELECT COUNT(*) FROM logs_normal WHERE user_id = 999;
EXPLAIN ANALYZE SELECT COUNT(*) FROM logs_partitioned WHERE user_id = 999;
-- 파티션이 없는 게 빠를 수도 있음 (ALL partition scan)
```

### 실험 2: DROP PARTITION vs DELETE 속도 비교

```sql
-- 1,000만 건 파티션 테이블 기준

-- DELETE 방식
SET @t = SYSDATE(6);
DELETE FROM logs_partitioned WHERE created_at < '2023-01-01';
SELECT CONCAT('DELETE time: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000000, 's');

-- DROP PARTITION 방식 (p2022 파티션)
SET @t = SYSDATE(6);
ALTER TABLE logs_partitioned DROP PARTITION p2022;
SELECT CONCAT('DROP PARTITION time: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000000, 's');

-- 수치 비교: DELETE는 수십 초~분, DROP PARTITION은 밀리초 단위
```

### 실험 3: 파티션 프루닝 동작 확인

```sql
-- EXPLAIN PARTITIONS 또는 EXPLAIN (MySQL 8.0에서 통합)
EXPLAIN SELECT * FROM logs_partitioned
WHERE created_at >= '2024-01-01';
-- partitions 컬럼: p2024,pmax → 프루닝 동작 확인

EXPLAIN SELECT * FROM logs_partitioned
WHERE user_id = 999;
-- partitions 컬럼: p2022,p2023,p2024,pmax → ALL Partition Scan!

-- EXPLAIN FORMAT=JSON 에서 더 상세 확인
EXPLAIN FORMAT=JSON SELECT * FROM logs_partitioned WHERE created_at >= '2024-01-01'\G
```

---

## 📊 성능/비용 비교

```
데이터 규모별 파티셔닝 효과:

100만 건 미만:
  인덱스로 충분 (0.001~0.01초)
  파티셔닝 도입 시: 외래키 불가, 복잡성 증가
  권장: 인덱스만 사용

1,000만 건 ~ 1억 건:
  인덱스로 여전히 빠름 (0.01~0.1초)
  파티셔닝 고려 조건: TTL 삭제가 잦은 경우
  권장: TTL이 있으면 파티셔닝, 없으면 인덱스 최적화

1억 건 이상:
  B-Tree 높이 증가 → 탐색 비용 증가
  파티션 프루닝 효과 두드러짐
  TTL 삭제 비용 급증 → DROP PARTITION 필수
  권장: 파티셔닝 + 파티션 내 인덱스

5억 건 이상:
  단일 테이블 인덱스만으로는 한계
  파티셔닝 + 인덱스 + Covering Index 조합 필요
  또는 아카이브 전략 (오래된 데이터 별도 테이블) 고려
```

---

## ⚖️ 트레이드오프

```
파티셔닝 도입 비용 vs 효과:

도입 이점:
  ✅ 대용량 범위 조회 성능 개선 (프루닝 동작 시)
  ✅ 주기적 데이터 삭제 극적 개선 (DROP PARTITION)
  ✅ 파티션별 독립 B-Tree → 높이 감소
  ✅ 파티션 단위 백업/복원 가능

도입 비용:
  ❌ FOREIGN KEY 지원 안 됨 → 데이터 무결성 애플리케이션 담당
  ❌ PRIMARY KEY에 파티션 키 포함 필수 → PK 설계 변경 필요
  ❌ 파티션 키 이외 컬럼 조회 시 ALL Partition Scan → 오히려 느림
  ❌ 파티션 수가 많으면 메타데이터 비용 증가
  ❌ ALTER TABLE 복잡도 증가
  ❌ 기존 테이블에 파티셔닝 추가 시 잠금 비용 (06 문서 참고)

도입 판단 기준:
  ① 테이블 크기 1억 건 이상
  ② 파티션 키로 조회하는 패턴이 주된 쿼리
  ③ 주기적 TTL 삭제가 필요
  위 3가지 중 2개 이상이면 파티셔닝 고려
```

---

## 📌 핵심 정리

```
파티셔닝 핵심:

파티셔닝 ≠ 인덱스 대체:
  파티셔닝 = 탐색 파티션 수 줄이기 (수평 분할)
  인덱스 = 파티션 내에서 Row 찾기
  두 가지가 함께 동작해야 최적

파티셔닝이 효과적인 경우:
  ① 1억 건 이상 대용량 테이블
  ② 파티션 키로 범위 조회가 주 패턴
  ③ 주기적 TTL 삭제 필요

파티셔닝이 오히려 불리한 경우:
  ① 파티션 키 이외 컬럼으로만 조회 → ALL Partition Scan
  ② 소규모 테이블 → 오버헤드 > 이득
  ③ FOREIGN KEY 필요 → 파티셔닝 불가

TTL 삭제:
  DELETE(Row별 삭제) >> DROP PARTITION(파일 삭제)
  수억 건 삭제: DELETE 수십 분, DROP PARTITION 밀리초

파티셔닝 도입 전 확인:
  주된 조회 패턴이 파티션 키를 포함하는가?
  FOREIGN KEY를 제거할 수 있는가?
  PK에 파티션 키를 추가할 수 있는가?
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 테이블과 쿼리 패턴에서 파티셔닝을 도입해야 하는가? 이유를 설명하라.

```
테이블: user_activities (3억 건)
주요 조회 패턴:
  A. WHERE user_id = ? AND activity_date >= ?  (80%)
  B. WHERE activity_date >= ? (관리자 집계용, 5%)
  C. WHERE activity_type = ? (5%)
  D. 매월 1일, 3개월 이전 데이터 DELETE (10%)
```

<details>
<summary>해설 보기</summary>

**파티셔닝을 도입하는 것이 유리합니다.** 단, 파티션 키 선택이 중요합니다.

**현황 분석**:
- A 패턴(80%): `user_id + activity_date` 복합 조건
- B 패턴(5%): `activity_date` 범위 조회 → 파티션 프루닝 효과 있음
- D 패턴(10%): 3개월 이전 삭제 → DROP PARTITION이 극적으로 빠름

**파티션 키 선택**: `activity_date` (RANGE 월별 파티션)

**이유**:
1. 3억 건 규모 → 인덱스 B-Tree 높이 증가
2. 월별 DROP PARTITION으로 TTL 삭제 극적 개선
3. B 패턴(관리자 집계)에서 프루닝 효과

**추가로 필요한 것**:
- `(user_id, activity_date)` 복합 인덱스 (A 패턴 80%를 위해)
- PRIMARY KEY에 `activity_date` 포함 필수
- FOREIGN KEY가 있다면 제거 필요

**C 패턴 주의**: `activity_type`으로만 조회 시 ALL Partition Scan → `activity_type` 인덱스 추가 필요 (파티션 내 인덱스).

</details>

---

**Q2.** 파티셔닝이 도입된 테이블에서 다음 DELETE가 왜 느리고, 어떻게 개선하는가?

```sql
-- 매월 실행되는 배치
DELETE FROM logs WHERE created_at < DATE_SUB(NOW(), INTERVAL 3 MONTH);
```

<details>
<summary>해설 보기</summary>

**왜 느린가**: 
- Row별로 하나씩 삭제: 수천만 건 × (언두 로그 + X Lock + 인덱스 제거)
- 파티셔닝이 있어도 WHERE 조건이 여러 파티션에 걸치면 해당 파티션 모두 Row별 삭제
- 시간: 수천만 건 기준 수십 분 ~ 수 시간

**개선: DROP PARTITION 활용**

```sql
-- 월별 파티션 구조 필요
-- 현재 날짜 기준 3개월 이전 파티션 드롭

-- 파티션 구조 (월별)
ALTER TABLE logs PARTITION BY RANGE (TO_DAYS(created_at)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    ...
);

-- 배치 스크립트 (예: 2024년 10월 기준, 3개월 전 = 7월 파티션 삭제)
ALTER TABLE logs DROP PARTITION p202407;
-- 밀리초 단위로 수천만 건 삭제
```

**제약 사항**: DROP PARTITION은 파티션 내 모든 데이터를 삭제합니다. 파티션 경계와 삭제 기준이 딱 맞아야 합니다. 삭제 기준이 파티션 중간에 걸린다면 `TRUNCATE PARTITION` 후 재삽입 또는 EXCHANGE PARTITION 방법을 고려합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 파티션 종류 ➡️](./02-partition-types-range-list-hash-key.md)**

</div>
