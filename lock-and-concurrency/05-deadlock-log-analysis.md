# 데드락 로그 분석과 해결 전략

---

## 🎯 핵심 질문

- `SHOW ENGINE INNODB STATUS`의 `LATEST DETECTED DEADLOCK` 섹션을 어떻게 읽는가?
- 로그에서 "holds the lock", "waiting for this lock"의 차이는?
- 데드락 로그에서 어떤 인덱스가 사용됐는지 확인하는 방법은?
- 락 순서 통일이 데드락을 방지하는 원리는?
- 데드락을 원천적으로 방지하는 설계 패턴은?

---

## 🔬 내부 동작 원리

### 1. 데드락 로그 전체 구조

```
SHOW ENGINE INNODB STATUS\G
-- "LATEST DETECTED DEADLOCK" 섹션:

------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-03-15 14:23:45 0x7f12345
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 5 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 101, OS thread handle 123, query id 456 localhost app
UPDATE accounts SET balance = balance + 100 WHERE id = 2

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 234 page no 5 n bits 72 index PRIMARY of table `mydb`.`accounts`
trx id 12345 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; ...
 0: len 8; hex 0000000000000001; asc         ;; -- id = 1

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 234 page no 5 n bits 72 index PRIMARY of table `mydb`.`accounts`
trx id 12345 lock_mode X locks rec but not gap waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 4; ...
 0: len 8; hex 0000000000000002; asc         ;; -- id = 2

*** (2) TRANSACTION:
TRANSACTION 12346, ACTIVE 4 sec starting index read
...
UPDATE accounts SET balance = balance - 100 WHERE id = 1

*** (2) HOLDS THE LOCK(S):
-- id = 2에 대한 X Lock 보유

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
-- id = 1에 대한 X Lock 대기

*** WE ROLL BACK TRANSACTION (2)
-- 12346이 더 작은 변경 → Victim 선택
```

### 2. 로그 항목별 해석

```
각 트랜잭션 섹션:

"TRANSACTION 12345, ACTIVE 5 sec":
  트랜잭션 ID = 12345, 5초간 실행 중

"LOCK WAIT 3 lock struct(s), 2 row lock(s)":
  3개의 Lock 구조체, 2개의 Row Lock 보유

현재 실행 쿼리:
  "UPDATE accounts SET balance = balance + 100 WHERE id = 2"
  → 이 쿼리가 Lock 대기 중

"HOLDS THE LOCK(S)":
  index PRIMARY: Primary Key 인덱스에 Lock
  lock_mode X locks rec but not gap: Record X Lock (Gap 제외)
  hex 0000000000000001: id = 1 (Big-endian 16진수)

"WAITING FOR THIS LOCK TO BE GRANTED":
  lock_mode X locks rec but not gap waiting: 대기 중
  hex 0000000000000002: id = 2 Lock 대기 중

Lock 데이터 hex 해석:
  BIGINT: 8 bytes Big-endian
  0000000000000001 → 1
  0000000000000064 → 100
  INT: 4 bytes
  00000001 → 1
```

### 3. 데드락 원인 분석 패턴

```
로그에서 원인 파악:

패턴 1: 락 순서 역전
  TRX 1: id=1 Lock → id=2 Lock 대기
  TRX 2: id=2 Lock → id=1 Lock 대기
  → 역방향 순서 = 데드락
  
  해결: 항상 id 오름차순으로 Lock 획득
  Java:
    Long[] ids = {id1, id2};
    Arrays.sort(ids);
    for (Long id : ids) {
        repository.findByIdWithLock(id);  // 오름차순
    }

패턴 2: S Lock 업그레이드 충돌
  TRX 1: FOR SHARE(id=1) → FOR UPDATE(id=1) 대기
  TRX 2: FOR SHARE(id=1) → FOR UPDATE(id=1) 대기
  → 둘 다 상대의 S Lock 때문에 X Lock 불가
  
  해결: 처음부터 FOR UPDATE 사용 (수정 의도가 있으면)

패턴 3: Gap Lock 충돌
  TRX 1: Gap(5,8) Lock → id=6 Insert 시도
  TRX 2: Gap(5,8) Lock → id=7 Insert 시도
  → 둘 다 상대의 Gap Lock 때문에 차단
  
  해결: READ COMMITTED 사용 또는 UNIQUE KEY로 범위 축소

패턴 4: Secondary Index → Primary Index 순서
  UPDATE WHERE non_unique_col = X
  → Secondary Index Lock → Primary Index Lock (Double Lookup)
  TRX A, TRX B가 다른 Secondary Index 레코드에서 같은 Primary Key로 진행
  
  해결: 인덱스 설계 재검토, 트랜잭션 순서 통일
```

### 4. 데드락 방지 전략

```java
// 전략 1: 락 획득 순서 통일 (가장 효과적)
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // 항상 작은 ID를 먼저 Lock (순서 고정)
    Long first = Math.min(fromId, toId);
    Long second = Math.max(fromId, toId);
    
    Account a1 = accountRepo.findByIdWithLock(first);   // FOR UPDATE
    Account a2 = accountRepo.findByIdWithLock(second);  // FOR UPDATE
    
    // 이제 fromId, toId 어느 조합으로 호출해도 같은 순서로 Lock
    Account from = fromId.equals(first) ? a1 : a2;
    Account to = toId.equals(second) ? a2 : a1;
    
    from.withdraw(amount);
    to.deposit(amount);
}

// 전략 2: 단일 쿼리로 처리 (Lock 범위 최소화)
// 여러 UPDATE 대신 하나의 UPDATE로:
@Modifying
@Query("UPDATE Account a SET a.balance = a.balance + :delta WHERE a.id = :id")
void adjustBalance(@Param("id") Long id, @Param("delta") BigDecimal delta);
// Lock 시간 최소화 → 충돌 확률 감소

// 전략 3: Optimistic Lock (데드락 근원 차단)
@Version
private Long version;

// → SELECT ... (Lock 없음) → UPDATE WHERE version = ?
// → 충돌 시 OptimisticLockException → 재시도
// → 충돌 빈도 낮으면 Pessimistic보다 유리

// 전략 4: 트랜잭션 크기 최소화
// BAD: 하나의 트랜잭션에서 여러 테이블을 오래 Lock
@Transactional
public void longTransaction() {
    updateA();  // Lock A
    doExpensiveCalculation();  // 오랜 시간 → A 계속 Lock
    updateB();  // Lock B
    // → A, B 모두 오랫동안 Lock → 충돌 확률↑
}

// GOOD: Lock 시간 최소화
@Transactional
public void shortTransaction() {
    BigDecimal result = doExpensiveCalculation();  // Lock 밖에서
    updateA(result);  // Lock 최소 시간
    updateB(result);  // Lock 최소 시간
}
```

---

## 💻 실전 실험

### 실험: 데드락 재현 → 로그 분석 → 수정

```sql
-- 1단계: 데드락 유발 코드 (의도적)
-- 세션 1
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- 세션 2
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;

-- 세션 1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 대기

-- 세션 2
UPDATE accounts SET balance = balance + 100 WHERE id = 1;  -- 데드락!

-- 2단계: 로그 확인
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 섹션을 복사하여 분석

-- 3단계: 수정된 코드 (항상 낮은 ID 먼저)
-- 세션 1
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 먼저
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 다음
COMMIT;

-- 세션 2
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 동일 순서!
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- → 세션 2가 id=1에서 대기 (정상 직렬화, 데드락 아님)
COMMIT;
```

---

## 📌 핵심 정리

```
데드락 로그 분석 핵심:

로그 구조:
  (1) TRANSACTION: 트랜잭션 정보, 현재 실행 쿼리
  HOLDS THE LOCK(S): 보유 중인 Lock (인덱스, 레코드)
  WAITING FOR THIS LOCK: 대기 중인 Lock
  WE ROLL BACK TRANSACTION (N): Victim 선택

hex 해석:
  BIGINT id = 8 bytes Big-endian
  0000000000000001 = 1

흔한 패턴:
  Lock 획득 순서 역전 → 순서 통일로 해결
  S Lock 업그레이드 → 처음부터 FOR UPDATE
  Gap Lock 충돌 → READ COMMITTED 또는 설계 개선

예방 전략:
  Lock 순서 통일 (ID 오름차순)
  트랜잭션 크기 최소화
  Optimistic Lock (충돌 빈도 낮은 경우)
  @Retryable로 데드락 에러 재시도 처리
```

---

## 🤔 생각해볼 문제

**Q1.** 데드락 로그에서 `lock_mode X locks rec but not gap`과 `lock_mode X`의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

- **`X locks rec but not gap`**: Record Lock만, Gap Lock 없음. 특정 Row에만 X Lock. REPEATABLE READ에서도 PK나 Unique Index 등치 조건에서 발생. Gap 범위는 잠기지 않아 인접 범위에 삽입 가능.

- **`X` (Next-Key Lock)**: Record Lock + 이전 Gap Lock의 조합. REPEATABLE READ에서 범위 조건, Non-unique Index, 존재하지 않는 레코드 조건 등에서 발생. Gap도 잠겨 해당 범위에 삽입 차단.

- **`X,GAP`**: Gap Lock만, Record Lock 없음. 존재하지 않는 레코드의 Gap에만 Lock.

로그 분석 시 이 차이로 어떤 쿼리 패턴이 데드락을 유발했는지 유추할 수 있습니다. `rec but not gap`이 보이면 PK 등치 조건, `X`가 보이면 범위 조건이나 비유니크 인덱스 관련입니다.

</details>

---

<div align="center">

**[⬅️ 데드락 감지](./04-deadlock-detection.md)** | **[홈으로 🏠](../README.md)** | **[다음: Optimistic vs Pessimistic ➡️](./06-optimistic-vs-pessimistic-lock.md)**

</div>
