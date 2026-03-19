# ACID — 교과서 정의와 실제 구현 사이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Atomicity를 보장하는 메커니즘이 "Undo Log"인 이유는 무엇인가?
- Durability가 "Dirty Page를 즉시 디스크에 쓰는 것"이 아닌 Redo Log로 구현되는 이유는?
- Consistency는 왜 DB 엔진이 아닌 애플리케이션의 책임인가?
- Spring `@Transactional`은 ACID 네 가지 중 어느 부분을 직접 담당하는가?
- `@Transactional(rollbackFor = Exception.class)`를 빠뜨리면 어떤 데이터 불일치가 생기는가?
- `innodb_flush_log_at_trx_commit = 2`는 어떤 장애 시나리오에서 데이터를 잃는가?

---

## 🔍 왜 이 개념이 중요한가

### "@Transactional을 쓰면 ACID가 자동으로 보장된다"는 오해

```
흔한 믿음:
  @Transactional을 붙이면
  → 원자성, 일관성, 격리성, 내구성이 모두 해결된다

실제 담당 구조:
  A (Atomicity)   → InnoDB Undo Log가 보장
  C (Consistency) → 애플리케이션 코드 + DB 제약조건이 보장
  I (Isolation)   → InnoDB MVCC + Lock이 보장 (격리 수준 선택 필요)
  D (Durability)  → InnoDB Redo Log + innodb_flush_log_at_trx_commit이 보장

@Transactional의 실제 역할:
  AOP 프록시로 메서드 진입 시 BEGIN 실행
  메서드 정상 종료 시 COMMIT 실행
  RuntimeException 발생 시 ROLLBACK 실행
  → InnoDB에 신호를 전달하는 것, ACID 자체를 구현하는 것이 아님

왜 이 구분이 중요한가:
  @Transactional만 붙이고 CheckedException을 던지면 COMMIT됨!
  격리 수준을 지정하지 않으면 DB 기본값(REPEATABLE_READ)이 적용됨
  innodb_flush_log_at_trx_commit=0이면 @Transactional과 무관하게 내구성 없음
  → 각 보장이 어디서 오는지 알아야 올바르게 설정할 수 있다
```

---

## 😱 잘못된 이해

### Before: 롤백과 내구성에 대한 두 가지 오해

```java
// 오해 1: CheckedException도 자동 롤백된다

@Transactional  // rollbackFor 없음
public void transfer(Long fromId, Long toId, BigDecimal amount)
        throws InsufficientBalanceException {

    accountRepository.decreaseBalance(fromId, amount);
    // ↑ 이 시점에 DB에서 차감이 발생함 (Buffer Pool에 반영)

    if (amount.compareTo(new BigDecimal("1000000")) > 0) {
        throw new InsufficientBalanceException("한도 초과");
        // ← CheckedException → ROLLBACK이 아니라 COMMIT 됨!
        // → fromId의 잔액만 줄고, toId는 증가하지 않음
        // → 데이터 불일치 발생
    }

    accountRepository.increaseBalance(toId, amount);
}
// 결과: InsufficientBalanceException이 발생해도
//       첫 번째 decreaseBalance는 COMMIT되어 돈이 사라짐
```

```sql
-- 오해 2: COMMIT하면 즉시 디스크에 안전하게 저장된다

-- innodb_flush_log_at_trx_commit = 0 설정 상태에서:
START TRANSACTION;
INSERT INTO orders (user_id, amount) VALUES (1, 50000);
COMMIT;
-- ↑ "완료"라고 응답받음

-- 하지만:
-- Redo Log Buffer → OS 버퍼 → 디스크 의 과정이
-- 1초마다 한 번씩 일어남
-- OS가 이 1초 안에 다운되면 이 INSERT는 영구히 사라짐!
```

---

## ✨ 올바른 이해

### After: ACID 각각의 구현 메커니즘을 분리해서 이해하기

```
A (Atomicity) = Undo Log의 역방향 적용:

  INSERT/UPDATE/DELETE 실행 전:
    Undo Log에 "이전 상태(Before Image)" 기록
    → INSERT: 삽입된 PK 기록 (롤백 시 DELETE)
    → UPDATE: 변경 전 컬럼 값 기록 (롤백 시 이전 값으로 복원)
    → DELETE: 삭제 전 Row 전체 기록 (롤백 시 재삽입)

  ROLLBACK 발생 시:
    Undo Log를 역방향으로 적용 (생성 역순)
    → 마지막 변경부터 첫 번째 변경까지 되돌림
    → 완전히 "없었던 것처럼" 복원

D (Durability) = Sequential Write 먼저, Random Write 나중:

  Buffer Pool의 Dirty Page를 즉시 디스크에 쓰면?
    Random I/O → HDD 기준 수ms × COMMIT 횟수 → 병목
    
  WAL (Write-Ahead Logging) 해결책:
    Redo Log = 변경 내용의 순차 기록 (Sequential I/O)
    COMMIT 시 Redo Log를 fsync → 내구성 보장
    Buffer Pool Dirty Page 플러시는 백그라운드에서 비동기 처리
    → COMMIT 속도 = Sequential Write 속도 (Random의 수십 배 빠름)

C (Consistency) = DB 제약 + 비즈니스 로직:

  DB가 보장: PK 유일성, FK 참조 무결성, NOT NULL, UNIQUE, CHECK
  App이 보장: "재고가 0 미만이 되면 안 된다", "BLOCKED 유저는 주문 불가"
  → 비즈니스 규칙은 @Transactional 내의 코드에서 검증해야 함
```

---

## 🔬 내부 동작 원리

### 1. Atomicity — Undo Log의 물리적 구조

```
Undo Log 기록 시점:
  변경이 Buffer Pool에 적용되기 직전에 Undo Log 기록

Undo Log Tablespace:
  MySQL 8.0: 별도 파일 (undo_001, undo_002)
  각 트랜잭션의 변경 이력을 Rollback Segment에 저장

Insert Undo vs Update Undo의 차이:
  INSERT undo:
    PK 값만 기록 → 롤백 시 그 PK를 DELETE
    COMMIT 직후 삭제 가능 (다른 트랜잭션이 볼 필요 없음)

  UPDATE / DELETE undo:
    변경 전 컬럼 값 전체 기록
    MVCC에서 이전 버전 읽기에 재사용됨
    → 다른 트랜잭션의 Read View가 사라질 때까지 보관

롤백 실행 순서 (역방향):
  트랜잭션이 A → B → C 순서로 변경했다면
  롤백은 C → B → A 순서로 Undo 적용
  → 생성 역순으로 각 변경을 취소

부분 롤백 (SAVEPOINT):
  SAVEPOINT sp1;   ← 여기까지는 유지
  변경 D, E, F
  ROLLBACK TO SAVEPOINT sp1;  ← D, E, F만 Undo 적용
  → A, B, C는 그대로
```

### 2. Durability — WAL과 Redo Log 플러시

```
COMMIT 내부 실행 순서:

  ① 트랜잭션 중 모든 변경:
       변경 내용 → Redo Log Buffer (메모리, 기본 16MB)
       동시에 → Buffer Pool에 Dirty Page 생성

  ② COMMIT 호출:
       Redo Log Buffer → Redo Log File (fsync, Sequential Write)
       ← 이 시점에 내구성 완전 보장
       클라이언트에게 "완료" 응답 전송

  ③ 이후 백그라운드:
       Buffer Pool Dirty Pages → 데이터 파일 (비동기 Random Write)
       Checkpoint 진행 → Redo Log 재활용 공간 확보

innodb_flush_log_at_trx_commit 상세:
  값 1 (기본): ② 단계에서 fsync 실행
               → 완전 내구성, IOPS 높음
  값 2:        ② 단계에서 write()만 실행 (OS 버퍼까지)
               1초마다 백그라운드에서 fsync
               → MySQL 크래시 안전, OS 크래시 시 최대 1초 손실
  값 0:        ② 단계에서 아무것도 안 함
               1초마다 write() + fsync
               → MySQL 크래시 시에도 최대 1초 손실
               → 프로덕션 절대 금지

Redo Log File (순환 파일):
  ib_logfile0, ib_logfile1 (MySQL 8.0.30 이전)
  innodb_redo_log_capacity (MySQL 8.0.30+, 단일 설정)
  순환 구조: Write Point가 Checkpoint를 따라잡으면 강제 Checkpoint 발생
```

### 3. Isolation — MVCC + Lock의 분업

```
읽기와 쓰기에 다른 메커니즘 사용:

일반 SELECT (Consistent Read):
  MVCC 스냅샷(Read View)으로 처리
  Lock 없음 → 다른 트랜잭션의 쓰기에 차단받지 않음
  → "읽기가 쓰기를 차단하지 않는다"의 기술적 근거

쓰기 (INSERT / UPDATE / DELETE):
  Row-Level Exclusive Lock 획득
  → 같은 Row를 수정하려는 다른 트랜잭션 차단
  → Deadlock 감지 후 비용 낮은 트랜잭션 롤백

격리 수준 = "Read View를 언제 생성하는가":
  READ UNCOMMITTED: Read View 없음 (Dirty Read 가능)
  READ COMMITTED:   각 SELECT마다 새 Read View 생성
  REPEATABLE READ:  트랜잭션 첫 읽기 시 1회 생성 후 고정 (기본값)
  SERIALIZABLE:     Read View + 모든 읽기에 S Lock

Spring @Transactional(isolation = READ_COMMITTED):
  메서드 진입 시: SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED
  메서드 종료 시: 이전 격리 수준으로 복원
  → HikariCP Connection Pool에서 Connection을 꺼낼 때 설정 변경
  → 반드시 완료 후 복원되어야 Pool이 오염되지 않음
```

### 4. Consistency — 애플리케이션이 책임지는 영역

```
DB 엔진이 보장하는 제약:
  PRIMARY KEY: 중복 불가
  FOREIGN KEY: 부모 Row 없이 자식 Row 삽입 불가
  NOT NULL: NULL 삽입 불가
  UNIQUE: 유니크 위반 불가
  CHECK (MySQL 8.0.16+): "CHECK (age >= 0)"

DB 엔진이 보장하지 못하는 비즈니스 규칙:
  "재고가 0 미만이 되면 안 된다"
  "BLOCKED 사용자는 주문할 수 없다"
  "주문 금액은 최소 주문금액 이상이어야 한다"
  "동시에 같은 좌석을 두 사람이 예약할 수 없다"

→ 이 규칙들은 @Transactional 메서드 내의 비즈니스 로직이 검증해야 함
→ 트랜잭션이 보장하는 것은 "이 검증 + 변경이 원자적으로 실행됨"이지
  "비즈니스 규칙 자체를 자동으로 지킴"이 아님

실제 구현 패턴:
  @Transactional
  public void placeOrder(Long userId, Long productId, int qty) {
    User user = userRepo.findById(userId);
    if (user.isBlocked()) throw new BlockedUserException();  // C 보장

    Product product = productRepo.findByIdWithLock(productId);  // Pessimistic Lock
    if (product.getStock() < qty) throw new InsufficientStockException(); // C 보장

    product.decreaseStock(qty);   // A 보장 (Undo Log)
    orderRepo.save(new Order(...)); // A 보장
    // COMMIT → D 보장 (Redo Log)
  }
```

### 5. @Transactional 롤백 규칙의 내부 원리

```
Spring AOP 프록시 동작:

정상 실행:
  [프록시 진입] → BEGIN
  → [실제 메서드 실행]
  → [프록시 종료] → COMMIT

RuntimeException (unchecked):
  [프록시 진입] → BEGIN
  → [실제 메서드 실행] → RuntimeException 발생
  → [프록시 catch] → ROLLBACK
  → 예외를 다시 throw

CheckedException (checked):
  [프록시 진입] → BEGIN
  → [실제 메서드 실행] → CheckedException 발생
  → [프록시 catch] → COMMIT (!)  ← 기본 동작
  → 예외를 다시 throw

왜 CheckedException은 롤백 안 되는가:
  설계 의도: CheckedException = "처리 가능한 예외"
  → 트랜잭션을 유지하면서 예외를 처리할 수 있는 케이스가 있음
  → 예: 락 타임아웃 → 재시도, 특정 상태 전환 실패 → 다른 경로

rollbackFor = Exception.class 효과:
  모든 Throwable의 서브클래스에서 롤백
  → Error 포함 (OutOfMemoryError 등)
  
실무 권장:
  @Transactional(rollbackFor = Exception.class)
  → 비즈니스 로직에서 CheckedException을 사용한다면 필수
  또는 RuntimeException으로 감싸서 throw:
  throw new BusinessException("원인", checkedException);
```

---

## 💻 실전 실험

### 실험 1: Atomicity — Undo Log 역방향 적용 확인

```sql
CREATE TABLE accounts (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    name    VARCHAR(50) NOT NULL,
    balance DECIMAL(15,2) NOT NULL DEFAULT 0,
    CHECK (balance >= 0)
) ENGINE=InnoDB;

INSERT INTO accounts (name, balance) VALUES ('Alice', 10000), ('Bob', 5000);

-- 부분 실패 시 전체 롤백 확인
START TRANSACTION;
UPDATE accounts SET balance = balance - 3000 WHERE name = 'Alice';
SELECT balance FROM accounts WHERE name = 'Alice';  -- 7000 확인
-- 여기서 의도적 롤백 (오류 시뮬레이션)
ROLLBACK;

SELECT * FROM accounts;
-- Alice: 10000 (완전 복원), Bob: 5000 (변경 없음)
-- → Undo Log가 Alice 차감을 역방향으로 적용

-- Undo Log 누적 확인
SHOW ENGINE INNODB STATUS\G
-- "History list length: N" 값 확인 (롤백 전/후 비교)
```

### 실험 2: Durability — flush 설정별 COMMIT 속도 비교

```sql
-- 설정 1 기준 (완전 내구성)
SET GLOBAL innodb_flush_log_at_trx_commit = 1;

-- 1만 건 INSERT 속도 측정
CREATE TABLE bench_redo (id BIGINT AUTO_INCREMENT PRIMARY KEY, data VARCHAR(100));

SET @start = NOW(6);
INSERT INTO bench_redo (data)
SELECT REPEAT('x', 100)
FROM information_schema.columns a
CROSS JOIN information_schema.columns b
LIMIT 10000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000 AS ms_setting1;

-- 설정 2로 변경 (MySQL 크래시 안전, OS 크래시 시 최대 1초 손실)
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
TRUNCATE TABLE bench_redo;

SET @start = NOW(6);
INSERT INTO bench_redo (data)
SELECT REPEAT('x', 100)
FROM information_schema.columns a
CROSS JOIN information_schema.columns b
LIMIT 10000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000 AS ms_setting2;
-- 예상: setting2가 2~5배 빠름 (fsync 빈도 대폭 감소)

-- 원복
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
DROP TABLE bench_redo;
```

### 실험 3: @Transactional 롤백 규칙 확인 (Java)

```java
// CheckedException 롤백 없는 케이스 재현
@Service
public class AccountService {

    // ❌ rollbackFor 없음 → CheckedException에서 COMMIT됨
    @Transactional
    public void badTransfer(Long fromId, Long toId, BigDecimal amount)
            throws InsufficientBalanceException {
        accountRepository.decreaseBalance(fromId, amount);
        // 위 작업은 DB에 반영됨 (Dirty Page 생성)
        
        throw new InsufficientBalanceException("잔액 부족");
        // → Spring이 CheckedException을 감지하고 COMMIT 실행!
        // → decreaseBalance는 영구 반영, increaseBalance는 실행 안 됨
        // → 돈이 사라짐
    }

    // ✅ rollbackFor = Exception.class → 모든 예외에서 롤백
    @Transactional(rollbackFor = Exception.class)
    public void safeTransfer(Long fromId, Long toId, BigDecimal amount)
            throws InsufficientBalanceException {
        accountRepository.decreaseBalance(fromId, amount);
        throw new InsufficientBalanceException("잔액 부족");
        // → Spring이 Exception.class 포함 여부 확인 → ROLLBACK 실행
        // → decreaseBalance 효과도 Undo Log로 복원됨
    }
}

// 실제 확인:
// before: alice=10000
// badTransfer 실행 후: alice=7000 (차감만 남음!)
// safeTransfer 실행 후: alice=10000 (정상 롤백)
```

### 실험 4: Consistency — DB 제약과 비즈니스 규칙의 경계

```sql
-- DB가 보장하는 Consistency (제약조건):
INSERT INTO accounts (name, balance) VALUES (NULL, 1000);
-- ERROR 1048: Column 'name' cannot be null → NOT NULL 제약

UPDATE accounts SET balance = -1 WHERE id = 1;
-- ERROR 3819: Check constraint 'balance >= 0' is violated → CHECK 제약

-- DB가 보장하지 못하는 비즈니스 규칙:
-- "BLOCKED 사용자는 출금 불가" → DB 레벨 제약 없음
-- 이것은 애플리케이션의 @Transactional 내에서 검증해야 함

-- 만약 @Transactional 없이 비즈니스 규칙 검증 후 별도 저장:
-- Thread A: user.isBlocked() = false → 출금 진행
-- Thread B: user.setBlocked(true) → COMMIT
-- Thread A: UPDATE balance (블록된 사용자의 잔액을 변경!)
-- → TOCTOU (Time-Of-Check-Time-Of-Use) 취약점
-- → @Transactional + FOR UPDATE (Pessimistic Lock) 또는
--   @Version (Optimistic Lock)이 필요한 이유
```

---

## 📊 성능 비교

```
innodb_flush_log_at_trx_commit 설정별 성능:

            내구성             TPS(상대)   장애 시 손실
──────────────────────────────────────────────────────
설정 1      완전 보장           1x         없음
설정 2      MySQL 크래시 안전   3~10x      OS 크래시 시 최대 1초
설정 0      보장 없음           5~15x      MySQL 크래시 시에도 최대 1초

실측 예시 (SSD, 단순 INSERT 1만 건):
  설정 1: 약 1,200ms (fsync 1만 번)
  설정 2: 약 200ms   (fsync ~1 번/초)
  설정 0: 약 150ms   (즉시 반환)

Undo Log가 성능에 미치는 영향:
  일반 UPDATE 한 건: Undo Record 1개 기록 ≈ 수십 bytes
  10만 건 UPDATE 후 ROLLBACK:
    Undo Log 역방향 적용 시간 ∝ 변경 Row 수
    → 대량 변경 트랜잭션의 롤백은 변경과 동일한 시간 소요
    → 5분 걸린 배치 UPDATE → 롤백도 5분 소요
```

---

## ⚖️ 트레이드오프

```
innodb_flush_log_at_trx_commit 선택:

설정 1 (기본):
  ✅ 완전한 내구성 (PostgreSQL, Oracle 기본과 동일 수준)
  ✅ OS 크래시, 전원 장애에도 안전
  ❌ COMMIT마다 fsync → IOPS 병목 → TPS 제한
  → 금융, 결제, 주문처럼 데이터 손실이 치명적인 서비스

설정 2:
  ✅ MySQL 프로세스 크래시에 안전
  ✅ 성능 3~10배 향상
  ❌ OS 크래시/전원 장애 시 최대 1초 손실
  → Replica 서버, 로그 수집 DB, 분석 DB

설정 0:
  ✅ 최고 성능
  ❌ MySQL 크래시 시에도 최대 1초 손실
  → 개발/테스트 환경에서만 권장

@Transactional(rollbackFor = Exception.class) 항상 붙여야 하는가:
  붙이는 비용: 없음 (런타임 성능 무관)
  안 붙이는 위험: CheckedException 발생 시 데이터 불일치
  → 비즈니스 로직에 CheckedException을 사용한다면 반드시 설정
  → 팀 컨벤션으로 "항상 붙이기"를 권장하는 이유

Undo Log 크기 관리:
  장시간 트랜잭션 → Undo Log 삭제 불가 → Tablespace 팽창
  배치 처리: LIMIT으로 소량씩 나눠 커밋 (Undo Log 정기 Purge)
  분석 쿼리: Read Replica 분리 (메인 서버 Undo Log에 영향 없음)
```

---

## 📌 핵심 정리

```
ACID 구현 담당자 매핑:

A (Atomicity):
  구현체: InnoDB Undo Log (Before Image 역방향 적용)
  Spring: @Transactional → 예외 시 ROLLBACK 호출
  함정: CheckedException → 기본 COMMIT → rollbackFor 필수

C (Consistency):
  DB: PK, FK, NOT NULL, UNIQUE, CHECK 제약조건
  App: 비즈니스 규칙 (@Transactional 내 검증 코드)
  함정: "트랜잭션이 일관성을 알아서 보장"은 틀림

I (Isolation):
  읽기: MVCC (Read View + Undo Log 버전 체인)
  쓰기: Row-Level Lock
  기본값: REPEATABLE_READ
  Spring: @Transactional(isolation=...)으로 조정

D (Durability):
  구현체: Redo Log + WAL (Sequential Write 먼저)
  설정: innodb_flush_log_at_trx_commit = 1 (완전 보장)
  함정: 설정 2/0은 특정 장애에서 데이터 손실

@Transactional = InnoDB에 BEGIN/COMMIT/ROLLBACK 신호 전달
                ACID를 직접 구현하지 않음
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 데이터 불일치가 발생하는 시나리오를 설명하라.

```java
@Transactional  // rollbackFor 없음
public void processPayment(Long orderId, Long userId, BigDecimal amount)
        throws PaymentGatewayException {
    
    Order order = orderRepository.findById(orderId);
    order.setStatus("PROCESSING");
    orderRepository.save(order);           // ①
    
    paymentGatewayService.charge(userId, amount);  // ② 외부 HTTP 호출
    // ↑ PaymentGatewayException (CheckedException) 발생 가능
    
    order.setStatus("COMPLETED");
    orderRepository.save(order);           // ③
}
```

<details>
<summary>해설 보기</summary>

**데이터 불일치 시나리오**:
`paymentGatewayService.charge()`에서 `PaymentGatewayException`이 발생하면:
- ①의 `status = "PROCESSING"` 변경은 **COMMIT됨** (CheckedException이므로 롤백 없음)
- ②의 외부 결제는 실패
- ③은 실행되지 않음

결과: `order.status = "PROCESSING"` 상태로 영구 저장. 결제는 실패. 주문이 중간 상태에 멈춤.

**올바른 해결책**:
1. `@Transactional(rollbackFor = Exception.class)` 추가
2. 외부 HTTP 호출은 트랜잭션 밖으로 분리 (트랜잭션 내에서 외부 IO는 안티패턴)
3. Outbox Pattern: DB 트랜잭션 내에 "결제 이벤트"를 기록 → 별도 프로세스가 재시도

외부 시스템 호출이 포함된 트랜잭션은 ACID만으로 해결 불가. Saga 패턴, Outbox 패턴이 필요한 이유입니다.

</details>

---

**Q2.** `innodb_flush_log_at_trx_commit = 2`인 환경에서 Replica(슬레이브)를 운영 중이다. Primary가 OS 크래시로 1초치 Redo Log를 잃었다. Replica에는 이 데이터가 있는가?

<details>
<summary>해설 보기</summary>

**있을 수도 있고 없을 수도 있습니다.** 핵심은 Binary Log와 Redo Log의 2PC(2-Phase Commit) 관계입니다.

`innodb_flush_log_at_trx_commit = 2`는 Redo Log의 내구성만 낮춥니다. **Binary Log**(`sync_binlog = 1` 설정 시)는 COMMIT마다 fsync되어 내구성이 보장됩니다.

시나리오:
- `sync_binlog = 1` (기본): Binary Log는 디스크에 남아 있음 → Replica는 해당 트랜잭션을 수신했을 가능성 있음
- `sync_binlog = 0`: Binary Log도 OS 버퍼에만 있으면 함께 손실

Primary가 크래시에서 복구되면 Redo Log와 Binary Log를 비교하는 **XA Recovery**가 실행됩니다. Binary Log에 있지만 Redo Log에 없는 트랜잭션은 롤백됩니다. 이때 Replica에 이미 적용된 데이터와 불일치가 발생할 수 있습니다.

프로덕션 권장: Primary는 `innodb_flush_log_at_trx_commit = 1` + `sync_binlog = 1`. Replica에서 설정 2 사용.

</details>

---

**Q3.** Spring Data JPA의 `save()` 메서드 내부에 `@Transactional`이 붙어 있음에도 불구하고, 호출자 메서드에서도 `@Transactional`을 붙여야 하는 이유는?

<details>
<summary>해설 보기</summary>

Spring의 트랜잭션 전파(Propagation) 때문입니다. 기본값 `REQUIRED`는 "기존 트랜잭션에 참여하고, 없으면 새로 생성"입니다.

**`@Transactional` 없는 호출자**:
```java
// 호출자 (트랜잭션 없음)
public void processOrders() {
    Order order = orderRepository.findById(1L);   // 트랜잭션 A 시작 → 완료
    
    // 이 사이에 다른 스레드가 order를 수정하고 COMMIT
    
    order.setStatus("SHIPPED");
    orderRepository.save(order);                   // 트랜잭션 B 시작 → 완료
}
// 결과: findById와 save가 서로 다른 트랜잭션 → 일관성 없음
```

**`@Transactional` 있는 호출자**:
```java
@Transactional  // 트랜잭션 C 시작
public void processOrders() {
    Order order = orderRepository.findById(1L);  // 트랜잭션 C에 참여
    order.setStatus("SHIPPED");
    orderRepository.save(order);                  // 트랜잭션 C에 참여
}
// 결과: findById와 save가 같은 트랜잭션 C → Dirty Write 감지 가능, 일관성 보장
```

또한 JPA의 1차 캐시(Persistence Context)는 트랜잭션 범위와 동일합니다. 호출자에 `@Transactional`이 없으면 각 `save()`마다 Persistence Context가 초기화되어 변경 감지(Dirty Checking)가 동작하지 않습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: MVCC 내부 원리 ➡️](./02-mvcc-internals.md)**

</div>
