# Index Selectivity와 Cardinality

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Cardinality가 낮은 컬럼(`status`, `gender`)에 인덱스를 걸면 왜 오히려 느릴 수 있는가?
- Optimizer가 인덱스를 사용할지 Full Scan을 선택할지 결정하는 기준은 무엇인가?
- `ANALYZE TABLE`은 언제 실행해야 하고, 통계가 잘못되면 어떤 문제가 생기는가?
- Selectivity가 낮아도 인덱스가 유용한 경우는 언제인가?
- Optimizer가 잘못된 인덱스를 선택할 때 `FORCE INDEX`로 힌트를 주는 방법은?
- 인덱스의 Prefix(앞 N 글자)만 인덱싱하는 방법과 트레이드오프는?

---

## 🔍 왜 이 개념이 중요한가

### "status 컬럼에 인덱스를 걸었는데 오히려 더 느려졌다"

```
실제 상황:
  orders 테이블 100만 건
  status 컬럼: 'PENDING', 'PAID', 'CANCELLED' 3가지 값
  각 값 약 33만 건씩 분포
  
  CREATE INDEX idx_status ON orders(status);
  
  쿼리: SELECT * FROM orders WHERE status = 'PAID';
  기대: 인덱스로 빠르게 33만 건 조회
  
  실제:
    Optimizer: "status='PAID'이면 33만 건, 즉 33% 비율"
              "Full Scan vs Index Scan 비용 계산..."
              → Full Scan 선택! (또는 인덱스를 타도 더 느림)
  
  이유:
    33만 건의 PK → Clustered Index 33만 번 Random I/O
    vs Full Scan: 수만 번의 Sequential I/O
    Random I/O × 33만 >> Sequential I/O (전체)

  교훈:
    인덱스는 "결과가 전체의 극히 일부"일 때 효과적
    결과 비율이 크면 Full Scan이 더 빠를 수 있음
    → Selectivity가 핵심 판단 기준
```

---

## 😱 잘못된 이해

### Before: "자주 검색하는 컬럼은 다 인덱스 걸어야지"

```sql
-- 잘못된 접근: 모든 WHERE 조건 컬럼에 인덱스
CREATE TABLE users (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    name    VARCHAR(50),
    gender  CHAR(1),       -- 'M', 'F', 'O' (3가지)
    status  VARCHAR(20),   -- 'ACTIVE', 'INACTIVE', 'BANNED' (3가지)
    grade   VARCHAR(10),   -- 'BRONZE', 'SILVER', 'GOLD', 'PLATINUM' (4가지)
    country VARCHAR(50),   -- 200여 개 나라
    email   VARCHAR(100)   -- 거의 고유
) ENGINE=InnoDB;

-- "WHERE 절에 쓰이니까 인덱스 걸자"
CREATE INDEX idx_gender ON users(gender);     -- ❌ 3가지만 → 매우 낮은 Selectivity
CREATE INDEX idx_status ON users(status);     -- ❌ 3가지만
CREATE INDEX idx_grade  ON users(grade);      -- ❌ 4가지만
CREATE INDEX idx_country ON users(country);   -- △ 200개 → 낮은 Selectivity
CREATE INDEX idx_email  ON users(email);      -- ✅ 거의 고유 → 높은 Selectivity

-- 결과:
-- gender, status, grade는 Optimizer가 거의 사용 안 함 (낮은 Selectivity)
-- 하지만 INSERT/UPDATE마다 불필요한 인덱스 3개 갱신 (쓰기 성능 저하)
-- Buffer Pool에서 의미없는 인덱스가 공간 차지
```

---

## ✨ 올바른 이해

### After: Selectivity가 인덱스 효과를 결정한다

```
Selectivity (선택도) 정의:
  Selectivity = Cardinality / 전체 Row 수
  = "이 인덱스가 결과를 얼마나 좁혀주는가"

  높은 Selectivity (1에 가까울수록):
    email: 100만 Row, 고유값 100만 개 → Selectivity ≈ 1.0
    → 인덱스 1건 → Clustered 1번 → 매우 효율적

  낮은 Selectivity (0에 가까울수록):
    status: 100만 Row, 고유값 3개 → Selectivity = 3/1,000,000 ≈ 0.000003
    → 인덱스 33만 건 → Clustered 33만 번 → 비효율

  Optimizer의 임계값:
    결과가 전체의 약 20~30% 이하: 인덱스 스캔이 유리
    결과가 전체의 약 20~30% 이상: Full Scan이 유리 (Sequential I/O 활용)
    (정확한 임계값은 통계, 하드웨어, 쿼리에 따라 다름)

Selectivity 계산:
  SELECT COUNT(DISTINCT status) / COUNT(*) AS selectivity
  FROM orders;
  -- 결과: 0.000003 → 매우 낮음 → 인덱스 비효율

  SELECT COUNT(DISTINCT email) / COUNT(*) AS selectivity
  FROM users;
  -- 결과: ~1.0 → 인덱스 매우 효율적
```

---

## 🔬 내부 동작 원리

### 1. InnoDB 비용 기반 Optimizer

```
Optimizer의 인덱스 선택 과정:

① 통계 정보 읽기:
   mysql.innodb_index_stats 테이블에서
   각 인덱스의 Cardinality, n_leaf_pages 등 읽기

② 각 접근 방법의 비용 계산:
   비용 = (I/O 비용) + (CPU 비용)
   
   Full Table Scan 비용:
     = 전체 Page 수 × per_page_cost
     = 12,346 × 1.0 = 12,346 비용 단위

   Index Scan + Lookup 비용 (status='PAID'):
     = Secondary Index Page 수 × per_page_cost
     + 예상 결과 건수 × per_lookup_cost
     = 수십 + 333,333 × 1.1 = ~366,666 비용 단위
     
     → Full Scan(12,346)이 Index Scan(366,666)보다 30배 싸다!
     → Optimizer: Full Scan 선택

③ 비용 기반 선택:
   가장 낮은 비용의 접근 방법 선택
   여러 인덱스가 있을 때 각 인덱스별 비용 비교

-- Optimizer 비용 확인 (MySQL 8.0+)
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE status = 'PAID'\G
-- 출력에서 "cost_info" 확인
-- "query_cost": "12345.67" ← 전체 쿼리 비용
-- "read_cost": "..."       ← I/O 비용
-- "eval_cost": "..."       ← 필터 평가 비용
```

### 2. 통계 정보와 ANALYZE TABLE

```sql
-- InnoDB 통계 테이블 확인
SELECT
    database_name,
    table_name,
    index_name,
    stat_name,
    stat_value,
    sample_size
FROM mysql.innodb_index_stats
WHERE database_name = 'deep_dive'
  AND table_name = 'orders'
ORDER BY index_name, stat_name;

-- 주요 stat_name:
-- n_diff_pfx01: 인덱스 첫 번째 컬럼의 고유값 수 (Cardinality)
-- n_diff_pfx02: 첫 번째+두 번째 컬럼 조합의 고유값 수
-- n_leaf_pages: Leaf Page 수
-- size: 전체 인덱스 Page 수

-- 통계 갱신 시점:
-- 자동: 테이블의 1/16 이상 변경 시 자동 갱신 (innodb_stats_auto_recalc = ON)
-- 수동: ANALYZE TABLE 실행

-- 통계가 부정확해지는 경우:
-- 대량 INSERT/DELETE 후 아직 자동 갱신 전
-- 데이터 분포가 편중되어 있을 때 샘플링 오류
-- 통계 테이블이 업데이트되지 않은 테이블 RENAME 후

-- ANALYZE TABLE 실행
ANALYZE TABLE orders;
-- 인덱스 통계 재계산 및 업데이트
-- MySQL 8.0: 비동기로 실행 가능 (테이블 잠금 없음)

-- 통계 샘플 페이지 수 조정
SHOW VARIABLES LIKE 'innodb_stats_persistent_sample_pages';
-- 기본값: 20 (페이지 샘플링 수)
-- 값을 높이면 통계 정확도 ↑, 분석 시간 ↑
-- 값을 낮추면 통계 정확도 ↓, 분석 시간 ↓

SET GLOBAL innodb_stats_persistent_sample_pages = 50;
ANALYZE TABLE orders;
-- 더 많은 Page를 샘플링 → 더 정확한 통계
```

### 3. 통계 오류로 인한 잘못된 실행 계획

```sql
-- 실제 문제 상황: 통계가 오래되어 Full Scan 대신 인덱스를 선택 (또는 반대)

-- 문제 재현 방법:
-- 1) 대량 INSERT
INSERT INTO orders (user_id, amount, status, created_at)
SELECT ...  -- 100만 건 추가

-- 2) 통계 미갱신 상태에서 쿼리
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- 예상: rows가 실제보다 크게 적거나 많을 수 있음
-- 이로 인해 잘못된 접근 방법 선택

-- 3) ANALYZE 후 재확인
ANALYZE TABLE orders;
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- rows 값이 더 정확해짐

-- Optimizer가 잘못된 선택을 했을 때 해결책:

-- 방법 1: ANALYZE TABLE (통계 갱신)
ANALYZE TABLE orders;

-- 방법 2: FORCE INDEX (인덱스 강제 지정)
SELECT * FROM orders FORCE INDEX (idx_user_id) WHERE user_id = 1;
-- 단, 실제로 인덱스가 더 느리다면 성능 저하 가능
-- 사용 전 반드시 성능 측정

-- 방법 3: USE INDEX (힌트 제공)
SELECT * FROM orders USE INDEX (idx_user_id) WHERE user_id = 1;
-- FORCE보다 약한 힌트 (Optimizer가 무시 가능)

-- 방법 4: IGNORE INDEX (특정 인덱스 제외)
SELECT * FROM orders IGNORE INDEX (idx_bad_index) WHERE user_id = 1;

-- 방법 5: optimizer_switch로 특정 최적화 제어
SET SESSION optimizer_switch = 'index_merge=off';
-- 특정 세션에서만 적용 (프로덕션에서 신중히 사용)
```

### 4. Cardinality가 낮아도 인덱스가 유용한 경우

```
낮은 Cardinality에서 인덱스가 효과적인 케이스:

① Composite Index의 앞 컬럼:
   (status, created_at) 인덱스
   WHERE status = 'PAID' ORDER BY created_at DESC LIMIT 10
   
   → status로 33만 건으로 좁힌 후, created_at 정렬 범위만 스캔
   → LIMIT 10이면 10건만 읽고 완료
   → 단독 status 인덱스보다 훨씬 효율적

② 결과를 극히 소수로 좁히는 다른 조건과 함께:
   WHERE status = 'CANCELLED' AND user_id = 12345
   → user_id=12345 AND status='CANCELLED' = 매우 적은 건수
   → (user_id, status) Composite Index → 효율적

③ LIMIT이 작은 경우:
   WHERE status = 'PENDING' ORDER BY created_at LIMIT 5
   → status='PENDING'인 Leaf 시작에서 created_at 순으로 5건만 읽으면 됨
   → 인덱스 활용이 Full Scan보다 훨씬 빠름

④ SELECT COUNT(*):
   SELECT COUNT(*) FROM orders WHERE status = 'PAID';
   → Covering Index로 Leaf만 스캔 (테이블 접근 없음)
   → status 단독 인덱스도 효율적 (COUNT만이라 Double Lookup 없음)

낮은 Cardinality 컬럼에서 인덱스가 비효율적인 케이스:
  SELECT * FROM orders WHERE status = 'PAID'  → 33만 건 Full → 비효율
  SELECT * FROM orders WHERE gender = 'M'     → 50만 건 Full → 비효율
  (결과가 많고 SELECT *로 모든 컬럼 필요 → Double Lookup 압도적 비용)
```

### 5. Prefix Index (인덱스 앞부분만 사용)

```sql
-- 긴 VARCHAR 컬럼에 전체 인덱스 생성 시 크기 문제:
-- email: VARCHAR(100) utf8mb4 → 최대 400 bytes → 인덱스 크기 큼

-- Prefix Index: 앞 N 글자만 인덱싱
CREATE INDEX idx_email_prefix ON users(email(20));
-- email의 앞 20 글자(80 bytes)만 인덱스로 사용

-- Prefix 길이 선택 기준:
-- 전체 vs Prefix Selectivity 비교
SELECT
    COUNT(DISTINCT LEFT(email, 5)) / COUNT(*) AS sel_5,
    COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) AS sel_10,
    COUNT(DISTINCT LEFT(email, 20)) / COUNT(*) AS sel_20,
    COUNT(DISTINCT email) / COUNT(*) AS sel_full
FROM users;
-- sel_5 ≈ 0.98, sel_10 ≈ 0.999, sel_20 ≈ 1.0, sel_full ≈ 1.0
-- → 앞 10~15 글자만으로도 전체와 거의 같은 Selectivity

-- Prefix Index의 한계:
-- Covering Index로 사용 불가!
-- SELECT email FROM users WHERE email LIKE 'john%';
-- → email 컬럼 전체 값을 SELECT하므로 Prefix만으로 응답 불가
-- → Clustered Index 접근 필요

-- Prefix Index가 적합한 경우:
-- ✅ 긴 VARCHAR 컬럼의 등치/시작 조건 검색 속도 향상
-- ✅ 인덱스 크기 절약이 중요한 경우
-- ❌ Covering Index 목적인 경우
-- ❌ LIKE '%suffix' 패턴 (뒤에서 검색 → 인덱스 사용 불가)
```

---

## 💻 실전 실험

### 실험 1: Selectivity 측정 및 Optimizer 결정 관찰

```sql
-- 컬럼별 Selectivity 계산
SELECT
    'status' AS col_name,
    COUNT(DISTINCT status) AS cardinality,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT status) / COUNT(*) AS selectivity
FROM orders
UNION ALL
SELECT 'user_id', COUNT(DISTINCT user_id), COUNT(*), COUNT(DISTINCT user_id) / COUNT(*) FROM orders
UNION ALL
SELECT 'amount', COUNT(DISTINCT amount), COUNT(*), COUNT(DISTINCT amount) / COUNT(*) FROM orders;

-- 각 인덱스 생성
CREATE INDEX idx_status ON orders(status);
CREATE INDEX idx_user_id ON orders(user_id);

-- Optimizer의 선택 확인
EXPLAIN SELECT * FROM orders WHERE status = 'PAID'\G          -- type: ?
EXPLAIN SELECT * FROM orders WHERE user_id = 12345\G          -- type: ?
-- status: Full Scan 선택 가능성 높음 (낮은 Selectivity)
-- user_id: Index Scan 선택 (높은 Selectivity)

-- 비용 확인 (MySQL 8.0+)
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE status = 'PAID'\G
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE user_id = 12345\G
-- "query_cost" 비교
```

### 실험 2: Cardinality 통계 확인

```sql
-- INFORMATION_SCHEMA에서 인덱스 Cardinality 확인
SELECT
    index_name,
    cardinality,
    (SELECT COUNT(*) FROM orders) AS table_rows,
    cardinality / (SELECT COUNT(*) FROM orders) AS selectivity_estimate
FROM information_schema.STATISTICS
WHERE table_schema = 'deep_dive' AND table_name = 'orders';

-- innodb_index_stats에서 실제 통계 확인
SELECT
    index_name,
    stat_name,
    stat_value,
    sample_size,
    last_update
FROM mysql.innodb_index_stats
WHERE database_name = 'deep_dive' AND table_name = 'orders'
ORDER BY index_name, stat_name;

-- 통계 갱신 후 변화 확인
ANALYZE TABLE orders;
-- 재확인
SELECT cardinality FROM information_schema.STATISTICS
WHERE table_schema = 'deep_dive' AND table_name = 'orders' AND index_name = 'idx_status';
```

### 실험 3: FORCE INDEX vs Optimizer 자동 선택 비교

```sql
-- 상황: Optimizer가 Full Scan을 선택했는데 인덱스가 더 빠른지 검증

-- Optimizer 자동 선택
EXPLAIN SELECT * FROM orders WHERE status = 'PAID'\G
SET @start = NOW(6);
SELECT * FROM orders WHERE status = 'PAID' LIMIT 100;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) AS auto_us;

-- 인덱스 강제 사용
SET @start = NOW(6);
SELECT * FROM orders FORCE INDEX (idx_status) WHERE status = 'PAID' LIMIT 100;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) AS forced_us;

-- 비교: Optimizer가 옳은지 확인
-- LIMIT이 작으면 인덱스가 빠를 수 있음 (전체 결과가 아닌 소수만 필요)
-- LIMIT 없이 전체 결과라면 Optimizer의 Full Scan 선택이 옳을 수 있음
```

### 실험 4: Prefix Index Selectivity 실험

```sql
-- 이메일 Prefix 길이별 Selectivity
SELECT
    LENGTH_5  AS prefix_len,
    COUNT(DISTINCT LEFT(email, 5)) AS cardinality_5,
    COUNT(DISTINCT LEFT(email, 5)) / COUNT(*) AS sel_5,
    COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) AS sel_10,
    COUNT(DISTINCT LEFT(email, 15)) / COUNT(*) AS sel_15,
    COUNT(DISTINCT email) / COUNT(*) AS sel_full,
    5 AS `5`,
    10 AS `10`,
    15 AS `15`
FROM users
CROSS JOIN (SELECT 1 AS LENGTH_5) t
LIMIT 1;

-- Prefix 길이 선택 후 인덱스 생성
CREATE INDEX idx_email_10 ON users(email(10));

-- Full vs Prefix 인덱스 크기 비교
CREATE INDEX idx_email_full ON users(email);

SELECT
    index_name,
    ROUND(stat_value * 16 / 1024, 2) AS size_kb
FROM mysql.innodb_index_stats
WHERE database_name = 'deep_dive'
  AND table_name = 'users'
  AND stat_name = 'size'
  AND index_name IN ('idx_email_10', 'idx_email_full');
```

---

## 📊 성능 비교

```
Selectivity별 인덱스 효율:

status (Selectivity ≈ 0.000003):
  인덱스 결과: 33만 건 → 33만 Random I/O
  Full Scan: 1.2만 Sequential I/O
  결론: Full Scan이 27배 빠름 → 인덱스 비효율

email (Selectivity ≈ 1.0):
  인덱스 결과: 1건 → 1 Random I/O
  Full Scan: 1.2만 Sequential I/O
  결론: 인덱스가 1.2만 배 빠름 → 인덱스 매우 효율적

status + LIMIT 10 (Composite: status, created_at):
  인덱스로 10건만 읽고 완료 → 수십 I/O
  Full Scan: 1.2만 I/O
  결론: 인덱스가 수백 배 빠름 → LIMIT 작으면 낮은 Selectivity도 OK

경험칙:
  결과가 전체 5% 미만: 인덱스 거의 항상 유리
  결과가 전체 5~30%: 상황에 따라 다름 (EXPLAIN + 측정 필요)
  결과가 전체 30% 초과: Full Scan이 유리한 경우 많음
```

---

## ⚖️ 트레이드오프

```
낮은 Cardinality 컬럼에 인덱스 추가 여부:

추가해야 하는 경우:
  ✅ Composite Index의 앞 컬럼으로 사용 (다른 컬럼과 조합)
  ✅ ORDER BY + LIMIT이 작은 쿼리
  ✅ COUNT(*) 집계만 필요한 경우 (Covering 가능)
  ✅ 특정 값이 극히 드문 경우 (예: status='BLOCKED' = 0.001%)

추가하지 말아야 하는 경우:
  ❌ SELECT * + 결과 비율 20% 이상
  ❌ 자주 UPDATE되는 컬럼 (인덱스 갱신 비용)
  ❌ 위의 케이스에 해당하지 않는데 "혹시 몰라서"

인덱스 과잉(Over-Indexing) 문제:
  쓰기 성능 저하:
    인덱스 1개 → INSERT 시 B+Tree 1개 추가 갱신
    인덱스 5개 → INSERT 시 B+Tree 5개 추가 갱신
  
  Buffer Pool 낭비:
    사용 안 하는 인덱스도 Buffer Pool 공간 차지
    Hit Rate 감소 → 성능 저하
  
  인덱스 제거 신호:
    SHOW INDEX FROM table → Cardinality 확인
    performance_schema.table_io_waits_summary_by_index_usage:
      count_read = 0이면 사용 안 하는 인덱스

통계 갱신 주기:
  자동 (innodb_stats_auto_recalc = ON):
    테이블의 1/16 이상 변경 시 백그라운드에서 갱신
    정확하지 않을 수 있음 (샘플링 기반)
  
  수동 ANALYZE TABLE:
    대규모 데이터 변경 후 Optimizer 계획이 이상할 때
    배포 후 데이터 초기화 후
    Slow Query가 갑자기 증가할 때
```

---

## 📌 핵심 정리

```
Selectivity와 Cardinality 핵심:

Selectivity = Cardinality / 전체 Row:
  높을수록 (1에 가까울수록): 인덱스 효율적
  낮을수록 (0에 가까울수록): Full Scan이 유리할 수 있음

Optimizer의 인덱스 선택:
  비용 기반 (I/O + CPU 비용 계산)
  통계 정보(Cardinality, Page 수)로 예상 결과 건수 추정
  인덱스 비용 vs Full Scan 비용 비교 후 선택

낮은 Selectivity도 인덱스가 유용한 경우:
  Composite Index의 앞 컬럼으로 사용
  ORDER BY + 작은 LIMIT
  COUNT(*) 집계 (Covering)

ANALYZE TABLE:
  통계 갱신 → Optimizer 재계산
  대규모 변경 후, Slow Query 급증 시 실행

FORCE INDEX:
  Optimizer가 잘못된 선택 시 강제 지정
  단, 실제 성능 측정 후 사용 (잘못하면 더 느려짐)

인덱스 과잉 방지:
  사용 안 하는 인덱스: 쓰기 성능 저하 + Buffer Pool 낭비
  performance_schema로 인덱스 사용 현황 모니터링
  Cardinality가 낮고 사용 빈도도 낮으면 제거 검토
```

---

## 🤔 생각해볼 문제

**Q1.** `orders` 테이블에서 `status='CANCELLED'`인 건수가 전체의 0.01%(100건)이고, `status='PAID'`는 전체의 40%(40만 건)이다. 동일한 `idx_status` 인덱스에서 두 쿼리의 실행 방식(Full Scan vs Index Scan)이 달라질 수 있는가?

<details>
<summary>해설 보기</summary>

**다를 수 있습니다.** Optimizer는 통계 기반으로 각 조건의 예상 결과 건수를 계산하여 비용을 비교합니다.

- `status='CANCELLED'` (100건, 0.01%): Index Scan → 100 Random I/O << Full Scan → Optimizer가 **인덱스 사용 선택**
- `status='PAID'` (40만 건, 40%): Index Scan → 40만 Random I/O >> Full Scan → Optimizer가 **Full Scan 선택**

InnoDB의 통계는 각 값의 분포를 정확히 알지 못하고 Cardinality(고유값 수)만 알기 때문에, 통계가 거칠게 계산된 경우 `status='CANCELLED'`도 비슷한 비율로 추정하여 Full Scan을 선택할 수 있습니다.

`ANALYZE TABLE`로 통계를 갱신하고 `innodb_stats_persistent_sample_pages` 값을 높이면 Optimizer가 더 정확한 비율을 계산할 수 있습니다. 또는 히스토그램(MySQL 8.0)을 생성하면 컬럼 값별 분포를 더 정확하게 파악합니다.

</details>

---

**Q2.** MySQL 8.0에서 도입된 히스토그램(Histogram)이 기존 Cardinality 통계와 어떻게 다르고, 어떤 경우에 유용한가?

<details>
<summary>해설 보기</summary>

**Cardinality 통계**: 고유값 수(몇 가지 값이 있는가)만 알려줍니다. `status`의 Cardinality=3이면 각 값이 균등하게 1/3씩 분포한다고 가정합니다. 실제 분포가 `PAID=90%, PENDING=9%, CANCELLED=1%`여도 구분 못 합니다.

**히스토그램**: 각 값(또는 범위)의 실제 분포 비율을 저장합니다. `PAID=90%, PENDING=9%, CANCELLED=1%`를 정확히 알고 있어서, `status='CANCELLED'` 조건은 1%로 추정하여 인덱스를 선택하고, `status='PAID'` 조건은 90%로 추정하여 Full Scan을 선택합니다.

```sql
-- 히스토그램 생성
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 100 BUCKETS;

-- 히스토그램 확인
SELECT * FROM information_schema.COLUMN_STATISTICS
WHERE table_name = 'orders' AND column_name = 'status'\G

-- 히스토그램 삭제
ANALYZE TABLE orders DROP HISTOGRAM ON status;
```

**특히 유용한 경우**: Cardinality가 낮지만 값별 분포가 극단적으로 불균등한 컬럼 (예: `status`, `category`, `is_deleted`). 인덱스가 없어도 JOIN 순서 최적화나 필터 순서 결정에 히스토그램 정보가 활용됩니다.

</details>

---

**Q3.** `innodb_stats_auto_recalc = ON` 상태에서 대규모 배치 INSERT(500만 건)를 실행했다. 배치 완료 직후 쿼리가 예상보다 느릴 수 있는 이유는 무엇이고, 해결 방법은?

<details>
<summary>해설 보기</summary>

**이유**: 통계 자동 갱신은 테이블의 1/16 이상 변경 시 **백그라운드**에서 비동기적으로 실행됩니다. 배치 INSERT 직후에는 통계가 아직 이전 값을 반영하고 있을 수 있습니다. 기존 100만 건 → 600만 건이 됐는데 통계는 100만 건 기준이라면:

- Optimizer가 인덱스 범위를 과소 평가 → 잘못된 실행 계획
- 예: Full Scan 비용을 과소 평가하여 비효율적인 인덱스 스캔 선택

**해결 방법**:
```sql
-- 즉시 통계 갱신
ANALYZE TABLE orders;
-- MySQL 8.0: 온라인으로 실행 (잠금 없음)
```

**예방 방법**:
- 배치 스크립트 마지막에 `ANALYZE TABLE` 자동 실행
- `innodb_stats_persistent_sample_pages` 값을 높여 통계 정확도 향상 (갱신 빈도가 줄어 더 안정적)
- 배치가 끝난 후 모니터링: Slow Query Log에서 갑작스런 증가 감지 → `ANALYZE TABLE` 실행

</details>

---

<div align="center">

**[⬅️ 이전: Composite Index](./04-composite-index-column-order.md)** | **[홈으로 🏠](../README.md)** | **[다음: 인덱스를 타지 않는 패턴 ➡️](./06-index-skip-patterns.md)**

</div>
