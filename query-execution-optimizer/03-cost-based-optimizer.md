# Cost-Based Optimizer — 인덱스 선택 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Optimizer는 "비용"을 어떤 단위로 계산하고, 무엇이 1 단위인가?
- `mysql.server_cost`와 `mysql.engine_cost` 테이블의 각 항목은 무엇을 나타내는가?
- 통계 정보가 부정확할 때 Optimizer가 잘못된 실행계획을 선택하는 구체적인 시나리오는?
- SSD vs HDD 환경에서 비용 상수를 조정해야 하는 이유는?
- Optimizer가 인덱스 A와 B 중 어느 것을 선택할지 결정하는 계산 과정은?
- 멀티 테이블 JOIN에서 Join 순서를 탐색하는 알고리즘은?

---

## 🔍 왜 이 개념이 중요한가

### "왜 Optimizer가 이 인덱스 대신 저 인덱스를 선택했나?"

```
실제 사례:
  테이블: orders (100만 건)
  인덱스 2개: idx_user_id, idx_status_created
  
  쿼리: SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC LIMIT 10
  
  예상: idx_user_id 사용 → user_id=1인 Row 찾기
  실제: Optimizer가 idx_status_created 선택!
        → 왜?
  
  이유: 통계 오류로 user_id=1인 Row를 50만 건으로 추정
        50만 건 Double Lookup > idx_status_created 전체 순회
        → Optimizer 입장에서는 올바른 선택 (잘못된 통계 때문)
  
  해결: ANALYZE TABLE → 통계 갱신 → 올바른 인덱스 선택
  
  이것을 이해하려면:
    Optimizer가 비용을 어떻게 계산하는지
    통계 정보가 무엇인지
    알아야 한다
```

---

## 🔬 내부 동작 원리

### 1. 비용 모델 개요

```
MySQL Optimizer의 비용 계산 공식:

Total Cost = Σ (Data Access Cost + CPU Cost)

Data Access Cost:
  Disk Read:  읽어야 하는 Page 수 × io_block_read_cost
  Memory Read: Buffer Pool 히트 시 × memory_block_read_cost

CPU Cost:
  Row 처리:  처리할 Row 수 × row_evaluate_cost
  Key 비교:  키 비교 횟수 × key_compare_cost

비용 단위:
  1.0 = 기준 단위 (임의의 상대적 값)
  절대적인 시간이 아닌 상대적 비교를 위한 가중치

비용 상수 위치:
  mysql.server_cost: CPU/메모리 연산 관련 비용
  mysql.engine_cost: Storage Engine I/O 관련 비용
```

### 2. mysql.engine_cost 테이블

```sql
-- Storage Engine 레벨 비용 상수
SELECT * FROM mysql.engine_cost;

-- 주요 항목:
-- io_block_read_cost:   1.0 (기본)
--   디스크에서 한 Block(Page)을 읽는 비용
--   HDD: 기본값 적절
--   SSD: 더 낮춰야 함 (HDD의 10~20배 빠름 → 비용도 낮춤)
--
-- memory_block_read_cost: 0.25 (기본)
--   Buffer Pool에서 한 Block을 읽는 비용
--   디스크보다 4배 빠름 (io_block_read_cost / 4)

-- io_block_read_cost 해석:
-- Full Table Scan 비용 계산 예시:
--   100만 건, 페이지 당 81건 → 12,346 페이지
--   모두 디스크에서 읽을 때:
--     비용 = 12,346 × io_block_read_cost × 2  (= n_blocks_on_disk * 비용)
--           ≈ 12,346 × 1.0 × 2 = 24,692

-- SSD 환경 최적화:
-- SSD 랜덤 읽기가 HDD Sequential과 비슷하게 빠를 때:
-- io_block_read_cost를 낮추면 Optimizer가 더 많은 경우에 인덱스 선택
UPDATE mysql.engine_cost
SET cost_value = 0.5
WHERE cost_name = 'io_block_read_cost';
FLUSH OPTIMIZER_COSTS;  -- 적용
-- 주의: 이 변경은 모든 DB의 비용 계산에 영향
```

### 3. mysql.server_cost 테이블

```sql
-- Server 레벨 비용 상수
SELECT * FROM mysql.server_cost;

-- 주요 항목:
-- disk_temptable_create_cost:  20.0
--   디스크 임시 테이블 생성 비용 (높음 → 임시 테이블 회피 동기)
-- disk_temptable_row_cost:  0.5
--   디스크 임시 테이블에 Row 추가 비용
-- key_compare_cost:  0.05
--   키 값 비교 1회 비용 (정렬, GROUP BY에서 사용)
-- memory_temptable_create_cost:  1.0
--   메모리 임시 테이블 생성 비용
-- memory_temptable_row_cost:  0.1
--   메모리 임시 테이블에 Row 추가 비용
-- row_evaluate_cost:  0.1
--   WHERE 조건 평가 1회 비용 (Row마다 조건 체크)

-- 비용 계산 예시:
-- SELECT * FROM orders WHERE user_id = 1 (10,000건 결과)
-- 
-- Option A: Full Table Scan (100만 건)
--   Data Access: 12,346 pages × io_block_read_cost = 12,346
--   CPU: 1,000,000 × row_evaluate_cost = 1,000,000 × 0.1 = 100,000
--   Total ≈ 112,346
-- 
-- Option B: Index Scan (idx_user_id)
--   Index Scan: 약 100 pages × 0.25 (메모리 히트 가정) = 25
--   Double Lookup: 10,000 × io_block_read_cost = 10,000
--   CPU: 10,000 × row_evaluate_cost = 1,000
--   Total ≈ 11,025
-- 
-- 선택: B (11,025 < 112,346) → 인덱스 선택
--
-- 만약 결과가 50만 건이면:
--   Option B Total: 25 + 500,000 + 50,000 = 550,025 > 112,346
--   → Full Scan 선택
```

### 4. 실행계획 후보 탐색 알고리즘

```
단일 테이블 Access Method 선택:

후보 목록 생성:
  1. Full Table Scan
  2. 각 인덱스별 Range Scan
  3. 각 인덱스별 Index Scan
  4. (해당시) Loose Index Scan

각 후보의 비용 계산:
  예상 결과 건수 = rows × filtered / 100
  각 접근 방식의 I/O + CPU 비용 계산
  최저 비용 선택

멀티 테이블 JOIN 순서 탐색:

테이블 N개의 가능한 JOIN 순서: N! (팩토리얼)
  2개: 2! = 2
  3개: 3! = 6
  5개: 5! = 120
  8개: 8! = 40,320 ← 탐색 비용이 커지기 시작
  10개: 10! = 3,628,800 ← 불가능

MySQL의 해결책:
  join_reorder_limit (기본: 8):
    ≤ 8개: Exhaustive Search (모든 순서 탐색)
    > 8개: Greedy Search (완전하지 않지만 빠른 탐색)

  Greedy Search:
    1개씩 테이블을 추가하면서 지금까지의 최선 순서 유지
    O(N²) → 훨씬 빠름
    단, 전역 최적을 보장하지 않음

  set optimizer_search_depth = 4;
  → Greedy Search에서 탐색 깊이 제한 (더 빠르지만 더 부정확)
```

### 5. 통계 오류와 잘못된 실행계획 시나리오

```sql
-- 시나리오 1: 대규모 INSERT 후 통계 미갱신

-- 배경: orders 테이블 100만 건, user_id=1은 100건
-- 통계: n_diff_pfx01(idx_user) = 10,000 → 예상 결과 100건

-- 10만 건 INSERT (user_id=1 집중):
INSERT INTO orders (user_id, amount, status, created_at)
SELECT 1, RAND()*10000, 'PAID', NOW()
FROM ... LIMIT 100000;
-- 이제 user_id=1인 Row = 100,100건

-- 통계 미갱신 상태:
-- Optimizer는 여전히 100건으로 추정
-- Index Scan 비용 = 100 (Double Lookup) → 인덱스 선택

EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- rows: 100 ← 통계 기반 추정 (실제는 100,100)
-- 실제 실행: 100,100건의 Double Lookup → 매우 느림

-- 통계 갱신 후:
ANALYZE TABLE orders;
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- rows: 100100 ← 이제 정확함
-- Optimizer: Full Scan이 더 싸다고 판단 → type: ALL 선택 (10만 건이면 10%)

-- 시나리오 2: 값 분포 편중 (히스토그램 없는 경우)
-- status: 'PAID'=90%, 'PENDING'=9%, 'CANCELLED'=1%
-- Cardinality: 3 → 각 값이 1/3씩이라고 가정

EXPLAIN SELECT * FROM orders WHERE status = 'CANCELLED'\G
-- rows ≈ 333,333 (100만 × 1/3) ← 실제는 10,000건
-- 비용 = 333,333 (과대 추정) → Full Scan 선택 (실제는 인덱스가 훨씬 빠름)

-- 히스토그램으로 해결 (MySQL 8.0):
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 10 BUCKETS;
EXPLAIN SELECT * FROM orders WHERE status = 'CANCELLED'\G
-- 이제 'CANCELLED'=1%로 정확히 추정 → rows ≈ 10,000 → 인덱스 선택
```

### 6. InnoDB 비용 관련 내부 통계

```sql
-- Optimizer가 사용하는 통계 정보 확인

-- 테이블/인덱스 통계
SELECT
    t.NAME AS table_name,
    t.N_ROWS AS estimated_rows,
    t.CLUSTERED_INDEX_SIZE AS clustered_pages,
    t.SUM_OF_OTHER_INDEX_SIZES AS secondary_index_pages
FROM information_schema.INNODB_TABLE_STATS t
WHERE t.DATABASE_NAME = 'deep_dive'
  AND t.TABLE_NAME = 'orders';
-- N_ROWS: 예상 Row 수 (통계 기반)
-- CLUSTERED_INDEX_SIZE: Clustered Index Page 수
-- SUM_OF_OTHER_INDEX_SIZES: 모든 Secondary Index Page 수의 합

-- 인덱스별 통계
SELECT
    database_name, table_name, index_name,
    stat_name, stat_value, sample_size, last_update
FROM mysql.innodb_index_stats
WHERE database_name = 'deep_dive'
  AND table_name = 'orders'
ORDER BY index_name, stat_name;
-- n_diff_pfx01: 첫 번째 컬럼 기준 고유 값 수 (Cardinality)
-- n_diff_pfx02: 첫 두 컬럼 조합 고유 값 수
-- n_leaf_pages: Leaf Node Page 수
-- size: 인덱스 전체 Page 수

-- Optimizer가 비용 계산 시 사용하는 공식:
-- 예상 결과 수 = N_ROWS / n_diff_pfx01 (인덱스 첫 컬럼 기준)
-- 예: N_ROWS=1,000,000, n_diff_pfx01=10,000
--     user_id=1 예상 결과 = 1,000,000 / 10,000 = 100건

-- 샘플링 페이지 수 조정:
SHOW VARIABLES LIKE 'innodb_stats_persistent_sample_pages';
-- 기본: 20 pages 샘플링
-- 높이면: 통계 정확도 ↑, 갱신 시간 ↑
SET GLOBAL innodb_stats_persistent_sample_pages = 100;
ANALYZE TABLE orders;  -- 더 정확한 통계 재계산
```

---

## 💻 실전 실험

### 실험 1: 비용 상수 확인 및 Optimizer Trace

```sql
-- 현재 비용 상수 확인
SELECT engine_name, device_type, cost_name, cost_value, last_update
FROM mysql.engine_cost;

SELECT cost_name, cost_value, last_update
FROM mysql.server_cost;

-- Optimizer Trace로 비용 계산 과정 확인
SET SESSION optimizer_trace = "enabled=on";

SELECT * FROM orders WHERE user_id = 1;

SELECT JSON_PRETTY(TRACE) FROM information_schema.OPTIMIZER_TRACE\G
-- "rows_estimation" 섹션에서:
-- {
--   "table": "orders",
--   "rows_estimation": [
--     {
--       "index": "idx_user_id",
--       "range_analysis": {
--         "rows_estimation": 100,
--         "range_scan_alternatives": [
--           {
--             "index": "idx_user_id",
--             "ranges": ["1 <= user_id <= 1"],
--             "index_dives_for_eq_ranges": true,
--             "rowid_ordered": false,
--             "using_mrr": false,
--             "index_only": false,
--             "rows": 100,
--             "cost": 105.26,
--             "chosen": true
--           }
--         ],
--         "analyzing_roworder_intersect": ...
--       }
--     }
--   ]
-- }

SET SESSION optimizer_trace = "enabled=off";
```

### 실험 2: SSD 환경에서 비용 상수 조정 효과

```sql
-- 기본값으로 EXPLAIN
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- rows: N, 특정 접근 방식 선택

-- io_block_read_cost 낮추기 (SSD 환경 시뮬레이션)
UPDATE mysql.engine_cost SET cost_value = 0.25 WHERE cost_name = 'io_block_read_cost';
FLUSH OPTIMIZER_COSTS;

-- 다시 EXPLAIN
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- 인덱스를 더 적극적으로 선택하는지 확인
-- (Full Scan의 I/O 비용이 낮아져서 오히려 Full Scan을 더 선호할 수도 있음)

-- 원복
UPDATE mysql.engine_cost SET cost_value = DEFAULT WHERE cost_name = 'io_block_read_cost';
FLUSH OPTIMIZER_COSTS;
```

### 실험 3: 통계 오류로 잘못된 실행계획 재현

```sql
-- 통계 고정 (자동 갱신 비활성화)
SET GLOBAL innodb_stats_auto_recalc = OFF;

-- 기준 EXPLAIN 확인
ANALYZE TABLE orders;
EXPLAIN SELECT * FROM orders WHERE user_id = 999\G
-- rows: 예상 값 기록

-- 대량 삽입으로 분포 변경
INSERT INTO orders (user_id, amount, status, created_at)
SELECT 999, RAND()*10000, 'PAID', DATE_ADD(NOW(), INTERVAL seq SECOND)
FROM seq_1_to_50000;

-- 통계 미갱신 상태 EXPLAIN
EXPLAIN SELECT * FROM orders WHERE user_id = 999\G
-- rows: 여전히 이전 값 → 실제(50,100)와 큰 차이

EXPLAIN ANALYZE SELECT COUNT(*) FROM orders WHERE user_id = 999\G
-- (estimated rows=100) vs (actual rows=50100) → 통계 오류 명확

-- 통계 갱신 후
ANALYZE TABLE orders;
EXPLAIN SELECT * FROM orders WHERE user_id = 999\G
-- rows: 50100 → 이제 정확, 실행계획 바뀔 수 있음

-- 복원
SET GLOBAL innodb_stats_auto_recalc = ON;
```

### 실험 4: 히스토그램으로 정확도 향상

```sql
-- 히스토그램 없이 (기본 통계)
EXPLAIN SELECT * FROM orders WHERE status = 'CANCELLED'\G
-- rows: N_ROWS / Cardinality(status) = 균등 분포 가정

-- 히스토그램 생성
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 10 BUCKETS;

-- 히스토그램 확인
SELECT
    column_name,
    JSON_PRETTY(histogram)
FROM information_schema.COLUMN_STATISTICS
WHERE table_schema = 'deep_dive'
  AND table_name = 'orders'
  AND column_name = 'status'\G

-- 히스토그램 적용 후 EXPLAIN
EXPLAIN SELECT * FROM orders WHERE status = 'CANCELLED'\G
-- rows: 실제 비율에 맞는 추정값 → 더 정확

-- 히스토그램 삭제
ANALYZE TABLE orders DROP HISTOGRAM ON status;
```

---

## 📊 성능 비교

```
비용 기반 선택의 정확성:

기본 통계 (Cardinality만):
  균등 분포 가정 → 편중 데이터에서 부정확
  예: status='CANCELLED'=1%, 추정=33% → 33배 과대 추정

히스토그램 추가 (MySQL 8.0):
  실제 분포 반영 → 대부분의 경우 정확
  단, 통계 갱신 후 최신 상태 유지 필요

innodb_stats_persistent_sample_pages 증가 (20→100):
  샘플링 페이지 5배 → 통계 정확도 향상
  ANALYZE TABLE 시간: 5배 증가
  자동 갱신 부하: 증가
  적용: 데이터 분포가 불균등하고 정확도가 중요한 테이블
```

---

## ⚖️ 트레이드오프

```
비용 상수 조정 주의사항:

io_block_read_cost 낮춤 (SSD 환경):
  효과: Optimizer가 Index Scan을 더 선호 (Random I/O 비용 상대적 감소)
  주의: 모든 쿼리에 적용 → 의도치 않은 계획 변경 가능
  권장: 변경 전 충분한 테스트 환경 검증

통계 샘플링 증가:
  효과: 더 정확한 통계 → 올바른 실행계획 선택
  비용: ANALYZE TABLE 시간 증가, 자동 갱신 부하 증가
  권장: 데이터 분포가 불규칙하고 중요한 핵심 테이블에만 적용

히스토그램:
  효과: 값 분포 정보 추가 → 정확한 예측
  비용: 저장 공간 (information_schema에 저장)
  갱신: ANALYZE TABLE UPDATE HISTOGRAM을 명시적으로 실행해야 함
        자동 갱신 없음!
  → 데이터 분포가 크게 변경될 때 히스토그램 재생성 필요

Optimizer Switch:
  SET optimizer_switch = 'condition_fanout_filter=off';
  → 특정 최적화 기법 비활성화
  → 특정 쿼리에서 잘못된 결과를 낼 때 임시 조치
  → 근본 해결책이 아닌 임시방편으로만 사용
```

---

## 📌 핵심 정리

```
Cost-Based Optimizer 핵심:

비용 = Data Access Cost + CPU Cost:
  Data: 읽어야 하는 Page 수 × 단가
  CPU: 처리해야 하는 Row 수 × 단가

비용 상수 테이블:
  mysql.engine_cost: I/O 비용 (io_block_read_cost 등)
  mysql.server_cost: CPU/임시 테이블 비용 (row_evaluate_cost 등)
  SSD 환경: io_block_read_cost 낮추면 인덱스를 더 적극 활용

통계 = Optimizer의 눈:
  N_ROWS, Cardinality(n_diff_pfx*)으로 예상 결과 수 추정
  추정이 틀리면 → 잘못된 실행계획 선택
  
통계 갱신:
  ANALYZE TABLE: 즉시 통계 재계산
  innodb_stats_auto_recalc: 1/16 변경 시 자동 갱신 (기본 ON)
  innodb_stats_persistent_sample_pages: 샘플 페이지 수 (기본 20)

히스토그램 (MySQL 8.0):
  ANALYZE TABLE ... UPDATE HISTOGRAM ON col:
  값 분포까지 통계에 포함 → 편중 데이터에서 정확도 크게 향상
  자동 갱신 없음 → 분포 변경 시 수동 재생성 필요

JOIN 순서 탐색:
  ≤ 8개 테이블: Exhaustive Search (완전 탐색)
  > 8개: Greedy Search (빠르지만 최적 보장 안 됨)
  join_reorder_limit 조정 가능
```

---

## 🤔 생각해볼 문제

**Q1.** SSD 서버에서 `io_block_read_cost`를 기본값(1.0)보다 낮추면 항상 성능이 향상되는가?

<details>
<summary>해설 보기</summary>

**반드시 그렇지는 않습니다.** `io_block_read_cost`를 낮추면 Optimizer가 Index Scan을 더 선호하게 됩니다. 하지만 상황에 따라:

1. **인덱스가 실제로 더 빠른 경우**: SSD에서 Random I/O 비용이 낮아졌으므로 인덱스 사용이 합리적 → 성능 향상.

2. **Full Scan이 더 빠른데 인덱스를 선택하는 경우**: 결과가 전체의 30%인 쿼리에서 Full Scan이 Sequential I/O로 더 빠르지만, `io_block_read_cost` 감소로 Index Scan도 싸 보여서 인덱스를 선택할 수 있음 → 오히려 성능 저하.

**실제 권장 접근**:
- 먼저 느린 쿼리를 EXPLAIN ANALYZE로 분석
- 통계 오류인지, 잘못된 비용 상수인지 구분
- 비용 상수 변경보다 통계 정확도 향상(ANALYZE TABLE, 히스토그램)이 먼저
- 비용 상수 변경 시 충분한 테스트 환경에서 검증 후 적용

</details>

---

**Q2.** 동일한 테이블에 인덱스가 5개 있고 특정 쿼리에서 Optimizer가 가장 비효율적인 인덱스를 선택했다. 이 상황에서 `FORCE INDEX`와 `ANALYZE TABLE` 중 어느 것을 먼저 시도해야 하는가?

<details>
<summary>해설 보기</summary>

**`ANALYZE TABLE`을 먼저 시도해야 합니다.** 이유:

1. **근본 원인 파악**: Optimizer가 잘못된 선택을 하는 주된 이유는 통계 오류입니다. ANALYZE TABLE로 통계를 갱신하면 대부분의 경우 올바른 인덱스를 자동으로 선택합니다.

2. **FORCE INDEX의 문제점**:
   - 하드코딩된 힌트 → 코드/스키마 변경 시 유지 관리 어려움
   - 데이터가 변화할 때 FORCE INDEX가 최적이 아닐 수 있음
   - JPA/ORM에서 적용하려면 네이티브 쿼리로 변경 필요

3. **순서**:
   1. ANALYZE TABLE → EXPLAIN 재확인
   2. 여전히 잘못된 선택이면 히스토그램 생성
   3. 통계가 정확한데도 잘못된 선택이면 FORCE INDEX (임시 조치)
   4. 장기적으로는 인덱스 재설계 또는 쿼리 재작성

FORCE INDEX는 "Optimizer가 통계가 정확한 상태에서도 특정 이유로 잘못된 선택을 하는 경우"에만 사용하고, 항상 주석으로 이유를 명시해야 합니다.

</details>

---

**Q3.** `join_reorder_limit = 8`일 때 10개 테이블 JOIN 쿼리를 작성했다. Optimizer가 최적 JOIN 순서를 찾지 못할 가능성이 있다. 이를 어떻게 감지하고 대응할 수 있는가?

<details>
<summary>해설 보기</summary>

**감지 방법**:
1. `EXPLAIN FORMAT=TREE`로 Optimizer가 선택한 JOIN 순서 확인
2. `STRAIGHT_JOIN`으로 다른 순서를 강제하여 성능 비교
3. `optimizer_trace`로 탐색 과정 확인 (Greedy Search 사용 여부)

```sql
-- Greedy Search 사용 여부 확인 (optimizer_trace에서)
-- "greedy_search_needed": true 이면 Greedy Search 사용
```

**대응 방법**:
1. **서브쿼리로 분리**: 큰 JOIN을 2~3개의 서브쿼리로 분리하여 각 최적화
2. **STRAIGHT_JOIN**: JOIN 순서를 명시적으로 지정
   ```sql
   SELECT STRAIGHT_JOIN o.*, u.name, p.title
   FROM orders o
   JOIN users u ON o.user_id = u.id  -- Driving
   JOIN products p ON o.product_id = p.id  -- 두 번째
   ...
   ```
3. **join_reorder_limit 증가**: `SET SESSION join_reorder_limit = 10;`
   단, 10! = 3,628,800 탐색이 오래 걸릴 수 있음
4. **인덱스 최적화**: 각 JOIN 조건에 적절한 인덱스가 있으면 JOIN 순서의 영향을 줄일 수 있음

실무에서 10개 이상의 테이블 JOIN은 설계 재검토 신호입니다. 집계 전용 뷰, 비정규화, 또는 쿼리 분리를 고려하세요.

</details>

---

<div align="center">

**[⬅️ 이전: EXPLAIN 읽는 법](./02-explain-analyze.md)** | **[홈으로 🏠](../README.md)** | **[다음: Join 알고리즘 ➡️](./04-join-algorithms.md)**

</div>
