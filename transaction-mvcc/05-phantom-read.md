# Phantom Read — 발생 조건과 방지 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Phantom Read와 Non-Repeatable Read는 정확히 무엇이 다른가?
- REPEATABLE_READ + Consistent Read에서 Phantom Read가 방지되는 원리는?
- REPEATABLE_READ + Current Read(FOR UPDATE)에서 Phantom Read가 발생하는 이유는?
- Gap Lock이 "공간"에 Lock을 거는 방식의 정확한 범위 계산은?
- 같은 Gap에 두 트랜잭션이 Gap Lock을 동시에 보유할 수 있는가?
- Gap Lock이 Deadlock의 원인이 되는 패턴은?

---

## 🔍 왜 이 개념이 중요한가

### "분명히 REPEATABLE_READ인데 왜 결과 건수가 달라지지?"

```
실제 버그 패턴:
  @Transactional (REPEATABLE_READ)
  public void allocateSeats(int concertId, int requestedCount) {
    int available = seatRepository.countAvailableSeats(concertId); // 일반 SELECT
    if (available < requestedCount) throw new NoSeatsException();

    // 이 사이에 다른 트랜잭션이 좌석 N개 예약 완료

    List<Seat> seats = seatRepository.findAvailableSeats(concertId, requestedCount); // FOR UPDATE
    // → 좌석 수가 방금 카운트한 것과 달라짐!
    // → Phantom Read 발생
  }

왜 이 현상이 일어나는가:
  첫 번째 COUNT: 일반 SELECT → Consistent Read → Read View 기준 결과 = 10
  두 번째 findAvailableSeats: FOR UPDATE → Current Read → 최신 커밋 버전 = 7

  같은 트랜잭션, 다른 읽기 방식 → 다른 결과
  = Phantom Read (결과 행의 수가 달라짐)

Phantom Read를 이해해야 하는 이유:
  FOR UPDATE를 써서 "안전하게" 처리한다고 생각했는데
  오히려 Phantom Read가 발생하는 역설적 상황을 이해
  Gap Lock이 이를 방지하는 방법 이해
  Gap Lock이 Deadlock의 원인이 될 수 있다는 트레이드오프 이해
```

---

## 😱 잘못된 이해

### Before: "REPEATABLE_READ면 Phantom Read가 절대 없다"

```sql
-- 잘못된 믿음:
-- "InnoDB REPEATABLE_READ는 Phantom Read도 막는다"
-- → 부분적으로만 사실 (Consistent Read에서만)

-- 잘못된 코드 패턴:
START TRANSACTION;  -- REPEATABLE_READ

SELECT COUNT(*) FROM orders WHERE amount > 5000;  -- 50건
-- "50건이라고 확인했으니 안전하다"

-- [다른 세션에서 amount=9999인 Row INSERT + COMMIT]

SELECT * FROM orders WHERE amount > 5000 FOR UPDATE;  -- 51건!
-- "FOR UPDATE니까 더 안전할 텐데 왜 건수가 다르지?"

-- 잘못된 이해의 결과:
-- "FOR UPDATE를 쓰면 더 안전하다" 라고 생각했는데
-- 오히려 FOR UPDATE가 Phantom Read를 일으킨 역설적 상황
-- → Consistent Read는 Phantom 방지, Current Read는 Phantom 가능을 몰랐기 때문
```

```
잘못된 또 다른 믿음:
  "Gap Lock = 모든 삽입을 차단한다"
  → 실제: 잠긴 Gap 범위에 대한 삽입만 차단
    잠기지 않은 다른 Gap에는 삽입 가능

  "Gap Lock이 있으면 동시 INSERT가 불가능하다"
  → 실제: Gap Lock 범위 밖 INSERT는 가능
    Gap Lock은 "이 공간"에 대한 Insert Intention Lock만 차단
    다른 위치는 자유롭게 삽입 가능
```

---

## ✨ 올바른 이해

### After: 읽기 방식에 따라 다른 Phantom Read 동작

```
Consistent Read (일반 SELECT) → Phantom Read 없음:
  Read View가 생성 시점으로 고정
  새로 INSERT된 Row: DB_TRX_ID = 삽입 트랜잭션 ID
  그 ID는 Read View의 m_low_limit_id 이상 (미래 트랜잭션)
  → is_visible() = False → 보이지 않음
  → 범위 쿼리를 반복해도 새 Row 안 보임 = Phantom 없음

Current Read (FOR UPDATE, FOR SHARE) → Phantom Read 가능:
  MVCC 버전 우회, Buffer Pool의 최신 커밋 버전 읽음
  최신 버전에는 새로 INSERT된 Row가 이미 포함
  → 범위 쿼리를 반복하면 새 Row가 보임 = Phantom Read

InnoDB의 Gap Lock 해결책:
  Current Read 실행 시:
    Record Lock (조건에 맞는 기존 Row)
    + Gap Lock (조건 범위의 빈 공간)
  → Gap Lock이 있는 공간에 다른 트랜잭션의 Insert Intention Lock 차단
  → 새 Row 삽입 불가 → Phantom Read 방지

핵심 요약:
  일반 SELECT + RR = Phantom 없음 (Read View)
  FOR UPDATE + RR = Phantom 방지 (Gap Lock)
  FOR UPDATE + RC = Phantom 가능 (Gap Lock 없음)
```

---

## 🔬 내부 동작 원리

### 1. Phantom Read vs Non-Repeatable Read 정확한 구분

```
Non-Repeatable Read:
  같은 Row를 두 번 읽었을 때 값이 달라짐
  원인: 기존 Row의 UPDATE
  예시:
    SELECT balance WHERE id=1 → 10000
    [다른 TRX: UPDATE balance=7000 + COMMIT]
    SELECT balance WHERE id=1 → 7000  ← 값이 변함

Phantom Read:
  같은 조건의 범위 쿼리를 두 번 실행했을 때 결과 행 수가 달라짐
  원인: 범위에 새 Row INSERT 또는 기존 Row DELETE
  예시:
    SELECT COUNT(*) WHERE amount > 5000 → 50건
    [다른 TRX: INSERT amount=9999 + COMMIT]
    SELECT COUNT(*) WHERE amount > 5000 → 51건  ← 행 수가 변함

InnoDB의 방지 메커니즘:
  Non-Repeatable Read: REPEATABLE_READ의 Read View 고정 → 방지
  Phantom Read (Consistent Read): Read View 고정 → 방지
  Phantom Read (Current Read): Gap Lock 필요 → RR에서 자동 적용

표준 SQL 기준 vs InnoDB:
  표준 SQL RR: Non-Repeatable 방지, Phantom 허용
  InnoDB RR:   Non-Repeatable + Phantom 모두 방지 (대부분)
```

### 2. Gap Lock 범위 계산 상세

```
인덱스 데이터: amount 인덱스에 [500, 1000, 2000, 5000, 10000]

Gap들:
  (-∞, 500): 500 이전 Gap
  (500, 1000): 500과 1000 사이 Gap
  (1000, 2000): ...
  (2000, 5000): ...
  (5000, 10000): ...
  (10000, +∞): 10000 이후 Gap

SELECT * FROM orders WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE:
  대상 Records: 1000, 2000, 5000
  획득하는 Lock:

  Next-Key Lock on 1000:  Gap (500, 1000] = Gap Lock(500,1000) + Record Lock(1000)
  Next-Key Lock on 2000:  Gap (1000, 2000] = Gap Lock(1000,2000) + Record Lock(2000)
  Next-Key Lock on 5000:  Gap (2000, 5000] = Gap Lock(2000,5000) + Record Lock(5000)
  Gap Lock on (5000, 10000): 범위 끝 이후의 Gap도 보호

  총 보호 범위: (500, 10000]
  → 이 범위 내 새 Row INSERT 차단 (amount=600, 1500, 3000, 7000 등)

존재하지 않는 값 조건:
  SELECT * FROM orders WHERE amount = 3000 FOR UPDATE (3000이 없음)
  → Record Lock 없음 (해당 레코드 없으니)
  → Gap Lock (2000, 5000): 3000이 삽입될 수 있는 공간 보호
  → amount=2500, amount=4000 INSERT 차단

Supremum Lock (범위 끝 없는 조건):
  SELECT * FROM orders WHERE amount > 5000 FOR UPDATE
  → 5000 초과의 모든 Record Lock
  → Gap Lock (5000, 10000), (10000, supremum)
  → supremum pseudo-record Lock
  → 미래에 amount=99999 INSERT도 차단
```

### 3. Gap Lock 공존과 충돌 규칙

```
Gap Lock끼리는 공존 가능 (같은 Gap에 여러 트랜잭션이 동시 보유):

이유:
  Gap Lock의 목적 = "이 공간에 삽입 차단"
  두 트랜잭션이 같은 공간을 보호해도 서로 방해하지 않음
  (두 경비원이 같은 문을 동시에 지키는 것과 같음)

TRX A: Gap Lock (1000, 2000) 보유
TRX B: Gap Lock (1000, 2000) 보유
→ 둘 다 보유 가능! Gap Lock 공존 ✅

Insert Intention Lock과의 충돌:
  삽입 시도: amount=1500 INSERT
  → Insert Intention Lock 획득 시도 on Gap (1000, 2000) at position 1500
  → TRX A의 Gap Lock과 충돌 → 차단!
  → TRX B의 Gap Lock과도 충돌 → 차단!
  → TRX A와 TRX B가 모두 COMMIT할 때까지 대기

Gap Lock이 Deadlock을 유발하는 패턴:
  TRX A: Gap Lock(1000,2000) 보유 → INSERT amount=1500 시도
  TRX B: Gap Lock(1000,2000) 보유 → INSERT amount=1700 시도

  TRX A의 Insert Intention Lock → TRX B의 Gap Lock이 차단
  TRX B의 Insert Intention Lock → TRX A의 Gap Lock이 차단
  → 순환 대기 → Deadlock!

  흔히 발생하는 실제 패턴:
    INSERT ... ON DUPLICATE KEY UPDATE 두 트랜잭션이 동시에 같은 범위에 실행
    INSERT INTO orders SELECT ... 두 세션이 같은 조건으로 동시 실행
    → Gap Lock 충돌 → Deadlock
```

### 4. READ_COMMITTED에서 Gap Lock 비활성화

```
READ_COMMITTED 특성:
  Gap Lock을 사용하지 않음 → Record Lock만 사용

효과:
  SELECT ... FOR UPDATE: 조건에 맞는 기존 Row에만 Record Lock
  범위 내 새 Row INSERT: 차단되지 않음 → 자유로운 삽입
  → Phantom Read 발생 가능
  → 그 대신 Gap Lock 관련 Deadlock 감소

READ_COMMITTED를 선택하면 감수해야 하는 것:
  SELECT COUNT(*) ... WHERE + 그 결과로 INSERT 하는 패턴에서
  COUNT와 INSERT 사이에 다른 트랜잭션이 같은 범위에 INSERT하면
  → 내가 카운트한 것보다 많은 데이터가 실제로 존재
  → 비즈니스 로직에서 이를 허용해야 함

실무 선택:
  Phantom Read가 치명적인 경우: REPEATABLE_READ 유지 + Gap Lock 감수
  Deadlock이 더 큰 문제인 경우: READ_COMMITTED + Phantom Read 감수
                               → 애플리케이션 레벨에서 보완 처리
```

### 5. Phantom Read 방지 실전 패턴

```java
// 올바른 패턴 1: Consistent Read → 집계, Current Read → 실제 작업

@Transactional
public List<Seat> reserveSeats(Long concertId, int count) {
    // ❌ 잘못된 패턴: 두 읽기 방식을 혼용
    int available = seatRepo.countAvailable(concertId);  // Consistent Read
    if (available < count) throw new NoSeatsException();
    return seatRepo.findAvailableWithLock(concertId, count);  // Current Read → Phantom!

    // ✅ 올바른 패턴: FOR UPDATE로 통일 (Current Read)
    List<Seat> seats = seatRepo.findAvailableWithLock(concertId, count);
    // FOR UPDATE → Gap Lock 획득 → 중간에 다른 트랜잭션 삽입/삭제 차단
    if (seats.size() < count) throw new NoSeatsException();
    seats.forEach(seat -> seat.setStatus("RESERVED"));
    return seats;
}

// 올바른 패턴 2: Consistent Read만 사용 + 비즈니스 재검증
@Transactional
public void processOrder(Long productId, int qty) {
    Product product = productRepo.findByIdWithLock(productId);  // FOR UPDATE
    // Gap Lock 관련 없음 (단건 PK 조회)
    // 재고를 최신 버전으로 확인하고 수정
    if (product.getStock() < qty) throw new InsufficientStockException();
    product.decreaseStock(qty);
}
```

---

## 💻 실전 실험

### 실험 1: Consistent Read vs Current Read의 Phantom 차이

```sql
CREATE TABLE phantom_test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    amount INT NOT NULL,
    INDEX idx_amount (amount)
) ENGINE=InnoDB;

INSERT INTO phantom_test (amount) VALUES (500), (1000), (2000), (5000), (10000);

-- 세션 1 (REPEATABLE_READ)
START TRANSACTION;
SELECT COUNT(*) FROM phantom_test WHERE amount > 1000;  -- 3건 (2000, 5000, 10000)
-- Read View 생성됨

-- 세션 2
INSERT INTO phantom_test (amount) VALUES (3000);
COMMIT;

-- 세션 1
-- Consistent Read (일반 SELECT)
SELECT COUNT(*) FROM phantom_test WHERE amount > 1000;  -- 여전히 3건! Phantom 없음

-- Current Read (FOR UPDATE)
SELECT COUNT(*) FROM phantom_test WHERE amount > 1000 FOR UPDATE;  -- 4건! Phantom Read!

ROLLBACK;
```

### 실험 2: Gap Lock 범위 확인

```sql
-- 세션 1
START TRANSACTION;
SELECT * FROM phantom_test WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE;
-- Record Lock: 1000, 2000, 5000
-- Gap Lock: (500,1000), (1000,2000), (2000,5000), (5000,10000)

-- Lock 상태 확인
SELECT LOCK_TYPE, LOCK_MODE, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'phantom_test'
ORDER BY LOCK_DATA;
-- X,REC_NOT_GAP: Record Lock (1000, 2000, 5000)
-- X,GAP: Gap Lock 목록

-- 세션 2: Gap 내부 삽입 시도
INSERT INTO phantom_test (amount) VALUES (1500);  -- 차단! (1000,2000) Gap
INSERT INTO phantom_test (amount) VALUES (7000);  -- 통과! (5000,10000) Gap... 아니 이것도 잠겨있음
INSERT INTO phantom_test (amount) VALUES (15000); -- 통과! (10000,supremum) Gap은?

-- 세션 1 COMMIT 후 세션 2 재시도
COMMIT;
```

### 실험 3: Gap Lock Deadlock 재현

```sql
-- 세션 1: amount 범위에 Gap Lock 획득
START TRANSACTION;
SELECT * FROM phantom_test WHERE amount BETWEEN 1000 AND 3000 FOR UPDATE;
-- Gap Lock (500,1000), (1000,2000), (2000,3000 = next gap after 2000)

-- 세션 2: 겹치는 Gap Lock 획득 후 삽입 시도
START TRANSACTION;
SELECT * FROM phantom_test WHERE amount BETWEEN 2000 AND 5000 FOR UPDATE;
-- Gap Lock (1000,2000), (2000,5000), ... → 세션 1과 (1000,2000) Gap Lock 공존 가능

-- 세션 1: 겹치는 Gap에 INSERT 시도
INSERT INTO phantom_test (amount) VALUES (2500);
-- → Insert Intention Lock → 세션 2의 Gap Lock과 충돌 → 대기!

-- 세션 2: 같은 Gap에 INSERT 시도
INSERT INTO phantom_test (amount) VALUES (1500);
-- → Insert Intention Lock → 세션 1의 Gap Lock과 충돌
-- → 세션 1도 세션 2를 기다리고 있음 → Deadlock!

-- 에러: ERROR 1213 (40001): Deadlock found
SHOW ENGINE INNODB STATUS\G  -- LATEST DETECTED DEADLOCK 확인
```

### 실험 4: READ_COMMITTED에서 Gap Lock 비활성화 확인

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM phantom_test WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE;

-- Lock 상태 확인
SELECT LOCK_TYPE, LOCK_MODE, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'phantom_test';
-- X,REC_NOT_GAP만 있고 X,GAP이 없음! → Gap Lock 없음

-- 세션 2: 범위 내 삽입 가능!
INSERT INTO phantom_test (amount) VALUES (1500);  -- 차단 없이 성공!

-- 세션 1: Phantom Read 발생
SELECT COUNT(*) FROM phantom_test WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE;
-- 원래 3건 → 삽입 후 4건으로 증가 → Phantom Read!

ROLLBACK;
```

---

## 📊 성능 비교

```
Gap Lock 있음 (RR) vs 없음 (RC) 동시성 비교:

같은 범위에 동시 INSERT 처리량:
  RC (Gap Lock 없음): 여러 트랜잭션이 같은 범위에 동시 INSERT 가능
                      → 삽입 처리량 높음
  RR (Gap Lock 있음): 같은 Gap 범위에 Sequential 삽입
                      → 삽입 처리량 낮음, Deadlock 가능성

Deadlock 빈도:
  RC: Gap Lock Deadlock 없음 (Insert Intention Lock끼리의 충돌 없음)
  RR: 같은 Gap 내 동시 INSERT 시 Deadlock 가능

Phantom Read 안전성:
  RC + Consistent Read: Phantom 가능
  RC + Current Read: Phantom 가능
  RR + Consistent Read: Phantom 없음 (Read View)
  RR + Current Read: Gap Lock으로 방지

실무 측정 (고동시성 삽입 워크로드):
  RC: TPS 약 20~30% 높음 (Gap Lock 없음)
  RR: Deadlock 재시도로 실질 TPS 감소 가능
  → 삽입이 많은 서비스: RC 고려 (Phantom 허용 가능한 경우)
  → 읽기 일관성이 중요한 서비스: RR 유지
```

---

## ⚖️ 트레이드오프

```
Phantom Read 방지 vs Gap Lock Deadlock:

Gap Lock의 장점:
  ✅ Current Read에서도 Phantom Read 방지
  ✅ Statement-based Replication에서 복제 일관성 보장

Gap Lock의 단점:
  ❌ 같은 Gap 범위에 동시 삽입 시 직렬화
  ❌ Gap Lock끼리의 충돌 → Deadlock 위험 증가
  ❌ INSERT ... ON DUPLICATE KEY UPDATE 패턴에서 특히 Deadlock 빈번

READ_COMMITTED 선택 시 추가 주의사항:
  Phantom Read 가능 → 비즈니스 로직에서 허용 여부 검토
  Statement-based Replication 사용 불가 → Row-based 필수
  (Statement 기반 복제에서 RC + 같은 시간 다른 순서 실행 → 다른 결과)

Deadlock 방지를 위한 INSERT 패턴:
  INSERT INTO orders (...) VALUES (...) ON DUPLICATE KEY UPDATE ...
  → 동시 실행 시 Gap Lock Deadlock 빈번
  해결 1: READ_COMMITTED 사용 (Phantom 허용)
  해결 2: INSERT IGNORE 사용
  해결 3: 삽입 전 SELECT FOR UPDATE로 명시적 Lock 순서 통일
```

---

## 📌 핵심 정리

```
Phantom Read 핵심:

정의:
  같은 범위 쿼리를 반복했을 때 결과 행 수가 달라지는 현상
  원인: 범위에 새 Row INSERT 또는 DELETE

읽기 방식별 동작 (REPEATABLE_READ):
  Consistent Read (일반 SELECT): Phantom 없음 (Read View 고정)
  Current Read (FOR UPDATE):     Gap Lock으로 방지

Gap Lock:
  인덱스 레코드 사이의 공간에 거는 Lock
  Insert Intention Lock (삽입 시도)과 충돌 → 삽입 차단
  동일 Gap에 여러 Gap Lock 공존 가능
  동일 Gap에 Gap Lock + Insert Intention → 충돌 → 차단

Gap Lock Deadlock 패턴:
  TRX A, B가 같은 Gap에 Gap Lock 보유
  둘 다 삽입 시도 → 서로의 Gap Lock이 차단
  → 순환 대기 → Deadlock

READ_COMMITTED:
  Gap Lock 없음 → Phantom 가능, Deadlock 감소
  Row-based Replication 필수

올바른 패턴:
  Current Read로 통일 (읽고 즉시 처리)
  Consistent Read + Current Read 혼용 금지 (Phantom 발생)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 Phantom Read가 발생할 수 있는가? 발생한다면 어떤 시나리오인지 설명하라.

```java
@Transactional  // REPEATABLE_READ
public void issueDiscountCoupons(int maxCoupons) {
    long issued = couponRepository.countIssuedToday();  // SELECT COUNT
    if (issued >= maxCoupons) throw new CouponExhaustedException();

    couponRepository.save(new Coupon(today));  // INSERT
}
```

<details>
<summary>해설 보기</summary>

**Phantom Read가 발생할 수 있습니다.** 그리고 이 Phantom Read가 실제 데이터 불일치를 일으킵니다.

시나리오:
1. 서버 A, B가 동시에 `issueDiscountCoupons(100)` 호출
2. 서버 A: `countIssuedToday() = 99` (Consistent Read, 아직 오늘 발급 99건)
3. 서버 B: `countIssuedToday() = 99` (Consistent Read, 같은 Read View 기준)
4. 서버 A: 100건 미만이므로 INSERT → COMMIT
5. 서버 B: 100건 미만이므로 INSERT → COMMIT

결과: 101번째 쿠폰이 발급됨 (maxCoupons = 100을 초과)

왜 발생하는가: `countIssuedToday()`는 일반 SELECT (Consistent Read)입니다. 서버 A의 INSERT + COMMIT이 서버 B의 COUNT와 INSERT 사이에 발생했지만, 서버 B의 COUNT는 Read View 고정으로 그 INSERT를 보지 못합니다.

올바른 해결책:
```java
// 방법 1: INSERT 전에 FOR UPDATE로 현재 상태 확인
long issued = couponRepository.countIssuedTodayForUpdate();  // FOR UPDATE
// Gap Lock 획득 → 다른 트랜잭션의 INSERT 차단

// 방법 2: DB 레벨 유니크 제약 + 예외 처리
// coupon 테이블에 날짜별 발급 순번 UNIQUE 제약
// INSERT 실패 → 발급 완료로 처리

// 방법 3: 분산 Lock (Redis 등)
// 쿠폰 발급 범위에 대한 명시적 Lock
```

</details>

---

**Q2.** `SELECT ... FOR UPDATE`를 사용했는데도 Deadlock이 발생했다. SHOW ENGINE INNODB STATUS에서 두 트랜잭션 모두 Gap Lock을 보유하고 서로 Insert Intention Lock을 기다리고 있다. 이 Deadlock을 방지하는 가장 간단한 방법은?

<details>
<summary>해설 보기</summary>

**`READ_COMMITTED` 격리 수준으로 변경**이 가장 간단한 방법입니다.

`READ_COMMITTED`에서는 Gap Lock이 없으므로 Gap Lock 간의 Deadlock이 원천 차단됩니다. Record Lock만 사용하여 실제 존재하는 Row에만 Lock을 걸게 됩니다.

단, 이 변경 전에 확인해야 할 것들:
1. **Phantom Read 허용 가능성**: 해당 비즈니스 로직에서 Phantom Read가 발생해도 괜찮은가?
2. **Replication 방식**: Statement-based Replication → Row-based로 변경 필요
3. **Statement 기반 복제 중단 없이**: `binlog_format=ROW` 설정 후 적용

또 다른 방법들:
- INSERT 순서를 통일하여 Gap Lock 범위가 겹치지 않도록 설계
- `INSERT IGNORE` 또는 `ON DUPLICATE KEY UPDATE` 대신 `SELECT FOR UPDATE` 후 조건 확인 → INSERT
- 트랜잭션 크기 최소화로 Gap Lock 보유 시간 단축

근본적 해결: 동일한 Gap에 여러 트랜잭션이 동시에 INSERT를 시도하지 않도록 설계 (예: 각 트랜잭션이 서로 다른 키 범위를 처리하도록 분산)

</details>

---

**Q3.** 다음 두 쿼리는 같은 결과를 반환하지만 Lock 패턴이 다르다. 어떤 차이가 있고 어느 것이 더 안전한가?

```sql
-- 쿼리 A
SELECT * FROM orders WHERE id IN (1, 2, 3) FOR UPDATE;

-- 쿼리 B
SELECT * FROM orders WHERE id BETWEEN 1 AND 3 FOR UPDATE;
```

<details>
<summary>해설 보기</summary>

두 쿼리가 반환하는 Row 집합은 같더라도 Lock 범위는 다릅니다.

**쿼리 A** (`id IN (1, 2, 3)`):
- 각 값에 대해 Point Lookup → id=1, 2, 3에 대한 Record Lock
- PK 등치 조건 → `REC_NOT_GAP` Lock (Gap Lock 없음)
- 다른 id에 대한 삽입/삭제는 자유롭게 가능

**쿼리 B** (`id BETWEEN 1 AND 3`):
- 범위 조건 → id=1, 2, 3에 Record Lock
- **Gap Lock 추가**: (이전 레코드, 1], (1, 2], (2, 3], (3, 이후) 범위에 Gap Lock
- 이 범위 내 새 Row 삽입 차단

**어느 것이 더 안전한가**:
- **Phantom Read 방지**: 쿼리 B가 더 안전 (Gap Lock으로 범위 내 삽입 차단)
- **Deadlock 위험**: 쿼리 B가 더 위험 (Gap Lock 범위가 넓어 다른 트랜잭션과 충돌 가능)
- **성능**: 쿼리 A가 더 효율적 (Gap Lock 없어 삽입 동시성 높음)

실무 선택: 정확히 알려진 PK 집합을 조회할 때는 쿼리 A (IN 조건) 사용이 Lock 범위를 최소화하여 Deadlock 위험 감소. 범위 조건이 반드시 필요할 때만 BETWEEN 사용.

</details>

---

<div align="center">

**[⬅️ 격리 수준](./04-isolation-levels.md)** | **[홈으로 🏠](../README.md)** | **[다음: Consistent vs Current Read ➡️](./06-consistent-read-vs-current-read.md)**

</div>
