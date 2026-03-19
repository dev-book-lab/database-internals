# MVCC — 락 없이 읽기 일관성을 보장하는 원리

---

## 🎯 핵심 질문

- `DB_TRX_ID`와 `DB_ROLL_PTR`는 Row 어디에 어떻게 저장되는가?
- Read View란 무엇이고 어느 순간에 생성되는가?
- "읽기가 쓰기를 차단하지 않는다"는 것이 가능한 이유는?
- 같은 SELECT를 두 번 실행했을 때 REPEATABLE_READ와 READ_COMMITTED가 다른 결과를 주는 이유는?
- MVCC가 없다면 Dirty Read를 막기 위해 어떤 Lock이 필요했을까?

---

## 🔍 왜 중요한가

```
Lock 기반 동시성 제어의 문제:
  읽기에도 Shared Lock → 다른 쓰기가 차단
  쓰기에도 Exclusive Lock → 다른 읽기가 차단
  → 동시성 ↓, 처리량 ↓

MVCC의 혁신:
  Row의 여러 버전을 동시에 유지
  읽기 → 내 트랜잭션 시점의 스냅샷 버전 읽기 (Lock 불필요)
  쓰기 → 새 버전 생성 (기존 버전 유지)
  
  결과:
    읽기와 쓰기가 서로 차단하지 않음
    → 동시성 대폭 향상
    → 실제 웹 서비스 워크로드에 최적
```

---

## 🔬 내부 동작 원리

### 1. Row의 숨겨진 컬럼

```
InnoDB Row = 사용자 정의 컬럼 + 숨겨진 시스템 컬럼

숨겨진 시스템 컬럼:
  ┌─────────────────────────────────────────────────┐
  │ [사용자 컬럼들] │ DB_TRX_ID │ DB_ROLL_PTR │ DB_ROW_ID │
  └─────────────────────────────────────────────────┘
  
  DB_TRX_ID (6 bytes):
    이 Row를 마지막으로 변경한 트랜잭션 ID
    단조 증가 (트랜잭션 시작 순서 반영)
  
  DB_ROLL_PTR (7 bytes):
    이전 버전을 가리키는 포인터 (Undo Log 위치)
    버전 체인의 링크 역할
    INSERT로 생성된 Row는 NULL (이전 버전 없음)
  
  DB_ROW_ID (6 bytes):
    PK가 없을 때만 자동 생성되는 내부 식별자
    PK가 있으면 존재하지 않음

버전 체인 예시:
  최초 Row:
    [id=1, name='Alice', amount=10000 | TRX_ID=100 | ROLL_PTR=NULL]
  
  TRX 200이 amount를 8000으로 업데이트:
    [id=1, name='Alice', amount=8000  | TRX_ID=200 | ROLL_PTR=→Undo1]
    ↓ Undo Log
    [id=1, name='Alice', amount=10000 | TRX_ID=100 | ROLL_PTR=NULL]  (이전 버전)
  
  TRX 300이 amount를 6000으로 업데이트:
    [id=1, name='Alice', amount=6000  | TRX_ID=300 | ROLL_PTR=→Undo2]
    ↓ Undo Log
    [id=1, name='Alice', amount=8000  | TRX_ID=200 | ROLL_PTR=→Undo1]
    ↓ Undo Log
    [id=1, name='Alice', amount=10000 | TRX_ID=100 | ROLL_PTR=NULL]
```

### 2. Read View — 스냅샷의 정의

```
Read View: "내 트랜잭션 시작 시점에 어떤 트랜잭션들이 활성 상태였는가"

Read View 구성:
  m_creator_trx_id: Read View를 만든 트랜잭션 ID
  m_ids:           Read View 생성 시점의 활성 트랜잭션 ID 목록 (아직 커밋 안 됨)
  m_up_limit_id:   m_ids 중 가장 작은 값 (이보다 작은 TRX_ID는 모두 커밋됨)
  m_low_limit_id:  Read View 생성 시점의 다음 TRX_ID (이보다 크거나 같으면 미래 트랜잭션)

Row 버전 가시성 판단 알고리즘 (row_trx_id = Row의 DB_TRX_ID):
  
  if row_trx_id == m_creator_trx_id:
    → 내 트랜잭션이 만든 버전 → 보임
  
  elif row_trx_id < m_up_limit_id:
    → Read View 생성 전에 커밋된 트랜잭션 → 보임
  
  elif row_trx_id >= m_low_limit_id:
    → Read View 생성 후 시작된 미래 트랜잭션 → 안 보임
  
  elif row_trx_id in m_ids:
    → Read View 생성 시점에 아직 활성인 트랜잭션 → 안 보임
  
  else:
    → Read View 생성 전에 커밋된 트랜잭션 → 보임
  
  안 보이면 → DB_ROLL_PTR을 따라 이전 버전으로 이동 → 반복

직관적 해석:
  "내 Read View 기준으로 이미 커밋된 데이터만 보인다
   미래에 커밋되는 데이터는 보이지 않는다"
```

### 3. REPEATABLE READ vs READ COMMITTED의 차이

```
REPEATABLE READ (기본값):
  Read View를 트랜잭션 시작 시 1번 생성
  이후 모든 SELECT는 동일한 Read View 사용
  
  결과:
    같은 SELECT를 반복해도 항상 같은 결과
    → Repeatable Read 보장
    트랜잭션 진행 중 다른 트랜잭션이 COMMIT해도 보이지 않음

READ COMMITTED:
  Read View를 각 SELECT마다 새로 생성
  
  결과:
    같은 SELECT를 반복하면 다른 결과 가능 (Non-Repeatable Read)
    항상 최신 커밋된 데이터 읽음

타임라인 예시:
  TRX A (REPEATABLE READ):
    T1: START TRANSACTION → Read View 생성 (활성: [TRX B])
    T2: SELECT amount FROM accounts WHERE id=1  → 10000 (TRX B 미커밋)
    
  TRX B:
    T3: UPDATE accounts SET amount=8000 WHERE id=1
    T4: COMMIT
    
  TRX A:
    T5: SELECT amount FROM accounts WHERE id=1
        → REPEATABLE READ: 10000 (Read View 그대로, TRX B는 활성이었음)
        → READ COMMITTED: 8000 (새 Read View, TRX B는 커밋됨)
```

### 4. 읽기가 쓰기를 차단하지 않는 이유

```
전통적 Lock 기반:
  읽기: Shared Lock (S Lock) 획득
  쓰기: Exclusive Lock (X Lock) 획득
  
  X Lock vs S Lock: 상호 배제
  → 읽기 중에는 쓰기 차단
  → 쓰기 중에는 읽기 차단

MVCC:
  읽기 (Consistent Read, 일반 SELECT):
    Lock 획득 없음!
    Read View + Undo Log 버전 체인으로 해결
    → 어떤 Lock도 차단받지 않음

  쓰기 (INSERT/UPDATE/DELETE):
    Row에 Exclusive Lock 획득
    새 버전 생성 → Undo Log에 이전 버전 보관
    → 동시 읽기 트랜잭션은 Undo Log의 이전 버전을 읽음

  결과:
    읽기: Lock 없음 → 쓰기에 차단받지 않음
    쓰기: X Lock → 다른 쓰기에만 차단, 읽기에는 무관
    
    → 읽기-쓰기 동시성 극대화
    → 웹 서비스의 Read-Heavy 워크로드에 최적
```

---

## 💻 실전 실험

### 실험 1: MVCC 버전 분기 확인

```sql
-- 세션 1 (REPEATABLE READ)
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT amount FROM accounts WHERE id = 1;  -- 10000

-- 세션 2 (별도 창)
UPDATE accounts SET amount = 8000 WHERE id = 1;
COMMIT;

-- 세션 1 (계속)
SELECT amount FROM accounts WHERE id = 1;  -- 여전히 10000 (Read View 고정)
COMMIT;

-- 세션 1 재시작 후
START TRANSACTION;
SELECT amount FROM accounts WHERE id = 1;  -- 이제 8000 (새 Read View)
COMMIT;
```

### 실험 2: 숨겨진 컬럼 확인

```sql
-- DB_TRX_ID 확인 (직접 조회는 불가하지만 간접 확인)
-- information_schema.INNODB_TRX로 현재 트랜잭션 ID 확인

START TRANSACTION;
SELECT trx_id FROM information_schema.INNODB_TRX
WHERE trx_mysql_thread_id = CONNECTION_ID();
-- 현재 트랜잭션 ID

UPDATE accounts SET amount = 9000 WHERE id = 1;
-- 이 Row의 DB_TRX_ID = 위에서 확인한 trx_id로 설정됨

COMMIT;
```

### 실험 3: 읽기-쓰기 동시성 확인

```sql
-- 세션 1: 장시간 읽기
START TRANSACTION;
SELECT COUNT(*) FROM orders;  -- 전체 집계 (시간 소요)

-- 세션 2: 동시에 쓰기 (차단 없음)
INSERT INTO orders (user_id, amount) VALUES (1, 5000);  -- 바로 실행됨!
COMMIT;

-- 세션 1은 차단 없이 계속 실행
-- (Lock 기반이었다면 세션 2의 INSERT가 세션 1 SELECT 완료까지 대기)
```

---

## 📌 핵심 정리

```
MVCC 핵심:

Row 구조:
  숨겨진 DB_TRX_ID (마지막 변경 트랜잭션)
  숨겨진 DB_ROLL_PTR (이전 버전 Undo Log 포인터)
  → 버전 체인 형성

Read View:
  생성 시점의 활성 트랜잭션 목록 스냅샷
  Row의 TRX_ID와 비교 → 가시성 결정
  
  REPEATABLE READ: 트랜잭션 시작 시 1회 생성
  READ COMMITTED: 각 SELECT마다 새로 생성

읽기-쓰기 비차단:
  읽기: Lock 없음, Undo Log 버전 체인 참조
  쓰기: X Lock, 새 버전 생성 + Undo Log 보관
  → 두 작업이 서로 차단하지 않음
```

---

## 🤔 생각해볼 문제

**Q1.** 장시간 실행되는 분석 쿼리(30분)가 있을 때 MVCC가 시스템에 미치는 부작용은?

<details>
<summary>해설 보기</summary>

장시간 실행 트랜잭션은 오래된 Read View를 유지합니다. 이 Read View가 참조할 수 있는 모든 Undo Log 버전이 삭제될 수 없어 Undo Tablespace가 팽창합니다.

구체적으로: 분석 쿼리가 시작된 시점 이후 발생한 모든 UPDATE/DELETE의 이전 버전이 Undo Log에 쌓입니다. Purge Thread가 이 버전들을 삭제하려 해도 활성 Read View가 참조할 수 있으므로 삭제 불가입니다.

해결책: 분석 쿼리는 별도의 Read Replica에서 실행하거나, `innodb_undo_tablespace` 크기를 충분히 확보하거나, 쿼리를 페이지네이션으로 분할합니다.

</details>

**Q2.** MVCC에서 INSERT한 Row는 왜 DB_ROLL_PTR이 NULL인가?

<details>
<summary>해설 보기</summary>

INSERT로 새로 생성된 Row는 이전 버전이 없습니다. ROLL_PTR은 "이전 버전의 Undo Log 위치"를 가리키는 포인터인데, INSERT Row는 이전 상태 자체가 존재하지 않으므로 가리킬 곳이 없어 NULL입니다.

롤백 시 INSERT된 Row는 Undo Log를 사용해 복원하는 것이 아니라 단순히 해당 Row를 DELETE합니다. INSERT의 Undo Log에는 "이 PK의 Row를 삭제하라"는 정보만 저장됩니다.

반면 UPDATE/DELETE는 반드시 이전 버전이 존재하므로 ROLL_PTR이 Undo Log를 가리킵니다.

</details>

---

<div align="center">

**[⬅️ ACID 구현](./01-acid-implementation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Undo Log ➡️](./03-undo-log.md)**

</div>
