# 서브쿼리 최적화 — IN/EXISTS/스칼라 서브쿼리의 내부 변환

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- IN 서브쿼리가 Dependent Subquery로 처리되면 왜 O(n²)이 되는가?
- EXISTS와 IN 중 무엇이 빠른지는 어떤 기준으로 판단하는가?
- 스칼라 서브쿼리가 매 Row마다 실행되는 조건은 무엇이고, 언제 캐시가 동작하는가?
- EXPLAIN에서 select_type: DEPENDENT SUBQUERY를 보면 무엇을 의심해야 하는가?
- 서브쿼리를 Join으로 바꾸는 것이 항상 옳은가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### "이 쿼리가 왜 이렇게 느리지?" — 범인은 서브쿼리

```
실제 운영 이슈:
  주문 목록 API 응답 시간이 갑자기 3초로 늘어남
  EXPLAIN 확인:
    id | select_type        | type | rows   | Extra
    1  | PRIMARY            | ALL  | 50000  |
    2  | DEPENDENT SUBQUERY | ref  | 1      | Using index

  "rows=1이면 빠른 거 아닌가요?" → 아님.
  DEPENDENT SUBQUERY는 외부 쿼리의 Row 수(50,000)만큼 반복 실행됨
  → 실제 실행 = 50,000 × 1회 = 서브쿼리 50,000번 실행!

결론:
  EXPLAIN의 rows는 한 번의 서브쿼리 실행 비용
  실제 비용은 (외부 rows) × (서브쿼리 실행 횟수)
  DEPENDENT 패턴이면 50,000 × 1 = 총 50,000번
```

서브쿼리가 어떻게 처리되는지 모르면, EXPLAIN을 보고도 느린 이유를 파악할 수 없다.

---

## 😱 흔한 실수 (Before)

### 1. IN 서브쿼리를 무심코 사용

```sql
-- Before: 최근 주문이 있는 사용자 목록
SELECT u.id, u.name
FROM users u
WHERE u.id IN (
    SELECT o.user_id
    FROM orders o
    WHERE o.created_at >= '2024-01-01'
);

-- 기대: orders 한 번 조회 후 IN 조건 적용
-- 실제: Dependent Subquery일 경우 users의 각 Row마다 서브쿼리 실행
-- users가 10만 건이면 서브쿼리 10만 번 실행
```

### 2. 스칼라 서브쿼리를 SELECT에 남발

```sql
-- Before: N+1과 동일한 패턴
SELECT
    o.id,
    o.amount,
    (SELECT u.name FROM users u WHERE u.id = o.user_id) AS user_name,
    (SELECT COUNT(*) FROM order_items oi WHERE oi.order_id = o.id) AS item_count,
    (SELECT SUM(oi.price) FROM order_items oi WHERE oi.order_id = o.id) AS total_price
FROM orders o
WHERE o.created_at >= '2024-01-01';

-- orders 10,000건이면 서브쿼리 총 30,000번 실행
-- = JPA N+1 문제와 구조적으로 동일
```

### 3. NOT IN의 NULL 함정 무시

```sql
-- Before: NULL이 포함될 수 있는데 NOT IN 사용
SELECT id FROM orders
WHERE user_id NOT IN (SELECT id FROM suspended_users);

-- suspended_users.id에 NULL이 하나라도 있으면 결과가 빈 집합
-- "왜 데이터가 안 나오지?" 원인 파악 어려움
```

---

## ✨ 올바른 접근 (After)

### 서브쿼리 종류별 처리 방식을 이해하고 선택

```
서브쿼리 처리 방식 3가지:

1. Non-Correlated Subquery (비상관):
   서브쿼리가 외부 쿼리와 무관 → 한 번만 실행
   WHERE user_id IN (SELECT id FROM vip_users)
   → vip_users 서브쿼리 1번 실행 후 결과 메모리 유지

2. Correlated Subquery (상관):
   서브쿼리가 외부 Row를 참조 → 외부 Row마다 실행
   WHERE EXISTS (SELECT 1 FROM refunds r WHERE r.order_id = o.id)
   → orders의 각 Row마다 refunds 서브쿼리 실행
   → 단, Optimizer가 Semi-Join으로 변환하기도 함

3. Scalar Subquery (스칼라):
   SELECT 절에서 단일 값 반환
   캐시 동작 여부가 성능을 결정
   → 외부 참조값 중복이 많으면 캐시 효과, 없으면 N번 실행
```

---

## 🔬 내부 동작 원리

### 1. Dependent Subquery가 O(n²)이 되는 구조

```
쿼리:
  SELECT * FROM users u
  WHERE u.id IN (
      SELECT o.user_id FROM orders o WHERE o.status = 'PAID'
  );

Optimizer가 Non-Correlated로 처리하면 (이상적):
  1단계: orders에서 status='PAID'인 user_id 목록 → 1번 실행
  2단계: users에서 해당 id들을 IN으로 필터링
  실행 횟수: 2번

Optimizer가 Dependent Subquery로 처리하면:
  users의 각 Row에 대해:
    "이 user.id가 orders에 status='PAID'로 존재하는가?" 확인
  실행 횟수: users 건수 × 1

  users 10만 건이면 → 서브쿼리 10만 번 실행

EXPLAIN에서 확인:
  id=2, select_type=DEPENDENT SUBQUERY → 반복 실행 (느림)
  id=2, select_type=SUBQUERY           → 독립 실행 (1번)
  id=2, select_type=MATERIALIZED       → 임시테이블 한 번 (중간)
```

### 2. Optimizer의 IN 서브쿼리 변환 판단

```
MySQL Optimizer 처리 순서:

단계 1: Semi-Join 변환 가능한가?
  조건:
    - WHERE/HAVING의 IN/= ANY 위치
    - 서브쿼리에 UNION, GROUP BY, HAVING, 집계함수 없음
    - 외부 쿼리가 UNION 내부가 아님
  가능하면 Semi-Join 전략으로 처리 (다음 문서에서 상세)

단계 2: Semi-Join 불가 → Materialization 시도
  서브쿼리 결과를 임시 테이블에 한 번 저장
  optimizer_switch: 'materialization=on' 필요

단계 3: Materialization도 불가 → Dependent Subquery (최악)
  외부 Row마다 서브쿼리 반복 실행

select_type 값 해석:
  SUBQUERY          → 독립, 1번 실행
  DEPENDENT SUBQUERY → 외부에 의존, 반복 실행
  MATERIALIZED      → 임시 테이블로 실체화
  DERIVED           → FROM 절 파생 테이블
```

### 3. EXISTS vs IN — 실행 비용 비교 원리

```
IN 처리 방식:
  WHERE a.x IN (SELECT b.x FROM B WHERE ...)

  B가 작으면 → B 먼저 실행, 결과로 A 필터 (Materialization)
  서브쿼리가 Non-Correlated → 한 번만 실행 → 빠름
  서브쿼리에 외부 참조 → Dependent → 반복 실행 → 느림

EXISTS 처리 방식:
  WHERE EXISTS (SELECT 1 FROM B WHERE B.x = A.x)

  항상 상관 서브쿼리 → A의 각 Row마다 B 확인
  하지만: 조건 일치 첫 Row 발견 즉시 중단 (Short-circuit)
  → B에서 결과가 빨리 찾힐수록 유리

선택 기준:
  서브쿼리가 Non-Correlated가 될 수 있으면 → IN 선호
    (Materialization으로 처리될 가능성)
  조기 종료가 중요한 경우 → EXISTS
  NULL 안전성이 중요한 경우 → NOT EXISTS 선택 (NOT IN 위험)
```

### 4. 스칼라 서브쿼리 캐시 동작

```
스칼라 서브쿼리 캐시:
  SELECT (SELECT name FROM dept WHERE dept.id = e.dept_id) AS dept_name
  FROM employees e;

  캐시 동작 조건: 동일한 외부 참조값(dept_id)이 반복될 때
    dept_id = 1인 employees가 100명 → 서브쿼리 1번 + 캐시 히트 99번
    dept_id = 2인 employees가 200명 → 서브쿼리 1번 + 캐시 히트 199번

  캐시 동작 안 하는 경우: 각 Row의 dept_id가 모두 다를 때
    → N번 실행 (employees 건수만큼)

실제 실행 횟수 확인:
  EXPLAIN ANALYZE의 "actual loops" 값 확인
  loops = 1000이면 서브쿼리 1,000번 실행됨

캐시보다 나은 방법:
  스칼라 서브쿼리 → LEFT JOIN으로 변환
  SELECT e.*, d.name AS dept_name
  FROM employees e
  LEFT JOIN dept d ON d.id = e.dept_id;
  → 반복 없음, 인덱스 조인으로 처리
```

### 5. NOT IN과 NULL의 함정

```
NOT IN에서 NULL이 있을 때:
  SELECT * FROM orders
  WHERE user_id NOT IN (SELECT id FROM deleted_users);

  deleted_users에 NULL이 하나라도 있으면:
    5 NOT IN (1, 2, NULL) → NULL (알 수 없음) → WHERE 불통과
    → 결과 전체가 빈 집합!

재현:
  SELECT 1 WHERE 5 NOT IN (1, 2, NULL);
  → 결과 없음

안전한 대안:
  NOT EXISTS:
    SELECT * FROM orders o
    WHERE NOT EXISTS (
        SELECT 1 FROM deleted_users d WHERE d.id = o.user_id
    );
    → NULL 처리 문제 없음

  또는 IS NOT NULL 조합:
    WHERE user_id NOT IN (
        SELECT id FROM deleted_users WHERE id IS NOT NULL
    );
```

---

## 💻 실전 실험

### 실험 1: select_type 차이 확인

```sql
-- 테이블 준비
CREATE TABLE exp_users (
    id     BIGINT AUTO_INCREMENT PRIMARY KEY,
    name   VARCHAR(50),
    status VARCHAR(20) DEFAULT 'ACTIVE'
) ENGINE=InnoDB;

CREATE TABLE exp_orders (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    status     VARCHAR(20),
    created_at DATETIME DEFAULT NOW(),
    INDEX idx_user_id (user_id),
    INDEX idx_status  (status)
) ENGINE=InnoDB;

-- 데이터 삽입 (생략, 각자 적절히 삽입)

-- Dependent vs Materialized 비교
EXPLAIN
SELECT u.id, u.name
FROM exp_users u
WHERE u.id IN (
    SELECT o.user_id FROM exp_orders o WHERE o.status = 'PAID'
);
-- select_type 확인: SUBQUERY, MATERIALIZED, DEPENDENT SUBQUERY 중 무엇?

-- Materialization 강제 비활성화 후 비교
SET optimizer_switch = 'materialization=off,semijoin=off';
EXPLAIN
SELECT u.id, u.name
FROM exp_users u
WHERE u.id IN (
    SELECT o.user_id FROM exp_orders o WHERE o.status = 'PAID'
);
-- DEPENDENT SUBQUERY로 변경됨을 확인

SET optimizer_switch = DEFAULT;
```

### 실험 2: 스칼라 서브쿼리 실행 횟수 측정

```sql
-- EXPLAIN ANALYZE로 실제 loops 확인
EXPLAIN ANALYZE
SELECT
    o.id,
    (SELECT u.name FROM exp_users u WHERE u.id = o.user_id) AS user_name
FROM exp_orders o
WHERE o.status = 'PAID'
LIMIT 1000;

-- 출력 예시:
-- -> Select #2 (actual time=0.021..0.021 rows=1 loops=1000)
-- loops=1000 → 서브쿼리 1,000번 실행

-- JOIN 변환 후 비교
EXPLAIN ANALYZE
SELECT
    o.id,
    u.name AS user_name
FROM exp_orders o
INNER JOIN exp_users u ON u.id = o.user_id
WHERE o.status = 'PAID'
LIMIT 1000;
-- loops 값이 크게 줄어듦 확인
```

### 실험 3: NOT IN NULL 함정

```sql
CREATE TABLE null_test (id BIGINT);
INSERT INTO null_test VALUES (1), (2), (NULL);

-- 예상: 3, 4, 5가 반환
SELECT val FROM (SELECT 3 val UNION SELECT 4 UNION SELECT 5) v
WHERE val NOT IN (SELECT id FROM null_test);
-- 실제: 결과 없음 (NULL 때문)

-- 안전한 NOT EXISTS
SELECT val FROM (SELECT 3 val UNION SELECT 4 UNION SELECT 5) v
WHERE NOT EXISTS (SELECT 1 FROM null_test WHERE id = val);
-- 결과: 3, 4, 5

DROP TABLE null_test;
```

### 실험 4: EXISTS Short-circuit 확인

```sql
-- LIMIT 1로 첫 Row 발견 시 즉시 중단
EXPLAIN ANALYZE
SELECT u.id
FROM exp_users u
WHERE EXISTS (
    SELECT 1 FROM exp_orders o WHERE o.user_id = u.id LIMIT 1
);

-- EXISTS는 내부적으로 Short-circuit 평가
-- vs COUNT(*) > 0 방식은 전체 집계 필요 → EXISTS가 유리
```

---

## 📊 성능/비용 비교

```
테이블 규모: users 10만 건, orders 100만 건

Dependent Subquery (최악):
  실행 횟수: 10만 × 서브쿼리 1회 = 10만 번
  예상 시간: 10~30초

Materialization (중간):
  서브쿼리를 임시 테이블에 한 번 저장 후 해시 조인
  예상 시간: 0.5~2초

Semi-Join 변환 (최적, 다음 문서 참고):
  두 테이블을 효율적인 Join으로 처리
  예상 시간: 0.1~0.5초

스칼라 서브쿼리 vs JOIN:
  스칼라 서브쿼리 (캐시 미스):
    10,000건 × 서브쿼리 1회 = 10,000번 → 2~5초
  JOIN 변환:
    단일 조인 연산 → 0.05~0.2초
  캐시 히트율이 높으면 (동일 참조값 반복):
    스칼라 서브쿼리도 수용 가능한 성능
```

---

## ⚖️ 트레이드오프

```
서브쿼리 유지 vs JOIN 변환:

서브쿼리:
  ✅ 가독성 높음, 의도 명확
  ✅ Optimizer가 자동 최적화하는 경우도 있음
  ❌ DEPENDENT SUBQUERY 처리 시 심각한 성능 저하
  ❌ EXPLAIN 없이는 실제 처리 방식 알 수 없음

JOIN 변환:
  ✅ 실행계획을 직접 제어 가능
  ✅ 일반적으로 더 빠른 실행계획 보장
  ❌ 중복 Row 주의 (DISTINCT 필요한 경우)
  ❌ 서브쿼리보다 의도 파악이 덜 직관적일 수 있음

IN vs EXISTS:
  NOT IN → NULL 위험 항상 존재 → NOT EXISTS로 교체 권장
  IN Materialization 동작 시 → IN도 충분히 빠름
  Short-circuit 필요 시 → EXISTS 유리

실무 기준:
  EXPLAIN에서 DEPENDENT SUBQUERY 발견 → 즉시 개선 대상
  SELECT 절 스칼라 서브쿼리 → LEFT JOIN 변환 검토
  NOT IN → NOT EXISTS 또는 IS NOT NULL 조합으로 교체
```

---

## 📌 핵심 정리

```
서브쿼리 최적화 핵심:

EXPLAIN select_type으로 처리 방식 파악:
  DEPENDENT SUBQUERY → 외부 Row마다 반복 실행 → 반드시 개선
  SUBQUERY          → 독립 실행, 1번만 → 빠름
  MATERIALIZED      → 임시 테이블 한 번 → 중간
  DERIVED           → FROM 절 파생 테이블 (03문서 참고)

IN vs EXISTS 선택 기준:
  NOT IN → NULL 위험 → NOT EXISTS 또는 IS NOT NULL 조합 사용
  IN Materialization 동작 → IN도 빠름
  Short-circuit 필요 → EXISTS

스칼라 서브쿼리 SELECT 절 금지:
  외부 Row마다 실행 = N+1과 동일
  LEFT JOIN으로 변환이 항상 더 안전하고 빠름

진단 도구:
  EXPLAIN: select_type 컬럼으로 처리 방식 확인
  EXPLAIN ANALYZE: loops 값으로 실제 실행 횟수 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 쿼리의 EXPLAIN에서 `select_type: DEPENDENT SUBQUERY`가 나왔다. 어떻게 개선하는가?

```sql
SELECT p.id, p.name
FROM products p
WHERE p.category_id IN (
    SELECT c.id FROM categories c
    WHERE c.parent_id = 5 AND c.is_active = 1
);
```

<details>
<summary>해설 보기</summary>

이 쿼리는 외부 참조가 없으므로 원래 `SUBQUERY`나 `MATERIALIZED`로 처리되어야 합니다. `DEPENDENT SUBQUERY`가 나왔다면 Optimizer가 비효율적인 경로를 선택한 것입니다.

**개선 방법 1 — JOIN으로 변환:**
```sql
SELECT DISTINCT p.id, p.name
FROM products p
INNER JOIN categories c ON c.id = p.category_id
WHERE c.parent_id = 5 AND c.is_active = 1;
```
`DISTINCT`는 하나의 product가 여러 category에 속할 수 없는 구조라면 생략 가능합니다.

**개선 방법 2 — Optimizer Hint로 Materialization 강제:**
```sql
SELECT p.id, p.name
FROM products p
WHERE p.category_id IN (
    SELECT /*+ SUBQUERY(MATERIALIZATION) */ c.id
    FROM categories c
    WHERE c.parent_id = 5 AND c.is_active = 1
);
```

어느 방법이든 `EXPLAIN ANALYZE`의 `loops` 값으로 개선 전후를 수치로 비교합니다.

</details>

---

**Q2.** 다음 두 쿼리의 결과가 다를 수 있다. 이유를 설명하라.

```sql
-- 쿼리 A
SELECT id FROM orders WHERE user_id NOT IN (SELECT id FROM suspended_users);

-- 쿼리 B
SELECT id FROM orders
WHERE NOT EXISTS (
    SELECT 1 FROM suspended_users WHERE id = orders.user_id
);
```

<details>
<summary>해설 보기</summary>

`suspended_users.id`에 `NULL`이 존재하면 두 쿼리의 결과가 달라집니다.

**쿼리 A (NOT IN)**: `NULL`이 포함된 집합과의 비교는 `NULL` → WHERE 조건 불통과 → `suspended_users`에 `id = NULL`인 Row가 하나라도 있으면 결과 전체가 빈 집합이 됩니다.

**쿼리 B (NOT EXISTS)**: `WHERE id = orders.user_id`에서 `NULL = 값`은 매칭 실패 → EXISTS 결과 FALSE → NOT EXISTS → TRUE → NULL이 있어도 정상 동작합니다.

**실무 교훈**: FK 컬럼이 NOT NULL임이 보장되지 않는 한 `NOT IN` 대신 항상 `NOT EXISTS`를 사용합니다. 또는 LEFT JOIN + IS NULL 패턴도 안전합니다:
```sql
SELECT o.id
FROM orders o
LEFT JOIN suspended_users s ON s.id = o.user_id
WHERE s.id IS NULL;
```

</details>

---

**Q3.** 스칼라 서브쿼리 캐시가 효과적인 경우와 그렇지 않은 경우를 각각 예를 들어 설명하라.

<details>
<summary>해설 보기</summary>

**효과적인 경우** — 외부 참조값의 중복이 많을 때:
```sql
-- order_items 100만 건, product 종류 1,000개
SELECT oi.id,
       (SELECT p.name FROM products p WHERE p.id = oi.product_id) AS product_name
FROM order_items oi;
-- product_id 1,000가지 → 서브쿼리 최대 1,000번 실행, 캐시 히트 999,000번
```

**효과적이지 않은 경우** — 외부 참조값이 모두 다를 때:
```sql
-- orders 10만 건, 각 order의 user_id가 모두 다름
SELECT o.id,
       (SELECT u.name FROM users u WHERE u.id = o.user_id) AS user_name
FROM orders o;
-- user_id 중복 없음 → 캐시 히트 없음 → 10만 번 실행
-- LEFT JOIN으로 변환이 훨씬 효율적
```

`EXPLAIN ANALYZE`의 `loops` 값을 확인합니다. `loops`가 외부 Row 수보다 훨씬 작으면 캐시가 잘 동작하는 것이고, `loops`가 외부 Row 수와 같으면 캐시 효과 없음 → JOIN 변환을 고려합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 세미조인 최적화 ➡️](./02-semijoin-optimization.md)**

</div>
