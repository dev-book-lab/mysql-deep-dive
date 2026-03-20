# 파티션 운영 — 추가/삭제/재구성, 마이그레이션

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ALTER TABLE ... ADD PARTITION` 시 Online DDL이 가능한가?
- `DROP PARTITION`이 `DELETE`보다 빠른 이유는 무엇인가?
- 기존 테이블에 파티션을 추가할 때 잠금 범위와 소요 시간은?
- MAXVALUE 파티션을 SPLIT하는 방법은?
- 파티션 재구성(REORGANIZE) 시 Online으로 진행할 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 운영 중 파티션 관리 실수 = 서비스 장애

```
실제 운영 이슈:

이슈 1: MAXVALUE 파티션 없이 새 파티션 추가 실패
  2024년까지 파티션 정의, 2025년 1월 데이터 INSERT 시도
  → ERROR 1526: Table has no partition for value 2025
  → 새 파티션 DDL 실행 중 INSERT 차단
  → 순간 서비스 장애

이슈 2: 기존 테이블에 파티션 추가 = 전체 테이블 재구성
  10억 건 테이블에 PARTITION BY RANGE 추가
  → 전체 테이블 복사 필요 → 수 시간 잠금
  → 운영 중 실행 불가 → 서비스 중단

이슈 3: pt-osc로 파티셔닝 마이그레이션 중
  트리거가 INSERT에 걸려있어 파티션 테이블의 FOREIGN KEY 제약과 충돌
  → 마이그레이션 실패

파티션 운영은 미리 설계하고 절차를 이해해야 한다
```

---

## 😱 흔한 실수 (Before)

### 1. MAXVALUE 파티션 없이 연도 파티션만 정의

```sql
-- Before: MAXVALUE 없이 정의
CREATE TABLE logs (
    id BIGINT AUTO_INCREMENT,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
    -- MAXVALUE 없음!
);

-- 2025년 1월 1일:
INSERT INTO logs (created_at) VALUES ('2025-01-01 00:00:00');
-- ERROR 1526: Table has no partition for value from column_list
-- 서비스 장애!

-- 새 파티션 추가하려면:
ALTER TABLE logs ADD PARTITION (PARTITION p2025 VALUES LESS THAN (2026));
-- 이 DDL 실행 동안 잠금 발생
```

### 2. REORGANIZE PARTITION 없이 MAXVALUE 분할

```sql
-- MAXVALUE 파티션이 있는 상태에서 새 연도 파티션 추가 시도
-- Before: ADD PARTITION으로 추가 시도 (잘못됨)
ALTER TABLE logs ADD PARTITION (PARTITION p2025 VALUES LESS THAN (2026));
-- Error: MAXVALUE 파티션이 있으면 ADD PARTITION으로 앞에 삽입 불가
-- MAXVALUE는 항상 마지막이어야 함

-- 올바른 방법: REORGANIZE PARTITION
ALTER TABLE logs REORGANIZE PARTITION pmax INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);
```

---

## ✨ 올바른 접근 (After)

```
파티션 운영 절차:

정기 파티션 추가 (MAXVALUE 있을 때):
  REORGANIZE PARTITION pmax INTO (새 파티션, pmax);
  → Online DDL 가능 (ALGORITHM=INPLACE, 데이터 없는 파티션만)
  → pmax가 빈 파티션이면 즉각, 데이터 있으면 재구성 비용

파티션 삭제 (TTL):
  ALTER TABLE logs DROP PARTITION p2022;
  → 파일 수준 삭제 (밀리초)
  → 해당 파티션 데이터 영구 삭제

파티션 비우기 (데이터 보존 필요 시):
  ALTER TABLE logs TRUNCATE PARTITION p2022;
  → 데이터 삭제 (DROP보다 안전, 구조 유지)
  → DROP보다 느리지만 Row별 DELETE보다 빠름

파티션 교환 (EXCHANGE):
  ALTER TABLE logs EXCHANGE PARTITION p2022 WITH TABLE logs_archive;
  → 파티션 데이터를 다른 테이블로 이동
  → 아카이브 패턴에 유용
```

---

## 🔬 내부 동작 원리

### 1. DROP PARTITION이 DELETE보다 빠른 이유

```
DELETE FROM logs WHERE created_at < '2023-01-01':

  Row 수준 처리:
  1. 조건에 맞는 Row 하나씩 찾기
  2. 각 Row에 X Lock 획득
  3. 각 Row에 Undo Log 기록 (MVCC 롤백 대비)
  4. 각 Row를 Deleted 마킹 (페이지에 즉시 삭제 안 됨)
  5. 연결된 Secondary Index Entry 각각 삭제
  6. COMMIT 후 Purge Thread가 실제 공간 해제

  비용: O(삭제 건수) × (Undo 기록 + Lock + 인덱스 수정)
  수천만 건 삭제: 수십 분 ~ 수 시간
  Undo Log 폭발 → ibdata1 또는 undo tablespace 증가

ALTER TABLE logs DROP PARTITION p2022:

  파일 수준 처리:
  1. p2022 파티션의 .ibd 파일 제거 (또는 세그먼트 해제)
  2. 메타데이터 딕셔너리 업데이트
  3. InnoDB 버퍼풀에서 해당 파티션 페이지 해제

  비용: 파일 삭제 비용 (OS 수준) + 메타데이터 업데이트
  수천만 건이어도: 밀리초 ~ 수 초
  Undo Log 없음 (Row 수준 처리 없음)
  Lock 범위: 해당 파티션만 (다른 파티션 영향 없음)
```

### 2. 기존 테이블에 파티셔닝 추가 — 마이그레이션 절차

```sql
-- 방법 1: 직접 ALTER TABLE (소량 데이터만 권장)
ALTER TABLE orders
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);
-- 전체 테이블 재구성 필요
-- ALGORITHM=COPY (기본): 임시 테이블 복사 → 원본 테이블 교체
-- 수억 건이면 수 시간, 이 동안 쓰기 차단
-- Online DDL로 ALGORITHM=INPLACE 불가 (파티셔닝 추가는 항상 COPY)

-- 방법 2: 새 파티션 테이블 생성 + 데이터 이관
-- 운영 무중단으로 마이그레이션

-- 1단계: 새 파티션 테이블 생성
CREATE TABLE orders_new (
    id         BIGINT AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    amount     DECIMAL(10,2),
    PRIMARY KEY (id, created_at),
    INDEX idx_user (user_id, created_at)
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
) ENGINE=InnoDB;

-- 2단계: 과거 데이터 배치 이관 (청크 단위)
INSERT INTO orders_new SELECT * FROM orders WHERE created_at < '2024-01-01' LIMIT 10000;
-- 위를 반복하며 과거 데이터 이관

-- 3단계: pt-online-schema-change로 실시간 데이터 동기화
-- 또는 애플리케이션 레벨 이중 쓰기 → 검증 → 전환

-- 4단계: 테이블 이름 교환 (서비스 중단 최소화)
RENAME TABLE orders TO orders_old, orders_new TO orders;
```

### 3. pt-online-schema-change로 파티셔닝 마이그레이션

```bash
# pt-osc로 파티션 추가 (주의: 파티션 테이블은 FK 불가)

# FK 없는 테이블의 파티셔닝 추가
pt-online-schema-change \
  --host=localhost \
  --user=root \
  --password=secret \
  --database=mydb \
  --table=orders \
  --alter="PARTITION BY RANGE COLUMNS (created_at) (
      PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
      PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
      PARTITION pmax VALUES LESS THAN (MAXVALUE)
  )" \
  --execute

# pt-osc 동작 원리:
# 1. orders_new (파티션 테이블) 생성
# 2. 트리거 설치 (INSERT/UPDATE/DELETE를 orders_new에 동기화)
# 3. 기존 데이터를 청크 단위로 orders_new에 복사
# 4. 동기화 완료 후 테이블 이름 교환 (RENAME)
# 5. 트리거 제거

# 주의:
# 파티션 테이블이 목적지이므로 FK 없어야 함
# FK 있는 원본 테이블이면 FK 제거 후 pt-osc 실행
```

### 4. EXCHANGE PARTITION — 아카이브 패턴

```sql
-- 아카이브 패턴: p2022 파티션을 별도 테이블로 이동

-- 1단계: 아카이브 테이블 준비 (파티션 없음, 동일 스키마)
CREATE TABLE logs_archive_2022 LIKE logs;
ALTER TABLE logs_archive_2022 REMOVE PARTITIONING;
-- 파티션 제거, 일반 테이블로 변환

-- 2단계: 파티션 데이터 교환
ALTER TABLE logs EXCHANGE PARTITION p2022 WITH TABLE logs_archive_2022;
-- logs.p2022 ↔ logs_archive_2022 데이터 교환 (순간적)
-- logs의 p2022는 빈 파티션이 됨
-- logs_archive_2022에 과거 데이터

-- 3단계: 빈 p2022 파티션 제거
ALTER TABLE logs DROP PARTITION p2022;

-- 4단계: 아카이브 테이블은 별도 DB나 스토리지로 이동 가능
-- mysqldump logs_archive_2022 > logs_archive_2022.sql

-- 장점:
-- EXCHANGE는 메타데이터만 변경 → 즉각 (데이터 복사 없음)
-- 아카이브 테이블에 데이터 보존 (DROP이 아님)
-- 규정 준수 (데이터 보존 기간) + 성능 유지 동시 달성
```

### 5. 파티션 자동화 배치

```sql
-- 매월 새 파티션 추가 배치 (Event Scheduler)
DELIMITER //
CREATE EVENT add_monthly_partition
ON SCHEDULE EVERY 1 MONTH
STARTS '2024-12-01 00:00:00'
DO
BEGIN
    SET @next_month = DATE_ADD(LAST_DAY(NOW()), INTERVAL 1 DAY);
    SET @next_next_month = DATE_ADD(LAST_DAY(DATE_ADD(NOW(), INTERVAL 1 MONTH)), INTERVAL 1 DAY);
    
    SET @sql = CONCAT(
        'ALTER TABLE logs REORGANIZE PARTITION pmax INTO (',
        'PARTITION p', DATE_FORMAT(@next_month, '%Y%m'), 
        ' VALUES LESS THAN (''', @next_next_month, '''),',
        'PARTITION pmax VALUES LESS THAN (MAXVALUE)',
        ')'
    );
    
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END //
DELIMITER ;

-- 주의: Event Scheduler 활성화 필요
SET GLOBAL event_scheduler = ON;
```

---

## 💻 실전 실험

### 실험 1: REORGANIZE PARTITION (MAXVALUE 분할)

```sql
-- MAXVALUE 파티션 있는 테이블
CREATE TABLE reorg_test (
    id INT AUTO_INCREMENT,
    ts DATE NOT NULL,
    PRIMARY KEY (id, ts)
)
PARTITION BY RANGE COLUMNS (ts) (
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)  -- 모든 미래 데이터
);

-- 2025년 파티션 분리
ALTER TABLE reorg_test REORGANIZE PARTITION pmax INTO (
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);

-- 파티션 확인
SELECT PARTITION_NAME, PARTITION_DESCRIPTION
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_NAME = 'reorg_test'
ORDER BY PARTITION_ORDINAL_POSITION;
-- p2023, p2024, pmax 3개 파티션 확인

DROP TABLE reorg_test;
```

### 실험 2: DROP PARTITION vs DELETE 속도 비교

```sql
-- 실험 (실제 대량 데이터 환경에서 측정)
CREATE TABLE perf_part (
    id  BIGINT AUTO_INCREMENT,
    ts  DATE NOT NULL,
    val VARCHAR(100),
    PRIMARY KEY (id, ts)
)
PARTITION BY RANGE COLUMNS (ts) (
    PARTITION p2022 VALUES LESS THAN ('2023-01-01'),
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);

-- 100만 건 삽입 (p2022에)
INSERT INTO perf_part (ts, val)
SELECT
    DATE_ADD('2022-01-01', INTERVAL seq DAY),
    REPEAT('x', 50)
FROM (SELECT @r := @r + 1 AS seq FROM information_schema.columns a, information_schema.columns b, (SELECT @r := 0) r LIMIT 365) d
CROSS JOIN (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8) m;

-- DELETE 방식 측정
SET @t = SYSDATE(6);
DELETE FROM perf_part WHERE ts < '2023-01-01' LIMIT 100000;
-- 전체 삭제가 오래 걸리므로 LIMIT으로 부분 측정
SELECT CONCAT('DELETE 100K: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

-- DROP PARTITION 측정
SET @t = SYSDATE(6);
ALTER TABLE perf_part DROP PARTITION p2022;
SELECT CONCAT('DROP PARTITION: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');
-- 밀리초 수준 vs DELETE 수 초 차이 확인

DROP TABLE perf_part;
```

### 실험 3: EXCHANGE PARTITION

```sql
CREATE TABLE ex_main (
    id INT AUTO_INCREMENT,
    ts DATE NOT NULL,
    val VARCHAR(50),
    PRIMARY KEY (id, ts)
)
PARTITION BY RANGE COLUMNS (ts) (
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);

INSERT INTO ex_main (ts, val) VALUES
('2023-05-01', 'old1'), ('2023-08-15', 'old2'),
('2024-01-10', 'new1'), ('2024-06-20', 'new2');

-- 아카이브 테이블 (동일 구조, 파티션 없음)
CREATE TABLE ex_archive LIKE ex_main;
ALTER TABLE ex_archive REMOVE PARTITIONING;

-- p2023 파티션 데이터를 아카이브 테이블로 이동
ALTER TABLE ex_main EXCHANGE PARTITION p2023 WITH TABLE ex_archive;

-- 확인
SELECT 'ex_main' AS tbl, id, ts, val FROM ex_main
UNION ALL
SELECT 'ex_archive', id, ts, val FROM ex_archive;
-- ex_main: p2024, pmax 데이터
-- ex_archive: 2023 데이터

DROP TABLE ex_main, ex_archive;
```

---

## 📊 성능/비용 비교

```
파티션 운영 작업별 비용:

ADD PARTITION (빈 파티션 추가):
  Online DDL 가능 (ALGORITHM=INPLACE)
  실행 시간: 밀리초 ~ 수 초
  잠금: 메타데이터 잠금만 (DDL 기간)
  서비스 영향: 거의 없음

REORGANIZE PARTITION (MAXVALUE 분리):
  분리되는 파티션에 데이터가 있으면 재구성 필요
  실행 시간: 파티션 내 데이터 양에 비례
  잠금: 해당 파티션 잠금 (다른 파티션 영향 최소)
  권고: 정기 배치로 미리 빈 파티션 분리

DROP PARTITION:
  실행 시간: 밀리초 (데이터 양 무관)
  잠금: 해당 파티션만 (짧은 메타데이터 잠금)
  Undo Log 없음
  서비스 영향: 거의 없음

EXCHANGE PARTITION:
  메타데이터 교환만 → 밀리초
  실행 시간: 데이터 양 무관
  잠금: 두 테이블 메타데이터 잠금 (짧음)
  서비스 영향: 거의 없음

기존 테이블 파티셔닝 추가 (ALTER TABLE PARTITION BY):
  전체 테이블 재구성 필요
  실행 시간: 테이블 크기 × 디스크 속도
  1억 건: 30분 ~ 2시간
  잠금: COPY 알고리즘은 DML 차단
  서비스 영향: 크림 → pt-osc 사용 권장
```

---

## ⚖️ 트레이드오프

```
파티션 운영 전략 트레이드오프:

MAXVALUE 파티션 유지 vs 없음:
  MAXVALUE 있음:
    ✅ INSERT 실패 없음 (미래 데이터 자동 수용)
    ✅ 장애 방지 (파티션 추가 잊어도 안전)
    ❌ MAXVALUE 분리 시 REORGANIZE 필요 (데이터 있으면 비용)

  MAXVALUE 없음:
    ✅ 새 파티션 ADD PARTITION으로 간단히 추가
    ❌ 파티션 추가 전 새 데이터 INSERT 실패 → 서비스 장애 위험

  권고: MAXVALUE 항상 유지 + 정기 배치로 분리

DROP vs EXCHANGE vs TRUNCATE:
  DROP PARTITION:
    빠름, 데이터 영구 삭제, 되돌릴 수 없음
  EXCHANGE PARTITION:
    빠름, 데이터 보존 (아카이브 테이블에 이동)
    규정 준수 데이터 보존에 유리
  TRUNCATE PARTITION:
    DROP보다 느리지만 구조 유지
    파티션 삭제 후 재추가 없이 재사용 가능

자동화 vs 수동 관리:
  Event Scheduler:
    ✅ 파티션 관리 자동화
    ❌ 오류 시 모니터링 없으면 장애
    → 실행 결과 알람 필수

  외부 배치 (Cron + Script):
    ✅ 더 세밀한 제어, 알람 통합 쉬움
    ❌ 별도 배치 인프라 필요
```

---

## 📌 핵심 정리

```
파티션 운영 핵심:

MAXVALUE 파티션:
  항상 유지 (INSERT 실패 방지)
  정기적으로 REORGANIZE로 새 파티션 분리
  REORGANIZE 시 pmax가 비어있으면 빠름

파티션 삭제 (TTL):
  ALTER TABLE logs DROP PARTITION p_old;
  → 밀리초 (데이터 양 무관)
  DELETE보다 수백~수천 배 빠름

아카이브 패턴:
  EXCHANGE PARTITION으로 데이터 보존 + 성능 유지
  아카이브 테이블 → 별도 스토리지로 이동 가능

기존 테이블 파티셔닝:
  소량: 직접 ALTER TABLE
  대량: pt-osc 또는 새 테이블 생성 + 이관 + RENAME
  ALGORITHM=COPY 기본 → 쓰기 차단 주의

운영 자동화:
  Event Scheduler 또는 외부 배치로 월별 파티션 추가 자동화
  실행 결과 알람 필수 (파티션 추가 실패 → 다음 달 INSERT 실패)

현황 확인:
  SELECT * FROM INFORMATION_SCHEMA.PARTITIONS
  WHERE TABLE_NAME = 'logs'
  ORDER BY PARTITION_ORDINAL_POSITION;
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 시나리오에서 어떤 파티션 운영 방법을 선택하겠는가?

```
상황:
  - 3년치 로그 데이터, 월별 RANGE 파티션 (36개 파티션)
  - 규정 상 3년간 데이터 보존 필요 (삭제 불가)
  - 3년 이후 데이터는 빠른 조회 필요 없음 (감사 목적만)
  - 현재 테이블 크기가 커서 성능 저하 시작
```

<details>
<summary>해설 보기</summary>

**EXCHANGE PARTITION + 별도 아카이브 테이블 패턴을 선택합니다.**

**이유**: 규정 상 데이터 삭제 불가 → DROP PARTITION 불가. 성능 개선을 위해 오래된 데이터를 활성 테이블에서 분리해야 함.

**절차**:

```sql
-- 1. 아카이브 테이블 생성
CREATE TABLE logs_archive LIKE logs;
ALTER TABLE logs_archive REMOVE PARTITIONING;
-- 파티션 없는 일반 테이블 (조회 빈도 낮으므로 파티션 불필요)

-- 2. 가장 오래된 파티션 교환 (예: p202101, 2021년 1월)
ALTER TABLE logs EXCHANGE PARTITION p202101 WITH TABLE logs_archive_202101;
-- 즉각 완료 (메타데이터 교환)

-- 3. 교환된 파티션 DROP (빈 파티션 제거 - 구조 정리)
ALTER TABLE logs DROP PARTITION p202101;

-- 4. 아카이브 테이블은 별도 DB/스토리지 이동 후 보존
-- mysqldump logs_archive_202101 > logs_archive_202101.sql
```

**장점**:
- 데이터 보존 (규정 준수)
- 활성 테이블 크기 감소 → 성능 개선
- EXCHANGE는 즉각적 → 서비스 영향 없음
- 아카이브 테이블은 읽기 전용으로 필요 시 조회 가능

</details>

---

**Q2.** 현재 1억 건의 비파티션 테이블을 무중단으로 파티션 테이블로 전환하는 절차를 단계별로 설명하라.

<details>
<summary>해설 보기</summary>

**전체 절차 (pt-online-schema-change 활용)**:

**전제 조건**:
- FOREIGN KEY 제거 (파티션 테이블 FK 불가)
- PK 재설계 (파티션 키 포함)
- FK 제거 후 무결성 보장 방법 준비

**단계별 절차**:

```bash
# 1단계: FK 제거
ALTER TABLE orders DROP FOREIGN KEY fk_user_id;
ALTER TABLE orders DROP FOREIGN KEY fk_product_id;

# 2단계: pt-osc로 파티션 테이블 생성 및 데이터 이관
pt-online-schema-change \
  --host=localhost \
  --user=dba \
  --database=prod \
  --table=orders \
  --alter="
    DROP PRIMARY KEY,
    ADD PRIMARY KEY (id, created_at),
    PARTITION BY RANGE COLUMNS (created_at) (
      PARTITION p2022 VALUES LESS THAN ('2023-01-01'),
      PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
      PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
      PARTITION pmax VALUES LESS THAN (MAXVALUE)
    )
  " \
  --chunk-size=5000 \
  --sleep=0.5 \  # 청크 간 슬립으로 부하 조절
  --execute

# 3단계: pt-osc 완료 후 검증
# - 데이터 건수 비교
# - 샘플 Row 비교
# - EXPLAIN으로 쿼리 실행계획 확인

# 4단계: 애플리케이션 무결성 보장 코드 배포
# (FK 대신 서비스 레이어에서 참조 무결성 확인)
```

**주의사항**:
- pt-osc는 트리거 기반이므로 원본 테이블의 DML이 new 테이블에 동기화됨
- `--chunk-size`와 `--sleep`으로 DB 부하 조절 필수
- 1억 건이면 수 시간 소요 → 충분한 디스크 여유 공간 필요 (임시 테이블)
- 완료 직전 짧은 메타데이터 락 발생 → 서비스 중단 최소화 (밀리초~수 초)

</details>

---

<div align="center">

**[⬅️ 파티션의 함정](./05-partition-pitfalls.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 Replication ➡️](../replication/01-binlog-replication-basics.md)**

</div>
