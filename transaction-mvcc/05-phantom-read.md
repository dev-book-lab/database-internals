# Phantom Read — 발생 조건과 방지 방법

---

## 🎯 핵심 질문

- Phantom Read와 Non-Repeatable Read의 차이는 무엇인가?
- REPEATABLE READ에서 Consistent Read(스냅샷 읽기)는 Phantom Read를 막는가?
- Current Read(잠금 읽기)에서 Phantom Read가 발생하는 이유는?
- Gap Lock이 어떻게 삽입을 차단하여 Phantom Read를 방지하는가?
- Next-Key Lock은 Gap Lock과 Record Lock의 조합인가?

---

## 🔬 내부 동작 원리

### 1. Phantom Read 정의

```
Non-Repeatable Read:
  동일 Row를 두 번 읽었을 때 값이 다름
  → 기존 Row의 값 변경(UPDATE)에 의해 발생

Phantom Read:
  동일 범위 쿼리를 두 번 실행했을 때 Row 수가 다름
  → 범위에 새 Row 삽입(INSERT) 또는 기존 Row 삭제(DELETE)에 의해 발생
  → "유령(Phantom) Row가 나타나거나 사라진다"

예시:
  TRX A (T1): SELECT COUNT(*) FROM orders WHERE amount > 1000  → 50건
  TRX B:       INSERT INTO orders (amount) VALUES (2000); COMMIT;
  TRX A (T2): SELECT COUNT(*) FROM orders WHERE amount > 1000  → 51건!
  → Phantom Read: 새 Row(2000)가 갑자기 나타남
```

### 2. Consistent Read에서의 Phantom Read (REPEATABLE READ)

```sql
-- REPEATABLE READ + 일반 SELECT

-- TRX A
START TRANSACTION;
SELECT COUNT(*) FROM orders WHERE amount > 1000;  -- 50건 (Read View 생성)

-- TRX B
INSERT INTO orders (user_id, amount) VALUES (1, 2000);
COMMIT;

-- TRX A (계속)
SELECT COUNT(*) FROM orders WHERE amount > 1000;  -- 여전히 50건!

이유:
  REPEATABLE READ의 Read View가 고정됨
  TRX B의 INSERT → 새 Row의 DB_TRX_ID = TRX B의 ID
  TRX A의 Read View: TRX B가 활성이었거나 이후에 시작됨
  → TRX B의 Row가 보이지 않음
  → Consistent Read (일반 SELECT)는 Phantom Read 방지!

결론:
  REPEATABLE READ + 일반 SELECT → Phantom Read 없음 (Read View 덕분)
  REPEATABLE READ + FOR UPDATE/SHARE → Phantom Read 발생 가능!
```

### 3. Current Read에서의 Phantom Read

```sql
-- Current Read: SELECT ... FOR UPDATE, SELECT ... FOR SHARE, UPDATE, DELETE, INSERT
-- → MVCC 버전이 아닌 "최신 커밋된 버전"을 읽고 Lock 획득

-- TRX A (REPEATABLE READ)
START TRANSACTION;
SELECT COUNT(*) FROM orders WHERE amount > 1000 FOR UPDATE;  -- 50건, 50개 Row에 X Lock

-- TRX B
INSERT INTO orders (user_id, amount) VALUES (1, 2000);
COMMIT;  -- 새 Row 삽입 (TRX A의 Lock 범위 밖에 있을 수 있음!)

-- TRX A (계속)
SELECT COUNT(*) FROM orders WHERE amount > 1000 FOR UPDATE;  -- 51건! (Phantom!)

이유:
  FOR UPDATE = Current Read = 최신 커밋 버전 읽기
  TRX B가 커밋한 새 Row(amount=2000)가 보임
  첫 번째 SELECT FOR UPDATE 시 기존 50개 Row에만 Lock
  새 Row가 그 사이에 삽입되면 두 번째 SELECT에서 보임
  → Phantom Read 발생!

InnoDB의 해결책: Gap Lock
```

### 4. Gap Lock — 삽입 차단으로 Phantom Read 방지

```
Gap Lock:
  "이 인덱스 범위에 새로운 Row 삽입을 차단하는 Lock"
  Record Lock(특정 Row)이 아닌 "공간(Gap)"에 대한 Lock

예시:
  orders 테이블, idx_amount (amount 인덱스)
  현재 데이터: amount = [500, 1000, 1500, 2000]
  
  SELECT ... FOR UPDATE WHERE amount > 1000
  
  조건에 해당하는 Record Lock: 1500, 2000 (X Lock)
  Gap Lock (amount > 1000인 공간):
    (1000, 1500) 사이의 Gap
    (1500, 2000) 사이의 Gap
    (2000, +∞) 사이의 Gap
  
  이 Gap에 INSERT 시도 → 차단!
  → TRX A가 COMMIT할 때까지 새 amount > 1000 삽입 불가
  → Phantom Read 방지

Next-Key Lock:
  Record Lock + 해당 Record 이전 Gap Lock의 조합
  InnoDB의 기본 Lock 단위 (REPEATABLE READ에서)
  
  amount = 1500에 대한 Next-Key Lock:
    Record Lock on 1500
    Gap Lock on (1000, 1500)
  
  → Record도 보호, 해당 Record 이전 Gap도 보호
```

### 5. Gap Lock의 부작용 — Deadlock

```sql
-- Gap Lock이 많을수록 Deadlock 위험 증가

-- 세션 1
START TRANSACTION;
SELECT * FROM orders WHERE amount BETWEEN 1000 AND 2000 FOR UPDATE;
-- Gap Lock: (999, 1000), 1000, (1000, 1500), 1500, (1500, 2000), 2000, (2000, ...)

-- 세션 2
START TRANSACTION;
SELECT * FROM orders WHERE amount BETWEEN 1500 AND 3000 FOR UPDATE;
-- Gap Lock: ..., (1500, 2000), 2000, (2000, ∞)
-- 세션 1의 Gap Lock과 겹침 → 대기

-- 세션 1 (계속)
INSERT INTO orders (amount) VALUES (2500);
-- 세션 2의 Gap Lock에 의해 차단

-- 결과: Deadlock!
-- InnoDB 감지 → 한쪽 롤백

-- Gap Lock 줄이는 방법:
-- 1. READ COMMITTED 사용 (Gap Lock 없음, Phantom Read 허용)
-- 2. 정확한 PK/Unique Key 조건으로 Point Lookup (Gap 없이 Record Lock만)
-- 3. 범위를 최소화하여 Lock 범위 감소
```

---

## 💻 실전 실험

### 실험 1: Consistent Read vs Current Read의 Phantom 차이

```sql
-- 세션 1: REPEATABLE READ
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;

-- 일반 SELECT (Consistent Read)
SELECT COUNT(*) FROM orders WHERE amount > 5000;  -- 기준값 기록

-- 세션 2: 새 Row 삽입
INSERT INTO orders (user_id, amount, status) VALUES (1, 9999, 'PAID');
COMMIT;

-- 세션 1 (계속)
SELECT COUNT(*) FROM orders WHERE amount > 5000;
-- → 증가 안 함 (Consistent Read, Read View 고정)

SELECT COUNT(*) FROM orders WHERE amount > 5000 FOR UPDATE;
-- → 증가! (Current Read, 최신 커밋 읽음 → Phantom!)

COMMIT;
```

### 실험 2: Gap Lock 확인

```sql
-- 세션 1: FOR UPDATE로 Gap Lock 획득
START TRANSACTION;
SELECT * FROM orders WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE;

-- 세션 2: 범위 내 삽입 시도
INSERT INTO orders (user_id, amount, status) VALUES (1, 3000, 'PAID');
-- → 차단! (세션 1이 COMMIT할 때까지 대기)

-- Gap Lock 확인
SELECT * FROM performance_schema.data_locks
WHERE LOCK_TYPE = 'RECORD'
  AND LOCK_MODE LIKE '%GAP%'\G

COMMIT;  -- 세션 1 COMMIT → 세션 2 INSERT 완료
```

---

## 📌 핵심 정리

```
Phantom Read 핵심:

정의:
  범위 쿼리를 같은 트랜잭션에서 반복 시 Row 수가 달라지는 현상
  원인: 범위에 INSERT/DELETE 발생

REPEATABLE READ에서:
  Consistent Read (일반 SELECT): Phantom Read 없음
    → Read View 고정으로 새 Row 보이지 않음
  Current Read (FOR UPDATE 등): Phantom Read 발생 가능
    → 최신 커밋 버전 읽음

InnoDB 방지 수단:
  Gap Lock: 인덱스 범위의 "공간"에 삽입 차단 Lock
  Next-Key Lock: Record Lock + 이전 Gap Lock 조합
  → FOR UPDATE 범위 내 삽입을 차단하여 Phantom Read 방지

부작용:
  Gap Lock 범위가 클수록 Deadlock 위험 증가
  READ COMMITTED에서는 Gap Lock 없음 → Phantom Read 허용 대신 Deadlock 감소
```

---

## 🤔 생각해볼 문제

**Q1.** `READ COMMITTED`에서 Deadlock이 줄어드는 이유는 Gap Lock과 어떤 관계인가?

<details>
<summary>해설 보기</summary>

`READ COMMITTED`에서는 **Gap Lock을 사용하지 않습니다**. Record Lock만 사용하여 실제로 존재하는 Row에만 Lock이 걸립니다.

REPEATABLE READ의 Gap Lock은 "이 범위에 새 Row가 들어오면 Phantom Read가 발생하니, 미리 그 공간을 잠근다"는 개념입니다. 이 때문에 서로 다른 트랜잭션이 겹치는 Gap에 대해 동시에 Lock을 시도하면 Deadlock이 발생합니다.

READ COMMITTED는 "Phantom Read는 허용하되 Gap Lock으로 인한 Deadlock은 없앤다"는 트레이드오프를 선택합니다. 쓰기가 많은 시스템에서 Deadlock이 빈번하게 발생하면 READ COMMITTED로 낮추는 것이 실용적인 해결책일 수 있습니다 (Phantom Read의 실제 영향이 작다면).

</details>

---

<div align="center">

**[⬅️ 격리 수준](./04-isolation-levels.md)** | **[홈으로 🏠](../README.md)** | **[다음: Consistent vs Current Read ➡️](./06-consistent-read-vs-current-read.md)**

</div>
