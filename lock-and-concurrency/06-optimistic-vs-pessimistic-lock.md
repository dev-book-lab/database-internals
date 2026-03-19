# Optimistic Lock vs Pessimistic Lock — 선택 기준

---

## 🎯 핵심 질문

- Pessimistic Lock이 충돌을 "사전 차단"하고 Optimistic Lock이 "사후 감지"하는 구체적 메커니즘은?
- JPA `@Version`이 DB 레벨에서 어떤 SQL로 변환되는가?
- 충돌 빈도에 따라 어떤 방식이 더 효율적인가?
- Optimistic Lock에서 `OptimisticLockException` 발생 시 재시도 전략은?
- Optimistic Lock은 데이터베이스 계층 없이도 구현 가능한가?

---

## 🔬 내부 동작 원리

### 1. Pessimistic Lock — 사전 차단

```
원리:
  "충돌이 발생할 것이라고 가정"
  → 데이터를 읽을 때부터 Lock 획득
  → 다른 트랜잭션의 접근 차단

동작:
  SELECT ... FOR UPDATE (X Lock)
  → 읽은 시점부터 COMMIT까지 다른 쓰기 차단
  → 수정할 때 항상 최신값 보장

SQL:
  SELECT balance FROM accounts WHERE id = ? FOR UPDATE
  [업무 로직]
  UPDATE accounts SET balance = ? WHERE id = ?
  COMMIT

비용:
  Lock 경합: 동시 접속자 증가 → 대기 시간 증가
  처리량 저하: Lock 보유 시간 동안 다른 트랜잭션 차단
  데드락 가능성: 여러 Row Lock 시 순서 불일치 시

적합한 상황:
  쓰기 충돌 빈도가 높음 (같은 Row를 여러 트랜잭션이 동시에 수정)
  충돌 시 재시도 비용이 큼 (복잡한 처리 후 처음부터)
  외부 시스템 호출 포함 (HTTP 요청 등) → 롤백 불가
```

### 2. Optimistic Lock — 사후 감지

```
원리:
  "충돌이 없을 것이라고 가정"
  → Lock 없이 읽기
  → 저장 시 "내가 읽은 이후 변경됐는가?" 확인
  → 변경됐으면 실패 (재시도 or 에러)

구현 방식 (버전 컬럼):
  테이블에 version 컬럼 추가 (BIGINT, 초기값 0)
  
  읽기:
    SELECT id, balance, version FROM accounts WHERE id = ?
    → balance=10000, version=5

  쓰기:
    UPDATE accounts
    SET balance = ?, version = version + 1
    WHERE id = ? AND version = 5  ← 내가 읽은 버전과 동일해야 함
  
  성공 (affected rows = 1):
    → 내가 읽은 이후 변경 없음 → 정상 커밋
  
  실패 (affected rows = 0):
    → 다른 트랜잭션이 먼저 수정 (version이 5가 아님)
    → OptimisticLockException

비용:
  충돌 없음: Lock 없이 빠름 (추가 비용 ≈ 0)
  충돌 발생: 전체 처리 재시도 (재시도 비용)
```

### 3. JPA @Version 구현

```java
@Entity
public class Account {
    @Id
    private Long id;
    
    private BigDecimal balance;
    
    @Version
    private Long version;  // Optimistic Lock 버전
}

// 읽기
Account account = accountRepository.findById(1L).orElseThrow();
// SQL: SELECT id, balance, version FROM accounts WHERE id = 1
// balance=10000, version=5

// 수정
account.setBalance(account.getBalance().subtract(new BigDecimal("3000")));
accountRepository.save(account);
// SQL: UPDATE accounts
//      SET balance = 7000, version = 6
//      WHERE id = 1 AND version = 5  ← 핵심!

// 동시에 다른 트랜잭션이 먼저 수정했다면:
// affected rows = 0 → Hibernate가 OptimisticLockException 발생

// 재시도 패턴:
@Retryable(
    value = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 100)  // 100ms 후 재시도
)
@Transactional
public void withdraw(Long accountId, BigDecimal amount) {
    Account account = accountRepository.findById(accountId).orElseThrow();
    account.withdraw(amount);  // 비즈니스 규칙 포함
    accountRepository.save(account);
}

// @Version 없이 직접 구현:
@Modifying
@Query("UPDATE Account a SET a.balance = :newBalance, a.version = :newVersion " +
       "WHERE a.id = :id AND a.version = :oldVersion")
int updateWithVersion(@Param("id") Long id,
                      @Param("newBalance") BigDecimal newBalance,
                      @Param("newVersion") Long newVersion,
                      @Param("oldVersion") Long oldVersion);

// affected rows = 0이면 충돌 감지
```

### 4. 충돌 빈도에 따른 선택 기준

```
판단 기준:

낮은 충돌 빈도 (같은 Row를 동시에 수정하는 경우가 드문):
  → Optimistic Lock
  이유: Lock 오버헤드 없이 대부분의 경우 빠름
       충돌 시 재시도 비용이 있지만 빈도가 낮아 총 비용 적음
  예: 사용자 프로필 수정, 설정 변경, 게시글 수정

높은 충돌 빈도 (핫한 Row를 동시에 많은 트랜잭션이 수정):
  → Pessimistic Lock
  이유: 계속 실패-재시도하면 Optimistic이 더 비쌈
       Lock으로 직렬화하여 확실히 처리
  예: 인기 상품 재고 차감, 좌석 예약, 계좌 이체

외부 시스템 포함 트랜잭션:
  → Pessimistic Lock (또는 보상 트랜잭션)
  이유: HTTP 호출 포함 트랜잭션은 롤백 후 재시도가 어려움
       외부 결제 시스템 호출 → 취소/환불 로직 복잡
  → 처음부터 Lock으로 차단하거나 Saga 패턴 사용

Optimistic Lock 재시도 전략:
  고정 대기: 100ms → 재시도 (간단하지만 thundering herd 가능)
  지수 백오프: 100ms → 200ms → 400ms (더 안전)
  최대 재시도: 3~5회 (무한 재시도 금지)
  재시도 실패 시: 사용자에게 명시적 에러 반환
```

### 5. Optimistic Lock without Database

```java
// DB 컬럼 없이 Application-level Optimistic Lock
// Redis를 활용한 버전 관리:

public boolean updateWithOptimisticLock(String key, Function<String, String> updater) {
    for (int attempt = 0; attempt < 3; attempt++) {
        // 현재 값 읽기 (version 포함)
        String current = redisTemplate.opsForValue().get(key);
        String updated = updater.apply(current);
        
        // CAS (Compare-And-Swap) 시도
        boolean success = redisTemplate.execute(
            luaScript,  -- WATCH key; MULTI; SET key updated; EXEC
            Collections.singletonList(key),
            current, updated
        );
        
        if (success) return true;
        
        // 충돌 → 잠시 대기 후 재시도
        Thread.sleep(50 * (attempt + 1));
    }
    return false;  // 재시도 초과
}

// DB의 timestamp를 version으로 활용:
@Query("UPDATE Product p SET p.stock = p.stock - :qty, p.updatedAt = NOW() " +
       "WHERE p.id = :id AND p.updatedAt = :lastUpdated AND p.stock >= :qty")
int decreaseStockOptimistically(
    @Param("id") Long id,
    @Param("qty") int qty,
    @Param("lastUpdated") LocalDateTime lastUpdated
);
// → updatedAt 컬럼이 version 역할
```

---

## 💻 실전 실험

```java
// JPA Optimistic Lock 충돌 재현
@Test
@Transactional
public void testOptimisticLockConflict() throws Exception {
    // 초기 데이터
    Account account = new Account(1L, new BigDecimal("10000"), 0L);
    accountRepository.save(account);
    
    // 트랜잭션 1에서 읽기
    Account a1 = accountRepository.findById(1L).get();  // version=0
    
    // 트랜잭션 2에서 먼저 수정 (별도 트랜잭션)
    updateInNewTransaction(1L, new BigDecimal("8000"), 1L);  // version=1
    
    // 트랜잭션 1에서 저장 시도
    a1.setBalance(new BigDecimal("7000"));
    
    assertThrows(OptimisticLockingFailureException.class, () -> {
        accountRepository.saveAndFlush(a1);
        // WHERE id=1 AND version=0 → affected rows=0 (이미 version=1)
        // → OptimisticLockException!
    });
}
```

```sql
-- DB 레벨에서 확인
SELECT id, balance, version FROM accounts WHERE id = 1;
-- 두 UPDATE 동시 실행:
-- T1: UPDATE accounts SET balance=7000, version=1 WHERE id=1 AND version=0
-- T2: UPDATE accounts SET balance=8000, version=1 WHERE id=1 AND version=0
-- 먼저 실행된 것 성공, 나중 것: affected_rows=0 → 충돌!
```

---

## 📌 핵심 정리

```
Optimistic vs Pessimistic 핵심:

Pessimistic Lock:
  SELECT ... FOR UPDATE → X Lock
  COMMIT까지 다른 쓰기 차단
  충돌 빈도 높을 때 유리
  데드락 가능성, 처리량 저하

Optimistic Lock (@Version):
  Lock 없이 읽기
  UPDATE WHERE version = 내가 읽은 버전
  affected rows = 0 → OptimisticLockException
  충돌 빈도 낮을 때 유리
  재시도 로직 필요

JPA @Version:
  INSERT: version = 0
  UPDATE: SET version = version+1 WHERE ... AND version = ?
  DELETE: WHERE ... AND version = ?
  affected = 0 → ObjectOptimisticLockingFailureException

선택 기준:
  충돌 빈도 낮음 + 재시도 가능 → Optimistic
  충돌 빈도 높음 → Pessimistic
  외부 시스템 포함 → Pessimistic 또는 Saga 패턴
```

---

## 🤔 생각해볼 문제

**Q1.** 좌석 예약 시스템에서 10,000명이 동시에 같은 좌석을 예약하려 할 때 Optimistic Lock과 Pessimistic Lock 중 어느 것이 더 적합한가? 이유는?

<details>
<summary>해설 보기</summary>

**Pessimistic Lock이 더 적합합니다.**

10,000명이 같은 Row(좌석)를 동시에 수정하려는 것은 극단적으로 높은 충돌 빈도입니다. Optimistic Lock에서는 대부분의 트랜잭션이 `OptimisticLockException`으로 실패하고 재시도하는 악순환이 발생합니다:

1. 10,000 트랜잭션이 동시에 읽음 (모두 version=0)
2. 하나만 성공 (version=1), 9,999가 실패
3. 9,999가 재시도 → 다시 대부분 실패
4. "Thundering herd" 현상 → 서버 부하 폭증

Pessimistic Lock (FOR UPDATE + SKIP LOCKED 패턴):
```sql
-- 예약 가능 좌석을 1개 잠금 (이미 잠긴 좌석 건너뜀)
SELECT * FROM seats WHERE concert_id = ? AND status = 'AVAILABLE'
LIMIT 1 FOR UPDATE SKIP LOCKED;
-- → 한 번에 1명씩 순서대로 처리, 나머지는 대기
```

실제로 티켓팅 시스템은 좌석 선점 + 결제 타임아웃 패턴을 사용합니다: FOR UPDATE로 잠금 → 5분 내 결제 완료 → 타임아웃 시 잠금 해제. 이것이 "선점 후 결제" UI의 기술적 근거입니다.

</details>

**Q2.** `@Version` 컬럼을 `Long` 대신 `LocalDateTime`으로 사용할 때의 장단점은?

<details>
<summary>해설 보기</summary>

**장점**:
- 별도 버전 컬럼 없이 이미 있는 `updated_at` 컬럼을 재사용 가능
- "마지막으로 수정된 시간"이라는 의미가 있어 감사(audit) 목적으로 활용 가능
- 애플리케이션 재배포 후 version이 0으로 리셋되는 문제 없음

**단점**:
- **동일 밀리초 내 두 번 수정 시 충돌 감지 실패**: `LocalDateTime`의 정밀도가 밀리초라면, 같은 밀리초에 두 트랜잭션이 수정 후 같은 timestamp를 가지면 충돌을 감지하지 못합니다.
- `Long version`은 원자적 증가로 이 문제가 없음
- DB 서버와 애플리케이션 서버의 시간 동기화 필요

**JPA에서 사용**:
```java
@Version
private LocalDateTime updatedAt;  // 가능하지만 정밀도 주의
```

일반적으로 `Long version`이 더 안전하고 권장됩니다. `LocalDateTime`은 `@LastModifiedDate` (Spring Data Auditing)와 별도로 사용하는 것이 명확합니다.

</details>

---

<div align="center">

**[⬅️ 데드락 로그](./05-deadlock-log-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Performance Tuning ➡️](../performance-tuning/01-slow-query-analysis.md)**

</div>
