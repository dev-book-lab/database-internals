# Clustered Index vs Non-Clustered Index

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Clustered Index가 "데이터를 물리적으로 정렬한다"는 말의 정확한 의미는?
- Secondary Index Leaf Node가 실제 Row 위치 대신 PK를 저장하는 이유는?
- `WHERE user_id = 1`을 Secondary Index로 조회할 때 발생하는 Double Lookup이란?
- JPA의 `@GeneratedValue(strategy = IDENTITY)` vs `UUID` PK가 InnoDB 성능에 미치는 차이는?
- `SELECT *`과 `SELECT id, user_id`는 같은 인덱스를 사용해도 왜 성능 차이가 날 수 있는가?
- Clustered Index를 PK가 아닌 다른 컬럼으로 설정할 수 있는가?

---

## 🔍 왜 이 개념이 중요한가

### Secondary Index는 "두 번 읽는다"

```
흔한 기대:
  CREATE INDEX idx_user_id ON orders(user_id);
  SELECT * FROM orders WHERE user_id = 1;
  → 인덱스로 빠르게 찾겠지?

실제 과정:
  ① Secondary Index(idx_user_id) B+Tree 탐색
     → Leaf Node에서 user_id=1인 Row들의 PK 목록 획득
     → 예: PK [42, 108, 513, 1024, ...]
  ② 각 PK로 Clustered Index(PRIMARY) B+Tree 재탐색
     → PK마다 3~4 I/O
     → 100개 PK → 100 × 3 I/O = 300 I/O (Random!)

이것이 "Double Lookup" (이중 탐색)
  → Secondary Index 결과가 많을수록 비용 급증
  → Optimizer가 Full Scan을 선택하는 이유 중 하나

이해의 실용적 가치:
  Double Lookup을 알면:
    Covering Index를 언제 써야 하는지 (Ch2-03)
    왜 SELECT *는 피해야 하는지 (인덱스 커버링 불가)
    JPA N+1 이 왜 Clustered Index 탐색을 N번 하는지
```

---

## 😱 잘못된 이해

### Before: 인덱스가 Row의 물리적 위치를 직접 가리킨다는 오해

```
잘못된 이해:
  Secondary Index Leaf → 디스크의 물리적 위치(파일 오프셋)를 저장
  → idx_user_id 탐색 → "디스크 위치 0x1A3F" → 즉시 데이터 접근

실제:
  Secondary Index Leaf → PK 값을 저장
  → idx_user_id 탐색 → PK=42 → Clustered Index 재탐색 → 데이터

왜 물리적 위치 대신 PK를 저장하는가?
  물리적 위치 방식의 문제:
    Row가 이동하면 (Page Split, OPTIMIZE TABLE, 파티션 이동)
    → 모든 Secondary Index의 포인터를 업데이트해야 함
    → Secondary Index가 5개 → 5번의 포인터 업데이트
    → UPDATE 비용 O(인덱스 수 × Row 이동 수)
  
  PK 방식의 장점:
    Row가 이동해도 PK는 변하지 않음
    → Secondary Index는 그대로
    → Row 이동 = Clustered Index만 수정
    → UPDATE 비용 O(인덱스 수) → 단지 PK 탐색만 추가

  단점:
    Secondary Index 조회마다 +1번의 Clustered Index 탐색 (Double Lookup)
    PK 크기가 클수록 Secondary Index Leaf Node 크기 증가
```

---

## ✨ 올바른 이해

### After: Clustered Index = 데이터 자체가 B+Tree의 Leaf

```
InnoDB Clustered Index의 핵심:
  Clustered Index의 Leaf Node = 실제 Row 데이터
  B+Tree의 Leaf Node가 곧 "테이블" 자체
  
  → 별도의 "데이터 파일"이 없음
  → .ibd 파일 = Clustered Index B+Tree = 데이터

물리적 정렬의 정확한 의미:
  Row들이 PK 순서로 Leaf Page에 저장됨
  "물리적 정렬" = Leaf Page들이 PK 순서로 연결됨 (논리적)
  실제 디스크의 물리적 위치는 아닐 수 있음 (Page Split 후 단편화 발생)
  
  그래도 PK 범위 스캔이 빠른 이유:
    Leaf 연결 리스트가 PK 순서 → 포인터 순회로 범위 스캔
    (물리적 연속성 여부와 무관하게 논리적 순서 보장)

Secondary Index 구조:
  Leaf Node: (인덱스 컬럼 값, PK 값)
  → user_id=1 행들의 PK 목록이 정렬된 형태로 Leaf에 저장
  → Double Lookup: Secondary Index → PK → Clustered Index

테이블당 Clustered Index는 하나:
  InnoDB는 반드시 Clustered Index 필요
  PK → PK를 Clustered Index로 사용
  PK 없음 → 첫 번째 UNIQUE NOT NULL 컬럼
  둘 다 없음 → 숨겨진 DB_ROW_ID(6 bytes) 생성
```

---

## 🔬 내부 동작 원리

### 1. Clustered Index B+Tree 구조 상세

```
orders 테이블 (id PK):

Clustered Index B+Tree:
  
  Root Node (Internal):
    [PK=1000 | PK=5000 | PK=9000]
     │              │              │
     ▼              ▼              ▼
  Internal       Internal       Internal
  [100|300|500]  [1200|1500]   [5500|7000]
     │
     ▼
  Leaf Pages (데이터 포함):
  ┌───────────────────────────────────────┐
  │ PK=1 │ user_id=42, amount=100, ...    │
  │ PK=2 │ user_id=17, amount=250, ...    │
  │ ...  │                                │
  │ PK=99│ user_id=8,  amount=50,  ...    │
  └───────────────────────────────────────┘
       ↕ (FIL_PAGE_NEXT/PREV)
  ┌───────────────────────────────────────┐
  │ PK=100│ user_id=5,  amount=300, ...   │
  │ ...   │                               │
  └───────────────────────────────────────┘

특징:
  Leaf에 전체 Row 저장 → Clustered Index 탐색으로 Row 직접 반환
  PK 순서로 Leaf 연결 → Range Scan = 연결 리스트 순회
```

### 2. Secondary Index B+Tree 구조 상세

```
idx_user_id (user_id 기준 Secondary Index):

Secondary Index B+Tree:
  
  Root Node (Internal):
    [user_id=500 | user_id=1000]
  
  Leaf Pages (인덱스 컬럼 + PK만 저장):
  ┌────────────────────────────┐
  │ user_id=1, PK=42           │  ← 실제 Row 없음, PK만!
  │ user_id=1, PK=108          │
  │ user_id=1, PK=513          │
  │ user_id=2, PK=7            │
  │ user_id=2, PK=89           │
  │ ...                        │
  └────────────────────────────┘
       ↕
  ┌────────────────────────────┐
  │ user_id=3, PK=201          │
  │ ...                        │
  └────────────────────────────┘

Double Lookup 발생:
  WHERE user_id = 1 실행:
  
  Step 1: idx_user_id 탐색
    → Leaf에서 user_id=1인 모든 (user_id, PK) 쌍 수집
    → PK 목록: [42, 108, 513, ...]
  
  Step 2: 각 PK로 Clustered Index 탐색
    → PK=42: Root→Internal→Leaf (3 I/O)
    → PK=108: Root→Internal→Leaf (3 I/O) [다른 Leaf Page!]
    → PK=513: Root→Internal→Leaf (3 I/O) [또 다른 Leaf Page!]
    → ...
  
  user_id=1인 Row가 100개 → 최소 300 I/O (Random I/O!)
```

### 3. MRR (Multi-Range Read) 최적화

```
Double Lookup의 Random I/O 문제를 완화하는 최적화:

MRR (Multi-Range Read):
  Secondary Index에서 PK 목록을 먼저 모두 수집
  → PK를 정렬 (PK 순 = Clustered Index의 Leaf 순서)
  → 정렬된 순서로 Clustered Index 탐색
  
  효과:
    PK 순서 = Clustered Index의 물리적 순서에 가까움
    → Random I/O → Sequential에 가까운 I/O
    → HDD에서 특히 효과적 (탐색 시간 감소)
    → SSD에서도 Prefetch 효과

-- MRR 활성화 확인
SHOW VARIABLES LIKE 'optimizer_switch'\G
-- 출력 중 mrr=on, mrr_cost_based=on 확인

-- EXPLAIN에서 MRR 사용 확인
EXPLAIN SELECT * FROM orders WHERE user_id BETWEEN 1 AND 100\G
-- Extra: Using MRR (MRR 사용 중)

-- MRR을 사용하는 JOIN 최적화 (BKA: Batched Key Access)
-- JOIN에서 Driving Table의 여러 행을 모아 Driven Table을 MRR로 탐색
EXPLAIN SELECT o.* FROM users u JOIN orders o ON o.user_id = u.id
WHERE u.status = 'ACTIVE'\G
-- Extra: Using join buffer (Batched Key Access)
```

### 4. PK 선택이 성능에 미치는 영향

```
PK 크기와 Secondary Index 크기:

Secondary Index Leaf = (인덱스 컬럼, PK)
→ PK가 크면 모든 Secondary Index의 Leaf가 커짐
→ Leaf Page당 저장 가능한 (인덱스 컬럼, PK) 쌍 수 감소
→ Secondary Index B+Tree가 더 커짐 → 더 많은 I/O

PK 크기별 Secondary Index 영향 (idx_user_id 예시):
  PK = BIGINT (8 bytes):
    Leaf 항목: user_id(8) + PK(8) = 16 bytes
    Leaf Page당: 16,384 / 16 ≈ 1,024쌍

  PK = CHAR(36) UUID (36 bytes):
    Leaf 항목: user_id(8) + PK(36) = 44 bytes
    Leaf Page당: 16,384 / 44 ≈ 372쌍
    → 2.7배 더 많은 Secondary Index Page 필요

  Secondary Index가 5개인 테이블에서:
    BIGINT PK: Secondary Index = N pages
    UUID PK:   Secondary Index = N × 2.7 pages × 5 = 13.5배!
    → Buffer Pool에서 Secondary Index가 차지하는 공간 대폭 증가
    → Cache Miss 증가 → 성능 저하

JPA @Id 전략 선택:
  @GeneratedValue(strategy = GenerationType.IDENTITY):
    → AUTO_INCREMENT BIGINT → 최적 PK 선택 ✅
  
  @GeneratedValue(strategy = GenerationType.UUID):
    → UUID (랜덤 36 chars) → 삽입 성능 저하, Secondary Index 비대화 ❌
    → UUID v7(시간 기반) 사용 권장 또는 ULID
  
  직접 구현:
    @Id @GeneratedValue(generator = "uuid7")
    @GenericGenerator(name = "uuid7", type = UUIDv7Generator.class)
    → 단조 증가 + 고유성 + 분산 생성 가능 ✅
```

### 5. Clustered Index와 범위 스캔 성능

```
PK 범위 스캔이 빠른 이유 (Clustered Index의 이점):

  SELECT * FROM orders WHERE id BETWEEN 1000 AND 2000;
  
  Clustered Index 탐색:
    id=1000인 Leaf Page 찾기 (3 I/O)
    → Leaf 연결 리스트로 id=2000까지 순회
    → 결과 Row가 연속된 Leaf Pages에 물리적으로 저장됨
    → Sequential I/O에 가까움
    → Prefetch 효과 (InnoDB Read-Ahead)

  Secondary Index 탐색 (user_id 범위):
    SELECT * FROM orders WHERE user_id BETWEEN 1 AND 10;
    
    idx_user_id 탐색 → PK 목록 수집
    → PK들이 Clustered Index에서 불연속적으로 분산
    → 각 PK → Random I/O
    → 결과가 많으면 Full Scan보다 느릴 수 있음

InnoDB Read-Ahead 최적화:
  Sequential Page 접근 감지 시 자동으로 다음 Page들을 미리 읽음
  두 종류:
    Linear Read-Ahead: 연속된 Extent의 Page를 순서대로 읽으면 다음 Extent 프리페치
    Random Read-Ahead: 같은 Extent의 여러 Page가 Buffer Pool에 있으면 나머지도 프리페치
  
  → Clustered Index Range Scan: Read-Ahead 효과 높음
  → Secondary Index Double Lookup: 각 PK가 다른 Extent → Read-Ahead 효과 낮음
```

---

## 💻 실전 실험

### 실험 1: Double Lookup 비용 측정

```sql
-- Secondary Index 탐색 I/O 측정
-- idx_user_id가 있다고 가정

-- I/O 카운터 리셋 (새 연결에서 측정)
SET @before = (SELECT variable_value FROM performance_schema.global_status
               WHERE variable_name = 'Innodb_pages_read');

-- Secondary Index → Double Lookup 발생
SELECT * FROM orders WHERE user_id = 1;

SET @after = (SELECT variable_value FROM performance_schema.global_status
              WHERE variable_name = 'Innodb_pages_read');
SELECT @after - @before AS pages_read_double_lookup;

-- 비교: Clustered Index (PK) 조회
SET @before = (SELECT variable_value FROM performance_schema.global_status
               WHERE variable_name = 'Innodb_pages_read');

SELECT * FROM orders WHERE id = 42;

SET @after = (SELECT variable_value FROM performance_schema.global_status
              WHERE variable_name = 'Innodb_pages_read');
SELECT @after - @before AS pages_read_pk_lookup;

-- user_id=1인 Row가 많을수록 double lookup 비용 차이 커짐
```

### 실험 2: EXPLAIN으로 Clustered vs Secondary Index 구분

```sql
-- Clustered Index (PK) 조회
EXPLAIN SELECT * FROM orders WHERE id = 42\G
-- type: const (PK 단일 조회 = 최고 효율)
-- key: PRIMARY
-- rows: 1

-- Secondary Index 조회
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- type: ref (Secondary Index 탐색 후 Clustered Index 재탐색)
-- key: idx_user_id
-- rows: 예상 결과 수 (통계 기반)

-- Secondary Index + Clustered Index 경로 시각화
EXPLAIN FORMAT=TREE SELECT * FROM orders WHERE user_id = 1\G
-- 출력에서 "index lookup on orders using idx_user_id" +
--          "lookup join" 또는 "table scan" 확인

-- 결과 건수가 많을 때 Optimizer의 Full Scan 선택 확인
EXPLAIN SELECT * FROM orders WHERE user_id > 0\G
-- type: ALL (전체 테이블이므로 Full Scan 선택)
-- key: NULL (인덱스 사용 안 함)
```

### 실험 3: PK 크기와 Secondary Index 크기 비교

```sql
-- BIGINT PK 테이블
CREATE TABLE t_bigint_pk (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    amount  DECIMAL(10,2),
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB;

-- UUID PK 테이블
CREATE TABLE t_uuid_pk (
    id      CHAR(36) PRIMARY KEY,
    user_id BIGINT NOT NULL,
    amount  DECIMAL(10,2),
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB;

-- 동일 데이터 삽입
INSERT INTO t_bigint_pk (user_id, amount)
SELECT FLOOR(RAND()*10000), ROUND(RAND()*1000,2)
FROM information_schema.columns a CROSS JOIN information_schema.columns b
LIMIT 100000;

INSERT INTO t_uuid_pk (id, user_id, amount)
SELECT UUID(), FLOOR(RAND()*10000), ROUND(RAND()*1000,2)
FROM information_schema.columns a CROSS JOIN information_schema.columns b
LIMIT 100000;

-- 인덱스 크기 비교
SELECT
    table_name,
    ROUND(data_length / 1024, 2) AS data_kb,
    ROUND(index_length / 1024, 2) AS index_kb,
    ROUND(index_length / data_length * 100, 1) AS index_data_ratio_pct
FROM information_schema.tables
WHERE table_schema = 'deep_dive'
  AND table_name IN ('t_bigint_pk', 't_uuid_pk');

-- 예상: t_uuid_pk의 index_kb가 t_bigint_pk보다 훨씬 큼
-- 이유: Secondary Index Leaf에 36 bytes UUID PK 저장 vs 8 bytes BIGINT
```

### 실험 4: MRR 효과 확인

```sql
-- MRR ON (기본)
SET optimizer_switch = 'mrr=on,mrr_cost_based=off';
EXPLAIN SELECT * FROM orders WHERE user_id BETWEEN 1 AND 100\G
-- Extra: Using index condition; Using MRR

-- MRR OFF
SET optimizer_switch = 'mrr=off';
EXPLAIN SELECT * FROM orders WHERE user_id BETWEEN 1 AND 100\G
-- Extra: Using index condition (MRR 없음)

-- 실제 실행 시간 비교
SET @start = NOW(6);
SELECT * FROM orders WHERE user_id BETWEEN 1 AND 100;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) AS us_without_mrr;

SET optimizer_switch = 'mrr=on,mrr_cost_based=off';
SET @start = NOW(6);
SELECT * FROM orders WHERE user_id BETWEEN 1 AND 100;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) AS us_with_mrr;

-- 원복
SET optimizer_switch = 'mrr=on,mrr_cost_based=on';
```

---

## 📊 성능 비교

```
Clustered Index vs Secondary Index 조회 비용:

단일 Row 조회 (결과 1건):
  Clustered (PK): 3~4 I/O
  Secondary:      3~4 (Secondary) + 3~4 (Clustered) = 6~8 I/O
  → Secondary가 2배 비용

범위 조회 (결과 N건, N이 큰 경우):
  Clustered Range:  3 + N/page_rows I/O (Sequential에 가까움)
  Secondary Range:  3 + N×3 I/O (Random I/O) — MRR 없이
  Secondary + MRR:  3 + N×3 I/O → 정렬 후 ~ 3 + k I/O (k ≪ N)
  
  → Secondary Index로 전체의 20% 이상 결과: Full Scan이 유리할 수 있음

SELECT * vs SELECT id, user_id (Covering Index):
  SELECT * → Double Lookup 필수 (Clustered까지 접근)
  SELECT id, user_id → idx_user_id가 (user_id, PK) 포함 → Index Only
  → Covering Index는 Double Lookup 완전히 제거 (Ch2-03에서 상세)
```

---

## ⚖️ 트레이드오프

```
Clustered Index 설계:

PK = AUTO_INCREMENT BIGINT:
  ✅ 단조 증가 삽입 → Page Split 최소
  ✅ Secondary Index Leaf 크기 최소 (8 bytes)
  ✅ Double Lookup 비용 최소
  ❌ 예측 가능 (보안 고려)
  ❌ 분산 환경에서 충돌 가능

PK = UUID v4:
  ✅ 예측 불가능
  ✅ 분산 생성 가능
  ❌ 랜덤 삽입 → Page Split 빈발 → 쓰기 성능 저하
  ❌ Secondary Index 비대화

PK = UUID v7 (시간 기반):
  ✅ 예측 불가능 + 분산 생성
  ✅ 단조 증가에 가까움 → Page Split 거의 없음
  ✅ UUID v4보다 Secondary Index 크기는 동일 (36 bytes)
  ❌ 여전히 BIGINT보다 Secondary Index가 2~4배 큼

결론:
  순수 성능: BIGINT AUTO_INCREMENT
  분산 + 보안: UUID v7 또는 ULID (26 chars, base32)
  UUID v4: 성능 고려 시 비권장
```

---

## 📌 핵심 정리

```
Clustered vs Secondary Index 핵심:

Clustered Index:
  InnoDB 테이블 = Clustered Index B+Tree
  Leaf Node = 전체 Row 데이터
  PK 기준 물리적 정렬 (논리적) → PK Range Scan 효율적
  테이블당 1개 (PK → UNIQUE NOT NULL → 숨겨진 ID 순)

Secondary Index:
  Leaf Node = (인덱스 컬럼, PK) — Row 데이터 없음
  PK를 물리적 위치 대신 저장 → Row 이동 시 업데이트 불필요
  조회 시 Double Lookup 발생 (Secondary → Clustered)

Double Lookup 비용:
  결과 1건: 6~8 I/O (2배)
  결과 N건: N × 3 I/O (Random I/O) → N이 클수록 비용 급증
  MRR로 PK 정렬 후 Sequential I/O에 가깝게 최적화 가능

PK 크기의 영향:
  Secondary Index Leaf = (인덱스 컬럼, PK) 크기
  PK가 클수록 모든 Secondary Index가 비대해짐
  BIGINT(8) vs UUID(36) → Secondary Index 2~4배 크기 차이

JPA 연결:
  @GeneratedValue IDENTITY → AUTO_INCREMENT → 최적
  UUID PK → UUID v7 또는 ULID 권장
  Lazy Loading N+1 = N번의 Double Lookup 가능성
```

---

## 🤔 생각해볼 문제

**Q1.** `SELECT user_id, amount FROM orders WHERE user_id = 1`을 실행할 때 EXPLAIN에서 `type: ref, key: idx_user_id`가 표시됐다. 이 경우 Double Lookup이 발생하는가?

<details>
<summary>해설 보기</summary>

`idx_user_id`가 `(user_id)`만 포함한다면 **Double Lookup 발생**합니다. Secondary Index Leaf에는 `(user_id, PK)`가 있는데, SELECT에 `amount` 컬럼이 있어 Clustered Index까지 접근해야 합니다.

만약 `idx_user_id`가 `(user_id, amount)`로 구성되어 있다면, Leaf에 `(user_id, amount, PK)` 형태가 되어 **Covering Index**로 동작하고 Double Lookup이 없어집니다. EXPLAIN의 `Extra: Using index`가 표시됩니다.

이것이 자주 실행되는 쿼리의 SELECT 컬럼을 포함하는 Composite Index를 만드는 이유입니다. `(user_id, amount)` 인덱스는 이 쿼리를 완전히 커버합니다.

</details>

---

**Q2.** InnoDB 테이블에 PK가 없으면 어떤 일이 발생하는가? 그리고 JPA 엔티티에 `@Id`를 선언하지 않으면 어떤 에러가 발생하는가?

<details>
<summary>해설 보기</summary>

**InnoDB 레벨**: PK가 없으면 첫 번째 UNIQUE NOT NULL 컬럼이 Clustered Index가 됩니다. 그것도 없으면 InnoDB가 내부적으로 6 bytes `DB_ROW_ID`를 자동 생성합니다. 이 경우:
- 모든 Row에 6 bytes 추가 공간 낭비
- 애플리케이션에서 Row를 고유하게 참조할 방법 없음
- 다른 테이블에서 FK로 참조 불가능

**JPA 레벨**: `@Id` 없는 엔티티 → `AnnotationException: No identifier specified for entity`로 애플리케이션 시작 실패합니다. JPA 스펙에서 모든 엔티티는 `@Id`가 필수이기 때문입니다. InnoDB의 `DB_ROW_ID`는 JPA가 알 수 없어서 직접 사용할 수도 없습니다.

실무 교훈: PK는 항상 명시적으로 정의하고, JPA 엔티티의 `@Id`는 InnoDB Clustered Index의 PK와 동일한 컬럼이어야 합니다.

</details>

---

**Q3.** `SELECT * FROM orders ORDER BY user_id LIMIT 10`은 `idx_user_id`를 사용하는가? 그리고 `SELECT * FROM orders ORDER BY user_id LIMIT 1000000`은 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

**LIMIT 10의 경우**: `idx_user_id`를 사용합니다. Secondary Index의 Leaf가 `user_id` 기준으로 정렬되어 있으므로, 가장 작은 `user_id` 10건의 PK를 빠르게 찾고 Double Lookup으로 Row를 가져옵니다. `type: index, Extra: Using index` 또는 `type: ref`가 표시됩니다.

**LIMIT 1,000,000의 경우**: Optimizer는 Full Table Scan을 선택할 가능성이 높습니다. 100만 건의 Double Lookup = 300만 I/O(Random)가 Full Scan의 순차 I/O보다 훨씬 비싸기 때문입니다. EXPLAIN에서 `type: ALL`이 나올 수 있습니다.

이것이 페이지네이션에서 `LIMIT offset, size`가 offset이 커질수록 느려지는 이유입니다. `LIMIT 1000000, 10`은 100만 건을 읽고 버리는 것과 유사합니다. 커서 기반 페이지네이션(`WHERE id > last_id LIMIT 10`)을 사용해야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: B-Tree 구조](./01-btree-why-not-binary-tree.md)** | **[홈으로 🏠](../README.md)** | **[다음: Covering Index ➡️](./03-covering-index-index-only-scan.md)**

</div>
