# JSON 타입과 Generated Column — MySQL 8.0 JSON 인덱싱 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JSON 컬럼 자체에 인덱스를 직접 걸 수 없는 이유는?
- Generated Column으로 JSON 경로를 가상 컬럼으로 추출해 인덱스를 거는 방법은?
- `JSON_EXTRACT()` vs `->` 연산자 차이는?
- Functional Index와 Generated Column Index의 차이는?
- JSON 타입 vs 정규화된 컬럼의 성능 트레이드오프는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### JSON 컬럼에 인덱스 없이 조회하면 Full Scan

```
실제 이슈:
  주문 테이블에 JSON 컬럼으로 배송 정보 저장
  CREATE TABLE orders (
      id   BIGINT AUTO_INCREMENT PRIMARY KEY,
      info JSON  -- {"city": "Seoul", "zip": "12345", "method": "express"}
  );

  배송 도시별 조회:
  SELECT * FROM orders WHERE JSON_EXTRACT(info, '$.city') = 'Seoul';
  → EXPLAIN: type=ALL, Extra: Using where
  → 100만 건 Full Scan!

  JSON 컬럼에 직접 인덱스:
  CREATE INDEX idx_city ON orders ((JSON_EXTRACT(info, '$.city')));
  → MySQL 8.0: Functional Index로 가능!
  또는 Generated Column:
  ALTER TABLE orders ADD COLUMN city VARCHAR(50)
    GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(info, '$.city'))) VIRTUAL;
  CREATE INDEX idx_city ON orders (city);
  → EXPLAIN: type=ref, key=idx_city
```

---

## 😱 흔한 실수 (Before)

### 1. JSON 컬럼에 인덱스 없이 조건 검색

```sql
-- Before: JSON 조건 검색에 인덱스 없음
SELECT * FROM orders WHERE info->>'$.city' = 'Seoul';
-- EXPLAIN: type=ALL (Full Scan)
-- 100만 건 이상에서 수 초 소요

-- 해결: Generated Column + 인덱스
ALTER TABLE orders
  ADD COLUMN city VARCHAR(50)
    GENERATED ALWAYS AS (info->>'$.city') VIRTUAL;
CREATE INDEX idx_city ON orders (city);
-- 이후 동일 쿼리 자동으로 인덱스 사용
```

### 2. JSON_EXTRACT() 반환값 타입 무시

```sql
-- Before: 숫자 비교에 타입 불일치
SELECT * FROM orders WHERE JSON_EXTRACT(info, '$.price') > 10000;
-- JSON_EXTRACT는 JSON 타입 반환 (문자열로 비교될 수 있음)
-- '9999' > '10000' → 문자열 비교로 '9' > '1' → 오작동!

-- 해결: JSON_UNQUOTE 또는 CAST
SELECT * FROM orders WHERE CAST(JSON_EXTRACT(info, '$.price') AS UNSIGNED) > 10000;
-- 또는 ->>(double-arrow)
SELECT * FROM orders WHERE info->>'$.price' + 0 > 10000;  -- 암묵적 변환

-- 가장 안전: Generated Column으로 타입 명시
ALTER TABLE orders
  ADD COLUMN price DECIMAL(10,2)
    GENERATED ALWAYS AS (CAST(info->>'$.price' AS DECIMAL(10,2))) STORED;
```

### 3. JSON_SEARCH를 배열 인덱스 대체로 착각

```sql
-- Before: 대용량 배열 검색에 Full Scan
SELECT * FROM orders WHERE JSON_SEARCH(info->'$.tags', 'one', 'urgent') IS NOT NULL;
-- info 예: {"tags": ["urgent", "gift", "fragile"]}
-- JSON_SEARCH: Full Scan 피할 수 없음 (배열 인덱스 지원 없음)

-- 해결: 배열 검색이 잦으면 정규화
CREATE TABLE order_tags (
    order_id BIGINT NOT NULL,
    tag      VARCHAR(50) NOT NULL,
    INDEX idx_tag (tag)
);
-- tag별 인덱스 탐색 가능
```

---

## ✨ 올바른 접근 (After)

```
JSON 컬럼 인덱싱 방법 3가지:

방법 1: Generated Column (VIRTUAL) + Index
  ALTER TABLE orders ADD COLUMN city VARCHAR(50)
    GENERATED ALWAYS AS (info->>'$.city') VIRTUAL;
  CREATE INDEX idx_city ON orders (city);

  특징:
    VIRTUAL: 디스크에 저장 안 함, 조회 시 계산
    인덱스만 디스크에 저장
    INSERT/UPDATE 시 자동으로 인덱스 갱신
    SELECT info->>'$.city'와 SELECT city 모두 인덱스 사용

방법 2: Generated Column (STORED) + Index
  ALTER TABLE orders ADD COLUMN price DECIMAL(10,2)
    GENERATED ALWAYS AS (CAST(info->>'$.price' AS DECIMAL(10,2))) STORED;
  CREATE INDEX idx_price ON orders (price);

  특징:
    STORED: 디스크에 저장 (추가 공간 필요)
    조회 시 계산 불필요 → 읽기 빠름
    쓰기 시 추가 저장 비용

방법 3: Functional Index (MySQL 8.0.13+)
  CREATE INDEX idx_city ON orders ((info->>'$.city'));

  특징:
    ALTER TABLE 없이 인덱스 생성
    내부적으로 HIDDEN Generated Column 자동 생성
    간편하지만 Generated Column보다 제어력 낮음

선택 기준:
  단순 문자열/숫자 경로 + 자주 조회: Functional Index (간편)
  타입 변환 필요 or 정확한 제어: Generated Column VIRTUAL
  읽기 중심 + 계산 비용 절감: Generated Column STORED
```

---

## 🔬 내부 동작 원리

### 1. MySQL JSON 타입 내부 저장 방식

```
JSON 타입 저장:
  MySQL 8.0: JSON을 바이너리 포맷으로 저장
  (텍스트 문자열이 아닌 최적화된 바이너리)

  JSON 바이너리 포맷의 특징:
    빠른 경로 접근 (O(log n) 수준)
    부분 업데이트 지원 (JSON_SET/JSON_REPLACE)
    크기: 텍스트 JSON보다 효율적

  JSON 바이너리 구조:
    헤더: 타입, 키 오프셋, 값 오프셋
    키 섹션: 키 이름들 (정렬됨)
    값 섹션: 값들

  부분 업데이트:
    JSON_SET(info, '$.city', 'Busan')
    이전 값 크기와 같거나 작으면 → 인플레이스 업데이트
    크면 → 새 바이너리 생성 (전체 재작성)

JSON vs VARCHAR(텍스트):
  JSON: 바이너리 저장 → 경로 접근 빠름, 유효성 검사 자동
  VARCHAR JSON 문자열: 텍스트 파싱 필요 → 느림
  → MySQL에서는 JSON 컬럼 사용 권장
```

### 2. JSON 경로 연산자 비교

```sql
-- JSON_EXTRACT(): JSON 타입 반환 (따옴표 포함)
SELECT JSON_EXTRACT('{"name": "Alice"}', '$.name');
-- 결과: "Alice" (따옴표 포함된 JSON 타입)

-- ->(arrow): JSON_EXTRACT의 단축 표기
SELECT '{"name": "Alice"}'->'$.name';
-- 결과: "Alice" (동일)

-- ->>(double-arrow): JSON_UNQUOTE(JSON_EXTRACT) 단축
SELECT '{"name": "Alice"}'->>'$.name';
-- 결과: Alice (따옴표 없는 문자열)

-- 비교 시 차이:
-- JSON 타입: '"Alice"' = '"Alice"' (따옴표 포함 비교)
-- 문자열 타입: 'Alice' = 'Alice'

-- 인덱스와 타입 일치 중요:
-- Generated Column: VARCHAR(50) GENERATED AS (info->>'$.city')
-- 쿼리: WHERE city = 'Seoul'        → 인덱스 사용 ✅
-- 쿼리: WHERE info->>'$.city' = 'Seoul' → Optimizer가 city로 변환 후 인덱스 사용 ✅
-- 쿼리: WHERE JSON_EXTRACT(info, '$.city') = 'Seoul' → 타입 불일치 가능 ⚠️
```

### 3. Generated Column VIRTUAL vs STORED

```sql
-- VIRTUAL Generated Column
ALTER TABLE orders ADD COLUMN city_virtual VARCHAR(50)
  GENERATED ALWAYS AS (info->>'$.city') VIRTUAL;

-- 동작:
-- 디스크 저장: NO (메타데이터만 저장)
-- SELECT city_virtual: 조회 시 JSON_UNQUOTE(JSON_EXTRACT) 계산
-- 인덱스 추가 시: 인덱스는 디스크에 저장됨
-- INSERT/UPDATE: 자동으로 인덱스 갱신 (컬럼 값은 저장 안 함)
-- 디스크 공간: 인덱스 크기만큼만 추가

-- STORED Generated Column
ALTER TABLE orders ADD COLUMN price_stored DECIMAL(10,2)
  GENERATED ALWAYS AS (CAST(info->>'$.price' AS DECIMAL(10,2))) STORED;

-- 동작:
-- 디스크 저장: YES (실제 계산된 값을 저장)
-- SELECT price_stored: 저장된 값 반환 (계산 없음)
-- INSERT/UPDATE: 자동 계산 후 저장
-- 디스크 공간: 컬럼 크기 × 행 수만큼 추가

-- 성능 비교:
-- 읽기: STORED > VIRTUAL (STORED는 계산 없음)
-- 쓰기: VIRTUAL >= STORED (STORED는 추가 저장)
-- 공간: VIRTUAL << STORED

-- 언제 STORED 선택:
-- 계산 비용이 높은 표현식 (복잡한 JSON 경로)
-- 집계/통계 쿼리에서 자주 사용
-- STORED만 지원하는 기능 (파티션 키, 외래키 등)
```

### 4. Functional Index 내부 동작

```sql
-- Functional Index 생성
CREATE INDEX idx_city ON orders ((info->>'$.city'));

-- 내부적으로 Hidden Generated Column 자동 생성:
-- `!hidden!info->>'$.city'` VIRTUAL GENERATED ALWAYS AS (info->>'$.city')

-- 인덱스 사용 확인:
EXPLAIN SELECT * FROM orders WHERE info->>'$.city' = 'Seoul'\G
-- key: idx_city (Functional Index 사용)
-- type: ref

-- Functional Index vs Generated Column + Index 차이:
-- Functional Index:
--   장점: DDL 한 줄로 간편
--   단점: Hidden Column 직접 접근 불가, Generated Column의 모든 기능 제한
-- Generated Column + Index:
--   장점: 컬럼을 명시적으로 SELECT, 타입 변환 명확
--   단점: 두 단계 (ADD COLUMN + CREATE INDEX)

-- MySQL 8.0.13+ 공식 권장: Functional Index (간편)
-- 세밀한 제어 필요: Generated Column
```

### 5. JSON 배열 처리

```sql
-- JSON 배열 인덱싱 (MySQL 8.0 다중값 인덱스 - Experimental)
CREATE TABLE orders_tags (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    tags JSON
    -- {"tags": ["express", "gift", "fragile"]}
);

-- MySQL 8.0.17+: Multi-Valued Index (배열 요소 인덱싱)
CREATE INDEX idx_tags ON orders_tags ((CAST(tags->'$' AS CHAR(50) ARRAY)));

-- 사용:
SELECT * FROM orders_tags WHERE 'express' MEMBER OF (tags->'$');
-- EXPLAIN: ref, Multi-Valued Index 사용

-- 주의: 아직 실험적 기능, 안정성 고려 필요
-- 안정적 대안: 별도 테이블로 배열 정규화
CREATE TABLE order_tags (
    order_id BIGINT NOT NULL,
    tag VARCHAR(50) NOT NULL,
    INDEX idx_tag (order_id, tag)
);
```

---

## 💻 실전 실험

### 실험 1: JSON Full Scan vs Generated Column Index 비교

```sql
-- 테스트 테이블
CREATE TABLE json_perf (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    data JSON
) ENGINE=InnoDB;

-- 데이터 삽입 (10만 건)
INSERT INTO json_perf (data)
SELECT JSON_OBJECT(
    'city', ELT(FLOOR(RAND()*5)+1, 'Seoul','Busan','Daegu','Incheon','Daejeon'),
    'amount', FLOOR(RAND()*100000),
    'status', ELT(FLOOR(RAND()*3)+1, 'PAID','PENDING','SHIPPED')
)
FROM (SELECT @r := @r+1 FROM information_schema.columns a, information_schema.columns b, (SELECT @r:=0) r LIMIT 100000) t;

-- 인덱스 없이 조회
EXPLAIN ANALYZE SELECT * FROM json_perf WHERE data->>'$.city' = 'Seoul';
-- type=ALL, Full Scan

-- Generated Column + Index 추가
ALTER TABLE json_perf
  ADD COLUMN city VARCHAR(20)
    GENERATED ALWAYS AS (data->>'$.city') VIRTUAL,
  ADD INDEX idx_city (city);

-- 인덱스 있는 조회
EXPLAIN ANALYZE SELECT * FROM json_perf WHERE data->>'$.city' = 'Seoul';
-- type=ref, idx_city 사용

-- 실행 시간 비교
SET @t = SYSDATE(6);
SELECT COUNT(*) FROM json_perf WHERE data->>'$.city' = 'Seoul';
SELECT CONCAT('조회 시간: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

DROP TABLE json_perf;
```

### 실험 2: Functional Index vs Generated Column

```sql
CREATE TABLE fi_test (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    data JSON
);

INSERT INTO fi_test (data)
SELECT JSON_OBJECT('val', FLOOR(RAND()*1000))
FROM (SELECT @r:=@r+1 FROM information_schema.columns, (SELECT @r:=0) r LIMIT 10000) t;

-- Functional Index
CREATE INDEX idx_val_fi ON fi_test ((CAST(data->>'$.val' AS UNSIGNED)));

-- Generated Column + Index
ALTER TABLE fi_test
  ADD COLUMN val_gc INT UNSIGNED
    GENERATED ALWAYS AS (CAST(data->>'$.val' AS UNSIGNED)) VIRTUAL;
CREATE INDEX idx_val_gc ON fi_test (val_gc);

-- 두 방법 모두 인덱스 사용 확인
EXPLAIN SELECT * FROM fi_test WHERE CAST(data->>'$.val' AS UNSIGNED) > 500;
EXPLAIN SELECT * FROM fi_test WHERE val_gc > 500;

DROP TABLE fi_test;
```

---

## 📊 성능/비용 비교

```
JSON 처리 방식별 성능 비교 (100만 건):

JSON 컬럼 인덱스 없음:
  SELECT * WHERE info->>'$.city' = 'Seoul'
  → Full Scan: 100만 건
  예상 시간: 1~3초

Functional Index:
  CREATE INDEX ON ((info->>'$.city'))
  → Index Scan: ~20만 건 (Seoul이 20%라면)
  예상 시간: 0.01~0.1초

Generated Column VIRTUAL + Index:
  동일 성능, 컬럼을 직접 SELECT 가능
  디스크 추가: 인덱스 크기만 (VIRTUAL이라 컬럼 저장 없음)

Generated Column STORED + Index:
  읽기: VIRTUAL과 동일 (인덱스 경유)
  쓰기: VIRTUAL보다 약간 느림 (컬럼 저장)
  디스크 추가: 인덱스 + 컬럼 크기

정규화된 컬럼:
  city VARCHAR(20) 일반 컬럼 + 인덱스
  → 가장 빠른 읽기 (JSON 파싱 없음)
  → 쓰기 시 city 컬럼 별도 관리 필요
```

---

## ⚖️ 트레이드오프

```
JSON 컬럼 vs 정규화된 컬럼:

JSON 컬럼:
  ✅ 스키마 변경 없이 새 필드 추가 가능 (유연성)
  ✅ 비정형 데이터 저장 용이
  ✅ 애플리케이션 레벨 스키마 관리
  ❌ 인덱스 추가 위해 Generated Column 별도 작업
  ❌ JSON 파싱 오버헤드 (VIRTUAL Generated Column 포함)
  ❌ 타입 강제 없음 (잘못된 타입 저장 가능)
  ❌ 쿼리 복잡도 증가

정규화된 컬럼:
  ✅ 빠른 인덱스 접근 (파싱 없음)
  ✅ 타입 강제 (DB 레벨 검증)
  ✅ 단순한 쿼리
  ❌ 스키마 변경 시 ALTER TABLE 필요
  ❌ 사전에 모든 필드 정의 필요

JSON 선택 기준:
  ① 필드가 자주 추가/변경되는 반정형 데이터
  ② 설정 정보, 이벤트 메타데이터 등
  ③ 자주 조회하는 필드는 Generated Column + Index

정규화 선택 기준:
  ① 항상 조회하는 필드 (name, email, status 등)
  ② 조인 키로 사용되는 필드
  ③ 집계/정렬에 사용되는 필드

실무 패턴:
  핵심 필드: 정규화된 컬럼
  부가 정보: JSON 컬럼 (Generated Column으로 자주 검색하는 것만 인덱싱)
```

---

## 📌 핵심 정리

```
JSON 타입과 인덱싱 핵심:

JSON 컬럼에 직접 인덱스 불가:
  JSON은 가변 구조 → B-Tree 인덱스 키 정의 불가
  경로 표현식 기반 Functional Index로 우회

인덱싱 방법:
  Functional Index (간편):
    CREATE INDEX idx ON t ((info->>'$.city'));
  Generated Column + Index (명시적):
    ADD COLUMN city VARCHAR(20) GENERATED AS (info->>'$.city') VIRTUAL;
    ADD INDEX idx_city (city);

연산자 차이:
  ->  : JSON_EXTRACT, JSON 타입 반환 (따옴표 포함)
  ->> : JSON_UNQUOTE(JSON_EXTRACT), 문자열 반환

VIRTUAL vs STORED:
  VIRTUAL: 디스크 저장 안 함, 조회 시 계산
  STORED: 디스크 저장, 읽기 빠름 (계산 생략)

설계 원칙:
  자주 검색하는 JSON 경로 → Generated Column + Index
  배열 검색 → Multi-Valued Index (실험적) 또는 정규화
  JSON 필드의 수가 많아지면 → 정규화 재검토
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 JSON 컬럼을 포함한 테이블에서 `city`별, `amount` 범위별 조회가 빈번하다. 최적의 인덱싱 전략을 설계하라.

```sql
CREATE TABLE user_profiles (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    profile JSON
    -- {"city": "Seoul", "age": 28, "tags": ["vip","active"]}
);
```

<details>
<summary>해설 보기</summary>

**설계**:
```sql
-- 자주 검색되는 city, age에 Generated Column + Index
ALTER TABLE user_profiles
  ADD COLUMN city VARCHAR(50)
    GENERATED ALWAYS AS (profile->>'$.city') VIRTUAL,
  ADD COLUMN age TINYINT UNSIGNED
    GENERATED ALWAYS AS (CAST(profile->>'$.age' AS UNSIGNED)) VIRTUAL;

-- 단일 인덱스
CREATE INDEX idx_city ON user_profiles (city);
CREATE INDEX idx_age ON user_profiles (age);

-- 복합 인덱스 (city + age 동시 조건이 많을 때)
CREATE INDEX idx_city_age ON user_profiles (city, age);

-- 태그 검색 (배열): 정규화 권장
CREATE TABLE user_tags (
    user_profile_id BIGINT NOT NULL,
    tag VARCHAR(50) NOT NULL,
    INDEX idx_tag (tag),
    INDEX idx_user_tag (user_profile_id, tag)
);

-- 인덱스 사용 확인
EXPLAIN SELECT user_id FROM user_profiles
WHERE city = 'Seoul' AND age BETWEEN 20 AND 30;
-- idx_city_age 사용 확인
```

</details>

---

**Q2.** Functional Index와 Generated Column + Index를 각각 언제 선택하는가? 차이점을 설명하라.

<details>
<summary>해설 보기</summary>

**Functional Index 선택 상황**:
- 빠른 인덱스 추가가 필요할 때 (ALTER TABLE 한 줄로 끝냄)
- Generated Column을 SELECT에서 직접 사용할 필요가 없을 때
- 단순한 경로 표현식 (복잡한 타입 변환 불필요)

```sql
CREATE INDEX idx_city ON orders ((info->>'$.city'));
```

**Generated Column + Index 선택 상황**:
- 컬럼을 직접 SELECT에서 사용해야 할 때 (`SELECT city FROM orders`)
- 명시적인 타입 변환이 필요할 때 (`CAST(... AS DECIMAL)`)
- 다른 Generated Column과 조합하여 복합 인덱스 구성 시
- 컬럼 기반 집계 쿼리 (GROUP BY city)

```sql
ALTER TABLE orders ADD COLUMN city VARCHAR(50)
  GENERATED ALWAYS AS (info->>'$.city') VIRTUAL;
CREATE INDEX idx_city ON orders (city);

-- 이후 직접 사용 가능:
SELECT city, COUNT(*) FROM orders GROUP BY city;  -- 인덱스 사용
```

**핵심 차이**: Functional Index는 Hidden Column으로 처리되어 SQL에서 직접 참조 불가. Generated Column은 일반 컬럼처럼 사용 가능.

</details>

---

<div align="center">

**[⬅️ 데이터 타입 선택](./01-data-type-selection.md)** | **[홈으로 🏠](../README.md)** | **[다음: 정규화 vs 비정규화 ➡️](./03-normalization-vs-denormalization.md)**

</div>
