# 사용자와 권한 설계 — 최소 권한 원칙, Role 기반 권한 관리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `GRANT` 단위를 DB/테이블/컬럼 레벨로 세분화하는 방법은?
- MySQL 8.0 Role로 권한 묶음을 관리하는 방식은?
- 애플리케이션 계정에 DML만 허용하고 DDL을 차단해야 하는 이유는?
- 최소 권한 원칙이 실제 보안 사고를 어떻게 막는가?
- 권한 설계를 점검하고 과도한 권한을 찾는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 과도한 권한 = 실수 한 번이 대형 사고

```
실제 보안 사고 패턴:

사고 1: 애플리케이션 계정에 SUPER 권한
  개발자가 편의상 app_user에 ALL PRIVILEGES 부여
  취약점을 통해 해커가 app_user 세션 획득
  → SELECT user, password FROM mysql.user 조회
  → LOAD DATA INFILE로 서버 파일 읽기
  → 서버 전체 장악 가능

사고 2: 공유 계정으로 데이터 삭제
  DBA, 개발자, 배치 모두 같은 계정 사용
  배치 스크립트 버그 → DELETE FROM orders WHERE 1=1 실행
  → 누가 실행했는지 감사 불가 (공유 계정)
  → 계정별 권한 분리였다면 배치 계정은 특정 테이블만 DELETE 가능

원칙:
  각 계정은 해당 역할에 필요한 최소한의 권한만
  각 환경(개발/운영)마다 별도 계정
  계정 공유 금지 (감사 추적 불가)
```

---

## 😱 흔한 실수 (Before)

### 1. 애플리케이션 계정에 ALL PRIVILEGES

```sql
-- Before: 편의를 위해 모든 권한 부여
CREATE USER 'app'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON mydb.* TO 'app'@'%';

-- 문제:
-- app 계정이 DROP TABLE, ALTER TABLE 실행 가능
-- 애플리케이션 코드 버그로 DROP TABLE 실행 → 데이터 손실
-- SQL Injection → 테이블 구조 변경 가능
-- 취약점 → 다른 DB도 접근 가능

-- 필요한 권한:
-- SELECT, INSERT, UPDATE, DELETE (DML만)
-- 특정 테이블만, DDL 금지
```

### 2. 와일드카드 호스트로 어디서든 접근 허용

```sql
-- Before: % 호스트로 어디서든 접근
CREATE USER 'app'@'%' IDENTIFIED BY 'password';
-- 인터넷 어디서든 3306 포트로 접근 가능
-- 브루트포스 공격 표적

-- 올바른 방법:
CREATE USER 'app'@'10.0.1.%' IDENTIFIED BY 'password';
-- 특정 서브넷에서만 접근
-- 또는 특정 IP: 'app'@'10.0.1.100'
```

### 3. 권한 부여 후 방치 (정기 점검 없음)

```sql
-- Before: 권한을 한 번 부여하고 수년간 방치
-- 직원이 퇴사해도 계정 존재 (zombie account)
-- 사용 안 하는 계정도 유지 → 잠재적 보안 위협

-- 확인:
SELECT user, host, password_last_changed
FROM mysql.user
WHERE password_last_changed < DATE_SUB(NOW(), INTERVAL 1 YEAR);
-- 1년 이상 비밀번호 미변경 계정 목록

SELECT user, host FROM mysql.user WHERE account_locked = 'N'
  AND user NOT IN (SELECT grantee FROM information_schema.USER_PRIVILEGES);
-- 권한 없는 계정 (잘못 생성된 계정)
```

---

## ✨ 올바른 접근 (After)

```
최소 권한 원칙 설계:

계정 분류:
  app_reader: SELECT만 (읽기 전용 서비스)
  app_writer: SELECT, INSERT, UPDATE, DELETE (쓰기 서비스)
  batch_user: 특정 테이블 SELECT, INSERT (배치 작업)
  dba_user: 모든 권한 (DBA, 인증된 관리자만)
  deploy_user: CREATE, ALTER, DROP (배포 전용, 배포 시만 활성화)
  monitoring: PROCESS, REPLICATION CLIENT (모니터링 에이전트)

권한 세분화:
  DB 레벨: GRANT SELECT ON mydb.* TO 'user'@'host'
  테이블 레벨: GRANT SELECT ON mydb.orders TO 'user'@'host'
  컬럼 레벨: GRANT SELECT (id, name) ON mydb.users TO 'user'@'host'
  (민감 컬럼 제외하고 부여)

MySQL 8.0 Role:
  역할(Role)을 만들어 권한 묶음 관리
  사용자에게 Role 할당 → 권한 일괄 관리
```

---

## 🔬 내부 동작 원리

### 1. GRANT 레벨별 권한 설정

```sql
-- 전체 권한 계층:
-- Global → DB → Table → Column → Routine

-- 1. DB 레벨 (일반적인 앱 계정)
CREATE USER 'app_writer'@'10.0.1.%' IDENTIFIED BY 'StrongPass123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app_writer'@'10.0.1.%';
-- DDL(CREATE, DROP, ALTER, TRUNCATE) 제외
-- mydb만 접근 가능 (다른 DB 접근 불가)

-- 2. 테이블 레벨 (배치 계정)
CREATE USER 'batch_user'@'10.0.2.100' IDENTIFIED BY 'BatchPass456!';
GRANT SELECT, INSERT ON mydb.raw_events TO 'batch_user'@'10.0.2.100';
GRANT SELECT ON mydb.products TO 'batch_user'@'10.0.2.100';
-- raw_events: 쓰기 가능, products: 읽기만

-- 3. 컬럼 레벨 (민감 데이터 제외)
CREATE USER 'support_user'@'10.0.3.%' IDENTIFIED BY 'SupportPass!';
GRANT SELECT (id, name, email, status, created_at)
  ON mydb.users TO 'support_user'@'10.0.3.%';
-- phone_number, social_id 등 민감 컬럼 제외

-- 4. 읽기 전용 (Replica 연결, 모니터링)
CREATE USER 'readonly'@'10.0.%' IDENTIFIED BY 'ReadOnlyPass!';
GRANT SELECT ON mydb.* TO 'readonly'@'10.0.%';
GRANT REPLICATION CLIENT ON *.* TO 'readonly'@'10.0.%';
-- REPLICATION CLIENT: SHOW MASTER/REPLICA STATUS 접근용

-- 5. 모니터링 계정 (최소 권한)
CREATE USER 'monitor'@'10.0.5.1' IDENTIFIED BY 'MonitorPass!';
GRANT PROCESS ON *.* TO 'monitor'@'10.0.5.1';
GRANT REPLICATION CLIENT ON *.* TO 'monitor'@'10.0.5.1';
GRANT SELECT ON performance_schema.* TO 'monitor'@'10.0.5.1';
GRANT SELECT ON sys.* TO 'monitor'@'10.0.5.1';
```

### 2. MySQL 8.0 Role 관리

```sql
-- Role 생성
CREATE ROLE 'app_read_role', 'app_write_role', 'dba_role', 'batch_role';

-- Role에 권한 부여
GRANT SELECT ON mydb.* TO 'app_read_role';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app_write_role';
GRANT ALL PRIVILEGES ON *.* TO 'dba_role';
GRANT SELECT, INSERT ON mydb.events TO 'batch_role';

-- 사용자에게 Role 할당
CREATE USER 'alice'@'10.0.1.50' IDENTIFIED BY 'AlicePass!';
GRANT 'app_write_role' TO 'alice'@'10.0.1.50';

CREATE USER 'bob'@'10.0.1.51' IDENTIFIED BY 'BobPass!';
GRANT 'app_read_role' TO 'bob'@'10.0.1.51';

CREATE USER 'charlie_dba'@'10.0.10.1' IDENTIFIED BY 'DbaPass!';
GRANT 'dba_role' TO 'charlie_dba'@'10.0.10.1';

-- Role 기본 활성화 설정 (로그인 시 자동 Role 적용)
SET DEFAULT ROLE 'app_write_role' TO 'alice'@'10.0.1.50';
SET DEFAULT ROLE ALL TO 'alice'@'10.0.1.50';  -- 할당된 모든 Role 기본 활성화

-- Role 확인
SHOW GRANTS FOR 'alice'@'10.0.1.50';
-- + Role의 실제 권한
SHOW GRANTS FOR 'alice'@'10.0.1.50' USING 'app_write_role';

-- Role 추가/제거
GRANT 'batch_role' TO 'alice'@'10.0.1.50';  -- 역할 추가
REVOKE 'batch_role' FROM 'alice'@'10.0.1.50';  -- 역할 제거
-- 역할 변경 시 alice에게 할당된 모든 동일 역할 사용자에게 즉시 반영
```

### 3. 권한 감사 쿼리

```sql
-- 현재 모든 사용자 권한 목록
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type,
    is_grantable
FROM information_schema.TABLE_PRIVILEGES
WHERE table_schema = 'mydb'
ORDER BY grantee, table_name, privilege_type;

-- 글로벌 권한을 가진 사용자 (위험!)
SELECT user, host, Super_priv, Grant_priv, File_priv,
       Process_priv, Shutdown_priv, Reload_priv
FROM mysql.user
WHERE Super_priv = 'Y' OR Grant_priv = 'Y' OR File_priv = 'Y'
ORDER BY user;
-- Super_priv, File_priv가 있는 비DBA 계정 즉시 검토

-- 오래된 계정 찾기 (1년 이상 미사용 또는 비밀번호 미변경)
SELECT user, host, password_last_changed,
       password_expired, account_locked
FROM mysql.user
WHERE user NOT IN ('root', 'mysql.sys', 'mysql.session', 'mysql.infoschema')
ORDER BY password_last_changed;

-- 불필요한 *.* 권한 찾기
SELECT GRANTEE, PRIVILEGE_TYPE, IS_GRANTABLE
FROM information_schema.USER_PRIVILEGES
WHERE GRANTEE NOT IN ("'root'@'localhost'", "'mysql.infoschema'@'localhost'");
```

### 4. 애플리케이션 계정 DDL 차단의 중요성

```
DDL 차단이 필요한 이유:

SQL Injection 공격 시나리오:
  URL: /users?id=1; DROP TABLE orders; --
  앱 계정에 DROP 권한 있으면:
    → orders 테이블 삭제!

  앱 계정에 DML만 있으면:
    → Error 1142: DROP command denied to user 'app'
    → 피해 없음

배포 실수 방지:
  개발자가 실수로 운영 DB에서 ALTER TABLE 실행
  앱 계정 = DDL 없음 → 오류 발생 → 실수 방지

마이그레이션 분리:
  deploy_user: DDL 권한 (마이그레이션 전용)
  app_user: DML만 (서비스 운영)
  → 배포 시만 deploy_user 활성화 (평상시 비활성화 또는 제한된 IP에서만)

설계 원칙:
  서비스 계정: SELECT, INSERT, UPDATE, DELETE만
  DDL은 별도 계정 (배포 파이프라인에서만 사용)
  DROP, TRUNCATE는 DBA만 수동 실행
```

---

## 💻 실전 실험

### 실험 1: 최소 권한 계정 설정 및 테스트

```sql
-- 테스트 환경 설정
CREATE DATABASE privilege_test;
CREATE TABLE privilege_test.orders (id INT PRIMARY KEY, amount DECIMAL(10,2));
CREATE TABLE privilege_test.users (id INT PRIMARY KEY, name VARCHAR(100), ssn CHAR(14));

-- 앱 계정 생성 (DML만, 민감 컬럼 제외)
CREATE USER 'test_app'@'localhost' IDENTIFIED BY 'TestApp123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON privilege_test.orders TO 'test_app'@'localhost';
GRANT SELECT (id, name) ON privilege_test.users TO 'test_app'@'localhost';  -- ssn 제외

-- test_app으로 접속 후 테스트
-- mysql -u test_app -p privilege_test

-- 허용된 쿼리:
SELECT * FROM orders;  -- OK
INSERT INTO orders VALUES (1, 100.00);  -- OK
SELECT id, name FROM users;  -- OK (허용된 컬럼만)

-- 차단된 쿼리:
SELECT ssn FROM users;  -- Error 1143: SELECT command denied for column 'ssn'
DROP TABLE orders;  -- Error 1142: DROP command denied
CREATE INDEX idx ON orders(amount);  -- Error 1142: CREATE command denied

-- 정리
DROP DATABASE privilege_test;
DROP USER 'test_app'@'localhost';
```

### 실험 2: Role 기반 권한 관리

```sql
-- Role 생성 및 권한 할당
CREATE ROLE 'dev_read', 'dev_write';
GRANT SELECT ON mydb.* TO 'dev_read';
GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'dev_write';

-- 개발자 계정에 Role 할당
CREATE USER 'dev_user1'@'10.0.%' IDENTIFIED BY 'DevUser123!';
GRANT 'dev_read' TO 'dev_user1'@'10.0.%';
SET DEFAULT ROLE 'dev_read' TO 'dev_user1'@'10.0.%';

-- dev_user1으로 확인
SHOW GRANTS FOR 'dev_user1'@'10.0.%';

-- Role에 권한 추가 → 모든 Role 사용자에게 즉시 반영
GRANT DELETE ON mydb.temp_data TO 'dev_write';
-- 'dev_write' Role을 가진 모든 사용자에게 자동 반영

-- 정리
DROP USER 'dev_user1'@'10.0.%';
DROP ROLE 'dev_read', 'dev_write';
```

---

## 📊 성능/비용 비교

```
권한 설계 방식별 관리 비용:

개별 권한 관리 (GRANT per user):
  설정: 각 사용자마다 GRANT 문 작성
  변경: 역할 변경 시 각 사용자 개별 변경
  감사: 사용자마다 SHOW GRANTS 확인
  비용: 사용자 수에 비례하여 관리 비용 증가

Role 기반 관리 (MySQL 8.0):
  설정: Role에 한 번 권한 부여 → 사용자에게 Role 할당
  변경: Role 변경 → 모든 해당 사용자 즉시 반영
  감사: Role 권한 한 번 확인으로 충분
  비용: Role 수에 비례 (사용자 수 무관)

10명 팀 vs 100명 팀:
  10명: 개별 관리 가능
  100명: Role 기반 필수 (개별 관리 불가)
  1,000명 이상: LDAP/IAM 연동 (Role 기반 +)
```

---

## ⚖️ 트레이드오프

```
최소 권한 원칙 트레이드오프:

이점:
  ✅ 보안 사고 피해 범위 최소화
  ✅ SQL Injection 피해 제한
  ✅ 실수에 의한 데이터 손실 방지
  ✅ 감사 추적 용이 (계정별 행위 구분)
  ✅ 규정 준수 (개인정보보호법, PCI DSS 등)

비용:
  ❌ 초기 설정 시간 증가 (계정 설계 필요)
  ❌ 개발 편의성 감소 (필요 권한 요청 절차)
  ❌ 계정 관리 복잡도 증가

허용 불가 편법:
  "빠른 개발을 위해 ALL PRIVILEGES" → 절대 금지
  "공유 계정 편하다" → 감사 추적 불가
  "개발 환경이니까 SUPER" → 개발 환경도 중요 데이터 포함 가능

Role 관리:
  ✅ 권한 변경이 단순해짐 (Role 수정 한 번)
  ✅ 신규 사용자 온보딩 빠름 (Role만 할당)
  ❌ Role 설계 초기 비용
  ❌ Role 잘못 설계 시 과도한 권한이 여러 사용자에게
```

---

## 📌 핵심 정리

```
사용자 권한 설계 핵심:

최소 권한 원칙:
  서비스 계정: DML만 (SELECT, INSERT, UPDATE, DELETE)
  DDL은 별도 계정 (배포 파이프라인만)
  읽기 전용: SELECT만
  DBA: 최소 인원, IP 제한, 감사 로그

GRANT 레벨:
  DB 레벨: GRANT priv ON db.* TO user
  테이블 레벨: GRANT priv ON db.table TO user
  컬럼 레벨: GRANT SELECT (col1, col2) ON db.table TO user

Role 관리 (MySQL 8.0):
  CREATE ROLE → GRANT privileges → GRANT role TO user
  역할 변경 → 모든 할당 사용자 즉시 반영
  SET DEFAULT ROLE ALL: 로그인 시 자동 활성화

보안 체크리스트:
  □ 서비스 계정에 DDL 없는지 확인
  □ 글로벌 권한(SUPER, FILE) 최소화
  □ % 호스트 대신 특정 IP/대역 지정
  □ 공유 계정 사용 금지
  □ 분기별 권한 점검 (zombie account, 과도한 권한)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 권한 설정의 문제점을 찾고 최소 권한 원칙에 맞게 수정하라.

```sql
CREATE USER 'api_server'@'%' IDENTIFIED BY 'pass123';
GRANT ALL PRIVILEGES ON *.* TO 'api_server'@'%' WITH GRANT OPTION;
```

<details>
<summary>해설 보기</summary>

**문제 4가지**:
1. `'%'` 호스트: 어디서든 접근 가능 → 브루트포스 공격 표적
2. `ALL PRIVILEGES ON *.*`: 모든 DB에 모든 권한 → 데이터 전체 접근/삭제 가능
3. `WITH GRANT OPTION`: 다른 사용자에게 권한 부여 가능 → 권한 확산 위험
4. 약한 비밀번호 (`pass123`)

**수정**:
```sql
-- 기존 계정 삭제
DROP USER 'api_server'@'%';

-- 올바른 계정 생성
CREATE USER 'api_server'@'10.0.1.%'
  IDENTIFIED BY 'ApiServer@2024#Secure!'
  PASSWORD EXPIRE INTERVAL 90 DAY
  FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;

-- 필요한 DB와 DML만 부여
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'api_server'@'10.0.1.%';
-- WITH GRANT OPTION 제외
-- DDL 권한 없음
-- 특정 IP 대역만 허용
```

</details>

---

**Q2.** MySQL 8.0 Role을 사용하여 다음 팀 구조에 맞는 권한 체계를 설계하라.

```
팀 구성:
  백엔드 개발자 10명: mydb 읽기/쓰기 (DDL 없음)
  분석가 5명: mydb 읽기 전용
  DBA 2명: 모든 권한
  배포 파이프라인 1개: mydb DDL + DML
```

<details>
<summary>해설 보기</summary>

```sql
-- Role 생성
CREATE ROLE 'backend_role', 'analyst_role', 'dba_role', 'deploy_role';

-- Role 권한 설정
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'backend_role';
GRANT SELECT ON mydb.* TO 'analyst_role';
GRANT ALL PRIVILEGES ON *.* TO 'dba_role';
GRANT CREATE, DROP, ALTER, INDEX ON mydb.* TO 'deploy_role';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'deploy_role';

-- 사용자 생성 및 Role 할당 (예시)
-- 백엔드 개발자 (개발 환경 DB 접근)
CREATE USER 'dev_alice'@'10.0.dev.%' IDENTIFIED BY 'AliceSecure2024!';
GRANT 'backend_role' TO 'dev_alice'@'10.0.dev.%';
SET DEFAULT ROLE ALL TO 'dev_alice'@'10.0.dev.%';

-- 분석가
CREATE USER 'analyst_bob'@'10.0.analytics.%' IDENTIFIED BY 'BobAnalytic2024!';
GRANT 'analyst_role' TO 'analyst_bob'@'10.0.analytics.%';
SET DEFAULT ROLE ALL TO 'analyst_bob'@'10.0.analytics.%';

-- DBA (IP 엄격 제한)
CREATE USER 'dba_charlie'@'10.0.dba.1' IDENTIFIED BY 'DbaCharlie@2024!!';
GRANT 'dba_role' TO 'dba_charlie'@'10.0.dba.1';
-- DEFAULT ROLE은 DBA가 직접 SET ROLE 명시적 활성화 권장

-- 배포 파이프라인 (CI/CD 서버 IP만)
CREATE USER 'deploy_ci'@'10.0.cicd.100' IDENTIFIED BY 'CiCdDeploy@2024!';
GRANT 'deploy_role' TO 'deploy_ci'@'10.0.cicd.100';
SET DEFAULT ROLE ALL TO 'deploy_ci'@'10.0.cicd.100';
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 연결 보안 ➡️](./02-connection-security.md)**

</div>
