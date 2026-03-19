# MVCC — 락 없이 읽기 일관성을 보장하는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `DB_TRX_ID`와 `DB_ROLL_PTR`은 Row의 어디에 어떻게 저장되는가?
- Read View의 `m_ids`, `m_up_limit_id`, `m_low_limit_id`는 각각 무엇이고 어떻게 사용되는가?
- Row 버전 가시성 판단 알고리즘을 단계별로 추적할 수 있는가?
- "읽기가 쓰기를 차단하지 않는다"는 것이 MVCC 없이는 왜 불가능한가?
- REPEATABLE READ와 READ COMMITTED가 다른 결과를 주는 이유를 Read View 생성 시점으로 설명할 수 있는가?
- 장시간 실행 트랜잭션이 Undo Log를 축적시키는 구체적 메커니즘은?

---

## 🔍 왜 이 개념이 중요한가

### "읽기와 쓰기가 서로를 차단하지 않는다"는 것의 기술적 의미

```
Lock 기반 동시성 제어 (MVCC 이전):
  읽기에도 S Lock → 다른 쓰기가 차단
  쓰기에도 X Lock → 다른 읽기가 차단

  초당 1000번의 SELECT + 100번의 UPDATE 서비스라면:
    모든 SELECT가 S Lock 보유 중 → UPDATE 대기
    UPDATE가 X Lock 보유 중 → 모든 SELECT 대기
    → 처리량 = 직렬 실행 수준으로 제한

MVCC (Multi-Version Concurrency Control):
  각 Row의 여러 버전을 동시에 유지
  읽기 → 내 트랜잭션 시점의 스냅샷 버전 읽기 (Lock 없음)
  쓰기 → 새 버전 생성 (기존 버전 유지, 이전 버전은 Undo Log로)

  결과:
    SELECT가 진행 중이어도 UPDATE 가능 (서로 차단 없음)
    UPDATE가 진행 중이어도 SELECT 가능
    → 웹 서비스의 Read-Heavy 워크로드에서 처리량 극대화

왜 JPA 개발자에게 중요한가:
  REPEATABLE READ 기본값이 Read View를 고정한다는 것을 모르면:
    → 트랜잭션 내에서 갱신된 데이터를 못 읽는 버그 발생
  READ COMMITTED로 바꿀 때 Non-Repeatable Read 발생 조건을 모르면:
    → 같은 메서드 내에서 데이터가 바뀌는 미스터리 현상 경험
```

---

## 😱 잘못된 이해

### Before: 트랜잭션 격리 = Lock으로만 구현된다는 오해

```sql
-- 잘못된 멘탈 모델:
-- "REPEATABLE READ는 읽는 Row에 S Lock을 걸어서 변경을 막는다"

-- 세션 1 (REPEATABLE READ)
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 10000

-- 세션 2
UPDATE accounts SET balance = 7000 WHERE id = 1;
COMMIT;

-- 세션 1
SELECT balance FROM accounts WHERE id = 1;
-- 잘못된 예상: S Lock이 있어서 세션 2의 UPDATE가 차단됐을 것
-- 실제: 세션 2의 UPDATE는 즉시 성공함 (MVCC라서 Lock 없음)
-- 세션 1의 결과: 10000 (Lock이 아닌 Read View로 이전 버전을 읽음)
```

```
잘못된 추론의 결과:
  "SELECT가 느려졌다 = Lock 경합이 생겼다"
  → 실제: SELECT는 MVCC로 처리되어 Lock 경합과 무관
  
  "READ COMMITTED로 바꾸면 Lock 경합이 줄어들어 빨라진다"
  → 실제: READ COMMITTED는 Gap Lock이 줄어서 빠를 수 있지만
          그 이유는 Lock 경합 감소가 아니라 Undo Log 버전 탐색 빈도 변화

  "MVCC는 낙관적 락의 일종이다"
  → 실제: MVCC와 낙관적 락은 전혀 다른 개념
          MVCC = 읽기 일관성 보장 (DB 엔진 내부)
          낙관적 락 = 쓰기 충돌 감지 (애플리케이션 레벨)
```

---

## ✨ 올바른 이해

### After: 버전 체인으로 적절한 스냅샷을 찾아가는 과정

```
MVCC의 핵심 구성 요소 3가지:

① Row의 숨겨진 컬럼 (버전 메타데이터):
  DB_TRX_ID  (6 bytes): 이 Row를 마지막으로 변경한 트랜잭션 ID
  DB_ROLL_PTR (7 bytes): 이전 버전이 있는 Undo Log 위치 포인터
  DB_ROW_ID  (6 bytes): PK 없을 때만 자동 생성 (있으면 존재 안 함)

② Undo Log (이전 버전 저장소):
  UPDATE/DELETE 시 이전 Row 값을 Undo Log에 기록
  DB_ROLL_PTR로 현재 버전 → 이전 버전 → 그 이전 버전 연결
  → "버전 체인(Version Chain)" 형성

③ Read View (내 트랜잭션의 스냅샷 기준):
  트랜잭션이 읽기 시작할 때 생성
  "이 시점에 어떤 트랜잭션들이 아직 활성 상태인가" 기록
  → Read View + 버전 체인으로 "내가 봐야 할 버전" 결정

핵심 원리:
  Row를 읽을 때 현재 버전의 DB_TRX_ID를 Read View와 비교
  "내가 봐야 할 버전이 맞는가?" 판단
  아니면 DB_ROLL_PTR 따라 이전 버전으로 이동 → 반복
```

---

## 🔬 내부 동작 원리

### 1. Row의 숨겨진 컬럼 상세

```
InnoDB Row 물리 구조 (COMPACT 포맷 기준):

  ┌──────────────────────────────────────────────────────────────┐
  │ 가변 컬럼 길이 목록 │ NULL 비트맵 │ Record Header (5 bytes)        │
  ├──────────────────────────────────────────────────────────────┤
  │ DB_TRX_ID (6 bytes) │ DB_ROLL_PTR (7 bytes) │                │
  ├──────────────────────────────────────────────────────────────┤
  │ 사용자 정의 컬럼들 (id, name, balance, ...)                      │
  └──────────────────────────────────────────────────────────────┘

DB_TRX_ID (6 bytes, 단조 증가):
  0 ~ 2^48 - 1 범위
  트랜잭션 시작 시 부여되는 고유 번호
  값이 클수록 나중에 시작된 트랜잭션
  → 크기 비교만으로 시간 순서 파악 가능

DB_ROLL_PTR (7 bytes):
  Undo Log 내 이전 버전의 정확한 위치 (파일 번호 + 오프셋)
  INSERT로 생성된 Row: NULL (이전 버전 없음)
  UPDATE/DELETE된 Row: Undo Log 위치를 가리킴

DB_ROW_ID (6 bytes, 조건부 존재):
  PK가 없고, UNIQUE NOT NULL 컬럼도 없을 때만 생성
  InnoDB가 내부적으로 Clustered Index 키로 사용
  권장: 반드시 PK를 명시적으로 정의 (DB_ROW_ID 오버헤드 방지)

버전 체인 예시:
  최초 삽입 (TRX 100):
    [id=1, balance=10000 | TRX_ID=100 | ROLL_PTR=NULL]

  TRX 200이 balance=8000으로 UPDATE:
    현재: [id=1, balance=8000  | TRX_ID=200 | ROLL_PTR=→UndoA]
    UndoA: [id=1, balance=10000 | TRX_ID=100 | ROLL_PTR=NULL]

  TRX 300이 balance=6000으로 UPDATE:
    현재: [id=1, balance=6000  | TRX_ID=300 | ROLL_PTR=→UndoB]
    UndoB: [id=1, balance=8000  | TRX_ID=200 | ROLL_PTR=→UndoA]
    UndoA: [id=1, balance=10000 | TRX_ID=100 | ROLL_PTR=NULL]
```

### 2. Read View 구조와 생성 시점

```
Read View 내부 필드:

  m_creator_trx_id:
    이 Read View를 만든 트랜잭션 ID
    "내가 변경한 버전은 항상 보인다"는 규칙에 사용

  m_ids (활성 트랜잭션 목록):
    Read View 생성 시점에 시작됐지만 아직 COMMIT 안 된 TRX ID 집합
    → 이 TRX들의 변경은 "커밋 안 된 것"이므로 보이면 안 됨

  m_up_limit_id:
    m_ids 중 가장 작은 값
    이 값보다 작은 TRX_ID는 Read View 생성 전에 모두 커밋됨
    → 반드시 보여야 하는 TRX들

  m_low_limit_id:
    Read View 생성 당시 다음에 발급될 TRX_ID
    이 값 이상인 TRX_ID는 Read View 생성 이후에 시작된 것
    → 절대 보이면 안 되는 미래 트랜잭션

타임라인 예시 (진행 중인 TRX: 150, 180, 220):
  m_up_limit_id = 150
  m_ids = {150, 180, 220}
  m_low_limit_id = 250 (다음 발급 예정)

  TRX_ID 100인 Row → 100 < 150 → 반드시 커밋됨 → 보임
  TRX_ID 180인 Row → 180 in {150,180,220} → 아직 활성 → 안 보임
  TRX_ID 260인 Row → 260 >= 250 → 미래 트랜잭션 → 안 보임
  TRX_ID 200인 Row → 150 <= 200 < 250, 200 not in {150,180,220}
                   → Read View 생성 전에 커밋됨 → 보임
```

### 3. 버전 가시성 판단 알고리즘

```
Row를 읽을 때 실행되는 pseudo-code:

function is_visible(row_trx_id, read_view):
  # 규칙 1: 내 트랜잭션이 만든 버전은 항상 보임
  if row_trx_id == read_view.m_creator_trx_id:
    return True

  # 규칙 2: Read View 생성 전에 커밋된 TRX → 보임
  if row_trx_id < read_view.m_up_limit_id:
    return True

  # 규칙 3: Read View 생성 이후 시작된 TRX → 안 보임
  if row_trx_id >= read_view.m_low_limit_id:
    return False

  # 규칙 4: Read View 생성 당시 활성인 TRX → 안 보임
  if row_trx_id in read_view.m_ids:
    return False

  # 규칙 5: 위 어디에도 해당 없음 = 생성 전 커밋 → 보임
  return True

버전 체인 탐색:
  current_version = 현재 Row
  while True:
    if is_visible(current_version.TRX_ID, read_view):
      return current_version  # 이 버전이 내가 읽어야 할 것
    if current_version.ROLL_PTR is None:
      return NULL  # 체인 끝 = 이 Row는 내 시점에 존재하지 않음
    current_version = undo_log[current_version.ROLL_PTR]  # 이전 버전으로

탐색 예시:
  Read View: m_ids={200, 250}, m_up_limit_id=200, m_low_limit_id=300

  현재 Row: TRX_ID=250
    → 250 in {200,250} → 안 보임 → ROLL_PTR 따라 이전 버전으로

  이전 버전(UndoB): TRX_ID=150
    → 150 < 200 (m_up_limit_id) → 보임! → 이 버전 반환
```

### 4. REPEATABLE READ vs READ COMMITTED — Read View 생성 시점의 차이

```
REPEATABLE READ (InnoDB 기본값):
  Read View 생성 시점: 트랜잭션의 첫 번째 읽기 연산 시 1번만 생성
  이후 모든 SELECT: 동일한 Read View 재사용

  타임라인:
    TRX A: START TRANSACTION
    TRX A: SELECT balance WHERE id=1  ← Read View 생성 (TRX B 활성)
           → balance = 10000
    TRX B: UPDATE balance = 7000; COMMIT
    TRX A: SELECT balance WHERE id=1  ← 동일 Read View 재사용
           → balance = 10000 (TRX B는 Read View 생성 시 활성이었음)
    TRX A: COMMIT

READ COMMITTED:
  Read View 생성 시점: 각 SELECT마다 새로운 Read View 생성
  → 항상 최신 커밋된 데이터를 읽음

  같은 타임라인:
    TRX A: START TRANSACTION
    TRX A: SELECT balance WHERE id=1  ← Read View1 생성 (TRX B 활성)
           → balance = 10000
    TRX B: UPDATE balance = 7000; COMMIT
    TRX A: SELECT balance WHERE id=1  ← Read View2 새로 생성 (TRX B 커밋됨)
           → balance = 7000  ← Non-Repeatable Read!
    TRX A: COMMIT

REPEATABLE READ의 특수 케이스 — 자신의 변경은 보임:
  START TRANSACTION;
  SELECT balance WHERE id=1;  -- 10000 (Read View 생성)
  UPDATE accounts SET balance = 8000 WHERE id=1;  -- 내 TRX가 변경
  SELECT balance WHERE id=1;  -- 8000 (내 TRX_ID = m_creator_trx_id → 보임)
  → 자신의 변경은 규칙 1(creator)에 의해 항상 보임
```

### 5. 읽기-쓰기 비차단의 메커니즘

```
Lock 기반 vs MVCC 비교:

Lock 기반:
  SELECT: S Lock 획득 → 동시 UPDATE 차단
  UPDATE: X Lock 획득 → 동시 SELECT 차단
  결과: 읽기가 많을수록 쓰기 대기, 쓰기가 많을수록 읽기 대기

MVCC (일반 SELECT = Consistent Read):
  SELECT 실행:
    Lock 획득 없음
    Read View를 사용해 Undo Log 버전 체인 탐색
    → 다른 어떤 Lock도 필요 없음

  UPDATE 실행:
    X Lock 획득 (Row에)
    새 버전 생성 → Undo Log에 이전 버전 보관
    현재 Row의 TRX_ID와 ROLL_PTR 갱신
    → 동시에 진행 중인 SELECT는 Undo Log의 이전 버전을 읽음

  결과:
    SELECT와 UPDATE가 동시에 실행 가능 → 서로를 차단하지 않음
    SELECT 수가 아무리 많아도 UPDATE를 차단하지 않음

장기 트랜잭션이 Undo Log를 축적하는 이유:
  TRX A가 30분째 실행 중 → Read View가 30분 전 상태 고정
  그동안 발생한 모든 UPDATE의 Undo Log:
    Purge Thread: "TRX A의 Read View가 아직 참조할 수 있어서 삭제 불가"
    → 30분치 Undo Log가 모두 Undo Tablespace에 쌓임
  History List Length가 계속 증가 → Undo Tablespace 팽창

진단:
  SHOW ENGINE INNODB STATUS\G
  → "History list length: N" → 수만 이상이면 위험
  SELECT trx_id, trx_started, TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS sec
  FROM information_schema.INNODB_TRX ORDER BY trx_started;
  → 수십 분 이상 실행 중인 트랜잭션 찾기
```

---

## 💻 실전 실험

### 실험 1: REPEATABLE READ에서 버전 체인 동작 확인

```sql
-- 세션 1 (REPEATABLE READ, 기본값)
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- 10000 확인
-- → Read View 생성됨 (현재 활성 TRX: 세션 2 없음)

-- 세션 2 (별도 창)
UPDATE accounts SET balance = 7000 WHERE id = 1;
COMMIT;
-- → 새 버전(7000) 생성, Undo Log에 이전 버전(10000) 보관

-- 세션 1 (계속)
SELECT balance FROM accounts WHERE id = 1;
-- → 7000이 아닌 10000 반환!
-- → Read View: 세션 2는 이미 커밋됐지만 Read View 생성 당시 활성이었음
-- → 7000 Row의 TRX_ID = 세션 2 ID, Read View에서 이 TRX는 활성
-- → 7000 안 보임 → ROLL_PTR 따라 Undo Log → 10000 반환

-- 세션 2 (다시)
UPDATE accounts SET balance = 5000 WHERE id = 1;
COMMIT;

-- 세션 1 (계속)
SELECT balance FROM accounts WHERE id = 1;
-- → 여전히 10000! (Read View 고정)
-- 버전 체인: 5000(현재) → 7000(Undo) → 10000(Undo) → NULL
-- 세션 1은 10000을 찾을 때까지 체인 탐색

COMMIT;  -- 세션 1 트랜잭션 종료
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;
-- → 5000 (새 트랜잭션 = 새 Read View = 최신 커밋 버전)
```

### 실험 2: 숨겨진 컬럼 간접 확인

```sql
-- DB_TRX_ID 간접 확인 (직접 SELECT는 불가)
START TRANSACTION;

-- 현재 트랜잭션 ID 확인
SELECT trx_id
FROM information_schema.INNODB_TRX
WHERE trx_mysql_thread_id = CONNECTION_ID();
-- 예: 12345

-- 이 트랜잭션이 Row를 변경하면 해당 Row의 DB_TRX_ID = 12345로 설정됨
UPDATE accounts SET balance = 9000 WHERE id = 1;

-- 변경사항 확인
SELECT trx_id, trx_rows_modified, trx_state
FROM information_schema.INNODB_TRX
WHERE trx_mysql_thread_id = CONNECTION_ID();
-- trx_rows_modified: 1 (방금 변경)

ROLLBACK;
-- Undo Log 역방향 적용 → balance 원래값으로 복원
-- DB_TRX_ID도 이전 값으로 복원됨
```

### 실험 3: Read View 생성 시점 비교

```sql
-- REPEATABLE READ vs READ COMMITTED 동작 비교
-- 두 세션 창을 준비

-- === 세션 1A (REPEATABLE READ) ===
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- T1: 10000

-- === 세션 1B (READ COMMITTED) ===
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- T1: 10000

-- === 세션 2: 공통 UPDATE ===
UPDATE accounts SET balance = 7000 WHERE id = 1;
COMMIT;

-- === 세션 1A (REPEATABLE READ) 계속 ===
SELECT balance FROM accounts WHERE id = 1;  -- 10000 (Read View 고정)

-- === 세션 1B (READ COMMITTED) 계속 ===
SELECT balance FROM accounts WHERE id = 1;  -- 7000 (새 Read View)

-- 결과 차이:
-- 1A: 10000 → REPEATABLE READ의 Read View 고정 효과
-- 1B: 7000  → READ COMMITTED의 매번 새 Read View
```

### 실험 4: History List Length 모니터링

```sql
-- 장기 트랜잭션이 Undo Log에 미치는 영향

-- 초기 History List Length 확인
SHOW ENGINE INNODB STATUS\G
-- "History list length: N" 기록

-- 장기 트랜잭션 시작 (세션 1)
START TRANSACTION;
SELECT COUNT(*) FROM orders;  -- Read View 고정

-- 다른 세션들에서 대량 UPDATE 실행 (세션 2)
UPDATE orders SET memo = CONCAT('update', id)
WHERE id BETWEEN 1 AND 100000;
COMMIT;

-- History List Length 다시 확인
SHOW ENGINE INNODB STATUS\G
-- "History list length: 증가" → 세션 1의 Read View 때문에 Purge 차단

-- 장기 트랜잭션 확인
SELECT trx_id, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS seconds_running,
       trx_state
FROM information_schema.INNODB_TRX
ORDER BY trx_started;

-- 세션 1 커밋
COMMIT;

-- History List Length 감소 확인 (Purge Thread 동작 후)
SHOW ENGINE INNODB STATUS\G
```

---

## 📊 성능 비교

```
MVCC vs Lock 기반 동시성:

처리량 비교 (읽기 100건 : 쓰기 1건 혼합):
  Lock 기반:    직렬 실행 → 쓰기 1건이 읽기 100건 완료 대기
  MVCC:         읽기 100건 + 쓰기 1건 동시 실행 가능

버전 체인 탐색 비용:
  체인 길이 1: 추가 비용 거의 없음 (현재 버전이 보임)
  체인 길이 N: N번의 Undo Log 읽기 필요
  
  장기 트랜잭션 존재 시:
    최신 버전 → 수십 개의 Undo Log 탐색 → SELECT 성능 저하
    → 장기 트랜잭션이 다른 세션의 SELECT도 느리게 만드는 이유

REPEATABLE READ vs READ COMMITTED 성능:
  Read View 생성 비용: SELECT마다 활성 TRX 목록 스냅샷
  READ COMMITTED: 모든 SELECT마다 Read View 생성 → 소량 오버헤드
  REPEATABLE READ: 트랜잭션당 1회 생성 → 더 적은 오버헤드
  (단, Gap Lock 차이가 더 큰 성능 요인)
```

---

## ⚖️ 트레이드오프

```
MVCC의 장점과 비용:

장점:
  ✅ 읽기-쓰기 동시성 최대화 (Lock 경합 없음)
  ✅ 읽기 트랜잭션이 오래 실행돼도 다른 트랜잭션에 차단 없음
  ✅ 일관된 읽기 스냅샷 보장 (동일 트랜잭션 내 일관성)

비용:
  ❌ Undo Log 공간: UPDATE마다 이전 버전 보관 → 디스크 공간 증가
  ❌ 버전 체인 탐색: 오래된 Read View가 있으면 SELECT도 느려짐
  ❌ Purge Thread 부하: 오래된 Undo Log 삭제 작업 (백그라운드)
  ❌ Undo Tablespace 팽창: 장기 트랜잭션 = Purge 차단 = 팽창

관리 권장사항:
  트랜잭션을 가능한 짧게 유지 (수초 이내)
  분석 쿼리는 Read Replica로 분리 (Main의 Undo Log 영향 없음)
  innodb_undo_log_truncate = ON (MySQL 8.0 기본) 유지
  History List Length 주기적 모니터링

격리 수준 선택 기준:
  REPEATABLE READ (기본): 일관된 읽기 보장, Gap Lock으로 Phantom 방지
  READ COMMITTED: Phantom Read 허용, Gap Lock 없어 Deadlock 감소
                 쓰기 집약적 + Phantom이 허용되는 서비스에 적합
```

---

## 📌 핵심 정리

```
MVCC 핵심:

Row 숨겨진 컬럼:
  DB_TRX_ID (6 bytes): 마지막 변경 트랜잭션 ID
  DB_ROLL_PTR (7 bytes): 이전 버전 Undo Log 위치 포인터
  → 버전 체인 형성 (현재 → 이전 → 그 이전 → ... → NULL)

Read View 구성:
  m_creator_trx_id: 내 트랜잭션 ID (규칙 1: 항상 보임)
  m_ids: 생성 시점 활성 TRX 집합 (규칙 4: 안 보임)
  m_up_limit_id: m_ids 최솟값 (규칙 2: 미만은 모두 보임)
  m_low_limit_id: 다음 발급 TRX_ID (규칙 3: 이상은 안 보임)

격리 수준 = Read View 생성 시점:
  REPEATABLE READ: 트랜잭션 첫 읽기 시 1회 → 고정 스냅샷
  READ COMMITTED:  각 SELECT마다 새 Read View → 최신 커밋 읽기

읽기-쓰기 비차단:
  SELECT: Lock 없음, 버전 체인 탐색으로 처리
  UPDATE: X Lock + 새 버전 생성 + Undo Log에 이전 버전 보관
  → 두 작업이 물리적으로 다른 데이터를 접근 → 서로 차단 없음

장기 트랜잭션 주의:
  Read View가 오래될수록 Undo Log 삭제 불가
  History List Length 증가 → Undo Tablespace 팽창 → 서비스 전체 영향
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 시나리오에서 각 SELECT의 결과를 예측하라. (REPEATABLE READ)

```sql
-- TRX 100: START TRANSACTION;
-- TRX 200: INSERT INTO t VALUES (1, 'A'); COMMIT;  -- TRX 100 시작 전
-- TRX 100: SELECT * FROM t WHERE id = 1;  -- (1)
-- TRX 300: UPDATE t SET name='B' WHERE id=1; COMMIT;  -- TRX 100 진행 중
-- TRX 100: SELECT * FROM t WHERE id = 1;  -- (2)
-- TRX 400: DELETE FROM t WHERE id=1; COMMIT;         -- TRX 100 진행 중
-- TRX 100: SELECT * FROM t WHERE id = 1;  -- (3)
-- TRX 100: COMMIT
```

<details>
<summary>해설 보기</summary>

TRX 100의 Read View가 생성된 시점을 먼저 파악해야 합니다. REPEATABLE READ에서 첫 번째 SELECT 시 Read View가 생성됩니다.

**Read View 생성 시점**: (1) SELECT 실행 시  
**활성 TRX 당시**: TRX 300, 400은 아직 시작 안 됨 (생성 이후 시작된 미래)  
**m_ids**: {} (빈 집합, TRX 300/400은 아직 없음)  
**m_up_limit_id**: 충분히 큰 값 (TRX 200보다 크거나 같음)  
**m_low_limit_id**: 현재 시점의 다음 TRX_ID (TRX 300보다 작거나 같음은 아님, 300 이전)

실제로 Read View가 생성되는 시점에 TRX 200은 이미 커밋되어 있습니다.

- **(1)**: `(1, 'A')` — TRX 200은 Read View 생성 전 커밋됨 → 보임
- **(2)**: `(1, 'A')` — TRX 300의 변경(B)은 Read View 생성 이후 시작 → 안 보임. Undo Log에서 이전 버전(A) 반환
- **(3)**: `(1, 'A')` — TRX 400의 DELETE도 Read View 이후 → 안 보임. DELETE라도 Undo Log에서 이전 버전(A) 반환. **Row가 삭제됐어도 Read View 기준 이전 버전이 보임** (MVCC의 핵심)

결론: REPEATABLE READ + Consistent Read는 DELETE된 Row도 이전 버전으로 계속 보일 수 있습니다.

</details>

---

**Q2.** 같은 트랜잭션에서 `SELECT balance`로 10000을 읽은 후 `UPDATE accounts SET balance = balance - 3000`을 실행하면 최종 balance는 7000인가 아니면 다른 값일 수 있는가?

<details>
<summary>해설 보기</summary>

**결과는 7000이 아닐 수 있습니다.** 핵심은 **UPDATE는 Current Read(최신 버전)를 사용**한다는 것입니다.

시나리오:
1. TRX A: `SELECT balance → 10000` (Read View 생성, 스냅샷=10000)
2. TRX B: `UPDATE balance = 8000` → COMMIT
3. TRX A: `UPDATE accounts SET balance = balance - 3000`

TRX A의 UPDATE에서:
- `WHERE id = 1` 부분: Current Read (X Lock 획득) → 최신 버전 8000 읽음
- `balance - 3000` 계산: 최신 값 8000 - 3000 = 5000 설정
- **결과: balance = 5000** (10000 - 3000 = 7000이 아님)

이것이 **"Consistent Read와 Current Read 혼용 시 발생하는 잠재적 버그"**입니다.

올바른 패턴: 값을 읽고 계산해서 업데이트할 때는 반드시 `SELECT ... FOR UPDATE`로 Current Read로 통일해야 합니다.
```sql
START TRANSACTION;
SELECT balance FROM accounts WHERE id=1 FOR UPDATE;  -- Current Read: 8000
UPDATE accounts SET balance = 8000 - 3000 WHERE id=1;  -- 5000
COMMIT;
-- 또는 단순히:
UPDATE accounts SET balance = balance - 3000 WHERE id=1;  -- 8000 - 3000 = 5000
```

</details>

---

**Q3.** MVCC가 있음에도 불구하고 `SELECT ... FOR UPDATE`가 존재하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

MVCC는 **읽기 일관성**을 보장하지만 **쓰기 충돌 방지**는 보장하지 않습니다.

시나리오 (MVCC 일반 SELECT):
1. TRX A: `SELECT balance → 10000` (스냅샷 읽기, Lock 없음)
2. TRX B: `SELECT balance → 10000` (스냅샷 읽기, Lock 없음)
3. TRX A: `UPDATE balance = 10000 - 3000 = 7000` → COMMIT
4. TRX B: `UPDATE balance = 10000 - 5000 = 5000` → COMMIT (TRX A 결과 덮어씀!)
   → **Lost Update**: TRX A의 차감(-3000)이 사라짐

`SELECT ... FOR UPDATE` (Current Read + X Lock):
1. TRX A: `SELECT balance FOR UPDATE → 10000` (X Lock 획득)
2. TRX B: `SELECT balance FOR UPDATE` → TRX A의 X Lock 때문에 **대기**
3. TRX A: `UPDATE balance = 7000` → COMMIT (X Lock 해제)
4. TRX B: X Lock 획득, 최신 값 7000 읽음 → `UPDATE balance = 7000 - 5000 = 2000` → COMMIT
   → **올바른 직렬화**: 두 차감 모두 반영

결론: MVCC는 "보는 것의 일관성", `FOR UPDATE`는 "쓰기 충돌 방지". 값을 읽고 그 값을 기반으로 업데이트하는 패턴에서는 반드시 `FOR UPDATE` 필요.

</details>

---

<div align="center">

**[⬅️ ACID 구현](./01-acid-implementation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Undo Log ➡️](./03-undo-log.md)**

</div>
