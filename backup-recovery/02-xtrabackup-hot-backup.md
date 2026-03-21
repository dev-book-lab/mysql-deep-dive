# XtraBackup — 물리적 백업의 원리, InnoDB Hot Backup이 가능한 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- XtraBackup이 InnoDB 데이터 파일을 복사하면서 일관성을 보장하는 방식은?
- `--prepare` 단계에서 Crash Recovery가 동작하는 원리는?
- 물리 백업이 Hot Backup(서비스 중 백업)이 가능한 이유는?
- mysqldump 대비 백업·복구 속도 차이는?
- 증분 백업(Incremental Backup)은 어떻게 동작하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 100GB DB를 새벽 2시에 백업하면 아침까지 끝나지 않는다

```
mysqldump의 한계:
  100GB DB에서 mysqldump:
    백업: 3~6시간 (SELECT + 텍스트 변환)
    복원: 6~15시간 (INSERT + 인덱스 재생성)
    합계: 최악의 경우 21시간
    → RTO: 최대 15시간 → 장애 발생 시 하루 서비스 중단

XtraBackup으로 전환:
  백업: 30~60분 (파일 복사 수준)
  복원: 30~60분 (파일 복사 + --prepare)
  합계: 1~2시간
  → RTO: 1~2시간 → 장애 대응 현실적

실무 선택 기준:
  DB 크기 50GB 이상 → XtraBackup 표준
  빠른 RTO 요구 → XtraBackup 필수
```

---

## 😱 흔한 실수 (Before)

### 1. --prepare 단계를 생략하고 복원 시도

```bash
# Before: --backup 후 바로 복원 시도
xtrabackup --backup --target-dir=/backup/

# 복원 시도 (잘못됨)
xtrabackup --copy-back --target-dir=/backup/
# 오류 또는 불완전한 상태로 복원!

# 이유:
# 백업 중 진행 중이던 트랜잭션들이 Redo Log에만 기록됨
# 아직 커밋/롤백 처리 안 됨
# 이 상태로 복원하면 InnoDB가 손상된 상태로 시작

# 올바른 순서:
# 1. --backup (백업)
# 2. --prepare (Crash Recovery 적용)
# 3. --copy-back (복원)
```

### 2. 증분 백업 prepare 순서 실수

```bash
# Before: 증분 백업 prepare 순서 잘못됨

# Full + Inc1 + Inc2가 있을 때
# 잘못된 순서:
xtrabackup --prepare --target-dir=/backup/inc1/
xtrabackup --prepare --target-dir=/backup/full/
xtrabackup --prepare --target-dir=/backup/inc2/

# 올바른 순서:
# Full에 Inc1 적용 → 결과에 Inc2 적용 → 최종 prepare
xtrabackup --prepare --apply-log-only --target-dir=/backup/full/
xtrabackup --prepare --apply-log-only --target-dir=/backup/full/ \
           --incremental-dir=/backup/inc1/
xtrabackup --prepare --target-dir=/backup/full/ \
           --incremental-dir=/backup/inc2/
# 마지막 prepare만 --apply-log-only 없음
```

### 3. 백업 파일을 그대로 datadir에 복사

```bash
# Before: 단순 파일 복사로 복원 시도
cp -r /backup/* /var/lib/mysql/

# 문제:
# 파일 권한 오류 (mysql 사용자 소유권 아님)
# /var/lib/mysql 내 기존 파일과 충돌
# innodb_data_home_dir, log 경로 등 설정 불일치

# 올바른 방법:
# 1. 기존 datadir 비우기
rm -rf /var/lib/mysql/*
# 2. XtraBackup copy-back 사용
xtrabackup --copy-back --target-dir=/backup/full/
# 3. 권한 수정
chown -R mysql:mysql /var/lib/mysql/
```

---

## ✨ 올바른 접근 (After)

```
XtraBackup 백업/복원 표준 절차:

Full Backup:
  xtrabackup \
    --user=backup_user \
    --password=secret \
    --backup \
    --target-dir=/backup/full_$(date +%Y%m%d) \
    --compress \           # 압축 (CPU vs 디스크 트레이드오프)
    --parallel=4           # 병렬 파일 복사 (빠른 백업)

  --backup 완료 후 --prepare:
  xtrabackup --prepare \
    --target-dir=/backup/full_$(date +%Y%m%d)

Incremental Backup (매일):
  xtrabackup \
    --backup \
    --target-dir=/backup/inc_$(date +%Y%m%d) \
    --incremental-basedir=/backup/full_20240101

복원:
  # MySQL 중단 후
  systemctl stop mysql
  rm -rf /var/lib/mysql/*

  xtrabackup --copy-back \
    --target-dir=/backup/full_20240101

  chown -R mysql:mysql /var/lib/mysql/
  systemctl start mysql
```

---

## 🔬 내부 동작 원리

### 1. Hot Backup이 가능한 이유 — Redo Log 추적

```
XtraBackup의 핵심 아이디어:
  InnoDB 데이터 파일(.ibd)을 복사하는 동시에
  Redo Log를 실시간으로 추적하여 일관성 보장

상세 동작:
  Thread 1 (데이터 파일 복사):
    각 .ibd 파일을 순서대로 복사
    복사 시간: 100GB → 수십 분
    이 시간 동안 서비스는 계속 변경 발생

  Thread 2 (Redo Log 추적):
    XtraBackup 시작 시점부터 Redo Log를 실시간 읽기
    새로운 Redo Log 레코드를 백업 디렉토리에 저장
    Thread 1의 파일 복사 완료까지 계속 추적

  결과:
    백업 = 복사된 데이터 파일 + 추적된 Redo Log
    아직 적용 안 된 Redo Log가 포함된 상태
    → --prepare 단계에서 Crash Recovery로 일관성 맞춤
```

### 2. --prepare 단계 Crash Recovery

```
--prepare 내부 동작:

InnoDB Crash Recovery와 동일한 과정:
  1. Redo Log 재실행 (Redo Phase):
     백업 시점 이후의 모든 Redo Log 레코드 재실행
     → 모든 커밋된 변경사항이 데이터 파일에 반영

  2. 미완료 트랜잭션 롤백 (Undo Phase):
     백업 당시 진행 중이었던 트랜잭션들
     (아직 커밋/롤백이 결정 안 된 상태)
     → Undo Log를 이용해 롤백
     → 커밋 확정된 상태만 남김

  결과:
    --prepare 완료 후:
    모든 데이터 파일이 일관된 상태 (Clean Shutdown과 동일)
    → 바로 MySQL에서 사용 가능한 상태

--apply-log-only 옵션:
  증분 백업을 적용할 때 사용
  Redo Phase만 실행, Undo Phase 보류
  → 이후 증분 백업을 더 적용할 수 있도록 열린 상태 유지
  마지막 증분 적용 후에는 --apply-log-only 없이 --prepare
  → Undo Phase까지 완료 (최종 일관성 보장)
```

### 3. 증분 백업 원리

```
InnoDB Page의 LSN (Log Sequence Number):
  각 데이터 페이지에 마지막 수정 LSN이 기록됨
  LSN: Redo Log에서의 위치 (단조증가)

증분 백업 동작:
  Full Backup 완료 시 xtrabackup_checkpoints에 LSN 기록:
    to_lsn: 백업 완료 시점의 LSN

  증분 백업:
    --incremental-basedir의 to_lsn 확인
    to_lsn보다 큰 LSN의 페이지만 복사
    → "Full Backup 이후 변경된 페이지만 백업"

  결과:
    Full: 100GB
    Inc 1일차: 변경된 페이지만 = 1~10GB
    Inc 2일차: Inc 1 이후 변경 = 1~10GB
    → 스토리지 효율적, 빠른 증분 백업

증분 백업 적용 (prepare):
  Full 데이터 파일 + Inc1 데이터 파일 + Inc2 데이터 파일
  → Redo Log를 순서대로 모두 적용
  → 최종 일관된 상태
```

### 4. Streaming 백업과 압축

```bash
# Streaming 백업 (네트워크로 다른 서버에 저장):
xtrabackup --backup --stream=xbstream | \
  ssh backup-server "xbstream -x -C /backup/today/"

# 또는 gzip 압축하여 전송:
xtrabackup --backup --stream=tar | \
  gzip | \
  ssh backup-server "cat > /backup/backup_$(date +%Y%m%d).tar.gz"

# --compress 옵션 (병렬 압축):
xtrabackup --backup \
  --compress \
  --compress-threads=4 \
  --target-dir=/backup/full/

# 복원 시 압축 해제:
xtrabackup --decompress --target-dir=/backup/full/
xtrabackup --prepare --target-dir=/backup/full/
```

### 5. Binary Log 위치 정보 확인

```bash
# 백업 완료 후 Binary Log 위치 확인
cat /backup/full/xtrabackup_binlog_info
# 출력 예:
# mysql-bin.000005    1234567
# 또는 GTID 환경:
# mysql-bin.000005    1234567    3E11FA47-...:1-5000

# 이 정보로 PITR 수행 시 시작 위치 결정
# Full 복원 후 mysql-bin.000005의 1234567 이후부터 Binary Log 재생
```

---

## 💻 실전 실험

### 실험 1: Full Backup과 --prepare 수행

```bash
# 환경: MySQL 8.0 + Percona XtraBackup 8.0

# Full Backup
xtrabackup \
  --user=root \
  --password=secret \
  --backup \
  --target-dir=/tmp/full_backup/ \
  --parallel=4

# 백업 완료 확인
ls /tmp/full_backup/
# xtrabackup_checkpoints: LSN 정보
# xtrabackup_binlog_info: Binary Log 위치

# --prepare 수행
xtrabackup --prepare --target-dir=/tmp/full_backup/
# "completed OK!" 메시지 확인

# 복원 (테스트 환경)
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/tmp/full_backup/
chown -R mysql:mysql /var/lib/mysql/
systemctl start mysql
```

### 실험 2: 증분 백업 수행

```bash
# Day 1: Full Backup
xtrabackup --backup --target-dir=/backup/full_d1/

# Day 2: Incremental
xtrabackup --backup \
  --target-dir=/backup/inc_d2/ \
  --incremental-basedir=/backup/full_d1/

# Day 3: Incremental
xtrabackup --backup \
  --target-dir=/backup/inc_d3/ \
  --incremental-basedir=/backup/inc_d2/

# 파일 크기 비교
du -sh /backup/full_d1/ /backup/inc_d2/ /backup/inc_d3/
# Full이 가장 크고 Inc는 변경량만큼 작음

# Full + Inc1 + Inc2 모두 적용해서 복원
xtrabackup --prepare --apply-log-only --target-dir=/backup/full_d1/
xtrabackup --prepare --apply-log-only --target-dir=/backup/full_d1/ \
           --incremental-dir=/backup/inc_d2/
xtrabackup --prepare --target-dir=/backup/full_d1/ \
           --incremental-dir=/backup/inc_d3/
# 마지막만 --apply-log-only 없음 (Undo Phase 포함)
```

### 실험 3: 백업 시간 측정

```bash
# XtraBackup 백업 시간 측정
time xtrabackup --backup \
  --target-dir=/tmp/xb_backup/ \
  --parallel=4

# mysqldump 백업 시간 측정
time mysqldump \
  --single-transaction \
  --quick \
  mydb > /tmp/mysqldump.sql

# 복원 시간 비교
time (xtrabackup --copy-back --target-dir=/tmp/xb_backup/ && \
      chown -R mysql:mysql /var/lib/mysql/)

time mysql mydb < /tmp/mysqldump.sql
```

---

## 📊 성능/비용 비교

```
XtraBackup vs mysqldump 상세 비교:

백업 속도 (100GB InnoDB DB):
  mysqldump: 3~6시간
  XtraBackup: 30~60분
  XtraBackup이 4~6배 빠름

복원 속도 (100GB):
  mysqldump: 6~15시간 (INSERT + 인덱스 재생성)
  XtraBackup: 30~60분 (파일 복사)
  XtraBackup이 10~20배 빠름

백업 파일 크기:
  mysqldump (원본): 150~300GB
  mysqldump (gzip): 40~80GB
  XtraBackup (원본): 100~120GB
  XtraBackup (압축): 40~70GB

서비스 영향:
  mysqldump --single-transaction: 없음 (Hot)
  XtraBackup: 없음 (Hot)
  mysqldump --lock-all-tables: 백업 중 쓰기 차단

증분 백업:
  mysqldump: 기본 지원 없음 (전체 재덤프)
  XtraBackup: 기본 지원 (LSN 기반 변경 페이지만)

운영 복잡도:
  mysqldump: 단순 (옵션만 기억하면 됨)
  XtraBackup: 복잡 (--prepare, 증분 적용 순서)
```

---

## ⚖️ 트레이드오프

```
XtraBackup 도입 트레이드오프:

이점:
  ✅ 빠른 백업 (mysqldump 대비 4~6배)
  ✅ 매우 빠른 복원 (mysqldump 대비 10~20배)
  ✅ 증분 백업으로 스토리지 효율화
  ✅ Hot Backup (서비스 영향 없음)
  ✅ Streaming으로 원격 서버에 직접 백업 가능

비용:
  ❌ 별도 설치 필요 (Percona XtraBackup)
  ❌ 복원 절차 복잡 (--backup → --prepare → --copy-back)
  ❌ MySQL 버전과 XtraBackup 버전 매핑 필요
  ❌ 특정 테이블만 선택적 복원 어려움 (전체 파일 기반)
  ❌ 이기종 OS/버전 간 마이그레이션 어려움

선택 기준:
  빠른 RTO 필요 → XtraBackup 필수
  특정 테이블 선택적 복원 필요 → mysqldump 병행
  이기종 마이그레이션 → mysqldump
  50GB 이상 대용량 → XtraBackup

혼합 전략 (권장):
  XtraBackup: 운영 백업 (빠른 복원)
  mysqldump: 스키마 버전 관리, 특정 테이블 마이그레이션
```

---

## 📌 핵심 정리

```
XtraBackup 핵심:

Hot Backup 원리:
  데이터 파일 복사 + Redo Log 실시간 추적
  서비스 중단 없이 100% 일관성 있는 백업 가능

--prepare 단계:
  Crash Recovery 적용 (Redo + Undo)
  커밋된 변경사항 반영 + 진행 중 트랜잭션 롤백
  이 단계 없으면 복원 불가

증분 백업:
  LSN 기반으로 변경된 페이지만 백업
  Full → Inc1 → Inc2 순서로 --apply-log-only 적용
  마지막 prepare만 --apply-log-only 없음

표준 절차:
  --backup → --prepare → --copy-back → chown

Binary Log 위치:
  xtrabackup_binlog_info 파일에 기록
  PITR 수행 시 여기서부터 Binary Log 재생 시작

mysqldump 대비:
  백업: 4~6배 빠름
  복원: 10~20배 빠름
  대용량 DB 표준 선택
```

---

## 🤔 생각해볼 문제

**Q1.** XtraBackup 증분 백업의 `--prepare` 단계에서 마지막 증분 파일을 적용할 때만 `--apply-log-only`를 생략하는 이유를 설명하라.

<details>
<summary>해설 보기</summary>

`--apply-log-only`는 Redo Log 재실행(Redo Phase)만 수행하고 미완료 트랜잭션의 롤백(Undo Phase)을 보류합니다.

**이유**: 증분 백업을 적용하는 과정에서 미완료 트랜잭션은 "아직 다음 증분 백업에서 커밋될 수 있는 상태"입니다. Inc1을 적용할 때 진행 중인 트랜잭션을 롤백해버리면, Inc2에 그 트랜잭션의 커밋 레코드가 있어도 이미 롤백된 상태이므로 데이터 불일치가 발생합니다.

따라서 모든 증분 백업을 적용한 후 마지막 단계에서만 Undo Phase를 실행해야 합니다. 마지막 증분까지 적용된 상태에서 미완료 트랜잭션은 진짜로 커밋되지 않은 것이므로 롤백이 올바른 처리입니다.

```bash
# 순서 요약:
xtrabackup --prepare --apply-log-only --target-dir=/full/  # Redo만
xtrabackup --prepare --apply-log-only --target-dir=/full/ --incremental-dir=/inc1/  # Redo만
xtrabackup --prepare --target-dir=/full/ --incremental-dir=/inc2/  # Redo + Undo (최종)
```

</details>

---

**Q2.** 운영 중 MySQL 서버에서 XtraBackup 실행 중 `xtrabackup: Error: The transaction log file is too small for innodb_log_buffer_size` 오류가 발생했다. 원인과 해결 방법을 설명하라.

<details>
<summary>해설 보기</summary>

**원인**: Redo Log 파일이 너무 작거나 백업 진행 중 Redo Log가 순환(wrap around)하여 XtraBackup이 추적해야 할 Redo Log 레코드가 덮어쓰여진 경우입니다.

XtraBackup은 백업 시작 시점부터의 모든 Redo Log를 캡처해야 하는데, 백업이 느리거나(대용량 DB) Redo Log 파일이 작으면 캡처 전에 덮어써집니다.

**해결 방법**:

```sql
-- Redo Log 크기 확인
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'innodb_log_files_in_group';
-- 기본: 각 48MB × 2 = 96MB (작음!)

-- 증설 (my.cnf):
-- innodb_log_file_size = 512M  -- 더 큰 Redo Log
-- innodb_log_files_in_group = 2
```

또는 XtraBackup 속도를 높여 Redo Log 순환 전에 백업 완료:
```bash
xtrabackup --backup --parallel=8 --target-dir=/backup/
# 더 많은 병렬 스레드로 파일 복사 속도 향상
```

MySQL 8.0에서는 Redo Log가 동적 크기 조정을 지원하므로 `innodb_redo_log_capacity`를 설정합니다.

</details>

---

<div align="center">

**[⬅️ mysqldump](./01-mysqldump-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: PITR ➡️](./03-pitr-point-in-time-recovery.md)**

</div>
