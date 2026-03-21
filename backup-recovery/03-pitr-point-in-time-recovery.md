# Binary Log를 이용한 PITR — 특정 시각으로 복구

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Full Backup + Binary Log를 조합해 특정 시각으로 복구하는 전체 절차는?
- `mysqlbinlog --start-datetime` / `--stop-position` 옵션으로 원하는 구간만 재생하는 방법은?
- GTID 환경에서 PITR 수행 시 주의사항은?
- 복구하려는 트랜잭션 바로 직전에 정확히 멈추는 방법은?
- Binary Log가 없거나 삭제된 경우 PITR이 불가능한 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### "실수로 DROP TABLE 했습니다" — PITR만이 답이다

```
실제 장애 시나리오:
  오전 10:30 개발자 실수: DROP TABLE orders;
  오전 10:35 이상 감지, DB 복구 요청
  최근 Full Backup: 전날 새벽 2:00

  Full Backup만 복원하면:
    어제 2시 이후 8시간 30분치 데이터 유실
    → 허용 불가

  PITR 수행:
    1. Full Backup 복원 (어제 2시 상태)
    2. Binary Log를 10:29까지 재생
       (DROP TABLE 바로 직전에 멈춤)
    3. 결과: 10:30 직전 상태로 완전 복구

  PITR의 전제 조건:
    Binary Log가 보존되어 있어야 함
    Full Backup에 Binary Log 위치 정보 포함 (--master-data)
    복구 목표 시점 또는 위치 특정 가능해야 함
```

---

## 😱 흔한 실수 (Before)

### 1. Binary Log를 삭제하여 PITR 불가

```bash
# Before: 디스크 절약을 위해 Binary Log 즉시 삭제
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 1 DAY);
# 또는 my.cnf에:
# expire_logs_days = 1  (1일만 보관)

# 문제:
# 어제 백업 후 오늘 장애 발생
# Binary Log가 1일치만 있어 복구 가능하지만
# 백업이 이틀 전이라면 하루치 Binary Log가 없음
# → Full Backup만 복원 가능, PITR 불가

# 올바른 설정:
# binlog_expire_logs_seconds = 604800  # 7일 보관
# Full Backup 주기보다 Binary Log 보관 기간이 길어야 함
```

### 2. 복구 목표 시점을 잘못 지정 (DROP TABLE 포함)

```bash
# Before: DROP TABLE이 발생한 시각을 포함하여 재생
mysqlbinlog --stop-datetime="2024-01-15 10:30:00" \
  mysql-bin.000005 | mysql -u root -p mydb

# 문제:
# DROP TABLE이 10:30:00에 실행됐다면
# 10:30:00을 포함하므로 DROP TABLE도 재실행됨!
# → 복원 효과 없음

# 올바른 접근:
# DROP TABLE의 정확한 위치(Position)를 먼저 찾은 후
# 그 직전 위치(--stop-position)까지만 재생
```

### 3. GTID 환경에서 잘못된 PITR 수행

```bash
# Before: GTID 환경에서 Binary Log를 직접 pipe로 넣음
mysqlbinlog --stop-datetime="2024-01-15 10:29:00" \
  mysql-bin.000005 | mysql -u root -p mydb
# Error: @@GLOBAL.GTID_MODE = ON일 때 이 방식은 오류 발생

# 이유:
# Full Backup 복원 시 GTID_PURGED가 설정됨
# Binary Log 재생 시 이미 있는 GTID와 충돌

# 올바른 방법:
mysqlbinlog --skip-gtids \
  --stop-datetime="2024-01-15 10:29:00" \
  mysql-bin.000005 | mysql -u root -p mydb
# --skip-gtids: Binary Log의 GTID 이벤트를 무시하고 재생
```

---

## ✨ 올바른 접근 (After)

```
PITR 전체 절차:

전제 조건 확인:
  ① Full Backup 파일 존재 확인
  ② Full Backup의 Binary Log 위치 정보 확인
     (mysqldump: --master-data, XtraBackup: xtrabackup_binlog_info)
  ③ Binary Log 파일들이 보존되어 있는지 확인
  ④ 복구 목표 시점 또는 위치 결정

단계별 절차:
  1단계: Full Backup 복원
     (별도 복구 서버에서 수행, 운영 서버 영향 없음)

  2단계: 복구 목표 직전의 Binary Log 위치 확인
     mysqlbinlog --start-position=X mysql-bin.000005 | \
       grep -i "drop table"
     → 정확한 Position 확인

  3단계: Binary Log 재생 (Full Backup 이후 ~ 목표 직전)
     mysqlbinlog \
       --start-position=[Full Backup 시점 위치] \
       --stop-position=[Drop Table 직전 위치] \
       --skip-gtids \    # GTID 환경에서
       mysql-bin.000005 | mysql -u root -p

  4단계: 복구 검증
     복구된 데이터 확인 후 운영 서버로 반영
```

---

## 🔬 내부 동작 원리

### 1. Binary Log와 PITR의 관계

```
Full Backup의 역할:
  특정 시점 T0의 DB 상태를 저장
  T0 = mysqldump 시작 시점 또는 XtraBackup 백업 시점

Binary Log의 역할:
  T0 이후의 모든 트랜잭션을 순서대로 기록
  각 트랜잭션의 타임스탬프, 위치(Position) 포함

PITR 원리:
  T0 상태 복원 → T0+1 트랜잭션 재실행 → T0+2 재실행 → ...
  → T_target 직전 트랜잭션까지 재실행
  = T_target 직전 상태 복원

Binary Log 이벤트 구조:
  각 트랜잭션:
    Gtid_log_event (GTID 환경)
    Query_log_event (BEGIN)
    Table_map_event
    Rows_log_event (INSERT/UPDATE/DELETE)
    Xid_log_event (COMMIT)

  각 이벤트마다:
    타임스탬프: 원본 실행 시간
    로그 위치: 파일 내 바이트 오프셋
```

### 2. 복구 목표 위치 찾기

```bash
# Binary Log에서 특정 SQL 위치 찾기

# 방법 1: mysqlbinlog로 이벤트 출력
mysqlbinlog --base64-output=DECODE-ROWS -v mysql-bin.000005 | \
  grep -i -B 10 "DROP TABLE"
# -B 10: 매칭 10줄 전 출력 (위치 정보 포함)

# 출력 예:
# # at 1234567                    ← 이 위치가 DROP TABLE의 Position
# #240115 10:30:02 server id 1    ← 타임스탬프
# COMMIT/*!*/;
# # at 1234700
# #240115 10:30:05 server id 1
# DROP TABLE `orders`

# → DROP TABLE의 Position: 1234700
# → 복구는 1234567까지 (DROP TABLE 직전 COMMIT 완료 후)

# 방법 2: 타임스탬프로 범위 확인
mysqlbinlog --start-datetime="2024-01-15 10:28:00" \
            --stop-datetime="2024-01-15 10:32:00" \
  mysql-bin.000005 | head -100
# 해당 시간대의 이벤트 목록 확인

# 방법 3: GTID로 특정 트랜잭션 찾기
mysqlbinlog mysql-bin.000005 | grep -A 5 "DROP TABLE"
# GTID 번호 확인
```

### 3. mysqlbinlog 옵션 상세

```bash
# 주요 옵션:

# --start-datetime: 이 시각 이후 이벤트부터 재생
# --stop-datetime: 이 시각 이전 이벤트까지만 재생 (이 시각 포함 안 됨)
mysqlbinlog --start-datetime="2024-01-15 02:00:00" \
            --stop-datetime="2024-01-15 10:30:00" \
  mysql-bin.000005

# --start-position: 이 위치부터 이벤트 재생
# --stop-position: 이 위치까지만 재생 (이 위치 포함 안 됨)
mysqlbinlog --start-position=1234567 \
            --stop-position=1234699 \  # DROP TABLE 직전 COMMIT 위치
  mysql-bin.000005

# --skip-gtids: GTID 이벤트 무시 (GTID 환경 PITR 필수)
mysqlbinlog --skip-gtids \
            --start-position=1234567 \
            --stop-position=1234699 \
  mysql-bin.000005 | mysql -u root -p mydb_restored

# 여러 Binary Log 파일 연속 재생:
mysqlbinlog --start-position=1234567 \
  mysql-bin.000005 mysql-bin.000006 mysql-bin.000007 | \
  mysql -u root -p mydb_restored

# --read-from-remote-server: 원격 서버의 Binary Log 직접 읽기
mysqlbinlog --read-from-remote-server \
  --host=source-server --user=repl \
  --raw mysql-bin.000005  # 바이너리 그대로 저장
```

### 4. GTID 환경 PITR 주의사항

```bash
# GTID 환경 PITR 전체 절차:

# 1. Full Backup 복원 (mysqldump 기준)
mysql -u root -p < full_backup.sql
# 복원 완료 후 GTID_PURGED에 Full Backup 시점 GTID 설정됨

# 2. GTID_PURGED 확인
mysql -u root -p -e "SELECT @@GLOBAL.GTID_PURGED;"
# 예: 3E11FA47-...:1-5000 (Full Backup에 포함된 GTID)

# 3. Binary Log에서 DROP TABLE GTID 확인
mysqlbinlog mysql-bin.000005 | grep -B 5 "DROP TABLE"
# 예: # GTID 3E11FA47-...:5001 (Full Backup 이후 첫 트랜잭션)
#     # GTID 3E11FA47-...:5050 (DROP TABLE)

# 4. DROP TABLE 직전까지 Binary Log 재생
mysqlbinlog --skip-gtids \
  --stop-position=1234699 \  # DROP TABLE 직전
  mysql-bin.000005 | mysql -u root -p

# --skip-gtids가 필수인 이유:
# Full Backup 복원 후 GTID_PURGED = :1-5000
# Binary Log에는 :5001~ 트랜잭션이 있음
# GTID 없이 재생하면 MySQL이 "이미 처리한 GTID" 충돌 검사 생략
# 올바른 재생 가능

# 주의: --skip-gtids 없이 재생 시
# ERROR 3546: @@GLOBAL.GTID_MODE = ON ... GTID conflict
```

### 5. 복구 검증

```sql
-- 복구 후 데이터 검증
-- 복구된 임시 서버에서 확인
SELECT COUNT(*) FROM orders;  -- 예상 건수와 비교
SELECT MAX(created_at) FROM orders;  -- 최신 데이터 타임스탬프 확인

-- 중요 데이터 샘플 확인
SELECT * FROM orders ORDER BY id DESC LIMIT 10;
-- DROP TABLE 직전까지의 데이터가 있는지 확인

-- 복구 완료 후 운영 DB 반영 방법:
-- 1. 전체 복원: 운영 서버에 동일하게 복원 절차 수행
-- 2. 선택적 복원: 복구된 서버에서 특정 테이블만 mysqldump → 운영 서버 적용
-- 3. 차이 데이터만 적용: pt-table-sync로 차이 데이터 동기화
```

---

## 💻 실전 실험

### 실험 1: DROP TABLE 시뮬레이션 및 PITR

```bash
# 환경 준비
mysql -u root -p -e "
CREATE DATABASE pitr_test;
USE pitr_test;
CREATE TABLE orders (id INT AUTO_INCREMENT PRIMARY KEY, val VARCHAR(50));
INSERT INTO orders (val) VALUES ('order1'), ('order2'), ('order3');
COMMIT;
"

# Full Backup
mysqldump --single-transaction --master-data=2 \
  pitr_test > /tmp/pitr_test_full.sql

# 백업 후 추가 데이터
mysql -u root -p -e "
USE pitr_test;
INSERT INTO orders (val) VALUES ('order4'), ('order5');
COMMIT;
"

# 실수로 DROP TABLE
mysql -u root -p -e "USE pitr_test; DROP TABLE orders;"

# 현재 Binary Log 위치 확인
mysql -u root -p -e "SHOW BINARY LOG STATUS\G"

# Full Backup의 Binary Log 위치 확인
grep "CHANGE MASTER\|CHANGE REPLICATION" /tmp/pitr_test_full.sql

# Binary Log에서 DROP TABLE 위치 찾기
mysqlbinlog --base64-output=DECODE-ROWS -v \
  /var/lib/mysql/mysql-bin.* 2>/dev/null | \
  grep -i -B 5 "drop table"

# PITR 수행
# 1. 데이터베이스 재생성
mysql -u root -p -e "DROP DATABASE IF EXISTS pitr_test;"
mysql -u root -p < /tmp/pitr_test_full.sql

# 2. DROP TABLE 직전까지 Binary Log 재생
# STOP POSITION을 DROP TABLE 직전 COMMIT 위치로 설정
mysqlbinlog --skip-gtids \
  --start-position=[Full Backup Position] \
  --stop-position=[DROP TABLE 직전 Position] \
  /var/lib/mysql/mysql-bin.000005 | mysql -u root -p

# 3. 복구 확인
mysql -u root -p -e "SELECT * FROM pitr_test.orders;"
# order1~5 모두 있어야 함
```

### 실험 2: 타임스탬프 기반 PITR

```bash
# 현재 시간 기록
echo "NOW: $(date '+%Y-%m-%d %H:%M:%S')"

# 데이터 추가
mysql -u root -p -e "INSERT INTO pitr_test.orders (val) VALUES ('order6');"

# 5초 대기
sleep 5

# 실수 DELETE
mysql -u root -p -e "DELETE FROM pitr_test.orders WHERE id > 3;"
MISTAKE_TIME=$(date '+%Y-%m-%d %H:%M:%S')
echo "MISTAKE TIME: $MISTAKE_TIME"

# 타임스탬프 기반 복구
# DELETE 직전까지 재생 (--stop-datetime에 DELETE 시각 미포함)
mysqlbinlog --skip-gtids \
  --stop-datetime="$MISTAKE_TIME" \
  /var/lib/mysql/mysql-bin.000005 | mysql -u root -p

# 주의: --stop-datetime은 해당 시각 이전까지만 재생
# 정확한 초 단위 타이밍이 불확실하면 --stop-position 사용 권장
```

---

## 📊 성능/비용 비교

```
PITR 복구 시간 요소:

Full Backup 복원 시간:
  mysqldump 100GB: 6~15시간
  XtraBackup 100GB: 30~60분
  → XtraBackup 사용 시 복원 기반 시간 절감

Binary Log 재생 시간:
  재생 속도: 초당 수천~수만 트랜잭션
  24시간치 Binary Log (500 TPS): ~4,000만 트랜잭션
  재생 시간: 30분~2시간 (트랜잭션 복잡도에 따라)

총 PITR 시간:
  mysqldump + Binary Log 재생: 7~17시간 (100GB 기준)
  XtraBackup + Binary Log 재생: 1~3시간
  → XtraBackup 기반 PITR의 압도적 우위

Binary Log 보관 비용:
  500 TPS, 1KB/트랜잭션 기준:
  일일 Binary Log: 500 × 86400 × 1KB = ~43GB/일
  7일 보관: ~300GB
  30일 보관: ~1.3TB
  → 스토리지 비용 vs RPO 요구사항 균형
```

---

## ⚖️ 트레이드오프

```
PITR 운영 트레이드오프:

Binary Log 보관 기간:
  길게 (30일):
    ✅ 오래된 시점으로 복구 가능
    ✅ 실수 발견이 늦어도 복구 가능
    ❌ 스토리지 비용 증가
    ❌ 장기 Binary Log 재생 시간 증가

  짧게 (1~7일):
    ✅ 스토리지 절약
    ❌ 오래된 시점 복구 불가
    ❌ Full Backup 주기보다 짧으면 PITR 불가 기간 발생

설계 원칙:
  Binary Log 보관 기간 > Full Backup 주기
  Full Backup: 매일 → Binary Log: 최소 3~7일
  Full Backup: 매주 → Binary Log: 최소 14일

위치 기반 vs 시간 기반 PITR:
  --stop-position (권장):
    정확한 트랜잭션 경계 제어
    DROP TABLE 직전 COMMIT까지 정확히 멈춤
    Binary Log 분석 선행 필요

  --stop-datetime:
    사람이 이해하기 쉬움
    초 단위 오차 가능 (같은 초에 여러 트랜잭션)
    위험한 트랜잭션이 경계 시각에 있으면 포함될 수 있음

복구 서버 분리:
  ✅ 운영 서버 영향 없이 복구 테스트 가능
  ✅ 복구 검증 후 운영 반영 결정
  ❌ 복구 서버 준비 및 유지 비용
```

---

## 📌 핵심 정리

```
PITR 핵심:

전제 조건:
  Full Backup + Binary Log 보관 (기간: Full Backup 주기의 2배 이상)
  Full Backup에 Binary Log 위치 정보 포함 (--master-data=2 또는 xtrabackup_binlog_info)

절차:
  1. Full Backup 복원 (복구 서버에서)
  2. Binary Log에서 복구 목표 위치 찾기 (mysqlbinlog | grep 대상)
  3. Full Backup 이후 ~ 목표 직전까지 Binary Log 재생
  4. 데이터 검증 후 운영 반영

핵심 옵션:
  --stop-position: 위치 기반 (정확, 권장)
  --stop-datetime: 시간 기반 (편리하지만 오차 가능)
  --skip-gtids: GTID 환경 PITR 필수

GTID 환경:
  --skip-gtids 필수 (GTID 충돌 방지)
  Full Backup 복원 시 GTID_PURGED 자동 설정됨

Binary Log 관리:
  binlog_expire_logs_seconds: 최소 Full Backup 주기 × 2
  삭제 전 원격 서버 아카이브 권장
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 시나리오에서 PITR이 가능한지 판단하고, 가능하다면 정확한 복구 절차를 설명하라.

```
상황:
  - 매주 일요일 새벽 2시 XtraBackup Full Backup
  - Binary Log: 14일 보관
  - 금요일 오후 3시: 실수로 UPDATE orders SET amount = 0 WHERE 1=1;
  - 금요일 오후 3시 30분에 발견
  - 현재: 금요일 오후 3시 30분
```

<details>
<summary>해설 보기</summary>

**PITR 가능** 합니다.

**가용한 백업 자료**:
- Full Backup: 지난 일요일 새벽 2시
- Binary Log: 14일 보관 → 일요일 이후 오늘까지 모두 존재

**복구 절차**:

```bash
# 1. 별도 복구 서버에 Full Backup 복원
xtrabackup --prepare --target-dir=/backup/full_sunday/
xtrabackup --copy-back --target-dir=/backup/full_sunday/
chown -R mysql:mysql /var/lib/mysql/
systemctl start mysql

# 2. Full Backup 이후 Binary Log 파일 목록 확인
cat /backup/full_sunday/xtrabackup_binlog_info
# 예: mysql-bin.000005  1234567

# 3. 잘못된 UPDATE 위치 찾기
mysqlbinlog --base64-output=DECODE-ROWS -v \
  mysql-bin.000005 ~ mysql-bin.000010 | \
  grep -i -B 5 "UPDATE orders SET amount"
# 정확한 Position 확인 (예: 9876543)

# 4. Binary Log 재생 (일요일 2시 이후 ~ UPDATE 직전)
mysqlbinlog --skip-gtids \
  --start-position=1234567 \
  --stop-position=9876542 \  # UPDATE 직전 COMMIT 위치
  mysql-bin.000005 mysql-bin.000006 ... mysql-bin.000010 | \
  mysql -u root -p

# 5. 복구 검증
SELECT COUNT(*) FROM orders WHERE amount = 0;
# 0이어야 함 (잘못된 UPDATE 포함 안 됨)

# 6. 운영 서버 반영 결정
# 전체 교체 or orders 테이블만 mysqldump → 운영 적용
```

**RPO**: 금요일 오후 3시 직전 상태로 복구 = 약 30분치 데이터 보존

</details>

---

**Q2.** GTID 환경에서 PITR 시 `--skip-gtids`를 사용하면 GTID 일관성이 깨지는 문제가 발생할 수 있다. 이를 해결하는 방법을 설명하라.

<details>
<summary>해설 보기</summary>

**문제**: `--skip-gtids`로 재생하면 이 트랜잭션들은 GTID가 없는 형태로 복원 서버에 적용됩니다. 이후 이 서버를 Replica로 사용하려 하면 GTID_EXECUTED와 실제 데이터 상태가 불일치할 수 있습니다.

**해결 방법**:

1. **복구 전용 서버로만 사용**: PITR 복구 서버를 복제 체계에 편입하지 않고 복구 목적으로만 사용. 데이터 검증 후 필요한 부분만 mysqldump로 운영 서버에 이관.

2. **GTID_PURGED 수동 설정**: PITR 완료 후 올바른 GTID_PURGED 설정
```sql
-- --skip-gtids로 재생한 후
-- 재생된 마지막 Binary Log의 GTID까지 PURGED로 설정
SET GLOBAL GTID_PURGED = 'server_uuid:1-5100';
-- 이후 Replica로 편입 가능
```

3. **Binary Log 재활성화 후 GTID 재부여**: PITR 완료 서버에서 Binary Log를 새로 생성하면 새 GTID가 자동 할당됩니다.

실무에서는 방법 1이 가장 안전합니다. 복구 전용 서버에서 검증 후 운영 서버에 mysqldump로 특정 테이블만 이관하는 방식을 권장합니다.

</details>

---

<div align="center">

**[⬅️ XtraBackup](./02-xtrabackup-hot-backup.md)** | **[홈으로 🏠](../README.md)** | **[다음: 복구 전략 설계 ➡️](./04-recovery-strategy-rto-rpo.md)**

</div>
