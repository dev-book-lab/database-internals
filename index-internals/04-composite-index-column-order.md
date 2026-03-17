# Composite Index — 컬럼 순서가 왜 중요한가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Leftmost Prefix Rule이 B+Tree 구조에서 어떻게 도출되는가?
- `(a, b, c)` 인덱스에서 `WHERE b = ?`가 인덱스를 타지 않는 구체적 이유는?
- 범위 조건(`>`, `<`, `BETWEEN`, `LIKE 'prefix%'`) 이후 컬럼은 왜 인덱스를 못 타는가?
- Cardinality 순서로 컬럼을 배치하라는 조언이 항상 옳은 것은 아닌 이유는?
- `key_len`으로 Composite Index의 어느 컬럼까지 사용됐는지 어떻게 확인하는가?
- 하나의 Composite Index로 여러 쿼리 패턴을 커버하는 방법은?

---

## 🔍 왜 이 개념이 중요한가

### 잘못된 순서의 Composite Index = 없는 것만 못하다

```
흔한 실수:
  CREATE INDEX idx_search ON orders(status, user_id, created_at);
  
  의도: "status, user_id로 검색 + created_at으로 정렬"
  실제 쿼리:
    WHERE user_id = 1 ORDER BY created_at
    → status 없이 user_id부터 시작 → Leftmost Prefix 불만족
    → idx_search 사용 불가 → Full Scan 또는 다른 인덱스

  같은 인덱스로 커버 가능한 쿼리 패턴:
    WHERE status = 'PAID' → idx_search 사용 ✅
    WHERE status = 'PAID' AND user_id = 1 → ✅
    WHERE status = 'PAID' AND user_id = 1 AND created_at > '2024-01-01' → ✅
    WHERE user_id = 1 → ✅ ← status 없어도 됨? 아니면?
  
  이 규칙을 모르면:
    인덱스를 만들어도 EXPLAIN에서 type: ALL만 보이는 상황
    "분명히 인덱스 걸었는데 왜 안 타지?" 혼란

왜 B+Tree 구조에서 이 규칙이 나오는가:
  B+Tree는 키를 사전 순으로 정렬
  (status, user_id) 복합 키:
    ("PAID", 1), ("PAID", 2), ..., ("PAID", 999)
    ("PENDING", 1), ("PENDING", 2), ...
  
  user_id만으로 탐색하려면?
  → 트리에서 user_id=1이 있는 위치를 알 수 없음
  → status에 따라 (PAID, 1), (PENDING, 1), (CANCELLED, 1) 등 여러 곳에 분산
  → 연속 범위로 읽기 불가능 → Index Range Scan 불가
```

---

## 😱 잘못된 이해

### Before: 컬럼 순서를 카디널리티 기준으로만 결정

```sql
-- 잘못된 관행: "카디널리티 높은 컬럼 먼저"
-- user_id: 고유 값 10만 개 (카디널리티 높음)
-- status: 값이 3가지 (카디널리티 낮음)

CREATE INDEX idx_wrong ON orders(user_id, status);

-- 실제 쿼리 패턴:
SELECT * FROM orders WHERE status = 'PAID' ORDER BY created_at LIMIT 20;
-- user_id 없이 status만 있는 쿼리
-- idx_wrong: (user_id, status) → user_id 없음 → Leftmost Prefix 불만족
-- EXPLAIN: type: ALL (Full Scan!)

-- 카디널리티 순서 조언이 적합한 경우:
-- WHERE user_id = ? AND status = ? 처럼 둘 다 등치 조건일 때만
-- user_id를 먼저 두면 user_id만으로도 인덱스 활용 가능 (Leftmost Prefix)

-- 올바른 접근: 쿼리 패턴 우선, 카디널리티는 보조 기준
CREATE INDEX idx_correct ON orders(status, created_at);
-- WHERE status = 'PAID' ORDER BY created_at → 완벽하게 커버
```

---

## ✨ 올바른 이해

### After: 쿼리 패턴에서 Composite Index 컬럼 순서를 결정

```
Leftmost Prefix Rule (좌측 접두사 규칙):

인덱스 (a, b, c)가 사용 가능한 조건:
  WHERE a = ?                → 사용 가능 (a만 사용)
  WHERE a = ? AND b = ?      → 사용 가능 (a, b 사용)
  WHERE a = ? AND b = ? AND c = ? → 사용 가능 (a, b, c 전부)
  WHERE a = ? AND c = ?      → a만 사용 (b가 중간에 빠졌으므로 c는 불가)
  WHERE b = ?                → 사용 불가 (a부터 시작 안 했으므로)
  WHERE b = ? AND c = ?      → 사용 불가
  WHERE a = ? AND b > ? AND c = ? → a, b까지만 사용 (b 범위 이후 c 불가)

왜 범위 조건 이후 컬럼을 못 타는가:
  인덱스 (status, amount, created_at):
  
  WHERE status = 'PAID' AND amount > 100 AND created_at > '2024-01-01'
  
  B+Tree에서 (PAID, 100, ...) 위치 찾기 → 시작점 OK
  amount > 100인 범위 스캔:
    (PAID, 101, 2024-01-01), (PAID, 101, 2024-02-01), ...
    (PAID, 200, 2023-01-01), (PAID, 200, 2024-12-31), ...
  
  amount > 100 범위 내에서 created_at은 정렬되어 있지 않음!
  → 각 amount 값마다 created_at이 뒤섞여 있음
  → created_at 조건으로 더 이상 범위를 좁힐 수 없음
  → ICP로 created_at 필터는 가능하지만 Range Scan 범위 축소는 불가
```

---

## 🔬 내부 동작 원리

### 1. Composite Index의 정렬 구조

```
인덱스 (status, user_id, created_at) B+Tree Leaf:

┌────────────────────────────────────────────────────┐
│ (CANCELLED, 1,    2024-01-05), PK=89               │
│ (CANCELLED, 1,    2024-03-12), PK=234              │
│ (CANCELLED, 5,    2024-02-01), PK=456              │
│ (PAID,      1,    2024-01-01), PK=12               │  ← status="PAID" 시작
│ (PAID,      1,    2024-02-15), PK=67               │
│ (PAID,      1,    2024-03-10), PK=103              │
│ (PAID,      2,    2024-01-20), PK=45               │
│ (PAID,      2,    2024-04-01), PK=789              │
│ (PAID,      3,    2023-12-01), PK=23               │
│ ...                                                │
│ (PENDING,   1,    2024-01-08), PK=34               │
│ ...                                                │
└────────────────────────────────────────────────────┘

관찰:
  status 기준으로 먼저 정렬됨 (CANCELLED < PAID < PENDING)
  동일 status 내에서 user_id 기준으로 정렬
  동일 (status, user_id) 내에서 created_at 기준으로 정렬

이 구조에서:
  WHERE status = 'PAID':
    → "PAID" 시작 위치 찾기 (B+Tree 탐색)
    → Leaf 연결 리스트로 "PAID" 범위 스캔 가능 ✅

  WHERE user_id = 1:
    → user_id=1이 (CANCELLED, 1), (PAID, 1), (PENDING, 1) 등 여러 곳에 산재
    → 연속된 범위가 없음 → Range Scan 불가 ❌

  WHERE status = 'PAID' AND user_id = 1:
    → (PAID, 1, ...) 시작 위치 찾기
    → Leaf에서 (PAID, 1, ...) 연속 범위 스캔 ✅
    
  WHERE status = 'PAID' AND created_at > '2024-01-01':
    → status='PAID' 범위 스캔 시작
    → 하지만 PAID 내에서 user_id 순으로 정렬됨
    → created_at은 PAID 내에서 user_id마다 리셋됨
    → created_at으로 범위 축소 불가 ❌ (ICP로 필터만 가능)
```

### 2. key_len으로 인덱스 사용 컬럼 수 확인

```sql
-- key_len: 인덱스에서 실제 사용된 바이트 수
-- 각 타입의 크기:
--   INT: 4 bytes / BIGINT: 8 bytes
--   VARCHAR(N) utf8mb4: N×4 + 2 bytes (길이 저장)
--   DATE: 3 bytes / DATETIME: 5 bytes (MySQL 5.6+)
--   NULL 허용 컬럼: +1 byte (NULL 플래그)

CREATE INDEX idx_abc ON orders(status, user_id, created_at);
-- status: VARCHAR(20) utf8mb4 NOT NULL → 20×4 + 2 = 82 bytes
-- user_id: BIGINT NOT NULL → 8 bytes
-- created_at: DATETIME NOT NULL → 5 bytes
-- 전체 key_len = 82 + 8 + 5 = 95 bytes

EXPLAIN SELECT * FROM orders WHERE status = 'PAID'\G
-- key: idx_abc
-- key_len: 82 ← status만 사용됨 (82 = VARCHAR(20) utf8mb4)

EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND user_id = 1\G
-- key_len: 90 ← status(82) + user_id(8)

EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND user_id = 1 AND created_at > '2024-01-01'\G
-- key_len: 95 ← 모든 컬럼 사용

-- 주의: key_len이 전체여도 범위 조건 이후는 Range 축소 불가 (필터만 가능)
EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND created_at > '2024-01-01'\G
-- key_len: 82 ← status만! created_at은 Leftmost Prefix 불만족
-- Extra: Using index condition (ICP로 created_at 필터만)
```

### 3. 범위 조건 이후 컬럼이 왜 인덱스를 못 타는가

```
인덱스 (a, b, c):

WHERE a = 1 AND b > 10 AND c = 5 실행:

B+Tree에서 (1, 10, ...) 위치 찾기
→ (1, 11, 모든_c), (1, 12, 모든_c), ... 순으로 Leaf 스캔

b > 10 범위 내 데이터:
  (1, 11, 3), (1, 11, 7), (1, 11, 99)  ← c가 정렬됨
  (1, 12, 1), (1, 12, 5), (1, 12, 100)  ← 각 b마다 c가 리셋되어 정렬됨
  (1, 13, 2), (1, 13, 5), ...

c = 5 조건:
  (1, 11, 5) → 있음
  (1, 12, 5) → 있음
  (1, 13, 5) → 있음 (건너뛰어야 함)
  
  c=5인 항목이 b=11, 12, 13, ... 마다 흩어져 있음
  연속된 범위로 c=5를 찾을 수 없음
  → c 조건으로 스캔 범위를 줄일 수 없음 → ICP 필터만 가능

결론:
  (a = 등치, b = 등치) → c까지 Range Scan 가능
  (a = 등치, b = 범위) → b까지 Range Scan, c는 ICP 필터만
  (a = 범위) → a까지 Range Scan, b, c는 ICP 필터만

실용적 순서 결정 규칙:
  ① 등치 조건 컬럼 먼저 (WHERE col = ?)
  ② 범위/ORDER BY 컬럼은 마지막
  ③ 복수의 범위 조건이 있으면 중요한 것 하나만 인덱스에 활용
```

### 4. 하나의 인덱스로 여러 쿼리 패턴 커버

```sql
-- 여러 쿼리 패턴을 분석하여 하나의 인덱스로 커버

-- 실제 서비스에서 발생하는 쿼리들:
-- Q1: SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 10
-- Q2: SELECT * FROM orders WHERE user_id = ? AND status = ? ORDER BY created_at DESC
-- Q3: SELECT COUNT(*) FROM orders WHERE user_id = ? AND status = ?
-- Q4: SELECT user_id, COUNT(*) FROM orders WHERE status = ? GROUP BY user_id

-- 분석:
-- Q1, Q2, Q3의 공통점: user_id 등치 조건
-- Q2, Q3: user_id + status 등치 조건
-- Q1, Q2: created_at 정렬 (ORDER BY)
-- Q4: status 등치 조건 (user_id 없음!)

-- Q1~Q3를 커버하는 인덱스:
CREATE INDEX idx_user_status_created ON orders(user_id, status, created_at);
-- Q1: user_id=? → key_len=8 → created_at ORDER BY (a, ?, c 구조 문제 있음!)
-- → user_id만 사용 후 created_at 정렬이 인덱스 순서와 맞지 않을 수 있음

-- 더 나은 접근:
CREATE INDEX idx_user_created ON orders(user_id, created_at DESC);
CREATE INDEX idx_user_status ON orders(user_id, status);
-- Q1: idx_user_created 사용 ✅
-- Q2, Q3: idx_user_status 사용 ✅

-- Q4는 별도 인덱스 필요:
CREATE INDEX idx_status_user ON orders(status, user_id);
-- Q4: status=? GROUP BY user_id → Covering index가 되면 집계도 빠름

-- 최종 인덱스 세트: 3개로 4가지 쿼리 패턴 커버
-- (하나의 인덱스로 모든 패턴을 커버하려는 것은 비현실적)
```

### 5. 인덱스 순서와 ORDER BY

```sql
-- ORDER BY가 인덱스 순서와 일치해야 filesort를 피할 수 있음

-- 인덱스: (user_id, created_at DESC)
CREATE INDEX idx_user_created_desc ON orders(user_id, created_at DESC);

-- ORDER BY 일치 → filesort 없음
SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC LIMIT 10;
-- EXPLAIN Extra: (없음) ← filesort 없음, 인덱스 순서로 정렬 완료

-- ORDER BY 불일치 → filesort 발생
SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at ASC LIMIT 10;
-- EXPLAIN Extra: Using filesort ← 역방향이라 정렬 필요

-- 복합 ORDER BY
-- 인덱스: (user_id, status, created_at)
CREATE INDEX idx_user_status_created ON orders(user_id, status, created_at);

-- 모두 ASC → filesort 없음
SELECT * FROM orders WHERE user_id = 1 ORDER BY status ASC, created_at ASC LIMIT 10;

-- 혼합 방향 → filesort 발생
SELECT * FROM orders WHERE user_id = 1 ORDER BY status ASC, created_at DESC LIMIT 10;
-- EXPLAIN Extra: Using filesort ← status ASC, created_at DESC 혼합

-- MySQL 8.0: 인덱스에 방향 지정 가능
CREATE INDEX idx_mixed ON orders(user_id, status ASC, created_at DESC);
-- 이제 ORDER BY status ASC, created_at DESC → filesort 없음!
```

---

## 💻 실전 실험

### 실험 1: Leftmost Prefix Rule 확인

```sql
CREATE INDEX idx_s_u_c ON orders(status, user_id, created_at);

-- Prefix 조건별 EXPLAIN
EXPLAIN SELECT * FROM orders WHERE status = 'PAID'\G
-- key: idx_s_u_c, key_len: 82 (status만)

EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND user_id = 1\G
-- key: idx_s_u_c, key_len: 90 (status + user_id)

EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- key: NULL, type: ALL ← Leftmost Prefix 불만족!

EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND created_at > '2024-01-01'\G
-- key: idx_s_u_c, key_len: 82 (status만! created_at은 user_id 건너뜀)
-- Extra: Using index condition (ICP로 created_at 필터)

EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND created_at > '2024-01-01'\G
-- key: NULL, type: ALL ← status부터 시작 안 했으므로
```

### 실험 2: 범위 조건 이후 컬럼 확인

```sql
CREATE INDEX idx_s_a_c ON orders(status, amount, created_at);

-- 등치만: 전체 인덱스 활용
EXPLAIN SELECT * FROM orders 
WHERE status = 'PAID' AND amount = 100 AND created_at = '2024-01-01'\G
-- key_len: 95 (전체)

-- amount 범위: amount까지만, created_at은 ICP
EXPLAIN SELECT * FROM orders 
WHERE status = 'PAID' AND amount > 100 AND created_at > '2024-01-01'\G
-- key_len: 90 (status + amount만)
-- Extra: Using index condition (ICP로 created_at 필터 처리)

-- 효율 비교
SELECT COUNT(*) FROM orders WHERE status = 'PAID' AND amount > 100 AND created_at > '2024-03-01';
-- amount > 100이 결과 범위를 크게 줄이면 빠름
-- created_at > '2024-03-01'은 ICP로 추가 필터 → Clustered 접근 건수 감소
```

### 실험 3: key_len으로 실제 사용 컬럼 진단

```sql
-- key_len 계산 실습
-- 테이블 컬럼 타입 확인
DESCRIBE orders;
-- id: BIGINT (8 bytes)
-- user_id: BIGINT (8 bytes)
-- amount: DECIMAL(10,2) (5 bytes)
-- status: VARCHAR(20) utf8mb4 (= 20×4+2 = 82 bytes, NOT NULL)
-- created_at: DATETIME (5 bytes)

-- key_len 해석
CREATE INDEX idx_test ON orders(status, user_id, amount);

EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND user_id = 1 AND amount > 50\G
-- 예상 key_len: 82 + 8 + 5 = 95 (모두 사용)
-- 실제 amount가 범위 조건이어도 마지막 컬럼이면 사용됨
-- (중간 컬럼이 범위일 때만 그 이후 컬럼이 잘림)

-- NULL 허용 컬럼의 key_len 변화
CREATE INDEX idx_nullable ON orders(status, user_id);
ALTER TABLE orders MODIFY user_id BIGINT NULL;

EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND user_id = 1\G
-- key_len: 82 + 8 + 1 = 91 (NULL 허용 → +1 byte)
```

---

## 📊 성능 비교

```
인덱스 순서 설계 전/후 성능 차이:

쿼리: WHERE status = 'PAID' ORDER BY created_at DESC LIMIT 10

잘못된 인덱스 (user_id, status, created_at):
  → status = 'PAID' 조건: Leftmost Prefix 불만족 (user_id부터 시작해야)
  → Full Scan 또는 다른 인덱스 사용
  → rows: 수십만 (전체 스캔)

올바른 인덱스 (status, created_at):
  → status='PAID' 시작점 → created_at DESC 연결 리스트 순회
  → LIMIT 10 → 10건만 읽고 완료
  → rows: 10 (예상)
  → Extra: Using index condition 또는 Using where

Composite Index vs 여러 단일 인덱스:
  WHERE user_id = 1 AND status = 'PAID':
  
  단일 인덱스 2개 (idx_user_id, idx_status):
    Optimizer가 하나를 선택 + 나머지 필터 (Index Merge 가능하지만 비효율)
  
  Composite Index (user_id, status):
    두 조건을 한 번에 처리 → 범위가 더 좁아짐 → 더 적은 I/O
  
  일반적으로 관련 컬럼은 Composite Index가 효율적
```

---

## ⚖️ 트레이드오프

```
컬럼 순서 선택 기준:

1. 쿼리 패턴 우선 (가장 중요):
   가장 자주 실행되는 쿼리의 WHERE 조건 컬럼 순서대로
   등치 조건 먼저, 범위/ORDER BY 나중

2. Cardinality 보조 기준:
   등치 조건이 여러 개일 때: 카디널리티 높은 것 먼저
   → 초기 B+Tree 탐색에서 더 좁은 범위로 진입
   → 단, 쿼리 패턴과 충돌하면 쿼리 패턴 우선

3. 인덱스 재사용성:
   (a, b, c) 인덱스는 (a), (a, b) 쿼리도 커버
   → 하위 Prefix 쿼리에 재사용 가능
   → 인덱스 수를 줄일 수 있음

4. Covering 목적 컬럼:
   WHERE/ORDER BY 컬럼 다음에 SELECT 컬럼 추가
   → 테이블 접근 제거 가능

Composite Index의 적정 컬럼 수:
  일반적으로 2~4개
  5개 이상: 인덱스 크기 증가, 쓰기 성능 저하
  → 필요한 만큼만, 남용 금지

여러 인덱스 vs Composite Index:
  두 컬럼이 항상 함께 WHERE에 등장: Composite 유리
  각 컬럼이 단독으로도 자주 사용: 단일 인덱스 2개가 유리
  (Composite Index에서 중간 컬럼 생략 시 이후 컬럼 사용 불가)
```

---

## 📌 핵심 정리

```
Composite Index 핵심:

Leftmost Prefix Rule:
  인덱스 (a, b, c)는 a로 시작하는 조건에서만 사용
  (a), (a, b), (a, b, c) — 이 세 가지 패턴만 인덱스 범위 스캔 활용
  (b), (c), (b, c) — 사용 불가

범위 조건 이후 컬럼:
  (a = ?, b > ?, c = ?) → a, b까지 Range Scan, c는 ICP 필터만
  b 범위 내에서 c가 정렬되어 있지 않아 c로 범위 축소 불가

컬럼 순서 결정 원칙:
  ① 등치 조건 컬럼 먼저
  ② 범위 조건 / ORDER BY 컬럼 나중
  ③ 카디널리티는 보조 기준 (등치 조건들 사이에서)

key_len 진단:
  EXPLAIN의 key_len = 사용된 컬럼들의 bytes 합계
  key_len이 기대보다 짧으면 중간 컬럼이 빠진 것

ORDER BY와 인덱스:
  ORDER BY가 인덱스 순서와 일치하면 filesort 없음
  방향(ASC/DESC) 불일치해도 filesort 발생
  MySQL 8.0: 인덱스 컬럼별 방향 지정 가능

개발자가 챙겨야 할 것:
  실제 쿼리 패턴을 EXPLAIN으로 확인
  key_len으로 의도한 컬럼이 모두 사용되는지 검증
  "인덱스 걸었는데 type: ALL" → Leftmost Prefix 위반 확인
```

---

## 🤔 생각해볼 문제

**Q1.** `CREATE INDEX idx ON orders(a, b, c)`가 있을 때, `WHERE a = ? AND c = ?` 쿼리에서 `key_len`이 어떻게 나오는가? `c`에 대한 조건이 인덱스를 활용하는가?

<details>
<summary>해설 보기</summary>

`key_len`은 `a` 컬럼의 크기만 반영됩니다. 예를 들어 a가 BIGINT이면 `key_len: 8`이 나옵니다.

이유: `(a, b, c)` 인덱스에서 b를 건너뛰면 c에 대한 Range Scan을 활용할 수 없습니다. Leaf Node에서 `a=?`인 범위를 찾은 후, 그 범위 내에서 b 값이 다양하게 혼재하고 각 b마다 c가 정렬되므로 c를 연속 범위로 읽을 수 없습니다.

결과: `WHERE a = ? AND c = ?` → a에 대해서만 인덱스 Range Scan, c는 ICP(Index Condition Pushdown)로 인덱스 레벨 필터링. `Extra: Using index condition`이 표시됩니다. c 조건이 없을 때와 비교하면 Clustered Index 접근 건수가 줄어들지만(ICP 덕분에), c로 스캔 범위 자체를 줄이지는 못합니다.

</details>

---

**Q2.** 다음 두 인덱스 중 `WHERE user_id = 1 AND status IN ('PAID', 'PENDING') ORDER BY created_at DESC LIMIT 10` 쿼리에 더 적합한 것은?

```sql
-- 인덱스 A
CREATE INDEX idx_a ON orders(user_id, status, created_at);

-- 인덱스 B
CREATE INDEX idx_b ON orders(user_id, created_at DESC);
```

<details>
<summary>해설 보기</summary>

**인덱스 A**가 이 쿼리에는 더 적합합니다. 단, `IN` 조건이 포함되므로 주의가 필요합니다.

`IN ('PAID', 'PENDING')`은 MySQL Optimizer에 따라 `= 'PAID' OR = 'PENDING'`으로 처리되거나 범위 조건처럼 처리될 수 있습니다. 인덱스 A에서:
- `(user_id=1, status='PAID')` 범위 → `created_at DESC` 순으로 소수 읽기
- `(user_id=1, status='PENDING')` 범위 → `created_at DESC` 순으로 소수 읽기
- 두 결과를 머지 후 LIMIT 10 선택

인덱스 B에서:
- `user_id=1` 범위를 `created_at DESC`로 스캔하면서 status가 PAID 또는 PENDING인 것 필터
- `LIMIT 10`을 빠르게 채울 수 있지만, status 조건에 맞는 Row가 드물면 많은 Leaf를 스캔

**실제로는 데이터 분포에 따라 다릅니다**. EXPLAIN으로 두 인덱스의 `rows` 추정치를 비교하고, `FORCE INDEX`로 강제 후 실제 실행 시간을 측정하는 것이 정확합니다. 상황에 따라 Optimizer가 최적을 선택하므로 인덱스 힌트보다는 올바른 인덱스 설계 후 Optimizer를 신뢰하는 것이 좋습니다.

</details>

---

**Q3.** JPA Querydsl에서 동적 쿼리를 사용하면 WHERE 조건이 실행마다 달라진다. 이런 경우 Composite Index를 어떻게 설계해야 하는가?

<details>
<summary>해설 보기</summary>

동적 쿼리에서 Composite Index 설계의 핵심은 **가장 자주 사용되는 조건 조합**을 분석하는 것입니다.

**접근 방법**:
1. **Slow Query Log 분석**: 실제 운영 데이터에서 자주 실행되는 쿼리 패턴 수집
2. **조건 선택성 분석**: 항상 포함되는 필수 조건(예: `user_id`)을 첫 번째 컬럼으로

**패턴 예시**:
```java
// 동적 조건들:
// - user_id (필수)
// - status (선택)
// - created_at range (선택)
// - amount range (선택)

// 인덱스 설계:
// (user_id, status, created_at) — 가장 흔한 조합 커버
// user_id만: key_len 8 ✅
// user_id + status: key_len 90 ✅
// user_id + status + created_at: key_len 95 ✅
// user_id + created_at (status 없이): key_len 8 (status 건너뜀) → created_at ICP만
```

**대안**: 자주 사용되는 필터 조합이 5개 이상으로 다양하다면 인덱스 여러 개를 상황별로 두거나, MySQL의 `EXPLAIN`을 통해 Optimizer가 자동으로 선택하게 합니다. 극단적으로 동적인 검색이 필요하다면 Elasticsearch 같은 검색 엔진을 고려합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Covering Index](./03-covering-index-index-only-scan.md)** | **[홈으로 🏠](../README.md)** | **[다음: Index Selectivity ➡️](./05-index-selectivity-cardinality.md)**

</div>
