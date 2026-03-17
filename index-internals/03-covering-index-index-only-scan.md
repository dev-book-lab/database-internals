# Covering Index와 Index-Only Scan

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- EXPLAIN의 `Extra: Using index`는 정확히 무엇을 의미하는가?
- Covering Index가 Double Lookup을 완전히 제거하는 원리는?
- Secondary Index Leaf에 PK가 자동으로 포함되는 이유와 그 실용적 의미는?
- Covering Index를 설계할 때 SELECT 컬럼, WHERE 컬럼, ORDER BY 컬럼을 어떻게 조합하는가?
- JPA `@Query`에서 Covering Index를 의도적으로 활용하는 패턴은?
- Covering Index가 항상 좋은 것은 아닌 이유는?

---

## 🔍 왜 이 개념이 중요한가

### "인덱스만으로 쿼리를 완전히 끝낸다"

```
일반 Secondary Index 조회 흐름:
  idx_user_id → PK 목록 → Clustered Index 접근 (Double Lookup)
  
  I/O 비용: N건 × (인덱스 I/O + 테이블 I/O)

Covering Index 조회 흐름:
  idx_user_id_amount → (user_id, amount, PK) 찾기 → 완료!
  테이블 접근 없음!
  
  I/O 비용: 인덱스 I/O만

실제 성능 차이:
  user_id=1인 Row 10,000건 조회
  일반 인덱스: 3 + 10,000 × 3 = 30,003 I/O (Random)
  Covering: 3 + Leaf Pages 순회만 (~10,000 / page_rows I/O)
  → 수십~수백 배 차이

실무 적용:
  API 응답에 필요한 컬럼만 SELECT
  + 그 컬럼들을 모두 포함하는 인덱스 설계
  = Covering Index 전략
  
  목록 조회, 페이지네이션, 집계에서 극적인 성능 향상 가능
```

---

## 😱 잘못된 이해

### Before: SELECT * 남발로 Covering Index 기회 상실

```java
// JPA Repository — 흔한 패턴
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // ❌ 전체 엔티티 로드
    List<Order> findByUserId(Long userId);
    // → SELECT * FROM orders WHERE user_id = ?
    // → idx_user_id가 있어도 Covering Index 불가
    // → Double Lookup 발생: user_id 10,000건 → 10,000 × 3 I/O
    
    // ❌ 필요한 건 상태와 금액뿐인데 전체 로드
    List<Order> findByUserIdAndStatus(Long userId, String status);
    // → SELECT * 로 실행됨
}
```

```sql
-- 잘못된 이해: "인덱스가 있으니 빠르겠지"
CREATE INDEX idx_user_status ON orders(user_id, status);

SELECT * FROM orders WHERE user_id = 1 AND status = 'PENDING';
-- EXPLAIN: type=ref, key=idx_user_status, Extra=(없음)
-- Double Lookup 발생!
-- idx_user_status Leaf: (user_id, status, PK)
-- SELECT * → amount, created_at 등 → Clustered Index 접근 필요
```

---

## ✨ 올바른 이해

### After: 쿼리가 필요한 컬럼을 인덱스가 모두 포함할 때 Covering

```sql
-- Covering Index 설계 원칙:
-- WHERE + ORDER BY + SELECT 컬럼 모두를 인덱스에 포함

-- 목표 쿼리:
-- SELECT user_id, status, amount FROM orders WHERE user_id = 1 ORDER BY created_at;

-- Covering Index 설계:
CREATE INDEX idx_covering ON orders(user_id, created_at, status, amount);
-- 구성: (WHERE의 user_id) + (ORDER BY의 created_at) + (SELECT의 status, amount)
-- Secondary Index Leaf: (user_id, created_at, status, amount, PK)

-- 실행 시 Covering:
SELECT user_id, status, amount FROM orders WHERE user_id = 1 ORDER BY created_at;
-- EXPLAIN: type=ref, key=idx_covering, Extra: Using index ← Covering!
-- Clustered Index 접근 없음 → Double Lookup 0회

-- PK는 자동으로 Secondary Index Leaf에 포함:
SELECT id, user_id, status FROM orders WHERE user_id = 1;
-- idx_user_status Leaf에 PK(id)가 포함되어 있어 이것도 Covering!
-- EXPLAIN: Extra: Using index
```

---

## 🔬 내부 동작 원리

### 1. Secondary Index Leaf에 PK가 자동 포함되는 이유

```
InnoDB Secondary Index Leaf 구조:
  
  CREATE INDEX idx_user_id ON orders(user_id);
  
  Index Leaf Node:
  ┌─────────────────────────────────────┐
  │ user_id=1, PK=42                    │  ← 명시하지 않아도 PK가 포함됨!
  │ user_id=1, PK=108                   │
  │ user_id=1, PK=513                   │
  │ user_id=2, PK=7                     │
  │ ...                                 │
  └─────────────────────────────────────┘

PK가 자동 포함되는 이유:
  Double Lookup을 위해 PK가 필요하기 때문
  Secondary Index → Row를 찾으려면 PK → Clustered Index 재탐색
  
  부수 효과 — PK를 포함하는 쿼리는 자동으로 Covering 가능:
    SELECT id FROM orders WHERE user_id = 1;
    → idx_user_id Leaf에 (user_id, PK=id) 있음
    → Extra: Using index ← PK만 필요하므로 Covering!

명시적으로 PK를 인덱스에 추가하면?
  CREATE INDEX idx_user_id ON orders(user_id, id);
  → user_id 인덱스에 id(PK)를 명시적으로 추가해도 중복
  → InnoDB가 자동으로 제거하거나 내부적으로 동일하게 처리
  → 불필요한 명시 (혼란 야기 가능)

Composite Index에서의 PK 위치:
  CREATE INDEX idx_user_status ON orders(user_id, status);
  Leaf: (user_id, status, PK) — PK는 항상 끝에 추가됨
  
  SELECT id, user_id, status FROM orders WHERE user_id = 1;
  → (user_id, status, PK) 모두 Leaf에 있음 → Covering!
```

### 2. Using index vs Using index condition vs Using where

```
EXPLAIN Extra 필드 해석:

Using index (Index-Only Scan):
  인덱스만으로 쿼리 완전히 처리 — 테이블 접근 없음
  SELECT 컬럼 + WHERE 컬럼이 모두 인덱스에 포함된 경우
  가장 효율적인 상태

Using index condition (Index Condition Pushdown, ICP):
  WHERE 조건 일부를 인덱스 레벨에서 필터링
  나머지는 테이블(Clustered Index) 접근 후 필터링
  
  예시:
    CREATE INDEX idx_user_status ON orders(user_id, status);
    SELECT * FROM orders WHERE user_id = 1 AND status LIKE 'P%';
    
    ICP 전: idx_user_status에서 user_id=1 모두 가져옴
            → Clustered로 이동 → status LIKE 'P%' 필터
    ICP 후: idx_user_status 레벨에서 user_id=1 AND status LIKE 'P%' 필터
            → 조건 통과한 것만 Clustered 접근
    → 불필요한 Clustered Index 접근 감소

Using where:
  인덱스로 Row를 찾은 후 추가적인 WHERE 조건 적용 (MySQL 레벨)
  인덱스가 WHERE 조건 일부만 처리하고 나머지는 MySQL이 필터링

Extra 조합 예시:
  Using index:                    Covering Index ← 최고
  Using index condition:          ICP 사용 (테이블 접근은 함)
  Using where; Using index:       인덱스 스캔 + 추가 필터 (Covering이지만 필터 있음)
  Using filesort:                 인덱스로 정렬 불가, 별도 정렬 필요
  Using temporary; Using filesort: 임시 테이블 + 정렬 (GROUP BY, DISTINCT)
```

### 3. Covering Index 설계 공식

```
목표 쿼리 분석 → 인덱스 컬럼 결정:

공식:
  인덱스 컬럼 = WHERE (등치 조건) + WHERE (범위/정렬 조건) + SELECT (추가 컬럼)
  
  순서:
    ① 등치 조건 컬럼 먼저 (WHERE col = ?)
    ② 범위 조건 또는 ORDER BY 컬럼 다음
    ③ SELECT에서 필요한 나머지 컬럼 마지막에 추가

예시 1: 단순 목록 조회
  쿼리: SELECT user_id, amount FROM orders WHERE status = 'PAID' ORDER BY created_at DESC LIMIT 20;
  
  인덱스:
    (status) → 등치 조건
    + (created_at) → ORDER BY
    + (user_id, amount) → SELECT에서 필요
    = CREATE INDEX idx_cov1 ON orders(status, created_at, user_id, amount);
  
  확인: EXPLAIN Extra: Using index + Using where (필요 시)

예시 2: 집계 쿼리
  쿼리: SELECT user_id, COUNT(*), SUM(amount) FROM orders WHERE status = 'PAID' GROUP BY user_id;
  
  인덱스:
    (status) → WHERE 등치
    + (user_id) → GROUP BY
    + (amount) → SUM에 필요
    = CREATE INDEX idx_cov2 ON orders(status, user_id, amount);
  
  → GROUP BY + SUM이 인덱스만으로 처리됨 (테이블 접근 없음)
  → EXPLAIN: Extra: Using index

예시 3: JPA 페이지네이션
  쿼리: SELECT id, user_id, amount FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 10;
  
  인덱스: (user_id, created_at DESC, amount)
  → Leaf: (user_id, created_at, amount, PK)
  → PK(id)가 자동 포함 → SELECT id도 Covering

  JPA 구현:
    @Query("SELECT new com.example.OrderDto(o.id, o.userId, o.amount) " +
           "FROM Order o WHERE o.userId = :userId ORDER BY o.createdAt DESC")
    List<OrderDto> findSummaryByUserId(@Param("userId") Long userId, Pageable pageable);
    -- DTO Projection + 정렬 컬럼 포함 인덱스 = Covering Index 활용 ✅
```

### 4. Covering Index의 한계

```
Covering Index가 적합하지 않은 경우:

① SELECT * 쿼리:
  모든 컬럼을 인덱스에 포함 = 인덱스 크기 = 테이블 크기
  → 의미 없음 (오히려 인덱스 공간 낭비)
  
② 매우 넓은 인덱스:
  (user_id, status, amount, created_at, description, ...)
  → 인덱스 크기 대폭 증가
  → Buffer Pool에서 인덱스 캐시 효율 감소
  → 쓰기 성능 저하 (모든 INSERT/UPDATE가 넓은 인덱스 갱신)
  
  경험칙: 인덱스 크기가 테이블의 50% 이상이면 재고

③ TEXT/BLOB 컬럼:
  인덱스에 포함 불가 (또는 Prefix만 가능)

④ 자주 변경되는 컬럼:
  Covering Index에 포함된 컬럼이 자주 UPDATE되면
  → 인덱스도 자주 갱신 → 쓰기 성능 저하
  
⑤ 인덱스 키 길이 제한:
  MySQL 기본 최대 키 길이: 3072 bytes (utf8mb4 기준)
  컬럼이 많아지면 제한 초과 가능

Covering Index 적합 쿼리:
  ✅ 자주 실행되는 목록/검색 쿼리
  ✅ 결과 건수가 많은 범위 조회
  ✅ 집계(COUNT, SUM, AVG)
  ✅ ORDER BY + LIMIT (페이지네이션)
  ❌ 랜덤 단건 조회 (이미 PK로 빠름)
  ❌ SELECT * (커버링 불가)
```

---

## 💻 실전 실험

### 실험 1: Covering Index 전/후 EXPLAIN 비교

```sql
-- 실험 준비: 인덱스 없는 상태
DROP INDEX IF EXISTS idx_user_id ON orders;
DROP INDEX IF EXISTS idx_covering ON orders;

-- 기본 인덱스 (커버링 불가)
CREATE INDEX idx_user_id ON orders(user_id);

-- 커버링 불가 쿼리
EXPLAIN SELECT user_id, amount, status FROM orders WHERE user_id = 1\G
-- type: ref
-- key: idx_user_id
-- Extra: (없음 또는 Using index condition) ← 테이블 접근 발생!

-- Covering Index 생성
CREATE INDEX idx_covering ON orders(user_id, amount, status);

-- 동일 쿼리
EXPLAIN SELECT user_id, amount, status FROM orders WHERE user_id = 1\G
-- type: ref
-- key: idx_covering
-- Extra: Using index ← Covering Index! 테이블 접근 없음

-- 실제 I/O 차이 측정
SET @before = (SELECT variable_value FROM performance_schema.global_status
               WHERE variable_name = 'Innodb_pages_read');

SELECT user_id, amount, status FROM orders WHERE user_id = 1;

SET @after = (SELECT variable_value FROM performance_schema.global_status
              WHERE variable_name = 'Innodb_pages_read');
SELECT @after - @before AS pages_covering;
```

### 실험 2: PK가 자동으로 Covering에 포함되는 확인

```sql
CREATE INDEX idx_user_id ON orders(user_id);

-- SELECT에 PK(id) 포함
EXPLAIN SELECT id, user_id FROM orders WHERE user_id = 1\G
-- Extra: Using index ← PK(id)가 idx_user_id Leaf에 자동 포함

-- SELECT에 PK만
EXPLAIN SELECT id FROM orders WHERE user_id = 1\G
-- Extra: Using index ← 당연히 Covering

-- SELECT에 PK 없는 다른 컬럼 추가
EXPLAIN SELECT id, user_id, amount FROM orders WHERE user_id = 1\G
-- Extra: (없음) ← amount가 idx_user_id에 없어서 Covering 불가, Double Lookup 발생
```

### 실험 3: 페이지네이션에서 Covering Index 활용

```sql
-- 비효율적 페이지네이션
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC LIMIT 10 OFFSET 10000\G
-- rows: 높은 값 (10000개 읽고 버림)
-- Extra: Using filesort (정렬 발생 가능)

-- Covering Index 기반 최적화 페이지네이션
CREATE INDEX idx_user_created ON orders(user_id, created_at DESC);

-- Step 1: 인덱스만으로 PK 목록 획득 (Covering)
SELECT id FROM orders
WHERE user_id = 1
ORDER BY created_at DESC
LIMIT 10 OFFSET 10000;
-- EXPLAIN: Extra: Using index ← Covering!

-- Step 2: 획득한 PK로 Clustered Index 단건 조회 (10번만)
SELECT o.* FROM orders o
JOIN (
    SELECT id FROM orders
    WHERE user_id = 1
    ORDER BY created_at DESC
    LIMIT 10 OFFSET 10000
) t ON o.id = t.id;
-- 전체 I/O: 인덱스 페이지 (OFFSET만큼) + 10 × Clustered 접근
-- 일반 방식 대비: OFFSET * Row 크기 대신 OFFSET * Index 크기 (훨씬 작음)
```

### 실험 4: 집계 쿼리에서 Covering Index

```sql
-- Covering Index 없는 집계
EXPLAIN SELECT user_id, COUNT(*), SUM(amount)
FROM orders
WHERE status = 'PAID'
GROUP BY user_id\G
-- Extra: Using where; Using temporary (임시 테이블 사용 가능)

-- Covering Index 생성
CREATE INDEX idx_status_user_amount ON orders(status, user_id, amount);

-- 동일 집계 쿼리
EXPLAIN SELECT user_id, COUNT(*), SUM(amount)
FROM orders
WHERE status = 'PAID'
GROUP BY user_id\G
-- Extra: Using index ← 인덱스만으로 집계 완료!
-- 임시 테이블 없음, 테이블 접근 없음

-- 성능 비교
SET @start = NOW(6);
SELECT user_id, COUNT(*), SUM(amount) FROM orders WHERE status = 'PAID' GROUP BY user_id;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) AS us_covering;
```

---

## 📊 성능 비교

```
Covering vs Non-Covering (100만 건, user_id=1인 Row 10,000건):

Non-Covering (idx_user_id만):
  Secondary Index 탐색: ~10 I/O (Leaf Pages)
  Double Lookup: 10,000 × 3 I/O = 30,000 I/O (Random)
  합계: ~30,010 I/O

Covering (idx_covering with user_id, amount):
  Secondary Index 탐색: ~10 I/O (Leaf Pages)
  Double Lookup: 0
  합계: ~10 I/O

차이: 3,000배!

페이지네이션 최적화 (Covering + 작은 PK 조회):
  일반 OFFSET 10000: 10,000 Row 읽기 → 버리기 (큰 Row 크기 × 10,000)
  Covering OFFSET 10000: 인덱스 항목 × 10,000 (작은 크기) + 최종 10건만 Row 접근
  → 메모리 사용: Row 크기가 줄어도 같음
  → I/O: 인덱스 크기가 Row 크기의 1/5~1/10이면 그만큼 빠름
```

---

## ⚖️ 트레이드오프

```
Covering Index 추가 비용:
  인덱스 크기 증가:
    컬럼 추가마다 Leaf Node 크기 증가
    → Buffer Pool에서 인덱스 캐시 공간 증가
    → 인덱스 B+Tree 높이 증가 가능 (Fan-out 감소)
  
  쓰기 성능 저하:
    INSERT/UPDATE/DELETE 시 넓은 인덱스 갱신
    amount가 업데이트되면 idx_covering도 갱신 필요
  
  설계 원칙:
    자주 실행 + 결과 건수 많음 → Covering 가치 높음
    드물게 실행 + 결과 건수 적음 → Covering 불필요 (Double Lookup 비용 작음)

인덱스 컬럼 수 vs 쓰기 성능:
  인덱스 컬럼 1개: INSERT 비용 +α
  인덱스 컬럼 5개: INSERT 비용 +5α (각 컬럼의 변경 추적 필요)
  
  권장: Covering Index는 자주 실행되는 핵심 쿼리 3~5개에만 적용
        모든 쿼리를 커버하려는 것은 과도한 인덱스로 이어짐

인덱스 vs 더 빠른 하드웨어:
  인덱스 최적화: 소프트웨어 레벨 해결 (비용 없음)
  하드웨어 업그레이드: 비용 발생
  → 인덱스 최적화를 먼저 충분히 시도할 것
```

---

## 📌 핵심 정리

```
Covering Index 핵심:

정의:
  SELECT + WHERE + ORDER BY의 모든 컬럼이 인덱스에 포함
  → Double Lookup 없이 인덱스만으로 쿼리 완결
  EXPLAIN Extra: Using index

PK 자동 포함:
  Secondary Index Leaf = (인덱스 컬럼..., PK)
  → PK(id)를 SELECT해도 Covering 가능
  → 인덱스에 PK를 명시적으로 추가할 필요 없음

설계 공식:
  (WHERE 등치 컬럼) + (ORDER BY / 범위 컬럼) + (SELECT 추가 컬럼)
  순서가 중요 (Leftmost Prefix Rule — Ch2-04에서 상세)

ICP (Index Condition Pushdown):
  Extra: Using index condition
  WHERE 조건을 인덱스 레벨에서 미리 필터링
  → 불필요한 Clustered Index 접근 감소
  → Covering보다는 못하지만 Non-Covering보다는 나음

활용 패턴:
  목록 조회 API: (where 조건, order by, 응답 컬럼) 조합으로 설계
  페이지네이션: Covering으로 PK 목록 → PK로 소수 건만 Row 접근
  집계(COUNT/SUM): Covering으로 테이블 접근 완전 제거

한계:
  SELECT *에는 적용 불가
  인덱스 크기 증가 → 쓰기 성능 저하
  핵심 고빈도 쿼리에만 선별 적용
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 인덱스와 쿼리 조합에서 Covering Index가 되는 것은 어느 것인가?

```sql
CREATE INDEX idx_a ON orders(user_id);
CREATE INDEX idx_b ON orders(user_id, status);
CREATE INDEX idx_c ON orders(user_id, status, amount);

-- 쿼리 1: SELECT id, user_id FROM orders WHERE user_id = 1;
-- 쿼리 2: SELECT user_id, status FROM orders WHERE user_id = 1;
-- 쿼리 3: SELECT user_id, status, amount FROM orders WHERE user_id = 1 AND status = 'PAID';
-- 쿼리 4: SELECT user_id, status, amount, created_at FROM orders WHERE user_id = 1;
```

<details>
<summary>해설 보기</summary>

- **쿼리 1** (`SELECT id, user_id`): `idx_a`로 Covering. `idx_a` Leaf = (user_id, PK=id) → id, user_id 모두 포함 ✅
- **쿼리 2** (`SELECT user_id, status`): `idx_b`로 Covering. `idx_b` Leaf = (user_id, status, PK) → 필요한 컬럼 포함 ✅. `idx_a`로는 status가 없어서 불가.
- **쿼리 3** (`SELECT user_id, status, amount WHERE ... AND status = 'PAID'`): `idx_c`로 Covering. `idx_c` Leaf = (user_id, status, amount, PK) → 모두 포함 ✅. `idx_b`로는 amount가 없어서 Double Lookup 발생.
- **쿼리 4** (`SELECT ..., created_at`): 어느 인덱스로도 Covering 불가. `created_at`이 어떤 인덱스에도 없어서 Double Lookup 필수 ❌.

쿼리 4를 Covering으로 만들려면 `CREATE INDEX idx_d ON orders(user_id, status, amount, created_at);`이 필요합니다.

</details>

---

**Q2.** `EXPLAIN`에서 `Extra: Using index; Using where`를 발견했다. 이것은 Covering Index인가, 아닌가?

<details>
<summary>해설 보기</summary>

**Covering Index입니다.** `Using index`는 인덱스만으로 쿼리를 처리함을 의미하고(테이블 접근 없음), `Using where`는 인덱스에서 가져온 결과에 MySQL 서버 레벨의 추가 필터링이 있음을 의미합니다.

예시:
```sql
CREATE INDEX idx_user_amount ON orders(user_id, amount);
SELECT user_id, amount FROM orders WHERE user_id = 1 AND amount > 100 AND amount < 500;
-- Extra: Using where; Using index
```

- `Using index`: Clustered Index 접근 없음 (Covering)
- `Using where`: `amount > 100 AND amount < 500`을 인덱스에서 필터링

`Using index condition`(ICP)과 혼동하지 마세요. `Using index condition`은 인덱스를 활용한 필터링이지만 **테이블 접근이 발생**합니다. `Using index`는 테이블 접근이 **없는** 경우입니다.

</details>

---

**Q3.** JPA에서 `@EntityGraph`나 `fetch join`을 사용해 연관 엔티티를 함께 로드할 때, Covering Index 전략이 효과적인가?

<details>
<summary>해설 보기</summary>

**제한적으로 효과적입니다.** 연관 엔티티 로드는 두 단계로 나뉩니다:

1. **기본 엔티티 조회**: 여기서 Covering Index를 적용할 수 있습니다.
2. **연관 엔티티 JOIN**: JOIN으로 다른 테이블의 컬럼도 필요하므로 기본 테이블의 Covering Index만으로는 처리 불가.

실용적 패턴:
```java
// ❌ 전체 엔티티 + 연관 모두 로드 → Covering 불가
@EntityGraph(attributePaths = {"orderItems"})
List<Order> findByUserId(Long userId);

// ✅ DTO Projection으로 필요한 컬럼만
@Query("SELECT new com.example.OrderSummaryDto(o.id, o.userId, o.amount, o.status) " +
       "FROM Order o WHERE o.userId = :userId ORDER BY o.createdAt DESC")
List<OrderSummaryDto> findSummaryByUserId(@Param("userId") Long userId);
// → SELECT id, user_id, amount, status FROM orders WHERE user_id = ?
// → (user_id, amount, status) 인덱스가 있으면 Covering
```

`@EntityGraph`나 `fetch join`이 필요한 경우는 연관 테이블이 필요한 경우이므로, Covering Index보다는 적절한 JOIN 인덱스를 설계하는 것이 더 중요합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Clustered vs Non-Clustered](./02-clustered-vs-non-clustered.md)** | **[홈으로 🏠](../README.md)** | **[다음: Composite Index ➡️](./04-composite-index-column-order.md)**

</div>
