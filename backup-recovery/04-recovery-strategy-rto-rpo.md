# 복구 전략 설계 — RTO/RPO 개념과 백업 주기 결정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- RTO(복구 목표 시간)와 RPO(데이터 손실 허용 범위)가 백업 주기에 어떤 영향을 미치는가?
- Full Backup 주기 + Binary Log 보관 기간의 트레이드오프 계산 방법은?
- 서비스 규모별 현실적인 백업 전략 설계 기준은?
- 복구 리허설이 필요한 이유는?
- 백업 비용과 복구 목표의 균형을 어떻게 잡는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### "백업은 있는데 복구가 안 된다" — RTO/RPO를 고려하지 않은 결과

```
실제 이슈:
  이슈 1: RPO 무시
    백업: 매주 일요일 1회
    장애: 금요일 오후
    → 5일치 데이터 유실 (Binary Log 없으면)
    → 서비스 제공사 SLA 위반, 고객 손해 배상

  이슈 2: RTO 무시
    백업: mysqldump로 매일 1회 (200GB DB)
    장애: 오전 10시 전체 DB 손상
    → mysqldump 복원: 12~20시간
    → 자정이 돼야 서비스 복구
    → 하루 서비스 중단

  이슈 3: 복구 리허설 없음
    백업 절차는 있지만 실제 복원한 적 없음
    실제 장애 시 복구 시도:
      Binary Log 위치 계산 실수
      파일 권한 오류
      복구 절차 문서 구식
    → 예상 2시간 → 실제 8시간

핵심:
  RTO/RPO는 비즈니스 요구사항
  백업 전략은 이를 달성하기 위한 기술 수단
  리허설 없이 설계만으로는 실제 장애 시 동작 보장 불가
```

---

## 😱 흔한 실수 (Before)

### 1. RTO/RPO 정의 없이 백업 주기 결정

```
Before: 기술 중심 결정
  "디스크가 1TB 있으니 매일 백업하자"
  "mysqldump가 쉬우니 이걸로 하자"

문제:
  DB 크기가 200GB로 늘어나면:
    mysqldump 복원: 15시간 → RTO 초과 자동 발생
  서비스가 금융 결제이면:
    하루 1회 백업 → RPO 23시간 → 허용 불가

올바른 접근:
  먼저: "장애 시 몇 시간 안에 복구해야 하는가?" (RTO)
  그 다음: "얼마만큼의 데이터 손실이 허용되는가?" (RPO)
  마지막: 이를 달성하는 가장 비용 효율적인 방법은?
```

### 2. 백업 파일 보존 후 복원 테스트 안 함

```bash
# Before: 백업만 하고 복원 테스트 없음

# 6개월 후 실제 장애 시:
mysql -u root -p mydb < backup_6months_ago.sql
# Error: Unknown character set: 'utf8mb4'  (버전 차이)
# Error: Table 'orders' doesn't exist (백업 시 일부 누락)
# Error: Cannot add foreign key constraint (복원 순서 문제)

# 테스트 없이는 "백업이 있다"가 아니라 "백업 파일이 있다"일 뿐
```

### 3. Binary Log 보관 없이 RTO/RPO 설정

```
Before: 매일 Full Backup + Binary Log 없음

RPO 목표: 1시간
실제 RPO: 최대 23시간 (전날 백업 시점)
→ RPO 달성 불가

Binary Log 활성화:
  binlog_expire_logs_seconds = 86400 * 7  # 7일
→ Full Backup(어제 2시) + Binary Log 재생
→ RPO: 수 분 (Binary Log 재생 간격) 달성 가능
```

---

## ✨ 올바른 접근 (After)

```
RTO/RPO 기반 백업 전략 설계 흐름:

1단계: 비즈니스 요구사항 정의
  RTO: "장애 발생 후 N시간 이내 복구"
  RPO: "최대 N시간치 데이터 손실 허용"
  → 서비스 SLA, 비즈니스 영향도로 결정

2단계: RTO 달성을 위한 백업 방식 선택
  RTO < 2시간 → XtraBackup (물리 백업) 필수
  RTO 2~8시간 → XtraBackup 또는 mydumper
  RTO > 8시간 → mysqldump 가능 (소규모)

3단계: RPO 달성을 위한 백업 주기 결정
  RPO < 1시간 → Binary Log + 빈번한 Full Backup
  RPO < 24시간 → Binary Log + 매일 Full Backup
  RPO < 1주일 → 매일 Full Backup
  RPO 무제한 → 주 1회 Full Backup

4단계: 비용 계산
  스토리지 비용, 백업 소요 시간, 인력 비용
  RTO/RPO vs 비용 균형점 결정

5단계: 복구 리허설 계획
  분기 1회 이상 실제 복구 수행
  복구 시간 측정, 절차 검증
```

---

## 🔬 내부 동작 원리

### 1. RTO와 RPO 정의

```
RTO (Recovery Time Objective):
  장애 발생부터 서비스 재개까지 허용되는 최대 시간
  "얼마나 빨리 복구해야 하는가?"

  예시:
    "DB 장애 후 2시간 이내 서비스 정상화"
    → RTO = 2시간

RPO (Recovery Point Objective):
  복구 시 얼마나 오래된 시점으로 복구해도 허용되는가
  "얼마만큼의 데이터 손실을 허용하는가?"

  예시:
    "최대 1시간치 데이터 손실 허용"
    → RPO = 1시간
    → 최소 1시간 이내 주기로 백업/Binary Log 필요

RTO vs RPO 관계:
  두 값 모두 낮을수록 → 비용 증가
  금융/결제: RTO 1시간, RPO 0 (무손실) → 고비용
  일반 웹서비스: RTO 4시간, RPO 1시간 → 중비용
  내부 도구: RTO 24시간, RPO 하루 → 저비용

MTTR (Mean Time To Recover):
  실제 평균 복구 시간
  복구 리허설로 MTTR을 측정하고 RTO와 비교
  MTTR > RTO → 즉시 개선 필요
```

### 2. Full Backup 주기와 Binary Log 보관 기간 계산

```
최적 Full Backup 주기 계산:

변수:
  DB 크기: D (GB)
  Binary Log 일일 생성량: B (GB/일)
  스토리지 예산: S (GB)
  RTO: T_rto (시간)
  RPO: T_rpo (시간)

RTO 기반 Full Backup 방식 결정:
  XtraBackup 복원 속도: 1~2 GB/분
  mysqldump 복원 속도: 0.1~0.3 GB/분

  RTO = 2시간 + D / 복원속도
  XtraBackup으로 100GB: 복원 ~1시간 + prepare ~30분 = 1.5시간 → OK
  mysqldump로 100GB: 복원 ~6시간 → RTO 2시간 달성 불가

Binary Log 보관 기간 최소값:
  Full Backup 주기 = F일
  Binary Log 보관 = F + 안전마진 일
  예: Full Backup 7일 → Binary Log 14일 이상

스토리지 계산:
  Full Backup 개수 보관 = N개
  스토리지 = D × N (Full Backup)
              + B × Binary Log 보관 기간
  예: 100GB DB, 주 1회 Full, 3개 보관, 14일 Binary Log(5GB/일)
      = 100 × 3 + 5 × 14 = 370GB 필요
```

### 3. 서비스 규모별 백업 전략

```
소규모 서비스 (DB 10GB 미만, 스타트업):
  RTO: 4~8시간
  RPO: 24시간

  전략:
    mysqldump --single-transaction --master-data=2 (매일)
    Binary Log: 7일 보관
    백업 스크립트를 cron으로 실행
    백업 파일 S3/GCS 자동 업로드

  월 비용: 스토리지 ~10GB/월 = $0.2

중규모 서비스 (DB 50~200GB, 스케일업 스타트업):
  RTO: 2~4시간
  RPO: 1~6시간

  전략:
    XtraBackup Full Backup (매일 새벽)
    XtraBackup Incremental (매 6시간)
    Binary Log: 14일 보관
    복구 서버 스펙: 프로덕션의 50% 이상

  월 비용: 스토리지 ~500GB, 복구 서버 대기 비용

대규모 서비스 (DB 500GB 이상, 기업):
  RTO: 30분~1시간
  RPO: 0~15분

  전략:
    XtraBackup Full Backup (매일)
    Semi-Sync Replication (데이터 유실 방지)
    Binary Log Streaming (별도 서버에 실시간 아카이브)
    Replica를 Warm Standby로 유지 (Failover 즉각)
    멀티 AZ 배포

  월 비용: 복제 서버 × 3, Binary Log 서버 등 상당한 비용
```

### 4. 복구 리허설 체계

```
복구 리허설 필요성:
  "백업이 있다 ≠ 복구가 된다"
  실제 복구 시 발견되는 문제들:
    ① my.cnf 설정 불일치 (innodb_buffer_pool_size 등)
    ② 백업 파일 권한 오류
    ③ 의존 패키지 버전 차이
    ④ GTID 설정 불일치
    ⑤ Binary Log 파일 누락 (아카이브 실패)
    ⑥ 복구 절차 문서가 구식

리허설 주기:
  최소: 분기 1회
  권장: 월 1회 (자동화 가능하면 더 빈번히)

리허설 항목:
  ① Full Backup 복원 (별도 서버)
  ② Binary Log 재생 (PITR 시뮬레이션)
  ③ 복구 시간 측정 (MTTR 기록)
  ④ 데이터 정합성 검증
  ⑤ 복구 절차 문서 업데이트

자동화 리허설:
  AWS RDS: 자동 복원 테스트 기능 제공
  GCP Cloud SQL: 유사 기능 제공
  자체 구축 환경: 스크립트 + CI/CD로 주기 자동화
  복구 완료 후 Slack/이메일 알림 + 검증 결과 기록
```

### 5. 3-2-1 백업 규칙

```
3-2-1 규칙:
  3개의 백업 복사본
  2가지 다른 미디어/스토리지
  1개는 오프사이트(지리적으로 다른 곳)

실무 적용:
  복사본 1: 로컬 디스크 (빠른 복원)
  복사본 2: 다른 서버/NAS (로컬 장애 대비)
  복사본 3: 클라우드 스토리지 S3/GCS (재해 대비)

클라우드 환경:
  복사본 1: 같은 AZ의 EBS/로컬 디스크
  복사본 2: 다른 AZ의 S3 버킷
  복사본 3: 다른 리전의 S3 버킷 (Cross-Region Replication)

Binary Log 아카이브:
  binlog_expire_logs_seconds로 DB 내 보관
  + mysqlbinlog --read-from-remote-server로 별도 서버에 스트리밍
  → DB 서버 장애로 Binary Log 유실 방지
```

---

## 💻 실전 실험

### 실험 1: RTO 측정 — 현재 백업 전략의 실제 RTO

```bash
# 현재 백업 방식의 실제 RTO 측정
# (별도 서버에서 수행, 운영 영향 없음)

# 1. 백업 파일 다운로드 시간 측정
time aws s3 cp s3://backup-bucket/latest.sql.gz /tmp/

# 2. 복원 시간 측정
time (
  gunzip -c /tmp/latest.sql.gz | \
  mysql -u root -p mydb_restore
)

# 3. Binary Log 재생 시간 측정 (최근 24시간치)
time (
  mysqlbinlog --skip-gtids \
    --start-position=[백업 시점 Position] \
    /backup/binlog/*.bin | \
  mysql -u root -p mydb_restore
)

# 총 RTO = 1 + 2 + 3 + (검증 시간)
# 측정값을 기록하고 RTO 목표와 비교
echo "RTO 측정 완료. 기록 필요."
```

### 실험 2: 복구 전략 비교 계산

```python
# Python으로 스토리지 비용 계산 (개념적)
# 실제 환경에 맞게 수정

db_size_gb = 100        # DB 크기 (GB)
binlog_daily_gb = 5     # 일일 Binary Log 생성량 (GB)

# 전략 A: mysqldump 매일 + Binary Log 7일
strategy_a = {
    "full_backup_count": 3,          # 3일치 보관
    "full_backup_size": db_size_gb * 2,  # 압축 전 2배
    "binlog_days": 7,
}
storage_a = (strategy_a["full_backup_size"] * strategy_a["full_backup_count"] * 0.5  # 압축 50%
             + binlog_daily_gb * strategy_a["binlog_days"])
print(f"전략 A 스토리지: {storage_a:.0f} GB")

# 전략 B: XtraBackup 매일 + Binary Log 14일
strategy_b = {
    "full_backup_count": 7,
    "full_backup_size": db_size_gb * 0.7,  # 압축 시 원본의 70%
    "binlog_days": 14,
}
storage_b = (strategy_b["full_backup_size"] * strategy_b["full_backup_count"]
             + binlog_daily_gb * strategy_b["binlog_days"])
print(f"전략 B 스토리지: {storage_b:.0f} GB")
```

### 실험 3: 복구 리허설 스크립트

```bash
#!/bin/bash
# 자동 복구 리허설 스크립트 (월 1회 실행)

RESTORE_HOST="restore-server"
BACKUP_PATH="/backup/latest/"
LOG_FILE="/var/log/recovery_drill_$(date +%Y%m%d).log"

echo "=== 복구 리허설 시작: $(date) ===" | tee -a $LOG_FILE

# 1. 복원 서버 준비
echo "1. 복원 서버 초기화" | tee -a $LOG_FILE
ssh $RESTORE_HOST "systemctl stop mysql && rm -rf /var/lib/mysql/*"

# 2. Full Backup 복원
START=$(date +%s)
echo "2. Full Backup 복원 시작" | tee -a $LOG_FILE
ssh $RESTORE_HOST "xtrabackup --copy-back --target-dir=$BACKUP_PATH && chown -R mysql:mysql /var/lib/mysql/ && systemctl start mysql"
RESTORE_TIME=$(($(date +%s) - START))
echo "   Full Backup 복원 시간: ${RESTORE_TIME}초" | tee -a $LOG_FILE

# 3. Binary Log 재생 (24시간치)
START=$(date +%s)
echo "3. Binary Log 재생 시작" | tee -a $LOG_FILE
# Binary Log 재생 명령 실행...
BINLOG_TIME=$(($(date +%s) - START))
echo "   Binary Log 재생 시간: ${BINLOG_TIME}초" | tee -a $LOG_FILE

# 4. 데이터 검증
echo "4. 데이터 검증" | tee -a $LOG_FILE
EXPECTED_COUNT=1000000  # 예상 orders 건수
ACTUAL_COUNT=$(ssh $RESTORE_HOST "mysql -u root -p$PASS -e 'SELECT COUNT(*) FROM mydb.orders'" 2>/dev/null | tail -1)

if [ "$ACTUAL_COUNT" -ge "$EXPECTED_COUNT" ]; then
    echo "   검증 성공: $ACTUAL_COUNT 건" | tee -a $LOG_FILE
else
    echo "   검증 실패: 예상 $EXPECTED_COUNT, 실제 $ACTUAL_COUNT" | tee -a $LOG_FILE
fi

TOTAL_RTO=$((RESTORE_TIME + BINLOG_TIME))
echo "=== 총 복구 시간(MTTR): ${TOTAL_RTO}초 ===" | tee -a $LOG_FILE

# 5. Slack 알림
curl -X POST $SLACK_WEBHOOK \
  -H 'Content-type: application/json' \
  --data "{\"text\": \"복구 리허설 완료. MTTR: ${TOTAL_RTO}초. 로그: $LOG_FILE\"}"
```

---

## 📊 성능/비용 비교

```
서비스 규모별 백업 전략 비용 비교:

소규모 (10GB DB):
  전략: mysqldump 매일 + S3 업로드
  백업 시간: 10분
  스토리지: 30GB/월 → $0.7/월
  복원 시간: 30분 (RTO ~1시간)
  RPO: 24시간 (Binary Log 없음)

중규모 (100GB DB):
  전략: XtraBackup 매일 + Binary Log 14일
  백업 시간: 1시간
  스토리지: ~600GB/월 → $14/월
  복원 시간: 1~2시간 (RTO ~2시간)
  RPO: 5분 (Binary Log 실시간)

대규모 (1TB DB):
  전략: XtraBackup + 증분백업 + Binary Log Streaming + Warm Standby
  백업 시간: 풀백업 3~4시간, 증분 30분
  스토리지: ~5TB/월 → $115/월
  복원 시간: 30분 (Failover) 또는 2~3시간 (파일 복원)
  RPO: 0 (Semi-Sync Replication) ~ 수 초

금융/결제:
  추가 비용: 멀티 AZ, Binary Log 다중 복사, 감사 로그
  스토리지: 수십 TB/월
  RPO: 0 (그룹 복제 또는 동기 복제)
```

---

## ⚖️ 트레이드오프

```
RTO/RPO 목표와 비용의 트레이드오프:

RTO 단축 방법 vs 비용:
  Warm Standby Replica:
    → Failover 시 RTO: 수 분
    → 비용: Replica 서버 유지 비용 (프로덕션 사양)

  물리 백업(XtraBackup):
    → 복원 RTO: 1~2시간 (100GB)
    → 비용: XtraBackup 도입, 운영 복잡도

  DB as a Service (RDS 등):
    → Multi-AZ: RTO 수 분
    → 비용: 클라우드 서비스 요금 (EC2 자체 구축 대비 2~3배)

RPO 단축 방법 vs 비용:
  Binary Log 실시간 아카이브:
    → RPO: 수 초~수 분
    → 비용: 아카이브 서버, 스토리지

  Semi-Sync Replication:
    → RPO: ~0 (Relay Log 수신 보장)
    → 비용: 추가 지연 (~1ms), 복잡한 운영

  Group Replication:
    → RPO: 0
    → 비용: 서버 3대 이상, 높은 운영 복잡도

현실적 조언:
  RTO/RPO 목표는 비즈니스 팀과 합의 필요
  100만원짜리 DB에 1억짜리 백업 인프라는 과도함
  서비스 다운타임 비용 계산 후 투자 결정
  작게 시작 → 측정 → 개선 반복
```

---

## 📌 핵심 정리

```
RTO/RPO 기반 백업 전략 핵심:

정의:
  RTO: 장애 후 허용 최대 복구 시간 (기술적 SLA)
  RPO: 허용 최대 데이터 손실 범위 (데이터적 SLA)

선택 기준:
  RTO < 2시간 → XtraBackup 필수
  RPO < 1시간 → Binary Log 필수 + 빈번한 Full Backup
  RPO = 0 → Semi-Sync/Group Replication 필요

Binary Log 보관:
  최소: Full Backup 주기 × 2
  권장: Full Backup 주기 × 3 + 여유

3-2-1 규칙:
  3 복사본, 2 미디어, 1 오프사이트
  클라우드: 같은 AZ + 다른 AZ + 다른 리전

복구 리허설:
  분기 1회 이상 실제 복구 수행
  MTTR 측정 후 RTO와 비교
  "백업 존재" ≠ "복구 가능" 검증 필수

비용 균형:
  서비스 다운타임 시간당 손실 × RTO 초과 시간
  > 백업 인프라 비용이면 투자 정당화
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 서비스에 적합한 RTO/RPO와 백업 전략을 설계하라.

```
서비스:
  - 소셜 미디어 게시물 플랫폼 (일 활성 사용자 100만)
  - DB 크기: 500GB (게시물, 댓글, 좋아요)
  - 수익: 광고 기반 (DB 다운 시 시간당 광고 손실 약 500만원)
  - 현재 인프라: MySQL 8.0, 단일 서버, 백업 없음
```

<details>
<summary>해설 보기</summary>

**RTO/RPO 설정**:
- 시간당 손실 500만원 → 4시간 다운 = 2,000만원 손실
- RTO = 2시간 (손실을 1,000만원 이내로)
- RPO = 30분 (최대 30분치 게시물 손실 허용, 이상 시 사용자 이탈 위험)

**백업 전략**:
```
1단계 (즉시 도입):
  XtraBackup Full Backup (매일 새벽 2시)
  Binary Log 활성화 + 14일 보관
  백업 파일 S3 자동 업로드
  RTO: 2시간, RPO: 30분 (Binary Log)

2단계 (1개월 내):
  Source-Replica 복제 (Semi-Sync)
  XtraBackup 증분 백업 (6시간마다)
  RTO: Failover 시 5분, 전체 복원 시 2시간
  RPO: ~0 (Semi-Sync) 또는 5분 (Replica Lag 허용)

3단계 (3개월 내):
  복구 리허설 월 1회 (MTTR 측정)
  Binary Log 실시간 아카이브 (별도 서버)
  멀티 AZ 배포 고려

비용 추정:
  Replica 서버: 월 50~100만원
  스토리지 (500GB × 3개 + Binary Log): 월 3~5만원
  총 월 비용: 55~110만원
  → 시간당 500만원 손실 vs 월 100만원 투자: 명백한 ROI
```

</details>

---

**Q2.** 현재 팀에서 "복구 리허설을 해야 한다"고 주장할 때 보안/운영 팀을 설득하는 논리를 만들어라.

<details>
<summary>해설 보기</summary>

**설득 포인트 3가지**:

**1. 백업 = 복구 가능 보장이 아님**
"백업 파일이 있다는 것은 파일이 있다는 것이지, 복구가 된다는 것이 아닙니다. 버전 차이, 파일 손상, 절차 오류로 실제 장애 시 복구 실패 사례가 빈번합니다. 리허설을 통해서만 '백업이 동작한다'를 증명할 수 있습니다."

**2. 장애 시 공황 상태에서 처음 해보면 시간이 3~5배 걸림**
"장애 상황은 스트레스가 극도로 높습니다. 처음 해보는 복구는 실수가 잦고 시간이 예측보다 3~5배 소요됩니다. 리허설을 통해 MTTR을 측정하고 RTO와 비교하면 실제 SLA 달성 여부를 사전에 알 수 있습니다."

**3. 복구 절차 문서의 유효성 검증**
"6개월 전 작성된 복구 문서는 현재 환경(DB 버전 업그레이드, 스키마 변경, 서버 이전)과 다를 수 있습니다. 리허설만이 문서의 실제 유효성을 보장합니다."

**비용 정당화**:
```
리허설 비용: 엔지니어 4시간 × 2인 × 시간당 5만원 = 40만원/회
연 4회 리허설: 160만원

장애 발생 시 복구 실패로 추가 소요 시간: 4~8시간 추가
다운타임 시간당 손실 100만원 × 6시간 = 600만원

리허설 ROI: 160만원 투자 → 잠재 손실 600만원 방지
```

</details>

---

<div align="center">

**[⬅️ PITR](./03-pitr-point-in-time-recovery.md)** | **[홈으로 🏠](../README.md)** | **[다음: 장애 복구 시나리오 ➡️](./05-disaster-recovery-scenario.md)**

</div>
