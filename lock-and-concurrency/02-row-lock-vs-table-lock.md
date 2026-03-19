# Row Lock vs Table Lock — InnoDB가 락을 거는 단위

---

## 🎯 핵심 질문

- InnoDB가 "인덱스 레코드에 Lock을 건다"는 말의 정확한 의미는?
- 인덱스가 없는 컬럼으로 WHERE 조건을 걸면 왜 Table Lock이 발생하는가?
- `LOCK TABLES`와 InnoDB의 자동 Row Lock은 어떻게 다른가?
- 인덱스 없는 UPDATE가 얼마나 많은 Row를 Lock하는가?

---

## 🔬 내부 동작 원리

### 1. InnoDB Lock = 인덱스 레코드 Lock

```
핵심 원칙:
  InnoDB의 Row Lock은 실제 Row가 아닌
  "인덱스 레코드"에 걸린다

왜 인덱스 레코드인가:
  InnoDB는 Clustered Index = 데이터 자체
  Row를 찾으려면 인덱스를 반드시 거쳐야 함
  → Lock도 인덱스 항목에 부착

인덱스 없는 컬럼 조건의 문제:
  WHERE status = 'PAID' (status에 인덱스 없음)
  
  InnoDB는 status = 'PAID'를 찾기 위해:
    Clustered Index(PK) 순서로 전체 스캔
    → 모든 PK 인덱스 레코드에 Lock 시도
    → 사실상 Full Table Lock!

증명:
  UPDATE orders SET memo = 'test' WHERE status = 'PAID';
  -- status 인덱스 없으면 전체 100만 건에 X Lock 걸림
  -- → 다른 모든 트랜잭션의 orders Row 접근 차단

  CREATE INDEX idx_status ON orders(status);
  UPDATE orders SET memo = 'test' WHERE status = 'PAID';
  -- 이제 status='PAID'인 33만 건에만 Lock
  -- → 다른 status Row는 자유롭게 접근 가능
```

### 2. 인덱스 유무에 따른 Lock 범위 비교

```sql
-- 실험: 인덱스 없는 UPDATE의 Lock 범위

-- 인덱스 없는 경우
ALTER TABLE orders DROP INDEX idx_status;

-- 세션 1
START TRANSACTION;
UPDATE orders SET memo = 'lock_test' WHERE status = 'PAID';
-- 내부적으로: 전체 Row 스캔 중 Lock 획득

-- 세션 2
UPDATE orders SET memo = 'blocked' WHERE status = 'PENDING';  -- 차단!
-- status='PENDING'인 Row에 접근하려 해도 세션 1이 전체 스캔 중 Lock 보유

-- Lock 상태 확인
SELECT COUNT(*) FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'orders' AND LOCK_TYPE = 'RECORD';
-- 매우 많은 수 (전체 Row 수에 근접)

-- 인덱스 추가 후 비교
CREATE INDEX idx_status ON orders(status);
ROLLBACK;  -- 세션 1

START TRANSACTION;
UPDATE orders SET memo = 'lock_test' WHERE status = 'PAID';

SELECT COUNT(*) FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'orders' AND LOCK_TYPE = 'RECORD';
-- status='PAID' 건수에만 Lock (훨씬 적음)
```

### 3. LOCK TABLES vs InnoDB Row Lock

```sql
-- LOCK TABLES: MySQL 서버 레벨 테이블 잠금
LOCK TABLES orders WRITE;
-- 다른 세션의 모든 orders 접근 차단 (SELECT 포함!)
-- UNLOCK TABLES까지 유지

-- InnoDB 자동 Row Lock: 트랜잭션 레벨
START TRANSACTION;
UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
-- id=1 Row에만 X Lock, 나머지 Row는 자유
COMMIT;  -- Lock 자동 해제

-- LOCK TABLES 사용이 필요한 경우:
-- MyISAM 테이블 (InnoDB가 아닌 경우)
-- 여러 테이블에 걸친 일관성 스냅샷 (mysqldump 등)
-- InnoDB 환경에서는 거의 불필요

-- DDL Lock (Metadata Lock):
-- DDL(ALTER TABLE 등)은 테이블 수준 MDL(Metadata Lock) 사용
-- 실행 중인 트랜잭션이 MDL과 충돌 시 ALTER TABLE 대기
-- → 장기 실행 트랜잭션이 있으면 ALTER TABLE이 멈출 수 있음

SHOW PROCESSLIST;
-- State: Waiting for table metadata lock → MDL 대기 중
```

### 4. 트랜잭션별 Lock 보유 현황

```sql
-- 현재 활성 트랜잭션과 보유 Lock 수
SELECT
    t.trx_id,
    t.trx_started,
    t.trx_state,
    t.trx_rows_locked,       -- 보유 중인 Row Lock 수
    t.trx_rows_modified,     -- 수정한 Row 수
    t.trx_tables_locked,     -- Lock 걸린 테이블 수
    LEFT(t.trx_query, 80) AS current_query
FROM information_schema.INNODB_TRX t
ORDER BY t.trx_started;

-- 특정 트랜잭션의 Lock 상세
SELECT
    dl.LOCK_TYPE,
    dl.LOCK_MODE,
    dl.LOCK_STATUS,
    dl.INDEX_NAME,
    dl.LOCK_DATA
FROM performance_schema.data_locks dl
WHERE dl.ENGINE_TRANSACTION_ID = 12345;  -- 특정 trx_id
```

---

## 📌 핵심 정리

```
Row Lock vs Table Lock 핵심:

InnoDB Row Lock = 인덱스 레코드 Lock:
  인덱스 있는 WHERE 조건 → 해당 인덱스 레코드에만 Lock
  인덱스 없는 WHERE 조건 → 전체 Row 스캔 = 사실상 Table Lock

핵심 교훈:
  UPDATE/DELETE의 WHERE 조건 컬럼에 반드시 인덱스!
  인덱스 없는 대량 DML = 전체 테이블 차단 = 서비스 장애

LOCK TABLES:
  서버 레벨 테이블 잠금, InnoDB 트랜잭션 환경에선 지양
  DDL(ALTER TABLE) 시 MDL로 인한 장기 대기 주의

Lock 진단:
  information_schema.INNODB_TRX: 트랜잭션별 Lock 수
  performance_schema.data_locks: 개별 Lock 상세
```

---

## 🤔 생각해볼 문제

**Q1.** `UPDATE orders SET status='SHIPPED' WHERE order_date = '2024-01-01'`에서 `order_date`에 인덱스가 없다. 이 UPDATE가 실행되는 동안 다른 트랜잭션이 `id=1`인 Row를 수정하려 한다. 차단되는가?

<details>
<summary>해설 보기</summary>

**차단될 가능성이 매우 높습니다.** `order_date` 인덱스가 없으면 InnoDB는 Clustered Index(PK)를 전체 스캔하면서 모든 Row에 X Lock을 시도합니다. `id=1` Row에도 Lock이 걸리므로 다른 트랜잭션의 수정이 차단됩니다.

단, InnoDB는 실제로 조건에 맞지 않는 Row는 Lock을 해제하는 최적화(Semi-consistent Read)를 `UPDATE`에서 부분적으로 적용합니다. 하지만 이는 `READ COMMITTED` 격리 수준에서만 일관되게 동작하고, `REPEATABLE READ`에서는 스캔한 모든 Row에 Lock이 유지될 수 있습니다.

결론: `order_date`에 인덱스를 추가하면 `order_date='2024-01-01'`인 Row에만 Lock이 걸려 `id=1`인 다른 Row는 영향받지 않습니다. 이것이 DML의 WHERE 조건 컬럼에 인덱스가 필수인 이유입니다.

</details>

---

<div align="center">

**[⬅️ 락 종류](./01-lock-types.md)** | **[홈으로 🏠](../README.md)** | **[다음: Gap Lock ➡️](./03-gap-lock-next-key-lock.md)**

</div>
