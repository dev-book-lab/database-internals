# WAL — Write-Ahead Logging과 장애 복구

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- WAL이 없다면 COMMIT 후 크래시 시 데이터가 사라질 수 있는 구체적인 이유는 무엇인가?
- Redo Log는 어떻게 "Buffer Pool의 Dirty Page가 디스크에 없어도 내구성을 보장"하는가?
- Crash Recovery 시 Redo Log를 어디서부터 어디까지 재생하는가?
- `innodb_flush_log_at_trx_commit` 0 / 1 / 2의 차이와 트레이드오프는?
- Group Commit이란 무엇이고, 왜 IOPS를 줄이는가?
- Redo Log와 Undo Log는 각각 어떤 역할을 하고 서로 어떻게 연관되는가?

---

## 🔍 왜 이 개념이 중요한가

### DB의 "D(Durability)"는 어떻게 보장되는가

```
ACID의 D = Durability (내구성):
  "COMMIT된 트랜잭션은 시스템 장애가 발생해도 손실되지 않는다"

순진한 구현:
  COMMIT 시 수정된 모든 데이터 Page를 디스크에 즉시 쓰기
  → 한 트랜잭션이 수십 개의 Page를 수정했다면:
     수십 번의 Random Disk I/O (각 Page가 서로 다른 위치)
     SSD에서도 수십 ms → HDD에서는 수백 ms
     초당 수십 개의 COMMIT도 버티기 어려움

WAL의 해결책:
  "모든 Page를 디스크에 쓰는 대신,
   무엇을 변경했는지 로그만 순서대로 디스크에 쓰자"

  변경 로그 = Redo Log
  Redo Log의 특성:
    항상 순서대로(Sequential) 쓰기 → 디스크 탐색 없음
    크기가 작음 (Page 전체가 아닌 변경 내용만)
    
  결과:
    COMMIT = Redo Log만 flush (Sequential 쓰기 1~2번)
    Buffer Pool의 Dirty Page는 나중에 비동기로 flush
    → COMMIT 응답 시간: 수십 배 단축

왜 Spring 개발자에게도 중요한가:
  @Transactional이 COMMIT을 호출할 때
  실제로 어느 수준의 I/O가 발생하는지 → 응답 시간 예측
  innodb_flush_log_at_trx_commit 설정이 성능과 내구성에 미치는 영향
  배치 처리에서 중간 COMMIT이 중요한 이유
```

---

## 😱 잘못된 이해

### Before: WAL 없이 COMMIT을 구현한다면

```
WAL 없는 순진한 COMMIT 구현:

  BEGIN
  UPDATE orders SET status = 'PAID' WHERE id = 1;
    → Buffer Pool의 orders Page를 수정 (Dirty Page)
    → 디스크는 아직 이전 상태
  UPDATE accounts SET balance = balance - 10000 WHERE id = 42;
    → Buffer Pool의 accounts Page를 수정 (Dirty Page)
  COMMIT
    → 수정된 두 Page를 디스크에 즉시 쓰기

문제 시나리오 — 부분 쓰기 후 크래시:
  orders Page 쓰기 성공 (디스크에 반영됨)
  ─── 여기서 서버 크래시! ───
  accounts Page 쓰기 미완료 (디스크에 이전 값)

  재시작 후:
    orders.status = 'PAID' (변경됨)
    accounts.balance = 이전 값 (변경 안 됨)
    → 돈은 받았는데 계좌 잔액은 안 줄어든 상태
    → 원자성(Atomicity) 위반!

Double Write Buffer (부분 쓰기 문제의 별도 해결책):
  16KB Page를 디스크에 쓸 때 "Torn Write" 가능
  → 4KB 섹터 HDD에서 16KB 쓰기 중 크래시
  → 12KB만 쓰인 손상된 Page 발생
  
  InnoDB는 이를 Double Write Buffer로 해결:
    Page를 실제 위치에 쓰기 전에 공유 공간에 먼저 쓰기
    크래시 시 Double Write Buffer에서 복구
  
  이것은 Page 물리적 손상 방지이고
  WAL은 트랜잭션 원자성/내구성 보장 — 다른 문제!
```

---

## ✨ 올바른 이해

### After: WAL로 내구성과 성능을 동시에 달성

```
WAL(Write-Ahead Logging) 원칙:
  "데이터 Page를 디스크에 쓰기 전에
   반드시 해당 변경에 대한 로그(Redo Log)를 디스크에 먼저 써야 한다"

  Write-Ahead = 데이터 쓰기 "앞서" 로그 쓰기

InnoDB에서의 WAL 흐름:

  ① 트랜잭션 시작
  
  ② 데이터 변경:
     Buffer Pool에서 Page 수정 (Dirty Page 생성)
     동시에 Redo Log Buffer에 변경 내용 기록
     
     Redo Log Buffer:
       메모리 내 임시 버퍼
       innodb_log_buffer_size (기본 16MB)
       여러 트랜잭션의 로그가 순서대로 쌓임
  
  ③ COMMIT:
     Redo Log Buffer → Redo Log 파일(ib_logfile0, ib_logfile1)에 fsync
     ← 이 순간이 "커밋된 것"으로 확정
     
     Buffer Pool의 Dirty Page는 아직 디스크에 없어도 됨!
  
  ④ 이후 (비동기):
     Page Cleaner Thread가 Dirty Page를 Checkpoint 기준으로 flush

  크래시 후 복구:
     Redo Log를 재생하면 커밋된 모든 트랜잭션이 복원됨
     → 내구성 보장
```

---

## 🔬 내부 동작 원리

### 1. Redo Log 구조

```
InnoDB Redo Log 파일:
  기본: ib_logfile0, ib_logfile1 (각 48MB, 총 96MB)
  설정: innodb_log_file_size, innodb_log_files_in_group

Redo Log의 물리적 구조:
  순환 로그 파일 (Circular Log):
    ib_logfile0: [Block1][Block2]...[BlockN]
    ib_logfile1: [Block1][Block2]...[BlockN]
    → ib_logfile0 가득 차면 ib_logfile1에 계속 쓰기
    → ib_logfile1 가득 차면 ib_logfile0을 재사용
  
  재사용 조건:
    Checkpoint가 해당 로그 위치를 지나야 재사용 가능
    (Checkpoint = "이 위치까지는 Buffer Pool이 디스크에 반영됨" 표시)

Redo Log Record 구조:
  각 로그 레코드:
    Log Record Type (변경 타입 — Page 헤더 수정, Row 삽입 등)
    Space ID + Page No (어느 Page의 변경인지)
    변경 내용 (Before/After 값 또는 델타)
    LSN (Log Sequence Number) — 단조 증가하는 로그 일련번호

LSN (Log Sequence Number):
  Redo Log에서 절대적 위치를 나타내는 번호 (단조 증가)
  
  각 Page의 File Header에 저장된 LSN:
    Page가 마지막으로 수정됐을 때의 LSN
    Checkpoint LSN과 비교하여 flush 여부 결정
    
  Page LSN < Checkpoint LSN:
    → 이 Page의 변경은 이미 디스크에 반영됨 → Redo Log 재생 불필요
  
  Page LSN > Checkpoint LSN:
    → 이 Page가 크래시 전 디스크에 기록됐는지 불확실
    → Crash Recovery 시 Redo Log에서 재생 필요
```

### 2. Crash Recovery 과정

```
MySQL 재시작 후 InnoDB 초기화:

① Redo Log에서 마지막 Checkpoint LSN 찾기
   → ibdata1의 System Page에 Checkpoint LSN 저장됨

② Checkpoint LSN 이후의 모든 Redo Log 레코드 수집

③ Redo Phase (Re-do):
   수집된 로그 레코드를 순서대로 재생
   → 각 Page를 크래시 직전 상태로 복원
   (COMMIT된 트랜잭션 + 미완료 트랜잭션 모두 포함)

④ Undo Phase (Un-do):
   미완료(COMMIT 없이 크래시) 트랜잭션을 Undo Log로 롤백
   → 원자성 보장: 미완료 트랜잭션의 변경 제거

결과:
  COMMIT된 트랜잭션: 모두 복원 ✅ (내구성)
  미완료 트랜잭션: 롤백 ✅ (원자성)

Recovery 시간 결정 요소:
  Checkpoint 이후 Redo Log 양
  → innodb_log_file_size가 크면:
     Checkpoint 간격이 길어짐 → Redo Log 재생량 증가 → 복구 느림
     But 평상시 쓰기 성능 향상 (Checkpoint 빈도 감소)
  → innodb_log_file_size가 작으면:
     Checkpoint 자주 → 복구 빠름
     But 평상시 빈번한 Dirty Page flush (쓰기 I/O 증가)
  
  권장: 실제 쓰기 워크로드에서 약 1시간치 로그가 쌓이는 크기
        보통 1~4GB 사이 (MySQL 8.0에서 기본값 변경됨)
```

### 3. innodb_flush_log_at_trx_commit 설정

```
설정값별 동작 비교:

─────────────────────────────────────────────────────────
값 = 1 (기본값, 완전한 ACID):
  COMMIT 시:
    Redo Log Buffer → Redo Log 파일 쓰기 + fsync()
    fsync = OS 버퍼까지 플러시 → 물리적 디스크 반영 확인
  
  크래시 시나리오:
    COMMIT 직후 크래시 → Redo Log에 기록됨 → 복구 가능 ✅
  
  비용:
    COMMIT마다 fsync() 1회
    SSD: ~100 μs/fsync → 초당 최대 ~10,000 TPS (fsync 병목)
    HDD: ~8ms/fsync → 초당 최대 ~125 TPS (매우 낮음)

─────────────────────────────────────────────────────────
값 = 2 (OS 크래시에는 안전, 서버 크래시에는 위험):
  COMMIT 시:
    Redo Log Buffer → Redo Log 파일 쓰기 (fsync 없음)
    OS 파일 버퍼에만 기록 → 물리적 디스크 반영 미확인
    OS는 주기적으로(~1초마다) 실제 디스크에 기록
  
  크래시 시나리오:
    MySQL 프로세스만 크래시 → OS 버퍼 유지 → 안전 ✅
    서버 전원 꺼짐 → OS 버퍼 소실 → 최대 1초치 트랜잭션 손실 ❌
  
  성능: 값=1 대비 2~5배 TPS 향상 (fsync 제거 효과)
  
  사용 케이스: 읽기 복제본, 스테이징 환경

─────────────────────────────────────────────────────────
값 = 0 (가장 위험, 가장 빠름):
  COMMIT 시:
    Redo Log Buffer에만 기록 → 파일 쓰기도 없음
    백그라운드 스레드가 초당 1회 파일에 쓰고 fsync
  
  크래시 시나리오:
    MySQL 프로세스 크래시 → 최대 1초치 손실 ❌
    서버 전원 꺼짐 → 최대 1초치 손실 ❌
  
  성능: 값=1 대비 5~10배 TPS 향상
  사용 케이스: 개발/테스트 환경, 재현 불가능한 성능 측정

─────────────────────────────────────────────────────────
요약:
  프로덕션 OLTP:     값=1 (내구성 최우선)
  성능 중요 (복제본): 값=2 (MySQL 크래시 안전)
  개발/테스트:        값=0 (빠르게)
```

### 4. Group Commit — IOPS를 줄이는 최적화

```
문제:
  높은 동시성에서 초당 수천 개의 COMMIT
  값=1 설정: 각 COMMIT마다 fsync() 1회
  → 초당 수천 번의 fsync → I/O 병목

Group Commit:
  여러 COMMIT을 그룹으로 묶어 한 번의 fsync로 처리

  동작 방식:
    ① 트랜잭션들이 Redo Log Buffer에 로그 기록
    ② fsync를 기다리는 트랜잭션들이 Queue에서 대기
    ③ Leader 트랜잭션이 대기 중인 모든 로그를 파일에 쓰고 fsync
    ④ Leader가 대기 중인 Follower 트랜잭션들에게 완료 통보
    ⑤ 모든 대기 트랜잭션이 한 번의 fsync로 COMMIT 완료

  효과:
    fsync 횟수 = (동시 COMMIT 수) / (그룹당 평균 크기)
    동시 COMMIT 100개 → 그룹 크기 100 → 1회 fsync로 처리
    → 처리량 100배 향상 가능

  MySQL Binary Log와의 연계:
    복제 환경에서 Binary Log도 함께 flush해야 함
    Two-Phase Commit (2PC):
      Phase 1: Redo Log에 준비(Prepare) 기록
      Phase 2: Binary Log flush → Redo Log에 COMMIT 기록
    
    Group Commit이 Binary Log에도 적용됨
    → sync_binlog = 1 (Binary Log도 COMMIT마다 flush)에서도 그룹화

  -- Group Commit 효과 확인
  SHOW STATUS LIKE 'Innodb_os_log_fsyncs';     -- 총 fsync 횟수
  SHOW STATUS LIKE 'Com_commit';               -- 총 COMMIT 횟수
  -- fsyncs / commits < 1이면 Group Commit 효과 발생 중
```

### 5. Redo Log와 Undo Log — 역할 비교

```
두 로그의 역할 비교:

Redo Log (Forward Recovery):
  목적: Durability (내구성) 보장
  내용: 변경된 데이터의 새 값 (After Image)
  사용: Crash Recovery 시 커밋된 트랜잭션 재현
  저장: ib_logfile0, ib_logfile1 (순환 파일)
  크기: 상대적으로 작음 (변경 델타만 저장)
  언제 삭제: Checkpoint 이후 공간 재사용

Undo Log (Backward Recovery + MVCC):
  목적: Atomicity (원자성) + MVCC (일관된 읽기)
  내용: 변경 이전 값 (Before Image)
  사용:
    ① Crash Recovery에서 미완료 트랜잭션 롤백 (Undo Phase)
    ② 실행 중 ROLLBACK
    ③ MVCC: 다른 트랜잭션이 이전 버전을 읽어야 할 때
  저장: Undo Tablespace (undo_001, undo_002)
  크기: 활성 트랜잭션 기간에 비례 (오래된 트랜잭션 → 무한 증가 가능)
  언제 삭제: 더 이상 이전 버전을 필요로 하는 트랜잭션이 없을 때 (Purge)

함께 동작하는 방식:
  
  UPDATE orders SET status='PAID' WHERE id=1;
  
  Redo Log에 기록:
    "orders 테이블, Page #42, Row id=1의 status를 'PENDING'에서 'PAID'로 변경"
  
  Undo Log에 기록:
    "orders 테이블, Page #42, Row id=1의 status 이전 값: 'PENDING'"
    "ROLLBACK 시: status를 'PENDING'으로 되돌릴 것"
  
  Crash Recovery 시:
    Redo Phase: status='PAID' 재현 (커밋됐다면)
    Undo Phase: 만약 커밋 전 크래시라면 status='PENDING'으로 복원
```

---

## 💻 실전 실험

### 실험 1: innodb_flush_log_at_trx_commit별 성능 비교

```sql
-- 성능 비교를 위한 프로시저
DELIMITER //
CREATE PROCEDURE benchmark_commits(IN n INT)
BEGIN
    DECLARE i INT DEFAULT 0;
    DECLARE start_time DATETIME(6) DEFAULT NOW(6);
    
    WHILE i < n DO
        INSERT INTO orders (user_id, amount, status, created_at)
        VALUES (1, 100.00, 'PENDING', NOW());
        COMMIT;
        SET i = i + 1;
    END WHILE;
    
    SELECT
        n AS total_commits,
        TIMESTAMPDIFF(MICROSECOND, start_time, NOW(6)) / 1000 AS ms_elapsed,
        ROUND(n / (TIMESTAMPDIFF(MICROSECOND, start_time, NOW(6)) / 1000000)) AS commits_per_sec;
END//
DELIMITER ;

-- 설정 1: 완전한 내구성
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
CALL benchmark_commits(1000);
-- 예상: SSD 기준 1000~5000 commits/sec

-- 설정 2: OS 크래시 안전
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
CALL benchmark_commits(1000);
-- 예상: 설정 1 대비 2~5배 빠름

-- 설정 0: 최대 성능
SET GLOBAL innodb_flush_log_at_trx_commit = 0;
CALL benchmark_commits(1000);
-- 예상: 설정 1 대비 5~10배 빠름

-- 원복
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
```

### 실험 2: Checkpoint와 Redo Log 모니터링

```sql
-- Redo Log 상태 확인
SHOW ENGINE INNODB STATUS\G

-- 출력 중 LOG 섹션:
-- ---
-- LOG
-- ---
-- Log sequence number          2345678901   ← 현재 LSN (최신 Redo Log 위치)
-- Log buffer assigned up to    2345678901
-- Log buffer completed up to   2345678901
-- Log written up to            2345678901
-- Log flushed up to            2345678901   ← 디스크에 쓰인 LSN
-- Added dirty pages up to      2345678901
-- Pages flushed up to          2345678890   ← Checkpoint LSN (여기까지 Page flush됨)
-- Last checkpoint at           2345678890
-- Log gap = LSN - Checkpoint = 11           ← 이 만큼의 Redo Log가 재생 필요할 수 있음

-- Redo Log 크기와 사용률
SELECT
    ROUND(@@innodb_log_file_size / 1024 / 1024, 0) AS log_file_size_mb,
    @@innodb_log_files_in_group AS log_files,
    ROUND(@@innodb_log_file_size * @@innodb_log_files_in_group / 1024 / 1024, 0) AS total_log_mb;

-- Redo Log 사용률 (높으면 Checkpoint가 자주 발생)
SHOW STATUS LIKE 'Innodb_os_log%';
-- Innodb_os_log_fsyncs: 총 fsync 횟수 → Group Commit 효과 측정에 활용
```

### 실험 3: Crash Recovery 시뮬레이션

```bash
# 주의: 실험 환경에서만 수행

# 1. 트랜잭션 실행 중 강제 종료 (충돌 시뮬레이션)
# MySQL 클라이언트 1에서:
mysql> BEGIN;
mysql> INSERT INTO orders (user_id, amount, status, created_at) VALUES (999, 100, 'TEST', NOW());
# 여기서 아직 COMMIT하지 않음

# 터미널에서 MySQL 프로세스 강제 종료:
docker exec -it mysql bash -c "kill -9 \$(pidof mysqld)"

# 2. 재시작 후 로그 확인
docker-compose restart mysql

# 3. 에러 로그에서 Recovery 과정 확인
docker exec mysql tail -n 50 /var/log/mysql/error.log
# 출력 예시:
# [Note] InnoDB: Starting crash recovery.
# [Note] InnoDB: Reading pages with largest LSN in each zone...
# [Note] InnoDB: Doing recovery: scanned up to log sequence number ...
# [Note] InnoDB: Database was not shut down normally!
# [Note] InnoDB: Starting crash recovery from checkpoint LSN=...
# [Note] InnoDB: Recovered ...
# [Note] InnoDB: 1 transaction(s) which must be rolled back ...
# [Note] InnoDB: Trx id counter is ...
# [Note] InnoDB: Starting an apply batch of log records to the database...
# [Note] InnoDB: Apply batch completed!
# [Note] InnoDB: Rolling back trx with id ..., 1 rows to undo

# 4. 강제 종료 전 INSERT가 롤백됐는지 확인
mysql> SELECT COUNT(*) FROM orders WHERE user_id = 999;
-- 결과: 0 (COMMIT 전이었으므로 롤백됨)
```

### 실험 4: Group Commit 효과 측정

```sql
-- Group Commit 효과를 측정하기 위한 동시 COMMIT 실험
-- (여러 연결에서 동시에 INSERT + COMMIT)

-- 연결 1~5에서 동시 실행 (별도 세션):
-- for i in 1 2 3 4 5; do
--   mysql -u root -proot deep_dive -e "
--     CALL benchmark_commits(200);" &
-- done

-- 직렬 실행 총 1000 COMMIT vs 병렬 5 × 200 COMMIT 시간 비교
-- Group Commit으로 병렬 실행이 더 효율적 (fsync 횟수 감소)

-- Group Commit 효과 확인
SELECT
    variable_name,
    variable_value
FROM performance_schema.global_status
WHERE variable_name IN ('Innodb_os_log_fsyncs', 'Com_commit');
-- Com_commit / Innodb_os_log_fsyncs 비율이 클수록 Group Commit 효과 ↑
```

---

## 📊 성능 비교

```
innodb_log_file_size 크기 선택:

너무 작을 때 (48MB, 기본):
  Redo Log가 자주 가득 참
  → Checkpoint 자주 발생 → Dirty Page flush 빈번
  → 쓰기 집약적 워크로드에서 I/O 병목
  
  측정: SHOW STATUS LIKE 'Innodb_log_waits';
  → 0이 아니면 Redo Log가 병목

너무 클 때 (16GB):
  Checkpoint가 드물게 발생
  → 크래시 시 Redo Log 재생량 증가 → Recovery 시간 증가
  
  계산: 1시간 동안 쌓이는 Redo Log 양 측정
  → Log sequence number 변화량 = 1시간 Redo Log 생성량
  → 이 값의 1~2배로 설정 (Checkpoint 간격 = 30분~1시간)

MySQL 8.0.30+ 변경:
  동적으로 Redo Log 크기 조정 가능
  innodb_redo_log_capacity (전체 용량, 기본 100MB)
  → 더 유연한 설정 가능

Group Commit 최대화:
  binlog_group_commit_sync_delay: fsync 전 추가 대기 (μs)
    → 더 많은 트랜잭션을 그룹에 포함
    → 지연 증가 vs 처리량 향상 트레이드오프
  
  binlog_group_commit_sync_no_delay_count: 대기 없이 즉시 fsync할 최소 트랜잭션 수
```

---

## ⚖️ 트레이드오프

```
내구성 vs 성능:

innodb_flush_log_at_trx_commit:
  1: 완전한 내구성, 낮은 TPS
  2: MySQL 크래시 안전, 높은 TPS (HW 장애 시 최대 1초 손실)
  0: 가장 빠름, 최대 1초 손실 가능 (개발 환경만)

innodb_log_file_size:
  작은 크기:  복구 빠름, 쓰기 성능 낮음 (잦은 Checkpoint)
  큰 크기:    쓰기 성능 높음, 복구 느림

sync_binlog (복제 환경):
  0: Binary Log를 OS에 맡김 (빠름, 손실 위험)
  1: 매 COMMIT마다 fsync (느림, 완전한 안전)
  N: N개 COMMIT마다 fsync (절충)
  
  innodb_flush_log_at_trx_commit=1 + sync_binlog=1:
    "Fully Durable" 설정 — 프로덕션 권장
    Group Commit으로 성능 손실 최소화

배치 처리에서의 COMMIT 전략:
  1건당 COMMIT:
    Redo Log fsync 매번 → 느림
    하지만 중간 실패 시 이미 COMMIT된 건은 안전
  
  1000건당 COMMIT:
    fsync 횟수 1/1000 → 빠름
    실패 시 최대 999건 재처리 필요
    Group COMMIT + 적절한 배치 크기 선택이 핵심
```

---

## 📌 핵심 정리

```
WAL 핵심:

원칙:
  데이터 Page 수정 전에 Redo Log를 반드시 먼저 디스크에 기록
  → "Write-Ahead" = 데이터보다 로그를 앞서 쓰기

COMMIT = Redo Log flush (데이터 Page는 나중에):
  COMMIT 속도 = Redo Log fsync 속도 (Sequential I/O)
  Data Page는 Page Cleaner가 비동기로 flush

Crash Recovery 흐름:
  Redo Phase: Checkpoint 이후 Redo Log 재생
              → 커밋된 트랜잭션 + 미완료 트랜잭션 모두 재현
  Undo Phase: 미완료 트랜잭션을 Undo Log로 롤백
              → 원자성 보장

innodb_flush_log_at_trx_commit:
  1: COMMIT 시 fsync (완전한 내구성, 프로덕션)
  2: COMMIT 시 write만 (MySQL 크래시 안전, 빠름)
  0: 백그라운드 1초마다 flush (최대 1초 손실, 개발용)

Group Commit:
  동시 COMMIT들을 그룹화 → 1회 fsync로 처리
  높은 동시성 환경에서 fsync 병목 자동 해소

Redo vs Undo:
  Redo: After Image → 크래시 시 커밋 재현 (내구성)
  Undo: Before Image → ROLLBACK + MVCC 이전 버전 제공 (원자성 + 일관성)

개발자가 챙겨야 할 것:
  @Transactional 범위 최소화 → Undo Log 축적 방지
  배치 처리 시 주기적 COMMIT → Redo Log 공간 재사용, Undo Log 정리
  innodb_flush_log_at_trx_commit 설정이 성능에 미치는 영향 인지
```

---

## 🤔 생각해볼 문제

**Q1.** `innodb_flush_log_at_trx_commit = 1` 환경에서 1건씩 COMMIT하는 JPA 배치 코드가 있다. 1000건을 처리할 때 왜 느리고, 어떻게 개선할 수 있는가?

<details>
<summary>해설 보기</summary>

**이유**: `innodb_flush_log_at_trx_commit = 1`에서 각 COMMIT은 Redo Log의 `fsync()`를 유발합니다. SSD 기준으로 `fsync()` 1회에 100~500μs가 소요됩니다. 1000건 × 500μs = 500ms 이상이 순수 fsync 대기 시간입니다.

**개선 방법**:

1. **배치 크기 조정**: 100~500건마다 COMMIT → fsync 횟수 1/100~1/500
   ```java
   @Transactional
   public void batchInsert(List<Order> orders) {
       for (int i = 0; i < orders.size(); i++) {
           em.persist(orders.get(i));
           if (i % 500 == 0) {
               em.flush();
               em.clear();
           }
       }
   } // 메서드 끝에서 한 번 COMMIT → 1회 fsync
   ```

2. **JDBC Batch Insert**: `spring.jpa.properties.hibernate.jdbc.batch_size = 500` 설정 → 여러 INSERT를 하나의 SQL로 묶어 전송

3. **Group Commit 활용**: 여러 스레드가 동시에 COMMIT하면 자동으로 그룹화됩니다.

4. **비동기 처리**: 중간 실패 허용 가능하다면 `innodb_flush_log_at_trx_commit = 2`로 변경 (복제본 서버에서 배치 처리 시)

</details>

---

**Q2.** `innodb_log_file_size`를 100MB에서 4GB로 늘렸더니 일반 쓰기 성능이 향상됐다. 그런데 장애 드릴(Disaster Recovery Drill) 팀에서 복구 시간이 30분 이상 걸린다고 문제를 제기했다. 어떻게 두 요구를 절충할 수 있는가?

<details>
<summary>해설 보기</summary>

**원인**: `innodb_log_file_size = 4GB`이면 Checkpoint가 드물게 발생합니다. 크래시 시 재생해야 할 Redo Log 양이 최대 4GB × 2파일 = 8GB에 달할 수 있습니다. Redo Log 재생 속도는 초당 수백 MB 수준이므로 수십 분이 걸릴 수 있습니다.

**절충 방법**:

1. **innodb_adaptive_flushing 최적화**: `innodb_adaptive_flushing = ON`(기본)이 이미 켜져 있지만, `innodb_adaptive_flushing_lwm`(Low Watermark) 조정으로 Checkpoint 빈도 증가

2. **Recovery 시간 목표(RTO) 기반 설정**:
   ```
   목표 복구 시간 = 5분
   Redo Log 재생 속도 = ~500MB/min
   최대 허용 Redo Log 양 = 5 × 500 = 2.5GB
   innodb_log_file_size = 1.25GB (× 2파일 = 2.5GB)
   ```

3. **MySQL 8.0.30+**: `innodb_redo_log_capacity`로 동적 조정 + InnoDB가 내부적으로 파일 수를 자동 관리하여 균형을 더 잘 맞출 수 있음

4. **쓰기 성능 유지 대안**: Redo Log 크기를 줄이는 대신 SSD RAID + 더 많은 Page Cleaner Thread로 Dirty Page flush 속도를 높여 Checkpoint 부담 감소

</details>

---

**Q3.** 다음 상황에서 서버가 재시작된 후 각 트랜잭션의 최종 상태는 어떻게 되는가?

```
LSN 1000: BEGIN (트랜잭션 A)
LSN 1001: INSERT INTO orders VALUES (1, ...) [트랜잭션 A]
LSN 1002: BEGIN (트랜잭션 B)
LSN 1003: INSERT INTO orders VALUES (2, ...) [트랜잭션 B]
LSN 1004: COMMIT [트랜잭션 B] ← Redo Log에 기록됨
LSN 1005: INSERT INTO orders VALUES (3, ...) [트랜잭션 A]
─── 크래시! (Checkpoint는 LSN 900에서 마지막으로 수행됨) ───
```

<details>
<summary>해설 보기</summary>

**복구 과정**:
1. Checkpoint LSN = 900이므로 LSN 901~1005까지 Redo Log 재생
2. Redo Phase 완료 후 DB 상태:
   - `id=1`: 삽입됨 (트랜잭션 A)
   - `id=2`: 삽입됨 (트랜잭션 B)
   - `id=3`: 삽입됨 (트랜잭션 A)
3. Undo Phase: 트랜잭션 A는 COMMIT 마커 없음 → Undo Log로 롤백
   - `id=1` 삭제 (롤백)
   - `id=3` 삭제 (롤백)

**최종 상태**:
- 트랜잭션 A: **롤백** — `id=1`, `id=3` 없음 (원자성 보장)
- 트랜잭션 B: **커밋** — `id=2` 존재 (내구성 보장)

이것이 WAL이 Atomicity + Durability를 동시에 보장하는 방식입니다. Redo로 커밋된 것을 재현하고, Undo로 미완료된 것을 취소합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Tablespace와 파일 구성](./04-tablespace.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Index 완전 분해 ➡️](../index-internals/01-btree-why-not-binary-tree.md)**

</div>
