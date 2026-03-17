# 인덱스 설계 전략 — 언제 걸고 언제 걸지 않는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 인덱스 추가가 INSERT/UPDATE/DELETE 성능을 어떻게, 얼마나 저하시키는가?
- "인덱스는 항상 좋다"는 생각이 틀린 이유는 무엇인가?
- 기존 인덱스가 새 인덱스로 대체될 수 있는 조건은 무엇인가?
- 인덱스를 제거해야 하는 신호는 어떻게 감지하는가?
- 복합 인덱스 vs 단일 인덱스 2개 중 어느 것이 더 효율적인가?
- 실무에서 인덱스 설계 프로세스는 어떻게 접근해야 하는가?

---

## 🔍 왜 이 개념이 중요한가

### 인덱스는 공짜가 아니다

```
인덱스가 많을수록 좋다는 믿음:
  orders 테이블에 10개의 인덱스 생성

INSERT INTO orders (user_id, amount, status, created_at, ...)
VALUES (1, 10000, 'PENDING', NOW(), ...);

실제로 일어나는 일:
  ① Clustered Index(PK) B+Tree에 삽입: 3~4 I/O
  ② idx_user_id B+Tree에 삽입: 3~4 I/O
  ③ idx_status B+Tree에 삽입: 3~4 I/O
  ④ idx_created_at B+Tree에 삽입: 3~4 I/O
  ⑤ idx_composite B+Tree에 삽입: 3~4 I/O
  ⑥ ... (10개 인덱스 × 3~4 I/O)
  
  1개 인덱스일 때: ~4 I/O
  10개 인덱스일 때: ~40 I/O (10배!)
  
  초당 1만 건 INSERT → 인덱스 1개: 4만 I/O
  초당 1만 건 INSERT → 인덱스 10개: 40만 I/O
  → I/O 병목 → TPS 급감

반대로, 인덱스가 없으면:
  자주 실행되는 SELECT가 Full Scan → 응답 시간 폭증

균형을 찾는 것이 인덱스 설계의 핵심
```

---

## 😱 잘못된 이해

### Before: "일단 다 걸고 보자" vs "인덱스는 부담스럽다"

```
극단 1: Over-Indexing
  "WHERE에 쓰이는 컬럼은 다 인덱스"
  "혹시 몰라서 인덱스 추가"
  → 쓰기 성능 지속 저하
  → Buffer Pool 낭비
  → 중복 인덱스 존재 (유지 비용만 있고 이득 없음)

극단 2: Under-Indexing
  "인덱스는 복잡하니까 나중에"
  "잘 모르겠으니 그냥 PK만"
  → 모든 조회가 Full Scan
  → DB 서버 CPU/I/O 과부하
  → 응답 시간 수십~수백 ms

올바른 접근:
  쿼리 패턴 분석 → 고빈도 쿼리 식별 → 해당 쿼리에 최적화된 인덱스
  정기적 모니터링 → 사용 안 하는 인덱스 제거
  쓰기 성능 vs 읽기 성능 트레이드오프 명시적으로 결정
```

---

## 🔬 내부 동작 원리

### 1. 인덱스가 쓰기 성능에 미치는 영향

```
INSERT 시 각 인덱스의 처리:

① Clustered Index (PK):
   Row를 PK 순서에 맞는 Leaf Page에 삽입
   AUTO_INCREMENT: 항상 마지막 Page → Split 거의 없음
   UUID: 랜덤 위치 → 잦은 Split

② Secondary Index마다:
   해당 인덱스의 Leaf Page에 (인덱스 컬럼 값, PK) 삽입
   인덱스 컬럼 값이 랜덤하면 잦은 Split 발생 가능
   Change Buffer 활용: Index Page가 Buffer Pool에 없으면
     즉시 디스크 I/O 대신 Change Buffer에 기록 → 나중에 Merge
     → 즉각적인 INSERT 속도 향상 (I/O 지연)

UPDATE 시:
  변경된 컬럼이 인덱스에 포함되면:
    기존 인덱스 항목 삭제 + 새 인덱스 항목 삽입
    = 인덱스 2번 수정

DELETE 시:
  Delete Mark 처리 (즉시 물리적 삭제 아님)
  Purge Thread가 나중에 인덱스에서도 제거
  모든 인덱스에서 해당 항목 제거 필요

성능 영향 측정:
  쓰기 집약적 테이블에서:
    인덱스 5개: INSERT 속도 기준 대비
    인덱스 1개: ~5배 빠를 수 있음
    (I/O가 병목인 경우, 인덱스별 차이는 워크로드에 따라 다름)
```

### 2. 중복 인덱스 감지와 제거

```sql
-- 중복 인덱스 패턴:

-- 패턴 1: 완전 중복
CREATE INDEX idx1 ON orders(user_id);
CREATE INDEX idx2 ON orders(user_id);  -- 완전 중복 → idx1 삭제

-- 패턴 2: Prefix 중복 (Redundant Index)
CREATE INDEX idx_user ON orders(user_id);
CREATE INDEX idx_user_status ON orders(user_id, status);
-- idx_user는 idx_user_status의 Prefix → idx_user가 중복!
-- idx_user_status로 user_id만 조회하는 쿼리도 처리 가능
-- → idx_user 제거 가능 (Composite가 단일을 포함)

-- 단, idx_user가 별도로 필요한 경우:
-- user_id만으로 Covering Index가 필요한 쿼리
-- idx_user_status는 (user_id, status, PK) — user_id만 필요해도 status까지 읽음
-- → 쿼리 패턴에 따라 단독 인덱스가 더 효율적일 수 있음

-- 중복 인덱스 자동 감지 (pt-duplicate-key-checker):
-- pt-duplicate-key-checker --host=localhost --user=root --password=...
-- 출력: 중복 인덱스 목록 + 제거 SQL 제안

-- INFORMATION_SCHEMA로 직접 확인:
SELECT
    t1.table_schema,
    t1.table_name,
    t1.index_name AS redundant_index,
    t2.index_name AS dominant_index,
    GROUP_CONCAT(t1.column_name ORDER BY t1.seq_in_index) AS redundant_cols,
    GROUP_CONCAT(t2.column_name ORDER BY t2.seq_in_index) AS dominant_cols
FROM information_schema.STATISTICS t1
JOIN information_schema.STATISTICS t2
    ON t1.table_schema = t2.table_schema
    AND t1.table_name = t2.table_name
    AND t1.index_name != t2.index_name
    AND t1.seq_in_index = 1
    AND t2.seq_in_index = 1
    AND t1.column_name = t2.column_name
WHERE t1.table_schema = 'deep_dive'
GROUP BY t1.table_schema, t1.table_name, t1.index_name, t2.index_name;
```

### 3. 인덱스 사용 현황 모니터링

```sql
-- performance_schema로 인덱스 사용 현황 확인
SELECT
    OBJECT_SCHEMA AS db,
    OBJECT_NAME AS tbl,
    INDEX_NAME,
    COUNT_READ,
    COUNT_WRITE,
    COUNT_FETCH,
    SUM_TIMER_WAIT / 1e12 AS total_wait_sec
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = 'deep_dive'
  AND OBJECT_NAME = 'orders'
ORDER BY COUNT_READ ASC;
-- COUNT_READ = 0인 인덱스 → 전혀 사용 안 함 → 제거 후보

-- 주의: performance_schema는 서버 재시작 후 초기화됨
-- 충분한 기간 동안 (최소 1주일, 이상적으로 1개월) 모니터링 후 판단

-- information_schema.INNODB_INDEXES로 인덱스 크기 확인
SELECT
    i.NAME AS index_name,
    ROUND(s.stat_value * 16 / 1024, 2) AS size_kb
FROM information_schema.INNODB_INDEXES i
JOIN information_schema.INNODB_TABLES t ON i.TABLE_ID = t.TABLE_ID
JOIN mysql.innodb_index_stats s
    ON s.database_name = SUBSTRING_INDEX(t.NAME, '/', 1)
    AND s.table_name = SUBSTRING_INDEX(t.NAME, '/', -1)
    AND s.index_name = i.NAME
    AND s.stat_name = 'size'
WHERE SUBSTRING_INDEX(t.NAME, '/', 1) = 'deep_dive'
  AND SUBSTRING_INDEX(t.NAME, '/', -1) = 'orders';
-- 크기가 크고 COUNT_READ=0인 인덱스는 제거 우선순위 높음
```

### 4. 인덱스 설계 프로세스

```
실무 인덱스 설계 5단계:

Step 1: 쿼리 패턴 수집
  소스:
    Slow Query Log (long_query_time = 1초 이상)
    APM 도구 (DataDog, New Relic 등)의 Top N 쿼리
    개발 중 작성한 주요 Repository 메서드
    
  분석 항목:
    실행 빈도
    평균 응답 시간
    영향받는 Row 수
    현재 인덱스 사용 여부 (EXPLAIN)

Step 2: 고빈도 + 고비용 쿼리 우선순위화
  우선순위 = 실행 빈도 × 평균 응답 시간 × Row 수
  상위 5~10개 쿼리에 집중

Step 3: 각 쿼리에 최적 인덱스 설계
  공식: (등치 조건) + (범위/ORDER BY) + (Covering용 SELECT 컬럼)
  EXPLAIN으로 검증
  실제 성능 측정

Step 4: 인덱스 통합 및 중복 제거
  기존 인덱스와 중복 여부 확인
  Prefix 중복 인덱스 통합
  하나의 Composite Index로 여러 쿼리 커버 가능한지 확인

Step 5: 정기 리뷰
  매 분기: 사용 안 하는 인덱스 확인 (performance_schema)
  데이터 분포 변경 시: ANALYZE TABLE로 통계 갱신
  쿼리 패턴 변경 시: 인덱스 재설계
```

### 5. 복합 인덱스 vs 단일 인덱스 2개 선택 기준

```sql
-- 시나리오: WHERE user_id = ? AND status = ?

-- 옵션 A: 단일 인덱스 2개
CREATE INDEX idx_user ON orders(user_id);
CREATE INDEX idx_status ON orders(status);

-- 동작: Optimizer가 더 효율적인 하나를 선택
--       또는 Index Merge (두 인덱스 결과 교집합)
-- Index Merge Union: 두 인덱스 결과 합치기 (OR 조건)
-- Index Merge Intersect: 두 인덱스 결과 교집합 (AND 조건) → 드물게 사용

-- 옵션 B: Composite Index
CREATE INDEX idx_user_status ON orders(user_id, status);

-- 동작: (user_id, status) 동시 탐색 → 정확히 원하는 범위만 스캔
-- 단일 인덱스 조합보다 효율적 (범위가 더 좁아짐)

-- 비교 (user_id가 1만 명, 각 user당 평균 status 3가지):
-- 옵션 A:
--   idx_user: user_id=1 → 100건 결과 예상
--   또는 idx_status: status='PENDING' → 33만 건 예상
--   Optimizer: idx_user 선택 → 100건 → status 필터
--   I/O: 3(탐색) + 100(Double Lookup) + 필터
-- 
-- 옵션 B:
--   idx_user_status: (user_id=1, status='PENDING') → 정확히 수십 건
--   I/O: 3(탐색) + 수십(Double Lookup)
--   → 옵션 A보다 Double Lookup 횟수 감소

-- 단일 인덱스가 유리한 경우:
-- user_id 단독 쿼리 + status 단독 쿼리 모두 빈번
-- → (user_id, status) Composite는 status 단독에 비효율
-- → idx_user + idx_status 각각이 더 유연

-- 복합 인덱스가 유리한 경우:
-- user_id AND status 조합이 가장 빈번
-- user_id 단독도 빈번 → (user_id, status)의 Prefix로 처리 가능
-- → Composite 1개로 두 패턴 모두 커버
```

### 6. 인덱스 제거 결정 프로세스

```sql
-- 인덱스 제거 체크리스트:

-- 1. 사용 빈도 확인 (충분한 기간 측정)
SELECT index_name, count_read, count_write
FROM performance_schema.table_io_waits_summary_by_usage  -- 실제 테이블명 확인
WHERE ... ;

-- 2. 제거 전 안전한 비활성화 (MySQL 8.0: Invisible Index)
-- 인덱스를 실제 제거하지 않고 Optimizer에서 보이지 않게 설정
ALTER TABLE orders ALTER INDEX idx_unused INVISIBLE;
-- Optimizer가 idx_unused를 무시하도록 설정
-- 성능 변화 모니터링 (1~2주)
-- 문제 없으면 실제 제거:
DROP INDEX idx_unused ON orders;
-- 문제 발생 시 즉시 복원:
ALTER TABLE orders ALTER INDEX idx_unused VISIBLE;

-- 3. 중복 인덱스 제거 시 기존 쿼리 검증
-- 제거 전: 해당 인덱스를 사용하는 쿼리가 다른 인덱스로 처리 가능한지 EXPLAIN 확인
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G  -- idx_user 제거 전 idx_user_status로 처리 가능?

-- 4. 인덱스 제거가 힘든 경우: Invisible + FORCE INDEX 테스트
ALTER TABLE orders ALTER INDEX idx_redundant INVISIBLE;
-- 해당 쿼리들의 EXPLAIN 재확인
-- 1주 모니터링 → 성능 저하 없으면 DROP
```

---

## 💻 실전 실험

### 실험 1: 인덱스 수에 따른 INSERT 성능 측정

```sql
-- 인덱스 0개
CREATE TABLE bench_idx0 (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT,
    amount     DECIMAL(10,2),
    status     VARCHAR(20),
    created_at DATETIME
) ENGINE=InnoDB;

-- 인덱스 5개
CREATE TABLE bench_idx5 LIKE bench_idx0;
CREATE INDEX idx1 ON bench_idx5(user_id);
CREATE INDEX idx2 ON bench_idx5(amount);
CREATE INDEX idx3 ON bench_idx5(status);
CREATE INDEX idx4 ON bench_idx5(created_at);
CREATE INDEX idx5 ON bench_idx5(user_id, status);

-- INSERT 성능 비교
SET @start = NOW(6);
INSERT INTO bench_idx0 (user_id, amount, status, created_at)
SELECT FLOOR(RAND()*100000), ROUND(RAND()*10000,2),
       ELT(FLOOR(RAND()*3)+1,'PAID','PENDING','CANCELLED'), NOW()
FROM information_schema.columns a CROSS JOIN information_schema.columns b
LIMIT 50000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS ms_0idx;

SET @start = NOW(6);
INSERT INTO bench_idx5 (user_id, amount, status, created_at)
SELECT FLOOR(RAND()*100000), ROUND(RAND()*10000,2),
       ELT(FLOOR(RAND()*3)+1,'PAID','PENDING','CANCELLED'), NOW()
FROM information_schema.columns a CROSS JOIN information_schema.columns b
LIMIT 50000;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS ms_5idx;

-- 예상: bench_idx5가 2~5배 느림
```

### 실험 2: Invisible Index로 안전한 인덱스 비활성화

```sql
-- 인덱스 생성
CREATE INDEX idx_test ON orders(status, amount);

-- 현재 EXPLAIN
EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND amount > 1000\G

-- Invisible로 변경
ALTER TABLE orders ALTER INDEX idx_test INVISIBLE;

-- EXPLAIN 재확인 (idx_test가 사용되지 않음)
EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND amount > 1000\G
-- key: NULL 또는 다른 인덱스 사용

-- 성능 변화 모니터링 후 문제 없으면 제거
DROP INDEX idx_test ON orders;

-- 또는 문제 발생 시 즉시 복원
ALTER TABLE orders ALTER INDEX idx_test VISIBLE;
```

### 실험 3: performance_schema로 사용 안 하는 인덱스 찾기

```sql
-- 인덱스 사용 현황 조회
SELECT
    OBJECT_NAME AS table_name,
    INDEX_NAME,
    COUNT_READ AS reads,
    COUNT_WRITE AS writes,
    COUNT_READ + COUNT_WRITE AS total_ops
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = 'deep_dive'
  AND OBJECT_NAME = 'orders'
  AND INDEX_NAME IS NOT NULL
ORDER BY COUNT_READ;

-- COUNT_READ=0인 인덱스 → 사용 안 하는 인덱스 후보
-- COUNT_WRITE>0이어도 COUNT_READ=0 → 쓰기 비용만 내고 읽기에 쓰이지 않음 → 제거 후보

-- 주의: performance_schema 데이터는 서버 시작 후부터 누적
-- TRUNCATE로 초기화:
TRUNCATE TABLE performance_schema.table_io_waits_summary_by_index_usage;
-- 초기화 후 새로 모니터링 시작
```

### 실험 4: 복합 인덱스 vs 단일 인덱스 2개 비교

```sql
-- 셋업
CREATE TABLE test_idx_compare (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT,
    status  VARCHAR(20),
    amount  DECIMAL(10,2)
) ENGINE=InnoDB;

INSERT INTO test_idx_compare (user_id, status, amount)
SELECT FLOOR(RAND()*1000), ELT(FLOOR(RAND()*3)+1,'PAID','PENDING','CANCELLED'),
       ROUND(RAND()*10000, 2)
FROM information_schema.columns a CROSS JOIN information_schema.columns b
LIMIT 100000;

-- 단일 인덱스 2개 버전
CREATE INDEX idx_user ON test_idx_compare(user_id);
CREATE INDEX idx_status ON test_idx_compare(status);

EXPLAIN SELECT * FROM test_idx_compare WHERE user_id = 500 AND status = 'PAID'\G
-- type: ref, key: idx_user 또는 Index Merge

-- 복합 인덱스 버전
DROP INDEX idx_user ON test_idx_compare;
DROP INDEX idx_status ON test_idx_compare;
CREATE INDEX idx_user_status ON test_idx_compare(user_id, status);

EXPLAIN SELECT * FROM test_idx_compare WHERE user_id = 500 AND status = 'PAID'\G
-- type: ref, key: idx_user_status, key_len: 더 길어짐
-- rows: 더 적은 값 (두 조건 동시 처리)
```

---

## 📊 성능 비교

```
인덱스 설계 결정 매트릭스:

쿼리 빈도: 높음 / 낮음
쓰기 빈도: 높음 / 낮음

          쿼리 높음     쿼리 낮음
쓰기 높음   인덱스 설계  인덱스 최소화
           신중하게      (쓰기 성능 우선)
쓰기 낮음   인덱스 적극  기본 PK만으로
           활용         충분할 수 있음

대표적인 시나리오:

OLTP (읽기 집약적 서비스, ex: 쇼핑몰 조회 API):
  읽기: 높음, 쓰기: 중간
  → 핵심 조회 인덱스 설계 (3~5개 인덱스)
  → Covering Index 전략 활용

OLTP (쓰기 집약적, ex: 로그 수집, IoT):
  읽기: 낮음, 쓰기: 매우 높음
  → 인덱스 최소화 (PK + 필수 1~2개)
  → 분석은 Read Replica에서

Analytics/DW:
  읽기: 복잡한 집계, 쓰기: 배치성
  → 컬럼 스토어(ClickHouse), 파티셔닝 고려
  → MySQL 인덱스보다 적합한 솔루션 검토
```

---

## ⚖️ 트레이드오프

```
인덱스 추가/제거 결정 기준:

추가해야 하는 신호:
  ✅ Slow Query Log에 동일 쿼리가 반복
  ✅ EXPLAIN type: ALL (Full Scan)인데 자주 실행
  ✅ 결과 비율 5% 이하, 실행 빈도 높음
  ✅ 응답 시간 SLA 위반
  ✅ DB CPU/I/O 사용률이 인덱스 없는 Full Scan으로 높음

제거해야 하는 신호:
  🗑️ COUNT_READ = 0 (1개월 이상 측정)
  🗑️ 동일 컬럼을 포함하는 더 넓은 Composite Index 존재
  🗑️ 완전 중복 인덱스
  🗑️ 매우 낮은 Selectivity + 단독 사용 쿼리 없음
  🗑️ 쓰기가 많은 테이블에서 거의 사용 안 되는 인덱스

인덱스 유지 비용:
  고정 비용: 디스크 공간 (인덱스 크기)
  변동 비용: INSERT/UPDATE/DELETE마다 B+Tree 갱신 I/O
  Buffer Pool: 인덱스 Page를 캐시하는 공간 차지

인덱스 없는 경우 비용:
  쿼리마다 Full Scan → CPU + I/O 급증
  동시 접속자가 많으면 DB 서버 부하 폭증

결론:
  정기적인 모니터링 + 데이터 기반 결정
  "필요하면 추가, 불필요하면 제거"
  인덱스는 영구적인 결정이 아님 — 언제든 추가/제거 가능
```

---

## 📌 핵심 정리

```
인덱스 설계 전략 핵심:

인덱스의 비용:
  INSERT: 인덱스마다 B+Tree 갱신 → 쓰기 I/O 증가
  UPDATE: 변경된 컬럼 인덱스마다 삭제+삽입
  DELETE: 모든 인덱스에서 해당 항목 제거
  → 인덱스 10개 = INSERT 비용 10배 (I/O 기준)

인덱스 설계 5단계:
  1. Slow Query Log / APM으로 고비용 쿼리 수집
  2. 빈도 × 비용으로 우선순위화
  3. 쿼리별 최적 인덱스 설계 (EXPLAIN 검증)
  4. 중복/Prefix 인덱스 통합
  5. 정기 리뷰 (사용 안 하는 인덱스 제거)

Composite vs 단일 인덱스:
  같이 쓰이는 컬럼 → Composite (범위 더 좁아짐)
  각각 독립적으로 자주 사용 → 단일 인덱스 각각

인덱스 안전한 제거:
  performance_schema로 사용 현황 충분히 측정 (1개월+)
  Invisible Index로 비활성화 → 영향 관찰 → 문제 없으면 DROP

Over-Indexing 방지:
  "혹시 몰라서" 인덱스 추가 금지
  쿼리 패턴과 측정 데이터 기반으로만 결정
  쓰기 집약적 테이블은 인덱스 최소화 원칙

Chapter 2 요약:
  B+Tree 구조 → 인덱스의 모든 동작 원리의 기반
  Clustered = 데이터 자체, Secondary = PK 포인터
  Covering = Double Lookup 제거 → 성능 극대화
  Composite = 순서가 핵심 (Leftmost Prefix)
  Selectivity = 인덱스 효과 결정 지표
  Anti-patterns = 함수/형변환/LIKE앞/OR 주의
  설계 = 측정 → 설계 → 검증 → 정기 리뷰 사이클
```

---

## 🤔 생각해볼 문제

**Q1.** 쇼핑몰 서비스에서 초당 1,000건의 주문이 생성되는 테이블에 현재 인덱스가 8개 있다. DBA가 "인덱스를 3개로 줄여야 한다"고 주장하고, 개발팀은 "조회 성능이 떨어진다"고 반대한다. 어떤 데이터를 수집하여 합리적인 결정을 내릴 수 있는가?

<details>
<summary>해설 보기</summary>

**수집해야 할 데이터**:

1. **쓰기 성능 현황**:
   - 현재 INSERT 평균 응답 시간 (APM, Slow Query Log)
   - DB 서버 I/O 사용률, CPU 사용률
   - `Innodb_rows_inserted`, `Innodb_os_log_fsyncs` per second

2. **인덱스별 사용 현황** (1개월):
   ```sql
   SELECT index_name, count_read, count_write
   FROM performance_schema.table_io_waits_summary_by_index_usage
   WHERE OBJECT_NAME = 'orders';
   ```
   - 8개 중 COUNT_READ=0인 인덱스 식별 → 제거 후보

3. **조회 성능 현황**:
   - Top 10 조회 쿼리의 현재 응답 시간
   - 각 쿼리의 EXPLAIN (어느 인덱스 사용)

4. **시뮬레이션**:
   - COUNT_READ=0인 인덱스를 `INVISIBLE`로 변경
   - 조회 성능 변화 모니터링 (1~2주)
   - 쓰기 성능 변화 측정

5. **결정 기준**:
   - 조회 응답 시간 SLA를 만족하는 최소 인덱스 세트 결정
   - 불필요한 인덱스(COUNT_READ=0, Invisible 후 성능 변화 없음) 순서대로 제거

이 데이터 없이 "줄여야 한다" vs "안 된다"는 추측에 기반한 논쟁입니다. 측정이 먼저입니다.

</details>

---

**Q2.** 신규 서비스 오픈 전 스키마 설계 단계에서 인덱스를 설계할 때, Slow Query Log 데이터가 없는 상황에서 어떻게 접근해야 하는가?

<details>
<summary>해설 보기</summary>

데이터 없는 설계 단계에서의 접근 방법:

**1. 쿼리 패턴 선제 분석**:
- 기획서, API 스펙에서 예상 조회 패턴 도출
- 목록 조회 API의 WHERE 조건과 ORDER BY 추출
- JPA Repository 메서드 시그니처로 예상 SQL 역산

**2. 최소한의 필수 인덱스만**:
- PK(필수)
- FK 컬럼(JOIN에서 반드시 필요)
- 확실한 고빈도 등치 조건 (user_id, 세션 토큰 등)
- 정렬이 필요한 목록 조회의 ORDER BY 컬럼

**3. 오픈 후 빠른 모니터링 체계 구축**:
```sql
-- Slow Query Log 설정
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;  -- 500ms 이상 기록
```
- 오픈 1~2주 후 Slow Query 분석 → 추가 인덱스 설계

**4. 과잉 인덱스 경계**:
"혹시 몰라서" 10개 미리 걸어두는 것보다 필수 5개로 오픈 후 데이터 기반으로 추가하는 것이 낫습니다. 인덱스는 나중에 추가할 수 있지만(Online DDL), 쓰기 성능 저하는 즉시 발생합니다.

</details>

---

**Q3.** 배치 처리(매일 밤 500만 건 INSERT)가 있는 테이블에 실시간 조회를 위한 인덱스가 7개 있다. 배치 처리 시간이 목표(1시간)를 초과하고 있다. 인덱스를 줄이지 않고 배치 성능을 개선할 수 있는 방법은?

<details>
<summary>해설 보기</summary>

인덱스를 유지하면서 배치 INSERT 성능을 높이는 방법:

**1. 배치 중 인덱스 비활성화 (테이블 격리 가능 시)**:
```sql
-- 주의: 실시간 조회가 없는 배치 전용 테이블에서만
ALTER TABLE batch_data DISABLE KEYS;
-- 배치 INSERT
ALTER TABLE batch_data ENABLE KEYS;  -- 완료 후 일괄 재빌드 (더 빠름)
```

**2. 배치 크기와 COMMIT 최적화**:
```java
// 1건씩 INSERT + COMMIT → 최악
// 1000건 묶어서 INSERT + COMMIT → 훨씬 빠름
// JDBC batch: addBatch() + executeBatch()
```

**3. INSERT 순서 최적화**:
AUTO_INCREMENT PK 테이블은 PK 순 INSERT가 최적. 다른 순서라면 정렬 후 삽입.

**4. innodb_flush_log_at_trx_commit = 2** (배치 중 임시):
COMMIT마다 fsync 없이 → 쓰기 속도 향상. 배치 완료 후 복원. 
MySQL 재시작(크래시) 시 최대 1초 손실 감수 필요.

**5. Change Buffer 극대화**:
`innodb_change_buffer_max_size = 50` (기본 25 → 50으로 증가)
Secondary Index Page가 Buffer Pool에 없을 때 Change Buffer에 기록 → 즉각적 I/O 감소.

**6. 파티셔닝으로 배치용 파티션 분리**:
날짜 기반 RANGE 파티셔닝 → 신규 배치 데이터가 새 파티션에 집중 → 인덱스 갱신 범위 좁아짐.

</details>

---

<div align="center">

**[⬅️ 이전: 인덱스를 타지 않는 패턴](./06-index-skip-patterns.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 쿼리 실행과 Optimizer ➡️](../query-execution-optimizer/01-query-execution-stages.md)**

</div>
