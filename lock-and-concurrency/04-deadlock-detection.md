# 데드락 발생 원리와 감지 알고리즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 데드락이 발생하기 위한 4가지 Coffman 조건은 무엇이며, InnoDB에서 각각 어떻게 해당되는가?
- Wait-for Graph에서 사이클을 찾는 DFS 알고리즘의 동작 방식은?
- ROLLBACK될 Victim 트랜잭션은 `trx_weight`가 낮은 쪽인데, 이 값은 어떻게 계산되는가?
- `innodb_deadlock_detect = OFF`가 성능을 향상시키는 이유와 그 위험성은?
- 데드락 vs Lock Timeout은 어떻게 다르고 각각 어떤 에러를 발생시키는가?
- 데드락 에러를 Spring에서 처리하고 재시도하는 패턴은?

---

## 🔍 왜 이 개념이 중요한가

### 데드락은 예방이 99%, 감지는 마지막 안전망

```
왜 데드락이 반복적으로 발생하는가:
  단순한 이체 서비스에서도:
    
    서비스 A: transferMoney(from=1, to=2, amount=100)
      → UPDATE accounts WHERE id=1 (Lock id=1)
      → UPDATE accounts WHERE id=2 (Lock id=2)
    
    서비스 B: transferMoney(from=2, to=1, amount=50)
      → UPDATE accounts WHERE id=2 (Lock id=2)
      → UPDATE accounts WHERE id=1 (Lock id=1 대기)
    
    서비스 A → id=2 대기 (서비스 B가 보유)
    서비스 B → id=1 대기 (서비스 A가 보유)
    → 데드락
  
  핵심: 두 트랜잭션이 서로 다른 순서로 같은 자원을 Lock

데드락을 이해해야 하는 이유:
  단순히 "데드락이 났다 → 재시도"로 처리하면:
    원인 미해결 → 계속 발생 → 서비스 불안정
  
  원인을 이해하면:
    Lock 획득 순서를 통일 → 데드락 원천 차단
    쿼리/인덱스 재설계 → Gap Lock Deadlock 방지
    트랜잭션 크기 최소화 → Lock 보유 시간 단축
```

---

## 😱 잘못된 이해

### Before: "데드락은 무작위로 발생하는 운 나쁜 상황"

```
잘못된 믿음:
  "동시성이 있으면 어쩔 수 없이 데드락이 난다"
  "데드락 나면 그냥 재시도하면 된다"
  "데드락을 완전히 없애는 것은 불가능하다"

실제:
  데드락은 무작위가 아닌 결정론적(deterministic) 원인 존재:
    원인 1: Lock 획득 순서가 트랜잭션마다 다름
    원인 2: S Lock → X Lock 업그레이드
    원인 3: Gap Lock 동시 보유 후 INSERT 시도
    
  각 원인에 대한 해결책이 존재:
    순서 불일치 → 항상 오름차순으로 Lock 획득
    S→X 업그레이드 → 처음부터 FOR UPDATE
    Gap Lock → READ_COMMITTED 또는 UNIQUE KEY

잘못된 재시도 로직:
  @Retryable(maxAttempts = 3)
  @Transactional
  public void transfer(Long from, Long to, BigDecimal amount) {
    updateBalance(from, -amount);  // Lock from
    updateBalance(to, +amount);    // Lock to (순서 고정 안 됨)
  }
  // → 재시도해도 같은 순서로 실행 → 계속 데드락
  // 재시도 횟수 초과 → 최종 실패 → 더 큰 문제
```

---

## ✨ 올바른 이해

### After: 데드락 = 순환 의존 = 제거 가능한 패턴

```
데드락의 본질:

  [TRX A] --보유--> [Lock X] --대기--> [TRX B]
  [TRX B] --보유--> [Lock Y] --대기--> [TRX A]
  
  사이클(Cycle) = 데드락

제거 방법:
  사이클을 만들지 않으면 데드락 없음
  → 모든 트랜잭션이 동일한 순서로 Lock 획득 시
  → 순환이 불가능
  
  예시:
    모든 트랜잭션: 항상 id 오름차순으로 Lock
    TRX A: Lock(id=1) → Lock(id=2)
    TRX B: Lock(id=1) 대기 → Lock(id=2)
    → 사이클 없음 → 데드락 없음

InnoDB의 감지 (마지막 안전망):
  순환 의존이 실제로 발생하면
  → Wait-for Graph에서 사이클 감지
  → 비용 가장 낮은 트랜잭션 롤백 (Victim)
  → 나머지 트랜잭션 계속 진행
  → 롤백된 트랜잭션: ERROR 1213 수신

이상적 전략:
  1순위: 설계로 데드락 발생 원인 제거
  2순위: InnoDB 감지 + 애플리케이션 재시도 로직 (보험)
```

---

## 🔬 내부 동작 원리

### 1. 데드락의 4가지 Coffman 조건

```
4가지 조건 모두 충족 시 데드락 발생:

조건 1 — Mutual Exclusion (상호 배제):
  X Lock은 단 하나의 트랜잭션만 보유 가능
  InnoDB 해당: ✅ X-X 비호환

조건 2 — Hold and Wait (보유하며 대기):
  Lock을 보유한 상태에서 다른 Lock을 기다림
  InnoDB 해당: ✅ UPDATE id=1 실행 중 UPDATE id=2 시도

조건 3 — No Preemption (선점 불가):
  Lock을 외부에서 강제로 해제할 수 없음
  InnoDB 해당: ✅ 트랜잭션이 ROLLBACK해야만 Lock 해제
  (KILL로 연결 종료 시 자동 ROLLBACK → Lock 해제 가능, 외부 강제 아님)

조건 4 — Circular Wait (순환 대기):
  TRX A → TRX B 기다림, TRX B → TRX A 기다림
  InnoDB 해당: ✅ Wait-for Graph에서 사이클 형성

4가지 조건 중 하나라도 제거하면 데드락 없음:

  조건 4 제거 (가장 실용적):
    모든 트랜잭션이 동일한 순서로 Lock 획득
    → 순환 형성 불가 → 데드락 불가
    Long fromFirst = Math.min(fromId, toId);
    Long toSecond  = Math.max(fromId, toId);
    lock(fromFirst); lock(toSecond);

  조건 2 부분 제거:
    트랜잭션 시작 시 필요한 모든 Lock을 한 번에 획득 (Pre-claim)
    → 실용적이지 않음 (어떤 Lock이 필요할지 미리 알기 어려움)
    → InnoDB에서 지원 안 함
```

### 2. Wait-for Graph 기반 데드락 감지

```
Wait-for Graph 구조:
  노드 (Node):    각 트랜잭션
  간선 (Edge):    TRX A → TRX B = "A가 B를 기다린다"

사이클 탐색 (DFS):
  새로운 Lock 대기 발생 시마다 그래프 업데이트 후 탐색
  DFS로 사이클 존재 여부 확인

예시:
  TRX A → TRX B 대기 (A가 B의 Lock 기다림)
  TRX B → TRX C 대기
  TRX C → TRX A 대기
  
  DFS: A → B → C → A  ← 사이클! 데드락!

감지 시점:
  Lock 대기 큐에 새 항목 추가 시마다 실행
  → 데드락 발생 즉시 감지 (밀리초 내)

n개의 트랜잭션이 대기 중일 때 DFS 비용:
  O(N + E) [N = 트랜잭션 수, E = 대기 간선 수]
  동시 대기 수천 개 → DFS 비용 급증
  → innodb_deadlock_detect = OFF 필요한 상황

탐지 후 처리:
  사이클에 포함된 트랜잭션 중 하나를 Victim으로 선택
  Victim: ROLLBACK (Undo Log 역방향 적용)
  나머지: Lock 획득 후 정상 진행
  에러: Victim 트랜잭션에 ERROR 1213 전송
```

### 3. Victim 선택 — trx_weight 기반

```sql
-- Victim 선택 기준:
-- 가장 낮은 trx_weight를 가진 트랜잭션 선택
-- trx_weight = trx_rows_modified + trx_rows_locked

-- 낮은 weight = 롤백 비용이 가장 적음 = "희생시키기 가장 쉬운 것"

SELECT
    trx_id,
    trx_rows_modified,
    trx_rows_locked,
    trx_rows_modified + trx_rows_locked AS trx_weight,
    LEFT(trx_query, 80) AS query
FROM information_schema.INNODB_TRX
ORDER BY trx_weight;
-- 가장 낮은 weight의 트랜잭션이 데드락 발생 시 Victim 후보

-- Victim 선택의 비결정성:
-- 여러 트랜잭션의 weight가 동일하면 내부 알고리즘에 의해 결정
-- 항상 같은 트랜잭션이 Victim이 되는 것은 아님
-- → "왜 내 트랜잭션만 계속 롤백되지?" 의문의 원인

-- Victim 트랜잭션이 받는 에러:
-- ERROR 1213 (40001): Deadlock found when trying to get lock;
--   try restarting transaction
-- SQLSTATE: 40001 → Spring의 DeadlockLoserDataAccessException
```

### 4. innodb_deadlock_detect OFF의 영향

```
innodb_deadlock_detect = ON (기본값):
  모든 Lock 대기 발생 시 Wait-for Graph 탐색
  사이클 발견 즉시 → Victim 롤백
  비용: DFS 탐색 (동시 대기가 많을수록 비용 증가)

innodb_deadlock_detect = OFF:
  Wait-for Graph 탐색 없음
  데드락이 발생해도 탐지 못함
  → innodb_lock_wait_timeout 경과 후 Lock Timeout으로 처리
  → 기본 50초! 두 트랜잭션 모두 50초 동안 차단

  성능 향상:
    DFS 탐색 오버헤드 제거
    고동시성(수천 TPS) 환경에서 유의미한 차이
    특히: 대부분의 Lock 대기가 데드락이 아닌 경우 (합법적 직렬화)

  위험:
    데드락 = 두 트랜잭션 모두 50초 대기
    Connection 고갈 → 서비스 전체 영향
    → innodb_lock_wait_timeout을 최소한 낮춰야 함 (5~10초)

  적합한 환경:
    데드락이 극히 드문 시스템
    Lock 대기가 합법적인 경우가 대부분
    초당 수만 TPS의 고동시성 시스템

  적합하지 않은 환경:
    복잡한 쿼리 패턴으로 데드락 가능성 있는 시스템
    Lock 대기가 50초면 서비스 장애인 시스템

innodb_lock_wait_timeout 설정:
  기본값: 50초 (너무 길다!)
  프로덕션 권장: 5~10초
  
  SET GLOBAL innodb_lock_wait_timeout = 10;
  -- 설정 후 에러: ERROR 1205 (HY000): Lock wait timeout exceeded;
  --   try restarting transaction
```

### 5. 데드락 예방 전략의 원리

```
전략 1: Lock 획득 순서 통일 (가장 효과적)

  원리: 순환 대기(Circular Wait) 조건 제거
  구현:
    항상 id 오름차순으로 Lock 획득
    → TRX A: Lock(1) → Lock(2)
    → TRX B: Lock(1) 먼저 시도 → 대기
    → 순환 없음 → 데드락 없음

  Java 구현:
    List<Long> ids = Arrays.asList(fromId, toId);
    Collections.sort(ids);  // 오름차순 정렬
    for (Long id : ids) {
        accountRepository.findByIdWithWriteLock(id);  // FOR UPDATE
    }

전략 2: 트랜잭션 크기 최소화

  원리: Lock 보유 시간 단축 → 충돌 확률 감소
  구현:
    비즈니스 로직을 트랜잭션 밖으로 이동
    트랜잭션 내에서 외부 IO(HTTP 호출) 제거
    배치 처리를 작은 단위로 분리

전략 3: Optimistic Lock 활용

  원리: Lock 없이 실행 → 충돌 발생 시 재시도
  데드락 완전 제거 (Lock이 없으므로)
  적합: 충돌 빈도 낮은 경우

전략 4: SELECT FOR UPDATE 대신 단일 UPDATE

  원리: 읽고 계산하고 쓰는 3단계를 1단계로
  SELECT stock FOR UPDATE;
  UPDATE SET stock = stock - qty;  ← 2개 문장
  
  →  UPDATE products SET stock = stock - qty
     WHERE id = ? AND stock >= qty;  ← 1개 문장
  Lock 보유 시간 단축 → 충돌 감소
```

---

## 💻 실전 실험

### 실험 1: 의도적 데드락 발생 및 에러 확인

```sql
CREATE TABLE deadlock_test (
    id      INT PRIMARY KEY,
    balance INT NOT NULL
) ENGINE=InnoDB;
INSERT INTO deadlock_test VALUES (1, 1000), (2, 2000);

-- 세션 1
START TRANSACTION;
UPDATE deadlock_test SET balance = balance - 100 WHERE id = 1;
-- id=1 X Lock 획득

-- 세션 2
START TRANSACTION;
UPDATE deadlock_test SET balance = balance - 200 WHERE id = 2;
-- id=2 X Lock 획득

-- 세션 1 (계속)
UPDATE deadlock_test SET balance = balance + 100 WHERE id = 2;
-- id=2 X Lock 요청 → 세션 2가 보유 → 대기

-- 세션 2 (계속)
UPDATE deadlock_test SET balance = balance + 200 WHERE id = 1;
-- id=1 X Lock 요청 → 세션 1이 보유 → 데드락!
-- 즉시 에러: ERROR 1213 Deadlock

-- 데드락 로그 확인
SHOW ENGINE INNODB STATUS\G
-- "LATEST DETECTED DEADLOCK" 섹션 확인

DROP TABLE deadlock_test;
```

### 실험 2: Lock 순서 통일로 데드락 방지

```sql
CREATE TABLE accounts_test (id INT PRIMARY KEY, balance INT);
INSERT INTO accounts_test VALUES (1, 10000), (2, 5000);

-- 순서 통일 전 (데드락 가능):
-- 항상 from → to 순서
-- from=2, to=1이면: id=2 Lock → id=1 Lock
-- 동시에 from=1, to=2이면: id=1 Lock → id=2 Lock → 순서 역전 → 데드락

-- 순서 통일 후 (데드락 불가):
-- 항상 작은 id 먼저
-- from=2, to=1이면: id=1 Lock → id=2 Lock (통일)
-- from=1, to=2이면: id=1 Lock → id=2 Lock (동일 순서)
-- → 사이클 불가 → 데드락 불가

-- 세션 1 (정렬된 순서)
START TRANSACTION;
SELECT * FROM accounts_test WHERE id = 1 FOR UPDATE;  -- 작은 id 먼저
SELECT * FROM accounts_test WHERE id = 2 FOR UPDATE;  -- 다음 id

-- 세션 2 (동일한 정렬된 순서)
START TRANSACTION;
SELECT * FROM accounts_test WHERE id = 1 FOR UPDATE;  -- 세션 1 대기 (Lock 충돌)
-- 세션 1이 COMMIT 후에야 진행 (데드락 없음, 직렬화됨)

COMMIT;  -- 세션 1
-- 세션 2 진행됨
DROP TABLE accounts_test;
```

### 실험 3: innodb_lock_wait_timeout과 데드락 비교

```sql
-- Lock Timeout 에러 (데드락이 아닌 경우)
SET SESSION innodb_lock_wait_timeout = 3;  -- 3초

-- 세션 1
START TRANSACTION;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;

-- 세션 2
START TRANSACTION;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- 3초 대기 후:
-- ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
-- (데드락이 아닌 단순 대기 → Timeout)

-- 데드락 에러 (순환 대기):
-- ERROR 1213 (40001): Deadlock found when trying to get lock;
--                     try restarting transaction

-- 에러 코드 차이:
-- 1205: Lock wait timeout exceeded → innodb_lock_wait_timeout 초과
-- 1213: Deadlock → Wait-for Graph에서 사이클 감지

SET SESSION innodb_lock_wait_timeout = 50;  -- 기본값 복원
```

### 실험 4: 데드락 통계 모니터링

```sql
-- 전체 데드락 발생 횟수
SHOW STATUS LIKE 'Innodb_deadlocks';
-- Innodb_deadlocks: 누적 데드락 발생 횟수 (MySQL 8.0.11+)

-- INFORMATION_SCHEMA로 상세 통계
SELECT * FROM information_schema.INNODB_METRICS
WHERE NAME IN ('lock_deadlocks', 'lock_timeouts', 'lock_row_lock_waits')
  AND STATUS = 'enabled'\G

-- 데드락 통계 활성화 (비활성인 경우):
SET GLOBAL innodb_monitor_enable = 'lock_deadlocks';
SET GLOBAL innodb_monitor_enable = 'lock_timeouts';

-- 주기적 모니터링 (초당 발생률):
-- 분당 발생률 = SHOW STATUS 10초 간격으로 두 번 조회 후 차이 계산
```

---

## 📊 성능 비교

```
innodb_deadlock_detect 설정별 성능:

ON (기본):
  매 Lock 대기마다 DFS 탐색
  동시 대기 100개: DFS O(100 + 간선 수) 실행
  고동시성(수천 TPS): DFS 오버헤드 + latch 경합 → CPU ~5~10% 추가
  
  장점: 데드락 즉시 감지 (밀리초) → 빠른 복구
  단점: 고동시성에서 오버헤드

OFF:
  DFS 탐색 없음 → CPU 절약
  데드락 발생 시 innodb_lock_wait_timeout(50초) 대기
  
  장점: 고동시성에서 성능 향상
  단점: 데드락 해소에 최대 50초 → 연쇄 차단 위험
  
  OFF 시 필수 설정:
    innodb_lock_wait_timeout = 5  (50초에서 5초로 단축)

데드락 발생 빈도별 전략:
  낮음 (분당 < 1건):
    → 설계 개선 + @Retryable로 충분
    → innodb_deadlock_detect = ON 유지
  
  높음 (분당 수십 건 이상):
    → 설계 개선이 우선 (순서 통일, Gap Lock 최소화)
    → 근본 해결 후에도 발생 시 READ_COMMITTED 고려
```

---

## ⚖️ 트레이드오프

```
데드락 감지 ON vs OFF:

ON (기본):
  ✅ 즉시 감지 → 빠른 복구 → 사용자 응답 빠름
  ✅ 데드락 원인 로그 기록 → 분석/개선 가능
  ❌ 고동시성에서 DFS 오버헤드 (CPU ~5%)

OFF:
  ✅ DFS 오버헤드 제거 → 고TPS에서 성능 향상
  ❌ 데드락 해소까지 최대 timeout 초 대기
  ❌ 연쇄 차단 → 서비스 영향 큼
  조건: innodb_lock_wait_timeout을 반드시 낮추기 (5~10초)
        데드락이 극히 드문 시스템에서만

재시도 전략:
  @Retryable만으로 해결:
    ✅ 단순 구현
    ❌ 근본 원인 미해결 → 재시도에도 같은 데드락 가능
    → 반드시 Lock 순서 통일과 함께 사용

  Lock 순서 통일 + @Retryable:
    ✅ 데드락 발생률 0에 가깝게 + 혹시 발생 시 재시도
    ✅ 가장 안전한 조합

  @Version (Optimistic Lock) + @Retryable:
    ✅ Lock 없음 → 데드락 불가능
    ✅ 낮은 처리량 저하
    ❌ 외부 IO 포함 트랜잭션에 부적합
    ❌ 충돌 빈도 높으면 재시도 폭증
```

---

## 📌 핵심 정리

```
데드락 핵심:

발생 조건 (Coffman 4가지):
  상호 배제 + 보유하며 대기 + 선점 불가 + 순환 대기
  → 4번째(순환 대기) 제거가 가장 실용적

Wait-for Graph:
  트랜잭션 = 노드, 대기 = 간선
  DFS로 사이클 탐색 = 데드락 감지
  매 Lock 대기 발생 시 탐색

Victim 선택:
  trx_weight = trx_rows_modified + trx_rows_locked
  가장 낮은 weight = ROLLBACK (롤백 비용 최소화)
  에러: ERROR 1213 → DeadlockLoserDataAccessException

innodb_deadlock_detect:
  ON (기본): DFS 탐색으로 즉시 감지
  OFF: 성능 향상, 데드락 = Timeout으로 처리
  OFF 시 innodb_lock_wait_timeout 반드시 단축

예방 전략 (우선순위):
  1. Lock 획득 순서 통일 (가장 효과적)
  2. 트랜잭션 크기 최소화
  3. Optimistic Lock (@Version)
  4. READ_COMMITTED (Gap Lock Deadlock)
  5. @Retryable (마지막 안전망)
```

---

## 🤔 생각해볼 문제

**Q1.** 세 트랜잭션 A → B → C → A의 순환 대기가 발생했다. InnoDB는 어떤 기준으로 세 트랜잭션 중 하나를 Victim으로 선택하는가? 만약 세 트랜잭션의 `trx_weight`가 동일하다면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**trx_weight 기준 (가장 낮은 값 선택)**:

세 트랜잭션의 trx_weight를 계산합니다:
- `trx_weight = trx_rows_modified + trx_rows_locked`

가장 낮은 trx_weight를 가진 트랜잭션이 Victim으로 선택됩니다. 이유: 롤백 비용이 가장 낮아 시스템 전체 영향이 최소화됩니다.

**trx_weight가 동일한 경우**:

InnoDB는 내부적으로 트랜잭션 ID(낮은 것 우선) 또는 기타 내부 기준으로 Victim을 결정합니다. 이 선택은 결정론적(deterministic)이지 않아 매번 다른 트랜잭션이 Victim이 될 수 있습니다.

실무적 의미: "내 트랜잭션만 항상 롤백된다"면 해당 트랜잭션의 trx_weight가 낮게 유지되는 것입니다 (간단한 작업). 중요한 비즈니스 로직을 가진 트랜잭션이 Victim이 되지 않도록 하려면 해당 트랜잭션을 먼저 더 많은 작업을 하도록 설계하거나(weight 증가), 애초에 데드락이 발생하지 않도록 Lock 순서를 통일해야 합니다.

</details>

---

**Q2.** Spring에서 다음 코드가 데드락 재시도를 올바르게 처리하지 못하는 이유는?

```java
@Retryable(
    value = DeadlockLoserDataAccessException.class,
    maxAttempts = 3
)
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepo.findByIdWithLock(fromId);  // FROM 먼저
    Account to   = accountRepo.findByIdWithLock(toId);    // TO 다음
    from.withdraw(amount);
    to.deposit(amount);
}
```

<details>
<summary>해설 보기</summary>

**두 가지 문제**가 있습니다.

**문제 1: @Retryable과 @Transactional의 AOP 순서**

Spring AOP에서 `@Retryable`과 `@Transactional`이 함께 있을 때, 두 어노테이션의 프록시 순서가 중요합니다. 기본 설정에서 `@Transactional`이 안쪽 프록시이면, 데드락으로 인한 예외가 트랜잭션 롤백 후 외부로 전파되어 `@Retryable`이 잡아 재시도할 수 있습니다. 하지만 만약 `@Transactional`이 바깥쪽이면 재시도 시 새 트랜잭션이 시작되지 않아 재시도가 같은 트랜잭션 내에서 실행됩니다. → `@Retryable` 메서드와 `@Transactional` 메서드를 분리해야 합니다.

**문제 2: 근본 원인 미해결 (Lock 순서 불일치)**

`fromId=2, toId=1`로 호출 시: Lock(id=2) → Lock(id=1)  
동시에 `fromId=1, toId=2`로 호출 시: Lock(id=1) → Lock(id=2)  
→ 재시도를 3번 해도 같은 패턴으로 계속 데드락 발생 가능

**올바른 코드**:
```java
@Retryable(DeadlockLoserDataAccessException.class, maxAttempts = 3)
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    doTransfer(fromId, toId, amount);  // @Transactional 분리
}

@Transactional
protected void doTransfer(Long fromId, Long toId, BigDecimal amount) {
    // 순서 통일: 항상 작은 ID 먼저
    Long first = Math.min(fromId, toId);
    Long second = Math.max(fromId, toId);
    Account a1 = accountRepo.findByIdWithLock(first);
    Account a2 = accountRepo.findByIdWithLock(second);
    // ... 처리
}
```

</details>

---

**Q3.** `innodb_deadlock_detect = OFF` + `innodb_lock_wait_timeout = 5`로 설정했다. 이 상태에서 실제 데드락이 발생하면 어떤 순서로 사건이 진행되고, 5초 후 어떤 결과가 나오는가?

<details>
<summary>해설 보기</summary>

**사건 진행 순서**:

1. TRX A와 TRX B가 순환 대기 상태에 진입 (데드락 발생)
2. `innodb_deadlock_detect = OFF` → Wait-for Graph DFS 탐색 없음 → 즉각 감지 안 됨
3. TRX A와 TRX B **모두** `innodb_lock_wait_timeout = 5초` 타이머 시작
4. 5초 경과
5. TRX A와 TRX B **중 먼저 타임아웃 된 것** (또는 둘 다 동시에)이 **ERROR 1205** 수신
   - `ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction`
   - 이 트랜잭션 자동 ROLLBACK
6. 나머지 트랜잭션: Lock이 해제됨 → Lock 획득 → 정상 진행

**`innodb_deadlock_detect = ON`과의 차이**:
- ON: 데드락 감지 즉시(~ms) → 한쪽 ROLLBACK → ERROR 1213
- OFF: 5초 대기 → 타임아웃 → ERROR 1205

**5초의 영향**:
- 5초 동안 두 트랜잭션이 차단
- 해당 Connection 2개는 5초 동안 다른 요청 처리 불가
- Connection Pool에 여유가 없다면 새 요청도 차단 → 연쇄 영향
- 데드락이 빈번하면 → 수십 개의 Connection이 동시에 5초 차단 → 서비스 장애

**결론**: `OFF + 5초`는 `ON`보다 복구 시간이 5초 더 길어지므로 데드락이 드문 환경에서만 적합합니다.

</details>

---

<div align="center">

**[⬅️ Gap Lock](./03-gap-lock-next-key-lock.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데드락 로그 분석 ➡️](./05-deadlock-log-analysis.md)**

</div>
