# 트랜잭션 격리 수준 4가지 — 구현 원리까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- READ_UNCOMMITTED가 Dirty Read를 일으키는 내부 이유는?
- READ_COMMITTED에서 Non-Repeatable Read가 발생하는 Read View 생성 시점 차이는?
- REPEATABLE_READ가 Phantom Read를 항상 방지하지 못하는 케이스는 무엇인가?
- SERIALIZABLE이 모든 이상 현상을 막는 구체적 메커니즘은?
- JPA `@Transactional(isolation = READ_COMMITTED)`가 DB에 어떤 SQL을 실행하는가?
- 격리 수준 변경 시 Connection Pool에서 발생할 수 있는 문제는?

---

## 🔍 왜 이 개념이 중요한가

### 격리 수준을 모르면 "동일 쿼리인데 왜 다른 결과가 나오지?"를 설명 못 한다

```
실제 버그 사례:
  Spring 서비스에서 @Transactional 내에서 같은 데이터를 두 번 읽는데
  두 번째에서 다른 결과가 나와 NPE 발생

  원인: DB가 READ COMMITTED로 설정된 상태에서
        두 번째 읽기 시점에 다른 트랜잭션이 COMMIT한 변경이 반영됨
        → Non-Repeatable Read

  다른 사례:
    서비스 A: REPEATABLE_READ에서 범위 집계 후 그 결과로 INSERT
    서비스 B: 같은 범위에 동시에 INSERT
    → 서비스 A의 집계는 정확했지만 실제 저장된 데이터는 이미 변경됨
    → Phantom Read

이 문서의 목표:
  "어느 격리 수준에서 어떤 이상 현상이 발생하는가"를 Read View 생성 시점으로 설명
  → 설정값이 아닌 메커니즘으로 이해 → 올바른 격리 수준 선택 가능
```

---

## 😱 잘못된 이해

### Before: "높은 격리 수준 = 무조건 안전, 낮은 격리 수준 = 무조건 위험"

```
잘못된 이분법:
  SERIALIZABLE = 완벽하게 안전 → 항상 써야 하는 것 아닌가?
  READ_UNCOMMITTED = 절대 쓰면 안 되는 것

실제:
  SERIALIZABLE:
    모든 SELECT에 S Lock → 동시 쓰기가 차단됨
    OLTP 환경에서 Lock 경합 폭증 → TPS 급감
    Deadlock 가능성 극대화

  READ_UNCOMMITTED:
    통계성 집계 (정확도보다 속도가 중요한 경우)
    "대략적인 COUNT" 용도로 허용되는 경우 있음
    실무에서 의도적으로 사용하는 사례 존재

잘못된 또 다른 믿음:
  "격리 수준이 높을수록 Lock이 오래 유지된다"
  → 읽기는 MVCC이므로 격리 수준과 Lock 유지 시간이 무관
  → Lock에 영향을 주는 것은 쓰기 (UPDATE/DELETE)의 종류

  "InnoDB REPEATABLE_READ에서는 Phantom Read가 절대 발생하지 않는다"
  → 일반 SELECT (Consistent Read)에서는 방지
  → SELECT ... FOR UPDATE (Current Read)에서는 Phantom Read 가능
```

---

## ✨ 올바른 이해

### After: 격리 수준 = "Read View를 언제 생성하는가"의 차이

```
4가지 격리 수준의 핵심 차이:

READ_UNCOMMITTED:
  Read View 없음
  Undo Log 버전 체인 탐색 없음
  Buffer Pool의 최신 버전을 그대로 읽음 (커밋 여부 무관)
  → Dirty Read 가능 (미커밋 데이터 읽기)

READ_COMMITTED:
  각 SELECT마다 새로운 Read View 생성
  → 항상 최신 커밋된 버전 읽음
  → Non-Repeatable Read 가능 (같은 SELECT 반복 시 다른 결과)
  → Phantom Read 가능

REPEATABLE_READ (InnoDB 기본값):
  트랜잭션의 첫 번째 읽기 시 Read View 1회 생성 후 고정
  → Non-Repeatable Read 방지 (Consistent Read에서)
  → Phantom Read: 일반 SELECT에서는 방지, FOR UPDATE에서는 가능
  InnoDB 추가 보장: Gap Lock으로 Current Read에서의 Phantom도 대부분 방지

SERIALIZABLE:
  REPEATABLE_READ + 모든 SELECT에 S Lock 자동 추가
  → Phantom Read 완전 방지 (Lock으로 삽입 차단)
  → 가장 높은 격리, 가장 낮은 동시성

비용 vs 안전성:
  낮은 격리 ←──────────────────── 높은 격리
  READ_UNCOMMITTED  RC  RR  SERIALIZABLE
  빠름 ←──────────────────────── 느림
  위험 ──────────────────────→ 안전
```

---

## 🔬 내부 동작 원리

### 1. READ_UNCOMMITTED — Dirty Read 발생 원리

```
Read View 없이 최신 Buffer Pool 버전을 직접 읽음:

타임라인:
  TRX A: START TRANSACTION;
  TRX B: START TRANSACTION;
  TRX B: UPDATE accounts SET balance = -99999 WHERE id = 1;
         -- Buffer Pool: balance = -99999 (아직 COMMIT 안 됨)

  TRX A (READ UNCOMMITTED):
    SELECT balance FROM accounts WHERE id = 1;
    -- Buffer Pool의 최신 버전 직접 읽음
    -- → -99999 읽음! (Dirty Read)

  TRX B: ROLLBACK;
         -- -99999 → 원래값으로 복원 (Undo Log 역방향 적용)

  TRX A는 이미 -99999를 읽어서 잘못된 처리를 했을 수 있음
  이 값은 존재하지 않았던 데이터

실무 사용 케이스:
  SELECT COUNT(*) FROM orders;  -- 대략적인 건수 파악
  → 약간의 부정확함은 허용, 속도가 더 중요한 대시보드 통계
  → 잘못된 COUNT는 있을 수 있지만 서비스 로직에 영향 없음

금지 케이스:
  계좌 잔액 조회, 재고 확인, 결제 처리
  → 미커밋 데이터를 읽어 잘못된 결제/재고 처리 가능
```

### 2. READ_COMMITTED — Non-Repeatable Read 발생 원리

```
각 SELECT마다 새로운 Read View 생성:

정확한 타임라인:
  TRX A (READ COMMITTED):
    T1: SELECT balance → Read View₁ 생성 (활성: {TRX B})
        TRX B가 활성 → TRX B의 변경은 안 보임 → balance = 10000

  TRX B:
    T2: UPDATE balance = 7000;
        COMMIT;

  TRX A:
    T3: SELECT balance → Read View₂ 생성 (활성: {})
        TRX B가 커밋됨 → TRX B의 변경이 보임 → balance = 7000

  같은 트랜잭션, 같은 쿼리, 다른 결과 = Non-Repeatable Read

Gap Lock 비활성화 효과:
  READ_COMMITTED에서는 Gap Lock을 사용하지 않음
  → WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE 실행 시
    해당 Range에 Record Lock만, Gap Lock 없음
  → 다른 트랜잭션이 이 범위에 새 Row INSERT 가능
  → Phantom Read 가능

장점:
  Gap Lock 없음 → Deadlock 빈도 감소
  항상 최신 커밋 데이터 읽기 → "내가 보는 것이 실제"라는 직관적 동작
  Statement-based Replication에서 RR보다 안전하지 않음 → Row-based 필수
```

### 3. REPEATABLE_READ — InnoDB 기본값의 특수 처리

```
Read View 1회 고정 + InnoDB의 추가 보장:

Consistent Read (일반 SELECT)에서:
  Read View 고정 → Non-Repeatable Read 방지
  Read View 고정 → 새로 INSERT된 Row의 TRX_ID > m_low_limit_id → 안 보임
  → Phantom Read도 방지 (일반 SELECT에서)

Current Read (FOR UPDATE, UPDATE, DELETE)에서:
  MVCC 버전이 아닌 최신 커밋 버전을 읽음
  → TRX A가 FOR UPDATE로 집계 후 TRX B가 INSERT하면
    TRX A의 두 번째 FOR UPDATE에서 새 Row가 보임 = Phantom Read

InnoDB의 특수 해결책 — Gap Lock:
  REPEATABLE_READ에서 FOR UPDATE 실행 시
  Record Lock + 해당 범위의 Gap Lock을 함께 획득
  → Gap Lock이 있는 범위에는 INSERT 불가
  → TRX A의 FOR UPDATE 범위에 TRX B가 INSERT 시도 → 차단!
  → Phantom Read 방지

ANSI 표준 REPEATABLE_READ와의 차이:
  ANSI 표준: REPEATABLE_READ에서 Phantom Read 허용
  InnoDB: Gap Lock으로 Current Read에서도 Phantom Read 대부분 방지
  → InnoDB의 RR은 ANSI 표준보다 강한 격리 제공
```

### 4. SERIALIZABLE — Lock 기반 완전 직렬화

```
모든 SELECT가 암묵적으로 SELECT ... FOR SHARE로 실행:

타임라인:
  TRX A (SERIALIZABLE):
    SELECT * FROM orders WHERE amount > 1000;
    -- 실제: SELECT * FROM orders WHERE amount > 1000 FOR SHARE
    -- 결과 Row에 S Lock + 범위에 Gap Lock 획득

  TRX B:
    INSERT INTO orders (amount) VALUES (2000);
    -- amount > 1000인 새 Row → TRX A의 Gap Lock 범위 내
    -- → 차단! TRX A COMMIT 대기

  TRX A:
    SELECT * FROM orders WHERE amount > 1000;  -- 이번에도 같은 결과
    COMMIT;  -- S Lock + Gap Lock 해제

  TRX B: INSERT 완료

Phantom Read 완전 방지:
  S Lock + Gap Lock → 범위 내 삽입/수정 완전 차단
  → TRX A가 실행 중인 동안 다른 어떤 변경도 범위에 영향 불가

SERIALIZABLE의 성능 비용:
  모든 SELECT에 S Lock 추가 → 모든 읽기가 쓰기를 일부 차단
  여러 트랜잭션이 같은 Row를 읽으면: 모두 S Lock (공존 가능)
  하나가 업데이트하려면: 모든 S Lock이 해제될 때까지 대기
  → 읽기 많은 OLTP에서 Lock 경합 폭증 → TPS 급감

사용 사례:
  은행 계좌 간 이체처럼 절대적 일관성이 필요한 경우
  그러나 현실에서는 SELECT FOR UPDATE (Pessimistic Lock)으로
  SERIALIZABLE과 동일한 보장을 더 좁은 범위에서 달성 권장
```

### 5. JPA @Transactional과 격리 수준 설정

```java
// Spring이 격리 수준을 설정하는 방식:

@Transactional(isolation = Isolation.READ_COMMITTED)
public OrderDto getOrderDetails(Long orderId) {
    // 실행 전:
    // Connection conn = hikariCP.getConnection();
    // conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
    // 또는: 
    // jdbcTemplate.execute("SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED");

    // 메서드 실행...

    // 실행 후:
    // conn.setTransactionIsolation(원래 격리 수준);  ← 반드시 복원!
    // hikariCP.returnConnection(conn);
}

// Connection Pool 오염 위험:
// HikariCP는 Connection을 재사용함
// @Transactional(isolation=READ_COMMITTED) → 메서드 완료 후 복원이 되어야 함
// Spring AbstractPlatformTransactionManager가 자동 처리하므로 보통 안전
// 단, 직접 Connection을 조작하는 코드가 있으면 복원 누락 가능

// 전역 설정 (application.properties):
// spring.datasource.hikari.transaction-isolation=TRANSACTION_READ_COMMITTED
// → 모든 Connection의 기본값 변경 (메서드별 설정보다 우선순위 낮음)

// 격리 수준 확인:
@Transactional
public void checkIsolation() {
    String level = jdbcTemplate.queryForObject(
        "SELECT @@transaction_isolation", String.class);
    log.info("Current isolation: {}", level);
    // REPEATABLE-READ (기본값)
}

// @Transactional 없을 때:
// 각 쿼리는 auto-commit 모드에서 독립 실행
// 격리 수준은 Connection의 기본값 적용
// 같은 서비스 메서드의 두 쿼리가 다른 트랜잭션!
```

---

## 💻 실전 실험

### 실험 1: 격리 수준별 동작 비교 (4가지 순서로)

```sql
-- 실험 준비: 세션 A, 세션 B 두 개 열기
-- 세션 B에서 공통으로 실행할 UPDATE:
-- UPDATE accounts SET balance = 7000 WHERE id = 1;
-- COMMIT;

-- 세션 A: READ UNCOMMITTED
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- T1 기록 (10000)

-- 세션 B: UPDATE + 아직 COMMIT 안 함
-- UPDATE accounts SET balance = 7000 WHERE id = 1;

-- 세션 A: 미커밋 변경이 보이는가?
SELECT balance FROM accounts WHERE id = 1;  -- Dirty Read! 7000 보임

-- 세션 B: ROLLBACK
-- 세션 A
SELECT balance FROM accounts WHERE id = 1;  -- 다시 10000 (롤백 반영)
ROLLBACK;

-- ─────────────────────────────────────────

-- 세션 A: READ COMMITTED
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- T1: 10000

-- 세션 B: UPDATE + COMMIT 완료

-- 세션 A:
SELECT balance FROM accounts WHERE id = 1;  -- 7000 (Non-Repeatable Read!)
ROLLBACK;

-- ─────────────────────────────────────────

-- 세션 A: REPEATABLE READ
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- T1: 10000 (Read View 생성)

-- 세션 B: UPDATE balance=5000 + COMMIT

-- 세션 A:
SELECT balance FROM accounts WHERE id = 1;  -- 10000 (변하지 않음!)
ROLLBACK;
```

### 실험 2: RR에서 Consistent Read vs Current Read의 Phantom 차이

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;

-- Consistent Read (일반 SELECT)
SELECT COUNT(*) FROM orders WHERE amount > 5000;  -- 예: 100건
-- Read View 생성됨

-- 세션 B에서 amount=9999인 새 Row INSERT + COMMIT

-- Consistent Read 재실행
SELECT COUNT(*) FROM orders WHERE amount > 5000;  -- 여전히 100건!
-- Read View 고정 → 새 Row의 TRX_ID > m_low_limit_id → 안 보임
-- → Phantom Read 없음

-- Current Read (FOR UPDATE)
SELECT COUNT(*) FROM orders WHERE amount > 5000 FOR UPDATE;  -- 101건!
-- Current Read = 최신 커밋 버전 읽음 → 새 Row도 보임
-- → Phantom Read 발생!

ROLLBACK;
```

### 실험 3: SERIALIZABLE의 Lock 경합 확인

```sql
-- 세션 A: SERIALIZABLE
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
SELECT * FROM orders WHERE amount > 5000;
-- → 내부적으로 FOR SHARE 실행 → S Lock + Gap Lock 획득

-- 세션 B: 범위 내 INSERT 시도
INSERT INTO orders (user_id, amount) VALUES (1, 9999);
-- → TRX A의 Gap Lock과 충돌 → 차단! (세션 A COMMIT까지 대기)

-- Lock 대기 상태 확인
SELECT * FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'orders' AND LOCK_STATUS = 'WAITING';

-- 세션 A: COMMIT → 세션 B의 INSERT 완료
COMMIT;
```

### 실험 4: @Transactional isolation 설정 검증

```java
// 실제 격리 수준이 적용됐는지 확인
@Test
public void testIsolationLevel() {
    // 기본 격리 수준 확인
    String defaultLevel = jdbcTemplate.queryForObject(
        "SELECT @@transaction_isolation", String.class);
    assertEquals("REPEATABLE-READ", defaultLevel);

    // READ_COMMITTED로 설정된 메서드 실행 후 확인
    // (해당 메서드 내에서 격리 수준 쿼리 실행)
    transactionalService.executeWithReadCommitted();
    // → 내부에서 READ-COMMITTED 확인

    // 메서드 종료 후 원복 확인
    String afterLevel = jdbcTemplate.queryForObject(
        "SELECT @@transaction_isolation", String.class);
    assertEquals("REPEATABLE-READ", afterLevel);  // 복원됨
}
```

---

## 📊 성능 비교

```
격리 수준별 동시성 처리량 (TPS 상대 비교):

READ_UNCOMMITTED:  기준 100 TPS (Lock 없음, 최고 성능)
READ_COMMITTED:    ~95 TPS (Record Lock만, Gap Lock 없음)
REPEATABLE_READ:   ~85 TPS (Record Lock + Gap Lock)
SERIALIZABLE:      ~30~60 TPS (S Lock + Record + Gap → 심한 경합)

Deadlock 발생 빈도:
  READ_COMMITTED:   낮음 (Gap Lock 없음)
  REPEATABLE_READ:  중간 (Gap Lock 간 충돌)
  SERIALIZABLE:     높음 (모든 읽기에 S Lock)

Read View 생성 비용:
  각 SELECT마다 Read View 생성(RC) vs 1회 생성(RR):
  고동시성에서 RC는 m_ids 목록 복사 비용이 더 자주 발생
  일반적으로 무시할 수 있는 수준 (microsecond 단위)
```

---

## ⚖️ 트레이드오프

```
격리 수준 선택 기준:

READ_COMMITTED를 선택해야 하는 경우:
  쓰기가 매우 많고 Deadlock이 빈번한 서비스
  Row 단위 최신 데이터가 필요하고 Non-Repeatable Read가 허용되는 경우
  Statement-based Replication은 사용 불가 → Row-based 필수
  MySQL 외부 DB(PostgreSQL 등)의 기본값과 동일하게 맞출 때

REPEATABLE_READ(기본값)를 유지해야 하는 경우:
  한 트랜잭션 내에서 일관된 스냅샷이 필요한 복잡한 비즈니스 로직
  Statement-based Replication 사용 중
  배치 처리에서 같은 데이터를 반복 읽는 패턴

SERIALIZABLE을 선택하는 경우:
  매우 드물지만 절대적 일관성 요구 (은행 원장 등)
  보통 SERIALIZABLE 대신 특정 쿼리에 SELECT FOR UPDATE 사용 권장

JPA 프로젝트 권장 설정:
  기본: REPEATABLE_READ (MySQL 기본값 유지)
  특정 서비스 레이어: @Transactional(isolation=READ_COMMITTED) 선택적 적용
  → 전역으로 RC로 낮추기보다 필요한 곳만 명시적으로 낮추는 것이 안전
```

---

## 📌 핵심 정리

```
격리 수준 핵심 비교표:

              Dirty    Non-Rep   Phantom    Read View 생성
              Read     Read      Read       시점
────────────────────────────────────────────────────────
READ_UNC      가능     가능      가능       없음 (직접 읽기)
READ_COM      방지     가능      가능       SELECT마다 새로
REPEATABLE    방지     방지      일부 방지  트랜잭션 1회 고정
SERIALIZABLE  방지     방지      방지       1회 고정 + S Lock

InnoDB REPEATABLE_READ의 특수성:
  Consistent Read (일반 SELECT): Phantom 방지 (Read View 고정)
  Current Read (FOR UPDATE): Phantom 가능
  → Gap Lock으로 Current Read도 대부분 방지

JPA 설정:
  @Transactional(isolation = Isolation.READ_COMMITTED)
  → 메서드 실행 전 SET SESSION TRANSACTION ISOLATION LEVEL
  → 완료 후 이전 수준으로 복원

격리 수준 ≠ Lock 유지 시간:
  읽기는 MVCC라서 격리 수준과 무관
  Lock은 쓰기(UPDATE/DELETE + FOR UPDATE)에서 발생
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 상황에서 TRX A의 두 번째 SELECT 결과를 각 격리 수준별로 예측하라.

```
초기 상태: accounts(id=1, balance=10000)

TRX A: START TRANSACTION;
TRX A: SELECT balance WHERE id=1;  → 10000

[동시에] TRX B: START TRANSACTION;
              UPDATE accounts SET balance=7000 WHERE id=1;
              COMMIT;

TRX A: SELECT balance WHERE id=1;  → ???
TRX A: COMMIT;
```

<details>
<summary>해설 보기</summary>

| 격리 수준 | TRX A 두 번째 SELECT 결과 | 이유 |
|-----------|--------------------------|------|
| READ_UNCOMMITTED | 7000 | Read View 없음, Buffer Pool 최신 버전 직독 |
| READ_COMMITTED | 7000 | 두 번째 SELECT 시 새 Read View → TRX B 커밋됨 → 보임 |
| REPEATABLE_READ | 10000 | 첫 번째 SELECT 시 생성된 Read View 고정, TRX B는 그 당시 활성(미래)이었음 |
| SERIALIZABLE | 10000 | REPEATABLE_READ + S Lock, 첫 번째 SELECT의 S Lock이 TRX B의 UPDATE를 차단했을 것 (실제로는 TRX B가 먼저 UPDATE를 완료했으므로 순서에 따라 다를 수 있음) |

실제로 SERIALIZABLE에서는 TRX B의 UPDATE가 TRX A의 S Lock이 풀릴 때까지 대기했을 것입니다. TRX A가 COMMIT 후에야 TRX B가 UPDATE를 완료합니다. 따라서 TRX A의 두 번째 SELECT 결과는 10000이 됩니다.

</details>

---

**Q2.** MySQL의 REPEATABLE_READ와 ANSI SQL 표준의 REPEATABLE_READ는 다르다. 어떻게 다른가?

<details>
<summary>해설 보기</summary>

**ANSI SQL 표준 REPEATABLE_READ**:
- Non-Repeatable Read를 방지
- **Phantom Read는 허용** (표준 상 SERIALIZABLE에서만 방지)

**InnoDB REPEATABLE_READ**:
- Non-Repeatable Read 방지
- **Phantom Read도 대부분 방지** (Gap Lock 덕분)
  - Consistent Read(일반 SELECT): Read View 고정으로 Phantom Read 없음
  - Current Read(FOR UPDATE 등): Gap Lock으로 새 Row 삽입 차단

즉, InnoDB의 REPEATABLE_READ는 ANSI 표준보다 강합니다. ANSI 표준에서 Phantom을 막으려면 SERIALIZABLE이 필요하지만, InnoDB에서는 REPEATABLE_READ + Gap Lock으로 대부분의 Phantom을 막습니다.

이것이 MySQL이 REPEATABLE_READ를 기본값으로 유지하는 이유 중 하나입니다. 실질적인 보호 수준은 SERIALIZABLE에 가깝지만 성능은 SERIALIZABLE보다 훨씬 낫습니다.

</details>

---

**Q3.** 다음 코드에서 격리 수준을 올리는 것이 해결책인지 판단하라.

```java
@Transactional  // REPEATABLE_READ (기본값)
public void updateInventory(Long productId, int quantity) {
    Product product = productRepo.findById(productId);
    // 이 사이에 다른 스레드가 같은 상품의 재고를 변경하고 COMMIT

    int newStock = product.getStock() - quantity;
    if (newStock < 0) throw new InsufficientStockException();
    product.setStock(newStock);
    productRepo.save(product);
}
// 문제: 두 스레드가 동시에 실행되면 재고가 음수가 될 수 있음
```

<details>
<summary>해설 보기</summary>

**격리 수준을 올리는 것은 해결책이 아닙니다.**

이 문제는 **Lost Update** 문제입니다. REPEATABLE_READ에서:
- 스레드 A: `findById` → stock=10, Read View 고정
- 스레드 B: stock=10-3=7로 UPDATE + COMMIT
- 스레드 A: UPDATE `SET stock = 10 - 5 = 5` (MVCC 읽기와 달리 UPDATE는 Current Read → 최신 7을 읽음)

실제 UPDATE는 7 - 5 = 2가 됩니다. 문제는 없어 보이지만, 스레드 A가 읽은 10을 기반으로 계산했기 때문에 의도와 다릅니다.

SERIALIZABLE로 올리면: 스레드 A의 `findById`에 S Lock이 걸리고, 스레드 B의 UPDATE가 A의 S Lock 때문에 차단되어 직렬화됩니다. 해결은 되지만 처리량이 크게 떨어집니다.

**올바른 해결책**: `SELECT ... FOR UPDATE` (Pessimistic Lock) 사용:
```java
Product product = productRepo.findByIdWithLock(productId);
// SELECT ... FOR UPDATE → X Lock 획득
// 스레드 B의 동시 수정이 스레드 A의 Lock 해제 전까지 차단
```
또는 `@Version` (Optimistic Lock):
```java
@Version Long version;
// UPDATE ... WHERE id=? AND version=? → 충돌 시 재시도
```

격리 수준은 "읽기의 일관성"을 제어하는 것이지 "동시 쓰기 충돌"을 막는 수단이 아닙니다.

</details>

---

<div align="center">

**[⬅️ Undo Log](./03-undo-log.md)** | **[홈으로 🏠](../README.md)** | **[다음: Phantom Read ➡️](./05-phantom-read.md)**

</div>
