# 운영 중 발생하는 문제 패턴 — Metadata Lock, Buffer Pool 부족, 디스크 풀

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- DDL 대기 → Metadata Lock이 뒤따르는 모든 DML을 차단하는 연쇄 패턴의 원인과 해결은?
- Buffer Pool 히트율 저하가 Disk I/O 폭증으로 이어지는 신호와 대응은?
- 디스크 풀로 인한 Binary Log / Undo Tablespace 팽창을 어떻게 진단하고 대응하는가?
- 각 문제 패턴을 어떤 도구로 진단하는가?
- 문제 발생 전 예방하는 모니터링 알람 설정은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 운영 중 발생하는 장애는 패턴이 있다

```
패턴 1: Metadata Lock 연쇄 차단
  오전 9시: 개발자가 ALTER TABLE 실행 (대형 테이블)
  오전 9시: ALTER TABLE이 장기 트랜잭션 때문에 MDL 대기
  오전 9시: ALTER TABLE 대기 중 → 이 테이블의 모든 SELECT/INSERT/UPDATE도 MDL 대기!
  오전 9:10: SHOW PROCESSLIST → 수십 개 쿼리가 "Waiting for table metadata lock"
  오전 9:15: 서비스 응답 없음 → 장애 선언

패턴 2: Buffer Pool 히트율 저하
  오전 2시: 배치 Full Scan 100GB 테이블 실행
  오전 2시: Buffer Pool이 배치 데이터로 가득 참 (Cache Pollution)
  오전 2시~4시: 서비스 쿼리 캐시 미스 → 디스크 I/O 폭증
  오전 2~4시: 응답 시간 10배 증가

패턴 3: 디스크 풀
  갑작스러운 디스크 사용량 100%
  MySQL 쓰기 실패 → ERROR 1030: Got error -1 from storage engine
  Binary Log 팽창 또는 Undo Tablespace 팽창
```

---

## 😱 흔한 실수 (Before)

### 1. 운영 시간에 대형 테이블 DDL 직접 실행

```sql
-- Before: 서비스 시간에 ALTER TABLE 직접 실행
ALTER TABLE orders ADD INDEX idx_new(status, created_at);
-- 이 명령이 MDL 대기 → 연쇄 차단 시작

-- 문제:
-- 이전에 열린 트랜잭션 하나만 있어도 MDL 대기
-- MDL 대기 → 이 테이블의 새 쿼리도 MDL 큐에 대기
-- 결과: 순간적으로 수십 개 쿼리 대기 → 서비스 응답 없음
```

### 2. Buffer Pool 오염 모니터링 없음

```sql
-- Before: innodb_buffer_pool_reads 모니터링 없음
-- 배치 실행 후 서비스 느려져도 원인 파악 불가

-- 증상: 배치 시간대 서비스 응답 시간 증가
-- 원인: Buffer Pool 오염으로 히트율 저하
-- 해결 모름: 배치 쿼리에 SQL_NO_CACHE 또는 다른 접근 필요
```

### 3. 디스크 사용량 알람 없이 Undo 팽창 방치

```bash
# Before: 디스크 사용량 알람 없음
# df -h → 98% (이미 위험한 상태)
# MySQL 쓰기 실패 시작

# 선행 징조 무시:
# - Undo Tablespace가 수 GB 이상으로 팽창
# - Binary Log 만료 설정 없음
# - innodb_undo_log_truncate = OFF
```

---

## ✨ 올바른 접근 (After)

```
문제 유형별 진단 및 대응:

Metadata Lock 연쇄 차단:
  진단: SHOW PROCESSLIST → "Waiting for table metadata lock"
  원인 찾기: 장기 트랜잭션이 MDL 보유
  즉각 대응: KILL [장기 트랜잭션 thread_id]
  예방: 운영 시간 DDL 금지, pt-osc/gh-ost 사용

Buffer Pool 히트율 저하:
  진단: Innodb_buffer_pool_reads 급증
  원인: Full Scan 배치 쿼리가 Buffer Pool 오염
  즉각 대응: 배치 쿼리 중단 또는 innodb_buffer_pool_size 증가
  예방: 배치 쿼리에 LIMIT + SLEEP, 읽기 전용 Replica에서 실행

디스크 풀:
  진단: df -h, du -sh /var/lib/mysql/*
  원인별 대응:
    Binary Log: PURGE BINARY LOGS, expire 설정
    Undo Tablespace: 장기 트랜잭션 종료, innodb_undo_log_truncate=ON
    임시 파일: KILL 대용량 쿼리
  예방: 디스크 80% 알람, 자동 Binary Log 만료 설정
```

---

## 🔬 내부 동작 원리

### 1. Metadata Lock 연쇄 차단 상세

```
MDL(Metadata Lock) 동작:

DDL 실행 시:
  ALTER TABLE orders ADD INDEX ... → X-MDL 필요
  X-MDL 획득 전: 기존 S-MDL 모두 해제될 때까지 대기

장기 트랜잭션의 영향:
  BEGIN;
  SELECT * FROM orders; → S-MDL 획득 (트랜잭션 종료까지 유지)
  [트랜잭션 미완료]

  이 상태에서:
  ALTER TABLE orders ... → X-MDL 대기 (S-MDL이 해제 안 됨)
  
  MDL 큐에 X-MDL 대기자가 있으면:
  새로운 SELECT orders → S-MDL 획득 시도 → X-MDL 대기자 뒤에 큐잉!
  = 새 SELECT도 차단됨 (MDL 큐 FIFO)

결과:
  수십 개의 쿼리가 "Waiting for table metadata lock" 상태
  = 서비스 응답 없음

진단:
  SHOW PROCESSLIST → "Waiting for table metadata lock"
  SELECT * FROM performance_schema.metadata_locks WHERE OBJECT_NAME='orders';
  SELECT * FROM information_schema.INNODB_TRX ORDER BY trx_started;
  → 가장 오래된 트랜잭션 찾기 → KILL
```

### 2. MDL 연쇄 차단 진단 및 해결

```sql
-- 1. MDL 대기 확인
SHOW PROCESSLIST;
-- State: Waiting for table metadata lock 여러 개 확인

-- 2. MDL 보유자 찾기
SELECT
    ml.OBJECT_SCHEMA,
    ml.OBJECT_NAME,
    ml.LOCK_TYPE,
    ml.LOCK_DURATION,
    ml.LOCK_STATUS,
    p.ID AS process_id,
    p.TIME AS waiting_sec,
    p.STATE,
    p.INFO AS query
FROM performance_schema.metadata_locks ml
JOIN performance_schema.threads t ON t.THREAD_ID = ml.OWNER_THREAD_ID
JOIN information_schema.PROCESSLIST p ON p.ID = t.PROCESSLIST_ID
WHERE ml.OBJECT_NAME = 'orders'
ORDER BY ml.LOCK_STATUS DESC;  -- GRANTED 먼저

-- 3. 장기 트랜잭션 찾기
SELECT
    t.trx_id,
    t.trx_started,
    TIMESTAMPDIFF(SECOND, t.trx_started, NOW()) AS duration_sec,
    p.ID AS thread_id,
    LEFT(p.INFO, 200) AS last_query
FROM information_schema.INNODB_TRX t
JOIN information_schema.PROCESSLIST p ON p.ID = t.trx_mysql_thread_id
WHERE TIMESTAMPDIFF(SECOND, t.trx_started, NOW()) > 30
ORDER BY t.trx_started;

-- 4. 즉각 종료
KILL [thread_id];  -- 장기 트랜잭션 종료
-- 이후 ALTER TABLE 진행, 대기 쿼리들도 해소

-- 5. 예방: lock_wait_timeout 설정
-- DDL 대기 최대 시간 제한 (기본: 365일 = 사실상 무제한)
SET GLOBAL lock_wait_timeout = 60;  -- 60초 대기 후 DDL 오류 반환
```

### 3. Buffer Pool 히트율 저하 패턴

```sql
-- Buffer Pool 실시간 모니터링
SELECT
    Innodb_buffer_pool_reads.VARIABLE_VALUE AS disk_reads,
    Innodb_buffer_pool_read_requests.VARIABLE_VALUE AS total_requests
FROM
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') Innodb_buffer_pool_reads,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') Innodb_buffer_pool_read_requests;

-- 기간별 히트율 변화 추적 (before/after 비교)
-- 배치 시작 전 기록
-- 배치 시작 후 주기적으로 재조회
-- disk_reads 증가 속도로 Buffer Pool 오염 감지

-- Buffer Pool 오염 증상:
-- 1. Innodb_buffer_pool_reads 급증 (디스크 읽기 증가)
-- 2. 서비스 응답 시간 증가 (캐시 미스로 I/O 대기)
-- 3. iostat에서 %util 100% 근접

-- 배치 쿼리에서 Buffer Pool 오염 방지:
-- Replica에서 실행 (서비스 DB 영향 없음)
-- 또는 innodb_buffer_pool_dump/restore 활용
--   배치 전 Buffer Pool dump → 배치 후 restore
--   서비스 쿼리용 Hot Page를 복구

-- Full Scan 배치에 버퍼 오염 방지:
-- innodb_read_ahead_threshold = 0 (Read-Ahead 비활성)
-- 또는 Read Only 세션으로 별도 처리
```

### 4. 디스크 풀 패턴 진단

```bash
# 디스크 사용량 확인
df -h /var/lib/mysql
du -sh /var/lib/mysql/* | sort -rh | head -20

# Binary Log 팽창 확인
ls -la /var/lib/mysql/mysql-bin.*
# 파일 크기 × 개수 = Binary Log 총 사용량

# Undo Tablespace 팽창 확인
du -sh /var/lib/mysql/undo_*
# 또는
SELECT file_name, round(total_extents * extent_size / 1024/1024, 2) AS size_mb
FROM information_schema.FILES
WHERE file_type = 'UNDO LOG';
```

```sql
-- Binary Log 긴급 정리
SHOW BINARY LOGS;
-- 파일 목록과 크기 확인

PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);
-- 3일 이전 Binary Log 삭제 (복제 지연 고려)
-- 또는
PURGE BINARY LOGS TO 'mysql-bin.000100';
-- 특정 파일 이전 삭제

-- 자동 만료 설정 (근본 해결)
SET GLOBAL binlog_expire_logs_seconds = 604800;  -- 7일
-- my.cnf: binlog_expire_logs_seconds = 604800

-- Undo Tablespace 팽창 원인 찾기
SELECT
    t.trx_id,
    t.trx_started,
    TIMESTAMPDIFF(SECOND, t.trx_started, NOW()) AS duration_sec,
    t.trx_rows_modified
FROM information_schema.INNODB_TRX t
ORDER BY trx_started;
-- 오래된 트랜잭션 → Undo Log 쌓임
-- KILL 후 innodb_undo_log_truncate = ON으로 자동 정리

-- 임시 파일 확인 (대용량 쿼리 실행 중)
SELECT
    processlist_id,
    processlist_time AS running_sec,
    processlist_info AS query
FROM performance_schema.threads
WHERE processlist_command = 'Query'
  AND processlist_time > 60
ORDER BY processlist_time DESC;
-- 60초 이상 실행 중인 쿼리 → KILL 고려
```

### 5. 예방적 모니터링 설정

```sql
-- 핵심 알람 지표:

-- 1. Metadata Lock 대기 누적 알람
-- SHOW PROCESSLIST의 "Waiting for table metadata lock" 수 > 5이면 알람

-- 2. Buffer Pool 히트율 알람
SELECT
    ROUND(100 * (1 - reads/requests), 2) AS hit_rate
FROM
    (SELECT VARIABLE_VALUE AS reads FROM performance_schema.global_status
     WHERE VARIABLE_NAME='Innodb_buffer_pool_reads') r,
    (SELECT VARIABLE_VALUE AS requests FROM performance_schema.global_status
     WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests') req;
-- < 99%: 경고 알람

-- 3. 장기 트랜잭션 알람
SELECT COUNT(*) AS long_trx_count
FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;
-- > 0: 알람

-- 4. 복제 지연 알람
-- SHOW REPLICA STATUS → Seconds_Behind_Source > 30: 경고

-- 5. 디스크 사용량 알람 (OS 레벨)
-- /var/lib/mysql: > 80% 경고, > 90% 위험

-- Prometheus + mysqld_exporter 활용:
-- mysql_global_status_innodb_buffer_pool_reads_rate → 히트율
-- mysql_info_schema_innodb_trx_seconds → 트랜잭션 지속 시간
-- mysql_global_status_threads_waiting_for_mdl → MDL 대기
```

---

## 💻 실전 실험

### 실험 1: MDL 연쇄 차단 재현 및 해결

```sql
-- 세션 1: 장기 트랜잭션 시작
START TRANSACTION;
SELECT * FROM orders LIMIT 1;  -- S-MDL 획득
-- COMMIT하지 않고 대기

-- 세션 2: DDL 시도
ALTER TABLE orders ADD COLUMN test_col INT;
-- State: Waiting for table metadata lock

-- 세션 3: 일반 SELECT 시도
SELECT * FROM orders LIMIT 1;
-- State: Waiting for table metadata lock (DDL 대기자 때문에 큐잉!)

-- 진단 (세션 4):
SHOW PROCESSLIST;
SELECT * FROM performance_schema.metadata_locks WHERE OBJECT_NAME='orders';

-- 해결 (세션 4):
-- 세션 1의 thread_id 찾아서 KILL
KILL [세션1_thread_id];
-- 세션 2의 DDL 진행, 세션 3도 해소
```

### 실험 2: Buffer Pool 히트율 모니터링

```bash
# 배치 실행 전 히트율 기록
mysql -u root -p -e "
SELECT
    ROUND(100 * (1 - reads/requests), 4) AS hit_rate
FROM
    (SELECT VARIABLE_VALUE AS reads FROM performance_schema.global_status
     WHERE VARIABLE_NAME='Innodb_buffer_pool_reads') r,
    (SELECT VARIABLE_VALUE AS requests FROM performance_schema.global_status
     WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests') req;
"

# 배치 실행 (대용량 Full Scan)
# mysql -e "SELECT * FROM big_table;" > /dev/null &

# 배치 중 히트율 변화 모니터링 (5초마다)
watch -n 5 'mysql -u root -p -e "
SELECT ROUND(100 * (1 - reads/requests), 4) AS hit_rate FROM ..."'
```

### 실험 3: Binary Log 크기 관리

```sql
-- 현재 Binary Log 사용량
SHOW BINARY LOGS;

-- 만료 설정 확인
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
-- 0이면 자동 만료 없음 (위험!)

-- 자동 만료 설정
SET GLOBAL binlog_expire_logs_seconds = 604800;  -- 7일

-- 긴급 삭제 (복제 Lag 확인 후)
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source 확인 후 안전한 시점 결정
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);
```

---

## 📊 성능/비용 비교

```
문제 패턴별 영향도와 복구 시간:

Metadata Lock 연쇄 차단:
  영향도: 매우 큼 (해당 테이블 모든 쿼리 차단)
  발생 속도: 즉각 (DDL 실행 시)
  복구 방법: 장기 트랜잭션 KILL
  복구 시간: KILL 후 수 초 이내
  예방: 운영 시간 DDL 금지, lock_wait_timeout 설정

Buffer Pool 히트율 저하:
  영향도: 중간~큼 (서비스 응답 시간 증가)
  발생 속도: 수 분~수십 분에 걸쳐 악화
  복구 방법: 배치 중단, Buffer Pool 자연 회복
  복구 시간: 수십 분 (Buffer Pool이 서비스 데이터로 채워지면서)
  예방: 배치를 Replica에서 실행, innodb_buffer_pool_size 증가

디스크 풀:
  영향도: 치명적 (쓰기 불가 = 서비스 중단)
  발생 속도: 서서히 (알람 없으면 모름)
  복구 방법: Binary Log 삭제, 장기 트랜잭션 종료
  복구 시간: 즉각 (공간 확보 후)
  예방: 80% 알람, Binary Log 만료 설정, Undo 트런케이트 활성화
```

---

## ⚖️ 트레이드오프

```
각 예방 조치 트레이드오프:

lock_wait_timeout 단축:
  ✅ MDL 연쇄 차단 자동 해소 (DDL이 실패하지만 차단 해소)
  ❌ 운영 시간 DDL 항상 실패 (적절한 시간 선택 필요)
  권장값: 30~60초 (대형 테이블 DDL은 트래픽 낮은 시간대 실행)

innodb_buffer_pool_size 증가:
  ✅ 히트율 향상, 배치 오염 영향 감소
  ❌ 메모리 비용 증가 (서버 RAM 비용)
  권장: RAM의 50~70% (OS 오버헤드 고려)

binlog_expire_logs_seconds:
  ✅ 디스크 자동 관리
  ❌ 너무 짧으면 PITR 범위 축소
  권장: Full Backup 주기 × 2 (일반적으로 7~14일)

innodb_undo_log_truncate:
  ON: Undo 자동 정리
  ❌ 트런케이트 작업 중 약간의 I/O 발생
  권장: ON (장기 트랜잭션 방지와 병행)
```

---

## 📌 핵심 정리

```
운영 문제 패턴 핵심:

Metadata Lock 연쇄 차단:
  원인: DDL 대기 + 새 쿼리 MDL 큐잉
  진단: SHOW PROCESSLIST → "Waiting for table metadata lock"
  해결: 장기 트랜잭션 KILL → DDL 진행
  예방: 운영 시간 DDL 금지, lock_wait_timeout 설정

Buffer Pool 히트율 저하:
  원인: Full Scan 배치가 Buffer Pool 오염
  진단: Innodb_buffer_pool_reads 급증
  해결: 배치 중단, Replica로 이동
  예방: Buffer Pool 히트율 알람 (< 99%)

디스크 풀:
  원인: Binary Log 무한 누적, Undo Tablespace 팽창
  진단: df -h, du -sh /var/lib/mysql/*
  해결: PURGE BINARY LOGS, 장기 트랜잭션 KILL
  예방: binlog 자동 만료, 디스크 80% 알람

공통 예방:
  장기 트랜잭션: innodb_lock_wait_timeout = 50 (기본)
  배치 쿼리: Replica에서 실행 (서비스 DB 영향 없음)
  모니터링: Prometheus + 알람 (Buffer Pool, 디스크, 복제 지연)
```

---

## 🤔 생각해볼 문제

**Q1.** 매일 오전 2시 배치 작업 이후 오전 2~4시 사이 서비스 응답 시간이 3배로 증가한다. 원인을 찾고 해결하는 절차를 설명하라.

<details>
<summary>해설 보기</summary>

**원인 분석**:

```sql
-- 1. 배치 시간대 Buffer Pool 히트율 확인
-- (Prometheus 또는 SHOW STATUS 주기적 기록)
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN ('Innodb_buffer_pool_reads', 'Innodb_buffer_pool_read_requests');
-- 오전 2시 이후 reads 급증 → Buffer Pool 오염

-- 2. 배치 쿼리 확인
SELECT query, exec_count, total_latency, no_index_used_count
FROM sys.statements_with_full_table_scans
WHERE last_seen BETWEEN '2024-01-15 02:00:00' AND '2024-01-15 03:00:00'
ORDER BY total_latency DESC;
-- Full Scan 배치 쿼리 식별
```

**해결 방법**:

```bash
# 단기: 배치를 Replica에서 실행
mysql -h replica-host -u batch_user -p < batch.sql
# 서비스 DB Buffer Pool 보호

# 중기: 배치 쿼리 최적화 (인덱스 추가)
# EXPLAIN으로 Full Scan 제거

# 장기: innodb_buffer_pool_size 증가
# 또는 innodb_buffer_pool_dump/restore로 배치 전후 Hot Page 보존
SET GLOBAL innodb_buffer_pool_dump_now = ON;  -- 배치 전
SET GLOBAL innodb_buffer_pool_load_now = ON;  -- 배치 후 (서비스 데이터 복구)
```

</details>

---

**Q2.** 디스크 사용량이 95%이고 MySQL 쓰기가 간헐적으로 실패하기 시작했다. 즉각 조치 순서를 설명하라.

<details>
<summary>해설 보기</summary>

**즉각 조치 순서 (우선순위 순)**:

```bash
# 1. 현황 파악 (1분)
df -h /var/lib/mysql
du -sh /var/lib/mysql/* | sort -rh | head -20
# Binary Log vs Undo vs 데이터 파일 중 무엇이 큰지 확인

# 2. Binary Log 긴급 삭제 (가장 빠른 공간 확보)
mysql -u root -p -e "SHOW BINARY LOGS;"
# 최근 3일치 제외하고 삭제 (복제 지연 확인 후)
mysql -u root -p -e "SHOW REPLICA STATUS\G"  # Seconds_Behind_Source 확인
mysql -u root -p -e "PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);"
```

```sql
-- 3. 장기 트랜잭션 확인 및 종료 (Undo 팽창 원인)
SELECT trx_id, TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS sec,
       trx_mysql_thread_id
FROM information_schema.INNODB_TRX
ORDER BY trx_started LIMIT 10;
-- 오래된 트랜잭션 KILL

-- 4. 대용량 임시 파일 생성 쿼리 확인
SELECT processlist_id, processlist_time, processlist_info
FROM performance_schema.threads
WHERE processlist_command = 'Query'
  AND processlist_time > 60
ORDER BY processlist_time DESC;
-- KILL [processlist_id]

-- 5. 단기 해소 후 장기 대책
-- binlog_expire_logs_seconds = 604800 영구 설정
-- innodb_undo_log_truncate = ON
-- 디스크 80% 알람 추가
```

</details>

---

<div align="center">

**[⬅️ MySQL 8.0 새 기능](./04-mysql8-new-features.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 보안 ➡️](../security-user-management/01-user-privilege-design.md)**

</div>
