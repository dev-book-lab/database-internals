# ACID — 교과서 정의와 실제 구현 사이

---

## 🎯 핵심 질문

- Atomicity를 보장하는 것이 Undo Log인 이유는?
- Durability가 Redo Log로 구현되는 원리는?
- Consistency는 왜 DB 엔진이 아닌 애플리케이션 책임인가?
- Spring `@Transactional`은 ACID의 어느 부분을 담당하는가?
- `@Transactional(rollbackFor = Exception.class)`가 없으면 무슨 일이 생기는가?

---

## 🔍 왜 중요한가

```
"트랜잭션을 쓰면 ACID가 자동으로 보장된다"는 오해:

실제:
  A (Atomicity)   → Undo Log로 InnoDB가 보장
  I (Isolation)   → MVCC + Lock으로 InnoDB가 보장 (격리 수준 선택 필요)
  D (Durability)  → Redo Log + innodb_flush_log_at_trx_commit으로 보장
  C (Consistency) → 애플리케이션 + 제약 조건으로 보장 (DB만으로는 불완전)

Spring @Transactional의 역할:
  트랜잭션 시작/커밋/롤백 관리 (AoP 프록시)
  → InnoDB에 BEGIN / COMMIT / ROLLBACK 신호 전달
  → ACID 보장은 결국 InnoDB가 담당
  → @Transactional 자체가 ACID를 구현하는 것이 아님
```

---

## 🔬 내부 동작 원리

### 1. Atomicity — Undo Log로 구현

```
원칙: "트랜잭션의 모든 변경은 전부 반영되거나 전혀 반영되지 않아야 한다"

구현 방식:
  모든 INSERT / UPDATE / DELETE 실행 전
  → Undo Log에 이전 상태(Before Image) 기록
  → Buffer Pool에 변경 적용 (메모리)

  정상 COMMIT:
    Undo Log 보관 (MVCC용, 나중에 Purge)
    변경사항이 영구화

  ROLLBACK 또는 크래시 복구:
    Undo Log를 역방향으로 적용 (undo 연산)
    → 변경 전 상태로 복원

Undo Log 구조:
  INSERT undo: 삽입된 PK만 기록 (롤백 시 DELETE)
  UPDATE undo: 변경 전 컬럼 값 기록 (롤백 시 이전 값으로 복원)
  DELETE undo: 삭제 전 전체 Row 기록 (롤백 시 재삽입)

Spring에서의 의미:
  @Transactional 메서드에서 예외 발생
  → AOP 프록시가 ROLLBACK 호출
  → InnoDB가 Undo Log 적용 → 원자적 복원

주의: CheckedException은 기본적으로 롤백 안 됨!
  @Transactional(rollbackFor = Exception.class) 설정 필요
```

### 2. Durability — Redo Log + WAL

```
원칙: "COMMIT된 트랜잭션은 시스템 장애 후에도 반영되어 있어야 한다"

왜 단순히 "디스크에 쓰면" 안 되는가:
  Buffer Pool의 Dirty Page를 즉시 디스크에 쓰면:
    Random I/O → 매우 느림
    COMMIT 속도 = 디스크 쓰기 속도로 제한
    
WAL (Write-Ahead Logging) 해결책:
  Redo Log = Sequential I/O (디스크 끝에 Append)
  Buffer Pool 변경 전에 Redo Log 먼저 기록 (Log-Before-Data 원칙)
  
  COMMIT 시:
    Redo Log Buffer → Redo Log File (fsync)
    ← 이 한 번의 Sequential Write로 내구성 보장
    Buffer Pool Dirty Pages는 나중에 비동기로 디스크에 반영

  장애 발생 후 재시작:
    Redo Log 재생 → Buffer Pool 상태 복원
    미완료 트랜잭션 → Undo Log로 롤백

innodb_flush_log_at_trx_commit:
  1 (기본): COMMIT마다 fsync → 완전한 내구성, IOPS 높음
  2: COMMIT마다 write() (OS 버퍼), 1초마다 fsync → MySQL 크래시 안전
  0: 1초마다 write + fsync → OS 크래시 시 1초 데이터 손실 가능
```

### 3. Isolation — MVCC + Lock

```
원칙: "동시 실행되는 트랜잭션은 서로 간섭하지 않아야 한다"

InnoDB 구현:
  읽기 → MVCC (Undo Log 버전 체인)
    → Lock 없이 일관된 스냅샷 읽기
    → 다른 트랜잭션의 쓰기에 영향 없음
  
  쓰기 → Row-Level Lock
    → 동시 쓰기 충돌 방지
    → Deadlock 감지 후 한쪽 롤백

격리 수준 (Isolation Level):
  완벽한 격리(SERIALIZABLE) vs 성능(READ_COMMITTED) 선택
  → 기본값: REPEATABLE_READ (InnoDB의 특수 Gap Lock으로 Phantom Read 방지)

Spring @Transactional(isolation = ...):
  → Hibernate가 SET SESSION TRANSACTION ISOLATION LEVEL 실행
  → 메서드 실행 전 격리 수준 변경, 완료 후 복원
  → 연결 풀에서 이미 열린 연결의 격리 수준을 바꾸는 것 주의
```

### 4. Consistency — 애플리케이션 책임

```
원칙: "트랜잭션 전후로 DB가 일관된 상태를 유지해야 한다"

DB 엔진이 보장하는 것:
  PRIMARY KEY 유일성 (중복 불가)
  FOREIGN KEY 참조 무결성 (orphan 방지)
  NOT NULL 제약
  UNIQUE 제약
  CHECK 제약 (MySQL 8.0.16+)

DB 엔진이 보장하지 못하는 것:
  비즈니스 규칙: "주문 금액 >= 최소 주문 금액"
  도메인 무결성: "재고가 0 미만이 되면 안 된다"
  관계 무결성: "사용자가 BLOCKED면 주문할 수 없다"

→ 이것은 애플리케이션이 @Transactional 내에서 검증해야 함

실제 사례:
  @Transactional
  public void placeOrder(Long userId, Long productId, int quantity) {
    User user = userRepository.findById(userId);
    if (user.isBlocked()) throw new BlockedUserException(); // 비즈니스 규칙
    
    Product product = productRepository.findById(productId);
    if (product.getStock() < quantity) throw new InsufficientStockException(); // 비즈니스 규칙
    
    product.decreaseStock(quantity);  // DB에 반영
    orderRepository.save(new Order(user, product, quantity));
    // → A: 두 작업 모두 성공 or 모두 실패 (Atomicity)
    // C: 비즈니스 규칙 검증은 위 코드가 담당
  }
```

---

## 💻 실전 실험

### 실험 1: Atomicity 확인

```sql
-- 트랜잭션 중 오류 발생 시 롤백 확인
CREATE TABLE accounts (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    name    VARCHAR(50),
    balance DECIMAL(15,2) NOT NULL DEFAULT 0
) ENGINE=InnoDB;

INSERT INTO accounts (name, balance) VALUES ('Alice', 10000), ('Bob', 5000);

-- 트랜잭션 시작
START TRANSACTION;
UPDATE accounts SET balance = balance - 3000 WHERE name = 'Alice';
-- Alice: 7000

-- 오류 시뮬레이션 (Bob은 업데이트 안 함)
ROLLBACK;  -- 또는 오류로 인한 자동 롤백

SELECT * FROM accounts;
-- Alice: 10000 (롤백됨), Bob: 5000
-- → Atomicity 보장: Alice 차감만 부분 적용되지 않음
```

### 실험 2: Spring @Transactional 롤백 규칙

```java
// RuntimeException → 자동 롤백
@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    accountRepository.decreaseBalance(fromId, amount);
    accountRepository.increaseBalance(toId, amount);
    
    if (amount.compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("Amount must be positive"); // RuntimeException
        // → 자동 롤백 (AOP 프록시가 catch 후 ROLLBACK 호출)
    }
}

// CheckedException → 기본 롤백 안 됨!
@Transactional  // ❌ rollbackFor 없음
public void transferWithCheck(Long fromId, Long toId, BigDecimal amount) 
    throws InsufficientBalanceException {
    accountRepository.decreaseBalance(fromId, amount);
    // 위 작업은 수행됨
    
    throw new InsufficientBalanceException("잔액 부족"); // CheckedException
    // → COMMIT 됨! decrease만 반영, increase 없음 → 데이터 불일치!
}

// 올바른 방법:
@Transactional(rollbackFor = Exception.class)  // ✅
public void transferWithCheckFixed(Long fromId, Long toId, BigDecimal amount) 
    throws InsufficientBalanceException {
    // ...
}
```

---

## 📌 핵심 정리

```
ACID 구현 매핑:

A (Atomicity):
  구현: Undo Log (Before Image)
  Spring: @Transactional + 예외 → ROLLBACK 호출
  주의: CheckedException은 rollbackFor = Exception.class 필요

C (Consistency):
  DB: 제약조건 (PK, FK, NOT NULL, UNIQUE, CHECK)
  App: 비즈니스 규칙 검증 (@Transactional 내에서)
  오해: "트랜잭션이 알아서 일관성 보장" → 아님

I (Isolation):
  구현: MVCC (읽기) + Row Lock (쓰기)
  선택: 격리 수준 (Read Uncommitted ~ Serializable)
  기본: REPEATABLE_READ

D (Durability):
  구현: Redo Log + WAL + fsync
  설정: innodb_flush_log_at_trx_commit=1 (완전 내구성)
  트레이드오프: 설정값 낮추면 성능↑, 내구성↓
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional` 없이 JPA `save()`를 두 번 호출하면 Atomicity가 보장되는가?

<details>
<summary>해설 보기</summary>

**보장되지 않습니다.** `@Transactional` 없이 각 `save()`는 별도의 트랜잭션으로 실행됩니다 (Spring Data JPA의 기본 구현에서 `save()`는 내부적으로 트랜잭션을 가지지만 호출자의 트랜잭션과 합쳐지지 않음).

첫 번째 `save()`가 성공하고 두 번째 `save()`에서 예외가 발생하면, 첫 번째 결과는 이미 커밋된 상태입니다. Atomicity를 보장하려면 두 `save()` 호출을 하나의 `@Transactional` 메서드 안에 넣어야 합니다.

`@Transactional(propagation = REQUIRED)` (기본값)은 기존 트랜잭션이 있으면 합류, 없으면 새로 생성합니다. 이를 통해 여러 Repository 호출을 원자적으로 묶을 수 있습니다.

</details>

**Q2.** `innodb_flush_log_at_trx_commit = 2`는 어떤 장애 시나리오에서 데이터 손실이 발생하는가?

<details>
<summary>해설 보기</summary>

설정 2는 COMMIT 시 Redo Log를 OS 버퍼에 write하고, 1초마다 OS 버퍼를 fsync합니다. **MySQL 프로세스 크래시**에서는 OS 버퍼가 남아있어 안전합니다.

데이터 손실 발생 시나리오: **OS 크래시 또는 전원 장애**. fsync 주기(1초) 사이에 OS가 다운되면 OS 버퍼의 Redo Log가 손실됩니다. 이 1초 동안 COMMIT된 트랜잭션의 데이터가 복구 불가능하게 유실됩니다.

금융 서비스처럼 완전한 내구성이 필요하면 설정 1을 사용해야 합니다. 내부 분석 DB, 로그 수집 등 일부 손실이 허용되는 경우 설정 2로 쓰기 성능을 크게 향상시킬 수 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: MVCC 내부 원리 ➡️](./02-mvcc-internals.md)**

</div>
