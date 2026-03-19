# Consistent Read vs Current Read

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Consistent Read와 Current Read를 유발하는 SQL의 정확한 목록은?
- `SELECT`와 `SELECT ... FOR UPDATE`가 읽는 데이터 버전이 다른 내부 이유는?
- 같은 트랜잭션에서 두 방식을 혼용하면 어떤 버그가 발생하는가?
- JPA `@Lock(PESSIMISTIC_WRITE)`가 생성하는 SQL과 그 효과는?
- `SELECT ... FOR UPDATE SKIP LOCKED`를 활용하는 적합한 패턴은?
- Lost Update 문제가 Consistent Read를 사용할 때 발생하는 이유는?

---

## 🔍 왜 이 개념이 중요한가

### "같은 트랜잭션인데 두 번 읽었더니 다른 결과가 나왔다" — Consistent와 Current Read 혼용 버그

```
실제 버그:

@Transactional  // REPEATABLE_READ
public void processRefund(Long orderId) {
    Order order = orderRepository.findById(orderId);
    // → Consistent Read: 스냅샷 기준 order.status = "PAID"
    
    // [다른 스레드: order.status = "REFUNDED" + COMMIT]
    
    if ("PAID".equals(order.getStatus())) {
        order.setStatus("REFUNDING");
        orderRepository.save(order);
        // → UPDATE 내부: Current Read → 최신 order.status = "REFUNDED" 확인
        // → 변경은 "REFUNDING"으로 덮어씀
        // → "REFUNDED" 상태가 사라짐!
    }
}

문제:
  findById (Consistent Read) → "PAID" 읽음
  save (Current Read + UPDATE) → "REFUNDED" 상태를 "REFUNDING"으로 덮어씀
  두 읽기 방식의 혼용 → 의도치 않은 상태 덮어쓰기

올바른 패턴:
  findById → findByIdWithLock (FOR UPDATE) 변경
  → Current Read로 통일 → "REFUNDED" 상태 확인
  → if 조건 실패 → 처리 중단
```

---

## 😱 잘못된 이해

### Before: SELECT와 UPDATE가 같은 버전을 읽는다고 가정

```sql
-- 잘못된 가정:
-- "SELECT로 읽은 값을 UPDATE에서 사용하면 일관성이 보장된다"

START TRANSACTION;  -- REPEATABLE_READ

-- 현재 재고 확인
SELECT stock FROM products WHERE id = 1;  -- Consistent Read → 10

-- [다른 세션: stock = 7로 UPDATE + COMMIT]

-- "10에서 5를 뺀 5로 설정"
UPDATE products SET stock = 10 - 5 WHERE id = 1;
-- 실제로는 Current Read → 최신 값 7 가져옴
-- 결과: stock = 7 - 5 = 2  (10 - 5 = 5가 아님!)

COMMIT;
-- 예상: 5
-- 실제: 2
-- → SELECT와 UPDATE가 다른 버전을 기준으로 실행됨
```

```
또 다른 잘못된 가정:
  "REPEATABLE_READ에서 모든 읽기가 같은 버전이다"
  → 실제: SELECT = Consistent Read (스냅샷)
           SELECT FOR UPDATE / UPDATE / DELETE = Current Read (최신)
  
  "UPDATE를 실행해도 트랜잭션 내의 SELECT 결과는 안 변한다"
  → 내 트랜잭션의 UPDATE는 반영됨 (규칙: creator_trx_id 항상 보임)
  → 다른 트랜잭션의 변경은 SELECT에서는 안 보임
     하지만 UPDATE의 WHERE 조건은 Current Read로 평가됨
```

---

## ✨ 올바른 이해

### After: 읽기 방식에 따라 참조하는 데이터 버전이 다름

```
Consistent Read (스냅샷 읽기):
  사용 SQL: SELECT (잠금 없는 일반 읽기)
  데이터 버전: Read View 기준 스냅샷 (MVCC Undo Log 버전 체인)
  Lock: 없음
  특성: 다른 어떤 Lock도 받지 않음

Current Read (잠금 읽기):
  사용 SQL:
    SELECT ... FOR UPDATE    → X Lock 획득
    SELECT ... FOR SHARE     → S Lock 획득
    UPDATE ... WHERE ...     → X Lock 후 수정
    DELETE ... WHERE ...     → X Lock 후 삭제
    INSERT (중복 체크 부분)  → 잠깐의 X Lock
  데이터 버전: Buffer Pool의 최신 커밋 버전 (MVCC 우회)
  Lock: X Lock 또는 S Lock (해당 Row에 대한)
  특성: 항상 최신 상태 읽음, COMMIT까지 Lock 유지

핵심 원칙:
  데이터를 읽고 그 값을 기반으로 수정할 때:
    읽기도 Current Read로 → FOR UPDATE 사용
  단순 조회만 할 때:
    일반 SELECT (Consistent Read) 사용

혼용하면 안 되는 이유:
  Consistent Read → 스냅샷 기준 값 A 읽음
  Current Read    → 최신값 B 기준으로 수정 실행
  A와 B가 다를 수 있음 → 읽은 값 기반 로직이 잘못된 값으로 실행됨
```

---

## 🔬 내부 동작 원리

### 1. 두 읽기 방식의 내부 경로 차이

```
Consistent Read 실행 경로:

  SELECT balance FROM accounts WHERE id = 1;

  ① InnoDB에서 id=1 Row 위치 찾기 (인덱스 탐색)
  ② Row의 DB_TRX_ID 확인
  ③ is_visible(DB_TRX_ID, read_view) 판단
     - True: 이 버전 반환
     - False: DB_ROLL_PTR 따라 이전 버전으로 → 반복
  ④ Lock 없음 → 즉시 반환

Current Read 실행 경로:

  SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

  ① InnoDB에서 id=1 Row 위치 찾기
  ② Lock 획득 시도 (X Lock)
     - 다른 트랜잭션의 Lock 있으면 대기
     - Lock 획득 완료
  ③ Buffer Pool의 최신 커밋 버전 직접 읽기 (MVCC 우회)
     (아직 COMMIT 안 된 버전은 건너뜀)
  ④ COMMIT까지 X Lock 유지

결정적 차이:
  Consistent: 스냅샷 기준, Lock 없음, 과거 버전 가능
  Current:    최신 커밋, Lock 있음, 항상 현재 상태
```

### 2. Lost Update 문제 — Consistent Read의 한계

```
Lost Update 시나리오:

  TRX A:
    T1: SELECT balance → 10000 (Consistent Read, Lock 없음)
    T3: UPDATE SET balance = 10000 - 3000 = 7000

  TRX B:
    T2: SELECT balance → 10000 (Consistent Read, Lock 없음)
    T4: UPDATE SET balance = 10000 - 5000 = 5000

  T4가 T3보다 늦게 실행되면:
    TRX A의 결과(7000)를 TRX B가 덮어씀
    최종: 5000 (TRX A의 -3000 효과가 사라짐 = Lost Update)

왜 발생하는가:
  두 트랜잭션 모두 Consistent Read로 10000을 읽음 (Lock 없음)
  UPDATE의 SET 절: "10000 - 3000" = 읽었던 값을 하드코딩
  UPDATE의 WHERE: Current Read → 최신 Row 찾아 Lock
  → SET 값은 스냅샷 기준, WHERE는 최신 기준 → 불일치

FOR UPDATE로 해결:
  TRX A:
    T1: SELECT balance FOR UPDATE → 10000 (X Lock 획득)
  TRX B:
    T2: SELECT balance FOR UPDATE → X Lock 요청 → TRX A 대기
  TRX A:
    T3: UPDATE SET balance = 7000
    COMMIT → X Lock 해제
  TRX B:
    T4: 이제 X Lock 획득, balance = 7000 (최신값) 읽음
        UPDATE SET balance = 7000 - 5000 = 2000
        COMMIT
  최종: 2000 (두 차감 모두 반영)

또는 단순 산술 UPDATE:
  UPDATE accounts SET balance = balance - 3000 WHERE id = 1;
  -- "balance - 3000"은 Current Read로 현재 최신값에서 차감
  -- → Lost Update 없음
```

### 3. SELECT FOR UPDATE의 Lock 종류와 범위

```
FOR UPDATE 세부 옵션:

-- 기본 FOR UPDATE (X Lock)
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- id=1 Row에 X Lock
-- 다른 트랜잭션의 S Lock / X Lock 모두 차단

-- FOR SHARE (S Lock)
SELECT * FROM orders WHERE id = 1 FOR SHARE;
-- id=1 Row에 S Lock
-- 다른 S Lock: 공존 가능
-- 다른 X Lock: 차단

-- SKIP LOCKED (다른 트랜잭션이 Lock한 Row 건너뜀)
SELECT * FROM tasks WHERE status = 'PENDING' LIMIT 5 FOR UPDATE SKIP LOCKED;
-- Lock된 Row를 건너뛰고 Lock 가능한 Row만 반환
-- 대기 없이 즉시 결과 반환
-- → 분산 작업 처리 (여러 워커가 동시에 작업 pick-up)에 최적

-- NOWAIT (Lock 즉시 실패)
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;
-- 다른 트랜잭션의 Lock이 있으면 즉시 에러 반환
-- → 대기 없이 "현재 사용 중" 즉시 알림
-- 사용자 UI에서 빠른 피드백 제공

-- 범위 FOR UPDATE
SELECT * FROM orders WHERE user_id = 1 FOR UPDATE;
-- user_id=1인 모든 Row에 X Lock
-- REPEATABLE_READ: Next-Key Lock (Row + Gap)
-- READ_COMMITTED: Record Lock만 (Gap Lock 없음)
```

### 4. JPA @Lock 어노테이션 구현

```java
// PESSIMISTIC_WRITE → SELECT ... FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdWithWriteLock(@Param("id") Long id);
// 생성 SQL: SELECT ... FROM accounts WHERE id = ? FOR UPDATE

// PESSIMISTIC_READ → SELECT ... FOR SHARE
@Lock(LockModeType.PESSIMISTIC_READ)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdWithReadLock(@Param("id") Long id);
// 생성 SQL: SELECT ... FROM accounts WHERE id = ? FOR SHARE

// PESSIMISTIC_FORCE_INCREMENT → FOR UPDATE + version 증가
@Lock(LockModeType.PESSIMISTIC_FORCE_INCREMENT)
Optional<Account> findByIdForVersionIncrement(Long id);

// 올바른 계좌 이체 구현:
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // 항상 작은 ID 먼저 Lock (Deadlock 방지를 위한 순서 통일)
    Long first = Math.min(fromId, toId);
    Long second = Math.max(fromId, toId);

    Account a1 = accountRepository.findByIdWithWriteLock(first)
        .orElseThrow(AccountNotFoundException::new);
    Account a2 = accountRepository.findByIdWithWriteLock(second)
        .orElseThrow(AccountNotFoundException::new);

    // Current Read로 최신 잔액 확인 (FOR UPDATE 사용했으므로)
    Account from = fromId.equals(first) ? a1 : a2;
    Account to = toId.equals(first) ? a1 : a2;

    from.withdraw(amount);  // 비즈니스 규칙 검증 포함
    to.deposit(amount);
    // COMMIT 시 두 변경 모두 반영 (Atomicity)
    // FOR UPDATE Lock으로 동시 이체 간 Lost Update 방지 (Isolation)
}

// SKIP LOCKED 활용 패턴 (JPA에서 Native Query 필요):
@Query(value = "SELECT * FROM tasks WHERE status = 'PENDING' " +
               "ORDER BY created_at LIMIT :size FOR UPDATE SKIP LOCKED",
       nativeQuery = true)
List<Task> findAndLockPendingTasks(@Param("size") int size);
// → 여러 워커 스레드가 동시에 호출해도 중복 없이 작업 분배
```

### 5. Consistent Read와 Current Read 혼용의 전체 패턴

```sql
-- 혼용이 문제를 일으키는 모든 패턴:

-- 패턴 1: SELECT 읽기 → UPDATE (일반적인 혼용)
START TRANSACTION;
SELECT stock FROM products WHERE id = 1;  -- Consistent Read → 10
-- [다른 세션이 stock=7로 변경 + COMMIT]
UPDATE products SET stock = stock - 5 WHERE id = 1;
-- UPDATE의 WHERE: Current Read → 최신 stock=7
-- UPDATE의 SET: stock - 5 = 7 - 5 = 2 (의도는 10 - 5 = 5였는데)
-- 그나마 "stock - 5"처럼 현재값 기반 산술은 괜찮음

-- 패턴 2: SELECT 결과를 변수에 저장 → UPDATE에 활용 (위험)
START TRANSACTION;
SET @stock := (SELECT stock FROM products WHERE id = 1);  -- Consistent Read → 10
-- [다른 세션이 stock=7로 변경 + COMMIT]
UPDATE products SET stock = @stock - 5 WHERE id = 1;
-- stock = 10 - 5 = 5 로 설정 (실제 최신값 7을 무시!)
-- Lost Update 발생: 다른 세션의 stock=7 변경이 사라짐
COMMIT;

-- 패턴 3: 올바른 방법 - FOR UPDATE로 통일
START TRANSACTION;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;  -- Current Read → X Lock
-- [다른 세션 차단됨]
UPDATE products SET stock = stock - 5 WHERE id = 1;  -- 최신 값 기반
COMMIT;

-- 또는 더 단순하게:
START TRANSACTION;
UPDATE products SET stock = stock - 5 WHERE id = 1;  -- Current Read 자동 사용
COMMIT;
```

---

## 💻 실전 실험

### 실험 1: Consistent Read vs Current Read 버전 차이 확인

```sql
-- 세션 1 (REPEATABLE_READ)
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- Consistent Read → 10000 (Read View 생성)

-- 세션 2
UPDATE accounts SET balance = 7000 WHERE id = 1;
COMMIT;

-- 세션 1 계속
SELECT balance FROM accounts WHERE id = 1;                    -- 10000 (Consistent, 스냅샷)
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;         -- 7000  (Current, 최신)
SELECT balance FROM accounts WHERE id = 1 FOR SHARE;          -- 7000  (Current, 최신)

-- 두 결과가 다름!
-- 같은 트랜잭션, 같은 Row, 다른 읽기 방식 → 다른 결과
COMMIT;
```

### 실험 2: Lost Update 발생 확인 및 FOR UPDATE 해결

```sql
-- Lost Update 재현 (두 세션 동시 실행)

-- 세션 1
START TRANSACTION;
SELECT stock FROM products WHERE id = 1;  -- 10 (Consistent Read)

-- 세션 2
START TRANSACTION;
SELECT stock FROM products WHERE id = 1;  -- 10 (Consistent Read)
UPDATE products SET stock = 10 - 3 WHERE id = 1;  -- 7로 설정
COMMIT;

-- 세션 1 (계속)
UPDATE products SET stock = 10 - 5 WHERE id = 1;  -- 5로 덮어씀!
COMMIT;
-- 최종: 5 (세션 2의 -3 효과 사라짐)

-- FOR UPDATE로 해결
-- 세션 1
START TRANSACTION;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;  -- X Lock 획득

-- 세션 2
START TRANSACTION;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;  -- 차단! 세션 1 대기

-- 세션 1
UPDATE products SET stock = stock - 5 WHERE id = 1;
COMMIT;  -- Lock 해제

-- 세션 2 (이제 진행)
-- stock = 5 (세션 1 결과) - 3 = 2
COMMIT;
-- 최종: 2 (두 차감 모두 반영)
```

### 실험 3: SKIP LOCKED 분산 작업 처리

```sql
-- 작업 큐 테이블
CREATE TABLE tasks (
    id     BIGINT AUTO_INCREMENT PRIMARY KEY,
    title  VARCHAR(100),
    status ENUM('PENDING', 'PROCESSING', 'DONE') DEFAULT 'PENDING'
) ENGINE=InnoDB;

INSERT INTO tasks (title) VALUES ('Task1'), ('Task2'), ('Task3'), ('Task4'), ('Task5');

-- 워커 1
START TRANSACTION;
SELECT * FROM tasks WHERE status = 'PENDING' LIMIT 2 FOR UPDATE SKIP LOCKED;
-- Task1, Task2 반환 + Lock

-- 워커 2 (동시에)
START TRANSACTION;
SELECT * FROM tasks WHERE status = 'PENDING' LIMIT 2 FOR UPDATE SKIP LOCKED;
-- Task3, Task4 반환 (Task1, Task2는 Lock됐으므로 건너뜀)

-- 각 워커가 자신의 Task 처리
UPDATE tasks SET status = 'PROCESSING' WHERE id IN (1, 2);  -- 워커 1
UPDATE tasks SET status = 'PROCESSING' WHERE id IN (3, 4);  -- 워커 2

COMMIT;  -- 두 워커 모두

-- 결과: 중복 없이 작업 분배 성공
```

### 실험 4: FOR SHARE 동작 확인

```sql
-- FOR SHARE: 읽기 Lock (다른 읽기와 공존, 쓰기 차단)

-- 세션 1: 부모 Row 읽기 보호
START TRANSACTION;
SELECT * FROM users WHERE id = 1 FOR SHARE;  -- S Lock

-- 세션 2
SELECT * FROM users WHERE id = 1 FOR SHARE;  -- S Lock → 공존 가능! 즉시 성공
UPDATE users SET name = 'New Name' WHERE id = 1;  -- X Lock 요청 → 세션 1의 S Lock이 차단!

-- 세션 1 COMMIT
COMMIT;

-- 세션 2의 UPDATE 이제 진행
-- S Lock은 부모 Row 삭제/수정 방지에 유용 (외래 키 무결성 보장 패턴)
```

---

## 📊 성능 비교

```
Consistent Read vs Current Read 성능:

Consistent Read (일반 SELECT):
  Lock 없음 → 다른 트랜잭션 차단 없음
  버전 체인 탐색 → Undo Log 접근 (스냅샷이 오래될수록 비용 증가)
  처리량: 높음 (Lock 경합 없음)

Current Read (FOR UPDATE):
  X Lock 획득 → 같은 Row의 다른 Lock 차단
  Buffer Pool 최신 버전 직독 → Undo Log 탐색 없음
  처리량: Lock 경합에 따라 감소

FOR UPDATE SKIP LOCKED vs 일반 FOR UPDATE:
  일반 FOR UPDATE: Lock 획득까지 대기 → 직렬 처리
  SKIP LOCKED: 즉시 사용 가능한 Row만 반환 → 병렬 처리 가능
  분산 작업 처리: SKIP LOCKED가 10~100배 처리량 향상 가능

FOR SHARE vs FOR UPDATE:
  FOR SHARE (S Lock): S Lock끼리 공존 → 읽기 처리량 높음
  FOR UPDATE (X Lock): 다른 모든 Lock 차단 → 처리량 낮음
  사용 기준: 읽기 후 수정 의도 → FOR UPDATE
             읽기 후 다른 수정 방지만 → FOR SHARE
```

---

## ⚖️ 트레이드오프

```
Pessimistic Lock (FOR UPDATE) vs Optimistic Lock (@Version):

FOR UPDATE (Pessimistic):
  ✅ 충돌이 실제 발생하기 전에 차단 → 재시도 없음
  ✅ 외부 시스템 호출 포함 흐름에 적합 (Lock이 유지되는 동안 보호)
  ❌ Lock 경합 → 처리량 저하
  ❌ Deadlock 가능성 (여러 FOR UPDATE 사용 시)

@Version (Optimistic):
  ✅ Lock 없음 → 높은 처리량
  ✅ Deadlock 없음
  ❌ 충돌 발생 시 재시도 필요 (OptimisticLockException)
  ❌ 재시도 중 다시 외부 호출이 있으면 복잡해짐

선택 기준:
  충돌 빈도 낮음 + 재시도 OK → @Version (Optimistic)
  충돌 빈도 높음 또는 재시도 불가 → FOR UPDATE (Pessimistic)
  외부 시스템 호출 포함 트랜잭션 → FOR UPDATE 또는 Saga 패턴

SKIP LOCKED 활용:
  ✅ 분산 워커가 중복 없이 작업 처리
  ✅ 대기 없음 → 높은 병렬성
  ❌ 처리 순서 보장 안 됨 (FIFO 필요 시 부적합)
  ❌ 실패한 작업의 재처리 로직 별도 필요
```

---

## 📌 핵심 정리

```
Consistent Read vs Current Read 핵심:

Consistent Read (일반 SELECT):
  버전: Read View 기준 스냅샷 (과거 가능)
  Lock: 없음
  사용: 단순 조회, 변경 의도 없을 때

Current Read (FOR UPDATE, FOR SHARE, DML):
  버전: Buffer Pool 최신 커밋 버전
  Lock: X Lock (FOR UPDATE, DML) 또는 S Lock (FOR SHARE)
  사용: 읽고 수정할 때, 동시 변경 차단 필요 시

혼용 금지 패턴:
  Consistent Read로 값 읽기 → 그 값 기반으로 UPDATE
  → 읽은 값과 실제 최신값이 다를 수 있음 → Lost Update

올바른 패턴:
  읽고 수정 → FOR UPDATE로 통일
  단순 산술 UPDATE → UPDATE SET col = col - N (Current Read 자동)

JPA Lock:
  @Lock(PESSIMISTIC_WRITE) → FOR UPDATE
  @Lock(PESSIMISTIC_READ)  → FOR SHARE
  @Version                 → Optimistic Lock (버전 CAS)

SKIP LOCKED / NOWAIT:
  SKIP LOCKED: Lock된 Row 건너뜀 → 분산 작업 큐에 적합
  NOWAIT:      Lock 있으면 즉시 에러 → 빠른 실패 패턴
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드의 문제점을 Consistent Read / Current Read 관점에서 분석하라.

```java
@Transactional
public boolean reserveSeat(Long seatId, Long userId) {
    Seat seat = seatRepository.findById(seatId).orElseThrow();
    
    if (!"AVAILABLE".equals(seat.getStatus())) {
        return false;  // 이미 예약됨
    }
    
    seat.setStatus("RESERVED");
    seat.setUserId(userId);
    seatRepository.save(seat);
    return true;
}
```

<details>
<summary>해설 보기</summary>

**TOCTOU(Time-of-Check-Time-of-Use) 취약점**입니다. Consistent Read와 Current Read 혼용 문제입니다.

**문제 시나리오**:
1. 사용자 A, B가 동시에 같은 seatId=1을 예약 시도
2. A: `findById` → Consistent Read → status="AVAILABLE" 확인 (Lock 없음)
3. B: `findById` → Consistent Read → status="AVAILABLE" 확인 (Lock 없음, A와 동시 가능)
4. A: `save` → UPDATE ... WHERE id=1 (Current Read, X Lock 획득) → status="RESERVED" 저장 + COMMIT
5. B: `save` → UPDATE ... WHERE id=1 (Current Read, X Lock 획득) → status="RESERVED" 저장 + COMMIT

결과: 같은 좌석을 두 명이 예약 성공. status="AVAILABLE" 체크가 의미 없어짐.

**올바른 해결책**:
```java
@Transactional
public boolean reserveSeat(Long seatId, Long userId) {
    // FOR UPDATE로 최신 상태 읽기 + Lock 획득
    Seat seat = seatRepository.findByIdWithLock(seatId)  // @Lock(PESSIMISTIC_WRITE)
        .orElseThrow();
    
    if (!"AVAILABLE".equals(seat.getStatus())) {
        return false;  // 이미 예약됨 (최신 상태 기준)
    }
    
    // B는 A의 X Lock이 해제될 때까지 여기서 대기
    // A가 COMMIT 후 B가 진행하면 status="RESERVED" 확인 → false 반환
    
    seat.setStatus("RESERVED");
    seat.setUserId(userId);
    seatRepository.save(seat);
    return true;
}
```

</details>

---

**Q2.** `FOR UPDATE SKIP LOCKED`를 사용하는 작업 큐에서 Task가 처리 중 실패(서버 재시작 등)했을 때 해당 Task는 영원히 잠기는가?

<details>
<summary>해설 보기</summary>

**영구적으로 잠기지 않습니다.** 이유는 Lock의 생명주기에 있습니다.

FOR UPDATE는 트랜잭션이 활성인 동안만 Lock을 유지합니다. 서버가 재시작되거나 연결이 끊기면:
1. MySQL 서버가 해당 Connection의 비정상 종료를 감지
2. 진행 중이던 트랜잭션을 자동으로 ROLLBACK
3. 트랜잭션 ROLLBACK → 모든 Lock 해제

그러나 **Task 상태는 "PROCESSING"으로 남아있는 문제**가 있습니다:

```java
// 문제 패턴:
@Transactional
public void processTask(Long taskId) {
    Task task = taskRepo.findByIdForUpdate(taskId);  // Lock + Current Read
    task.setStatus("PROCESSING");
    taskRepo.save(task);  // COMMIT (Lock은 해제됨!)
    
    externalApiCall();  // 실패 → 트랜잭션 밖이므로 ROLLBACK 없음
    
    task.setStatus("DONE");
    taskRepo.save(task);  // 여기까지 도달 못 하면 PROCESSING 상태로 남음
}
```

올바른 패턴 - 타임아웃 기반 재처리:
```java
// Task 테이블에 locked_at 컬럼 추가
// 주기적으로 locked_at이 오래된 PROCESSING Task를 PENDING으로 리셋
UPDATE tasks SET status='PENDING', locked_at=NULL
WHERE status='PROCESSING'
  AND locked_at < NOW() - INTERVAL 5 MINUTE;
```

</details>

---

**Q3.** JPA에서 `@Lock(PESSIMISTIC_WRITE)` 없이 Spring Retry로만 `OptimisticLockException`을 처리하는 패턴의 한계는?

<details>
<summary>해설 보기</summary>

Optimistic Lock (@Version) + Spring Retry 패턴의 한계:

**1. 재시도 중 외부 시스템 호출 문제**:
```java
@Retryable(ObjectOptimisticLockingFailureException.class)
@Transactional
public void processPayment(Long orderId) {
    Order order = orderRepo.findById(orderId);
    paymentGateway.charge(order.getAmount());  // 외부 HTTP 호출 (실제 결제)
    order.setStatus("PAID");
    orderRepo.save(order);
}
// 충돌 발생 → 재시도 → paymentGateway.charge()가 다시 호출됨!
// → 중복 결제 발생 가능
```

**2. 높은 충돌률에서 무한 재시도 위험**:
```java
// 10개 스레드가 동시에 같은 Row를 업데이트
// 첫 번째만 성공 → 9개 재시도 → 다시 1개 성공 → 8개 재시도...
// → Thundering Herd 현상 → 서버 부하 급증
```

**3. 재시도 횟수 초과 처리**:
```java
// maxAttempts = 3으로 설정 후 모두 실패하면 MaxAttemptsExceededException 발생
// → 이 경우 처리 실패, 사용자에게 명시적 에러 반환 필요
// → Pessimistic Lock이라면 이 상황 자체가 발생하지 않음
```

**결론**: Optimistic Lock은 충돌 빈도가 낮고 재시도가 부작용 없는 순수 DB 업데이트에 적합합니다. 외부 시스템 호출, 이메일 발송, 결제 처리가 포함되면 Pessimistic Lock (FOR UPDATE) 또는 Saga 패턴이 더 안전합니다.

</details>

---

<div align="center">

**[⬅️ Phantom Read](./05-phantom-read.md)** | **[홈으로 🏠](../README.md)** | **[다음: Redo Log ➡️](./07-redo-log-durability.md)**

</div>
