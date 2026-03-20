# 실행계획 개선 사례 — type: ALL → ref, Using filesort 제거

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Full Table Scan(type: ALL)이 발생하는 5가지 패턴은?
- Using filesort를 제거할 수 있는 조건은?
- EXPLAIN ANALYZE로 개선 전후를 수치로 비교하는 방법은?
- 인덱스를 추가해도 Optimizer가 인덱스를 선택하지 않는 이유는?
- 실제 슬로우 쿼리를 3단계(진단 → 원인 → 해결)로 개선하는 방법론은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 실행계획을 보고 문제를 찾아 수치로 개선을 증명해야 한다

```
실무 시나리오:
  "이 쿼리가 갑자기 느려졌습니다"
  슬로우 쿼리 로그에서 발견, 실행 시간 8초

  일반적인 접근 (잘못된):
    "인덱스 걸면 되겠지" → 아무 인덱스나 추가
    → 나아지지 않거나 오히려 느려지기도 함

  올바른 접근:
    1. EXPLAIN으로 현재 실행계획 확인 → 문제 파악
    2. EXPLAIN ANALYZE로 예측 비용 vs 실제 비용 비교
    3. 원인 파악 (타입 불일치? 인덱스 없음? 선택도 낮음?)
    4. 변경 후 EXPLAIN ANALYZE 재실행 → 수치로 개선 확인

이 문서는 1~7번 챕터의 지식을 통합해
실제 슬로우 쿼리 개선 전 과정을 다룬다
```

---

## 😱 흔한 실수 (Before)

### 1. EXPLAIN은 보지만 해석을 못함

```sql
-- EXPLAIN 결과를 보고 어디가 문제인지 모름
EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status = 'PAID' ORDER BY created_at DESC;

-- 출력:
-- id | type | key       | rows  | Extra
-- 1  | ref  | idx_user  | 5000  | Using where; Using filesort

-- "ref니까 인덱스 쓰는 거 아닌가요?" → rows=5000에서 5000건 정렬
-- "Using filesort가 뭐죠?" → 대부분 무시

-- 놓친 포인트:
-- rows=5000 → 5000건 읽은 후 WHERE status='PAID' 필터 → 정렬
-- status 조건이 인덱스에 없어서 5000건을 읽고 필터링 중
-- created_at ORDER BY가 인덱스에 없어서 filesort 발생
```

### 2. 인덱스를 추가했는데 Optimizer가 안 씀

```sql
-- After: 인덱스 추가
CREATE INDEX idx_status ON orders(status);

-- EXPLAIN 재실행
EXPLAIN SELECT * FROM orders WHERE status = 'ACTIVE';
-- type: ALL (여전히 Full Scan!)

-- 이유:
-- status = 'ACTIVE'가 전체의 95% → 인덱스가 오히려 비효율
-- Optimizer: "인덱스 써봤자 95% Row에 다 접근 → Full Scan이 더 빠름"
-- 통계 정보 기반 Cardinality 판단
```

---

## ✨ 올바른 접근 (After)

```
EXPLAIN 읽기 방법론:

1단계: type 컬럼 확인 (빠른 → 느린 순서)
  system > const > eq_ref > ref > range > index > ALL
  ALL → 즉시 개선 대상
  index → Full Index Scan (ALL보다 낫지만 주의)

2단계: key 컬럼 확인
  NULL → 인덱스 미사용 → 인덱스 추가 또는 쿼리 수정
  인덱스 있지만 NULL → 형변환/함수 등으로 무력화 의심

3단계: rows 컬럼 확인
  조건 후 결과 건수 대비 rows가 너무 크면
  → 인덱스 선택도 문제 또는 통계 정보 부정확

4단계: Extra 컬럼 확인
  Using filesort → ORDER BY에 인덱스 미사용
  Using temporary → 임시 테이블 생성
  Using index → Covering Index (좋음)
  Using where → 인덱스 후 추가 필터링 (rows 확인 필요)

5단계: EXPLAIN ANALYZE로 실제 실행 비용 확인
  actual rows vs estimated rows 차이 → 통계 부정확 신호
  actual loops → 실제 실행 횟수 (서브쿼리 Dependent 여부)
```

---

## 🔬 내부 동작 원리

### 1. Full Table Scan이 발생하는 5가지 패턴

```sql
-- 패턴 1: 인덱스 없음
-- orders.memo 컬럼에 인덱스 없음
SELECT * FROM orders WHERE memo LIKE '%배송%';
-- type: ALL
-- 해결: 전문 검색이 필요하면 FULLTEXT INDEX
--       단순 매칭이면 정확한 조건으로 인덱스 설계

-- 패턴 2: 인덱스 있지만 함수/형변환으로 무력화
-- created_at에 INDEX 있음
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
-- type: ALL (YEAR() 함수로 인덱스 무력화)
-- 해결: WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- 패턴 3: LIKE 앞 와일드카드
-- name에 INDEX 있음
SELECT * FROM users WHERE name LIKE '%Kim%';
-- type: ALL (%로 시작하면 B-Tree 탐색 불가)
-- 해결: FULLTEXT INDEX 또는 검색 엔진 도입

-- 패턴 4: OR 조건이 다른 인덱스 컬럼에 걸릴 때
SELECT * FROM orders WHERE user_id = 1 OR status = 'PAID';
-- user_id, status 각각 인덱스가 있어도
-- OR 조건은 두 인덱스 결과의 UNION → 경우에 따라 Full Scan
-- type: index_merge 또는 ALL
-- 해결: UNION ALL로 분리 후 결합

-- 패턴 5: Cardinality 낮은 컬럼 + 많은 비율 매칭
SELECT * FROM orders WHERE status = 'PAID';  -- 90%가 PAID
-- Optimizer: 인덱스 쓰면 90% Row에 Clustered Index 접근 필요
--            Full Scan이 더 효율적 → type: ALL 선택
-- 해결: 복합 인덱스로 추가 조건과 결합 또는 파티셔닝
```

### 2. Using filesort 제거 방법론

```sql
-- 인덱스: (user_id)

-- Case A: ORDER BY 컬럼을 인덱스에 추가
-- Before: Using filesort
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC;
-- key: idx_user, rows: 5000, Extra: Using filesort

-- After: 복합 인덱스 추가
CREATE INDEX idx_user_created ON orders (user_id, created_at);
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC;
-- key: idx_user_created, rows: 5000
-- Extra: Using index condition (filesort 없음)

-- Case B: Covering Index로 filesort 제거
-- Before
EXPLAIN SELECT id, user_id, created_at FROM orders WHERE user_id = 1 ORDER BY created_at DESC;
-- Extra: Using index condition; Using filesort (SELECT에 다른 컬럼 없어도)

-- After: SELECT 컬럼을 인덱스에 포함 (Covering Index)
CREATE INDEX idx_user_created_cover ON orders (user_id, created_at, id);
-- 또는 SELECT 컬럼 줄이기
SELECT id, created_at FROM orders WHERE user_id = 1 ORDER BY created_at DESC;
-- Extra: Using index (Covering Index + filesort 없음)

-- Case C: ORDER BY 방향 통일
-- Before: 방향 혼합으로 filesort
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at ASC, amount DESC;
-- Extra: Using filesort

-- After: 방향 통일
SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC, amount DESC;
-- idx_user_created_amount DESC DESC 인덱스 또는 방향 변경 검토
```

### 3. EXPLAIN ANALYZE로 수치 측정

```sql
-- EXPLAIN ANALYZE 출력 해석
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1 AND status = 'PAID'
ORDER BY created_at DESC LIMIT 10;

-- 출력 예시:
-- -> Limit: 10 row(s)
--    (cost=4523 rows=10) (actual time=156.2..156.3 rows=10 loops=1)
--   -> Filter: (orders.status = 'PAID')
--      (cost=4523 rows=450) (actual time=0.5..156.2 rows=50 loops=1)
--     -> Index scan on orders using idx_user (user_id=1)
--        (cost=4523 rows=5000) (actual time=0.4..151.0 rows=5000 loops=1)

-- 해석:
-- rows=5000 (예측) vs actual rows=5000 (실제) → 통계 정확
-- 5000건을 읽어 status 필터 후 50건 → 5000건 읽는 비용 낭비
-- actual time=151ms → 5000건 읽기 시간 큼

-- 개선 후 재측정
CREATE INDEX idx_user_status_created ON orders (user_id, status, created_at);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1 AND status = 'PAID'
ORDER BY created_at DESC LIMIT 10;

-- 개선 후 예상 출력:
-- -> Limit: 10 row(s) (actual time=0.1..0.1 rows=10 loops=1)
--   -> Index scan using idx_user_status_created (user_id=1, status='PAID')
--      (actual time=0.05..0.1 rows=10 loops=1)
-- actual time이 156ms → 0.1ms로 개선
```

### 4. 실제 슬로우 쿼리 3가지 케이스

```sql
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
-- 케이스 1: 서브쿼리 Dependent → JOIN
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

-- Before (느림): 각 user마다 서브쿼리 실행
SELECT u.id, u.name
FROM users u
WHERE u.id IN (
    SELECT o.user_id FROM orders o WHERE o.status = 'PAID'
    AND o.created_at >= '2024-01-01'
);
-- EXPLAIN: DEPENDENT SUBQUERY, type=ALL on orders

-- After (빠름): Semi-Join 또는 직접 JOIN
SELECT DISTINCT u.id, u.name
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.status = 'PAID'
AND o.created_at >= '2024-01-01';
-- EXPLAIN: eq_ref on users, ref on orders
-- + CREATE INDEX idx_orders_status_date ON orders(status, created_at, user_id);

-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
-- 케이스 2: 함수 적용 → 범위 조건 변환
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

-- Before (느림): MONTH() 함수 적용
SELECT COUNT(*) FROM orders WHERE MONTH(created_at) = 3;
-- type: ALL (MONTH() 함수 → 인덱스 무력화)

-- After (빠름): 범위 조건으로 변환
SELECT COUNT(*) FROM orders
WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
-- type: range (created_at 인덱스 활용)
-- + CREATE INDEX idx_created ON orders(created_at);

-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
-- 케이스 3: 불필요한 filesort + 커버링 인덱스
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

-- Before (느림): 목록 API, 매번 filesort
SELECT id, user_id, status, created_at, amount
FROM orders
WHERE user_id = 1
ORDER BY created_at DESC
LIMIT 20;
-- EXPLAIN: Using filesort (created_at 인덱스 없음)

-- After (빠름): 복합 인덱스 + SELECT 컬럼 최소화
CREATE INDEX idx_user_created ON orders (user_id, created_at, id, status, amount);
-- 또는 커버링 인덱스: (user_id, created_at) + SELECT에서 나머지는 PK로 조회
-- 실제로는 SELECT * 대신 필요한 컬럼만

SELECT id, user_id, status, created_at, amount
FROM orders
WHERE user_id = 1
ORDER BY created_at DESC
LIMIT 20;
-- EXPLAIN: Using index (Covering Index, filesort 없음)
```

---

## 💻 실전 실험

### 실험 1: Full Scan → ref 개선 실습

```sql
-- 실험 데이터 (10만 건 주문)
CREATE TABLE perf_orders (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    status     VARCHAR(20) NOT NULL,
    created_at DATETIME NOT NULL,
    amount     DECIMAL(10,2)
) ENGINE=InnoDB;

-- 데이터 삽입 (실제 환경에서는 더 많이)
INSERT INTO perf_orders (user_id, status, created_at, amount)
SELECT
    FLOOR(RAND()*1000)+1,
    ELT(FLOOR(RAND()*3)+1, 'PAID','PENDING','CANCELLED'),
    NOW() - INTERVAL FLOOR(RAND()*365) DAY,
    ROUND(RAND()*100000, 2)
FROM information_schema.columns a
CROSS JOIN information_schema.columns b
LIMIT 100000;

-- Before: 인덱스 없음
EXPLAIN ANALYZE
SELECT * FROM perf_orders WHERE user_id = 1 ORDER BY created_at DESC LIMIT 20;

-- 인덱스 추가
CREATE INDEX idx_user_created ON perf_orders (user_id, created_at);

-- After: 인덱스 있음
EXPLAIN ANALYZE
SELECT * FROM perf_orders WHERE user_id = 1 ORDER BY created_at DESC LIMIT 20;
-- actual time 비교
```

### 실험 2: 통계 부정확으로 인덱스를 안 쓰는 케이스

```sql
-- Optimizer가 인덱스를 거부하는 경우 재현
-- status 값이 99%가 'PAID'인 데이터
CREATE TABLE biased_orders LIKE perf_orders;
INSERT INTO biased_orders (user_id, status, created_at, amount)
SELECT user_id, 'PAID', created_at, amount FROM perf_orders;
-- 99% PAID

CREATE INDEX idx_status ON biased_orders(status);

EXPLAIN SELECT * FROM biased_orders WHERE status = 'PAID';
-- type: ALL (Optimizer가 인덱스 거부 → rows가 전체에 가까우므로)

EXPLAIN SELECT * FROM biased_orders WHERE status = 'CANCELLED';
-- type: ref (CANCELLED는 소수 → 인덱스 사용)

-- 힌트로 강제
EXPLAIN SELECT * FROM biased_orders FORCE INDEX(idx_status) WHERE status = 'PAID';
-- 강제로 인덱스 사용 → 실제로 더 느릴 수 있음

DROP TABLE biased_orders;
```

### 실험 3: EXPLAIN ANALYZE 수치 비교

```sql
-- 쿼리 개선 전후 수치 기록 방법

-- 개선 전 측정
EXPLAIN ANALYZE
SELECT p.id, p.user_id
FROM perf_orders p
WHERE p.user_id IN (
    SELECT user_id FROM perf_orders WHERE status = 'PAID' AND created_at >= '2024-01-01'
)
LIMIT 100;
-- actual time, actual rows 기록

-- 개선 후 (JOIN으로 변환)
CREATE INDEX idx_status_date_user ON perf_orders (status, created_at, user_id);

EXPLAIN ANALYZE
SELECT DISTINCT p.id, p.user_id
FROM perf_orders p
JOIN (
    SELECT user_id FROM perf_orders
    WHERE status = 'PAID' AND created_at >= '2024-01-01'
) sub ON sub.user_id = p.user_id
LIMIT 100;
-- actual time 비교

DROP TABLE perf_orders;
```

---

## 📊 성능/비용 비교

```
개선 전후 수치 비교 (실제 측정 예시):

케이스 1: Dependent Subquery → JOIN
  Before:
    EXPLAIN: DEPENDENT SUBQUERY, type=ALL
    actual time: 8,500ms (users 10만 × orders 서브쿼리)
  After (JOIN + 인덱스):
    EXPLAIN: eq_ref, ref
    actual time: 45ms
  개선: 약 190배

케이스 2: 함수 적용 → 범위 조건
  Before:
    EXPLAIN: type=ALL, Extra: Using where
    actual time: 2,100ms
  After (범위 조건 + 인덱스):
    EXPLAIN: type=range, Extra: Using index condition
    actual time: 12ms
  개선: 약 175배

케이스 3: filesort 제거
  Before:
    EXPLAIN: ref + Using filesort
    actual time: 350ms (5,000건 읽고 정렬)
  After (복합 인덱스):
    EXPLAIN: ref + Using index
    actual time: 0.8ms (인덱스 순서 20건만 읽음)
  개선: 약 440배
```

---

## ⚖️ 트레이드오프

```
실행계획 개선 시 트레이드오프:

인덱스 추가:
  ✅ 조회 성능 개선 (ref, range)
  ❌ INSERT/UPDATE/DELETE 성능 소폭 감소
  ❌ 인덱스 크기 → 메모리(Buffer Pool) 압박
  ❌ 불필요한 인덱스는 오히려 Optimizer를 혼란시킬 수 있음
  판단 기준: 읽기 대비 쓰기 비율, 조회 빈도, 데이터 크기

Optimizer 힌트:
  ✅ 특정 케이스에서 잘못된 실행계획 강제 수정
  ❌ 힌트가 잘못되면 원래보다 나빠짐
  ❌ 통계 갱신 후 힌트가 불필요해져도 남아있을 수 있음
  → 힌트보다 인덱스와 통계 개선이 우선

통계 갱신:
  ANALYZE TABLE orders;
  ✅ 통계 부정확으로 잘못된 실행계획 선택 시 개선
  ❌ 대용량 테이블에서 ANALYZE TABLE 자체가 비용
  → 자동 갱신 설정 확인 (innodb_stats_auto_recalc)

쿼리 구조 변경:
  ✅ 근본적인 성능 개선 (Dependent → JOIN)
  ❌ 레거시 코드 변경 시 회귀 테스트 필요
  ❌ JPA 사용 시 JPQL 또는 QueryDSL 변경 필요
```

---

## 📌 핵심 정리

```
실행계획 개선 방법론:

1단계 진단:
  EXPLAIN: type/key/rows/Extra 확인
  EXPLAIN ANALYZE: actual time/rows/loops 수치화

2단계 원인 파악:
  type=ALL + 인덱스 있음 → 형변환/함수로 무력화 의심
  type=ALL + 인덱스 없음 → 인덱스 추가
  DEPENDENT SUBQUERY → JOIN으로 변환
  Using filesort → ORDER BY에 인덱스 추가
  Using temporary → GROUP BY/ORDER BY 인덱스 검토

3단계 개선:
  인덱스 추가 (가장 흔한 해결)
  쿼리 구조 변경 (Subquery → JOIN, 함수 → 범위 조건)
  Covering Index로 Using index 달성

4단계 검증:
  EXPLAIN ANALYZE 재실행 → actual time 수치 비교
  SHOW STATUS LIKE 'Handler_read%' → 실제 Row 접근 수
  Slow Query Log에서 해당 쿼리 사라짐 확인

자주 보이는 문제 패턴:
  type: ALL + Using where → 인덱스 없거나 무력화
  DEPENDENT SUBQUERY → 반복 실행
  rows가 결과보다 훨씬 큼 → 인덱스 선택도 문제
  Using filesort → ORDER BY 인덱스 불일치
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 EXPLAIN 결과에서 문제를 찾고 개선 방법을 제시하라.

```
id | select_type        | table  | type | key       | rows  | Extra
1  | PRIMARY            | users  | ALL  | NULL      | 50000 | Using where
2  | DEPENDENT SUBQUERY | orders | ref  | idx_user  | 3     | Using where
```

<details>
<summary>해설 보기</summary>

**문제 분석**:
1. `users` 테이블: `type=ALL`, `key=NULL` → Full Table Scan, 인덱스 미사용
2. `select_type=DEPENDENT SUBQUERY` → `users`의 각 Row(50,000건)마다 `orders` 서브쿼리 실행
3. 실제 실행 횟수: 50,000 × 1 = orders 서브쿼리 50,000번

**원인**: 아마도 다음과 같은 쿼리 패턴일 것입니다:
```sql
SELECT * FROM users u
WHERE u.id IN (SELECT user_id FROM orders o WHERE o.status = u.status);
-- 또는
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'PAID');
```

**개선 방법**:

1. `users` Full Scan 개선 — WHERE 조건에 해당하는 인덱스 추가
2. DEPENDENT SUBQUERY → JOIN으로 변환:
```sql
SELECT DISTINCT u.id, u.name
FROM users u
JOIN orders o ON o.user_id = u.id AND o.status = 'PAID';
-- CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

3. EXPLAIN ANALYZE로 개선 전후 actual time 비교해 효과 측정.

</details>

---

**Q2.** 인덱스를 추가했는데도 Optimizer가 여전히 Full Scan을 선택한다. 어떻게 조사하고 해결하는가?

<details>
<summary>해설 보기</summary>

**조사 단계**:

1. **통계 확인**: 인덱스 Cardinality가 낮으면 Optimizer가 Full Scan을 선호합니다.
```sql
SHOW INDEX FROM orders;
-- Cardinality 확인: 너무 낮으면 인덱스 효율 낮음
```

2. **통계 갱신**: 대량 데이터 변경 후 통계가 부정확할 수 있습니다.
```sql
ANALYZE TABLE orders;
EXPLAIN SELECT ...; -- 통계 갱신 후 재확인
```

3. **데이터 분포 확인**: 조건 해당 비율 확인.
```sql
SELECT status, COUNT(*) / (SELECT COUNT(*) FROM orders) AS ratio
FROM orders GROUP BY status;
-- 조건 해당 비율이 30% 이상이면 Full Scan이 유리할 수 있음
```

4. **복합 인덱스 검토**: 단일 컬럼 인덱스보다 복합 인덱스가 선택도 높을 수 있습니다.
```sql
-- status + created_at 복합 인덱스
CREATE INDEX idx_status_created ON orders(status, created_at);
```

5. **힌트로 강제 후 비교**: 실제로 인덱스가 더 빠른지 확인.
```sql
-- FORCE INDEX로 강제
EXPLAIN ANALYZE SELECT * FROM orders FORCE INDEX(idx_status) WHERE status = 'ACTIVE';
-- IGNORE INDEX로 Full Scan 강제
EXPLAIN ANALYZE SELECT * FROM orders IGNORE INDEX(idx_status) WHERE status = 'ACTIVE';
-- 두 결과의 actual time 비교
```

인덱스를 강제했을 때 오히려 더 느리면 Optimizer의 판단이 맞는 것입니다. 그 경우 쿼리 구조 자체를 재검토하거나 파티셔닝 등 다른 접근을 고려합니다.

</details>

---

<div align="center">

**[⬅️ 문자셋과 콜레이션](./06-charset-collation-implicit-conversion.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 파티셔닝 ➡️](../partitioning/01-when-to-use-partitioning.md)**

</div>
