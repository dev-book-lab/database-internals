# 대용량 테이블 최적화 — 파티셔닝과 샤딩

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Partition Pruning이 스캔 범위를 줄이는 구체적 원리는?
- Range / List / Hash Partition의 각 사용 케이스와 차이는?
- 파티션 키를 WHERE 조건에 포함하지 않으면 왜 전체 파티션을 스캔하는가?
- 파티션과 인덱스를 함께 사용할 때 인덱스가 파티션 경계에서 어떻게 동작하는가?
- 수직 샤딩과 수평 샤딩의 정확한 차이와 각각이 해결하는 문제는?
- 샤딩 후 JOIN이 어려워지는 이유와 애플리케이션 레벨 해결책은?

---

## 🔍 왜 이 개념이 중요한가

### 단일 테이블이 5억 건을 넘으면 인덱스도 느려진다

```
인덱스가 있어도 느린 이유:
  B-Tree 높이 = ceil(log_fan-out(N))
  N=1억 건, fan-out=1000:
    높이 = ceil(log_1000(1억)) = ceil(100,000,000 / 1000 / 1000) = 약 3단계

  N=10억 건:
    높이 = 약 4단계 → Root → Branch1 → Branch2 → Branch3 → Leaf
    한 번의 PK 조회 = 4번의 디스크 I/O

  하지만 실제 문제:
    Leaf Node = 수백만 개
    Buffer Pool에 모두 캐시 불가
    → 자주 접근하지 않는 과거 데이터는 항상 디스크 I/O
    → 테이블 크기 ∝ 접근 불가 Leaf Node 비율

파티셔닝 해결책:
  "이번 달 데이터만 파티션 P_2024_03에 있다"
  → WHERE created_at >= '2024-03-01' 조건 → P_2024_03만 스캔
  → 나머지 파티션 = 스캔 없음 (Partition Pruning)
  → 실제 스캔 대상: 전체의 1/36 (3년치 데이터 중 1개월)

샤딩 해결책:
  단일 DB 한계 초과 시 → 여러 DB로 분산
  각 샤드: 전체 데이터의 1/N → 각 테이블 크기 1/N
```

---

## 😱 잘못된 이해

### Before: "파티셔닝 = 자동으로 모든 쿼리가 빨라진다"

```sql
-- 잘못된 기대:
-- "파티셔닝하면 모든 SELECT가 자동으로 빨라진다"

-- 파티션 설정:
CREATE TABLE orders (
    id BIGINT,
    user_id BIGINT,
    created_at DATE
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- 잘못된 기대를 하는 쿼리:
SELECT * FROM orders WHERE user_id = 12345;
-- 기대: "파티셔닝했으니까 빠를 것"
-- 실제: WHERE 조건에 파티션 키(created_at/YEAR)가 없음
-- → 모든 파티션 스캔 (Partition Pruning 없음!)
-- → 파티셔닝이 오히려 오버헤드 추가

-- 올바른 파티셔닝 활용:
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2024-04-01';
-- ← 파티션 키 포함 → p2024만 스캔 (Pruning!)

-- 파티셔닝의 함정:
-- 파티션 키가 WHERE에 없으면 = 전체 파티션 스캔
-- = 파티션 없는 것보다 오히려 느릴 수 있음
-- (메타데이터 오버헤드 때문)
```

---

## ✨ 올바른 이해

### After: 파티셔닝 = 파티션 키 기반 쿼리에서만 효과, 샤딩 = 물리적 분산

```
파티셔닝 (단일 MySQL 인스턴스):
  논리적 분리: 하나의 테이블처럼 보이지만 내부 물리적 파일 분리
  효과 조건: WHERE에 파티션 키 포함 → Partition Pruning
  주 용도:
    시계열 데이터 관리 (월별 파티션으로 오래된 데이터 DROP)
    특정 범위 쿼리 최적화 (최근 1개월 조회)
    대용량 테이블의 통계 쿼리 가속

샤딩 (여러 MySQL 인스턴스):
  물리적 분산: 데이터가 여러 서버에 분산
  효과: 쓰기 처리량, 저장 용량, 읽기 처리량 모두 분산
  비용:
    JOIN 불가 (다른 서버에 데이터 있음)
    트랜잭션 불가 (분산 트랜잭션 필요)
    운영 복잡도 대폭 증가
  주 용도: 단일 DB가 한계에 달했을 때만 (최후의 수단)

선택 기준:
  단계 1: 인덱스 최적화 + 쿼리 최적화
  단계 2: Read Replica로 읽기 분산
  단계 3: 파티셔닝 (시계열 데이터)
  단계 4: 수직 샤딩 (테이블 분리, 다른 DB 서버)
  단계 5: 수평 샤딩 (행 분리, 여러 DB 서버)
  → 각 단계에서 실제 한계에 부딪혔을 때만 다음 단계
```

---

## 🔬 내부 동작 원리

### 1. Range / List / Hash Partition 비교

```sql
-- Range Partition (범위 기반 — 시계열 데이터에 최적):
CREATE TABLE orders (
    id       BIGINT AUTO_INCREMENT,
    order_date DATE NOT NULL,
    amount   DECIMAL(10,2),
    PRIMARY KEY (id, order_date)  -- 파티션 키를 PK에 포함
) ENGINE=InnoDB
PARTITION BY RANGE (TO_DAYS(order_date)) (
    PARTITION p2022 VALUES LESS THAN (TO_DAYS('2023-01-01')),
    PARTITION p2023 VALUES LESS THAN (TO_DAYS('2024-01-01')),
    PARTITION p2024 VALUES LESS THAN (TO_DAYS('2025-01-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
-- 사용 케이스: 주문 내역, 로그, 이벤트 데이터 (시간 기반)
-- 장점: 오래된 파티션 DROP으로 효율적 데이터 정리
--   DROP PARTITION p2022 → 인덱스 포함 순간 삭제 (DELETE 수백만 건보다 빠름)
-- 쿼리:
--   WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31'
--   → p2024 파티션만 스캔 (Pruning)

-- List Partition (목록 기반 — 카테고리 데이터):
CREATE TABLE products (
    id       BIGINT,
    category TINYINT NOT NULL
) ENGINE=InnoDB
PARTITION BY LIST (category) (
    PARTITION p_electronics VALUES IN (1, 2, 3),
    PARTITION p_clothing      VALUES IN (4, 5),
    PARTITION p_books         VALUES IN (6, 7, 8),
    PARTITION p_other         VALUES IN (9, 10, 11, 12)
);
-- 사용 케이스: 카테고리, 지역, 상태값으로 분리
-- 쿼리:
--   WHERE category IN (1, 2) → p_electronics만 스캔
-- 단점: 새 카테고리 추가 시 파티션 재구성 필요

-- Hash Partition (균등 분산 — 부하 분산):
CREATE TABLE user_actions (
    id      BIGINT,
    user_id BIGINT NOT NULL
) ENGINE=InnoDB
PARTITION BY HASH(user_id)
PARTITIONS 8;  -- 8개 파티션
-- 사용 케이스: 특정 범위 쿼리 없고 균등 분산이 목적
-- 쿼리:
--   WHERE user_id = 12345 → 12345 % 8 = 5 → partition 5만 스캔
-- 단점: 파티션 키(user_id)가 정확히 맞아야 Pruning
--        범위 조건(user_id BETWEEN ...)은 Pruning 안 됨
```

### 2. Partition Pruning 상세

```sql
-- Partition Pruning 확인:
EXPLAIN SELECT * FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31'\G
-- partitions: p2024  ← 이것이 Pruning! p2022, p2023은 스캔 안 함

-- Pruning 안 되는 케이스:
EXPLAIN SELECT * FROM orders WHERE user_id = 12345\G
-- partitions: p2022,p2023,p2024,p_future  ← 모든 파티션 스캔!

-- 파티션 + 인덱스 조합:
ALTER TABLE orders ADD INDEX idx_user_id (user_id);
-- 각 파티션마다 독립적인 인덱스 생성됨
-- user_id = 12345 조회:
--   파티션별 idx_user_id로 검색 → 각 파티션에서 인덱스 사용
--   하지만 파티션 Pruning 없음 → 여전히 모든 파티션 인덱스 조회
--   글로벌 인덱스 없음 (MySQL에서 지원 안 함)

-- 파티션 관리 (시계열 데이터의 핵심 장점):
-- 2022년 데이터 삭제:
ALTER TABLE orders DROP PARTITION p2022;
-- → DELETE FROM orders WHERE order_date < '2023-01-01' 대비:
--   DELETE: 수백만 건 × 인덱스 갱신 = 매우 느림 (수분~수시간)
--   DROP PARTITION: 파일 삭제 = 순간 완료 (< 1초)

-- 새 파티션 추가:
ALTER TABLE orders REORGANIZE PARTITION p_future INTO (
    PARTITION p2025 VALUES LESS THAN (TO_DAYS('2026-01-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 3. 인덱스 파티셔닝의 제약사항

```sql
-- MySQL 파티셔닝의 인덱스 제약:

-- 1. 글로벌 인덱스 없음:
--    각 파티션이 독립적인 Local Index 보유
--    → user_id 인덱스 = 파티션마다 별도 인덱스
--    → user_id = 12345 → 모든 파티션의 user_id 인덱스 조회

-- 2. 파티션 키가 PK에 포함되어야 함 (UNIQUE 제약):
--    PRIMARY KEY (id, order_date) ← order_date가 파티션 키
--    PRIMARY KEY (id) 만으로는 PARTITION BY RANGE(order_date) 불가
--    에러: "Field in list of fields for partition function not found in table"

-- 3. 외래키 사용 불가:
--    파티션 테이블은 FOREIGN KEY 지원 안 함
--    → 참조 무결성을 애플리케이션 레벨에서 관리해야 함

-- 4. 파티셔닝 + ORDER BY 성능:
--    파티션 내에서는 정렬됨
--    여러 파티션 걸치면 각 파티션의 결과를 다시 정렬 (오버헤드)

-- 5. 파티션 수 제한:
--    최대 8192개 파티션
--    파티션 수가 많을수록 메타데이터 오버헤드 증가
--    월별 파티션: 10년 = 120개 (OK)
--    일별 파티션: 10년 = 3650개 (OK이지만 관리 복잡)
```

### 4. 수직 샤딩 vs 수평 샤딩

```
수직 샤딩 (Vertical Sharding) — 테이블/컬럼 단위 분리:

  원본 (단일 DB):
    DB: orders, order_items, users, products, inventory, payments
  
  수직 샤딩 적용:
    주문 DB:   orders, order_items
    사용자 DB: users, user_preferences, user_addresses
    상품 DB:   products, inventory, categories
    결제 DB:   payments, refunds
  
  장점:
    각 DB의 테이블 수/크기 감소
    서비스별 독립 스케일링 (결제 DB만 스케일 업)
    서로 다른 스토리지 엔진/설정 가능
    마이크로서비스 아키텍처와 자연스럽게 결합
  
  단점:
    Cross-DB JOIN 불가 → 애플리케이션 레벨 JOIN 필요
    Cross-DB 트랜잭션 불가 → 분산 트랜잭션 또는 보상 트랜잭션

수평 샤딩 (Horizontal Sharding) — 행 단위 분리:

  원본: users 테이블 10억 건
  
  수평 샤딩 (user_id % 4 기준):
    Shard 0: user_id % 4 = 0 (user_id: 0, 4, 8, ...)
    Shard 1: user_id % 4 = 1 (user_id: 1, 5, 9, ...)
    Shard 2: user_id % 4 = 2
    Shard 3: user_id % 4 = 3
  
  각 샤드: 2.5억 건 (1/4 크기)
  
  장점:
    데이터 크기 및 쓰기 처리량 수평 확장
    각 샤드가 동일한 스키마
  
  단점:
    샤드 키(user_id) 없는 쿼리 = 모든 샤드 조회 (Scatter-Gather)
    샤드 재분배 (Rebalancing) = 데이터 이동 필요 (운영 비용)
    분산 트랜잭션 불가

샤딩 키 선택 원칙:
  높은 카디널리티: user_id (중복 없음) → 균등 분산
  주요 쿼리 패턴 포함: user_id로 조회가 많으면 user_id로 샤딩
  Hot Spot 방지: 특정 값에 쏠리지 않는 키
```

### 5. 샤딩에서 JOIN 문제 해결

```java
// 샤딩 후 Cross-Shard JOIN 문제:
// orders (Shard by user_id) + products (Shard by product_id)
// → 같은 샤드에 없음 → SQL JOIN 불가

// 해결책 1: 애플리케이션 레벨 JOIN
public OrderDetailDto getOrderDetail(Long orderId, Long userId) {
    // 샤드 1에서 order 조회
    Order order = orderRepository.findById(orderId);
    // userId % 4 = shard 결정
    
    // 상품 DB에서 product 조회 (별도 요청)
    Product product = productRepository.findById(order.getProductId());
    // productId % 4 = 다른 shard
    
    return new OrderDetailDto(order, product);
    // → 2번의 DB 조회로 애플리케이션에서 조합
}

// 해결책 2: 중복 저장 (Denormalization)
// order_items에 product_name, product_price 복사 저장
// → Join 불필요하지만 상품 가격 변경 시 orders도 업데이트 필요

// 해결책 3: Broadcast 테이블 (읽기 전용 작은 테이블)
// products는 작고 자주 변경 안 됨
// → 모든 샤드에 products 복사 저장
// → 각 샤드에서 로컬 JOIN 가능
// 단점: 데이터 일관성 관리 복잡

// 해결책 4: CQRS + 읽기 전용 데이터 저장소
// 쓰기: 각 샤드 (정규화)
// 읽기: Elasticsearch, Redis 등에 비정규화된 데이터 저장
// → 복잡한 조합 쿼리는 읽기 저장소에서
```

---

## 💻 실전 실험

### 실험 1: Partition Pruning 효과 확인

```sql
-- 파티션 테이블 생성
CREATE TABLE log_events (
    id         BIGINT AUTO_INCREMENT,
    event_date DATE NOT NULL,
    event_type VARCHAR(50),
    payload    TEXT,
    PRIMARY KEY (id, event_date)
) ENGINE=InnoDB
PARTITION BY RANGE (TO_DAYS(event_date)) (
    PARTITION p2022 VALUES LESS THAN (TO_DAYS('2023-01-01')),
    PARTITION p2023 VALUES LESS THAN (TO_DAYS('2024-01-01')),
    PARTITION p2024 VALUES LESS THAN (TO_DAYS('2025-01-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- 파티션 정보 확인
SELECT PARTITION_NAME, TABLE_ROWS, DATA_LENGTH
FROM information_schema.PARTITIONS
WHERE TABLE_NAME = 'log_events'\G

-- Pruning 확인
EXPLAIN SELECT * FROM log_events
WHERE event_date BETWEEN '2024-01-01' AND '2024-03-31'\G
-- partitions: p2024 (Pruning!)

EXPLAIN SELECT * FROM log_events
WHERE event_type = 'CLICK'\G
-- partitions: p2022,p2023,p2024,p_future (Pruning 없음!)
```

### 실험 2: DROP PARTITION vs DELETE 성능 비교

```sql
-- 100만 건 데이터가 있는 파티션 삭제

-- 방법 1: DELETE (느림)
SET @start = NOW(6);
DELETE FROM log_events WHERE event_date < '2023-01-01';
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS delete_ms;
-- 예상: 수십 초 ~ 수분

-- 방법 2: DROP PARTITION (순간)
SET @start = NOW(6);
ALTER TABLE log_events DROP PARTITION p2022;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS drop_ms;
-- 예상: < 100ms (파일 삭제)
```

### 실험 3: 잘못된 파티션 키 선택으로 Full Scan

```sql
-- 파티션 키가 조건에 없는 경우:
CREATE TABLE orders_bad (
    id       BIGINT,
    user_id  BIGINT NOT NULL,  -- 주요 쿼리 기준
    created_at DATE
    -- 파티션 키: YEAR(created_at)
    -- 주요 쿼리: WHERE user_id = ?
) PARTITION BY RANGE (YEAR(created_at)) ( ... );

EXPLAIN SELECT * FROM orders_bad WHERE user_id = 12345;
-- partitions: 모든 파티션 → Pruning 없음!

-- 비교: 파티션 키를 쿼리 패턴에 맞게 변경하거나
-- user_id 인덱스 추가 (파티션별 Local Index)
ALTER TABLE orders_bad ADD INDEX idx_user_id (user_id);
EXPLAIN SELECT * FROM orders_bad WHERE user_id = 12345;
-- type: ref, key: idx_user_id → 각 파티션에서 인덱스 사용
-- 하지만 여전히 모든 파티션 조회 필요
```

---

## 📊 성능 비교

```
파티셔닝 전후 비교 (1억 건 orders, 월별 파티션):

파티션 없는 전체 스캔:
  SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31'
  → 1억 건 전체 스캔 (인덱스 있어도 ~30% = 3000만 건)
  → 실행 시간: 8초

파티셔닝 후 (월별 30개 파티션):
  → 3개 파티션만 스캔 (3/30 = 10%)
  → 실행 시간: 0.8초 (10배 향상)

오래된 파티션 삭제:
  DELETE (1000만 건): 15분
  DROP PARTITION:     0.1초
  → 9000배 차이

수평 샤딩 효과 (user_id 기준, 4 샤드):
  단일 DB 1억 건 → 각 샤드 2500만 건
  쓰기 TPS: 4배 향상 (각 샤드에 병렬)
  조회 TPS: 4배 향상 (user_id 기준)
  Cross-Shard 조회: 4배 느려짐 (모든 샤드 조회)

파티션 수별 오버헤드:
  10개 파티션: 무시할 수 있는 오버헤드
  100개 파티션: 경미한 메타데이터 오버헤드
  1000개 파티션: 눈에 띄는 오버헤드 (쿼리 파싱 시간)
  권장: 100개 이하
```

---

## ⚖️ 트레이드오프

```
파티셔닝:
  ✅ 파티션 키 기반 쿼리 성능 향상
  ✅ 시계열 데이터 효율적 정리 (DROP PARTITION)
  ✅ 관리자 관점: 파티션 통계, 백업 단위 관리
  ❌ 파티션 키 없는 쿼리 = 전체 파티션 스캔 (오히려 느릴 수 있음)
  ❌ 글로벌 인덱스 없음 → 비파티션 키 조회 비효율
  ❌ 외래키 사용 불가
  ❌ UNIQUE 제약에 파티션 키 포함 강제

수직 샤딩:
  ✅ 서비스별 독립 스케일링
  ✅ 마이크로서비스와 자연스럽게 결합
  ✅ 각 DB 크기 감소
  ❌ Cross-DB JOIN 불가
  ❌ Cross-DB 트랜잭션 복잡
  ❌ 데이터 일관성 관리 어려움

수평 샤딩:
  ✅ 무제한 수평 확장 (이론적)
  ✅ 쓰기 처리량 선형 증가
  ❌ 샤드 키 없는 쿼리 = Scatter-Gather (모든 샤드)
  ❌ 재샤딩(Rebalancing) = 데이터 이동 (운영 중 어려움)
  ❌ 분산 트랜잭션 복잡성
  ❌ 개발/운영 복잡도 대폭 증가

결론: 샤딩은 진정한 최후의 수단
  1. 인덱스 최적화 → 2. Read Replica → 3. 파티셔닝 →
  4. 수직 샤딩 → 5. 수평 샤딩
  각 단계에서 실제 한계를 측정하고 다음 단계 진행
```

---

## 📌 핵심 정리

```
파티셔닝 핵심:

Partition Pruning:
  WHERE에 파티션 키 포함 → 관련 파티션만 스캔
  포함 안 하면 → 전체 파티션 스캔 (오버헤드만 추가!)

파티션 종류:
  Range: 시계열 데이터 (YEAR, MONTH), 범위 쿼리
  List:  카테고리, 상태값, 지역 코드
  Hash:  균등 분산, 특정 범위 없음

핵심 장점:
  오래된 파티션 DROP: DELETE 수분 → DROP PARTITION 즉시
  최근 데이터 Buffer Pool 집중: 최신 파티션만 캐시

제약:
  글로벌 인덱스 없음 (파티션별 Local Index)
  PK에 파티션 키 포함 필수
  외래키 불가

샤딩 선택 기준:
  수직 샤딩: 기능별 DB 분리 (마이크로서비스)
  수평 샤딩: 단일 테이블이 수십억 건 이상일 때만
  샤딩 전: 반드시 파티셔닝, Read Replica 먼저 시도
```

---

## 🤔 생각해볼 문제

**Q1.** 주문(orders) 테이블을 `user_id % 4`로 수평 샤딩했다. `SELECT * FROM orders WHERE order_date = '2024-01-15'` 쿼리는 어떻게 처리되는가? 이를 효율적으로 만드는 방법은?

<details>
<summary>해설 보기</summary>

**현재 처리 방식 (비효율)**:
`order_date`는 샤드 키(user_id)가 아니므로, 어느 샤드에 데이터가 있는지 알 수 없습니다. → **Scatter-Gather**: 4개 샤드 모두에 쿼리를 보내고 결과를 병합합니다.
- 샤드 0, 1, 2, 3에 각각 `SELECT * FROM orders WHERE order_date = '2024-01-15'`
- 결과를 애플리케이션에서 합치기
- 응답 시간 ≈ 가장 느린 샤드의 응답 시간 (병렬이라도 4배 부하)

**효율적으로 만드는 방법**:

**방법 1: 날짜 기반 라우팅 테이블**
```sql
-- 별도 라우팅 DB:
routing_orders (order_date DATE, shard_ids VARCHAR(10))
-- order_date='2024-01-15' → shard_ids='0,2' (해당 날짜 데이터가 있는 샤드)
```
사전에 날짜-샤드 매핑을 관리하면 필요한 샤드만 조회 가능. 하지만 관리 복잡합니다.

**방법 2: CQRS 읽기 모델**
```
쓰기: 샤드별 저장 (user_id 기반)
읽기: Elasticsearch 또는 별도 분석 DB에 모든 데이터 동기화
      날짜 기반 쿼리는 읽기 모델에서 처리
```

**방법 3: 복합 샤드 키**
`user_id`만이 아닌 `(user_id, year_month)` 복합 키 사용 시 날짜로도 일부 라우팅 가능.

**실무 교훈**: 샤드 키를 선택할 때 예상 쿼리 패턴을 모두 고려해야 합니다. 비샤드 키 쿼리가 많으면 Scatter-Gather 비용이 커집니다.

</details>

---

**Q2.** Range 파티셔닝에서 `p_future VALUES LESS THAN MAXVALUE`를 항상 마지막에 두는 이유는 무엇인가? 이것 없이 2024년까지만 파티션을 정의하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

`p_future MAXVALUE`는 **범위 밖의 데이터를 위한 기본 파티션**입니다.

**없을 때 발생하는 문제**:
```sql
-- 2024년까지만 정의:
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
    -- p_future 없음!
);

-- 2025년 1월 1일 주문 INSERT:
INSERT INTO orders (order_date, ...) VALUES ('2025-01-15', ...);
-- ERROR 1526 (HY000): Table has no partition for value 2025
-- → 삽입 실패! 서비스 장애!
```

**`p_future MAXVALUE` 있을 때**:
```sql
PARTITION p_future VALUES LESS THAN MAXVALUE
-- 2025년 데이터 → p_future 파티션에 저장
-- INSERT 실패 없음

-- 나중에 2025년 파티션 추가:
ALTER TABLE orders REORGANIZE PARTITION p_future INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
-- p_future의 2025년 데이터 → p2025로 이동
-- p_future는 다시 비어있음
```

**실무 주의사항**: 연말에 다음 해 파티션을 미리 추가하는 자동화 작업(Cron Job)을 설정하는 것이 일반적입니다. `p_future`가 있어도 MAXVALUE 파티션에 데이터가 쌓이면 나중에 `REORGANIZE`가 오래 걸릴 수 있습니다.

</details>

---

**Q3.** 파티셔닝을 적용한 테이블에서 `SELECT * FROM orders WHERE user_id = 123 AND order_date >= '2024-01-01'`을 실행했다. 이 쿼리의 실행 계획이 어떻게 되는지 단계별로 설명하라. (파티션 키: order_date, user_id 인덱스: 있음)

<details>
<summary>해설 보기</summary>

**실행 계획 단계**:

1. **Partition Pruning**: `order_date >= '2024-01-01'` 조건으로 p2024 파티션만 선택. 이전 파티션들은 스캔 안 함.

2. **인덱스 사용**: 선택된 파티션(p2024) 내에서 `user_id = 123` 조건으로 `idx_user_id` 인덱스 사용.

3. **Clustered Index Double Lookup**: `idx_user_id`에서 `user_id=123`인 PK 목록 조회 → PK로 실제 Row 조회.

**EXPLAIN 결과**:
```
partitions: p2024
type: ref
key: idx_user_id
rows: (user_id=123인 2024년 주문 건수 예측)
Extra: Using index condition
```

**비용**:
- Partition Pruning으로 과거 파티션 I/O 절약
- p2024 내에서는 인덱스 사용으로 효율적
- 결론: 두 조건 모두 활용되어 최적의 실행

**만약 order_date 조건이 없었다면**:
- Pruning 없음 → 모든 파티션에서 idx_user_id 사용
- 각 파티션 인덱스에서 user_id=123 조회 후 합치기
- → 파티션이 많을수록 오버헤드 증가

이 쿼리 패턴이 많다면 `(order_date, user_id)` 복합 인덱스를 고려하거나, 파티션 키와 인덱스를 쿼리 패턴에 맞게 설계하는 것이 중요합니다.

</details>

---

<div align="center">

**[⬅️ 페이징 최적화](./03-pagination-optimization.md)** | **[홈으로 🏠](../README.md)** | **[다음: HikariCP ➡️](./05-connection-pool-hikaricp.md)**

</div>
