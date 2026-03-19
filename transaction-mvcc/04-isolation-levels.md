# 트랜잭션 격리 수준 4가지 — 구현 원리까지

---

## 🎯 핵심 질문

- 격리 수준 4가지가 Read View를 언제 생성하느냐의 차이로 설명될 수 있는가?
- READ_COMMITTED에서 Non-Repeatable Read가 발생하는 정확한 메커니즘은?
- REPEATABLE_READ는 Phantom Read를 완전히 방지하는가?
- JPA `@Transactional(isolation = READ_COMMITTED)`는 실제로 어떻게 동작하는가?
- SERIALIZABLE은 어떻게 구현되고 성능 비용은 얼마나 되는가?

---

## 🔬 내부 동작 원리

### 1. 격리 수준과 발생 가능한 이상 현상

```
격리 수준          Dirty Read  Non-Repeatable Read  Phantom Read
─────────────────────────────────────────────────────────────────
READ UNCOMMITTED      가능            가능               가능
READ COMMITTED        불가            가능               가능
REPEATABLE READ       불가            불가         거의 불가(InnoDB)
SERIALIZABLE          불가            불가               불가

MySQL InnoDB 기본값: REPEATABLE READ
Spring Boot 기본값: DB 기본값 사용 (= REPEATABLE READ)

핵심 차이: Read View를 언제 생성하는가
  READ UNCOMMITTED: Read View 없음 (Lock도 없이 최신 버전 읽기)
  READ COMMITTED:   각 SELECT마다 새로운 Read View 생성
  REPEATABLE READ:  트랜잭션 첫 번째 읽기 시 Read View 1회 생성
  SERIALIZABLE:     Read View + 모든 읽기에 Shared Lock
```

### 2. READ UNCOMMITTED — Dirty Read

```sql
-- Dirty Read: 아직 COMMIT되지 않은 데이터를 읽는 것

-- 세션 1
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;

-- 세션 2
START TRANSACTION;
UPDATE accounts SET balance = -99999 WHERE id = 1;
-- 아직 COMMIT 안 함!

-- 세션 1
SELECT balance FROM accounts WHERE id = 1;
-- Read Uncommitted: -99999 읽음 (Dirty Read!)
-- 세션 2가 ROLLBACK하면 이 값은 존재하지 않았던 것

-- 세션 2
ROLLBACK;
-- 세션 1이 읽은 -99999는 이제 없는 값

READ UNCOMMITTED 구현:
  Read View 없음
  Undo Log 버전 체인 탐색 없음
  단순히 Buffer Pool의 최신 버전 직접 읽기
  → 가장 빠름, 가장 위험
  → 실무 사용: 거의 없음 (데이터 정확성 불필요한 경우만)
```

### 3. READ COMMITTED — Non-Repeatable Read

```sql
-- Non-Repeatable Read: 같은 트랜잭션 내에서 같은 Row를 두 번 읽었을 때 다른 값

-- 세션 1
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 10000

-- 세션 2
UPDATE accounts SET balance = 7000 WHERE id = 1;
COMMIT;

-- 세션 1 (계속)
SELECT balance FROM accounts WHERE id = 1;  -- 7000 (다른 값!)
COMMIT;

READ COMMITTED 구현:
  각 SELECT마다 새로운 Read View 생성
  두 번째 SELECT의 Read View: 세션 2의 트랜잭션이 이미 커밋됨
  → 7000 보임

  첫 번째 SELECT의 Read View: 세션 2가 아직 활성
  → 10000 보임 (Undo Log의 이전 버전)

실무 사용 사례:
  MySQL 기본값보다 낮은 격리가 필요한 경우:
    - 쓰기 집약적 시스템에서 Lock 경합 감소
    - 보고서 쿼리 (최신 데이터가 중요, 일관성은 덜 중요)
    - 단일 SELECT 위주 (같은 트랜잭션에서 반복 읽기 없음)
  
  Spring 설정:
    @Transactional(isolation = Isolation.READ_COMMITTED)
```

### 4. REPEATABLE READ — InnoDB 기본값

```sql
-- 같은 트랜잭션 내 반복 읽기 = 항상 같은 결과 보장

-- 세션 1
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 10000 (Read View 생성)

-- 세션 2
UPDATE accounts SET balance = 7000 WHERE id = 1;
COMMIT;

-- 세션 1 (계속)
SELECT balance FROM accounts WHERE id = 1;  -- 여전히 10000!
-- Read View가 고정됨 → 세션 2의 변경은 Read View 이후 → 보이지 않음

COMMIT;
-- 트랜잭션 종료 후 새 트랜잭션
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 이제 7000 (새 Read View)

REPEATABLE READ 구현:
  트랜잭션의 첫 SELECT 또는 DML 시 Read View 생성
  (정확히는 "첫 번째 일관된 읽기 시")
  이후 모든 SELECT는 동일한 Read View 사용

주의 - 쓰기에는 다르게 동작:
  UPDATE 실행 시: MVCC 버전이 아닌 최신 버전(Current Read)에 Lock
  → 세션 2가 변경하고 COMMIT한 7000에 대해 업데이트 가능!
  → MVCC 읽기(Consistent Read)와 쓰기(Current Read)가 다른 버전 참조
```

### 5. SERIALIZABLE — 완전한 직렬화

```sql
-- 모든 읽기에 Shared Lock → 완전한 직렬 실행 효과

SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 세션 1
START TRANSACTION;
SELECT * FROM accounts WHERE balance > 5000;
-- 모든 결과 Row에 S Lock 획득!

-- 세션 2
INSERT INTO accounts (name, balance) VALUES ('Charlie', 8000);
-- balance > 5000인 새 Row → Gap Lock에 의해 차단!
-- 세션 1이 COMMIT 할 때까지 대기

SERIALIZABLE 구현:
  모든 SELECT → SELECT ... FOR SHARE와 동일
  → S Lock (Shared Lock) 획득
  → 다른 트랜잭션의 해당 Row 변경 차단
  → Phantom Read 완전 방지 (Lock으로 새 삽입도 차단)

성능 비용:
  Lock 경합 대폭 증가 → Deadlock 가능성 증가 → 처리량 감소
  OLTP 환경에서 거의 사용하지 않음
  사용 사례: 금융 계좌 이체처럼 절대적 일관성 필요한 경우 (단일 쿼리 수준)
```

### 6. JPA @Transactional과 격리 수준

```java
// Spring의 격리 수준 설정
@Transactional(isolation = Isolation.READ_COMMITTED)
public OrderDto getOrderSummary(Long orderId) {
    // 이 메서드 실행 전: SET TRANSACTION ISOLATION LEVEL READ COMMITTED
    // 이 메서드 완료 후: 이전 격리 수준으로 복원
}

// 주의: 이미 열려있는 Connection을 재사용할 때
// HikariCP 등 Connection Pool에서 Connection을 꺼낼 때
// 이전 트랜잭션이 격리 수준을 변경했다면?
// → Spring은 트랜잭션 시작/완료 시 격리 수준 설정/복원을 처리
// → 하지만 Connection Pool의 Connection 자체는 재사용됨

// 실제 동작:
// HikariCP → Connection 획득
// Spring: SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED
// @Transactional 메서드 실행
// Spring: COMMIT
// Spring: SET SESSION TRANSACTION ISOLATION LEVEL [이전값]  ← 복원
// HikariCP → Connection 반환

// 격리 수준 설정 확인
@Transactional
public void checkIsolation() {
    String level = jdbcTemplate.queryForObject(
        "SELECT @@transaction_isolation", String.class);
    log.info("Current isolation: {}", level);
}

// 전역 설정 (application.properties):
// spring.datasource.hikari.transaction-isolation=TRANSACTION_READ_COMMITTED
// → 모든 Connection의 기본 격리 수준 변경
```

---

## 💻 실전 실험

### 실험: 4가지 격리 수준 비교

```sql
-- 세션 1: 격리 수준별 동작 비교
-- (세션 2에서 별도로 UPDATE + COMMIT)

-- READ UNCOMMITTED (Dirty Read 가능)
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- T1: 세션 2 UPDATE 전
-- (세션 2에서 UPDATE balance=7000, 아직 COMMIT 안 함)
SELECT balance FROM accounts WHERE id = 1;  -- T2: 7000 보임 (Dirty Read!)
COMMIT;

-- READ COMMITTED (Non-Repeatable Read 가능)
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- T1: 10000
-- (세션 2에서 UPDATE balance=7000, COMMIT)
SELECT balance FROM accounts WHERE id = 1;  -- T2: 7000 (Non-Repeatable)
COMMIT;

-- REPEATABLE READ (Non-Repeatable Read 불가)
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- T1: 10000 (Read View 생성)
-- (세션 2에서 UPDATE balance=7000, COMMIT)
SELECT balance FROM accounts WHERE id = 1;  -- T2: 10000 (Read View 고정)
COMMIT;

-- SERIALIZABLE (모든 이상 현상 불가, Lock으로)
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- S Lock 획득
-- (세션 2에서 UPDATE balance=7000 → 이 시점에 차단!)
-- 세션 1 COMMIT 후에야 세션 2 UPDATE 완료
COMMIT;
```

---

## 📌 핵심 정리

```
격리 수준 핵심:

Read View 생성 시점:
  READ UNCOMMITTED: Read View 없음 (최신 버전 직접 읽기)
  READ COMMITTED:   각 SELECT마다 새 Read View
  REPEATABLE READ:  트랜잭션 첫 읽기 시 1회
  SERIALIZABLE:     Read View + 모든 읽기에 S Lock

이상 현상:
  Dirty Read:           미커밋 데이터 읽기 → RC 이상에서 방지
  Non-Repeatable Read:  동일 Row 두 번 읽어 다른 값 → RR 이상에서 방지
  Phantom Read:         범위 쿼리에서 다른 결과 → SERIALIZABLE에서 완전 방지
                        InnoDB RR: Gap Lock으로 대부분 방지

성능 순서 (빠름→느림):
  READ UNCOMMITTED > READ COMMITTED > REPEATABLE READ > SERIALIZABLE

실무 선택:
  대부분의 OLTP: REPEATABLE READ (MySQL 기본)
  읽기 일관성 포기 + 성능 우선: READ COMMITTED
  완전한 직렬화 (드문 경우): SERIALIZABLE
```

---

## 🤔 생각해볼 문제

**Q1.** REPEATABLE READ에서 자신이 UPDATE한 Row를 다시 SELECT하면 어떤 값이 보이는가?

<details>
<summary>해설 보기</summary>

**자신의 변경 사항이 보입니다.** Read View 가시성 판단에서 `row_trx_id == m_creator_trx_id` (내 트랜잭션이 만든 버전)이면 항상 보입니다.

```sql
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 10000
UPDATE accounts SET balance = 8000 WHERE id = 1;
SELECT balance FROM accounts WHERE id = 1;  -- 8000 (내 변경 보임)
ROLLBACK;
```

Read View는 "다른 트랜잭션의 변경"을 가시성 판단하는 것이지, "자신의 변경"을 숨기지 않습니다. 자신의 트랜잭션 내에서는 항상 최신 상태를 볼 수 있습니다.

</details>

**Q2.** Spring에서 `@Transactional` 없이 여러 Repository 호출을 하면 각 호출마다 별도 트랜잭션이 생기는가? 이것이 REPEATABLE READ와 어떤 관계가 있는가?

<details>
<summary>해설 보기</summary>

**각 Repository 호출마다 별도의 Connection이 사용되고 각각 독립적인 트랜잭션으로 실행됩니다** (Spring Data JPA의 SimpleJpaRepository에 `@Transactional`이 붙어 있으므로 각 메서드가 독립 트랜잭션).

REPEATABLE READ와의 관계: 각 호출이 별도 트랜잭션이므로 각자 독립적인 Read View를 가집니다. 

```java
// @Transactional 없음
public void processOrder() {
    Order order = orderRepository.findById(1L);   // 트랜잭션 A, Read View A
    // 이 사이에 다른 스레드가 order를 수정하고 커밋
    User user = userRepository.findById(1L);       // 트랜잭션 B, Read View B
    // order와 user가 서로 다른 시점의 스냅샷!
}

// @Transactional 있음
@Transactional
public void processOrderConsistent() {
    Order order = orderRepository.findById(1L);   // 트랜잭션 A, Read View A (생성)
    User user = userRepository.findById(1L);       // 트랜잭션 A, Read View A (재사용)
    // order와 user가 같은 시점의 스냅샷 → 일관성 보장
}
```

REPEATABLE READ는 **같은 트랜잭션 내**에서만 일관성을 보장합니다. `@Transactional` 없이 여러 독립 트랜잭션을 사용하면 REPEATABLE READ의 이점이 없습니다.

</details>

---

<div align="center">

**[⬅️ Undo Log](./03-undo-log.md)** | **[홈으로 🏠](../README.md)** | **[다음: Phantom Read ➡️](./05-phantom-read.md)**

</div>
