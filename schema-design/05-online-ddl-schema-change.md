# 대용량 스키마 변경 — Online DDL vs pt-online-schema-change 비교

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- MySQL 8.0 Online DDL이 COPY/INPLACE/INSTANT 알고리즘 중 하나를 선택하는 기준은?
- `ALGORITHM=INSTANT`가 메타데이터만 변경하는 방식과 지원되지 않는 DDL 유형은?
- pt-osc가 트리거 기반으로 Shadow 테이블을 동기화하는 방식은?
- 운영 중 스키마 변경 시 어떤 방법을 언제 선택해야 하는가?
- Online DDL 중 발생할 수 있는 장애 상황과 대처 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 운영 중 ALTER TABLE = 서비스 장애 위험

```
실제 이슈:
  이슈 1: ALTER TABLE 중 DML 차단
    ALTER TABLE orders ADD INDEX idx_status(status);
    → MySQL 5.6 이하: 테이블 락 → orders 전체 차단
    → 수억 건 테이블: 수 시간 서비스 중단

  이슈 2: Online DDL 중 디스크 공간 부족
    ALTER TABLE orders ADD COLUMN memo TEXT;
    → INPLACE 알고리즘: 임시 파일 생성
    → 100GB 테이블 → 100GB 임시 파일 필요
    → 디스크 여유 없으면 → DDL 실패 + 테이블 손상 위험

  이슈 3: pt-osc FK 제약으로 실패
    ALTER TABLE order_items ... (FK 있는 테이블)
    pt-osc → FK 때문에 트리거 설치 실패
    → 별도 절차 필요

올바른 이해:
  DDL 알고리즘별 동작 차이 이해
  테이블 크기 × 서비스 특성별 최적 방법 선택
```

---

## 😱 흔한 실수 (Before)

### 1. Online DDL 없이 운영 중 ALTER TABLE 실행

```sql
-- Before: MySQL 5.5 이하 방식으로 운영 중 실행
ALTER TABLE orders ADD COLUMN delivery_memo TEXT;
-- 내부적으로:
-- 1. 전체 테이블 복사 (orders → orders_tmp)
-- 2. orders에 MDL(Metadata Lock) 획득 (DML 차단!)
-- 3. 복사 완료 후 orders_tmp → orders 이름 교체
-- 100GB 테이블 = 수 시간 DML 차단 = 서비스 중단

-- Online DDL이 지원되는 MySQL 5.6+에서도 알고리즘 확인 없이 실행
ALTER TABLE orders ADD COLUMN delivery_memo TEXT;
-- INPLACE로 처리되면 OK, 하지만 확인하지 않으면 예상 못한 COPY 발생 가능
```

### 2. ALGORITHM=INSTANT 지원 여부 확인 없이 사용

```sql
-- Before: 모든 DDL에 INSTANT 시도
ALTER TABLE orders
  ADD COLUMN tracking_number VARCHAR(50),
  ALGORITHM=INSTANT;

-- INSTANT 지원 안 되는 경우 (예: 컬럼 순서 변경):
ALTER TABLE orders
  MODIFY COLUMN user_id BIGINT FIRST,
  ALGORITHM=INSTANT;
-- Error 1846: ALGORITHM=INSTANT is not supported.

-- 안전한 방법:
-- INSTANT 시도 → 실패 시 INPLACE → 실패 시 COPY
-- 또는 사전에 MySQL 문서에서 지원 여부 확인
```

### 3. pt-osc FK 테이블에서 오류 무시

```bash
# Before: FK 있는 테이블에 pt-osc 그냥 실행
pt-online-schema-change \
  --alter "ADD COLUMN memo TEXT" \
  --execute D=mydb,t=order_items

# Error:
# The original table has foreign keys ...
# pt-osc가 Shadow 테이블에 트리거 설치 불가 (FK 제약)
# 또는 FK 무시 옵션으로 실행 시 정합성 문제

# 올바른 방법:
# FK 있는 테이블: MySQL Online DDL (INPLACE) 사용
# 또는 gh-ost (트리거 없이 Binary Log 기반으로 동기화)
```

---

## ✨ 올바른 접근 (After)

```
DDL 방법 선택 흐름:

1단계: INSTANT 가능 여부 확인
  ADD COLUMN (끝에 추가), DROP COLUMN, RENAME COLUMN 등
  → INSTANT: 메타데이터만 변경 → 즉각, 락 없음

2단계: INPLACE 가능 여부 확인
  ADD INDEX, MODIFY COLUMN (일부), ADD/DROP COLUMN (INSTANT 불가 시)
  → INPLACE: 임시 테이블 없이 원본 테이블에서 진행
  → DML 허용 (Concurrent DML), 단 임시 파일 필요

3단계: COPY 불가피할 때
  컬럼 타입 변경 (BIGINT → VARCHAR 등)
  → COPY: 전체 테이블 복사 → 서비스 중 실행 금지 (대형 테이블)
  → pt-osc 또는 gh-ost로 대체

도구별 적합 상황:
  Online DDL INSTANT: 메타데이터 변경 (즉각)
  Online DDL INPLACE: 인덱스 추가/삭제 (DML 허용)
  pt-osc: FK 없는 테이블, 레거시 환경
  gh-ost: FK 있는 테이블, 높은 TPS 환경
```

---

## 🔬 내부 동작 원리

### 1. ALGORITHM=INSTANT 동작

```
INSTANT 알고리즘:
  메타데이터(데이터 딕셔너리)만 변경
  실제 데이터 파일(.ibd) 수정 없음

  지원되는 DDL (MySQL 8.0):
    ADD COLUMN (테이블 끝에 추가, ROW_FORMAT=DYNAMIC)
    DROP COLUMN
    RENAME COLUMN
    ADD/DROP GENERATED COLUMN (VIRTUAL)
    CHANGE INDEX VISIBILITY
    MODIFY COLUMN (일부 제한)

  지원 안 되는 DDL:
    ADD COLUMN (테이블 중간에 삽입, FIRST 또는 AFTER)
    CHANGE COLUMN (타입 변경)
    ADD PRIMARY KEY
    대부분의 다른 DDL

  동작 방식:
    InnoDB 내부: Row에 '새 컬럼 없음' 마커
    새 컬럼 조회 시: 기본값 반환
    새 Row 삽입 시: 새 컬럼 포함하여 저장
    → 즉각 완료, 락 없음, 디스크 추가 없음

  ROW_FORMAT=DYNAMIC 확인:
    SHOW TABLE STATUS WHERE Name='orders'\G
    Row_format: Dynamic → INSTANT 지원
    Row_format: Compact → 일부 제한
```

### 2. ALGORITHM=INPLACE 동작

```
INPLACE 알고리즘:
  원본 테이블을 임시 테이블 없이 직접 수정
  대신 임시 로그 파일(Innodb Alter Log)에 변경 기록

  인덱스 추가 시:
    Phase 1: Read 단계
      원본 테이블 Full Scan → 인덱스 키 수집
      동시에 DML은 임시 Alter Log에 기록
    Phase 2: Sort + Build 단계
      수집된 키 정렬 → 인덱스 B-Tree 생성
    Phase 3: Apply Log 단계
      Phase 1~2 중 발생한 DML을 Alter Log에서 새 인덱스에 반영
      이 단계에서 짧은 MDL 획득 (수 초)

  DML 허용 여부:
    대부분의 INPLACE DDL: DML(SELECT/INSERT/UPDATE/DELETE) 허용
    일부: DML 제한 (ADD PRIMARY KEY 등)
    MDL은 DDL 시작/종료 시만 짧게 획득

  임시 파일 크기:
    인덱스 추가: 인덱스 크기만큼 임시 파일
    ROW REBUILD 필요한 DDL: 테이블 크기만큼 임시 파일
    → 디스크 여유 공간 확인 필수!

주의:
  LOCK=NONE이 기본 (DML 허용)
  LOCK=SHARED: 읽기만 허용
  LOCK=EXCLUSIVE: 모든 DML 차단
```

### 3. pt-online-schema-change (pt-osc) 동작

```
pt-osc 처리 흐름:

1단계: Shadow 테이블 생성
  CREATE TABLE orders_new LIKE orders;
  ALTER TABLE orders_new [변경사항 적용];
  → orders_new: 변경된 스키마의 빈 테이블

2단계: 트리거 설치 (원본 테이블에)
  AFTER INSERT ON orders → orders_new에도 INSERT
  AFTER UPDATE ON orders → orders_new에도 UPDATE
  AFTER DELETE ON orders → orders_new에도 DELETE
  → 트리거: 원본에 발생하는 DML을 orders_new에 동기화

3단계: 기존 데이터 청크 복사
  SELECT * FROM orders WHERE id BETWEEN X AND Y;
  INSERT INTO orders_new SELECT ...;
  (청크 단위, 수천~수만 건씩 반복)
  → 서비스 부하 조절하며 점진적 복사

4단계: 완료 확인
  orders_new에 모든 데이터 동기화 완료

5단계: 테이블 교체
  RENAME TABLE orders TO orders_old, orders_new TO orders;
  → 원자적 이름 교체 (수 밀리초 락)

6단계: 정리
  DROP TABLE orders_old;
  트리거 제거

pt-osc의 한계:
  FK 있는 테이블: Shadow 테이블에 트리거 설치 불가
  트리거 이미 있는 테이블: 트리거 중복 설치 불가
  높은 TPS: 트리거 오버헤드로 성능 영향 가능
```

### 4. gh-ost (GitHub's Online Schema Change) 동작

```
gh-ost vs pt-osc 차이:
  pt-osc: 트리거 기반 동기화
  gh-ost: Binary Log 기반 동기화 (트리거 없음)

gh-ost 처리 흐름:
  1. Shadow 테이블 생성 (pt-osc와 동일)
  2. Binary Log 읽기 시작 (Replica 처럼 연결)
     → 원본 테이블에 발생하는 모든 변경을 Binary Log로 수신
  3. 기존 데이터 청크 복사 (pt-osc와 동일)
  4. Binary Log 이벤트를 Shadow 테이블에 실시간 적용
  5. 동기화 완료 후 테이블 교체

gh-ost 장점:
  트리거 없음 → FK 있는 테이블도 가능 (주의 필요)
  Binary Log 기반 → 트리거 오버헤드 없음
  Throttle 제어 정밀 (Binary Log 처리 속도 조절)
  Pause/Resume 가능 (pt-osc 불가)
  테스트 모드: Shadow 테이블만 생성, 실제 교체 없음

사용법:
gh-ost \
  --host=prod-db \
  --database=mydb \
  --table=orders \
  --alter="ADD COLUMN memo TEXT" \
  --allow-on-master \  # Replica 없으면
  --execute
```

### 5. DDL 방법별 비교표

```
100GB 테이블 기준 비교:

DDL: ADD INDEX idx_status(status)
  Online DDL INPLACE:
    시간: 30~60분 (인덱스 빌드)
    DML: 허용 (Concurrent)
    락: DDL 시작/종료 시 수 초
    디스크: 인덱스 크기 추가 (약 10~20GB)

DDL: ADD COLUMN memo TEXT (끝에 추가)
  Online DDL INSTANT:
    시간: < 1초
    DML: 허용 (즉각)
    락: 거의 없음
    디스크: 없음

DDL: MODIFY COLUMN amount DECIMAL(15,4) [컬럼 타입 변경]
  Online DDL COPY:
    시간: 1~3시간 (전체 복사)
    DML: 금지 (서비스 중단)
    디스크: 100GB 임시 파일 필요

  pt-osc 또는 gh-ost:
    시간: 1~3시간 (청크 복사)
    DML: 허용 (트리거/Binary Log 동기화)
    디스크: 100GB Shadow 테이블 필요
    복잡도: 높음 (FK 주의)
```

---

## 💻 실전 실험

### 실험 1: ALGORITHM=INSTANT 동작 확인

```sql
-- INSTANT 지원 확인
CREATE TABLE instant_test (
    id  BIGINT AUTO_INCREMENT PRIMARY KEY,
    val INT
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;

INSERT INTO instant_test (val) VALUES (1),(2),(3);

-- INSTANT 컬럼 추가 (끝에)
ALTER TABLE instant_test
  ADD COLUMN new_col VARCHAR(50),
  ALGORITHM=INSTANT;
-- 즉각 완료 확인 (수 밀리초)

-- 기존 Row의 new_col 확인
SELECT id, val, new_col FROM instant_test;
-- new_col = NULL (기본값)

-- INSTANT 불가 케이스 (중간 삽입)
ALTER TABLE instant_test
  ADD COLUMN mid_col VARCHAR(50) AFTER val,
  ALGORITHM=INSTANT;
-- Error 1846 확인

DROP TABLE instant_test;
```

### 실험 2: Online DDL vs COPY 시간 비교

```sql
-- 대용량 테이블 DDL 시간 측정 (테스트 환경에서만)
CREATE TABLE ddl_perf (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    val1 INT, val2 INT, val3 INT
) ENGINE=InnoDB;

-- 1만 건 데이터 삽입
INSERT INTO ddl_perf (val1, val2, val3)
SELECT RAND()*1000, RAND()*1000, RAND()*1000
FROM (SELECT @r:=@r+1 FROM information_schema.columns, (SELECT @r:=0) r LIMIT 10000) t;

-- INSTANT
SET @t = SYSDATE(6);
ALTER TABLE ddl_perf ADD COLUMN new1 INT, ALGORITHM=INSTANT;
SELECT CONCAT('INSTANT: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

-- INPLACE (인덱스 추가)
SET @t = SYSDATE(6);
ALTER TABLE ddl_perf ADD INDEX idx_val1(val1), ALGORITHM=INPLACE;
SELECT CONCAT('INPLACE: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

-- COPY (강제)
SET @t = SYSDATE(6);
ALTER TABLE ddl_perf ADD COLUMN new2 INT, ALGORITHM=COPY;
SELECT CONCAT('COPY: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

DROP TABLE ddl_perf;
```

### 실험 3: pt-osc 기본 사용법

```bash
# pt-osc 설치 후 테스트 (Docker MySQL 권장)

# Dry-run (실제 변경 없이 계획만 확인)
pt-online-schema-change \
  --user=root --password=secret \
  --host=localhost \
  --database=testdb \
  --table=orders \
  --alter="ADD COLUMN delivery_memo TEXT" \
  --dry-run

# 실제 실행
pt-online-schema-change \
  --user=root --password=secret \
  --host=localhost \
  --database=testdb \
  --table=orders \
  --alter="ADD COLUMN delivery_memo TEXT" \
  --chunk-size=5000 \      # 청크 크기
  --sleep=0.5 \            # 청크 간 슬립 (DB 부하 조절)
  --progress=time,30 \     # 30초마다 진행률 출력
  --execute

# FK 있는 테이블 (오류 확인 후 gh-ost 사용 결정)
pt-online-schema-change \
  --alter="ADD COLUMN note TEXT" \
  --dry-run \
  D=testdb,t=order_items
# Error: 'order_items' has foreign keys...
```

---

## 📊 성능/비용 비교

```
방법별 성능 비교 (100GB 테이블):

ALGORITHM=INSTANT (ADD COLUMN 끝에):
  소요 시간: < 1초
  서비스 영향: 없음 (MDL 수 밀리초)
  디스크 추가: 없음
  적용: 컬럼 추가 (끝에), 컬럼 삭제, 컬럼 이름 변경

ALGORITHM=INPLACE (ADD INDEX):
  소요 시간: 30~120분 (데이터 크기에 비례)
  서비스 영향: 없음 (DML 허용)
  디스크 추가: 인덱스 크기 (10~30GB)
  MDL: DDL 시작/종료 시 수 초
  적용: 인덱스 추가/삭제, 일부 컬럼 변경

ALGORITHM=COPY (강제 또는 필요 시):
  소요 시간: 1~3시간
  서비스 영향: 큼 (DML 차단 또는 제한)
  디스크 추가: 원본 크기 100GB 임시 파일
  적용: 컬럼 타입 변경, 컬럼 순서 변경

pt-osc / gh-ost:
  소요 시간: COPY와 유사 (점진적)
  서비스 영향: 최소 (트리거/Binary Log 동기화)
  디스크 추가: Shadow 테이블 100GB
  복잡도: 높음 (모니터링 필요)
```

---

## ⚖️ 트레이드오프

```
Online DDL vs pt-osc/gh-ost:

Online DDL (INSTANT/INPLACE):
  ✅ MySQL 내장 (별도 도구 없음)
  ✅ INSTANT: 메타데이터만, 즉각
  ✅ INPLACE: DML 허용, 안전
  ❌ COPY: 서비스 영향
  ❌ FK 있는 테이블 일부 DDL 어려움
  ❌ 임시 파일 크기 예측 어려움

pt-osc:
  ✅ DML 허용 (트리거 기반)
  ✅ 속도 조절 가능 (--sleep, --chunk-size)
  ✅ 검증된 도구 (Percona)
  ❌ FK 있는 테이블 어려움
  ❌ 트리거 이미 있는 테이블 불가
  ❌ Shadow 테이블 100% 추가 공간 필요
  ❌ 높은 TPS에서 트리거 오버헤드

gh-ost:
  ✅ 트리거 없음 (Binary Log 기반)
  ✅ FK 있는 테이블 처리 용이
  ✅ Pause/Resume 가능
  ✅ 높은 TPS 환경에서 안전
  ❌ Binary Log 설정 필요 (ROW 포맷)
  ❌ Replica 설정 권장 (Master에서 실행 시 추가 설정)
  ❌ pt-osc보다 복잡한 설정

선택 가이드:
  컬럼 끝에 추가 → INSTANT (즉각)
  인덱스 추가 → INPLACE (안전)
  FK 없는 테이블 컬럼 타입 변경 → pt-osc
  FK 있거나 높은 TPS → gh-ost
  확신 없으면 → gh-ost (더 안전)
```

---

## 📌 핵심 정리

```
Online DDL 핵심:

알고리즘 3가지:
  INSTANT: 메타데이터만 변경 (수 밀리초, 락 없음)
    지원: ADD COLUMN(끝), DROP COLUMN, RENAME COLUMN
  INPLACE: 원본 테이블에서 직접 (DML 허용, 수십 분)
    지원: ADD/DROP INDEX, 일부 컬럼 변경
  COPY: 임시 테이블 복사 (DML 차단, 수 시간)
    사용: 불가피한 경우에만 (pt-osc로 대체 권장)

사전 확인:
  디스크 여유 공간 (INPLACE: 인덱스 크기, COPY: 원본 크기)
  ALGORITHM=INSTANT으로 먼저 시도 → 실패 시 INPLACE
  대형 테이블 COPY → pt-osc 또는 gh-ost로 대체

pt-osc vs gh-ost:
  FK 없음 + 단순 → pt-osc
  FK 있음 + 높은 TPS → gh-ost

운영 중 DDL 원칙:
  1. EXPLAIN으로 영향도 파악
  2. 테스트 환경에서 먼저 수행 + 시간 측정
  3. 트래픽 낮은 시간대 실행
  4. 모니터링 준비 (Seconds_Behind_Source, DML 응답 시간)
  5. Rollback 계획 수립
```

---

## 🤔 생각해볼 문제

**Q1.** 운영 중인 5억 건 `orders` 테이블에 `delivery_zone VARCHAR(20)` 컬럼을 추가해야 한다. 어떤 방법을 선택하고, 실행 전 확인해야 할 사항은 무엇인가?

<details>
<summary>해설 보기</summary>

**방법 선택: ALGORITHM=INSTANT**

```sql
-- 먼저 ROW_FORMAT 확인
SHOW TABLE STATUS WHERE Name='orders'\G
-- Row_format: Dynamic → INSTANT 지원

-- 컬럼을 테이블 끝에 추가 (INSTANT 조건)
ALTER TABLE orders
  ADD COLUMN delivery_zone VARCHAR(20),
  ALGORITHM=INSTANT;
-- 수 밀리초 내 완료, DML 차단 없음
```

**실행 전 확인 사항**:
1. ROW_FORMAT이 Dynamic인지 확인 (INSTANT 지원 여부)
2. 컬럼을 테이블 끝에 추가하는지 (AFTER/FIRST 없이)
3. 테스트 환경에서 먼저 `ALGORITHM=INSTANT`로 시도
4. 만약 INSTANT 불가 판단이 나면:
   - INPLACE 가능 여부 확인
   - COPY 필요하면 pt-osc 또는 gh-ost로 전환

**만약 INSTANT 지원 안 되면 (예: 오래된 ROW_FORMAT)**:

```bash
# pt-osc 사용
pt-online-schema-change \
  --alter="ADD COLUMN delivery_zone VARCHAR(20)" \
  --chunk-size=10000 \
  --sleep=0.1 \
  --progress=time,60 \
  --execute D=mydb,t=orders
```

</details>

---

**Q2.** pt-osc와 gh-ost 모두 Shadow 테이블과 원본 테이블 간 동기화 방법이 다르다. FK가 있는 테이블에서 pt-osc가 실패하는 이유를 설명하고, gh-ost로 해결할 수 있는 이유를 설명하라.

<details>
<summary>해설 보기</summary>

**pt-osc 실패 이유 (FK)**:

pt-osc는 원본 테이블에 AFTER INSERT/UPDATE/DELETE 트리거를 설치하여 Shadow 테이블을 동기화합니다. FK가 있는 테이블의 경우:

1. Shadow 테이블 생성 시 FK를 어떻게 처리할지 결정이 어려움
2. Shadow 테이블에 FK를 걸면 Shadow 테이블이 참조하는 테이블의 데이터가 없어 트리거 실행 시 FK 위반 오류
3. FK 없이 Shadow 테이블 생성하면 교체 후 정합성 문제

```
Error: The original table 'order_items' has foreign keys.
To alter this table, use --alter-foreign-keys-method.
```

**gh-ost 해결 이유**:

gh-ost는 트리거 없이 Binary Log를 직접 읽어 Shadow 테이블에 적용합니다. FK와 관계없이 Binary Log 이벤트를 Shadow 테이블에 적용하므로 FK 제약을 우회합니다.

```bash
# gh-ost: FK 있는 테이블도 처리
gh-ost \
  --database=mydb \
  --table=order_items \
  --alter="ADD COLUMN note TEXT" \
  --allow-on-master \
  --execute

# Binary Log로 동기화 → 트리거 불필요 → FK 제약 없음
# Shadow 테이블에 FK 없이 생성 후 데이터 복사 + Binary Log 적용
# 교체 직전 최종 동기화 확인 후 RENAME
```

단, gh-ost 사용 시에도 Shadow 테이블에 FK가 없는 상태로 운영되므로 교체 후 FK 무결성 검증이 권장됩니다.

</details>

---

<div align="center">

**[⬅️ AUTO_INCREMENT 함정](./04-auto-increment-pitfalls.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 모니터링 ➡️](../monitoring-diagnosis/01-performance-schema-guide.md)**

</div>
