# mysqldump — 논리적 백업의 동작 원리와 한계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `--single-transaction` 없이 mysqldump를 실행하면 왜 일관성 없는 백업이 생성되는가?
- `--single-transaction`이 REPEATABLE READ 스냅샷으로 일관성을 보장하는 원리는?
- mysqldump가 스키마와 데이터를 SQL로 덤프하는 내부 과정은?
- 대용량 테이블에서 논리 백업의 속도·크기 한계는?
- 복구 시간이 물리 백업보다 긴 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 백업했는데 복원이 안 된다 — 잘못된 mysqldump 옵션의 결과

```
실제 장애 시나리오:
  매일 새벽 2시 mysqldump 자동 백업 중
  백업 진행 중 서비스는 정상 운영 (트랜잭션 발생)

  --single-transaction 없이 백업:
    orders 테이블 덤프 중 (10만 건)
    그 동안 다른 트랜잭션이 order_items에 INSERT
    order_items 덤프 시점: orders와 다른 시점
    → orders 10건이 있지만 해당 order_items 없음
    → 참조 무결성 불일치 상태로 백업됨

  복원 시:
    orders 복원 → 성공
    order_items 복원 → Foreign Key 오류!
    → 백업이 있어도 복원 불가능한 상황

  --single-transaction 추가:
    REPEATABLE READ 스냅샷으로 전체 백업 일관성 보장
    백업 시작 시점의 스냅샷 유지
```

---

## 😱 흔한 실수 (Before)

### 1. --single-transaction 없이 InnoDB 테이블 백업

```bash
# Before: 일관성 없는 백업
mysqldump -u root -p mydb > backup.sql

# 문제:
# 각 테이블을 독립적으로 덤프
# 테이블 간 덤프 시점 차이 존재 (락 없이)
# → 테이블 간 데이터 불일치 가능
# → 복원 시 FK 위반, 불완전 데이터

# 올바른 방법:
mysqldump -u root -p \
  --single-transaction \
  --master-data=2 \
  mydb > backup.sql
```

### 2. MyISAM 테이블에 --single-transaction 사용

```bash
# Before: MyISAM 테이블에 --single-transaction 사용
mysqldump --single-transaction mydb_with_myisam > backup.sql

# 문제:
# --single-transaction은 InnoDB 전용
# MyISAM은 트랜잭션 미지원 → 스냅샷 없음
# --single-transaction이어도 MyISAM 테이블은 덤프 시 테이블 락 필요
# → --lock-tables가 병행 필요하지만 --single-transaction과 충돌

# 올바른 방법 (MyISAM 포함):
mysqldump --lock-all-tables mydb > backup.sql
# 또는 MyISAM → InnoDB 전환 후 --single-transaction 사용
```

### 3. --master-data 없이 백업하여 PITR 불가

```bash
# Before: Binary Log 위치 정보 없이 백업
mysqldump --single-transaction mydb > backup.sql

# 문제:
# Binary Log 위치 정보가 백업에 포함 안 됨
# 복원 후 Binary Log를 어디서부터 재생해야 할지 모름
# → PITR (Point-In-Time Recovery) 불가

# 올바른 방법:
mysqldump \
  --single-transaction \
  --master-data=2 \   # 백업 시점의 Binary Log 위치 주석으로 포함
  mydb > backup.sql

# 백업 파일에서 확인:
# -- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000005', MASTER_LOG_POS=1234567;
```

---

## ✨ 올바른 접근 (After)

```
mysqldump 권장 옵션 조합:

InnoDB 전용 데이터베이스 (일반적):
  mysqldump \
    --user=backup_user \
    --password=secret \
    --single-transaction \     # InnoDB REPEATABLE READ 스냅샷
    --master-data=2 \          # Binary Log 위치 주석으로 포함
    --routines \               # 스토어드 프로시저, 함수 포함
    --triggers \               # 트리거 포함
    --events \                 # 이벤트 포함
    --set-gtid-purged=ON \     # GTID 환경에서 필요
    --databases mydb \
    > backup_$(date +%Y%m%d_%H%M%S).sql

대용량 테이블 (메모리 효율):
  mysqldump \
    --single-transaction \
    --quick \                  # Row별 스트리밍 (메모리 절약)
    --no-autocommit \          # 복원 시 INSERT 묶음으로 빠름
    mydb > backup.sql

압축 백업:
  mysqldump --single-transaction mydb | gzip > backup.sql.gz
```

---

## 🔬 내부 동작 원리

### 1. --single-transaction 스냅샷 원리

```
--single-transaction 동작:

시작:
  START TRANSACTION WITH CONSISTENT SNAPSHOT;
  → InnoDB: 현재 시점의 MVCC 스냅샷 획득
  → 이 스냅샷을 백업 종료까지 유지

스냅샷의 의미:
  백업 시작 시점의 커밋된 데이터만 보임
  이후 다른 트랜잭션의 INSERT/UPDATE/DELETE:
    다른 세션에서 새 트랜잭션 커밋 → 백업 스냅샷에는 반영 안 됨
    백업 세션: 항상 백업 시작 시점의 데이터 읽음

테이블 간 일관성:
  orders 덤프 시점 = order_items 덤프 시점 = 백업 시작 시점
  → 모든 테이블이 동일한 시점의 스냅샷
  → 테이블 간 참조 무결성 보장

주의:
  REPEATABLE READ이므로 DML 쓰기는 차단하지 않음
  → 서비스는 정상 운영 가능 (Hot Backup 가능)
  단, DDL(ALTER TABLE, DROP TABLE)은 백업 일관성 깸
  → 백업 중 DDL 실행 금지

MyISAM:
  트랜잭션 미지원 → 스냅샷 없음
  --single-transaction이어도 MyISAM은 덤프 시점마다 다름
  → MyISAM 테이블 있으면 --lock-all-tables 사용
```

### 2. mysqldump 덤프 과정 상세

```sql
-- mysqldump 내부 실행 순서:

-- 1. 메타데이터 수집
SHOW VARIABLES LIKE 'character_set_server';
SELECT @@global.gtid_executed;

-- 2. 스냅샷 시작 (--single-transaction)
START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */;

-- 3. Binary Log 위치 기록 (--master-data=2)
SHOW MASTER STATUS;
-- 결과를 주석으로 백업 파일에 기록

-- 4. 각 테이블 스키마 덤프
SHOW CREATE TABLE orders\G
-- CREATE TABLE orders (...) 형태로 백업 파일에 기록

-- 5. 각 테이블 데이터 덤프
SELECT * FROM orders;
-- 결과를 INSERT INTO 또는 COPY 형태로 백업 파일에 기록

-- --quick 옵션 없을 때:
-- 전체 테이블을 메모리에 로드 후 덤프
-- 대용량 테이블 → OOM 위험

-- --quick 옵션 있을 때:
-- Row별로 스트리밍 읽기 (mysql_use_result)
-- 메모리 사용량 최소화

-- 6. 스토어드 프로시저, 트리거, 이벤트 덤프 (--routines --triggers --events)

-- 7. 트랜잭션 종료
COMMIT;
```

### 3. 백업 파일 구조

```sql
-- backup.sql 파일 내용 예시:
-- MySQL dump 8.0.36
-- Host: localhost    Database: mydb
-- ...

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;  -- FK 비활성화 (복원 순서 무관)
SET UNIQUE_CHECKS = 0;       -- UK 비활성화 (빠른 복원)
SET AUTOCOMMIT = 0;          -- 자동 커밋 비활성화

-- 1. 스키마 (CREATE TABLE)
DROP TABLE IF EXISTS `orders`;
CREATE TABLE `orders` (
    `id` bigint NOT NULL AUTO_INCREMENT,
    `user_id` bigint NOT NULL,
    ...
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;

-- 2. 데이터 (INSERT)
INSERT INTO `orders` VALUES
(1, 100, 'PAID', ...),
(2, 101, 'PENDING', ...),
...;   -- 기본: 여러 Row를 하나의 INSERT로 묶음 (extended-insert)

-- --skip-extended-insert 옵션:
-- INSERT INTO orders VALUES (1, ...);
-- INSERT INTO orders VALUES (2, ...);
-- 더 느리지만 PITR 재생 시 Row별 필터링 가능

COMMIT;
SET FOREIGN_KEY_CHECKS = 1;
SET UNIQUE_CHECKS = 1;
```

### 4. 논리 백업의 한계

```
크기 한계:
  SQL 텍스트 형태 → 원본 데이터보다 1.5~3배 큼
  테이블 100GB → 백업 파일 150~300GB
  gzip 압축 시 50~100GB 수준

속도 한계:
  백업:
    테이블 전체 SELECT → 디스크 읽기 + 텍스트 변환
    대형 테이블: 시간당 10~30GB 수준
    100GB DB: 수 시간 소요

  복구:
    SQL 파싱 + INSERT 실행 + 인덱스 재생성
    CREATE TABLE → INSERT → 인덱스 빌드 순서
    대형 테이블: 백업보다 2~5배 느림
    100GB DB 복원: 수 시간 ~ 수십 시간

  비교:
    XtraBackup (물리 백업): 백업/복구 모두 수십 분 수준

메모리 한계:
  --quick 없으면 테이블 전체를 메모리에 로드
  100GB 테이블 → 100GB 메모리 필요
  → 항상 --quick 사용 권장

DDL 변경 영향:
  백업 중 ALTER TABLE 실행 시 일관성 깨짐
  → 백업 윈도우 동안 DDL 금지 필요
```

---

## 💻 실전 실험

### 실험 1: --single-transaction 유무에 따른 일관성 차이

```sql
-- 테스트 환경 준비
CREATE TABLE parent_t (
    id INT PRIMARY KEY,
    val VARCHAR(50)
) ENGINE=InnoDB;

CREATE TABLE child_t (
    id INT AUTO_INCREMENT PRIMARY KEY,
    parent_id INT,
    val VARCHAR(50),
    FOREIGN KEY (parent_id) REFERENCES parent_t(id)
) ENGINE=InnoDB;

INSERT INTO parent_t VALUES (1,'P1'), (2,'P2'), (3,'P3');
INSERT INTO child_t (parent_id, val) VALUES (1,'C1'), (2,'C2'), (3,'C3');

-- 실험 1-A: --single-transaction 없이 (비권장)
-- 터미널 1: 백업 시작
-- mysqldump -u root -p testdb parent_t > /tmp/test_no_single.sql
-- 터미널 2: 백업 중 다른 트랜잭션
-- INSERT INTO parent_t VALUES (4, 'P4_NEW');
-- INSERT INTO child_t (parent_id, val) VALUES (4, 'C4_NEW');
-- COMMIT;
-- → parent_t에는 없고 child_t에는 있는 불일치 가능

-- 실험 1-B: --single-transaction 있음
-- mysqldump -u root -p --single-transaction testdb > /tmp/test_single.sql
-- 동일한 INSERT 실행해도 백업에 반영 안 됨 (스냅샷)
-- → 일관성 유지

DROP TABLE child_t, parent_t;
```

### 실험 2: 백업 파일의 Binary Log 위치 확인

```bash
# --master-data=2 옵션으로 백업
mysqldump \
  --single-transaction \
  --master-data=2 \
  mydb > backup_with_pos.sql

# 백업 파일에서 Binary Log 위치 확인
grep "CHANGE MASTER\|CHANGE REPLICATION" backup_with_pos.sql
# 출력 예:
# -- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000005', MASTER_LOG_POS=1234567;
# 또는 GTID 환경:
# -- SET @@GLOBAL.GTID_PURGED='+abc-def:1-1000';
```

### 실험 3: 복원 속도 측정

```bash
# 복원 시간 측정
time mysql -u root -p mydb < backup.sql
# 또는
time pv backup.sql | mysql -u root -p mydb
# pv: 진행률과 속도 표시

# 병렬 복원 (mydumper/myloader 사용):
# mydumper: 병렬 덤프 (mysqldump보다 훨씬 빠름)
# myloader: 병렬 복원
mydumper \
  --user=root \
  --password=secret \
  --database=mydb \
  --threads=8 \       # 8개 스레드로 병렬 덤프
  --outputdir=/tmp/dump/

myloader \
  --user=root \
  --password=secret \
  --database=mydb_new \
  --threads=8 \
  --directory=/tmp/dump/
```

---

## 📊 성능/비용 비교

```
mysqldump vs XtraBackup 비교:

100GB InnoDB 데이터베이스 기준:

백업 속도:
  mysqldump: 2~4시간 (SELECT + 텍스트 변환)
  XtraBackup: 20~40분 (파일 복사)
  차이: 4~6배

백업 파일 크기:
  mysqldump: 150~300GB (SQL 텍스트, 압축 전)
  mysqldump + gzip: 40~80GB
  XtraBackup: 100~120GB (데이터 파일 그대로)
  XtraBackup + 압축: 40~70GB
  비슷한 수준

복원 속도:
  mysqldump: 4~12시간 (SQL 파싱 + INSERT + 인덱스 재생성)
  XtraBackup: 20~40분 (파일 복사 후 --prepare)
  차이: 10~20배 (복원이 가장 큰 차이)

서비스 영향:
  mysqldump --single-transaction: 영향 거의 없음
  mysqldump --lock-all-tables: 백업 중 쓰기 차단
  XtraBackup: 영향 거의 없음 (Hot Backup)

용도 적합성:
  mysqldump: 소규모 DB, 스키마 버전 관리, 이기종 DB 마이그레이션
  XtraBackup: 대용량 DB, 빠른 복구 필요, 운영 환경
```

---

## ⚖️ 트레이드오프

```
mysqldump 선택 트레이드오프:

이점:
  ✅ 단순 설치 (MySQL 기본 제공)
  ✅ 텍스트 형태 → 사람이 읽기 가능 (특정 테이블만 복원 용이)
  ✅ 이기종 MySQL 버전 간 마이그레이션 가능
  ✅ 특정 테이블, 특정 조건으로 선택적 백업/복원
  ✅ 스키마 버전 관리 도구와 연동 용이

비용:
  ❌ 대용량 DB에서 백업 속도 느림
  ❌ 복원 속도가 매우 느림 (INSERT + 인덱스 재생성)
  ❌ 백업 파일 크기 큼 (SQL 텍스트)
  ❌ 백업 중 DDL 실행 시 일관성 깨짐
  ❌ --quick 없으면 대용량 테이블 OOM 위험

선택 기준:
  DB 크기 < 10GB → mysqldump 충분
  DB 크기 10~50GB → mydumper 검토
  DB 크기 > 50GB → XtraBackup 권장
  이기종 마이그레이션 → mysqldump 유일한 선택
  빠른 복구 필요 → XtraBackup 필수

mydumper (대안):
  mysqldump의 병렬 버전
  다중 스레드로 빠른 덤프
  테이블별 별도 파일 → 선택적 복원 용이
  mysqldump와 XtraBackup의 중간 지점
```

---

## 📌 핵심 정리

```
mysqldump 핵심:

--single-transaction:
  InnoDB REPEATABLE READ 스냅샷으로 일관성 보장
  백업 시작 시점의 스냅샷 유지
  다른 DML 트랜잭션 차단하지 않음 (Hot Backup)
  MyISAM에는 효과 없음

--master-data=2:
  백업 시점의 Binary Log 파일명 + 위치를 주석으로 포함
  PITR(Point-In-Time Recovery) 를 위해 필수

--quick:
  Row별 스트리밍 읽기 (메모리 효율)
  대용량 테이블 필수 옵션

한계:
  복원 속도: INSERT + 인덱스 재생성으로 물리 백업보다 10~20배 느림
  백업 파일 크기: 원본보다 1.5~3배 큼
  100GB 이상 → XtraBackup 또는 mydumper 검토

권장 옵션:
  mysqldump --single-transaction --master-data=2 --quick \
            --routines --triggers --events mydb > backup.sql
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 상황에서 mysqldump 백업 파일을 사용해 복원할 수 있는가? 이유를 설명하라.

```
상황:
  매일 새벽 2시 mysqldump 백업 (--single-transaction 없음)
  오전 10시 orders 테이블의 데이터 오염 발견
  orders 테이블에만 문제, 다른 테이블은 정상
```

<details>
<summary>해설 보기</summary>

**복원은 가능하지만 위험합니다.**

**문제점**:
1. `--single-transaction` 없이 백업했으므로 `orders`와 다른 테이블(예: `order_items`)이 서로 다른 시점의 스냅샷으로 백업됨
2. `orders`만 복원하면 이후 시점에 추가된 `order_items`와 불일치 발생 가능 (FK 오류)
3. 다른 테이블도 오전 10시까지의 변경사항은 복원 파일에 없음

**올바른 접근**:
```bash
# orders 테이블만 선택적 복원
mysql mydb < <(grep -A 999999 "Table structure for table \`orders\`" backup.sql \
              | grep -B 999999 "Table structure for table" \
              | head -n -1)
# 더 안전한 방법: mysqldump의 --tables 옵션으로 테이블별 백업 관리

# 이후 오전 2시부터 오염 직전까지 Binary Log 재생 (PITR)
# → 오염된 쿼리 직전 위치에서 중단
```

**교훈**: `--single-transaction`과 `--master-data=2`는 항상 함께 사용해야 안전한 복원이 가능합니다.

</details>

---

**Q2.** 200GB MySQL DB를 하루 한 번 백업할 때 mysqldump와 XtraBackup 중 어느 것을 선택하고, 백업 시간과 RTO를 어떻게 줄이겠는가?

<details>
<summary>해설 보기</summary>

**XtraBackup 선택** 이유:
- 200GB mysqldump 백업: 6~12시간 → 새벽 2시 시작해도 오전까지 백업 중
- XtraBackup: 1~2시간 수준
- 복원: mysqldump는 12~24시간, XtraBackup은 1~2시간

**RTO 최소화 전략**:
```bash
# 1. XtraBackup으로 Full Backup (주 1회)
xtrabackup --backup --compress --target-dir=/backup/full/

# 2. 매일 증분 백업 (Incremental)
xtrabackup --backup \
  --incremental-basedir=/backup/full/ \
  --target-dir=/backup/inc_$(date +%Y%m%d)/

# 3. Binary Log 실시간 아카이브 (5분 주기)
# → Full + Incremental + Binary Log로 RPO: 5분 이내

# 4. 복원 리허설 (월 1회)
# 별도 서버에서 실제 복원 후 RTO 측정
# 문제 발견 시 절차 개선
```

이렇게 하면 RTO: 2~3시간, RPO: 5분으로 운영할 수 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: XtraBackup ➡️](./02-xtrabackup-hot-backup.md)**

</div>
