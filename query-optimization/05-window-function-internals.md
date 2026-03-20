# 윈도우 함수 실행 원리와 성능 특성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 윈도우 함수는 내부적으로 어떤 과정을 거쳐 실행되는가?
- PARTITION BY와 ORDER BY가 윈도우 함수 실행에 미치는 영향은?
- ROW_NUMBER(), RANK(), DENSE_RANK()의 비용 차이는?
- 같은 결과를 GROUP BY로도 얻을 수 있을 때, 윈도우 함수와 성능 차이는?
- EXPLAIN에서 `Using temporary`와 윈도우 함수의 관계는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 윈도우 함수를 잘못 쓰면 예상 밖의 임시 테이블이 생긴다

```
실제 이슈:
  "사용자별 최근 3개 주문" 쿼리
  ROW_NUMBER()로 구현 → 개발 환경 빠름
  운영 환경 100만 건 → 갑자기 느림

  원인:
    PARTITION BY user_id → user_id별로 정렬된 임시 테이블 생성
    ORDER BY created_at DESC → 전체를 정렬 후 처리
    → 100만 건 전체를 임시 테이블에 올린 후 처리
    → 예상보다 큰 임시 테이블 생성

  해결:
    WHERE user_id = ? 로 필터 후 윈도우 함수 적용
    또는 서브쿼리로 대상 범위 먼저 줄이기
```

---

## 😱 흔한 실수 (Before)

### 1. 필터 없이 대용량 테이블에 윈도우 함수 적용

```sql
-- Before: 전체 테이블에 ROW_NUMBER
SELECT
    user_id,
    order_id,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM orders;
-- 100만 건 전체를 임시 테이블에 올려 user_id별 정렬
-- → 메모리/디스크 임시 테이블 생성
-- → 결과: 쿼리 자체는 느리지만 WHERE rn <= 3을 바깥에서 감싸야 함
```

### 2. 윈도우 함수 결과로 필터링 시 중첩 쿼리 구조 모름

```sql
-- 잘못된 기대: WHERE 절에서 직접 사용 불가
SELECT user_id, order_id
FROM orders
WHERE ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) <= 3;
-- 오류: Window function cannot be used in WHERE clause
```

### 3. 동일 파티션에 여러 윈도우 함수를 각각 다른 ORDER BY로 적용

```sql
-- Before: 같은 파티션인데 ORDER BY가 달라 두 번 정렬
SELECT
    user_id,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn_date,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn_amount
FROM orders;
-- 두 개의 독립적인 정렬 + 임시 테이블
```

---

## ✨ 올바른 접근 (After)

```
윈도우 함수의 실행 모델:

1. FROM + WHERE + JOIN 처리 (일반 필터링)
2. GROUP BY + HAVING (집계가 있는 경우)
3. 윈도우 함수 처리:
   a. PARTITION BY: 파티션별로 행 그룹화
   b. ORDER BY: 각 파티션 내 정렬
   c. Frame: 각 행의 윈도우 범위 결정
   d. 함수 계산: 각 행에 대해 결과 계산
4. ORDER BY (전체 결과 정렬)
5. LIMIT / OFFSET

핵심: 윈도우 함수는 WHERE보다 늦게 실행됨
  → WHERE로 먼저 범위를 줄이면 윈도우 함수 비용도 줄어듦
  → 하지만 WHERE에서 윈도우 함수 결과를 직접 필터링 불가
  → 서브쿼리나 CTE로 감싸서 외부에서 필터링
```

---

## 🔬 내부 동작 원리

### 1. 윈도우 함수 실행 단계

```
내부 처리 과정:

단계 1: 기본 쿼리 실행
  FROM, WHERE, JOIN, GROUP BY, HAVING 처리
  → 결과 행 집합 준비

단계 2: 임시 버퍼에 적재
  윈도우 함수 처리를 위해 현재까지 결과를 버퍼에 저장
  → EXPLAIN Extra: Using temporary 발생 원인

단계 3: PARTITION BY 처리
  파티션 키(user_id 등)로 행 그룹화
  내부적으로 파티션 키 기준 정렬 필요

단계 4: ORDER BY 처리
  각 파티션 내에서 ORDER BY 키 기준 정렬

단계 5: Frame 적용
  ROWS BETWEEN / RANGE BETWEEN 등의 프레임 정의에 따라
  각 행의 "윈도우" 범위 결정

단계 6: 함수 계산
  각 행에 대해 해당 윈도우 내의 값으로 함수 계산
  SUM, AVG: 프레임 내 누적 계산
  ROW_NUMBER, RANK: 순서 기반 번호 부여

비용 요인:
  파티션 수 × 파티션당 행 수 = 전체 행 수
  정렬 비용: O(n log n) (n = 파티션 내 행 수)
  임시 테이블 크기: 결과 행 전체
```

### 2. ROW_NUMBER vs RANK vs DENSE_RANK

```sql
-- 비교 예시
SELECT
    user_id,
    amount,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS row_num,
    RANK()       OVER (PARTITION BY user_id ORDER BY amount DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS dense_rnk
FROM orders
WHERE user_id = 1;

-- 예시 데이터: amount = 1000, 800, 800, 600
-- ROW_NUMBER: 1, 2, 3, 4 (동일 값도 다른 번호, 번호 연속)
-- RANK:       1, 2, 2, 4 (동일 값 같은 번호, 번호 건너뜀)
-- DENSE_RANK: 1, 2, 2, 3 (동일 값 같은 번호, 번호 연속)

-- 비용 차이:
-- ROW_NUMBER: 정렬만 필요, 동등 비교 없음 → 가장 빠름
-- RANK: 이전 행과 현재 행의 값 비교 필요
-- DENSE_RANK: RANK + 별도 밀집 번호 계산
-- 실제 차이는 미미하지만 ROW_NUMBER가 가장 단순
```

### 3. 동일 PARTITION BY, 다른 ORDER BY 최적화

```sql
-- 최적화 전: 각 윈도우 함수가 독립 정렬
SELECT
    user_id,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn_date,
    SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM orders;
-- 파티션은 같지만 ORDER BY가 달라 두 번 정렬

-- MySQL Optimizer의 윈도우 함수 최적화:
-- 동일 파티션 + 동일 ORDER BY → 한 번 정렬로 처리 가능
SELECT
    user_id,
    ROW_NUMBER() OVER w AS rn,
    SUM(amount) OVER w AS running_total
FROM orders
WINDOW w AS (PARTITION BY user_id ORDER BY created_at DESC);
-- WINDOW 절로 재사용 명시 → 한 번만 정렬

-- EXPLAIN에서 윈도우 함수 확인
EXPLAIN FORMAT=JSON
SELECT ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM orders WHERE status = 'PAID';
-- "windowing" 섹션에 sort_cost 등 표시
```

### 4. FRAME 절과 성능

```sql
-- ROWS BETWEEN: 물리적 행 수 기준
SELECT
    order_id,
    amount,
    SUM(amount) OVER (
        ORDER BY created_at
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW  -- 현재 포함 최근 3건 합계
    ) AS rolling_3_sum
FROM orders WHERE user_id = 1;

-- RANGE BETWEEN: 값 범위 기준
SELECT
    order_id,
    amount,
    SUM(amount) OVER (
        ORDER BY created_at
        RANGE BETWEEN INTERVAL 7 DAY PRECEDING AND CURRENT ROW  -- 7일 이내 합계
    ) AS weekly_sum
FROM orders WHERE user_id = 1;

-- UNBOUNDED PRECEDING: 처음부터 현재까지 누적 (기본 프레임)
SELECT
    order_id,
    SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS cumulative_total
FROM orders;
-- ORDER BY가 있는 윈도우의 기본 프레임: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- 성능 차이:
-- ROWS BETWEEN: 고정 행 수 → 효율적
-- RANGE BETWEEN: 값 비교 → ROWS보다 약간 비용 높음
-- UNBOUNDED PRECEDING: 누적 집계 → 이전 결과 재사용 가능 (최적화됨)
```

---

## 💻 실전 실험

### 실험 1: 윈도우 함수 실행계획 확인

```sql
-- 테이블 준비 (orders 10만 건 가정)

-- 윈도우 함수 EXPLAIN
EXPLAIN SELECT
    user_id,
    order_id,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM orders
WHERE status = 'PAID';
-- Extra: Using temporary (임시 테이블 사용)

-- WHERE로 필터 후 윈도우 함수
EXPLAIN SELECT
    user_id,
    order_id,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM orders
WHERE user_id BETWEEN 1 AND 100 AND status = 'PAID';
-- 필터 후 범위가 줄어 임시 테이블 크기 감소
```

### 실험 2: 사용자별 최근 3건 주문 — 두 가지 방법 비교

```sql
-- 방법 1: 윈도우 함수 + 서브쿼리
EXPLAIN ANALYZE
SELECT user_id, order_id, created_at
FROM (
    SELECT
        user_id, order_id, created_at,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
    WHERE status = 'PAID'
) ranked
WHERE rn <= 3;

-- 방법 2: 상관 서브쿼리 (비교용)
EXPLAIN ANALYZE
SELECT o.user_id, o.order_id, o.created_at
FROM orders o
WHERE (
    SELECT COUNT(*) FROM orders o2
    WHERE o2.user_id = o.user_id
    AND o2.created_at > o.created_at
    AND o2.status = 'PAID'
) < 3 AND o.status = 'PAID';

-- 두 방법의 실행 시간 및 실행계획 비교
```

### 실험 3: WINDOW 절 재사용 효과

```sql
-- 동일 PARTITION + ORDER BY를 WINDOW 절로 재사용
EXPLAIN ANALYZE
SELECT
    user_id, order_id, amount, created_at,
    ROW_NUMBER() OVER w AS rn,
    SUM(amount) OVER w AS running_total,
    AVG(amount) OVER w AS running_avg
FROM orders
WHERE user_id = 1
WINDOW w AS (PARTITION BY user_id ORDER BY created_at);
-- 한 번 정렬로 세 함수 처리 확인 (loops 확인)

-- WINDOW 절 없이 각각 선언
EXPLAIN ANALYZE
SELECT
    user_id, order_id, amount, created_at,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS rn,
    SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total,
    AVG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_avg
FROM orders WHERE user_id = 1;
-- 비교: 정렬 횟수 차이 확인
```

---

## 📊 성능/비용 비교

```
윈도우 함수 vs GROUP BY 집계 비교:

작업: 사용자별 주문 합계

GROUP BY:
  SELECT user_id, SUM(amount) FROM orders GROUP BY user_id;
  → 집계 결과만 반환, 개별 Row 불필요
  → 임시 테이블 크기 = GROUP 수
  예상 시간: 빠름

윈도우 함수 (SUM OVER PARTITION BY):
  SELECT user_id, order_id, SUM(amount) OVER (PARTITION BY user_id) FROM orders;
  → 모든 개별 Row + 각 Row에 합계 컬럼 추가
  → 임시 테이블 크기 = 전체 Row 수
  예상 시간: GROUP BY보다 느림 (더 많은 Row 처리)

결론:
  집계 결과만 필요 → GROUP BY (더 효율적)
  개별 Row + 그룹 집계값 모두 필요 → 윈도우 함수

ROW_NUMBER vs GROUP BY로 "상위 N건":
  ROW_NUMBER 방법: 전체 정렬 + 서브쿼리 필터
  GROUP BY + JOIN 방법: 더 복잡한 쿼리지만 경우에 따라 빠름
  → 데이터 분포에 따라 다름, EXPLAIN ANALYZE로 비교 필수

WINDOW 절 재사용:
  동일 파티션 + ORDER BY: 1번 정렬로 여러 함수 처리
  다른 ORDER BY: 각각 별도 정렬 → 비용 N배
```

---

## ⚖️ 트레이드오프

```
윈도우 함수 사용 판단:

윈도우 함수 선택:
  ✅ 개별 Row 유지 + 그룹 집계값 동시 필요
  ✅ 누적 합계, 이동 평균 등 복잡한 분석
  ✅ 행 순위/번호 부여 (ROW_NUMBER)
  ✅ GROUP BY로 표현하기 어려운 패턴
  ❌ 집계 결과만 필요 시 GROUP BY보다 느림
  ❌ 대용량 테이블 전체 스캔 후 윈도우 처리 시 비용 큼

GROUP BY 선택:
  ✅ 집계 결과만 필요할 때
  ✅ 결과 행 수를 줄여야 할 때
  ❌ 개별 Row + 집계값을 동시에 원하면 복잡한 JOIN 필요

윈도우 함수 최적화 포인트:
  WHERE로 먼저 대상 범위 축소 → 임시 테이블 크기 감소
  WINDOW 절로 동일 파티션 함수 재사용 → 정렬 횟수 감소
  필요한 컬럼만 SELECT → 임시 테이블 Row 크기 감소

tmp_table_size 관련:
  윈도우 함수도 임시 테이블 사용
  대용량 파티션 → tmp_table_size 초과 → 디스크 임시 테이블
  → GROUP 5 WHERE 조건으로 파티션 크기 제한이 중요
```

---

## 📌 핵심 정리

```
윈도우 함수 핵심:

실행 순서:
  WHERE → GROUP BY → 윈도우 함수 처리 → ORDER BY → LIMIT
  → WHERE로 먼저 필터링 → 윈도우 함수 대상 행 수 감소

내부 처리:
  PARTITION BY → ORDER BY → FRAME → 함수 계산
  항상 임시 테이블 사용 (EXPLAIN Extra: Using temporary)

함수 선택:
  ROW_NUMBER: 중복 없는 연속 번호
  RANK: 동률 처리 (번호 건너뜀)
  DENSE_RANK: 동률 처리 (번호 연속)
  성능 차이는 미미 → 의미에 맞게 선택

최적화:
  WINDOW 절로 같은 파티션/정렬 재사용 → 정렬 횟수 감소
  WHERE 먼저 필터링 → 임시 테이블 크기 감소
  집계만 필요하면 GROUP BY가 더 효율적

주의:
  WHERE 절에 윈도우 함수 사용 불가
  → 서브쿼리/CTE로 감싸고 외부에서 필터링
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 두 쿼리는 동일한 결과를 반환한다. 어떤 상황에서 어느 쿼리가 더 빠른가?

```sql
-- 쿼리 A: 윈도우 함수
SELECT user_id, SUM(amount) OVER (PARTITION BY user_id) AS total
FROM orders WHERE status = 'PAID';

-- 쿼리 B: GROUP BY + JOIN
SELECT o.user_id, g.total
FROM orders o
JOIN (
    SELECT user_id, SUM(amount) AS total
    FROM orders WHERE status = 'PAID'
    GROUP BY user_id
) g ON g.user_id = o.user_id
WHERE o.status = 'PAID';
```

<details>
<summary>해설 보기</summary>

**쿼리 A (윈도우 함수)**:
- 전체 Row를 임시 테이블에 올린 후 파티션별 SUM 계산
- 반환 Row 수 = 전체 orders Row 수 (status='PAID')
- 임시 테이블 크기가 큼

**쿼리 B (GROUP BY + JOIN)**:
- GROUP BY로 user별 합계 먼저 계산 → 결과 Row 수 = user 수
- JOIN으로 원래 orders와 연결
- orders 테이블을 두 번 스캔하지만 GROUP BY 임시 테이블이 작음

**어느 쪽이 빠른가**:
- 반환 Row 수가 많고 전체 orders를 조회해야 한다면 → 비슷하거나 A가 단순
- user 수가 적고 orders가 많다면 → B가 GROUP BY 임시 테이블이 작아 유리할 수 있음
- 실제로는 `EXPLAIN ANALYZE`로 직접 비교하는 것이 정확합니다

**중요한 차이**: 쿼리 A는 `user_id`별 합계를 각 Row에 붙여 반환합니다. 쿼리 B도 같은 결과지만 옵티마이저가 다르게 처리할 수 있습니다.

</details>

---

**Q2.** 다음 쿼리에서 에러가 발생한다. 이유와 올바른 작성 방법을 설명하라.

```sql
SELECT user_id, order_id
FROM orders
WHERE ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) <= 3;
```

<details>
<summary>해설 보기</summary>

**에러 이유**: 윈도우 함수는 쿼리 실행 순서상 WHERE 절보다 늦게 처리됩니다. WHERE 절이 평가될 시점에는 아직 윈도우 함수 결과가 없기 때문에 `WHERE`에서 윈도우 함수를 사용할 수 없습니다.

**올바른 방법 1 — 서브쿼리로 감싸기:**
```sql
SELECT user_id, order_id
FROM (
    SELECT
        user_id,
        order_id,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) ranked
WHERE rn <= 3;
```

**올바른 방법 2 — CTE 사용:**
```sql
WITH ranked AS (
    SELECT
        user_id,
        order_id,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
)
SELECT user_id, order_id FROM ranked WHERE rn <= 3;
```

두 방법 모두 윈도우 함수를 먼저 계산한 후 외부에서 필터링합니다. 성능 측면에서는 전체 orders를 먼저 처리한 후 필터링하므로, 가능하면 내부 쿼리에 `WHERE` 조건을 추가해 대상 행 수를 미리 줄이는 것이 좋습니다.

</details>

---

<div align="center">

**[⬅️ GROUP BY / ORDER BY](./04-groupby-orderby-optimization.md)** | **[홈으로 🏠](../README.md)** | **[다음: 문자셋과 콜레이션 ➡️](./06-charset-collation-implicit-conversion.md)**

</div>
