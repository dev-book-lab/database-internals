# Gap Lock과 Next-Key Lock — Phantom Read 방지 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Gap Lock이 잠그는 "공간"의 경계는 인덱스 데이터에 따라 어떻게 결정되는가?
- Next-Key Lock의 정확한 범위를 인덱스 값 목록으로부터 계산하는 방법은?
- 존재하지 않는 값에 FOR UPDATE를 하면 어떤 Gap이 잠기는가?
- Gap Lock끼리는 공존 가능하지만 Insert Intention Lock과는 충돌하는 이유는?
- Supremum pseudo-record에 Gap Lock이 걸리는 시나리오는?
- READ_COMMITTED에서 Gap Lock이 비활성화되면 Statement-based Replication이 왜 위험해지는가?

---

## 🔍 왜 이 개념이 중요한가

### "Deadlock이 나는데 DELETE/INSERT가 충돌할 리 없는데?" — Gap Lock Deadlock

```
실제 케이스:
  쿠폰 발급 서비스
  동시에 여러 사용자가 쿠폰 신청 → INSERT INTO coupon_claims ...
  
  이상하게도 Deadlock 발생:
    ERROR 1213: Deadlock found when trying to get lock

  원인 파악:
    SHOW ENGINE INNODB STATUS\G
    LATEST DETECTED DEADLOCK:
    TRX A: HOLDS ... X,GAP LOCK on idx_user (100, 200)
           WAITING ... X,INSERT_INTENTION at position 150
    TRX B: HOLDS ... X,GAP LOCK on idx_user (100, 200)
           WAITING ... X,INSERT_INTENTION at position 180

  두 트랜잭션이 같은 Gap을 잠그고 있고
  서로 그 Gap에 INSERT를 시도 → 순환 대기 → Deadlock

  근본 원인:
    INSERT 전에 SELECT ... FOR UPDATE로 중복 체크를 했고
    그 FOR UPDATE가 Gap Lock을 걸었음
    두 트랜잭션이 같은 Gap Lock 보유 → INSERT Intention Lock 충돌

Gap Lock을 이해해야:
  이 Deadlock이 왜 발생했는지 설명 가능
  Gap Lock 없애는 방향 (READ_COMMITTED) vs 유지하는 방향 (RR) 선택 가능
  쿼리/인덱스 설계로 Gap Lock 범위를 최소화하는 방법 찾기 가능
```

---

## 😱 잘못된 이해

### Before: "FOR UPDATE는 해당 Row만 잠근다"

```sql
-- 인덱스: id [1, 5, 10, 20, 30]

-- 잘못된 예상:
SELECT * FROM t WHERE id = 15 FOR UPDATE;
-- id=15가 없음
-- 기대: "해당하는 Row가 없으니 Lock도 없다"
-- 실제: (10, 20) 사이의 Gap에 Lock이 걸린다!

-- 왜 이것이 문제인가:
-- 다른 트랜잭션이 id=15 INSERT 시도 → 차단!
-- "없는 값에 Lock이 왜 걸리지?" 이해 못 하면 디버깅 불가

-- 또 다른 잘못된 믿음:
-- "Gap Lock은 INSERT를 막지만 UPDATE는 괜찮다"
-- → Gap은 원래 없는 공간이므로 UPDATE 대상이 없음
-- → Gap Lock은 오직 INSERT를 차단 (UPDATE는 기존 Record에만 가능)

-- "READ_COMMITTED로 바꾸면 Gap Lock만 없어진다"
-- → Gap Lock + Next-Key Lock 둘 다 없어짐
-- → Record Lock만 남음
-- → Phantom Read 가능해짐 (이것이 트레이드오프)
```

---

## ✨ 올바른 이해

### After: Gap Lock = 인덱스 공간의 "경계 없는 빈 땅"에 대한 Lock

```
인덱스를 수직선으로 생각:

  -∞                                                          +∞
   |  1  |  5  |  10  |  20  |  30  |
   
   Gap:
   (-∞, 1)   → Infimum Gap
   (1, 5)    → 1과 5 사이
   (5, 10)   → 5와 10 사이
   (10, 20)  → 10과 20 사이
   (20, 30)  → 20과 30 사이
   (30, +∞)  → Supremum Gap

Record Lock:
  특정 값(예: id=10)에 걸리는 Lock
  해당 Row의 수정/삭제 차단

Gap Lock:
  두 Record 사이의 공간에 걸리는 Lock
  해당 공간에 새 Row INSERT 차단
  
Next-Key Lock = Record Lock + 해당 Record 이전 Gap Lock:
  id=20에 대한 Next-Key Lock:
  = Record Lock(id=20) + Gap Lock(10, 20)
  = "id=20 Row 보호 + id=10~20 사이에 INSERT 차단"

이것이 Phantom Read를 막는 원리:
  WHERE id BETWEEN 10 AND 20 FOR UPDATE 실행 시
  id=10, 20에 Record Lock + (5,10), (10,20), (20,30) Gap Lock
  → 이 범위 내 새 INSERT 차단 → 다음에 같은 WHERE 실행 시 같은 결과
```

---

## 🔬 내부 동작 원리

### 1. Next-Key Lock 범위 계산 공식

```
인덱스 데이터: amount = [500, 1000, 2000, 5000, 10000]

SELECT * FROM t WHERE amount = 2000 FOR UPDATE:
  Record Lock: amount=2000
  Gap Lock (이전 Gap): (1000, 2000)
  → Next-Key Lock = (1000, 2000]

SELECT * FROM t WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE:
  대상 Records: 1000, 2000, 5000

  Next-Key Lock on 1000: (500, 1000]
    = Gap(500, 1000) + Record(1000)
  Next-Key Lock on 2000: (1000, 2000]
    = Gap(1000, 2000) + Record(2000)
  Next-Key Lock on 5000: (2000, 5000]
    = Gap(2000, 5000) + Record(5000)
  
  추가 Gap Lock (범위 끝 이후):
  Gap Lock on (5000, 10000)
    = amount=5000 다음 Gap도 보호 (범위 끝 이후의 첫 Gap)

  총 보호 범위: (500, 10000)
  → 이 범위 내 새 INSERT 차단

존재하지 않는 값:
  SELECT * FROM t WHERE amount = 3000 FOR UPDATE (3000 없음):
    Record Lock: 없음 (해당 레코드 없음)
    Gap Lock: (2000, 5000) = 3000이 삽입될 수 있는 공간
    → amount=2500, 3000, 4000 INSERT 차단

Supremum Lock (범위 초과):
  SELECT * FROM t WHERE amount > 5000 FOR UPDATE:
    Records: 10000에 Lock
    Gap Lock: (5000, 10000) + (10000, supremum)
    supremum pseudo-record: 인덱스의 논리적 최댓값
    → 미래에 추가될 amount=99999 INSERT도 차단

Infimum Lock (최솟값 미만):
  SELECT * FROM t WHERE amount < 500 FOR UPDATE:
    Records: 없음 (500 미만 데이터 없음)
    Gap Lock: (infimum, 500)
    → amount=100, 200 INSERT 차단
```

### 2. Gap Lock 공존과 Insert Intention Lock의 충돌

```
Gap Lock 동시 보유:

TRX A: SELECT * FROM t WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE;
→ Gap Lock(500,1000), (1000,2000), (2000,5000), (5000,10000) 보유

TRX B: SELECT * FROM t WHERE amount BETWEEN 2000 AND 8000 FOR UPDATE;
→ Gap Lock(1000,2000), (2000,5000), (5000,10000) 요청

결과: Gap Lock끼리는 공존 ✅
→ TRX A와 TRX B 모두 같은 Gap에 Gap Lock 보유 가능
→ Gap Lock의 목적은 "삽입 차단"이지 서로를 차단하는 게 아님

Insert Intention Lock:
  INSERT 실행 직전에 획득하는 특수 Lock
  "나는 이 Gap의 특정 위치에 삽입하려 한다"는 신호
  
  특성:
    같은 Gap 내 다른 위치에 삽입: 공존 ✅
    TRX A: INSERT amount=1500 → Insert Intention Lock on (1000,2000) at 1500
    TRX B: INSERT amount=1700 → Insert Intention Lock on (1000,2000) at 1700
    → 1500 ≠ 1700 → 공존 가능 (동시 삽입 허용)

    Gap Lock vs Insert Intention Lock: 충돌 ❌
    TRX A의 Gap Lock(1000,2000) 보유
    TRX C: INSERT amount=1500 → Insert Intention Lock 획득 시도
    → TRX A의 Gap Lock과 충돌 → 차단!

Gap Lock Deadlock 메커니즘:
  TRX A: Gap Lock(1000,2000) 보유 → INSERT amount=1500 시도 (Insert Intention Lock 요청)
  TRX B: Gap Lock(1000,2000) 보유 → INSERT amount=1700 시도 (Insert Intention Lock 요청)

  TRX A의 Insert Intention → TRX B의 Gap Lock이 차단
  TRX B의 Insert Intention → TRX A의 Gap Lock이 차단
  → 순환 대기 → Deadlock!
```

### 3. FOR UPDATE에서 Gap Lock 범위가 결정되는 세부 규칙

```
PK (Clustered Index) 등치 조건:
  WHERE id = 10 FOR UPDATE (id=10 존재):
    X,REC_NOT_GAP Lock → Record Lock만, Gap Lock 없음
    이유: PK 등치는 고유값 보장 → Gap Lock 불필요 (새 Row 삽입 불가)

  WHERE id = 15 FOR UPDATE (id=15 없음):
    X,GAP Lock on (10, 20) → Gap Lock만 (Record 없으니)

PK 범위 조건:
  WHERE id BETWEEN 5 AND 10 FOR UPDATE:
    Next-Key Lock on 5: (이전 Gap, 5]
    Next-Key Lock on 10: (5, 10]
    Gap Lock on (10, 다음 값)

Non-unique Secondary Index:
  WHERE user_id = 1 FOR UPDATE (user_id=1인 Row 3개):
    각 user_id=1 Entry에 X Lock
    각 Entry의 이전 Gap에 Gap Lock
    마지막 Entry의 이후 Gap에도 Gap Lock
    이유: Non-unique → 같은 값이 더 추가될 수 있음 → Gap 보호 필요

Unique Secondary Index:
  WHERE email = 'a@b.com' FOR UPDATE (unique):
    X,REC_NOT_GAP → PK와 동일하게 Record Lock만
    이유: Unique 보장 → 같은 email이 추가될 수 없음 → Gap 불필요

범위와 격리 수준:
  REPEATABLE READ: 위 규칙대로 Gap Lock 적용
  READ COMMITTED: 모든 Gap Lock 비활성
    → 오직 X,REC_NOT_GAP (존재하는 Row의 Record Lock)만
```

### 4. READ_COMMITTED에서 Gap Lock 비활성화의 전체 영향

```sql
-- Gap Lock 없음 (READ_COMMITTED):
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

START TRANSACTION;
SELECT * FROM t WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE;
-- Record Lock만: amount=1000, 2000, 5000 Row에만 X Lock
-- Gap Lock 없음!

-- 이제 다른 트랜잭션이 범위 내 INSERT 가능:
-- INSERT INTO t (amount) VALUES (3000);  → 즉시 성공!

-- 두 번째 같은 SELECT FOR UPDATE → 3000도 포함!
SELECT * FROM t WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE;
-- Phantom Read 발생!

ROLLBACK;

-- Statement-based Replication에서의 위험:
-- Primary: TRX A, TRX B가 순서 X로 실행
-- Replica: 같은 Statement를 순서 Y로 실행 가능
-- Gap Lock 없으면 서로 다른 삽입 위치 → Primary와 Replica의 최종 상태 다름
-- → 복제 불일치
-- 해결: READ_COMMITTED 사용 시 반드시 Row-based Replication (binlog_format=ROW)

-- Gap Lock Deadlock 감소 효과:
-- RC에서는 INSERT Intention Lock vs Gap Lock 충돌 없음
-- → Deadlock 빈도 크게 감소 (INSERT 관련 Deadlock 거의 사라짐)
```

---

## 💻 실전 실험

### 실험 1: Gap Lock 범위 직접 확인

```sql
CREATE TABLE gap_test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    val INT NOT NULL,
    INDEX idx_val (val)
) ENGINE=InnoDB;

INSERT INTO gap_test (val) VALUES (10), (20), (30), (50), (80);

-- 세션 1: 존재하는 값 FOR UPDATE
START TRANSACTION;
SELECT * FROM gap_test WHERE val = 30 FOR UPDATE;

-- Lock 상태 확인
SELECT LOCK_TYPE, LOCK_MODE, INDEX_NAME, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'gap_test'
  AND ENGINE_TRANSACTION_ID = (
    SELECT trx_id FROM information_schema.INNODB_TRX
    WHERE trx_mysql_thread_id = CONNECTION_ID()
  )
ORDER BY INDEX_NAME, LOCK_DATA;
-- IX (Table Intention Lock)
-- X,REC_NOT_GAP on PRIMARY id=3 (val=30의 PK)
-- X on idx_val: val=30 (Next-Key Lock: (20,30])
--              val=50 이후 Gap도? (범위 끝 Gap: (30, 50))

-- 세션 2: Gap 내부 INSERT 시도
INSERT INTO gap_test (val) VALUES (25);  -- (20,30) Gap → 차단!
INSERT INTO gap_test (val) VALUES (35);  -- (30,50) Gap → 차단! (범위 끝 Gap)
INSERT INTO gap_test (val) VALUES (55);  -- (50,80) Gap → 통과!

ROLLBACK;  -- 세션 1

-- 존재하지 않는 값:
START TRANSACTION;
SELECT * FROM gap_test WHERE val = 40 FOR UPDATE;  -- val=40 없음

SELECT LOCK_TYPE, LOCK_MODE, INDEX_NAME, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'gap_test';
-- X,GAP on idx_val: val=50 이전 Gap = (30, 50)
-- Record Lock 없음!

INSERT INTO gap_test (val) VALUES (40);  -- 차단! (30,50) Gap Lock
INSERT INTO gap_test (val) VALUES (45);  -- 차단! 같은 Gap
INSERT INTO gap_test (val) VALUES (55);  -- 통과! (50, 80) Gap

ROLLBACK;
DROP TABLE gap_test;
```

### 실험 2: Gap Lock Deadlock 재현

```sql
CREATE TABLE coupon_claims (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    coupon_id BIGINT NOT NULL,
    INDEX idx_user (user_id)
) ENGINE=InnoDB;

INSERT INTO coupon_claims (user_id, coupon_id) VALUES (100, 1), (200, 1), (300, 1);

-- 세션 1: 중복 체크 후 INSERT 패턴
START TRANSACTION;
SELECT * FROM coupon_claims WHERE user_id = 150 FOR UPDATE;
-- user_id=150 없음 → Gap Lock(100, 200) 획득

-- 세션 2: 동일한 패턴 (다른 사용자)
START TRANSACTION;
SELECT * FROM coupon_claims WHERE user_id = 170 FOR UPDATE;
-- user_id=170 없음 → Gap Lock(100, 200) 획득 (공존!)

-- 세션 1: INSERT 시도
INSERT INTO coupon_claims (user_id, coupon_id) VALUES (150, 1);
-- Insert Intention Lock on (100,200) at 150
-- 세션 2의 Gap Lock(100,200)과 충돌 → 차단!

-- 세션 2: INSERT 시도
INSERT INTO coupon_claims (user_id, coupon_id) VALUES (170, 1);
-- Insert Intention Lock on (100,200) at 170
-- 세션 1의 Gap Lock(100,200)과 충돌 → 차단!
-- → Deadlock! ERROR 1213

SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 섹션 확인

DROP TABLE coupon_claims;
```

### 실험 3: Supremum Gap Lock 확인

```sql
CREATE TABLE sup_test (id INT PRIMARY KEY);
INSERT INTO sup_test VALUES (1), (3), (5);

-- 세션 1: 최대값 초과 조건
START TRANSACTION;
SELECT * FROM sup_test WHERE id > 3 FOR UPDATE;

SELECT LOCK_TYPE, LOCK_MODE, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'sup_test';
-- X,REC_NOT_GAP on id=5 (or Next-Key Lock)
-- X,GAP on supremum pseudo-record → (5, +∞) 차단

-- 세션 2: id=6, 7, 100 INSERT 시도
INSERT INTO sup_test VALUES (6);    -- 차단! supremum Gap 내
INSERT INTO sup_test VALUES (100);  -- 차단! 역시 supremum Gap 내
INSERT INTO sup_test VALUES (2);    -- 통과! (1,3) Gap은 잠기지 않음

ROLLBACK;
DROP TABLE sup_test;
```

### 실험 4: READ_COMMITTED에서 Gap Lock 없음 확인

```sql
CREATE TABLE rc_test (id INT PRIMARY KEY, val INT, INDEX idx_val(val));
INSERT INTO rc_test VALUES (1,10),(2,20),(3,30),(4,40),(5,50);

-- REPEATABLE READ (기본)
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT * FROM rc_test WHERE val BETWEEN 20 AND 40 FOR UPDATE;

SELECT COUNT(*) AS lock_count, LOCK_MODE
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'rc_test' AND LOCK_TYPE = 'RECORD'
GROUP BY LOCK_MODE;
-- X: Next-Key Locks (Record + Gap 포함)
-- X,REC_NOT_GAP: Record only

ROLLBACK;

-- READ COMMITTED
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM rc_test WHERE val BETWEEN 20 AND 40 FOR UPDATE;

SELECT COUNT(*) AS lock_count, LOCK_MODE
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'rc_test' AND LOCK_TYPE = 'RECORD'
GROUP BY LOCK_MODE;
-- X,REC_NOT_GAP만 있음: Record Lock만, Gap Lock 없음!

-- 범위 내 INSERT 자유롭게 가능:
-- INSERT INTO rc_test VALUES (6, 25);  -- 즉시 성공!

ROLLBACK;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
DROP TABLE rc_test;
```

---

## 📊 성능 비교

```
Gap Lock 있음(RR) vs 없음(RC) 처리량:

동시 INSERT 워크로드 (같은 인덱스 범위에 100개 트랜잭션 동시 삽입):
  REPEATABLE READ:
    Gap Lock 충돌로 대부분 직렬화 또는 Deadlock
    TPS: 낮음, Deadlock 재시도로 추가 오버헤드
  
  READ COMMITTED:
    Gap Lock 없음 → 다른 위치 삽입은 동시 실행 가능
    TPS: 높음 (삽입 처리량 최대화)

SELECT FOR UPDATE 후 INSERT 패턴 (중복 체크):
  REPEATABLE READ:
    Gap Lock → Deadlock 빈번
    → 실제 서비스에서 쿠폰 발급, 예약 시스템에서 자주 발생
  
  READ COMMITTED:
    Deadlock 없음, Phantom Read 가능
    → 중복 체크를 UNIQUE KEY + INSERT IGNORE로 대체 권장

Phantom Read 안전성:
  RR + Consistent Read: 완전 방지
  RR + Current Read: Gap Lock으로 방지
  RC: 방지 안 됨

Replication 안전성:
  RR: Statement-based, Row-based 모두 OK
  RC: Row-based만 OK (binlog_format=ROW 필수)
```

---

## ⚖️ 트레이드오프

```
REPEATABLE READ (Gap Lock 있음):
  ✅ Phantom Read 방지 (Consistent Read + Current Read 모두)
  ✅ Statement-based Replication과 호환
  ❌ Gap Lock Deadlock 위험 (INSERT 집약적 시스템)
  ❌ 동시 삽입 처리량 감소

READ COMMITTED (Gap Lock 없음):
  ✅ Gap Lock Deadlock 없음 → INSERT 집약 시스템에서 높은 처리량
  ✅ Semi-consistent Read → 인덱스 없는 UPDATE 피해 감소
  ❌ Phantom Read 가능
  ❌ Row-based Replication 필수
  ❌ 복잡한 비즈니스 로직에서 Phantom 관련 버그 위험

Gap Lock Deadlock 회피 전략 (RR 유지하면서):
  1. UNIQUE KEY 활용:
     unique index 존재 → Gap Lock 대신 X,REC_NOT_GAP → Gap Lock 없음
     INSERT ... ON DUPLICATE KEY UPDATE (Unique 위반 처리)
  
  2. PK 등치 조건:
     WHERE id = 특정값 FOR UPDATE → X,REC_NOT_GAP → Gap Lock 없음
  
  3. 트랜잭션 분리:
     SELECT FOR UPDATE와 INSERT를 같은 Gap에서 동시에 하지 않는 설계

  4. READ COMMITTED 부분 적용:
     특정 서비스만 @Transactional(isolation=READ_COMMITTED)
     전역 변경보다 안전
```

---

## 📌 핵심 정리

```
Gap Lock / Next-Key Lock 핵심:

Gap Lock:
  인덱스 레코드 사이의 빈 공간을 잠그는 Lock
  목적: 새 Row 삽입 차단 → Phantom Read 방지
  범위: 인접한 두 인덱스 값 사이 (또는 -∞~첫값, 마지막값~+∞)

Next-Key Lock:
  = Record Lock + 해당 Record 이전 Gap Lock
  InnoDB REPEATABLE READ의 기본 Lock 단위
  범위 공식: (이전 인덱스 값, 현재 인덱스 값]

Lock 결정 규칙:
  PK/Unique 등치 존재: X,REC_NOT_GAP (Gap Lock 없음)
  PK/Unique 등치 없음: X,GAP (Gap만)
  Non-unique, 범위 조건: Next-Key Lock (Record + Gap)
  Supremum/Infimum: 범위 끝 이후/이전 Gap도 보호

공존 규칙:
  Gap Lock + Gap Lock: ✅ 공존
  Gap Lock + Insert Intention Lock: ❌ 충돌
  → 같은 Gap에 두 트랜잭션이 Gap Lock + INSERT 시도 → Deadlock

READ_COMMITTED:
  Gap Lock / Next-Key Lock 모두 비활성
  Record Lock (X,REC_NOT_GAP)만 사용
  Phantom Read 가능, Deadlock 대폭 감소
  Row-based Replication 필수
```

---

## 🤔 생각해볼 문제

**Q1.** 인덱스 데이터 `amount = [100, 500, 1000, 2000]`에서 다음 쿼리의 Next-Key Lock 범위를 정확히 계산하라.

```sql
SELECT * FROM orders WHERE amount IN (500, 1500) FOR UPDATE;
```

<details>
<summary>해설 보기</summary>

`IN` 조건의 각 값을 독립적으로 처리합니다.

**amount = 500 (존재)**:
- idx_amount Secondary Index의 amount=500 Entry에 X Lock (또는 Next-Key Lock)
- Non-unique index라면: Next-Key Lock (100, 500] = Gap(100, 500) + Record(500)

**amount = 1500 (존재하지 않음)**:
- amount=1500은 없음 → Record Lock 없음
- 1500이 위치할 Gap: (1000, 2000)
- Gap Lock on (1000, 2000)

**추가 Gap Lock (범위 끝)**:
- 각 Next-Key Lock에 해당하는 이후 Gap:
  - amount=500 이후의 Gap: (500, 1000)도 잠길 수 있음 (구현에 따라)

**차단되는 INSERT**:
- amount=100~499: (100,500) Gap Lock → 차단
- amount=500: Record Lock → 차단
- amount=501~999: (500, 1000) Gap → 잠길 수 있음
- amount=1001~1999: (1000, 2000) Gap Lock → 차단
- amount=1500: Gap Lock → 차단

**차단되지 않는 INSERT**:
- amount=50: (infimum, 100) → 이 Gap은 잠기지 않음
- amount=2001~: (2000, +∞) → 이 Gap도 잠기지 않음

실제로 확인하려면: `SELECT LOCK_MODE, LOCK_DATA FROM performance_schema.data_locks`

</details>

---

**Q2.** 다음 패턴에서 Deadlock을 회피하는 방법을 두 가지 제시하라.

```sql
-- 두 트랜잭션이 동시에 실행:
-- TRX A:
START TRANSACTION;
SELECT * FROM reservations WHERE seat_id = 50 FOR UPDATE;  -- Gap Lock
INSERT INTO reservations (seat_id, user_id) VALUES (50, 100);

-- TRX B:
START TRANSACTION;
SELECT * FROM reservations WHERE seat_id = 50 FOR UPDATE;  -- Gap Lock (공존)
INSERT INTO reservations (seat_id, user_id) VALUES (50, 200);
-- → Deadlock (양쪽이 서로의 Gap Lock 때문에 Insert Intention 차단)
```

<details>
<summary>해설 보기</summary>

**방법 1: UNIQUE KEY 활용**

```sql
-- seat_id에 UNIQUE INDEX 추가 (좌석은 하나만 예약 가능하다는 비즈니스 규칙)
ALTER TABLE reservations ADD UNIQUE KEY uk_seat (seat_id);

-- 이제 SELECT FOR UPDATE 시:
-- seat_id=50이 없으면: Gap Lock → X,GAP (변화 없음, 여전히 위험)
-- seat_id=50이 있으면: X,REC_NOT_GAP → Gap Lock 없음

-- 더 나은 접근: INSERT ... ON DUPLICATE KEY UPDATE
INSERT INTO reservations (seat_id, user_id) VALUES (50, 100)
ON DUPLICATE KEY UPDATE user_id = VALUES(user_id);
-- Gap Lock 없이 직접 삽입, 중복 시 자동 처리
```

**방법 2: READ_COMMITTED 사용**

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Gap Lock이 완전히 비활성화됨
-- SELECT FOR UPDATE → X,REC_NOT_GAP (존재하는 Record만 Lock)
-- seat_id=50 없으면 Lock 자체 없음 → INSERT Intention 충돌 없음
-- → Deadlock 불가능

-- 단, Phantom Read 허용됨:
-- TRX A와 TRX B가 모두 "seat_id=50 없음"을 확인 후 INSERT
-- 두 INSERT 중 하나가 성공, 나머지는 UNIQUE KEY 위반 (uk_seat 있으면)
-- → INSERT IGNORE나 ON DUPLICATE KEY로 처리
```

두 방법의 선택:
- UNIQUE KEY + INSERT IGNORE: 비즈니스 규칙(좌석 중복 예약 불가)을 DB가 보장, RR 유지 가능
- READ_COMMITTED: 더 넓은 적용 (UNIQUE가 불가능한 경우), Phantom 주의

</details>

---

**Q3.** `SELECT * FROM t WHERE id > 0 FOR UPDATE`를 실행했다. id 데이터가 `[1, 2, 3]`일 때 Supremum Lock이 걸리는 이유와 그것이 방지하는 Phantom Read 시나리오를 설명하라.

<details>
<summary>해설 보기</summary>

**Supremum Lock이 걸리는 이유**:

`id > 0`은 현재 모든 레코드(1, 2, 3)를 포함하고 상한이 없습니다. InnoDB는 이 범위의 끝을 표현하기 위해 B+Tree의 논리적 최댓값인 `supremum pseudo-record`를 사용합니다.

Next-Key Lock 계산:
- Record Lock + Gap: id=1, 2, 3에 각각 Next-Key Lock
- 마지막 레코드(id=3) 이후: **(3, supremum)에 Gap Lock** = Supremum Lock

**Supremum Lock이 방지하는 시나리오**:

```sql
-- TRX A:
SELECT * FROM t WHERE id > 0 FOR UPDATE;
-- (3, supremum) Gap Lock 보유

-- 동시에 TRX B:
INSERT INTO t VALUES (4, ...);
-- id=4는 supremum 이전 → (3, supremum) Gap 내부
-- TRX A의 Supremum Gap Lock과 충돌 → 차단!
-- → TRX A가 COMMIT할 때까지 id > 3인 값 삽입 불가

-- TRX A가 다시 같은 쿼리:
SELECT * FROM t WHERE id > 0 FOR UPDATE;
-- id=4 없음 (TRX B가 차단됨) → 이전과 동일한 결과 = Phantom Read 없음
```

Supremum Lock이 없었다면:
- TRX B의 id=4 INSERT가 성공
- TRX A의 두 번째 SELECT에서 id=4가 나타남 = Phantom Read 발생

결론: Supremum Lock = "현재 인덱스의 최대값 이상에 대한 삽입을 차단"하는 메커니즘으로, 범위의 상한이 없는 쿼리(`WHERE id > N`)에서 Phantom Read를 막기 위한 필수 요소입니다.

</details>

---

<div align="center">

**[⬅️ Row Lock](./02-row-lock-vs-table-lock.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데드락 감지 ➡️](./04-deadlock-detection.md)**

</div>
