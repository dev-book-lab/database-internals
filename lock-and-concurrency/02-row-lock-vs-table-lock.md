# Row Lock vs Table Lock — InnoDB가 락을 거는 단위

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- "InnoDB는 인덱스 레코드에 Lock을 건다"는 말의 정확한 의미는?
- 인덱스 없는 컬럼으로 UPDATE를 실행하면 몇 개의 Row에 Lock이 걸리는가?
- Semi-consistent Read란 무엇이고, UPDATE에서 어떻게 동작하는가?
- `LOCK TABLES`와 InnoDB 자동 Row Lock은 어떻게 다르고 InnoDB 환경에서 왜 `LOCK TABLES`를 지양해야 하는가?
- `INNODB_TRX`의 `trx_rows_locked`가 실제보다 많이 나타날 수 있는 이유는?
- MDL(Metadata Lock)이 `ALTER TABLE`을 장시간 차단하는 시나리오는?

---

## 🔍 왜 이 개념이 중요한가

### 인덱스 없는 DML = 서비스 장애

```
실제 장애 재현:
  신규 기능 배포 직후 트래픽 급증
  "주문 처리가 안 돼요!" 신고 쏟아짐

  원인 파악:
    배포된 코드:
      UPDATE order_events SET processed = 1
      WHERE event_type = 'PAYMENT_COMPLETED';
    
    event_type에 인덱스 없음!
    → InnoDB가 전체 order_events를 PK 순서로 스캔
    → 스캔하는 모든 Row에 X Lock 시도
    → order_events의 100만 건 모두 Lock 걸림
    → 다른 모든 주문 이벤트 INSERT 차단!

  해결:
    1. 장기 UPDATE 쿼리 kill
    2. CREATE INDEX idx_event_type ON order_events(event_type)
    3. UPDATE 재실행 → 이제 해당 Row만 Lock

얼마나 자주 일어나는가:
  "이 컬럼에는 인덱스가 없지만 WHERE에 쓸 일이 거의 없어서"
  → 어드민 스크립트, 배치 처리, 긴급 수동 쿼리에서 발생
  → 결과는 매번 서비스 전체 차단
```

---

## 😱 잘못된 이해

### Before: "Row Lock이니까 다른 Row에는 영향 없다"

```sql
-- 잘못된 믿음:
-- "InnoDB는 Row Lock이라 UPDATE 시 해당 Row만 잠긴다"

-- 실제 상황:
-- orders 테이블, status 컬럼에 인덱스 없음

-- 어드민이 실행:
UPDATE orders SET memo = '긴급처리' WHERE status = 'PENDING';

-- 기대: status='PENDING'인 Row만 Lock
-- 실제: orders의 모든 Row에 X Lock!
--   → status='PAID'인 Row에도 Lock
--   → status='SHIPPED'인 Row에도 Lock
--   → 이 UPDATE가 완료될 때까지 모든 orders 변경 불가

-- 동시에 실행 중인 다른 트랜잭션들:
UPDATE orders SET status = 'SHIPPED' WHERE id = 12345;
-- id=12345는 status='PENDING'과 전혀 무관한 Row인데도 차단!
-- WHY? 인덱스 없음 → Full Scan → 이미 id=12345도 스캔하면서 Lock
```

```
더 잘못된 응용:
  "그러면 인덱스 있는 컬럼에 조건을 달면 괜찮겠지?"
  
  부분적으로 맞지만:
  SELECT * FROM orders WHERE user_id = 1 FOR UPDATE;
  -- user_id에 Secondary Index 있음
  -- Secondary Index → Clustered Index(PK) Double Lookup 과정에서
  -- Secondary Index Records에도 Lock
  -- Clustered Index Records에도 Lock
  -- 인덱스 설계와 쿼리 패턴 모두 중요
```

---

## ✨ 올바른 이해

### After: InnoDB Lock = 인덱스 레코드에 거는 Lock

```
핵심 원칙:
  InnoDB에서 Row Lock의 정확한 의미:
  "Row 자체"가 아닌 "Row를 가리키는 인덱스 레코드"에 Lock

왜 인덱스 레코드인가:
  InnoDB는 Clustered Index = 데이터 자체 (데이터가 B+Tree 형태)
  Row를 찾으려면 인덱스 탐색 필수
  → Lock도 인덱스 탐색 경로에 걸림

인덱스 유무에 따른 Lock 범위:
  인덱스 있는 조건 (WHERE user_id = 1, idx_user_id 있음):
    Secondary Index 탐색 → user_id=1인 Entry에 Lock
    Clustered Index(PK) 탐색 → 해당 PK에 Lock
    다른 user_id의 Row는 완전히 무관

  인덱스 없는 조건 (WHERE status = 'PAID', status 인덱스 없음):
    Clustered Index Full Scan
    → 스캔한 모든 PK Entry에 Lock 시도
    → 사실상 테이블 전체 Lock

REPEATABLE READ와 READ COMMITTED의 차이:
  REPEATABLE READ:
    스캔한 Row에 Lock이 모두 유지됨 (조건 불일치도)
  READ COMMITTED (Semi-consistent Read):
    조건 불일치 Row의 Lock을 즉시 해제
    → 인덱스 없어도 피해가 약간 줄어듦
    → 하지만 여전히 전체를 스캔하며 Lock을 걸었다 해제하는 과정
```

---

## 🔬 내부 동작 원리

### 1. InnoDB Lock이 인덱스 레코드에 걸리는 원리

```
B+Tree 구조에서의 Lock 위치:

테이블 구조:
  CREATE TABLE orders (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Clustered Index
    user_id BIGINT NOT NULL,
    status  VARCHAR(20),
    amount  DECIMAL(10,2),
    INDEX idx_user (user_id)                   -- Secondary Index
  );

SELECT * FROM orders WHERE user_id = 1 FOR UPDATE:

  탐색 경로:
  idx_user (Secondary Index B+Tree):
    Leaf: [user_id=1, PK=10] ← Lock 획득
          [user_id=1, PK=25] ← Lock 획득
          [user_id=1, PK=37] ← Lock 획득
          (Gap Lock도 각 Entry 사이에 획득)

  Clustered Index B+Tree:
    Leaf: [PK=10, user_id=1, status='PAID', amount=5000] ← Lock 획득
          [PK=25, user_id=1, status='PENDING', amount=3000] ← Lock 획득
          [PK=37, user_id=1, status='PAID', amount=8000] ← Lock 획득

  결과:
    Secondary Index의 3개 Entry에 Lock
    Clustered Index의 3개 Entry에 Lock
    총 6개 인덱스 레코드에 Lock

Lock이 걸린 후:
  다른 트랜잭션이 user_id=2의 Row 수정 → 완전히 자유 (다른 인덱스 Entry)
  다른 트랜잭션이 user_id=1의 Row 수정 → Lock 충돌 → 차단
```

### 2. 인덱스 없는 DML의 Lock 범위 — Semi-consistent Read

```
REPEATABLE READ에서 인덱스 없는 UPDATE:

UPDATE orders SET memo = 'test' WHERE status = 'PAID';
(status 인덱스 없음)

탐색 경로:
  Clustered Index Full Scan (PK=1부터 PK=1,000,000까지)
  
  PK=1: X Lock 획득 → status 확인 → 'PENDING' (조건 불일치)
    REPEATABLE READ: Lock 유지! (조건 불일치여도)
  PK=2: X Lock 획득 → status 확인 → 'PAID' (조건 일치) → 수정
  PK=3: X Lock 획득 → status 확인 → 'CANCELLED' (불일치) → Lock 유지
  ...
  PK=1,000,000까지 반복
  
  결과: 100만 건 모두 X Lock → 사실상 Table Lock

READ COMMITTED에서의 Semi-consistent Read:
  PK=1: X Lock 획득 → status 확인 → 'PENDING' (불일치)
    → 즉시 Lock 해제! ← Semi-consistent Read의 핵심
  PK=2: X Lock 획득 → status 확인 → 'PAID' (일치) → 수정, Lock 유지
  PK=3: X Lock 획득 → status 확인 → 'CANCELLED' (불일치)
    → 즉시 Lock 해제!
  ...
  
  결과: 조건 일치 Row만 Lock 유지 (조건 불일치는 해제)
  
  그러나:
    여전히 전체를 스캔 = 느린 쿼리
    스캔 중 일시적으로 Lock → 다른 트랜잭션이 일시적 차단 경험
    → 인덱스 추가가 근본 해결책

Semi-consistent Read가 기본값(REPEATABLE READ)에서 비활성인 이유:
  스냅샷 일관성 요구사항: Lock 해제 = 다른 트랜잭션이 그 Row 수정 가능
  = 같은 트랜잭션 내에서 같은 Row의 상태가 바뀔 수 있음
  = REPEATABLE READ 보장 깨짐
```

### 3. LOCK TABLES vs InnoDB Row Lock 비교

```sql
-- LOCK TABLES: MySQL 서버 레벨 테이블 잠금
LOCK TABLES orders WRITE;
-- 효과:
--   다른 모든 세션의 orders 접근 차단 (SELECT 포함!)
--   InnoDB Row Lock과 무관 (서버 레벨)
--   트랜잭션 외부에서 작동
--   UNLOCK TABLES까지 유지 (트랜잭션 커밋과 무관)

UNLOCK TABLES;

-- 문제점:
-- ① 세밀도 없음: 테이블 전체 또는 없음 (Row 수준 제어 불가)
-- ② InnoDB 트랜잭션과 충돌:
--    LOCK TABLES 실행 시 기존 InnoDB 트랜잭션 암묵적 COMMIT
--    → 트랜잭션 원자성 보장 깨짐
-- ③ 다른 세션의 SELECT도 차단 → MVCC의 장점을 버림

-- InnoDB Row Lock (권장):
START TRANSACTION;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- 효과:
--   id=1 Row만 잠금 (다른 Row는 자유)
--   다른 세션의 일반 SELECT는 MVCC로 차단 없음
--   COMMIT/ROLLBACK 시 자동 해제
--   트랜잭션과 완전히 통합
COMMIT;

-- LOCK TABLES가 필요한 경우:
-- MyISAM 테이블 (InnoDB가 아닌 경우)
-- mysqldump --single-transaction 없이 일관된 스냅샷 필요 시
-- InnoDB 환경에서는 사실상 불필요
```

### 4. MDL(Metadata Lock)과 DDL 차단

```
MDL(Metadata Lock)이란:
  테이블 구조(스키마)에 대한 Lock
  DDL(ALTER TABLE 등)과 DML/쿼리 간의 충돌 방지

MDL 종류:
  Shared MDL (S-MDL):
    일반 쿼리 실행 시 자동 획득
    다른 Shared MDL과 공존
    Exclusive MDL(DDL)과 충돌

  Exclusive MDL (X-MDL):
    ALTER TABLE, DROP TABLE, CREATE INDEX 등 DDL 시 획득
    다른 모든 MDL과 충돌

장기 트랜잭션이 ALTER TABLE을 차단하는 시나리오:
  T=0: TRX A 시작 (Shared MDL 획득)
  T=1: ALTER TABLE orders ADD COLUMN note TEXT; (X-MDL 대기)
  T=1~10분: ALTER TABLE 대기 중
             → 동안 새로운 SELECT도 차단!
             (X-MDL이 대기 중이면 새로운 Shared MDL도 대기 = MDL Queue)
  T=10분: TRX A COMMIT → ALTER TABLE 진행 → 완료
          → 그동안 쌓인 SELECT도 처리

진단:
  SHOW PROCESSLIST;
  -- State: Waiting for table metadata lock

  SELECT * FROM performance_schema.metadata_locks
  WHERE OBJECT_NAME = 'orders';

  SELECT * FROM performance_schema.threads
  JOIN performance_schema.events_statements_current USING(thread_id)
  WHERE processlist_state = 'Waiting for table metadata lock';

  -- 원인 트랜잭션 찾기:
  SELECT trx_id, trx_started, trx_state, LEFT(trx_query, 100)
  FROM information_schema.INNODB_TRX
  ORDER BY trx_started;
  -- 가장 오래 실행 중인 트랜잭션이 MDL을 보유 중일 가능성
```

### 5. 트랜잭션별 Lock 보유 현황

```sql
-- 전체 활성 트랜잭션과 Lock 상태
SELECT
    t.trx_id,
    t.trx_started,
    TIMESTAMPDIFF(SECOND, t.trx_started, NOW()) AS seconds_running,
    t.trx_rows_locked,       -- 보유 중인 Row Lock 수 (예상값)
    t.trx_rows_modified,     -- 수정한 Row 수
    t.trx_tables_locked,     -- Lock 걸린 테이블 수
    t.trx_state,             -- RUNNING, LOCK WAIT, ROLLING BACK
    LEFT(t.trx_query, 100) AS current_query
FROM information_schema.INNODB_TRX t
ORDER BY t.trx_started;

-- trx_rows_locked 주의사항:
-- 실제 Lock 수보다 높게 나올 수 있음
-- Gap Lock, Next-Key Lock은 별도 카운트
-- 정확한 Lock 수는 data_locks 조회 필요

-- 특정 트랜잭션의 Lock 상세
SELECT
    dl.LOCK_TYPE,
    dl.LOCK_MODE,
    dl.INDEX_NAME,
    dl.LOCK_DATA
FROM performance_schema.data_locks dl
WHERE dl.ENGINE_TRANSACTION_ID = '12345';  -- 트랜잭션 ID 지정

-- 인덱스 없는 UPDATE 감지 (data_locks에서 많은 Row Lock):
SELECT
    ENGINE_TRANSACTION_ID,
    OBJECT_NAME AS table_name,
    COUNT(*) AS lock_count
FROM performance_schema.data_locks
WHERE LOCK_TYPE = 'RECORD'
GROUP BY ENGINE_TRANSACTION_ID, OBJECT_NAME
ORDER BY lock_count DESC
LIMIT 10;
-- lock_count가 테이블 Row 수에 가까우면 인덱스 없는 DML!
```

---

## 💻 실전 실험

### 실험 1: 인덱스 유무에 따른 Lock 범위 비교

```sql
-- 테스트 환경
CREATE TABLE lock_range_test (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    status  VARCHAR(20) NOT NULL,
    amount  DECIMAL(10,2)
    -- status 인덱스 없음 (의도적)
) ENGINE=InnoDB;

INSERT INTO lock_range_test (status, amount)
SELECT ELT(FLOOR(RAND()*3)+1, 'PAID', 'PENDING', 'CANCELLED'),
       ROUND(RAND()*10000, 2)
FROM information_schema.columns a
CROSS JOIN information_schema.columns b
LIMIT 10000;

-- 세션 1: 인덱스 없는 조건으로 UPDATE
START TRANSACTION;
UPDATE lock_range_test SET amount = amount * 1.1 WHERE status = 'PAID';

-- Lock 수 확인
SELECT COUNT(*) AS lock_count
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'lock_range_test'
  AND LOCK_TYPE = 'RECORD';
-- 예상: 10,000에 가까운 수 (전체 Row에 Lock)

-- 세션 2: 다른 status Row 수정 시도
UPDATE lock_range_test SET amount = 1 WHERE status = 'CANCELLED' LIMIT 1;
-- 차단! (인덱스 없으므로 세션 1이 이 Row도 Lock 보유 중)

ROLLBACK;  -- 세션 1

-- 인덱스 추가 후 재실험
CREATE INDEX idx_status ON lock_range_test(status);

START TRANSACTION;
UPDATE lock_range_test SET amount = amount * 1.1 WHERE status = 'PAID';

SELECT COUNT(*) AS lock_count_with_index
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'lock_range_test'
  AND LOCK_TYPE = 'RECORD';
-- 예상: status='PAID'인 건수만 (전체의 ~1/3)

ROLLBACK;
DROP TABLE lock_range_test;
```

### 실험 2: LOCK TABLES와 InnoDB 트랜잭션 충돌

```sql
-- LOCK TABLES가 트랜잭션을 암묵적으로 COMMIT하는 것 확인
CREATE TABLE lt_test (id INT PRIMARY KEY, val INT) ENGINE=InnoDB;
INSERT INTO lt_test VALUES (1, 10);

START TRANSACTION;
UPDATE lt_test SET val = 20 WHERE id = 1;
-- 아직 COMMIT 안 함

LOCK TABLES lt_test WRITE;
-- ← 이 시점에 암묵적 COMMIT 발생!
-- val = 20이 COMMIT됨

ROLLBACK;  -- 이미 COMMIT됐으므로 효과 없음

SELECT val FROM lt_test WHERE id = 1;  -- 20 (COMMIT됨)

UNLOCK TABLES;
DROP TABLE lt_test;
```

### 실험 3: MDL 차단 시뮬레이션

```sql
-- 세션 1: 장기 트랜잭션 시작
START TRANSACTION;
SELECT COUNT(*) FROM orders;  -- Shared MDL 획득
-- COMMIT 안 함

-- 세션 2: ALTER TABLE 시도
ALTER TABLE orders ADD COLUMN test_col INT DEFAULT 0;
-- State: Waiting for table metadata lock → 차단!

-- 세션 3: 새 SELECT도 차단 (MDL Queue)
SELECT * FROM orders LIMIT 1;
-- State: Waiting for table metadata lock → 연쇄 차단!

-- MDL 상태 확인
SELECT * FROM performance_schema.metadata_locks
WHERE OBJECT_TYPE = 'TABLE' AND OBJECT_NAME = 'orders'\G

SHOW PROCESSLIST;
-- 세션 2, 3이 모두 "Waiting for table metadata lock"

-- 세션 1 COMMIT → 세션 2 ALTER 진행 → 세션 3 SELECT 진행
COMMIT;
```

### 실험 4: trx_rows_locked vs 실제 Lock 수 비교

```sql
START TRANSACTION;
UPDATE orders SET memo = 'test' WHERE user_id IN (1, 2, 3);
-- user_id에 인덱스 있다고 가정

-- INNODB_TRX에서 확인
SELECT trx_id, trx_rows_locked, trx_rows_modified
FROM information_schema.INNODB_TRX
WHERE trx_mysql_thread_id = CONNECTION_ID();

-- data_locks에서 실제 Lock 수
SELECT COUNT(*) AS actual_lock_count
FROM performance_schema.data_locks
WHERE ENGINE_TRANSACTION_ID = (
    SELECT trx_id FROM information_schema.INNODB_TRX
    WHERE trx_mysql_thread_id = CONNECTION_ID()
);
-- trx_rows_locked ≠ actual_lock_count 가능 (Gap Lock 포함 여부 등)

ROLLBACK;
```

---

## 📊 성능 비교

```
인덱스 유무에 따른 UPDATE Lock 범위:

테이블: 100만 건, status 컬럼 (PAID=33%, PENDING=33%, CANCELLED=33%)

인덱스 없는 UPDATE WHERE status='PAID':
  Lock 건수: ~100만 건 (전체)
  Lock 유지 시간: UPDATE 완료까지 (~30초 for 33만 건 수정)
  영향: 다른 모든 UPDATE/INSERT/FOR UPDATE 차단
  → 사실상 Table Lock

인덱스 있는 UPDATE WHERE status='PAID':
  Lock 건수: ~33만 건 (해당 status만)
  Lock 유지 시간: UPDATE 완료까지 (~10초)
  영향: status='PENDING', 'CANCELLED' Row는 자유롭게 수정 가능
  → 진정한 Row-level Lock

READ COMMITTED + Semi-consistent Read:
  인덱스 없어도 조건 불일치 Row의 Lock을 해제
  최대 Lock 수 = 스캔 중 순간적으로 전체, 완료 후 = 일치 Row만
  장점: Lock 충돌 기간이 REPEATABLE READ보다 짧음
  단점: 여전히 전체 스캔 → 쿼리 자체가 느림

쿼리 성능:
  인덱스 없음: Full Scan → O(전체 Row 수)
  인덱스 있음: Index Scan → O(결과 Row 수 × log n)
```

---

## ⚖️ 트레이드오프

```
Row Lock vs Table Lock의 선택:

InnoDB Row Lock (자동):
  ✅ 필요한 Row만 잠금 → 높은 동시성
  ✅ MVCC와 통합 → 읽기-쓰기 비차단
  ✅ 트랜잭션과 자동 통합 (COMMIT/ROLLBACK 시 해제)
  ❌ 잘못된 쿼리(인덱스 없는 DML)로 전체 차단 위험
  ❌ Gap Lock, Next-Key Lock → Deadlock 가능성

LOCK TABLES:
  ✅ 명시적 테이블 잠금 → 예측 가능한 동작
  ✅ MyISAM에서는 유일한 선택
  ❌ 테이블 단위 잠금 → 낮은 동시성
  ❌ InnoDB 트랜잭션과 충돌 (암묵적 COMMIT)
  ❌ 사용 중 다른 세션의 SELECT도 차단

DDL과 MDL 고려:
  ALTER TABLE이 필요한 경우:
    장기 트랜잭션이 없는 시간대(새벽)에 실행
    또는 pt-online-schema-change, gh-ost 같은 도구 사용
    (내부적으로 점진적 변경 → MDL 보유 최소화)

인덱스 없는 DML 방지:
  코드 리뷰에서 WHERE 조건 컬럼 인덱스 확인
  slow_query_log + log_queries_not_using_indexes = ON 설정
  EXPLAIN으로 type: ALL 감지 시 경고
```

---

## 📌 핵심 정리

```
Row Lock vs Table Lock 핵심:

InnoDB Lock = 인덱스 레코드 Lock:
  인덱스 있는 조건: 해당 인덱스 Entry에만 Lock → 진정한 Row Lock
  인덱스 없는 조건: Clustered Index Full Scan → 모든 Row Lock = Table Lock

Semi-consistent Read (READ COMMITTED):
  조건 불일치 Row는 Lock 즉시 해제
  → 인덱스 없어도 REPEATABLE READ보다 피해 작음
  → 하지만 인덱스 추가가 근본 해결책

LOCK TABLES:
  InnoDB 환경에서 지양
  트랜잭션과 충돌, 낮은 동시성
  특수 목적(mysqldump, MyISAM)에만 사용

MDL (Metadata Lock):
  DDL은 X-MDL 필요 → 진행 중인 트랜잭션 완료 대기
  장기 트랜잭션 → DDL 차단 → 새 쿼리도 연쇄 차단
  진단: SHOW PROCESSLIST에서 "Waiting for table metadata lock"

핵심 교훈:
  WHERE 조건 컬럼에 인덱스 없음 = 테이블 전체 Lock 위험
  UPDATE/DELETE 전 반드시 EXPLAIN으로 type: ALL 아닌지 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 쿼리를 REPEATABLE READ에서 실행할 때와 READ COMMITTED에서 실행할 때 Lock 동작의 차이를 설명하라. `email` 컬럼에는 인덱스가 없다.

```sql
UPDATE users SET last_login = NOW() WHERE email = 'user@example.com';
```

<details>
<summary>해설 보기</summary>

**REPEATABLE READ**:
- `email` 인덱스 없음 → Clustered Index Full Scan
- 스캔하는 모든 Row에 X Lock 획득 시도
- 조건 불일치(email이 다른) Row에도 X Lock 유지 (Semi-consistent Read 비활성)
- UPDATE 완료까지 `users` 테이블의 모든 Row가 사실상 Lock됨
- 동시에 다른 사용자 정보를 UPDATE하려는 트랜잭션 모두 차단

**READ COMMITTED** (Semi-consistent Read 활성):
- 동일하게 Full Scan 수행
- email 불일치 Row의 X Lock을 즉시 해제
- 결과: 일치하는 1개 Row만 최종적으로 Lock 보유
- 동시에 다른 사용자 정보 UPDATE는 자유롭게 실행 가능

**근본 해결책**: `email` 컬럼에 인덱스 추가
```sql
CREATE INDEX idx_email ON users(email);
-- 이후: email='user@example.com'인 Row(최대 1건)에만 Lock
-- 격리 수준 무관하게 안전
```

격리 수준을 낮추는 것은 임시방편이며 인덱스 추가가 올바른 해결입니다. 이메일은 UNIQUE 컬럼이므로 UNIQUE INDEX가 적절합니다.

</details>

---

**Q2.** `ALTER TABLE orders ADD INDEX idx_amount(amount)`를 실행하려는데 계속 "Waiting for table metadata lock"에서 멈춘다. 원인을 찾고 해결하는 절차를 설명하라.

<details>
<summary>해설 보기</summary>

**원인**: 누군가 `orders` 테이블에 대한 Shared MDL을 보유 중인 트랜잭션이 COMMIT/ROLLBACK하지 않고 있습니다.

**진단 절차**:

```sql
-- 1. MDL 상태 확인
SELECT * FROM performance_schema.metadata_locks
WHERE OBJECT_TYPE = 'TABLE' AND OBJECT_NAME = 'orders';

-- 2. 차단 원인 트랜잭션 찾기
SELECT 
    p.id AS process_id,
    p.user,
    p.host,
    p.time AS running_seconds,
    p.state,
    LEFT(p.info, 100) AS query,
    t.trx_id,
    t.trx_started
FROM information_schema.PROCESSLIST p
LEFT JOIN information_schema.INNODB_TRX t 
    ON p.id = t.trx_mysql_thread_id
WHERE t.trx_id IS NOT NULL  -- 트랜잭션이 열려있는 것
ORDER BY t.trx_started;

-- 3. 발견된 장기 트랜잭션의 processlist id로 종료
KILL [process_id];
```

**해결 옵션**:
1. **원인 트랜잭션이 어드민/배치**: KILL 후 ALTER TABLE 재실행
2. **원인 트랜잭션이 서비스 쿼리**: 트래픽이 낮은 시간대로 미룸
3. **즉각 실행 필요**: pt-online-schema-change 또는 gh-ost 사용
   - 원본 테이블 수정 없이 새 테이블에 점진적 복사
   - MDL 보유 시간 최소화
   - Online DDL이 가능한 경우라면 `ALGORITHM=INPLACE, LOCK=NONE` 옵션

</details>

---

**Q3.** InnoDB 환경에서 `START TRANSACTION; LOCK TABLES orders WRITE; ...`를 실행했다. 어떤 부작용이 발생하고 이것이 위험한 이유는?

<details>
<summary>해설 보기</summary>

**`LOCK TABLES`는 현재 열려있는 트랜잭션을 암묵적으로 COMMIT합니다.**

```sql
START TRANSACTION;
UPDATE orders SET status = 'PROCESSING' WHERE id = 1;
-- 아직 COMMIT 안 됨

LOCK TABLES orders WRITE;
-- ← 이 순간 위의 UPDATE가 자동으로 COMMIT됨!
-- 이후 ROLLBACK을 해도 이미 반영된 UPDATE는 취소 불가

UPDATE orders SET status = 'FAILED' WHERE id = 1;
-- LOCK TABLES로 잠긴 상태에서 실행

ROLLBACK;  -- LOCK TABLES 이후의 변경만 효과 있음 (사실 auto-commit 모드가 됨)
UNLOCK TABLES;
```

**왜 위험한가**:
1. **원자성 파괴**: `START TRANSACTION`으로 원자성을 의도했지만 `LOCK TABLES`가 중간에 COMMIT을 강제. 이후 실패해도 처음 UPDATE는 이미 영구 반영됨.
2. **예측 불가능한 동작**: 코드를 읽는 사람이 "트랜잭션 내에 있다"고 기대하지만 실제는 다름.
3. **LOCK TABLES + InnoDB 혼용**: `LOCK TABLES`는 MySQL 서버 레벨, InnoDB 트랜잭션은 엔진 레벨. 두 Lock 시스템이 독립적으로 작동 → 예상 밖의 동작.

**올바른 InnoDB 패턴**:
```sql
START TRANSACTION;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;  -- Row Lock
UPDATE orders SET status = 'PROCESSING' WHERE id = 1;
-- 필요한 다른 작업
COMMIT;  -- 명시적 커밋, 트랜잭션 원자성 보장
```

</details>

---

<div align="center">

**[⬅️ 락 종류](./01-lock-types.md)** | **[홈으로 🏠](../README.md)** | **[다음: Gap Lock ➡️](./03-gap-lock-next-key-lock.md)**

</div>
