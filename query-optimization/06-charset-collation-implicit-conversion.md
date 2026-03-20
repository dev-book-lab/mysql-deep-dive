# 문자셋과 콜레이션 — 조인 시 묵시적 형변환이 인덱스를 망치는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- utf8mb4와 utf8 컬럼을 JOIN할 때 인덱스가 무력화되는 이유는?
- 숫자형 PK와 문자열 FK를 조인할 때 무슨 일이 발생하는가?
- Collation 불일치가 어떻게 Full Table Scan을 유발하는가?
- `EXPLAIN`에서 묵시적 형변환이 일어났음을 어떻게 파악하는가?
- JPA 엔티티 설계에서 자주 발생하는 형변환 함정은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 인덱스를 추가했는데 왜 Full Scan이 계속 나오지?

```
실제 운영 이슈:
  users(id BIGINT PK) + orders(user_id VARCHAR(20))
  orders.user_id에 인덱스 있음
  JOIN 시 orders 인덱스가 안 탐

  원인:
    SELECT * FROM users u JOIN orders o ON u.id = o.user_id
    u.id(BIGINT)와 o.user_id(VARCHAR) 타입 불일치
    → MySQL이 묵시적으로 o.user_id를 BIGINT로 변환 시도
    → CONVERT(o.user_id, BIGINT) 적용 = 인덱스 무력화!
    → orders 전체 Full Scan

  해결:
    orders.user_id를 BIGINT로 변경
    또는 JOIN 시 명시적 CAST 없이 타입 일치

또 다른 이슈:
  users 테이블: character_set = utf8, collation = utf8_general_ci
  orders 테이블: character_set = utf8mb4, collation = utf8mb4_unicode_ci
  JOIN 조건: u.email = o.email
  → 문자셋/콜레이션 불일치 → 묵시적 변환 → 인덱스 무력화
```

---

## 😱 흔한 실수 (Before)

### 1. 타입 불일치 JOIN

```sql
-- users.id BIGINT, orders.user_id VARCHAR(20)
-- Before: 아무 생각 없이 JOIN
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id;
-- u.id(BIGINT) = o.user_id(VARCHAR)
-- MySQL: o.user_id를 숫자로 변환 시도
-- EXPLAIN type: ALL (orders 전체 스캔)

-- 또는
WHERE user_id = 1;  -- user_id가 VARCHAR인데 숫자 리터럴 비교
-- VARCHAR 컬럼에 숫자를 비교 → CONVERT(user_id, SIGNED) → 인덱스 무력화
```

### 2. 문자셋 불일치를 모르고 배포

```sql
-- 레거시 테이블: utf8 (= utf8mb3, BMP 문자만 지원)
-- 신규 테이블: utf8mb4 (이모지 등 4바이트 문자 지원)

-- 두 테이블 JOIN
SELECT a.col1, b.col2
FROM legacy_table a       -- utf8, utf8_general_ci
JOIN new_table b ON a.key = b.key;  -- utf8mb4, utf8mb4_unicode_ci
-- key 컬럼 문자셋 불일치 → 묵시적 변환 → 인덱스 무력화
-- 개발 환경(소량)에서는 느린지 모름
-- 운영 환경에서 수백만 건 조인 시 문제 드러남
```

### 3. JPA에서 타입 매핑 오류

```java
// Entity 설계 실수
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;  // BIGINT

    // users 테이블의 id는 Long이지만
    @Column(name = "user_id")
    private String userId;  // VARCHAR로 잘못 매핑
    // → JPQL join 시 타입 불일치 → 인덱스 미사용
}
```

---

## ✨ 올바른 접근 (After)

```
형변환 규칙 이해:

MySQL 타입 우선순위 (높음 → 낮음):
  DECIMAL > DOUBLE > FLOAT > BIGINT > INT > 나머지 정수 > STRING

비교 시 형변환 방향:
  STRING vs INT → STRING이 INT로 변환 (낮은 쪽이 높은 쪽으로 변환)
  STRING vs STRING (다른 문자셋/콜레이션) → 한쪽이 변환

인덱스가 무력화되는 조건:
  컬럼에 함수가 적용될 때:
    WHERE YEAR(created_at) = 2024 → YEAR() 함수 적용 → 인덱스 무효
    WHERE user_id = 1 (VARCHAR vs INT) → CONVERT(user_id) → 인덱스 무효
    JOIN에서 문자셋 불일치 → CONVERT() → 한쪽 인덱스 무효

인덱스가 유지되는 조건:
  상수가 컬럼 타입으로 변환될 때:
    WHERE id = '123' (BIGINT vs VARCHAR 리터럴) → '123'이 BIGINT로 변환
    → 컬럼은 변환 없음 → 인덱스 사용 가능
```

---

## 🔬 내부 동작 원리

### 1. 숫자 vs 문자열 비교 — 어느 쪽이 변환되는가

```sql
-- 케이스 1: 컬럼이 INT, 리터럴이 STRING
-- user_id BIGINT에 INDEX 있음
WHERE user_id = '123';
-- '123' (STRING 리터럴) → BIGINT 변환
-- 컬럼은 그대로 → 인덱스 사용 가능 ✅

-- 케이스 2: 컬럼이 VARCHAR, 리터럴이 INT
-- user_code VARCHAR(20)에 INDEX 있음
WHERE user_code = 123;
-- CONVERT(user_code, INT) 적용
-- 컬럼에 함수 → 인덱스 무효 ❌

-- 케이스 3: 두 컬럼 타입 불일치 JOIN
-- users.id BIGINT, orders.user_id VARCHAR
ON users.id = orders.user_id;
-- orders.user_id(VARCHAR) → CONVERT to BIGINT
-- orders.user_id에 함수 적용 → orders 인덱스 무효 ❌

-- EXPLAIN으로 확인:
EXPLAIN SELECT * FROM users u JOIN orders o ON u.id = o.user_id;
-- orders의 type이 ALL 또는 range가 아닌 ref/eq_ref이어야 인덱스 사용
-- ALL이면 Full Scan → 타입 불일치 의심

-- 경고 확인:
SHOW WARNINGS;
-- "Impossible WHERE noticed after reading const tables" 등 메시지로 형변환 힌트
```

### 2. 문자셋 불일치와 Collation 변환

```sql
-- 테이블 문자셋/콜레이션 확인
SHOW TABLE STATUS WHERE Name IN ('legacy_table', 'new_table')\G
-- Collation 항목 확인

-- 컬럼별 확인
SHOW FULL COLUMNS FROM users;
-- Field, Type, Collation 확인

-- 콜레이션 우선순위 규칙 (Coercibility):
-- 낮은 숫자 = 높은 우선순위 (이쪽으로 변환됨)
-- 0: 명시적 COLLATE 절
-- 1: 컬럼의 COLLATE
-- 2: 연결 문자열 함수
-- 3: CONVERT()
-- 4: 시스템 변수
-- 5: 리터럴
-- 6: 결정 불가 (COERCIBLE)

-- 두 컬럼 콜레이션 불일치 JOIN:
-- legacy_table.key (utf8_general_ci, coercibility=2)
-- new_table.key (utf8mb4_unicode_ci, coercibility=2)
-- 동일 우선순위 → MySQL이 한쪽을 변환 시도
-- → 인덱스 무력화

-- 해결: CONVERT() 명시
SELECT * FROM legacy_table a
JOIN new_table b ON CONVERT(a.key USING utf8mb4) COLLATE utf8mb4_unicode_ci = b.key;
-- 또는 legacy_table.key를 utf8mb4로 변경 (근본 해결)
```

### 3. 콜레이션과 대소문자 처리

```sql
-- utf8mb4_general_ci (ci = case insensitive)
-- WHERE name = 'KIM' → 'kim', 'Kim', 'KIM' 모두 매칭

-- utf8mb4_bin (binary)
-- WHERE name = 'KIM' → 'KIM'만 매칭 (바이트 비교)

-- utf8mb4_unicode_ci vs utf8mb4_general_ci:
-- general_ci: 빠름, 일부 유니코드 규칙 미준수
-- unicode_ci: 정확한 유니코드 비교, 약간 느림
-- MySQL 8.0 기본: utf8mb4_0900_ai_ci (가장 정확하고 빠름)

-- LIKE와 콜레이션:
-- utf8mb4_general_ci: LIKE 'abc%' → 'ABC%', 'Abc%' 모두 매칭
-- 의도치 않은 대소문자 무시 → 버그 가능

-- 특정 비교에서만 콜레이션 변경:
WHERE name = 'KIM' COLLATE utf8mb4_bin;
-- 이 비교에서만 바이너리 비교 (인덱스는 collation 불일치 → 무력화될 수 있음)
```

### 4. JPA / Spring에서 발생하는 형변환 함정

```java
// 함정 1: ID 타입 불일치
@Entity
public class Order {
    @Column(name = "user_id")
    private String userId;  // DB는 BIGINT인데 Java String

    // JPQL: WHERE o.userId = :userId
    // 파라미터가 String → WHERE user_id = '123' (타입 불일치)
    // → 인덱스 동작에 따라 다름 (실제로는 BIGINT 컬럼에 STRING → 변환 OK)
    // 반대: DB가 VARCHAR인데 Long 파라미터 → 반드시 인덱스 무력화
}

// 함정 2: @Enumerated(STRING) vs 정수 비교
@Column(name = "status")
@Enumerated(EnumType.STRING)
private OrderStatus status;  // DB: VARCHAR

// Native Query에서 정수로 비교하면 타입 불일치
// JPQL: WHERE o.status = :status (파라미터는 OrderStatus enum)
// → 자동으로 .name()으로 변환 → 안전

// Native Query: WHERE status = 1 (실수)
// → status VARCHAR vs INT → 형변환 → 인덱스 무력화
```

---

## 💻 실전 실험

### 실험 1: 타입 불일치 인덱스 무력화 확인

```sql
-- 실험 테이블
CREATE TABLE type_mismatch_test (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    code    VARCHAR(20) NOT NULL,
    INDEX idx_code (code)
) ENGINE=InnoDB;

INSERT INTO type_mismatch_test (code)
SELECT CONCAT('CODE', seq)
FROM (SELECT @r := @r + 1 AS seq FROM information_schema.columns, (SELECT @r := 0) r LIMIT 100000) nums;

-- 인덱스 동작: STRING 리터럴 vs VARCHAR 컬럼
EXPLAIN SELECT * FROM type_mismatch_test WHERE code = 'CODE123';
-- type: ref (인덱스 사용 ✅)

-- 인덱스 무력화: INT 리터럴 vs VARCHAR 컬럼
EXPLAIN SELECT * FROM type_mismatch_test WHERE code = 123;
-- type: ALL (Full Scan ❌)
-- Extra: Using where

-- SHOW WARNINGS로 형변환 경고 확인
SHOW WARNINGS;
```

### 실험 2: 문자셋 불일치 JOIN 인덱스 비교

```sql
-- utf8 테이블
CREATE TABLE utf8_table (
    id   INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    INDEX idx_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_general_ci;

-- utf8mb4 테이블
CREATE TABLE utf8mb4_table (
    id   INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    INDEX idx_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

INSERT INTO utf8_table (name) SELECT CONCAT('Name', seq) FROM (SELECT @r := @r + 1 AS seq FROM information_schema.columns, (SELECT @r := 0) r LIMIT 10000) n;
INSERT INTO utf8mb4_table (name) SELECT name FROM utf8_table;

-- 문자셋 불일치 JOIN
EXPLAIN SELECT a.id, b.id
FROM utf8_table a
JOIN utf8mb4_table b ON a.name = b.name;
-- 한쪽 인덱스 무력화 확인 (type: ALL 나타남)

-- 해결: CONVERT() 명시
EXPLAIN SELECT a.id, b.id
FROM utf8_table a
JOIN utf8mb4_table b ON CONVERT(a.name USING utf8mb4) COLLATE utf8mb4_unicode_ci = b.name;
-- 여전히 utf8_table 쪽은 Full Scan
-- 근본 해결: utf8_table을 utf8mb4로 마이그레이션

DROP TABLE utf8_table, utf8mb4_table;
```

### 실험 3: BIGINT vs VARCHAR JOIN

```sql
CREATE TABLE users_int (id BIGINT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));
CREATE TABLE orders_str (id BIGINT AUTO_INCREMENT PRIMARY KEY, user_id VARCHAR(20), INDEX idx_uid (user_id));

INSERT INTO users_int (name) SELECT CONCAT('U', seq) FROM (SELECT @r := @r + 1 AS seq FROM information_schema.columns, (SELECT @r := 0) r LIMIT 1000) n;
INSERT INTO orders_str (user_id) SELECT id FROM users_int;

-- 타입 불일치 JOIN
EXPLAIN SELECT u.name, o.id
FROM users_int u
JOIN orders_str o ON u.id = o.user_id;
-- orders_str.user_id(VARCHAR) vs users_int.id(BIGINT) 불일치
-- orders_str 인덱스 무력화 예상

-- 타입 일치 (명시적 CAST)
EXPLAIN SELECT u.name, o.id
FROM users_int u
JOIN orders_str o ON CAST(u.id AS CHAR) = o.user_id;
-- orders_str 인덱스 사용 ✅

DROP TABLE users_int, orders_str;
```

---

## 📊 성능/비용 비교

```
형변환 발생 여부에 따른 성능:

orders 100만 건, user_id 인덱스 있음:

타입 일치 JOIN (BIGINT = BIGINT):
  type: ref
  읽는 Row: user_id 해당 건수 (예: 10건)
  예상 시간: 0.001초

타입 불일치 JOIN (BIGINT = VARCHAR):
  type: ALL (Full Scan)
  읽는 Row: 100만 건
  예상 시간: 0.5~2초

콜레이션 불일치 JOIN:
  한쪽 인덱스 무력화 → Full Scan
  예상 시간: 0.5~2초

영향도:
  소량 데이터 (10만 건 미만): 체감 어려움
  100만 건 이상: 수 초 → 운영 장애 수준
  JOIN 조건이 포함된 쿼리에서 발생 시 파급력 큼
```

---

## ⚖️ 트레이드오프

```
타입 통일 vs 현재 구조 유지:

타입 통일 (근본 해결):
  ✅ 인덱스 완전 활용
  ✅ 형변환 없음 → 예측 가능한 성능
  ❌ 스키마 변경 비용 (Online DDL 또는 pt-osc)
  ❌ 애플리케이션 코드 변경 필요

CONVERT() 명시 (임시 해결):
  ✅ 스키마 변경 없이 적용 가능
  ❌ 변환하는 쪽의 인덱스는 여전히 무력화
  ❌ 쿼리마다 적용 필요

DB 전체 문자셋 통일:
  ✅ 근본적으로 콜레이션 불일치 제거
  ❌ 테이블 재생성 또는 ALTER TABLE 필요
  권장: 신규 서비스는 utf8mb4_0900_ai_ci 통일

JPA 엔티티 타입 주의사항:
  DB 컬럼 BIGINT → Java Long (항상 일치)
  DB 컬럼 VARCHAR → Java String
  타입 불일치 시 Native Query에서 형변환 발생 가능
  JPQL은 JPA가 타입을 알고 있어 비교적 안전
  Native Query는 명시적 타입 검증 필요
```

---

## 📌 핵심 정리

```
형변환과 인덱스 핵심:

인덱스 무력화 형변환 조건:
  컬럼에 함수/변환이 적용될 때
    WHERE CONVERT(varchar_col, SIGNED) = 123 → 인덱스 무효
    JOIN ON bigint_col = varchar_col → varchar_col에 변환 → 무효

인덱스 유지 형변환:
  상수(리터럴)가 컬럼 타입으로 변환될 때
    WHERE bigint_col = '123' → '123'이 BIGINT로 변환, 컬럼 그대로 → 인덱스 OK

가장 흔한 실수:
  ① VARCHAR FK + BIGINT PK JOIN → VARCHAR 쪽 인덱스 무력화
  ② 문자셋/콜레이션 불일치 JOIN → 한쪽 인덱스 무력화
  ③ WHERE varchar_col = integer_literal → Full Scan

진단:
  EXPLAIN에서 의심: type=ALL + 인덱스 있는 테이블
  SHOW WARNINGS: 형변환 경고 메시지
  EXPLAIN FORMAT=JSON: "attached_condition"에 CONVERT 포함 여부

근본 해결:
  조인 컬럼 타입 통일
  전체 DB utf8mb4 통일 (신규 서비스)
  JPA 엔티티 컬럼 타입 DB와 일치
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 쿼리에서 `orders.user_id`에 인덱스가 있음에도 Full Scan이 발생하는 이유를 설명하고, 인덱스가 동작하도록 수정하라.

```sql
-- users.id: BIGINT, orders.user_id: VARCHAR(20)
SELECT u.name, COUNT(*) AS order_count
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.id;
```

<details>
<summary>해설 보기</summary>

**Full Scan 이유**: `u.id(BIGINT)`와 `o.user_id(VARCHAR)` 타입 불일치. MySQL이 `o.user_id`를 BIGINT로 묵시적 변환(`CONVERT(o.user_id, SIGNED)`)하려 시도 → `orders.user_id` 컬럼에 함수 적용 → 인덱스 무력화 → Full Scan.

**수정 방법 1 — 근본 해결 (스키마 변경):**
```sql
ALTER TABLE orders MODIFY COLUMN user_id BIGINT NOT NULL;
-- 이후 JOIN에서 타입 일치 → 인덱스 정상 동작
```

**수정 방법 2 — 쿼리에서 CAST 활용:**
```sql
SELECT u.name, COUNT(*) AS order_count
FROM users u
JOIN orders o ON CAST(u.id AS CHAR) = o.user_id
-- u.id를 VARCHAR로 변환 → orders.user_id(VARCHAR)와 타입 일치
-- orders.user_id에 함수 없음 → 인덱스 사용 가능
GROUP BY u.id;
```

단, 방법 2는 임시방편이며 `u.id`에 함수가 적용되어 `users` 쪽 최적화가 달라질 수 있습니다. 스키마 타입 통일이 가장 좋은 해결책입니다.

</details>

---

**Q2.** JPA를 사용하는 프로젝트에서 `utf8` 테이블과 `utf8mb4` 테이블 간의 JOIN이 느리다는 것을 발견했다. DB 스키마를 바꾸지 않고 단기적으로 적용할 수 있는 방법과, 장기적 해결책을 제시하라.

<details>
<summary>해설 보기</summary>

**단기 해결 — Native Query에서 CONVERT() 명시:**
```java
@Query(value = """
    SELECT a.id, b.id
    FROM utf8_table a
    JOIN utf8mb4_table b
      ON CONVERT(a.name USING utf8mb4) COLLATE utf8mb4_unicode_ci = b.name
    """, nativeQuery = true)
List<Object[]> findMatches();
```
단, `utf8_table.name` 쪽 인덱스는 여전히 무력화됩니다.

**단기 해결 — 애플리케이션에서 분리 조회:**
```java
// utf8mb4 테이블에서 먼저 조회 후 메모리에서 매핑
List<Utf8mb4Entity> items = utf8mb4Repo.findAll();
List<Long> ids = items.stream().map(e -> e.getId()).collect(toList());
List<Utf8Table> matched = utf8Repo.findAllByNameIn(ids);
// DB JOIN 대신 애플리케이션 조인 → 문자셋 변환 없음
// 단, 결과 건수가 작을 때만 유효
```

**장기 해결 — 스키마 마이그레이션:**
```sql
-- 단계적 마이그레이션 (pt-online-schema-change 권장)
ALTER TABLE utf8_table CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-- 또는 pt-osc를 사용해 무중단으로 마이그레이션
```

신규 테이블은 처음부터 `utf8mb4`와 `utf8mb4_0900_ai_ci`(MySQL 8.0 기본)으로 통일하는 것이 가장 근본적인 해결책입니다.

</details>

---

<div align="center">

**[⬅️ 윈도우 함수](./05-window-function-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: 실행계획 개선 사례 ➡️](./07-explain-improvement-cases.md)**

</div>
