# 락의 종류 — Shared, Exclusive, Intention Lock

---

## 🎯 핵심 질문

- S Lock과 X Lock의 호환 행렬에서 어떤 조합이 차단을 일으키는가?
- IS/IX Intention Lock은 왜 테이블 레벨에서 필요한가?
- `performance_schema.data_locks`로 현재 Lock 상태를 어떻게 읽는가?
- AUTO-INC Lock은 무엇이고 왜 성능 문제를 일으킬 수 있는가?

---

## 🔬 내부 동작 원리

### 1. S Lock과 X Lock

```
Shared Lock (S Lock, 공유 잠금):
  획득 SQL: SELECT ... FOR SHARE (또는 LOCK IN SHARE MODE)
  의미: "이 Row를 읽겠다. 다른 읽기는 허용하지만 쓰기는 차단"
  
  허용: 다른 S Lock과 공존 (여러 트랜잭션이 동시 읽기 가능)
  차단: X Lock (쓰기 트랜잭션이 S Lock이 있는 Row 수정 불가)

Exclusive Lock (X Lock, 배타 잠금):
  획득 SQL: SELECT ... FOR UPDATE, UPDATE, DELETE
  의미: "이 Row를 수정하겠다. 다른 어떤 Lock도 허용하지 않음"
  
  차단: 다른 모든 S Lock, X Lock

호환 행렬:
          기존 S Lock    기존 X Lock
  요청 S     호환 ✅        충돌 ❌
  요청 X     충돌 ❌        충돌 ❌

→ S-S만 공존 가능, 나머지는 모두 대기 or 데드락
```

### 2. Intention Lock — 테이블 레벨 의도 표시

```
왜 필요한가:
  LOCK TABLES t WRITE (테이블 전체 X Lock) 시도 시
  InnoDB는 테이블 내 모든 Row의 Lock 상태를 일일이 확인해야 하는가?
  → 100만 건이면 100만 번 확인 → 비용 폭발

Intention Lock 해결책:
  Row Lock 획득 전에 테이블 레벨 Intention Lock 먼저 획득
  → 테이블 수준에서 "이 테이블에 Row Lock이 있다"는 신호

  IS (Intention Shared):
    Row에 S Lock을 걸기 전 테이블에 획득
  IX (Intention Exclusive):
    Row에 X Lock을 걸기 전 테이블에 획득

테이블 레벨 호환 행렬:
          IS      IX     S(table)  X(table)
  IS      ✅      ✅       ✅        ❌
  IX      ✅      ✅       ❌        ❌
  S(tbl)  ✅      ❌       ✅        ❌
  X(tbl)  ❌      ❌       ❌        ❌

LOCK TABLES t WRITE 시도:
  테이블에 X Lock 요청 → 기존 IX Lock과 충돌 → 대기
  (IX가 있다는 것 = 어딘가 Row X Lock이 있다는 신호)
  → Row 하나하나 확인 없이 테이블 레벨 충돌로 빠르게 감지
```

### 3. AUTO-INC Lock

```
AUTO_INCREMENT PK에 INSERT 시 AUTO-INC Lock 사용:

innodb_autoinc_lock_mode:
  0 (Traditional): 모든 INSERT에 테이블 레벨 AUTO-INC Lock
                   → 한 번에 1 INSERT만 AUTO_INCREMENT 획득
                   → 동시성 매우 낮음
  
  1 (Consecutive, MySQL 5.7 기본):
    Simple INSERT (값 수 확정 가능): 경량 뮤텍스 사용 (즉시 해제)
    Bulk INSERT (값 수 불확정): 테이블 레벨 AUTO-INC Lock
  
  2 (Interleaved, MySQL 8.0 기본):
    모든 INSERT에 경량 뮤텍스 → 동시성 최고
    단, Bulk INSERT에서 AUTO_INCREMENT 값이 연속적이지 않을 수 있음
    → Statement-based Replication과 비호환 (Row-based 사용 권장)

실무 영향:
  대용량 Bulk INSERT 병렬 실행 시 mode 0/1에서 병목
  mode 2 + Row-based Replication이 현대 권장 설정
```

### 4. performance_schema.data_locks 읽기

```sql
-- 현재 Lock 상태 조회
SELECT
    ENGINE_LOCK_ID,
    ENGINE_TRANSACTION_ID,
    OBJECT_SCHEMA AS db,
    OBJECT_NAME AS tbl,
    INDEX_NAME,
    LOCK_TYPE,    -- TABLE 또는 RECORD
    LOCK_MODE,    -- S, X, IS, IX, GAP, AUTO_INC 등
    LOCK_STATUS,  -- GRANTED (보유 중) 또는 WAITING (대기 중)
    LOCK_DATA     -- 잠긴 레코드의 키 값
FROM performance_schema.data_locks;

-- LOCK_MODE 값 해석:
-- S: Shared Lock (Record Lock)
-- X: Exclusive Lock (Record Lock)
-- S,GAP: Shared Gap Lock
-- X,GAP: Exclusive Gap Lock
-- S,REC_NOT_GAP: Record만 (Gap 제외) S Lock
-- X,REC_NOT_GAP: Record만 (Gap 제외) X Lock
-- X,INSERT_INTENTION: Insert Intention Lock (삽입 직전 짧게 획득)

-- 대기 관계 확인:
SELECT
    r.trx_id AS waiting_trx,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id;
```

---

## 💻 실전 실험

```sql
-- S Lock과 X Lock 충돌 확인
-- 세션 1
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;  -- S Lock

-- 세션 2 (별도 창)
START TRANSACTION;
UPDATE accounts SET balance = 5000 WHERE id = 1;  -- X Lock 요청 → 대기!

-- performance_schema로 확인
SELECT LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'accounts'
ORDER BY LOCK_STATUS;
-- GRANTED: 세션 1의 S Lock
-- WAITING: 세션 2의 X Lock

-- 세션 1 COMMIT → 세션 2 X Lock 획득
COMMIT;
```

---

## 📌 핵심 정리

```
Lock 종류 핵심:

S Lock: 읽기 (FOR SHARE)
  S-S 공존, S-X 충돌

X Lock: 쓰기 (FOR UPDATE, UPDATE, DELETE)
  모든 다른 Lock과 충돌

IS/IX: 테이블 레벨 의도 표시
  Row Lock 전에 자동 획득
  LOCK TABLES와의 충돌을 효율적으로 감지

AUTO-INC Lock:
  innodb_autoinc_lock_mode=2 + Row-based Replication 권장

진단:
  performance_schema.data_locks: 현재 Lock 상태
  LOCK_STATUS=WAITING: 차단된 트랜잭션 식별
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 Row에 두 트랜잭션이 동시에 `SELECT ... FOR SHARE`하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**두 트랜잭션 모두 즉시 S Lock을 획득하고 동시에 실행됩니다.** S-S 호환이므로 서로 차단하지 않습니다. 두 트랜잭션 모두 해당 Row를 읽을 수 있습니다.

문제는 이후에 발생합니다. 둘 다 S Lock을 보유한 상태에서 동시에 그 Row를 UPDATE(X Lock 요청)하려 하면 데드락이 발생합니다. 이것이 "S Lock 업그레이드 데드락" 패턴입니다.

TRX A: S Lock 보유 → X Lock 요청 → TRX B의 S Lock이 차단
TRX B: S Lock 보유 → X Lock 요청 → TRX A의 S Lock이 차단
→ 순환 대기 → 데드락!

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Row Lock vs Table Lock ➡️](./02-row-lock-vs-table-lock.md)**

</div>
