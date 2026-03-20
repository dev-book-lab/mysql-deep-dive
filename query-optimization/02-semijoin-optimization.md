# 세미조인(Semi-Join) 최적화 — IN 서브쿼리가 Join이 되는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Semi-Join이 일반 Inner Join과 다른 점은 무엇인가?
- Optimizer가 IN 서브쿼리를 Semi-Join으로 변환하는 조건은?
- Materialization, FirstMatch, LooseScan, DuplicateWeedout, Table Pullout 각각이 언제 선택되는가?
- EXPLAIN에서 `select_type: MATERIALIZED`와 `Extra: FirstMatch`를 어떻게 읽는가?
- Semi-Join 최적화가 적용되지 않는 서브쿼리 패턴은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 같은 쿼리인데 왜 어떨 때는 빠르고 어떨 때는 느린가

```
운영 이슈:
  개발 환경(소량 데이터)에서는 빠른 IN 서브쿼리가
  운영 환경(대용량)에서 느려지는 현상

  원인:
    개발 환경: Optimizer가 Semi-Join으로 변환 → 빠름
    운영 환경: 통계 차이로 Dependent Subquery 선택 → 느림

  같은 SQL이지만 실행계획이 다름
  → Semi-Join이 언제 적용되는지 알면 예측 가능

핵심:
  Optimizer가 IN 서브쿼리를 어떤 전략으로 처리하는지 알면
  느린 이유를 정확히 파악하고
  힌트로 올바른 전략을 강제할 수 있다
```

---

## 😱 흔한 실수 (Before)

### 1. Semi-Join 변환이 안 되는 패턴을 모름

```sql
-- Before: GROUP BY가 있으면 Semi-Join 불가
SELECT * FROM orders o
WHERE o.user_id IN (
    SELECT user_id FROM order_items GROUP BY user_id HAVING COUNT(*) > 5
);
-- GROUP BY로 인해 Semi-Join 변환 불가 → Dependent Subquery 가능성

-- Before: 서브쿼리에 LIMIT이 있으면 Semi-Join 불가
SELECT * FROM products p
WHERE p.id IN (
    SELECT product_id FROM hot_items ORDER BY rank LIMIT 10
);
-- LIMIT으로 인해 Semi-Join 변환 불가
```

### 2. EXPLAIN의 Semi-Join 관련 힌트를 무시

```sql
-- EXPLAIN 결과를 보고도 어떤 전략인지 모름
-- Extra: FirstMatch(t1) → Semi-Join FirstMatch 전략
-- Extra: Start temporary; End temporary → DuplicateWeedout 전략
-- select_type: MATERIALIZED → Materialization 전략
-- 이 정보를 모르면 최적화 포인트를 찾을 수 없음
```

---

## ✨ 올바른 접근 (After)

### Semi-Join = 중복 없는 Join

```
Semi-Join의 의미:
  일반 Inner Join:
    SELECT u.*, o.*
    FROM users u JOIN orders o ON o.user_id = u.id
    → user 1명이 order 5개면 결과 5 Row (중복 발생)

  Semi-Join:
    SELECT u.*
    FROM users u
    WHERE u.id IN (SELECT o.user_id FROM orders o)
    → user 1명이 order 5개여도 결과 1 Row
    → "orders에 존재하는지" 여부만 중요, 중복 제거 필요

Optimizer 관점:
  IN 서브쿼리의 의미 = Semi-Join
  → 중복 제거를 처리하는 여러 전략 중 하나를 선택
  → 각 전략이 데이터 분포에 따라 성능 차이
```

---

## 🔬 내부 동작 원리

### 1. Semi-Join 변환 조건

```
Semi-Join으로 변환 가능한 조건:

필수 조건:
  ① WHERE/HAVING 절의 IN 또는 = ANY 형태
  ② 서브쿼리에 UNION 없음
  ③ 서브쿼리에 GROUP BY, HAVING 없음
  ④ 서브쿼리에 집계함수(SUM, COUNT 등) 없음
  ⑤ 서브쿼리에 LIMIT 없음
  ⑥ 외부 쿼리가 UNION 내부가 아님
  ⑦ 외부 쿼리에 SELECT 레벨 서브쿼리가 아닌 것

변환 불가 예시:
  IN (SELECT ... GROUP BY ...)     → GROUP BY 있음
  IN (SELECT ... HAVING ...)       → HAVING 있음
  IN (SELECT COUNT(*) FROM ...)    → 집계함수
  IN (SELECT ... LIMIT 5)          → LIMIT 있음
  IN (SELECT ... UNION SELECT ...) → UNION 있음

변환 가능 예시:
  WHERE id IN (SELECT user_id FROM orders WHERE status='PAID')
  WHERE id IN (SELECT DISTINCT user_id FROM orders)  → DISTINCT는 OK
```

### 2. Semi-Join 전략 5가지

```
전략 1: Table Pullout (당겨내기)
  적용 조건:
    서브쿼리 테이블에서 UNIQUE/PRIMARY KEY로 조인 가능
    = 서브쿼리 결과가 반드시 0 또는 1건 (중복 없음 보장)

  동작:
    서브쿼리를 완전히 외부 쿼리로 당겨서 일반 JOIN으로 변환
    EXPLAIN에서 서브쿼리 없어지고 일반 JOIN으로 표시됨

  예시:
    WHERE order_id IN (SELECT id FROM orders WHERE status='PAID')
    → orders.id가 PRIMARY KEY → 결과 중복 없음 보장
    → 일반 JOIN으로 변환 가능, 가장 빠름

전략 2: FirstMatch (첫 번째 매칭)
  적용 조건:
    서브쿼리 테이블을 Driving Table로 사용하기 어려울 때
    외부 테이블을 먼저 스캔하는 것이 유리할 때

  동작:
    외부 테이블의 각 Row에 대해 서브쿼리 테이블에서 첫 번째 매칭 Row를 찾으면 즉시 다음 Row로 이동
    Short-circuit 평가로 중복 제거

  EXPLAIN Extra:
    FirstMatch(subquery_table)

  예시:
    users 테이블이 작고 orders가 클 때
    → users 각 Row에 대해 orders에서 첫 매칭만 찾으면 됨

전략 3: LooseScan (느슨한 스캔)
  적용 조건:
    서브쿼리 테이블을 인덱스를 통해 스캔할 수 있을 때
    인덱스의 첫 번째 컬럼이 서브쿼리의 GROUP BY/DISTINCT 역할

  동작:
    인덱스의 각 키 값에서 첫 번째 Row만 읽음 (느슨하게 스캔)
    → 인덱스 기반으로 자연스럽게 중복 제거

  EXPLAIN Extra:
    LooseScan

  예시:
    WHERE category_id IN (SELECT category_id FROM products)
    products.category_id에 인덱스 있음
    → 인덱스의 각 category_id 값 첫 Row만 읽어 중복 없이 처리

전략 4: Materialization (실체화)
  적용 조건:
    서브쿼리 결과를 임시 테이블로 저장하는 것이 유리할 때
    서브쿼리 결과가 크지 않을 때

  동작:
    1단계: 서브쿼리를 한 번 실행 → 임시 테이블(해시 인덱스 포함)로 저장
    2단계: 외부 테이블과 임시 테이블을 해시 조인

  EXPLAIN:
    select_type: MATERIALIZED
    임시 테이블이 별도 Row로 표시됨

  예시:
    WHERE id IN (SELECT user_id FROM orders WHERE created_at > '2024-01-01')
    orders 서브쿼리 → 임시 테이블 → users와 해시 조인

전략 5: DuplicateWeedout (중복 제거)
  적용 조건:
    다른 전략이 불가능할 때의 후보
    Semi-Join을 일반 Inner Join으로 변환 후 중복 제거

  동작:
    1단계: 일반 Inner Join으로 실행 (중복 포함)
    2단계: 임시 테이블을 사용해 외부 테이블 PK 기준으로 중복 제거

  EXPLAIN Extra:
    Start temporary; End temporary

  성능:
    임시 테이블 + 중복 제거 비용 추가
    다른 전략보다 비용이 높은 편
```

### 3. EXPLAIN에서 Semi-Join 전략 식별

```sql
-- 예시 쿼리
EXPLAIN
SELECT u.id, u.name
FROM users u
WHERE u.id IN (
    SELECT o.user_id
    FROM orders o
    WHERE o.status = 'PAID'
    AND o.created_at >= '2024-01-01'
);

-- Table Pullout:
--   orders가 사라지고 users만 남음 (JOIN으로 변환)
--   id=1, select_type=SIMPLE, type=eq_ref

-- FirstMatch:
--   Extra: FirstMatch(u) 또는 FirstMatch(orders)

-- LooseScan:
--   Extra: LooseScan

-- Materialization:
--   id=1, select_type=PRIMARY
--   id=2, select_type=MATERIALIZED

-- DuplicateWeedout:
--   Extra: Start temporary (시작)
--   Extra: End temporary (종료)

-- 비교: Dependent Subquery (Semi-Join 미적용):
--   id=2, select_type=DEPENDENT SUBQUERY
--   → 반복 실행, 개선 필요
```

### 4. optimizer_switch로 전략 제어

```sql
-- Semi-Join 관련 optimizer_switch 항목
SHOW VARIABLES LIKE 'optimizer_switch'\G
-- semijoin: Semi-Join 전체 활성화/비활성화
-- materialization: Materialization 전략
-- firstmatch: FirstMatch 전략
-- loosescan: LooseScan 전략
-- duplicateweedout: DuplicateWeedout 전략

-- 특정 전략 비활성화 (테스트용)
SET optimizer_switch = 'firstmatch=off';
EXPLAIN SELECT ...; -- FirstMatch 제외한 전략 선택

-- 힌트로 특정 전략 강제
SELECT /*+ SEMIJOIN(MATERIALIZATION) */ u.id
FROM users u
WHERE u.id IN (SELECT o.user_id FROM orders o WHERE o.status = 'PAID');

SELECT /*+ NO_SEMIJOIN(FIRSTMATCH, LOOSESCAN) */ u.id
FROM users u
WHERE u.id IN (SELECT o.user_id FROM orders o WHERE o.status = 'PAID');
-- FirstMatch와 LooseScan을 제외한 나머지 전략으로 선택
```

---

## 💻 실전 실험

### 실험 1: 각 Semi-Join 전략 유도

```sql
-- Table Pullout 유도
-- 조건: 서브쿼리가 UNIQUE/PRIMARY KEY 조인
EXPLAIN
SELECT o.id, o.amount
FROM orders o
WHERE o.user_id IN (
    SELECT u.id FROM users u WHERE u.status = 'VIP'
);
-- users.id = PRIMARY KEY → Table Pullout 가능
-- EXPLAIN에서 서브쿼리가 사라지고 JOIN으로 표시

-- Materialization 유도
-- 조건: 서브쿼리 결과를 임시 테이블로 저장
SET optimizer_switch = 'firstmatch=off,loosescan=off';
EXPLAIN
SELECT u.id
FROM users u
WHERE u.id IN (
    SELECT o.user_id FROM orders o WHERE o.status = 'PAID'
);
-- select_type: MATERIALIZED 확인

-- DuplicateWeedout 유도
SET optimizer_switch = 'firstmatch=off,loosescan=off,materialization=off';
EXPLAIN
SELECT u.id
FROM users u
WHERE u.id IN (
    SELECT o.user_id FROM orders o WHERE o.status = 'PAID'
);
-- Extra: Start temporary / End temporary 확인

SET optimizer_switch = DEFAULT;
```

### 실험 2: Semi-Join 변환 불가 패턴 확인

```sql
-- GROUP BY → Semi-Join 불가
EXPLAIN
SELECT u.id
FROM users u
WHERE u.id IN (
    SELECT user_id FROM orders GROUP BY user_id HAVING COUNT(*) > 3
);
-- select_type: DEPENDENT SUBQUERY 또는 SUBQUERY (Semi-Join 아님)

-- LIMIT → Semi-Join 불가
EXPLAIN
SELECT u.id
FROM users u
WHERE u.id IN (
    SELECT user_id FROM orders WHERE status='PAID' ORDER BY created_at DESC LIMIT 10
);
-- 마찬가지로 Semi-Join 미적용 확인
```

### 실험 3: 전략별 실행 시간 비교

```sql
-- 100만 건 orders, 1만 건 users 기준
-- 각 전략 강제 후 실행 시간 비교

-- Materialization
SET optimizer_switch = 'semijoin=on,materialization=on,firstmatch=off,loosescan=off';
SET @t = SYSDATE(6);
SELECT COUNT(*) FROM users u WHERE u.id IN (SELECT o.user_id FROM orders o WHERE o.status='PAID');
SELECT CONCAT('Materialization: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms') AS result;

-- DuplicateWeedout
SET optimizer_switch = 'semijoin=on,materialization=off,firstmatch=off,loosescan=off';
SET @t = SYSDATE(6);
SELECT COUNT(*) FROM users u WHERE u.id IN (SELECT o.user_id FROM orders o WHERE o.status='PAID');
SELECT CONCAT('DuplicateWeedout: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms') AS result;

SET optimizer_switch = DEFAULT;
```

---

## 📊 성능/비용 비교

```
전략별 성능 특성 (users 1만, orders 100만, status='PAID' 30만):

Table Pullout (최적):
  적용 조건: UNIQUE/PK 조인만 가능
  비용: 일반 JOIN과 동일
  예상 시간: 0.05~0.2초

LooseScan (빠름):
  적용 조건: 서브쿼리 테이블에 적절한 인덱스 필요
  비용: 인덱스 스캔 + 첫 Row만 읽음
  예상 시간: 0.1~0.5초

FirstMatch (보통):
  적용 조건: 외부 테이블이 작을 때
  비용: Short-circuit으로 조기 종료 가능
  예상 시간: 0.3~1초

Materialization (보통):
  적용 조건: 서브쿼리 결과가 메모리에 적합
  비용: 임시 테이블 생성 + 해시 조인
  예상 시간: 0.2~0.8초
  메모리 사용: 서브쿼리 결과 크기만큼

DuplicateWeedout (느림):
  적용 조건: 다른 전략 불가 시
  비용: JOIN + 임시 테이블로 중복 제거
  예상 시간: 0.5~2초

Dependent Subquery (최악):
  Semi-Join 미적용
  예상 시간: 10~30초
```

---

## ⚖️ 트레이드오프

```
Semi-Join 전략 선택 트레이드오프:

Materialization:
  ✅ 서브쿼리를 한 번만 실행 → 반복 없음
  ✅ 해시 인덱스로 빠른 조인
  ❌ 임시 테이블 메모리 사용
  ❌ 서브쿼리 결과가 너무 크면 디스크 임시 테이블 → 느림

FirstMatch:
  ✅ Short-circuit → 조기 종료
  ✅ 추가 임시 테이블 불필요
  ❌ 외부 테이블이 클 때 비효율
  ❌ 매 외부 Row마다 서브쿼리 탐색

LooseScan:
  ✅ 인덱스 기반 효율적 중복 제거
  ❌ 서브쿼리 테이블에 적절한 인덱스 필요

optimizer_switch 조작:
  ✅ 문제 있는 전략을 특정 쿼리에서 비활성화 가능
  ❌ 전역 설정 변경 시 다른 쿼리에 영향
  → 쿼리 힌트(/*+ ... */) 또는 세션 수준 SET 권장

Semi-Join 변환 불가 서브쿼리:
  GROUP BY / HAVING / 집계함수 / LIMIT 포함 시
  → 직접 JOIN + DISTINCT로 리팩토링하거나
  → CTE로 먼저 실체화 후 JOIN
```

---

## 📌 핵심 정리

```
Semi-Join 핵심:

Semi-Join = IN 서브쿼리를 중복 없는 Join으로 처리하는 최적화

변환 조건:
  WHERE/HAVING의 IN (서브쿼리)
  서브쿼리에 UNION/GROUP BY/HAVING/집계함수/LIMIT 없음

전략 5가지:
  Table Pullout  → UNIQUE/PK 조인 → 일반 JOIN으로 변환 (최적)
  FirstMatch     → Short-circuit 첫 매칭만 → Extra: FirstMatch(t)
  LooseScan      → 인덱스로 중복 없이 스캔 → Extra: LooseScan
  Materialization → 임시 테이블 실체화 → select_type: MATERIALIZED
  DuplicateWeedout → JOIN 후 임시 테이블로 중복 제거 → Extra: Start/End temporary

진단:
  EXPLAIN select_type = DEPENDENT SUBQUERY → Semi-Join 미적용 → 개선 필요
  Semi-Join 적용 시 위 Extra 값 중 하나가 나타남

강제 방법:
  /*+ SEMIJOIN(MATERIALIZATION) */ 힌트로 특정 전략 지정
  SET optimizer_switch = 'firstmatch=off' 으로 특정 전략 비활성화
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 쿼리에 Semi-Join이 적용되지 않는 이유를 설명하고, 동일한 결과를 내면서 Semi-Join이 적용될 수 있도록 리팩토링하라.

```sql
SELECT c.id, c.name
FROM customers c
WHERE c.id IN (
    SELECT o.customer_id
    FROM orders o
    GROUP BY o.customer_id
    HAVING COUNT(*) >= 3
);
```

<details>
<summary>해설 보기</summary>

**Semi-Join 미적용 이유**: 서브쿼리에 `GROUP BY`와 `HAVING`이 있어 Semi-Join 변환 조건을 만족하지 못합니다.

**리팩토링 방법 1 — CTE로 먼저 실체화:**
```sql
WITH frequent_customers AS (
    SELECT customer_id
    FROM orders
    GROUP BY customer_id
    HAVING COUNT(*) >= 3
)
SELECT c.id, c.name
FROM customers c
WHERE c.id IN (SELECT customer_id FROM frequent_customers);
-- CTE 결과는 Materialized임시 테이블 → 두 번째 IN은 단순 IN 조회
```

**리팩토링 방법 2 — JOIN + 집계:**
```sql
SELECT DISTINCT c.id, c.name
FROM customers c
INNER JOIN (
    SELECT customer_id
    FROM orders
    GROUP BY customer_id
    HAVING COUNT(*) >= 3
) freq ON freq.customer_id = c.id;
-- FROM 절 파생 테이블로 먼저 집계 → INNER JOIN
```

**리팩토링 방법 3 — EXISTS:**
```sql
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id
    GROUP BY o.customer_id
    HAVING COUNT(*) >= 3
);
-- EXISTS는 Semi-Join과 다르게 처리되지만, 의미는 동일
```

각 방법에 대해 `EXPLAIN ANALYZE`로 실행 비용을 비교하는 것이 중요합니다.

</details>

---

**Q2.** EXPLAIN에서 `Extra: Start temporary; End temporary`가 보인다. 이것이 어떤 Semi-Join 전략이고, 이 전략이 선택된 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

`Start temporary` / `End temporary`는 **DuplicateWeedout** 전략이 적용된 것입니다.

**동작 방식**: 서브쿼리를 일반 Inner Join으로 변환한 후, Join 결과의 중복을 임시 테이블로 제거합니다. `Start temporary`와 `End temporary` 사이의 테이블들이 Join되고, 이 구간의 결과를 임시 테이블에 저장하며 외부 쿼리 Row의 PK 기준으로 중복을 제거합니다.

**이 전략이 선택되는 이유**: Optimizer가 Table Pullout, FirstMatch, LooseScan, Materialization을 모두 적용할 수 없거나 비용이 더 높다고 판단할 때 DuplicateWeedout이 선택됩니다. 예를 들어 서브쿼리 테이블에 적절한 인덱스가 없거나, 서브쿼리 결과가 크기 때문에 Materialization 비용이 높을 때 선택됩니다.

**개선 포인트**: DuplicateWeedout이 보이면 서브쿼리 테이블에 인덱스를 추가해 LooseScan이나 FirstMatch가 동작하도록 유도하거나, 쿼리 힌트로 Materialization을 강제하는 것을 검토합니다.

</details>

---

<div align="center">

**[⬅️ 서브쿼리 최적화](./01-subquery-optimization.md)** | **[홈으로 🏠](../README.md)** | **[다음: 파생 테이블과 CTE ➡️](./03-derived-table-cte.md)**

</div>
