# GROUP BY / ORDER BY 최적화 — 인덱스가 정렬을 대체하는 조건

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Extra: Using filesort`가 발생하는 정확한 조건은?
- 인덱스가 ORDER BY를 대체할 수 있는 정확한 조건은?
- GROUP BY가 인덱스를 타는 조건과 타지 않는 조건은?
- ORDER BY ASC, DESC 혼합이 인덱스를 타지 않는 이유는?
- GROUP BY + ORDER BY + LIMIT 조합에서 인덱스가 동작하는 경우와 아닌 경우는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### Filesort는 항상 느린가? — 아니다. 하지만 대용량에서는 치명적

```
실제 이슈:
  게시판 최신순 정렬 API가 페이지마다 속도 다름

  Page 1 (LIMIT 20): 빠름
  Page 500 (OFFSET 10000, LIMIT 20): 5초

  원인:
    OFFSET 10000이면 10,020건을 읽고 10,000건을 버림
    + Using filesort로 10,020건 전체를 정렬 후 잘라냄

  해결:
    커서 기반 페이지네이션 + 인덱스 정렬 활용

핵심:
  Using filesort = 정렬에 인덱스를 못 쓴 것
  인덱스 정렬 = 이미 정렬된 인덱스를 순서대로 읽음 → 추가 정렬 없음
  언제 인덱스 정렬이 동작하는지 알면 Filesort를 제거할 수 있다
```

---

## 😱 흔한 실수 (Before)

### 1. 인덱스 컬럼 순서와 ORDER BY 순서를 맞추지 않음

```sql
-- 인덱스: (status, created_at)
-- Before: ORDER BY 순서가 인덱스와 다름
SELECT * FROM orders
WHERE status = 'PAID'
ORDER BY created_at DESC, status ASC;  -- 순서가 인덱스와 다름
-- Using filesort 발생

-- 또는
SELECT * FROM orders
ORDER BY created_at, status;  -- WHERE 없이 인덱스 두 번째 컬럼부터 시작
-- Using filesort 발생 (인덱스 leftmost prefix 위반)
```

### 2. ASC/DESC 혼합

```sql
-- 인덱스: (a ASC, b ASC)
SELECT * FROM t ORDER BY a ASC, b DESC;
-- MySQL 8.0 이전: Using filesort
-- MySQL 8.0 이후: Descending Index가 있으면 가능하지만 복잡

-- 흔한 실수: 복합 정렬에 무심코 방향 혼합
SELECT * FROM orders
WHERE user_id = 1
ORDER BY created_at ASC, amount DESC;
-- 인덱스 (user_id, created_at, amount)가 있어도
-- 방향이 다르면 Filesort 발생 가능
```

### 3. GROUP BY 후 ORDER BY NULL을 모름

```sql
-- Before: GROUP BY 결과에 불필요한 정렬 추가
SELECT status, COUNT(*) AS cnt
FROM orders
GROUP BY status;
-- MySQL은 GROUP BY 결과를 기본적으로 정렬 (구버전에서)
-- ORDER BY NULL을 추가하면 정렬 제거 가능 (MySQL 5.7 이하)

-- MySQL 8.0에서는 GROUP BY가 암묵적 정렬 안 함
-- 하지만 특정 환경에서는 여전히 Using filesort 발생
```

---

## ✨ 올바른 접근 (After)

```
인덱스가 ORDER BY를 대체하는 원리:

인덱스 (a, b, c) 기준:
  B+Tree Leaf 노드가 (a, b, c) 순서로 정렬되어 연결됨
  ORDER BY a, b, c → 인덱스 순서와 동일 → Leaf 순서대로 읽으면 됨
  별도 정렬 불필요 → Using filesort 없음

인덱스로 ORDER BY 처리 가능 조건:
  ① ORDER BY 컬럼이 인덱스의 leftmost prefix를 만족
  ② WHERE 등치 조건이 먼저 오고, ORDER BY가 나머지 인덱스 컬럼
  ③ 모든 ORDER BY 방향이 동일 (모두 ASC 또는 모두 DESC)
  ④ ORDER BY에 표현식/함수 없음

인덱스로 ORDER BY 처리 불가 조건:
  ① ORDER BY 컬럼에 함수 적용 (ORDER BY DATE(created_at))
  ② ASC/DESC 혼합
  ③ 인덱스에 없는 컬럼 포함
  ④ 여러 테이블 JOIN에서 ORDER BY가 드라이빙 테이블 외 컬럼
```

---

## 🔬 내부 동작 원리

### 1. Using filesort의 두 가지 방식

```
Filesort 방식 1: Single-pass (단일 패스)
  SELECT 컬럼 전체를 sort buffer에 올림
  → 정렬 후 바로 결과 반환
  장점: 디스크 접근 1회
  단점: sort buffer 크기 = (컬럼 크기 × Row 수)
  → 넓은 테이블에서 sort_buffer_size 초과 → 디스크 sort

Filesort 방식 2: Two-pass (두 번 패스)
  1단계: 정렬 키 + Row 포인터만 sort buffer에 올려 정렬
  2단계: 정렬 순서대로 Row 포인터로 데이터 재조회
  장점: sort buffer 크기 = (키 크기 × Row 수) → 작음
  단점: 디스크 접근 2회 가능성

MySQL 선택 기준:
  SELECT 컬럼 크기 합 > max_length_for_sort_data (기본 4096바이트)
  → Two-pass 선택
  그 외: Single-pass 선택

sort_buffer_size 튜닝:
  정렬 Row가 sort_buffer에 다 들어오면 → 메모리 정렬 (빠름)
  넘치면 → 임시 파일로 Merge Sort (느림)
  기본값: 256KB
  권장: 256KB ~ 4MB (과도하게 크게 설정 금지, 세션당 할당됨)
```

### 2. 인덱스 정렬 동작 조건 상세

```sql
-- 인덱스: INDEX idx_user_date (user_id, created_at)

-- 케이스 1: 인덱스 정렬 동작 (filesort 없음)
SELECT * FROM orders
WHERE user_id = 1
ORDER BY created_at DESC;
-- user_id = 1 (등치 조건으로 인덱스 범위 고정)
-- created_at DESC (인덱스 순서 역방향)
-- 인덱스 역방향 스캔으로 처리 → Using filesort 없음

-- 케이스 2: 인덱스 정렬 동작
SELECT * FROM orders
WHERE user_id = 1
ORDER BY user_id, created_at;
-- user_id 등치 후 created_at 순서 → 인덱스 순서와 동일

-- 케이스 3: filesort 발생
SELECT * FROM orders
ORDER BY created_at DESC;
-- user_id 조건 없이 created_at부터 정렬 → leftmost prefix 위반

-- 케이스 4: filesort 발생 (방향 혼합)
SELECT * FROM orders
WHERE user_id = 1
ORDER BY created_at ASC, amount DESC;
-- created_at은 인덱스에 있지만 amount의 DESC가 다름
-- amount가 인덱스에 없으면 filesort

-- 케이스 5: 인덱스 정렬 동작 (Covering Index)
SELECT user_id, created_at FROM orders  -- 인덱스 컬럼만 SELECT
WHERE user_id = 1
ORDER BY created_at;
-- Extra: Using index (Covering Index + 인덱스 정렬)
```

### 3. GROUP BY 인덱스 최적화

```sql
-- 인덱스: INDEX idx_status_date (status, created_at)

-- 케이스 1: Loose Index Scan (인덱스로 GROUP BY 처리)
SELECT status, MIN(created_at), MAX(created_at)
FROM orders
GROUP BY status;
-- status가 인덱스 첫 번째 컬럼 → Loose Index Scan 가능
-- EXPLAIN Extra: Using index for group-by

-- 케이스 2: Tight Index Scan
SELECT user_id, COUNT(*)
FROM orders
WHERE user_id BETWEEN 1 AND 100
GROUP BY user_id;
-- user_id 범위 조건 후 GROUP BY → Tight Index Scan

-- 케이스 3: 임시 테이블 + filesort
SELECT YEAR(created_at) AS yr, COUNT(*)
FROM orders
GROUP BY YEAR(created_at);
-- 함수 적용 → 인덱스 GROUP BY 불가 → Using temporary; Using filesort

-- GROUP BY + ORDER BY 동일 컬럼
SELECT status, COUNT(*) AS cnt
FROM orders
GROUP BY status
ORDER BY status;  -- GROUP BY 컬럼과 동일 → 추가 정렬 불필요

-- GROUP BY + ORDER BY 다른 컬럼
SELECT status, COUNT(*) AS cnt
FROM orders
GROUP BY status
ORDER BY cnt DESC;  -- cnt는 계산값 → filesort 불가피
```

### 4. GROUP BY + ORDER BY + LIMIT 최적화

```sql
-- 인덱스: INDEX idx_user_created (user_id, created_at)

-- filesort + LIMIT: 전체 정렬 후 자름 (느림)
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 10;
-- created_at 단독 인덱스 없으면 → 전체 filesort → LIMIT 10

-- 인덱스 정렬 + LIMIT: 인덱스 순서로 10건만 읽음 (빠름)
-- created_at에 단독 인덱스 있으면:
CREATE INDEX idx_created ON orders (created_at);
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 10;
-- 인덱스 역순으로 10건만 읽고 종료 → Extra: Using index

-- GROUP BY + ORDER BY NULL (MySQL 8.0에서는 효과 없지만 명시적 의도 표현)
SELECT status, COUNT(*) FROM orders GROUP BY status ORDER BY NULL;
-- MySQL 8.0: GROUP BY 기본 정렬 없음 → ORDER BY NULL은 no-op
-- MySQL 5.7 이하: GROUP BY가 암묵적 정렬 → ORDER BY NULL로 비용 제거

-- 커버링 인덱스 + LIMIT
SELECT id, user_id, created_at FROM orders
WHERE user_id = 1
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
-- (user_id, created_at) 커버링 인덱스 시 → Extra: Using index
```

---

## 💻 실전 실험

### 실험 1: filesort 발생/미발생 비교

```sql
-- 인덱스 준비
CREATE TABLE sort_test (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    status     VARCHAR(20),
    created_at DATETIME NOT NULL,
    amount     DECIMAL(10,2),
    INDEX idx_user_created (user_id, created_at),
    INDEX idx_created (created_at)
) ENGINE=InnoDB;

-- filesort 없는 케이스
EXPLAIN SELECT id, created_at FROM sort_test
WHERE user_id = 1 ORDER BY created_at DESC LIMIT 20;
-- Extra: Using index condition (또는 Using where, 인덱스 사용)
-- filesort 없음 확인

-- filesort 발생 케이스
EXPLAIN SELECT * FROM sort_test
ORDER BY amount DESC LIMIT 20;
-- Extra: Using filesort (amount 인덱스 없음)

-- filesort 발생 케이스 2
EXPLAIN SELECT * FROM sort_test
WHERE user_id = 1
ORDER BY created_at ASC, amount DESC LIMIT 20;
-- Extra: Using filesort (방향 혼합)
```

### 실험 2: Loose Index Scan 확인

```sql
EXPLAIN SELECT status, MIN(created_at), MAX(created_at)
FROM sort_test
GROUP BY status;
-- Extra: Using index for group-by → Loose Index Scan
-- status에 인덱스 없으면 Using temporary; Using filesort

-- status 단독 인덱스 추가 후 비교
CREATE INDEX idx_status ON sort_test(status);
EXPLAIN SELECT status, MIN(created_at), MAX(created_at)
FROM sort_test GROUP BY status;
-- Using index for group-by 확인
```

### 실험 3: sort_buffer_size 영향 측정

```sql
-- 작은 sort_buffer_size로 디스크 sort 유도
SET sort_buffer_size = 32768;  -- 32KB

FLUSH STATUS;
SELECT * FROM sort_test ORDER BY amount DESC;
SHOW STATUS LIKE 'Sort_merge_passes';
-- Sort_merge_passes > 0 → 디스크 sort 발생

-- 큰 sort_buffer_size
SET sort_buffer_size = 4 * 1024 * 1024;  -- 4MB

FLUSH STATUS;
SELECT * FROM sort_test ORDER BY amount DESC;
SHOW STATUS LIKE 'Sort_merge_passes';
-- Sort_merge_passes = 0 → 메모리 정렬

SET sort_buffer_size = DEFAULT;
DROP TABLE sort_test;
```

---

## 📊 성능/비용 비교

```
정렬 방식별 성능 (100만 건, amount 인덱스 없음):

인덱스 정렬 (created_at 인덱스 있음):
  ORDER BY created_at DESC LIMIT 20
  → 인덱스 역순으로 20건만 읽고 종료
  읽은 Row: 20건
  예상 시간: 0.001초

메모리 Filesort (sort_buffer 충분):
  ORDER BY amount DESC LIMIT 20
  → 100만 건 읽기 + 메모리 정렬 + LIMIT
  읽은 Row: 100만 건
  예상 시간: 0.5~1초

디스크 Filesort (sort_buffer 부족):
  같은 쿼리, sort_buffer 작음
  → 100만 건 읽기 + 디스크 Merge Sort
  예상 시간: 3~10초

GROUP BY 비교:
  Loose Index Scan (인덱스 있음):
    각 GROUP 값에서 첫 Row만 읽음
    읽은 Row: GROUP 수 (예: status 종류 3가지면 3건)
    예상 시간: 0.001초

  Full Scan + 임시 테이블 (인덱스 없음):
    전체 읽기 + GROUP 집계 + 정렬
    읽은 Row: 전체
    예상 시간: 0.5~2초
```

---

## ⚖️ 트레이드오프

```
인덱스 정렬 최적화 트레이드오프:

ORDER BY를 위한 인덱스 추가:
  ✅ filesort 제거 → 빠른 정렬
  ✅ LIMIT과 조합 시 필요한 건수만 읽고 종료
  ❌ 인덱스 유지 비용 (INSERT/UPDATE/DELETE 느려짐)
  ❌ 인덱스 크기 증가 → 메모리 압박

sort_buffer_size 증가:
  ✅ 메모리 정렬 가능 → 디스크 sort 없음
  ❌ 세션당 할당 → 동시 연결 많으면 메모리 고갈
  권장: 전역 설정보다 특정 쿼리에서 세션 단위로 증가

GROUP BY ORDER BY 패턴:
  GROUP BY 결과를 집계값으로 정렬 (ORDER BY cnt DESC):
    filesort 불가피
    → 결과 건수가 작으면 (GROUP 수 작으면) 무시 가능
    → GROUP 수가 많으면 비용 증가

MySQL 8.0 Descending Index:
  CREATE INDEX idx_desc ON t (a DESC);
  ASC/DESC 혼합 정렬 지원
  → 혼합 방향 정렬이 자주 필요한 경우에 고려
  → 유지 비용은 일반 인덱스와 동일
```

---

## 📌 핵심 정리

```
GROUP BY / ORDER BY 최적화 핵심:

인덱스로 ORDER BY 처리 조건:
  ① ORDER BY 컬럼이 인덱스 leftmost prefix 만족
  ② 모든 방향이 동일 (모두 ASC 또는 모두 DESC)
  ③ ORDER BY에 함수/표현식 없음
  ④ WHERE 등치 조건이 인덱스 앞부분을 커버

Using filesort:
  인덱스 정렬 불가 시 발생
  메모리 sort (빠름) vs 디스크 sort (느림)
  sort_buffer_size로 메모리 sort 가능 범위 조정

GROUP BY 인덱스 최적화:
  GROUP BY 컬럼이 인덱스 leftmost prefix → Loose Index Scan
  EXPLAIN Extra: Using index for group-by → 최적화 적용됨

GROUP BY + ORDER BY + LIMIT:
  인덱스로 ORDER BY 처리 + LIMIT → 필요한 건수만 읽음 (최적)
  인덱스 없이 filesort + LIMIT → 전체 정렬 후 자름 (비효율)

진단:
  EXPLAIN Extra: Using filesort → 인덱스 정렬 불가
  EXPLAIN Extra: Using temporary → GROUP BY 임시 테이블
  SHOW STATUS LIKE 'Sort_merge_passes' → 디스크 sort 발생 여부
```

---

## 🤔 생각해볼 문제

**Q1.** 인덱스 `(user_id, created_at)`가 있을 때, 다음 쿼리들 중 Using filesort가 발생하는 것은 어느 것인가?

```sql
-- A
SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC;

-- B
SELECT * FROM orders ORDER BY user_id, created_at;

-- C
SELECT * FROM orders WHERE user_id = 1 ORDER BY user_id DESC, created_at ASC;

-- D
SELECT * FROM orders WHERE user_id BETWEEN 1 AND 100 ORDER BY created_at;
```

<details>
<summary>해설 보기</summary>

- **A**: filesort 없음. `user_id = 1` (등치)로 인덱스 범위 고정 후 `created_at DESC` 역순 스캔.

- **B**: filesort 없음. `ORDER BY user_id, created_at`은 인덱스 순서와 동일. WHERE 없이도 인덱스 full scan으로 정렬 가능.

- **C**: filesort 발생 가능. `user_id DESC, created_at ASC` — 방향이 혼합됨. `user_id = 1` 등치 조건이 있으면 `user_id DESC`는 의미 없고 `created_at ASC`만 남아 filesort 없을 수도 있지만, 정확한 판단은 EXPLAIN으로 확인 필요. MySQL 8.0의 Optimizer가 `user_id = 1` 등치를 인식하면 filesort 없을 수 있음.

- **D**: filesort 발생 가능. `user_id BETWEEN 1 AND 100`은 범위 조건 → 인덱스의 두 번째 컬럼 `created_at`으로 정렬이 보장되지 않음. EXPLAIN으로 확인 필요.

핵심: 실제 확인은 항상 `EXPLAIN`으로 합니다. 특히 범위 조건(`BETWEEN`, `>`, `<`) 후의 ORDER BY는 인덱스 정렬이 동작하지 않는 경우가 많습니다.

</details>

---

**Q2.** 다음 쿼리가 느린 이유를 EXPLAIN 관점에서 설명하고 개선 방법을 제시하라.

```sql
SELECT status, COUNT(*) AS cnt
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY status
ORDER BY cnt DESC
LIMIT 5;
```

<details>
<summary>해설 보기</summary>

**느린 이유 분석**:

1. `WHERE created_at >= '2024-01-01'` — `created_at` 인덱스가 있으면 범위 스캔, 없으면 Full Scan
2. `GROUP BY status` — status별 집계 → 임시 테이블 생성 (`Using temporary`)
3. `ORDER BY cnt DESC` — `cnt`는 집계 결과값 → 인덱스 없음 → filesort (`Using filesort`)

**EXPLAIN 예상**:
```
Extra: Using where; Using temporary; Using filesort
```

**개선 방법**:

방법 1 — 복합 인덱스로 WHERE 범위 스캔 최적화:
```sql
CREATE INDEX idx_created_status ON orders (created_at, status);
-- created_at 범위 후 status로 GROUP BY 처리
-- Tight Index Scan 가능성
```

방법 2 — 결과 건수가 적으면 수용:
`status` 종류가 5가지라면 GROUP BY 결과는 5건. 5건 정렬은 비용이 거의 없음. WHERE의 범위 스캔만 최적화해도 충분할 수 있음.

방법 3 — 집계 테이블 사전 계산:
실시간으로 이 쿼리를 자주 실행한다면, 주기적으로 집계 결과를 별도 테이블에 저장하고 조회.

`GROUP BY` 결과를 `ORDER BY`하는 패턴에서 `ORDER BY`에 사용되는 컬럼이 집계값(cnt, sum 등)이면 filesort는 피할 수 없습니다. 비용이 문제라면 결과 건수(GROUP 수)를 줄이는 방향으로 쿼리를 개선합니다.

</details>

---

<div align="center">

**[⬅️ 파생 테이블과 CTE](./03-derived-table-cte.md)** | **[홈으로 🏠](../README.md)** | **[다음: 윈도우 함수 ➡️](./05-window-function-internals.md)**

</div>
