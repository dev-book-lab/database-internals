# 힌트와 강제 인덱스 사용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `FORCE INDEX`, `USE INDEX`, `IGNORE INDEX`는 각각 어떻게 다르게 동작하는가?
- Optimizer가 `FORCE INDEX` 힌트를 무시하는 경우가 있는가?
- MySQL 8.0의 Optimizer Hint (`/*+ ... */`) 방식이 기존 방식보다 나은 이유는?
- JPA에서 인덱스 힌트를 적용하는 방법은?
- 힌트에 의존하는 코드가 장기적으로 위험한 이유는?
- 힌트 없이 올바른 실행계획을 유도하는 근본적인 해결책은?

---

## 🔍 왜 이 개념이 중요한가

### 힌트는 최후의 수단, 근본 해결이 먼저

```
힌트가 필요한 상황:
  EXPLAIN: type: ALL (Full Scan)
  분명히 인덱스가 있고 사용해야 하는데 Optimizer가 안 씀
  ANALYZE TABLE 해도 여전히 Full Scan 선택
  
  이때 임시 조치: FORCE INDEX(idx_user_id)
  → 즉각적인 성능 개선

힌트의 문제점:
  코드에 하드코딩 → 스키마 변경 시 인덱스명 바뀌면 오류
  데이터 변화 → 힌트가 맞지 않을 수 있음
  ORM에서 사용 어려움 (네이티브 쿼리 강제)
  
  "힌트가 있어야 하는 쿼리" = 다음 중 하나:
    통계가 부정확함 → ANALYZE TABLE, 히스토그램
    인덱스 설계가 잘못됨 → 인덱스 재설계
    쿼리가 잘못됨 → 쿼리 재작성
  
힌트를 사용하기 전 반드시:
  ① EXPLAIN ANALYZE로 실제 vs 예측 비교 (통계 오류 확인)
  ② ANALYZE TABLE + 히스토그램 시도
  ③ 인덱스 재설계 고려
  ④ 그래도 안 되면 힌트 (이유 주석 필수)
```

---

## 🔬 내부 동작 원리

### 1. 인덱스 힌트 3종 비교

```sql
-- FORCE INDEX: 지정한 인덱스 중 하나를 반드시 사용
-- Optimizer가 해당 인덱스를 사용하지 않을 수 없음
-- 단, 인덱스로 접근이 불가능한 경우(해당 없는 조건 등) → Full Scan 선택
SELECT * FROM orders FORCE INDEX (idx_user_id)
WHERE user_id = 1;
-- Optimizer는 idx_user_id를 반드시 고려
-- 비용 계산 없이 이 인덱스만 사용

-- USE INDEX: 지정한 인덱스만 후보로 제한 (다른 인덱스 무시)
-- FORCE와 달리, 지정한 인덱스보다 Full Scan이 싸면 Full Scan 선택 가능
SELECT * FROM orders USE INDEX (idx_user_id)
WHERE user_id = 1;
-- Optimizer: idx_user_id vs Full Scan 비용 비교
-- idx_user_id가 더 비싸면 Full Scan 선택 가능

-- IGNORE INDEX: 지정한 인덱스를 무시
SELECT * FROM orders IGNORE INDEX (idx_bad_index)
WHERE user_id = 1;
-- idx_bad_index 제외한 다른 인덱스 + Full Scan 중 선택

-- 사용 범위 지정:
FORCE INDEX FOR JOIN (idx_user_id)    -- JOIN에서만
FORCE INDEX FOR ORDER BY (idx_created) -- ORDER BY에서만
FORCE INDEX FOR GROUP BY (idx_status)  -- GROUP BY에서만

-- 여러 인덱스 지정 (USE/IGNORE):
USE INDEX (idx_user_id, idx_status)
-- 이 두 인덱스만 후보

-- FORCE INDEX가 무시되는 경우:
-- 힌트에서 지정한 인덱스가 쿼리 조건과 전혀 관계없을 때
-- 예: FORCE INDEX (idx_created_at) WHERE user_id = 1 ORDER BY amount
-- → idx_created_at은 user_id나 amount 조건에 무관 → 사용 불가 → Full Scan
```

### 2. MySQL 8.0 Optimizer Hint

```sql
-- 기존 인덱스 힌트 방식의 문제:
-- 테이블 레퍼런스 뒤에만 가능 → 복잡한 쿼리에서 위치 제한

-- MySQL 8.0 Optimizer Hint: /*+ ... */ 방식
-- SELECT 바로 뒤에 위치 → 모든 쿼리 타입에 적용 가능

-- INDEX 힌트:
SELECT /*+ INDEX(o idx_user_id) */ *
FROM orders o
WHERE o.user_id = 1;
-- o는 테이블 별칭, idx_user_id는 사용할 인덱스

-- NO_INDEX 힌트:
SELECT /*+ NO_INDEX(o idx_bad_index) */ *
FROM orders o
WHERE o.user_id = 1;

-- JOIN_ORDER 힌트:
SELECT /*+ JOIN_ORDER(u, o) */ *
FROM users u JOIN orders o ON o.user_id = u.id
WHERE u.grade = 'GOLD';
-- u를 Driving, o를 Driven으로 고정

-- JOIN_PREFIX / JOIN_SUFFIX:
SELECT /*+ JOIN_PREFIX(u) JOIN_SUFFIX(oi) */ *
FROM orders o JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id;
-- u를 첫 번째, oi를 마지막으로 (중간 순서는 Optimizer에게)

-- SEMIJOIN 힌트:
SELECT /*+ SEMIJOIN(@subq1 FIRSTMATCH) */ *
FROM users u
WHERE u.id IN (SELECT /*+ QB_NAME(subq1) */ user_id FROM orders);

-- BKA / NO_BKA (Batched Key Access):
SELECT /*+ BKA(o) */ *
FROM users u JOIN orders o ON o.user_id = u.id;
-- o 테이블 접근 시 BKA 사용 (MRR + Join Buffer)

-- HASH_JOIN / NO_HASH_JOIN:
SELECT /*+ HASH_JOIN(o) */ *
FROM users u JOIN orders o ON o.user_id = u.id;
-- o 테이블과의 JOIN에 Hash Join 강제

-- 힌트 적용 여부 확인:
EXPLAIN FORMAT=TREE SELECT /*+ INDEX(o idx_user_id) */ * FROM orders o WHERE o.user_id = 1\G
-- "Index lookup on o using idx_user_id" 확인
```

### 3. JPA에서 힌트 적용

```java
// 방법 1: @QueryHints (JPA 표준)
@QueryHints(value = {
    @QueryHint(name = "jakarta.persistence.query.hint.comment", value = "/*+ INDEX(o idx_user_id) */")
})
List<Order> findByUserId(Long userId);
// 단, JPA QueryHints는 주로 캐시, 페치 설정용
// SQL 레벨 Optimizer Hint는 아래 방법 사용

// 방법 2: Native Query
@Query(value = "SELECT /*+ INDEX(o idx_user_id) */ * FROM orders o WHERE o.user_id = ?1",
       nativeQuery = true)
List<Order> findByUserIdHinted(Long userId);
// 직접 SQL 레벨 힌트 삽입 가능
// 단, JPA 엔티티 매핑 불가 → Map 또는 Projection 반환 필요

// 방법 3: JPQL Comment로 Optimizer Hint 전달
// Hibernate 설정:
// spring.jpa.properties.hibernate.use_sql_comments=true
@Query("/* /*+ INDEX(o idx_user_id) */ */ SELECT o FROM Order o WHERE o.userId = :userId")
// 주의: JPQL 파서가 /* */를 일반 주석으로 처리할 수 있음 → Native Query 권장

// 방법 4: Querydsl에서 Native Query 래핑
@Repository
public class OrderRepositoryCustomImpl {
    @PersistenceContext
    EntityManager em;
    
    public List<Order> findByUserIdHinted(Long userId) {
        return em.createNativeQuery(
            "SELECT /*+ INDEX(o idx_user_id) */ * FROM orders o WHERE o.user_id = ?1",
            Order.class
        ).setParameter(1, userId).getResultList();
    }
}

// 실무 권장: 힌트 필요 시 Repository에 별도 메서드로 분리
// 이유와 임시 조치임을 주석으로 명확히 기록
```

### 4. 힌트 없이 올바른 실행계획 유도하기

```sql
-- 근본 해결 5단계:

-- Step 1: EXPLAIN ANALYZE로 문제 정확히 파악
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1\G
-- (cost=100 rows=100) vs (actual rows=50000 loops=1)
-- → rows 추정 100, 실제 50000 → 통계 오류

-- Step 2: 통계 갱신
ANALYZE TABLE orders;
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- rows 값이 바뀌었는지 확인

-- Step 3: 히스토그램 (편중 분포인 경우)
ANALYZE TABLE orders UPDATE HISTOGRAM ON user_id WITH 1024 BUCKETS;
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G

-- Step 4: innodb_stats_persistent_sample_pages 증가
SET GLOBAL innodb_stats_persistent_sample_pages = 100;
ANALYZE TABLE orders;
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G

-- Step 5: 인덱스 재설계 (쿼리 패턴 재분석)
-- 현재: idx_user_id (user_id)
-- 개선: idx_user_status (user_id, status) + Covering
-- 목표 쿼리: WHERE user_id = 1 AND status = 'PAID'
-- → (user_id, status)로 범위 더 좁힘 → 올바른 선택 유도

-- Step 6: 쿼리 재작성 (최후의 수단)
-- 서브쿼리 → JOIN 변환, IN → EXISTS 변환 등
-- Optimizer가 서로 다른 실행계획을 선택하도록 유도

-- 힌트 최후 사용 예시 (이유 문서화):
SELECT /*+ INDEX(o idx_user_id) */ *  -- TODO: ANALYZE 후 제거 예정 (#issue-1234)
-- 또는
SELECT * FROM orders FORCE INDEX (idx_user_id)
-- TODO: 통계 오류로 임시 힌트 적용
--       2024-Q1 배치 후 ANALYZE TABLE로 해결 예정
WHERE user_id = 1;
```

### 5. Invisible Index로 힌트 없이 테스트

```sql
-- 힌트 대신 Invisible Index로 안전하게 실행계획 테스트

-- 특정 인덱스를 Invisible로 만들어 Optimizer에서 제외
ALTER TABLE orders ALTER INDEX idx_bad_index INVISIBLE;

-- Optimizer가 idx_bad_index 없이 어떤 계획을 선택하는지 확인
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- 이제 idx_bad_index를 제외한 실행계획 표시

-- 문제 없으면 인덱스 삭제, 문제 있으면 복원
ALTER TABLE orders ALTER INDEX idx_bad_index VISIBLE;  -- 복원

-- Invisible Index를 세션에서만 사용 가능하게:
-- (특정 쿼리에서만 Invisible 인덱스를 사용해보고 싶을 때)
SET SESSION optimizer_switch = 'use_invisible_indexes=on';
-- 이 세션에서는 Invisible 인덱스도 Optimizer가 고려함
-- → FORCE INDEX 없이도 특정 인덱스를 테스트 가능

EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- idx_bad_index(Invisible이지만 세션에서 활성화됨)를 포함한 실행계획

SET SESSION optimizer_switch = 'use_invisible_indexes=off';  -- 복원
```

---

## 💻 실전 실험

### 실험 1: FORCE vs USE vs IGNORE 동작 차이

```sql
CREATE INDEX idx_user ON orders(user_id);
CREATE INDEX idx_amount ON orders(amount);

-- 기본 실행계획
EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND amount > 5000\G
-- Optimizer 선택 인덱스 확인

-- FORCE INDEX: 무조건 지정 인덱스
EXPLAIN SELECT * FROM orders FORCE INDEX (idx_amount)
WHERE user_id = 1 AND amount > 5000\G
-- key: idx_amount (강제)

-- USE INDEX: 지정 인덱스만 후보
EXPLAIN SELECT * FROM orders USE INDEX (idx_amount)
WHERE user_id = 1 AND amount > 5000\G
-- key: idx_amount 또는 NULL (Full Scan이 더 싸면)

-- IGNORE INDEX: 특정 인덱스 제외
EXPLAIN SELECT * FROM orders IGNORE INDEX (idx_user)
WHERE user_id = 1 AND amount > 5000\G
-- key: idx_amount 또는 NULL (idx_user 제외)

-- 비용 비교 (FORMAT=JSON):
EXPLAIN FORMAT=JSON SELECT * FROM orders FORCE INDEX (idx_amount)
WHERE user_id = 1 AND amount > 5000\G
-- "cost_info" 확인
```

### 실험 2: Optimizer Hint vs 기존 힌트

```sql
-- 기존 방식
EXPLAIN SELECT * FROM orders FORCE INDEX (idx_user_id)
WHERE user_id = 1\G

-- 새로운 방식 (Optimizer Hint)
EXPLAIN SELECT /*+ INDEX(orders idx_user_id) */ *
FROM orders
WHERE user_id = 1\G

-- JOIN에서 순서 힌트
EXPLAIN SELECT /*+ JOIN_ORDER(u, o) */
    u.name, COUNT(o.id)
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.id\G
-- 출력에서 JOIN 순서 확인 (u → o)

-- 반대 순서 강제
EXPLAIN SELECT /*+ JOIN_ORDER(o, u) */
    u.name, COUNT(o.id)
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.id\G
-- 출력에서 JOIN 순서 (o → u) 확인
-- 성능 차이 측정
```

### 실험 3: 힌트 필요 없이 통계로 해결

```sql
-- 힌트가 필요한 상황 재현
-- user_id=99999에 데이터 집중 후 통계 오류 발생 (이전 실험과 동일)

-- 1단계: 힌트 없이 최적화 시도
ANALYZE TABLE orders;
EXPLAIN SELECT * FROM orders WHERE user_id = 99999\G

-- 2단계: 히스토그램 추가
ANALYZE TABLE orders UPDATE HISTOGRAM ON user_id WITH 512 BUCKETS;
EXPLAIN SELECT * FROM orders WHERE user_id = 99999\G
-- rows 추정이 더 정확해졌는지 확인

-- 3단계: 힌트 없이 올바른 실행계획 달성 여부 확인
-- 올바른 계획: user_id=99999가 10%면 Full Scan, 0.01%면 Index Scan

-- 비교: 힌트 강제 vs 통계 최적화 후 자동 선택
EXPLAIN SELECT /*+ INDEX(orders idx_user) */ * FROM orders WHERE user_id = 1\G
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G  -- 통계 개선 후
-- 두 실행계획이 동일하면 힌트 제거 가능
```

---

## 📊 성능 비교

```
힌트 사용 방식별 관리 비용:

기존 인덱스 힌트 (FORCE INDEX):
  장점: 즉각적 효과, 문법 간단
  단점: 인덱스명 하드코딩, 위치 제한, 복잡한 쿼리에서 관리 어려움
  위험: 인덱스 삭제/이름 변경 시 오류 또는 조용한 성능 저하

MySQL 8.0 Optimizer Hint:
  장점: SELECT 직후 위치, 세밀한 제어 (JOIN 순서, 알고리즘)
  단점: 문법이 복잡, 인덱스명 하드코딩은 동일
  권장: 기존 방식 대신 이 방식 사용

통계 기반 자동 선택 (이상적):
  장점: 코드 변경 없음, 데이터 변화에 적응
  단점: 통계 관리 필요 (ANALYZE TABLE, 히스토그램)
  비용: 주기적인 통계 갱신 작업
  권장: 힌트보다 이 방식을 먼저 추구
```

---

## ⚖️ 트레이드오프

```
힌트 사용 결정 기준:

힌트를 사용해야 하는 경우 (예외적):
  ① 통계를 최대한 정확하게 해도 Optimizer가 잘못 선택
  ② 특수한 하드웨어/워크로드 환경에서 Optimizer의 비용 모델과 실제가 다름
  ③ 임시 핫픽스가 필요한 긴급 상황

힌트를 사용하면 안 되는 경우:
  ❌ 단순히 EXPLAIN을 보지 않고 "혹시 더 빠르겠지"
  ❌ 통계 문제인데 힌트로 땜빵
  ❌ JPA 전체를 Native Query로 바꾸는 대가가 너무 큼

힌트 사용 시 필수 사항:
  ✅ 이유를 주석으로 문서화
  ✅ 이슈 트래커에 기록 (티켓 번호)
  ✅ 통계 정비 후 힌트 제거 계획
  ✅ 정기적으로 힌트가 여전히 필요한지 재검토

FORCE INDEX의 숨겨진 위험:
  인덱스 이름 변경 → 힌트 무시 → 다른 인덱스 선택 → 성능 저하
  인덱스 삭제 → 에러 또는 Full Scan
  데이터 분포 변화 → 힌트가 맞지 않는 인덱스 강제
  → 힌트는 "이 시점에만 올바른 선택"일 수 있음
```

---

## 📌 핵심 정리

```
힌트 핵심:

인덱스 힌트 3종:
  FORCE INDEX: 지정 인덱스 중 하나를 반드시 사용 (가능하면)
  USE INDEX: 지정 인덱스만 후보 (여전히 Full Scan 가능)
  IGNORE INDEX: 지정 인덱스 제외

MySQL 8.0 Optimizer Hint:
  /*+ INDEX(alias index_name) */: 인덱스 지정
  /*+ JOIN_ORDER(t1, t2) */: JOIN 순서 지정
  /*+ HASH_JOIN(t) */: 알고리즘 강제
  기존 방식보다 세밀한 제어 가능

JPA에서의 힌트:
  @Query(nativeQuery=true): 네이티브 쿼리로 직접 힌트 삽입
  ORM의 추상화를 깨는 것이므로 최소화

근본 해결책 (힌트 전에):
  1. EXPLAIN ANALYZE → 실제 vs 예측 비교
  2. ANALYZE TABLE → 통계 갱신
  3. 히스토그램 생성 (편중 분포)
  4. 인덱스 재설계
  5. 마지막 수단으로 힌트 (이유 문서화 필수)

Invisible Index:
  ALTER TABLE ... ALTER INDEX ... INVISIBLE:
  힌트 없이 인덱스 비활성화 효과를 안전하게 테스트
  문제 없으면 DROP, 문제 있으면 VISIBLE로 즉시 복원
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 상황에서 `FORCE INDEX`를 사용하는 것이 옳은가? 옳지 않다면 올바른 접근을 설명하라.

```sql
-- 상황: Optimizer가 idx_created_at을 선택했지만, idx_user_id가 더 빠른 것을 확인
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- type: range, key: idx_created_at ← 잘못된 선택!
-- EXPLAIN ANALYZE: (cost=500, rows=100) vs (actual rows=10 loops=1)
-- idx_user_id로 FORCE INDEX 시: (actual time=1ms rows=10)
```

<details>
<summary>해설 보기</summary>

**FORCE INDEX를 바로 사용하면 안 됩니다.** 먼저 왜 Optimizer가 잘못 선택했는지 원인을 파악해야 합니다.

**원인 분석**:
- `idx_created_at`가 선택됐다는 것 = Optimizer가 `idx_created_at`의 비용을 `idx_user_id`보다 낮게 계산했다는 의미
- `WHERE user_id = 1`인데 `idx_created_at`를 선택한 이유: `user_id = 1`의 예상 결과가 매우 높아서 (통계 오류) → `created_at` Range Scan이 더 싸 보임

**올바른 순서**:
1. `ANALYZE TABLE orders;` → 통계 갱신
2. `ANALYZE TABLE orders UPDATE HISTOGRAM ON user_id WITH 256 BUCKETS;`
3. `EXPLAIN SELECT * FROM orders WHERE user_id = 1;` → 올바른 인덱스 선택되는지 확인

**그래도 안 되면**: `FORCE INDEX(idx_user_id)` 사용, 단 주석으로 이유 기록.

EXPLAIN ANALYZE의 `(cost=500, rows=100) vs (actual rows=10)`은 통계 오류의 명확한 증거입니다. FORCE INDEX보다 통계 정비가 근본 해결책입니다.

</details>

---

**Q2.** `IGNORE INDEX (PRIMARY)`는 어떤 상황에서 사용할 수 있는가?

<details>
<summary>해설 보기</summary>

`IGNORE INDEX(PRIMARY)`는 Clustered Index(PK)를 힌트 목적에서 제외하는 것입니다. 하지만 **Clustered Index는 데이터 자체**이므로 완전히 무시할 수 없습니다.

실제 동작:
- `FORCE INDEX(PRIMARY)`: PK Range Scan 또는 Full Index Scan을 강제
- `IGNORE INDEX(PRIMARY)`: PK를 ORDER BY나 GROUP BY 최적화 목적으로 무시하도록 힌트
- Clustered Index를 건너뛰고 Secondary Index를 통한 접근만 허용

**유용한 케이스**: `ORDER BY id`가 있는 쿼리에서 Optimizer가 PK 기반 정렬을 선택하지만 다른 인덱스로 더 효율적인 실행이 가능할 때:
```sql
-- Optimizer가 ORDER BY id 때문에 Full Index Scan (PK)을 선택
-- 실제로는 idx_status + filesort가 더 빠른 경우
SELECT * FROM orders IGNORE INDEX(PRIMARY)
WHERE status = 'PAID' ORDER BY id LIMIT 10;
-- Optimizer가 idx_status를 사용하고 id 정렬은 filesort로 처리
```

실무에서 `IGNORE INDEX(PRIMARY)`는 드물게 사용되며, 대부분 인덱스 재설계나 쿼리 수정으로 해결할 수 있습니다.

</details>

---

**Q3.** 스타트업에서 빠르게 성장 중인 서비스의 DB에 `FORCE INDEX` 힌트가 20개 있다. 기술 부채를 해결하기 위한 접근 방법은?

<details>
<summary>해설 보기</summary>

**단계적 접근**:

**1. 현황 파악**:
```sql
-- 힌트가 있는 모든 쿼리 목록화
-- 각 힌트에 대해:
-- - 왜 추가됐는지 (이슈 트래커, git blame)
-- - 현재도 여전히 필요한지 테스트
```

**2. 힌트별 처리 방향 분류**:
- **통계 문제**: ANALYZE TABLE + 히스토그램으로 해결 후 힌트 제거
- **인덱스 문제**: 더 나은 인덱스 설계 → 힌트 불필요
- **Optimizer 버그**: MySQL 버전 업그레이드로 해결 가능한지 확인
- **데이터 분포 특성**: 근본 해결 어려운 경우 → 힌트 유지 (문서화)

**3. 안전한 제거 프로세스**:
```sql
-- 1) Invisible Index로 테스트 환경에서 시뮬레이션
-- 2) EXPLAIN ANALYZE로 힌트 있을 때 vs 없을 때 비교
-- 3) Read Replica에서 힌트 제거 후 성능 모니터링 (1~2주)
-- 4) 문제 없으면 프로덕션에 반영
```

**4. 예방 체계**:
- 코드 리뷰에서 힌트 추가 시 이슈 티켓 번호 필수
- 정기적 통계 갱신 자동화 (배치 스크립트)
- 신규 힌트는 Optimizer Hint 방식 + 주석 강제

빠르게 성장하는 서비스에서 힌트가 20개 있다는 것은 통계 관리 체계가 없었다는 신호입니다. 통계 갱신 자동화를 먼저 도입하면 대부분의 힌트가 자연스럽게 불필요해집니다.

</details>

---

<div align="center">

**[⬅️ 이전: Statistics](./05-statistics.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Transaction & MVCC ➡️](../transaction-mvcc/01-acid-guarantees.md)**

</div>
