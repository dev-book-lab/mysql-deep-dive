# 파생 테이블(Derived Table)과 CTE — 임시 테이블이 생성되는 조건

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- FROM 절 서브쿼리(파생 테이블)가 항상 임시 테이블을 만드는가?
- CTE(WITH 절)가 Materialized로 처리되는 조건과 Merged로 처리되는 조건은 무엇인가?
- EXPLAIN에서 `select_type: DERIVED`와 `MATERIALIZED`의 차이는?
- 파생 테이블 대신 CTE를 쓰면 무엇이 달라지는가?
- 임시 테이블이 디스크로 넘어가는 조건과 `tmp_table_size` 설정의 의미는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 임시 테이블이 디스크에 생성되면 성능이 폭락한다

```
실제 이슈:
  통계 집계 쿼리가 평소에는 1초, 특정 기간엔 30초 걸림

  원인:
    FROM 절에 서브쿼리로 집계 후 JOIN
    평소: 서브쿼리 결과 10만 건 → 메모리 임시 테이블 (빠름)
    월말 집계: 결과 500만 건 → tmp_table_size 초과
             → 디스크 임시 테이블(MyISAM/TempTable) 생성
             → 디스크 I/O 폭발 → 30초

  해결:
    쿼리 구조 개선 또는 tmp_table_size/max_heap_table_size 조정

알아야 할 것:
  어떤 쿼리가 임시 테이블을 만드는가
  메모리 vs 디스크 임시 테이블 전환 조건
  CTE가 이 문제를 어떻게 다르게 처리하는가
```

---

## 😱 흔한 실수 (Before)

### 1. 파생 테이블 중첩

```sql
-- Before: 불필요하게 중첩된 FROM 서브쿼리
SELECT *
FROM (
    SELECT *
    FROM (
        SELECT user_id, SUM(amount) AS total
        FROM orders
        WHERE status = 'PAID'
        GROUP BY user_id
    ) AS sub1
    WHERE total > 10000
) AS sub2
WHERE user_id IN (SELECT id FROM vip_users);

-- 각 서브쿼리 계층마다 임시 테이블 가능성
-- 3개의 임시 테이블이 순서대로 생성될 수 있음
```

### 2. CTE를 재사용할 때 반복 실체화 예상

```sql
-- Before: CTE가 참조될 때마다 재실행된다고 오해
WITH monthly_stats AS (
    SELECT user_id, COUNT(*) AS cnt
    FROM orders
    WHERE created_at >= '2024-01-01'
    GROUP BY user_id
)
SELECT u.name, ms.cnt
FROM users u
JOIN monthly_stats ms ON ms.user_id = u.id
WHERE ms.cnt > 10;
-- "monthly_stats가 매번 실행되는 건 아닌가?" → Materialized면 1번만
```

### 3. 모든 CTE가 Materialized라고 가정

```sql
-- Before: CTE를 쓰면 항상 임시 테이블이 생긴다고 오해
WITH active_users AS (
    SELECT id FROM users WHERE status = 'ACTIVE'
)
SELECT * FROM orders o
WHERE o.user_id IN (SELECT id FROM active_users);
-- 실제: 단순 CTE는 Merged(인라인화)될 수 있어 임시 테이블 없음
```

---

## ✨ 올바른 접근 (After)

```
파생 테이블(DERIVED) 처리:
  MySQL 8.0: Derived Table Merging 최적화 존재
  → 가능한 경우 파생 테이블을 외부 쿼리로 합침 (임시 테이블 없음)
  → 불가능한 경우에만 임시 테이블 생성

CTE(Materialized vs Merged):
  Merged:
    CTE가 단순 필터/컬럼 선택 → 외부 쿼리에 인라인화
    임시 테이블 없음, 외부 쿼리의 인덱스 활용 가능
  Materialized:
    CTE에 집계, DISTINCT, GROUP BY, UNION 등 포함 → 한 번 실체화
    임시 테이블 1번 생성, 여러 번 참조해도 재실행 없음

임시 테이블이 생성되는 조건:
  집계함수(SUM, COUNT 등)
  GROUP BY 또는 DISTINCT
  UNION / UNION ALL
  window 함수 일부
  ORDER BY + LIMIT 조합 (특정 조건)
  Merge가 불가능한 서브쿼리 구조
```

---

## 🔬 내부 동작 원리

### 1. Derived Table Merging — 임시 테이블 없이 처리

```sql
-- 파생 테이블이 Merge되는 예시
SELECT t.*
FROM (
    SELECT id, name, status
    FROM users
    WHERE status = 'ACTIVE'
) t
WHERE t.name LIKE 'Kim%';

-- Merge 가능 조건: 서브쿼리가 단순 SELECT/WHERE/JOIN
-- Optimizer가 다음과 동일하게 처리:
SELECT id, name, status
FROM users
WHERE status = 'ACTIVE' AND name LIKE 'Kim%';

-- EXPLAIN에서:
-- Merge된 경우: select_type=SIMPLE, 서브쿼리 사라짐
-- Merge 안 된 경우: select_type=DERIVED

-- Merge 불가 조건:
--   GROUP BY, DISTINCT, 집계함수, WINDOW 함수
--   UNION / UNION ALL
--   LIMIT (ORDER BY 없는 단독 LIMIT은 가능)
--   외부 쿼리에서 파생 테이블의 집계 결과를 참조
```

### 2. CTE Materialized vs Merged 결정

```sql
-- Merged CTE (임시 테이블 없음)
EXPLAIN
WITH active AS (
    SELECT id, name FROM users WHERE status = 'ACTIVE'
)
SELECT o.id, a.name
FROM orders o
JOIN active a ON a.id = o.user_id;
-- CTE가 단순 → Merged → EXPLAIN에서 CTE 사라지고 users 직접 접근

-- Materialized CTE (임시 테이블 생성)
EXPLAIN
WITH order_stats AS (
    SELECT user_id, COUNT(*) AS cnt, SUM(amount) AS total
    FROM orders
    GROUP BY user_id    -- GROUP BY → Materialized
)
SELECT u.name, os.cnt, os.total
FROM users u
JOIN order_stats os ON os.user_id = u.id
WHERE os.cnt > 5;
-- select_type: MATERIALIZED → 임시 테이블 생성됨

-- MATERIALIZED 강제 힌트
WITH /*+ MATERIALIZED */ my_cte AS (
    SELECT id FROM users WHERE status = 'ACTIVE'
)
SELECT * FROM orders WHERE user_id IN (SELECT id FROM my_cte);

-- MERGE 강제 힌트
WITH /*+ MERGE */ my_cte AS (
    SELECT id FROM users WHERE status = 'ACTIVE'
)
SELECT * FROM orders WHERE user_id IN (SELECT id FROM my_cte);
```

### 3. 임시 테이블 메모리 vs 디스크 전환

```
임시 테이블 저장 위치 결정:

메모리 임시 테이블 (TempTable 엔진, MySQL 8.0):
  조건: 결과 크기 < tmp_table_size AND max_heap_table_size
  기본값: tmp_table_size = 16MB
  빠름 (메모리 접근)

디스크 임시 테이블:
  조건: 결과 크기 > tmp_table_size 또는
        BLOB/TEXT 컬럼 포함 (무조건 디스크)
        또는 temptable_max_ram 초과
  느림 (디스크 I/O)

MySQL 8.0 TempTable vs 구버전 MEMORY 엔진:
  MEMORY: BLOB/TEXT 지원 안 함, VARCHAR 고정 길이
  TempTable: BLOB/TEXT 지원, 더 효율적인 메모리 관리

확인 방법:
  EXPLAIN Extra: Using temporary → 임시 테이블 사용
  SHOW STATUS LIKE 'Created_tmp%';
    Created_tmp_tables: 생성된 메모리 임시 테이블 수
    Created_tmp_disk_tables: 디스크로 넘어간 임시 테이블 수

튜닝:
  SET tmp_table_size = 64 * 1024 * 1024;  -- 64MB
  SET max_heap_table_size = 64 * 1024 * 1024;
  -- 두 값이 같아야 함 (둘 중 작은 값이 적용)

디스크 임시 테이블을 줄이는 방법:
  1. tmp_table_size 증가
  2. 쿼리 개선 → GROUP BY 결과 건수 감소
  3. BLOB/TEXT 컬럼을 가능하면 피함
  4. 파생 테이블 제거 → Merge 가능한 구조로 변환
```

### 4. CTE 재귀 (Recursive CTE)

```sql
-- 계층 구조 조회 (재귀 CTE)
-- database-internals에서 다루지 않은 MySQL 8.0 기능
WITH RECURSIVE category_tree AS (
    -- 기본 조건 (재귀 종료 기준)
    SELECT id, name, parent_id, 0 AS depth
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- 재귀 조건
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON ct.id = c.parent_id
    WHERE ct.depth < 10  -- 무한 루프 방지
)
SELECT * FROM category_tree ORDER BY depth, id;

-- 주의사항:
-- 재귀 CTE는 항상 Materialized (임시 테이블)
-- cte_max_recursion_depth 기본값 = 1000
-- 큰 계층 구조는 성능 주의
```

---

## 💻 실전 실험

### 실험 1: Derived Table Merging 확인

```sql
-- Merge되는 케이스
EXPLAIN
SELECT t.id, t.name
FROM (
    SELECT id, name FROM users WHERE status = 'ACTIVE'
) t
WHERE t.name LIKE 'A%';
-- select_type이 SIMPLE이면 Merge 성공

-- Merge 안 되는 케이스
EXPLAIN
SELECT t.user_id, t.total
FROM (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    GROUP BY user_id
) t
WHERE t.total > 50000;
-- select_type=DERIVED → 임시 테이블 생성됨

-- Extra: Using temporary 여부도 확인
```

### 실험 2: CTE Materialized vs Merged 비교

```sql
-- Merged CTE 확인
EXPLAIN FORMAT=JSON
WITH cte AS (SELECT id FROM users WHERE status = 'ACTIVE')
SELECT COUNT(*) FROM orders o WHERE o.user_id IN (SELECT id FROM cte);
-- output의 "materialized" 여부 확인

-- Materialized CTE 강제
EXPLAIN FORMAT=JSON
WITH /*+ MATERIALIZED */ cte AS (SELECT id FROM users WHERE status = 'ACTIVE')
SELECT COUNT(*) FROM orders o WHERE o.user_id IN (SELECT id FROM cte);

-- 실행 시간 비교
SET @t = SYSDATE(6);
WITH cte AS (SELECT id FROM users WHERE status = 'ACTIVE')
SELECT COUNT(*) FROM orders o WHERE o.user_id IN (SELECT id FROM cte);
SELECT CONCAT('Merged: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

SET @t = SYSDATE(6);
WITH /*+ MATERIALIZED */ cte AS (SELECT id FROM users WHERE status = 'ACTIVE')
SELECT COUNT(*) FROM orders o WHERE o.user_id IN (SELECT id FROM cte);
SELECT CONCAT('Materialized: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');
```

### 실험 3: 임시 테이블 메모리/디스크 전환 확인

```sql
-- 임시 테이블 통계 초기화
FLUSH STATUS;

-- 메모리 임시 테이블 생성
SELECT user_id, COUNT(*), SUM(amount)
FROM orders
GROUP BY user_id;

SHOW STATUS LIKE 'Created_tmp%';
-- Created_tmp_tables: 증가
-- Created_tmp_disk_tables: 디스크 전환 여부

-- 강제로 작은 tmp_table_size 설정 후 디스크 전환 유도
SET tmp_table_size = 1024;      -- 1KB (매우 작게)
SET max_heap_table_size = 1024;

FLUSH STATUS;
SELECT user_id, COUNT(*), SUM(amount) FROM orders GROUP BY user_id;
SHOW STATUS LIKE 'Created_tmp%';
-- Created_tmp_disk_tables 증가 확인

-- 원복
SET tmp_table_size = DEFAULT;
SET max_heap_table_size = DEFAULT;
```

### 실험 4: 중첩 파생 테이블 개선

```sql
-- Before: 중첩 파생 테이블
EXPLAIN ANALYZE
SELECT *
FROM (
    SELECT user_id, total
    FROM (
        SELECT user_id, SUM(amount) AS total
        FROM orders WHERE status = 'PAID'
        GROUP BY user_id
    ) sub1
    WHERE total > 10000
) sub2
ORDER BY total DESC
LIMIT 10;

-- After: CTE로 리팩토링 (더 명확하고 동일한 실행계획)
EXPLAIN ANALYZE
WITH paid_totals AS (
    SELECT user_id, SUM(amount) AS total
    FROM orders WHERE status = 'PAID'
    GROUP BY user_id
    HAVING SUM(amount) > 10000
)
SELECT * FROM paid_totals
ORDER BY total DESC
LIMIT 10;
-- 실행계획 비교 (동일하거나 더 좋음)
```

---

## 📊 성능/비용 비교

```
파생 테이블 처리 방식별 비용:

Merged (임시 테이블 없음):
  비용: 일반 쿼리와 동일
  메모리: 추가 없음
  가장 빠름

Materialized (메모리 임시 테이블):
  비용: 서브쿼리 실행 + 임시 테이블 생성 + 조인
  메모리: 서브쿼리 결과 크기
  빠름 (메모리 안에 있을 때)

Materialized (디스크 임시 테이블):
  비용: 서브쿼리 실행 + 디스크 기록 + 조인
  I/O: 디스크 쓰기 + 읽기
  느림 → tmp_table_size 증가 또는 쿼리 개선 필요

CTE Materialized + 여러 번 참조:
  실체화: 1번 실행
  참조: N번 참조해도 재실행 없음
  → 동일 집계를 여러 곳에서 쓸 때 CTE가 유리

같은 서브쿼리를 여러 번 반복 (Without CTE):
  각 참조마다 서브쿼리 실행 가능
  → CTE Materialization이 더 효율적
```

---

## ⚖️ 트레이드오프

```
파생 테이블 vs CTE:

파생 테이블 (FROM 서브쿼리):
  ✅ SQL 표준, 모든 DB에서 동작
  ✅ 명시적인 데이터 흐름
  ❌ 중첩 시 가독성 저하
  ❌ 동일 서브쿼리를 여러 곳에서 재사용 불가

CTE (WITH 절):
  ✅ 가독성 높음, 재사용 가능
  ✅ 재귀 CTE로 계층 구조 처리 가능
  ✅ Materialized 시 여러 참조에서 재실행 없음
  ❌ MySQL 8.0 이상 필요 (구버전 불가)
  ❌ Materialized CTE는 임시 테이블 메모리 사용

Materialization 선택:
  한 번만 참조되는 단순 필터 → Merged 선호 (임시 테이블 없음)
  집계 결과를 여러 번 참조 → Materialized 선호 (재실행 없음)
  BLOB/TEXT 컬럼 포함 → 항상 디스크 임시 테이블 → 컬럼 설계 재검토

tmp_table_size 튜닝:
  크게 설정 → 메모리 사용 증가, 디스크 전환 줄어듦
  너무 작게 → 빈번한 디스크 임시 테이블 → 느림
  권장: Created_tmp_disk_tables / Created_tmp_tables 비율 모니터링
  → 10% 이상이면 tmp_table_size 증가 고려
```

---

## 📌 핵심 정리

```
파생 테이블과 CTE 핵심:

파생 테이블(DERIVED):
  단순 필터/컬럼 선택 → Derived Table Merging → 임시 테이블 없음
  집계/GROUP BY/UNION 포함 → 임시 테이블 생성
  EXPLAIN select_type=DERIVED → 임시 테이블 생성 확인

CTE(WITH 절):
  단순 필터 → Merged → 임시 테이블 없음 (외부 쿼리 인라인화)
  집계/GROUP BY/DISTINCT → Materialized → 임시 테이블 1번 생성
  재귀 CTE → 항상 Materialized

임시 테이블 메모리 전환:
  결과 크기 > tmp_table_size → 디스크 전환 → 성능 저하
  BLOB/TEXT 컬럼 → 항상 디스크
  모니터링: Created_tmp_disk_tables 증가 추이

진단:
  EXPLAIN Extra: Using temporary → 임시 테이블 사용
  EXPLAIN select_type: DERIVED → 파생 테이블
  EXPLAIN select_type: MATERIALIZED → CTE Materialized
  SHOW STATUS LIKE 'Created_tmp%' → 디스크 전환 비율
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 CTE가 Materialized로 처리되는가, Merged로 처리되는가? 이유를 설명하라.

```sql
WITH recent_orders AS (
    SELECT user_id, MAX(created_at) AS last_order_date
    FROM orders
    WHERE created_at >= '2024-01-01'
    GROUP BY user_id
)
SELECT u.name, ro.last_order_date
FROM users u
JOIN recent_orders ro ON ro.user_id = u.id;
```

<details>
<summary>해설 보기</summary>

이 CTE는 **Materialized**로 처리됩니다.

이유: `GROUP BY`와 집계함수(`MAX`)가 포함되어 있어 Merge 조건을 만족하지 못합니다. Merged로 처리하려면 CTE 내용을 외부 쿼리에 인라인화해야 하는데, `GROUP BY`가 있으면 외부 쿼리의 `JOIN` 조건과 합칠 수 없습니다.

**실제 처리 과정**:
1. `recent_orders` CTE를 한 번 실행 → 결과를 임시 테이블에 저장
2. `users` 테이블과 임시 테이블을 JOIN

**장점**: `recent_orders`를 여러 번 참조해도 GROUP BY 집계는 1번만 실행됩니다.

**EXPLAIN 확인**:
```sql
EXPLAIN SELECT ...
-- id=1: select_type=PRIMARY, users + 임시 테이블 JOIN
-- id=2: select_type=MATERIALIZED (recent_orders CTE)
```

</details>

---

**Q2.** `Created_tmp_disk_tables` 값이 계속 증가하고 있다. 이를 줄이기 위한 방법 2가지를 설명하라.

<details>
<summary>해설 보기</summary>

**방법 1: `tmp_table_size`와 `max_heap_table_size` 증가**

```sql
-- 현재 설정 확인
SHOW VARIABLES LIKE 'tmp_table_size';
SHOW VARIABLES LIKE 'max_heap_table_size';

-- 증가 (세션 또는 글로벌)
SET GLOBAL tmp_table_size = 64 * 1024 * 1024;      -- 64MB
SET GLOBAL max_heap_table_size = 64 * 1024 * 1024;
-- 두 값이 동일해야 함
```
이 방법은 메모리 사용량을 늘리는 트레이드오프가 있습니다.

**방법 2: 임시 테이블이 크게 생성되는 쿼리 개선**

`EXPLAIN`에서 `Using temporary`를 찾아 해당 쿼리의 GROUP BY 결과 건수를 줄이거나, BLOB/TEXT 컬럼이 집계에 포함되어 있으면 제거합니다.

```sql
-- BLOB/TEXT 컬럼을 집계에 포함하지 않도록 변경
-- Before:
SELECT user_id, GROUP_CONCAT(memo) AS memos FROM orders GROUP BY user_id;
-- After:
SELECT user_id, COUNT(*) AS cnt FROM orders GROUP BY user_id;
-- BLOB/TEXT가 없어지면 메모리 임시 테이블 사용 가능
```

비율 모니터링:
```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
-- Created_tmp_disk_tables / Created_tmp_tables > 10%면 개선 필요
```

</details>

---

<div align="center">

**[⬅️ 세미조인 최적화](./02-semijoin-optimization.md)** | **[홈으로 🏠](../README.md)** | **[다음: GROUP BY / ORDER BY 최적화 ➡️](./04-groupby-orderby-optimization.md)**

</div>
