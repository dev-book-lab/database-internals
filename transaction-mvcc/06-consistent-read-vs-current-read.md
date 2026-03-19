# Consistent Read vs Current Read

---

## 🎯 핵심 질문

- Consistent Read와 Current Read를 유발하는 SQL은 각각 무엇인가?
- `SELECT`와 `SELECT ... FOR UPDATE`가 읽는 버전이 다른 이유는?
- JPA `@Lock(LockModeType.PESSIMISTIC_WRITE)`는 어떤 SQL을 생성하는가?
- `SELECT ... FOR UPDATE`로 읽은 후 UPDATE할 때 Lost Update가 방지되는 이유는?
- Consistent Read와 Current Read를 같은 트랜잭션에서 혼용하면 어떤 문제가 생기는가?

---

## 🔬 내부 동작 원리

### 1. 두 읽기 방식의 차이

```
Consistent Read (스냅샷 읽기):
  SQL: SELECT (잠금 없는 일반 SELECT)
  버전: Read View 기준의 스냅샷 (MVCC)
  Lock: 없음
  REPEATABLE READ: Read View 고정 (항상 같은 스냅샷)
  READ COMMITTED: SELECT마다 최신 커밋 스냅샷

Current Read (잠금 읽기):
  SQL: SELECT ... FOR UPDATE
       SELECT ... FOR SHARE
       UPDATE ... (WHERE 조건으로 Row 찾는 부분)
       DELETE ... (WHERE 조건으로 Row 찾는 부분)
       INSERT (중복 체크 시)
  버전: 항상 최신 커밋된 버전 (MVCC 우회)
  Lock: FOR UPDATE → X Lock, FOR SHARE → S Lock
  격리 수준에 무관하게 항상 최신 버전

왜 Current Read가 필요한가:
  UPDATE/DELETE는 변경할 Row를 찾아야 함
  MVCC 스냅샷이 아닌 현재 상태를 Lock해야 함
  → 다른 트랜잭션의 변경과 충돌 방지 가능
  → 스냅샷 기준 UPDATE = 이미 다른 트랜잭션이 변경한 Row에 겹쳐 쓰기 위험
```

### 2. 혼용 시 발생하는 문제

```sql
-- 문제 시나리오: Consistent Read 후 Current Read

-- REPEATABLE READ
START TRANSACTION;  -- TRX A

-- Consistent Read (스냅샷): balance = 10000
SELECT balance FROM accounts WHERE id = 1;  

-- 세션 B에서: balance = 8000으로 변경 후 COMMIT

-- Current Read (최신 버전): balance = 8000
UPDATE accounts SET balance = balance - 2000 WHERE id = 1;
-- UPDATE는 Current Read → 현재 최신 값 8000 기준으로 적용
-- 결과: 8000 - 2000 = 6000

COMMIT;

-- 결과: SELECT는 10000 보임, 실제 UPDATE는 8000 기준
-- → 개발자 예상: 10000 - 2000 = 8000
-- → 실제 결과: 6000
-- → 버그 발생!

올바른 패턴:
  SELECT ... FOR UPDATE (Current Read) → 최신 값 읽기 + Lock
  UPDATE ... (Current Read) → Lock된 Row 변경
  → 일관된 Current Read 사용
```

### 3. SELECT FOR UPDATE — Lost Update 방지

```sql
-- Lost Update 문제:
-- TRX A: balance 읽음 (10000) → balance - 3000 계산 → UPDATE (7000)
-- TRX B: balance 읽음 (10000) → balance - 5000 계산 → UPDATE (5000)
-- 최종: 5000 (TRX A의 차감 3000이 사라짐 = Lost Update!)

-- 일반 SELECT로는 Lost Update 방지 불가:
-- TRX A
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 10000 (스냅샷)

-- TRX B (동시)
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 10000 (스냅샷)
UPDATE accounts SET balance = 5000 WHERE id = 1;  -- 10000 - 5000
COMMIT;

-- TRX A (계속)
UPDATE accounts SET balance = 7000 WHERE id = 1;  -- Current Read → 5000이 최신
-- 5000으로 Lock 후 7000으로 UPDATE (TRX B의 차감이 사라짐!)
COMMIT;

-- FOR UPDATE로 해결:
-- TRX A
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- X Lock 획득! TRX B의 UPDATE 차단

-- TRX B
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- TRX A의 X Lock이 있어서 대기!

-- TRX A (계속)
UPDATE accounts SET balance = balance - 3000 WHERE id = 1;
COMMIT;  -- X Lock 해제

-- TRX B (Lock 해제 후 재시작)
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- 7000 (TRX A 반영값)
UPDATE accounts SET balance = balance - 5000 WHERE id = 1;  -- 7000 - 5000 = 2000
COMMIT;
-- 최종: 2000 (두 차감 모두 반영)
```

### 4. JPA @Lock 어노테이션

```java
// PESSIMISTIC_WRITE: SELECT ... FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdWithLock(@Param("id") Long id);

// 생성 SQL:
// SELECT * FROM accounts WHERE id = ? FOR UPDATE
// → X Lock 획득

// PESSIMISTIC_READ: SELECT ... FOR SHARE
@Lock(LockModeType.PESSIMISTIC_READ)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdWithShareLock(@Param("id") Long id);

// 생성 SQL:
// SELECT * FROM accounts WHERE id = ? FOR SHARE (또는 LOCK IN SHARE MODE)
// → S Lock 획득 (다른 S Lock과 공존 가능, X Lock과 충돌)

// OPTIMISTIC: 버전 필드 기반 (Lock 없음)
@Version
private Long version;

// → UPDATE ... WHERE id = ? AND version = ? (버전 불일치 시 OptimisticLockException)

// 올바른 계좌 이체:
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // FOR UPDATE로 두 계좌 모두 Lock (교착상태 방지: ID 순서대로)
    Account from = accountRepository.findByIdWithLock(Math.min(fromId, toId));
    Account to = accountRepository.findByIdWithLock(Math.max(fromId, toId));
    
    from.withdraw(amount);  // 비즈니스 규칙 검증 포함
    to.deposit(amount);
    // → COMMIT 시 두 변경 모두 반영 (Atomicity)
    // → FOR UPDATE Lock으로 동시 이체 시 Lost Update 방지 (Isolation)
}
```

### 5. SELECT FOR SKIP LOCKED / NOWAIT

```sql
-- FOR UPDATE SKIP LOCKED: Lock된 Row 건너뛰기 (Queue 처리에 유용)
SELECT * FROM tasks WHERE status = 'PENDING' LIMIT 1 FOR UPDATE SKIP LOCKED;
-- 다른 트랜잭션이 처리 중인 Task를 건너뛰고 사용 가능한 Task 반환
-- → 여러 워커가 동시에 Task를 처리하는 패턴에 적합

-- FOR UPDATE NOWAIT: Lock 즉시 실패 (대기 없음)
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;
-- 다른 트랜잭션이 Lock 중이면 즉시 에러 반환 (대기 없음)
-- → 비관적 Lock이지만 대기로 인한 타임아웃 방지

-- 활용 사례:
-- SKIP LOCKED: 메시지 큐, 분산 작업 처리
-- NOWAIT: 사용자 UI에서 즉각적 피드백 필요 (재시도 유도)
```

---

## 💻 실전 실험

### 실험: Consistent Read vs Current Read 버전 차이

```sql
-- 세션 1 (REPEATABLE READ)
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 10000 (스냅샷 Read View 생성)

-- 세션 2
UPDATE accounts SET balance = 7000 WHERE id = 1;
COMMIT;

-- 세션 1 (계속)
SELECT balance FROM accounts WHERE id = 1;           -- 10000 (Consistent Read, 스냅샷)
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE; -- 7000  (Current Read, 최신)

-- 두 결과가 다름! → 같은 트랜잭션에서 혼용 주의
COMMIT;
```

---

## 📌 핵심 정리

```
Consistent Read vs Current Read 핵심:

Consistent Read (일반 SELECT):
  MVCC 스냅샷 (Read View 기준)
  Lock 없음 → 쓰기와 비차단
  REPEATABLE READ: 항상 같은 스냅샷
  READ COMMITTED: 최신 커밋 스냅샷

Current Read (FOR UPDATE, FOR SHARE, DML):
  최신 커밋 버전 (MVCC 우회)
  Lock 획득 (X Lock 또는 S Lock)
  격리 수준 무관하게 최신 버전
  Phantom Read 방지: Gap Lock + Next-Key Lock

올바른 패턴:
  값 읽고 조건 검사 후 업데이트: FOR UPDATE 사용
  단순 조회: 일반 SELECT
  혼용 금지: Consistent + Current 섞으면 버그 유발

JPA:
  @Lock(PESSIMISTIC_WRITE): FOR UPDATE
  @Lock(PESSIMISTIC_READ): FOR SHARE
  @Version: Optimistic Lock (Lock 없음, 버전 충돌 시 예외)
```

---

## 🤔 생각해볼 문제

**Q1.** `SELECT ... FOR UPDATE`와 `SELECT ... FOR SHARE`의 차이를 실제 사용 사례와 함께 설명하라.

<details>
<summary>해설 보기</summary>

**FOR UPDATE (X Lock)**:
- 읽은 Row에 Exclusive Lock. 다른 트랜잭션의 읽기(FOR SHARE도)와 쓰기를 모두 차단.
- 사용 사례: 읽고 나서 반드시 수정할 때 (계좌 이체, 재고 차감, 상태 변경)

**FOR SHARE (S Lock)**:
- 읽은 Row에 Shared Lock. 다른 트랜잭션의 FOR SHARE는 허용, 쓰기(FOR UPDATE, UPDATE)는 차단.
- 사용 사례: 읽은 값이 변경되지 않아야 하지만 직접 수정은 안 할 때, 외래 키 확인 (부모 Row가 삭제되지 않도록 보장)

```java
// FOR SHARE 예시: 부모 Row 삭제 방지
@Transactional
public void createOrderItem(Long orderId, Long productId) {
    // Order가 삭제되지 않도록 S Lock
    Order order = orderRepository.findByIdWithShareLock(orderId);
    // Product도 S Lock
    Product product = productRepository.findByIdWithShareLock(productId);
    
    orderItemRepository.save(new OrderItem(order, product));
    // COMMIT 시까지 Order, Product 삭제 불가
}
```

</details>

---

<div align="center">

**[⬅️ Phantom Read](./05-phantom-read.md)** | **[홈으로 🏠](../README.md)** | **[다음: Redo Log ➡️](./07-redo-log-durability.md)**

</div>
