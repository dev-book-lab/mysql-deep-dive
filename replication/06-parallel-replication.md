# 병렬 복제(Parallel Replication) — Replica가 지연을 따라잡는 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 단일 SQL Thread가 Source의 병렬 트랜잭션을 직렬로 재생할 때 Lag이 누적되는 구조는?
- `replica_parallel_workers` 설정과 `LOGICAL_CLOCK` 방식으로 독립 트랜잭션을 병렬 재생하는 원리는?
- DATABASE 방식 vs LOGICAL_CLOCK 방식의 차이는?
- 병렬 복제 시 트랜잭션 순서는 어떻게 보장되는가?
- 병렬 복제로도 해결 안 되는 Lag 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 단일 SQL Thread = Source 병렬성을 영원히 따라갈 수 없다

```
구조적 문제:
  Source: 32개 스레드가 동시에 트랜잭션 처리 (병렬)
  SQL Thread: 1개 스레드로 Relay Log 순서대로 직렬 재생

  수학적으로:
    Source: 32 TPS × 32 스레드 = 1,024 TPS 처리
    단일 SQL Thread: 최대 50~100 TPS (디스크 I/O 병목)
    → 초당 900+ TPS씩 Relay Log 누적
    → 시간이 지날수록 Lag 폭발

실제 측정 예시:
  Source: 500 TPS
  단일 SQL Thread: 150 TPS
  → 초당 350 TPS씩 Relay Log 쌓임
  → 1시간 후 Lag: 350 × 3600 = 126만 트랜잭션 미처리

병렬 복제 적용:
  8 Workers LOGICAL_CLOCK:
  → 800~1,200 TPS 처리 가능 (Source 병렬성 근접)
  → Lag 따라잡기 가능
```

---

## 😱 흔한 실수 (Before)

### 1. DATABASE 방식으로 단일 스키마 환경에서 병렬 복제 적용

```sql
-- Before: DATABASE 방식 설정
SET GLOBAL replica_parallel_type = 'DATABASE';
SET GLOBAL replica_parallel_workers = 8;
STOP REPLICA; START REPLICA;

-- 기대: 8개 Worker가 병렬로 처리
-- 실제: 모든 트랜잭션이 동일 DB에 접근 → Worker 1에만 집중
--       7개 Worker는 유휴 → 단일 SQL Thread와 동일한 성능

-- DATABASE 방식은 여러 DB를 사용할 때만 효과
-- 단일 스키마(일반적) → LOGICAL_CLOCK 사용 필수
```

### 2. replica_preserve_commit_order 없이 병렬 복제

```sql
-- Before: 순서 보장 없이 병렬 복제
SET GLOBAL replica_parallel_workers = 8;
-- replica_preserve_commit_order = OFF (기본)

-- 문제:
-- Workers가 트랜잭션을 병렬 처리 후 완료 순서가 제각각
-- Worker 3이 Worker 1,2보다 먼저 완료 가능
-- Replica에서 쿼리 시 아직 커밋 안 된 앞 트랜잭션 데이터 못 봄
-- = 커밋 순서 불일치 → 데이터 불일치 가능

-- 반드시 함께 설정:
SET GLOBAL replica_preserve_commit_order = ON;
```

### 3. 대용량 단일 트랜잭션에 병렬 복제로 해결 시도

```sql
-- Before: "병렬 복제 켜면 대용량 UPDATE도 빨라지겠지"

-- 실제:
-- UPDATE orders SET status='DONE' WHERE year(created_at) = 2020;
-- → 300만 건, 단일 트랜잭션
-- → 이 트랜잭션은 하나의 Worker에 할당
-- → 나머지 7 Workers는 이 트랜잭션이 의존성이 있으면 대기
-- → 병렬 복제 효과 없음

-- 병렬 복제는 "독립적인 여러 트랜잭션"에서 효과
-- 하나의 큰 트랜잭션 = 병렬화 불가
-- → 청크 단위 DML 분리가 선행되어야 함
```

---

## ✨ 올바른 접근 (After)

```
병렬 복제 최적 설정:

LOGICAL_CLOCK 방식 (MySQL 5.7+, 권장):
  replica_parallel_type = LOGICAL_CLOCK
  replica_parallel_workers = CPU 코어 수 기준 (4~16)
  replica_preserve_commit_order = ON (커밋 순서 보장)

Source 측 그룹 커밋 최적화:
  binlog_group_commit_sync_delay = 100  (100ms 대기, 더 많이 묶기)
  → 한 번에 묶이는 트랜잭션 수 증가
  → 같은 last_committed → 더 많은 병렬화 가능

MySQL 8.0 추가 최적화:
  binlog_transaction_dependency_tracking = WRITESET
  → Row 수준 쓰기 집합 비교 → 더 정밀한 병렬화
  → LOGICAL_CLOCK보다 더 많은 트랜잭션을 병렬 처리 가능

병렬 복제 효과가 나지 않는 경우:
  → 단일 대용량 트랜잭션 → DML 청크 분리 선행
  → 핫스팟 테이블 (모든 트랜잭션이 같은 Row 접근) → 병렬화 한계
  → Workers 간 데드락 발생 → workers 수 줄이기 (8 → 4)
```

---

## 🔬 내부 동작 원리

### 1. 단일 SQL Thread의 구조적 한계

```
단일 SQL Thread 처리:
  Relay Log 이벤트 순서:
    Trx A (user_id=1 INSERT)
    Trx B (user_id=2 INSERT)
    Trx C (product_id=5 UPDATE)
    Trx D (user_id=1 UPDATE)
    Trx E (order_id=10 INSERT)

  단일 SQL Thread:
    Trx A 완료 → Trx B 완료 → Trx C 완료 → Trx D 완료 → Trx E 완료
    완전 직렬

  실제 독립성:
    Trx B, C, E는 서로 다른 데이터에 접근 → 독립 실행 가능
    하지만 단일 스레드는 이를 활용 못함

  병렬 복제:
    Worker 1: Trx A, Trx D (user_id=1 관련, 순서 보장)
    Worker 2: Trx B (user_id=2, 독립)
    Worker 3: Trx C (product_id=5, 독립)
    Worker 4: Trx E (order_id=10, 독립)
    → 실질적으로 4개 동시 처리
```

### 2. DATABASE 방식 (MySQL 5.6)

```sql
-- DATABASE 방식:
-- 서로 다른 DB에 접근하는 트랜잭션 = 병렬 가능
-- 같은 DB = 직렬

-- 예시:
-- Worker 1: db_user 스키마 트랜잭션
-- Worker 2: db_order 스키마 트랜잭션
-- Worker 3: db_product 스키마 트랜잭션

-- 한계:
-- 단일 스키마 사용 시 (일반적) → 모든 트랜잭션이 Worker 1
-- → 단일 SQL Thread와 동일

-- 멀티 테넌트 환경 (테넌트별 DB)에서는 효과적:
-- db_tenant1, db_tenant2, ... → 각각 독립 Worker

SET GLOBAL replica_parallel_type = 'DATABASE';
SET GLOBAL replica_parallel_workers = 4;
```

### 3. LOGICAL_CLOCK 방식 (MySQL 5.7+, 권장)

```
LOGICAL_CLOCK 원리:
  Binary Log에 각 트랜잭션마다 두 값이 기록됨:
  - sequence_number: 트랜잭션 순번 (단조증가)
  - last_committed: 이 트랜잭션 시작 시점의 마지막 커밋 번호

  예시:
  Trx A: sequence=1, last_committed=0
  Trx B: sequence=2, last_committed=0  ← A와 동시 실행 중 커밋
  Trx C: sequence=3, last_committed=0  ← A,B와 동시 실행 중 커밋
  Trx D: sequence=4, last_committed=3  ← A,B,C 완료 후 시작

  같은 last_committed (0) → 동시 커밋 → 충돌 없음 → 병렬 가능
  다른 last_committed (3) → D는 A,B,C 완료 후 시작 → 순서 보장 필요

Worker 배분:
  Worker 1: Trx A, B, C (last_committed=0, 동시 실행)
  → 세 Worker에 분배 또는 하나의 Worker가 순서대로
  → 실제: Workers가 각각 독립 트랜잭션을 가져감

그룹 커밋과 LOGICAL_CLOCK:
  binlog_group_commit_sync_delay로 더 많은 트랜잭션을 같은 그룹에 묶음
  → 같은 last_committed가 늘어남 → 더 많은 병렬화 가능
  → Source TPS가 높을수록 자연스럽게 많이 묶임
```

### 4. WRITESET 방식 (MySQL 8.0, 더 정밀)

```sql
-- WRITESET: Row 수준 쓰기 집합으로 의존성 분석
SET GLOBAL binlog_transaction_dependency_tracking = WRITESET;

-- 동작 원리:
-- Source에서 트랜잭션 실행 시
-- 변경된 각 Row의 PK/UK를 해시하여 기록
-- 두 트랜잭션의 WRITESET이 겹치지 않으면 → 독립 → 병렬 가능

-- 예시:
-- Trx A: UPDATE orders SET status='PAID' WHERE id=1 → WRITESET={orders:id=1}
-- Trx B: UPDATE orders SET status='PAID' WHERE id=2 → WRITESET={orders:id=2}
-- 겹침 없음 → 병렬 가능!

-- LOGICAL_CLOCK 한계 극복:
-- LOGICAL_CLOCK: 같은 그룹 커밋 기간에 커밋 안 됐으면 직렬
-- WRITESET: 그룹 커밋 기간과 무관하게 Row 겹침으로 판단
-- → 더 많은 트랜잭션을 병렬 처리 가능

-- 주의:
-- WRITESET은 비교적 최신 기능 → MySQL 8.0에서 안정화
-- UUID 컬럼 등 모든 Row에 별도 UK가 있으면 효과 극대화
```

### 5. replica_preserve_commit_order 역할

```sql
-- replica_preserve_commit_order = ON:
-- Workers가 병렬로 실행해도 커밋 순서는 Source와 동일하게 유지

-- 동작:
-- Worker 3이 Worker 1보다 먼저 완료해도
-- Worker 1이 완료될 때까지 커밋 대기
-- → Replica의 커밋 순서 = Source의 커밋 순서 보장

-- replica_preserve_commit_order = OFF:
-- Workers가 완료되는 순서대로 커밋
-- → Replica에서 쿼리 시 아직 커밋 안 된 앞 트랜잭션 데이터 못 보는 현상
-- → 일시적 데이터 불일치 (트랜잭션 간 읽기 일관성 저하)

-- 설정:
SET GLOBAL replica_preserve_commit_order = ON;  -- 기본값 (MySQL 8.0)
-- 성능 vs 일관성 트레이드오프:
-- ON: 약간의 오버헤드, 커밋 순서 보장
-- OFF: 더 빠른 처리, 일시적 불일치 허용
-- 일반적으로 ON 권장
```

---

## 💻 실전 실험

### 실험 1: 단일 SQL Thread vs 병렬 복제 TPS 비교

```sql
-- 테스트 준비 (Source에서 데이터 로드)
CREATE TABLE parallel_test (
    id   INT AUTO_INCREMENT PRIMARY KEY,
    grp  INT,
    val  VARCHAR(100)
) ENGINE=InnoDB;

-- 100만 건 삽입 (여러 그룹 데이터)
INSERT INTO parallel_test (grp, val)
SELECT MOD(seq, 100), REPEAT('x', 50)
FROM (SELECT @r := @r + 1 AS seq FROM information_schema.columns a, information_schema.columns b, (SELECT @r := 0) r LIMIT 1000000) n;

-- 단일 SQL Thread 설정
SET GLOBAL replica_parallel_workers = 0;
STOP REPLICA; START REPLICA;

-- Source에서 고부하 생성 (여러 세션에서 동시 실행)
-- Replica에서 Lag 측정

-- 병렬 복제 설정
SET GLOBAL replica_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL replica_parallel_workers = 8;
SET GLOBAL replica_preserve_commit_order = ON;
STOP REPLICA; START REPLICA;

-- 동일 부하에서 Lag 비교
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source 비교

DROP TABLE parallel_test;
```

### 실험 2: LOGICAL_CLOCK 동작 확인

```sql
-- Binary Log에서 last_committed, sequence_number 확인
-- (Source에서)
SHOW BINARY LOGS;
-- 최신 파일 확인

-- 최근 이벤트에서 GTID + last_committed 확인
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 50;
-- 또는
-- mysqlbinlog --base64-output=DECODE-ROWS -v mysql-bin.000001 | grep -E "last_committed|sequence_number"

-- 출력 예시:
-- #241015 10:30:01 server id 1  end_log_pos 234 CRC32 0x...
-- # last_committed=0  sequence_number=1  ...  (Trx A)
-- # last_committed=0  sequence_number=2  ...  (Trx B, 동시 커밋)
-- # last_committed=0  sequence_number=3  ...  (Trx C, 동시 커밋)
-- # last_committed=3  sequence_number=4  ...  (Trx D, 위 완료 후 시작)
-- last_committed=0인 것들 → 병렬 처리 가능
```

### 실험 3: 병렬 복제 Worker 상태 확인

```sql
-- Replica에서 Worker 상태 조회
SELECT
    WORKER_ID,
    SERVICE_STATE,
    LAST_APPLIED_TRANSACTION,
    LAST_APPLIED_TRANSACTION_END_APPLY_TIMESTAMP,
    APPLYING_TRANSACTION,
    LAST_ERROR_NUMBER,
    LAST_ERROR_MESSAGE
FROM performance_schema.replication_applier_status_by_worker;

-- 여러 Worker가 APPLYING_TRANSACTION을 동시에 가지면 병렬 동작 중
-- LAST_ERROR_MESSAGE가 있으면 Worker 오류 확인
```

---

## 📊 성능/비용 비교

```
병렬 복제 방식별 처리량 비교 (Source 500 TPS 기준):

단일 SQL Thread:
  처리량: 150~200 TPS
  Lag 누적: 초당 300~350 트랜잭션
  1시간 후 Lag: 100만+ 트랜잭션

DATABASE 방식 (단일 스키마):
  처리량: 150~200 TPS (단일과 동일)
  단일 스키마에서 Worker 1개 집중
  → 효과 없음

LOGICAL_CLOCK (4 Workers):
  처리량: 400~600 TPS
  Lag 누적: 초당 0~100 트랜잭션 (부하에 따라)
  효과: Source와 거의 동등

LOGICAL_CLOCK (8 Workers):
  처리량: 700~1,000 TPS
  Lag 누적: 거의 없음 (Source 따라잡기)
  Worker 간 오버헤드: 존재하지만 이득이 큼

WRITESET (MySQL 8.0, 8 Workers):
  처리량: 900~1,200 TPS
  LOGICAL_CLOCK보다 20~30% 추가 향상 가능
  핫스팟 없는 워크로드에서 최적
```

---

## ⚖️ 트레이드오프

```
병렬 복제 설정 트레이드오프:

Workers 수:
  많을수록: 처리량 증가 (4 → 8 → 16)
  너무 많으면: Worker 간 조율 오버헤드, 데드락 증가
  권장: CPU 코어 수 기준 (일반적으로 4~16)

replica_preserve_commit_order:
  ON (기본): 커밋 순서 보장, 약간의 오버헤드
  OFF: 더 빠른 처리, 일시적 읽기 불일치 허용
  일반적으로 ON 권장 (예측 가능한 동작)

LOGICAL_CLOCK vs WRITESET:
  LOGICAL_CLOCK: 간단, 대부분 충분
  WRITESET: 더 정밀하지만 설정 복잡 (MySQL 8.0)
  → 기존 Lag 개선 필요 시 LOGICAL_CLOCK 먼저 시도

Source 그룹 커밋 최적화:
  binlog_group_commit_sync_delay 증가:
    ✅ Replica 병렬화 효과 증가
    ❌ Source 커밋 지연 증가 (설정값만큼 지연)
  → Source 지연 허용 범위 내에서 설정

병렬 복제의 한계:
  단일 대용량 트랜잭션 → 효과 없음 (청크 분리 필요)
  핫스팟 테이블 → Worker 모두가 같은 Row 경합 → 실질 병렬화 불가
  매우 낮은 TPS → 병렬화 대상 자체 부족
```

---

## 📌 핵심 정리

```
병렬 복제 핵심:

문제:
  단일 SQL Thread = Source 병렬성 따라잡기 불가
  SQL Thread 병목으로 Lag 지속 누적

해결: LOGICAL_CLOCK 기반 병렬 복제 (MySQL 5.7+)
  replica_parallel_type = LOGICAL_CLOCK
  replica_parallel_workers = 4~16 (CPU 코어 기준)
  replica_preserve_commit_order = ON

LOGICAL_CLOCK 원리:
  same last_committed = Source에서 동시 커밋 = 충돌 없음
  → Replica에서도 병렬 재실행 가능

DATABASE 방식:
  단일 스키마에서 효과 없음 → LOGICAL_CLOCK 사용

WRITESET (MySQL 8.0):
  Row 수준 쓰기 집합으로 더 정밀한 병렬화
  LOGICAL_CLOCK보다 20~30% 추가 향상

한계:
  단일 대용량 트랜잭션 → 병렬화 불가 (LIMIT 청크 분리 필요)
  핫스팟 테이블 → 실질 병렬화 제한

모니터링:
  performance_schema.replication_applier_status_by_worker
  → Workers 상태, 오류, 처리 중 트랜잭션 확인
```

---

## 🤔 생각해볼 문제

**Q1.** LOGICAL_CLOCK 방식으로 병렬 복제를 활성화했는데 Lag이 여전히 크다. 원인 진단 방법을 3가지 제시하라.

<details>
<summary>해설 보기</summary>

**진단 1: Workers 상태 확인 — 단일 Worker 집중 여부**
```sql
SELECT WORKER_ID, SERVICE_STATE, APPLYING_TRANSACTION
FROM performance_schema.replication_applier_status_by_worker;
-- 하나의 Worker만 계속 APPLYING이고 나머지가 IDLE이면
-- 대용량 단일 트랜잭션이 그 Worker를 독점 → 청크 분리 필요
```

**진단 2: Binary Log의 last_committed 분포 확인**
```bash
mysqlbinlog --base64-output=DECODE-ROWS -v mysql-bin.000001 \
  | grep "last_committed" | head -50
# 모든 트랜잭션의 last_committed가 다르면 (모두 직렬)
# → 그룹 커밋이 안 됨 → binlog_group_commit_sync_delay 증가 고려
# last_committed가 반복되면 → 병렬화 가능한 그룹이 있음
```

**진단 3: IO Thread vs SQL Thread 병목 구분**
```sql
SHOW REPLICA STATUS\G
-- Read_Source_Log_Pos ≈ Source 최신 → IO Thread OK, SQL Thread 병목
-- Read_Source_Log_Pos << Source 최신 → IO Thread도 병목
-- SQL Thread 병목이라면 Workers 증가 고려
-- IO Thread 병목이라면 Binary Log 압축, 네트워크 대역폭 확인
```

</details>

---

**Q2.** Source의 `binlog_group_commit_sync_delay`를 증가시키면 Replica 병렬 복제에 어떤 영향을 주는가? 트레이드오프를 설명하라.

<details>
<summary>해설 보기</summary>

**효과 — Replica 병렬화 향상**:

`binlog_group_commit_sync_delay = 100` (100ms):
- Source가 Binary Log를 기록하기 전에 최대 100ms 대기
- 이 100ms 동안 도착한 트랜잭션들을 하나의 그룹으로 묶음
- 같은 그룹의 트랜잭션 = 같은 `last_committed` = Replica에서 병렬 재실행 가능
- → 더 많은 트랜잭션을 병렬 처리 가능

**트레이드오프 — Source 응답 지연**:
- 클라이언트가 COMMIT 완료를 기다리는 시간 증가
- 0ms 설정: COMMIT 즉시 Binary Log 기록 → 빠른 응답, 적은 병렬화
- 100ms 설정: 100ms 추가 지연 → 느린 응답, 더 많은 병렬화

**실무 조언**:
```sql
-- 점진적으로 증가하며 효과 측정
SET GLOBAL binlog_group_commit_sync_delay = 0;    -- 기준
SET GLOBAL binlog_group_commit_sync_delay = 50;   -- 50ms
SET GLOBAL binlog_group_commit_sync_delay = 100;  -- 100ms

-- Source 응답 시간 SLA 허용 범위 내에서 설정
-- Replica Lag 개선 vs Source 응답 지연 균형점 찾기
-- 일반적으로 100ms 이내 설정 권장
```

</details>

---

<div align="center">

**[⬅️ Semi-Sync](./05-semi-sync-replication.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring Read/Write 분리 ➡️](./07-spring-read-write-separation.md)**

</div>
