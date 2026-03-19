# Redo Log와 트랜잭션 내구성

---

## 🎯 핵심 질문

- COMMIT 시 Redo Log를 Dirty Page보다 먼저 디스크에 쓰는 이유는?
- Redo Log Buffer에서 Redo Log File로 플러시되는 정확한 시점은?
- `innodb_flush_log_at_trx_commit` 0/1/2의 내구성 차이는?
- Group Commit이 IOPS를 줄이는 원리는?
- MySQL 8.0.30에서 Redo Log 관리 방식이 어떻게 바뀌었는가?

---

## 🔬 내부 동작 원리

### 1. WAL 원칙과 Redo Log의 역할

```
문제: Buffer Pool의 Dirty Page를 COMMIT마다 즉시 디스크에 쓰면?
  → Random I/O (데이터 파일의 임의 위치)
  → 매우 느림 → COMMIT 속도 = HDD Random I/O 속도로 제한

WAL (Write-Ahead Logging) 해결책:
  Redo Log = 변경 내용의 순차 기록 (Sequential I/O)
  원칙: "데이터 파일 변경 전에 Redo Log를 반드시 먼저 디스크에 기록"

COMMIT 과정:
  1. 트랜잭션 중 모든 변경 → Redo Log Buffer에 순차 기록
  2. COMMIT 호출
  3. Redo Log Buffer → Redo Log File (fsync, Sequential Write)
     ← 이 시점에 내구성 보장! 크래시 후에도 복구 가능
  4. 클라이언트에 COMMIT 완료 응답
  5. Buffer Pool Dirty Pages → 나중에 비동기로 데이터 파일에 반영

이점:
  COMMIT 시 필요한 I/O = Redo Log Sequential Write (빠름)
  Dirty Page Flush = 백그라운드에서 비동기 처리 (COMMIT을 차단 안 함)
  → COMMIT 속도 대폭 향상

크래시 복구:
  재시작 시 Redo Log 재생 → COMMIT된 트랜잭션 모두 복원
  미완료 트랜잭션 → Undo Log로 롤백
```

### 2. Redo Log Buffer와 File

```
Redo Log 흐름:
  트랜잭션 변경 작업
    → Redo Log Buffer (메모리, innodb_log_buffer_size, 기본 16MB)
    → Redo Log File (디스크, 순환 파일)

LSN (Log Sequence Number):
  Redo Log에서의 위치를 나타내는 단조 증가 숫자
  Buffer Pool Page마다 해당 Page의 마지막 변경 LSN 저장 (Page LSN)
  Redo Log File의 현재 끝 위치 = Latest LSN
  Checkpoint LSN = 이 LSN 이전의 모든 Dirty Page가 디스크에 반영됨을 보장

Redo Log File 구조 (MySQL 8.0.30 이전):
  ib_logfile0, ib_logfile1 (순환 사용)
  innodb_log_file_size: 각 파일 크기 (기본 48MB → 권장 256MB~수 GB)
  
  순환 구조:
    |--------ib_logfile0--------|--------ib_logfile1--------|
    ↑ Checkpoint                                   ↑ Current Write Point
    Checkpoint 이전 = 덮어써도 됨 (이미 Dirty Page 반영 완료)
    Checkpoint에서 Write Point 사이 = 아직 Dirty Page에 없는 변경
    Write Point가 Checkpoint에 가까워지면: 강제 Checkpoint (Dirty Page Flush)

MySQL 8.0.30+: innodb_redo_log_capacity
  기존 innodb_log_file_size × innodb_log_files_in_group 대신
  단일 파라미터로 전체 Redo Log 크기 설정
  내부적으로 작은 파일(32개)로 자동 관리
  SET GLOBAL innodb_redo_log_capacity = 8 * 1024 * 1024 * 1024;  -- 8GB
```

### 3. innodb_flush_log_at_trx_commit

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
-- Value: 1 (기본값)

-- 설정값별 동작:

-- 설정 1 (기본, 완전 내구성):
-- COMMIT 시: write() + fsync() 모두 실행
-- → OS 버퍼 우회하여 디스크에 물리적으로 기록
-- → MySQL 크래시 + OS 크래시 모두 안전
-- → 단점: COMMIT마다 fsync → IOPS 높음 → TPS 제한

-- 설정 2 (MySQL 크래시 안전):
-- COMMIT 시: write()만 실행 (OS 버퍼에 기록)
-- 1초마다: fsync() (백그라운드 스레드)
-- → MySQL 프로세스 크래시: OS 버퍼 남아있어 안전
-- → OS 크래시/전원 장애: 최대 1초 데이터 손실 가능
-- → 성능: 설정 1 대비 수배 향상 (fsync 빈도 대폭 감소)

-- 설정 0 (개발/테스트 환경):
-- COMMIT 시: 아무것도 하지 않음
-- 1초마다: write() + fsync()
-- → MySQL 크래시 시에도 최대 1초 손실 가능
-- → 가장 빠름, 가장 위험
-- → 프로덕션 사용 금지

성능 비교 (TPS 기준, 예시):
  설정 1: 기준 100 TPS
  설정 2: ~300~500 TPS (fsync 빈도 감소)
  설정 0: ~1000 TPS (fsync 거의 없음)
```

### 4. Group Commit — IOPS 절감

```
문제:
  동시 접속자 1000명이 각자 COMMIT
  설정 1: 1000번의 개별 fsync = 1000 IOPS!
  SSD도 IOPS 한계 있음 → 병목

Group Commit 해결책:
  여러 트랜잭션의 COMMIT을 묶어 한 번의 fsync로 처리

과정:
  TRX A: COMMIT → Redo Log Buffer flush 대기 (리더가 되거나 팔로워로 대기)
  TRX B: COMMIT → 대기
  TRX C: COMMIT → 대기
  ...
  
  리더(TRX A): 대기 중인 모든 트랜잭션의 Redo Log를 모아서
  한 번의 write() + fsync() 실행
  → A, B, C 모두 COMMIT 완료로 응답
  
  1번의 fsync = N개의 COMMIT
  → IOPS: N 대신 1 → 처리량 N배 향상

Group Commit 확인:
  SHOW STATUS LIKE 'Innodb_os_log_fsyncs';  -- 총 fsync 횟수
  SHOW STATUS LIKE 'Com_commit';            -- 총 COMMIT 횟수
  
  비율: Com_commit / Innodb_os_log_fsyncs
  >> 1이면 Group Commit 효과적으로 동작 중

BinLog Group Commit (MySQL 5.7+):
  Binary Log와 Redo Log를 함께 Group Commit
  binlog_group_commit_sync_delay: 의도적 지연 (더 많은 트랜잭션을 묶기 위해)
  binlog_group_commit_sync_no_delay_count: 이 수만큼 모이면 지연 없이 바로 flush
```

### 5. Checkpoint와 Dirty Page Flush

```
Checkpoint:
  "Redo Log의 이 LSN까지 모든 변경사항이 데이터 파일에 반영됨"
  을 기록하는 점

Checkpoint가 진행되어야 하는 이유:
  Redo Log는 순환 파일 → 오래된 Log 덮어써야 함
  덮어쓰기 전: 해당 Log의 변경이 데이터 파일에 반영됐는지 확인 필요
  → Checkpoint 전진 = 더 오래된 Redo Log 공간 재사용 가능

Checkpoint 트리거:
  Sharp Checkpoint: 서버 종료 시 모든 Dirty Page Flush
  Fuzzy Checkpoint: 백그라운드에서 점진적으로 진행
    - Innodb_log_file 공간이 75% 이상 사용 시 Checkpoint 가속
    - 공간이 90% 이상이면 강제 Checkpoint (쓰기 중단 위험!)

innodb_log_file_size가 너무 작은 경우:
  Redo Log가 빨리 차서 강제 Checkpoint 빈발
  → Dirty Page Flush가 비정기적으로 급격히 발생
  → 쓰기 성능 불안정 (스파이크)
  → 권장: 총 Redo Log 크기 = 1시간 분량의 변경 데이터
  → 확인: innodb_os_log_written 상태 변수로 시간당 쓰기량 측정
```

---

## 💻 실전 실험

### 실험 1: Redo Log 크기와 성능

```sql
-- 현재 Redo Log 상태 확인
SHOW VARIABLES LIKE 'innodb_log%';
-- innodb_log_file_size: 현재 파일 크기
-- innodb_log_files_in_group: 파일 수

-- Redo Log 쓰기 속도 측정 (분당 기록량)
SET @before = (SELECT variable_value FROM performance_schema.global_status
               WHERE variable_name = 'Innodb_os_log_written');
SELECT SLEEP(60);  -- 1분 대기
SET @after = (SELECT variable_value FROM performance_schema.global_status
              WHERE variable_name = 'Innodb_os_log_written');

SELECT (@after - @before) / 1024 / 1024 AS redo_mb_per_minute;
-- 이 값의 60배 = 시간당 쓰기량 → Redo Log 크기 권장값 산정

-- Checkpoint LSN vs Latest LSN 확인
SHOW ENGINE INNODB STATUS\G
-- LOG 섹션에서:
-- Log sequence number: Latest LSN
-- Log flushed up to: OS 버퍼에 쓰인 LSN
-- Pages flushed up to: Dirty Page에서 디스크에 쓰인 LSN (Checkpoint)
-- Last checkpoint at: 마지막 Checkpoint LSN
-- 
-- (Log sequence number - Last checkpoint at) = Checkpoint Gap
-- 이 값이 Redo Log 크기에 가까워지면 → 위험!
```

### 실험 2: flush_log_at_trx_commit별 성능 비교

```sql
-- 설정 1 기준 INSERT 성능
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
SET @start = NOW(6);
INSERT INTO bench_table (data) SELECT REPEAT('x', 100) FROM seq_1_to_10000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS ms_setting1;

-- 설정 2로 변경
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
TRUNCATE bench_table;
SET @start = NOW(6);
INSERT INTO bench_table (data) SELECT REPEAT('x', 100) FROM seq_1_to_10000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS ms_setting2;

-- 원복
SET GLOBAL innodb_flush_log_at_trx_commit = 1;

-- 예상: 설정 2가 수 배 빠름 (fsync 빈도 감소)
```

### 실험 3: Group Commit 효과 측정

```sql
-- Group Commit 비율 확인
FLUSH STATUS;  -- 통계 초기화

-- 동시에 많은 INSERT (별도 스레드에서 실행)
-- (실제로는 여러 클라이언트 연결에서 동시 실행)

SHOW STATUS LIKE 'Innodb_os_log_fsyncs';
SHOW STATUS LIKE 'Com_commit';
-- Com_commit / Innodb_os_log_fsyncs = Group Commit 배수
-- 값이 1이면 Group Commit 없음, 클수록 효과적
```

---

## 📌 핵심 정리

```
Redo Log 핵심:

WAL 원칙:
  데이터 파일 변경 전 Redo Log 먼저 디스크에 기록
  Sequential Write (빠름) → COMMIT 속도 향상
  크래시 후 Redo Log 재생으로 복구

innodb_flush_log_at_trx_commit:
  1 (기본): COMMIT마다 fsync → 완전 내구성, 낮은 TPS
  2: COMMIT마다 write, 1초마다 fsync → MySQL 크래시 안전
  0: 1초마다 write+fsync → 가장 빠름, 가장 위험

Group Commit:
  동시 COMMIT들을 묶어 1번의 fsync로 처리
  N개 COMMIT / 1 fsync → IOPS 절감, TPS 향상

Redo Log 크기:
  너무 작으면 강제 Checkpoint 빈발 → 쓰기 불안정
  권장: 1시간 분량의 Redo Log 기록량
  MySQL 8.0.30+: innodb_redo_log_capacity로 관리

LSN과 Checkpoint:
  LSN: Redo Log 내 위치 (단조 증가)
  Checkpoint: 이 LSN까지 데이터 파일 반영 완료
  Checkpoint Gap = 최신 LSN - Checkpoint LSN
  Gap이 Redo Log 크기의 75% 초과 → 성능 경고
```

---

## 🤔 생각해볼 문제

**Q1.** Replica(슬레이브) 서버의 `innodb_flush_log_at_trx_commit`을 2로 설정해도 안전한가?

<details>
<summary>해설 보기</summary>

**일반적으로 안전하지만 상황에 따라 다릅니다.**

Replica는 소스(마스터)의 Binary Log를 재생하여 동기화됩니다. Replica가 OS 크래시로 1초치 Redo Log를 잃더라도, **소스의 Binary Log를 기반으로 재동기화**가 가능합니다 (`CHANGE MASTER TO ... RELAY_LOG_FILE/POS` 또는 GTID 기반 자동 복구).

따라서 Replica에서 설정 2는 흔히 사용됩니다. 이로 인해:
- Replica의 쓰기 성능이 소스와 동일하거나 더 빠름
- 복제 지연 감소

단, 다음 경우에는 주의:
- Replica가 Failover 대상(소스 승격)이면 설정 1 권장 (승격 후 데이터 손실 방지)
- Semi-Sync 복제에서 Replica가 ACK를 보내는 역할이면 설정 1 필요

</details>

**Q2.** Redo Log와 Binary Log의 역할이 겹치는 것처럼 보이는데, 두 Log가 모두 필요한 이유는?

<details>
<summary>해설 보기</summary>

**두 Log의 목적이 다릅니다:**

**Redo Log** (InnoDB 내부):
- 목적: InnoDB 내 **크래시 복구** (단일 서버)
- 형식: 물리적 변경 기록 (어떤 Page의 어떤 오프셋이 어떤 값으로)
- 범위: 하나의 MySQL 인스턴스
- 순환 파일 (오래된 것 재사용)

**Binary Log** (MySQL 서버):
- 목적: **복제(Replication)** + **Point-in-Time Recovery** (PITR)
- 형식: 논리적 변경 기록 (어떤 SQL 또는 Row 이벤트)
- 범위: Replica 서버들과 공유
- 순차 증가 파일 (모두 보관 가능)

함께 필요한 이유:
- Redo Log만으로는 복제 불가 (물리적 형식, 단일 서버용)
- Binary Log만으로는 크래시 복구 불가 (Buffer Pool 상태 불명)

MySQL에서 COMMIT의 2-Phase Commit:
1. Redo Log 준비 (Prepare 단계)
2. Binary Log 기록
3. Redo Log 커밋 (Commit 단계)
→ 두 Log의 일관성 보장 (하나만 기록된 경우 복구 가능)

</details>

---

<div align="center">

**[⬅️ Consistent vs Current Read](./06-consistent-read-vs-current-read.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Lock & Concurrency ➡️](../lock-and-concurrency/01-row-lock-types.md)**

</div>
