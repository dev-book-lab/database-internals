# Join 알고리즘 — Nested Loop, Hash Join, Sort Merge Join

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Nested Loop Join에서 인덱스가 없으면 왜 O(n²)이 되는가?
- Driving Table과 Driven Table을 어떻게 구분하고, Optimizer는 어떤 기준으로 순서를 정하는가?
- Hash Join이 Nested Loop보다 유리한 경우와 불리한 경우는?
- MySQL 8.0.18에서 추가된 Hash Join이 이전 버전(Block Nested Loop)과 다른 점은?
- Sort Merge Join은 MySQL에서 직접 지원하는가?
- JPA Fetch Join이 생성하는 SQL과 그 실행계획은 어떻게 생겼는가?

---

## 🔍 왜 이 개념이 중요한가

### JOIN이 느린 근본 이유를 알아야 올바르게 최적화할 수 있다

```
"JOIN을 쓰면 느리다"는 오해:
  JOIN 자체가 문제가 아니라
  JOIN에 필요한 인덱스가 없거나
  JOIN 순서가 최적이 아닐 때 느려짐

실제 상황:
  users (1만 명) JOIN orders (100만 건) ON users.id = orders.user_id
  WHERE users.grade = 'GOLD'  (GOLD 등급 = 100명)

  Case A: users를 Driving Table로 (올바름)
    100명 → 각각 orders에서 user_id 인덱스로 조회 (평균 100건)
    총: 100 × 100 = 10,000 I/O (인덱스 사용)
    
  Case B: orders를 Driving Table로 (잘못됨)
    100만 건 → 각각 users에서 grade 필터
    총: 100만 × 전체 필터 비용
    
  → Driving Table 선택이 수만 배 성능 차이를 만든다
  → JOIN 알고리즘과 Driving Table을 이해하면 올바른 인덱스를 설계할 수 있다
```

---

## 🔬 내부 동작 원리

### 1. Nested Loop Join (NLJ)

```
기본 Nested Loop Join:

Driving Table (Outer): 한 번만 스캔
Driven Table (Inner): Driving Table의 Row마다 스캔

의사 코드:
  for each row r1 in driving_table:
    for each row r2 in driven_table where r2.join_col = r1.join_col:
      emit(r1, r2)

인덱스가 있는 경우 (Index Nested Loop Join):
  for each row r1 in driving_table:
    r2 = index_lookup(driven_table.join_col, r1.join_col)  ← O(log n)
    emit(r1, r2)
  
  복잡도: O(m × log n)  [m = Driving Table 결과 수]
  → m이 작을수록, driven의 인덱스가 있을수록 효율적

인덱스가 없는 경우 (Simple Nested Loop Join):
  for each row r1 in driving_table:
    for each row r2 in driven_table:  ← Full Scan! O(n)
      if r2.join_col = r1.join_col:
        emit(r1, r2)
  
  복잡도: O(m × n)  ← O(n²)에 가까움
  → 100만 × 1만 = 100억 Row 비교!

MySQL 8.0 이전: Block Nested Loop (BNL)
  메모리 버퍼(join_buffer_size)에 Driving Table 결과를 모아서
  Driven Table을 1번만 Full Scan하며 버퍼와 매칭
  복잡도: O(m × n / buffer_size) → BNL이 없는 것보다 나음
  단, 여전히 Full Scan 발생
```

### 2. Hash Join (MySQL 8.0.18+)

```
Hash Join 동작 원리:

Phase 1: Build Phase (빌드 단계)
  더 작은 테이블(Build Input)의 결과를 메모리 Hash Table에 로드
  
  Hash Table:
    키: JOIN 조건 컬럼 값
    값: Row 데이터
    
  예: users WHERE grade='GOLD' → {user_id: row_data} 해시 테이블
  메모리 요구: join_buffer_size (기본 256KB, 권장 수 MB)

Phase 2: Probe Phase (프로브 단계)
  큰 테이블(Probe Input)을 순차 스캔하며 Hash Table에서 매칭
  
  for each row r2 in probe_table:
    key = r2.join_col
    if key in hash_table:
      emit(hash_table[key], r2)
  
  복잡도: O(m + n)  [m = Build Input 크기, n = Probe Input 크기]
  → NLJ(인덱스 없음)의 O(m × n)보다 압도적으로 빠름

Hash Join이 유리한 경우:
  ① Driven Table에 인덱스가 없는 대용량 JOIN
  ② 두 테이블이 모두 대용량이고 Equijoin (=) 조건
  ③ 분석용 쿼리 (OLAP), 배치 처리

Hash Join이 불리한 경우:
  ① 메모리 부족 → 디스크 Hash Table (급격한 성능 저하)
  ② Non-equijoin (>, <, BETWEEN) → Hash Join 사용 불가
  ③ Driving Table 결과가 매우 작고 인덱스가 있는 경우 → NLJ가 더 빠름
  ④ 스트리밍 결과 필요 (첫 번째 Row를 빨리 반환해야 할 때)
     → Hash Build Phase가 완료되어야 Probe 시작 가능

MySQL 8.0.18+에서 Hash Join 자동 선택:
  Optimizer가 NLJ vs Hash Join 비용 비교 후 선택
  EXPLAIN Extra: Using hash join 또는 Using join buffer (hash join)
```

### 3. Driving Table 선택 기준

```
Optimizer의 Driving Table 선택:

원칙: "WHERE 조건으로 가장 많이 줄어드는 테이블 = Driving Table"
      = 결과 건수가 가장 적은 테이블을 먼저

계산 예시:
  users: 1만 명, grade='GOLD' → 100명 (Driving 후보)
  orders: 100만 건, status='PAID' → 50만 건 (Driving 후보)
  
  Case A: users(100명) → Driving
    100 × idx_lookup(orders, user_id) = 100 × 100건 Double Lookup
    총 비용: ~10,100
  
  Case B: orders(50만건) → Driving
    500,000 × idx_lookup(users, id) = 500,000 × eq_ref(1건)
    총 비용: ~500,000 (훨씬 비쌈)
  
  → Optimizer: Case A 선택 (users를 Driving)

Optimizer가 최적이 아닌 선택을 하는 경우:
  통계 오류: users.grade='GOLD' 결과를 1,000명으로 잘못 추정
  → 100(실제) × 100 = 10,000이 아닌 1,000 × 100 = 100,000으로 계산
  → Case B보다 비싸 보여서 잘못된 순서 선택

STRAIGHT_JOIN으로 순서 강제:
  SELECT STRAIGHT_JOIN ...
  FROM users u  ← 명시적 Driving Table
  JOIN orders o ON o.user_id = u.id
  WHERE u.grade = 'GOLD';

JOIN 순서 힌트 (MySQL 8.0):
  SELECT /*+ JOIN_ORDER(u, o) */ ...
  FROM users u JOIN orders o ON ...
```

### 4. JOIN 최적화 체크리스트

```sql
-- JOINt에서 반드시 확인해야 할 인덱스:
-- Driving Table: WHERE 조건 컬럼에 인덱스
-- Driven Table: JOIN 조건 컬럼(FK)에 인덱스

-- 예시 테이블:
-- users (id PK, name, grade)
-- orders (id PK, user_id FK, amount, status, created_at)
-- order_items (id PK, order_id FK, product_id, quantity)
-- products (id PK, name, price)

-- 쿼리:
SELECT u.name, o.amount, p.name AS product, oi.quantity
FROM users u
JOIN orders o ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE u.grade = 'GOLD' AND o.status = 'PAID';

-- 필요한 인덱스:
-- users: (grade) ← WHERE 조건
-- orders: (user_id, status) ← JOIN + WHERE
-- order_items: (order_id) ← JOIN
-- products: PK(id) ← JOIN (이미 있음)

-- 인덱스 없이 실행 시:
EXPLAIN SELECT ... \G
-- Extra: Using join buffer (hash join) or Block Nested Loop

-- 인덱스 추가 후:
CREATE INDEX idx_grade ON users(grade);
CREATE INDEX idx_user_status ON orders(user_id, status);
CREATE INDEX idx_order_id ON order_items(order_id);

EXPLAIN SELECT ... \G
-- type(orders): ref (user_id 인덱스 사용)
-- type(order_items): ref (order_id 인덱스 사용)
-- Extra: (없음) → NLJ with index 사용
```

### 5. JPA Fetch Join과 실행계획

```java
// JPA 엔티티 관계:
// @Entity Order { @ManyToOne User user; @OneToMany List<OrderItem> items; }

// N+1 문제 (기본 Lazy Loading):
List<Order> orders = orderRepository.findByStatus("PAID");
// SQL: SELECT * FROM orders WHERE status = 'PAID'  → 결과 N건
for (Order o : orders) {
    o.getUser().getName();  // 각 order마다 별도 SQL!
    // SQL: SELECT * FROM users WHERE id = ? (N번!)
}
// 총 N+1번 쿼리

// Fetch Join으로 해결:
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findWithUserByStatus(@Param("status") String status);
// SQL (Hibernate 생성):
// SELECT o.*, u.*
// FROM orders o
// INNER JOIN users u ON o.user_id = u.id
// WHERE o.status = 'PAID'
// EXPLAIN: Join Nested Loop, orders → users (eq_ref)

// OneToMany Fetch Join (주의: 중복 발생):
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findWithItemsByStatus(@Param("status") String status);
// SQL:
// SELECT DISTINCT o.*, oi.*
// FROM orders o
// INNER JOIN order_items oi ON oi.order_id = o.id
// WHERE o.status = 'PAID'
// 주의: orders와 items의 카테시안 곱 발생 가능
//       orders N건 × items M건 = N×M 결과행 → 메모리에서 DISTINCT 처리
// → 대규모에서는 @BatchSize나 별도 쿼리가 더 효율적

// MultipleBagFetchException 방지:
// OneToMany를 2개 이상 Fetch Join 시 예외 발생
// 해결: @BatchSize(size=100) 또는 FetchType.EAGER 대신 별도 쿼리

// EXPLAIN으로 JPA 쿼리 확인:
// spring.jpa.show-sql=true + spring.jpa.properties.hibernate.format_sql=true
// 생성된 SQL을 EXPLAIN으로 분석
```

---

## 💻 실전 실험

### 실험 1: NLJ vs Hash Join 비교

```sql
-- 테스트 셋업
CREATE TABLE jtest_users (
    id    BIGINT AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(50),
    grade CHAR(1)  -- A, B, C
) ENGINE=InnoDB;

CREATE TABLE jtest_orders (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    amount  DECIMAL(10,2)
    -- user_id 인덱스 없음 (의도적)
) ENGINE=InnoDB;

-- 데이터 삽입 (users 1만, orders 100만)

-- 인덱스 없이 JOIN → Hash Join or BNL
EXPLAIN SELECT u.name, COUNT(*)
FROM jtest_users u
JOIN jtest_orders o ON o.user_id = u.id
GROUP BY u.name\G
-- Extra: Using hash join (MySQL 8.0.18+)
--        또는 Using join buffer (Block Nested Loop) (이전 버전)

-- 성능 측정
SET @start = NOW(6);
SELECT u.name, COUNT(*) FROM jtest_users u
JOIN jtest_orders o ON o.user_id = u.id GROUP BY u.name;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS ms_no_index;

-- 인덱스 추가 후 NLJ
CREATE INDEX idx_user_id ON jtest_orders(user_id);

EXPLAIN SELECT u.name, COUNT(*)
FROM jtest_users u
JOIN jtest_orders o ON o.user_id = u.id
GROUP BY u.name\G
-- type(jtest_orders): ref ← NLJ with index

SET @start = NOW(6);
SELECT u.name, COUNT(*) FROM jtest_users u
JOIN jtest_orders o ON o.user_id = u.id GROUP BY u.name;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS ms_with_index;
```

### 실험 2: Driving Table 변경 효과

```sql
-- WHERE 조건으로 결과 크게 줄이는 테이블이 Driving이 효율적

-- Optimizer 선택 확인
EXPLAIN SELECT u.name, o.amount
FROM jtest_users u
JOIN jtest_orders o ON o.user_id = u.id
WHERE u.grade = 'A'\G  -- grade='A' = 1/3 (약 3,333명)
-- Optimizer는 users를 Driving으로 선택할 것

-- STRAIGHT_JOIN으로 반대 순서 강제 (비교용)
EXPLAIN SELECT STRAIGHT_JOIN u.name, o.amount
FROM jtest_orders o  ← Driving (100만 건!)
JOIN jtest_users u ON o.user_id = u.id
WHERE u.grade = 'A'\G

-- 성능 비교
SET @start = NOW(6);
SELECT u.name, o.amount FROM jtest_users u
JOIN jtest_orders o ON o.user_id = u.id WHERE u.grade = 'A' LIMIT 1000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS correct_order_ms;

SET @start = NOW(6);
SELECT STRAIGHT_JOIN u.name, o.amount FROM jtest_orders o
JOIN jtest_users u ON o.user_id = u.id WHERE u.grade = 'A' LIMIT 1000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS wrong_order_ms;
```

### 실험 3: join_buffer_size 효과 (BNL/Hash Join)

```sql
-- join_buffer_size 조정 (인덱스 없는 JOIN에서)
DROP INDEX idx_user_id ON jtest_orders;

-- 작은 버퍼
SET SESSION join_buffer_size = 32768;  -- 32KB
SET @start = NOW(6);
SELECT COUNT(*) FROM jtest_users u JOIN jtest_orders o ON o.user_id = u.id;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS small_buffer_ms;

-- 큰 버퍼
SET SESSION join_buffer_size = 67108864;  -- 64MB
SET @start = NOW(6);
SELECT COUNT(*) FROM jtest_users u JOIN jtest_orders o ON o.user_id = u.id;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS large_buffer_ms;

-- 결과: 버퍼가 클수록 Driven Table 순회 횟수 감소 → 빠름

-- 원복
SET SESSION join_buffer_size = DEFAULT;
CREATE INDEX idx_user_id ON jtest_orders(user_id);
```

---

## 📊 성능 비교

```
JOIN 알고리즘별 비용 비교 (users 1만 건 중 100건 결과, orders 100만 건):

Index Nested Loop Join (인덱스 있음):
  100 × idx_lookup(orders, user_id) = 100 × (3 I/O + 100 Double Lookup)
  총 ≈ 10,300 I/O
  → 가장 빠름 (결과가 소수일 때)

Hash Join (인덱스 없음):
  Build: users 100건 → Hash Table
  Probe: orders 100만 건 Full Scan + Hash 매칭
  총 ≈ 12,346 I/O + CPU 100만 × hash_lookup
  → NLJ(no index)보다 훨씬 빠름, NLJ(index)보다는 느림

Block Nested Loop (인덱스 없음, MySQL < 8.0.18):
  버퍼 크기에 따라 orders Full Scan 횟수 결정
  join_buffer_size=256KB → 여러 번 Full Scan
  총 >> Hash Join
  → 가장 느린 방법

결론:
  인덱스 있음 + 선택적 Driving → NLJ 최적
  인덱스 없음 + 대용량 → Hash Join (8.0.18+)
  인덱스 없음 + 대용량 + 구버전 → join_buffer_size 최대화
```

---

## ⚖️ 트레이드오프

```
JOIN 방식 선택:

NLJ + 인덱스:
  ✅ 결과 건수가 적을 때 매우 빠름
  ✅ 첫 번째 Row를 빠르게 반환 (스트리밍)
  ✅ LIMIT 있는 쿼리에서 전체를 읽지 않아도 됨
  ❌ 인덱스 없으면 O(n²)으로 최악

Hash Join:
  ✅ 인덱스 없어도 O(m+n) — 대용량 equijoin에 강함
  ✅ 병렬 처리 친화적 (미래 버전에서 개선 여지)
  ❌ Build Phase에서 메모리 필요 (join_buffer_size)
  ❌ 첫 Row 반환이 늦음 (Build 완료 후 Probe 시작)
  ❌ Non-equijoin 불가 (>, <, BETWEEN)
  ❌ join_buffer_size 초과 시 디스크 Hash → 급격한 성능 저하

JPA Fetch Join 주의사항:
  @OneToMany Fetch Join:
    카테시안 곱 → 메모리에서 DISTINCT 처리
    rows가 orders × items가 됨
    N×M이 크면 메모리 OOM 가능
  
  대안:
    @BatchSize(size=100): IN (PK목록) 쿼리로 N+100회
    별도 쿼리: 주 엔티티 조회 후 ID 목록으로 연관 엔티티 조회
    EntityGraph: 특정 Association만 EAGER 로드
```

---

## 📌 핵심 정리

```
JOIN 알고리즘 핵심:

Nested Loop Join:
  Driving (작은 쪽) → Driven (큰 쪽, 인덱스 필수)
  인덱스 있음: O(m × log n) → 빠름
  인덱스 없음: O(m × n) → 느림 (지양)
  
Hash Join (MySQL 8.0.18+):
  Build Phase: 작은 테이블 → Hash Table
  Probe Phase: 큰 테이블 Full Scan + Hash 매칭
  O(m + n) → 인덱스 없는 대용량 JOIN에 효율적
  메모리 필요 (join_buffer_size)
  Non-equijoin 불가

Driving Table:
  WHERE 조건 후 결과가 가장 적은 테이블
  Optimizer가 자동 선택 (통계 기반)
  잘못 선택 시 STRAIGHT_JOIN 또는 JOIN_ORDER 힌트

JOIN 인덱스 체크리스트:
  ① Driving Table: WHERE 조건 컬럼에 인덱스
  ② Driven Table: JOIN 조건 컬럼(FK)에 인덱스
  없으면 → Hash Join (8.0.18+) 또는 BNL → 성능 저하

JPA:
  Lazy Loading N+1 → Fetch Join으로 해결
  @OneToMany Fetch Join: 카테시안 곱 주의, DISTINCT 필수 또는 @BatchSize 대안
  생성된 SQL을 EXPLAIN으로 반드시 확인
```

---

## 🤔 생각해볼 문제

**Q1.** `INNER JOIN`과 `LEFT JOIN`은 JOIN 알고리즘(NLJ, Hash Join)에서 어떻게 다르게 처리되는가?

<details>
<summary>해설 보기</summary>

**INNER JOIN**: 매칭되는 Row만 결과에 포함. Driving Table의 Row에 Driven Table에서 매칭 Row가 없으면 결과에서 제외. NLJ와 Hash Join 모두 동일한 방식으로 처리합니다.

**LEFT JOIN**: Driving Table(왼쪽)의 모든 Row를 결과에 포함. Driven Table에서 매칭 Row가 없으면 NULL로 채움.

알고리즘 차이:
- **NLJ + LEFT JOIN**: Driven Table에서 매칭 Row를 못 찾으면 NULL Row를 생성하여 emit.
- **Hash Join + LEFT JOIN**: Probe Phase에서 Hash Table에 키가 없으면 NULL Row 생성. Hash Join이 LEFT JOIN을 완전히 지원합니다 (MySQL 8.0.18+).

**중요한 최적화 차이**: LEFT JOIN에서 Optimizer는 반드시 왼쪽 테이블을 Driving으로 사용해야 합니다. INNER JOIN처럼 자유롭게 순서를 바꿀 수 없습니다. 이것이 LEFT JOIN이 때로 INNER JOIN보다 느린 이유입니다. 조건이 만족되면 LEFT JOIN을 INNER JOIN으로 변환하는 `outer_join_simplification` 최적화가 있습니다.

</details>

---

**Q2.** JPA에서 `@ManyToMany` 관계를 Fetch Join 할 때 성능 문제가 발생하는 이유는?

<details>
<summary>해설 보기</summary>

`@ManyToMany`는 중간 테이블을 통해 연결됩니다. Fetch Join 시:

```sql
-- 예: User ↔ Role (ManyToMany, user_roles 중간 테이블)
SELECT u.*, r.*
FROM users u
INNER JOIN user_roles ur ON ur.user_id = u.id
INNER JOIN roles r ON r.id = ur.role_id
WHERE u.status = 'ACTIVE'
```

문제:
1. **카테시안 곱**: user 1명이 role 5개를 가지면 5행 반환. Hibernate가 중복을 제거하려면 메모리에서 처리 → 대규모에서 메모리 낭비.
2. **2번의 JOIN**: users → user_roles → roles 두 번의 JOIN.
3. **DISTINCT 없이 사용 시**: 동일한 User가 Role 수만큼 중복 등장.

**올바른 처리**:
```java
// HQL: DISTINCT 추가
@Query("SELECT DISTINCT u FROM User u JOIN FETCH u.roles WHERE u.status = 'ACTIVE'")

// 또는 @BatchSize로 분리
@ManyToMany
@BatchSize(size = 100)
Set<Role> roles;
// → SELECT * FROM users WHERE status='ACTIVE'
// → SELECT * FROM user_roles WHERE user_id IN (1,2,...,100)
// → SELECT * FROM roles WHERE id IN (...)
// 중복 없음, 메모리 효율적
```

대규모 ManyToMany는 Fetch Join보다 @BatchSize나 별도 쿼리가 안전합니다.

</details>

---

**Q3.** Hash Join의 Build Input(작은 테이블)이 `join_buffer_size`를 초과하면 어떤 일이 일어나는가?

<details>
<summary>해설 보기</summary>

`join_buffer_size`를 초과하면 **On-Disk Hash Join**이 발생합니다.

과정:
1. 메모리가 가득 찬 시점에서 Build Input을 디스크의 여러 파티션으로 분할 (Partition Spill).
2. Probe Input도 동일한 해시 함수로 파티션.
3. 각 파티션 쌍을 메모리에 로드하여 매칭.

비용 영향:
- 디스크 I/O 급증: Build Input과 Probe Input 모두 디스크에 쓰고 읽음 → 2 Pass I/O.
- 성능 저하: 메모리 Hash Join 대비 수십 배 느릴 수 있음.

감지 방법:
```sql
SHOW STATUS LIKE 'Created_tmp_disk_tables';  -- 디스크 임시 테이블 생성 수
-- 또는 performance_schema events_statements_history의 CREATED_TMP_DISK_TABLES
```

해결:
```sql
SET SESSION join_buffer_size = 128 * 1024 * 1024;  -- 128MB로 증가
-- 또는 쿼리를 분리하여 Build Input 크기 감소
-- 또는 인덱스 추가 → NLJ로 전환 (메모리 불필요)
```

`join_buffer_size`는 세션 단위로 조정 가능하므로, 특정 배치 쿼리에서만 임시로 크게 설정하는 것이 좋습니다. 전역 설정은 모든 연결에 적용되어 메모리 사용량을 급증시킵니다.

</details>

---

<div align="center">

**[⬅️ 이전: Cost-Based Optimizer](./03-cost-based-optimizer.md)** | **[홈으로 🏠](../README.md)** | **[다음: Statistics ➡️](./05-statistics.md)**

</div>
