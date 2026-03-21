# Binary Log 포맷 3가지 — STATEMENT vs ROW vs MIXED

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- STATEMENT 포맷에서 `NOW()`, `UUID()`가 복제 불일치를 일으키는 이유는?
- ROW 포맷에서 대용량 UPDATE가 Binary Log를 폭증시키는 이유는?
- MIXED 포맷이 두 방식을 전환하는 기준은?
- `binlog_row_image=MINIMAL`이 ROW 포맷 크기를 줄이는 방법은?
- 어떤 상황에서 어떤 포맷을 선택하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### Binary Log 포맷이 잘못되면 복제 불일치 또는 스토리지 폭발

```
이슈 1: STATEMENT 포맷 복제 불일치
  Source: INSERT INTO logs VALUES (UUID(), NOW());
  Binary Log(STATEMENT): "INSERT INTO logs VALUES (UUID(), NOW())"
  Replica: 이 SQL을 재실행하면?
    → UUID(), NOW()가 Replica 실행 시점에 다시 평가됨
    → Source와 다른 UUID, 다른 시간 값 저장
    → 복제 불일치! Replica 데이터 오염

이슈 2: ROW 포맷 Binary Log 폭증
  UPDATE orders SET status='PAID' WHERE created_at < '2020-01-01';
  → 500만 건 UPDATE
  Binary Log(ROW): 500만 건의 Before+After Row 데이터 기록
  → Binary Log 파일 수십 GB 생성
  → 디스크 꽉 참 → MySQL 중단

포맷 선택이 데이터 안전성과 운영 비용을 결정한다
```

---

## 😱 흔한 실수 (Before)

### 1. STATEMENT 포맷에서 비결정적 함수 사용

```sql
-- STATEMENT 포맷 환경에서
INSERT INTO users (uuid, created_at) VALUES (UUID(), NOW());

-- Binary Log에 기록되는 것:
-- INSERT INTO users (uuid, created_at) VALUES (UUID(), NOW())
-- (실제 값이 아닌 함수 호출 자체)

-- Replica에서 재실행:
-- UUID() → 다른 UUID 생성
-- NOW() → Replica 재생 시점의 시간
-- → Source와 Replica 데이터 불일치!

-- MySQL의 경고:
-- [Warning] Unsafe statement written to binary log
-- using statement format since BINLOG_FORMAT = STATEMENT.
-- The statement is unsafe because it uses a system function
-- that may return a different value on the slave.
```

### 2. ROW 포맷에서 무심코 대용량 DML

```sql
-- ROW 포맷 환경에서 배치 작업
UPDATE orders SET is_archived = 1
WHERE created_at < '2020-01-01';
-- 1,000만 건 업데이트

-- Binary Log에 기록되는 것:
-- Row #1: before=(id=1, is_archived=0, ...) after=(id=1, is_archived=1, ...)
-- Row #2: before=(id=2, is_archived=0, ...) after=(id=2, is_archived=1, ...)
-- ... 1,000만 건
-- Binary Log 크기: 1행당 ~100바이트 × 2(before+after) × 1,000만 = ~2GB

-- 결과: 2GB Binary Log 기록 → 전송 → Replica Lag 폭증
```

---

## ✨ 올바른 접근 (After)

```
포맷 특성 이해:

STATEMENT (명령문 기반):
  기록: 실행된 SQL 문 자체
  크기: 작음 (SQL 텍스트)
  장점: Binary Log 크기 최소, 사람이 읽기 쉬움
  단점: 비결정적 함수/표현식 → 복제 불일치 위험
  사용: 단순 INSERT/UPDATE, 비결정적 요소 없을 때

ROW (행 기반):
  기록: 변경된 각 Row의 Before + After 값
  크기: 변경 Row 수에 비례 (대량 DML 시 매우 큼)
  장점: 항상 정확한 데이터 복제 (비결정적 함수 문제 없음)
  단점: Binary Log 크기 큼, 배치 DML 시 폭증
  사용: 기본값 (MySQL 5.7.7 이후), 정확성 우선

MIXED (혼합):
  기본: STATEMENT 방식 사용
  비결정적 감지 시: ROW 방식으로 자동 전환
  장점: 크기 효율 + 안전성 균형
  단점: 전환 조건이 복잡, 예측 어려움
  사용: 레거시 환경 호환성 필요 시
```

---

## 🔬 내부 동작 원리

### 1. STATEMENT 포맷에서 비결정적 함수 문제

```sql
-- 비결정적 함수 목록 (복제 불일치 위험):
NOW(), SYSDATE(), CURRENT_TIMESTAMP()  -- 시간
UUID(), UUID_SHORT()                    -- 고유 식별자
USER(), CURRENT_USER()                  -- 사용자
RAND()                                  -- 난수
@@server_id, @@hostname                 -- 서버 변수
LOAD_FILE()                             -- 파일 읽기

-- STATEMENT 포맷에서 안전한 패턴:
-- 비결정적 함수를 변수에 먼저 할당 후 사용
SET @uuid = UUID();
SET @now = NOW();
INSERT INTO users (uuid, created_at) VALUES (@uuid, @now);
-- Binary Log: SET @uuid='abc-def-...'; SET @now='2024-01-15 10:30:00'; INSERT ...
-- Replica에서 변수값이 그대로 재실행됨 → 안전

-- MySQL이 자동으로 경고하는 경우:
-- BINLOG_FORMAT=STATEMENT에서 비결정적 함수 사용 시
-- "Unsafe statement" 경고 발생
-- → MIXED로 전환하거나 ROW 포맷 사용 권장
```

### 2. ROW 포맷 기록 방식과 binlog_row_image

```
ROW 포맷 기본 기록 (binlog_row_image=FULL):
  Before image: 변경 전 Row의 모든 컬럼 값
  After image: 변경 후 Row의 모든 컬럼 값

  UPDATE orders SET status='PAID' WHERE id=1:
  Before: {id:1, user_id:100, status:'PENDING', amount:50000, created_at:'2024-01-01', ...}
  After:  {id:1, user_id:100, status:'PAID',    amount:50000, created_at:'2024-01-01', ...}

binlog_row_image=MINIMAL (MySQL 5.6+):
  Before image: PK 컬럼만 (Row 식별에 필요한 최소)
  After image: 변경된 컬럼만

  UPDATE orders SET status='PAID' WHERE id=1:
  Before: {id:1}                          ← PK만
  After:  {status:'PAID'}                 ← 변경된 컬럼만

  크기: FULL 대비 ~50~90% 감소
  장점: Binary Log 크기 대폭 감소
  단점: 일부 복구 도구에서 Before 정보 부족 (pt-table-checksum 등)

binlog_row_image=NOBLOB:
  Before image: BLOB/TEXT 제외 모든 컬럼
  After image: 변경된 컬럼 (BLOB/TEXT 포함)
  대형 BLOB이 있는 테이블에서 유용

설정:
  SET GLOBAL binlog_row_image = 'MINIMAL';
  또는 my.cnf: binlog_row_image = MINIMAL
```

### 3. MIXED 포맷의 전환 기준

```
MIXED 포맷에서 ROW로 전환되는 조건:

① 비결정적 함수 사용:
   NOW(), UUID(), RAND() 등 포함된 DML
   → ROW로 전환

② 비결정적 표현식:
   User-defined functions, 일부 트리거
   → ROW로 전환

③ 테이블에 AUTO_INCREMENT 컬럼이 있고 TRIGGER가 있을 때:
   INSERT... SELECT 패턴
   → ROW로 전환

④ MIXED에서도 STATEMENT 유지되는 경우:
   단순 INSERT VALUES (상수만)
   WHERE 조건이 결정적인 UPDATE/DELETE

확인 방법:
  SHOW BINLOG EVENTS → Event_type 컬럼
  Query: STATEMENT 기록
  Table_map + Write_rows/Update_rows/Delete_rows: ROW 기록

MIXED 설정:
  SET GLOBAL binlog_format = 'MIXED';
```

### 4. 각 포맷의 Binary Log 내용 비교

```sql
-- 예시 쿼리:
UPDATE orders SET status = 'PAID', updated_at = NOW()
WHERE id BETWEEN 1 AND 3;

-- STATEMENT 포맷 Binary Log:
-- Event: Query
-- SQL: UPDATE orders SET status = 'PAID', updated_at = NOW()
--      WHERE id BETWEEN 1 AND 3;
-- 크기: ~100바이트 (SQL 길이)

-- ROW 포맷 Binary Log:
-- Event: Table_map (테이블 메타데이터)
-- Event: Update_rows
--   Row 1: before=(id:1, status:'PENDING', updated_at:'2024-01-01 09:00:00', ...)
--           after= (id:1, status:'PAID',    updated_at:'2024-01-15 10:30:00', ...)
--   Row 2: before=(id:2, ...) after=(id:2, ...)
--   Row 3: before=(id:3, ...) after=(id:3, ...)
-- 크기: Row 수 × Row 크기 (3 × ~200바이트 = ~600바이트)
-- NOW()가 실제 값으로 기록됨 → Replica에서 정확히 재현

-- mysqlbinlog로 확인:
-- mysqlbinlog --base64-output=DECODE-ROWS -v mysql-bin.000001
```

### 5. DDL과 Binary Log

```sql
-- DDL은 포맷과 무관하게 항상 STATEMENT로 기록
ALTER TABLE orders ADD INDEX idx_status(status);
-- Binary Log: "ALTER TABLE orders ADD INDEX idx_status(status)"
-- ROW 포맷이어도 DDL은 SQL 문으로 기록

-- DDL 복제의 특성:
-- Replica에서 동일한 ALTER TABLE 재실행
-- 대용량 테이블 ALTER TABLE → Replica에서도 동일 시간 소요
-- → Replica Lag의 주요 원인 중 하나

-- Online DDL + 복제:
-- Source에서 Online DDL (ALGORITHM=INPLACE): 빠름
-- Replica에서 재생 시: 같은 Online DDL 실행
-- 하지만 Source만큼 빠르지 않을 수 있음 (자원 차이)
```

---

## 💻 실전 실험

### 실험 1: 포맷별 Binary Log 크기 비교

```sql
-- 현재 포맷 확인
SHOW VARIABLES LIKE 'binlog_format';

-- 테스트 테이블
CREATE TABLE format_test (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    status     VARCHAR(20),
    memo       TEXT,
    updated_at DATETIME
) ENGINE=InnoDB;

INSERT INTO format_test (status, memo, updated_at)
SELECT 'PENDING', REPEAT('x', 100), NOW()
FROM information_schema.columns LIMIT 10000;

-- Binary Log 위치 기록
SHOW BINARY LOG STATUS\G  -- 현재 파일 + 위치

-- STATEMENT 포맷으로 UPDATE
SET SESSION binlog_format = 'STATEMENT';
SET @start_pos = (SELECT File_size FROM information_schema.FILES WHERE ... ); -- 현재 크기
UPDATE format_test SET status = 'PAID', updated_at = NOW();
-- 이후 Binary Log 크기 증가량 확인

-- ROW 포맷으로 같은 UPDATE (원복 후)
UPDATE format_test SET status = 'PENDING';
SET SESSION binlog_format = 'ROW';
UPDATE format_test SET status = 'PAID', updated_at = NOW();
-- Binary Log 크기 증가량 비교 → ROW가 훨씬 큼

-- Binary Log 이벤트 확인
SHOW BINLOG EVENTS IN 'mysql-bin.XXXXXX' FROM [위치]\G

DROP TABLE format_test;
SET SESSION binlog_format = DEFAULT;
```

### 실험 2: STATEMENT 포맷 비결정적 함수 경고

```sql
SET SESSION binlog_format = 'STATEMENT';

CREATE TABLE unsafe_test (
    id UUID DEFAULT (UUID()),
    ts DATETIME DEFAULT NOW(),
    val INT
);

-- 비결정적 함수 포함 INSERT
INSERT INTO unsafe_test (val) VALUES (1);
-- Warning: Unsafe statement written to the binary log...

-- MIXED 포맷에서는 자동으로 ROW 전환
SET SESSION binlog_format = 'MIXED';
INSERT INTO unsafe_test (val) VALUES (2);
SHOW BINLOG EVENTS...; -- Table_map + Write_rows 이벤트 확인 (ROW)

DROP TABLE unsafe_test;
SET SESSION binlog_format = DEFAULT;
```

### 실험 3: binlog_row_image 크기 비교

```sql
-- FULL vs MINIMAL 크기 비교
CREATE TABLE image_test (
    id     INT AUTO_INCREMENT PRIMARY KEY,
    col1   VARCHAR(100),
    col2   VARCHAR(100),
    col3   VARCHAR(100),
    col4   VARCHAR(100),
    col5   VARCHAR(100)
) ENGINE=InnoDB;

INSERT INTO image_test (col1, col2, col3, col4, col5)
SELECT REPEAT('a',50), REPEAT('b',50), REPEAT('c',50), REPEAT('d',50), REPEAT('e',50)
FROM information_schema.columns LIMIT 1000;

SET SESSION binlog_format = 'ROW';

-- FULL 이미지
SET SESSION binlog_row_image = 'FULL';
FLUSH LOGS; -- 새 Binary Log 파일 시작
SET @pos_start = (SELECT @@global.binlog_cache_size); -- 간접적 확인
UPDATE image_test SET col1 = 'updated' WHERE id <= 100;
SHOW BINARY LOG STATUS; -- 파일 크기 확인

-- MINIMAL 이미지
SET SESSION binlog_row_image = 'MINIMAL';
FLUSH LOGS;
UPDATE image_test SET col1 = 'updated2' WHERE id <= 100;
SHOW BINARY LOG STATUS; -- 파일 크기 비교

DROP TABLE image_test;
SET SESSION binlog_row_image = DEFAULT;
```

---

## 📊 성능/비용 비교

```
포맷별 Binary Log 크기 비교 (UPDATE 10만 건, Row당 200바이트):

STATEMENT:
  Binary Log: ~200바이트 (SQL 문 하나)
  네트워크 전송: 매우 작음
  Replica Lag 영향: 최소

ROW (FULL):
  Binary Log: 10만 × 400바이트 = ~40MB
  네트워크 전송: 40MB
  Replica Lag 영향: 40MB 전송 + 재실행 시간

ROW (MINIMAL):
  Binary Log: 10만 × (PK + 변경컬럼) = ~5MB (추정)
  네트워크 전송: 5MB
  Replica Lag 영향: 중간

MIXED:
  비결정적 없으면 STATEMENT → 작음
  비결정적 있으면 ROW → 해당 트랜잭션만 큼

실무 권장:
  MySQL 8.0 기본: ROW + binlog_row_image=FULL
  대량 배치 DML이 잦은 환경: MINIMAL 고려
  정확성 최우선: ROW 유지
```

---

## ⚖️ 트레이드오프

```
포맷 선택 트레이드오프:

STATEMENT:
  ✅ Binary Log 크기 최소
  ✅ 사람이 읽기 쉬움 (PITR 분석 용이)
  ❌ 비결정적 함수 → 복제 불일치 위험
  ❌ 일부 DDL+DML 조합에서 위험
  현재: 사용 권장 안 함 (ROW가 기본값)

ROW:
  ✅ 항상 정확한 복제 보장
  ✅ MySQL 5.7.7+ 기본값 (검증된 선택)
  ❌ 대용량 DML 시 Binary Log 폭증
  ❌ 배치 작업 전 binlog_row_image=MINIMAL 고려 필요

MIXED:
  ✅ 크기와 정확성 균형
  ❌ 전환 기준 복잡 → 예측 어려움
  ❌ 일부 도구 호환성 문제
  현재: 레거시 환경 호환성 필요 시만 사용

binlog_row_image 설정:
  FULL (기본): pt-table-checksum 등 도구 호환 최대
  MINIMAL: Binary Log 크기 대폭 절감, 일부 도구 제한
  → 배치 DML이 빈번한 환경: MINIMAL 검토
  → 정밀한 복구/감사 필요: FULL 유지
```

---

## 📌 핵심 정리

```
Binary Log 포맷 핵심:

STATEMENT:
  SQL 문 자체 기록
  비결정적 함수 → 복제 불일치 위험
  현재는 레거시, ROW가 표준

ROW (권장):
  변경된 각 Row의 Before + After 기록
  항상 정확한 복제 보장
  대량 DML 시 Binary Log 크기 주의

MIXED:
  기본 STATEMENT + 비결정적 감지 시 ROW 자동 전환
  예측 어려움, 신규 환경에서는 ROW 직접 설정 권장

크기 최적화:
  binlog_row_image=MINIMAL → PK + 변경 컬럼만 기록
  대용량 배치 DML 전 고려
  단, 복구 도구 호환성 확인 필요

배치 DML 주의:
  ROW 포맷에서 수백만 건 DML → 수십 GB Binary Log 가능
  → LIMIT으로 청크 나누기 또는 작업 전 임시 binlog_format 변경 고려
```

---

## 🤔 생각해볼 문제

**Q1.** ROW 포맷에서 다음 배치 쿼리가 Binary Log를 폭증시키지 않도록 개선하라.

```sql
-- 3년 이전 주문 아카이브 처리
UPDATE orders SET is_archived = 1 WHERE created_at < '2021-01-01';
-- 예상: 500만 건 업데이트
```

<details>
<summary>해설 보기</summary>

**문제**: ROW 포맷에서 500만 건 × (Before + After Row) = 수십 GB Binary Log 생성 가능.

**개선 방법 1 — 청크 단위 처리**:
```sql
-- 10,000건씩 나눠서 처리
SET @done = 0;
REPEAT
    UPDATE orders SET is_archived = 1
    WHERE created_at < '2021-01-01'
    AND is_archived = 0
    LIMIT 10000;
    SET @done = (ROW_COUNT() = 0);
    DO SLEEP(0.1); -- Binary Log 전송 시간 확보, Replica Lag 완화
UNTIL @done END REPEAT;
```

**개선 방법 2 — binlog_row_image=MINIMAL 임시 적용**:
```sql
SET SESSION binlog_row_image = 'MINIMAL';
UPDATE orders SET is_archived = 1 WHERE created_at < '2021-01-01';
SET SESSION binlog_row_image = 'FULL';
-- Before: PK만, After: is_archived만 → 크기 대폭 감소
```

**개선 방법 3 — 파티셔닝 활용**:
created_at으로 파티셔닝된 경우 `ALTER TABLE orders DROP PARTITION p2020`으로 Binary Log 없이 대량 삭제/아카이브 가능.

실무에서는 방법 1 (청크 단위 처리)이 가장 안전합니다. Replica Lag을 실시간 모니터링하며 `SLEEP()` 간격을 조절합니다.

</details>

---

<div align="center">

**[⬅️ Replication 아키텍처](./01-replication-architecture-binlog.md)** | **[홈으로 🏠](../README.md)** | **[다음: GTID 기반 복제 ➡️](./03-gtid-based-replication.md)**

</div>
