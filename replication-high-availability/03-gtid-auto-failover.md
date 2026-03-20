# GTID 기반 복제와 자동 페일오버

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- GTID의 형식 `server_uuid:transaction_id`에서 각 부분이 의미하는 것은?
- `gtid_executed`와 `gtid_purged`의 차이는?
- Position 기반 복제 대비 GTID가 Failover에서 유리한 이유는?
- MHA와 Orchestrator의 Failover 처리 방식 차이는?
- Primary 장애 후 어떤 Replica를 새 Primary로 승격해야 하는가?
- GTID Failover 후 다른 Replica들이 새 Primary에 재연결하는 원리는?

---

## 🔍 왜 이 개념이 중요한가

### HA 없는 MySQL = 장애 시 수동 복구, 수십 분 다운타임

```
Position 기반 복제의 Failover 문제:

Primary 장애 발생:
  Replica A: mysql-binlog.000023 Position 1234567 까지 적용
  Replica B: mysql-binlog.000023 Position 1230000 까지 적용

새 Primary = Replica A (가장 최신)
다른 Replica들 재연결:
  Replica B: "나는 mysql-binlog.000023:1230000 이후를 주세요"
  → 새 Primary A: "나는 Position 기반 Binary Log 없음 (새로 시작)"
  → 에러! Replica B는 어디서부터 새 Primary를 따라야 하는지 모름

수동 해결 (복잡하고 오류 가능성 높음):
  각 Replica에서 Binary Log Position을 비교
  어느 이벤트까지 적용됐는지 binlog 파일 직접 분석
  CHANGE REPLICATION SOURCE TO ... 수동 설정
  → 전문 DBA가 수십 분 ~ 수 시간 소요

GTID로 해결:
  모든 트랜잭션에 전역 고유 ID 부여
  각 서버가 "내가 적용한 GTID 집합" 추적
  Failover 시:
    Replica B: "나는 uuid:1-9500까지 적용됨"
    새 Primary A: "나는 uuid:1-10000까지 있음"
    → "9501-10000 주세요" 자동 계산
    → CHANGE REPLICATION SOURCE TO SOURCE_AUTO_POSITION=1 한 줄로 해결!
```

---

## 😱 잘못된 이해

### Before: "GTID는 복제 설정을 복잡하게 만든다"

```
잘못된 믿음:
  "Position 기반이 이해하기 쉽고 충분하다"
  "GTID를 사용하면 제약이 많아진다"

실제 GTID의 장점:
  Position: Binary Log 파일명 + 오프셋 (같은 서버에서만 의미 있음)
  GTID: 전역 고유 ID (어느 서버에서도 동일)

  Failover 비교:
  Position: 복잡한 Binary Log 위치 계산 + 수동 CHANGE REPLICATION SOURCE
  GTID: SOURCE_AUTO_POSITION=1 한 줄로 자동 처리

잘못된 또 다른 믿음:
  "GTID를 쓰면 특정 트랜잭션을 재실행/건너뛸 수 없다"
  
  실제:
    GTID_EXECUTED에 특정 GTID를 수동으로 추가하면 건너뜀
    SET @@SESSION.GTID_NEXT = 'uuid:N';
    BEGIN; COMMIT;  -- 빈 트랜잭션으로 해당 GTID 건너뜀 표시
    → 에러 이벤트 건너뛰기 가능
```

---

## ✨ 올바른 이해

### After: GTID = 트랜잭션의 여권번호, 전역적으로 추적 가능

```
GTID 구조:
  source_uuid:transaction_id
  ├── source_uuid: Primary 서버의 고유 UUID
  │   (server_uuid 변수, /var/lib/mysql/auto.cnf에 저장)
  └── transaction_id: 해당 서버에서 발행한 일련번호 (1부터 단조 증가)

예시:
  3E11FA47-71CA-11E1-9E33-C80AA9429562:1
  3E11FA47-71CA-11E1-9E33-C80AA9429562:2
  3E11FA47-71CA-11E1-9E33-C80AA9429562:3
  ...

GTID 집합 (Set):
  3E11FA47-71CA-11E1-9E33-C80AA9429562:1-1000
  → UUID 서버에서 발생한 트랜잭션 1번~1000번이 모두 포함
  
  여러 서버 포함:
  3E11FA47:1-1000, 7B12CD89:1-500
  → 두 서버의 트랜잭션 집합

gtid_executed:
  이 서버에서 실행(적용)된 모든 GTID 집합
  → Primary: 자신이 실행한 GTID
  → Replica: Primary에서 받아 적용한 GTID

gtid_purged:
  Binary Log에서 삭제된 (purge된) GTID 집합
  오래된 Binary Log 삭제 시 증가
  새 Replica가 full dump 후 시작할 때 이 값 이후부터 복제
```

---

## 🔬 내부 동작 원리

### 1. GTID 발행과 추적

```sql
-- GTID 설정 (my.cnf):
-- [mysqld]
-- gtid_mode = ON
-- enforce_gtid_consistency = ON  ← GTID와 비호환 구문 금지
-- server-id = 1

-- GTID 상태 확인:
SHOW VARIABLES LIKE 'gtid_mode';           -- ON
SHOW VARIABLES LIKE 'server_uuid';         -- Primary의 UUID
SELECT @@global.gtid_executed;             -- 이 서버에서 적용된 GTID 집합
SELECT @@global.gtid_purged;               -- 삭제된 Binary Log의 GTID

-- Replica 상태에서 GTID 확인:
SHOW REPLICA STATUS\G
-- Retrieved_Gtid_Set: IO Thread가 수신한 GTID
-- Executed_Gtid_Set:  SQL Thread가 적용한 GTID
-- Retrieved와 Executed의 차이 = 아직 적용 안 된 GTID (Lag)

-- GTID 복제 설정 (Replica에서):
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = 'primary-host',
    SOURCE_USER = 'replication_user',
    SOURCE_PASSWORD = 'password',
    SOURCE_AUTO_POSITION = 1;  -- ← GTID 자동 위치 결정!
    -- Position을 지정할 필요 없음
    -- Replica의 gtid_executed를 Primary에 보내서 자동 계산

START REPLICA;

-- 자동 위치 결정 원리:
-- Replica: "나는 UUID:1-9500을 가지고 있음" 전송
-- Primary: "나는 UUID:1-10000을 가지고 있음"
-- Primary가 자동으로 9501-10000을 전송
```

### 2. Failover 시나리오 — Position vs GTID 비교

```
시나리오: Primary 갑작스런 장애
  Primary: GTID uuid-P:1-10000 (uuid-P = Primary UUID)
  Replica A: uuid-P:1-10000 적용 (완전 동기화)
  Replica B: uuid-P:1-9500 적용 (500개 뒤처짐)
  Replica C: uuid-P:1-9800 적용 (200개 뒤처짐)

---새 Primary 선택: Replica A (가장 최신)---

Position 기반 (수동, 복잡):
  Replica A의 Binary Log 확인 필요
  Replica B: uuid-P:9501부터 필요
    → Replica A의 Binary Log에서 9501 위치 찾기 (어떤 파일?)
    → Replica A의 새 Binary Log는 Position 1부터 시작
    → 이전 Primary의 Binary Log 구조를 새 Primary에서 재현 불가
    → 수동으로 각 Replica의 gtid 차이를 계산하고 mysqlbinlog로 덤프...
    → 전문 DBA가 수십 분 작업

GTID 기반 (자동):
  Replica A가 새 Primary로 승격
  Replica A: RESET REPLICA; (또는 STOP REPLICA)
  Replica A: "나는 uuid-P:1-10000을 가지고 있음"

  Replica B 재연결:
    CHANGE REPLICATION SOURCE TO
        SOURCE_HOST = 'replica-a-host',
        SOURCE_AUTO_POSITION = 1;
    -- Replica B: "나는 uuid-P:1-9500만 있음"
    -- Replica A: "9501-10000 보내줄게"
    → 자동으로 필요한 이벤트만 전송!

  Replica C 재연결:
    동일하게 SOURCE_AUTO_POSITION = 1
    "나는 9800까지" → "9801-10000 보내줄게" → 자동!

소요 시간:
  Position 기반: 30분 ~ 2시간 (수동 작업)
  GTID 기반: 1~5분 (MHA/Orchestrator 자동화 포함)
```

### 3. MHA vs Orchestrator Failover 비교

```
MHA (Master High Availability Manager):
  방식: 에이전트 기반 (각 MySQL 서버에 에이전트 설치)
  감지: MHA Manager가 Primary 주기적 ping
  Failover 과정:
    ① Primary 감지 실패 → 각 Replica에서 SSH로 Binary Log 복사
    ② 가장 최신 Replica 선택 (Position 비교)
    ③ Missing transaction 다른 Replica에서 채우기
    ④ 선택된 Replica를 새 Primary로 승격
    ⑤ 다른 Replica들 재설정 (CHANGE REPLICATION SOURCE)
    ⑥ VIP 전환 (가상 IP → 새 Primary로)
  소요 시간: 30초 ~ 2분
  단점: 단일 장애점 (MHA Manager), SSH 의존성

Orchestrator:
  방식: HTTP API 기반 (중앙 서버에서 관리)
  기능: 토폴로지 시각화 + 자동 Failover + 수동 조작
  Failover 과정 (GTID 사용 시):
    ① 가상 IP 또는 DNS TTL로 트래픽 차단
    ② 가장 최신 Replica 선택 (GTID 집합 비교)
    ③ Replica들 재설정: SOURCE_AUTO_POSITION = 1
    ④ 새 Primary로 트래픽 라우팅 (VIP/DNS 전환)
  특징: 
    GTID와 완벽 통합
    UI 대시보드로 토폴로지 확인 가능
    수동 토폴로지 변경 (Failover, Relocate)

MySQL InnoDB Cluster (MySQL 공식):
  방식: Group Replication 기반 동기 복제
  Failover: 자동 (쿼럼 기반, 다수결 투표)
  특징: 
    최고 수준의 HA
    쓰기 성능 약간 저하 (동기 복제)
    Primary 선출 완전 자동
```

### 4. 새 Primary 선택 기준

```sql
-- 어느 Replica를 새 Primary로 선택하는가?

-- 기준 1: 가장 최신 데이터를 가진 Replica
-- Executed_Gtid_Set이 가장 큰 것 (가장 많은 GTID 적용)
SELECT @@global.gtid_executed;  -- 각 Replica에서 실행

-- Relay Log에 있지만 아직 적용 안 된 것까지 포함 원하면:
-- STOP REPLICA SQL_THREAD; (안전하게 현재 적용 완료)

-- 기준 2: 데이터 위치뿐 아니라 서버 상태
-- 하드웨어 사양, 네트워크 위치, 부하 상태
-- Orchestrator: 가중치(weight) 설정으로 선호 서버 지정 가능

-- 기준 3: 원래 Primary와 같은 데이터센터
-- 네트워크 Lag 최소화를 위해 같은 리전 Replica 선호

-- Orchestrator 설정 예시:
-- {
--   "RecoveryPeriodBlockSeconds": 3600,
--   "PromotionIgnoreHostnameFilters": ["^reporting-.*"],
--   "DetachLostReplicasAfterMasterFailover": true
-- }

-- 승격 후 처리:
-- 1. 새 Primary에서 READ ONLY 해제:
SET GLOBAL READ_ONLY = OFF;
SET GLOBAL SUPER_READ_ONLY = OFF;

-- 2. 이전 Primary의 행방 확인 (Split Brain 방지):
-- 이전 Primary가 실제로 죽었는지 확인 (STONITH - Shoot The Other Node In The Head)
-- 불확실하면 두 Primary가 동시에 쓰기 허용 → 데이터 불일치!
```

### 5. GTID를 이용한 에러 복구

```sql
-- 복제 에러 발생 시 해당 GTID 건너뛰기 (GTID 방식):

-- 에러 확인:
SHOW REPLICA STATUS\G
-- Last_SQL_Errno: 1062 (Duplicate entry)
-- Gtid_Next: 3E11FA47:5001  ← 문제가 된 GTID

-- 건너뛰기:
STOP REPLICA SQL_THREAD;
SET @@SESSION.GTID_NEXT = '3E11FA47-71CA-11E1-9E33-C80AA9429562:5001';
BEGIN;
COMMIT;  -- 빈 트랜잭션으로 이 GTID를 "처리됨"으로 표시
SET @@SESSION.GTID_NEXT = 'AUTOMATIC';
START REPLICA SQL_THREAD;

-- 확인:
SHOW REPLICA STATUS\G
-- Last_SQL_Errno: 0 (에러 없음)
-- Executed_Gtid_Set: uuid:1-5000,uuid:5001,uuid:5002-...

-- 주의: 에러를 건너뛰면 해당 트랜잭션이 Replica에 미반영됨
-- → 데이터 불일치 가능
-- → 반드시 원인 파악 후 수동으로 데이터 보정 필요
-- 예: Duplicate entry = Replica에 이미 해당 Row가 있음
--    → Primary와 Replica의 해당 Row 직접 비교 후 판단
```

---

## 💻 실전 실험

### 실험 1: GTID 상태 확인

```sql
-- Primary에서:
SELECT @@global.gtid_mode;           -- ON
SELECT @@global.server_uuid;         -- UUID 확인
SELECT @@global.gtid_executed;       -- 실행된 GTID 집합
-- 예: 3E11FA47-71CA-11E1-9E33-C80AA9429562:1-12345

-- 트랜잭션 실행 후 GTID 확인:
INSERT INTO test_gtid VALUES (1, NOW());
SELECT @@global.gtid_executed;
-- 예: 3E11FA47-71CA-11E1-9E33-C80AA9429562:1-12346  ← 12346 추가됨

-- Replica에서:
SHOW REPLICA STATUS\G
-- Retrieved_Gtid_Set: 수신한 GTID
-- Executed_Gtid_Set: 적용한 GTID
-- 차이 = 아직 미적용 GTID (Lag)
```

### 실험 2: GTID 기반 Replica 재설정 (Failover 시뮬레이션)

```sql
-- 현재 Primary에서 Replica 연결 해제 후 새 Primary 지정:

-- Replica에서:
STOP REPLICA;
RESET REPLICA ALL;  -- 기존 복제 설정 삭제

-- 새 Primary로 재연결 (GTID):
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = 'new-primary-host',
    SOURCE_PORT = 3306,
    SOURCE_USER = 'repl_user',
    SOURCE_PASSWORD = 'password',
    SOURCE_AUTO_POSITION = 1;  -- GTID 자동 위치!

START REPLICA;
SHOW REPLICA STATUS\G  -- 정상 복제 확인
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes
-- Seconds_Behind_Source: 0~N (따라잡는 중)
```

---

## 📊 성능 비교

```
Position vs GTID 기반 Failover:

              Position          GTID
설정 복잡도   높음              낮음 (AUTO_POSITION=1)
Failover 시간 30분~2시간(수동) 1~5분(자동)
오류 가능성   높음              낮음
Multi-Source  복잡              지원
에러 복구     SK_COUNTER        GTID_NEXT로 정밀 제어
롤링 업그레이드 복잡            용이 (GTID 집합으로 추적)

MHA vs Orchestrator:
              MHA               Orchestrator
구조          에이전트 기반     HTTP API 중앙
SSH 의존      있음              없음
UI            없음              있음 (웹 UI)
GTID 지원     지원              완벽 통합
Multi-Master  없음              지원
활발한 개발   감소 중           활발
MySQL 버전    5.6+ 지원         8.0 권장

InnoDB Cluster:
  가장 완전한 HA (자동 선출, 동기 복제)
  쓰기 지연 약간 증가
  3노드 이상 필요 (쿼럼)
```

---

## ⚖️ 트레이드오프

```
GTID 사용:
  ✅ Failover 간소화 (AUTO_POSITION)
  ✅ 복제 상태 추적 용이 (GTID 집합 비교)
  ✅ Multi-Source 복제 지원
  ✅ 에러 복구 정밀 제어 (GTID_NEXT)
  ❌ gtid_mode 변경은 재시작 필요 (온라인 변경 가능은 MySQL 5.7.6+)
  ❌ GTID와 비호환 구문 불가:
     CREATE TABLE ... SELECT ... (제한적)
     트랜잭션 내 non-transactional 테이블 수정

세미-동기 복제 (Failover 시 데이터 손실 최소화):
  ✅ 장애 시 최소 1개 Replica가 데이터 보유 보장
  ❌ Primary 응답 시간 증가 (네트워크 RTT만큼)
  ❌ 모든 Replica 응답 불가 시 자동으로 비동기 전환
  → rpl_semi_sync_master_timeout 이후 비동기 전환

InnoDB Cluster vs 전통 복제:
  InnoDB Cluster:
    ✅ 완전 자동 Failover (<30초)
    ✅ 동기 복제 → 데이터 손실 없음
    ✅ MySQL Router가 자동 라우팅
    ❌ 쓰기 성능 10~30% 저하 (동기 Ack 대기)
    ❌ 3노드 이상 필요 (과반수 필요)
    ❌ 지리적으로 분산된 환경에서 지연 증가
  
  전통 복제 + Orchestrator:
    ✅ 기존 인프라 활용 (설치 용이)
    ✅ 쓰기 성능 최대
    ❌ Failover 수십 초 ~ 수 분
    ❌ 비동기 복제 → 데이터 손실 가능
```

---

## 📌 핵심 정리

```
GTID 핵심:

GTID 형식:
  source_uuid:transaction_id
  server_uuid는 /var/lib/mysql/auto.cnf에 영구 저장

주요 변수:
  gtid_executed: 이 서버에서 실행된 모든 GTID
  gtid_purged: Binary Log 삭제로 더 이상 없는 GTID
  Retrieved_Gtid_Set: IO Thread 수신 GTID
  Executed_Gtid_Set: SQL Thread 적용 GTID

GTID 복제 설정:
  SOURCE_AUTO_POSITION = 1
  → Replica가 자신의 gtid_executed를 Primary에 전송
  → Primary가 자동으로 필요한 GTID만 전송

Failover:
  Position 기반: 수동, 복잡, 오류 가능성 높음
  GTID 기반: AUTO_POSITION=1로 자동 재연결
  도구: Orchestrator (권장), MHA, InnoDB Cluster

새 Primary 선택:
  가장 큰 Executed_Gtid_Set을 가진 Replica
  같은 데이터센터 선호
  SUPER_READ_ONLY = OFF로 승격
```

---

## 🤔 생각해볼 문제

**Q1.** Primary와 Replica 모두 장애 후 복구됐는데, Replica가 Primary보다 더 많은 GTID를 가지고 있다. 이 상황은 어떻게 발생했고 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

**발생 원인**:
`sync_binlog=0` 상태에서 Primary 크래시 시 발생 가능합니다.

1. Primary: 트랜잭션 T1000 실행 → Binary Log에 write()(OS 버퍼) → Replica에 전송
2. Replica: T1000 수신 및 적용 완료
3. Primary: OS 크래시 → OS 버퍼의 Binary Log(T1000) 손실
4. Primary 재시작: Redo Log에 T1000이 있으면 복구 (DB에는 있음), 하지만 Binary Log에는 없음
   - 또는 innodb_flush_log_at_trx_commit=2 + OS 크래시 → T1000 자체가 DB에도 없음

**처리 방법**:

**시나리오 A: Replica가 Primary보다 앞서지만 Primary DB에 데이터 있음**
```sql
-- Primary에서 GTID 수동 추가:
SET @@GLOBAL.GTID_PURGED = 'uuid:1-1000';  -- Replica의 범위와 일치
-- → "이 GTID들은 이미 처리됨"으로 표시
-- 이후 Replica가 재연결하면 정상 복제 시작
```

**시나리오 B: Primary DB에도 T1000 데이터 없음 (가장 심각)**
```sql
-- Replica를 새 Primary로 승격하는 것이 안전
-- 이전 Primary의 T1000 이후 데이터는 이미 손실됨
-- Replica가 더 최신이므로 Replica가 정확한 상태
```

**예방책**: `sync_binlog=1` + `innodb_flush_log_at_trx_commit=1` (완전 내구성 설정) 사용으로 Binary Log 손실 방지.

</details>

---

**Q2.** GTID가 `enforce_gtid_consistency=ON`인 환경에서 `CREATE TABLE new_table AS SELECT * FROM old_table`이 실패하는 이유와 대안을 설명하라.

<details>
<summary>해설 보기</summary>

**실패 이유**:

`CREATE TABLE ... SELECT ...`는 DDL과 DML이 동시에 발생하는 혼합 문장입니다. GTID 모드에서 이것은 하나의 GTID로 표현할 수 없습니다:
- DDL(CREATE TABLE): Auto-commit으로 즉시 커밋
- SELECT의 결과 INSERT: 별도 트랜잭션

MySQL은 `enforce_gtid_consistency=ON`에서 이런 "non-transactional" 혼합을 허용하지 않습니다.

```sql
-- 에러:
CREATE TABLE new_table AS SELECT * FROM old_table;
-- ERROR 1786 (HY000): Statement violates GTID consistency:
-- CREATE TABLE ... SELECT.
```

**대안**:

```sql
-- 방법 1: DDL과 DML 분리
CREATE TABLE new_table LIKE old_table;
INSERT INTO new_table SELECT * FROM old_table;

-- 방법 2: CREATE TABLE 후 INSERT SELECT
CREATE TABLE new_table (
    id BIGINT,
    name VARCHAR(100)
    -- old_table의 구조와 동일하게
);
INSERT INTO new_table SELECT id, name FROM old_table;

-- 방법 3: 임시로 enforce_gtid_consistency 완화 (비권장)
-- SET GLOBAL enforce_gtid_consistency = WARN;  -- 위반 시 경고만
-- 사용 후 반드시 ON으로 복원
```

GTID 비호환 구문 목록:
- `CREATE TABLE ... SELECT`
- `CREATE TEMPORARY TABLE` (트랜잭션 내)
- `INSERT INTO ... SELECT` from non-transactional to transactional table (특정 조건)

이것이 GTID 환경에서 `pt-online-schema-change`나 `gh-ost` 같은 도구가 분리된 DDL + DML 방식으로 동작하는 이유이기도 합니다.

</details>

---

**Q3.** 3개 Replica (A, B, C)가 있는 환경에서 Primary 장애 발생 시 Orchestrator가 어떤 순서로 Failover를 처리하고, "Split Brain"을 방지하는 방법은 무엇인가?

<details>
<summary>해설 보기</summary>

**Orchestrator Failover 순서**:

1. **장애 감지**: Orchestrator가 Primary에 ping 실패 → 여러 번 재시도 (false positive 방지)

2. **후보 선택**: `Executed_Gtid_Set`이 가장 큰 Replica 선택 (A = uuid:1-10000, B = uuid:1-9500, C = uuid:1-9800 → A 선택)

3. **빠진 이벤트 전달**: A가 9501-10000을 B에게, 9801-10000을 C에게 전달 (GTID Auto Position으로 자동)

4. **새 Primary 승격**: Replica A를 Primary로 설정
   ```sql
   -- A에서:
   STOP REPLICA;
   RESET REPLICA ALL;
   SET GLOBAL READ_ONLY = OFF;
   SET GLOBAL SUPER_READ_ONLY = OFF;
   ```

5. **B, C 재연결**: 새 Primary A로 SOURCE_AUTO_POSITION=1으로 재연결

6. **트래픽 전환**: VIP 또는 DNS TTL 변경으로 앱이 새 Primary A에 연결

**Split Brain 방지**:

Split Brain = 이전 Primary가 완전히 죽지 않고 쓰기를 계속 허용할 때 두 Primary가 동시 존재하는 상태.

방지 방법:
1. **STONITH (Shoot The Other Node In The Head)**: 이전 Primary를 강제 종료하는 스크립트
   ```bash
   # Orchestrator hook:
   fencing_script.sh old-primary-host  # ssh/IPMI로 강제 재시작
   ```

2. **SUPER_READ_ONLY 설정**: 정상 운영에서 항상 Replica에 `SUPER_READ_ONLY=ON`
   → Failover 시 명시적으로 해제할 때까지 이전 Primary도 자동으로 Read-Only 상태 유지

3. **VIP (Virtual IP)**: 이전 Primary는 VIP를 잃어버려 앱이 연결 못 함
   ```bash
   # Keepalived/Pacemaker로 VIP를 새 Primary로 이동
   # 이전 Primary는 더 이상 VIP 없음 → 앱 연결 불가
   ```

4. **ProxySQL 중앙 라우팅**: 모든 트래픽이 ProxySQL을 통해 → Orchestrator가 ProxySQL 업데이트 → 이전 Primary로의 트래픽 즉시 차단

</details>

---

<div align="center">

**[⬅️ Read Replica 정합성](./02-read-replica-consistency.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring R/W 분리 ➡️](./04-spring-read-write-separation.md)**

</div>
