# B-Tree 인덱스 — 왜 Binary Tree가 아닌 B-Tree인가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Binary Search Tree를 DB 인덱스로 쓰면 왜 느린가?
- B-Tree Node가 수백 개의 키를 담는 것이 성능에 어떤 영향을 주는가?
- B+Tree에서 데이터가 Leaf Node에만 존재하는 이유는 무엇인가?
- 범위 스캔(WHERE id BETWEEN 100 AND 200)이 왜 B+Tree에서 효율적인가?
- InnoDB의 B+Tree는 몇 단계(Height)이고, 그 단계당 I/O는 몇 번인가?
- 인덱스를 타는 것과 Full Table Scan 중 Optimizer는 어떤 기준으로 선택하는가?

---

## 🔍 왜 이 개념이 중요한가

### "인덱스를 걸었는데 왜 여전히 느리지?"의 근본 원인

```
인덱스에 대한 흔한 오해:
  "인덱스를 걸면 무조건 빠르다"
  "인덱스는 그냥 정렬된 목록이다"

실제:
  인덱스 = 트리 자료구조
  트리의 구조를 이해해야만:
    왜 특정 쿼리에서 인덱스가 무시되는지
    Composite Index 컬럼 순서가 왜 중요한지
    왜 Covering Index가 빠른지

핵심 연결 관계:
  B+Tree 구조
    └── Node = 1 Page (16KB) → Fan-out 수백 개
          └── 트리 높이 3~4 → 조회당 3~4 I/O
                └── vs Full Scan의 수천 I/O

이것을 이해하면:
  EXPLAIN 결과의 rows, key_len, type이 의미하는 바를
  직관적으로 해석할 수 있다
```

---

## 😱 잘못된 이해

### Before: Binary Search Tree(BST)로 인덱스를 구현한다면

```
BST로 100만 건 인덱스를 만든다면:

이상적 BST (균형 트리):
  높이 = log₂(1,000,000) ≈ 20
  → 탐색 = 20번의 Node 방문

각 Node 방문 = 1번의 디스크 I/O:
  20번 I/O × 8ms(HDD) = 160ms
  → 단일 조회에 160ms → 너무 느림

더 큰 문제 — 균형 유지:
  INSERT/DELETE마다 트리 재균형(Rebalancing)
  → AVL Tree나 Red-Black Tree는 회전 연산 많음
  → 디스크에서 회전 = 여러 Page 수정 = 많은 I/O

최악의 경우 — 불균형:
  순차 삽입(1, 2, 3, 4, ...) → 사실상 연결 리스트
  높이 = N → O(N) 탐색 → Full Scan보다 나쁨

결론:
  BST는 메모리 자료구조에 적합
  디스크 기반 인덱스에는 부적합
```

---

## ✨ 올바른 이해

### After: B+Tree의 설계 목표 — I/O 횟수 최소화

```
B-Tree 설계 목표:
  한 번의 I/O(한 Page = 16KB)로 최대한 많은 정보를 담는다
  → Node 하나에 수백 개의 키 저장
  → 트리 높이를 극도로 낮춤

B+Tree (InnoDB가 실제 사용):
  B-Tree에서 파생
  차이점:
    B-Tree:  Internal Node에도 데이터(Row) 저장
    B+Tree:  Internal Node에는 키만, 데이터는 Leaf Node에만

  B+Tree를 선택한 이유:
    ① Internal Node에 키만 저장 → Node당 더 많은 키 → Fan-out 증가 → 높이 감소
    ② 모든 데이터가 Leaf에 → Leaf 레벨 연결 리스트 → 범위 스캔 효율
    ③ 탐색 경로가 항상 Root → Leaf (균일한 성능)

Fan-out 예시 계산:
  Page 크기: 16KB = 16,384 bytes
  Internal Node의 키 크기: INT(4) + 포인터(6) = 10 bytes
  Fan-out: 16,384 / 10 ≈ 1,638
  
  높이 3인 B+Tree:
    Level 1 (Root):   1 Node × 1,638 포인터
    Level 2:       1,638 Nodes × 1,638 포인터
    Level 3 (Leaf): 1,638² ≈ 268만 개의 Leaf Node
    → 1 Leaf Node에 수십 개 Row 저장
    → 전체 수억 건을 높이 3으로 커버!
```

---

## 🔬 내부 동작 원리

### 1. B+Tree 구조 상세

```
InnoDB B+Tree (Clustered Index 기준):

                    [Root Node]
                   키: 500, 1000
                  /      |       \
         [Internal]  [Internal]  [Internal]
         키: 100,300  키: 600,800  키: 1100,1300
        /   |   \    ...           ...
      [L1] [L2] [L3] ...
      데이터 데이터 데이터 ←→ 연결 리스트 연결

각 Node = 1 Page (16KB):
  Root Node: 최초에 1 Page, 분할되면 Internal Node가 됨
  Internal Node: 키 + 자식 포인터만 저장 (데이터 없음)
  Leaf Node: 키 + 실제 Row 데이터 (Clustered Index) 또는 PK (Secondary Index)

Leaf Node 간 연결:
  각 Leaf Page의 File Header에 FIL_PAGE_PREV / FIL_PAGE_NEXT 포인터
  → 양방향 연결 리스트
  → 범위 스캔: B+Tree로 시작 Leaf 찾기 → 포인터로 순회
  → 추가 트리 탐색 없음 → 효율적

InnoDB가 B+Tree를 사용하는 이유:
  ① Leaf Node에만 데이터 → Internal Node에 더 많은 키 → Fan-out ↑ → 높이 ↓
  ② 범위 스캔이 Leaf 연결 리스트 순회로 처리 → O(k) (k = 결과 수)
  ③ 항상 Root→Leaf 탐색 → 최악/최선 성능이 동일 (예측 가능)
```

### 2. B+Tree 탐색 예시

```
테이블: orders(id PK, user_id, amount, status, created_at)
인덱스: Clustered Index on id (B+Tree)
트리 높이: 3 (일반적인 수백만 건 테이블)

SELECT * FROM orders WHERE id = 42;

① 탐색 시작: Root Node 읽기 (1 I/O)
   Root: [10 | 500 | 1000]
         → id=42는 10~500 사이 → 왼쪽 Internal Node 포인터

② Internal Node 읽기 (1 I/O)
   Internal: [10 | 20 | 30 | 40 | 50]
             → id=42는 40~50 사이 → Leaf Node 포인터

③ Leaf Node 읽기 (1 I/O)
   Leaf: [40, Row데이터] [41, Row데이터] [42, Row데이터] [43, ...]
         → id=42 발견, Row 반환

총 I/O: 3회 (트리 높이만큼)

Buffer Pool에 Root Node가 캐시된 경우:
  Root Node: 항상 Buffer Pool에 있을 가능성 높음 (자주 접근)
  → 실질적 I/O: 1~2회로 감소
  → EXPLAIN의 type: const / eq_ref가 이 상태

범위 탐색: SELECT * FROM orders WHERE id BETWEEN 40 AND 50;

① Root → Internal → Leaf (id=40) 탐색: 3 I/O
② Leaf 연결 리스트로 id=50까지 순회: k I/O (k = 범위 내 Page 수)
  → 추가 트리 탐색 없이 포인터만 따라가면 됨
  → k가 작으면 효율적, k가 크면 많은 I/O
```

### 3. 트리 높이와 I/O 비용

```
실제 트리 높이 추정:

주요 변수:
  Page 크기: 16KB
  키 크기: INT PK = 4 bytes, BIGINT PK = 8 bytes
  Internal Node 포인터: 6 bytes
  Leaf Node Row 크기: 평균 200 bytes (예시)

Fan-out 계산 (BIGINT PK):
  Internal Node: (8 + 6) bytes = 14 bytes/키
  Fan-out = 16,384 / 14 ≈ 1,170

Leaf Node 용량:
  Row 크기 200 bytes → 16,384 / 200 ≈ 81 rows/Leaf

높이별 최대 Row 수:
  높이 1 (Root만): 81 rows (매우 작은 테이블)
  높이 2:          1,170 × 81 ≈ 94,770 rows (10만 건)
  높이 3:          1,170² × 81 ≈ 1.1억 rows (1억 건)
  높이 4:          1,170³ × 81 ≈ 1,300억 rows (1,300억 건)

결론:
  대부분의 실제 테이블 = 높이 3~4
  → SELECT by PK = 3~4번의 I/O (Buffer Pool 미스 시)
  → Buffer Pool 히트 시 0~1번

Full Table Scan과 비교:
  100만 건, 평균 Row 200 bytes:
    Page 수 = 1,000,000 / 81 ≈ 12,346
    Full Scan = 12,346 I/O
  
  Index Scan (결과 1건):
    3 I/O
  
  12,346 / 3 ≈ 4,100배 차이!
```

### 4. 페이지 분할(Page Split)

```
B+Tree INSERT와 페이지 분할:

정상적인 삽입:
  Leaf Node에 빈 공간이 있으면 단순 삽입
  → Page 재정렬만 (포인터로 논리적 순서 유지)

Leaf Node가 가득 찼을 때:
  Page Split 발생:
    ① 기존 Leaf Page를 두 개로 분할 (각 절반의 키)
    ② 새 Leaf Page 할당
    ③ 부모 Internal Node에 새 분기점 키 추가
    ④ 부모가 가득 차면 부모도 분할 (재귀)
    ⑤ Root가 분할되면 새 Root 생성 → 트리 높이 1 증가

Page Split의 비용:
  ① 새 Page 할당 (디스크 I/O)
  ② 기존 Page의 절반 데이터를 새 Page로 이동 (복사 I/O)
  ③ 부모 Node 수정 (잠금, I/O)
  → 삽입 1건에 최대 수 개의 Page I/O 발생

UUID vs AUTO_INCREMENT의 성능 차이:
  AUTO_INCREMENT PK:
    항상 Leaf의 맨 끝에 삽입
    → Page Split이 최소 (마지막 Page가 꽉 찼을 때만)
    → 삽입 성능 최적

  UUID(랜덤) PK:
    랜덤 위치에 삽입
    → 이미 꽉 찬 중간 Page에 삽입 → Page Split 빈발
    → 삽입 성능 저하 + 인덱스 단편화
    → 수백만 건 이상에서 눈에 띄는 성능 차이

  해결책:
    UUID v7 (시간 기반, 단조 증가) 사용
    또는 ULID 사용
    → 랜덤성 + 단조 증가 = UUID의 분산 특성 + AUTO_INCREMENT의 삽입 성능
```

### 5. B+Tree 삭제와 병합

```
DELETE와 B+Tree:

즉시 키 제거하지 않음:
  InnoDB는 삭제된 레코드에 "삭제 표시(Delete Mark)"만 함
  → Page 크기가 즉시 줄지 않음
  → Purge Thread가 나중에 물리적으로 제거

언더플로우(Underflow):
  Node의 키 수가 최소값(Fan-out/2) 미만이 될 때
  → 형제 Node에서 키를 빌려오거나 (Redistribution)
  → 형제 Node와 병합 (Merge)
  → 부모 Node에서 분기점 키 제거 → 재귀적 병합 가능

실제 InnoDB는 병합을 지연:
  삭제 후 바로 병합하지 않고 페이지가 "충분히 비었을 때"만 병합
  → DELETE 집약적 워크로드에서 페이지 단편화 발생
  → OPTIMIZE TABLE로 단편화 제거 (트리 재빌드)
```

---

## 💻 실전 실험

### 실험 1: 트리 높이 확인

```sql
-- InnoDB 내부 통계로 인덱스 높이 추정
SELECT
    s.name AS schema_name,
    t.name AS table_name,
    i.name AS index_name,
    i.PAGE_NO AS root_page_no,
    -- 높이 직접 확인은 innodb_ruby나 information_schema의 통계로만 가능
    s2.stat_value AS n_leaf_pages,
    s3.stat_value AS n_pages_total
FROM information_schema.INNODB_INDEXES i
JOIN information_schema.INNODB_TABLES t ON i.TABLE_ID = t.TABLE_ID
JOIN information_schema.INNODB_TABLESPACES s ON t.SPACE = s.SPACE
LEFT JOIN mysql.innodb_index_stats s2
    ON s2.database_name = SUBSTRING_INDEX(t.name, '/', 1)
    AND s2.table_name = SUBSTRING_INDEX(t.name, '/', -1)
    AND s2.index_name = i.name
    AND s2.stat_name = 'n_leaf_pages'
LEFT JOIN mysql.innodb_index_stats s3
    ON s3.database_name = SUBSTRING_INDEX(t.name, '/', 1)
    AND s3.table_name = SUBSTRING_INDEX(t.name, '/', -1)
    AND s3.index_name = i.name
    AND s3.stat_name = 'size'
WHERE SUBSTRING_INDEX(t.name, '/', 1) = 'deep_dive'
  AND SUBSTRING_INDEX(t.name, '/', -1) = 'orders';

-- n_leaf_pages와 size(전체 페이지 수)로 높이 추정:
-- 높이 ≈ log(size) / log(Fan-out)
-- 또는 (size - n_leaf_pages) / n_leaf_pages 비율로 Internal Node 비율 파악
```

### 실험 2: AUTO_INCREMENT vs UUID 삽입 성능

```sql
-- AUTO_INCREMENT 테이블
CREATE TABLE orders_auto (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    amount     DECIMAL(10,2),
    created_at DATETIME DEFAULT NOW()
) ENGINE=InnoDB;

-- UUID 테이블
CREATE TABLE orders_uuid (
    id         CHAR(36) PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    amount     DECIMAL(10,2),
    created_at DATETIME DEFAULT NOW()
) ENGINE=InnoDB;

-- AUTO_INCREMENT 삽입 시간
SET @start = NOW(6);
INSERT INTO orders_auto (user_id, amount)
SELECT FLOOR(RAND() * 100000), ROUND(RAND() * 1000, 2)
FROM information_schema.columns a
CROSS JOIN information_schema.columns b
LIMIT 100000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000 AS auto_ms;

-- UUID 삽입 시간
SET @start = NOW(6);
INSERT INTO orders_uuid (id, user_id, amount)
SELECT UUID(), FLOOR(RAND() * 100000), ROUND(RAND() * 1000, 2)
FROM information_schema.columns a
CROSS JOIN information_schema.columns b
LIMIT 100000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000 AS uuid_ms;

-- 예상: UUID가 2~5배 느림 (Page Split 빈발)
-- Page Split 횟수 확인:
SHOW STATUS LIKE 'Innodb_page_splits';  -- (실제로는 없는 상태 변수, 간접 측정)
SHOW ENGINE INNODB STATUS\G  -- 출력 중 "page splits" 검색
```

### 실험 3: 범위 스캔의 B+Tree 효율 확인

```sql
-- EXPLAIN으로 인덱스 사용과 범위 스캔 확인
EXPLAIN SELECT * FROM orders WHERE id BETWEEN 1 AND 1000\G
-- type: range ← 범위 스캔 (Leaf 연결 리스트 순회)
-- key: PRIMARY ← Clustered Index 사용
-- rows: ~1000 ← 예상 결과 수
-- Extra: Using index condition (또는 없음)

-- 범위가 커질수록 rows가 증가 → I/O 증가
EXPLAIN SELECT * FROM orders WHERE id BETWEEN 1 AND 100000\G
-- rows: ~100000 → 이제 Full Scan이 더 빠를 수 있는 범위

-- Optimizer의 Full Scan 선택 확인
EXPLAIN SELECT * FROM orders WHERE id > 0\G
-- 전체 테이블이면 type: ALL (Full Scan 선택)
-- 인덱스가 있어도 Optimizer가 Full Scan을 선택한 것
```

---

## 📊 성능 비교

```
BST vs B+Tree 성능 비교 (100만 건, SSD 기준):

                    BST (이론적)     B+Tree (InnoDB)
─────────────────────────────────────────────────────
트리 높이           ~20              3~4
단일 조회 I/O       ~20              3~4
단일 조회 시간      ~2ms             ~0.3ms
범위 스캔 (1000건)  복잡              3 + k I/O
삽입 (균형 유지)    복잡 회전         Page Split (드물게)
디스크 최적화       없음             Page = Node 크기 설계

실제 InnoDB 벤치마크 (100만 건 테이블, Buffer Pool 미스 가정):
  PK 단일 조회:  3~4 I/O × 0.1ms = 0.3~0.4ms
  Full Table Scan: 12,346 I/O (순차) = ~1.2초 (HDD 기준 훨씬 더 큼)
  인덱스 범위 (1000건): 3 + 13 I/O = 16 I/O = ~1.6ms
```

---

## ⚖️ 트레이드오프

```
B+Tree 인덱스의 트레이드오프:

읽기 성능:
  ✅ 단일 조회: O(log_f N) — 매우 빠름 (f = Fan-out)
  ✅ 범위 스캔: O(log_f N + k) — k개 결과에 선형
  ❌ 선택도 낮은 컬럼: Random I/O가 Full Scan보다 느릴 수 있음

쓰기 성능:
  ❌ INSERT: 최악 O(log_f N) Page I/O + Page Split 비용
  ❌ DELETE: Delete Mark + 나중에 Purge (즉시 트리 수정 X)
  ❌ UPDATE on indexed column: DELETE + INSERT와 동일

공간 비용:
  ❌ 인덱스 크기 = 테이블 크기의 10~50%
  ❌ 인덱스가 많을수록 INSERT/UPDATE/DELETE 모두 느려짐

PK 선택:
  AUTO_INCREMENT BIGINT:
    ✅ 단조 증가 → Page Split 최소 → 삽입 최적
    ✅ 작은 크기(8 bytes) → Secondary Index 포인터 오버헤드 작음
    ❌ 예측 가능 (보안 고려 필요)
  
  UUID v4 (랜덤):
    ✅ 예측 불가능 (보안)
    ✅ 분산 생성 가능
    ❌ 랜덤 삽입 → Page Split 빈발 → 쓰기 성능 저하
    ❌ 36 bytes CHAR → Secondary Index 크기 대폭 증가
```

---

## 📌 핵심 정리

```
B+Tree 핵심:

왜 B+Tree인가:
  디스크 I/O 최소화 = 한 Node에 수백 개 키 (Fan-out ↑ = 높이 ↓)
  Page 크기 = Node 크기 → 1 I/O = 1 Node 처리
  높이 3~4 → 수억 건 커버 가능

B-Tree vs B+Tree:
  B-Tree: Internal Node에도 데이터
  B+Tree: Internal Node는 키만 → Fan-out 더 높음
          Leaf 레벨 연결 리스트 → 범위 스캔 효율적
  InnoDB: B+Tree 사용

탐색 비용:
  단일 조회: 높이만큼의 I/O (보통 3~4)
  범위 스캔: 높이 I/O + 결과 Page 수 I/O
  Full Scan 대비: 수백~수천 배 빠름 (선택도 높은 컬럼에서)

Page Split:
  Leaf Node 가득 → 두 개로 분할 → 부모도 수정 필요
  AUTO_INCREMENT: 맨 끝 삽입 → Split 최소
  UUID: 랜덤 삽입 → Split 빈발 → 쓰기 성능 저하

개발자가 챙겨야 할 것:
  PK는 AUTO_INCREMENT BIGINT 또는 UUID v7/ULID 권장
  인덱스 = I/O 최소화 도구, 구조를 이해해야 제대로 활용
  EXPLAIN의 type: range / ref / const는 B+Tree 탐색 방식을 나타냄
```

---

## 🤔 생각해볼 문제

**Q1.** 100만 건 테이블에서 `BIGINT` PK 기준 B+Tree 높이가 3이다. 같은 테이블에 `CHAR(36)` UUID PK를 사용하면 높이가 몇이 되는가? (Fan-out이 달라지는 이유를 계산하라)

<details>
<summary>해설 보기</summary>

**BIGINT PK (8 bytes) 기준:**
- Internal Node: 8(키) + 6(포인터) = 14 bytes → Fan-out = 16,384 / 14 ≈ 1,170
- Leaf Node: Row 200 bytes → 81 rows/page
- 높이 3의 최대 Row: 1,170² × 81 ≈ 1.1억 → 100만 건은 높이 3으로 충분

**CHAR(36) UUID PK (36 bytes, utf8mb4 기준 최대 144 bytes지만 ascii라면 36 bytes) 기준:**
- Internal Node: 36(키) + 6(포인터) = 42 bytes → Fan-out = 16,384 / 42 ≈ 390
- Leaf Node: Row가 키만큼 커짐 → (200 - 8 + 36) = 228 bytes → 71 rows/page
- 높이 2의 최대 Row: 390 × 71 ≈ 27,690 → 100만 건에 부족
- 높이 3의 최대 Row: 390² × 71 ≈ 1,080만 → 100만 건에 충분

결론: 둘 다 높이 3이지만, Fan-out이 1,170 → 390으로 줄었습니다. Secondary Index를 가진 테이블에서 차이가 더 커집니다. Secondary Index는 PK를 포인터로 사용하므로 UUID PK는 Secondary Index의 Leaf Node 크기도 크게 늘립니다.

</details>

---

**Q2.** `WHERE id IN (1, 5, 1000, 50000, 200000)`과 `WHERE id BETWEEN 1 AND 200000`은 모두 인덱스를 사용하지만 I/O 비용이 크게 다르다. 왜인가?

<details>
<summary>해설 보기</summary>

**IN 조건**: 5개의 개별 B+Tree 탐색입니다. 각 값이 트리의 서로 다른 Leaf Page에 있을 가능성이 높으므로 `3 I/O × 5 = 15 I/O` (Buffer Pool 미스 시). 결과가 5건이므로 효율적입니다.

**BETWEEN 조건**: 시작 값(1)의 Leaf Page로 1번 탐색(3 I/O) 후, Leaf 연결 리스트를 따라 끝까지 순회합니다. 1부터 200,000 사이에 수십만 건이 있다면 수백~수천 개의 Leaf Page를 순회합니다 → 수백~수천 I/O.

핵심 차이: `IN`은 "특정 소수의 포인트 탐색", `BETWEEN`은 "연속 범위의 순차 스캔"입니다. 결과 건수가 많을수록 `BETWEEN`의 I/O가 선형으로 증가합니다. Optimizer는 결과 건수 추정치에 따라 인덱스를 버리고 Full Scan을 선택할 수도 있습니다.

</details>

---

**Q3.** B+Tree에서 Internal Node는 데이터를 저장하지 않고 키와 포인터만 저장한다. 그렇다면 `SELECT id FROM orders WHERE id = 42`는 왜 Leaf Node까지 내려가야 하는가? Internal Node에서 id=42를 찾으면 되지 않는가?

<details>
<summary>해설 보기</summary>

좋은 질문입니다. `SELECT id FROM orders WHERE id = 42`는 실제로 Leaf Node까지 내려가야 합니다. 이유는 두 가지입니다:

1. **Clustered Index에서**: 데이터(Row 전체)가 Leaf Node에 있으므로 반드시 Leaf까지 탐색해야 합니다. Internal Node의 키는 경로 안내용이지, 그 키에 해당하는 Row가 거기 있는 게 아닙니다.

2. **존재 확인**: Internal Node에 id=42가 키로 있다면 "42보다 크거나 같은 서브트리"를 가리키는 포인터일 수 있습니다. 실제 id=42 Row가 존재하는지 확인하려면 Leaf까지 가야 합니다.

단, `SELECT id FROM orders WHERE id = 42`에서 `id`가 PK라면 `EXPLAIN`에서 `type: const`로 표시되고 실질적으로 매우 빠릅니다. 이 경우는 Covering Index의 특수 케이스로, PK 값 자체가 Leaf Node에 저장된 Row에 있으므로 Leaf Page 1회 접근으로 완료됩니다. Secondary Index + PK만 SELECT하는 경우에 "Index-Only Scan"이 실질적으로 의미를 갖습니다 (03문서에서 상세 설명).

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Clustered vs Non-Clustered ➡️](./02-clustered-vs-non-clustered.md)**

</div>
