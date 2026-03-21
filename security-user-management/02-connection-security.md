# 연결 보안 — SSL/TLS, 비밀번호 정책, 감사 로그

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `require_secure_transport`로 암호화되지 않은 연결을 거부하는 방법은?
- `validate_password` 플러그인의 비밀번호 복잡도를 강제하는 방식은?
- MySQL Enterprise Audit 또는 오픈소스 감사 로그로 접근 이력을 기록하는 방법은?
- SSL/TLS 인증서를 설정하고 연결 확인하는 방법은?
- 계정 잠금과 비밀번호 만료 정책을 설정하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 암호화 없는 연결 = 네트워크 도청으로 자격증명 탈취

```
보안 위협 시나리오:

위협 1: 평문 연결 도청
  app_server → MySQL (비암호화)
  같은 네트워크의 공격자: tcpdump로 패킷 캡처
  → MySQL 패킷에서 사용자명/비밀번호 추출
  → DB 직접 접속 가능

  방어: require_secure_transport = ON
  → 모든 연결 SSL/TLS 강제
  → 네트워크 도청해도 복호화 불가

위협 2: 약한 비밀번호 브루트포스
  DBA 계정 비밀번호: "admin123"
  자동화 공격으로 수천 번 시도 → 성공
  → DB 전체 접근권한 탈취

  방어: validate_password + 계정 잠금
  → 복잡도 요구사항 강제
  → 5회 실패 시 자동 잠금

위협 3: 접근 이력 없음 (사후 조사 불가)
  내부자가 개인정보 대량 조회
  조사 시: 누가 언제 무엇을 조회했는지 알 수 없음
  → 감사 로그 없으면 책임 소재 불명확
```

---

## 😱 흔한 실수 (Before)

### 1. SSL 없이 운영

```bash
# Before: SSL 비활성화 상태 확인 안 함
mysql -u app -p mydb
# 연결 후 확인:
SHOW STATUS LIKE 'Ssl_cipher';
# Empty set (SSL 사용 안 함)

# 또는
mysql -u app -p --ssl-mode=DISABLED mydb
# --ssl-mode=DISABLED: 강제로 비암호화 연결
# 운영 환경에서 이 옵션 사용 = 매우 위험
```

### 2. 비밀번호 정책 없이 단순 비밀번호 허용

```sql
-- Before: 비밀번호 정책 없음
CREATE USER 'user1'@'%' IDENTIFIED BY '1234';
-- MySQL이 허용함! (validate_password 미설치)
-- 약한 비밀번호로 브루트포스 공격에 취약
```

### 3. 감사 로그 없이 운영

```
Before: 감사 로그 없음
  보안 감사 지적: "2024년 3월에 users 테이블에서 개인정보를 조회한 내역을 제출하세요"
  → 아무것도 없음
  → 개인정보보호법 위반
  → 과징금, 과태료, 서비스 중단

개인정보보호법 제29조: 개인정보처리자는 개인정보의 안전한 관리를 위해
  접근 통제 및 접근 권한의 제한 조치 + 접속기록 보관 필요
```

---

## ✨ 올바른 접근 (After)

```
연결 보안 3단계:

1. 전송 암호화 (SSL/TLS):
  require_secure_transport = ON (서버 설정)
  각 계정별 REQUIRE SSL 또는 REQUIRE X509
  클라이언트: --ssl-mode=REQUIRED

2. 인증 강화 (비밀번호 정책):
  validate_password 플러그인 활성화
  복잡도: 대문자 + 소문자 + 숫자 + 특수문자, 최소 12자
  비밀번호 만료: 90일
  계정 잠금: 5회 실패 시 자동 잠금

3. 감사 로그 (Audit Log):
  모든 접속/종료 기록
  DDL 실행 기록
  민감 테이블 SELECT 기록
  로그 보관: 최소 1년 (법적 요구사항)
```

---

## 🔬 내부 동작 원리

### 1. SSL/TLS 설정

```bash
# my.cnf 설정
[mysqld]
ssl_ca     = /etc/mysql/ssl/ca.pem
ssl_cert   = /etc/mysql/ssl/server-cert.pem
ssl_key    = /etc/mysql/ssl/server-key.pem
require_secure_transport = ON  # 비SSL 연결 거부

# SSL 인증서 생성 (자체 서명)
mysql_ssl_rsa_setup --datadir=/var/lib/mysql
# 자동으로 ca.pem, server-cert.pem, server-key.pem 생성

# 또는 Let's Encrypt 등 공인 인증서 사용
```

```sql
-- SSL 설정 확인
SHOW VARIABLES LIKE '%ssl%';
-- have_ssl: YES
-- require_secure_transport: ON

-- 현재 연결의 SSL 상태 확인
SHOW STATUS LIKE 'Ssl_cipher';
-- Ssl_cipher: TLS_AES_256_GCM_SHA384 (암호화 중)
-- Empty set → 비암호화 연결 (require_secure_transport=ON이면 이 경우 연결 불가)

-- 특정 계정에 SSL 강제
ALTER USER 'app'@'10.0.1.%' REQUIRE SSL;
-- 이 계정은 SSL 없이 접속 불가

-- X.509 클라이언트 인증서 필요
ALTER USER 'admin'@'%' REQUIRE X509;
-- 클라이언트 인증서 없으면 연결 거부 (더 강력한 보안)
```

### 2. validate_password 플러그인

```sql
-- 플러그인 설치 (MySQL 8.0은 기본 컴포넌트로 내장)
INSTALL COMPONENT 'file://component_validate_password';

-- 정책 설정 (my.cnf 또는 SET GLOBAL)
SET GLOBAL validate_password.policy = STRONG;
-- LOW: 최소 길이만
-- MEDIUM: 길이 + 숫자 + 대소문자 + 특수문자
-- STRONG: MEDIUM + 사전 단어 확인

SET GLOBAL validate_password.length = 12;
SET GLOBAL validate_password.mixed_case_count = 1;  -- 대/소문자 각 1개 이상
SET GLOBAL validate_password.number_count = 1;       -- 숫자 1개 이상
SET GLOBAL validate_password.special_char_count = 1; -- 특수문자 1개 이상

-- 설정 확인
SHOW VARIABLES LIKE 'validate_password%';

-- 비밀번호 강도 테스트
SELECT VALIDATE_PASSWORD_STRENGTH('password');  -- 0~100점
-- 0: 너무 약함, 100: 강력

SELECT VALIDATE_PASSWORD_STRENGTH('MyStr0ng@Pass!');  -- 100점 기대

-- 계정 잠금 정책 (5회 실패 시 1일 잠금)
CREATE USER 'secured_user'@'%'
  IDENTIFIED BY 'MyStr0ng@Pass!'
  FAILED_LOGIN_ATTEMPTS 5
  PASSWORD_LOCK_TIME 1;  -- 1일

-- 비밀번호 만료 정책
CREATE USER 'timed_user'@'%'
  IDENTIFIED BY 'MyStr0ng@Pass!'
  PASSWORD EXPIRE INTERVAL 90 DAY;

-- 비밀번호 재사용 방지
CREATE USER 'no_reuse_user'@'%'
  IDENTIFIED BY 'MyStr0ng@Pass!'
  PASSWORD HISTORY 12;  -- 최근 12개 비밀번호 재사용 불가

-- 기존 계정에 정책 적용
ALTER USER 'app'@'10.0.1.%'
  PASSWORD EXPIRE INTERVAL 90 DAY
  FAILED_LOGIN_ATTEMPTS 5
  PASSWORD_LOCK_TIME 1
  PASSWORD HISTORY 12;
```

### 3. 감사 로그 설정

```bash
# MySQL Enterprise Audit (상용)
# my.cnf:
# [mysqld]
# plugin-load-add = audit_log.so
# audit_log_file = /var/log/mysql/audit.log
# audit_log_format = JSON
# audit_log_policy = ALL  # ALL: 모든 이벤트

# 오픈소스 대안: Percona Audit Log Plugin (무료)
# 또는 MariaDB Audit Plugin
```

```sql
-- Percona Audit Log Plugin 설치
INSTALL PLUGIN audit_log SONAME 'audit_log.so';

-- 설정 확인
SHOW VARIABLES LIKE 'audit_log%';

-- 감사 정책 설정 (my.cnf):
-- audit_log_policy = ALL       : 모든 이벤트
-- audit_log_policy = LOGINS    : 로그인/로그아웃만
-- audit_log_policy = QUERIES   : 쿼리만
-- audit_log_policy = NONE      : 비활성화

-- 특정 사용자만 감사 (고급):
-- audit_log_include_accounts = 'admin@%,dba@%'

-- 감사 로그 JSON 형식 예시:
-- {
--   "timestamp": "2024-01-15T10:30:00",
--   "id": 12345,
--   "class": "general",
--   "event": "query",
--   "connection_id": 99,
--   "account": {"user": "admin", "host": "10.0.1.50"},
--   "login": {"user": "admin", "os": "", "ip": "10.0.1.50", "proxy": ""},
--   "query": "SELECT * FROM users WHERE id = 1"
-- }
```

### 4. 연결 통계 모니터링

```sql
-- 현재 연결 중 SSL 사용 비율
SELECT
    SSL_CIPHER,
    COUNT(*) AS connections
FROM performance_schema.processlist
GROUP BY SSL_CIPHER;
-- NULL: 비SSL 연결 (require_secure_transport=ON이면 없어야 함)

-- 로그인 실패 통계
SELECT
    USER,
    HOST,
    ACCESS_DENIED
FROM performance_schema.host_summary_by_statement_type
WHERE statement_type = 'access denied';

-- 비밀번호 만료 예정 계정
SELECT user, host, password_last_changed,
       DATE_ADD(password_last_changed, INTERVAL 90 DAY) AS expires_on
FROM mysql.user
WHERE password_expired = 'N'
  AND DATE_ADD(password_last_changed, INTERVAL 90 DAY) < DATE_ADD(NOW(), INTERVAL 14 DAY);
-- 14일 내 만료 예정 계정 알람용
```

---

## 💻 실전 실험

### 실험 1: SSL 연결 강제 설정 테스트

```bash
# SSL 없이 연결 시도 (require_secure_transport=ON인 경우)
mysql -u app -p -h db-server --ssl-mode=DISABLED mydb
# Error 3159: Connections using insecure transport are prohibited
# while --require_secure_transport=ON

# SSL 있는 정상 연결
mysql -u app -p -h db-server \
  --ssl-mode=REQUIRED \
  --ssl-ca=/etc/mysql/ssl/ca.pem \
  mydb

# 연결 후 확인
SHOW STATUS LIKE 'Ssl_cipher';
# Ssl_cipher: TLS_AES_256_GCM_SHA384
```

### 실험 2: validate_password 테스트

```sql
-- 약한 비밀번호 거부 확인
CREATE USER 'test_weak'@'localhost' IDENTIFIED BY 'abc';
-- Error 1819: Your password does not satisfy the current policy requirements

-- 강한 비밀번호 허용 확인
CREATE USER 'test_strong'@'localhost' IDENTIFIED BY 'MyStr0ng@2024!';
-- Query OK

-- 강도 점수 확인
SELECT VALIDATE_PASSWORD_STRENGTH('abc') AS weak;  -- 0
SELECT VALIDATE_PASSWORD_STRENGTH('MyStr0ng@2024!') AS strong;  -- 100

-- 계정 잠금 테스트
CREATE USER 'locktest'@'localhost'
  IDENTIFIED BY 'LockTest@2024!'
  FAILED_LOGIN_ATTEMPTS 3
  PASSWORD_LOCK_TIME 1;

-- 3회 잘못된 비밀번호 시도 후 잠금 확인
-- (커맨드라인에서 3번 틀린 비밀번호 시도)
SELECT user, account_locked FROM mysql.user WHERE user = 'locktest';
-- account_locked: N (자동 잠금은 별도 필드)

-- 수동 잠금
ALTER USER 'locktest'@'localhost' ACCOUNT LOCK;
SELECT user, account_locked FROM mysql.user WHERE user = 'locktest';
-- account_locked: Y

DROP USER 'test_strong'@'localhost', 'locktest'@'localhost';
```

### 실험 3: 감사 로그 분석

```bash
# 감사 로그 파일에서 민감 테이블 접근 추출 (jq 사용)
cat /var/log/mysql/audit.log | \
  jq 'select(.query != null and (.query | test("users"; "i")))' | \
  jq '{time: .timestamp, user: .account.user, host: .login.ip, query: .query}'

# 특정 시간대 모든 접근 기록
cat /var/log/mysql/audit.log | \
  jq 'select(.timestamp >= "2024-01-15T00:00:00" and .timestamp <= "2024-01-15T23:59:59")' | \
  jq '{time: .timestamp, user: .account.user, event: .event}'

# 로그인 실패 기록
grep '"event":"failed"' /var/log/mysql/audit.log | \
  jq '{time: .timestamp, user: .account.user, ip: .login.ip}'
```

---

## 📊 성능/비용 비교

```
보안 설정 성능 영향:

SSL/TLS:
  CPU 오버헤드: 연결 설정 시 TLS 핸드쉐이크 (~1ms 추가)
  데이터 전송: AES-256 암호화/복호화 (~2~5% CPU)
  장기 연결: 오버헤드 희석됨 (한 번 핸드쉐이크, 계속 사용)
  실무 영향: 커넥션 풀 사용 시 미미 (< 1%)

validate_password:
  비밀번호 검증: < 1ms (로그인 시만)
  서비스 영향: 없음

감사 로그:
  디스크 I/O: 쿼리마다 로그 기록 → ~5~10% 성능 영향
  최소화: 중요 이벤트만 기록 (audit_log_policy = LOGINS + DDL)
  디스크 공간: 500 TPS × 1KB = 500MB/시간 = 12GB/일
  → 로그 로테이션 필수 (logrotate 설정)

종합 영향:
  커넥션 풀 기반 서비스: SSL 영향 < 1%
  감사 로그: ALL 정책 시 5~10%, LOGINS만 < 1%
```

---

## ⚖️ 트레이드오프

```
보안 설정 트레이드오프:

require_secure_transport = ON:
  ✅ 모든 연결 암호화 강제
  ❌ 레거시 클라이언트 SSL 미지원 시 연결 불가 (마이그레이션 필요)
  ❌ SSL 인증서 관리 필요 (갱신 주기)
  권장: 새 서비스는 기본 ON, 레거시는 단계적 전환

validate_password:
  ✅ 약한 비밀번호 방지
  ❌ 개발 편의성 감소 (복잡한 비밀번호)
  ❌ 비밀번호 만료 시 긴급 대응 절차 필요
  권장: 운영 환경 STRONG 정책, 개발은 MEDIUM

감사 로그:
  ✅ 법적 요구사항 충족 (개인정보보호법)
  ✅ 보안 사고 사후 조사 가능
  ❌ 디스크 공간, 성능 영향
  ❌ 로그 분석 도구 필요 (ELK, Splunk 등)
  권장: 최소 로그인 + DDL + 민감 테이블 쿼리 기록

인증서 만료 리스크:
  SSL 인증서 만료 = 모든 연결 차단 (서비스 중단)
  → 인증서 만료 30일 전 알람 설정 필수
  → 자동 갱신 (certbot, AWS ACM 등) 검토
```

---

## 📌 핵심 정리

```
연결 보안 핵심:

SSL/TLS:
  my.cnf: require_secure_transport = ON
  계정별: ALTER USER ... REQUIRE SSL
  확인: SHOW STATUS LIKE 'Ssl_cipher'

비밀번호 정책:
  INSTALL COMPONENT 'file://component_validate_password'
  validate_password.policy = STRONG
  길이 12자 이상, 대/소문자/숫자/특수문자 포함
  PASSWORD EXPIRE INTERVAL 90 DAY
  FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1

감사 로그:
  Percona Audit Log Plugin (오픈소스) 또는 MySQL Enterprise Audit
  audit_log_policy: 최소 LOGINS + DDL 기록
  보관: 최소 1년 (개인정보보호법)
  분석: jq, ELK Stack 등

체크리스트:
  □ SHOW STATUS LIKE 'Ssl_cipher' → 암호화 연결 확인
  □ validate_password 활성화 확인
  □ 감사 로그 기록 여부 확인
  □ 인증서 만료일 모니터링 설정
  □ 비밀번호 만료 예정 계정 알람
```

---

## 🤔 생각해볼 문제

**Q1.** 운영 환경에서 `require_secure_transport = ON`을 갑자기 활성화하면 어떤 문제가 발생할 수 있으며, 안전하게 전환하는 절차는?

<details>
<summary>해설 보기</summary>

**발생 가능한 문제**:
- SSL을 지원하지 않는 레거시 클라이언트 → 모든 연결 차단 → 서비스 중단
- 모니터링 에이전트, 배치 스크립트 등 SSL 미설정 → 연결 실패

**안전한 전환 절차**:

```bash
# 1단계: 현재 SSL 미사용 연결 파악
SELECT user, host, ssl_cipher
FROM performance_schema.session_ssl_status
WHERE ssl_cipher = '';
# SSL 없는 연결 목록 확인

# 2단계: 각 연결 주체 SSL 설정 업데이트
# 애플리케이션: --ssl-mode=REQUIRED 또는 SSL URL 파라미터 추가
# JDBC: jdbc:mysql://host/db?useSSL=true&requireSSL=true

# 3단계: 모든 연결이 SSL 사용하는지 확인 후
SELECT COUNT(*) FROM performance_schema.session_ssl_status
WHERE ssl_cipher = '';  -- 0이어야 함

# 4단계: require_secure_transport 활성화
SET GLOBAL require_secure_transport = ON;
-- my.cnf에도 추가 (재시작 후에도 유지)

# 5단계: 연결 테스트
mysql --ssl-mode=DISABLED -u test -p  -- Error 3159 확인
```

</details>

---

**Q2.** 개인정보보호법 준수를 위해 감사 로그에서 최소한 기록해야 할 이벤트와 보관 기간을 제시하라.

<details>
<summary>해설 보기</summary>

**최소 기록 이벤트** (개인정보보호법 제29조, 고시 기준):

```
1. 개인정보 처리 시스템 접속 기록:
   - 사용자 ID
   - 접속 일시
   - 접속 IP
   - 처리한 정보주체 수 (가능한 경우)

2. 필수 로그:
   - 로그인/로그아웃 (성공/실패)
   - 개인정보 테이블 SELECT (users, customers 등)
   - DDL 실행 (스키마 변경)
   - 권한 변경 (GRANT/REVOKE)
```

```sql
-- Percona Audit 설정 예시 (개인정보보호법 기준):
SET GLOBAL audit_log_policy = 'ALL';  -- 모든 이벤트

-- 또는 선택적 (성능 절충):
-- 로그인 + 개인정보 테이블 쿼리 + DDL
-- audit_log_include_databases = 'mydb'
-- 특정 테이블(users, payments 등) 접근 감사
```

**보관 기간**:
- 개인정보보호법: 접속기록 최소 6개월 이상 보관 (고시 기준)
- 실무 권장: 1년 이상 (감사/소송 대비)
- 금융권: 5년 이상 (금융감독원 기준)

**로그 보관 방법**:
```bash
# logrotate 설정 (/etc/logrotate.d/mysql-audit)
/var/log/mysql/audit.log {
    daily
    rotate 365  # 1년치 보관
    compress
    notifempty
    missingok
    sharedscripts
    postrotate
        # Percona Audit Plugin 로그 순환 트리거
        mysql -u root -p -e "SET GLOBAL audit_log_rotate_on_size = 0"
    endscript
}
```

</details>

---

<div align="center">

**[⬅️ 사용자 권한 설계](./01-user-privilege-design.md)** | **[홈으로 🏠](../README.md)** | **[다음: 개발/운영 환경 분리 ➡️](./03-dev-prod-separation.md)**

</div>
