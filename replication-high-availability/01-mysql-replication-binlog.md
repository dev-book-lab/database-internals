# MySQL Replication 원리 — Binary Log 기반 복제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Primary → IO Thread → Relay Log → SQL Thread로 이어지는 3단계 복제 구조의 각 단계 역할은?
- Binary Log에 기록되는 Statement-Based / Row-Based / Mixed 포맷의 정확한 차이와 각각의 trade-off는?
- Binary Log가 Redo Log와 함께 2PC(2-Phase Commit)로 동기화되는 이유는?
- `sync_binlog=1`이 없으면 Primary 크래시 후 Replica가 Primary보다 앞설 수 있는 이유는?
- Replica의 IO Thread와 SQL Thread를 분리한 설계의 의도는?
- `SHOW REPLICA STATUS\G`에서 복제 상태를 어떻게 해석하는가?

---

## 🔍 왜 이 개념이 중요한가

### 복제를 모르면 "왜 읽기가 느리지?"와 "왜 데이터가 안 보이지?"를 설명할 수 없다

```
Read Replica 없이 단일 DB:
  모든 읽기/쓰기 → Primary 1대
  트래픽 증가 → Primary CPU/IO 포화
  해결: Read Replica 추가
  
Read Replica 추가 후:
  쓰기 → Primary
  읽기 → Replica (부하 분산)
  
하지만 새 문제:
  사용자가 상품 리뷰 작성 (Primary에 INSERT)
  바로 리뷰 목록 조회 (Replica에서 SELECT)
  → 리뷰가 안 보임! (Replica에 아직 복제 안 됨)
  
  이것을 이해하려면:
  ① Primary가 어떻게 변경을 기록하는가 (Binary Log)
  ② Replica가 어떻게 수신하는가 (IO Thread, Relay Log)
  ③ 언제 적용되는가 (SQL Thread, Replication Lag)
  → 복제 원리를 알면 Lag이 왜 생기는지, 어떻게 다루는지 설계 가능
```

---

## 😱 잘못된 이해

### Before: "Replica는 Primary와 동일한 데이터를 실시간으로 가진다"

```
잘못된 믿음:
  "Replica에서 읽으면 Primary의 최신 데이터를 즉시 볼 수 있다"
  "복제는 동기적이라 Primary 커밋 = Replica도 동시 반영"

실제 비동기 복제:
  Primary: COMMIT 완료 → 클라이언트에 응답
  (이 시점에 Replica는 아직 받지도 않았을 수 있음)
  
  1단계: Primary → Binary Log 기록
  2단계: Replica IO Thread → Binary Log 요청 및 Relay Log 수신 (네트워크 I/O)
  3단계: Replica SQL Thread → Relay Log를 순서대로 적용
  
  각 단계마다 시간 소요:
  → Replication Lag 발생 (밀리초 ~ 수 분)

잘못된 결과:
  "방금 쓴 데이터를 Replica에서 읽으면 항상 나온다"
  → 짧은 Lag에서는 우연히 맞을 수 있음
  → Lag이 증가하면 안 보임
  → 이런 버그는 재현이 어려워 오래 살아남음
```

---

## ✨ 올바른 이해

### After: Replication = 비동기 이벤트 스트림, Lag은 구조적 특성

```
복제 3단계 구조:

단계 1 — Binary Log 기록 (Primary):
  모든 DML(INSERT/UPDATE/DELETE)을 Binary Log에 기록
  COMMIT 시 Redo Log와 2PC로 원자적 기록
  파일: mysql-binlog.000001, .000002, ...

단계 2 — IO Thread (Replica):
  Primary에 연결 → Binary Log 변경 이벤트 요청
  수신한 이벤트 → Relay Log 파일에 기록
  Primary와는 독립적 (Primary 부하에 영향 없음)

단계 3 — SQL Thread (Replica):
  Relay Log에서 이벤트를 순서대로 읽기
  자신의 DB에 SQL 적용 (단일 스레드 또는 병렬)
  적용 완료 위치 → relay-log.info 파일에 저장

비동기 특성:
  각 단계가 독립적으로 동작
  네트워크 지연, Replica 처리 속도에 따라 Lag 발생
  Primary 장애 시 최대 Lag만큼 데이터 손실 가능 (RPO = Lag)
  
세미-동기 복제 (Semi-Synchronous):
  COMMIT 전 최소 1개 Replica가 Relay Log 수신 확인
  → 데이터 손실 가능성 감소 (하지만 0은 아님)
  → Primary 응답 시간 약간 증가 (네트워크 RTT만큼)
```

---

## 🔬 내부 동작 원리

### 1. Binary Log 기록 구조

```
Binary Log 파일:
  /var/lib/mysql/mysql-binlog.000001
  /var/lib/mysql/mysql-binlog.000002 (크기 초과 또는 FLUSH LOGS 시 새 파일)
  /var/lib/mysql/mysql-binlog.index (파일 목록)

Binary Log 이벤트 구조:
  [이벤트 헤더]
    Timestamp: 이벤트 발생 시각
    Event Type: QUERY / TABLE_MAP / WRITE_ROWS / UPDATE_ROWS / DELETE_ROWS / XID
    Server ID: 이 이벤트를 생성한 서버 고유 ID
    Event Length: 이벤트 데이터 크기
    Next Position: 다음 이벤트 시작 위치 (Replica가 읽어야 할 위치 추적에 사용)
    Flags
  [이벤트 데이터]
    SBR: SQL 텍스트
    RBR: 변경된 Row의 Before/After Image

Binary Log 포맷 (binlog_format 설정):
  Statement-Based (SBR):
    기록: 실행된 SQL 텍스트
    예: "UPDATE orders SET status='PAID' WHERE id=42"
    장점: 로그 크기 작음 (WHERE 조건에 맞는 N개 Row도 SQL 1개)
    단점: 비결정적 함수 → Replica에서 다른 결과
          NOW(), UUID(), RAND() → SBR에서 위험
    예: UPDATE t SET ts = NOW() 
        Primary: NOW() = 14:00:00.001
        Replica: NOW() = 14:00:00.050 → 다른 값!

  Row-Based (RBR):
    기록: 변경된 Row의 실제 데이터 (Before/After Image)
    예: [table=orders, row=id:42, before:{status:'PENDING'}, after:{status:'PAID'}]
    장점: 결정론적 → 항상 동일한 결과 보장
          UDF, 트리거, Stored Procedure에서도 안전
    단점: 로그 크기 큼 (100만 건 UPDATE = 100만 개 Row Image)

  Mixed (MBR):
    기본: SBR
    비결정적 요소 감지 시: 자동으로 RBR 전환
    MySQL 8.0 기본값: RBR (Row-Based 권장)

sync_binlog 설정:
  0: OS 버퍼에만 → OS 크래시 시 Binary Log 손실 → Replica와 불일치 가능
  1: 각 COMMIT마다 fsync → 완전한 내구성
  N: N번 커밋마다 fsync → 절충
  
  sync_binlog = 0 상태에서 Primary 크래시:
    Binary Log의 일부 이벤트가 Replica에는 전송됐지만
    Primary의 Binary Log 파일에는 없는 상태 가능
    → Replica가 Primary보다 앞서는 "ahead of primary" 상태
    → Failover 시 데이터 불일치!
  
  권장: sync_binlog = 1 (완전 내구성, Primary에서 필수)
```

### 2. IO Thread와 SQL Thread — 분리 설계의 의도

```
Replica의 두 스레드:

IO Thread:
  역할: Primary에서 Binary Log 이벤트를 가져와 Relay Log에 기록
  Primary와 연결을 항상 유지
  병목: 네트워크 대역폭, Primary의 Binary Log 읽기 속도
  
  IO Thread가 독립적인 이유:
    SQL Thread가 느려도 IO Thread는 계속 가져옴
    → Relay Log = 버퍼 역할 (Primary와 Replica 속도 차이 흡수)
    → Primary가 바쁠 때도 Binary Log 전달은 즉시

SQL Thread:
  역할: Relay Log의 이벤트를 순서대로 자신의 DB에 적용
  기본: 단일 스레드 (순서 보장)
  병목: Relay Log 적용 속도 (CPU, 디스크 I/O)
  
  병렬 복제 (Multi-Threaded Replica):
    MySQL 5.7+: DB 단위 병렬화 (각 DB = 독립 워커)
    MySQL 8.0+: LOGICAL_CLOCK 방식 (Primary의 Bingroup Commit 순서 보존)
    설정: replica_parallel_workers = 4 (또는 8)
    → 단일 스레드 대비 복제 Lag 대폭 감소

분리 설계 덕분에:
  Primary 장애 → IO Thread가 Relay Log에 최대한 받아둠
  SQL Thread가 느려도 Primary에는 영향 없음
  Relay Log 크기 = 아직 적용 안 된 이벤트 = Lag의 크기
```

### 3. 2PC — Binary Log와 Redo Log의 일관성

```
왜 2PC가 필요한가:
  Binary Log: MySQL 서버 레벨 (복제 목적)
  Redo Log: InnoDB 레벨 (내구성 목적)
  
  두 로그가 불일치하면:
    Binary Log에 있지만 Redo Log에 없음
    → Replica에는 트랜잭션 있음, Primary 재시작 후는 없음 → 불일치!

2PC 실행 순서:
  ① InnoDB: 트랜잭션 PREPARE (Redo Log에 "PREPARE" 기록)
  ② MySQL 서버: Binary Log에 WRITE + fsync (sync_binlog=1)
  ③ InnoDB: Redo Log에 "COMMIT" 기록
  ④ 클라이언트에 성공 응답

크래시 복구 판단:
  Redo Log: COMMIT 있음 + Binary Log: 있음 → 정상 COMMIT
  Redo Log: PREPARE + Binary Log: 있음 → XA Recovery → COMMIT 처리
  Redo Log: PREPARE + Binary Log: 없음 → XA Recovery → ROLLBACK 처리

결과:
  Binary Log ↔ Redo Log 완전 동기화
  Replica도 Primary와 동일한 상태 보장 (sync_binlog=1 조건)

Binary Log 위치 추적:
  SHOW BINARY LOG STATUS\G  (또는 SHOW MASTER STATUS)
  → File: mysql-binlog.000023
  → Position: 1234567
  → 이 위치까지 Replica가 적용했는지 비교 가능
```

### 4. 복제 설정과 상태 확인

```sql
-- Primary 설정 (my.cnf):
-- [mysqld]
-- server-id = 1                   ← 복제 그룹 내 고유 ID (필수)
-- log-bin = mysql-binlog           ← Binary Log 활성화
-- binlog-format = ROW              ← Row-Based 복제 (권장)
-- sync-binlog = 1                  ← 내구성 (필수)
-- binlog-expire-logs-seconds = 604800  ← Binary Log 7일 보관

-- Replica 설정:
-- [mysqld]
-- server-id = 2                   ← Primary와 다른 ID
-- relay-log = relay-log            ← Relay Log 파일명
-- read-only = 1                   ← Replica에 쓰기 방지
-- replica-parallel-workers = 4    ← 병렬 복제 (MySQL 8.0)
-- replica-parallel-type = LOGICAL_CLOCK

-- Replica 초기 설정 (MySQL 8.0):
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='primary-host',
    SOURCE_USER='replication_user',
    SOURCE_PASSWORD='password',
    SOURCE_LOG_FILE='mysql-binlog.000001',
    SOURCE_LOG_POS=157;
START REPLICA;

-- 복제 상태 확인 (핵심 항목):
SHOW REPLICA STATUS\G
-- 확인 항목:
-- Replica_IO_Running: Yes/No   ← IO Thread 동작 여부
-- Replica_SQL_Running: Yes/No  ← SQL Thread 동작 여부
-- Source_Log_File: mysql-binlog.000023  ← 현재 읽는 Primary Binary Log
-- Read_Source_Log_Pos: 1234567          ← IO Thread가 읽은 위치
-- Relay_Log_File: relay-log.000015     ← 현재 적용 중인 Relay Log
-- Exec_Source_Log_Pos: 1230000         ← SQL Thread가 적용한 Primary 위치
-- Seconds_Behind_Source: 5             ← 복제 Lag (초 단위)
-- Last_SQL_Error: ""                   ← 에러 메시지 (있으면 복제 중단!)

-- Binary Log 내용 확인:
SHOW BINARY LOGS;
-- Log_name: mysql-binlog.000023, File_size: 1073741824

SHOW BINLOG EVENTS IN 'mysql-binlog.000023' LIMIT 20\G
-- 각 이벤트의 타입, 위치, SQL 내용 확인

-- mysqlbinlog 툴 (커맨드라인):
-- mysqlbinlog --base64-output=DECODE-ROWS --verbose mysql-binlog.000023
-- → Row 이벤트도 사람이 읽을 수 있는 형태로 출력
```

---

## 💻 실전 실험

### 실험 1: Binary Log 이벤트 직접 확인

```sql
-- Binary Log 활성화 확인
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';

-- 현재 Binary Log 파일 확인
SHOW BINARY LOG STATUS\G
-- File: mysql-binlog.000001
-- Position: 157 (초기값)

-- DML 실행 후 Binary Log 이벤트 확인
CREATE TABLE binlog_test (id INT PRIMARY KEY, val VARCHAR(100));
INSERT INTO binlog_test VALUES (1, 'hello');
UPDATE binlog_test SET val = 'world' WHERE id = 1;
DELETE FROM binlog_test WHERE id = 1;

-- 이벤트 확인
SHOW BINLOG EVENTS IN 'mysql-binlog.000001'\G
-- 이벤트 타입: TABLE_MAP, WRITE_ROWS, UPDATE_ROWS, DELETE_ROWS (RBR)
-- 또는: QUERY (SBR)

-- RBR에서 Row 데이터 확인 (커맨드라인):
-- mysqlbinlog --base64-output=DECODE-ROWS -v mysql-binlog.000001

DROP TABLE binlog_test;
```

### 실험 2: SBR vs RBR 로그 크기 비교

```sql
-- SBR로 설정
SET SESSION binlog_format = 'STATEMENT';
SHOW BINARY LOG STATUS\G  -- 시작 위치 기록

-- 대량 UPDATE
UPDATE orders SET memo = CONCAT('test_', id) WHERE id <= 10000;

SHOW BINARY LOG STATUS\G  -- 종료 위치 기록
-- 위치 차이 = SBR 로그 크기

-- RBR로 설정
SET SESSION binlog_format = 'ROW';
SHOW BINARY LOG STATUS\G

UPDATE orders SET memo = CONCAT('test_', id) WHERE id <= 10000;

SHOW BINARY LOG STATUS\G
-- 위치 차이 = RBR 로그 크기
-- SBR: SQL 1개 → 수십 bytes
-- RBR: Row 10000개 × 각 크기 → 수십 MB

SET SESSION binlog_format = 'ROW';  -- 원복 (기본값)
```

### 실험 3: Replica 상태 모니터링

```sql
-- Replica 서버에서:
SHOW REPLICA STATUS\G

-- IO Thread, SQL Thread 모두 Yes인지 확인
-- Seconds_Behind_Source: 0이면 동기화됨

-- 복제 에러 시뮬레이션 (Replica에서):
-- Primary: 존재하지 않는 Row DELETE
-- → Replica: Row 없어서 에러 → SQL Thread 중단

-- 에러 복구 방법:
-- STOP REPLICA SQL_THREAD;
-- SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;  -- 에러 이벤트 건너뜀
-- START REPLICA SQL_THREAD;
-- (운영에서는 원인 파악 후 사용)
```

---

## 📊 성능 비교

```
Binary Log 포맷별 특성:

              SBR          RBR            MBR
로그 크기     작음          큼             중간
결정론적      아님          항상           조건부
복제 안전성   낮음          높음           높음
DDL 복제      지원          지원           지원
비결정 함수   위험          안전           안전
MySQL 8.0기본 아님          ✅             아님

Replication Lag에 영향을 주는 요인:
  네트워크 대역폭: Binary Log 전송 속도
  Replica SQL Thread: 단일 스레드 처리 속도
  대용량 트랜잭션: 큰 트랜잭션 = 오래 적용
  Replica 하드웨어: SSD vs HDD (Relay Log 쓰기)

단일 SQL Thread vs 병렬 복제:
  단일 스레드: Lag 발생 시 선형 증가
  병렬 (workers=4): Primary TPS ≤ 4× Replica TPS → Lag 0 유지
  Large transaction (100만 건 UPDATE):
    병렬화 불가 → 적용 동안 Lag 급증
```

---

## ⚖️ 트레이드오프

```
binlog_format 선택:

RBR (권장, MySQL 8.0 기본):
  ✅ 결정론적 복제 → 안전
  ✅ GTID 복제와 필수 조합
  ✅ read_committed 격리 수준과 호환
  ❌ 로그 크기 큼 (대량 DML 시)
  ❌ 디스크/네트워크 사용량 증가

SBR:
  ✅ 로그 크기 작음
  ❌ 비결정적 함수 → 복제 불일치 위험
  ❌ read_committed에서 안전하지 않음

sync_binlog 선택:
  1 (권장, Primary 필수):
    ✅ 완전한 내구성
    ❌ 쓰기 성능 저하 (COMMIT마다 fsync)
  0:
    ✅ 성능 향상
    ❌ Primary 크래시 시 Binary Log 손실 → Replica와 불일치

세미-동기 복제:
  ✅ Primary 크래시 시 데이터 손실 위험 감소
  ✅ 적어도 1개 Replica가 받았음을 보장
  ❌ Primary COMMIT 응답 시간 증가 (네트워크 RTT)
  ❌ 모든 Replica 응답 불가 시 비동기로 자동 전환
```

---

## 📌 핵심 정리

```
Binary Log 복제 핵심:

3단계 구조:
  Primary → Binary Log 기록
  IO Thread → Relay Log 수신 (네트워크)
  SQL Thread → Relay Log 적용 (DB에 실행)

Binary Log 포맷:
  RBR (Row-Based): MySQL 8.0 기본, 결정론적, 로그 큼
  SBR (Statement): 로그 작음, 비결정적 위험
  혼합: Mixed (조건부 자동 선택)

2PC:
  Binary Log + Redo Log 동기화 보장
  sync_binlog=1 + innodb_flush_log_at_trx_commit=1 = 완전 내구성

Replica 상태 확인:
  SHOW REPLICA STATUS\G
  Seconds_Behind_Source = 복제 Lag
  Replica_IO_Running + Replica_SQL_Running = Yes = 정상

병렬 복제:
  replica_parallel_workers = 4 (MySQL 8.0)
  LOGICAL_CLOCK 방식 → Primary 처리 순서 보존
```

---

## 🤔 생각해볼 문제

**Q1.** Primary에서 `UPDATE orders SET status='PAID' WHERE created_at < '2024-01-01'`이 실행되어 100만 건이 변경됐다. RBR 포맷에서 이 이벤트를 Replica가 적용하는 동안 어떤 현상이 발생하는가?

<details>
<summary>해설 보기</summary>

**Replication Lag 급증**이 발생합니다.

RBR에서 100만 건 UPDATE는 Binary Log에 100만 개의 Row 이벤트로 기록됩니다. Replica의 SQL Thread는 이를 단일 스레드로 순서대로 적용합니다.

발생하는 현상:
1. **Binary Log 크기**: 100만 Row × 각 Row의 Before/After Image → 수백 MB ~ 수 GB
2. **IO Thread**: 대용량 Binary Log를 Relay Log에 빠르게 수신 (I/O 부하)
3. **SQL Thread**: 100만 건 적용 = Primary 실행 시간과 유사 (Lag 급증)
4. **Replica Lag**: 적용 완료까지 수분 ~ 수십 분 (대형 트랜잭션은 병렬화 불가)
5. **이 기간 동안**: Replica에서 읽는 애플리케이션은 오래된 데이터를 봄

예방 방법:
```sql
-- 대량 DML은 소량씩 나눠 실행
UPDATE orders SET status='PAID'
WHERE created_at < '2024-01-01' AND id > ? LIMIT 1000;
-- 반복 실행 → 각 1000건씩 → Lag이 크게 생기지 않음
```

실무에서 pt-online-schema-change, gh-ost도 이 원리로 동작합니다: 대량 변경을 소량씩 나눠 Replica Lag이 임계값을 초과하면 일시 중지 후 재개합니다.

</details>

---

**Q2.** Replica의 IO Thread는 Running인데 SQL Thread가 Stopped 상태다. `Last_SQL_Error`에는 에러 메시지가 없다. 어떤 원인을 의심하고 어떻게 진단하는가?

<details>
<summary>해설 보기</summary>

**SQL Thread가 에러 없이 중단되는 주요 원인**:

1. **명시적으로 중단된 경우**: `STOP REPLICA SQL_THREAD`가 실행됐거나 재시작 스크립트가 SQL Thread만 중단

2. **대형 트랜잭션 처리 중**: 매우 큰 트랜잭션 적용 중에는 `Replica_SQL_Running: Yes`처럼 보이지만, 특정 버전에서는 `Stopped`처럼 보일 수 있음

3. **DDL 충돌**: Replica에 이미 같은 이름의 테이블이 있거나, 존재하지 않는 테이블에 DML이 왔을 때 (일부 버전에서 에러 없이 중단)

**진단 방법**:
```sql
-- 상세 상태 확인
SHOW REPLICA STATUS\G
-- Last_SQL_Errno: 0이어도 Last_SQL_Error 추가 확인
-- Relay_Log_File, Relay_Log_Pos 확인

-- Performance Schema로 더 상세히:
SELECT * FROM performance_schema.replication_applier_status\G
SELECT * FROM performance_schema.replication_applier_status_by_worker\G

-- MySQL 에러 로그 확인:
-- $ tail -100 /var/log/mysql/error.log

-- 수동으로 해당 위치 이벤트 확인:
SHOW RELAYLOG EVENTS IN 'relay-log.000015' FROM [Relay_Log_Pos]\G
-- 문제가 되는 이벤트 타입 확인

-- 재시작 시도:
START REPLICA SQL_THREAD;
SHOW REPLICA STATUS\G  -- 바로 또 중단되는지 확인
```

</details>

---

**Q3.** `binlog_format=ROW`에서 `UPDATE orders SET amount = amount * 1.1` (전체 Row 업데이트)을 실행했다. `binlog_row_image` 설정이 `FULL`, `MINIMAL`, `NOBLOB`일 때 각각 Binary Log에 기록되는 내용의 차이를 설명하라.

<details>
<summary>해설 보기</summary>

`binlog_row_image`는 RBR에서 Row 이미지에 포함할 컬럼을 제어합니다.

**FULL (기본값)**:
- Before Image: 모든 컬럼의 이전 값
- After Image: 모든 컬럼의 새 값
- 크기: 가장 큼
- 용도: 완전한 복구 가능, Point-in-Time Recovery에 유리
- 예: `{id:1, amount:1000, status:'PAID', created_at:'...', ...}` → `{id:1, amount:1100, ...}`

**MINIMAL (권장)**:
- Before Image: PK 또는 UNIQUE 컬럼만 (식별자)
- After Image: 실제로 변경된 컬럼만
- 크기: 가장 작음 (10~50% 감소 가능)
- 예: Before `{id:1}` → After `{amount:1100}`
- 주의: 역방향 복구(롤백) 시 PK만으로 충분

**NOBLOB**:
- MINIMAL처럼 동작하지만 BLOB/TEXT 컬럼도 포함
- BLOB 컬럼이 변경되지 않았어도 After Image에 포함
- 용도: BLOB 컬럼 변경 추적이 필요한 경우

**실무 권장**:
```sql
-- MINIMAL 설정 (디스크/네트워크 절약):
SET GLOBAL binlog_row_image = 'MINIMAL';
-- 또는 my.cnf: binlog_row_image = minimal
```
대용량 테이블에서 FULL → MINIMAL 변경 시 Binary Log 크기 30~60% 감소 효과를 볼 수 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Read Replica 정합성 ➡️](./02-read-replica-consistency.md)**

</div>
