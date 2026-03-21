# 실전 장애 복구 시나리오 — 실수로 DELETE된 테이블 복구 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `DROP TABLE` 또는 대량 `DELETE` 이후 복구 가능한 범위와 불가능한 범위는?
- Full Backup → Binary Log 재생 → 문제 트랜잭션 직전에 멈추는 전체 과정은?
- 복구 시간을 단축하기 위해 사전에 준비해야 할 것은?
- 복구 중 실수로 2차 피해를 방지하는 방법은?
- 복구 후 운영 서버에 안전하게 반영하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 장애 복구는 연습한 만큼 빠르다

```
실제 장애 시나리오:
  금요일 오후 2시: DBA가 실수로 실행
    DELETE FROM orders WHERE 1=1;
    (WHERE 조건 실수, 전체 삭제)
  오후 2:03: 모니터링 알림 "orders 테이블 0건"
  오후 2:05: 복구 시작

  복구 리허설 없던 팀:
    Binary Log 찾기: 30분
    복구 서버 준비: 1시간
    절차 확인/시행착오: 2시간
    총 복구 시간: 4~5시간 (데이터 손실 2~3시간치)

  복구 리허설 있던 팀:
    Binary Log 위치 즉시 확인: 5분
    복구 서버 준비 (자동화): 15분
    Full Backup 복원: 30분
    Binary Log 재생: 20분
    총 복구 시간: 1~1.5시간 (데이터 손실 0)

차이: 4배의 복구 속도 = 3시간 더 빠른 서비스 복구
```

---

## 😱 흔한 실수 (Before)

### 1. 복구 전에 운영 서버에서 작업 시도

```bash
# Before: 운영 서버에서 직접 복구 시도
# (절대 하면 안 되는 것)

# 운영 서버에서:
mysql -u root -p mydb < full_backup.sql
# 문제 1: 운영 서비스가 DB에 계속 쓰기 중
#         복원 중 데이터 충돌 발생
# 문제 2: 복원 실패 시 원래 있던 데이터도 사라질 수 있음
# 문제 3: 다른 테이블까지 덮어쓰여 2차 피해

# 올바른 원칙:
# 항상 별도 복구 서버(Restore Server)에서 먼저 복원
# 검증 완료 후 운영 서버에 반영
```

### 2. 복구 목표 시점 이후까지 Binary Log 재생

```bash
# Before: 시간 기반으로 대충 지정
mysqlbinlog --stop-datetime="2024-01-19 14:02:00" \
  mysql-bin.000010 | mysql -u root -p mydb

# 문제:
# DELETE가 14:02:00에 실행됐으면 포함됨
# 또는 다른 중요 트랜잭션을 건너뛸 수 있음
# → 항상 정확한 Position 확인 후 --stop-position 사용

# 올바른 방법:
# 먼저 Binary Log에서 DELETE 위치를 정확히 찾기
mysqlbinlog --base64-output=DECODE-ROWS -v \
  mysql-bin.000010 | grep -i -B 10 "DELETE FROM orders"
# Position 정확히 확인 후 --stop-position 지정
```

### 3. InnoDB_force_recovery 남용

```bash
# Before: DB가 안 뜰 때 무작정 높은 값 설정
# my.cnf: innodb_force_recovery = 6  # 가장 높은 값

# 문제:
# force_recovery = 4 이상: 데이터 쓰기 차단 (SELECT만 가능)
# force_recovery = 6: InnoDB가 손상된 페이지 무시하고 강제 기동
#   → 일부 데이터가 누락된 상태로 시작
#   → 이 상태에서 mysqldump 후 복원하면 불완전한 데이터

# 올바른 접근:
# 1 → 2 → 3 → 4 → 5 → 6 순서로 낮은 값부터 시도
# 기동 가능한 가장 낮은 값 사용
# 즉시 mysqldump로 데이터 추출 후 정상 복원
```

---

## ✨ 올바른 접근 (After)

```
장애 복구 기본 원칙:

원칙 1: 냉정함 유지
  패닉 상태에서 추가 실수 발생 → 2차 피해
  먼저 현황 파악, 팀에 공유, 역할 분담

원칙 2: 운영 서버 격리 우선
  추가 피해 방지를 위해 문제 서버 격리 검토
  복구는 항상 별도 서버에서 먼저

원칙 3: Binary Log 즉시 보존
  장애 직후 현재 Binary Log 위치 기록
  Binary Log 파일 삭제/덮어쓰기 방지

원칙 4: 단계적 복구, 검증 후 반영
  복구 서버에서 완전 검증
  문제없으면 운영 서버에 반영

원칙 5: 복구 과정 문서화
  타임라인, 수행한 명령어, 결과 기록
  사후 회고에서 재발 방지책 수립
```

---

## 🔬 내부 동작 원리

### 1. 복구 가능 범위와 불가능 범위

```
복구 가능:
  ① Binary Log가 있는 경우:
     마지막 Full Backup 이후부터 문제 발생 직전까지
     Full Backup + Binary Log 재생으로 복구

  ② Replica가 있는 경우:
     Replica에서 데이터 추출 가능
     (Replica Lag이 없었다면 문제 발생 전 상태)

  ③ Flashback (ROW 포맷 Binary Log):
     DELETE된 Row를 Binary Log에서 역순으로 복원
     percona/mariadb-flashback 도구 사용

복구 불가능:
  ① Binary Log가 없거나 삭제된 경우:
     마지막 Full Backup 시점까지만 복구
     이후 데이터는 영구 유실

  ② Full Backup도 없는 경우:
     복구 불가

  ③ Binary Log 포맷이 STATEMENT이고 특정 케이스:
     Flashback 불가 (ROW 포맷만 지원)

  ④ 테이블스페이스 삭제/암호화 키 손실:
     물리 파일 기반 복구 불가
```

### 2. 전체 복구 절차 (시나리오: DELETE FROM orders WHERE 1=1)

```bash
# ==========================================
# 0단계: 즉각 대응 (장애 발견 후 2분 이내)
# ==========================================

# 1. 현재 Binary Log 위치 즉시 기록 (원격으로 확인)
mysql -h prod-db -u readonly -p \
  -e "SHOW BINARY LOG STATUS\G" > /tmp/current_binlog_pos.txt
cat /tmp/current_binlog_pos.txt

# 2. Binary Log 파일 보존 (삭제 방지)
# 운영 서버에서 Binary Log 자동 삭제 설정이 있다면 임시 비활성화
mysql -h prod-db -u root -p \
  -e "SET GLOBAL binlog_expire_logs_seconds = 0;"

# 3. 팀 공유: Slack/Email로 상황 전파
echo "🚨 DB 장애 발생. orders 테이블 전체 삭제 확인. 복구 작업 시작." | \
  slack-notify #db-incident

# ==========================================
# 1단계: 복구 서버 준비 (10분)
# ==========================================

# 복구 서버 시작 (별도 서버 또는 임시 EC2)
# 같은 MySQL 버전, 충분한 디스크 공간 확인

# ==========================================
# 2단계: Full Backup 복원 (30~60분, XtraBackup 기준)
# ==========================================

# Full Backup 파일 다운로드 (S3에서)
aws s3 sync s3://backup-bucket/full_20240119/ /backup/restore/

# XtraBackup --prepare
xtrabackup --prepare --target-dir=/backup/restore/

# 복원
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backup/restore/
chown -R mysql:mysql /var/lib/mysql/
systemctl start mysql

# Full Backup 시점 확인
cat /backup/restore/xtrabackup_binlog_info
# mysql-bin.000010    5678901  (Full Backup 기준 Binary Log 위치)

# ==========================================
# 3단계: Binary Log에서 DELETE 위치 찾기 (5~10분)
# ==========================================

# Binary Log 파일 다운로드
# (운영 서버에서 복구 서버로 복사)
rsync -av prod-db:/var/log/mysql/mysql-bin.000010 /backup/binlogs/

# DELETE 트랜잭션 위치 찾기
mysqlbinlog --base64-output=DECODE-ROWS -v \
  /backup/binlogs/mysql-bin.000010 | \
  grep -i -B 15 "DELETE FROM .orders"

# 출력 예시:
# # at 7890123           ← 이 Position이 DELETE 트랜잭션 시작
# #240119 14:02:03 server id 1
# GTID ...
# ...
# # at 7890100           ← 바로 이전 COMMIT (이 직전까지 재생해야 함)
# DELETE FROM `orders` WHERE 1=1

# DELETE 직전 COMMIT Position: 7890099 (이 위치까지 재생)
DELETE_POSITION=7890099

# ==========================================
# 4단계: Binary Log 재생 (20~30분)
# ==========================================

# Full Backup 이후부터 DELETE 직전까지 재생
mysqlbinlog \
  --skip-gtids \
  --start-position=5678901 \
  --stop-position=$DELETE_POSITION \
  /backup/binlogs/mysql-bin.000010 | \
  mysql -u root -p

# ==========================================
# 5단계: 복구 검증 (10분)
# ==========================================

# orders 건수 확인
mysql -u root -p -e "SELECT COUNT(*) FROM mydb.orders;"
# 예상 건수: 100,000건 이상

# 최신 데이터 확인 (DELETE 직전 데이터 있는지)
mysql -u root -p -e "
SELECT MAX(created_at), MIN(created_at), COUNT(*)
FROM mydb.orders;
"
# MAX(created_at)이 14:02 이전이어야 함

# 관련 테이블 정합성 확인
mysql -u root -p -e "
SELECT o.id FROM mydb.orders o
LEFT JOIN mydb.order_items oi ON oi.order_id = o.id
WHERE oi.id IS NULL
LIMIT 5;
"
# 결과 없어야 함 (FK 무결성)

# ==========================================
# 6단계: 운영 서버에 반영 (15~30분)
# ==========================================

# 방법 1: 복구된 orders 테이블만 mysqldump → 운영 서버 적용
mysqldump -u root -p mydb orders > /tmp/orders_recovered.sql
# 운영 서버에서:
mysql -h prod-db -u root -p mydb < /tmp/orders_recovered.sql

# 방법 2: pt-table-sync (차이 데이터만 동기화)
pt-table-sync \
  --execute \
  h=restore-server,D=mydb,t=orders \
  h=prod-db,D=mydb,t=orders

echo "✅ 복구 완료!" | slack-notify #db-incident
```

### 3. Flashback (Binary Log ROW 포맷 역재생)

```bash
# Flashback: DELETE된 Row를 Binary Log에서 역순으로 복원
# 도구: MyFlash (Alibaba), binlog_rollback

# 전제 조건:
# binlog_format = ROW
# binlog_row_image = FULL

# MyFlash 사용 예:
# flashback --binlogFileNames=mysql-bin.000010 \
#   --tables=orders \
#   --startPosition=7890100 \
#   --stopPosition=7890200 \
#   --sqlType=DELETE \
#   > /tmp/flashback.sql

# flashback.sql 내용:
# (DELETE된 Row들의 INSERT문 = Undo 효과)
# INSERT INTO orders VALUES (1, ...);
# INSERT INTO orders VALUES (2, ...);
# ...

# 적용:
# mysql -u root -p mydb < /tmp/flashback.sql

# 장점:
#   Full Backup 복원 불필요 → 빠른 복구 (수 분)
#   기타 테이블에 영향 없음

# 한계:
#   ROW 포맷 + FULL 이미지만 지원
#   STATEMENT 포맷 불가
#   복잡한 종속 데이터 복구 어려움
```

### 4. Replica에서 데이터 복구

```bash
# Replica가 있고 Lag이 없었다면:
# 장애 직후 즉시 Replica SQL Thread 중단!
STOP REPLICA SQL_THREAD;  # Replica에서

# 이 시점의 Replica = 장애 직전 상태
# Replica에서 데이터 추출
mysqldump -u root -p \
  --single-transaction \
  mydb orders > /tmp/orders_from_replica.sql

# 운영 서버에 적용
mysql -h prod-db -u root -p mydb < /tmp/orders_from_replica.sql

# Replica 복제 재시작
START REPLICA SQL_THREAD;

# 주의:
# Replica Lag이 있었다면 Lag만큼의 데이터 없음
# Lag 확인: SHOW REPLICA STATUS\G → Seconds_Behind_Source
```

### 5. 복구 후 재발 방지

```sql
-- 위험 쿼리 실행 방지 (sql_safe_updates):
SET GLOBAL sql_safe_updates = 1;
-- WHERE 조건 없는 UPDATE/DELETE 차단
-- 애플리케이션에서는 WHERE 조건 있는 쿼리만 허용

-- 특정 사용자의 DELETE 권한 제거:
REVOKE DELETE ON mydb.orders FROM 'app_user'@'%';
-- 애플리케이션 사용자는 DELETE 불필요한 경우

-- Soft Delete 패턴 도입:
-- DELETE 대신 is_deleted = 1로 표시
ALTER TABLE orders ADD COLUMN is_deleted TINYINT DEFAULT 0;
-- DELETE 쿼리를 UPDATE is_deleted=1로 변경
-- → 실수 DELETE 시 UPDATE로 복구 가능

-- 변경 전 확인 쿼리 습관화:
-- DELETE 전:
SELECT COUNT(*) FROM orders WHERE 1=1;
-- 예상 건수 확인 후 DELETE 실행

-- DML 감사 로그 (Audit Log):
-- MySQL Enterprise Audit Plugin 또는
-- Percona Audit Log Plugin으로 모든 DML 기록
-- → 누가 언제 어떤 쿼리를 실행했는지 추적 가능
```

---

## 💻 실전 실험

### 실험 1: DROP TABLE 복구 실습 (안전 환경)

```bash
# 테스트 환경에서만 수행!

# 환경 준비
mysql -u root -p -e "
CREATE DATABASE recovery_test;
USE recovery_test;
CREATE TABLE test_orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT, amount DECIMAL(10,2),
    created_at DATETIME DEFAULT NOW()
);
INSERT INTO test_orders (user_id, amount) VALUES
(1,1000), (2,2000), (3,3000), (4,4000), (5,5000);
"

# Full Backup
mysqldump --single-transaction --master-data=2 \
  recovery_test > /tmp/recovery_test_full.sql
echo "Full Backup 완료"

# 추가 데이터
mysql -u root -p -e "
USE recovery_test;
INSERT INTO test_orders (user_id, amount) VALUES (6,6000),(7,7000);
"

# 실수 DROP TABLE
mysql -u root -p -e "USE recovery_test; DROP TABLE test_orders;"
echo "DROP TABLE 실행됨!"

# 현재 Binary Log 위치
mysql -u root -p -e "SHOW BINARY LOG STATUS\G"

# Full Backup Binary Log 위치 확인
grep "CHANGE MASTER\|CHANGE REPLICATION" /tmp/recovery_test_full.sql

# DROP TABLE 위치 찾기
mysqlbinlog --base64-output=DECODE-ROWS -v \
  $(mysql -u root -p -e "SHOW BINARY LOGS" | tail -1 | awk '{print "/var/lib/mysql/"$1}') | \
  grep -i -B 5 "DROP TABLE"

# 복구 수행 (DB 재생성)
mysql -u root -p -e "DROP DATABASE IF EXISTS recovery_test;"
mysql -u root -p < /tmp/recovery_test_full.sql

# Binary Log 재생 (DROP TABLE 직전까지)
# [--stop-position에 DROP TABLE 직전 Position 입력]
mysqlbinlog --skip-gtids \
  --start-position=[Full Backup Position] \
  --stop-position=[DROP TABLE 직전 Position] \
  /var/lib/mysql/mysql-bin.000001 | mysql -u root -p

# 검증
mysql -u root -p -e "SELECT * FROM recovery_test.test_orders;"
# id 1~7 모두 있어야 함
```

### 실험 2: 복구 시간 측정 템플릿

```bash
#!/bin/bash
# 복구 시간 측정 스크립트

START_TIME=$(date +%s)
echo "=== 복구 시작: $(date) ==="

# [Phase 1] Binary Log 위치 찾기
PHASE1_START=$(date +%s)
# ... Binary Log 분석 ...
PHASE1_TIME=$(($(date +%s) - PHASE1_START))
echo "Phase 1 (Binary Log 분석): ${PHASE1_TIME}초"

# [Phase 2] Full Backup 복원
PHASE2_START=$(date +%s)
# ... 복원 명령 ...
PHASE2_TIME=$(($(date +%s) - PHASE2_START))
echo "Phase 2 (Full Backup 복원): ${PHASE2_TIME}초"

# [Phase 3] Binary Log 재생
PHASE3_START=$(date +%s)
# ... Binary Log 재생 ...
PHASE3_TIME=$(($(date +%s) - PHASE3_START))
echo "Phase 3 (Binary Log 재생): ${PHASE3_TIME}초"

# [Phase 4] 검증 및 운영 반영
PHASE4_START=$(date +%s)
# ... 검증 ...
PHASE4_TIME=$(($(date +%s) - PHASE4_START))
echo "Phase 4 (검증 및 반영): ${PHASE4_TIME}초"

TOTAL_TIME=$(($(date +%s) - START_TIME))
echo "=== 총 복구 시간(MTTR): ${TOTAL_TIME}초 ($(($TOTAL_TIME/60))분) ==="
```

---

## 📊 성능/비용 비교

```
복구 방법별 시간 비교 (100GB DB, orders 100만 건 삭제):

방법 1: Flashback (ROW 포맷 Binary Log 있을 때):
  분석 시간: 5분
  Flashback SQL 생성: 5분
  적용 시간: 10~20분
  총 복구 시간: 20~30분 ★최속
  전제: ROW 포맷, FULL 이미지

방법 2: Replica에서 복구 (Lag 없을 때):
  Replica SQL Thread 중단: 즉시
  mysqldump: 5~15분
  운영 서버 적용: 10~30분
  총 복구 시간: 20~50분
  전제: Replica Lag = 0이었을 것

방법 3: Full Backup + PITR (XtraBackup):
  Full Backup 복원: 30~60분
  Binary Log 재생: 20~40분
  검증 및 반영: 15~30분
  총 복구 시간: 65~130분

방법 4: Full Backup + PITR (mysqldump):
  Full Backup 복원: 6~15시간
  Binary Log 재생: 20~40분
  검증 및 반영: 15~30분
  총 복구 시간: 7~16시간 ★최장

최선 전략:
  ROW 포맷 Binary Log → Flashback 시도 (가장 빠름)
  Replica 있고 Lag 없음 → Replica 복구
  위 둘 다 없으면 → XtraBackup + PITR
```

---

## ⚖️ 트레이드오프

```
복구 방법 선택 트레이드오프:

Flashback:
  ✅ 빠름 (수십 분), 운영 서버 영향 없음
  ✅ 특정 테이블만 복구 가능
  ❌ ROW 포맷 + FULL 이미지 필수
  ❌ 복잡한 FK 관계에서 제약

Replica 복구:
  ✅ 빠름, 실시간에 가까운 복구
  ✅ 검증된 데이터 (실제 운영 데이터)
  ❌ Replica Lag이 있으면 복구 불완전
  ❌ Replica SQL Thread 중단이 늦으면 이미 적용됨

Full Backup + PITR:
  ✅ 가장 확실한 복구 (어느 시점이든 가능)
  ✅ 다양한 장애 유형에 적용 가능
  ❌ XtraBackup 없으면 복원 시간 매우 큼
  ❌ Binary Log 보관이 선행되어야 함

복구 우선순위:
  1순위: Replica에서 즉각 복구 (Lag 없으면)
  2순위: Flashback (ROW 포맷이면)
  3순위: Full Backup(XtraBackup) + PITR
  4순위: Full Backup(mysqldump) + PITR (최후 수단)
```

---

## 📌 핵심 정리

```
장애 복구 핵심:

장애 직후 즉각 조치:
  ① Binary Log 현재 위치 즉시 기록
  ② Binary Log 자동 삭제 일시 중단
  ③ Replica SQL Thread 즉시 중단 (Replica 보존)
  ④ 패닉하지 않고 별도 복구 서버에서 작업

복구 원칙:
  항상 별도 복구 서버에서 먼저 복원 (운영 서버 영향 방지)
  복구 검증 후 운영 서버에 반영
  --stop-position으로 정확한 지점에 멈추기
  GTID 환경: --skip-gtids 필수

복구 속도 순서:
  Replica 복구 > Flashback > XtraBackup+PITR > mysqldump+PITR

사전 준비 체크리스트:
  □ Binary Log 활성화 + 14일 이상 보관
  □ Full Backup 매일 (XtraBackup 권장)
  □ --master-data=2 또는 xtrabackup_binlog_info
  □ Binary Log ROW 포맷 (Flashback 대비)
  □ 복구 서버 또는 Replica 대기
  □ 복구 절차 문서 최신화
  □ 분기 1회 복구 리허설

재발 방지:
  sql_safe_updates = ON (WHERE 없는 UPDATE/DELETE 차단)
  Soft Delete 패턴 도입
  DML 전 COUNT 확인 습관
  Audit Log로 DML 추적
```

---

## 🤔 생각해볼 문제

**Q1.** 오전 10시에 `UPDATE orders SET amount = 0 WHERE 1=1;`을 실수로 실행했고 오전 10시 5분에 발견했다. 다음 환경에서 최적의 복구 방법을 선택하고 절차를 설명하라.

```
환경:
  - Binary Log: ROW 포맷, FULL 이미지, 보관 중
  - Replica: 1대 존재, 평소 Lag 2~3초
  - Full Backup: 전날 새벽 2시 XtraBackup
  - DB 크기: 50GB
```

<details>
<summary>해설 보기</summary>

**최적 방법: Replica 복구 (가장 빠름)**

10시 5분에 발견했을 때 Replica Lag이 2~3초라면 이미 Replica에도 적용됐을 가능성이 있습니다. 먼저 확인합니다.

```bash
# 1. Replica 확인 (Replica에서)
SHOW REPLICA STATUS\G
# Seconds_Behind_Source 확인
# Executed_Gtid_Set 또는 Exec_Source_Log_Pos 확인

# Replica가 UPDATE를 아직 실행 안 했다면 (확률 낮음):
STOP REPLICA SQL_THREAD;
# Replica에서 orders 복사

# UPDATE가 이미 적용됐다면 → Flashback 시도
```

**Flashback 방법** (ROW 포맷 활용):

```bash
# UPDATE 위치 찾기
mysqlbinlog --base64-output=DECODE-ROWS -v \
  mysql-bin.XXXXX | grep -i -B 20 "UPDATE.*orders.*amount=0"

# Flashback으로 UPDATE 역방향 적용
# MyFlash 또는 binlog_rollback 도구 사용:
# flashback --sqlType=UPDATE --tables=orders \
#   --startPosition=[UPDATE 시작] --stopPosition=[UPDATE 끝] \
#   mysql-bin.XXXXX > /tmp/rollback.sql

# rollback.sql 검토 후 적용
mysql -u root -p mydb < /tmp/rollback.sql
```

**Flashback이 안 되면** → XtraBackup + PITR:
- 복구 서버에 Full Backup 복원: 30~60분
- Binary Log 재생 (UPDATE 직전까지): 20~30분
- orders 테이블만 mysqldump → 운영 서버 적용: 20분
- 총: 1~2시간

10시 5분 발견, 복구 완료 예상: 11~12시 (1~2시간 복구)

</details>

---

**Q2.** 복구 리허설을 자동화하기 위한 스크립트 설계를 제안하라. 어떤 단계를 자동화하고 어떤 부분에서 사람의 판단이 필요한가?

<details>
<summary>해설 보기</summary>

**자동화 가능한 단계**:

```bash
#!/bin/bash
# 자동 복구 리허설 스크립트

# 1. 자동화 가능: 복구 서버 프로비저닝
terraform apply -var="purpose=recovery-drill"

# 2. 자동화 가능: Full Backup 다운로드 및 복원
aws s3 sync s3://backup/latest/ /backup/restore/
xtrabackup --prepare --target-dir=/backup/restore/
xtrabackup --copy-back --target-dir=/backup/restore/
chown -R mysql:mysql /var/lib/mysql/
systemctl start mysql

# 3. 자동화 가능: Binary Log 재생 (마지막 24시간)
BINLOG_POS=$(cat /backup/restore/xtrabackup_binlog_info | awk '{print $2}')
mysqlbinlog --skip-gtids --start-position=$BINLOG_POS \
  /backup/binlogs/mysql-bin.* | mysql -u root -p

# 4. 자동화 가능: 데이터 검증
ORDERS_COUNT=$(mysql -u root -p -se "SELECT COUNT(*) FROM mydb.orders")
EXPECTED=1000000
if [ "$ORDERS_COUNT" -lt "$EXPECTED" ]; then
  echo "ERROR: 예상 건수 미달 ($ORDERS_COUNT < $EXPECTED)"
  exit 1
fi

# 5. 자동화 가능: MTTR 계산 및 보고서 생성
echo "MTTR: ${TOTAL_TIME}초" > /tmp/drill_report.txt
curl -X POST $SLACK_WEBHOOK -d "{\"text\": \"복구 리허설 완료: MTTR ${TOTAL_TIME}초\"}"

# 6. 자동화 가능: 복구 서버 자동 종료 (비용 절감)
terraform destroy -var="purpose=recovery-drill" -auto-approve
```

**사람의 판단이 필요한 단계**:

1. **복구 목표 시점 결정**: 어느 시점으로 복구할 것인지 → Binary Log 분석으로 확인
2. **복구된 데이터 품질 검증**: 자동 건수 확인 외 실제 데이터 샘플 검토
3. **운영 반영 여부 결정**: 복구 데이터를 운영에 반영할지는 사람이 최종 판단
4. **이상 감지 시 대응**: 자동화 스크립트 실패 시 원인 파악

**권장 자동화 수준**: 단계 1~5 자동화, 6번(종료)은 검증 완료 후 수동 또는 자동. 결과는 Slack + 이메일로 보고하고 MTTR 추이를 대시보드에 기록합니다.

</details>

---

<div align="center">

**[⬅️ 복구 전략 설계](./04-recovery-strategy-rto-rpo.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 스키마 설계 ➡️](../schema-design/01-data-type-selection.md)**

</div>
