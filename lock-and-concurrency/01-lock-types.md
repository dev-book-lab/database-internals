# 락의 종류 — Shared, Exclusive, Intention Lock

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- S Lock과 X Lock의 호환 행렬에서 어떤 조합이 차단을 일으키고 왜인가?
- IS/IX Intention Lock이 없으면 `LOCK TABLES`가 얼마나 비효율적인가?
- `performance_schema.data_locks`의 `LOCK_MODE` 값에서 `X,GAP`과 `X,REC_NOT_GAP`은 어떻게 다른가?
- AUTO-INC Lock이 `innodb_autoinc_lock_mode=0`에서 병목이 되는 이유는?
- S Lock을 보유한 두 트랜잭션이 X Lock을 요청하면 왜 데드락이 발생하는가?
- LOCK_STATUS = 'WAITING'인 트랜잭션을 찾아 차단 원인을 파악하는 방법은?

---

## 🔍 왜 이 개념이 중요한가

### Lock을 모르면 "이 쿼리가 왜 멈추지?"를 설명할 수 없다

```
실제 운영 이슈:
  오전 10시, 사용자들이 "주문이 안 돼요!" 신고
  DB 모니터링 보니 active connections 폭증, CPU는 낮음
  
  원인 파악:
    SELECT * FROM performance_schema.data_locks;
    → LOCK_STATUS = 'WAITING'인 트랜잭션 수백 개
    → 하나의 FOR UPDATE가 수백 개 트랜잭션을 차단 중
  
  더 나쁜 케이스:
    어드민이 "데이터 확인"으로 실행한 SELECT ... FOR UPDATE가
    수분째 열려있음 → 그동안 모든 주문 처리 블로킹
    
이것을 이해하려면:
  S Lock / X Lock이 무엇이고 어떻게 충돌하는지
  Intention Lock이 왜 있는지
  `data_locks`로 지금 Lock 상태를 어떻게 읽는지
  → 장애 발생 시 수초 내에 원인 파악 + 해결 가능
```

---

## 😱 잘못된 이해

### Before: Lock = 느린 것, 가능하면 피해야 하는 것

```
잘못된 믿음 1: "트랜잭션을 짧게 하면 Lock 문제가 없다"
  → Lock 보유 시간은 짧아지지만 Lock 자체의 작동 방식은 동일
  → FOR UPDATE를 사용하면 어떤 트랜잭션이든 차단 발생 가능

잘못된 믿음 2: "SELECT는 Lock을 걸지 않는다"
  → 일반 SELECT (Consistent Read): Lock 없음 ← 이건 맞음
  → SELECT ... FOR SHARE: S Lock
  → SELECT ... FOR UPDATE: X Lock
  → 구분하지 않으면 "왜 SELECT 이후 UPDATE가 블로킹이지?" 이해 불가

잘못된 믿음 3: "Row Lock이면 다른 Row는 영향 없다"
  → 인덱스 없는 WHERE 조건의 UPDATE/DELETE:
    → 전체 테이블 스캔 → 모든 Row에 Lock 시도
    → 사실상 Table Lock = 해당 테이블 완전 차단
  → 이것이 WHERE 조건 컬럼에 인덱스가 필수인 이유

잘못된 실험 (검색에서 흔히 보이는):
  "FOR UPDATE는 느리니까 SELECT 후 if 조건 확인하고 UPDATE"
  → Lost Update 발생 가능
  → FOR UPDATE의 목적이 "느린 것"이 아닌 "안전한 것"임을 모름
```

---

## ✨ 올바른 이해

### After: Lock = 동시성 제어 도구, 올바르게 사용하면 데이터 무결성 보장

```
Lock의 두 가지 역할:

역할 1 — 충돌 방지 (X Lock):
  "내가 이 Row를 수정하는 동안 다른 사람이 수정하지 못하게"
  → UPDATE, DELETE, SELECT FOR UPDATE
  → 동시 수정으로 인한 Lost Update, Dirty Write 방지

역할 2 — 읽기 일관성 보장 (S Lock):
  "내가 이 Row를 읽는 동안 다른 사람이 수정하지 못하게"
  → SELECT FOR SHARE
  → 부모 Row 삭제 방지 등 참조 무결성 보장

Lock 호환 행렬 (핵심):
  요청\기존   S Lock    X Lock
  S Lock      ✅ 공존   ❌ 차단
  X Lock      ❌ 차단   ❌ 차단

  → "S Lock끼리만 공존 가능"
  → X Lock은 항상 혼자 독점

Intention Lock (IS/IX) = 테이블 수준 Lock 의도 표시:
  Row에 Lock을 걸기 전에 테이블에 먼저 의도 표시
  → LOCK TABLES가 테이블 전체 Lock 필요 여부를 O(1)로 판단 가능
  (없으면 테이블의 모든 Row Lock 상태를 일일이 확인 필요)
```

---

## 🔬 내부 동작 원리

### 1. S Lock과 X Lock 상세

```
Shared Lock (S Lock, 공유 잠금):
  획득 SQL: SELECT ... FOR SHARE
            (구 문법: SELECT ... LOCK IN SHARE MODE)
  의미: "이 Row를 읽겠다. 다른 읽기는 허용, 쓰기는 차단"
  
  획득 조건:
    기존 S Lock: ✅ 공존 가능 (여러 트랜잭션 동시 읽기)
    기존 X Lock: ❌ 차단 (X Lock 해제까지 대기)
  
  실용 케이스:
    부모 Row 보호: users를 FOR SHARE → 다른 트랜잭션의 DELETE users 차단
    복잡한 조건 읽기: 여러 Row를 읽는 도중 변경 방지

Exclusive Lock (X Lock, 배타 잠금):
  획득 SQL: SELECT ... FOR UPDATE
            UPDATE ... WHERE ...
            DELETE ... WHERE ...
            INSERT (중복 체크 부분)
  의미: "이 Row를 수정하겠다. 다른 모든 Lock 차단"
  
  획득 조건:
    기존 S Lock: ❌ 차단 (S Lock 전부 해제될 때까지 대기)
    기존 X Lock: ❌ 차단 (기존 X Lock 해제까지 대기)
  
  해제 시점: COMMIT 또는 ROLLBACK (트랜잭션 종료)

호환 행렬 완전판:
         기존: IS    IX    S     X
  요청 IS      ✅    ✅    ✅    ❌
  요청 IX      ✅    ✅    ❌    ❌
  요청 S       ✅    ❌    ✅    ❌
  요청 X       ❌    ❌    ❌    ❌

  규칙:
    X는 모든 것과 충돌
    S는 S/IS와만 공존
    IS/IX는 IX/IS와 공존 (Row Level 충돌을 테이블에서 흡수)
```

### 2. Intention Lock — 테이블-레벨 효율성

```
왜 Intention Lock이 필요한가:

시나리오 없이 (Intention Lock 없는 세계):
  TRX A: Row id=1에 X Lock 보유 중
  TRX B: LOCK TABLES orders WRITE 요청
  → 테이블 전체 X Lock을 걸려면 모든 Row에 Lock이 없는지 확인 필요
  → orders에 100만 Row가 있다면 100만 번 확인 → 수초 소요

Intention Lock이 있는 세계:
  TRX A: Row id=1에 X Lock 획득 직전
    → 테이블 orders에 IX (Intention Exclusive) Lock 먼저 획득
  TRX B: LOCK TABLES orders WRITE 요청
    → 테이블에 IX Lock이 있음을 즉시 확인
    → "어딘가 Row X Lock이 있다" → 차단 결정
    → Row를 하나도 확인하지 않고 즉시 판단 가능!

Intention Lock 규칙:
  Row에 S Lock 걸기 전 → 테이블에 IS Lock 먼저
  Row에 X Lock 걸기 전 → 테이블에 IX Lock 먼저
  자동으로 InnoDB가 획득 (개발자가 명시적으로 쓰는 것이 아님)
  트랜잭션 종료 시 자동 해제

LOCK TABLES와의 상호작용:
  LOCK TABLES t READ  → 테이블에 S Lock 요청
    기존 IX Lock 있으면: S-IX 비호환 → 차단 (쓰기 트랜잭션 완료 대기)
  LOCK TABLES t WRITE → 테이블에 X Lock 요청
    기존 IS Lock 있으면: X-IS 비호환 → 차단 (읽기 트랜잭션 완료 대기)
    기존 IX Lock 있으면: X-IX 비호환 → 차단
```

### 3. S Lock 업그레이드 데드락 패턴

```
가장 흔한 데드락 유형 중 하나:

TRX A: SELECT * FROM accounts WHERE id=1 FOR SHARE;  → S Lock 보유
TRX B: SELECT * FROM accounts WHERE id=1 FOR SHARE;  → S Lock 보유 (공존)

TRX A: UPDATE accounts SET balance=... WHERE id=1;
  → X Lock 요청 시도
  → 현재 TRX B의 S Lock이 존재 → 차단 (TRX B의 S Lock 해제 대기)

TRX B: UPDATE accounts SET balance=... WHERE id=1;
  → X Lock 요청 시도
  → 현재 TRX A의 S Lock이 존재 → 차단 (TRX A의 S Lock 해제 대기)

결과:
  TRX A → TRX B의 S Lock 기다림
  TRX B → TRX A의 S Lock 기다림
  → 순환 대기 → Deadlock!

InnoDB 감지: Wait-for Graph에서 사이클 발견 → 비용 낮은 쪽 롤백

예방:
  처음부터 FOR UPDATE 사용 (S Lock 업그레이드 불필요)
  FOR SHARE → UPDATE 패턴 지양

Spring에서의 실수:
  @Lock(PESSIMISTIC_READ) 으로 읽고 나서
  변경 사항을 save()하는 패턴 → S Lock 업그레이드 데드락 위험
  → @Lock(PESSIMISTIC_WRITE) 처음부터 사용이 안전
```

### 4. AUTO-INC Lock과 innodb_autoinc_lock_mode

```
AUTO_INCREMENT PK INSERT 시 사용되는 특수 Lock:

테이블 레벨 AUTO-INC Lock (경량 뮤텍스 전 단계):
  목적: AUTO_INCREMENT 값의 연속성 보장
  방식: INSERT 완료까지 테이블 레벨 Lock 보유

innodb_autoinc_lock_mode 값별 동작:

값 0 (Traditional):
  모든 INSERT에 테이블 레벨 AUTO-INC Lock
  Lock 해제: 각 INSERT 완료 후
  → 한 번에 1개 INSERT만 AUTO_INCREMENT 값 획득
  → 동시 INSERT 처리량 = 단일 INSERT 처리량
  → Statement-based Replication에서 안전 (값 순서 보장)
  → 대용량 Bulk INSERT 병렬 실행 시 심각한 병목

값 1 (Consecutive — MySQL 5.7 기본):
  Simple INSERT (VALUES 개수 확정 가능):
    경량 뮤텍스 사용 → 값 발급 즉시 해제 → 높은 동시성
  Bulk INSERT (SELECT, LOAD DATA — 개수 불확정):
    테이블 레벨 AUTO-INC Lock → INSERT 완료까지 보유
  → 혼합 워크로드에서 균형

값 2 (Interleaved — MySQL 8.0 기본):
  모든 INSERT에 경량 뮤텍스 → 즉시 해제
  → 최고 동시성, AUTO_INCREMENT 값이 삽입 순서와 다를 수 있음
  → Statement-based Replication에서 비안전 (복제 서버와 값 다를 수 있음)
  → Row-based Replication 사용 권장 (MySQL 8.0 기본이 RBR이므로 OK)

실무 영향:
  대용량 병렬 Bulk INSERT:
    모드 0: 직렬화 → 매우 느림
    모드 2: 병렬 실행 → 빠름
  Replication 방식:
    Statement-based → 모드 0 또는 1 필요
    Row-based → 모드 2 사용 가능 (권장)
```

### 5. performance_schema.data_locks 완전 해석

```sql
SELECT
    ENGINE_LOCK_ID,          -- Lock 고유 ID
    ENGINE_TRANSACTION_ID,   -- Lock 보유 트랜잭션 ID
    OBJECT_SCHEMA AS db,     -- 데이터베이스명
    OBJECT_NAME AS tbl,      -- 테이블명
    INDEX_NAME,              -- 잠긴 인덱스명 (NULL이면 테이블 Lock)
    LOCK_TYPE,               -- TABLE (테이블 레벨) 또는 RECORD (Row 레벨)
    LOCK_MODE,               -- 잠금 모드 상세
    LOCK_STATUS,             -- GRANTED (보유중) 또는 WAITING (대기중)
    LOCK_DATA                -- 잠긴 레코드의 키 값
FROM performance_schema.data_locks
ORDER BY ENGINE_TRANSACTION_ID, LOCK_STATUS DESC;

LOCK_MODE 값 완전 해석:
  IS:             Intention Shared (테이블 레벨)
  IX:             Intention Exclusive (테이블 레벨)
  S:              Shared Lock (Record + Gap = Next-Key Lock)
  X:              Exclusive Lock (Record + Gap = Next-Key Lock)
  S,REC_NOT_GAP:  Shared Record Lock만 (Gap 없음, PK 등치 조건)
  X,REC_NOT_GAP:  Exclusive Record Lock만 (Gap 없음, PK 등치 조건)
  S,GAP:          Shared Gap Lock만 (해당 위치 존재 안 해도)
  X,GAP:          Exclusive Gap Lock만
  X,INSERT_INTENTION: Insert Intention Lock (INSERT 직전 짧게)
  AUTO_INC:       AUTO_INCREMENT 잠금

LOCK_DATA 해석:
  "1"             → id = 1인 Row (PK가 INT/BIGINT)
  "supremum pseudo-record" → 인덱스 최대값 이상의 Gap
  NULL            → 테이블 레벨 Lock (Row 특정 없음)

차단 관계 분석:
SELECT
    w.trx_id AS waiting_trx,
    w.trx_query AS waiting_query,
    b.trx_id AS blocking_trx,
    b.trx_query AS blocking_query,
    dl_w.LOCK_MODE AS waiting_mode,
    dl_b.LOCK_MODE AS blocking_mode,
    dl_w.OBJECT_NAME AS table_name
FROM performance_schema.data_lock_waits dlw
JOIN information_schema.INNODB_TRX w ON w.trx_id = dlw.REQUESTING_ENGINE_TRANSACTION_ID
JOIN information_schema.INNODB_TRX b ON b.trx_id = dlw.BLOCKING_ENGINE_TRANSACTION_ID
JOIN performance_schema.data_locks dl_w
    ON dl_w.ENGINE_LOCK_ID = dlw.REQUESTING_ENGINE_LOCK_ID
JOIN performance_schema.data_locks dl_b
    ON dl_b.ENGINE_LOCK_ID = dlw.BLOCKING_ENGINE_LOCK_ID;
```

---

## 💻 실전 실험

### 실험 1: S Lock과 X Lock 충돌 확인

```sql
-- 세션 1
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;  -- S Lock 획득
-- 성공

-- 세션 2 (별도 창)
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;  -- S Lock 요청
-- 성공! (S-S 공존)
COMMIT;

-- 세션 3 (별도 창)
START TRANSACTION;
UPDATE accounts SET balance = 5000 WHERE id = 1;  -- X Lock 요청
-- 차단! (세션 1의 S Lock이 있어서)

-- 실시간 Lock 상태 확인
SELECT OBJECT_NAME, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'accounts'
ORDER BY LOCK_STATUS DESC;
-- GRANTED: 세션 1의 IS, S Lock
-- GRANTED: 세션 2의 IS, S Lock (이미 COMMIT됨)
-- WAITING: 세션 3의 IX, X Lock

-- 세션 1 COMMIT → 세션 3의 X Lock 획득
COMMIT;
```

### 실험 2: IS/IX Lock 자동 획득 확인

```sql
-- IX Lock이 자동으로 테이블 레벨에 생성되는지 확인
START TRANSACTION;
UPDATE accounts SET balance = 9000 WHERE id = 1;

SELECT LOCK_TYPE, LOCK_MODE, INDEX_NAME, LOCK_DATA
FROM performance_schema.data_locks
WHERE ENGINE_TRANSACTION_ID = (
    SELECT trx_id FROM information_schema.INNODB_TRX
    WHERE trx_mysql_thread_id = CONNECTION_ID()
);
-- LOCK_TYPE=TABLE, LOCK_MODE=IX: 자동 획득된 Intention Exclusive Lock
-- LOCK_TYPE=RECORD, LOCK_MODE=X,REC_NOT_GAP: 실제 Row X Lock (PK 등치)

ROLLBACK;
```

### 실험 3: S Lock 업그레이드 데드락 재현

```sql
-- 세션 1
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;  -- S Lock 획득

-- 세션 2
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;  -- S Lock 획득 (공존)

-- 세션 1 (계속)
UPDATE accounts SET balance = 7000 WHERE id = 1;  -- X Lock 요청 → 세션 2 S Lock에 차단

-- 세션 2 (계속)
UPDATE accounts SET balance = 6000 WHERE id = 1;  -- X Lock 요청 → 세션 1 S Lock에 차단
-- → Deadlock!
-- ERROR 1213: Deadlock found when trying to get lock; try restarting transaction

SHOW ENGINE INNODB STATUS\G  -- LATEST DETECTED DEADLOCK 섹션 확인
```

### 실험 4: AUTO-INC Lock 동시성 비교

```sql
-- innodb_autoinc_lock_mode 확인
SHOW VARIABLES LIKE 'innodb_autoinc_lock_mode';
-- MySQL 8.0 기본: 2 (Interleaved)

-- 모드별 동시 INSERT 성능 비교
CREATE TABLE autoinc_test (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    data VARCHAR(100)
) ENGINE=InnoDB;

-- 모드 2 (기본): 고동시성 INSERT
SET GLOBAL innodb_autoinc_lock_mode = 2;
-- (여러 세션에서 동시에 INSERT 실행)

-- 모드 0: 직렬화 INSERT
SET GLOBAL innodb_autoinc_lock_mode = 0;
-- (동일한 여러 세션 INSERT → 처리량 감소 확인)

-- AUTO-INC Lock 상태 확인
SELECT LOCK_TYPE, LOCK_MODE, OBJECT_NAME
FROM performance_schema.data_locks
WHERE LOCK_MODE = 'AUTO_INC';
-- 모드 0/1 Bulk INSERT 중에만 확인 가능

SET GLOBAL innodb_autoinc_lock_mode = 2;  -- 원복
DROP TABLE autoinc_test;
```

---

## 📊 성능 비교

```
Lock 방식별 처리량 영향:

S Lock (FOR SHARE):
  읽기 처리량: S-S 공존 → N개의 읽기 트랜잭션 동시 실행 가능
  쓰기 처리량: S-X 충돌 → 읽기가 진행 중이면 쓰기 차단
  사용 적합: 부모 Row 참조 무결성 보장, 읽기 후 조건 검사

X Lock (FOR UPDATE):
  읽기 처리량: X-S 충돌 → 쓰기 진행 중이면 읽기(FOR SHARE) 차단
              일반 SELECT는 MVCC이므로 차단 없음!
  쓰기 처리량: X-X 충돌 → 동시 쓰기 불가 (직렬 실행)
  사용 적합: 값 읽고 수정, Lost Update 방지

Intention Lock 오버헤드:
  IS/IX 획득 비용: 경량 (메모리 작업)
  이점: LOCK TABLES 판단 O(1)
  결론: 무시할 수 있는 오버헤드 대비 큰 이점

AUTO-INC Lock 처리량:
  모드 0 vs 2 병렬 INSERT:
    모드 0: 직렬 실행 → TPS = 단일 INSERT TPS
    모드 2: 완전 병렬 → TPS ≈ (단일 INSERT TPS × 동시 세션 수)
    차이: 동시 세션 10개 기준 최대 10배 차이
```

---

## ⚖️ 트레이드오프

```
FOR SHARE vs FOR UPDATE 선택:

FOR SHARE (S Lock):
  ✅ 여러 트랜잭션이 동시에 같은 Row를 읽기 가능
  ✅ 다른 트랜잭션의 쓰기만 차단 (읽기는 허용)
  ❌ S Lock 업그레이드(→X Lock) 시 데드락 위험
  ❌ 수정 의도가 있을 때 쓰면 안 됨
  사용: 읽기만 하고 수정하지 않을 때, 참조 무결성 보호

FOR UPDATE (X Lock):
  ✅ 다른 모든 Lock과 충돌 → 완전 독점
  ✅ 처음부터 X Lock이라 S Lock 업그레이드 데드락 없음
  ❌ 읽기(FOR SHARE 포함) 트랜잭션도 일부 차단
  ❌ 여러 FOR UPDATE가 다른 순서로 실행 시 데드락 위험
  사용: 읽고 수정할 때, Lost Update 방지가 필요할 때

AUTO-INC 설정 선택:
  Statement-based Replication 사용 중: 모드 0 또는 1
  Row-based Replication 사용 중: 모드 2 (높은 동시성)
  MySQL 8.0 기본 (RBR): 모드 2 → 변경 불필요
  대용량 Bulk INSERT 최적화: 모드 2 필수

data_locks 테이블 쿼리 비용:
  LOCK 수가 많은 환경에서 data_locks 조회 자체가 무거울 수 있음
  → 장애 시 1번 진단용으로는 문제없음
  → 주기적 모니터링에는 경량 쿼리 사용
```

---

## 📌 핵심 정리

```
Lock 종류 핵심:

S Lock (FOR SHARE):
  S-S 공존 ✅, S-X 충돌 ❌
  수정 의도 없는 읽기 보호에만 사용
  → 수정 의도 있으면 처음부터 X Lock 사용

X Lock (FOR UPDATE, UPDATE, DELETE):
  모든 다른 Lock과 충돌
  읽고 수정하는 모든 패턴에 사용

IS/IX (Intention Lock):
  Row Lock 전에 테이블에 자동 획득
  LOCK TABLES 판단을 O(1)로 만들어줌
  개발자가 직접 사용하지 않음

AUTO-INC:
  innodb_autoinc_lock_mode = 2 (MySQL 8.0 기본)
  + Row-based Replication = 현대 권장 설정

진단:
  performance_schema.data_locks: GRANTED/WAITING Lock 상태
  data_lock_waits: 차단 관계 (blocking_trx → waiting_trx)
  LOCK_STATUS = WAITING: 지금 멈춰있는 트랜잭션 식별
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 시나리오에서 세션 B는 차단되는가? 차단된다면 어떤 Lock이 문제인가?

```sql
-- 세션 A
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- X Lock
-- COMMIT 전

-- 세션 B
SELECT balance FROM accounts WHERE id = 1;  -- 일반 SELECT
```

<details>
<summary>해설 보기</summary>

**세션 B는 차단되지 않습니다.** 일반 `SELECT`는 Consistent Read (MVCC)로 처리되므로 Lock을 전혀 사용하지 않습니다. 세션 A의 X Lock이 있어도 세션 B의 일반 SELECT는 Undo Log의 이전 버전을 읽어 즉시 결과를 반환합니다.

이것이 "읽기가 쓰기를 차단하지 않는다"는 MVCC의 핵심 특성입니다.

차단이 발생하는 경우:
- `SELECT balance FROM accounts WHERE id = 1 FOR SHARE` → S Lock 요청 → 세션 A의 X Lock과 충돌 → 차단
- `SELECT balance FROM accounts WHERE id = 1 FOR UPDATE` → X Lock 요청 → 세션 A의 X Lock과 충돌 → 차단
- `UPDATE accounts SET balance = ... WHERE id = 1` → X Lock 요청 → 차단

결론: Lock이 문제를 일으키는 것은 항상 **쓰기 또는 잠금 읽기(FOR UPDATE/SHARE)** 와의 충돌에서만 발생합니다. 일반 SELECT는 Lock 충돌과 완전히 무관합니다.

</details>

---

**Q2.** `LOCK TABLES orders WRITE`를 실행했을 때 현재 진행 중인 `SELECT ... FOR UPDATE` 트랜잭션이 있다면 어떻게 되는가? Intention Lock의 관점에서 설명하라.

<details>
<summary>해설 보기</summary>

`LOCK TABLES orders WRITE`는 테이블 레벨 X Lock을 요청합니다. `SELECT ... FOR UPDATE`를 실행 중인 트랜잭션은 이미 테이블에 **IX (Intention Exclusive) Lock**을 보유하고 있습니다.

호환 행렬에서: X(table) vs IX(table) = 비호환 → **차단**

`LOCK TABLES orders WRITE`는 테이블의 IX Lock이 해제될 때까지, 즉 `FOR UPDATE` 트랜잭션이 COMMIT/ROLLBACK할 때까지 대기합니다.

Intention Lock 없이 이것을 판단했다면:
- "orders 테이블의 모든 Row에 Lock이 있는가?"를 확인해야 함
- 100만 Row = 100만 번 확인 → 수초 이상 소요

Intention Lock 덕분에:
- 테이블 레벨 IX Lock 확인 1번 → 즉시 차단 여부 판단 가능
- O(Row 수) → O(1)

실무 주의사항: `LOCK TABLES`와 InnoDB 트랜잭션을 함께 사용하면 이런 충돌이 발생합니다. InnoDB 환경에서는 `LOCK TABLES` 대신 트랜잭션 내에서 `FOR UPDATE`로 원하는 Row만 잠그는 것이 더 세밀하고 효율적입니다.

</details>

---

**Q3.** `performance_schema.data_locks`에서 `LOCK_MODE = 'X,REC_NOT_GAP'`과 `LOCK_MODE = 'X'`가 발생하는 조건의 차이를 설명하고, 어떤 쿼리 패턴에서 각각 나타나는지 예를 들어라.

<details>
<summary>해설 보기</summary>

**`X,REC_NOT_GAP`** (Record Lock만, Gap 없음):
- PK 또는 Unique Index의 **등치 조건**에서 발생
- "이 레코드만 잠금, 앞뒤 공간은 자유"
- 예시:
  ```sql
  SELECT * FROM orders WHERE id = 42 FOR UPDATE;
  -- id=42가 존재: X,REC_NOT_GAP on id=42
  ```
- Gap Lock이 없으므로 id=41이나 id=43에 삽입 가능

**`X`** (Next-Key Lock = Record + Gap):
- Non-unique Index, 범위 조건, 존재하지 않는 값, 또는 REPEATABLE READ 전반에서 발생
- "이 레코드 + 이전 Gap을 모두 잠금"
- 예시:
  ```sql
  SELECT * FROM orders WHERE amount = 5000 FOR UPDATE;
  -- amount는 non-unique index
  -- X Lock on each amount=5000 record + Gap Lock on each preceding gap
  
  SELECT * FROM orders WHERE id BETWEEN 10 AND 20 FOR UPDATE;
  -- X (Next-Key Lock) on id=10, 11, ..., 20 + respective gaps
  ```

**`X,GAP`** (Gap Lock만):
- 존재하지 않는 레코드의 위치에 Gap Lock만 걸릴 때
- 예시:
  ```sql
  SELECT * FROM orders WHERE id = 15 FOR UPDATE;
  -- id=15가 없고, id=10, 20이 인접: X,GAP on (10, 20) gap
  ```

판별 기준: **해당 레코드가 실제로 존재하는가** + **조건이 등치/범위인가** + **인덱스 유일성** 세 가지를 함께 봐야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Row Lock vs Table Lock ➡️](./02-row-lock-vs-table-lock.md)**

</div>
