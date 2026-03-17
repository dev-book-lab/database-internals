# 인덱스를 타지 않는 쿼리 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `WHERE YEAR(created_at) = 2024`가 인덱스를 무력화하는 정확한 이유는?
- 암묵적 형변환(`WHERE user_id = '1'`)이 인덱스를 사용 못 하는 원리는?
- `LIKE '%keyword'`와 `LIKE 'keyword%'`의 인덱스 사용 여부 차이는?
- `OR` 조건이 인덱스 활용을 복잡하게 만드는 이유는?
- `NOT IN`, `NOT LIKE`, `!=`는 인덱스를 사용하는가?
- JPA에서 자주 발생하는 인덱스 미사용 패턴은 어떤 것이 있는가?

---

## 🔍 왜 이 개념이 중요한가

### 인덱스 있는데 Full Scan — 실무에서 가장 많이 만나는 성능 문제

```
실제 지원 요청 패턴:
  "created_at 컬럼에 인덱스 있는데 왜 느리죠?"
  
  코드를 보면:
    WHERE DATE_FORMAT(created_at, '%Y') = '2024'
    
  해석:
    created_at 값마다 DATE_FORMAT 함수 실행 → 인덱스 Range Scan 불가
    → Full Scan으로 100만 건 전체 함수 적용 후 필터
    
  수정 후:
    WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
    → Range Scan → 수백 배 성능 향상

이 장에서 다루는 패턴:
  ① 인덱스 컬럼에 함수/연산 적용
  ② 암묵적 형변환
  ③ LIKE 패턴 (앞 와일드카드)
  ④ OR 조건
  ⑤ NULL 관련 조건
  ⑥ 부정 조건 (NOT IN, !=)
  ⑦ JPA 특유 패턴

각각의 내부 원리를 이해하면
EXPLAIN에서 type: ALL을 보는 즉시 원인 진단 가능
```

---

## 🔬 내부 동작 원리

### 1. 인덱스 컬럼에 함수/연산 적용

```sql
-- ❌ 인덱스 무력화 패턴들

-- 패턴 1: 날짜 함수 적용
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15';
SELECT * FROM orders WHERE DATE_FORMAT(created_at, '%Y-%m') = '2024-01';
-- 이유: created_at에 함수를 적용한 결과는 인덱스에 없음
--       인덱스는 created_at 원래 값으로 정렬됨
--       YEAR(created_at) 결과값은 별도로 정렬되어 있지 않음
--       → Range Scan 범위를 계산할 수 없음 → Full Scan

-- 패턴 2: 산술 연산
SELECT * FROM orders WHERE amount * 1.1 > 10000;
SELECT * FROM products WHERE price - discount = 50000;
-- 이유: amount * 1.1의 결과는 인덱스에 없음

-- 패턴 3: 문자열 변환
SELECT * FROM orders WHERE CONCAT(first_name, ' ', last_name) = 'John Doe';
SELECT * FROM orders WHERE LOWER(email) = 'john@example.com';
-- 이유: CONCAT, LOWER 결과는 인덱스에 없음
--       단, ci (case-insensitive) Collation 컬럼은 LOWER 없이도 대소문자 무시

-- ✅ 올바른 변환 패턴

-- 날짜 범위로 변환
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
SELECT * FROM orders WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16';
-- created_at에 함수 없음 → 인덱스 Range Scan 가능

-- 산술 연산 이항 이동
SELECT * FROM orders WHERE amount > 10000 / 1.1;  -- 9090.9...
-- 연산을 상수 쪽으로 이동 → amount에 함수 없음

-- Collation 활용 (대소문자 무시)
-- ALTER TABLE users MODIFY email VARCHAR(100) COLLATE utf8mb4_general_ci;
SELECT * FROM users WHERE email = 'John@Example.com';  -- LOWER 없이도 동작
```

### 2. 암묵적 형변환 (Implicit Type Conversion)

```sql
-- 테이블: user_id는 BIGINT(INT)
-- ❌ 문자열로 비교 → 형변환 발생
SELECT * FROM orders WHERE user_id = '12345';
-- 동작: MySQL이 '12345'를 숫자로 변환 시도
-- 실제: user_id 컬럼을 VARCHAR로 변환하여 비교
-- → 인덱스는 숫자 기준으로 정렬됨 → 형변환 후 인덱스 사용 불가
-- → Full Scan!

-- 반대 방향도 문제
-- 테이블: phone은 VARCHAR
SELECT * FROM users WHERE phone = 01012345678;
-- 숫자 01012345678을 phone VARCHAR로 비교
-- → phone 컬럼을 숫자로 형변환 시도
-- → 01012345678 → '01012345678' (맞을 수도 있지만 불안정)
-- → 인덱스 사용 여부가 불확실

-- ✅ 올바른 패턴: 타입 일치
SELECT * FROM orders WHERE user_id = 12345;     -- BIGINT 리터럴
SELECT * FROM users WHERE phone = '01012345678'; -- VARCHAR 리터럴

-- JPA에서의 형변환 함정:
-- @Column에서 타입이 다르게 매핑된 경우:
-- 엔티티: private String userId; (String)
-- DB: user_id BIGINT
-- → Hibernate가 WHERE user_id = '12345' 생성 → 형변환 발생
-- → 해결: @Column 타입을 DB 타입과 일치시키거나 @Type 어노테이션 사용

-- 형변환 감지:
EXPLAIN SELECT * FROM orders WHERE user_id = '12345'\G
-- Extra: Using where (형변환으로 인덱스 활용 제한)
-- 또는 type: ALL (Full Scan)
```

### 3. LIKE 패턴

```sql
-- ✅ 인덱스 사용 가능: 앞 고정, 뒤 와일드카드
SELECT * FROM users WHERE name LIKE 'John%';
-- 이유: 'John'으로 시작하는 값들은 인덱스에서 연속 범위
-- B+Tree에서 'John' 이상 'JohO' 미만으로 Range Scan 가능

-- ❌ 인덱스 사용 불가: 앞 와일드카드
SELECT * FROM users WHERE name LIKE '%ohn';
SELECT * FROM orders WHERE description LIKE '%배송%';
-- 이유: 'ohn'으로 끝나는 값들은 인덱스에서 불연속적으로 산재
-- B+Tree를 사전순으로 탐색해도 'ohn'이 끝에 있는 값들의 범위를 특정할 수 없음
-- → Full Scan

-- ❌ 인덱스 사용 불가: 양쪽 와일드카드
SELECT * FROM users WHERE name LIKE '%oh%';
-- 동일한 이유 → Full Scan

-- ✅ Full-Text Index로 해결 (LIKE '%keyword%' 대안)
ALTER TABLE products ADD FULLTEXT INDEX ft_description(description);
SELECT * FROM products WHERE MATCH(description) AGAINST('배송' IN BOOLEAN MODE);
-- Full-Text Search: 토큰 기반 역인덱스 → LIKE '%keyword%' 보다 효율적

-- ✅ Elasticsearch 연동 (대규모 텍스트 검색)
-- 실시간성이 허용되면 Elasticsearch에서 검색 후 PK → MySQL에서 조회

-- LIKE 최적화 실험:
EXPLAIN SELECT * FROM users WHERE name LIKE 'A%'\G    -- type: range ← 인덱스 사용
EXPLAIN SELECT * FROM users WHERE name LIKE '%A'\G    -- type: ALL ← Full Scan
EXPLAIN SELECT * FROM users WHERE name LIKE '%A%'\G   -- type: ALL ← Full Scan
```

### 4. OR 조건

```sql
-- OR 조건과 인덱스:

-- 같은 컬럼 OR → Range Scan 가능
SELECT * FROM orders WHERE user_id = 1 OR user_id = 2;
-- → WHERE user_id IN (1, 2)와 동일 처리
-- → type: range, 인덱스 사용 가능

-- 다른 컬럼 OR → Index Merge 또는 Full Scan
SELECT * FROM orders WHERE user_id = 1 OR status = 'PAID';
-- 인덱스: idx_user_id, idx_status 둘 다 있을 때:
--   Index Merge Union: 두 인덱스를 각각 스캔 후 결과 합집합
--   → EXPLAIN type: index_merge, Extra: Using union(idx_user_id, idx_status)
-- 
-- 인덱스 중 하나만 있거나 Optimizer가 비효율적이라 판단하면:
--   Full Scan

-- Index Merge가 비효율적인 경우:
-- 각 인덱스 결과 건수가 많으면 합집합 처리 비용이 큼
-- → Optimizer가 Full Scan 선택

-- OR 조건 최적화:
-- 방법 1: UNION으로 분리 (각각 인덱스 활용)
SELECT * FROM orders WHERE user_id = 1
UNION ALL
SELECT * FROM orders WHERE status = 'PAID' AND user_id != 1;
-- 두 쿼리 각각 인덱스 사용 → 더 효율적일 수 있음

-- 방법 2: 복합 쿼리를 애플리케이션에서 처리
-- 각 조건으로 쿼리 → 결과를 애플리케이션에서 병합

-- OR in JPA:
-- @Query에서 OR 조건:
-- "FROM Order o WHERE o.userId = :userId OR o.status = :status"
-- → EXPLAIN 확인 필수! Index Merge 여부, type 확인
```

### 5. NULL 관련 조건

```sql
-- NULL 비교와 인덱스:

-- IS NULL: 인덱스 사용 가능 (InnoDB는 NULL도 인덱스에 포함)
SELECT * FROM orders WHERE memo IS NULL;
-- EXPLAIN: type: ref, Extra: Using where ← 인덱스 사용

-- IS NOT NULL: 인덱스 사용 가능 (대부분 범위 스캔)
SELECT * FROM orders WHERE memo IS NOT NULL;
-- EXPLAIN: type: range ← 인덱스 Range Scan

-- 단, NULL이 많은 컬럼의 IS NOT NULL은 Full Scan과 비슷
-- NULL이 90%라면 IS NOT NULL 결과 = 10% → 인덱스 유리
-- NULL이 10%라면 IS NOT NULL 결과 = 90% → Full Scan 유리

-- = NULL: 항상 결과 없음 (SQL 표준)
SELECT * FROM orders WHERE memo = NULL;  -- 항상 0건, 인덱스 무의미

-- COALESCE/IFNULL 함수 사용 시 인덱스 무력화:
SELECT * FROM orders WHERE COALESCE(memo, '') = '';
-- ❌ COALESCE 함수 → 인덱스 사용 불가

-- ✅ 올바른 NULL 처리
SELECT * FROM orders WHERE memo IS NULL OR memo = '';
-- → memo IS NULL: 인덱스, memo = '': 인덱스 → Index Merge 또는 각각 Range
```

### 6. 부정 조건 (NOT IN, !=, NOT LIKE)

```sql
-- != (부등호): 인덱스 Range Scan 가능
SELECT * FROM orders WHERE status != 'CANCELLED';
-- EXPLAIN: type: range ← Full Scan이 아닌 Range Scan!
-- 이유: status != 'CANCELLED' = status < 'CANCELLED' OR status > 'CANCELLED'
--       B+Tree에서 이 두 범위를 스캔 가능

-- NOT IN: 인덱스 Range Scan 가능 (소수 값 제외 시)
SELECT * FROM orders WHERE status NOT IN ('CANCELLED', 'REFUNDED');
-- EXPLAIN: type: range ← 제외할 값이 적으면 Range Scan

-- NOT LIKE: 인덱스 사용 가능 (특정 패턴에 따라)
SELECT * FROM users WHERE name NOT LIKE 'A%';
-- = name < 'A' OR name >= 'B' → Range Scan 가능

-- 단, != / NOT IN으로 선택되는 비율이 크면:
-- Optimizer가 Full Scan 선택 (Selectivity 문제)
-- status = 3가지, NOT IN ('CANCELLED') → 2/3 = 67% → Full Scan 가능

-- NOT EXISTS vs NOT IN:
-- NOT IN (서브쿼리): NULL 처리 문제 있음
SELECT * FROM orders WHERE user_id NOT IN (SELECT id FROM blocked_users);
-- blocked_users에 NULL이 있으면 항상 결과 없음! (SQL NULL 논리)

-- ✅ NOT EXISTS 사용 권장:
SELECT o.* FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM blocked_users b WHERE b.id = o.user_id);
-- NULL 안전하고 인덱스 활용 더 효율적

-- 부정 조건과 JPA:
-- Specification.not(): 내부적으로 != 또는 NOT IN 생성
-- EXPLAIN으로 반드시 확인
```

### 7. JPA에서 자주 발생하는 인덱스 미사용 패턴

```java
// 패턴 1: 형변환으로 인덱스 무력화
@Query("SELECT o FROM Order o WHERE o.userId = :userId")
// userId가 Long인데 파라미터가 String으로 바인딩될 때:
orderRepository.findByUserId("12345");  // String으로 전달
// → WHERE user_id = '12345' → 형변환 → 인덱스 무력화

// 패턴 2: JPQL 함수 사용
@Query("SELECT o FROM Order o WHERE FUNCTION('YEAR', o.createdAt) = :year")
// → WHERE YEAR(created_at) = ? → 인덱스 무력화

// ✅ 올바른 패턴:
@Query("SELECT o FROM Order o WHERE o.createdAt >= :start AND o.createdAt < :end")
List<Order> findByCreatedAtBetween(
    @Param("start") LocalDateTime start,
    @Param("end") LocalDateTime end);
// → WHERE created_at >= ? AND created_at < ? → 인덱스 Range Scan

// 패턴 3: Pageable의 정렬 컬럼 불일치
repository.findByUserId(1L, PageRequest.of(0, 10, Sort.by("createdAt").descending()));
// → ORDER BY created_at DESC
// 인덱스 (user_id, created_at DESC)가 있으면 filesort 없음
// 인덱스 (user_id, created_at ASC)만 있으면 ORDER BY created_at DESC → filesort 발생

// 패턴 4: @Where 어노테이션의 함수 사용
@Where(clause = "YEAR(created_at) = 2024")  // ❌
// → 모든 쿼리에 YEAR(created_at) = 2024 조건 추가 → 인덱스 무력화

// 패턴 5: Querydsl의 대소문자 무시 검색
QUser user = QUser.user;
queryFactory.select(user)
    .from(user)
    .where(user.email.lower().eq(email.toLowerCase()))  // ❌
    // → WHERE LOWER(email) = ? → 인덱스 무력화
    // ✅ Collation을 ci로 설정하거나 eq() 직접 사용 (ci collation이면 대소문자 무시)
    .where(user.email.equalsIgnoreCase(email))
    // Querydsl이 내부적으로 처리하는 방식 확인 필요
```

---

## 💻 실전 실험

### 실험 1: 함수 적용 전/후 EXPLAIN 비교

```sql
CREATE INDEX idx_created ON orders(created_at);

-- ❌ 함수 적용 (인덱스 무력화)
EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2024\G
-- type: ALL, key: NULL

-- ✅ 범위 조건으로 변환
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'\G
-- type: range, key: idx_created

-- 성능 차이 측정
SET @start = NOW(6);
SELECT COUNT(*) FROM orders WHERE YEAR(created_at) = 2024;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) AS func_us;

SET @start = NOW(6);
SELECT COUNT(*) FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) AS range_us;
```

### 실험 2: 암묵적 형변환 감지

```sql
-- user_id BIGINT 컬럼
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G       -- ✅ type: ref
EXPLAIN SELECT * FROM orders WHERE user_id = '1'\G     -- type: ?

-- MySQL은 '1' → 1로 변환하여 인덱스를 사용하기도 하지만
-- 더 복잡한 경우에는 Full Scan 유발
-- 예: 문자열에 공백이 있는 경우
EXPLAIN SELECT * FROM orders WHERE user_id = ' 1'\G    -- Full Scan!

-- WARNING 확인 (MySQL 8.0+)
EXPLAIN SELECT * FROM orders WHERE user_id = '1'\G
SHOW WARNINGS;
-- "Incorrect integer value" 또는 형변환 경고가 나올 수 있음
```

### 실험 3: LIKE 패턴 인덱스 사용 여부

```sql
CREATE INDEX idx_name ON users(name);

EXPLAIN SELECT * FROM users WHERE name LIKE 'A%'\G
-- type: range, key: idx_name ← 인덱스 사용

EXPLAIN SELECT * FROM users WHERE name LIKE '%A'\G
-- type: ALL ← Full Scan

EXPLAIN SELECT * FROM users WHERE name LIKE '%A%'\G
-- type: ALL ← Full Scan

-- LIKE 'A%' 최적화 범위 확인
EXPLAIN SELECT * FROM users WHERE name >= 'A' AND name < 'B'\G
-- 위와 동일한 결과 (Optimizer가 내부적으로 이렇게 변환)
```

### 실험 4: OR 조건과 Index Merge

```sql
CREATE INDEX idx_user ON orders(user_id);
CREATE INDEX idx_status ON orders(status);

-- 같은 컬럼 OR
EXPLAIN SELECT * FROM orders WHERE user_id = 1 OR user_id = 2\G
-- type: range, key: idx_user

-- 다른 컬럼 OR (Index Merge 시도)
EXPLAIN SELECT * FROM orders WHERE user_id = 1 OR status = 'PAID'\G
-- type: index_merge 또는 ALL (데이터에 따라 다름)
-- Extra: Using union(idx_user,idx_status) 또는 Using where

-- Index Merge 비용 확인
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE user_id = 1 OR status = 'PAID'\G
-- "cost_info" 섹션에서 각 접근 방법의 비용 비교

-- UNION으로 분리하는 방법
EXPLAIN
SELECT * FROM orders WHERE user_id = 1
UNION ALL
SELECT * FROM orders WHERE status = 'PAID' AND user_id != 1\G
-- 각 SELECT가 각자의 인덱스를 사용
```

---

## 📊 성능 비교

```
각 패턴별 성능 영향 (100만 건, idx_created on created_at):

YEAR(created_at) = 2024:
  type: ALL → Full Scan
  rows: 1,000,000
  예상 실행 시간: 100~500ms

created_at >= '2024-01-01' AND < '2025-01-01':
  type: range → 인덱스 Range Scan
  rows: 365,000 (2024년 데이터)
  예상 실행 시간: 1~10ms (Buffer Pool 히트 시)

LIKE '%keyword%':
  type: ALL → Full Scan
  rows: 1,000,000
  예상 실행 시간: 100~500ms

LIKE 'keyword%':
  type: range → 인덱스 Range Scan
  rows: 적음 (Selectivity에 따라)
  예상 실행 시간: 0.1~1ms

형변환 (user_id = '12345'):
  MySQL 버전, 데이터에 따라 다름
  안전한 방법: user_id = 12345 (타입 일치)
```

---

## ⚖️ 트레이드오프

```
패턴별 대안과 트레이드오프:

함수 적용 대안:
  애플리케이션 레벨에서 범위 계산 후 쿼리 전달
  트레이드오프: 쿼리가 길어지지만 인덱스 활용

LIKE '%keyword%' 대안:
  Full-Text Index: 한국어는 추가 설정 필요 (ngram parser)
  Elasticsearch: 별도 인프라 비용
  트레이드오프: 인프라 복잡도 증가 vs 검색 성능

OR 조건 대안:
  UNION ALL: 쿼리 복잡도 증가 vs 각 인덱스 활용
  Index Merge: Optimizer가 선택하면 자동 (명시적 변경 불필요)

NULL 처리:
  NULL 허용 컬럼의 IS NOT NULL: 인덱스 사용 가능
  COALESCE 대신 IS NULL OR col = ''로 분리
  트레이드오프: 쿼리 가독성 저하

Function-Based Index (MySQL 8.0.13+):
  인덱스 생성 시 함수 지정:
    ALTER TABLE orders ADD INDEX idx_year ((YEAR(created_at)));
    SELECT * FROM orders WHERE YEAR(created_at) = 2024;
    → 함수 기반 인덱스 사용 가능!
  
  트레이드오프:
    인덱스 크기 증가 (함수 결과를 별도 저장)
    쓰기 시 함수 계산 비용
    선택적으로 사용 권장
```

---

## 📌 핵심 정리

```
인덱스를 타지 않는 패턴 핵심:

공통 원인:
  인덱스는 "컬럼의 원래 값"으로 정렬됨
  함수/연산/형변환을 거친 값은 인덱스에 없음
  → Range Scan의 시작/끝 범위를 계산할 수 없음 → Full Scan

패턴별 해결:
  함수 적용: 범위 조건으로 변환 (함수를 상수 쪽으로 이동)
  형변환: 컬럼 타입과 리터럴 타입 일치
  LIKE '%prefix': Full-Text Index 또는 Elasticsearch
  LIKE 'prefix%': 인덱스 Range Scan 가능 (앞 고정)
  OR 다른 컬럼: UNION ALL 또는 Index Merge (EXPLAIN 확인)
  NULL: IS NULL / IS NOT NULL → 인덱스 사용 가능
  부정 조건: != / NOT IN → 인덱스 Range Scan 가능하지만 Selectivity 주의

JPA 주의사항:
  파라미터 타입과 컬럼 타입 일치시키기
  JPQL 함수 대신 범위 조건으로 쿼리 작성
  Pageable 정렬 컬럼이 인덱스와 방향 일치 확인

진단 방법:
  EXPLAIN type: ALL → 인덱스 미사용 원인 파악
  MySQL 8.0: EXPLAIN ANALYZE로 실제 실행 통계 확인
  Slow Query Log → 함수/형변환 패턴 발견
```

---

## 🤔 생각해볼 문제

**Q1.** `WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'`과 `WHERE created_at >= '2024-01-01' AND created_at <= '2024-12-31'`은 동일한 결과를 반환하는가? 인덱스 사용 방식은 같은가?

<details>
<summary>해설 보기</summary>

**결과 동일**: `BETWEEN a AND b`는 내부적으로 `>= a AND <= b`와 동일합니다. 양 끝 경계값 포함입니다.

**인덱스 사용 방식 동일**: MySQL Optimizer는 두 형식을 동일하게 처리합니다. 둘 다 `EXPLAIN type: range`로 인덱스 Range Scan을 사용합니다.

주의할 점: `DATETIME` 컬럼에서 `BETWEEN '2024-01-01' AND '2024-12-31'`은 `'2024-12-31'`이 `'2024-12-31 00:00:00'`으로 해석됩니다. 2024년 12월 31일의 00:00:00 이후 데이터는 포함되지 않습니다.

완전한 2024년 데이터를 가져오려면:
```sql
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
```
이 방식이 `DATETIME` 처리에서 더 명확합니다.

</details>

---

**Q2.** 다음 쿼리에서 인덱스가 사용되는 경우와 그렇지 않은 경우를 구분하고, 이유를 설명하라. (인덱스: idx_amount on amount, idx_user on user_id)

```sql
-- A: WHERE amount + 1000 > 50000
-- B: WHERE amount > 50000 - 1000
-- C: WHERE amount > 49000
-- D: WHERE ABS(amount) > 49000
-- E: WHERE amount != 0 AND user_id = 1
```

<details>
<summary>해설 보기</summary>

- **A** (`amount + 1000 > 50000`): ❌ `amount` 컬럼에 연산 적용 → 인덱스 무력화, Full Scan
- **B** (`amount > 50000 - 1000`): ✅ 상수 계산이 쿼리 실행 전 처리 → `amount > 49000`과 동일 → Range Scan
- **C** (`amount > 49000`): ✅ 컬럼에 직접 비교 → `idx_amount` Range Scan
- **D** (`ABS(amount) > 49000`): ❌ `ABS()` 함수 적용 → 인덱스 무력화, Full Scan
- **E** (`amount != 0 AND user_id = 1`): `idx_user`로 `user_id = 1` Range Scan 가능. `amount != 0`은 추가 필터. Optimizer가 `idx_amount`로 `amount != 0` Range Scan + `user_id=1` 필터를 할 수도 있고, `idx_user`로 `user_id=1` + `amount != 0` 필터를 할 수도 있습니다. 각 Selectivity에 따라 결정.

B와 C의 차이가 핵심입니다. 연산이 컬럼 쪽에 있으면 인덱스 무력화, 상수 쪽에 있으면 영향 없음.

</details>

---

**Q3.** Spring Data JPA의 `findByEmailIgnoreCase(String email)` 메서드는 인덱스를 사용하는가? 사용한다면 어떤 조건에서, 사용하지 않는다면 어떻게 개선하는가?

<details>
<summary>해설 보기</summary>

**사용 여부는 Collation에 달려 있습니다.**

Spring Data JPA의 `IgnoreCase` → Hibernate가 `WHERE LOWER(email) = LOWER(?)`로 SQL 생성. 이 경우 `email` 컬럼에 LOWER 함수가 적용되어 **인덱스 무력화**.

**해결 방법 1 — Collation 변경** (권장):
```sql
ALTER TABLE users MODIFY email VARCHAR(100) COLLATE utf8mb4_unicode_ci;
-- ci = case-insensitive
```
이후 `WHERE email = 'John@Example.com'`은 대소문자 무시로 동작, `LOWER()` 불필요 → 인덱스 사용.

**해결 방법 2 — Function-Based Index** (MySQL 8.0.13+):
```sql
ALTER TABLE users ADD INDEX idx_email_lower ((LOWER(email)));
```
`WHERE LOWER(email) = LOWER(?)`가 `idx_email_lower`를 사용.

**해결 방법 3 — 애플리케이션에서 소문자 변환 후 저장**:
저장 시 `email = email.toLowerCase()`, 조회 시도 소문자로 변환 후 일반 `=` 비교. 인덱스 그대로 활용.

실무에서는 Collation을 ci로 설정하는 방법 1이 가장 간단하고 효과적입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Index Selectivity](./05-index-selectivity-cardinality.md)** | **[홈으로 🏠](../README.md)** | **[다음: 인덱스 설계 전략 ➡️](./07-index-design-strategy.md)**

</div>
