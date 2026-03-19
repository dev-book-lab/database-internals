# Gap Lock과 Next-Key Lock — Phantom Read 방지 메커니즘

---

## 🎯 핵심 질문

- Gap Lock이 잠그는 "간격(Gap)"은 정확히 어디에서 어디까지인가?
- Next-Key Lock = Record Lock + Gap Lock을 어떻게 계산하는가?
- 같은 Gap에 두 트랜잭션이 Gap Lock을 동시에 보유할 수 있는가?
- READ_COMMITTED에서 Gap Lock이 비활성화되면 어떤 결과가 생기는가?
- Insert Intention Lock은 Gap Lock과 어떻게 상호작용하는가?

---

## 🔬 내부 동작 원리

### 1. Gap Lock 범위 계산

```
인덱스 데이터: id = [1, 3, 5, 8, 12]

가능한 Gap:
  (-∞, 1):  id=1 이전
  (1, 3):   id=1과 id=3 사이
  (3, 5):   id=3과 id=5 사이
  (5, 8):   ...
  (8, 12):  ...
  (12, +∞): id=12 이후

SELECT * FROM t WHERE id = 5 FOR UPDATE;
  → Record Lock on id=5
  → Next-Key Lock: Gap (3, 5] + Record 5
  
  실제로는:
    Record Lock: id=5 (X)
    Gap Lock: (3, 5) — id=5 이전 Gap

SELECT * FROM t WHERE id BETWEEN 3 AND 8 FOR UPDATE;
  → Record Lock: id=3, 5, 8 (X)
  → Gap Lock: (1,3), (3,5), (5,8), (8,12)
  → Next-Key Lock: (1,3], (3,5], (5,8], (8,12]
  
  → (1, 12] 범위 전체가 보호됨
  → 이 범위 내 새 Row 삽입 차단!

범위 끝의 Gap:
  WHERE id > 8 FOR UPDATE
  → Record Lock: id=12
  → Gap Lock: (8,12), (12,+∞)
  → id>8인 모든 공간 보호

존재하지 않는 값 조건:
  WHERE id = 7 FOR UPDATE (id=7 없음)
  → Record Lock: 없음 (해당 레코드 없으므로)
  → Gap Lock: (5, 8) — id=7이 위치할 Gap
  → id=6, 7 삽입 차단!
```

### 2. Gap Lock의 공존 특성

```
Gap Lock은 여러 트랜잭션이 동일 Gap을 동시에 보유 가능!

이유:
  Gap Lock의 목적 = "이 공간에 삽입 차단"
  두 트랜잭션이 같은 Gap을 보호해도 충돌 없음
  (두 경비원이 같은 문을 동시에 지켜도 문제없음)

하지만 삽입 시도는 차단:
  TRX A: Gap (5,8) Gap Lock 보유
  TRX B: Gap (5,8) Gap Lock 보유
  → 두 Gap Lock 공존 ✅
  
  TRX C: id=6 INSERT 시도
  → Insert Intention Lock 획득 시도
  → TRX A의 Gap Lock과 충돌 → 대기
  → TRX B의 Gap Lock과도 충돌 → 대기

데드락 위험:
  TRX A: Gap Lock(5,8) 보유 → id=6 INSERT 시도
  TRX B: Gap Lock(5,8) 보유 → id=7 INSERT 시도
  
  → TRX A: TRX B의 Gap Lock 기다림
  → TRX B: TRX A의 Gap Lock 기다림
  → 데드락!
  
  이것이 Gap Lock이 Deadlock을 유발하는 흔한 패턴
```

### 3. Insert Intention Lock

```
INSERT 직전에 획득하는 특수한 Lock:

특성:
  Gap Lock의 일종이지만 서로 다른 위치에 삽입 시 공존 가능
  같은 Gap 내 다른 위치 = 공존 ✅
  같은 Gap 내 같은 위치 = 충돌 ❌

예시:
  Gap (5,8):
  TRX A: id=6 INSERT → Insert Intention Lock on (5,8) 위치 6
  TRX B: id=7 INSERT → Insert Intention Lock on (5,8) 위치 7
  → 다른 위치 → 공존 ✅ (동시 삽입 허용)

  TRX A: id=6 INSERT
  TRX B: id=6 INSERT (같은 위치)
  → 충돌 → 하나는 대기 (UNIQUE KEY 위반 또는 다음 처리)

Insert Intention Lock vs Gap Lock 충돌:
  Gap Lock (기존): 해당 Gap 전체 삽입 차단
  Insert Intention Lock (신규): 위 Gap Lock과 충돌 → 대기
  → Gap Lock이 있으면 삽입이 차단됨

LOCK_MODE 확인:
  SELECT LOCK_MODE FROM performance_schema.data_locks;
  -- X,INSERT_INTENTION → 삽입 직전 대기 중
```

### 4. READ_COMMITTED에서 Gap Lock 비활성화

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

START TRANSACTION;
SELECT * FROM orders WHERE amount BETWEEN 1000 AND 5000 FOR UPDATE;
-- READ_COMMITTED: Record Lock만 (1000~5000 사이 존재하는 Row에만)
-- Gap Lock 없음!

-- 다른 세션에서 amount=3000 INSERT → 즉시 성공 (Gap Lock 없으므로)
-- → Phantom Read 발생 가능

-- 장점:
-- Gap Lock 없음 → Deadlock 감소 (Gap Lock 충돌이 가장 큰 Deadlock 원인)
-- 삽입 동시성 향상

-- 단점:
-- Phantom Read 가능 (범위 재조회 시 다른 결과)
-- Statement-based Replication에서 안전하지 않음
--   (마스터와 슬레이브의 실행 순서 차이 → 다른 결과)
--   → READ COMMITTED + Row-based Replication 조합 필요

-- 실무 선택:
-- 쓰기 충돌/Deadlock이 많다 → READ COMMITTED 고려
-- 데이터 일관성 중요 → REPEATABLE READ 유지
-- READ COMMITTED 선택 시 → Row-based Replication 필수
```

### 5. Supremum 레코드와 Infimum

```
InnoDB B+Tree 특수 레코드:
  Infimum: 인덱스의 최솟값보다 작은 가상 레코드 (경계)
  Supremum: 인덱스의 최댓값보다 큰 가상 레코드 (경계)

Supremum Lock:
  WHERE id > 100 FOR UPDATE (id 최댓값 = 500)
  → Next-Key Lock on Supremum
  → (500, +∞) 범위 삽입 차단
  
  LOCK_DATA: supremum pseudo-record 확인 가능
  SELECT LOCK_DATA FROM performance_schema.data_locks;
  -- "supremum pseudo-record"

이 덕분에:
  범위 조건의 상한이 없는 경우에도 Phantom Read 방지
  미래에 추가될 최댓값 이상의 Row도 차단
```

---

## 💻 실전 실험

```sql
-- Gap Lock 범위 확인
CREATE TABLE gap_test (
    id INT PRIMARY KEY
) ENGINE=InnoDB;
INSERT INTO gap_test VALUES (1), (3), (5), (8), (12);

-- 세션 1
START TRANSACTION;
SELECT * FROM gap_test WHERE id = 5 FOR UPDATE;

-- Lock 상태 확인
SELECT LOCK_TYPE, LOCK_MODE, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'gap_test';
-- Record Lock on 5: X,REC_NOT_GAP
-- Gap Lock (3,5): X,GAP
-- (Next-Key Lock = 위 두 개 조합)

-- 세션 2: Gap 내 삽입 시도
INSERT INTO gap_test VALUES (4);  -- (3,5) Gap → 차단!
INSERT INTO gap_test VALUES (6);  -- (5,8) Gap → 통과! (Lock 범위 밖)

ROLLBACK;  -- 세션 1

-- READ_COMMITTED: Gap Lock 없음 확인
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM gap_test WHERE id = 5 FOR UPDATE;

SELECT LOCK_TYPE, LOCK_MODE, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'gap_test';
-- Record Lock only: X,REC_NOT_GAP on 5
-- Gap Lock 없음!

INSERT INTO gap_test VALUES (4);  -- 즉시 성공!
ROLLBACK;
```

---

## 📌 핵심 정리

```
Gap Lock / Next-Key Lock 핵심:

Gap Lock:
  인덱스 레코드 사이의 공간에 걸리는 Lock
  목적: 삽입 차단 → Phantom Read 방지
  특성: 동일 Gap에 여러 Gap Lock 공존 가능

Next-Key Lock:
  Record Lock + 해당 레코드 이전 Gap Lock 조합
  InnoDB REPEATABLE READ의 기본 Lock 단위

Lock 범위 계산:
  WHERE id = X → id=X 이전 Gap + id=X Record
  WHERE id BETWEEN A AND B → A~B 범위 + 양 끝 Gap
  존재하지 않는 값 → 해당 값이 위치할 Gap만

INSERT Intention Lock:
  삽입 직전 획득, Gap 내 다른 위치끼리 공존 가능
  기존 Gap Lock과 충돌 → 대기

READ_COMMITTED:
  Gap Lock 없음 → Deadlock 감소, Phantom Read 허용
  Row-based Replication과 함께 사용 필요
```

---

## 🤔 생각해볼 문제

**Q1.** `WHERE id IN (3, 7, 15) FOR UPDATE`를 실행할 때 id=3, 7, 15가 각각 존재/부재인 경우 Gap Lock은 어떻게 결정되는가? (인덱스 데이터: 1, 3, 5, 8, 12)

<details>
<summary>해설 보기</summary>

각 값별로 다르게 처리됩니다:

- **id=3 (존재)**: Record Lock on 3 + Gap Lock (1,3) → Next-Key Lock (1,3]
- **id=7 (없음)**: Record Lock 없음 + Gap Lock (5,8) → 위치할 Gap에만 Gap Lock
- **id=15 (없음, 최댓값 초과)**: Gap Lock (12, +∞) → Supremum Gap Lock

총 Lock 범위:
- (1,3]: id=3 보호
- (5,8): id=7이 삽입될 수 있는 Gap 보호
- (12, supremum): id=15가 삽입될 수 있는 범위 보호

`IN` 조건의 각 값은 독립적으로 처리되며, 존재 여부에 따라 Record Lock + Gap Lock 또는 Gap Lock만 걸립니다.

</details>

---

<div align="center">

**[⬅️ Row Lock](./02-row-lock-vs-table-lock.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데드락 감지 ➡️](./04-deadlock-detection.md)**

</div>
