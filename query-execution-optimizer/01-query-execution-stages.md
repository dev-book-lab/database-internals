# 쿼리 실행 단계 — Parse → Optimize → Execute

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- SQL 문자열이 MySQL에 도달한 순간부터 결과가 반환되기까지 몇 단계를 거치는가?
- Parse Tree와 Query Tree는 어떻게 다르고, 각각 무엇을 나타내는가?
- Logical Plan을 Physical Plan으로 변환하는 Optimization에서 무슨 결정이 내려지는가?
- `Handler_read_*` 상태 변수는 각각 어떤 B+Tree 탐색 동작에 대응하는가?
- 같은 결과를 반환하는 쿼리라도 실행 경로가 다를 수 있는 이유는?
- 쿼리 캐시는 MySQL 8.0에서 왜 제거됐는가?

---

## 🔍 왜 이 개념이 중요한가

### "인덱스는 있는데 왜 느리지?"를 정확히 진단하려면

```
인덱스와 실행 계획을 이해하는 두 가지 수준:

수준 1 (일반): EXPLAIN을 보고 type, key를 확인
수준 2 (깊은): 왜 그 실행계획이 선택됐는지, 어떻게 실행되는지 이해

수준 2가 필요한 상황:
  EXPLAIN: type=ref, key=idx_user 인데 왜 여전히 느린가?
  → 예상 rows=100, 실제 rows=50,000 → 통계 오류
  → Handler_read_next가 50,000 → 실제로 50,000 Row를 읽음
  → 인덱스는 올바르게 선택됐지만 결과 건수가 예상보다 많음

  이 진단을 하려면:
    쿼리가 Execute 단계에서 어떻게 Row를 읽는지
    Handler 변수가 각 B+Tree 연산에 어떻게 대응하는지
    알아야 한다

실행 단계 이해의 실용적 가치:
  Slow Query의 원인이 Parse/Optimize 단계인지 Execute 단계인지 분리
  Handler 변수로 인덱스 사용 패턴을 EXPLAIN 없이도 파악
  실행계획 선택의 기반인 통계 정보를 이해 → 통계 관리 중요성 인식
```

---

## 🔬 내부 동작 원리

### 1. MySQL 서버 아키텍처와 쿼리 경로

```
클라이언트 → MySQL 서버 쿼리 처리 경로:

┌────────────────────────────────────────────────────────┐
│                    MySQL Server                        │
│                                                        │
│  ① Connection Layer                                    │
│     TCP 연결 수락, 인증, 스레드 할당                         │
│                                                        │
│  ② SQL Layer (Query Processing)                        │
│     Parser → Preprocessor → Optimizer → Executor       │
│                                                        │
│  ③ Storage Engine Layer (InnoDB)                       │
│     Handler API → B+Tree 탐색 → Buffer Pool → 디스크      │
│                                                        │
└────────────────────────────────────────────────────────┘

SQL Layer ↔ InnoDB 인터페이스:
  Executor는 Handler API를 통해 InnoDB를 호출
  Handler API 주요 메서드:
    ha_index_init()      : 인덱스 스캔 시작
    ha_index_read()      : 특정 키로 첫 번째 Row 읽기
    ha_index_next()      : 다음 Row 읽기 (순방향)
    ha_index_prev()      : 이전 Row 읽기 (역방향)
    ha_index_first()     : 인덱스 첫 번째 Row
    ha_rnd_init()        : Full Table Scan 시작
    ha_rnd_next()        : 다음 Row (Full Scan)
    ha_write_row()       : Row 삽입
    ha_update_row()      : Row 수정

각 Handler 호출 = Handler_read_* 상태 변수 1씩 증가
```

### 2. 쿼리 실행 5단계 상세

```
SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC LIMIT 10;

단계 1: Connection & Parsing
  ─────────────────────────────
  Lexical Analysis (어휘 분석):
    SQL 문자열 → Token 스트림
    "SELECT", "*", "FROM", "orders", "WHERE", ...
  
  Syntax Analysis (구문 분석):
    Token 스트림 → Parse Tree (AST)
    문법 오류 감지 (테이블명, 컬럼명은 아직 검증 안 함)
  
  Parse Tree:
    SelectStatement
      ├─ SelectList: [*]
      ├─ FromClause: orders
      ├─ WhereClause: user_id = 1
      ├─ OrderByClause: created_at DESC
      └─ LimitClause: 10

단계 2: Preprocessing (전처리)
  ─────────────────────────────
  Parse Tree → Query Tree (의미 있는 트리)
  
  수행 작업:
    테이블/컬럼 존재 확인 (information_schema 조회)
    권한 검사 (user_id 컬럼에 접근 권한 있는가?)
    * → 실제 컬럼 목록으로 확장 (id, user_id, amount, status, created_at)
    뷰(View)를 실제 쿼리로 확장
    서브쿼리 구조 분석
  
  출력: Resolved Query Tree (컬럼과 테이블이 확정된 트리)

단계 3: Optimization (최적화)
  ─────────────────────────────
  Resolved Query Tree → Physical Execution Plan
  
  수행 작업:
    A. Logical Transformations (논리 변환):
       서브쿼리 → JOIN으로 변환 가능한지 확인
       상수 폴딩: WHERE 1+1=2 → TRUE
       불필요 조건 제거: WHERE TRUE AND user_id = 1 → WHERE user_id = 1
       Predicate Pushdown: 조건을 최대한 이른 단계에서 필터링
    
    B. Join 순서 결정 (멀티 테이블인 경우):
       테이블 N개 → N! 가능한 Join 순서 탐색
       비용 기반으로 최적 순서 선택
       N이 크면 Greedy 알고리즘으로 탐색 제한
    
    C. Access Method 결정 (각 테이블):
       Full Table Scan vs Index Scan 비용 비교
       어느 인덱스를 사용할지 결정
       통계 정보(Cardinality, Page 수)로 예상 비용 계산
    
    D. Physical Plan 생성:
       최저 비용 실행계획 선택
       Iterator 트리 구성 (MySQL 8.0 Iterator Model)
  
  출력: Physical Execution Plan (EXPLAIN이 보여주는 것)

단계 4: Execution (실행)
  ─────────────────────────────
  Physical Execution Plan → 실제 Row 읽기
  
  MySQL 8.0 Iterator Model:
    각 Plan Node = Iterator (행을 하나씩 반환하는 인터페이스)
    부모 Iterator가 자식 Iterator에게 next() 호출
    Pull-based 실행 모델 (Volcano/Iterator Model)
  
  예시:
    LimitIterator(10)
      └─ SortIterator(created_at DESC)
            └─ IndexLookupIterator(idx_user_id, user_id=1)
                  └─ InnoDB(orders)
  
  LimitIterator.next() 호출 →
    SortIterator.next() 호출 →
      IndexLookupIterator.next() →
        ha_index_read() / ha_index_next() (InnoDB 호출) →
        Row 반환 →
      Sort Buffer에 누적 →
    Sort 완료 후 정렬 순서로 반환 →
  10번 반환 후 완료

단계 5: Result Return
  ─────────────────────────────
  실행 결과를 클라이언트에게 전송
  Protocol 버퍼에 Row 직렬화 → 네트워크 전송
```

### 3. Handler_read_* 상태 변수 완전 해석

```sql
-- Handler 변수 확인
FLUSH STATUS;  -- 현재 세션 통계 초기화
SELECT * FROM orders WHERE user_id = 1;
SHOW SESSION STATUS LIKE 'Handler_read%';

-- 각 변수의 의미:

Handler_read_first:
  ha_index_first() 호출 횟수
  인덱스의 첫 번째 Row를 읽는 횟수
  → Full Index Scan 시작 시 1 증가
  → "SELECT * FROM orders ORDER BY id" (인덱스 전체 순회)

Handler_read_key:
  ha_index_read() 호출 횟수
  특정 키 값으로 인덱스를 탐색하는 횟수
  → WHERE user_id = 1 (등치 조건으로 인덱스 탐색 시작)
  → 탐색 시작점을 찾을 때마다 1 증가
  → 높은 값 = 인덱스 Point Lookup을 많이 함 (좋은 신호)

Handler_read_next:
  ha_index_next() 호출 횟수
  인덱스에서 다음 Row를 순방향으로 읽는 횟수
  → Range Scan, ORDER BY ASC
  → Handler_read_key로 시작점 찾은 후 여기서 순회
  → user_id=1인 Row가 100개이면 +100
  → 매우 높은 값 = 넓은 범위 스캔 (주의 필요)

Handler_read_prev:
  ha_index_prev() 호출 횟수
  인덱스에서 이전 Row를 역방향으로 읽는 횟수
  → ORDER BY DESC
  → WHERE id > 100 ORDER BY id DESC

Handler_read_rnd:
  ha_rnd_read() 호출 횟수 (랜덤 Row 읽기)
  → Full Table Scan에서 임의 위치의 Row 읽기
  → 또는 filesort 후 임시 테이블에서 Row 읽기
  → 높은 값 = Random I/O 많음 (성능 주의)

Handler_read_rnd_next:
  ha_rnd_next() 호출 횟수
  Full Table Scan에서 다음 Row를 순차적으로 읽기
  → SELECT * FROM orders (인덱스 없이 Full Scan)
  → 값 = 전체 Row 수
  → 높은 값 = Full Scan 발생 (인덱스 누락 신호)

Handler_read_last:
  ha_index_last() 호출 횟수
  인덱스의 마지막 Row를 읽기
  → ORDER BY DESC + LIMIT 1 최적화 시 사용

진단 패턴:
  Handler_read_key: 1, Handler_read_next: 100
    → 인덱스 Point Lookup(1회) + 100건 순회 → idx 사용 중
  
  Handler_read_rnd_next: 1,000,000
    → Full Scan 100만 건 → 인덱스 미사용!
  
  Handler_read_key: 1000, Handler_read_rnd: 1000
    → 1000번의 랜덤 읽기 → Double Lookup (Secondary → Clustered)
```

### 4. 쿼리 캐시 제거 (MySQL 8.0)

```
MySQL 5.7까지: Query Cache 존재
  동일한 SQL 문자열 + 동일한 조건 → 캐싱된 결과 즉시 반환
  
  문제점:
    ① 캐시 무효화 비용:
       테이블에 INSERT/UPDATE/DELETE가 발생하면
       해당 테이블을 참조하는 모든 캐시 항목 즉시 삭제
       → 쓰기 집약적 테이블에서 캐시 HitRate ≈ 0
       → 캐시 삭제 오버헤드만 발생
    
    ② 캐시 경합 (Global Mutex):
       쿼리 캐시 조회/저장에 전역 뮤텍스 사용
       → 동시 접속자 많을수록 경합 심화
       → 고부하 환경에서 오히려 성능 저하
    
    ③ 메모리 단편화:
       다양한 크기의 결과셋 → 캐시 메모리 단편화
       → 효율 저하

MySQL 8.0: Query Cache 완전 제거
  이유: 위 문제들로 인해 대부분의 실제 환경에서 비활성화 권장
        ProxySQL, Redis 등 애플리케이션 레벨 캐시로 대체 권장

현대적 캐싱 전략:
  애플리케이션 캐시 (Redis, Caffeine):
    캐시 무효화 정책을 애플리케이션에서 세밀하게 제어
    결과셋이 크더라도 직렬화 후 캐시 가능
  
  Connection Pooling + Prepared Statement:
    Parse + Preprocessing 단계 재사용
    → 반복 쿼리의 Parse 비용 절약
```

### 5. Prepared Statement와 실행 단계 재사용

```java
// JDBC Prepared Statement
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM orders WHERE user_id = ? AND status = ?"
);
// 최초 실행: Parse → Preprocess → Optimize → Execute
// 이후 실행 (파라미터만 변경):
//   Execute만 재실행 (Parse, Preprocess, Optimize 생략)
ps.setLong(1, userId);
ps.setString(2, "PAID");
ps.executeQuery();

// JPA/Hibernate의 Prepared Statement:
// spring.jpa.properties.hibernate.jdbc.batch_size=50
// → 배치 INSERT 시 Prepared Statement 재사용으로 Parse 비용 절약

// 서버 사이드 Prepared Statement (MySQL 프로토콜 레벨):
// useServerPrepStmts=true (JDBC URL에 설정)
// → MySQL 서버에 Parse Tree를 캐시
// → 동일 쿼리 템플릿의 반복 호출에서 Parse 단계 완전 생략

// 주의: Optimizer는 매 실행마다 또는 파라미터에 따라 재최적화할 수 있음
// MySQL: Prepare 시 1회 최적화, Execute 시 재최적화 없음 (파라미터 추정 문제 있을 수 있음)
```

---

## 💻 실전 실험

### 실험 1: Handler 변수로 실행 경로 추적

```sql
-- 실험 준비
CREATE TABLE handler_test (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status  VARCHAR(20),
    amount  DECIMAL(10,2),
    INDEX idx_user (user_id)
) ENGINE=InnoDB;

INSERT INTO handler_test (user_id, status, amount)
SELECT FLOOR(RAND()*1000)+1, ELT(FLOOR(RAND()*3)+1,'PAID','PENDING','CANCELLED'),
       ROUND(RAND()*10000, 2)
FROM information_schema.columns a CROSS JOIN information_schema.columns b
LIMIT 100000;

-- 실험 A: 인덱스 Point Lookup
FLUSH STATUS;
SELECT * FROM handler_test WHERE user_id = 500;
SHOW SESSION STATUS LIKE 'Handler_read%';
-- Handler_read_key: 1 (시작점 탐색 1회)
-- Handler_read_next: ~100 (user_id=500인 Row 수만큼)
-- Handler_read_rnd_next: 0 (Full Scan 없음)

-- 실험 B: Full Table Scan
FLUSH STATUS;
SELECT * FROM handler_test WHERE status = 'PAID';  -- status 인덱스 없음
SHOW SESSION STATUS LIKE 'Handler_read%';
-- Handler_read_rnd_next: ~100,000 (전체 Row 수)
-- Handler_read_key: 0 (인덱스 미사용)

-- 실험 C: 범위 스캔
FLUSH STATUS;
SELECT * FROM handler_test WHERE user_id BETWEEN 100 AND 200;
SHOW SESSION STATUS LIKE 'Handler_read%';
-- Handler_read_key: 1 (user_id=100 시작점)
-- Handler_read_next: ~10,100 (user_id 100~200 사이 Row 수)

-- 실험 D: ORDER BY DESC
FLUSH STATUS;
SELECT * FROM handler_test WHERE user_id = 500 ORDER BY id DESC LIMIT 10;
SHOW SESSION STATUS LIKE 'Handler_read%';
-- Handler_read_prev: ~10 (역방향 읽기)
```

### 실험 2: Optimizer Trace로 최적화 과정 보기

```sql
-- Optimizer가 어떤 과정으로 실행계획을 선택했는지 추적
SET SESSION optimizer_trace = "enabled=on";

SELECT * FROM handler_test WHERE user_id = 500 AND status = 'PAID';

SELECT * FROM information_schema.OPTIMIZER_TRACE\G
-- 출력 구조:
-- {
--   "steps": [
--     {
--       "join_preparation": {  ← Preprocessing 단계
--         "select#": 1,
--         "steps": [...]
--       }
--     },
--     {
--       "join_optimization": {  ← Optimization 단계
--         "select#": 1,
--         "steps": [
--           {"rows_estimation": [...]},    ← 각 인덱스별 예상 Row 수
--           {"considered_execution_plans": [  ← 고려한 실행계획들
--             {"plan_prefix": [], "table": "handler_test",
--              "best_access_path": {"considered_access_paths": [
--                {"access_type": "ref", "index": "idx_user", "rows": 100, "cost": 101},
--                {"access_type": "scan", "rows": 100000, "cost": 10001}
--              ]}}
--           ]},
--           {"attaching_conditions_to_tables": [...]}
--         ]
--       }
--     },
--     {"join_execution": {...}}  ← Execute 단계
--   ]
-- }

-- Trace 비활성화
SET SESSION optimizer_trace = "enabled=off";
```

### 실험 3: 쿼리 단계별 시간 측정

```sql
-- Performance Schema로 단계별 시간 측정 (MySQL 8.0)
-- events_statements_history에서 확인

UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%';

-- 쿼리 실행
SELECT * FROM handler_test WHERE user_id = 500;

-- 최근 쿼리의 단계별 시간 확인
SELECT
    EVENT_NAME,
    TIMER_WAIT / 1e9 AS wait_ms,
    LOCK_TIME / 1e9 AS lock_ms,
    ROWS_SENT,
    ROWS_EXAMINED,
    NO_INDEX_USED,
    NO_GOOD_INDEX_USED
FROM performance_schema.events_statements_history
WHERE SQL_TEXT LIKE '%handler_test%'
ORDER BY TIMER_START DESC
LIMIT 5;

-- NO_INDEX_USED = 1: 인덱스 미사용 (Full Scan)
-- NO_GOOD_INDEX_USED = 1: 인덱스는 있지만 비효율적으로 사용
-- ROWS_EXAMINED >> ROWS_SENT: 많이 읽고 적게 반환 (비효율)
```

---

## 📊 성능 비교

```
Handler 변수로 읽는 쿼리 효율:

쿼리: SELECT * FROM orders WHERE user_id = 1 (user_id=1인 Row 100건)

인덱스 있음:
  Handler_read_key: 1
  Handler_read_next: 100
  Handler_read_rnd: 100  (Double Lookup)
  총 I/O 작업: ~201회

인덱스 없음 (Full Scan):
  Handler_read_rnd_next: 1,000,000
  총 I/O 작업: ~1,000,000회

차이: 약 5,000배!

Rows Examined vs Rows Sent:
  인덱스 사용: Examined ≈ Sent (≈ 100)
  Full Scan: Examined = 1,000,000, Sent = 100
  → Examined/Sent 비율 = 10,000:1 → 비효율 지표
  → 이 비율이 크면 인덱스 추가 또는 쿼리 개선 필요
```

---

## ⚖️ 트레이드오프

```
실행 단계별 최적화 포인트:

Parse/Preprocess:
  비용: 작음 (메모리 연산)
  최적화: Prepared Statement로 재사용
  영향: 초당 수만 쿼리 환경에서 의미 있음

Optimization:
  비용: 중간 (테이블 수 증가시 기하급수)
  제한: join_reorder_limit (기본 8) 이상 테이블은 Greedy
  영향: 복잡한 JOIN 쿼리에서 최적 계획 못 찾을 수 있음

Execution (Handler 레벨):
  비용: 가장 큼 (실제 I/O 발생)
  최적화: 인덱스 설계, Buffer Pool 크기, SSD
  영향: 전체 쿼리 시간의 95% 이상을 차지
  진단: Handler_read_* 변수로 정밀 측정

단계별 문제 진단:
  Parse 오류: Error message (문법 오류)
  Preprocess 오류: "Table doesn't exist", "Unknown column"
  Optimize 문제: EXPLAIN rows가 실제와 크게 다름, 잘못된 인덱스 선택
  Execute 문제: Handler_read_rnd_next 높음, Rows Examined >> Sent
```

---

## 📌 핵심 정리

```
쿼리 실행 단계 핵심:

5단계:
  Connection → Parse → Preprocess → Optimize → Execute → Return
  
  Parse: SQL → Parse Tree (문법만 확인)
  Preprocess: 테이블/컬럼 확인, 권한 검사, * 확장
  Optimize: 최저 비용 실행계획 선택 (어떤 인덱스, 어떤 JOIN 순서)
  Execute: Handler API로 InnoDB 호출, 실제 Row 읽기

Handler_read_* 해석:
  read_key: 인덱스 탐색 시작 횟수 (높을수록 좋음)
  read_next: 순방향 연속 읽기 (범위 스캔 크기 반영)
  read_prev: 역방향 연속 읽기 (ORDER BY DESC)
  read_rnd_next: Full Scan Row 수 (높으면 인덱스 미사용!)

Query Cache 제거 (8.0):
  쓰기 집약적 환경에서 오히려 해로움
  Global Mutex로 경합
  → 애플리케이션 캐시(Redis)로 대체

Prepared Statement:
  Parse/Preprocess/Optimize 재사용
  동일 쿼리 템플릿 반복 시 성능 향상
  JPA: Hibernate는 기본적으로 Prepared Statement 사용
```

---

## 🤔 생각해볼 문제

**Q1.** `FLUSH STATUS` 후 `SELECT * FROM orders`를 실행했을 때 `Handler_read_key: 0`, `Handler_read_rnd_next: 1,000,000`이 나왔다. 이 정보만으로 어떤 진단을 내릴 수 있는가?

<details>
<summary>해설 보기</summary>

`Handler_read_rnd_next: 1,000,000`은 Full Table Scan으로 100만 건을 전부 읽었음을 의미합니다. `Handler_read_key: 0`은 인덱스 탐색이 전혀 없었음을 확인합니다.

진단: 이 쿼리는 100% Full Scan으로 처리됐습니다. 이유는 두 가지 중 하나입니다:
1. WHERE 조건이 없어서 전체를 읽어야 함 (`SELECT *`이므로)
2. WHERE 조건이 있었는데 인덱스가 없거나 인덱스 사용이 불가능한 조건

`SELECT *`에 WHERE 절이 없다면 Full Scan은 피할 수 없습니다. WHERE 절이 있는데 Full Scan이라면 EXPLAIN으로 인덱스 미사용 원인을 파악하고 인덱스를 추가하거나 쿼리를 수정해야 합니다.

추가 조사: `SHOW SESSION STATUS LIKE 'Rows%'`로 `Rows_examined: 1,000,000`, `Rows_sent: N`을 확인. N이 1,000,000이면 전체를 반환하는 쿼리, N이 작으면 필터링이 일어났지만 인덱스가 없는 것입니다.

</details>

---

**Q2.** JPA에서 N+1 문제가 발생할 때 `Handler_read_key`와 `Handler_read_next` 값은 어떻게 나타나는가?

<details>
<summary>해설 보기</summary>

N+1 문제는 주 엔티티 1건을 조회 후 연관 엔티티를 N번 개별 조회하는 패턴입니다. 예: 주문 100건 조회 후 각 주문의 주문자 정보를 100번 별도 쿼리.

Handler 변수 패턴:
- **주 쿼리** (orders 1회): `Handler_read_rnd_next` 또는 `Handler_read_next`로 100건 읽기
- **연관 쿼리** (users 100회): 각 쿼리마다 `Handler_read_key: 1` → 총 `Handler_read_key: 100`

특징적 패턴: `Handler_read_key`가 결과 건수와 동일하게 높은 값 (100). 각 PK lookup이 별개의 세션에서 일어나므로 세션 통계로는 개별 확인이 어렵고, performance_schema의 `events_statements_history_long`에서 100개의 동일한 쿼리 패턴을 발견할 수 있습니다.

해결: `@EntityGraph`, `fetch join`, `@BatchSize`, Spring Data JPA의 `@Query`로 JOIN으로 단일 쿼리 처리 → `Handler_read_key: 1`, `Handler_read_next: 100`으로 개선.

</details>

---

**Q3.** `EXPLAIN FORMAT=TREE`와 `EXPLAIN ANALYZE`의 차이는 무엇인가? 언제 각각을 사용하는가?

<details>
<summary>해설 보기</summary>

**EXPLAIN FORMAT=TREE**: 실제 실행하지 않고 Optimizer가 선택한 실행계획을 트리 형태로 표시. 각 노드의 예상 비용과 Row 수를 보여줍니다. 쿼리를 실행하지 않으므로 프로덕션에서 영향 없이 빠르게 실행계획 확인 가능.

**EXPLAIN ANALYZE** (MySQL 8.0.18+): 실제로 쿼리를 실행하면서 각 Iterator 노드의 실제 실행 시간, 실제 Row 수, 루프 수를 측정. 예측(estimated) vs 실제(actual)를 비교하여 통계 오류를 발견할 수 있습니다.

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1\G
-- 출력 예시:
-- -> Index lookup on orders using idx_user_id (user_id=1)
--    (cost=101.0 rows=100)  ← 예측
--    (actual time=0.5..10.2 rows=5000 loops=1)  ← 실제!
-- rows 예측 100 vs 실제 5000 → 통계 오류! ANALYZE TABLE 필요
```

**사용 기준**:
- `EXPLAIN`: 쿼리 실행 없이 계획 확인 (일상적 사용)
- `EXPLAIN FORMAT=TREE`: 복잡한 Join/서브쿼리 구조를 트리로 시각적으로 파악
- `EXPLAIN ANALYZE`: 실제 vs 예측 비교로 통계 오류, 비효율 노드 발견 (느린 쿼리 디버깅)
- 주의: EXPLAIN ANALYZE는 실제로 쿼리를 실행하므로 DML에서는 사용 시 트랜잭션 관리 필요

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: EXPLAIN 읽는 법 ➡️](./02-explain-analyze.md)**

</div>
