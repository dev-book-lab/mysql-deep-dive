<div align="center">

# ⚙️ MySQL Deep Dive

**"MySQL을 운영하는 것과, 장애 상황에서 원인을 찾고 튜닝하는 것은 다르다"**

<br/>

> *"EXPLAIN을 읽을 줄 알지만 실행계획을 개선할 줄 모르고, 파티셔닝을 걸었는데 왜 빨라지지 않는지 모르고, Replication Lag이 왜 발생하는지 설명 못한다면 — MySQL을 아직 절반만 알고 있는 것이다"*

서브쿼리가 내부에서 어떻게 세미조인으로 변환되는지, 파티션 프루닝이 동작하지 않는 이유, Binary Log 포맷 3가지의 실질적 차이, `mysqldump`와 XtraBackup이 백업 원리 자체가 다른 이유까지  
**실무 장애 상황에서 왜 이게 문제가 되는가** 라는 질문으로 MySQL 운영과 튜닝을 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=flat-square&logo=mysql&logoColor=white)](https://dev.mysql.com/doc/)
[![Prerequisite](https://img.shields.io/badge/선행-database--internals-orange?style=flat-square&logo=mysql&logoColor=white)](https://github.com/dev-book-lab/database-internals)
[![Docs](https://img.shields.io/badge/Docs-38개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

[database-internals](https://github.com/dev-book-lab/database-internals)에서 InnoDB 내부 구조를 배웠다면, 이제 다음 질문이 남습니다.

> **"그래서, 실무에서 어떻게 쓰나?"**

| 이미 아는 것 (database-internals) | 이 레포에서 배우는 것 |
|----------------------------------|----------------------|
| B-Tree 인덱스 구조, Clustered Index 원리 | 서브쿼리가 어떻게 Join으로 변환되는가, GROUP BY가 인덱스를 타는 조건 |
| EXPLAIN의 type, key, rows, Extra 읽기 | `type: ALL → ref`, `Extra: Using filesort 제거` 실전 개선 사례 |
| MVCC, Undo Log, 트랜잭션 격리 수준 | 파티셔닝이 MVCC와 어떻게 충돌하는가, Online DDL이 Lock을 최소화하는 방식 |
| Binary Log 개념 | ROW vs STATEMENT 포맷 차이가 Replication Lag에 미치는 영향 |
| InnoDB 락 종류 (S/X/Gap/Next-Key) | Metadata Lock 대기가 운영 장애로 이어지는 시나리오 |
| WAL, Redo Log 내구성 원리 | `--single-transaction`이 없으면 mysqldump가 왜 일관성을 깨는가 |

---

## ⚠️ 선행 학습

이 레포는 **[database-internals](https://github.com/dev-book-lab/database-internals) 완료를 전제**로 합니다.

아래 개념을 모르면 이 레포의 내용을 제대로 흡수하기 어렵습니다:

```
필수 선행 지식:
  ✅ InnoDB B-Tree 인덱스 구조 (Clustered / Secondary Index 차이)
  ✅ EXPLAIN 기본 읽기 (type, key, rows, Extra 컬럼 해석)
  ✅ MVCC — Read View, Undo Log 버전 체인
  ✅ 트랜잭션 격리 수준 4가지 (구현 원리까지)
  ✅ InnoDB 락 종류 — S/X/IS/IX/Gap/Next-Key Lock
  ✅ WAL — Write-Ahead Logging과 Redo Log
```

모른다면 → [database-internals](https://github.com/dev-book-lab/database-internals) 먼저

---

## 🚀 빠른 시작

지금 당장 풀어야 할 문제가 있다면:

[![Query](https://img.shields.io/badge/🔹_느린쿼리-서브쿼리_최적화부터-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./query-optimization/01-subquery-optimization.md)
[![Partition](https://img.shields.io/badge/🔹_파티셔닝-프루닝이_안되는_이유-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./partitioning/03-partition-pruning.md)
[![Replication](https://img.shields.io/badge/🔹_Replication-Lag_원인_분석-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./replication/04-replication-lag-analysis.md)
[![Backup](https://img.shields.io/badge/🔹_백업복구-PITR_특정시점_복구-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./backup-recovery/03-pitr-point-in-time-recovery.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Query Optimization 심화

> **핵심 질문:** 옵티마이저는 IN 서브쿼리를 어떻게 Join으로 바꾸는가? GROUP BY와 ORDER BY가 인덱스를 타는 정확한 조건은? 형변환이 어떻게 인덱스를 무력화하는가?

<details>
<summary><b>서브쿼리 변환부터 실행계획 개선 사례까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 서브쿼리 최적화 — IN/EXISTS/스칼라 서브쿼리의 내부 변환](./query-optimization/01-subquery-optimization.md) | IN 서브쿼리가 Dependent Subquery로 처리될 때 O(n²)이 되는 원리, EXISTS vs IN의 실행 비용 차이가 데이터 분포에 따라 달라지는 이유, 스칼라 서브쿼리가 매 Row마다 실행되는 조건과 캐시 동작 방식 |
| [02. 세미조인(Semi-Join) 최적화 — IN 서브쿼리가 Join이 되는 원리](./query-optimization/02-semijoin-optimization.md) | Optimizer가 IN 서브쿼리를 Semi-Join으로 변환하는 조건, Semi-Join 전략 5가지(Materialization, FirstMatch, LooseScan, DuplicateWeedout, Table pullout) 각각의 동작 방식과 EXPLAIN에서 식별하는 법 |
| [03. 파생 테이블(Derived Table)과 CTE — 임시 테이블이 생성되는 조건](./query-optimization/03-derived-table-cte.md) | FROM 절 서브쿼리(파생 테이블)가 디스크 임시 테이블을 생성하는 조건, CTE(WITH절)가 Materialized vs Merged로 처리되는 판단 기준, `EXPLAIN`의 `select_type: DERIVED`와 `MATERIALIZED` 구분 |
| [04. GROUP BY / ORDER BY 최적화 — 인덱스가 정렬을 대체하는 조건](./query-optimization/04-groupby-orderby-optimization.md) | `Extra: Using filesort`가 발생하는 정확한 조건, 인덱스 순서와 ORDER BY 방향이 일치해야 Filesort를 피할 수 있는 이유, GROUP BY + ORDER BY + LIMIT의 조합에서 인덱스가 동작하는 경우와 그렇지 않은 경우 |
| [05. 윈도우 함수 실행 원리와 성능 특성](./query-optimization/05-window-function-internals.md) | 윈도우 함수가 내부적으로 정렬 + 임시 테이블을 사용하는 방식, PARTITION BY와 ORDER BY가 실행계획에 미치는 영향, ROW_NUMBER() vs RANK() vs DENSE_RANK()의 비용 차이, 집계 쿼리 대비 성능 트레이드오프 |
| [06. 문자셋과 콜레이션 — 조인 시 묵시적 형변환이 인덱스를 망치는 이유](./query-optimization/06-charset-collation-implicit-conversion.md) | utf8mb4 vs utf8 컬럼을 JOIN할 때 인덱스가 무력화되는 원리, 숫자형 PK와 문자열 FK 조인 시 형변환 비용, `CONVERT()` 없이 콜레이션 불일치를 해결하는 방법, JPA 엔티티 설계에서 자주 발생하는 형변환 함정 |
| [07. 실행계획 개선 사례 — type: ALL → ref, Using filesort 제거](./query-optimization/07-explain-improvement-cases.md) | Full Table Scan이 발생하는 5가지 패턴과 인덱스로 해결하는 방법, `EXPLAIN ANALYZE` 전후 비교로 개선 효과를 수치화하는 방법론, 실제 슬로우 쿼리 3가지 케이스의 진단 → 원인 → 해결 전 과정 |

</details>

<br/>

### 🔹 Chapter 2: 파티셔닝 (Partitioning)

> **핵심 질문:** 파티션을 걸었는데 왜 쿼리가 빨라지지 않는가? 프루닝이 동작하지 않는 정확한 조건은? 파티션과 인덱스는 어떻게 함께 동작하는가?

<details>
<summary><b>파티셔닝 판단 기준부터 운영 전략까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 파티셔닝이 필요한 상황과 오해 — 인덱스와 파티셔닝은 다르다](./partitioning/01-when-to-use-partitioning.md) | 파티셔닝이 인덱스를 대체하지 않는 이유, 수억 건 이상의 테이블에서 파티션이 효과를 내는 원리(파티션 별 독립 B-Tree), 파티셔닝이 오히려 느려지는 시나리오, 데이터 보관 주기(TTL) 삭제에 파티션이 효과적인 이유 |
| [02. 파티션 종류 (RANGE, LIST, HASH, KEY)](./partitioning/02-partition-types-range-list-hash-key.md) | RANGE 파티션이 날짜/ID 범위 쿼리에 적합한 이유, LIST 파티션의 Enum 값 분산 방식, HASH vs KEY 파티션의 분산 알고리즘 차이, 각 종류별 프루닝 가능 여부와 실행계획에서 확인하는 방법 |
| [03. 파티션 프루닝 — 쿼리가 파티션을 건너뛰는 조건](./partitioning/03-partition-pruning.md) | Partition Pruning이 동작하는 정확한 조건(파티션 키 컬럼에 직접 비교 연산 필요), 함수 적용/형변환/OR 조건이 프루닝을 막는 원리, `EXPLAIN PARTITIONS`로 프루닝 여부를 확인하는 방법, 프루닝이 안 될 때 ALL 파티션을 스캔하는 비용 |
| [04. 파티션과 인덱스 — 로컬 인덱스 vs 글로벌 인덱스](./partitioning/04-partition-and-index.md) | MySQL이 글로벌 인덱스를 지원하지 않는 이유, 파티션별 로컬 인덱스의 구조와 파티션 키가 인덱스 컬럼에 포함되어야 하는 이유, 비파티션 키 컬럼으로 조회 시 발생하는 Full Partition Scan 문제 |
| [05. 파티션의 함정 — PRIMARY KEY 제약, 외래키 불가, 파티션 키 실수](./partitioning/05-partition-pitfalls.md) | 파티션 테이블에서 PRIMARY KEY에 파티션 키를 반드시 포함해야 하는 이유, 외래키(FOREIGN KEY)가 파티션 테이블에서 지원되지 않는 근본적 이유, 파티션 키를 잘못 선택했을 때 파티션 쏠림(Hot Partition) 현상 |
| [06. 파티션 운영 — 추가/삭제/재구성, 마이그레이션](./partitioning/06-partition-operations.md) | `ALTER TABLE ... ADD PARTITION` 시 Online DDL 가능 여부, `DROP PARTITION`이 `DELETE`보다 빠른 이유(파일 수준 삭제), 기존 테이블에 파티션을 추가하는 `PARTITION BY` 마이그레이션 절차와 잠금 범위 |

</details>

<br/>

### 🔹 Chapter 3: Replication 원리

> **핵심 질문:** Binary Log 포맷 3가지는 실제로 어떻게 다른가? Replication Lag의 원인을 어디서 찾는가? GTID는 왜 장애 복구를 단순하게 만드는가?

<details>
<summary><b>Replication 아키텍처부터 Spring Read/Write 분리까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. MySQL Replication 아키텍처 — Source/Replica와 Binary Log 흐름](./replication/01-replication-architecture-binlog.md) | Source의 Binary Log → Replica의 IO Thread → Relay Log → SQL Thread 재생 전 과정, Binary Log가 InnoDB Redo Log와 다른 이유, 비동기 복제에서 Replica가 뒤처지는 구조적 이유 |
| [02. Binary Log 포맷 3가지 — STATEMENT vs ROW vs MIXED](./replication/02-binlog-formats-statement-row-mixed.md) | STATEMENT 포맷이 비결정적 함수(`NOW()`, `UUID()`)에서 복제 불일치를 일으키는 원리, ROW 포맷이 모든 변경된 Row를 기록하므로 대용량 UPDATE에서 Binary Log가 폭증하는 이유, MIXED가 두 포맷을 전환하는 기준 |
| [03. GTID 기반 복제 — 장애 복구 시 왜 GTID가 중요한가](./replication/03-gtid-based-replication.md) | GTID(Global Transaction ID)가 `server_uuid:transaction_id` 형식으로 트랜잭션을 전역 식별하는 방식, 기존 포지션 기반 복제에서 Failover 시 Binary Log 위치를 수동으로 찾아야 하는 문제, GTID로 Replica 전환이 자동화되는 원리 |
| [04. Replication Lag 발생 원인 — IO Thread vs SQL Thread 병목 분석](./replication/04-replication-lag-analysis.md) | `SHOW REPLICA STATUS`의 `Seconds_Behind_Source` 지표의 정확한 의미와 오해, IO Thread 병목(네트워크/Binary Log 수신)과 SQL Thread 병목(Relay Log 재생 속도)을 구분하는 방법, 대용량 트랜잭션이 Lag을 발생시키는 구조 |
| [05. Semi-Synchronous Replication — 동기/비동기 복제의 트레이드오프](./replication/05-semi-sync-replication.md) | 비동기 복제에서 Source 장애 시 커밋된 트랜잭션이 유실되는 시나리오, Semi-Sync가 최소 1개 Replica의 Relay Log 수신을 확인 후 커밋 완료로 처리하는 방식, `rpl_semi_sync_source_timeout` 타임아웃 시 비동기로 자동 강등되는 조건 |
| [06. 병렬 복제(Parallel Replication) — Replica가 지연을 따라잡는 메커니즘](./replication/06-parallel-replication.md) | 단일 SQL Thread가 Source의 병렬 트랜잭션을 직렬로 재생할 때 Lag이 누적되는 구조적 문제, `slave_parallel_workers` 설정과 `LOGICAL_CLOCK` 방식으로 독립 트랜잭션을 병렬 재생하는 원리, 병렬 복제 시 트랜잭션 순서 보장 방식 |
| [07. Spring에서 Read/Write 분리 — AbstractRoutingDataSource와 Replica 지연 대응](./replication/07-spring-read-write-separation.md) | `AbstractRoutingDataSource`로 `@Transactional(readOnly=true)`를 Replica로 라우팅하는 구현 패턴, Replica Lag으로 인한 Stale Read 문제와 `@Transactional` 전파 설정으로 방어하는 전략, Source 장애 시 Failover와 Spring DataSource 재설정 |

</details>

<br/>

### 🔹 Chapter 4: 백업과 복구

> **핵심 질문:** mysqldump와 XtraBackup은 원리가 어떻게 다른가? Binary Log로 어디까지 복구 가능한가? RTO/RPO란 무엇이고 백업 주기를 어떻게 결정하는가?

<details>
<summary><b>논리 백업부터 실전 장애 복구 시나리오까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. mysqldump — 논리적 백업의 동작 원리와 한계](./backup-recovery/01-mysqldump-internals.md) | `--single-transaction`이 없으면 왜 일관성 없는 백업이 생성되는가(REPEATABLE READ 스냅샷 원리), mysqldump가 스키마와 데이터를 SQL로 덤프하는 과정, 대용량 테이블에서 논리 백업의 속도·크기 한계와 복구 시간 |
| [02. XtraBackup — 물리적 백업의 원리, InnoDB Hot Backup이 가능한 이유](./backup-recovery/02-xtrabackup-hot-backup.md) | XtraBackup이 InnoDB 데이터 파일을 복사하면서 동시에 Redo Log를 기록해 일관성을 보장하는 방식, `--prepare` 단계에서 불완전한 트랜잭션을 롤백하고 Redo Log를 적용하는 Crash Recovery 과정, mysqldump 대비 백업·복구 속도 차이 |
| [03. Binary Log를 이용한 PITR — 특정 시각으로 복구](./backup-recovery/03-pitr-point-in-time-recovery.md) | Full Backup + Binary Log를 조합해 특정 시각(또는 특정 트랜잭션 직전)으로 복구하는 전체 절차, `mysqlbinlog --start-datetime` / `--stop-position` 옵션으로 원하는 구간만 재생하는 방법, GTID 환경에서 PITR 수행 시 주의사항 |
| [04. 복구 전략 설계 — RTO/RPO 개념과 백업 주기 결정](./backup-recovery/04-recovery-strategy-rto-rpo.md) | RTO(복구 목표 시간)와 RPO(데이터 손실 허용 범위)가 백업 주기와 방식 선택에 미치는 영향, Full Backup 주기 + Binary Log 보관 기간의 트레이드오프 계산, 서비스 규모별 현실적인 백업 전략 설계 기준 |
| [05. 실전 장애 복구 시나리오 — 실수로 DELETE된 테이블 복구 과정](./backup-recovery/05-disaster-recovery-scenario.md) | `DROP TABLE` 또는 대량 `DELETE` 이후 복구 가능한 범위와 불가능한 범위, Full Backup에서 복원 → Binary Log로 해당 시점까지 재생 → 문제 트랜잭션 직전에 멈추는 전체 복구 과정 실습, 복구 시간 단축을 위한 사전 준비 체크리스트 |

</details>

<br/>

### 🔹 Chapter 5: 스키마 설계와 데이터 타입

> **핵심 질문:** DATETIME과 TIMESTAMP는 언제 무엇을 써야 하는가? JSON 컬럼에 인덱스를 거는 방법은? 대용량 스키마 변경을 무중단으로 하는 방법은?

<details>
<summary><b>데이터 타입 선택부터 Online DDL까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 데이터 타입 선택 전략 — INT vs BIGINT, DATETIME vs TIMESTAMP, CHAR vs VARCHAR](./schema-design/01-data-type-selection.md) | BIGINT PK가 AUTO_INCREMENT 소진 시 발생하는 문제와 ID 전략 선택 기준, TIMESTAMP의 2038년 문제와 타임존 자동 변환 동작, VARCHAR vs CHAR 저장 방식 차이가 인덱스 크기와 성능에 미치는 영향 |
| [02. JSON 타입과 Generated Column — MySQL 8.0 JSON 인덱싱 원리](./schema-design/02-json-type-generated-column.md) | JSON 컬럼 자체에 인덱스를 직접 걸 수 없는 이유, Generated Column으로 JSON 경로를 가상 컬럼으로 추출해 인덱스를 거는 방법, `JSON_EXTRACT()` vs `->` 연산자 차이와 Functional Index 활용 |
| [03. 정규화 vs 비정규화 — 조인 비용과 데이터 중복의 트레이드오프](./schema-design/03-normalization-vs-denormalization.md) | 3NF 정규화가 조회 시 JOIN 비용을 증가시키는 원리, 비정규화로 데이터를 중복 저장할 때 UPDATE 정합성 위험과 관리 비용, 읽기 중심/쓰기 중심 서비스 특성에 따른 정규화 수준 선택 기준 |
| [04. AUTO_INCREMENT의 함정 — 갭(Gap), 재사용 불가, 분산 환경 한계](./schema-design/04-auto-increment-pitfalls.md) | 트랜잭션 롤백 후 AUTO_INCREMENT 값이 재사용되지 않아 Gap이 생기는 이유, 서버 재시작 후 InnoDB가 AUTO_INCREMENT 최대값을 재계산하는 방식(MySQL 8.0 개선 전/후 차이), 분산 환경에서 채번 충돌이 발생하는 시나리오와 대안(Snowflake ID, UUID) |
| [05. 대용량 스키마 변경 — Online DDL vs pt-online-schema-change 비교](./schema-design/05-online-ddl-schema-change.md) | MySQL 8.0 Online DDL이 Copy/In-place/Instant 알고리즘 중 하나를 선택하는 기준, `ALGORITHM=INSTANT`가 메타데이터만 변경하는 방식과 지원되지 않는 DDL 유형, pt-osc가 트리거 기반으로 Shadow 테이블을 점진적으로 동기화하는 방식 |

</details>

<br/>

### 🔹 Chapter 6: 성능 모니터링과 진단

> **핵심 질문:** 어떤 쿼리가 서비스를 느리게 만드는지 어떻게 찾는가? `SHOW ENGINE INNODB STATUS` 출력에서 무엇을 봐야 하는가? 운영 중 발생하는 문제 패턴을 어떻게 진단하는가?

<details>
<summary><b>Performance Schema부터 운영 장애 패턴까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Performance Schema 완전 가이드 — 어디서 얼마나 걸리는가](./monitoring-diagnosis/01-performance-schema-guide.md) | `events_statements_summary_by_digest`로 가장 느린/자주 실행된 쿼리를 추출하는 방법, `events_waits_summary` 테이블로 Lock 대기, IO 대기, 뮤텍스 병목을 찾는 쿼리, `setup_instruments`와 `setup_consumers` 설정으로 수집 범위 조정 |
| [02. sys 스키마 — 운영자를 위한 DBA 뷰 활용법](./monitoring-diagnosis/02-sys-schema-usage.md) | `sys.statements_with_full_table_scans`로 Full Scan 쿼리 즉시 확인, `sys.schema_unused_indexes`로 사용 안 되는 인덱스 제거 후보 추출, `sys.innodb_lock_waits`로 현재 Lock 대기 체인을 한 쿼리로 파악하는 방법 |
| [03. InnoDB 상태 분석 — SHOW ENGINE INNODB STATUS 완전 해석](./monitoring-diagnosis/03-innodb-status-analysis.md) | `TRANSACTIONS` 섹션에서 Lock 대기 중인 트랜잭션을 찾는 법, `BUFFER POOL AND MEMORY`에서 Buffer Pool 히트율과 오염된 Page 수를 읽는 법, `LATEST DETECTED DEADLOCK` 섹션에서 어떤 Lock이 충돌했는지 분석하는 방법 |
| [04. MySQL 8.0 새 기능 — 히스토그램, Invisible Index, 쿼리 힌트](./monitoring-diagnosis/04-mysql8-new-features.md) | 히스토그램이 Cardinality 통계를 보완해 불균등 데이터 분포에서 실행계획을 개선하는 원리, Invisible Index로 인덱스를 실제 삭제 없이 비활성화해 영향도를 측정하는 방법, `/*+ NO_HASH_JOIN */` 등 Optimizer Hint의 실전 활용 |
| [05. 운영 중 발생하는 문제 패턴 — Metadata Lock, Buffer Pool 부족, 디스크 풀](./monitoring-diagnosis/05-operational-problem-patterns.md) | DDL 대기 → Metadata Lock이 뒤따르는 모든 DML을 차단하는 연쇄 차단 패턴과 해결법, Buffer Pool 히트율 저하가 Disk I/O 폭증으로 이어지는 신호와 대응, 디스크 풀로 인한 Binary Log / Undo Tablespace 팽창 진단 |

</details>

<br/>

### 🔹 Chapter 7: 보안과 사용자 관리

> **핵심 질문:** 최소 권한 원칙을 MySQL에서 어떻게 설계하는가? 개발/운영 환경을 어떻게 분리하는가?

<details>
<summary><b>권한 설계부터 민감 데이터 관리까지 (3개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 사용자와 권한 설계 — 최소 권한 원칙, Role 기반 권한 관리](./security-user-management/01-user-privilege-design.md) | `GRANT` 단위를 DB/테이블/컬럼 레벨로 세분화하는 방법, MySQL 8.0 Role로 권한 묶음을 관리하는 방식, 애플리케이션 계정에 DML만 허용하고 DDL을 차단해야 하는 이유 |
| [02. 연결 보안 — SSL/TLS, 비밀번호 정책, 감사 로그](./security-user-management/02-connection-security.md) | `require_secure_transport`로 암호화되지 않은 연결을 거부하는 설정, `validate_password` 플러그인의 비밀번호 복잡도 강제 방식, MySQL Enterprise Audit 또는 오픈소스 감사 로그로 접근 이력을 기록하는 방법 |
| [03. 개발/운영 환경 분리 전략 — 접근 제어, 마스킹, 민감 데이터 관리](./security-user-management/03-dev-prod-separation.md) | 개발 환경에서 운영 DB에 직접 접근하지 않도록 계정과 네트워크를 분리하는 설계, 개인정보/카드번호 등 민감 컬럼에 Dynamic Data Masking 적용 방법, 운영 데이터 마이그레이션 시 마스킹 처리 절차 |

</details>

---

## 🐳 실험 환경 (Docker Compose)

모든 챕터의 실험은 아래 환경에서 재현 가능합니다. Replication 관련 실험은 Source + Replica 구성을 사용합니다.

```yaml
# docker-compose.yml
services:
  mysql-source:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: deep_dive
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - ./source.cnf:/etc/mysql/conf.d/source.cnf
    ports:
      - "3306:3306"
    command: >
      --slow_query_log=ON
      --slow_query_log_file=/var/log/mysql/slow.log
      --long_query_time=0.1
      --general_log=ON
      --general_log_file=/var/log/mysql/general.log

  mysql-replica:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - ./replica.cnf:/etc/mysql/conf.d/replica.cnf
    ports:
      - "3307:3306"
    depends_on:
      - mysql-source
```

```ini
# source.cnf
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-format=ROW
gtid-mode=ON
enforce-gtid-consistency=ON
binlog-expire-logs-seconds=604800

# replica.cnf
[mysqld]
server-id=2
relay-log=relay-bin
gtid-mode=ON
enforce-gtid-consistency=ON
slave-parallel-workers=4
slave-parallel-type=LOGICAL_CLOCK
```

```sql
-- init.sql: 공통 실험 데이터
-- 쿼리 최적화: 100만 건 주문 데이터 (orders, order_items)
-- 파티셔닝: 날짜별 로그 테이블 (access_logs, payment_logs)
-- 백업/복구: 계정 이체 시나리오 (accounts, transfers)
-- 스키마: JSON 컬럼 포함 상품 테이블 (products, product_options)
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 실무에서 중요한가** | 실제 운영에서 마주치는 장애/문제 상황과의 연결 |
| 😱 **흔한 실수 (Before)** | 원리를 모를 때의 잘못된 운영/설계 패턴과 그 결과 |
| ✨ **올바른 접근 (After)** | 원리를 알고 난 후의 설계/운영 방식 |
| 🔬 **내부 동작 원리** | MySQL 내부 메커니즘 분석 + ASCII 구조도 |
| 💻 **실전 실험** | 재현 가능한 시나리오와 쿼리 (EXPLAIN 전후 비교 포함) |
| 📊 **성능/비용 비교** | 수치로 보는 방식별 차이 |
| ⚖️ **트레이드오프** | 이 설계/설정의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "느린 쿼리가 있고 당장 분석해야 한다" — 실전 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch6-01  Performance Schema로 슬로우 쿼리 추출
       Ch6-02  sys 스키마로 Full Scan 쿼리 즉시 확인
Day 2  Ch1-07  실행계획 개선 사례 — type: ALL → ref, filesort 제거
       Ch1-01  서브쿼리 최적화 — 의심되는 IN 쿼리 점검
Day 3  Ch1-04  GROUP BY / ORDER BY 인덱스 조건 점검
       Ch1-06  형변환 패턴 확인 — 문자셋 불일치 체크
```

</details>

<details>
<summary><b>🟡 "Replication Lag이 발생했다, 원인을 찾아야 한다" — 복제 장애 대응 (2일)</b></summary>

<br/>

```
Day 1  Ch3-01  Replication 아키텍처 전체 흐름 파악
       Ch3-04  Lag 원인 분석 — IO Thread vs SQL Thread 병목
       Ch3-02  Binary Log 포맷 — ROW 포맷 로그 폭증 가능성 확인
Day 2  Ch3-06  병렬 복제 설정으로 SQL Thread 병목 해소
       Ch3-03  GTID 설정 확인 — Failover 시나리오 대비
       Ch3-07  Spring Read/Write 분리 — Stale Read 방어 전략
```

</details>

<details>
<summary><b>🔴 "장애 복구 절차를 처음부터 설계해야 한다" — 백업/복구 전략 수립 (3일)</b></summary>

<br/>

```
Day 1  Ch4-04  RTO/RPO 개념 → 백업 전략 요구사항 정의
       Ch4-01  mysqldump 원리 — --single-transaction 이해
       Ch4-02  XtraBackup 원리 — Hot Backup 가능한 이유
Day 2  Ch4-03  PITR — Binary Log로 특정 시각 복구 실습
Day 3  Ch4-05  실전 시나리오 — DELETE 복구 전 과정 실습
       Ch6-03  INNODB STATUS로 복구 중 상태 모니터링
```

</details>

<details>
<summary><b>⚫ "처음부터 끝까지 완전 정복" — 전체 학습 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — 쿼리 최적화 심화
        → EXPLAIN ANALYZE로 각 서브쿼리 패턴별 실행계획 비교

2주차  Chapter 2 전체 — 파티셔닝
        → EXPLAIN PARTITIONS로 프루닝 동작/미동작 직접 확인

3주차  Chapter 3 전체 — Replication
        → Docker Compose Source + Replica 구성 → Lag 발생 → 병렬 복제로 해소

4주차  Chapter 4 전체 — 백업과 복구
        → Full Backup → 데이터 삭제 → PITR 복구 전 과정 실습

5주차  Chapter 5 전체 — 스키마 설계
        → Online DDL 알고리즘별 Lock 범위 비교

6주차  Chapter 6 전체 — 모니터링과 진단
        → SHOW ENGINE INNODB STATUS 실시간 분석 + 운영 장애 패턴 재현

7주차  Chapter 7 전체 — 보안과 사용자 관리
        → Role 기반 계정 설계 실습 → 환경별 접근 제어 구성
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [database-internals](https://github.com/dev-book-lab/database-internals) | InnoDB 내부 구조, 인덱스, MVCC, 락, EXPLAIN 기초 | **전 챕터 선행 필수** |
| [spring-data-transaction-deep-dive](https://github.com/dev-book-lab/spring-data-transaction-deep-dive) | JPA 영속성 컨텍스트, `@Transactional` 전파 | Ch1(EXPLAIN 심화), Ch3(Read/Write 분리) |
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | AOP Proxy, Bean 생명주기 | Ch3(`@Transactional`과 DataSource 라우팅) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Auto-configuration, Actuator | Ch6(HikariCP 설정, DataSource 모니터링) |

> 💡 이 레포는 **MySQL 운영과 튜닝**에 집중합니다. Spring을 모르더라도 Chapter 1~6은 순수 DB 관점으로 학습 가능합니다. 단, Chapter 3의 Read/Write 분리는 Spring Data JPA 지식이 있을 때 더 깊이 연결됩니다.

---

## 🙏 Reference

- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
- [Real MySQL 8.0 — 개발자와 DBA를 위한 MySQL 실전 가이드](https://wikibook.co.kr/realmysql801/)
- [High Performance MySQL, 4th Edition](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)
- [Percona Blog — MySQL Performance](https://www.percona.com/blog/)
- [MySQL EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [Percona XtraBackup Documentation](https://docs.percona.com/percona-xtrabackup/latest/)
- [MySQL Binary Log](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"MySQL을 운영하는 것과, 장애 상황에서 원인을 찾고 튜닝하는 것은 다르다"*

</div>
