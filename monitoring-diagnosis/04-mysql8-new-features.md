# MySQL 8.0 새 기능 — 히스토그램, Invisible Index, 쿼리 힌트

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 히스토그램이 Cardinality 통계를 보완해 불균등 데이터에서 실행계획을 개선하는 원리는?
- Invisible Index로 인덱스를 삭제 없이 비활성화해 영향도를 측정하는 방법은?
- `/*+ NO_HASH_JOIN */` 등 Optimizer Hint의 실전 활용법은?
- 히스토그램을 어떤 컬럼에 생성해야 효과적인가?
- 이 기능들이 기존 EXPLAIN + 인덱스 설계와 어떻게 보완 관계인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### Optimizer가 잘못된 실행계획을 선택하는 이유 — 통계 부정확

```
기존 통계의 한계:
  Cardinality: 컬럼의 고유값 수 (전체 분포 모름)
  예: status 컬럼, 값: PENDING(95%), PAID(4%), CANCELLED(1%)
  Cardinality = 3 (3가지 값)

  Optimizer의 오판:
  WHERE status = 'PAID' → 전체의 4%
  WHERE status = 'PENDING' → 전체의 95%
  Cardinality=3이면 두 경우를 비슷하게 추정 → 잘못된 실행계획

히스토그램 적용:
  status 컬럼의 실제 분포 기록:
  PAID=4%, PENDING=95%, CANCELLED=1%

  WHERE status = 'PAID':
    Optimizer: "4%인 걸 알아 → 인덱스 사용"
  WHERE status = 'PENDING':
    Optimizer: "95%인 걸 알아 → Full Scan이 더 효율적"
  → 데이터 분포에 맞는 올바른 실행계획
```

---

## 😱 흔한 실수 (Before)

### 1. 히스토그램 없이 불균등 데이터 조회

```sql
-- Before: 불균등 데이터 분포에서 잘못된 실행계획
-- orders.status: PAID=95%, CANCELLED=5%
-- status에 인덱스 있음

EXPLAIN SELECT * FROM orders WHERE status = 'PAID';
-- Optimizer: Cardinality=2 → 50% 예상 → Full Scan 선택 가능

EXPLAIN SELECT * FROM orders WHERE status = 'CANCELLED';
-- Optimizer: 같은 Cardinality → 인덱스 사용 또는 Full Scan

-- 실제 PAID=95%, CANCELLED=5%인데
-- 두 쿼리를 동일하게 취급 → 비효율
```

### 2. Invisible Index 없이 인덱스 삭제 영향 테스트

```sql
-- Before: 인덱스 삭제 후 영향 확인
DROP INDEX idx_old ON orders;
-- 삭제 후 슬로우 쿼리 발생
-- 되돌리기:
CREATE INDEX idx_old ON orders (status);
-- 100GB 테이블 → 수십 분 소요

-- Invisible Index 활용 (안전한 방법):
ALTER TABLE orders ALTER INDEX idx_old INVISIBLE;
-- 인덱스 데이터는 유지, Optimizer가 사용 안 함
-- 문제 발생 시 즉각 복구:
ALTER TABLE orders ALTER INDEX idx_old VISIBLE;
-- 수 초 내 복구!
```

### 3. Hint 없이 잘못된 JOIN 알고리즘 방치

```sql
-- Before: Optimizer가 잘못된 Hash Join 선택
SELECT o.id, u.name
FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.status = 'PAID';
-- EXPLAIN: hash join (users 전체를 해시 테이블로)
-- 실제로는 Nested Loop가 빠른 케이스

-- Optimizer Hint로 강제:
SELECT /*+ NO_HASH_JOIN(o, u) */ o.id, u.name
FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.status = 'PAID';
-- Nested Loop 사용
```

---

## ✨ 올바른 접근 (After)

```
MySQL 8.0 새 기능 활용 체계:

히스토그램 (데이터 분포 통계):
  불균등 데이터 컬럼에 생성
  Cardinality 통계만으로 Optimizer 오판 시 적용
  자동 갱신 안 됨 → 데이터 분포 변화 시 수동 재생성

Invisible Index (안전한 인덱스 관리):
  DROP 전 영향도 테스트
  Optimizer가 선택하지 않는 인덱스 검증 → 삭제 결정
  추가 비용 없이 "논리적 삭제" 효과

Optimizer Hint (실행계획 제어):
  EXPLAIN으로 잘못된 실행계획 확인
  통계/인덱스로도 개선 안 될 때 힌트 사용
  특정 쿼리만 영향, 다른 쿼리에 무관
```

---

## 🔬 내부 동작 원리

### 1. 히스토그램 생성과 동작

```sql
-- 히스토그램 생성
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 100 BUCKETS;
-- 100개 버킷으로 status 컬럼 분포 통계 수집
-- 실행 시간: 테이블 크기에 비례 (메모리에서 처리)

-- 히스토그램 확인
SELECT
    COLUMN_NAME,
    HISTOGRAM->>'$."histogram-type"' AS type,
    JSON_LENGTH(HISTOGRAM->>'$."buckets"') AS bucket_count
FROM information_schema.COLUMN_STATISTICS
WHERE SCHEMA_NAME = 'mydb' AND TABLE_NAME = 'orders';

-- 분포 상세 확인
SELECT
    COLUMN_NAME,
    HISTOGRAM
FROM information_schema.COLUMN_STATISTICS
WHERE SCHEMA_NAME = 'mydb' AND TABLE_NAME = 'orders'
  AND COLUMN_NAME = 'status'\G
-- 각 버킷에 누적 빈도(cumulative-frequency) 기록

-- 히스토그램 종류:
-- singleton: 각 값별 정확한 빈도 (Cardinality 낮을 때)
-- equi-height: 동등 높이 버킷 (Cardinality 높을 때)
```

### 2. 히스토그램 효과 측정

```sql
-- 히스토그램 적용 전/후 실행계획 비교

-- 테스트 데이터: status = 'PAID' 95%, 'CANCELLED' 5%
CREATE TABLE hist_test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    status VARCHAR(20),
    INDEX idx_status (status)
);

-- 히스토그램 없을 때
EXPLAIN SELECT * FROM hist_test WHERE status = 'CANCELLED';
-- rows: 예측값 부정확 가능

-- 히스토그램 생성
ANALYZE TABLE hist_test UPDATE HISTOGRAM ON status WITH 50 BUCKETS;

-- 히스토그램 있을 때
EXPLAIN SELECT * FROM hist_test WHERE status = 'CANCELLED';
-- rows: 실제 5%에 가까운 예측
-- filtered: 5.00 (정확)

-- 히스토그램으로 인한 실행계획 변화 확인
EXPLAIN FORMAT=JSON SELECT * FROM hist_test WHERE status = 'CANCELLED'\G
-- "cost_info"에 실제 예측 비용 확인
```

### 3. Invisible Index 상세

```sql
-- Invisible Index 생성
ALTER TABLE orders ADD INDEX idx_test(status, created_at);
ALTER TABLE orders ALTER INDEX idx_test INVISIBLE;

-- 확인
SELECT INDEX_NAME, IS_VISIBLE
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'orders';
-- IS_VISIBLE: NO → Optimizer가 이 인덱스 무시

-- 특정 세션에서만 Invisible Index 사용 (디버깅)
SET optimizer_switch = 'use_invisible_indexes=on';
EXPLAIN SELECT * FROM orders WHERE status = 'PAID';
-- idx_test 사용 확인 (이 세션만)

-- 영향도 측정 절차:
-- 1. 삭제할 인덱스를 INVISIBLE로 설정
-- 2. use_invisible_indexes=off (기본값)로 슬로우 쿼리 모니터링
-- 3. 영향 없으면 DROP, 영향 있으면 VISIBLE로 복구

-- INVISIBLE → VISIBLE 복구 (즉각)
ALTER TABLE orders ALTER INDEX idx_test VISIBLE;
```

### 4. Optimizer Hint 활용

```sql
-- 힌트 종류와 용도:

-- 인덱스 힌트
SELECT /*+ INDEX(orders idx_status) */ * FROM orders WHERE status = 'PAID';
SELECT /*+ NO_INDEX(orders idx_status) */ * FROM orders WHERE status = 'PAID';

-- JOIN 순서 힌트
SELECT /*+ JOIN_ORDER(u, o) */ u.name, o.id
FROM users u JOIN orders o ON o.user_id = u.id;
-- u(users)를 Outer(Driving) 테이블로 고정

-- JOIN 알고리즘 힌트
SELECT /*+ HASH_JOIN(u, o) */ u.name, o.id
FROM users u JOIN orders o ON o.user_id = u.id;

SELECT /*+ NO_HASH_JOIN(u, o) */ u.name, o.id
FROM users u JOIN orders o ON o.user_id = u.id;
-- MySQL 8.0.18+: Hash Join 지원

-- 세미조인 힌트
SELECT /*+ SEMIJOIN(MATERIALIZATION) */ u.id
FROM users u WHERE u.id IN (SELECT user_id FROM orders WHERE status='PAID');

-- BKA (Batched Key Access) 힌트
SELECT /*+ BKA(o) */ o.id FROM orders o JOIN users u ON u.id = o.user_id;

-- 힌트 테이블 레벨 적용 (특정 테이블만)
EXPLAIN SELECT /*+ INDEX(o idx_status_created) */ o.id
FROM orders o WHERE o.status = 'PAID' AND o.created_at > '2024-01-01';
```

### 5. 히스토그램 관리

```sql
-- 히스토그램 삭제 (데이터 분포 변화 후 재생성 전)
ANALYZE TABLE orders DROP HISTOGRAM ON status;

-- 여러 컬럼 히스토그램 한 번에
ANALYZE TABLE orders
  UPDATE HISTOGRAM ON status, payment_method, delivery_zone
  WITH 100 BUCKETS;

-- 히스토그램이 효과적인 컬럼:
-- 1. 인덱스 없는 컬럼에서 조건 검색 (인덱스 추가 불가한 경우)
-- 2. 불균등 분포 컬럼 (95% vs 5%)
-- 3. JOIN의 드라이빙 테이블 선택에 영향주는 컬럼

-- 히스토그램이 불필요한 컬럼:
-- 1. 균등 분포 컬럼 (모든 값이 비슷한 빈도)
-- 2. Cardinality가 매우 높은 컬럼 (id, email 등 → 이미 정확)
-- 3. 인덱스가 있고 Optimizer가 올바르게 사용 중인 컬럼

-- 재생성 주기:
-- 자동 갱신 없음 → 데이터 분포가 크게 바뀌면 수동 재실행
-- 대규모 데이터 변경(DELETE, INSERT) 후 재생성
```

---

## 💻 실전 실험

### 실험 1: 히스토그램 효과 측정

```sql
-- 불균등 데이터 테이블
CREATE TABLE hist_demo (
    id     INT AUTO_INCREMENT PRIMARY KEY,
    status VARCHAR(20),
    val    INT,
    INDEX idx_status (status)
);

-- 불균등 데이터 삽입 (ACTIVE 98%, INACTIVE 2%)
INSERT INTO hist_demo (status, val)
SELECT
    IF(MOD(seq, 50) = 0, 'INACTIVE', 'ACTIVE'),
    FLOOR(RAND() * 10000)
FROM (SELECT @r := @r+1 AS seq FROM information_schema.columns, (SELECT @r:=0) r LIMIT 100000) t;

-- 히스토그램 없이 실행계획
EXPLAIN SELECT COUNT(*) FROM hist_demo WHERE status = 'INACTIVE'\G
-- rows, filtered 확인

-- 히스토그램 생성
ANALYZE TABLE hist_demo UPDATE HISTOGRAM ON status WITH 10 BUCKETS;

-- 히스토그램 있는 실행계획
EXPLAIN SELECT COUNT(*) FROM hist_demo WHERE status = 'INACTIVE'\G
-- rows와 filtered가 더 정확해짐 확인

-- 히스토그램 내용 확인
SELECT COLUMN_NAME, HISTOGRAM
FROM information_schema.COLUMN_STATISTICS
WHERE SCHEMA_NAME = DATABASE() AND TABLE_NAME = 'hist_demo'\G

DROP TABLE hist_demo;
```

### 실험 2: Invisible Index 영향도 테스트

```sql
CREATE TABLE invis_demo (
    id   INT AUTO_INCREMENT PRIMARY KEY,
    col1 INT,
    col2 INT,
    INDEX idx_col1 (col1),
    INDEX idx_col1_col2 (col1, col2)
);

-- 미사용 가능성 있는 인덱스 (idx_col1이 idx_col1_col2에 포함)
-- idx_col1 INVISIBLE로 테스트
ALTER TABLE invis_demo ALTER INDEX idx_col1 INVISIBLE;

-- 인덱스 목록 확인
SHOW INDEX FROM invis_demo;
-- Non_visible: YES → idx_col1 비활성화

-- 쿼리 실행계획 변화 확인
EXPLAIN SELECT * FROM invis_demo WHERE col1 = 100;
-- idx_col1 없이도 idx_col1_col2로 처리되는지 확인

-- 슬로우 쿼리 없으면 안전하게 삭제 가능
DROP INDEX idx_col1 ON invis_demo;

-- 영향 있으면 즉시 복구
-- ALTER TABLE invis_demo ALTER INDEX idx_col1 VISIBLE;

DROP TABLE invis_demo;
```

### 실험 3: Optimizer Hint 효과 비교

```sql
-- Hint 없음 vs Hash Join 강제
CREATE TABLE hint_users (id INT PRIMARY KEY, name VARCHAR(100));
CREATE TABLE hint_orders (id INT AUTO_INCREMENT PRIMARY KEY, user_id INT, INDEX idx_user (user_id));

-- 데이터 삽입
INSERT INTO hint_users SELECT seq, CONCAT('User', seq) FROM (SELECT @r:=@r+1 AS seq FROM information_schema.columns, (SELECT @r:=0) r LIMIT 1000) t;
INSERT INTO hint_orders (user_id) SELECT FLOOR(RAND()*1000)+1 FROM (SELECT @r:=@r+1 FROM information_schema.columns, (SELECT @r:=0) r LIMIT 10000) t;

-- Hint 없음 (Optimizer 선택)
EXPLAIN FORMAT=TREE SELECT u.name, COUNT(*) FROM hint_users u JOIN hint_orders o ON o.user_id = u.id GROUP BY u.id\G

-- Hash Join 강제
EXPLAIN FORMAT=TREE
SELECT /*+ HASH_JOIN(u, o) */ u.name, COUNT(*)
FROM hint_users u JOIN hint_orders o ON o.user_id = u.id
GROUP BY u.id\G

-- Nested Loop 강제
EXPLAIN FORMAT=TREE
SELECT /*+ NO_HASH_JOIN(u, o) */ u.name, COUNT(*)
FROM hint_users u JOIN hint_orders o ON o.user_id = u.id
GROUP BY u.id\G

DROP TABLE hint_users, hint_orders;
```

---

## 📊 성능/비용 비교

```
각 기능 적용 효과 및 비용:

히스토그램:
  생성 비용: 테이블 Full Scan (한 번만)
  저장 비용: information_schema에 JSON (수 KB)
  쿼리 개선: 불균등 컬럼에서 10~100배 실행계획 개선 가능
  한계: 데이터 변경 시 자동 갱신 안 됨

Invisible Index:
  생성/전환 비용: 메타데이터만 변경 (수 밀리초)
  인덱스 유지 비용: VISIBLE일 때와 동일 (쓰기 비용 동일!)
  읽기 영향: Optimizer가 사용 안 함 → 읽기 성능 변화 측정 가능
  복구 속도: 즉각 (VISIBLE 전환 수 밀리초)

Optimizer Hint:
  쿼리 개발 비용: 힌트 추가 (약간의 코드 변경)
  실행 비용: 힌트 파싱 (무시 가능)
  효과: Optimizer 오판 즉각 수정
  리스크: 데이터 분포 변화 시 힌트가 역효과 가능
```

---

## ⚖️ 트레이드오프

```
히스토그램 트레이드오프:
  ✅ 인덱스 추가 없이 실행계획 개선 (단순 컬럼에 효과적)
  ✅ Full Scan 후 단 한 번 생성
  ❌ 수동 관리 필요 (데이터 변화 후 재생성)
  ❌ 모든 상황에서 효과 없음 (균등 분포, 높은 Cardinality)

Invisible Index 트레이드오프:
  ✅ 인덱스 삭제 전 안전한 영향도 테스트
  ✅ 즉각 복구 가능 (VISIBLE 전환)
  ❌ 인덱스가 INVISIBLE이어도 쓰기 비용은 유지 (유의미한 공간/CPU 낭비)
  → 최종적으로는 DROP이 목표 (INVISIBLE은 검증 단계)

Optimizer Hint 트레이드오프:
  ✅ 즉각적인 실행계획 제어
  ✅ 통계/인덱스로 해결 안 되는 케이스
  ❌ 코드에 힌트 포함 → 유지보수 복잡
  ❌ 데이터 분포 변화 시 힌트가 역효과 가능 (재검토 필요)
  → 통계/인덱스 개선이 우선, 힌트는 최후 수단
```

---

## 📌 핵심 정리

```
MySQL 8.0 새 기능 핵심:

히스토그램:
  목적: Cardinality 통계를 넘어 실제 분포 파악
  적용: 불균등 분포 컬럼, 인덱스 없는 WHERE 조건 컬럼
  생성: ANALYZE TABLE t UPDATE HISTOGRAM ON col WITH N BUCKETS
  주의: 자동 갱신 없음 → 분포 변화 시 재생성

Invisible Index:
  목적: 인덱스 삭제 전 영향도 안전 측정
  적용: 미사용 인덱스 의심 시, DROP 전 검증
  사용: ALTER TABLE t ALTER INDEX idx INVISIBLE
  복구: ALTER TABLE t ALTER INDEX idx VISIBLE (수 밀리초)
  주의: INVISIBLE이어도 쓰기 비용 유지 → 결국 DROP 필요

Optimizer Hint:
  목적: 잘못된 실행계획 즉각 수정
  종류: INDEX, NO_INDEX, JOIN_ORDER, HASH_JOIN, SEMIJOIN 등
  사용: SELECT /*+ 힌트 */ ...
  주의: 최후 수단 (통계/인덱스 개선이 우선)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 상황에서 히스토그램, Invisible Index, Optimizer Hint 중 어떤 것을 사용하는가?

```
상황:
  users 테이블 (100만 건)
  age 컬럼: 20~30세 80%, 31~60세 20% (불균등)
  age에 인덱스 없음
  WHERE age < 25 쿼리가 Full Scan으로 동작 중
```

<details>
<summary>해설 보기</summary>

**히스토그램 생성 (최적)**:

인덱스가 없는 컬럼(`age`)에서 조건 검색 시 Optimizer는 분포를 모릅니다. 히스토그램을 생성하면 `age < 25`가 전체의 몇 %인지 알 수 있습니다.

```sql
ANALYZE TABLE users UPDATE HISTOGRAM ON age WITH 100 BUCKETS;

-- 효과 확인
EXPLAIN SELECT * FROM users WHERE age < 25;
-- filtered가 더 정확해짐 (실제 ~50% 예측)
```

단, 히스토그램은 인덱스 없이 조회할 때 **rows 예측을 개선**하지만 실제 Full Scan 자체를 막지는 않습니다. 성능 개선을 원한다면 **인덱스 추가가 근본 해결**:

```sql
CREATE INDEX idx_age ON users (age);
```

히스토그램은 인덱스가 있는 경우에도 JOIN 드라이빙 테이블 선택이나 실행계획 최적화에 기여합니다.

</details>

---

**Q2.** Optimizer Hint를 남용하면 어떤 문제가 발생하는가? 힌트 대신 근본적인 해결책이 무엇인가?

<details>
<summary>해설 보기</summary>

**남용 시 문제**:

1. **데이터 분포 변화 시 역효과**: `/*+ HASH_JOIN(u, o) */`를 추가했지만, 이후 데이터 분포가 바뀌어 Nested Loop가 더 빠른 상황이 돼도 힌트 때문에 계속 Hash Join 사용

2. **유지보수 복잡**: 힌트가 SQL 곳곳에 퍼져 있으면 나중에 어떤 이유로 추가했는지 알기 어려움

3. **Optimizer 발전 불활용**: MySQL 버전 업그레이드로 Optimizer가 개선돼도 힌트가 개선을 막을 수 있음

**근본 해결책**:
- **통계 갱신**: `ANALYZE TABLE orders` → 최신 통계로 Optimizer 재판단
- **히스토그램 추가**: 불균등 분포 컬럼에 분포 통계 제공
- **인덱스 추가/개선**: 올바른 인덱스로 Optimizer가 자연스럽게 좋은 계획 선택
- **쿼리 구조 변경**: JOIN 순서, 서브쿼리 → CTE 등으로 Optimizer가 더 좋은 계획 찾도록

힌트는 위 방법으로 해결 안 될 때만 최후 수단으로 사용하고, 항상 힌트 추가 이유를 주석으로 기록합니다.

</details>

---

<div align="center">

**[⬅️ InnoDB 상태 분석](./03-innodb-status-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: 운영 문제 패턴 ➡️](./05-operational-problem-patterns.md)**

</div>
