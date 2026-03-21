# GTID 기반 복제 — 장애 복구 시 왜 GTID가 중요한가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- GTID가 `server_uuid:transaction_id` 형식으로 트랜잭션을 전역 식별하는 방식은?
- 포지션 기반 복제에서 Failover 시 Binary Log 위치를 수동으로 찾아야 하는 문제는?
- GTID로 Replica 전환이 자동화되는 원리는?
- `@@GLOBAL.GTID_EXECUTED`와 `@@GLOBAL.GTID_PURGED`의 차이는?
- GTID 환경에서 허용되지 않는 SQL 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### Failover 시 "어디서부터 복제를 시작하지?" — 포지션 기반의 고통

```
포지션 기반 복제의 Failover 문제:
  Source 장애 → Replica 중 하나를 새 Source로 승격
  나머지 Replica들: "새 Source의 어느 위치부터 복제 시작하지?"

  포지션 기반:
    각 Replica가 보던 것: 구 Source의 Binary Log 파일명 + 위치
    새 Source: Binary Log 파일이 구 Source와 다름
    → 각 Replica가 새 Source의 어느 위치에서 시작할지 수동 계산
    → mysqlbinlog 분석, 타임스탬프 비교, 수동 CHANGE REPLICATION SOURCE 실행
    → 실수 시 데이터 중복 또는 누락

  GTID 기반:
    각 트랜잭션 = 고유한 전역 ID
    Replica: "내가 이미 처리한 GTID 집합은 X"
    새 Source: "내가 가진 GTID 집합은 Y"
    → 차집합 Y-X를 자동으로 찾아 복제 시작
    → 수동 위치 계산 불필요
    → Orchestrator, MHA 등 자동화 도구 연동 용이
```

---

## 😱 흔한 실수 (Before)

### 1. GTID 제약 없이 마이그레이션 시 기존 코드 충돌

```sql
-- Before: 포지션 기반 환경에서 작동하던 패턴

-- 패턴 1: CREATE TABLE ... SELECT (GTID 불가)
CREATE TABLE orders_backup SELECT * FROM orders;
-- GTID 활성화 시: Error 1786
-- "Statement violates GTID consistency"

-- 패턴 2: 트랜잭션 내 임시 테이블 DDL
BEGIN;
CREATE TEMPORARY TABLE tmp_calc (val INT);
INSERT INTO tmp_calc SELECT amount FROM orders WHERE user_id = 1;
COMMIT;
-- GTID 환경에서 오류 발생

-- 패턴 3: MyISAM + InnoDB 혼합 트랜잭션
BEGIN;
UPDATE innodb_table SET status = 'DONE' WHERE id = 1;
UPDATE myisam_log SET ts = NOW() WHERE id = 1;  -- MyISAM
COMMIT;
-- GTID 불가 (비트랜잭션 테이블 혼합)
```

### 2. 포지션 기반 Failover에서 실수로 데이터 중복/누락

```sql
-- Failover 시 잘못된 위치 지정
CHANGE REPLICATION SOURCE TO
    SOURCE_LOG_FILE = 'mysql-bin.000005',
    SOURCE_LOG_POS = 1230000;  -- 잘못된 위치

-- 이미 처리한 트랜잭션을 다시 받으면: 중복 INSERT
-- 처리하지 않은 트랜잭션을 건너뛰면: 데이터 누락
-- → 데이터 정합성 오염, 발견도 어려움
```

---

## ✨ 올바른 접근 (After)

```
GTID 기반 Failover:

포지션 기반:
  CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='new-source',
    SOURCE_LOG_FILE='mysql-bin.000005',  -- 수동 계산
    SOURCE_LOG_POS=1234567;              -- 수동 계산

GTID 기반:
  CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='new-source',
    SOURCE_AUTO_POSITION=1;  -- 끝!

  내부 동작:
    Replica → 새 Source에 GTID_EXECUTED 전송
    새 Source → "이 Replica가 없는 GTID부터 전송"
    → 자동으로 정확한 위치에서 복제 재개

사전 조건:
  모든 서버에 gtid_mode = ON
  enforce_gtid_consistency = ON
  기존 SQL 패턴 중 GTID 비호환 코드 사전 제거
```

---

## 🔬 내부 동작 원리

### 1. GTID 형식과 생성

```
GTID 형식:
  server_uuid:transaction_id
  예: 3E11FA47-71CA-11E1-9E33-C80AA9429562:1-1000

  server_uuid: MySQL 인스턴스 고유 ID
               data directory의 auto.cnf 파일에 저장
               서버 시작 시 자동 생성, 변경 금지

  transaction_id: 해당 서버에서의 커밋 순번 (1부터 단조증가)

GTID Set (집합):
  여러 서버의 GTID 범위를 표현
  3E11FA47-...:1-1000,8A4AC10B-...:1-500
  → 서버 A의 1~1000번 + 서버 B의 1~500번

GTID 생성 흐름:
  Source에서 트랜잭션 COMMIT 시 → Gtid_log_event 생성
  Binary Log에 기록 → Replica IO Thread가 수신
  Replica가 적용 완료 시 → 자신의 GTID_EXECUTED에 추가

중요 시스템 변수:
  @@GLOBAL.GTID_EXECUTED: 이 서버에서 실행(적용) 완료된 모든 GTID 집합
  @@GLOBAL.GTID_PURGED:   Binary Log에서 삭제된 GTID (파일은 없지만 적용은 됨)
  @@GLOBAL.GTID_OWNED:    현재 진행 중인 트랜잭션의 GTID
```

### 2. GTID Failover 자동화 원리

```sql
-- GTID 기반 SOURCE_AUTO_POSITION=1의 내부 동작:

-- 1단계: Replica가 새 Source에 접속 시 GTID_EXECUTED를 전송
--   Replica: "나는 uuid_A:1-500, uuid_B:1-200까지 처리했어"

-- 2단계: 새 Source가 차집합 계산
--   Source GTID_EXECUTED: uuid_A:1-520, uuid_B:1-200
--   차집합: uuid_A:501-520 (Replica에 없는 20개 트랜잭션)

-- 3단계: Source가 차집합부터 전송
--   uuid_A:501 ~ 520번 트랜잭션 순서대로 전송
--   → Replica가 자동으로 따라잡음

-- 이미 처리한 GTID는 건너뜀:
--   Replica가 이미 가진 GTID를 Source가 다시 전송해도
--   Replica는 해당 GTID를 GTID_EXECUTED에서 확인 후 skip
--   → 중복 적용 없음

SHOW REPLICA STATUS\G
-- Retrieved_Gtid_Set: IO Thread가 수신한 GTID
-- Executed_Gtid_Set:  SQL Thread가 실행 완료한 GTID
-- 차이: 수신됐지만 미실행 트랜잭션 = Relay Log 재고
```

### 3. GTID 환경 설정

```sql
-- 온라인 활성화 (MySQL 8.0, 순서 중요)
SET GLOBAL enforce_gtid_consistency = WARN;        -- 위반 경고 (하위 호환)
SET GLOBAL enforce_gtid_consistency = ON;          -- 위반 차단
SET GLOBAL gtid_mode = OFF_PERMISSIVE;             -- GTID + 기존 혼재
SET GLOBAL gtid_mode = ON_PERMISSIVE;              -- GTID 우선
SET GLOBAL gtid_mode = ON;                         -- GTID 전용

-- my.cnf (영구 설정):
-- [mysqld]
-- gtid_mode = ON
-- enforce_gtid_consistency = ON
-- log_bin = mysql-bin
-- server_id = 1  ← 각 서버 고유값 필수

-- GTID 복제 시작:
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = 'source-server',
    SOURCE_USER = 'repl',
    SOURCE_PASSWORD = 'secret',
    SOURCE_AUTO_POSITION = 1;
START REPLICA;
```

### 4. GTID 제약사항 상세

```sql
-- 제약 1: CREATE TABLE ... SELECT 불가
CREATE TABLE orders_copy SELECT * FROM orders;
-- Error 1786: Statement violates GTID consistency
-- 이유: DDL(CREATE)과 DML(INSERT SELECT)이 하나의 이벤트 안에 섞이면
--       단일 GTID로 처리 불가 (DDL은 암묵적 커밋)

-- 해결:
CREATE TABLE orders_copy LIKE orders;
INSERT INTO orders_copy SELECT * FROM orders;
-- 각각 독립 트랜잭션 → 각각 고유 GTID 할당

-- 제약 2: 트랜잭션 내 TEMPORARY TABLE DDL 불가
BEGIN;
CREATE TEMPORARY TABLE tmp (val INT);  -- Error!
COMMIT;
-- 해결: BEGIN 밖에서 CREATE TEMPORARY TABLE

-- 제약 3: 비트랜잭션 테이블 + 트랜잭션 혼합
BEGIN;
UPDATE innodb_t SET v=1;
UPDATE myisam_t SET v=1;  -- Error!
COMMIT;
-- 해결: MyISAM → InnoDB 전환 또는 별도 실행

-- enforce_gtid_consistency=ON으로 사전 검증:
-- 운영 전 WARN 모드로 위반 패턴 로그 확인 후 코드 수정
```

### 5. 백업/복원 시 GTID 처리

```sql
-- mysqldump에서 GTID 정보 포함
-- mysqldump --single-transaction --master-data=2 --set-gtid-purged=ON db > backup.sql

-- 백업 파일 내 GTID 정보:
-- SET @@GLOBAL.GTID_PURGED='+3E11FA47-...:1-1000';
-- → 복원 후 이 GTID들을 이미 처리된 것으로 표시
-- → 복제 재시작 시 중복 적용 방지

-- XtraBackup:
-- xtrabackup_binlog_info 파일에 GTID_EXECUTED 기록
-- 복원 후 SET GLOBAL GTID_PURGED='...' 설정 필요

-- GTID 상태 확인:
SELECT @@GLOBAL.GTID_EXECUTED;
SELECT @@GLOBAL.GTID_PURGED;
SELECT GTID_SUBTRACT(
    (SELECT @@GLOBAL.GTID_EXECUTED FROM source),
    @@GLOBAL.GTID_EXECUTED
) AS missing_on_replica;
```

---

## 💻 실전 실험

### 실험 1: GTID 상태 확인

```sql
-- GTID 모드 확인
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';

-- 현재 GTID 집합
SELECT @@GLOBAL.GTID_EXECUTED\G
-- 예: 3E11FA47-71CA-11E1-9E33-C80AA9429562:1-5000

-- Replica에서 복제 상태
SHOW REPLICA STATUS\G
-- Retrieved_Gtid_Set vs Executed_Gtid_Set 차이 확인
```

### 실험 2: GTID 제약 확인

```sql
-- enforce_gtid_consistency=ON 환경에서 위반 패턴 테스트
CREATE TABLE gtid_test SELECT 1 AS val;
-- Error 1786 확인

-- 올바른 패턴
CREATE TABLE gtid_test (val INT);
INSERT INTO gtid_test VALUES (1);
-- 성공

DROP TABLE gtid_test;
```

### 실험 3: GTID 누락 트랜잭션 확인

```sql
-- Source GTID_EXECUTED
-- (Source 서버에서 실행)
SELECT @@GLOBAL.GTID_EXECUTED INTO @source_gtid;

-- Replica에서 누락 확인
SELECT GTID_SUBTRACT(
    '3E11FA47-71CA-11E1-9E33-C80AA9429562:1-1000',  -- Source 값
    @@GLOBAL.GTID_EXECUTED                           -- Replica 현재 값
) AS missing_gtids;
-- 빈 문자열이면 동기화 완료
-- 값이 있으면 해당 GTID 미적용 (Lag 상태)
```

---

## 📊 성능/비용 비교

```
포지션 기반 vs GTID 기반 비교:

Failover 소요 시간:
  포지션 기반:
    mysqlbinlog 분석: 5~30분
    수동 위치 계산: 5~15분
    각 Replica 재설정: 2~5분 × Replica 수
    총 Failover 시간: 15분~1시간 이상

  GTID 기반:
    Orchestrator/MHA 자동 감지: 10~30초
    SOURCE_AUTO_POSITION=1 설정: 수 초
    총 Failover 시간: 30초~3분

운영 복잡도:
  포지션 기반:
    Binary Log 파일명 + 위치 추적 필요
    Failover 시 수동 개입 필수
    실수 발생 시 데이터 오염 위험

  GTID 기반:
    트랜잭션 ID로 추적 (직관적)
    자동화 도구 연동 용이
    중복/누락 자동 방지
    단, 기존 코드 GTID 호환성 검토 필요
```

---

## ⚖️ 트레이드오프

```
GTID 도입 트레이드오프:

이점:
  ✅ Failover 자동화 → MTTR 대폭 감소
  ✅ 복제 위치 추적 단순화
  ✅ 중복/누락 트랜잭션 자동 방지
  ✅ Orchestrator, MHA, ProxySQL 연동 용이
  ✅ 백업 복원 후 복제 재시작 간소화

비용:
  ❌ CREATE TABLE ... SELECT 불가 → 코드 수정 필요
  ❌ 트랜잭션 내 TEMPORARY TABLE DDL 불가
  ❌ MyISAM + InnoDB 혼합 트랜잭션 불가
  ❌ 기존 포지션 기반 환경에서 전환 절차 필요

도입 판단:
  신규 서비스 → GTID 기본 적용 권장
  레거시 서비스 → enforce_gtid_consistency=WARN으로 위반 패턴 사전 파악
                  위반 코드 수정 완료 후 전환

절충안:
  GTID 없이 포지션 기반 유지 + pt-heartbeat로 Lag 모니터링
  Failover 자동화는 Orchestrator의 포지션 기반 로직 사용
  → GTID보다 복잡하지만 코드 수정 없이 운영 가능
```

---

## 📌 핵심 정리

```
GTID 핵심:

형식:
  server_uuid:transaction_id
  전 세계에서 유일한 트랜잭션 식별자

Failover 자동화:
  SOURCE_AUTO_POSITION=1
  Replica가 GTID_EXECUTED 전송 → Source가 차집합 자동 계산
  수동 Binary Log 위치 계산 불필요

중요 변수:
  GTID_EXECUTED: 적용 완료된 GTID 집합
  GTID_PURGED:   Binary Log 삭제됐지만 적용 완료된 GTID
  Retrieved vs Executed: IO Thread 수신 vs SQL Thread 적용 차이

GTID 제약:
  CREATE TABLE ... SELECT 불가 → 2단계로 분리
  트랜잭션 내 임시 테이블 DDL 불가
  비트랜잭션 테이블 혼합 불가
  → enforce_gtid_consistency=WARN으로 사전 검증

도입 절차:
  WARN → enforce → OFF_PERMISSIVE → ON_PERMISSIVE → ON
  단계적 전환으로 위반 패턴 사전 파악
```

---

## 🤔 생각해볼 문제

**Q1.** 포지션 기반 복제 환경에서 Source 장애 후 Failover 절차와, GTID 환경에서 동일 상황의 절차를 비교하라.

<details>
<summary>해설 보기</summary>

**포지션 기반 Failover (복잡)**:
```sql
-- 1. 각 Replica의 복제 상태 확인 (수동)
SHOW REPLICA STATUS\G
-- Exec_Source_Log_Pos 비교 → 가장 앞선 Replica 선정

-- 2. 선정된 Replica를 새 Source로 승격
STOP REPLICA;
RESET REPLICA ALL;

-- 3. 나머지 Replica들: 새 Source의 Binary Log 위치 수동 계산
-- mysqlbinlog으로 새 Source의 Binary Log 분석 필요
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='new-source',
    SOURCE_LOG_FILE='mysql-bin.000005',  -- 수동 계산
    SOURCE_LOG_POS=1234567;              -- 수동 계산
START REPLICA;
```

**GTID 기반 Failover (자동화)**:
```sql
-- 1. 각 Replica의 GTID_EXECUTED 비교 (자동화 도구 수행)
-- 가장 큰 GTID 집합을 가진 Replica를 새 Source로 선정

-- 2. 선정된 Replica를 새 Source로 승격
STOP REPLICA;
RESET REPLICA ALL;

-- 3. 나머지 Replica들: 모두 동일한 명령
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='new-source',
    SOURCE_AUTO_POSITION=1;  -- GTID가 알아서 위치 결정
START REPLICA;
```

GTID 환경에서는 Orchestrator, MHA 등 자동화 도구가 1~3단계 전체를 30초 이내에 수행합니다.

</details>

---

**Q2.** `enforce_gtid_consistency=WARN` 모드로 설정 후 Error Log에서 다음 메시지가 발견됐다. 원인 코드를 찾아 수정 방법을 제시하라.

```
[Warning] Statement violates GTID consistency: CREATE TABLE ... SELECT
[Warning] Statement violates GTID consistency: Updates to non-transactional table
```

<details>
<summary>해설 보기</summary>

**첫 번째 경고 — CREATE TABLE ... SELECT**:
```sql
-- 위반 코드
CREATE TABLE report_2024 SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- 수정
CREATE TABLE report_2024 LIKE orders;
INSERT INTO report_2024 SELECT * FROM orders WHERE YEAR(created_at) = 2024;
```

**두 번째 경고 — 비트랜잭션 테이블 혼합**:
```sql
-- 위반 코드 (MyISAM 테이블과 InnoDB 혼합)
BEGIN;
UPDATE orders SET status = 'PROCESSED' WHERE id = 1;
UPDATE audit_log SET ts = NOW() WHERE id = 1;  -- audit_log가 MyISAM
COMMIT;

-- 수정 방법 1: audit_log를 InnoDB로 전환
ALTER TABLE audit_log ENGINE=InnoDB;

-- 수정 방법 2: 트랜잭션 밖으로 분리 (원자성 포기)
BEGIN;
UPDATE orders SET status = 'PROCESSED' WHERE id = 1;
COMMIT;
UPDATE audit_log SET ts = NOW() WHERE id = 1;  -- 별도 실행
```

`enforce_gtid_consistency=WARN` 모드를 일정 기간 운영하며 Error Log를 수집 → 위반 패턴 전수 수정 완료 후 → ON으로 전환합니다.

</details>

---

<div align="center">

**[⬅️ Binary Log 포맷](./02-binlog-formats-statement-row-mixed.md)** | **[홈으로 🏠](../README.md)** | **[다음: Replication Lag 분석 ➡️](./04-replication-lag-analysis.md)**

</div>
