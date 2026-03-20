# Slow Query 분석 방법론

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `long_query_time`을 어떤 기준으로 초기값을 설정하고, 이후 어떻게 낮춰가는가?
- `mysqldumpslow -s t`와 `-s at`의 차이는 무엇이고 어떤 상황에서 각각 사용하는가?
- `events_statements_summary_by_digest`의 `examine_send_ratio`가 높으면 무엇을 의심하는가?
- Slow Query Log가 기록하는 `Rows_examined`와 `Rows_sent`의 차이가 성능 문제를 드러내는 원리는?
- `log_queries_not_using_indexes`를 ON으로 설정했을 때 로그가 폭증하는 이유와 억제 방법은?
- Slow Query 발견 후 EXPLAIN → EXPLAIN ANALYZE → Handler 변수로 이어지는 진단 워크플로우는?

---

## 🔍 왜 이 개념이 중요한가

### "서버가 갑자기 느려졌다" — 원인이 무엇인지 모르면 해결할 수 없다

```
실제 상황:
  CPU 사용률 정상, 메모리 정상, 네트워크 정상
  하지만 API 응답 시간 P99이 200ms → 3초로 증가
  
  어디서부터 시작해야 하는가?

Slow Query Log 없이:
  "코드 어딘가에 문제가 있겠지" → 무작위 코드 리뷰
  또는 "DB 서버를 스케일 업하자" → 근본 해결 안 됨
  또는 "APM에 이상한 것 없는데..." → 쿼리 레벨 분석 불가

Slow Query Log + performance_schema 있으면:
  1분 안에: "orders 테이블의 status 조건 쿼리가
             평균 2.8초, 초당 500번 실행 중"
  추가 2분: EXPLAIN → type:ALL, key:NULL → 인덱스 없음
  해결: CREATE INDEX idx_status ON orders(status);
  결과: P99 200ms로 복귀

이것이 가능한 이유:
  `events_statements_summary_by_digest`:
    어떤 쿼리 패턴이 → 총 얼마의 시간을 → 몇 번 사용했는지
    → "어디서 가장 많은 시간을 쓰는가" 즉시 파악
```

---

## 😱 잘못된 이해

### Before: "Slow Query Log = long_query_time 이상인 쿼리만 문제"

```
잘못된 믿음 1:
  "long_query_time = 1초면 0.9초짜리 쿼리는 괜찮다"
  
  실제:
    0.5초 쿼리 × 초당 1000번 = 총 500초/초 소비
    1분간 DB가 이 쿼리에만 사용하는 시간 = 500 × 60 = 30,000초
    (병렬 처리로 가능하지만 I/O 경합, Lock 경합은 별개)
    
    examine_send_ratio = 100,000 / 10 = 10,000
    (10만 행을 읽어서 10행 반환 → 인덱스 미사용)
    → 0.5초지만 인덱스 추가 시 0.001초로 500배 개선 가능

잘못된 믿음 2:
  "log_queries_not_using_indexes = ON이면 모든 문제 쿼리를 잡는다"
  
  실제:
    인덱스 없는 작은 테이블 조회도 기록됨
    → 수천 건짜리 내부 테이블을 매초 조회하면 로그 폭증
    → 정작 중요한 쿼리가 로그에 묻힘
    → log_throttle_queries_not_using_indexes 설정 필요

잘못된 믿음 3:
  "Slow Query가 없으면 DB는 최적 상태다"
  
  실제:
    long_query_time = 1초이면 0.9초 쿼리는 기록 안 됨
    "느리지 않은 쿼리" × "엄청난 호출 빈도" = 실제 문제
    performance_schema의 SUM_TIMER_WAIT 기준으로 봐야 함
```

---

## ✨ 올바른 이해

### After: Slow Query 분석 = "총 비용이 가장 큰 쿼리"를 찾는 과정

```
핵심 지표 두 가지:

개별 쿼리 성능:
  avg_timer_wait (평균 실행 시간)
  max_timer_wait (최대 실행 시간)
  → 하나의 실행이 느린 것 파악

총 시스템 영향:
  sum_timer_wait (총 소비 시간) = avg × count
  → 서비스 전체에서 가장 많은 시간을 차지하는 쿼리
  → 이것이 실제 "가장 개선 효과가 큰 쿼리"

우선순위 공식:
  개선 효과 ≈ sum_timer_wait (총 소비 시간이 클수록)
  개선 용이성 ≈ examine_send_ratio가 클수록 쉬운 개선

분석 도구 선택:
  Slow Query Log + mysqldumpslow:
    특정 임계값 초과 쿼리만 → 빠른 문제 쿼리 파악
    파일 기반 → 과거 기록 보존 가능
  
  performance_schema.events_statements_summary_by_digest:
    모든 쿼리의 누적 통계 → 전체 그림 파악
    임계값 없음 → 0.5초 쿼리도 포착
    실시간 → 현재 상태 즉시 확인
    단점: 재시작 시 초기화
  
  두 도구를 함께 사용하는 것이 완전한 분석
```

---

## 🔬 내부 동작 원리

### 1. Slow Query Log 설정과 형식

```sql
-- 현재 설정 확인
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'log_queries_not_using_indexes';

-- 런타임 활성화 (재시작 불필요)
SET GLOBAL slow_query_log = ON;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
SET GLOBAL long_query_time = 1.0;

-- long_query_time 선택 전략:
-- 1단계 (초기): 3.0초 → 확실한 문제 쿼리만
-- 2단계: 1.0초 → 서비스에 영향을 주는 쿼리
-- 3단계: 0.5초 → 조금 더 세밀하게
-- 4단계: 0.1초 → 엄격한 성능 요구사항 (로그 급증 주의)

-- 인덱스 미사용 쿼리 기록
SET GLOBAL log_queries_not_using_indexes = ON;
-- 로그 폭증 억제 (분당 최대 10개)
SET GLOBAL log_throttle_queries_not_using_indexes = 10;

-- 영구 설정 (my.cnf):
-- [mysqld]
-- slow_query_log = 1
-- slow_query_log_file = /var/log/mysql/slow.log
-- long_query_time = 1
-- log_queries_not_using_indexes = 1
-- log_throttle_queries_not_using_indexes = 10
-- log_slow_extra = 1        ← MySQL 8.0: 추가 정보 기록

-- Slow Query Log 형식:
-- # Time: 2024-03-15T14:23:45.678910Z
-- # User@Host: app[app] @ 10.0.0.1 []  Id: 12345
-- # Query_time: 3.456789  Lock_time: 0.000234
-- # Rows_sent: 10  Rows_examined: 1000000
-- # Rows_affected: 0  Bytes_sent: 2048
-- SET timestamp=1710509025;
-- SELECT * FROM orders WHERE status = 'PAID';

-- 핵심 항목 해석:
-- Query_time: 전체 실행 시간 (Lock_time 포함)
-- Lock_time: Lock 획득을 기다린 시간
-- Rows_sent: 클라이언트에 반환한 행 수
-- Rows_examined: 스캔한 총 행 수
-- Rows_examined / Rows_sent = examine_send_ratio
--   >> 1이면 많이 읽고 조금 반환 → 인덱스 미사용 가능성
--   = 1이면 읽은 것 모두 반환 → 효율적
```

### 2. mysqldumpslow 집계 방법

```bash
# 기본 사용법
mysqldumpslow [옵션] /var/log/mysql/slow.log

# 정렬 기준 (-s 옵션):
# t  (sum time)   : 총 실행 시간 기준 → "가장 많은 DB 시간을 소비하는 쿼리"
# at (avg time)   : 평균 실행 시간 기준 → "개별 실행이 가장 느린 쿼리"
# c  (count)      : 호출 횟수 기준 → "가장 자주 실행되는 쿼리"
# l  (lock time)  : Lock 대기 시간 기준 → "Lock 경합이 심한 쿼리"
# r  (rows sent)  : 반환 행 수 기준
# ar (avg rows)   : 평균 반환 행 수 기준

# 주요 분석 커맨드:

# 1. 총 시간 기준 상위 20개 (가장 먼저 볼 것)
mysqldumpslow -s t -t 20 /var/log/mysql/slow.log

# 2. 평균 시간 기준 상위 10개 (개별 쿼리 최적화)
mysqldumpslow -s at -t 10 /var/log/mysql/slow.log

# 3. 빈도 기준 상위 10개 (자주 호출되는 것 최적화)
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log

# 출력 예시 해석:
# Count: 5432  Time=4.32s (23495s)  Lock=0.00s (0s)  Rows=1000.0 (5432000)
# SELECT * FROM orders WHERE user_id = N
#
# Count: 5432번 호출
# Time=4.32s: 평균 4.32초
# (23495s): 총 23,495초 = 6.5시간 (!)
# Lock=0.00s: Lock 대기 거의 없음
# Rows=1000.0: 평균 1000행 반환
# (5432000): 총 543만 행 반환
# 파라미터 N으로 추상화 → 패턴별 집계

# pt-query-digest (더 상세한 분석):
pt-query-digest /var/log/mysql/slow.log > analysis_report.txt
# 설치: apt install percona-toolkit
# 제공: 실행계획, P95/P99 응답시간, 시간대별 분포
```

### 3. performance_schema로 실시간 분석

```sql
-- 총 실행 시간 기준 상위 쿼리 (가장 중요한 쿼리)
SELECT
    LEFT(DIGEST_TEXT, 100) AS query_pattern,
    COUNT_STAR AS exec_count,
    ROUND(AVG_TIMER_WAIT / 1e9, 2) AS avg_ms,
    ROUND(MAX_TIMER_WAIT / 1e9, 2) AS max_ms,
    ROUND(SUM_TIMER_WAIT / 1e9, 2) AS total_ms,
    ROUND(SUM_ROWS_EXAMINED / NULLIF(COUNT_STAR, 0), 0) AS avg_rows_examined,
    ROUND(SUM_ROWS_SENT / NULLIF(COUNT_STAR, 0), 0) AS avg_rows_sent,
    ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0), 0) AS examine_send_ratio,
    SUM_NO_INDEX_USED AS no_index_count
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = DATABASE()
  AND COUNT_STAR > 10          -- 최소 10번 이상 실행된 것만
ORDER BY SUM_TIMER_WAIT DESC   -- 총 시간 기준 정렬
LIMIT 20;

-- examine_send_ratio 해석:
-- 1: 최적 (읽은 것 모두 반환)
-- 10: 읽은 것의 1/10만 반환 → 비효율 (필터링 많음)
-- 1000+: 심각 → 인덱스 미사용 또는 극도로 나쁜 쿼리

-- 인덱스 미사용 쿼리 찾기
SELECT
    LEFT(DIGEST_TEXT, 120) AS query_pattern,
    COUNT_STAR,
    SUM_NO_INDEX_USED AS no_index_count,
    ROUND(SUM_NO_INDEX_USED / COUNT_STAR * 100, 1) AS no_index_pct
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = DATABASE()
  AND SUM_NO_INDEX_USED > 0
  AND COUNT_STAR > 5
ORDER BY SUM_NO_INDEX_USED DESC
LIMIT 10;

-- Lock 대기가 많은 쿼리
SELECT
    LEFT(DIGEST_TEXT, 100) AS query_pattern,
    COUNT_STAR,
    ROUND(SUM_LOCK_TIME / 1e9 / COUNT_STAR, 2) AS avg_lock_ms,
    ROUND(SUM_LOCK_TIME / 1e9, 0) AS total_lock_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = DATABASE()
  AND SUM_LOCK_TIME > 0
ORDER BY SUM_LOCK_TIME DESC
LIMIT 10;

-- 통계 초기화 (새 배포 후 재측정)
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
```

### 4. 진단 워크플로우 — 느린 쿼리 발견 후 처리

```sql
-- Step 1: 문제 쿼리 식별 (perf_schema 또는 Slow Log)
-- 예: SELECT * FROM orders WHERE status='PAID' AND created_at > '2024-01-01'

-- Step 2: EXPLAIN으로 실행계획 확인
EXPLAIN SELECT * FROM orders
WHERE status = 'PAID' AND created_at > '2024-01-01'\G

-- 핵심 확인 항목:
-- type: ALL      → Full Table Scan (가장 나쁨)
-- type: index    → Index Full Scan (나쁨)
-- type: range    → 인덱스 범위 스캔 (OK)
-- type: ref      → Non-unique 인덱스 룩업 (좋음)
-- type: eq_ref   → Unique 인덱스 룩업 (매우 좋음)
-- key: NULL      → 인덱스 미사용 (문제!)
-- rows: 1000000  → 예상 스캔 행 수 (많으면 위험)
-- Extra: Using filesort → 인덱스로 정렬 못 함
-- Extra: Using temporary → 임시 테이블 사용 (느림)

-- Step 3: EXPLAIN ANALYZE로 실제 vs 예측 비교
EXPLAIN ANALYZE SELECT * FROM orders
WHERE status = 'PAID' AND created_at > '2024-01-01'\G
-- (actual rows=N loops=1): 실제 처리 행 수
-- (rows=M loops=1): 옵티마이저 예측 행 수
-- N >> M이면: 통계 오래됨 → ANALYZE TABLE 필요

-- Step 4: Handler 변수로 실제 I/O 확인
FLUSH STATUS;
-- 쿼리 실행
SELECT * FROM orders WHERE status = 'PAID';
SHOW SESSION STATUS LIKE 'Handler_read%';
-- Handler_read_rnd_next: Full Scan 행 수 (높으면 나쁨)
-- Handler_read_key:      인덱스 룩업 횟수 (높으면 좋음)
-- Handler_read_next:     인덱스 순차 읽기 횟수

-- Step 5: 인덱스 추가 및 검증
CREATE INDEX idx_status_created ON orders(status, created_at);
EXPLAIN SELECT * FROM orders WHERE status='PAID' AND created_at > '2024-01-01';
-- 이제 type: range, key: idx_status_created 확인

-- Step 6: 개선 효과 측정
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
-- 운영 후 재측정
SELECT avg_ms, total_ms FROM perf_schema... -- 개선된 수치 확인
```

### 5. Optimizer Trace로 최적화 과정 분석

```sql
-- 옵티마이저가 왜 이 실행계획을 선택했는지 보기
SET optimizer_trace = 'enabled=on';
SET optimizer_trace_max_mem_size = 1048576;  -- 1MB

SELECT * FROM orders WHERE status = 'PAID' LIMIT 100;

SELECT JSON_PRETTY(TRACE) FROM information_schema.OPTIMIZER_TRACE\G
-- 주요 섹션:
-- join_optimization > rows_estimation > table_scan_cost: Full Scan 비용
-- join_optimization > rows_estimation > index_scan_cost: 인덱스 비용
-- join_optimization > considered_execution_plans: 고려된 계획들
-- best_access_path: 선택된 접근 경로와 이유

-- 옵티마이저가 인덱스를 선택 안 한 이유 파악:
-- Full Scan 비용 < Index Scan 비용이면 Full Scan 선택
-- 이유: rows가 많거나, 선택도가 낮거나, 통계가 부정확

SET optimizer_trace = 'enabled=off';

-- 통계 강제 갱신:
ANALYZE TABLE orders;  -- 통계 업데이트
-- 이후 EXPLAIN에서 rows 예측값이 달라지면 통계 문제였음
```

---

## 💻 실전 실험

### 실험 1: Slow Query Log 실시간 확인

```sql
-- 실험용 Slow Query Log 설정
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0;  -- 모든 쿼리 기록 (실험용!)
SET GLOBAL slow_query_log_file = '/tmp/test_slow.log';

-- 의도적으로 느린 쿼리 실행
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
-- YEAR() 함수 사용 → created_at 인덱스 무력화 → Full Scan

-- 파일에서 확인
-- $ tail -20 /tmp/test_slow.log

-- Rows_examined vs Rows_sent 비교 확인
-- → Rows_examined: 전체 테이블 행 수 (예: 1,000,000)
-- → Rows_sent: 2024년인 행 수 (예: 200,000)
-- → examine_send_ratio = 5 (5행 스캔당 1행 반환)

SET GLOBAL long_query_time = 1;  -- 원복
```

### 실험 2: performance_schema 분석 워크플로우

```sql
-- 통계 초기화 (깨끗한 상태로 시작)
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;

-- 다양한 쿼리 실행 (워크로드 시뮬레이션)
-- (여러 번 실행하여 통계 쌓기)
SELECT * FROM orders WHERE status = 'PAID' LIMIT 10;
SELECT * FROM orders WHERE user_id = 1;
SELECT COUNT(*) FROM orders GROUP BY status;

-- 통계 확인
SELECT
    SUBSTR(DIGEST_TEXT, 1, 60) AS query,
    COUNT_STAR,
    ROUND(AVG_TIMER_WAIT/1e9, 3) AS avg_ms,
    ROUND(SUM_TIMER_WAIT/1e9, 1) AS total_ms,
    ROUND(SUM_ROWS_EXAMINED/NULLIF(SUM_ROWS_SENT,0), 0) AS ratio
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = DATABASE()
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

### 실험 3: EXPLAIN ANALYZE로 예측 vs 실제 비교

```sql
-- 통계가 부정확한 상황 만들기
INSERT INTO orders SELECT ... LIMIT 100000;  -- 대량 삽입
-- (통계 업데이트 전)

EXPLAIN SELECT * FROM orders WHERE status = 'PAID';
-- rows: (부정확한 예측치)

ANALYZE TABLE orders;  -- 통계 갱신

EXPLAIN SELECT * FROM orders WHERE status = 'PAID';
-- rows: (더 정확한 예측치)

-- EXPLAIN ANALYZE로 실제 확인
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'PAID' LIMIT 1000\G
-- actual rows vs estimated rows 비교
```

### 실험 4: Handler 변수로 I/O 경로 추적

```sql
-- Full Scan vs Index Scan의 Handler 차이

-- Full Scan:
FLUSH STATUS;
SELECT * FROM orders WHERE memo = 'test';  -- memo에 인덱스 없음
SHOW SESSION STATUS LIKE 'Handler_read%';
-- Handler_read_rnd_next: 전체 행 수 (100만 등)

-- Index Scan:
FLUSH STATUS;
SELECT * FROM orders WHERE user_id = 1;  -- user_id에 인덱스 있음
SHOW SESSION STATUS LIKE 'Handler_read%';
-- Handler_read_key: 1 (인덱스 포인트 조회)
-- Handler_read_next: N (인덱스 순차 읽기)
-- Handler_read_rnd_next: 0 or 낮음
```

---

## 📊 성능 비교

```
분석 도구별 특성:

Slow Query Log:
  커버리지: long_query_time 초과 쿼리만
  저장: 파일 (영구, 재시작 후에도 유지)
  실시간성: 즉시 기록
  집계: mysqldumpslow 또는 pt-query-digest 필요
  오버헤드: 매우 낮음 (파일 쓰기만)

performance_schema:
  커버리지: 모든 쿼리 (임계값 없음)
  저장: 메모리 (재시작 시 초기화)
  실시간성: 즉시 반영
  집계: SQL 쿼리로 즉시 가능
  오버헤드: 낮음 (기본 활성화)

Optimizer Trace:
  커버리지: 활성화된 세션의 특정 쿼리
  저장: 메모리 (information_schema.OPTIMIZER_TRACE)
  실시간성: 즉시
  집계: JSON 직접 분석
  오버헤드: 높음 (활성화 세션만)

long_query_time별 예상 로그 양:
  3.0초: 하루 수십~수백 건 (관리 용이)
  1.0초: 하루 수천 건 (분석 가능)
  0.1초: 하루 수십만 건 (분석 어려움, 디스크 주의)
  0초:   하루 수백만 건 (실험 환경에서만)
```

---

## ⚖️ 트레이드오프

```
long_query_time 설정:

낮게 설정 (0.1초):
  ✅ 미묘한 성능 문제도 포착
  ❌ 로그 파일 폭증 (디스크 공간 소진 위험)
  ❌ 로그 쓰기 자체가 I/O 부하
  ❌ 중요한 쿼리가 로그에 묻힘
  적합: 성능 집중 분석 기간 (단기)

높게 설정 (3초):
  ✅ 확실한 문제 쿼리만 기록
  ✅ 로그 관리 용이
  ❌ 중간 속도 문제 쿼리 놓침
  적합: 초기 도입, 운영 환경 기본값

권장 접근:
  평상시: long_query_time = 1 (기본)
  성능 분석 기간: 0.5 또는 0.1로 일시적 낮춤
  → performance_schema로 보완 (임계값 없음)

log_queries_not_using_indexes:
  ✅ Full Scan 쿼리 포착 (long_query_time 무관)
  ❌ 작은 테이블도 기록 → 로그 폭증
  해결: log_throttle_queries_not_using_indexes = 10
        → 분당 최대 10개로 제한

Optimizer Trace 사용:
  ✅ 옵티마이저 판단 과정 완전 가시화
  ❌ 프로덕션에서 활성화 금지 (높은 오버헤드)
  ❌ 개발/스테이징에서만 사용
```

---

## 📌 핵심 정리

```
Slow Query 분석 핵심:

설정:
  long_query_time = 1.0 (시작점, 점진적으로 낮춤)
  log_queries_not_using_indexes = ON
  log_throttle_queries_not_using_indexes = 10
  innodb_print_all_deadlocks = ON (데드락 포함)

집계:
  mysqldumpslow -s t -t 20: 총 시간 기준 상위 20개
  mysqldumpslow -s c -t 20: 빈도 기준 상위 20개
  pt-query-digest: P95/P99, 실행계획까지 분석

perf_schema 핵심 지표:
  SUM_TIMER_WAIT: 총 소비 시간 (개선 우선순위)
  AVG_TIMER_WAIT: 평균 실행 시간
  examine_send_ratio >> 1: 인덱스 문제
  SUM_NO_INDEX_USED > 0: 인덱스 미사용 횟수

진단 워크플로우:
  ① perf_schema로 문제 쿼리 파악
  ② EXPLAIN으로 실행계획 확인 (type, key, rows)
  ③ EXPLAIN ANALYZE로 예측 vs 실제 비교
  ④ Handler 변수로 실제 I/O 확인
  ⑤ 인덱스 추가 또는 쿼리 재작성
  ⑥ 통계 초기화 후 개선 효과 재측정
```

---

## 🤔 생각해볼 문제

**Q1.** `events_statements_summary_by_digest`에서 `avg_ms = 0.3ms`이지만 `total_ms = 300,000ms`(5분)인 쿼리가 있다. 이 쿼리가 `avg_ms = 2000ms`이지만 `total_ms = 1,000ms`인 다른 쿼리보다 개선 우선순위가 높은 이유는?

<details>
<summary>해설 보기</summary>

**총 시스템 영향(sum_timer_wait)이 개선 우선순위의 핵심 지표**이기 때문입니다.

- 쿼리 A: avg=0.3ms × 실행횟수=1,000,000회 → total=300,000ms (5분)
- 쿼리 B: avg=2,000ms × 실행횟수=0.5회 → total=1,000ms (1초)

쿼리 A를 50% 개선하면 → 150,000ms(2.5분) 절약  
쿼리 B를 50% 개선하면 → 500ms(0.5초) 절약

**실제 효과는 쿼리 A 개선이 300배 크다.** 쿼리 A는 빠르게 느껴지지만 초당 수천 번 호출되어 누적으로 거대한 DB 시간을 소비합니다. 

이 패턴은 N+1 문제에서 자주 발생합니다: 개별 쿼리는 0.5ms로 빠르지만 목록 100개 × 관계 조회 = 100번 실행 → 50ms 누적. 초당 1000번의 API 호출이면 초당 50,000ms = DB가 이 쿼리에 50초를 소비하는 것처럼 보입니다.

</details>

---

**Q2.** EXPLAIN에서 `type: range`로 인덱스를 사용하는데도 쿼리가 느리다. 어떤 상황인지 세 가지 원인을 설명하라.

<details>
<summary>해설 보기</summary>

**원인 1: 선택도가 낮은 범위 조건**  
`WHERE created_at > '2020-01-01'` 같이 거의 모든 Row가 조건에 맞으면, `type: range`로 인덱스를 사용해도 전체 테이블의 80%를 스캔합니다. 옵티마이저가 인덱스를 선택했지만 실제로는 Full Scan과 유사한 비용입니다.

**원인 2: 커버링 인덱스가 아닌 경우 + 대량 결과**  
인덱스로 조건을 만족하는 PK를 찾은 후, Clustered Index (실제 데이터)를 다시 읽는 "Double Lookup"이 발생합니다. 결과가 10만 건이라면 10만 번의 랜덤 I/O → 느림. 커버링 인덱스(SELECT 컬럼이 모두 인덱스에 포함)로 해결 가능합니다.

**원인 3: 정렬(filesort) 추가 비용**  
`WHERE status='PAID' ORDER BY created_at`에서 `status`에만 인덱스가 있으면, PAID인 Row를 찾은 후 `created_at`으로 정렬이 필요합니다. `Extra: Using filesort`가 나타나며, 대량의 결과를 정렬할 때 느려집니다. `(status, created_at)` 복합 인덱스로 해결합니다.

추가 확인: `EXPLAIN ANALYZE`로 `actual rows`와 `estimated rows` 비교 → 차이가 크면 통계 오류 (`ANALYZE TABLE` 필요).

</details>

---

**Q3.** `log_queries_not_using_indexes = ON`으로 설정했는데 5분 만에 Slow Query Log가 10GB가 됐다. 원인과 즉각적인 대처 방법, 그리고 재발 방지 설정을 설명하라.

<details>
<summary>해설 보기</summary>

**원인**: 작은 테이블(수천 건)을 자주 조회하는 쿼리들이 인덱스를 사용하지 않아도 `long_query_time` 미만으로 빠르게 완료됩니다. `log_queries_not_using_indexes`는 시간과 무관하게 인덱스 미사용 쿼리를 모두 기록하므로, 초당 수천 번 실행되는 내부 쿼리들이 모두 기록됩니다.

**즉각 대처**:
```sql
-- Slow Query Log 즉시 비활성화
SET GLOBAL slow_query_log = OFF;
-- 또는 인덱스 미사용 기록만 비활성화
SET GLOBAL log_queries_not_using_indexes = OFF;

-- 로그 파일 정리
-- $ truncate -s 0 /var/log/mysql/slow.log
```

**재발 방지 설정**:
```sql
-- 방법 1: 스로틀링 (분당 최대 N개)
SET GLOBAL log_throttle_queries_not_using_indexes = 10;
-- → 동일 패턴 쿼리는 분당 10개만 기록

-- 방법 2: 최소 실행 시간 함께 적용
-- my.cnf에서:
-- log_queries_not_using_indexes = 1
-- log_throttle_queries_not_using_indexes = 10
-- min_examined_row_limit = 1000  ← 1000행 이상 스캔한 것만 기록

-- 방법 3: performance_schema로 대체
-- log_queries_not_using_indexes를 낮은 빈도로 유지하고
-- SUM_NO_INDEX_USED 컬럼으로 주기적 모니터링
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: N+1 문제 ➡️](./02-n-plus-one-db-perspective.md)**

</div>
