# Replication Lag 발생 원인 — IO Thread vs SQL Thread 병목 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Seconds_Behind_Source`의 정확한 의미와 자주 하는 오해는?
- IO Thread 병목과 SQL Thread 병목을 어떻게 구분하는가?
- 대용량 단일 트랜잭션이 Lag을 발생시키는 구조는?
- Replication Lag을 줄이는 현실적 방법은?
- 모니터링에서 어떤 지표를 어떻게 봐야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### Lag의 원인을 모르면 잘못된 해결책을 적용한다

```
실제 운영 이슈:
  Seconds_Behind_Source = 120초

  오진단 → 잘못된 해결:
    "네트워크 문제겠지" → 네트워크 대역폭 증설 (비용 낭비)
    실제 원인: 매일 밤 배치가 실행하는 UPDATE 100만 건짜리
               단일 트랜잭션이 SQL Thread를 2분 동안 차단

  또 다른 오진단:
    "병렬 복제 켜면 해결" → slave_parallel_workers = 16 설정
    실제 원인: IO Thread가 네트워크 대역폭 한계에 걸림
    → SQL Thread가 아무리 빨라도 받는 속도가 느리면 의미 없음

Lag 원인 파악 순서:
  1. SHOW REPLICA STATUS에서 Read vs Exec 위치 비교
  2. IO Thread 병목인지 SQL Thread 병목인지 구분
  3. 원인에 맞는 해결책 적용
```

---

## 😱 흔한 실수 (Before)

### 1. Seconds_Behind_Source = 0을 완전 동기화로 착각

```
Before (오해):
  Seconds_Behind_Source = 0
  → "Replica가 완전히 따라잡았다!" 판단
  → 즉시 Replica에서 읽기 시작

실제:
  SQL Thread가 처리할 이벤트가 없을 때 0 표시
  IO Thread가 뒤처져서 Relay Log가 비어있어도 0
  → 다음 이벤트가 도착하는 순간 즉시 증가

  Source와 Replica 간 실제 차이는
  Source의 최신 Binary Log 위치 vs Replica의 Read 위치로 확인
```

### 2. Lag 발생 시 무조건 병렬 복제 적용

```sql
-- Before: 원인 파악 없이 바로 병렬 복제 설정
SET GLOBAL replica_parallel_workers = 8;

-- 문제:
-- IO Thread 병목인 경우:
--   SQL Thread가 8배 빨라져도 받는 이벤트 자체가 느리면 효과 없음
-- 대용량 단일 트랜잭션인 경우:
--   하나의 긴 트랜잭션은 여러 Worker로 분산 불가 → 효과 없음

-- 병렬 복제는 SQL Thread 병목 + 독립적인 소규모 트랜잭션이 많을 때 효과
```

### 3. Lag 임계치 없이 Replica를 Read에 무조건 사용

```java
// Before: Lag 확인 없이 항상 Replica로 읽기 라우팅
@Transactional(readOnly = true)
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
    // Lag 120초일 때 2분 전 데이터를 반환할 수 있음
}

// 결과:
// 사용자: "방금 주문했는데 주문 내역이 없어요"
// → Lag 모니터링 없이 Replica를 무조건 사용한 결과
```

---

## ✨ 올바른 접근 (After)

```
Lag 진단 방법론:

단계 1: Seconds_Behind_Source 확인
  0이어도 안심 금지 → Source 최신 위치 비교 필수
  NULL이면 SQL Thread 중단 → 복제 오류 확인

단계 2: Read vs Exec 위치 비교
  Read_Source_Log_Pos ≈ Source 최신 위치 → IO Thread 따라잡음
  Read_Source_Log_Pos << Source 최신 위치 → IO Thread 병목

  Exec_Source_Log_Pos ≈ Read_Source_Log_Pos → SQL Thread 따라잡음
  Exec_Source_Log_Pos << Read_Source_Log_Pos → SQL Thread 병목

단계 3: 원인별 해결
  IO Thread 병목 → 네트워크 대역폭, Binary Log 압축
  SQL Thread 병목 → 병렬 복제, Replica 성능 향상
  대용량 단일 트랜잭션 → DML 청크 분리 (근본 해결)

단계 4: Lag 임계치 알람 설정
  경고: > 30초
  위험: > 300초
  애플리케이션 폴백: Lag 초과 시 Source로 읽기 전환
```

---

## 🔬 내부 동작 원리

### 1. Seconds_Behind_Source 계산 원리

```
Seconds_Behind_Source 계산:
  SQL Thread가 현재 실행 중인 이벤트의 원본 타임스탬프
  vs 현재 시간의 차이

  예시:
    Source에서 트랜잭션 발생 시각: 10:00:00
    현재 시각: 10:02:00
    SQL Thread가 처리 중인 이벤트가 10:00:00에 생성된 것이라면
    → Seconds_Behind_Source = 120

오해 포인트:
  ① SQL Thread가 이벤트 없이 대기 중 → 0 표시
     다음 이벤트 도착 → 즉시 증가
  
  ② IO Thread 지연은 반영 안 됨
     IO Thread가 1분 뒤처져도 SQL Thread가 따라잡았으면 0
     = Source 최신 데이터와 Replica의 실제 차이가 아님
  
  ③ SQL Thread 중단 시 NULL
     NULL ≠ 0, NULL = "측정 불가"
     복제 오류 또는 SQL Thread stop 명령 확인 필요

진짜 Lag 측정:
  Source 현재 binlog 위치 - Replica Read 위치 = 미전송 데이터 양
  또는 pt-heartbeat 도구: Source에 타임스탬프 기록 → Replica에서 비교
  → 더 정확한 Lag 측정 가능
```

### 2. IO Thread 병목 진단

```sql
-- IO Thread 병목 확인
SHOW REPLICA STATUS\G

-- 비교:
-- 1. Source 현재 위치 (Source 서버에서)
--    SHOW BINARY LOG STATUS → File, Position

-- 2. Replica의 IO Thread 수신 위치
--    Source_Log_File, Read_Source_Log_Pos

-- 3. Replica의 SQL Thread 실행 위치
--    Relay_Source_Log_File, Exec_Source_Log_Pos

-- 진단:
-- 케이스 A: Source 최신 = Read_Source_Log_Pos (거의 같음)
--          Read_Source_Log_Pos >> Exec_Source_Log_Pos
--          → IO Thread는 따라잡음, SQL Thread가 병목

-- 케이스 B: Source 최신 >> Read_Source_Log_Pos
--          → IO Thread가 병목 (수신 느림)
--          Exec와 Read의 차이는 부차적 문제

-- IO Thread 병목 원인:
-- ① 네트워크 대역폭 포화 (Binary Log 생성량 > 전송 가능량)
-- ② Source-Replica 간 지리적 거리 큼 (RTT 높음)
-- ③ Relay Log 쓰기 디스크 병목 (HDD 사용 시)

-- IO Thread 병목 해결:
-- Binary Log 압축 (MySQL 8.0.20+)
SET GLOBAL binlog_transaction_compression = ON;
-- 네트워크 대역폭 증설
-- Relay Log를 SSD에 위치하도록 변경
```

### 3. SQL Thread 병목 진단

```sql
-- SQL Thread 병목 확인:
-- Read_Source_Log_Pos ≫ Exec_Source_Log_Pos 일 때

-- Read는 최신을 따라잡았지만 Exec가 한참 뒤처짐
-- = Relay Log에 이벤트가 쌓이고 있음 → SQL Thread가 소화 못 함

-- SQL Thread 병목 원인 1: 대용량 단일 트랜잭션
-- Source: UPDATE orders SET status='ARCHIVED' WHERE created_at < '2021-01-01'
--         → 500만 건, 5분 소요
-- SQL Thread: 5분 동안 이 하나의 이벤트 처리
--            후속 트랜잭션 전부 대기
--            Seconds_Behind_Source 급증

-- SQL Thread 병목 원인 2: 단일 직렬 처리
-- Source: 100 TPS로 병렬 처리
-- SQL Thread(단일): 20~30 TPS 처리 가능
-- → 초당 70~80 TPS씩 Relay Log 누적

-- SQL Thread 병목 원인 3: Replica 서버 성능 부족
-- Source: NVMe SSD, 32 Core
-- Replica: SATA HDD, 8 Core → 재실행 속도 낮음

-- 확인 지표:
SHOW GLOBAL STATUS LIKE 'Slave_retried_transactions';
-- 재시도 트랜잭션 수 → 증가하면 락 경합 발생 중

SELECT * FROM performance_schema.replication_applier_status_by_worker\G
-- 각 Worker의 현재 처리 중인 트랜잭션 확인 (병렬 복제 시)
```

### 4. 대용량 단일 트랜잭션이 Lag을 발생시키는 구조

```
시나리오: 배치 작업
  Source에서 실행:
  BEGIN;
    UPDATE orders SET status = 'ARCHIVED'
    WHERE created_at < '2021-01-01';
    -- 500만 건, 6분 소요
  COMMIT;

Relay Log 기록 (IO Thread 시점):
  Source가 6분 동안 트랜잭션 처리
  IO Thread는 이 기간 동안 Row 이벤트들을 스트리밍으로 수신
  Relay Log에 수십 GB 누적
  → Seconds_Behind_Source 서서히 증가

SQL Thread 처리:
  Relay Log에서 이 6분짜리 트랜잭션을 재실행 시작
  재실행 중 후속 트랜잭션들은 모두 대기
  재실행 완료 후 후속 트랜잭션 처리 시작
  → 최악의 경우 6분 + 누적된 후속 트랜잭션 처리 시간 = Lag 폭발

근본 해결: 배치 DML을 청크 단위로 분리
  WHILE rows_updated > 0 DO
    UPDATE orders SET status = 'ARCHIVED'
    WHERE created_at < '2021-01-01'
    AND status != 'ARCHIVED'
    LIMIT 1000;
    SET rows_updated = ROW_COUNT();
    DO SLEEP(0.1);  -- Replica 여유 시간 확보
  END WHILE;

  각 트랜잭션: 1000건 = 수 초
  SQL Thread: 처리 후 다음 트랜잭션 받아서 처리 가능
  → Lag 누적 방지
```

### 5. 모니터링 지표 체계

```sql
-- 핵심 지표 쿼리
SELECT
    Seconds_Behind_Source,
    Replica_IO_Running,
    Replica_SQL_Running,
    Read_Source_Log_Pos,
    Exec_Source_Log_Pos,
    (Read_Source_Log_Pos - Exec_Source_Log_Pos) AS relay_log_backlog,
    Last_Error
FROM (SHOW REPLICA STATUS) AS replica_status\G

-- Prometheus 지표 (MySQL Exporter):
-- mysql_slave_status_seconds_behind_master
-- mysql_slave_status_slave_io_running (1=Running)
-- mysql_slave_status_slave_sql_running (1=Running)
-- mysql_slave_status_read_master_log_pos
-- mysql_slave_status_exec_master_log_pos

-- pt-heartbeat (더 정확한 Lag 측정):
-- Source: pt-heartbeat --update --database mydb
-- Replica: pt-heartbeat --monitor --database mydb
-- → 실제 데이터 전파 지연 시간을 초 단위로 측정

-- Lag 임계치 알람 설정 기준:
-- 30초: 경고 (Read/Write 분리 서비스 영향 시작)
-- 300초: 위험 (5분 이전 데이터 제공 중)
-- NULL: 즉시 대응 (SQL Thread 중단, 복제 오류)
```

---

## 💻 실전 실험

### 실험 1: Lag 진단 — Read vs Exec 위치 비교

```sql
-- Replica에서 실행
SHOW REPLICA STATUS\G
-- 확인 항목:
-- Source_Log_File: mysql-bin.000010
-- Read_Source_Log_Pos: 50000000    ← IO Thread 수신 위치
-- Relay_Source_Log_File: mysql-bin.000010
-- Exec_Source_Log_Pos: 45000000    ← SQL Thread 실행 위치
-- Seconds_Behind_Source: 30

-- Read - Exec = 5,000,000 바이트 = Relay Log 재고
-- Source 최신 위치 (Source 서버에서): SHOW BINARY LOG STATUS
-- Source 최신 = Read → IO Thread 따라잡음, SQL Thread가 병목
-- Source 최신 >> Read → IO Thread도 병목
```

### 실험 2: 대용량 트랜잭션 Lag 시뮬레이션

```sql
-- Source에서 대용량 트랜잭션 실행
-- (주의: 실제 운영 환경이 아닌 테스트 환경에서 수행)
CREATE TABLE lag_test (id INT AUTO_INCREMENT PRIMARY KEY, val VARCHAR(100)) ENGINE=InnoDB;
INSERT INTO lag_test (val) SELECT REPEAT('x', 100)
FROM information_schema.columns a, information_schema.columns b LIMIT 100000;

-- 대용량 UPDATE 실행
UPDATE lag_test SET val = REPEAT('y', 100);

-- 다른 세션에서 Replica Lag 모니터링
-- (Replica에서 주기적으로 실행)
-- SHOW REPLICA STATUS\G → Seconds_Behind_Source 증가 확인

DROP TABLE lag_test;
```

### 실험 3: 청크 단위 DML vs 단일 DML Lag 비교

```sql
-- 테스트 테이블 (Source에서)
CREATE TABLE chunk_test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    status VARCHAR(20) DEFAULT 'PENDING',
    val VARCHAR(100)
) ENGINE=InnoDB;

INSERT INTO chunk_test (val)
SELECT REPEAT('a', 50)
FROM information_schema.columns a
CROSS JOIN information_schema.columns b
LIMIT 50000;

-- 방법 1: 단일 대용량 트랜잭션
SET @t = SYSDATE(6);
UPDATE chunk_test SET status = 'DONE';
SELECT CONCAT('단일 트랜잭션: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');
-- Replica에서 이 기간 동안 Lag 모니터링

-- 상태 리셋
UPDATE chunk_test SET status = 'PENDING';

-- 방법 2: 청크 단위 트랜잭션
SET @done = 0;
SET @t = SYSDATE(6);
REPEAT
    UPDATE chunk_test SET status = 'DONE'
    WHERE status = 'PENDING' LIMIT 1000;
    SET @done = (ROW_COUNT() = 0);
    DO SLEEP(0.05);
UNTIL @done END REPEAT;
SELECT CONCAT('청크 방식: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');
-- Replica에서 Lag 최대치 비교

DROP TABLE chunk_test;
```

---

## 📊 성능/비용 비교

```
단일 트랜잭션 vs 청크 분리 Lag 비교 (50만 건 UPDATE):

단일 트랜잭션:
  Source 실행 시간: 30초
  Binary Log 생성: ~100MB
  SQL Thread 차단 시간: 30초
  최대 Seconds_Behind_Source: 30~60초
  후속 트랜잭션 지연: 30초 동안 전부 대기

청크 분리 (1,000건 × 500회, SLEEP 0.05초):
  Source 총 실행 시간: ~30초 (슬립 포함)
  개별 트랜잭션: 0.05~0.1초
  SQL Thread 차단 시간: 트랜잭션당 0.05~0.1초
  최대 Seconds_Behind_Source: 0.1~1초
  후속 트랜잭션 지연: 트랜잭션 간 여유 시간에 끼어들기 가능

IO Thread 병목 해결 효과:
  Binary Log 압축 비활성:  100MB/s Binary Log → 네트워크 포화
  Binary Log 압축 활성화:  ~30MB/s → 동일 트랜잭션 더 빠른 전송
  Lag 감소 효과: 전송 속도에 따라 30~70% 개선 가능
```

---

## ⚖️ 트레이드오프

```
Lag 해결 방법별 트레이드오프:

병렬 복제 (SQL Thread 병목 해결):
  ✅ SQL Thread 처리량 3~5배 향상 가능
  ✅ Source의 병렬성에 가깝게 재생
  ❌ 단일 대용량 트랜잭션 → 효과 없음
  ❌ 워커 간 데드락 발생 가능 → 재시도 비용
  ❌ replica_preserve_commit_order=ON 필수 → 약간의 오버헤드

Binary Log 압축 (IO Thread 병목 해결):
  ✅ 네트워크 전송량 30~70% 감소
  ✅ 네트워크 대역폭 제한 환경에서 효과적
  ❌ Source/Replica CPU 사용량 증가 (압축/압축해제)
  ❌ mysqlbinlog으로 확인 시 압축 해제 필요

청크 단위 DML (대용량 트랜잭션 해결):
  ✅ Lag 최대치 대폭 감소
  ✅ 후속 트랜잭션 블로킹 방지
  ✅ 배치 실패 시 부분 완료 상태 (전체 롤백 불필요)
  ❌ 전체 작업 시간 증가 (SLEEP 포함)
  ❌ 중간 상태 발생 (부분 완료)
  ❌ 애플리케이션 코드 변경 필요

Replica 성능 향상:
  ✅ 가장 직접적인 해결
  ❌ 비용 증가
  ❌ Lag이 아닌 근본 트래픽 증가는 Replica 증설로는 한계
```

---

## 📌 핵심 정리

```
Replication Lag 핵심:

진단 방법:
  SHOW REPLICA STATUS에서 3가지 위치 비교:
  ① Source 최신 위치 (SHOW BINARY LOG STATUS)
  ② Read_Source_Log_Pos (IO Thread 수신 위치)
  ③ Exec_Source_Log_Pos (SQL Thread 실행 위치)

  ① ≈ ② : IO Thread 따라잡음
  ② >> ③ : SQL Thread 병목
  ① >> ② : IO Thread 병목

Seconds_Behind_Source 주의사항:
  0 = 완전 동기화 아님 (SQL Thread 쉬는 중)
  NULL = SQL Thread 중단 (오류 확인)
  정확한 Lag = pt-heartbeat 사용 권장

원인별 해결:
  IO Thread 병목 → Binary Log 압축, 네트워크 대역폭
  SQL Thread 병목 → 병렬 복제 (LOGICAL_CLOCK)
  대용량 단일 트랜잭션 → LIMIT 청크 분리 (근본 해결)

알람 기준:
  > 30초: 경고
  > 300초: 위험
  NULL: 즉시 대응
```

---

## 🤔 생각해볼 문제

**Q1.** `Seconds_Behind_Source = 0`인데 Replica에서 최근 INSERT 데이터가 안 보인다. 가능한 원인 3가지를 설명하라.

<details>
<summary>해설 보기</summary>

**원인 1: IO Thread가 뒤처져 있음**
SQL Thread는 받은 이벤트를 모두 처리해서 0이지만, IO Thread가 Source를 따라잡지 못해 최신 데이터를 아직 받지 못한 상태. Seconds_Behind_Source는 SQL Thread 기준이라 IO Thread 지연이 반영 안 됨.

진단: Source 최신 위치 vs `Read_Source_Log_Pos` 비교.

**원인 2: SQL Thread가 이벤트 없이 대기 중**
마지막 이벤트를 처리한 직후이고 다음 이벤트가 아직 도착하지 않은 상태. 확인한 타이밍이 이 사이였을 경우 0이지만 곧 증가.

**원인 3: 쿼리 타이밍 문제**
Source에서 COMMIT 완료 직후 확인했을 때 Binary Log 전송 + IO Thread 수신 + SQL Thread 재실행까지 수백 ms의 시간이 필요. 이 사이에 조회했다면 Seconds_Behind_Source는 0이지만 데이터가 없을 수 있음.

근본 해결: `Seconds_Behind_Source = 0`을 신뢰하지 말고 pt-heartbeat로 실제 데이터 전파 지연을 측정. Read/Write 분리 시 쓰기 직후 읽기는 Source에서 처리.

</details>

---

**Q2.** 다음 배치 쿼리가 매일 밤 실행될 때 Replication Lag을 최소화하도록 개선하라.

```sql
-- 매일 밤 11시 실행
DELETE FROM user_logs WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
-- 예상 삭제 건수: 200만 건
```

<details>
<summary>해설 보기</summary>

**문제**: 200만 건 단일 트랜잭션 DELETE → Binary Log에 200만 Row 이벤트 기록 → SQL Thread 수분 차단 → Lag 폭발.

**개선 방법**:

```sql
-- 방법 1: LIMIT 청크 분리 (저장 프로시저)
DELIMITER //
CREATE PROCEDURE cleanup_old_logs()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE batch_size INT DEFAULT 5000;

    REPEAT
        DELETE FROM user_logs
        WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY)
        LIMIT batch_size;

        SET done = (ROW_COUNT() < batch_size);
        DO SLEEP(0.1);  -- Replica 여유 시간 (Lag 확인 후 조정)
    UNTIL done END REPEAT;
END //
DELIMITER ;

-- 방법 2: 파티셔닝 활용 (근본 해결)
-- created_at으로 월별 RANGE 파티션 구성 시
ALTER TABLE user_logs DROP PARTITION p_before_90days;
-- 밀리초 단위, Binary Log 없음 → Lag 전혀 없음

-- 방법 3: Lag 모니터링 추가
-- 배치 중 Replica Lag 확인하며 속도 조절
-- Lag > 30초 → SLEEP 증가
-- Lag < 5초 → SLEEP 감소
```

실무에서는 파티셔닝을 처음부터 설계하는 방법이 가장 효과적이고, 기존 테이블은 청크 분리가 현실적인 해결책입니다.

</details>

---

<div align="center">

**[⬅️ GTID](./03-gtid-based-replication.md)** | **[홈으로 🏠](../README.md)** | **[다음: Semi-Sync ➡️](./05-semi-sync-replication.md)**

</div>
