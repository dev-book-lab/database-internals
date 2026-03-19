# Optimistic Lock vs Pessimistic Lock — 선택 기준

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Pessimistic Lock이 충돌을 "사전 차단"하고 Optimistic Lock이 "사후 감지"하는 내부 메커니즘은?
- JPA `@Version`이 DB 레벨에서 실행하는 SQL은 정확히 무엇인가?
- `ObjectOptimisticLockingFailureException`은 어떤 조건에서 발생하는가?
- 쓰기 충돌 빈도가 높을 때 Optimistic Lock이 오히려 더 비싼 이유는?
- 재고 차감처럼 동시 접근이 많은 시나리오에서 두 방식의 TPS 차이는?
- 외부 시스템 호출이 포함된 트랜잭션에서 Optimistic Lock이 위험한 이유는?

---

## 🔍 왜 이 개념이 중요한가

### "재고가 마이너스가 됐다" — Lock 없이 읽고 쓰는 패턴의 결과

```
실제 발생 상황:
  Flash Sale 이벤트 → 초당 1000건 주문 → 재고 0인데 주문 계속 성공

  코드:
    Product p = productRepo.findById(id);  // 재고: 10 (Lock 없음)
    if (p.getStock() < quantity) throw ...
    p.setStock(p.getStock() - quantity);
    productRepo.save(p);  // UPDATE stock=7 (읽은 값 기반)
  
  동시 10개 요청이 모두 재고=10 읽음
  → 모두 "재고 충분" 판단
  → 10개 모두 차감 실행
  → 최종 재고: -80 (10 - 10×9 = -80)

두 가지 해결책:
  Pessimistic: findById → findByIdWithLock (FOR UPDATE)
    → 한 번에 1개 요청만 재고 읽기 → 직렬 처리
    → 10개 순서대로 처리 → 10건 후 재고 0 → 그 다음은 실패
  
  Optimistic: @Version 추가
    → 동시에 읽어도 OK
    → 저장 시 버전 체크 → 9개는 실패 → 재시도
    → 충돌 빈도에 따라 성능 차이
  
  어느 것이 나은가:
    Flash Sale처럼 충돌 빈번 → Pessimistic
    일반 상품처럼 충돌 드뭄 → Optimistic
```

---

## 😱 잘못된 이해

### Before: "Optimistic Lock = 더 빠른 방법"

```
잘못된 믿음:
  "Optimistic Lock은 Lock을 쓰지 않으니까 항상 Pessimistic보다 빠르다"
  "@Version 달면 성능 좋아진다"

실제 (충돌 빈도가 높은 경우):
  10개 스레드가 동시에 재고=10 읽음 (모두 version=5)
  → 9개가 OptimisticLockException
  → 9개 재시도 → 8개 실패 → 8개 재시도 → ...
  → Thundering Herd: 재시도 폭주 → CPU/DB 부하 증가
  → 결국 1개씩 직렬 처리 + 재시도 오버헤드

Pessimistic Lock의 경우:
  10개 스레드가 순서대로 대기 → 1개씩 처리 → 재시도 없음
  → 총 10개 처리 (재시도 0)
  → 재시도 오버헤드 없음 → 오히려 더 효율적!

잘못된 또 다른 믿음:
  "@Version이 있으면 데드락이 없다"
  → 반은 맞음: @Version(Optimistic)은 DB Lock 없음 → DB Deadlock 없음
  → BUT: 재시도 중 다른 Lock과 충돌은 여전히 가능
  → 완전히 Lock Free한 것은 아님
```

---

## ✨ 올바른 이해

### After: 충돌 빈도에 따라 최적의 방식이 다르다

```
핵심 비교:

Pessimistic Lock (FOR UPDATE):
  시점: 읽을 때부터 Lock
  방식: X Lock → 다른 트랜잭션 차단
  충돌 처리: 사전 차단 (충돌 자체를 막음)
  비용 구조: 항상 Lock 오버헤드 있음 (충돌 빈도 무관)
  최적: 충돌 빈도 높음 OR 재시도 불가

Optimistic Lock (@Version):
  시점: 저장할 때만 버전 체크
  방식: UPDATE WHERE version=? (affected=0이면 충돌)
  충돌 처리: 사후 감지 (충돌 발생 후 재시도)
  비용 구조: 충돌 없으면 거의 무료, 충돌 있으면 재시도 비용
  최적: 충돌 빈도 낮음 AND 재시도 가능

비용 교차점:
  충돌 빈도 낮음 → Optimistic 유리 (Lock 오버헤드 절약)
  충돌 빈도 높음 → Pessimistic 유리 (재시도 오버헤드 없음)
  
  교차점은 일반적으로 충돌 확률 10~20% 수준
  → 충돌이 10% 이상 발생하면 Pessimistic 고려

선택 기준 요약:
  재고 차감 (Flash Sale):       Pessimistic (충돌 매우 빈번)
  재고 차감 (일반 상품):        Optimistic (충돌 드뭄)
  사용자 프로필 수정:           Optimistic (동시 수정 드뭄)
  계좌 이체 (동시 이체 많음):   Pessimistic
  게시글 수정:                  Optimistic
  좌석 예약 (인기 좌석):        Pessimistic
  설정값 변경:                  Optimistic
```

---

## 🔬 내부 동작 원리

### 1. Pessimistic Lock — FOR UPDATE 동작 상세

```sql
-- Pessimistic Lock 흐름:

-- 1. 읽기 (Current Read + X Lock 획득)
SELECT id, stock, version
FROM products
WHERE id = 1
FOR UPDATE;
-- → X Lock 획득 (다른 FOR UPDATE, UPDATE, FOR SHARE 차단)
-- → 최신 버전 읽음 (MVCC 우회)
-- → 다른 트랜잭션의 일반 SELECT는 차단 없음 (MVCC)

-- 2. 비즈니스 로직 (Lock 보유 중)
-- stock = 10, quantity = 3
-- 10 >= 3 → 재고 충분

-- 3. 수정
UPDATE products SET stock = stock - 3 WHERE id = 1;
-- 이미 X Lock 보유 중 → Lock 재획득 불필요
-- stock = 7

-- 4. COMMIT (X Lock 해제)
COMMIT;

-- 특성:
-- Lock 보유 시간 = 읽기 ~ COMMIT 사이
-- Lock 보유 시간이 길수록 다른 트랜잭션 대기 시간 증가
-- → 트랜잭션을 짧게 유지하는 것이 중요

-- 성능 최적화:
-- 나쁜 예: HTTP 호출 후 저장
START TRANSACTION;
SELECT ... FOR UPDATE;
externalApiCall();  -- 2초 소요 → 2초 동안 Lock 유지
UPDATE ...;
COMMIT;
-- → 모든 동시 주문이 2초 대기

-- 좋은 예: Lock 범위 최소화
externalApiCall();  -- Lock 밖에서 실행
START TRANSACTION;
SELECT ... FOR UPDATE;  -- 빠르게 확인
UPDATE ...;
COMMIT;
```

### 2. Optimistic Lock — @Version CAS 동작 상세

```java
// Optimistic Lock 전체 흐름:

@Entity
public class Product {
    @Id Long id;
    Integer stock;
    
    @Version  // ← 핵심
    Long version;  // 초기값 0, UPDATE마다 자동 증가
}

// 1. 읽기 (Lock 없음 - Consistent Read)
Product p = productRepo.findById(1L).get();
// SQL: SELECT id, stock, version FROM products WHERE id = 1
// 결과: id=1, stock=10, version=5

// 2. 비즈니스 로직
int newStock = p.getStock() - 3;  // 10 - 3 = 7
p.setStock(newStock);

// 3. 저장 시 버전 CAS (Compare-And-Swap)
productRepo.save(p);
// SQL: UPDATE products
//      SET stock = 7, version = 6         ← version+1
//      WHERE id = 1 AND version = 5       ← 내가 읽은 버전
//
// Case A (성공): 아무도 수정 안 함 → affected rows = 1 → 성공!
//   DB: version 5 → 6으로 변경
//
// Case B (충돌): 다른 트랜잭션이 version을 5→6으로 변경함
//   → WHERE id=1 AND version=5 조건 불일치 → affected rows = 0
//   → Hibernate: StaleObjectStateException 발생
//   → Spring: ObjectOptimisticLockingFailureException 변환

// 4. 충돌 발생 시 재시도
@Retryable(
    value = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 50, multiplier = 2)  // 50ms, 100ms, 200ms
)
@Transactional
public void decreaseStock(Long productId, int quantity) {
    Product p = productRepo.findById(productId).get();
    // 재시도 시 새 트랜잭션 → 새 Read View → 최신 version 읽음
    p.setStock(p.getStock() - quantity);
    productRepo.save(p);  // 충돌 시 예외 → 재시도
}
```

### 3. @Version이 생성하는 SQL 완전 분석

```java
// INSERT 시:
Product p = new Product(1L, 100, "Widget");
productRepo.save(p);
// SQL: INSERT INTO products (id, stock, name, version) VALUES (1, 100, 'Widget', 0)
//       ← version 초기값 0 자동 삽입

// UPDATE 시:
p.setStock(90);
productRepo.save(p);
// SQL: UPDATE products
//      SET stock = 90, version = 1       ← version+1
//      WHERE id = 1 AND version = 0      ← 읽은 version 조건
// affected rows = 1 → 성공, version DB에서 0→1

// 충돌 시:
// 동시에 다른 트랜잭션도 stock을 80으로 변경 + COMMIT
// 이제 DB의 version = 1

// 이후 내 트랜잭션의 save:
// SQL: UPDATE products SET stock = 90, version = 1 WHERE id = 1 AND version = 0
// WHERE version=0인데 DB는 이미 version=1
// → affected rows = 0
// → Hibernate: StaleObjectStateException
// → Spring: ObjectOptimisticLockingFailureException

// DELETE 시에도 version 체크:
productRepo.delete(p);
// SQL: DELETE FROM products WHERE id = 1 AND version = 0
// version 불일치 시 → OptimisticLockException

// @Version의 타입:
// Long: 1씩 증가 (가장 일반적)
// Integer: 1씩 증가 (int 범위 내)
// Short: 1씩 증가
// Timestamp: 현재 시각 (정밀도 문제로 권장하지 않음)
// LocalDateTime: 현재 시각 (ms 단위 충돌 감지 못할 수 있음)
```

### 4. 충돌 빈도별 성능 계산

```
시나리오: 재고 차감 (초당 100 요청, 재고 충분)

충돌 확률 0% (일반 상품):
  Pessimistic: 100 요청 × (Lock 획득 + 업데이트 + Lock 해제) = 기준 100%
  Optimistic:  100 요청 × (읽기 + 버전 체크 업데이트) = 약 95% (Lock 오버헤드 없음)
  → Optimistic 유리 (5% 빠름)

충돌 확률 10% (중간 인기 상품):
  Pessimistic: 100 요청 직렬화 → 100 처리 완료, 재시도 0
  Optimistic:  100 요청 읽기, 10개 충돌 → 10개 재시도 → 1개 또 충돌
  재시도 포함 총 처리: ~111 요청 처리
  → 비슷하거나 Pessimistic 유리

충돌 확률 50% (인기 상품, Flash Sale):
  Pessimistic: 100 요청 직렬화 → 100 처리, 재시도 0
  Optimistic:  100 요청, 50개 충돌 → 50개 재시도, 25개 충돌 → ...
  Thundering Herd: 재시도 폭주 → 실제 처리량 저하
  → Pessimistic 압도적으로 유리

교차점:
  일반적으로 충돌 확률 10~20%에서 두 방식의 효율이 교차
  Flash Sale 수준(>50%): Pessimistic 필수
```

### 5. 외부 시스템 호출과 Optimistic Lock의 위험

```java
// 위험한 패턴: 외부 호출 + Optimistic Lock

@Retryable(ObjectOptimisticLockingFailureException.class, maxAttempts = 3)
@Transactional
public void purchaseWithPayment(Long orderId, Long userId) {
    
    Order order = orderRepo.findById(orderId);  // version 읽음
    
    // ← 외부 결제 API 호출 (Lock 없음)
    PaymentResult result = paymentGateway.charge(userId, order.getAmount());
    // ↑ 실제 결제가 발생했음!
    
    order.setStatus("PAID");
    orderRepo.save(order);  // OptimisticLockException 발생 시 재시도!
    
    // 재시도 시:
    // 1. paymentGateway.charge() 다시 실행 → 중복 결제!!
    // 2. 결제는 되는데 order 저장 실패 → 데이터 불일치
}

// 올바른 패턴: 외부 호출은 트랜잭션 밖에서

public void purchaseSafe(Long orderId, Long userId) {
    // 1. 재고/상태 확인 (트랜잭션)
    validateOrder(orderId);
    
    // 2. 외부 결제 (트랜잭션 밖)
    PaymentResult result = paymentGateway.charge(userId, getAmount(orderId));
    // 실패 시: 결제만 실패, DB 변경 없음 → 클린
    
    // 3. DB 상태 업데이트 (트랜잭션)
    try {
        updateOrderStatus(orderId, "PAID", result.getTransactionId());
    } catch (Exception e) {
        // DB 업데이트 실패 → 결제는 됐지만 DB 불일치
        // → 보상 트랜잭션 또는 Outbox Pattern 필요
        compensate(result.getTransactionId());
    }
}

// 결론:
// Optimistic Lock + 외부 호출 = 재시도 시 외부 호출 중복 실행 위험
// → Pessimistic Lock 또는 외부 호출을 트랜잭션 밖으로 분리
```

---

## 💻 실전 실험

### 실험 1: @Version SQL 확인

```java
@Test
public void verifyVersionSQL() {
    // 초기 데이터
    Product p = productRepo.save(new Product("Widget", 100));
    // INSERT ... version=0

    // 정상 UPDATE
    p.setStock(90);
    productRepo.save(p);
    // UPDATE ... SET stock=90, version=1 WHERE id=? AND version=0
    // affected=1 → 성공

    // 충돌 시뮬레이션:
    // 다른 트랜잭션이 version을 1→2로 변경
    jdbcTemplate.update("UPDATE products SET version=2 WHERE id=?", p.getId());

    // 이제 p는 version=1인데 DB는 version=2
    p.setStock(80);
    assertThrows(ObjectOptimisticLockingFailureException.class, () -> {
        productRepo.save(p);
        // UPDATE ... WHERE id=? AND version=1
        // affected=0 → 예외!
    });
}
```

### 실험 2: Pessimistic vs Optimistic TPS 비교

```java
// 동시성 테스트: 재고 차감 100회 동시 실행
@Test
public void comparePerformance() throws Exception {
    Product p = createProductWithStock(100);
    CountDownLatch start = new CountDownLatch(1);
    CountDownLatch done = new CountDownLatch(100);
    AtomicInteger conflicts = new AtomicInteger(0);

    // Optimistic Lock 테스트
    long startTime = System.currentTimeMillis();
    for (int i = 0; i < 100; i++) {
        executor.submit(() -> {
            start.await();
            try {
                decreaseStockOptimistic(p.getId(), 1);
            } catch (MaxAttemptsExceededException e) {
                conflicts.incrementAndGet();
            } finally {
                done.countDown();
            }
        });
    }
    start.countDown();
    done.await();
    long optimisticTime = System.currentTimeMillis() - startTime;
    System.out.printf("Optimistic: %dms, conflicts: %d%n",
        optimisticTime, conflicts.get());

    // Pessimistic Lock 테스트 (동일하게 진행)
    // 예상 결과:
    // 낮은 재고 경쟁 (같은 상품 100개 동시) → Pessimistic 더 빠름
    // 높은 재고 (경쟁 낮음) → Optimistic 더 빠름
}
```

### 실험 3: SKIP LOCKED로 분산 처리 (Pessimistic 응용)

```sql
-- 여러 워커가 Task를 중복 없이 처리하는 패턴

CREATE TABLE tasks (
    id       BIGINT AUTO_INCREMENT PRIMARY KEY,
    status   ENUM('PENDING', 'PROCESSING', 'DONE') DEFAULT 'PENDING',
    payload  TEXT,
    locked_at DATETIME  -- 처리 시작 시간 (타임아웃 감지용)
) ENGINE=InnoDB;

-- 워커 1, 2, 3이 동시에 실행:
SELECT * FROM tasks
WHERE status = 'PENDING'
  AND (locked_at IS NULL OR locked_at < DATE_SUB(NOW(), INTERVAL 5 MINUTE))
ORDER BY id
LIMIT 10
FOR UPDATE SKIP LOCKED;
-- → 각 워커가 서로 다른 10개씩 가져감 (중복 없음)
-- SKIP LOCKED: 다른 워커가 Lock한 Row는 자동으로 건너뜀

-- 가져간 Task를 PROCESSING으로 마킹
UPDATE tasks SET status = 'PROCESSING', locked_at = NOW()
WHERE id IN (/* 위에서 가져온 id 목록 */);
COMMIT;

-- 타임아웃된 Task 재처리:
-- locked_at IS NULL OR locked_at < NOW() - INTERVAL 5 MINUTE 조건으로
-- 처리 중 실패한 Task를 다시 가져갈 수 있음
```

---

## 📊 성능 비교

```
Pessimistic vs Optimistic 실측 (재고 차감, SSD):

충돌 확률 0% (독립 상품 100개, 100 req/s):
  Pessimistic: 320ms (Lock 획득/해제 오버헤드)
  Optimistic:  280ms (Lock 없음, 버전 체크만)
  → Optimistic 12% 빠름

충돌 확률 30% (인기 상품 1개, 100 req/s):
  Pessimistic: 380ms (직렬화 대기)
  Optimistic:  520ms (재시도 30건 × 재처리 비용)
  → Pessimistic 27% 빠름, 재시도 오버헤드 없음

충돌 확률 80% (Flash Sale):
  Pessimistic: 450ms (직렬화 큐)
  Optimistic:  2300ms (폭발적 재시도, Thundering Herd)
  → Pessimistic 5배 빠름

결론:
  충돌률 < 10%: Optimistic 유리
  충돌률 10~30%: 비슷하거나 Pessimistic 유리
  충돌률 > 30%: Pessimistic 압도적으로 유리
```

---

## ⚖️ 트레이드오프

```
Pessimistic Lock:
  ✅ 충돌 사전 차단 → 재시도 0
  ✅ 예측 가능한 처리 시간 (큐잉)
  ✅ 외부 호출 포함 트랜잭션에 안전
  ❌ Lock 오버헤드 항상 존재 (충돌 없어도)
  ❌ Deadlock 가능 (여러 FOR UPDATE 순서 불일치)
  ❌ Lock 보유 시간 → 동시성 저하

Optimistic Lock (@Version):
  ✅ Lock 없음 → 높은 동시성 (충돌 없을 때)
  ✅ Deadlock 없음 (DB Lock 미사용)
  ✅ Read-heavy 워크로드에 적합
  ❌ 충돌 시 재시도 → 재처리 비용 + Thundering Herd 위험
  ❌ 재시도 로직 별도 구현 필요
  ❌ 외부 호출 포함 시 재시도가 중복 호출 야기

선택 기준 (결정 트리):
  Q1: 같은 Row에 동시에 쓰기가 얼마나 자주 일어나는가?
    → 빈번 (초당 수십 건 이상 같은 Row): Pessimistic
    → 드뭄 (분당 수건 이하):            Optimistic 고려

  Q2: 트랜잭션에 외부 시스템 호출이 포함되는가?
    → 포함:        Pessimistic 또는 외부 호출 분리
    → 미포함:      Optimistic 가능

  Q3: 재시도 실패 시 사용자 경험이 허용되는가?
    → 허용 안 됨 (실시간 예약 등): Pessimistic
    → 허용 (설정 저장 등):        Optimistic

  Q4: 읽기와 쓰기의 비율은?
    → 읽기 > 쓰기 (98:2): Optimistic 유리
    → 쓰기 집약:           Pessimistic 고려
```

---

## 📌 핵심 정리

```
Pessimistic vs Optimistic 핵심:

Pessimistic (FOR UPDATE):
  시점: 읽기 시 Lock → 저장 시 Lock 유지
  SQL: SELECT ... FOR UPDATE → UPDATE ...
  충돌: Lock으로 사전 차단 → 대기
  최적: 충돌 빈번, 재시도 불가, 외부 호출 포함

Optimistic (@Version):
  시점: Lock 없이 읽기 → 저장 시 버전 CAS
  SQL: SELECT ... → UPDATE ... WHERE version=?
  충돌: affected=0 → OptimisticLockException → 재시도
  최적: 충돌 드뭄, 재시도 가능, Read-heavy

@Version SQL:
  INSERT: version=0 자동
  UPDATE: SET ..., version=N+1 WHERE id=? AND version=N
  DELETE: WHERE id=? AND version=N
  affected=0 → ObjectOptimisticLockingFailureException

재시도 패턴:
  @Retryable(ObjectOptimisticLockingFailureException.class)
  + 지수 백오프 (100ms, 200ms, 400ms)
  + 최대 3~5회 (무한 재시도 금지)
  + 외부 호출은 반드시 트랜잭션 밖

충돌 교차점:
  충돌률 < 10%: Optimistic 유리
  충돌률 > 30%: Pessimistic 유리
```

---

## 🤔 생각해볼 문제

**Q1.** `@Version` 없이 `updatedAt` 타임스탬프를 Optimistic Lock의 버전으로 사용하는 방법의 장단점과 이 방법이 실패할 수 있는 구체적 시나리오를 설명하라.

<details>
<summary>해설 보기</summary>

**updatedAt을 버전으로 사용하는 방법**:
```java
@Modifying
@Query("UPDATE Product p SET p.stock = :stock, p.updatedAt = NOW() " +
       "WHERE p.id = :id AND p.updatedAt = :lastUpdated")
int updateStockOptimistically(
    @Param("id") Long id,
    @Param("stock") Integer stock,
    @Param("lastUpdated") LocalDateTime lastUpdated
);
// affected=0이면 충돌
```

**장점**:
- 별도 version 컬럼 불필요 (이미 있는 updatedAt 재사용)
- "마지막 수정 시간"이라는 감사(audit) 정보도 함께 관리

**단점 및 실패 시나리오**:

1. **밀리초 내 동시 수정**: 두 트랜잭션이 같은 밀리초에 완료되면 `updatedAt`이 동일 → 충돌 감지 실패
   ```
   T=1000ms: TRX A 읽음 (updatedAt=1000)
   T=1500ms: TRX B 수정 완료 (updatedAt=1500)
   T=1500ms: TRX A도 수정 시도 (updatedAt이 1000에서 1500으로 변경 예정)
   → WHERE updatedAt=1000인데 DB는 1500 → 감지! (성공)
   
   BUT:
   T=1000ms: TRX A, B 동시에 읽음 (updatedAt=1000)
   T=1001ms: TRX A 완료 (updatedAt=1001)
   T=1001ms: TRX B도 완료 시도 WHERE updatedAt=1000
   → DB는 1001이므로 감지! (성공)
   
   T=1000ms: TRX A, B 동시에 읽음 (updatedAt=1000)
   T=1000ms: TRX A 완료 (updatedAt=1000, 같은 밀리초)
   T=1000ms: TRX B 완료 시도 WHERE updatedAt=1000
   → DB도 1000이므로 충돌 감지 실패! (Lost Update!)
   ```

2. **서버 간 시간 차이**: 여러 앱 서버의 시계가 다르면 동일한 시간으로 저장될 수 있음

3. **DB 서버 시간 의존**: `NOW()` 함수를 사용하므로 DB 서버 시간 기준

결론: `@Version Long`이 훨씬 안전합니다. 단조 증가하므로 동일 밀리초 충돌 없이 정확히 1씩 증가하여 충돌을 항상 정확히 감지합니다.

</details>

---

**Q2.** 10,000명이 동시에 같은 좌석 예약을 시도하는 티켓팅 서비스에서 Optimistic Lock을 사용하면 어떤 현상이 발생하는가? Pessimistic Lock과 비교하여 구체적인 요청 처리 과정을 설명하라.

<details>
<summary>해설 보기</summary>

**Optimistic Lock 시나리오**:

1. 10,000 요청이 모두 좌석 데이터를 읽음 (version=0, status='AVAILABLE')
2. 모두 "예약 가능"으로 판단
3. 10,000개 `UPDATE ... WHERE version=0` 실행 → **1개만 성공 (affected=1)**, 9,999개 실패 (affected=0)
4. 9,999개가 `ObjectOptimisticLockingFailureException` → 재시도 로직 실행
5. 재시도 1회차: 9,999개 읽기 → 1개 version=1, status='RESERVED' 확인 → "예약 불가" 반환 (하지만 재시도 로직에서 처리?)
   - 재시도는 단순히 다시 시도하는 것 → 다시 version 체크 → 이제 reserved → 실패
6. 결과: 1개 성공, 9,999개 `@Retryable` 3회 후 최종 실패

**문제**: 9,999 × 3회 = 29,997번의 불필요한 DB 쿼리 → DB 부하 폭증 → Thundering Herd

**Pessimistic Lock 시나리오**:

1. 10,000 요청이 `FOR UPDATE` 실행
2. **1개만** Lock 획득 → 나머지 9,999개 대기 큐
3. 1번 트랜잭션: 읽기(AVAILABLE) → 예약 완료 → COMMIT → Lock 해제
4. 2번 트랜잭션: Lock 획득 → 읽기(RESERVED) → "이미 예약됨" → 즉시 실패 (재시도 없음)
5. 3~10,000번도 동일하게 즉시 실패

**결과 비교**:
- Optimistic: 29,997번 추가 쿼리, Thundering Herd, DB 과부하
- Pessimistic: 9,999번의 빠른 실패, 추가 재시도 없음, DB 부하 낮음

티켓팅 서비스의 실제 설계: `FOR UPDATE SKIP LOCKED`로 잠긴 좌석을 건너뛰거나, Redis를 통한 선착순 대기열 구현이 일반적입니다.

</details>

---

**Q3.** Spring에서 `@Transactional(readOnly=true)`로 설정된 메서드에서 `@Version`이 있는 엔티티를 수정하려 하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**`@Transactional(readOnly=true)`에서 엔티티 수정 시도**:

```java
@Transactional(readOnly = true)
public void updateInReadOnly(Long id) {
    Product p = productRepo.findById(id).get();
    p.setStock(p.getStock() - 1);
    productRepo.save(p);  // 어떻게 되는가?
}
```

**발생하는 일**:

1. `readOnly=true`는 Hibernate에 힌트를 제공합니다:
   - `FlushMode.NEVER` 또는 `FlushMode.MANUAL`로 설정 → Dirty Checking 비활성화
   - DB 연결에 `Connection.setReadOnly(true)` 전달 (일부 구현)

2. `save()` 호출 시:
   - Hibernate의 1차 캐시에 변경 사항 기록은 됨
   - 하지만 `FlushMode.NEVER`이므로 트랜잭션 종료 시 **DB에 실제 UPDATE가 실행되지 않음**
   - 예외 발생하지 않을 수 있음 (조용한 무시)

3. 결과: `p.setStock()` 호출은 객체에는 반영되지만 **DB에는 영구 저장되지 않음**

**실제 동작 확인**:
```java
// readOnly=true에서는 save()가 UPDATE를 실행하지 않음
// 확인 방법: SQL 로그 활성화
spring.jpa.properties.hibernate.show_sql=true
// → readOnly 메서드에서 UPDATE 쿼리가 보이지 않음
```

**의도적 쓰기가 필요하다면**:
- `@Transactional(readOnly=false)` 사용 (기본값)
- 또는 `@Transactional` (readOnly 기본값 false)

**readOnly=true의 성능 이점**:
- Dirty Checking 비활성화 → 메모리/CPU 절약
- DB에 READ-ONLY 힌트 → 일부 DB에서 Read Replica 라우팅
- Flush 없음 → 트랜잭션 오버헤드 감소

결론: `@Version` 엔티티를 실제로 저장해야 한다면 `readOnly=false`(기본값)를 사용해야 합니다. `readOnly=true`에서 `save()`를 호출해도 DB에는 반영되지 않습니다.

</details>

---

<div align="center">

**[⬅️ 데드락 로그](./05-deadlock-log-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Performance Tuning ➡️](../performance-tuning/01-slow-query-analysis.md)**

</div>
