# EXPLAIN과 EXPLAIN ANALYZE — 실행계획 읽는 법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- EXPLAIN의 `type` 컬럼에서 `system`, `const`, `eq_ref`, `ref`, `range`, `index`, `ALL`은 각각 어떤 상황인가?
- `key_len`으로 Composite Index의 어느 컬럼까지 실제 사용됐는지 어떻게 계산하는가?
- `rows` 컬럼이 정확하지 않은 이유와 그것이 문제를 일으키는 경우는?
- `Extra: Using filesort`가 발생하는 조건과 이를 제거할 수 있는 인덱스 설계는?
- `Extra: Using temporary`는 언제 발생하고 왜 비용이 높은가?
- `EXPLAIN ANALYZE`의 `actual time=X..Y rows=N loops=M`은 각각 무엇을 의미하는가?

---

## 🔍 왜 이 개념이 중요한가

### EXPLAIN은 DB 성능 진단의 시작점

```
"느린 쿼리를 받았을 때 첫 번째 행동":
  EXPLAIN SELECT ...;
  
  EXPLAIN을 읽지 못하면:
    어떤 인덱스가 사용됐는지 모름
    얼마나 많은 Row를 읽는지 모름
    왜 느린지 원인을 모름
    → 무작위로 인덱스 추가, 쿼리 변경 시도

  EXPLAIN을 읽을 수 있으면:
    type: ALL → Full Scan → 인덱스 없음 → 추가 필요
    type: ref, rows: 50,000 → 너무 많이 읽음 → Selectivity 문제
    Extra: Using filesort → 정렬 발생 → ORDER BY 컬럼 인덱스 추가
    → 정확한 원인 진단 → 올바른 해결책

이 문서의 목표:
  EXPLAIN 각 컬럼을 InnoDB 내부 동작과 연결해서
  "이 값은 이런 B+Tree 연산이다"로 해석
```

---

## 🔬 내부 동작 원리

### 1. type 컬럼 — 접근 방식 (빠름 → 느림)

```sql
-- type 값의 의미와 발생 조건:

-- system: 테이블에 Row가 1건 (또는 0건)
-- 예: SELECT * FROM (SELECT 1) t;

-- const: PK 또는 UNIQUE 인덱스의 등치 조건 → 결과 항상 0 or 1건
-- Optimizer가 실행 전에 결과를 상수로 대체할 수 있음
EXPLAIN SELECT * FROM orders WHERE id = 42\G
-- type: const ← PK 등치, 결과 최대 1건, 최적!

-- eq_ref: JOIN에서 Driven Table의 PK/UNIQUE로 접근
EXPLAIN SELECT o.*, u.name
FROM orders o JOIN users u ON o.user_id = u.id
WHERE o.status = 'PAID'\G
-- type(users): eq_ref ← users.id(PK)로 JOIN, 최적 JOIN

-- ref: Non-Unique 인덱스의 등치 조건
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- type: ref ← idx_user_id(non-unique), user_id=1인 여러 건

-- fulltext: FULLTEXT 인덱스 사용
EXPLAIN SELECT * FROM articles WHERE MATCH(content) AGAINST('MySQL')\G

-- ref_or_null: ref와 동일하지만 NULL도 포함
EXPLAIN SELECT * FROM orders WHERE user_id = 1 OR user_id IS NULL\G

-- index_merge: 여러 인덱스를 결합하여 사용
EXPLAIN SELECT * FROM orders WHERE user_id = 1 OR status = 'PAID'\G
-- type: index_merge, Extra: Using union(idx_user, idx_status)

-- range: 인덱스 범위 스캔
EXPLAIN SELECT * FROM orders WHERE id > 1000 AND id < 2000\G
-- type: range ← PK Range Scan

-- index: 인덱스 전체를 순회 (Full Index Scan)
-- 테이블보다는 작지만 여전히 전체 순회
EXPLAIN SELECT user_id FROM orders ORDER BY user_id\G
-- type: index ← idx_user_id 전체 순회 (Covering)

-- ALL: Full Table Scan — 최악
EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2024\G
-- type: ALL ← 함수로 인덱스 무력화 → Full Scan

-- type별 I/O 비용 비교 (100만 건 기준):
-- const:      1 I/O (단 1건)
-- eq_ref:     3~4 I/O (JOIN 1건)
-- ref:        3~4 + 결과 건수 I/O
-- range:      3 + 범위 내 Page 수 I/O
-- index:      전체 인덱스 Page 수 I/O
-- ALL:        전체 테이블 Page 수 I/O (가장 큼)
```

### 2. key_len 계산

```sql
-- key_len: 인덱스에서 실제 사용된 bytes 수
-- 이것으로 Composite Index의 몇 번째 컬럼까지 사용됐는지 역산

-- 각 데이터 타입의 크기:
-- TINYINT:      1 byte
-- SMALLINT:     2 bytes
-- MEDIUMINT:    3 bytes
-- INT:          4 bytes
-- BIGINT:       8 bytes
-- FLOAT:        4 bytes
-- DOUBLE:       8 bytes
-- DATE:         3 bytes
-- TIME:         3 bytes (MySQL 5.6+ 마이크로초 제외)
-- DATETIME:     5 bytes (MySQL 5.6+)
-- TIMESTAMP:    4 bytes
-- YEAR:         1 byte
-- CHAR(N):      N × charset_bytes (utf8mb4: 4 bytes/char)
-- VARCHAR(N):   N × charset_bytes + 2 bytes (길이 저장)
-- NULL 허용:    +1 byte (NULL 플래그)

-- 예시 계산:
-- CREATE INDEX idx_a ON orders(user_id, status, created_at);
-- user_id: BIGINT NOT NULL → 8 bytes
-- status: VARCHAR(20) utf8mb4 NOT NULL → 20×4+2 = 82 bytes
-- created_at: DATETIME NOT NULL → 5 bytes

EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- key_len: 8 ← user_id만 사용됨 (BIGINT = 8 bytes)

EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status = 'PAID'\G
-- key_len: 90 ← user_id(8) + status(82) = 90 bytes

EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status = 'PAID' AND created_at > '2024-01-01'\G
-- key_len: 95 ← 전체 = 8 + 82 + 5 = 95 bytes

-- 컬럼이 NULL 허용인 경우:
-- user_id BIGINT NULL → key_len에서 8+1 = 9 bytes
ALTER TABLE orders MODIFY user_id BIGINT NULL;
EXPLAIN SELECT * FROM orders WHERE user_id = 1\G
-- key_len: 9 ← NULL 허용 +1 byte

-- key_len이 예상보다 짧으면 Composite Index의 중간 컬럼 조건이 누락됐거나
-- 범위 조건 이후 컬럼이 사용되지 않은 것
```

### 3. rows와 filtered — 통계 기반 추정

```sql
-- rows: 이 단계에서 읽을 것으로 예상되는 Row 수 (통계 기반 추정!)
-- filtered: rows 중 WHERE 조건을 통과할 비율 (%)
-- 실제 결과 수 추정 = rows × filtered / 100

EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status = 'PAID'\G
-- rows: 100, filtered: 33.33
-- 예상 결과 수: 100 × 0.3333 ≈ 33건

-- rows가 부정확한 경우:
-- 1. 대규모 INSERT 후 통계 미갱신
-- 2. 데이터 분포가 편중 (status='PAID' = 90%인데 1/3으로 추정)
-- 3. 샘플링 페이지 수 부족

-- rows 오류 진단 (EXPLAIN ANALYZE로 실제 vs 예측 비교):
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1\G
-- -> Index lookup on orders using idx_user_id (user_id=1)
--    (cost=101.0 rows=100)              ← 예측: 100건
--    (actual time=0.3..15.2 rows=5000 loops=1)  ← 실제: 5000건!
-- 
-- rows 예측(100) vs 실제(5000) → 50배 오차
-- → ANALYZE TABLE orders; 로 통계 갱신 필요

-- filtered 값의 의미:
-- 100% → WHERE 조건이 인덱스로 완전히 처리됨 (추가 필터 없음)
-- 낮은 값 → 인덱스로 가져온 후 WHERE로 많이 버림 → 비효율
```

### 4. Extra 컬럼 — 핵심 상세 정보

```sql
-- Extra 주요 값들:

-- Using index (Covering Index):
EXPLAIN SELECT user_id, status FROM orders WHERE user_id = 1\G
-- Extra: Using index ← 인덱스만으로 완결, 테이블 접근 없음

-- Using where:
-- 인덱스로 Row 찾은 후 MySQL 서버 레벨 WHERE 필터 적용
EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND amount > 1000\G
-- (user_id 인덱스만 있고 amount 인덱스 없을 때)
-- Extra: Using where ← amount > 1000은 서버 레벨 필터

-- Using index condition (ICP, Index Condition Pushdown):
-- WHERE 조건 일부를 인덱스 레벨에서 필터
EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status LIKE 'P%'\G
-- (idx_user_status 있을 때)
-- Extra: Using index condition ← status LIKE 'P%'를 인덱스 레벨에서 처리

-- Using filesort:
-- ORDER BY를 인덱스로 처리할 수 없어 별도 정렬 수행
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY amount DESC\G
-- (user_id만 인덱스에 있고 amount는 없을 때)
-- Extra: Using filesort ← amount 기준 정렬 추가 처리 필요

-- Using filesort 발생 조건:
-- 1. ORDER BY 컬럼이 인덱스에 없음
-- 2. ORDER BY 컬럼이 인덱스에 있지만 WHERE 조건과 불연속
-- 3. ORDER BY 방향(ASC/DESC)이 인덱스와 불일치 (MySQL 8.0 이전)

-- Using filesort 제거 방법:
-- ORDER BY 컬럼을 인덱스에 포함
CREATE INDEX idx_user_amount ON orders(user_id, amount DESC);
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY amount DESC\G
-- Extra: (없음) ← filesort 제거!

-- Using temporary:
-- 임시 테이블 생성 필요 (GROUP BY, DISTINCT, UNION, ORDER BY에서)
EXPLAIN SELECT status, COUNT(*) FROM orders GROUP BY status\G
-- (status 인덱스 없을 때)
-- Extra: Using temporary; Using filesort

-- Using temporary가 비싼 이유:
-- 임시 테이블 생성 → 모든 결과 Row를 임시 테이블에 삽입
-- 임시 테이블이 tmp_table_size/max_heap_table_size 초과 시
-- → 메모리 임시 → 디스크 임시 테이블로 전환 → I/O 급증

-- Using temporary 제거:
CREATE INDEX idx_status ON orders(status);
EXPLAIN SELECT status, COUNT(*) FROM orders GROUP BY status\G
-- Extra: Using index ← 인덱스로 그룹핑, 임시 테이블 없음

-- Backward index scan (MySQL 8.0):
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY id DESC\G
-- Extra: Backward index scan ← 역방향 인덱스 순회 (filesort 없음)

-- Using join buffer (Block Nested Loop / Hash Join):
EXPLAIN SELECT * FROM orders o JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE'\G
-- (users.id가 인덱스 없을 때)
-- Extra(orders): Using join buffer (hash join) ← Hash Join 사용
```

### 5. EXPLAIN ANALYZE 상세 해석

```sql
-- EXPLAIN ANALYZE (MySQL 8.0.18+)
-- 실제 실행 후 예측 vs 실제 비교

EXPLAIN ANALYZE
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'PAID'
ORDER BY o.created_at DESC
LIMIT 10\G

-- 출력 예시:
-- -> Limit: 10 row(s)
--    (actual time=15.2..15.3 rows=10 loops=1)
--   -> Sort: orders.created_at DESC
--      (actual time=15.1..15.2 rows=10 loops=1)
--     -> Nested loop inner join
--        (cost=12345.0 rows=33333)    ← 예측
--        (actual time=0.5..14.9 rows=33333 loops=1)  ← 실제
--       -> Index lookup on orders using idx_status (status='PAID')
--          (cost=3456.0 rows=33333)
--          (actual time=0.3..8.5 rows=33333 loops=1)
--       -> Single-row index lookup on users using PRIMARY (id=o.user_id)
--          (cost=0.25 rows=1)
--          (actual time=0.0002..0.0002 rows=1 loops=33333)

-- 각 노드 해석:
-- actual time=X..Y:
--   X = 첫 번째 Row를 반환하는 데 걸린 시간 (ms)
--   Y = 모든 Row를 반환하는 데 걸린 시간 (ms)
-- rows=N: 실제 반환한 Row 수
-- loops=M: 이 노드가 반복 실행된 횟수 (JOIN의 Driven Table은 M = Driving Table 결과 수)

-- 진단 포인트:
-- users의 loops=33333 → users를 33,333번 개별 조회
--   = N+1과 유사한 패턴, 그러나 여기서는 JOIN이므로 정상 (eq_ref)
--   만약 이 비용이 크다면 orders의 결과(33,333)를 줄이는 것이 핵심

-- Sort의 time=15.1 >> Index lookup의 time=8.5:
--   Sort 비용이 JOIN 비용보다 큼
--   → (status, created_at DESC) Composite Index로 Sort 제거 가능
```

---

## 💻 실전 실험

### 실험 1: type 변화 확인

```sql
-- 다양한 type 값 실험
CREATE TABLE explain_test (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status  VARCHAR(20) NOT NULL,
    amount  DECIMAL(10,2),
    email   VARCHAR(100) UNIQUE,
    INDEX idx_user (user_id),
    INDEX idx_status (status)
) ENGINE=InnoDB;

-- 100만 건 삽입 (생략)

-- const: PK/UNIQUE 등치
EXPLAIN SELECT * FROM explain_test WHERE id = 1\G          -- type: const
EXPLAIN SELECT * FROM explain_test WHERE email = 'a@b.com'\G  -- type: const

-- ref: Non-Unique 등치
EXPLAIN SELECT * FROM explain_test WHERE user_id = 100\G   -- type: ref

-- range: 범위 조건
EXPLAIN SELECT * FROM explain_test WHERE id > 100 AND id < 200\G  -- type: range
EXPLAIN SELECT * FROM explain_test WHERE user_id IN (1,2,3)\G     -- type: range

-- index: Full Index Scan (Covering)
EXPLAIN SELECT user_id FROM explain_test ORDER BY user_id\G       -- type: index

-- ALL: Full Table Scan
EXPLAIN SELECT * FROM explain_test WHERE amount > 5000\G          -- type: ALL (amount 인덱스 없음)
```

### 실험 2: Using filesort vs 인덱스 정렬

```sql
-- filesort 발생
EXPLAIN SELECT * FROM explain_test WHERE user_id = 100 ORDER BY amount DESC\G
-- Extra: Using filesort

-- 인덱스로 해결
CREATE INDEX idx_user_amount ON explain_test(user_id, amount DESC);
EXPLAIN SELECT * FROM explain_test WHERE user_id = 100 ORDER BY amount DESC\G
-- Extra: (없음) 또는 Using index condition

-- 성능 측정
SET @start = NOW(6);
SELECT * FROM explain_test WHERE user_id = 100 ORDER BY amount DESC LIMIT 10;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) AS with_filesort_us;

-- 인덱스 추가 후 재측정
SET @start = NOW(6);
SELECT * FROM explain_test WHERE user_id = 100 ORDER BY amount DESC LIMIT 10;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) AS without_filesort_us;
```

### 실험 3: EXPLAIN ANALYZE로 통계 오류 감지

```sql
-- 통계 오류 시뮬레이션
-- 특정 user_id에 대규모 데이터 편중
INSERT INTO explain_test (user_id, status, amount, email)
SELECT 99999, 'PAID', ROUND(RAND()*1000,2), CONCAT('u', seq, '@test.com')
FROM (SELECT SEQ_1_TO_10000().seq) t;
-- user_id=99999인 Row가 갑자기 10,000건 추가

-- 통계 갱신 전
EXPLAIN SELECT * FROM explain_test WHERE user_id = 99999\G
-- rows: 예전 통계 기반 (예: 100) ← 실제는 10,100건

-- EXPLAIN ANALYZE로 실제 vs 예측 비교
EXPLAIN ANALYZE SELECT * FROM explain_test WHERE user_id = 99999\G
-- (cost=.. rows=100)             ← 예측: 100
-- (actual time=.. rows=10100 loops=1)  ← 실제: 10,100 → 101배 오차!

-- 통계 갱신
ANALYZE TABLE explain_test;

-- 재확인
EXPLAIN SELECT * FROM explain_test WHERE user_id = 99999\G
-- rows: 10100 ← 이제 정확함
```

### 실험 4: Using temporary 발생 및 제거

```sql
-- Using temporary + filesort 발생
EXPLAIN SELECT status, COUNT(*) FROM explain_test GROUP BY status\G
-- Extra: Using temporary; Using filesort

-- 임시 테이블 크기 관련 설정 확인
SHOW VARIABLES LIKE '%tmp_table%';
-- tmp_table_size: 16MB (기본값)
-- max_heap_table_size: 16MB
-- 임시 테이블이 이 크기 초과 → 디스크 임시 테이블

-- 인덱스로 해결
-- status에 이미 idx_status가 있다면:
EXPLAIN SELECT status, COUNT(*) FROM explain_test GROUP BY status\G
-- Extra: Using index ← 인덱스로 그룹핑, 임시 테이블 없음!

-- DISTINCT의 Using temporary 제거
EXPLAIN SELECT DISTINCT user_id FROM explain_test\G
-- idx_user가 있으면: Extra: Using index (Covering)
-- idx_user가 없으면: Extra: Using temporary; Using filesort
```

---

## 📊 성능 비교

```
EXPLAIN Extra 값별 성능 영향:

Using index (Covering):
  테이블 접근 0회 → 최고 성능
  I/O: 인덱스 Page만

(Extra 없음, type=ref):
  인덱스 → Double Lookup → 테이블 접근
  I/O: 인덱스 + Clustered Index

Using index condition (ICP):
  인덱스 레벨 필터 → 테이블 접근 건수 감소
  I/O: 인덱스 + 조건 통과한 것만 테이블 접근

Using filesort:
  정렬 버퍼에 결과 수집 → 정렬 → 반환
  메모리 정렬: sort_buffer_size (기본 256KB)
  sort_buffer 초과 시: 디스크 임시 파일 사용 → I/O 추가
  비용: 결과 건수 × log(결과 건수) (비교 연산)

Using temporary:
  임시 테이블 생성 (메모리 또는 디스크)
  tmp_table_size 초과 시 디스크 → 매우 느림
  비용: 전체 결과를 임시 테이블에 쓰고 읽기

비용 순서 (빠름→느림):
  Using index < (없음, type=const) < Using index condition
  < Using where < Using filesort < Using temporary + filesort
```

---

## ⚖️ 트레이드오프

```
EXPLAIN 활용의 한계:

rows는 추정치:
  통계 기반 추정 → 실제와 다를 수 있음
  → EXPLAIN ANALYZE로 실제 값 확인 필요
  → 단, EXPLAIN ANALYZE는 실제 실행 → 부하 발생

filtered는 더 부정확:
  단일 컬럼 통계만 사용 (다중 컬럼 상관관계 무시)
  → filtered=10%이지만 실제 0.1%일 수 있음

EXPLAIN ANALYZE 주의사항:
  쿼리를 실제로 실행 → DELETE/UPDATE에서 조심
  대용량 쿼리는 오래 걸림
  프로덕션에서는 EXPLAIN으로 먼저 확인, 필요 시 ANALYZE

EXPLAIN FORMAT=JSON:
  JSON 출력으로 더 많은 정보 (비용 세부 등)
  파싱이 어려움 → 도구 활용 권장 (explain.depesz.com MySQL 버전)
```

---

## 📌 핵심 정리

```
EXPLAIN 핵심 읽기 순서:

1. type 확인:
   ALL / index → 경고! Full Scan 또는 전체 인덱스 순회
   range → 범위 스캔 (결과 비율에 따라 적절할 수도)
   ref / eq_ref / const → 인덱스 효율적 사용

2. key 확인:
   NULL → 인덱스 미사용 → 인덱스 추가 필요 or 쿼리 수정
   예상한 인덱스가 아닌 경우 → 통계 문제 or 쿼리 패턴 문제

3. rows 확인:
   너무 높으면 결과 이전에 너무 많이 읽음 → Selectivity 문제
   실제와 크게 다르면 → ANALYZE TABLE

4. key_len 확인:
   예상보다 짧으면 Composite Index가 일부만 사용됨
   Leftmost Prefix 위반 여부 확인

5. Extra 확인:
   Using index → 최고 (Covering)
   Using filesort → ORDER BY 컬럼 인덱스 추가 검토
   Using temporary → GROUP BY / DISTINCT 최적화 검토
   Using where → 인덱스 적용 후 추가 필터 (정상적일 수 있음)

EXPLAIN ANALYZE:
  (cost=X rows=Y) vs (actual time=A..B rows=C loops=D)
  rows Y vs actual rows C 비교로 통계 오류 감지
  time A..B: 첫 Row까지 vs 전체 Row까지 소요 시간
```

---

## 🤔 생각해볼 문제

**Q1.** EXPLAIN 결과에서 `type: ALL, rows: 1000000`인 경우와 `type: range, rows: 900000`인 경우 중 어느 것이 더 나쁜가?

<details>
<summary>해설 보기</summary>

**`type: range, rows: 900,000`이 더 나쁠 수 있습니다.** 언뜻 보면 `type: ALL`이 나쁘고 `range`가 좋아 보이지만 `rows`를 함께 봐야 합니다.

- `type: ALL, rows: 1,000,000`: Full Table Scan. 100만 건을 Sequential I/O로 읽음.
- `type: range, rows: 900,000`: 인덱스 Range Scan이지만 90만 건. Secondary Index 90만 건 → 각각 Clustered Index로 Double Lookup → 90만 번의 **Random I/O**. Random I/O가 Sequential I/O보다 HDD에서 100배 이상 비쌀 수 있습니다.

Optimizer는 이 비용을 계산해서 `range`임에도 Full Scan을 선택할 수 있습니다. `type: range`가 항상 좋은 것이 아니며, `rows` 값을 함께 보고 전체 결과의 몇 %인지 판단해야 합니다. 경험칙: 전체의 20~30% 이상이면 Full Scan이 유리할 수 있습니다.

</details>

---

**Q2.** `Extra: Using index; Using where`와 `Extra: Using index condition`의 차이를 InnoDB 동작 레벨에서 설명하라.

<details>
<summary>해설 보기</summary>

**Using index; Using where** (Covering Index + 서버 레벨 필터):
- 인덱스만으로 쿼리가 완결됩니다 (테이블 접근 없음).
- WHERE 조건이 인덱스에서 읽은 결과에 MySQL 서버 레벨에서 추가 필터링됩니다.
- 예: `CREATE INDEX idx_user_amount ON (user_id, amount)` 후 `WHERE user_id = 1 AND amount > 100 AND amount < 1000`. 인덱스에서 user_id=1 범위를 읽고, amount 조건은 서버가 추가로 확인합니다. 테이블 접근은 없습니다.

**Using index condition** (ICP, Index Condition Pushdown):
- WHERE 조건의 일부를 Storage Engine(InnoDB) 레벨에서 평가합니다.
- 테이블 접근이 발생하지만, ICP 덕분에 Clustered Index 접근 횟수가 줄어듭니다.
- 예: `idx_user`만 있을 때 `WHERE user_id = 1 AND status LIKE 'P%'`. user_id=1인 Secondary Index Leaf에서 status LIKE 'P%'를 InnoDB가 먼저 평가. 조건 통과한 것만 Clustered Index로 접근. **테이블 접근은 발생**하지만 모든 user_id=1 Row를 접근하지 않아도 됩니다.

핵심 차이: **Using index**는 테이블 접근 없음, **Using index condition**은 테이블 접근 있지만 최소화.

</details>

---

**Q3.** `EXPLAIN ANALYZE` 결과에서 `loops=10000`인 노드가 있다. 이것은 무엇을 의미하고, 어떤 문제를 나타낼 수 있는가?

<details>
<summary>해설 보기</summary>

`loops=10000`은 해당 노드(Iterator)가 10,000번 반복 호출됐음을 의미합니다. 주로 JOIN의 **Driven Table**에서 발생합니다. Driving Table 결과가 10,000건이면, Driven Table에 대한 접근을 10,000번 반복합니다.

```
-> Nested loop join (loops=1)
   -> Index scan on orders WHERE status='PAID'   (rows=10000 loops=1)
   -> Index lookup on users by id (loops=10000)  ← 10000번 반복!
      (cost=0.25 rows=1 loops=10000)
```

**문제가 되는 경우**: Driven Table의 각 lookup이 **인덱스 없이** 발생하면 10,000번의 Full Scan = 10,000 × 전체 Row 수의 I/O. 또는 loops × actual rows가 매우 크면 처리량이 급증합니다.

**해결 방법**:
1. Driven Table에 JOIN 컬럼으로 인덱스 추가 → 각 lookup 비용 감소
2. Driving Table의 결과를 줄임 (WHERE 조건 추가, 더 선택적 인덱스)
3. Hash Join: 대량 데이터 JOIN에서 Nested Loop 대신 Hash Join이 더 효율적

`loops=10000`이 나오면 반드시 Driving Table과 Driven Table의 관계와 각 접근 비용을 확인해야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 쿼리 실행 단계](./01-query-execution-stages.md)** | **[홈으로 🏠](../README.md)** | **[다음: Cost-Based Optimizer ➡️](./03-cost-based-optimizer.md)**

</div>
