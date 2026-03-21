# Performance Schema 완전 가이드 — 어디서 얼마나 걸리는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `events_statements_summary_by_digest`로 가장 느린/자주 실행된 쿼리를 추출하는 방법은?
- `events_waits_summary`로 Lock 대기, IO 대기, 뮤텍스 병목을 찾는 쿼리는?
- `setup_instruments`와 `setup_consumers` 설정으로 수집 범위를 조정하는 방법은?
- Performance Schema의 오버헤드를 최소화하면서 활용하는 방법은?
- 특정 쿼리의 실행 계획과 실행 횟수를 함께 분석하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### "어떤 쿼리가 서비스를 느리게 만드는가" — 데이터 기반 진단

```
전통적 접근 (비효율):
  개발자: "이 쿼리가 느릴 것 같아요" (추측)
  → EXPLAIN 분석 → 인덱스 추가
  → 실제 문제 쿼리가 아닐 수 있음

Performance Schema 기반 접근:
  1. events_statements_summary_by_digest 조회
     → 총 실행 시간 = 실행 횟수 × 평균 실행 시간
     → TOP 10 쿼리 식별 (데이터 기반)
  2. 식별된 쿼리의 EXPLAIN ANALYZE 분석
  3. 인덱스 추가 후 성능 개선 측정

차이:
  추측 기반: 5번 시도 → 1번 성공
  데이터 기반: 정확한 병목 식별 → 즉각 개선
```

---

## 😱 흔한 실수 (Before)

### 1. Performance Schema가 비활성화된 채로 운영

```sql
-- 확인
SHOW VARIABLES LIKE 'performance_schema';
-- performance_schema: OFF → 아무것도 수집 안 됨!

-- 원인: my.cnf에 performance_schema=OFF 설정 또는 기본값이 OFF인 구버전
-- MySQL 5.7.9+, 8.0: 기본 ON

-- 활성화 (재시작 필요):
-- my.cnf: performance_schema=ON

-- 부분 활성화 상태 (instrument 비활성):
SELECT NAME, ENABLED FROM performance_schema.setup_instruments
WHERE NAME LIKE 'statement/%' LIMIT 5;
-- ENABLED: NO → 쿼리 통계 수집 안 됨
```

### 2. summary 테이블을 보지 않고 raw 이벤트 테이블만 봄

```sql
-- Before: 현재 실행 중인 이벤트만 확인 (과거 누적 없음)
SELECT * FROM performance_schema.events_statements_current;
-- 현재 실행 중인 쿼리만 있음 → 과거 통계 없음

-- 올바른 방법: 누적 통계 테이블 사용
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
-- 재시작 이후 모든 쿼리의 누적 통계
```

### 3. 오버헤드 걱정으로 모든 instrument 비활성화

```sql
-- Before: 성능 영향 걱정으로 모든 것 끔
UPDATE performance_schema.setup_instruments SET ENABLED='NO';
-- 수집 없음 → 진단 불가

-- 올바른 접근: 필요한 instrument만 선택적 활성화
-- 기본 활성화된 것으로 충분한 경우가 대부분
-- 추가 활성화는 진단 목적으로 임시 사용
```

---

## ✨ 올바른 접근 (After)

```
Performance Schema 활용 체계:

Layer 1: 항상 활성화 (기본값, 낮은 오버헤드)
  events_statements_summary_by_digest → 쿼리 통계
  events_waits_summary_global_by_event_name → 대기 통계
  table_io_waits_summary_by_table → 테이블별 I/O

Layer 2: 문제 진단 시 임시 활성화
  events_statements_history_long → 최근 1000개 쿼리 상세
  events_waits_history_long → 대기 이벤트 상세

Layer 3: 상세 프로파일링 (진단 후 비활성화)
  stage/sql/* instruments → 쿼리 실행 단계별 시간

진단 워크플로:
  1. summary 테이블로 TOP 쿼리 식별 (항상 가능)
  2. 해당 쿼리의 EXPLAIN ANALYZE로 실행계획 분석
  3. 필요 시 events_waits_history_long으로 대기 원인 분석
  4. 개선 후 summary 테이블 리셋 후 재측정
```

---

## 🔬 내부 동작 원리

### 1. Performance Schema 아키텍처

```
수집 흐름:

SQL 실행
  → Instrument (측정 포인트): 각 이벤트 발생 시 측정
  → Consumer (저장소): 측정된 데이터를 테이블에 저장

Instrument 종류:
  statement/sql/*     : SQL 문 실행 (SELECT, INSERT 등)
  wait/io/file/*      : 파일 I/O 대기
  wait/lock/table/*   : 테이블 락 대기
  wait/synch/mutex/*  : 뮤텍스 대기
  memory/*            : 메모리 할당/해제
  stage/sql/*         : SQL 실행 단계 (Sorting, Copying to tmp table 등)

Consumer 종류:
  events_statements_summary_by_digest : 쿼리별 집계 (다이제스트 기반)
  events_statements_current           : 현재 실행 중인 쿼리
  events_statements_history           : 최근 N개 쿼리 (스레드별)
  events_statements_history_long      : 최근 N개 쿼리 (전체)
  events_waits_summary_*              : 대기 이벤트 집계

오버헤드:
  statement instrument: ~1~3% CPU 오버헤드 (기본 활성화, 허용 가능)
  mutex instrument: ~5~10% (기본 비활성화, 필요 시만)
  메모리 사용: 약 500MB ~ 1GB (테이블 크기 설정에 따라)
```

### 2. 가장 느린 쿼리 찾기

```sql
-- TOP 10 총 실행 시간 기준 쿼리
SELECT
    DIGEST_TEXT,
    COUNT_STAR AS exec_count,
    SUM_TIMER_WAIT / 1e12 AS total_sec,
    AVG_TIMER_WAIT / 1e12 AS avg_sec,
    MAX_TIMER_WAIT / 1e12 AS max_sec,
    SUM_ROWS_EXAMINED AS total_rows_examined,
    SUM_ROWS_SENT AS total_rows_sent,
    SUM_NO_INDEX_USED + SUM_NO_GOOD_INDEX_USED AS full_scan_count,
    FIRST_SEEN,
    LAST_SEEN
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- 해석:
-- total_sec: 이 쿼리가 전체 DB 시간 중 몇 초를 차지하는가
-- avg_sec: 평균 실행 시간
-- full_scan_count: 인덱스 없이 실행된 횟수 (0이 이상적)
-- SUM_ROWS_EXAMINED / SUM_ROWS_SENT: 비율이 높으면 인덱스 개선 필요

-- 실행 횟수 대비 최적화 우선순위:
-- 1위: SUM_TIMER_WAIT 높음 (총 DB 자원 소비 최대)
-- 2위: AVG_TIMER_WAIT 높음 + 자주 실행 (개선 효과 큼)
-- 3위: full_scan_count > 0 (인덱스 없는 쿼리)
```

### 3. 대기 이벤트로 병목 찾기

```sql
-- 전체 대기 이벤트 TOP 10 (재시작 이후 누적)
SELECT
    EVENT_NAME,
    COUNT_STAR AS event_count,
    SUM_TIMER_WAIT / 1e12 AS total_wait_sec,
    AVG_TIMER_WAIT / 1e12 AS avg_wait_sec
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- 주요 EVENT_NAME 해석:
-- wait/io/file/innodb/innodb_data_file : InnoDB 데이터 파일 I/O
-- wait/io/file/innodb/innodb_log_file  : Redo Log 파일 I/O
-- wait/lock/table/sql/handler          : 테이블 락 대기
-- wait/synch/mutex/innodb/buf_pool_mutex : Buffer Pool 뮤텍스 경합
-- idle                                  : 쿼리 없이 대기 (정상)

-- Buffer Pool I/O가 높으면: innodb_buffer_pool_size 증가 고려
-- Log 파일 I/O가 높으면: innodb_log_file_size, sync_binlog 검토
-- 테이블 락이 높으면: 쿼리 최적화 또는 MVCC 설정 검토
```

### 4. 테이블별 I/O 분석

```sql
-- 테이블별 I/O 통계
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_STAR AS total_io,
    COUNT_READ AS read_io,
    COUNT_WRITE AS write_io,
    SUM_TIMER_WAIT / 1e12 AS total_io_sec,
    SUM_TIMER_READ / 1e12 AS read_sec,
    SUM_TIMER_WRITE / 1e12 AS write_sec
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql','performance_schema','sys','information_schema')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- I/O가 높은 테이블: 인덱스 추가 또는 쿼리 최적화 대상
-- write_io >> read_io: 쓰기 집중 테이블 → 파티셔닝 또는 아카이브 검토
```

### 5. setup_instruments와 setup_consumers 설정

```sql
-- 현재 활성화된 instrument 확인
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
WHERE NAME LIKE 'statement/%'
LIMIT 10;

-- 특정 instrument 활성화 (진단 목적)
UPDATE performance_schema.setup_instruments
SET ENABLED='YES', TIMED='YES'
WHERE NAME LIKE 'stage/sql/%';

-- Consumer 활성화 (진단 목적)
UPDATE performance_schema.setup_consumers
SET ENABLED='YES'
WHERE NAME = 'events_statements_history_long';

-- 진단 후 원복
UPDATE performance_schema.setup_instruments
SET ENABLED='NO', TIMED='NO'
WHERE NAME LIKE 'stage/sql/%';

-- 통계 리셋 (기준점 설정)
TRUNCATE performance_schema.events_statements_summary_by_digest;
TRUNCATE performance_schema.events_waits_summary_global_by_event_name;
-- 개선 작업 후 리셋 → 이후 통계로 개선 효과 측정
```

---

## 💻 실전 실험

### 실험 1: 슬로우 쿼리 TOP 10 추출

```sql
-- 실전 진단 쿼리 (운영 환경 즉시 실행 가능)
SELECT
    SUBSTR(DIGEST_TEXT, 1, 100) AS query_sample,
    COUNT_STAR AS exec_count,
    ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
    ROUND(AVG_TIMER_WAIT / 1e12, 4) AS avg_sec,
    SUM_NO_INDEX_USED AS no_index_count,
    ROUND(SUM_ROWS_EXAMINED / NULLIF(COUNT_STAR, 0)) AS avg_rows_examined
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'mydb'  -- 특정 DB
  AND DIGEST_TEXT IS NOT NULL
  AND SUM_TIMER_WAIT > 1e12  -- 1초 이상 총 실행 시간
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

### 실험 2: 현재 실행 중인 쿼리 모니터링

```sql
-- 현재 1초 이상 실행 중인 쿼리 (실시간)
SELECT
    t.PROCESSLIST_ID,
    t.PROCESSLIST_USER,
    t.PROCESSLIST_HOST,
    t.PROCESSLIST_DB,
    t.PROCESSLIST_TIME AS running_sec,
    s.SQL_TEXT
FROM performance_schema.threads t
JOIN performance_schema.events_statements_current s
  ON s.THREAD_ID = t.THREAD_ID
WHERE t.PROCESSLIST_COMMAND = 'Query'
  AND t.PROCESSLIST_TIME > 1
ORDER BY t.PROCESSLIST_TIME DESC;
```

### 실험 3: 쿼리 실행 단계 프로파일링

```sql
-- stage instrument 임시 활성화
UPDATE performance_schema.setup_instruments
SET ENABLED='YES', TIMED='YES'
WHERE NAME LIKE 'stage/sql/%';

UPDATE performance_schema.setup_consumers
SET ENABLED='YES'
WHERE NAME LIKE '%stages%';

-- 느린 쿼리 실행 후
SELECT SQL_TEXT, EVENT_NAME, TIMER_WAIT/1e12 AS stage_sec
FROM performance_schema.events_stages_history_long
WHERE NESTING_EVENT_ID IN (
    SELECT EVENT_ID FROM performance_schema.events_statements_history_long
    WHERE SQL_TEXT LIKE '%slow_query%'
)
ORDER BY TIMER_START;
-- 출력: 어느 단계(Sorting, Copying to tmp table 등)에서 시간 소비하는지 확인

-- 원복
UPDATE performance_schema.setup_instruments
SET ENABLED='NO', TIMED='NO'
WHERE NAME LIKE 'stage/sql/%';
```

---

## 📊 성능/비용 비교

```
Performance Schema 오버헤드:

기본 설정 (권장):
  statement instruments: ~1~2% CPU
  memory 사용: ~200~500MB
  서비스 영향: 거의 없음 (운영 환경 사용 가능)

모든 instrument 활성화:
  CPU: ~10~20% 추가 (mutex 등 포함)
  memory 사용: ~1~2GB
  → 진단 목적으로만 짧게 사용

대안 도구 비교:
  Slow Query Log:
    오버헤드: 거의 없음
    정보: 쿼리 텍스트, 실행 시간
    단점: 실시간 집계 불가, 텍스트 파싱 필요

  Performance Schema:
    오버헤드: 낮음 (기본 설정)
    정보: 집계 통계, 대기 분류, 실시간
    장점: SQL로 즉시 분석, 다양한 차원

  sys 스키마:
    오버헤드: Performance Schema 활성화 필요
    정보: Performance Schema를 사용하기 쉬운 뷰로 제공
    장점: 단순한 쿼리로 진단 (다음 문서 참고)
```

---

## ⚖️ 트레이드오프

```
Performance Schema 설정 트레이드오프:

모든 것 활성화:
  ✅ 가장 상세한 진단 정보
  ❌ 높은 오버헤드 (10~20% CPU)
  ❌ 높은 메모리 사용
  → 단기 진단 목적으로만

기본 설정 (권장):
  ✅ 낮은 오버헤드 (~1~2%)
  ✅ 핵심 통계 수집
  ✅ 운영 환경 상시 사용 가능
  ❌ 세밀한 대기 분석 제한

모두 비활성화:
  ✅ 오버헤드 없음
  ❌ 진단 불가
  → 권장하지 않음

리셋 전략:
  정기적으로 summary 테이블 리셋 후 측정
  (서버 시작 이후 누적되면 과거 패턴에 묻힘)
  주기: 주 1회 또는 배포 후 측정

memory 테이블 크기 조정:
  performance_schema_events_statements_history_long_size = 1000 (기본)
  크게 설정할수록 더 오래된 이벤트 보존 (메모리 비용)
```

---

## 📌 핵심 정리

```
Performance Schema 핵심:

진단 목적별 주요 테이블:
  느린 쿼리 찾기:
    events_statements_summary_by_digest
    → ORDER BY SUM_TIMER_WAIT DESC
  대기 병목 찾기:
    events_waits_summary_global_by_event_name
    → ORDER BY SUM_TIMER_WAIT DESC
  테이블별 I/O:
    table_io_waits_summary_by_table
  현재 실행 중인 쿼리:
    threads + events_statements_current

설정 제어:
  setup_instruments: 수집 대상 설정
  setup_consumers: 저장 대상 설정
  기본 설정으로 충분한 경우가 대부분

측정 팁:
  TRUNCATE summary 테이블 → 기준점 설정
  개선 작업 후 재측정 → 개선 효과 수치화
  sys 스키마 뷰로 더 편리하게 조회 (다음 문서)
```

---

## 🤔 생각해볼 문제

**Q1.** `events_statements_summary_by_digest`에서 `SUM_NO_INDEX_USED`가 높은 쿼리를 찾았다. 이 쿼리에 인덱스를 추가한 후 효과를 측정하는 방법을 설명하라.

<details>
<summary>해설 보기</summary>

```sql
-- 1. 기준점 설정: 개선 전 통계 기록
SELECT DIGEST, DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT,
       SUM_NO_INDEX_USED, AVG_TIMER_WAIT
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 100
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 5;
-- 수치 기록

-- 2. 통계 리셋 (기준점 초기화)
TRUNCATE performance_schema.events_statements_summary_by_digest;

-- 3. 인덱스 추가
CREATE INDEX idx_new ON orders (status, created_at);

-- 4. 일정 시간 후 재측정
SELECT DIGEST, DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT,
       SUM_NO_INDEX_USED, AVG_TIMER_WAIT
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%orders%'
ORDER BY SUM_TIMER_WAIT DESC;

-- 비교:
-- SUM_NO_INDEX_USED: 높음 → 0 (인덱스 사용)
-- AVG_TIMER_WAIT: 기존 대비 감소 확인
-- SUM_TIMER_WAIT: 인덱스 추가 전의 동일 실행 횟수 대비 감소
```

</details>

---

**Q2.** `events_waits_summary_global_by_event_name`에서 `wait/io/file/innodb/innodb_data_file`이 가장 높다. 이것이 의미하는 것과 대응 방법을 설명하라.

<details>
<summary>해설 보기</summary>

**의미**: InnoDB 데이터 파일(`.ibd`)에 대한 I/O 대기가 많다는 것은 Buffer Pool이 부족하여 디스크 I/O가 빈번하게 발생한다는 신호입니다.

**원인 분석**:
```sql
-- Buffer Pool 히트율 확인
SELECT
    (1 - (reads / read_requests)) * 100 AS buffer_pool_hit_rate
FROM (
    SELECT
        SUM(VARIABLE_VALUE) AS reads
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) r,
(
    SELECT
        SUM(VARIABLE_VALUE) AS read_requests
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) rr;
-- 히트율 < 99% → Buffer Pool 부족
```

**대응 방법**:
1. `innodb_buffer_pool_size` 증가 (서버 RAM의 50~70%)
2. Full Scan 쿼리 최적화 (인덱스 추가) → Buffer Pool 오염 감소
3. 자주 접근하는 데이터 파악 → Hot Data를 Buffer Pool에 상주시키기
4. `innodb_read_ahead_threshold` 조정 (Read-Ahead 최적화)

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: sys 스키마 ➡️](./02-sys-schema-usage.md)**

</div>
