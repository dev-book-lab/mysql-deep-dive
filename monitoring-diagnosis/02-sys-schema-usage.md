# sys 스키마 — 운영자를 위한 DBA 뷰 활용법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `sys.statements_with_full_table_scans`으로 Full Scan 쿼리를 즉시 확인하는 방법은?
- `sys.schema_unused_indexes`로 사용 안 되는 인덱스 제거 후보를 추출하는 방법은?
- `sys.innodb_lock_waits`로 현재 Lock 대기 체인을 파악하는 방법은?
- sys 스키마가 Performance Schema와 다른 점은?
- 운영 중 즉각 활용할 수 있는 sys 스키마 핵심 뷰 10가지는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### Performance Schema는 강력하지만 쿼리가 복잡하다

```
Performance Schema 직접 사용:
  SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT / 1e12 AS total_sec,
         SUM_ROWS_EXAMINED / COUNT_STAR AS avg_rows_examined
  FROM performance_schema.events_statements_summary_by_digest
  WHERE SUM_NO_INDEX_USED > 0
  ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
  → 매번 작성 + 단위 변환 필요

sys 스키마 사용:
  SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;
  → 한 줄, 결과도 읽기 쉬움

sys 스키마 = Performance Schema + information_schema의
             가독성 높은 뷰 모음
             단위 변환 (피코초 → ms/s), 포맷팅 포함
```

---

## 😱 흔한 실수 (Before)

### 1. sys 스키마 없이 복잡한 쿼리 직접 작성

```sql
-- Before: Performance Schema 직접 쿼리
SELECT
    OBJECT_SCHEMA, OBJECT_NAME,
    SUM(COUNT_READ + COUNT_WRITE) AS total_io,
    SUM(SUM_TIMER_WAIT) / 1e12 AS total_sec
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA = 'mydb'
GROUP BY OBJECT_SCHEMA, OBJECT_NAME
ORDER BY total_io DESC;

-- sys 스키마로 동일한 정보를 더 쉽게:
SELECT * FROM sys.io_global_by_file_by_bytes LIMIT 10;
```

### 2. 인덱스 삭제 전 사용 여부 미확인

```sql
-- Before: 느낌상 안 쓰이는 것 같은 인덱스 바로 삭제
DROP INDEX idx_old_column ON orders;

-- 문제: 이 인덱스가 특정 쿼리에서 사용 중이었다면
-- 삭제 후 해당 쿼리 Full Scan → 서비스 영향

-- 올바른 방법:
SELECT * FROM sys.schema_unused_indexes
WHERE object_schema = 'mydb';
-- 재시작 이후 한 번도 사용 안 된 인덱스만 제거 후보
-- Invisible Index로 먼저 비활성화 후 영향 확인 (04 문서 참고)
```

### 3. Lock 대기 원인을 SHOW PROCESSLIST로만 확인

```sql
-- Before: SHOW PROCESSLIST로 Lock 확인 시도
SHOW PROCESSLIST;
-- State: Waiting for table lock (누구 때문에 기다리는지 모름)

-- 올바른 방법:
SELECT * FROM sys.innodb_lock_waits;
-- waiting_query: 대기 중인 쿼리
-- blocking_query: 락을 보유한 쿼리
-- 즉시 원인 파악 가능
```

---

## ✨ 올바른 접근 (After)

```
sys 스키마 핵심 뷰 분류:

쿼리 성능 분석:
  sys.statements_with_full_table_scans  → Full Scan 쿼리
  sys.statements_with_sorting           → 정렬 사용 쿼리
  sys.statements_with_temp_tables       → 임시 테이블 사용 쿼리
  sys.top_threads                       → CPU 사용 많은 스레드

인덱스 분석:
  sys.schema_unused_indexes             → 미사용 인덱스
  sys.schema_redundant_indexes          → 중복 인덱스
  sys.schema_index_statistics           → 인덱스 사용 통계

Lock/대기 분석:
  sys.innodb_lock_waits                 → 현재 Lock 대기
  sys.schema_table_lock_waits           → 테이블 락 대기

I/O 분석:
  sys.io_global_by_file_by_bytes        → 파일별 I/O
  sys.io_global_by_wait_by_bytes        → 대기별 I/O

메모리 분석:
  sys.memory_global_by_current_bytes    → 메모리 사용
  sys.memory_by_host_by_current_bytes   → 호스트별 메모리
```

---

## 🔬 내부 동작 원리

### 1. sys 스키마 구조

```
sys 스키마 = 뷰 모음:
  실제 데이터 저장 없음
  Performance Schema + information_schema 뷰
  가독성 향상 처리:
    피코초 → 'x ms', 'x s', 'x min' (format_time 함수)
    바이트 → 'x KiB', 'x MiB', 'x GiB'

sys 스키마 설치:
  MySQL 5.7.7+, 8.0: 기본 설치됨
  없다면: github.com/mysql/mysql-sys에서 설치

sys.statements_with_full_table_scans 내부:
  FROM performance_schema.events_statements_summary_by_digest
  WHERE (SUM_NO_INDEX_USED > 0 OR SUM_NO_GOOD_INDEX_USED > 0)
  → Full Scan 이력이 있는 쿼리만 필터링
  → 실행 시간, 횟수 등 포맷팅하여 반환
```

### 2. 핵심 뷰 상세

```sql
-- 1. Full Scan 쿼리 확인
SELECT
    db,
    query,
    exec_count,
    total_latency,
    rows_examined_avg,
    rows_sent_avg,
    tmp_tables,
    no_index_used_count,
    last_seen
FROM sys.statements_with_full_table_scans
WHERE db = 'mydb'
ORDER BY total_latency DESC
LIMIT 10;

-- 2. 미사용 인덱스 확인
SELECT
    object_schema,
    object_name AS table_name,
    index_name,
    reason
FROM sys.schema_unused_indexes
WHERE object_schema = 'mydb'
  AND object_name NOT LIKE '%test%'
ORDER BY object_name, index_name;
-- reason: INDEX_NEVER_USED 또는 INDEX_NOT_VISIBLE

-- 3. 중복 인덱스 확인
SELECT
    table_schema,
    table_name,
    redundant_index_name,
    redundant_index_columns,
    dominant_index_name,
    dominant_index_columns
FROM sys.schema_redundant_indexes
WHERE table_schema = 'mydb';
-- redundant_index가 dominant_index의 prefix인 경우
-- 예: idx(a) 와 idx(a, b)가 동시 존재 → idx(a)는 중복

-- 4. 현재 Lock 대기 체인
SELECT
    wait_started,
    wait_age,
    waiting_trx_id,
    waiting_pid,
    waiting_query,
    blocking_trx_id,
    blocking_pid,
    blocking_query,
    blocking_trx_started,
    blocking_trx_age
FROM sys.innodb_lock_waits;

-- 5. 임시 테이블 사용 쿼리
SELECT
    db, query, exec_count,
    total_latency, memory_tmp_tables, disk_tmp_tables,
    tmp_tables_to_disk_pct
FROM sys.statements_with_temp_tables
WHERE db = 'mydb'
ORDER BY disk_tmp_tables DESC
LIMIT 10;
-- disk_tmp_tables > 0: 임시 테이블이 디스크로 넘어감
-- → tmp_table_size 또는 max_heap_table_size 증가 검토
```

### 3. 인덱스 관리 뷰 활용

```sql
-- 인덱스 통계 (테이블별 인덱스 사용 현황)
SELECT
    table_schema,
    table_name,
    index_name,
    rows_selected,
    rows_inserted,
    rows_updated,
    rows_deleted
FROM sys.schema_index_statistics
WHERE table_schema = 'mydb'
ORDER BY rows_selected DESC
LIMIT 20;
-- rows_selected: 이 인덱스로 조회된 행 수
-- 0이면 인덱스 미사용 → schema_unused_indexes에서도 확인

-- 인덱스 크기 확인
SELECT
    table_schema,
    table_name,
    index_name,
    sys.format_bytes(stat_value * @@innodb_page_size) AS index_size
FROM mysql.innodb_index_stats
WHERE stat_name = 'size'
  AND table_schema = 'mydb'
ORDER BY stat_value DESC
LIMIT 20;
```

### 4. 메모리 분석

```sql
-- 현재 메모리 사용 상위 항목
SELECT
    event_name,
    current_alloc,
    high_alloc
FROM sys.memory_global_by_current_bytes
LIMIT 20;

-- 주요 이벤트:
-- memory/innodb/buf_pool_page_hash: Buffer Pool 페이지 해시 → 가장 큰 메모리
-- memory/performance_schema/*: Performance Schema 메모리
-- memory/sql/Relay_log_info::cache: Relay Log 캐시

-- 연결별 메모리 사용
SELECT
    host,
    current_allocated,
    total_allocated
FROM sys.memory_by_host_by_current_bytes
ORDER BY current_allocated DESC
LIMIT 10;
-- 특정 호스트가 메모리를 많이 사용 중이면 해당 애플리케이션 쿼리 확인
```

### 5. I/O 분석

```sql
-- 파일별 I/O (가장 많이 읽힌 파일)
SELECT
    file,
    count_read,
    total_read,
    count_write,
    total_written,
    total
FROM sys.io_global_by_file_by_bytes
ORDER BY total DESC
LIMIT 10;

-- 해석:
-- .ibd 파일이 상위: 해당 테이블에 I/O 집중
-- ib_logfile: Redo Log I/O 많음 → innodb_log_file_size 증가 검토
-- binlog: Binary Log I/O → sync_binlog 설정 검토

-- 대기 유형별 I/O
SELECT
    event_name,
    total_read,
    total_written,
    total_requested
FROM sys.io_global_by_wait_by_bytes
ORDER BY total_requested DESC
LIMIT 10;
```

---

## 💻 실전 실험

### 실험 1: 즉각 진단 — 서비스 느릴 때

```sql
-- STEP 1: Full Scan 쿼리 확인 (30초 내 진단)
SELECT db, query, exec_count, total_latency, no_index_used_count
FROM sys.statements_with_full_table_scans
WHERE db = 'mydb'
ORDER BY total_latency DESC
LIMIT 5;

-- STEP 2: 현재 Lock 대기 확인
SELECT wait_age, waiting_query, blocking_query
FROM sys.innodb_lock_waits;

-- STEP 3: 임시 테이블 사용 쿼리 확인
SELECT query, disk_tmp_tables, total_latency
FROM sys.statements_with_temp_tables
WHERE db = 'mydb' AND disk_tmp_tables > 0
ORDER BY total_latency DESC LIMIT 5;

-- STEP 4: 가장 느린 쿼리 TOP5
SELECT query, exec_count, avg_latency, total_latency
FROM sys.statement_analysis
WHERE db = 'mydb'
ORDER BY total_latency DESC
LIMIT 5;
```

### 실험 2: 인덱스 정리

```sql
-- 미사용 인덱스 목록 추출
SELECT
    CONCAT('DROP INDEX ', index_name, ' ON ', object_schema, '.', object_name, ';') AS drop_sql
FROM sys.schema_unused_indexes
WHERE object_schema = 'mydb'
  AND index_name NOT LIKE 'PRIMARY'  -- PK 제외
ORDER BY object_name, index_name;

-- 출력된 DROP INDEX SQL을 바로 실행하지 말고:
-- 1. Invisible Index로 먼저 비활성화
-- 2. 1~2주간 모니터링
-- 3. 문제 없으면 DROP

-- 중복 인덱스 확인
SELECT
    CONCAT('-- Redundant: ', redundant_index_name,
           ', Dominant: ', dominant_index_name) AS info,
    CONCAT('DROP INDEX ', redundant_index_name, ' ON mydb.', table_name, ';') AS drop_sql
FROM sys.schema_redundant_indexes
WHERE table_schema = 'mydb';
```

---

## 📊 성능/비용 비교

```
sys 스키마 vs 직접 쿼리 생산성 비교:

Full Scan 쿼리 찾기:
  직접 쿼리: 10~15줄, 단위 변환 직접
  sys 스키마: SELECT * FROM sys.statements_with_full_table_scans
  생산성: sys가 10배 빠름

Lock 대기 분석:
  직접 쿼리: information_schema.INNODB_LOCKS + INNODB_LOCK_WAITS 조인 (복잡)
  sys 스키마: SELECT * FROM sys.innodb_lock_waits
  생산성: sys가 5~10배 빠름

오버헤드:
  sys 스키마 자체: 뷰이므로 추가 저장 없음 (쿼리 시에만 비용)
  Performance Schema 의존: 기본 활성화 필요
  조회 비용: 복잡한 뷰는 조금 더 느릴 수 있지만 진단 쿼리 수준에서 무시
```

---

## ⚖️ 트레이드오프

```
sys 스키마 활용 트레이드오프:

장점:
  ✅ 운영자 친화적 (단순한 쿼리)
  ✅ 단위 자동 변환 (피코초 → ms)
  ✅ 진단 속도 향상 (즉각 결과)
  ✅ 추가 설치 불필요 (MySQL 5.7.7+)

단점:
  ❌ Performance Schema 활성화 필수
  ❌ 일부 뷰는 쿼리 비용 높음 (복잡한 조인)
  ❌ 커스터마이징 제한 (뷰 수정 권장하지 않음)
  ❌ 모든 정보를 sys로 커버 불가 (SHOW ENGINE INNODB STATUS 등)

적합한 용도:
  일상적인 운영 모니터링 → sys 스키마
  정밀한 성능 분석 → Performance Schema 직접 쿼리
  InnoDB 내부 상태 → SHOW ENGINE INNODB STATUS
  조합: sys로 빠른 진단 → 필요 시 다른 방법으로 심화 분석
```

---

## 📌 핵심 정리

```
sys 스키마 핵심:

쿼리 성능 문제:
  Full Scan → sys.statements_with_full_table_scans
  임시 테이블 → sys.statements_with_temp_tables
  정렬 → sys.statements_with_sorting
  종합 분석 → sys.statement_analysis

인덱스 정리:
  미사용 → sys.schema_unused_indexes
  중복 → sys.schema_redundant_indexes
  → Invisible Index로 검증 후 DROP (안전)

Lock 대기:
  InnoDB 락 → sys.innodb_lock_waits
  테이블 락 → sys.schema_table_lock_waits

즉각 진단 순서:
  1. sys.innodb_lock_waits (현재 Lock)
  2. sys.statements_with_full_table_scans (Full Scan)
  3. sys.statement_analysis (전체 쿼리 통계)
  4. sys.statements_with_temp_tables (디스크 임시 테이블)
```

---

## 🤔 생각해볼 문제

**Q1.** `sys.schema_unused_indexes`에서 반환된 인덱스를 즉시 삭제해도 안전한가? 어떤 추가 검증이 필요한가?

<details>
<summary>해설 보기</summary>

**즉시 삭제는 안전하지 않습니다.** `sys.schema_unused_indexes`는 서버 재시작 이후 Performance Schema에 수집된 통계만 반영합니다. 다음 상황에서 실제로 사용 중인 인덱스도 "미사용"으로 나올 수 있습니다.

- 서버 최근 재시작 후 아직 해당 인덱스를 사용하는 쿼리가 실행 안 됨
- 특정 기간에만 실행되는 배치 쿼리 (월말 정산 등)
- 통계 수집이 비활성화된 instrument로 인해 집계 누락

**안전한 절차**:
```sql
-- 1. 충분한 기간 통계 수집 (최소 2~4주, 배치 포함)
-- 2. 미사용 인덱스 목록 추출
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = 'mydb';

-- 3. Invisible Index로 비활성화 (MySQL 8.0)
ALTER TABLE orders ALTER INDEX idx_old INVISIBLE;

-- 4. 1~2주 모니터링 (Optimizer가 인덱스 건너뜀)
-- EXPLAIN으로 영향받는 쿼리 없는지 확인

-- 5. 문제 없으면 최종 삭제
DROP INDEX idx_old ON orders;

-- 문제 발생 시 즉각 복구:
ALTER TABLE orders ALTER INDEX idx_old VISIBLE;
```

</details>

---

**Q2.** `sys.innodb_lock_waits`에서 blocking_query가 SELECT 쿼리인데 왜 락을 보유하는가?

<details>
<summary>해설 보기</summary>

일반 SELECT는 락을 보유하지 않지만, 다음 경우에는 SELECT도 락을 보유합니다.

**1. SELECT FOR UPDATE / LOCK IN SHARE MODE**:
```sql
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- X Lock 보유, 다른 SELECT FOR UPDATE 차단
```

**2. 트랜잭션이 열린 상태로 SELECT**:
```sql
BEGIN;
SELECT * FROM accounts WHERE user_id = 1;  -- S Lock 보유
-- 이 트랜잭션이 길게 열려있으면 DDL을 차단할 수 있음 (MDL)
```

**3. SERIALIZABLE 격리 수준**:
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT * FROM orders WHERE status = 'PAID';  -- S Lock 자동 획득
-- 다른 트랜잭션의 INSERT/UPDATE가 대기 상태
```

`sys.innodb_lock_waits`에서 blocking_query가 SELECT이고 blocking_trx_age가 길다면, `KILL [blocking_pid]`로 강제 종료하거나 해당 트랜잭션이 COMMIT/ROLLBACK되도록 조치합니다.

</details>

---

<div align="center">

**[⬅️ Performance Schema](./01-performance-schema-guide.md)** | **[홈으로 🏠](../README.md)** | **[다음: InnoDB 상태 분석 ➡️](./03-innodb-status-analysis.md)**

</div>
