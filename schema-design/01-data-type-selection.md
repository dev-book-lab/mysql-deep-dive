# 데이터 타입 선택 전략 — INT vs BIGINT, DATETIME vs TIMESTAMP, CHAR vs VARCHAR

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- BIGINT AUTO_INCREMENT가 소진되면 어떤 일이 발생하는가?
- TIMESTAMP의 2038년 문제와 타임존 자동 변환 동작은?
- VARCHAR와 CHAR의 저장 방식 차이가 인덱스에 미치는 영향은?
- DECIMAL vs DOUBLE을 금액 계산에서 선택하는 기준은?
- 각 데이터 타입이 인덱스 크기와 성능에 미치는 영향은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 잘못된 타입 선택은 나중에 고치기가 매우 어렵다

```
실제 이슈:
  이슈 1: INT AUTO_INCREMENT 소진
    소셜 서비스 likes 테이블: INT(4바이트) AUTO_INCREMENT
    일 1,000만 건 × 200일 = 약 20억 건
    INT 최대값: 2,147,483,647 (약 21억)
    → 소진 시: ERROR 1062 Duplicate entry for key 'PRIMARY'
    → 새 like 등록 불가, 서비스 다운

    BIGINT로 변경:
      ALTER TABLE likes MODIFY id BIGINT AUTO_INCREMENT;
      → 대형 테이블 ALTER = 수 시간 작업

  이슈 2: TIMESTAMP 타임존 이슈
    서버 timezone을 UTC → Asia/Seoul로 변경
    TIMESTAMP 컬럼의 모든 값이 9시간 변환됨!
    "2024-01-01 00:00:00"이 "2024-01-01 09:00:00"으로 읽힘
    → 데이터 오염처럼 보임 (실제로는 저장값은 같음)

타입 선택은 처음에 제대로 해야 한다
```

---

## 😱 흔한 실수 (Before)

### 1. INT PK를 사용하다가 소진

```sql
-- Before: INT AUTO_INCREMENT PK
CREATE TABLE user_events (
    id         INT AUTO_INCREMENT PRIMARY KEY,  -- 최대 21억
    user_id    INT NOT NULL,
    event_type VARCHAR(50),
    created_at DATETIME
) ENGINE=InnoDB;

-- 일 1,000만 건 서비스:
-- 21억 / 10,000,000 = 215일 → 7개월 만에 소진
-- 소진 시: 새 행 INSERT 불가
-- INSERT INTO user_events ... → Error 1062: Duplicate entry '2147483647'

-- 올바른 설계:
CREATE TABLE user_events (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,  -- 최대 9.2 × 10^18
    ...
```

### 2. 금액 컬럼에 FLOAT/DOUBLE 사용

```sql
-- Before: 금액에 부동소수점 사용
CREATE TABLE payments (
    id     INT AUTO_INCREMENT PRIMARY KEY,
    amount DOUBLE NOT NULL  -- 부동소수점!
);

INSERT INTO payments (amount) VALUES (0.1 + 0.2);
SELECT amount FROM payments;
-- 출력: 0.30000000000000004  (부동소수점 오차!)

SELECT SUM(amount) FROM payments;
-- 100만 건 누적 시 오차가 수백 원 ~ 수만 원 수준으로 증가

-- 올바른 설계:
CREATE TABLE payments (
    id     BIGINT AUTO_INCREMENT PRIMARY KEY,
    amount DECIMAL(15, 2) NOT NULL  -- 정확한 고정소수점
);
```

### 3. 모든 문자열에 VARCHAR(255) 남발

```sql
-- Before: 생각 없이 VARCHAR(255)
CREATE TABLE users (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    gender     VARCHAR(255),  -- 실제로는 'M'/'F'/'N' 3가지
    country    VARCHAR(255),  -- 최대 2자리 국가코드
    status     VARCHAR(255),  -- 5가지 상태값
    email      VARCHAR(255)   -- 이건 괜찮음
);

-- 문제:
-- gender에 VARCHAR(255) → 인덱스 키 크기 낭비
-- 잘못된 값 저장 가능 ('MALE' 대신 'MALAKASD' 입력)
-- CHAR(1) 또는 ENUM이 더 적절

-- 올바른 설계:
CREATE TABLE users (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    gender  ENUM('M', 'F', 'N') NOT NULL DEFAULT 'N',
    country CHAR(2),              -- 항상 2자리
    status  VARCHAR(20) NOT NULL, -- 가변이지만 짧음
    email   VARCHAR(255)          -- 실제로 긴 값
);
```

---

## ✨ 올바른 접근 (After)

```
데이터 타입 선택 기준:

정수형:
  TINYINT  (1바이트): 0~255 또는 -128~127
  SMALLINT (2바이트): 0~65535 또는 -32768~32767
  MEDIUMINT(3바이트): 0~16,777,215
  INT      (4바이트): 0~4,294,967,295 (약 43억)
  BIGINT   (8바이트): 0~18,446,744,073,709,551,615

  PK 기본 선택: BIGINT (향후 확장성)
  상태 코드, 소량 분류: TINYINT/SMALLINT
  INT를 PK로 쓸 때: 최대 43억 초과 여부 계산 필수

부동소수점 vs 고정소수점:
  FLOAT/DOUBLE: 부동소수점 오차 있음 → 금액, 수량 절대 사용 금지
  DECIMAL(M, D): 정확한 고정소수점 → 금액 필수
    M: 전체 자릿수 (최대 65)
    D: 소수점 자릿수
    예: DECIMAL(15, 2) = 최대 9,999,999,999,999.99

날짜/시간:
  DATETIME : 타임존 변환 없음, '1000-01-01'~'9999-12-31', 5~8바이트
  TIMESTAMP: UTC 저장 + 타임존 자동 변환, '1970-01-01'~'2038-01-19', 4바이트
  DATE     : 날짜만 (3바이트)
  TIME     : 시간만 (3바이트)

문자열:
  CHAR(N)    : 고정 길이, 짧은 코드값에 적합
  VARCHAR(N) : 가변 길이, 1~2바이트 오버헤드
  TEXT       : 큰 텍스트, 인덱스 제한 있음
  ENUM/SET   : 제한된 집합, 인덱스 효율 좋음 (숫자로 저장)
```

---

## 🔬 내부 동작 원리

### 1. TIMESTAMP vs DATETIME 상세 비교

```
TIMESTAMP 동작:
  저장: 입력값을 UTC로 변환 후 저장 (4바이트 유닉스 타임스탬프)
  조회: 저장된 UTC 값을 @@session.time_zone으로 변환 후 반환

  예시 (서버 timezone = Asia/Seoul, UTC+9):
    INSERT INTO t VALUES ('2024-01-15 10:00:00');
    -- 저장값: 2024-01-15 01:00:00 (UTC, 9시간 빼기)

    SELECT ts FROM t;
    -- 반환: 2024-01-15 10:00:00 (Asia/Seoul로 변환)

    -- 서버 timezone 변경 시:
    SET time_zone = 'UTC';
    SELECT ts FROM t;
    -- 반환: 2024-01-15 01:00:00 (UTC 그대로)
    → 같은 데이터인데 다르게 보임!

TIMESTAMP 2038년 문제:
  최대 저장 가능 시각: 2038-01-19 03:14:07 UTC
  이후 값 저장 시도 → 오류 또는 0값 저장
  2038년까지 10년 이상 남았지만
  "created_at TIMESTAMP" → 2038년 운영 중인 서비스 영향

DATETIME 동작:
  저장: 입력값을 그대로 저장 (타임존 변환 없음)
  조회: 저장한 값 그대로 반환

  장점: 타임존 혼란 없음, 2038년 문제 없음 (9999년까지)
  단점: 타임존 정보 미포함, 4바이트 아닌 5~8바이트

선택 기준:
  타임존 이슈 중요 (글로벌 서비스): DATETIME + 별도 timezone 관리
  빠른 타임존 변환 필요: TIMESTAMP
  2038년 이후 서비스 지속: DATETIME 필수
  단순 국내 서비스: TIMESTAMP 가능 (timezone = KST 고정)
```

### 2. CHAR vs VARCHAR 내부 저장 방식

```
CHAR(N):
  항상 N바이트 공간 할당
  짧은 값은 공백(스페이스)으로 패딩
  CHAR(10) = 'AB' → 'AB        ' (8 스페이스 패딩)

  장점:
    고정 크기 → 행 이동 없이 직접 접근 (UPDATE 빠름)
    인덱스 키 예측 가능
  단점:
    공간 낭비 (VARCHAR보다 클 수 있음)
  적합: 국가코드(CHAR(2)), 성별(CHAR(1)), 해시값(CHAR(32))

VARCHAR(N):
  실제 저장된 길이만큼만 사용
  1바이트(N≤255) 또는 2바이트(N>255) 길이 정보 추가
  VARCHAR(100) = 'AB' → 3바이트 (길이1 + 데이터2)

  장점: 공간 효율적
  단점:
    UPDATE로 값이 길어지면 행 재배치(row overflow) 가능
    인덱스 키 크기 가변적
  적합: 이메일, 이름, 설명

인덱스에서의 차이:
  CHAR(50) 인덱스 키: 항상 50바이트
  VARCHAR(50) 인덱스 키: 실제 길이만큼
  → CHAR 인덱스가 약간 더 크지만 일정한 크기
  → VARCHAR 인덱스는 작지만 가변적

UTF-8mb4 고려:
  CHAR(10): 최대 10 × 4바이트 = 40바이트 (ASCII이면 10바이트)
  VARCHAR(255): 실제 글자 수 × 최대 4바이트
  → 영문만: VARCHAR가 효율적
  → 한국어/이모지: 멀티바이트 고려 필요
```

### 3. DECIMAL 내부 저장

```
DECIMAL(M, D) 저장:
  정수 부분과 소수 부분을 각각 압축하여 저장
  9자리마다 4바이트 사용

  DECIMAL(15, 2):
    정수 부분: 13자리 → 4 + 4 = 8바이트 (9자리 + 4자리)
    소수 부분: 2자리 → 1바이트
    합계: 9바이트

  DECIMAL(10, 2):
    정수 부분: 8자리 → 4바이트
    소수 부분: 2자리 → 1바이트
    합계: 5바이트

금액 컬럼 설계 권장:
  원화(KRW): DECIMAL(15, 2) (최대 999,999,999,999.99원)
  달러(USD): DECIMAL(12, 4) (최대 99,999,999.9999달러)

DECIMAL vs DOUBLE 비교:
  DOUBLE: 8바이트, IEEE 754 부동소수점, ~15자리 정밀도
           0.1 + 0.2 = 0.30000000000000004 (오차 발생)
  DECIMAL: 압축 저장, 정확한 십진수 산술
           0.1 + 0.2 = 0.3 (정확)
  → 금액은 반드시 DECIMAL
```

### 4. ENUM 내부 동작

```sql
-- ENUM 내부 저장:
CREATE TABLE orders (
    status ENUM('PENDING', 'PAID', 'SHIPPED', 'CANCELLED', 'REFUNDED')
);

-- 내부적으로 숫자로 저장:
-- PENDING=1, PAID=2, SHIPPED=3, CANCELLED=4, REFUNDED=5
-- 값이 256개 미만: 1바이트, 256개 이상: 2바이트

-- 장점:
-- VARCHAR('PENDING') 7바이트 → ENUM 1바이트 저장
-- 인덱스 크기 감소
-- 유효값 자동 검증 (정의 외 값 INSERT 시 오류)

-- 단점:
-- 값 추가/삭제 시 ALTER TABLE 필요 (DDL 변경)
-- 순서 기반 비교: 'PENDING' < 'PAID' (알파벳 아닌 정의 순서)
-- ORM에서 추가 설정 필요

-- 실무 선택:
-- 값이 거의 안 바뀌는 상태: ENUM (인덱스 효율)
-- 값이 자주 바뀌는 상태: VARCHAR + CHECK CONSTRAINT (MySQL 8.0)
-- 비즈니스 도메인 코드: TINYINT + 별도 코드 테이블 (확장성)
```

---

## 💻 실전 실험

### 실험 1: TIMESTAMP 타임존 동작 확인

```sql
-- 타임존 동작 테스트
CREATE TABLE tz_test (
    id   INT AUTO_INCREMENT PRIMARY KEY,
    ts   TIMESTAMP,
    dt   DATETIME
);

SET time_zone = 'Asia/Seoul';
INSERT INTO tz_test (ts, dt) VALUES (NOW(), NOW());

-- Asia/Seoul에서 조회
SELECT ts, dt FROM tz_test;
-- ts: 2024-01-15 10:00:00
-- dt: 2024-01-15 10:00:00

-- UTC로 변경 후 조회
SET time_zone = 'UTC';
SELECT ts, dt FROM tz_test;
-- ts: 2024-01-15 01:00:00  ← 9시간 차이 (UTC 기반)
-- dt: 2024-01-15 10:00:00  ← 변화 없음 (타임존 무관)

DROP TABLE tz_test;
```

### 실험 2: DECIMAL vs DOUBLE 정밀도 비교

```sql
CREATE TABLE precision_test (
    d DOUBLE,
    dec DECIMAL(10, 2)
);

INSERT INTO precision_test VALUES (0.1 + 0.2, 0.1 + 0.2);
SELECT d, dec FROM precision_test;
-- d:   0.30000000000000004  (오차!)
-- dec: 0.30                 (정확)

-- 누적 오차
INSERT INTO precision_test VALUES (1.00, 1.00);
-- 10만 건 SUM:
SELECT SUM(d), SUM(dec)
FROM (SELECT 1.00 AS d, 1.00 AS dec FROM information_schema.columns LIMIT 100000) t;
-- SUM(d): 99999.99999XXXX (오차 축적)
-- SUM(dec): 100000.00 (정확)

DROP TABLE precision_test;
```

### 실험 3: CHAR vs VARCHAR 인덱스 크기 비교

```sql
-- CHAR vs VARCHAR 인덱스 크기
CREATE TABLE char_test (
    id    INT AUTO_INCREMENT PRIMARY KEY,
    code  CHAR(20),
    INDEX idx_code (code)
);

CREATE TABLE varchar_test (
    id    INT AUTO_INCREMENT PRIMARY KEY,
    code  VARCHAR(20),
    INDEX idx_code (code)
);

-- 같은 데이터 삽입 (짧은 값)
INSERT INTO char_test (code)
SELECT REPEAT('A', 3)
FROM information_schema.columns LIMIT 10000;

INSERT INTO varchar_test (code)
SELECT REPEAT('A', 3)
FROM information_schema.columns LIMIT 10000;

-- 인덱스 크기 비교
SELECT
    table_name,
    index_name,
    index_length / 1024 AS index_kb
FROM information_schema.statistics
WHERE table_schema = DATABASE()
  AND index_name = 'idx_code'
  AND table_name IN ('char_test', 'varchar_test');
-- char_test의 인덱스가 큼 (공백 패딩으로 항상 20바이트)

DROP TABLE char_test, varchar_test;
```

---

## 📊 성능/비용 비교

```
데이터 타입별 크기와 성능 영향:

정수 타입:
  TINYINT: 1바이트 → 인덱스 키 1바이트
  INT:     4바이트 → 인덱스 키 4바이트
  BIGINT:  8바이트 → 인덱스 키 8바이트

  PK를 INT→BIGINT 변경 시:
  PK 인덱스: 4 → 8바이트 (2배)
  FK 인덱스: 동일 비율 증가
  100만 건 테이블: 인덱스 크기 약 8MB → 16MB 증가

날짜/시간:
  TIMESTAMP: 4바이트 (인덱스 4바이트)
  DATETIME:  5바이트 (인덱스 5바이트)
  차이: 미미 (1바이트)

문자열:
  CHAR(10) 값 'AB': 10바이트 (패딩)
  VARCHAR(10) 값 'AB': 3바이트 (1 + 2)
  단, 인덱스 B-Tree: CHAR는 고정, VARCHAR는 실제 길이

  대용량 테이블 영향:
  1억 건, status CHAR(20) vs VARCHAR(20) with avg 8자:
    CHAR 인덱스: ~2GB
    VARCHAR 인덱스: ~1GB (8/20 절감)
```

---

## ⚖️ 트레이드오프

```
주요 타입 선택 트레이드오프:

INT vs BIGINT (PK):
  INT: 4바이트 절약, 최대 43억 제한
  BIGINT: 8바이트, 사실상 무제한
  → 초기부터 BIGINT 권장 (나중 변경 비용이 훨씬 큼)

TIMESTAMP vs DATETIME:
  TIMESTAMP: 4바이트 작음, 타임존 자동 변환, 2038년 한계
  DATETIME: 5바이트, 타임존 고정, 9999년까지
  → 글로벌 서비스: DATETIME + UTC 명시
  → 국내 단순 서비스: TIMESTAMP 가능

ENUM vs VARCHAR:
  ENUM: 1바이트, 자동 검증, ALTER TABLE 필요
  VARCHAR: 가변 크기, 유연성, 검증 없음
  → 안정적 상태값: ENUM
  → 자주 바뀌는 분류: VARCHAR + CHECK CONSTRAINT

DECIMAL vs DOUBLE:
  DECIMAL: 정확, 압축 저장, 느린 산술
  DOUBLE: 빠른 산술, 부동소수점 오차
  → 금액: 반드시 DECIMAL
  → 과학 계산, 통계: DOUBLE 가능 (오차 허용 시)
```

---

## 📌 핵심 정리

```
데이터 타입 선택 핵심:

PK:
  INT: 단기 서비스, 43억 초과 불가 → BIGINT 기본 선택
  BIGINT: 모든 서비스에서 안전

금액:
  DECIMAL(M, D): 반드시 사용 (FLOAT/DOUBLE 절대 금지)
  설계: DECIMAL(15, 2) 일반 금액, DECIMAL(20, 8) 암호화폐

날짜/시간:
  TIMESTAMP: 4바이트, 타임존 변환, 2038년 한계
  DATETIME: 5바이트, 타임존 고정, 9999년까지
  → 새 서비스: DATETIME 기본 (미래 안전)

문자열:
  CHAR: 고정 길이 코드, 해시값, 국가코드
  VARCHAR: 이름, 이메일, 설명
  ENUM: 제한적 상태값 (변경 드물 때)
  TEXT: 대용량 텍스트 (인덱스 제한 주의)

황금 규칙:
  "나중에 바꾸기 어렵다"
  처음부터 BIGINT PK
  처음부터 DECIMAL 금액
  처음부터 DATETIME (글로벌 대비)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 테이블 설계의 문제점을 찾고 개선하라.

```sql
CREATE TABLE product_orders (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    user_id    INT NOT NULL,
    amount     FLOAT NOT NULL,
    ordered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status     VARCHAR(255) DEFAULT 'PENDING'
);
```

<details>
<summary>해설 보기</summary>

**문제 4가지**:

1. **`id INT` — 소진 위험**: 주문이 많은 서비스에서 43억 소진 가능
2. **`amount FLOAT` — 부동소수점 오차**: 금액에 FLOAT 사용 불가. 정산 시 오차 누적
3. **`ordered_at TIMESTAMP` — 2038년 한계 + 타임존 이슈**: 글로벌 서비스라면 DATETIME 권장
4. **`status VARCHAR(255)` — 과도한 크기**: 상태값은 짧고 제한적, ENUM이나 VARCHAR(20) 적절

**개선된 설계**:
```sql
CREATE TABLE product_orders (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,           -- BIGINT
    user_id    BIGINT NOT NULL,
    amount     DECIMAL(15, 2) NOT NULL,                    -- 정확한 금액
    ordered_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP, -- 타임존 안전
    status     ENUM('PENDING','PAID','SHIPPED','CANCELLED') -- 제한된 상태
               NOT NULL DEFAULT 'PENDING',
    INDEX idx_user_id (user_id),
    INDEX idx_ordered_at (ordered_at)
) ENGINE=InnoDB;
```

</details>

---

**Q2.** `TIMESTAMP` 컬럼이 있는 테이블의 서버 timezone을 `UTC`에서 `Asia/Seoul`로 변경할 때 발생하는 현상과 대응 방법을 설명하라.

<details>
<summary>해설 보기</summary>

**현상**:
TIMESTAMP는 UTC로 저장되고, 조회 시 `@@session.time_zone`으로 변환합니다. timezone을 변경해도 저장된 UTC 값은 그대로지만 조회 시 9시간이 더해진 값으로 보입니다.

- 기존 UTC 기준 `2024-01-01 00:00:00` 저장 → Asia/Seoul 변경 후 `2024-01-01 09:00:00`으로 조회됨
- **데이터가 변한 것이 아니라 해석 방식이 바뀐 것**

**문제 발생 상황**:
- 애플리케이션이 `2024-01-01 00:00:00` 값을 하드코딩으로 비교하고 있을 때 → 9시간 차이로 잘못된 결과
- 날짜 기준 파티셔닝 쿼리 → 파티션 프루닝 기준이 달라짐

**대응 방법**:
```sql
-- 1. timezone 변경 전 영향 범위 파악
SELECT COUNT(*) FROM orders WHERE ordered_at BETWEEN '2024-01-01' AND '2024-01-02';
-- timezone 변경 후 같은 쿼리 결과가 달라지는지 확인

-- 2. 애플리케이션이 UTC 기준으로 저장/조회하도록 명시
-- JDBC URL: serverTimezone=UTC
-- Python: connect(..., charset='utf8mb4', time_zone='+00:00')

-- 3. TIMESTAMP → DATETIME 마이그레이션 (근본 해결)
ALTER TABLE orders ADD COLUMN ordered_at_new DATETIME;
UPDATE orders SET ordered_at_new = ordered_at;
ALTER TABLE orders DROP COLUMN ordered_at;
ALTER TABLE orders RENAME COLUMN ordered_at_new TO ordered_at;
-- 대용량이면 pt-osc 사용
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: JSON 타입과 Generated Column ➡️](./02-json-type-generated-column.md)**

</div>
