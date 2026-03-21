# InnoDB 상태 분석 — SHOW ENGINE INNODB STATUS 완전 해석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `TRANSACTIONS` 섹션에서 Lock 대기 중인 트랜잭션을 찾는 방법은?
- `BUFFER POOL AND MEMORY`에서 Buffer Pool 히트율과 오염된 Page 수를 읽는 방법은?
- `LATEST DETECTED DEADLOCK` 섹션에서 Lock 충돌을 분석하는 방법은?
- `ROW OPERATIONS`에서 현재 DB 부하를 파악하는 방법은?
- 각 섹션을 어떤 순서로 읽어야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### SHOW ENGINE INNODB STATUS = InnoDB의 현재 상태 스냅샷

```
언제 필요한가:
  ① "DB가 갑자기 느려졌다" → Buffer Pool, Row Operations 확인
  ② "데드락이 발생했다" → LATEST DETECTED DEADLOCK 분석
  ③ "Lock 때문에 쿼리가 안 된다" → TRANSACTIONS 확인
  ④ "디스크 I/O가 높다" → LOG, BUFFER POOL 확인

주기적으로 모니터링:
  innodb_status_output = ON → 에러 로그에 주기적으로 기록
  60초마다 자동 출력 (innodb_status_output_locks 포함)
  → 문제 발생 전후 상태 비교 가능
```

---

## 😱 흔한 실수 (Before)

### 1. 출력 전체를 읽으려다 포기

```
Before:
  SHOW ENGINE INNODB STATUS\G 실행
  → 수백 줄의 출력
  → 어디서부터 읽어야 할지 몰라 포기

올바른 접근:
  문제 유형에 따라 보는 섹션이 다름
  느린 응답 → BUFFER POOL + ROW OPERATIONS
  Lock 문제 → TRANSACTIONS
  데드락 → LATEST DETECTED DEADLOCK
  섹션별로 읽는 법을 알면 30초 내 진단 가능
```

### 2. 스냅샷 시점 이해 부족

```sql
-- Before: 현재 상태가 아닌 과거 스냅샷이라고 오해
SHOW ENGINE INNODB STATUS\G

-- 출력 상단:
-- Per second averages calculated from the last 37 seconds
-- 37초 동안의 평균값이 "per second" 수치

-- 스냅샷 시점:
-- SHOW ENGINE INNODB STATUS 실행 순간의 스냅샷
-- "Per second" 수치는 마지막 내부 측정 이후 경과 시간 기준
-- → 연속으로 실행하면 시간이 지날수록 평균이 변함
```

### 3. LATEST DETECTED DEADLOCK을 현재 데드락으로 오해

```sql
-- Before: LATEST DETECTED DEADLOCK = 현재 진행 중 데드락
-- 실제: 마지막으로 발생한 데드락 (과거 이벤트)
-- 오래전 데드락이 여기 남아 있을 수 있음

-- 현재 진행 중인 Lock 대기:
SELECT * FROM sys.innodb_lock_waits;
-- 또는
SELECT * FROM performance_schema.data_lock_waits;
```

---

## ✨ 올바른 접근 (After)

```
SHOW ENGINE INNODB STATUS 섹션별 읽기 순서:

문제 유형별 우선 확인 섹션:

DB 전반적 느림:
  1. BUFFER POOL AND MEMORY → 히트율 확인
  2. ROW OPERATIONS → 초당 읽기/쓰기 량
  3. LOG → Redo Log 쓰기 속도

Lock 대기 문제:
  1. TRANSACTIONS → waiting for lock 쿼리 찾기
  2. LATEST DETECTED DEADLOCK → 데드락 패턴

데드락 발생:
  1. LATEST DETECTED DEADLOCK
     → 어떤 두 트랜잭션, 어떤 Lock이 충돌했는지

I/O 높음:
  1. BUFFER POOL AND MEMORY → dirty pages, read/write
  2. LOG → log I/O speed
  3. I/O THREAD → pending I/O
```

---

## 🔬 내부 동작 원리

### 1. TRANSACTIONS 섹션 읽기

```
SHOW ENGINE INNODB STATUS\G 출력:

---TRANSACTION 1234567, ACTIVE 120 sec
2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 5
MySQL thread id 99, OS thread handle 140..., query id 10000 10.0.0.1 app
UPDATE orders SET status='SHIPPED' WHERE id=12345

---TRANSACTION 1234568, ACTIVE 5 sec starting index read
MySQL thread id 101, ...
------- TRX HAS BEEN WAITING 5 sec FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 123 page no 45 n bits 72 index PRIMARY of table `mydb`.`orders`
trx id 1234568 lock_mode X locks rec but not gap waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...

해석:
  TRANSACTION 1234567: id=12345 Row에 X Lock 보유 (120초 째)
  TRANSACTION 1234568: 같은 Row에 X Lock 대기 (5초 째)

행동 지침:
  1234567이 120초 동안 COMMIT/ROLLBACK 안 함
  → 장기 미완료 트랜잭션
  → 원인: 애플리케이션이 트랜잭션을 닫지 않음
  → KILL 99 (MySQL thread id)
```

### 2. BUFFER POOL AND MEMORY 섹션

```
BUFFER POOL AND MEMORY 출력:
  Total large memory allocated 137428992
  Dictionary memory allocated 1048576
  Buffer pool size   8192
  Free buffers       5120
  Database pages     3072
  Old database pages 1131
  Modified db pages  512
  Pending reads      0
  Pending writes: LRU 0, flush list 0, single page 0
  Pages made young 12345, not young 67890
  0.00 youngs/s, 0.00 non-youngs/s
  Pages read 100000, created 2000, written 150000
  0.00 reads/s, 0.00 creates/s, 0.00 writes/s
  Buffer pool hit rate 999 / 1000, young-making rate 0 / 1000

해석:
  Buffer pool size: 8192 pages (8192 × 16KB = 128MB)
  Free buffers: 5120 → 여유 있음
  Database pages: 3072 → 현재 사용 중인 페이지
  Modified db pages: 512 → Dirty Pages (아직 디스크에 안 씀)

Buffer pool hit rate: 999/1000 = 99.9%
  → 1000번 읽기 중 999번은 Buffer Pool에서 (Good!)
  < 990/1000 = 99%: 경고 (디스크 I/O 증가)
  < 950/1000 = 95%: 심각 (innodb_buffer_pool_size 증가 필요)

Dirty Pages (Modified db pages) 비율:
  Dirty / Database pages = 512 / 3072 = 16.7%
  > 75%: Checkpointing 병목 (쓰기 I/O 급증 가능)
```

### 3. LATEST DETECTED DEADLOCK 섹션

```
LATEST DETECTED DEADLOCK 출력:
  *** (1) TRANSACTION:
  TRANSACTION 1234567, ACTIVE 2 sec
  2 lock struct(s), heap size 1136, 2 row lock(s)
  MySQL thread id 50, ...
  *** (1) HOLDS THE LOCK(S):
  RECORD LOCKS ... index idx_user of table `mydb`.`orders`
  lock_mode X locks rec but not gap
  *** (1) WAITING FOR THIS LOCK TO BE GRANTED:
  RECORD LOCKS ... index PRIMARY of table `mydb`.`orders`
  lock_mode X locks rec but not gap waiting

  *** (2) TRANSACTION:
  TRANSACTION 1234568, ACTIVE 2 sec
  3 lock struct(s), heap size 1136, 2 row lock(s)
  MySQL thread id 51, ...
  *** (2) HOLDS THE LOCK(S):
  RECORD LOCKS ... index PRIMARY of table `mydb`.`orders`
  *** (2) WAITING FOR THIS LOCK TO BE GRANTED:
  RECORD LOCKS ... index idx_user of table `mydb`.`orders`
  *** WE ROLL BACK TRANSACTION (1)

분석:
  TX1: idx_user Lock 보유 → PRIMARY Lock 대기
  TX2: PRIMARY Lock 보유 → idx_user Lock 대기
  → 순환 대기 = 데드락
  → MySQL이 TX1을 롤백 (더 적은 undo rows 기준)

데드락 패턴:
  위 패턴: 두 트랜잭션이 서로 다른 인덱스를 반대 순서로 접근
  원인: 같은 Row를 두 개의 인덱스 경로로 접근
  해결: 항상 같은 순서로 리소스에 접근하도록 애플리케이션 수정
```

### 4. ROW OPERATIONS 섹션

```
ROW OPERATIONS 출력:
  0 queries inside InnoDB, 0 queries in queue
  0 read views open inside InnoDB
  Process ID=1234, Main thread id=..., state: sleeping
  Number of rows inserted 12345, updated 6789, deleted 1000, read 9876543
  0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 1000.00 reads/s

해석:
  queries inside InnoDB: 현재 InnoDB에서 처리 중인 쿼리 수
  queries in queue: InnoDB 동시 처리 한계로 큐에 쌓인 쿼리
    → queue > 0: innodb_thread_concurrency 설정 검토

  reads/s: 초당 Row 읽기 수 (1000/s)
    높으면: Full Scan 또는 대용량 JOIN
    갑자기 높아지면: 새 쿼리나 배치 작업 확인

  read views open: 열린 트랜잭션 수 (MVCC 스냅샷)
    많으면: 장기 미완료 트랜잭션 → Undo Log 누적
```

### 5. LOG 섹션

```
LOG 출력:
  Log sequence number 123456789
  Log buffer assigned up to 123456789
  Log buffer completed up to 123456789
  Log written up to 123456789
  Log flushed up to 123456700
  Last checkpoint at 123456600

  LSN(Log Sequence Number) 흐름:
    sequence → assigned → completed → written → flushed → checkpoint

  flushed vs checkpoint 차이:
    flushed: 디스크(Redo Log 파일)에 기록됨
    checkpoint: 데이터 파일(ibdata, .ibd)까지 반영됨

  Log written - last checkpoint = 미반영 데이터
    = Crash 시 복구해야 할 Redo Log 양
    너무 크면: Checkpoint가 느림 → innodb_log_file_size 검토

  I/O 관련:
    0.00 log i/o's per second: 정상
    높으면: Redo Log 쓰기 병목 → innodb_flush_log_at_trx_commit 검토
```

---

## 💻 실전 실험

### 실험 1: 장기 트랜잭션 찾기

```sql
-- SHOW ENGINE INNODB STATUS의 TRANSACTIONS 섹션 파싱
-- 직접 조회:
SELECT
    t.trx_id,
    t.trx_started,
    TIMESTAMPDIFF(SECOND, t.trx_started, NOW()) AS duration_sec,
    t.trx_rows_locked,
    t.trx_rows_modified,
    t.trx_state,
    LEFT(t.trx_query, 200) AS current_query,
    p.host,
    p.user
FROM information_schema.INNODB_TRX t
LEFT JOIN information_schema.PROCESSLIST p ON p.id = t.trx_mysql_thread_id
ORDER BY t.trx_started
LIMIT 10;
-- duration_sec > 60: 1분 이상 미완료 트랜잭션 → 즉시 조사
```

### 실험 2: Buffer Pool 히트율 모니터링

```sql
-- Buffer Pool 핵심 지표
SELECT
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN (
    'Innodb_buffer_pool_reads',        -- 디스크에서 읽은 횟수
    'Innodb_buffer_pool_read_requests', -- 전체 읽기 요청
    'Innodb_buffer_pool_dirty_pages',  -- Dirty pages
    'Innodb_buffer_pool_pages_total',  -- 전체 페이지
    'Innodb_buffer_pool_pages_free'    -- 여유 페이지
);

-- 히트율 계산
SELECT
    reads.VARIABLE_VALUE AS disk_reads,
    requests.VARIABLE_VALUE AS total_requests,
    ROUND(100 * (1 - reads.VARIABLE_VALUE / requests.VARIABLE_VALUE), 2) AS hit_rate_pct
FROM
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') reads,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') requests;
-- 히트율 < 99%: innodb_buffer_pool_size 증가 검토
```

### 실험 3: 데드락 시뮬레이션 및 분석

```sql
-- 두 세션에서 데드락 유발 (테스트 환경)
-- 세션 1:
START TRANSACTION;
UPDATE orders SET status='A' WHERE id=1;
-- 잠시 대기 후:
UPDATE orders SET status='A' WHERE id=2;

-- 세션 2 (동시):
START TRANSACTION;
UPDATE orders SET status='B' WHERE id=2;
-- 세션 1이 id=1 Lock 획득 후:
UPDATE orders SET status='B' WHERE id=1;  -- 데드락!

-- 데드락 후 분석:
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 섹션 확인
```

---

## 📊 성능/비용 비교

```
SHOW ENGINE INNODB STATUS 주요 지표 임계값:

Buffer Pool 히트율:
  > 99%: 정상
  95~99%: 주의 (innodb_buffer_pool_size 증가 고려)
  < 95%: 즉각 대응 (innodb_buffer_pool_size 증가)

Dirty Pages 비율 (Modified / Total):
  < 25%: 정상
  25~75%: 주의 (Checkpoint 속도 확인)
  > 75%: 즉각 대응 (innodb_io_capacity 증가)

장기 트랜잭션:
  < 30초: 허용
  30~120초: 주의 (확인 필요)
  > 120초: 즉각 종료 (KILL thread_id)

Queries in queue:
  0: 정상
  > 0: innodb_thread_concurrency 설정 검토
```

---

## ⚖️ 트레이드오프

```
innodb_status_output 설정:

ON (에러 로그에 주기 출력):
  ✅ 문제 발생 시 과거 상태 비교 가능
  ✅ 주기적 모니터링 자동화
  ❌ 에러 로그 크기 빠르게 증가
  ❌ 일부 민감한 정보 로그에 기록 (innodb_status_output_locks)

OFF (필요 시만 수동 실행):
  ✅ 로그 크기 절약
  ❌ 문제 발생 시 과거 기록 없음

권장:
  innodb_status_output = ON (에러 로그 로테이션 설정)
  innodb_status_output_locks = ON (Lock 정보 포함, 디버깅용)
  문제 해결 후 OFF 전환 가능

SHOW ENGINE INNODB STATUS 실행 빈도:
  문제 발생 시: 30초~1분 간격으로 연속 실행 (변화 추세 파악)
  정기 모니터링: 5~10분 간격으로 핵심 수치 기록
```

---

## 📌 핵심 정리

```
SHOW ENGINE INNODB STATUS 핵심:

문제 유형별 확인 섹션:
  Lock 대기 → TRANSACTIONS
  데드락 → LATEST DETECTED DEADLOCK
  느린 응답 → BUFFER POOL + ROW OPERATIONS
  I/O 높음 → LOG + BUFFER POOL

핵심 지표:
  Buffer pool hit rate: > 999/1000 (99.9%) 목표
  Dirty pages: < 25% of total pages
  Queries in queue: 0이어야 정상
  Long transactions: duration_sec > 60 즉시 조사

데드락 분석:
  HOLDS THE LOCK → WAITING FOR LOCK 관계 파악
  두 트랜잭션이 서로 상대방의 Lock 대기 = 순환 대기
  해결: 같은 순서로 리소스 접근, SELECT FOR UPDATE 최소화

보완 도구:
  sys.innodb_lock_waits (현재 Lock)
  information_schema.INNODB_TRX (현재 트랜잭션)
  performance_schema.data_lock_waits (상세 Lock)
```

---

## 🤔 생각해볼 문제

**Q1.** TRANSACTIONS 섹션에서 다음 출력을 발견했다. 어떻게 대응하는가?

```
---TRANSACTION 12345, ACTIVE 450 sec
MySQL thread id 88, query id 50000 10.0.0.5 app_user
---TRANSACTION 12346, ACTIVE 3 sec starting index read
------- TRX HAS BEEN WAITING 3 sec FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS ... lock_mode X ... waiting
```

<details>
<summary>해설 보기</summary>

**상황 분석**:
- TX 12345: 450초(7.5분) 동안 COMMIT/ROLLBACK 없이 Lock 보유
- TX 12346: TX 12345가 보유한 Lock을 3초째 대기 중

**대응 절차**:

```sql
-- 1. TX 12345 원인 파악
SELECT t.trx_id, t.trx_started, p.host, p.user, p.state,
       LEFT(p.info, 200) AS last_query
FROM information_schema.INNODB_TRX t
JOIN information_schema.PROCESSLIST p ON p.id = t.trx_mysql_thread_id
WHERE t.trx_id = 12345;
-- 원인: 애플리케이션이 트랜잭션을 닫지 않음 (커넥션 풀 반환 후 COMMIT 누락 등)

-- 2. 즉각 종료 (서비스 영향 최소화)
KILL 88;  -- MySQL thread id
-- TX 12345가 롤백되고 TX 12346이 진행 가능

-- 3. 재발 방지
-- 애플리케이션에서 @Transactional 또는 try-finally에서 commit/rollback 보장
-- innodb_lock_wait_timeout = 30 (초) → 30초 대기 후 자동 롤백 (기본 50초)
-- max_execution_time 설정으로 장기 쿼리 자동 차단
```

</details>

---

**Q2.** Buffer pool hit rate가 970/1000 (97%)이다. 어떤 조치를 순서대로 취하는가?

<details>
<summary>해설 보기</summary>

97%는 경고 수준입니다. 전체 읽기의 3%가 디스크에서 읽히고 있습니다.

**조치 순서**:

1. **Full Scan 쿼리 확인 및 최적화 (먼저)**
```sql
SELECT * FROM sys.statements_with_full_table_scans
WHERE db = 'mydb' ORDER BY total_latency DESC LIMIT 5;
-- Full Scan이 Buffer Pool을 오염시키는 주범
-- 인덱스 추가로 먼저 해결 시도
```

2. **현재 Buffer Pool 크기 vs 실제 필요 크기 확인**
```sql
SELECT
    VARIABLE_VALUE AS bp_size_bytes,
    ROUND(VARIABLE_VALUE / 1024 / 1024 / 1024, 2) AS bp_size_gb
FROM performance_schema.global_variables
WHERE VARIABLE_NAME = 'innodb_buffer_pool_size';
-- 서버 RAM 대비 비율 확인 (현재 RAM의 몇 %인가)
```

3. **innodb_buffer_pool_size 증가 (퀴리 최적화로 부족할 때)**
```sql
-- MySQL 8.0: 온라인 변경 가능
SET GLOBAL innodb_buffer_pool_size = 4 * 1024 * 1024 * 1024;  -- 4GB
-- 권장: 서버 RAM의 50~70%
```

4. **장기적: innodb_buffer_pool_instances 검토**
```
-- 병렬 처리 향상 (Buffer Pool 분할)
-- innodb_buffer_pool_size >= 1GB: instances = 8 (권장)
```

Full Scan 최적화로도 히트율이 개선되는 경우가 많으므로, 인프라 증설 전 쿼리 최적화를 먼저 시도합니다.

</details>

---

<div align="center">

**[⬅️ sys 스키마](./02-sys-schema-usage.md)** | **[홈으로 🏠](../README.md)** | **[다음: MySQL 8.0 새 기능 ➡️](./04-mysql8-new-features.md)**

</div>
