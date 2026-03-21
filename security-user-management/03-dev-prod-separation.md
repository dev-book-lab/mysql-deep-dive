# 개발/운영 환경 분리 전략 — 접근 제어, 마스킹, 민감 데이터 관리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 개발 환경에서 운영 DB에 직접 접근하지 않도록 계정과 네트워크를 분리하는 설계는?
- 개인정보/카드번호 등 민감 컬럼에 Dynamic Data Masking을 적용하는 방법은?
- 운영 데이터를 개발 환경으로 마이그레이션 시 마스킹 처리 절차는?
- 환경별 접근 통제를 기술적으로 강제하는 방법은?
- 마스킹 데이터의 품질을 유지하면서 테스트 가능한 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 개발자가 운영 DB에 접근하면 안 되는 이유

```
실제 사고 사례:

사고 1: 개발자 실수로 운영 DB 데이터 삭제
  개발자가 로컬 PC에서 테스트 쿼리 실행
  연결 설정 실수 → 운영 DB 연결
  DELETE FROM test_data WHERE 1=1 → 운영 데이터 삭제!
  
사고 2: 개인정보 유출
  개발팀 공유 운영 DB 계정 → 50명의 개발자가 운영 개인정보 직접 접근
  퇴사한 직원의 PC에서 고객 개인정보 발견
  → 개인정보보호법 위반 (과징금 수억 원)

사고 3: 실수로 인한 복구 불가
  개발자 테스트 목적으로 운영 DB 스키마 변경
  ALTER TABLE orders MODIFY COLUMN amount VARCHAR(20)
  → 기존 DECIMAL 데이터 일부 손실
  → 복구 불가 (백업 없음)

환경 분리 = 실수와 악의적 접근 모두 방어
```

---

## 😱 흔한 실수 (Before)

### 1. 개발/스테이징/운영이 같은 DB 계정 공유

```bash
# Before: 모든 환경에서 같은 계정 사용
# dev/.env:
DATABASE_URL=mysql://app:password@prod-db:3306/mydb

# 문제:
# 개발자 PC에서 운영 DB 직접 접근
# 잘못된 쿼리가 운영 데이터 영향
# 로컬 개발 환경에서 개인정보 노출
```

### 2. 운영 데이터를 마스킹 없이 개발 환경에 복사

```bash
# Before: 운영 덤프를 그대로 개발 환경으로 복사
mysqldump prod_mydb | mysql dev_mydb
# 개인정보(이름, 주민번호, 카드번호, 연락처)가
# 개발 환경 DB에 그대로 존재
# 개발자 모두 운영 개인정보 접근 가능
# → 개인정보보호법 위반
```

### 3. 환경 구분을 애플리케이션 코드에만 의존

```python
# Before: 환경 변수로만 환경 구분
import os
if os.getenv('ENV') == 'production':
    DB_HOST = 'prod-db'
else:
    DB_HOST = 'dev-db'

# 문제:
# ENV 환경변수가 잘못 설정되면 개발 코드가 운영 DB 접근
# 기술적 강제 없음 → 사람이 실수할 수 있음
# DB 레벨의 계정 분리가 필요
```

---

## ✨ 올바른 접근 (After)

```
환경 분리 3단계 접근:

1. 네트워크 분리:
  운영 DB: VPC Private Subnet (운영 서버만 접근 가능)
  개발 DB: 별도 서브넷 (개발팀 접근 가능)
  방화벽: 운영 DB 3306 포트 → 운영 서버 IP만 허용

2. 계정 분리:
  운영 계정: app_prod@10.0.prod.% (운영 서버 IP만)
  개발 계정: app_dev@10.0.dev.% (개발 서브넷만)
  개발자는 운영 계정 비밀번호 모름

3. 데이터 마스킹:
  개발/스테이징 환경: 운영 데이터의 마스킹 버전
  개인정보 컬럼: 비식별화 처리 후 복사
  Dynamic Data Masking: 운영 DB에서 개발팀 조회 시 마스킹 적용
```

---

## 🔬 내부 동작 원리

### 1. 네트워크와 계정 분리 설계

```sql
-- 운영 계정: 운영 서버 IP만 허용
CREATE USER 'app_prod'@'10.0.prod.%'
  IDENTIFIED BY 'ProdStrongPass@2024!'
  PASSWORD EXPIRE INTERVAL 90 DAY
  REQUIRE SSL;

GRANT SELECT, INSERT, UPDATE, DELETE ON prod_mydb.* TO 'app_prod'@'10.0.prod.%';

-- 스테이징 계정: 스테이징 서버만
CREATE USER 'app_staging'@'10.0.staging.%'
  IDENTIFIED BY 'StagingPass@2024!'
  REQUIRE SSL;

GRANT SELECT, INSERT, UPDATE, DELETE ON staging_mydb.* TO 'app_staging'@'10.0.staging.%';

-- 개발 계정: 개발 서브넷만, 개발 DB만
CREATE USER 'app_dev'@'10.0.dev.%'
  IDENTIFIED BY 'DevPass@2024!'
  PASSWORD EXPIRE INTERVAL 30 DAY;

GRANT SELECT, INSERT, UPDATE, DELETE ON dev_mydb.* TO 'app_dev'@'10.0.dev.%';

-- 읽기 전용 (분석/BI)
CREATE USER 'readonly_prod'@'10.0.analytics.%'
  IDENTIFIED BY 'ReadOnlyProd@2024!'
  REQUIRE SSL;

GRANT SELECT ON prod_mydb.* TO 'readonly_prod'@'10.0.analytics.%';

-- DBA: 별도 관리 서버에서만
CREATE USER 'dba_admin'@'10.0.mgmt.1'
  IDENTIFIED BY 'DbaAdmin@2024!!'
  REQUIRE SSL;

GRANT ALL PRIVILEGES ON *.* TO 'dba_admin'@'10.0.mgmt.1' WITH GRANT OPTION;
```

### 2. Dynamic Data Masking (MySQL 8.0.31+)

```sql
-- MySQL 8.0.31+에서 Dynamic Data Masking 컴포넌트 설치
INSTALL COMPONENT 'file://component_masking_functions';

-- 마스킹 함수 종류:
-- mask_inner(str, left, right): 안쪽 마스킹
-- mask_outer(str, left, right): 바깥쪽 마스킹
-- mask_pan(card_num): 신용카드 번호 마스킹
-- mask_ssn(ssn): 주민번호 마스킹
-- gen_rnd_email(): 랜덤 이메일 생성
-- gen_rnd_phone(): 랜덤 전화번호 생성

-- 마스킹 함수 사용 예시
SELECT
    id,
    mask_inner(name, 1, 1) AS name_masked,
    -- '홍길동' → '홍*동'
    mask_pan(card_number) AS card_masked,
    -- '1234-5678-9012-3456' → '1234-****-****-3456'
    gen_rnd_email() AS fake_email
    -- 'real@email.com' 대신 랜덤 이메일
FROM users;

-- 마스킹 뷰 생성 (개발팀용)
CREATE VIEW users_masked AS
SELECT
    id,
    mask_inner(name, 1, 0) AS name,
    mask_inner(email, 3, 4) AS email,
    CONCAT(SUBSTR(phone, 1, 3), '-****-', SUBSTR(phone, 8, 4)) AS phone,
    created_at,
    status
FROM users;

-- 개발 계정에 마스킹 뷰만 접근 허용
GRANT SELECT ON prod_mydb.users_masked TO 'dev_readonly'@'10.0.dev.%';
REVOKE SELECT ON prod_mydb.users FROM 'dev_readonly'@'10.0.dev.%';
```

### 3. 운영 데이터 마이그레이션 시 마스킹 처리

```bash
#!/bin/bash
# 운영 → 개발 환경 데이터 마이그레이션 (마스킹 포함)

PROD_DB="prod_mydb"
DEV_DB="dev_mydb"
MASKED_DUMP="/tmp/masked_dump.sql"

echo "Step 1: 비민감 테이블 덤프 (그대로 복사)"
mysqldump \
  --single-transaction \
  --no-data \
  $PROD_DB > /tmp/schema.sql  # 스키마만

mysqldump \
  --single-transaction \
  --ignore-table=$PROD_DB.users \
  --ignore-table=$PROD_DB.payments \
  --ignore-table=$PROD_DB.personal_info \
  $PROD_DB products categories orders >> /tmp/non_sensitive.sql

echo "Step 2: 민감 테이블 마스킹하여 덤프"
mysql -u root -p $PROD_DB << 'SQL'
SELECT
    id,
    CONCAT(FLOOR(RAND()*9000)+1000, '@masked.com') AS email,
    CONCAT(
        FLOOR(RAND()*90)+10, '-',
        FLOOR(RAND()*9000)+1000, '-',
        FLOOR(RAND()*9000)+1000
    ) AS phone,
    REPEAT('*', LENGTH(name)-1) + SUBSTR(name,1,1) AS name,
    -- 기타 비민감 컬럼은 그대로
    status, created_at, updated_at
INTO OUTFILE '/tmp/users_masked.csv'
FIELDS TERMINATED BY ','
FROM users;
SQL

echo "Step 3: 개발 DB에 적용"
mysql -u root -p $DEV_DB < /tmp/schema.sql
mysql -u root -p $DEV_DB < /tmp/non_sensitive.sql
mysql -u root -p $DEV_DB -e "
LOAD DATA INFILE '/tmp/users_masked.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
(id, email, phone, name, status, created_at, updated_at);
"

echo "마스킹 완료. 개발 환경에 실제 개인정보 없음."
```

### 4. 마스킹 데이터 품질 유지

```sql
-- 마스킹 후에도 테스트 가능한 데이터 생성 원칙:

-- 나쁜 마스킹: 형식 파괴 (테스트 불가)
-- email: 'real@email.com' → 'XXXXXXXXXXXXXXX' (이메일 형식 아님)
-- phone: '010-1234-5678' → 'XXXXXXXXXXXXXX' (전화번호 형식 아님)

-- 좋은 마스킹: 형식 유지 (테스트 가능)
-- email: 'real@email.com' → 'masked123@test.com' (유효한 이메일 형식)
-- phone: '010-1234-5678' → '010-0000-0001' (유효한 전화번호 형식)

-- 마스킹 함수 예시 (형식 유지):
SELECT
    id,
    CONCAT('user', id, '@masked-test.com') AS email,
    CONCAT('010-', LPAD(FLOOR(RAND()*10000), 4, '0'), '-',
                   LPAD(FLOOR(RAND()*10000), 4, '0')) AS phone,
    CASE LENGTH(name)
        WHEN 2 THEN CONCAT(SUBSTR(name,1,1), '*')
        WHEN 3 THEN CONCAT(SUBSTR(name,1,1), '*', SUBSTR(name,3,1))
        ELSE CONCAT(SUBSTR(name,1,1), REPEAT('*', LENGTH(name)-2), SUBSTR(name,-1))
    END AS name,
    -- 카드번호: 형식 유지, 앞 6자리 (BIN)만 실제값
    CONCAT(SUBSTR(card_number, 1, 6), '-XXXX-XXXX-', FLOOR(RAND()*9000)+1000) AS card_number
FROM users;

-- 결정적(Deterministic) 마스킹: 같은 입력 → 항상 같은 출력
-- (FK 관계 테스트에 필요)
-- user_id=1의 email → 항상 'user1@masked-test.com'
-- CONCAT('user', id, '@masked-test.com') 방식이 결정적
```

### 5. 운영 DB 접근 로그 기록 및 알람

```sql
-- 운영 DB에 개발자 계정이 접근하면 알람
-- (감사 로그 + 모니터링 연동)

-- 허용된 운영 계정 목록
CREATE TABLE allowed_prod_accounts (
    account VARCHAR(100) PRIMARY KEY,
    description VARCHAR(200)
);

INSERT INTO allowed_prod_accounts VALUES
    ('app_prod@10.0.prod.%', '운영 애플리케이션 서버'),
    ('readonly_prod@10.0.analytics.%', '분석팀 읽기 전용'),
    ('dba_admin@10.0.mgmt.1', 'DBA 관리');

-- 감사 로그에서 허용되지 않은 접근 탐지
-- (Elasticsearch + Kibana 또는 SIEM 도구 연동)

-- Prometheus alert 예시:
-- alert: UnauthorizedProdAccess
-- expr: mysql_audit_login_count{env="prod", user!~"app_prod|readonly_prod|dba_admin"} > 0
-- for: 0m
-- annotations:
--   summary: "운영 DB에 허용되지 않은 계정 접속 시도"
```

---

## 💻 실전 실험

### 실험 1: 환경별 계정 분리 테스트

```sql
-- 운영 계정 (운영 서버 IP만)
CREATE USER 'app_prod'@'10.0.1.%' IDENTIFIED BY 'ProdPass@2024!';
GRANT SELECT, INSERT, UPDATE, DELETE ON prod_db.* TO 'app_prod'@'10.0.1.%';

-- 개발 계정 (개발 IP만)
CREATE USER 'app_dev'@'10.0.2.%' IDENTIFIED BY 'DevPass@2024!';
GRANT SELECT, INSERT, UPDATE, DELETE ON dev_db.* TO 'app_dev'@'10.0.2.%';

-- 크로스 접근 불가 확인
-- dev 계정으로 prod_db 접근 시도:
-- mysql -u app_dev -p prod_db → Access denied!
-- prod 계정으로 dev IP에서 접근 시도:
-- mysql -h 10.0.2.100 -u app_prod -p → Access denied!

-- 계정 목록 확인
SELECT user, host FROM mysql.user WHERE user LIKE 'app_%';
-- app_prod  10.0.1.%
-- app_dev   10.0.2.%

DROP USER 'app_prod'@'10.0.1.%', 'app_dev'@'10.0.2.%';
```

### 실험 2: 마스킹 뷰 생성 및 접근 제어

```sql
-- 테스트 테이블
CREATE TABLE test_users (
    id        INT AUTO_INCREMENT PRIMARY KEY,
    name      VARCHAR(50),
    email     VARCHAR(100),
    phone     CHAR(13),
    ssn       CHAR(14)
);

INSERT INTO test_users (name, email, phone, ssn) VALUES
('홍길동', 'hong@example.com', '010-1234-5678', '900101-1234567'),
('김영희', 'kim@example.com', '010-9876-5432', '850515-2345678');

-- 마스킹 뷰 생성
CREATE VIEW test_users_masked AS
SELECT
    id,
    CONCAT(SUBSTR(name,1,1), REPEAT('*', GREATEST(LENGTH(name)-1, 1))) AS name,
    CONCAT(SUBSTR(email,1,3), '***@masked.com') AS email,
    CONCAT(SUBSTR(phone,1,3), '-****-', SUBSTR(phone,9,4)) AS phone,
    CONCAT(SUBSTR(ssn,1,6), '-*******') AS ssn
FROM test_users;

-- 마스킹 결과 확인
SELECT * FROM test_users_masked;
-- name: 홍**, email: hon***@masked.com, phone: 010-****-5678

-- 개발 계정에 마스킹 뷰만 접근 허용
CREATE USER 'dev_view'@'localhost' IDENTIFIED BY 'DevView@2024!';
GRANT SELECT ON test_db.test_users_masked TO 'dev_view'@'localhost';

-- dev_view로 원본 테이블 접근 불가 확인
-- SELECT * FROM test_users; → Error 1142

DROP VIEW test_users_masked;
DROP TABLE test_users;
DROP USER 'dev_view'@'localhost';
```

---

## 📊 성능/비용 비교

```
환경 분리 구현 방식별 비용:

네트워크 수준 분리 (VPC, 방화벽):
  구현 비용: AWS VPC, Security Group 설정 (낮음)
  운영 비용: 거의 없음 (인프라 자동 관리)
  효과: 물리적 분리 (가장 강력)

DB 계정 수준 분리:
  구현 비용: 계정 생성 및 권한 설정 (낮음)
  운영 비용: 계정 관리, 비밀번호 갱신
  효과: DB 레벨 접근 제어

Dynamic Data Masking:
  구현 비용: 마스킹 뷰 설계 (중간)
  쿼리 성능: 마스킹 함수 오버헤드 (~5%)
  효과: 데이터 레벨 민감정보 보호

운영 데이터 마이그레이션:
  구현 비용: 마스킹 스크립트 개발 (높음)
  실행 시간: 데이터 크기에 비례
  효과: 개발 환경에 실제 데이터 없음
  필요 빈도: 분기 1회 또는 이벤트 기반

총 보안 효과 vs 비용:
  네트워크 분리 + 계정 분리: 높은 효과, 낮은 비용 (최우선)
  데이터 마스킹: 중간 효과, 중간 비용 (개인정보 있는 서비스)
  Dynamic Data Masking: 실시간 보호 (MySQL 8.0.31+)
```

---

## ⚖️ 트레이드오프

```
환경 분리 트레이드오프:

엄격한 분리:
  ✅ 최고 수준 보안
  ✅ 법적 요구사항 충족
  ❌ 개발 편의성 감소
  ❌ 운영 데이터 재현 테스트 어려움
  ❌ 환경별 별도 DB 유지 비용

마스킹 데이터 사용:
  ✅ 개발/테스트에 실제와 유사한 데이터
  ✅ 개인정보 노출 없음
  ❌ 마스킹 후 일부 엣지 케이스 테스트 어려움
  ❌ 마스킹 데이터 관리 복잡도

Real-time 운영 DB 읽기 허용 (최소 마스킹):
  ✅ 개발 편의성 높음
  ❌ 개인정보 노출 위험
  ❌ 개발자 실수로 운영 데이터 영향 가능
  → 개인정보보호법 위반 위험: 권장하지 않음

현실적 타협안:
  운영 DB: 완전 분리 (개발자 접근 불가)
  스테이징 DB: 마스킹된 운영 데이터 주기적 복사
  개발 DB: 합성 데이터 또는 마스킹 데이터
  분석: 읽기 전용 + Dynamic Masking
```

---

## 📌 핵심 정리

```
개발/운영 환경 분리 핵심:

3단계 분리:
  1. 네트워크: 운영 DB는 운영 서버 IP만 접근
  2. 계정: 환경별 별도 계정, 개발자는 운영 계정 비밀번호 모름
  3. 데이터: 개발 환경에 실제 개인정보 없음 (마스킹)

계정 설계:
  app_prod@10.0.prod.%: 운영 환경만
  app_staging@10.0.staging.%: 스테이징만
  app_dev@10.0.dev.%: 개발 환경만
  각 환경은 각자의 DB 스키마 사용

마스킹:
  Dynamic Data Masking (MySQL 8.0.31+): 실시간 뷰 마스킹
  마이그레이션 마스킹: 운영 → 개발 복사 시 비식별화
  형식 유지: 유효한 이메일/전화번호 형식으로 마스킹 (테스트 가능)

마스킹 원칙:
  결정적(Deterministic): 같은 원본 → 같은 마스킹 (FK 관계 유지)
  형식 보존: 이메일은 이메일 형식으로, 전화번호는 전화번호 형식으로
  비가역적: 마스킹 → 원본 역산 불가

법적 요구:
  개인정보보호법: 개발 환경에 개인정보 저장 = 위반 가능
  PCI DSS: 카드번호는 운영 환경 외 저장 금지
  → 법적 처벌 전에 기술적 통제로 예방
```

---

## 🤔 생각해볼 문제

**Q1.** 마스킹된 개발 데이터를 사용하면서 FK 관계가 깨지지 않도록 하는 방법을 설계하라.

<details>
<summary>해설 보기</summary>

**결정적(Deterministic) 마스킹이 핵심입니다.**

FK 관계가 있는 테이블:
- `orders.user_id → users.id`
- `users.email`이 마스킹되면 `orders`에서 user를 찾을 수 있어야 함

```sql
-- 나쁜 마스킹: 랜덤 → FK 관계 깨짐
UPDATE users SET email = gen_rnd_email();  -- 매번 다른 값
-- orders.user_id=1 → users.id=1 → email='random123@test.com'
-- 다음 마이그레이션 시 → email='different456@test.com'
-- 테스트 코드가 이메일로 사용자를 찾으면 매번 다른 결과

-- 좋은 마스킹: 결정적 (id 기반으로 고정)
UPDATE users
SET email = CONCAT('user', id, '@masked-dev.com'),  -- id 기반으로 항상 같음
    phone = CONCAT('010-', LPAD(id*3 % 10000, 4, '0'), '-', LPAD(id*7 % 10000, 4, '0')),
    name = CONCAT('테스트유저', id);

-- user_id=1 → 항상 user1@masked-dev.com
-- 마이그레이션 반복해도 동일 값 유지

-- 마스킹 스크립트 (결정적, 형식 유지):
UPDATE users SET
    email = CONCAT('test_', id, '@dev-masked.com'),
    phone = CONCAT(
        '010-',
        LPAD(MOD(id * 1234, 10000), 4, '0'),
        '-',
        LPAD(MOD(id * 5678, 10000), 4, '0')
    ),
    name = CASE
        WHEN MOD(id, 3) = 0 THEN CONCAT('마스킹', id, '가')
        WHEN MOD(id, 3) = 1 THEN CONCAT('마스킹', id, '나')
        ELSE CONCAT('마스킹', id, '다')
    END;
```

</details>

---

**Q2.** 스타트업에서 개발 팀이 빠른 개발을 위해 "운영 DB 읽기 전용 접근"을 요청했다. 이를 허용해야 하는가? 허용한다면 어떤 조건에서 허용하는가?

<details>
<summary>해설 보기</summary>

**원칙적으로 불허하되, 불가피하다면 엄격한 조건부 허용.**

**불허 이유**:
- 개인정보보호법: 개발 목적 개인정보 접근은 원칙적 위반
- 실수 위험: 읽기 전용이어도 개발자가 개인정보를 로컬 파일로 저장 가능
- 데이터 유출: 개발자 PC 분실/해킹 시 개인정보 유출

**불가피하게 허용 시 조건**:

```sql
-- 1. Dynamic Data Masking 강제
CREATE USER 'dev_limited'@'10.0.dev.%'
  IDENTIFIED BY 'DevLimited@2024!'
  REQUIRE SSL;

-- 마스킹 뷰만 접근 허용 (원본 테이블 직접 접근 불가)
GRANT SELECT ON prod_db.orders TO 'dev_limited'@'10.0.dev.%';
GRANT SELECT ON prod_db.products TO 'dev_limited'@'10.0.dev.%';
-- 개인정보 테이블은 마스킹 뷰만:
GRANT SELECT ON prod_db.users_masked TO 'dev_limited'@'10.0.dev.%';

-- 2. 감사 로그 강화 (모든 쿼리 기록)
-- audit_log_include_accounts = 'dev_limited@10.0.dev.%'

-- 3. 접근 가능 시간 제한 (옵션, MySQL 플러그인 필요)
-- GRANT ... WITH MAX_CONNECTIONS_PER_HOUR 100

-- 4. 정기 접근 리뷰 (분기별)
-- 더 이상 필요 없으면 계정 삭제 또는 잠금
```

**보안 체크리스트 (허용 전)**:
- [ ] 개인정보보호책임자(CPO) 승인
- [ ] 해당 개발자 동의서 + 보안 서약
- [ ] 접근 목적 문서화 + 기간 명시
- [ ] Dynamic Masking 또는 마스킹 뷰 적용 확인
- [ ] 감사 로그 100% 기록 확인

실무 권장: 운영 데이터 접근보다 **좋은 마스킹 데이터 파이프라인 구축이 장기적으로 더 안전하고 효율적**입니다.

</details>

---

<div align="center">

**[⬅️ 연결 보안](./02-connection-security.md)** | **[홈으로 🏠](../README.md)**

</div>
