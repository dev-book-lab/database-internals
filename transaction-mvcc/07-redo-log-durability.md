# Redo Log와 트랜잭션 내구성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- COMMIT 시 Buffer Pool의 Dirty Page를 직접 쓰지 않고 Redo Log를 먼저 쓰는 이유는?
- WAL(Write-Ahead Logging) 원칙이 "Log를 Data보다 먼저 써야 한다"는 이유는?
- Redo Log Buffer가 Redo Log File로 플러시되는 세 가지 시점은?
- `innodb_flush_log_at_trx_commit` 0/1/2 각각에서 어떤 장애에 데이터를 잃는가?
- Group Commit이 "N번의 fsync를 1번으로 줄인다"는 구체적 메커니즘은?
- Redo Log 크기가 너무 작으면 왜 쓰기 성능이 불안정해지는가?

---

## 🔍 왜 이 개념이 중요한가

### "COMMIT 완료" 응답을 받았는데 서버 재시작 후 데이터가 사라졌다

```
실제 장애 시나리오:
  innodb_flush_log_at_trx_commit = 0 설정된 DB 서버
  매초 수천 건의 주문이 들어오는 서비스
  갑작스러운 OS 크래시 (전원 장애)
  
  재시작 후:
    최근 1초치 COMMIT 완료된 주문 데이터 모두 없음
    고객에게는 "주문 완료" 응답이 갔지만 DB에는 없음
    → 주문 취소/환불 대란

이것이 가능한 이유:
  설정 0: COMMIT 시 아무것도 하지 않음
          1초마다 OS 버퍼 → 디스크 flush
  → "COMMIT 완료" 응답 시점에 데이터는 메모리에만 존재
  → OS 크래시 = 메모리 소실 = 데이터 소실

개발자가 이것을 알아야 하는 이유:
  프로덕션 DB 설정이 적절한지 판단 가능
  "완전한 내구성이 필요한가, 성능이 더 중요한가" 의식적 선택
  innodb_flush_log_at_trx_commit = 1이 기본값인 이유 이해
  Replica에서 설정 2를 사용하는 이유와 안전성 판단
```

---

## 😱 잘못된 이해

### Before: COMMIT하면 즉시 데이터 파일에 기록된다

```
잘못된 멘탈 모델:

  START TRANSACTION;
  INSERT INTO orders (...) VALUES (...);  → orders.ibd 파일에 기록
  UPDATE accounts SET balance = ...;      → accounts.ibd 파일에 기록
  COMMIT;                                 → "완료" (파일이 이미 업데이트됨)

실제 흐름:
  START TRANSACTION;
  INSERT/UPDATE ...  → Buffer Pool (메모리)에 Dirty Page 생성
                     → Redo Log Buffer (메모리)에 변경 내용 기록
  COMMIT;            → Redo Log Buffer → Redo Log File (디스크, fsync)
                        ← 이 시점에 내구성 보장
                     → Dirty Pages는 여전히 메모리에 (아직 디스크 아님)
  
  나중에 백그라운드: Dirty Pages → 데이터 파일 (Random I/O, 비동기)

잘못된 결론들:
  "COMMIT이 느린 이유는 데이터 파일을 쓰기 때문이다"
  → 실제: COMMIT 성능 = Redo Log Sequential Write 성능
  
  "데이터 파일이 최신 상태를 유지한다"
  → 실제: 데이터 파일은 항상 Redo Log보다 뒤처짐 (Checkpoint 지연)
  
  "크래시 후 데이터 파일이 손상되면 복구 불가"
  → 실제: Redo Log로 크래시 이전 COMMIT까지 완전 복구 가능
```

---

## ✨ 올바른 이해

### After: WAL — 내구성과 성능을 동시에 달성하는 설계

```
핵심 통찰: Random I/O와 Sequential I/O의 비용 차이를 활용

Random I/O (Dirty Page → 데이터 파일):
  HDD: 디스크 헤드 이동 → 수 ms/페이지
  SSD: 랜덤 쓰기 → 수십~수백 μs/페이지
  특성: I/O 위치가 불규칙 → 하드웨어 최적화 어려움

Sequential I/O (Redo Log Append):
  HDD: 헤드 이동 없음, 순차 기록 → 수십 μs/KB
  SSD: 순차 쓰기 → 수 μs/KB
  특성: I/O 위치가 예측 가능 → 하드웨어 최적화 용이

WAL 전략:
  COMMIT 시: Sequential Write만 (Redo Log fsync)
  → "내구성 = 고속 Sequential I/O"
  
  Dirty Page 기록: 백그라운드 비동기 (Random I/O, COMMIT 차단 않음)
  → "성능 저하 없이 내구성 보장"

크래시 복구 원리:
  데이터 파일 ≠ 최신 상태 (Dirty Page 아직 반영 안 됨)
  하지만 Redo Log = 최신 COMMIT까지 완전 기록
  
  재시작 시:
    ① Redo Log 재생 (Checkpoint 이후 기록된 모든 변경)
    → Buffer Pool 상태 복원
    ② 완료되지 않은 트랜잭션 → Undo Log로 롤백
    → COMMIT된 모든 데이터 복구 완료
```

---

## 🔬 내부 동작 원리

### 1. Redo Log의 물리적 구조

```
Redo Log 파일 구조 (MySQL 8.0):

MySQL 8.0.30 이전:
  datadir/ib_logfile0, ib_logfile1 (기본 2개)
  innodb_log_file_size: 각 파일 크기 (기본 48MB → 권장 256MB~수 GB)
  innodb_log_files_in_group: 파일 수 (기본 2)
  총 Redo Log 크기 = innodb_log_file_size × innodb_log_files_in_group

MySQL 8.0.30+:
  단일 파라미터: innodb_redo_log_capacity
  내부적으로 작은 파일 32개로 자동 관리
  SET GLOBAL innodb_redo_log_capacity = 8 * 1024 * 1024 * 1024;  -- 8GB

순환(Circular) 구조:
  ┌─────────────────────────────────────────────────────────┐
  │                  ib_logfile0                            │
  │  [이미 반영됨]│[Checkpoint 위치]│[쓰기 진행 중]│[빈 공간]       │
  └─────────────────────────────────────────────────────────┘
                         ↑ Checkpoint LSN        ↑ Current Write LSN

  Checkpoint 이전 = 해당 변경이 데이터 파일에 이미 반영됨
                  → 이 공간은 덮어써도 됨 (재활용)
  Checkpoint ~ Write LSN = 아직 데이터 파일에 반영 안 됨
                          → 보존 필수 (크래시 복구에 필요)

LSN (Log Sequence Number):
  Redo Log 내 위치를 나타내는 단조 증가 숫자 (64-bit)
  각 변경마다 LSN 증가
  Buffer Pool의 각 Page는 "Page LSN" = 마지막 변경 LSN 저장
  Checkpoint LSN = 이 LSN 이전의 모든 변경이 데이터 파일에 반영됨

Redo Log Record 구조:
  [Space ID: 4 bytes] [Page Number: 4 bytes]
  [변경 타입: 1 byte] [변경 내용: 가변]
  → "어느 파일, 어느 페이지, 어느 오프셋에 무슨 값이 쓰였다"
  → 물리적(Physiological) 로그: Page 수준의 정확한 위치 기록
```

### 2. Redo Log Buffer 플러시 타이밍

```
Redo Log Buffer (메모리, innodb_log_buffer_size, 기본 16MB):
  트랜잭션 실행 중 변경 내용이 먼저 버퍼에 쌓임
  메모리-레벨 I/O → 매우 빠름

세 가지 플러시 시점:

시점 1: COMMIT 시 (innodb_flush_log_at_trx_commit에 따라 다름)
  설정 1: write() + fsync() → OS 버퍼 우회, 디스크에 물리적 기록
  설정 2: write()만 → OS 버퍼까지
  설정 0: 아무것도 안 함

시점 2: Redo Log Buffer가 절반 이상 차면
  대용량 트랜잭션(많은 변경) → 버퍼가 빨리 참
  절반 차면 자동으로 flush (COMMIT 여부 무관)
  → 트랜잭션 중간에도 플러시 발생 가능

시점 3: 백그라운드 스레드가 1초마다
  innodb_flush_log_at_trx_commit = 0/2인 경우의 주기적 flush
  트랜잭션이 없어도 1초마다 실행

실제 COMMIT 시퀀스 (설정 1 기준):
  ① 트랜잭션 COMMIT 요청
  ② Redo Log Buffer의 이 트랜잭션 변경 내용 확인
  ③ write(): OS 버퍼에 Redo Log 데이터 복사
  ④ fsync(): OS 버퍼 → 실제 디스크 (전원 장애에도 안전)
  ⑤ 클라이언트에 COMMIT 완료 응답 전송
  ⑥ (나중에) Dirty Pages → 데이터 파일 (백그라운드)
```

### 3. innodb_flush_log_at_trx_commit 상세 비교

```
설정 1 (기본값 — 완전 내구성):
  COMMIT마다: write() + fsync()
  내구성: MySQL 크래시 ✅ / OS 크래시 ✅ / 전원 장애 ✅
  성능: 가장 낮음 (매 COMMIT마다 fsync)
  사용: 금융, 결제, 주문 — 데이터 손실이 치명적인 서비스
  
  fsync()란:
    write()는 OS 버퍼에만 쓰고 즉시 반환 (빠름)
    fsync()는 OS 버퍼 → 물리 디스크까지 강제 플러시
    전원이 나가도 디스크에 기록된 데이터는 남음
    단, SSD의 Write Cache가 활성화되어 있으면 fsync도 믿기 어려움
    → innodb_flush_method = O_DIRECT_NO_FSYNC 조합 고려

설정 2 (MySQL 크래시 안전):
  COMMIT마다: write()만 (OS 버퍼까지)
  1초마다: fsync() (백그라운드)
  내구성: MySQL 크래시 ✅ / OS 크래시 ❌(최대 1초 손실) / 전원 장애 ❌
  성능: 설정 1 대비 3~10배 향상
  사용: Replica 서버, 분석 DB, 로그 DB
  이유: MySQL이 죽어도 OS 버퍼가 남아있어 복구 가능
       OS 자체가 다운되면 OS 버퍼도 소실

설정 0 (최고 성능, 최저 내구성):
  COMMIT마다: 아무것도 안 함
  1초마다: write() + fsync()
  내구성: MySQL 크래시 ❌(최대 1초 손실) / OS 크래시 ❌ / 전원 장애 ❌
  성능: 설정 1 대비 5~15배 향상
  사용: 개발/테스트 환경에서만
  
  MySQL 크래시 시에도 손실:
    설정 2는 write()를 했으므로 OS 버퍼에 남음 → MySQL 재시작 후 복구
    설정 0은 write()도 안 했으므로 OS 버퍼에 없음 → 복구 불가

비교 표:
           COMMIT 동작     MySQL 크래시   OS 크래시    성능
  설정 1   write+fsync     안전          안전         ★★★
  설정 2   write만         안전          최대 1초     ★★★★★
  설정 0   아무것도 없음   최대 1초      최대 1초     ★★★★★★
```

### 4. Group Commit — 동시성에서 IOPS 절감

```
문제:
  동시 접속자 1000명, 각자 초당 1 COMMIT
  → 1초에 1000번 fsync
  → SSD도 IOPS 한계 존재 → 병목
  → 병목 = 처리량 감소

Group Commit 해결책:
  여러 COMMIT 요청을 묶어 단 1번의 fsync로 처리

동작 원리 (3단계 파이프라인):
  Stage 1 — Flush 단계:
    리더 스레드: 자신의 Redo Log를 OS 버퍼에 write()
    팔로워 스레드들: 리더 뒤에 줄 서서 대기
    리더가 자신 포함 대기 중인 모든 스레드의 Redo Log를 OS 버퍼에 write()

  Stage 2 — Sync 단계:
    리더 스레드: 1번의 fsync() 실행
    → 자신 + 모든 팔로워의 Redo Log 한 번에 디스크에 기록

  Stage 3 — Commit 단계:
    리더가 각 스레드의 COMMIT을 순서대로 완료 처리
    모든 스레드에 "COMMIT 완료" 응답 전송

결과:
  1000개의 COMMIT → 1번의 fsync
  IOPS: 1000 → 1 (이상적 경우)
  처리량: 최대 1000배 향상 (이론적)

실제 효과 측정:
  SHOW STATUS LIKE 'Innodb_os_log_fsyncs';   -- 총 fsync 횟수
  SHOW STATUS LIKE 'Com_commit';              -- 총 COMMIT 횟수
  
  비율: Com_commit / Innodb_os_log_fsyncs
  >> 1이면 Group Commit 효과적으로 동작 중
  = 1이면 각 COMMIT마다 개별 fsync (동시성 낮거나 Group Commit 미작동)

innodb_commit_concurrency 설정:
  동시에 COMMIT 단계에 진입 가능한 스레드 수 제한
  0 (기본): 제한 없음
  N: 최대 N개 스레드 동시 COMMIT
  → 일반적으로 기본값 0이 최적
```

### 5. Checkpoint와 Redo Log 공간 재활용

```
Checkpoint란:
  "Redo Log의 이 LSN 이전의 모든 변경이 데이터 파일에 기록됨"
  을 공식적으로 선언하는 이벤트

왜 Checkpoint가 필요한가:
  Redo Log는 순환 파일 → 오래된 내용을 덮어쓰며 재사용
  하지만 아직 데이터 파일에 반영 안 된 변경을 덮어쓰면?
  → 크래시 후 해당 변경을 재생할 Redo Log가 없음 → 데이터 손실

  따라서:
    Dirty Page를 데이터 파일에 flush → Checkpoint 전진
    → Checkpoint 이전 Redo Log 공간 재사용 가능

Sharp Checkpoint vs Fuzzy Checkpoint:
  Sharp Checkpoint:
    서버 종료 시: 모든 Dirty Page를 데이터 파일에 flush
    → 재시작 시 Redo Log 재생 불필요 → 빠른 시작
    → 하지만 shutdown 시 모든 Dirty Page flush → 시간 걸림

  Fuzzy Checkpoint (일반 운영):
    InnoDB Buffer Pool에서 Dirty Page를 점진적으로 flush
    Page Cleaner 스레드가 백그라운드에서 처리
    Checkpoint LSN을 점진적으로 전진

Checkpoint Gap이 성능에 미치는 영향:
  Checkpoint Gap = Current LSN - Checkpoint LSN
  
  Gap이 Redo Log 크기의 75% 초과:
    → InnoDB가 Checkpoint 가속 (Dirty Page flush 강제)
    → 갑작스러운 I/O 스파이크
  
  Gap이 90% 초과:
    → 쓰기 작업 일시 중단
    → "스로틀링" → 응답 시간 급증
  
  예방:
    innodb_redo_log_capacity를 충분히 크게 설정
    권장: 1시간 분량의 Redo Log 쓰기량
    측정: SHOW STATUS LIKE 'Innodb_os_log_written'; (초당 쓰기량 측정)
```

### 6. Binary Log와 Redo Log의 2PC

```
Binary Log (MySQL 서버 레벨):
  목적: Replication + PITR (Point-in-Time Recovery)
  형식: 논리적 변경 기록 (SQL 또는 Row 이벤트)
  위치: MySQL 서버 관리 (InnoDB 외부)

Redo Log (InnoDB 레벨):
  목적: 단일 서버 크래시 복구
  형식: 물리적 변경 기록 (Page 수준)
  위치: InnoDB 내부 관리

2PC (2-Phase Commit) — 두 Log의 일관성 보장:
  COMMIT 내부 실행 순서:
    ① Redo Log에 "PREPARE" 기록 (innodb 레벨 준비 완료)
    ② Binary Log 기록 + fsync
    ③ Redo Log에 "COMMIT" 기록
    ④ 클라이언트에 완료 응답

  크래시 복구 시 판단:
    Redo Log: COMMIT 있음 + Binary Log: 있음 → 정상 커밋
    Redo Log: PREPARE만 있음 + Binary Log: 있음 → COMMIT 완료로 처리
    Redo Log: PREPARE만 있음 + Binary Log: 없음 → ROLLBACK 처리

  이 구조로:
    Redo Log와 Binary Log의 불일치 방지
    Replica에서 이미 적용된 트랜잭션이 Primary에서 사라지는 사태 방지

sync_binlog 설정 (Binary Log 내구성):
  0: OS가 알아서 flush (빠름, 내구성 없음)
  1: 트랜잭션마다 fsync (느림, 완전 내구성)
  N: N번 커밋마다 fsync (중간)
  
  완전 내구성: innodb_flush_log_at_trx_commit=1 + sync_binlog=1
  Primary 권장 설정
```

---

## 💻 실전 실험

### 실험 1: flush 설정별 INSERT 성능 비교

```sql
CREATE TABLE bench_durability (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    data VARCHAR(200) NOT NULL
) ENGINE=InnoDB;

-- 설정 1: 완전 내구성
SET GLOBAL innodb_flush_log_at_trx_commit = 1;

SET @start = NOW(6);
-- 자동 커밋으로 개별 INSERT (각각 별도 트랜잭션)
INSERT INTO bench_durability (data) VALUES (REPEAT('a', 200));
-- ... (1000번 반복)
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS ms_setting1;

-- 설정 2: MySQL 크래시 안전
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
TRUNCATE TABLE bench_durability;

SET @start = NOW(6);
-- 동일한 1000번 INSERT
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS ms_setting2;

-- 배치 vs 개별 커밋 비교 (설정 1 기준)
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
TRUNCATE TABLE bench_durability;

SET @start = NOW(6);
START TRANSACTION;
-- 1000번 INSERT를 하나의 트랜잭션으로
INSERT INTO bench_durability (data)
SELECT REPEAT('b', 200) FROM information_schema.columns LIMIT 1000;
COMMIT;  -- fsync 1번
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS ms_batch;
-- 개별 INSERT 1000번 vs 배치 1번의 차이 확인

-- 원복
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
DROP TABLE bench_durability;
```

### 실험 2: Redo Log 상태 모니터링

```sql
-- Redo Log 쓰기 속도 측정 (분당 기록량)
SET @before = (SELECT variable_value
               FROM performance_schema.global_status
               WHERE variable_name = 'Innodb_os_log_written');
SELECT SLEEP(60);  -- 1분 대기
SET @after = (SELECT variable_value
              FROM performance_schema.global_status
              WHERE variable_name = 'Innodb_os_log_written');

SELECT ROUND((@after - @before) / 1024 / 1024, 2) AS redo_mb_per_minute;
-- 권장 Redo Log 크기 = 이 값의 60배 이상 (1시간치)

-- Checkpoint 상태 확인
SHOW ENGINE INNODB STATUS\G
-- LOG 섹션:
-- Log sequence number: 현재 쓰기 위치 (최신 LSN)
-- Log flushed up to: OS 버퍼에 write()된 LSN
-- Pages flushed up to: Dirty Page가 디스크에 기록된 LSN (Checkpoint)
-- Last checkpoint at: 최신 Checkpoint LSN

-- Checkpoint Gap 계산:
-- (Log sequence number) - (Last checkpoint at) = Gap
-- Gap / innodb_redo_log_capacity × 100 = Gap 비율 (%)
-- 75% 이상이면 성능 저하 시작, 90% 이상이면 쓰기 중단 위험

-- Group Commit 효과 측정
FLUSH STATUS;
-- (많은 동시 쓰기 실행)
SHOW STATUS LIKE 'Innodb_os_log_fsyncs';
SHOW STATUS LIKE 'Com_commit';
-- Com_commit / Innodb_os_log_fsyncs > 1이면 Group Commit 동작 중
```

### 실험 3: 크래시 복구 시뮬레이션 (안전한 방법)

```sql
-- 설정 1에서의 내구성 확인

-- 1. 고유한 데이터 삽입
SET GLOBAL innodb_flush_log_at_trx_commit = 1;

START TRANSACTION;
INSERT INTO orders (user_id, amount, status, created_at)
VALUES (99999, 88888.88, 'DURABILITY_TEST', NOW());
COMMIT;
-- → COMMIT 시 Redo Log fsync 완료

-- 2. Redo Log 상태 확인
SHOW ENGINE INNODB STATUS\G
-- 방금 COMMIT된 LSN이 "Log flushed up to"에 반영됐는지 확인

-- 3. 실제 크래시 시뮬레이션은 위험하므로 이론적 확인:
-- COMMIT이 완료됐고 "Log flushed up to" = 최신 LSN이면
-- 어떤 크래시에도 이 데이터는 복구 가능

-- 설정 0에서의 취약점 확인
SET GLOBAL innodb_flush_log_at_trx_commit = 0;
INSERT INTO orders (user_id, amount, status) VALUES (99999, 77777, 'RISKY_TEST');
COMMIT;
-- "완료" 응답 받았지만 OS 버퍼에만 존재
-- 이 시점에 OS가 다운되면 이 레코드는 사라짐

SHOW ENGINE INNODB STATUS\G
-- "Log flushed up to"가 방금 COMMIT한 LSN보다 낮을 수 있음

SET GLOBAL innodb_flush_log_at_trx_commit = 1;  -- 원복!
```

### 실험 4: Redo Log 크기와 성능 안정성

```sql
-- 현재 Redo Log 설정 확인
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';
-- MySQL 8.0.30+

SHOW VARIABLES LIKE 'innodb_log_file%';
-- MySQL 8.0.29 이하

-- Redo Log 용량 대비 현재 쓰기 부하 비율
SELECT
    variable_value AS redo_capacity_bytes
FROM performance_schema.global_variables
WHERE variable_name = 'innodb_redo_log_capacity';

-- 시간당 Redo Log 쓰기량 vs 총 용량 비율 계산
-- redo_mb_per_minute × 60 / (innodb_redo_log_capacity / 1024^2) × 100
-- 이 비율이 100% 이상이면 Redo Log가 너무 작아서 Checkpoint Gap 문제 발생

-- Adaptive Flushing 동작 확인
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_dirty';
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_flushed';
-- 지속적으로 증가하면 Checkpoint 진행 중
-- 갑자기 flushed가 급증하면 강제 Checkpoint 발생 (쓰기 스파이크)
```

---

## 📊 성능 비교

```
innodb_flush_log_at_trx_commit 설정별 성능 (SSD 기준):

단건 INSERT 1만 건 (각각 별도 COMMIT):
  설정 1: ~5,000ms  (1만번 fsync)
  설정 2: ~500ms    (1초에 1번 fsync, ~10번)
  설정 0: ~400ms    (1초에 1번 fsync)
  배치(설정 1): ~50ms (1번 fsync)

Group Commit 효과 (동시 접속자 100명):
  Group Commit 없음: 100 COMMIT/초 × 100 TPS = 10,000 fsync/초
  Group Commit 있음: 동시 COMMIT들을 묶어 ~100 fsync/초
  → IOPS 100배 절감 → TPS 100배 향상 가능

Redo Log 크기 영향:
  너무 작음 (48MB): Checkpoint Gap 75% 도달 빈번
    → 주기적 쓰기 스파이크 (p99 응답시간 급증)
  적절한 크기 (4GB): Checkpoint Gap 여유 있음
    → 안정적인 쓰기 성능
  너무 큼 (100GB): Checkpoint 간격 길어짐
    → 크래시 후 복구 시간 증가 (Redo Log 재생 시간 ∝ Log 크기)

fsync 지연 측정:
  SELECT variable_value / 1e9 AS avg_fsync_ms
  FROM performance_schema.global_status  -- 평균 fsync 시간 확인
  WHERE variable_name = 'Innodb_dblwr_writes';
```

---

## ⚖️ 트레이드오프

```
innodb_flush_log_at_trx_commit 선택:

설정 1 (완전 내구성):
  ✅ 전원 장애에도 COMMIT된 데이터 100% 보장
  ✅ 금융/결제/주문 서비스에 필수
  ❌ 개별 COMMIT이 많으면 IOPS 병목
  해결: 배치 처리로 COMMIT 수 줄이기 + Group Commit 활용

설정 2 (MySQL 크래시 안전):
  ✅ 성능 3~10배 향상
  ✅ MySQL 프로세스 크래시에 안전
  ❌ OS 크래시/전원 장애 시 최대 1초 손실
  적합: Replica 서버, 배치 DB, 분석 DB
  부적합: 결제, 계좌 이체가 있는 Primary

설정 0:
  ✅ 최고 성능 (개발/테스트에서 빠른 피드백)
  ❌ MySQL 크래시에도 데이터 손실 가능
  절대 금지: 프로덕션 환경

Redo Log 크기 설정:
  너무 작으면: Checkpoint 스파이크 → 응답시간 불안정
  너무 크면: 크래시 복구 시간 증가
  권장: 1시간 분량의 Redo Log 쓰기량 (최소 2GB, 고쓰기 환경 4~8GB)

Binary Log 내구성 조합:
  Primary (완전): innodb_flush_log_at_trx_commit=1 + sync_binlog=1
  Replica (성능): innodb_flush_log_at_trx_commit=2 + sync_binlog=0
  → Primary 데이터는 완전 보호, Replica는 재동기화 가능
```

---

## 📌 핵심 정리

```
Redo Log 핵심:

WAL 원칙:
  데이터 파일 변경 전 Redo Log 먼저 디스크에 기록
  COMMIT = Sequential Redo Log Write (빠름)
  Dirty Page → 데이터 파일 = 비동기 백그라운드 (COMMIT 차단 않음)

innodb_flush_log_at_trx_commit:
  1 (기본): COMMIT마다 write + fsync → 완전 내구성
  2: COMMIT마다 write만, 1초마다 fsync → MySQL 크래시 안전
  0: 1초마다 write + fsync → 최고 성능, 최저 내구성

Group Commit:
  여러 동시 COMMIT을 묶어 1번 fsync로 처리
  높은 동시성에서 IOPS 수십~수백 배 절감
  Com_commit / Innodb_os_log_fsyncs 비율로 효과 측정

Checkpoint:
  Dirty Pages가 데이터 파일에 반영된 지점
  Checkpoint Gap이 크면 쓰기 스파이크 발생
  적절한 Redo Log 크기(1시간 분량)가 안정적 성능 보장

Binary Log + Redo Log 2PC:
  PREPARE → Binary Log Write → COMMIT
  불일치 방지 → Replica 동기화 신뢰성 보장
  Primary: 두 설정 모두 1 (완전 내구성)
```

---

## 🤔 생각해볼 문제

**Q1.** `innodb_flush_log_at_trx_commit = 1`인 서버에서 하나의 트랜잭션으로 100만 건을 INSERT하는 배치와 100만 건을 건건이 INSERT(auto-commit)하는 방식의 성능 차이가 수십~수백 배 나는 이유를 Redo Log 관점에서 설명하라.

<details>
<summary>해설 보기</summary>

**fsync 횟수의 차이**가 핵심입니다.

**건건이 INSERT (auto-commit)**:
- 각 INSERT마다 1개의 트랜잭션 → COMMIT → fsync
- 100만 건 = 100만 번의 fsync
- SSD fsync 평균 ~0.1ms → 100만 × 0.1ms = 100,000ms = 약 **100초**

**하나의 트랜잭션으로 배치 INSERT**:
- 100만 건을 메모리(Redo Log Buffer + Buffer Pool)에 쌓음
- 마지막 COMMIT 1번 → fsync 1번
- 100만 건 삽입 + fsync 1회 = 약 **0.5~2초**

차이 원인: Redo Log Buffer는 메모리 → 매우 빠름. 디스크에 쓰는 것은 COMMIT 시 1번뿐.

주의사항: 100만 건을 하나의 트랜잭션으로 처리하면 Undo Log도 100만 건 쌓임. 실패 시 롤백 비용도 100만 건. 실무에서는 1만~10만 건 단위 배치 커밋이 균형점.

이것이 JPA의 `spring.jpa.properties.hibernate.jdbc.batch_size=50` 설정이 효과적인 이유입니다. 50건씩 묶어 INSERT하면 fsync 횟수를 1/50로 줄입니다.

</details>

---

**Q2.** Replica 서버에서 `innodb_flush_log_at_trx_commit = 2`로 설정했을 때, Primary가 정상인 상황에서 Replica가 OS 크래시로 다운됐다면 어떻게 복구하는가?

<details>
<summary>해설 보기</summary>

**Replica는 Primary의 Binary Log로 재동기화하면 됩니다.**

복구 과정:
1. Replica 재시작
2. MySQL이 Redo Log를 확인 — `innodb_flush_log_at_trx_commit=2`이므로 최대 1초치 Redo Log 손실
3. 손실된 트랜잭션들은 InnoDB 수준에서 롤백됨
4. Replica는 Primary에게 "내가 적용한 마지막 Binary Log position"을 보고
5. Primary는 그 position 이후의 Binary Log를 Replica에 재전송
6. Replica가 누락된 트랜잭션을 재실행 → 동기화 완료

GTID(Global Transaction ID) 방식에서는 더 간단합니다:
```sql
CHANGE MASTER TO MASTER_AUTO_POSITION=1;
START REPLICA;
-- MySQL이 자동으로 누락된 GTID를 감지하고 재동기화
```

이것이 Replica에서 `innodb_flush_log_at_trx_commit=2`가 허용되는 이유입니다. Primary가 완전한 내구성(설정 1)을 유지하는 한, Replica의 일부 손실은 항상 재동기화로 복구 가능합니다.

단, Replica가 Failover 대상(Primary로 승격 가능)이라면 설정 1로 유지해야 합니다. 승격 후 최신 데이터를 보장해야 하기 때문입니다.

</details>

---

**Q3.** `innodb_redo_log_capacity`를 1GB에서 8GB로 늘렸다. 예상되는 변화 세 가지를 설명하라. (좋은 것 2가지, 나쁜 것 1가지)

<details>
<summary>해설 보기</summary>

**좋아지는 것:**

1. **쓰기 성능 안정화** (스파이크 감소):
   Redo Log 공간이 커지면 Checkpoint Gap이 75%에 도달하는 빈도가 낮아집니다. 강제 Checkpoint(Dirty Page 급격한 flush)가 덜 발생 → 쓰기 응답시간의 p99가 안정화됩니다. 특히 쓰기 부하가 급격히 증가하는 트래픽 스파이크 시 효과적입니다.

2. **대용량 트랜잭션 처리 능력 향상**:
   1GB Redo Log로는 커밋 전에 Redo Log를 가득 채우는 대형 배치가 Checkpoint를 강제로 유발했습니다. 8GB에서는 더 큰 트랜잭션도 중간 중단 없이 처리 가능합니다.

**나빠지는 것:**

3. **크래시 후 복구 시간 증가**:
   Redo Log 크기 = 크래시 후 재생해야 할 최대 Log 양. 8GB Redo Log는 1GB 대비 최대 8배 더 긴 복구 시간이 필요할 수 있습니다. 서비스 다운타임이 길어질 수 있으므로, `innodb_redo_log_capacity`를 무작정 크게 설정하는 것은 피해야 합니다.

**실무 권장**: 1시간 분량의 Redo Log 쓰기량을 측정하고, 그것의 1~2배로 설정합니다. 쓰기량이 시간당 2GB라면 2~4GB가 적절합니다.

</details>

---

<div align="center">

**[⬅️ Consistent vs Current Read](./06-consistent-read-vs-current-read.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Lock & Concurrency ➡️](../lock-and-concurrency/01-lock-types.md)**

</div>
