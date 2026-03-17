# Statistics — Optimizer의 판단 근거

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `information_schema.INNODB_TABLE_STATS`의 `n_rows`는 어떻게 수집되고 얼마나 정확한가?
- `innodb_stats_auto_recalc`는 언제 자동 갱신을 트리거하는가?
- 대량 데이터 변경 후 실행계획이 갑자기 바뀌는 근본 원인은?
- `innodb_stats_persistent` vs `innodb_stats_transient`의 차이와 각각의 적합한 상황은?
- 통계 샘플링이 너무 적으면 발생하는 구체적인 문제는?
- 히스토그램은 기존 통계의 어떤 한계를 보완하는가?

---

## 🔍 왜 이 개념이 중요한가

### 통계 = Optimizer가 세상을 보는 눈

```
Optimizer는 실제 데이터를 모르고 통계 정보만 본다:

테이블: orders (실제 1,000,000행)
통계: n_rows = 800,000 (통계 오래됨)

WHERE user_id = 1 예상 결과:
  실제: 50건
  통계 기반: 800,000 / Cardinality(10,000) = 80건

인덱스 비용 추정:
  80건 × Double Lookup ≈ 240 비용
vs Full Scan:
  800,000 / 81 = 9,876 페이지 × 1.0 = 9,876 비용

→ 인덱스 선택! (올바른 결론)

하지만 통계가 심하게 틀리면:
통계: n_rows = 100,000 (실제 1,000,000인데)
WHERE user_id = 1 예상: 100,000 / 10,000 = 10건

비용 추정:
  인덱스: 10 × 3 = 30 비용 (과소 추정)
vs Full Scan: 100,000 / 81 = 1,234 비용 (과소 추정)

→ 비율은 같아도 다른 쿼리와 조합 시 잘못된 판단 가능

통계를 이해하면:
  갑작스러운 성능 저하의 원인을 파악할 수 있다
  ANALYZE TABLE이 왜 필요한지 알 수 있다
  통계 설정을 환경에 맞게 조정할 수 있다
```

---

## 🔬 내부 동작 원리

### 1. InnoDB 통계 수집 메커니즘

```
InnoDB 통계 수집 방법: 샘플링(Sampling)

전수조사 대신 샘플링 이유:
  전수 조사: 정확하지만 ANALYZE TABLE 시간 = 테이블 크기에 비례
  샘플링: 빠르지만 부정확 가능
  → 실용적 타협

샘플링 과정:
  innodb_stats_persistent_sample_pages = 20 (기본)
  → B+Tree의 Leaf Page 중 무작위로 20개 샘플링
  → 20개 Page의 Row 통계로 전체 추정

추정 방법 (n_rows 계산):
  sampled_rows / 20 = 평균 Page당 Row 수
  n_rows = 평균 Page당 Row 수 × 전체 Leaf Page 수 (n_leaf_pages)
  
  예시:
    20 Page 샘플링 → 1,600 rows 확인
    평균 80 rows/page
    전체 n_leaf_pages = 12,346
    n_rows 추정 = 80 × 12,346 = 987,680

Cardinality 추정 (n_diff_pfx01):
  20개 샘플 Page에서 고유 값 수 확인
  고유 값 비율 × n_rows = Cardinality
  
  예시:
    20 Page에서 user_id 고유 값 200개
    비율: 200/1,600 = 12.5%
    Cardinality = 12.5% × 987,680 ≈ 123,460
  
  오차 발생:
    실제 고유 user_id = 100,000
    추정 = 123,460 → 23% 과대 추정
    → WHERE user_id=1 예상 결과 = 987,680/123,460 ≈ 8건 (실제 10건)
    → 근접한 추정이지만 편중 데이터에서는 큰 오차 가능
```

### 2. 통계 저장 방식: Persistent vs Transient

```sql
-- innodb_stats_persistent (기본: ON)
-- 통계를 mysql.innodb_table_stats / mysql.innodb_index_stats에 영구 저장
-- → MySQL 재시작 후에도 통계 유지
-- → ANALYZE TABLE 실행 시 업데이트

SHOW VARIABLES LIKE 'innodb_stats_persistent';
-- Value: ON (기본, 권장)

-- 테이블별 설정도 가능:
CREATE TABLE orders (...)
    STATS_PERSISTENT = 1  -- 이 테이블은 영구 통계
    STATS_AUTO_RECALC = 1  -- 자동 갱신 활성화
    STATS_SAMPLE_PAGES = 50;  -- 이 테이블은 50페이지 샘플링

-- 영구 통계 확인:
SELECT * FROM mysql.innodb_table_stats
WHERE database_name = 'deep_dive' AND table_name = 'orders'\G
-- last_update: 마지막 통계 갱신 시간
-- n_rows: 추정 Row 수
-- clustered_index_size: Clustered Index Page 수
-- sum_of_other_index_sizes: Secondary Index 전체 Page 수

SELECT * FROM mysql.innodb_index_stats
WHERE database_name = 'deep_dive' AND table_name = 'orders'\G
-- n_diff_pfx01, n_diff_pfx02, ...: 각 컬럼 조합의 고유값 추정 수
-- n_leaf_pages: Leaf Page 수
-- size: 전체 Page 수

-- innodb_stats_persistent = OFF 시:
-- 통계를 메모리에만 유지
-- MySQL 재시작 시 통계 소실 → 재시작 후 첫 쿼리가 느릴 수 있음
-- 소규모 임시 테이블에 적합
```

### 3. 자동 통계 갱신 조건

```sql
-- innodb_stats_auto_recalc = ON (기본)
-- 조건: 테이블의 1/10 이상 Row가 변경됐을 때 백그라운드에서 갱신
-- (MySQL 문서에는 1/16이라고도 나오지만 실제는 약 10% 임계값)

-- 자동 갱신의 특성:
-- ① 비동기: 백그라운드에서 실행 → 즉각적이지 않음
-- ② 지연 가능: 서버 부하에 따라 갱신이 지연될 수 있음
-- ③ 샘플링: 전수조사가 아닌 샘플링 → 항상 정확하지 않음

-- 자동 갱신이 문제가 되는 경우:
-- 대규모 배치 INSERT (500만 건 추가) →
-- 1/10 임계값 도달 → 자동 갱신 트리거 →
-- 갱신 중에도 쿼리 실행 → 갱신 중인 통계로 실행계획 불안정

-- 안정적인 통계 관리 전략:
-- 1. 배치 작업 후 명시적 ANALYZE TABLE
-- 2. 중요 테이블은 innodb_stats_auto_recalc = 0으로 자동 갱신 비활성화
--    → 배치 스크립트에서 명시적으로 ANALYZE TABLE 실행

-- 특정 테이블 자동 갱신 비활성화:
ALTER TABLE orders STATS_AUTO_RECALC = 0;
-- 이후 배치 스크립트 마지막에:
ANALYZE TABLE orders;

-- 전역 비활성화 (주의!):
SET GLOBAL innodb_stats_auto_recalc = 0;
-- 모든 테이블의 자동 갱신 중단 → 반드시 수동 관리 체계 필요
```

### 4. 대량 데이터 변경 후 실행계획 변화 시나리오

```
실제 발생 시나리오:

타임라인:
  D-30: orders 100만 건 (user_id=1 → 100건)
  D-0:  배치로 user_id=1 관련 100만 건 INSERT
        → orders 200만 건 (user_id=1 → 100만 건)

D-0 이전 통계:
  n_rows ≈ 1,000,000
  n_diff_pfx01(idx_user) ≈ 10,000
  user_id=1 예상 결과 = 1,000,000 / 10,000 = 100건
  → 인덱스 비용: 100 × 3 = 300 < Full Scan: 12,346
  → 인덱스 선택

D-0 이후 자동 갱신 전 (문제 구간):
  n_rows ≈ 1,000,000 (여전히 이전 값)
  WHERE user_id = 1 예상 결과 = 100건 (실제: 100만)
  → 인덱스 선택 → 실제 100만 건 Double Lookup
  → 쿼리 시간 수분으로 급증

자동 갱신 후:
  n_rows ≈ 2,000,000
  n_diff_pfx01 ≈ 10,001 (user_id 하나 더 추가는 아님, 여전히 10,000)
  user_id=1 예상 = 2,000,000 / 10,000 = 200건 (실제 100만과 큰 차이)
  → 실행계획은 개선되지만 여전히 부정확

히스토그램으로 근본 해결:
  ANALYZE TABLE orders UPDATE HISTOGRAM ON user_id WITH 1024 BUCKETS;
  → user_id=1의 실제 비율(50%) 반영
  → 예상 결과 = 2,000,000 × 50% = 1,000,000건
  → Full Scan 선택 (올바른 판단)
```

### 5. 통계 정보와 쿼리 성능 모니터링

```sql
-- 통계 신선도 모니터링
SELECT
    database_name,
    table_name,
    last_update,
    n_rows,
    TIMESTAMPDIFF(HOUR, last_update, NOW()) AS hours_since_update
FROM mysql.innodb_table_stats
WHERE database_name = 'deep_dive'
ORDER BY last_update;

-- 통계가 오래된 테이블 + Slow Query 연관 분석
-- Step 1: Slow Query Log에서 느린 테이블 확인
-- SELECT ... FROM orders ... → Slow Query
-- Step 2: 통계 갱신 시간 확인
-- hours_since_update가 높은 orders → ANALYZE TABLE

-- 실행계획 변경 감지:
-- Performance Schema의 events_statements_summary_by_digest:
SELECT
    SCHEMA_NAME,
    DIGEST_TEXT,
    COUNT_STAR AS exec_count,
    AVG_TIMER_WAIT / 1e9 AS avg_ms,
    MAX_TIMER_WAIT / 1e9 AS max_ms,
    SUM_ROWS_EXAMINED / COUNT_STAR AS avg_rows_examined,
    LAST_SEEN
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'deep_dive'
  AND AVG_TIMER_WAIT > 1e9  -- 1ms 이상
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 20;

-- avg_rows_examined이 갑자기 증가하면 통계 문제 가능성
-- ANALYZE TABLE → 재확인
```

---

## 💻 실전 실험

### 실험 1: 통계 샘플링과 정확도

```sql
-- 기준 통계 확인
ANALYZE TABLE orders;  -- 최신 통계로 시작
SELECT n_rows, clustered_index_size
FROM mysql.innodb_table_stats
WHERE database_name = 'deep_dive' AND table_name = 'orders';

-- 실제 Row 수와 비교
SELECT COUNT(*) AS actual_rows FROM orders;

-- 차이 비율: |n_rows - actual_rows| / actual_rows × 100 = 오차율
-- 목표: 오차율 5% 이하

-- 샘플 페이지 수 조정 실험
SET GLOBAL innodb_stats_persistent_sample_pages = 200;
ANALYZE TABLE orders;
SELECT n_rows FROM mysql.innodb_table_stats
WHERE database_name = 'deep_dive' AND table_name = 'orders';
-- 이전보다 actual_rows에 더 가까운지 확인

-- 원복
SET GLOBAL innodb_stats_persistent_sample_pages = 20;
```

### 실험 2: 자동 갱신 시점 관찰

```sql
-- 초기 통계
ANALYZE TABLE orders;
SELECT last_update, n_rows
FROM mysql.innodb_table_stats
WHERE database_name = 'deep_dive' AND table_name = 'orders';

-- 10% 이상 변경 (10만 건 테이블에서 1만 건 이상 INSERT)
INSERT INTO orders (user_id, amount, status, created_at)
SELECT FLOOR(RAND()*10000)+1, ROUND(RAND()*10000,2), 'PAID', NOW()
FROM seq_1_to_11000;  -- 11,000건 (10% 초과)

-- 잠시 대기 후 통계 갱신 여부 확인
SELECT SLEEP(2);
SELECT last_update, n_rows
FROM mysql.innodb_table_stats
WHERE database_name = 'deep_dive' AND table_name = 'orders';
-- last_update가 변경됐으면 자동 갱신 발생!
-- 변경 안 됐으면 백그라운드에서 아직 처리 중 또는 조건 미충족

-- 명시적 갱신 후 비교
ANALYZE TABLE orders;
SELECT last_update, n_rows
FROM mysql.innodb_table_stats
WHERE database_name = 'deep_dive' AND table_name = 'orders';
```

### 실험 3: 히스토그램 효과 측정

```sql
-- status 분포 확인 (편중된 데이터 가정)
SELECT status, COUNT(*), COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS pct
FROM orders GROUP BY status;
-- PAID: 85%, PENDING: 14%, CANCELLED: 1%

-- 히스토그램 없이 EXPLAIN (균등 분포로 가정)
EXPLAIN SELECT * FROM orders WHERE status = 'CANCELLED'\G
-- rows: n_rows / Cardinality(3) ≈ 전체의 33%

-- 히스토그램 생성
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 10 BUCKETS;

-- 히스토그램 확인
SELECT column_name, JSON_PRETTY(histogram) AS hist_json
FROM information_schema.COLUMN_STATISTICS
WHERE table_schema = 'deep_dive'
  AND table_name = 'orders'
  AND column_name = 'status'\G
-- buckets: [["CANCELLED", 0.01], ["PAID", 0.86], ["PENDING", 1.0]]

-- 히스토그램 적용 후 EXPLAIN
EXPLAIN SELECT * FROM orders WHERE status = 'CANCELLED'\G
-- rows: n_rows × 0.01 ≈ 실제 1%에 맞는 추정값
-- → 인덱스 선택 (이전에는 33%로 보고 Full Scan 선택했을 수 있음)

EXPLAIN SELECT * FROM orders WHERE status = 'PAID'\G
-- rows: n_rows × 0.85 ≈ 실제 85%에 맞는 추정값
-- → Full Scan 선택 (이전에도 Full Scan이었겠지만 이제 정확한 이유)
```

---

## 📊 성능 비교

```
통계 설정별 정확도:

innodb_stats_persistent_sample_pages = 20 (기본):
  100만 건 테이블 → 20 Page 샘플 = 1,620 Row (0.16%)
  오차 범위: 수 % ~ 수십 % (데이터 분포에 따라)
  ANALYZE TABLE 시간: 빠름

innodb_stats_persistent_sample_pages = 100:
  같은 테이블 → 100 Page = 8,100 Row (0.81%)
  오차 범위: 5배 더 정확 (이론적)
  ANALYZE TABLE 시간: 5배 더 느림

히스토그램 추가:
  Cardinality만으로는 측정 불가능한 값 분포 정보 추가
  편중 데이터에서 극적인 정확도 향상
  갱신: 수동으로만 (배치 후 명시적 실행 필요)

InnoDB vs 기타 통계 방식:
  InnoDB: 샘플링 (빠름, 근사치)
  PostgreSQL: 테이블 10% 전수 조사 + 히스토그램 자동 갱신 → 더 정확
  MySQL: 샘플링 + 수동 히스토그램 → PostgreSQL보다 통계 관리 더 많이 필요
```

---

## ⚖️ 트레이드오프

```
통계 정확도 vs 갱신 비용:

높은 정확도가 필요한 경우:
  데이터 분포가 불균등 (status, category 등)
  데이터 변화가 빈번하고 실행계획 변화에 민감
  → innodb_stats_persistent_sample_pages 높이기 + 히스토그램

빠른 갱신이 중요한 경우:
  테이블이 매우 크고 ANALYZE TABLE이 오래 걸림
  → 기본 샘플 수 유지, 주기적 ANALYZE

자동 갱신 비활성화가 적합한 경우:
  대규모 배치 처리로 통계가 자주 무효화됨
  → STATS_AUTO_RECALC = 0 + 배치 후 수동 ANALYZE
  장점: 배치 중 불안정한 통계로 인한 실행계획 변동 방지

InnoDB 통계의 한계:
  단일 컬럼 통계만 수집 (다중 컬럼 상관관계 없음)
  WHERE a = 1 AND b = 2일 때 상관관계 고려 안 됨
  → a와 b가 높은 상관관계가 있어도 독립적으로 계산
  → 이 한계를 히스토그램도 완전히 해결하지 못함
  → 복잡한 조건의 filtered 값이 부정확할 수 있음
```

---

## 📌 핵심 정리

```
Statistics 핵심:

통계 수집 방법:
  샘플링 (기본 20 Pages)
  n_rows = 평균 rows/page × n_leaf_pages
  n_diff_pfx* = 샘플 내 고유값 비율 × n_rows

통계 저장:
  mysql.innodb_table_stats (테이블 통계)
  mysql.innodb_index_stats (인덱스 통계)
  last_update로 신선도 확인 가능

자동 갱신 (innodb_stats_auto_recalc):
  1/10 이상 변경 시 비동기 갱신
  즉각적이지 않음 → 대규모 변경 후 명시적 ANALYZE TABLE 권장

히스토그램 (MySQL 8.0):
  값별 실제 분포 정보 추가
  편중 데이터의 Cardinality 한계 보완
  자동 갱신 없음 → 분포 변경 시 수동 재생성

관리 권장사항:
  대규모 배치 작업 후: ANALYZE TABLE
  중요 테이블: innodb_stats_persistent_sample_pages 높이기
  편중 분포 컬럼: 히스토그램 생성
  주기적 모니터링: last_update + Slow Query 연관 분석
```

---

## 🤔 생각해볼 문제

**Q1.** `ANALYZE TABLE`은 테이블에 잠금을 거는가? 운영 중인 서비스에서 실행해도 안전한가?

<details>
<summary>해설 보기</summary>

**MySQL 8.0 (InnoDB)**: `ANALYZE TABLE`은 읽기 잠금(Shared Lock)만 사용합니다. 실행 중에 SELECT는 허용되지만 DML(INSERT/UPDATE/DELETE)은 잠깐 블로킹될 수 있습니다.

**실제 동작**:
- 내부적으로 Online DDL과 유사하게 동작
- InnoDB 테이블: 짧은 메타데이터 잠금(MDL) 후 백그라운드에서 샘플링
- MyISAM: 전체 테이블 읽기 잠금 (InnoDB와 다름)

**운영 중 실행 시 주의사항**:
1. **대용량 테이블**: 샘플링에 수 초 ~ 수십 초 소요. 이 시간 동안 MDL 유지 → 긴 트랜잭션과 충돌 가능.
2. **최적 실행 시간**: 부하가 낮은 새벽, 또는 읽기 전용 점검 시간.
3. **대안**: `pt-online-schema-change` 같은 도구로 무중단 통계 갱신 (단, ANALYZE TABLE 자체가 이미 온라인임).

**실제 권장**:
- 소규모 테이블: 언제든 실행 가능
- 대용량 핵심 테이블: 부하 낮은 시간대에 실행, 또는 Read Replica에서 먼저 테스트

</details>

---

**Q2.** `innodb_stats_persistent = ON`인 상태에서 MySQL 서버를 재시작했다. 통계 정보는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**`innodb_stats_persistent = ON`**: 통계가 `mysql.innodb_table_stats`와 `mysql.innodb_index_stats` 테이블에 **영구 저장**됩니다. MySQL 재시작 후에도 이 테이블에서 통계를 읽으므로 **통계가 유지**됩니다. 재시작 직후 첫 쿼리부터 이전 통계를 기반으로 실행계획을 선택합니다.

**`innodb_stats_persistent = OFF` (Transient Statistics)**: 통계가 메모리에만 유지됩니다. MySQL 재시작 시 통계 소실. 재시작 후 첫 쿼리 실행 시 통계를 새로 수집합니다. 이 수집 과정에서 일시적 성능 저하 가능.

**실용적 의미**: Persistent 통계가 있어도 재시작 후 데이터가 크게 변경됐다면 ANALYZE TABLE을 실행하는 것이 좋습니다. 오래된 Persistent 통계보다 최신 통계가 더 정확합니다.

</details>

---

**Q3.** `innodb_stats_persistent_sample_pages`를 20에서 200으로 높이면 통계가 10배 더 정확해지는가?

<details>
<summary>해설 보기</summary>

**이론적으로 정확도가 높아지지만, 반드시 10배는 아닙니다.**

통계 정확도는 샘플 비율뿐 아니라 **데이터 분포의 균일성**에 크게 의존합니다:

1. **균일한 분포**: 20페이지 샘플이 이미 충분히 대표적 → 200페이지로 늘려도 큰 개선 없음.
2. **편중된 분포**: 특정 값이 특정 페이지에 집중 → 샘플이 많을수록 정확도 향상. 하지만 여전히 완벽하지 않음.

**수학적 관점**: 샘플 크기를 n배 늘리면 통계의 표준 오차는 √n배 감소합니다. 20 → 200 (10배 증가) → 오차는 약 √10 ≈ 3.16배 감소. 10배 정확해지는 것이 아닙니다.

**히스토그램이 더 효과적인 경우**: 특정 값의 분포가 Cardinality 통계로 잡히지 않는 편중 데이터는 샘플 수를 아무리 늘려도 한계가 있습니다. 히스토그램이 직접 분포를 측정하므로 이런 경우에 훨씬 효과적입니다.

**비용 고려**: 샘플 10배 증가 → ANALYZE TABLE 시간도 약 10배 증가. ROI를 고려하여 적절한 값을 선택해야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Join 알고리즘](./04-join-algorithms.md)** | **[홈으로 🏠](../README.md)** | **[다음: 힌트와 강제 인덱스 ➡️](./06-hints-force-index.md)**

</div>
