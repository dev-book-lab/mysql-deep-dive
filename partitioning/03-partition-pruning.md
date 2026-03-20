# 파티션 프루닝 — 쿼리가 파티션을 건너뛰는 조건

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Partition Pruning이 동작하기 위한 정확한 조건은?
- 함수 적용/형변환/OR 조건이 프루닝을 막는 이유는?
- EXPLAIN에서 프루닝 여부를 어떻게 확인하는가?
- 프루닝이 안 될 때 ALL Partition Scan의 비용은 얼마나 큰가?
- 동적 프루닝과 정적 프루닝의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 프루닝이 동작하지 않으면 파티셔닝의 의미가 없다

```
실제 이슈:
  월별 RANGE 파티션 (12개 파티션)
  다음 쿼리의 EXPLAIN 결과:
    WHERE YEAR(created_at) = 2024 → 12개 파티션 ALL Scan
    WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01' → p2024 1개

  "같은 의미 아닌가요?" → DB에게는 다름!
  함수 적용 → 파티션 결정 불가 → ALL Scan
  범위 조건 → 파티션 경계와 직접 비교 가능 → 프루닝

  결과: WHERE 절 작성 방식 하나로
        12개 파티션 스캔 vs 1개 파티션 스캔 = 12배 성능 차이
```

---

## 😱 흔한 실수 (Before)

### 1. 파티션 키에 함수 적용

```sql
-- RANGE 파티션: RANGE COLUMNS(created_at)
-- Before: 함수 적용으로 프루닝 불가
SELECT * FROM logs WHERE YEAR(created_at) = 2024;
-- partitions: ALL Scan (YEAR() 함수 → 어느 파티션인지 계산 불가)

SELECT * FROM logs WHERE DATE_FORMAT(created_at, '%Y-%m') = '2024-01';
-- partitions: ALL Scan

SELECT * FROM logs WHERE UNIX_TIMESTAMP(created_at) > 1704067200;
-- partitions: ALL Scan

-- After: 직접 비교로 프루닝 동작
SELECT * FROM logs WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
-- partitions: p2024 (1개만)
```

### 2. OR 조건으로 프루닝 깨짐

```sql
-- RANGE 파티션 (월별)
-- Before: OR 조건
SELECT * FROM logs WHERE created_at = '2024-01-15' OR user_id = 999;
-- partitions: ALL Scan
-- "user_id = 999는 어느 파티션에 있을지 모름"
-- → 두 조건 중 하나라도 프루닝 불가면 ALL Scan

-- After: UNION ALL로 분리
SELECT * FROM logs WHERE created_at = '2024-01-15'
UNION ALL
SELECT * FROM logs WHERE user_id = 999;
-- 첫 번째: 특정 파티션만 (프루닝)
-- 두 번째: ALL Scan (어쩔 수 없음, user_id는 파티션 키 아님)
```

### 3. 형변환으로 프루닝 실패

```sql
-- RANGE 파티션: HASH(user_id), user_id BIGINT
-- Before: 문자열로 비교
SELECT * FROM users WHERE user_id = '12345';
-- '12345' → BIGINT 변환 → 프루닝 가능할 수 있지만
-- 복잡한 표현식에서는 불확실

-- Before: 파티션 키 컬럼에 연산
SELECT * FROM logs WHERE created_at + INTERVAL 1 DAY > '2024-02-01';
-- 컬럼 연산 → 파티션 결정 불가 → ALL Scan

-- After: 상수를 이동
SELECT * FROM logs WHERE created_at > '2024-01-31';
-- 컬럼 직접 비교 → 프루닝 동작
```

---

## ✨ 올바른 접근 (After)

```
프루닝이 동작하는 조건:

1. 파티션 키 컬럼이 WHERE에 직접 등장
   WHERE created_at >= '2024-01-01' (created_at이 파티션 키)
   ✅ 프루닝 가능

2. 파티션 키 컬럼에 함수/연산 없음
   WHERE YEAR(created_at) = 2024 → ❌ 함수 적용
   WHERE created_at >= '2024-01-01' → ✅ 직접 비교

3. 파티션 키 컬럼과 상수/파라미터 비교
   WHERE created_at = ? (파라미터) → ✅
   WHERE created_at = other_table.date → ❌ (서브쿼리/조인 컬럼)

4. AND 조건은 프루닝 유지
   WHERE created_at >= '2024-01-01' AND status = 'PAID' → ✅

5. OR 조건은 프루닝 불가 (파티션 키 외 컬럼 포함 시)
   WHERE created_at >= '2024-01-01' OR user_id = 999 → ❌
```

---

## 🔬 내부 동작 원리

### 1. 정적 프루닝 vs 동적 프루닝

```
정적 프루닝 (Static Pruning):
  쿼리 파싱 시점에 어느 파티션을 접근할지 결정
  WHERE created_at = '2024-01-15' (상수)
  → 파서가 '2024-01-15'가 p2024에 속함을 미리 계산
  → 실행 계획 자체에 p2024만 포함

동적 프루닝 (Dynamic Pruning):
  실행 시점에 파티션 결정 (파라미터 바인딩 후)
  WHERE created_at = ? (PreparedStatement 파라미터)
  → 파라미터 값이 들어온 후 파티션 결정
  → 파라미터가 '2024-01-15'면 p2024
  → 실행마다 다른 파티션 접근 가능

EXPLAIN 동작:
  정적 프루닝: EXPLAIN에 접근 파티션 확정 표시
  동적 프루닝: EXPLAIN에 접근 파티션이 불확정 (모든 파티션 표시될 수 있음)
  → 실제 실행 시 동적으로 결정
```

### 2. 프루닝 판단 알고리즘

```
MySQL Optimizer의 파티션 프루닝 판단:

RANGE(created_at) 파티션, p2023 < p2024 < pmax:

WHERE created_at >= '2024-01-01':
  "2024-01-01 이상이면 p2024 이상"
  → p2023 제외, p2024, pmax 접근

WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31':
  "2024-01-01 ~ 2024-12-31 범위"
  → p2024에만 해당 (경계 확인)
  → p2024만 접근

WHERE created_at >= '2024-01-01' AND created_at < '2024-04-01':
  → p2024만 접근 (2024년 1분기)

WHERE YEAR(created_at) = 2024:
  Optimizer 입장: "YEAR() 적용 후 값이 2024이면 어느 범위?"
  → created_at 값 자체 범위 결정 불가
  → 모든 파티션 접근

WHERE created_at IN ('2024-01-15', '2023-06-01'):
  → p2024, p2023 각각 확인 → 두 파티션만 접근
  → LIST 파티션에서도 IN 조건 프루닝 동작
```

### 3. 조인에서의 파티션 프루닝

```sql
-- JOIN 시 프루닝 가능 여부

-- 파티션 키와 상수 비교 → 프루닝
SELECT * FROM logs l
JOIN events e ON e.id = l.event_id
WHERE l.created_at >= '2024-01-01';  -- logs의 파티션 키
-- logs: p2024 이후 파티션만 접근 (프루닝)
-- events: 프루닝 없음 (파티션 키 조건 없음)

-- 파티션 키와 다른 테이블 컬럼 비교 → 프루닝 어려움
SELECT * FROM logs l
JOIN date_ranges d ON l.created_at BETWEEN d.start_date AND d.end_date;
-- d.start_date, d.end_date는 런타임에 결정 → 동적 프루닝 (제한적)

-- WHERE절 아닌 ON절에서도 파티션 키 조건은 동작
SELECT * FROM logs l
JOIN orders o ON o.id = l.order_id AND l.created_at >= '2024-01-01';
-- l.created_at 조건이 ON 절이어도 logs 파티션 프루닝 동작
```

### 4. EXPLAIN으로 프루닝 확인

```sql
-- EXPLAIN에서 partitions 컬럼 확인
-- MySQL 8.0: EXPLAIN 기본으로 partitions 포함
EXPLAIN SELECT * FROM logs WHERE created_at >= '2024-01-01'\G

-- 출력:
-- id: 1
-- select_type: SIMPLE
-- table: logs
-- partitions: p2024,pmax    ← 이 부분! 프루닝 동작
-- type: ALL
-- ...

-- 프루닝 안 되는 경우:
EXPLAIN SELECT * FROM logs WHERE YEAR(created_at) = 2024\G
-- partitions: p2022,p2023,p2024,pmax   ← ALL Partition Scan

-- 포맷 지정
EXPLAIN FORMAT=JSON SELECT * FROM logs WHERE created_at >= '2024-01-01'\G
-- JSON 출력에서 "partition_filter" 또는 "partitions" 키 확인

-- 시각적 확인
EXPLAIN FORMAT=TREE SELECT * FROM logs WHERE created_at >= '2024-01-01';
-- 출력에서 "Partition: p2024,pmax" 형태로 표시
```

### 5. 프루닝이 안 될 때 ALL Partition Scan 비용

```
비용 비교 (12개 RANGE 파티션, 총 1억 건, 파티션당 약 833만 건):

프루닝 동작 (1개 파티션):
  스캔 대상: 833만 건
  인덱스 사용 시: O(log 8,333,333) ≈ 23 수준
  비교적 빠름

ALL Partition Scan (12개 파티션):
  스캔 대상: 1억 건 (12 × 833만)
  = 12배 더 많은 데이터 접근
  파티션 전환 오버헤드 × 12
  실질적으로 파티셔닝 없는 것과 동일하거나 더 느림

수치 예시:
  1개 파티션 Full Scan: 0.5초
  12개 파티션 Full Scan: 6~7초 (파티션 전환 비용 포함)
  → 잘못된 WHERE 절 하나가 12배 성능 차이
```

---

## 💻 실전 실험

### 실험 1: 프루닝 동작 여부 비교

```sql
-- 실험 테이블
CREATE TABLE pruning_test (
    id         BIGINT AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    amount     DECIMAL(10,2),
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p2022 VALUES LESS THAN ('2023-01-01'),
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);

-- 프루닝 동작 케이스
EXPLAIN SELECT * FROM pruning_test WHERE created_at >= '2024-01-01';
-- partitions: p2024,pmax (2개)

EXPLAIN SELECT * FROM pruning_test WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
-- partitions: p2024 (1개)

-- 프루닝 실패 케이스
EXPLAIN SELECT * FROM pruning_test WHERE YEAR(created_at) = 2024;
-- partitions: p2022,p2023,p2024,pmax (ALL)

EXPLAIN SELECT * FROM pruning_test WHERE created_at >= '2024-01-01' OR user_id = 999;
-- partitions: p2022,p2023,p2024,pmax (OR로 인해 ALL)

-- OR → UNION ALL 개선
EXPLAIN SELECT * FROM pruning_test WHERE created_at >= '2024-01-01'
UNION ALL
SELECT * FROM pruning_test WHERE user_id = 999 AND created_at < '2024-01-01';
-- 첫 번째 쿼리: p2024,pmax (프루닝)
-- 두 번째 쿼리: p2022,p2023 (날짜 조건으로 프루닝)
```

### 실험 2: 함수 변환 방법 비교

```sql
-- 월별 조회 패턴: 함수 vs 범위 비교

-- 느림: 함수 적용 → ALL Scan
EXPLAIN ANALYZE SELECT COUNT(*) FROM pruning_test
WHERE YEAR(created_at) = 2024 AND MONTH(created_at) = 3;

-- 빠름: 범위 변환
EXPLAIN ANALYZE SELECT COUNT(*) FROM pruning_test
WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';

-- 실행 시간 비교

-- 특정 월 전체 집계 쿼리 성능 차이
SET @t = SYSDATE(6);
SELECT COUNT(*) FROM pruning_test WHERE YEAR(created_at) = 2024 AND MONTH(created_at) = 3;
SELECT CONCAT('함수 방식: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

SET @t = SYSDATE(6);
SELECT COUNT(*) FROM pruning_test WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
SELECT CONCAT('범위 방식: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

DROP TABLE pruning_test;
```

### 실험 3: 동적 프루닝 (PreparedStatement)

```sql
-- PreparedStatement에서 파라미터 바인딩 시 동적 프루닝
PREPARE stmt FROM 'SELECT COUNT(*) FROM orders WHERE created_at >= ?';
SET @date = '2024-01-01';
EXECUTE stmt USING @date;
-- 실행 시 파라미터 값으로 파티션 결정

-- EXPLAIN으로 확인 (파라미터 없이 EXPLAIN 시 파티션 불확정)
EXPLAIN SELECT COUNT(*) FROM orders WHERE created_at >= ?;
-- partitions: ALL (파라미터 값 모름)
-- 실제 실행 시에는 동적으로 특정 파티션만 접근

DEALLOCATE PREPARE stmt;
```

---

## 📊 성능/비용 비교

```
프루닝 동작 여부별 성능 비교:
(12개 RANGE 파티션, 총 5,000만 건, 파티션당 약 417만 건)

조건: 2024년 1개월 데이터 조회 (약 416,667건)

범위 조건 (프루닝 동작):
  접근 파티션: 1개 (p2024)
  스캔 행 수: 416만 건 (인덱스 없을 경우)
  인덱스 있을 경우: 해당 건수만
  예상 시간: 0.3~1초

YEAR() 함수 (프루닝 실패):
  접근 파티션: 12개 (ALL)
  스캔 행 수: 5,000만 건
  예상 시간: 3~10초

OR 조건 (프루닝 실패):
  접근 파티션: ALL
  스캔 행 수: 5,000만 건
  예상 시간: 3~10초

UNION ALL로 분리 후:
  첫 번째: 1개 파티션
  두 번째: 원하는 파티션만
  합산 예상 시간: 0.5~2초
```

---

## ⚖️ 트레이드오프

```
프루닝 최적화 vs 쿼리 복잡도:

함수 제거 (YEAR() → 범위 변환):
  ✅ 프루닝 동작 → 성능 개선
  ❌ 애플리케이션 코드에서 날짜 범위 직접 계산 필요
  ❌ 동적 날짜 계산 복잡해짐
  예: "이번 달" → '2024-03-01' ~ '2024-04-01' 직접 계산

OR → UNION ALL:
  ✅ 각 조건에 대해 최적 프루닝 적용
  ❌ 쿼리 복잡도 증가
  ❌ 두 결과의 중복 Row 처리 필요 (UNION은 중복 제거)

동적 프루닝 (파라미터):
  ✅ 파라미터 값에 따라 최적 파티션만 접근
  ❌ EXPLAIN 시 파티션 확정 불가 (테스트 어려움)
  → 실제 파라미터로 EXPLAIN 실행 시 동적 프루닝 확인 가능

쿼리 작성 가이드:
  파티션 키 컬럼은 항상 직접 비교 (함수 없이)
  애플리케이션에서 날짜/범위 변환 후 파라미터로 전달
  OR 조건은 UNION ALL로 분리 고려
```

---

## 📌 핵심 정리

```
파티션 프루닝 핵심:

프루닝 동작 조건:
  ① 파티션 키 컬럼을 WHERE에서 직접 비교
  ② 함수/연산/형변환 없이 컬럼 직접 사용
  ③ AND 조건은 프루닝 유지
  ④ 파라미터(동적) 값도 실행 시점에 프루닝 적용

프루닝 실패 원인:
  YEAR(col), MONTH(col) 등 함수 적용
  col + 1, UNIX_TIMESTAMP(col) 등 컬럼 연산
  OR 조건 (파티션 키 외 컬럼 포함 시)
  NOT IN, != 등 부정 조건 (RANGE 파티션)

확인 방법:
  EXPLAIN SELECT ... → partitions 컬럼
  ALL Partition = 프루닝 실패 → WHERE 절 점검
  1~2개 파티션 = 프루닝 성공

쿼리 작성 규칙:
  WHERE YEAR(created_at) = 2024  → ❌
  WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01' → ✅
  WHERE created_at OR user_id = 1 → ❌ (OR)
  → UNION ALL로 분리                → ✅
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 쿼리들 중 RANGE COLUMNS (created_at) 파티션에서 프루닝이 동작하는 것은?

```sql
-- A
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- B
WHERE YEAR(created_at) = 2024

-- C
WHERE created_at = '2024-06-15'

-- D
WHERE created_at >= NOW() - INTERVAL 30 DAY

-- E
WHERE created_at >= '2024-01-01' OR status = 'PAID'
```

<details>
<summary>해설 보기</summary>

- **A**: ✅ 프루닝 동작. 직접 범위 비교, 파티션 경계와 바로 비교 가능.

- **B**: ❌ 프루닝 실패. `YEAR()` 함수 적용 → Optimizer가 어느 파티션 범위인지 계산 불가 → ALL Partition Scan.

- **C**: ✅ 프루닝 동작. 등치 비교 → 정확히 어느 파티션인지 결정 가능.

- **D**: ✅ 동적 프루닝 동작. `NOW() - INTERVAL 30 DAY`는 실행 시점에 상수로 평가됨. Optimizer가 실행 시 날짜 계산 후 해당 파티션만 접근 (동적 프루닝).

- **E**: ❌ 프루닝 부분 실패. `created_at` 조건은 프루닝 가능하지만 `OR status = 'PAID'`가 포함되어 MySQL이 모든 파티션을 접근할 수 있음 (OR 조건 전체 영향). UNION ALL로 분리 필요.

실제 확인은 항상 `EXPLAIN`의 `partitions` 컬럼으로 검증합니다.

</details>

---

**Q2.** 애플리케이션에서 "이번 달 데이터" 조회 쿼리를 파티션 프루닝이 동작하도록 작성하는 방법을 설명하라. (Java/Spring 코드 예시 포함)

<details>
<summary>해설 보기</summary>

**핵심**: 함수(`YEAR()`, `MONTH()`)를 사용하지 않고 날짜 범위로 변환해서 파라미터로 전달합니다.

```java
// Repository (JPQL or Native Query)
@Repository
public class LogRepository {
    
    @Query(value = """
        SELECT * FROM logs
        WHERE created_at >= :startDate AND created_at < :endDate
        """, nativeQuery = true)
    List<Log> findByMonth(@Param("startDate") LocalDateTime startDate,
                          @Param("endDate") LocalDateTime endDate);
}

// Service에서 범위 계산 후 전달
@Service
public class LogService {
    
    public List<Log> getThisMonthLogs() {
        YearMonth current = YearMonth.now();
        LocalDateTime startDate = current.atDay(1).atStartOfDay();
        LocalDateTime endDate = current.plusMonths(1).atDay(1).atStartOfDay();
        
        return logRepository.findByMonth(startDate, endDate);
        // → WHERE created_at >= '2024-03-01 00:00:00' AND created_at < '2024-04-01 00:00:00'
        // → 파티션 프루닝 동작!
    }
    
    // 잘못된 방법 (사용 금지)
    public List<Log> getThisMonthLogsWrong() {
        int year = LocalDate.now().getYear();
        int month = LocalDate.now().getMonthValue();
        return logRepository.findByYearMonth(year, month);
        // → WHERE YEAR(created_at) = 2024 AND MONTH(created_at) = 3
        // → ALL Partition Scan!
    }
}
```

JPA QueryDSL 사용 시도 동일하게 `BooleanExpression`에서 `between()` 또는 `goe()/lt()` 조합으로 날짜 범위를 직접 비교합니다.

</details>

---

<div align="center">

**[⬅️ 파티션 종류](./02-partition-types-range-list-hash-key.md)** | **[홈으로 🏠](../README.md)** | **[다음: 파티션과 인덱스 ➡️](./04-partition-and-index.md)**

</div>
