# 데드락 로그 분석과 해결 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `LATEST DETECTED DEADLOCK` 섹션에서 "HOLDS THE LOCK(S)"와 "WAITING FOR THIS LOCK"을 어떻게 구분하는가?
- `hex` 값으로 잠긴 Row의 PK를 역산하는 방법은?
- `lock_mode X locks rec but not gap`과 `lock_mode X`의 차이는?
- 데드락 로그에서 어떤 인덱스가 사용됐는지 확인하는 방법은?
- Lock 순서 역전 패턴과 Gap Lock Deadlock 패턴을 로그에서 어떻게 구분하는가?
- 데드락 로그를 영구적으로 파일에 기록하는 방법은?

---

## 🔍 왜 이 개념이 중요한가

### 로그를 읽지 못하면 "왜 데드락이 나는지" 영원히 모른다

```
데드락 발생 후 일반적인 처리:
  ERROR 1213 확인 → "데드락이네, 재시도 추가"
  → 재시도 추가 → 여전히 발생 → "랜덤하게 나는 것 같아"
  → 결국 포기 또는 격리 수준 낮춤

로그를 읽는 개발자의 처리:
  SHOW ENGINE INNODB STATUS\G
  → LATEST DETECTED DEADLOCK 확인
  → "TRX A: accounts id=1 보유, accounts id=2 대기"
  → "TRX B: accounts id=2 보유, accounts id=1 대기"
  → 원인: id=1, id=2 를 다른 순서로 Lock
  → 해결: 항상 min(id) 먼저 Lock으로 수정
  → 데드락 0으로 감소

차이:
  로그 없이: "재시도로 덮기" → 근본 미해결
  로그 있이: "원인 파악 → 코드 수정" → 완전 해결
```

---

## 😱 잘못된 이해

### Before: "데드락 로그는 너무 복잡해서 읽기 어렵다"

```
실제 데드락 로그의 복잡해 보이는 이유:

1. hex 값으로 된 레코드 데이터:
   0: len 8; hex 0000000000000001; asc         ;;
   → "이게 무슨 의미야?"
   → 실제: 8 bytes BIGINT Big-endian = 1

2. 여러 섹션이 혼재:
   (1) HOLDS THE LOCK(S)
   (1) WAITING FOR THIS LOCK
   (2) HOLDS THE LOCK(S)
   (2) WAITING FOR THIS LOCK
   → 어느 것이 어느 트랜잭션의 것인지 헷갈림

3. 물리적 정보 (page no, space id):
   space id 234 page no 5 n bits 72
   → "물리적 위치는 왜 나오지?"
   → 실제로는 분석에 불필요한 정보 (무시해도 됨)

핵심만 읽는 방법:
  TRANSACTION (N): TRX 정보 + 현재 실행 쿼리
  HOLDS THE LOCK(S): 이 TRX가 보유 중인 Lock
  WAITING FOR THIS LOCK: 이 TRX가 기다리는 Lock
  WE ROLL BACK TRANSACTION (N): 롤백된 TRX 번호
  
  hex → int 변환: hex 값을 10진수로
  핵심: 어떤 인덱스의 어떤 값이 충돌했는가
```

---

## ✨ 올바른 이해

### After: 데드락 로그 = "누가 무엇을 가지고, 무엇을 기다리는가"만 파악하면 된다

```
데드락 로그 읽기 5단계:

Step 1: 트랜잭션 수 파악
  (1) TRANSACTION, (2) TRANSACTION → 2개 트랜잭션
  (n개 TRANSACTION → n개 순환 대기)

Step 2: 각 TRX의 현재 실행 쿼리 확인
  TRANSACTION 12345, ACTIVE 5 sec
  UPDATE accounts SET balance = ... WHERE id = 2
  → TRX 12345가 id=2를 UPDATE하려 함

Step 3: HOLDS 확인 (무엇을 가지고 있나)
  HOLDS THE LOCK(S):
  index PRIMARY of table accounts
  lock_mode X locks rec but not gap
  hex 0000000000000001 → id = 1
  → TRX 12345가 accounts.id=1에 X Lock 보유

Step 4: WAITING 확인 (무엇을 기다리나)
  WAITING FOR THIS LOCK:
  index PRIMARY of table accounts
  lock_mode X locks rec but not gap waiting
  hex 0000000000000002 → id = 2
  → TRX 12345가 accounts.id=2를 기다림

Step 5: 순환 관계 파악
  TRX 12345: id=1 보유 → id=2 대기
  TRX 12346: id=2 보유 → id=1 대기
  → 역방향 Lock 순서 → 해결: 순서 통일
```

---

## 🔬 내부 동작 원리

### 1. 데드락 로그 전체 구조 분석

```
SHOW ENGINE INNODB STATUS\G
에서 찾을 위치:

[출력에서 아래 라인 사이의 내용]
---
LATEST DETECTED DEADLOCK
---
[날짜 시간]
[데드락 정보]
[섹션 구분자]

전체 구조:

------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-03-15 14:23:45 0x7f1234567890    ← 발생 시각
*** (1) TRANSACTION:                  ← 첫 번째 트랜잭션 섹션 시작
TRANSACTION 54321, ACTIVE 3 sec starting index read
                    ^^^^^^^^^^        ← 트랜잭션 ID
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
           ^                               ^^^^^^^^^^^^^^^  ← 보유 Lock 수
MySQL thread id 101, OS thread handle 123456789, query id 456 localhost app
UPDATE accounts SET balance = balance + 100 WHERE id = 2   ← 현재 실행 중 쿼리

*** (1) HOLDS THE LOCK(S):            ← 이 TRX가 보유한 Lock들
RECORD LOCKS space id 234 page no 5 n bits 72 index PRIMARY of table `db`.`accounts`
              ^^^^^^^^^^^^^^^^^^^^ ← 물리 위치 (무시해도 됨)
                                                  ^^^^^^^         ^^^^^^^^
                                                  인덱스명        테이블명
trx id 54321 lock_mode X locks rec but not gap
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ← Lock 종류!
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 0000000000000001; asc         ;;   ← 0번 필드: id=1 (PK)
 1: len 8; hex 00000000000027d8; asc    '  ;;     ← 1번 필드: user_id 값
 2: ...                                            ← 2번 필드: 다른 컬럼들

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:  ← 이 TRX가 기다리는 Lock
RECORD LOCKS space id 234 page no 5 n bits 72 index PRIMARY of table `db`.`accounts`
trx id 54321 lock_mode X locks rec but not gap waiting
                                               ^^^^^^^  ← waiting 추가됨
Record lock, heap no 5 PHYSICAL RECORD: ...
 0: len 8; hex 0000000000000002; asc         ;;  ← id=2를 기다림

*** (2) TRANSACTION:                  ← 두 번째 트랜잭션 섹션
... (동일한 구조)

*** WE ROLL BACK TRANSACTION (1)      ← TRX 1이 Victim으로 선택됨
```

### 2. hex 값 역산 방법

```
로그에서 자주 나오는 데이터 타입 변환:

BIGINT (8 bytes, Big-endian):
  hex 0000000000000001 = 1
  hex 0000000000000064 = 100
  hex 00000000000003E8 = 1000
  
  변환 방법:
    0000000000000001 → 16진수 1 → 10진수 1
    000000000000007B → 16진수 7B → 10진수 123
  
  MySQL에서 변환:
  SELECT CONV('000000000000007B', 16, 10);  -- 결과: 123

INT (4 bytes, Big-endian):
  hex 00000001 = 1
  hex 000003E8 = 1000

VARCHAR (가변 길이):
  len N; hex [ASCII 코드들]; asc [ASCII 문자들]
  hex 50414944 = P,A,I,D = 'PAID'
  asc 필드가 있으면 바로 읽을 수 있음

DATETIME (5 bytes in MySQL 5.6+):
  복잡한 인코딩 → 직접 해석보다 로그의 다른 정보 활용

실용적 팁:
  PK가 AUTO_INCREMENT BIGINT인 경우:
    마지막 8 hex 글자만 보면 됨
    0000000000000001 → 마지막 1 → id = 1
    000000000000270F → 마지막 270F = 9999 → id = 9999
```

### 3. lock_mode 값 완전 해석

```
HOLDS/WAITING에서 나오는 lock_mode 값:

X locks rec but not gap:
  Exclusive Record Lock만 (Gap 없음)
  발생: PK/Unique Index 등치 조건
  의미: 해당 Row만 잠금, 인접 Gap은 열려있음
  
X (= Next-Key Lock):
  Exclusive Record Lock + 이전 Gap Lock
  발생: Non-unique Index, 범위 조건, REPEATABLE READ
  의미: 해당 Row + 이전 Gap까지 잠금

X,GAP:
  Gap Lock만 (Record 없음)
  발생: 존재하지 않는 값에 FOR UPDATE
  의미: 그 위치에 새 삽입 차단

S locks rec but not gap:
  Shared Record Lock만 (FOR SHARE + PK 등치)
  
S:
  Shared Next-Key Lock (FOR SHARE + 범위)

X,INSERT_INTENTION:
  INSERT 직전 획득하는 Lock
  Gap Lock과 충돌 → 삽입 대기 중
  로그에서 보이면: Gap Lock Deadlock 패턴!

로그에서 패턴 구분:

패턴 1 — Lock 순서 역전:
  TRX1: HOLDS X on id=1, WAITING X on id=2
  TRX2: HOLDS X on id=2, WAITING X on id=1
  → 역방향 순서 → Lock 순서 통일로 해결

패턴 2 — Gap Lock Deadlock:
  TRX1: HOLDS X,GAP on (100, 200), WAITING X,INSERT_INTENTION at 150
  TRX2: HOLDS X,GAP on (100, 200), WAITING X,INSERT_INTENTION at 170
  → INSERT Intention vs Gap Lock → READ_COMMITTED or UNIQUE KEY

패턴 3 — S Lock 업그레이드:
  TRX1: HOLDS S on id=1, WAITING X on id=1
  TRX2: HOLDS S on id=1, WAITING X on id=1
  → 같은 Row에 두 S Lock 보유 후 X Lock 요청
  → 처음부터 FOR UPDATE 사용
```

### 4. 데드락 로그 파일에 영구 기록

```sql
-- SHOW ENGINE INNODB STATUS는 가장 최근 1건만 보존 (이전 것 덮임)
-- 모든 데드락을 영구 기록하는 방법:

-- 방법 1: innodb_print_all_deadlocks (MySQL 8.0)
SET GLOBAL innodb_print_all_deadlocks = ON;
-- 모든 데드락을 MySQL 에러 로그 파일에 기록
-- /var/log/mysql/error.log 또는 설정된 error_log 경로

-- my.cnf에서 영구 설정:
-- [mysqld]
-- innodb_print_all_deadlocks = ON

-- 에러 로그에서 데드락 찾기:
-- grep -A 50 "LATEST DETECTED DEADLOCK" /var/log/mysql/error.log

-- 방법 2: 애플리케이션에서 에러 캐치 후 로깅
-- (Java Spring 예시)
try {
    transferService.transfer(from, to, amount);
} catch (DeadlockLoserDataAccessException e) {
    log.error("Deadlock detected - from:{}, to:{}, amount:{}", from, to, amount, e);
    // SHOW ENGINE INNODB STATUS 결과를 자동으로 가져와 기록하는 것도 가능
    // (DBA 도구 또는 커스텀 로직)
}

-- 방법 3: pt-deadlock-logger (Percona Toolkit)
-- 지속적으로 데드락을 모니터링하고 DB 테이블에 기록
-- pt-deadlock-logger --host localhost --user root --password ... \
--   --dest deadlock_log_db.deadlock_log
-- → deadlock_log 테이블에 시간별 데드락 정보 저장
```

### 5. 해결 전략별 코드 패턴

```java
// 해결 전략 1: Lock 순서 통일

// 잘못된 코드 (순서 불일치):
public void transfer(Long from, Long to, BigDecimal amount) {
    Account fromAccount = accountRepo.findByIdWithLock(from);  // FROM 먼저
    Account toAccount   = accountRepo.findByIdWithLock(to);    // TO 다음
    // from=2, to=1이면: Lock(2)→Lock(1)
    // from=1, to=2이면: Lock(1)→Lock(2)
    // → 역방향 발생 가능 → 데드락!
}

// 올바른 코드 (오름차순 정렬):
@Transactional
public void transfer(Long from, Long to, BigDecimal amount) {
    Long first  = Math.min(from, to);
    Long second = Math.max(from, to);
    Account a1 = accountRepo.findByIdWithLock(first);   // 항상 작은 id 먼저
    Account a2 = accountRepo.findByIdWithLock(second);  // 항상 큰 id 다음
    
    Account fromAccount = from.equals(first) ? a1 : a2;
    Account toAccount   = to.equals(second) ? a2 : a1;
    
    fromAccount.withdraw(amount);
    toAccount.deposit(amount);
}
// 항상 오름차순이므로 순환 불가 → 데드락 불가능

// 해결 전략 2: Gap Lock Deadlock 방지 (UNIQUE KEY 활용)

// 잘못된 패턴 (Gap Lock Deadlock 위험):
@Transactional
public boolean claimCoupon(Long userId, Long couponId) {
    // SELECT FOR UPDATE → Gap Lock 획득
    Optional<CouponClaim> existing = couponClaimRepo
        .findByUserAndCoupon(userId, couponId, LockModeType.PESSIMISTIC_WRITE);
    if (existing.isPresent()) return false;
    
    couponClaimRepo.save(new CouponClaim(userId, couponId));
    // INSERT → Insert Intention → Gap Lock 충돌 → Deadlock!
    return true;
}

// 올바른 패턴 (UNIQUE KEY + INSERT IGNORE):
@Transactional
public boolean claimCoupon(Long userId, Long couponId) {
    try {
        couponClaimRepo.insertIfNotExists(userId, couponId);
        // INSERT IGNORE INTO coupon_claims (user_id, coupon_id) VALUES (?, ?)
        // UNIQUE KEY uk_user_coupon (user_id, coupon_id) 필요
        return true;
    } catch (DuplicateKeyException e) {
        return false;  // 이미 등록됨
    }
}
// Gap Lock 없음 → Deadlock 불가능

// 해결 전략 3: @Retryable (보험)
@Retryable(
    value = DeadlockLoserDataAccessException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)  // 100ms, 200ms 지수 백오프
)
public void retryableTransfer(Long from, Long to, BigDecimal amount) {
    doTransfer(from, to, amount);
}

@Transactional
protected void doTransfer(Long from, Long to, BigDecimal amount) {
    // ... (Lock 순서 통일된 코드)
}
```

---

## 💻 실전 실험

### 실험 1: 데드락 발생 후 로그 분석 워크플로우

```sql
-- 1. 의도적 데드락 발생
CREATE TABLE dl_analysis (id INT PRIMARY KEY, val INT);
INSERT INTO dl_analysis VALUES (1, 100), (2, 200), (3, 300);

-- 세션 1
START TRANSACTION;
UPDATE dl_analysis SET val = val + 10 WHERE id = 1;

-- 세션 2
START TRANSACTION;
UPDATE dl_analysis SET val = val + 20 WHERE id = 2;

-- 세션 1
UPDATE dl_analysis SET val = val + 10 WHERE id = 2;  -- 대기

-- 세션 2
UPDATE dl_analysis SET val = val + 20 WHERE id = 1;  -- Deadlock!

-- 2. 로그 즉시 확인
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 섹션 복사

-- 3. 로그 분석 체크리스트:
-- □ 발생 시각 확인
-- □ 트랜잭션 수 (보통 2개)
-- □ 각 TRX의 실행 쿼리 확인
-- □ HOLDS 인덱스와 hex 값 (어떤 Row Lock 보유)
-- □ WAITING 인덱스와 hex 값 (어떤 Row Lock 대기)
-- □ WE ROLL BACK (어떤 TRX가 Victim)
-- □ 패턴 분류: 순서역전/S→X업그레이드/Gap Lock
-- □ 해결책 결정: 순서통일/FOR UPDATE/READ_COMMITTED

DROP TABLE dl_analysis;
```

### 실험 2: lock_mode별 발생 조건 확인

```sql
CREATE TABLE lock_mode_test (
    id INT PRIMARY KEY,
    user_id INT,
    val INT,
    INDEX idx_user (user_id)
) ENGINE=InnoDB;
INSERT INTO lock_mode_test VALUES (1,1,10),(2,2,20),(3,1,30),(5,3,50);

-- 케이스 A: PK 등치 → X,REC_NOT_GAP
START TRANSACTION;
SELECT * FROM lock_mode_test WHERE id = 2 FOR UPDATE;
SELECT INDEX_NAME, LOCK_MODE, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'lock_mode_test' AND LOCK_TYPE = 'RECORD';
-- X,REC_NOT_GAP on PRIMARY id=2
ROLLBACK;

-- 케이스 B: Non-unique Index → X (Next-Key)
START TRANSACTION;
SELECT * FROM lock_mode_test WHERE user_id = 1 FOR UPDATE;
SELECT INDEX_NAME, LOCK_MODE, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'lock_mode_test' AND LOCK_TYPE = 'RECORD';
-- X,REC_NOT_GAP on PRIMARY id=1, id=3
-- X on idx_user: user_id=1/id=1, user_id=1/id=3 (Next-Key Locks)
-- X,GAP on idx_user: 범위 끝 Gap
ROLLBACK;

-- 케이스 C: 없는 값 → X,GAP
START TRANSACTION;
SELECT * FROM lock_mode_test WHERE id = 4 FOR UPDATE;
SELECT INDEX_NAME, LOCK_MODE, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'lock_mode_test' AND LOCK_TYPE = 'RECORD';
-- X,GAP on PRIMARY: id=5 이전 Gap = (3, 5)
ROLLBACK;

DROP TABLE lock_mode_test;
```

### 실험 3: 데드락 로그를 에러 로그에 기록

```sql
-- innodb_print_all_deadlocks 활성화
SET GLOBAL innodb_print_all_deadlocks = ON;
SHOW VARIABLES LIKE 'innodb_print_all_deadlocks';

-- 에러 로그 경로 확인
SHOW VARIABLES LIKE 'log_error';
-- /var/log/mysql/error.log 또는 유사

-- 데드락 발생 후 에러 로그에서 확인:
-- $ tail -100 /var/log/mysql/error.log | grep -A 50 "DEADLOCK"
-- 또는:
-- $ grep -c "DEADLOCK" /var/log/mysql/error.log  # 총 데드락 발생 횟수

SET GLOBAL innodb_print_all_deadlocks = OFF;  -- 실험 후 원복
```

---

## 📊 성능 비교

```
데드락 패턴별 빈도와 영향:

Lock 순서 역전 (가장 흔함):
  빈도: 높음 (코드에서 순서를 의식하지 않으면 자주 발생)
  영향: 관련 Row만 차단 (범위 제한적)
  해결 난이도: 쉬움 (코드 수정 1줄)
  해결: 항상 min(id) 먼저 Lock

Gap Lock Deadlock (INSERT 집약 시스템):
  빈도: 중간 (동시 INSERT가 많은 경우)
  영향: INSERT 처리량 저하, Deadlock 빈발
  해결 난이도: 중간 (설계 또는 격리 수준 변경)
  해결: READ_COMMITTED 또는 UNIQUE KEY

S Lock 업그레이드:
  빈도: 낮음 (FOR SHARE를 의도적으로 쓰는 경우)
  영향: 동시 업데이트가 많은 Row에서 반복 발생
  해결: 처음부터 FOR UPDATE 사용

전형적인 데드락 감지 시간:
  ON 상태: ~1ms (DFS 탐색)
  Lock Timeout (OFF): 설정값 (5~50초)
  → ON 상태가 빠른 복구로 사용자 경험 훨씬 양호
```

---

## ⚖️ 트레이드오프

```
데드락 해결 전략 선택:

전략 1 — Lock 순서 통일:
  ✅ 데드락 원천 차단 (Zero Deadlock 가능)
  ✅ 성능 영향 없음
  ✅ 코드 변경 최소
  ❌ 모든 코드 경로에서 순서 유지 필요 (실수 가능)

전략 2 — READ_COMMITTED:
  ✅ Gap Lock Deadlock 완전 제거
  ✅ Semi-consistent Read로 인덱스 없는 DML 피해 감소
  ❌ Phantom Read 허용
  ❌ Row-based Replication 필수 (Statement-based 불가)
  ❌ 전역 변경 → 테스트 부담

전략 3 — Optimistic Lock (@Version):
  ✅ Lock 없음 → Deadlock 원천 불가
  ✅ 높은 처리량 (충돌 적을 때)
  ❌ 충돌 빈번하면 재시도 폭증 (Thundering Herd)
  ❌ 외부 시스템 호출 포함 시 재시도 복잡

전략 4 — @Retryable:
  ✅ 구현 간단
  ✅ 다른 전략과 함께 사용 (보험)
  ❌ 근본 미해결 → 재시도 중 또 Deadlock 가능
  ❌ 지수 백오프 없으면 재시도 폭주

현장 권장 조합:
  1단계: 로그 분석으로 원인 파악
  2단계: 원인에 맞는 근본 해결 (순서 통일, UNIQUE KEY 등)
  3단계: @Retryable 추가 (예상 못한 경우 대비)
```

---

## 📌 핵심 정리

```
데드락 로그 분석 핵심:

읽기 위치:
  SHOW ENGINE INNODB STATUS\G
  → "LATEST DETECTED DEADLOCK" 섹션
  
  영구 기록: innodb_print_all_deadlocks = ON
             → MySQL error.log에 모든 데드락 기록

로그 구조:
  (N) TRANSACTION: 트랜잭션 정보 + 현재 쿼리
  HOLDS THE LOCK(S): 이 TRX가 보유 중인 Lock
  WAITING FOR THIS LOCK: 이 TRX가 기다리는 Lock
  WE ROLL BACK TRANSACTION (N): Victim 번호

hex 역산:
  8 bytes BIGINT: hex 0000000000000001 = 1
  SELECT CONV('0000000000000001', 16, 10) → 1

lock_mode 해석:
  X locks rec but not gap: PK/Unique 등치 → Record Lock만
  X (Next-Key Lock): 범위/Non-unique → Record + Gap
  X,GAP: 없는 값 위치 → Gap Lock만
  X,INSERT_INTENTION: INSERT 직전 → Gap Lock과 충돌

패턴별 해결:
  순서 역전: Lock 획득 순서 통일 (오름차순)
  S→X 업그레이드: 처음부터 FOR UPDATE
  Gap Lock Deadlock: READ_COMMITTED 또는 UNIQUE KEY
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 데드락 로그 요약을 보고 원인과 해결책을 설명하라.

```
*** (1) TRANSACTION: 99001, ACTIVE 2 sec
  UPDATE orders SET status='PROCESSING' WHERE id=500

*** (1) HOLDS THE LOCK(S):
  index idx_user of table `shop`.`orders`
  lock_mode X
  0: user_id=1001, id=500

*** (1) WAITING FOR THIS LOCK:
  index idx_user of table `shop`.`orders`  
  lock_mode X waiting
  0: user_id=1001, id=300

*** (2) TRANSACTION: 99002, ACTIVE 1 sec
  UPDATE orders SET status='CANCELLED' WHERE id=300

*** (2) HOLDS THE LOCK(S):
  index idx_user of table `shop`.`orders`
  lock_mode X
  0: user_id=1001, id=300

*** (2) WAITING FOR THIS LOCK:
  index idx_user of table `shop`.`orders`
  lock_mode X waiting
  0: user_id=1001, id=500
```

<details>
<summary>해설 보기</summary>

**원인 분석**:

- TRX 99001: idx_user에서 user_id=1001, id=500 Lock 보유 → id=300 대기
- TRX 99002: idx_user에서 user_id=1001, id=300 Lock 보유 → id=500 대기
- 순환 대기 → Deadlock

**Lock이 Secondary Index(idx_user)에 걸린 이유**:
- `WHERE id=500`은 PK 등치이므로 PRIMARY에 X,REC_NOT_GAP이어야 함
- 하지만 idx_user(user_id=1001)에도 Lock이 걸림
- 이유: `UPDATE orders SET status=... WHERE id=500`이 실행될 때
  - 어딘가 idx_user를 통한 접근이 있거나
  - Secondary Index Leaf에도 Lock을 거는 경우
  - 또는 다른 쿼리가 idx_user를 통해 접근 중

더 정확한 원인 파악에는 원래 로그의 전체 쿼리가 필요하지만, 핵심은:

**두 트랜잭션이 같은 user_id=1001의 orders를 서로 역방향으로 접근했다.**

**해결책**:
1. `WHERE id=PK` 조건을 사용하면 Primary Index에만 Lock이 걸려야 함 → 실제 쿼리 확인
2. 같은 user_id=1001의 주문들을 동시에 여러 트랜잭션이 처리하는 패턴이라면: 동일 사용자의 주문은 순서를 정해 처리 (id 오름차순)
3. 또는 user_id 단위로 분산 Lock 사용

</details>

---

**Q2.** 데드락 로그에서 `lock_mode X`(Next-Key Lock)가 보이면 반드시 Gap Lock Deadlock인가? 이를 Lock 순서 역전 데드락과 어떻게 구분하는가?

<details>
<summary>해설 보기</summary>

**반드시 Gap Lock Deadlock은 아닙니다.**

`lock_mode X` (Next-Key Lock)는 Record Lock + Gap Lock을 포함하지만, 이것이 Deadlock의 원인이 Gap Lock인지 Record Lock인지를 확인해야 합니다.

**Lock 순서 역전 + Next-Key Lock**:
```
TRX A: HOLDS X on idx_amount val=1000 (이 인덱스 레코드의 Next-Key Lock)
       WAITING X on idx_amount val=2000
TRX B: HOLDS X on idx_amount val=2000
       WAITING X on idx_amount val=1000
→ 서로 다른 Record를 역방향으로 요청 = Lock 순서 역전
→ 해결: 항상 작은 val 먼저 Lock
```

**Gap Lock Deadlock 구별 방법**:
- WAITING에 `X,INSERT_INTENTION`이 보이면 Gap Lock Deadlock
- 같은 `LOCK_DATA`(Gap의 경계값)에 두 트랜잭션이 Gap Lock + Insert Intention을 기다리면 Gap Lock Deadlock
- 로그에서 INSERT 쿼리가 보이면 Gap Lock Deadlock 의심

**판별 체크리스트**:
1. WAITING에 `INSERT_INTENTION` 있음 → Gap Lock Deadlock
2. 두 TRX가 서로 다른 Record를 역방향으로 보유/대기 → Lock 순서 역전
3. 같은 Row에 S Lock 보유 + X Lock 대기 × 2 → S Lock 업그레이드

</details>

---

**Q3.** 데드락 로그에서 `WE ROLL BACK TRANSACTION (1)`이 표시됐는데, 이후 TRX 1을 실행한 Java 코드에서 아무런 에러 처리를 하지 않으면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

TRX 1(Victim)은 InnoDB에 의해 ROLLBACK되고 `ERROR 1213`을 반환합니다. Java JDBC 드라이버가 이를 `SQLException`으로 받아 Spring이 `DeadlockLoserDataAccessException`으로 변환합니다.

**에러 처리 없는 경우 발생하는 문제**:

1. **스택 트레이스 발생**: 처리되지 않은 예외가 Controller까지 전파 → HTTP 500 응답.

2. **부분 실패 상태**: TRX 1의 모든 변경이 ROLLBACK됨. 하지만 TRX 1이 이전에 실행한 외부 작업(이메일 발송, 알림 전송 등)은 ROLLBACK 되지 않음 → 불일치.

3. **사용자 경험 저하**: 사용자는 "오류가 발생했습니다"를 보게 되고, 작업이 실패했는지 성공했는지 모름.

**올바른 처리**:

```java
// 방법 1: @Retryable (Spring Retry)
@Retryable(
    value = DeadlockLoserDataAccessException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
@Transactional
public void processOrder(Long orderId) {
    // ...
}

// 방법 2: 명시적 catch + 재시도
@Transactional
public void processWithRetry(Long orderId) {
    for (int attempt = 0; attempt < 3; attempt++) {
        try {
            doProcess(orderId);
            return;  // 성공 시 종료
        } catch (DeadlockLoserDataAccessException e) {
            if (attempt == 2) throw e;  // 3번째 실패 시 재throw
            Thread.sleep(100 * (attempt + 1));  // 대기 후 재시도
        }
    }
}
```

재시도 전 반드시: Lock 순서를 통일하여 데드락 발생 원인을 제거해야 합니다. 재시도만으로는 같은 패턴의 데드락이 반복될 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 데드락 감지](./04-deadlock-detection.md)** | **[홈으로 🏠](../README.md)** | **[다음: Optimistic vs Pessimistic ➡️](./06-optimistic-vs-pessimistic-lock.md)**

</div>
