# Semi-Synchronous Replication — 동기/비동기 복제의 트레이드오프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 비동기 복제에서 Source 장애 시 커밋된 트랜잭션이 유실되는 시나리오는?
- Semi-Sync가 최소 1개 Replica의 Relay Log 수신을 확인 후 커밋 완료하는 방식은?
- `rpl_semi_sync_source_timeout` 타임아웃 후 비동기로 강등되는 조건과 복귀 조건은?
- After-Commit vs After-Sync(Loss-less)의 차이는?
- Semi-Sync가 성능에 미치는 실제 영향은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 비동기 복제의 치명적 약점 — 커밋된 데이터가 사라진다

```
비동기 복제에서 데이터 유실 시나리오:

T=0.000: 결제 트랜잭션 Source에서 COMMIT 완료
T=0.000: 클라이언트에게 "결제 성공" 응답 전송
T=0.001: Binary Log Dump Thread가 Replica에 전송 시작
T=0.050: Source 서버 하드웨어 장애로 다운!
         전송 50ms 진행 중 → Replica는 아직 못 받음
T=0.100: Replica가 새 Source로 승격 (Failover)
T=0.200: 고객이 결제 내역 조회 → 없음!
          "분명히 결제 완료됐는데 내역이 없어요"

실제 발생하는가?
  Source-Replica 간 네트워크 전송 = 비동기
  장애 타이밍이 맞으면 발생 가능 (낮은 확률이지만 금전 데이터는 0%가 아님)

Semi-Sync로 방어:
  T=0.000: 결제 트랜잭션 COMMIT 준비 (Binary Log 기록)
  T=0.001: Replica에 전송 시작
  T=0.050: Replica: Relay Log 수신 완료 → ACK 전송
  T=0.051: Source: ACK 확인 후 COMMIT 완료, 클라이언트 응답
  → Source 장애 시 Relay Log에는 반드시 존재
  → Failover 후 SQL Thread가 적용 → 데이터 보존
```

---

## 😱 흔한 실수 (Before)

### 1. Semi-Sync가 "완전한 동기 복제"라고 오해

```
Before (잘못된 이해):
  "Semi-Sync 쓰면 Source와 Replica가 항상 동일하다"

실제:
  Semi-Sync는 "Relay Log 수신"을 보장 (IO Thread)
  SQL Thread의 "적용 완료"는 보장하지 않음

  Relay Log에 있음 ≠ Replica에 반영됨
  
  확인 시점에 따라:
    SQL Thread가 아직 Relay Log 이벤트를 처리 중
    → Replica에서 해당 데이터 조회 불가
    → Stale Read 여전히 발생
```

### 2. 타임아웃 설정 없이 Semi-Sync 적용

```sql
-- Before: 기본 타임아웃(10초) 그대로 사용
SET GLOBAL rpl_semi_sync_source_enabled = ON;

-- 문제:
-- Replica 장애 또는 네트워크 일시 단절 시
-- Source가 모든 트랜잭션마다 최대 10초 대기
-- → 10초 동안 클라이언트 응답 없음 → 서비스 장애 수준 지연

-- 타임아웃 후 비동기로 강등되어 계속 진행하지만
-- 그 10초 동안의 피해는 이미 발생

-- 적절한 설정:
SET GLOBAL rpl_semi_sync_source_timeout = 1000;  -- 1초
-- Replica가 1초 내에 ACK 안 보내면 비동기로 전환
```

### 3. After-Commit 방식으로 팬텀 커밋 발생

```
After-Commit (구버전) 방식:
  1. InnoDB 엔진 커밋 완료 (다른 세션에서 이 데이터 보임)
  2. Binary Log 기록
  3. Replica 전송 + ACK 대기
  4. ACK 수신 후 클라이언트 응답

  문제 시나리오:
    1번 완료 후 3번 대기 중 Source 장애
    다른 세션에서 이미 이 데이터를 읽었음 (InnoDB 커밋 완료)
    Replica에는 없음 (ACK 미수신 = Relay Log 미기록)
    → Failover 후 데이터 불일치 (팬텀 커밋)

After-Sync (Loss-less, MySQL 5.7+ 기본):
  1. Binary Log 기록
  2. Replica 전송 + ACK 대기
  3. ACK 수신
  4. InnoDB 엔진 커밋 완료
  5. 클라이언트 응답

  장점: InnoDB 커밋 전 Replica 수신 확인
        장애 시 InnoDB 미커밋 → 다른 세션에도 안 보임
        → 팬텀 커밋 없음, 완전한 Loss-less
```

---

## ✨ 올바른 접근 (After)

```
Semi-Sync 적용 기준:

데이터 유실 허용 불가 + 같은 데이터센터 Replica 있음:
  → Semi-Sync After-Sync 적용
  → rpl_semi_sync_source_timeout: 500~2000ms
  → 추가 지연: 네트워크 RTT (~1ms)

데이터 유실 허용 불가 + 지리적으로 먼 Replica:
  → Semi-Sync + 타임아웃 짧게 (500ms)
  → 빠른 비동기 강등으로 레이턴시 영향 최소화
  → Rpl_semi_sync_source_no_tx 모니터링

데이터 유실 일부 허용 가능 + 고성능 필요:
  → 비동기 복제 유지 + pt-heartbeat로 Lag 모니터링
  → Failover 절차를 자동화 도구로 빠르게

완전한 데이터 유실 불허 + 지연 허용 불가:
  → MySQL Group Replication (완전 동기)
  → Galera Cluster
  → 3개 이상 노드에서 과반수 동의 후 커밋
```

---

## 🔬 내부 동작 원리

### 1. Semi-Sync 동작 상세 흐름

```
After-Sync (Loss-less) 동작:

Source 쪽:
  ① 트랜잭션 실행
  ② 그룹 커밋 → Binary Log에 이벤트 기록 (flush/sync)
  ③ Semi-Sync Dump Thread: Replica에게 이벤트 전송 + ACK 대기
  ④ Replica의 ACK 수신 (또는 타임아웃)
  ⑤ InnoDB 엔진 커밋 완료 (다른 세션에서 보이기 시작)
  ⑥ 클라이언트에게 응답

Replica 쪽:
  ① IO Thread: Source로부터 이벤트 수신
  ② Relay Log 파일에 기록 + sync
  ③ Source에게 ACK 전송 ("Relay Log에 기록 완료")
  ④ SQL Thread: Relay Log 이벤트 재실행 (비동기)

핵심:
  ACK = "Relay Log에 기록됨" (fsync 완료)
  ACK ≠ "SQL Thread가 적용 완료"
  ACK 이후 Source 장애 → Relay Log 존재 → 복원 가능

그룹 커밋과 Semi-Sync 결합:
  여러 트랜잭션을 묶어 Binary Log에 기록
  묶음 전체에 대해 ACK 1번 수신
  → TPS 높을수록 그룹 커밋 효과 → 오버헤드 상대적 감소
```

### 2. 타임아웃과 자동 강등/복귀

```sql
-- 타임아웃 동작:
-- rpl_semi_sync_source_timeout = 1000 (1초)

-- 시나리오:
-- T=0: 트랜잭션 COMMIT 준비, Replica에 전송
-- T=0: Replica 네트워크 단절 또는 IO Thread 중단
-- T=1: ACK 1초 대기 후 타임아웃
-- T=1: Semi-Sync → 비동기로 강등
-- T=1: 클라이언트 응답 반환 (1초 지연 발생)

-- 강등 확인:
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_source_status';
-- OFF: 비동기로 강등됨

-- 강등 이후:
-- Rpl_semi_sync_source_no_tx: 비동기로 처리된 트랜잭션 수 증가
-- 이 기간 동안은 데이터 유실 위험 존재

-- 자동 복귀:
-- Replica IO Thread가 재연결 → ACK 전송 재개
-- Source: 자동으로 Semi-Sync 모드 복귀
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_source_status';
-- ON: 복귀 확인

-- 모니터링 핵심 지표:
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync%';
-- Rpl_semi_sync_source_status: ON/OFF
-- Rpl_semi_sync_source_no_tx: 비동기로 처리된 건수 (0이어야 이상적)
-- Rpl_semi_sync_source_net_avg_wait_time: 평균 ACK 대기 시간 (μs)
-- Rpl_semi_sync_source_yes_tx: Semi-Sync로 처리된 건수
```

### 3. 성능 영향 정량 분석

```
추가 지연 계산:
  Semi-Sync 추가 지연 = Replica ACK 왕복 시간 (RTT)
  
  같은 데이터센터 (RTT ~0.3ms):
    트랜잭션당 추가 지연: ~0.3ms
    1,000 TPS 환경: 순서 처리 시 1,000 × 0.3ms = 300ms
    실제: 그룹 커밋으로 묶음 처리 → 1번 ACK로 여러 트랜잭션 처리
    실제 영향: ~10% 지연 증가 수준

  다른 데이터센터 (RTT ~20ms):
    트랜잭션당 추가 지연: ~20ms
    100 TPS 환경에서도 영향 큼
    → rpl_semi_sync_source_timeout을 짧게, 빠른 강등 허용

그룹 커밋과 Semi-Sync:
  binlog_group_commit_sync_delay 설정으로
  더 많은 트랜잭션을 묶어 1번 ACK로 처리
  → TPS 높을수록 Semi-Sync 오버헤드 희석됨
  → 저TPS 환경에서 Semi-Sync 영향이 상대적으로 큼
```

### 4. Semi-Sync와 Failover 관계

```
Semi-Sync Failover 시나리오:

장애 시점: Source 다운
  Semi-Sync였다면:
    마지막 커밋들은 Relay Log에 ACK 확인됨
    → Failover 후 SQL Thread가 Relay Log 적용 완료
    → 데이터 보존

  비동기였다면:
    일부 커밋이 Relay Log에 없을 수 있음
    → Failover 후 일부 데이터 없음

Semi-Sync가 100% 보장하지 못하는 케이스:
  타임아웃 후 비동기로 강등된 기간의 트랜잭션
  → 이 기간 동안은 데이터 유실 가능
  
  Relay Log 수신 완료 후 SQL Thread 적용 전 Replica 장애
  → Relay Log 유실 가능 (sync_relay_log=1로 보완)

sync_relay_log 설정:
  SET GLOBAL sync_relay_log = 1;
  -- Relay Log 쓰기마다 fsync → 유실 방지
  -- I/O 오버헤드 증가 (SSD 환경에서는 영향 미미)
```

---

## 💻 실전 실험

### 실험 1: Semi-Sync 플러그인 설치 및 확인

```sql
-- Source에서:
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
SET GLOBAL rpl_semi_sync_source_enabled = ON;
SET GLOBAL rpl_semi_sync_source_timeout = 1000;  -- 1초
SET GLOBAL rpl_semi_sync_source_wait_point = AFTER_SYNC;  -- Loss-less

-- Replica에서:
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';
SET GLOBAL rpl_semi_sync_replica_enabled = ON;
STOP REPLICA; START REPLICA;  -- 재연결로 Semi-Sync 활성화

-- 상태 확인 (Source):
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync%';
-- Rpl_semi_sync_source_status: ON
-- Rpl_semi_sync_source_clients: 1 (연결된 Replica 수)
```

### 실험 2: 타임아웃 시뮬레이션

```sql
-- Replica에서 IO Thread 중단 (ACK 전송 불가 상태 만들기)
STOP REPLICA IO_THREAD;

-- Source에서 트랜잭션 실행
INSERT INTO test_table VALUES (1, 'test');
-- 1초 대기 후 완료 (타임아웃 후 비동기로 처리)

-- 강등 확인
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_source_status';
-- OFF (비동기로 강등)
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_source_no_tx';
-- 1 (비동기로 처리된 트랜잭션)

-- Replica IO Thread 재시작
-- (Replica에서)
START REPLICA IO_THREAD;

-- Source에서 자동 복귀 확인
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_source_status';
-- ON (복귀)
```

### 실험 3: After-Commit vs After-Sync 비교

```sql
-- After-Commit 설정
SET GLOBAL rpl_semi_sync_source_wait_point = AFTER_COMMIT;

-- 트랜잭션 실행 후 다른 세션에서 보이는 타이밍 확인
-- (1세션) BEGIN; UPDATE t SET v=99 WHERE id=1; COMMIT;
-- (2세션) LOOP: SELECT v FROM t WHERE id=1; -- 99 보이는 시점?
-- After-Commit: InnoDB 커밋 후 → ACK 전 다른 세션에서 보임

-- After-Sync 설정
SET GLOBAL rpl_semi_sync_source_wait_point = AFTER_SYNC;
-- After-Sync: ACK 수신 후 InnoDB 커밋 → 다른 세션에서 보임
-- 팬텀 커밋 없음
```

---

## 📊 성능/비용 비교

```
복제 모드별 성능 비교 (같은 데이터센터 기준):

비동기 복제:
  추가 지연: 0ms
  데이터 유실 위험: 있음
  클라이언트 응답 시간: 기준

Semi-Sync (After-Sync):
  추가 지연: ~RTT (1ms 내외, 같은 DC)
  데이터 유실 위험: 매우 낮음 (타임아웃 기간 제외)
  클라이언트 응답 시간: 기준 + ~1ms
  고TPS 환경 (그룹 커밋 효과): 오버헤드 희석

MySQL Group Replication (완전 동기):
  추가 지연: RTT × 왕복 (합의 프로토콜)
  데이터 유실 위험: 없음 (과반수 동의 후 커밋)
  클라이언트 응답 시간: 기준 + 수~수십 ms
  오버헤드: Semi-Sync보다 높음

선택 기준:
  금융/결제 데이터 → Semi-Sync After-Sync 최소한
  일반 서비스 데이터 → 비동기 + Lag 모니터링
  완전 유실 불허 + 지연 허용 → Group Replication
```

---

## ⚖️ 트레이드오프

```
Semi-Sync 도입 트레이드오프:

이점:
  ✅ 데이터 유실 위험 극적 감소
  ✅ After-Sync: 팬텀 커밋 없음 (InnoDB 커밋 전 ACK 확인)
  ✅ Failover 후 데이터 보존 확률 높음
  ✅ 비동기 대비 추가 설정 최소 (플러그인 설치 정도)

비용:
  ❌ 추가 지연 = 네트워크 RTT (같은 DC: ~1ms, 원거리: ~20ms)
  ❌ Replica 장애/지연 시 Source 응답 지연
  ❌ 타임아웃 설정이 너무 길면 장애 시 서비스 영향 큼
  ❌ 타임아웃 후 비동기 강등 기간은 여전히 유실 위험

타임아웃 설정 트레이드오프:
  짧게 (500ms):
    네트워크 일시 지연에도 빠르게 비동기 강등
    → 유실 위험 증가 but 서비스 응답 유지
  길게 (10초):
    Replica 장애 시 10초 응답 없음 → 서비스 사실상 중단
    → 유실 위험 감소 but 가용성 저하

권장 타임아웃:
  같은 데이터센터: 1,000~2,000ms
  다른 데이터센터: 500~1,000ms
  → Rpl_semi_sync_source_no_tx 모니터링으로 강등 빈도 확인
```

---

## 📌 핵심 정리

```
Semi-Sync 핵심:

목적:
  비동기 복제의 데이터 유실 위험 제거
  커밋된 트랜잭션이 최소 1 Replica의 Relay Log에 보장

After-Sync (Loss-less, MySQL 5.7+, 권장):
  Binary Log → Replica 전송 → ACK → InnoDB 커밋 → 응답
  팬텀 커밋 없음, 완전한 Loss-less

타임아웃 동작:
  rpl_semi_sync_source_timeout 초과 → 비동기 자동 강등
  Replica 복귀 시 Semi-Sync 자동 복귀
  강등 기간: 데이터 유실 위험 존재

성능 영향:
  같은 데이터센터: ~1ms 추가 (허용 가능)
  다른 데이터센터: ~10~50ms (서비스 특성 검토 필요)
  그룹 커밋 효과로 고TPS에서 오버헤드 희석

모니터링:
  Rpl_semi_sync_source_status: ON=정상
  Rpl_semi_sync_source_no_tx: 0이어야 이상적 (비동기 처리 건수)
  Rpl_semi_sync_source_net_avg_wait_time: ACK 평균 대기 시간
```

---

## 🤔 생각해볼 문제

**Q1.** Semi-Sync After-Sync를 사용해도 여전히 데이터가 유실될 수 있는 시나리오를 설명하라.

<details>
<summary>해설 보기</summary>

**시나리오 1: 타임아웃 후 비동기 강등 기간**
Replica 장애 또는 네트워크 단절 → 1초(기본) 대기 후 비동기로 강등 → 강등 기간 동안의 트랜잭션은 Replica에 없을 수 있음 → Source 장애 시 해당 트랜잭션 유실.

**시나리오 2: Relay Log 기록 후 Replica 장애**
ACK 전송 완료 (Relay Log에 기록) → SQL Thread 적용 전 Replica 서버 하드웨어 장애 → Relay Log 파일 손상 → 유실 가능.

방어: `sync_relay_log=1` (Relay Log 쓰기마다 fsync) + RAID 스토리지.

**시나리오 3: 유일한 Replica 장애**
Semi-Sync 환경에서 Replica가 1개이고 그 Replica가 장애 → 타임아웃 후 비동기 강등 → Source에서만 커밋 → 유실 위험.

방어: Replica를 2개 이상 두고 `rpl_semi_sync_source_wait_for_replica_count=1` (기본 1) 유지.

완전한 유실 방지를 위해서는 MySQL Group Replication (과반수 동의 후 커밋) 또는 Galera Cluster를 고려해야 합니다.

</details>

---

**Q2.** 다음 상황에서 `rpl_semi_sync_source_timeout`을 몇으로 설정하는 것이 적절한가? 이유를 설명하라.

```
환경:
  - 결제 서비스 DB (데이터 유실 불가)
  - Source와 Replica는 같은 데이터센터 (RTT ~0.5ms)
  - SLA: 트랜잭션 응답 시간 < 100ms
  - Replica는 1대
```

<details>
<summary>해설 보기</summary>

**권장 설정**: `rpl_semi_sync_source_timeout = 1000` (1초)

**이유**:
- 같은 데이터센터 RTT = 0.5ms → 정상 동작 시 ACK 1ms 이내 도착
- SLA 100ms 대비 1ms 오버헤드 = 1% 미만 → 허용 가능
- 타임아웃 1초: Replica 일시 장애 시 최대 1초 지연 → SLA 위반이지만 데이터 유실 방지 우선

**추가 고려사항**:
- Replica 1대 → 해당 Replica 장애 시 1초마다 비동기 강등 → `Rpl_semi_sync_source_no_tx` 알람 설정 필수
- 결제 데이터 유실 불가라면 Replica를 2대로 늘리는 것 권장 (1대 장애 시에도 나머지 1대로 Semi-Sync 유지)
- 타임아웃을 500ms로 줄이면 네트워크 일시 지연(패킷 재전송 등)에도 비동기 강등 위험 → 1초가 현실적 균형

```sql
SET GLOBAL rpl_semi_sync_source_timeout = 1000;
SET GLOBAL rpl_semi_sync_source_wait_point = AFTER_SYNC;  -- Loss-less

-- 모니터링 알람:
-- Rpl_semi_sync_source_no_tx > 0 → 즉시 알람 (비동기 처리 발생)
-- Rpl_semi_sync_source_status = OFF → 즉시 알람
```

</details>

---

<div align="center">

**[⬅️ Replication Lag](./04-replication-lag-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: 병렬 복제 ➡️](./06-parallel-replication.md)**

</div>
