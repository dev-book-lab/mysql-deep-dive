# MySQL Replication 아키텍처 — Source/Replica와 Binary Log 흐름

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Binary Log → IO Thread → Relay Log → SQL Thread 전 과정은?
- Binary Log와 InnoDB Redo Log는 왜 별개로 존재하는가?
- 비동기 복제에서 Replica가 구조적으로 뒤처질 수밖에 없는 이유는?
- `SHOW REPLICA STATUS`에서 복제 상태를 어떻게 읽는가?
- Source 장애 시 Replica에 없는 데이터가 생기는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### Replication을 모르면 장애 대응이 불가능하다

```
실제 장애 시나리오:
  Source DB 서버 다운
  → Replica를 새 Source로 승격 (Failover)
  → 일부 사용자: "방금 주문했는데 내역이 없어요"

  원인:
    Source: 주문 INSERT 커밋 완료 (클라이언트 응답 반환)
    Replica: 아직 해당 트랜잭션 미수신 (비동기 복제)
    → Replica를 Source로 승격했지만 해당 주문 데이터 없음
    → 데이터 유실

  이 문제를 이해하려면:
    ① 복제가 비동기로 동작하는 이유
    ② Binary Log 전송과 적용 사이의 시간 차이
    ③ 각 스레드의 역할과 병목 지점
```

---

## 😱 흔한 실수 (Before)

### 1. "커밋됐으면 Replica에도 있겠지"

```
잘못된 믿음:
  Source에서 COMMIT 완료 = Replica에도 즉시 반영

실제:
  COMMIT → Binary Log 기록 → [네트워크 전송] → IO Thread 수신
  → Relay Log 기록 → [SQL Thread 처리 대기] → SQL Thread 재생
  → 최종적으로 Replica에 반영

  이 과정에서 수백 ms ~ 수 초의 지연 발생
  대량 트랜잭션이면 수십 초 ~ 분 단위 지연 가능

실무 영향:
  Source에서 INSERT 후 즉시 Replica에서 SELECT → 데이터 없음!
  Read/Write 분리 구현 시 반드시 고려해야 할 Stale Read 문제
```

### 2. Binary Log와 Redo Log를 같은 것으로 오해

```
오해:
  "Binary Log가 바로 InnoDB의 복구 로그 아닌가요?"

실제:
  Redo Log (InnoDB 내부):
    목적: InnoDB 엔진 크래시 복구
    내용: 페이지 레벨의 물리적 변경 (WAL)
    범위: 해당 MySQL 인스턴스만

  Binary Log (MySQL 서버 레벨):
    목적: 복제 + 시점 복구 (Point-in-Time Recovery)
    내용: SQL 문 또는 Row 변경 이벤트 (논리적)
    범위: Replica로 전송 가능, 인스턴스 독립적

  두 로그는 별개이며 각각 독립적으로 기록됨
  둘 다 있어야 안전한 복제 + 복구 보장
```

---

## ✨ 올바른 접근 (After)

```
MySQL Replication 전체 흐름:

Source 쪽:
  클라이언트 → SQL 실행 → InnoDB 처리
  ① InnoDB: Redo Log 기록 (크래시 복구용)
  ② MySQL: Binary Log 기록 (복제 + PITR용)
  ③ 클라이언트에 COMMIT 완료 응답 반환
  ④ Binary Log Dump Thread: Replica에게 새 이벤트 전송

Replica 쪽:
  ① IO Thread: Source의 Binary Log 이벤트 수신
               → Relay Log 파일에 기록
  ② SQL Thread: Relay Log에서 이벤트 읽기
                → SQL 또는 Row 변경 재실행 (apply)
  ③ InnoDB: 변경 적용 완료

비동기 복제:
  Source는 Replica의 수신 여부와 무관하게 COMMIT 완료 반환
  → Replica가 뒤처져도 Source 성능에 영향 없음
  → 하지만 Source 장애 시 미전송 데이터 유실 가능
```

---

## 🔬 내부 동작 원리

### 1. Binary Log 기록 시점과 2단계 커밋

```
InnoDB의 2단계 커밋 (2PC) 원리:

단계 1 (Prepare):
  InnoDB: Redo Log에 prepare 상태 기록
  → "이 트랜잭션을 커밋할 준비가 됐다"

단계 2 (Commit):
  MySQL: Binary Log에 이벤트 기록
  InnoDB: Redo Log에 commit 상태 기록
  → 순서 보장: Binary Log가 먼저, 그 다음 InnoDB commit

왜 2PC인가:
  Binary Log + InnoDB Redo Log 두 로그의 일관성 보장
  MySQL 크래시 시:
    Binary Log에 있고 Redo Log에 커밋 없음
    → Binary Log 기반으로 InnoDB 커밋 완료 처리
  
  두 로그가 일관되어야:
    Replica에 전송된 데이터 = InnoDB에 실제 반영된 데이터

MySQL 8.0 binlog_order_commits:
  그룹 커밋 최적화로 여러 트랜잭션을 묶어 Binary Log 기록
  → Binary Log fsync 횟수 감소 → 성능 향상
```

### 2. Replica IO Thread와 SQL Thread 분리 설계

```
IO Thread (Binary Log 수신):
  Source의 Binary Log Dump Thread와 연결 유지
  새 이벤트 수신 → Relay Log 파일에 순서대로 기록
  작업: 네트워크 수신 + 파일 쓰기 (단순)
  병목: 네트워크 대역폭, Relay Log 쓰기 속도

SQL Thread (Relay Log 재생):
  Relay Log에서 이벤트 읽기 → SQL 또는 Row 변경 재실행
  Source에서 실행된 것과 동일한 순서로 직렬 실행 (기본)
  작업: SQL 파싱 + InnoDB 실행 + 인덱스 갱신
  병목: 대용량 트랜잭션, 락 경합, 디스크 I/O

분리 설계의 이점:
  IO Thread: 네트워크 끊겨도 재연결 후 이어받기 가능
  SQL Thread: 느려도 IO Thread는 계속 수신 가능
              Relay Log에 버퍼 쌓임 (디스크 공간 주의)

상태 확인:
  SHOW REPLICA STATUS\G
  Replica_IO_Running: Yes/No   → IO Thread 상태
  Replica_SQL_Running: Yes/No  → SQL Thread 상태
  둘 다 Yes이어야 정상 복제 중
```

### 3. Binary Log 파일 구조

```
Binary Log 파일:
  mysql-bin.000001, mysql-bin.000002, ...
  binlog_index 파일에 목록 관리

  각 이벤트:
    Header: 이벤트 타입, 타임스탬프, 서버 ID
    Data: STATEMENT: SQL 문자열
          ROW: 변경 전/후 Row 데이터 (바이너리)

Binary Log Position:
  파일명 + 파일 내 Offset으로 특정 위치 지정
  SHOW MASTER STATUS: 현재 Source의 Binary Log 위치
  예: mysql-bin.000005 / Position: 1234567

  복제 시작 시:
    Replica에게 "파일명 + 위치부터 이벤트 전송" 요청
    CHANGE REPLICATION SOURCE TO
      SOURCE_LOG_FILE='mysql-bin.000005',
      SOURCE_LOG_POS=1234567;

Binary Log 보존:
  binlog_expire_logs_seconds (기본 30일)
  만료된 파일 자동 삭제
  PITR을 위해 충분한 보존 기간 설정 필요
```

### 4. 복제 지연이 발생하는 구조적 원인

```
구조적 지연 원인:

원인 1: 비동기 네트워크 전송
  Source COMMIT → Binary Log 기록 → 전송 → Replica 수신
  네트워크 지연 = Source-Replica 간 거리, 부하

원인 2: SQL Thread 단일 직렬 처리 (기본)
  Source: 병렬로 100개 트랜잭션 동시 처리
  SQL Thread: 1개 스레드로 100개 트랜잭션 순서대로 처리
  → Source의 병렬성을 따라갈 수 없음
  → Lag 누적 (병렬 복제로 해결: 06 문서)

원인 3: 대용량 단일 트랜잭션
  Source: UPDATE 100만 건 = 5분 걸림
  Relay Log: 이벤트 기록 완료 (Binary Log 스트리밍)
  SQL Thread: 5분짜리 UPDATE를 재실행
  → 이 5분 동안 후속 트랜잭션 대기
  → Seconds_Behind_Source 급증

원인 4: Replica 리소스 부족
  Source: 고성능 서버
  Replica: 저사양 서버 (비용 절감)
  → SQL Thread가 Source 속도 못 따라감
```

### 5. 복제 상태 모니터링

```sql
-- 기본 복제 상태
SHOW REPLICA STATUS\G

-- 핵심 항목:
-- Replica_IO_Running: Yes         → IO Thread 정상
-- Replica_SQL_Running: Yes        → SQL Thread 정상
-- Seconds_Behind_Source: 0        → 지연 초 (0이면 따라잡음)
-- Source_Log_File: mysql-bin.000005  → Source의 현재 Binary Log
-- Read_Source_Log_Pos: 1234567    → IO Thread가 읽은 위치
-- Relay_Source_Log_File: mysql-bin.000005 → SQL Thread가 처리 중인 파일
-- Exec_Source_Log_Pos: 1230000    → SQL Thread가 실행 완료한 위치

-- Read_Source_Log_Pos - Exec_Source_Log_Pos = SQL Thread 미처리 양
-- 이 차이가 크면 SQL Thread 병목 → 병렬 복제 고려

-- Performance Schema로 더 상세 확인 (MySQL 8.0)
SELECT * FROM performance_schema.replication_applier_status_by_worker\G
-- 병렬 복제 시 각 워커 상태 확인

-- Binary Log 목록 확인
SHOW BINARY LOGS;
SHOW MASTER STATUS;  -- MySQL 8.0에서는 SHOW BINARY LOG STATUS

-- Source에서 특정 위치 이후 이벤트 확인
SHOW BINLOG EVENTS IN 'mysql-bin.000005' FROM 1234567 LIMIT 10;
```

---

## 💻 실전 실험

### 실험 1: 복제 설정 확인

```sql
-- Source에서
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'server_id';
SHOW BINARY LOG STATUS;

-- Replica에서
SHOW REPLICA STATUS\G
-- Replica_IO_Running, Replica_SQL_Running 확인
-- Seconds_Behind_Source 확인
```

### 실험 2: Binary Log 이벤트 확인

```sql
-- Source에서 Binary Log 이벤트 조회
SHOW BINARY LOGS;
-- Log_name, File_size 확인

-- 특정 Binary Log의 이벤트 목록
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 20;
-- Event_type, Pos, Info 확인
-- Transaction 경계: Gtid_log_event, Query(BEGIN), ..., Xid(COMMIT)

-- mysqlbinlog 도구로 상세 확인 (터미널)
-- mysqlbinlog --base64-output=DECODE-ROWS -v mysql-bin.000001 | head -100
```

### 실험 3: 복제 지연 시뮬레이션

```sql
-- Source에서 대용량 트랜잭션 실행
START TRANSACTION;
UPDATE big_table SET col = col + 1;  -- 수백만 건 업데이트
COMMIT;

-- 다른 세션에서 Replica 지연 확인
-- (Replica에서)
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source 증가 확인
-- Exec_Source_Log_Pos와 Read_Source_Log_Pos 차이 확인
```

---

## 📊 성능/비용 비교

```
동기 복제 vs 비동기 복제 비교:

비동기 복제 (기본):
  Source COMMIT 응답 지연: 없음 (Replica 무관)
  데이터 유실 위험: 있음 (Source 장애 시 미전송 데이터)
  Replica 지연: 발생 가능 (Seconds_Behind_Source > 0)
  성능 영향: 없음

Semi-Sync 복제 (05 문서):
  Source COMMIT 응답 지연: 최소 1 Replica 수신 확인 후 완료
  데이터 유실 위험: 매우 낮음 (Relay Log 수신 보장)
  Replica 지연: 최소 (수신은 보장, 적용은 비동기)
  성능 영향: 네트워크 RTT만큼 지연 추가

완전 동기 복제 (Galera Cluster 등):
  Source COMMIT 응답 지연: 모든 노드 적용 확인 후 완료
  데이터 유실 위험: 없음
  성능 영향: 네트워크 RTT × 왕복 횟수
```

---

## ⚖️ 트레이드오프

```
비동기 복제 선택 트레이드오프:

이점:
  ✅ Source 성능에 Replica 영향 없음
  ✅ Replica 다운돼도 Source 정상 운영
  ✅ 구현 단순, 설정 최소

비용:
  ❌ Source 장애 시 데이터 유실 가능
  ❌ Replica Lag으로 Stale Read 발생 가능
  ❌ Failover 후 데이터 불일치 처리 필요

선택 기준:
  허용 가능한 데이터 유실 = 비동기 + 모니터링
  데이터 유실 불허 = Semi-Sync 또는 동기 복제 검토
  Read 일관성 필요 = Source에서만 읽기 또는 지연 허용 범위 내 Replica
```

---

## 📌 핵심 정리

```
Replication 아키텍처 핵심:

흐름:
  Source: COMMIT → Binary Log → Dump Thread → 전송
  Replica: IO Thread 수신 → Relay Log → SQL Thread 재실행

Binary Log vs Redo Log:
  Redo Log: InnoDB 엔진 크래시 복구 (물리적, 내부용)
  Binary Log: 복제 + PITR (논리적, 네트워크 전송 가능)
  2PC로 두 로그 일관성 보장

비동기 복제 한계:
  Source COMMIT ≠ Replica 반영
  Lag = IO Thread 수신 지연 + SQL Thread 처리 지연
  Source 장애 시 미전송 트랜잭션 유실 가능

모니터링:
  SHOW REPLICA STATUS: Seconds_Behind_Source, 스레드 상태
  Seconds_Behind_Source = 0: 지연 없음
  Read vs Exec 위치 차이: SQL Thread 병목 신호
```

---

## 🤔 생각해볼 문제

**Q1.** Source에서 COMMIT이 완료된 트랜잭션이 Replica에 없을 수 있는 정확한 이유를 흐름 순서대로 설명하라.

<details>
<summary>해설 보기</summary>

비동기 복제의 흐름:
1. Source에서 `COMMIT` → InnoDB Redo Log + Binary Log에 기록 완료
2. 클라이언트에 커밋 성공 응답 반환 (이 시점에서 클라이언트 입장에서는 완료)
3. Source의 Binary Log Dump Thread: 새 이벤트를 Replica IO Thread에게 전송 시작
4. **이 3번 단계가 완료되기 전에 Source 서버 장애 발생**
5. Replica IO Thread: 해당 이벤트 미수신 → Relay Log에 없음
6. SQL Thread: 미수신 이벤트 재실행 불가 → Replica에 데이터 없음

핵심: Source에서 COMMIT 완료와 Replica IO Thread의 수신은 독립 비동기 이벤트입니다. `COMMIT 완료 → 응답 반환 → 네트워크 전송` 순서로 동작하므로, 전송 완료 전 장애 시 유실이 발생합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Binary Log 포맷 ➡️](./02-binlog-formats-statement-row-mixed.md)**

</div>
