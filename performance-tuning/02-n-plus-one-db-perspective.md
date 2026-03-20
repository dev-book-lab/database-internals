# N+1 문제 — DB 관점에서의 근본 원인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- N+1 문제가 Slow Query Log에 어떤 패턴으로 나타나는가?
- 각 Lazy Loading이 독립 Connection을 사용할 때 HikariCP Pool이 소진되는 시나리오는?
- `p6spy` / `datasource-proxy`로 실제 실행 쿼리 수를 계측하는 방법은?
- Fetch Join이 생성하는 SQL과 실행계획은 어떻게 다른가?
- `@EntityGraph`와 Fetch Join의 내부 동작 차이는?
- MultipleBagFetchException이 발생하는 이유와 해결책은?

---

## 🔍 왜 이 개념이 중요한가

### N+1은 코드에서 보이지 않아서 더 위험하다

```
개발 환경:
  테스트 데이터 10건 → 쿼리 11번 → 10ms → "정상"
  
프로덕션 환경:
  주문 목록 100건 → 쿼리 101번 → 사용자 조회 × 100
  초당 10명이 주문 목록 조회:
    → 초당 1010번의 쿼리 실행
    → HikariCP maxPoolSize=10이면?
    → 각 HTTP 요청이 101개 쿼리 × 각 쿼리 Connection 점유
    → 10 Connection × 10 요청 = 100 Connection 필요 >> 10
    → HikariPool: Connection is not available
    → 요청 큐 폭증 → 타임아웃 → 503 Service Unavailable

왜 코드에서 안 보이는가:
  @ManyToOne(fetch = FetchType.LAZY)
  User user;  ← "이게 N번 쿼리를 날린다"는 것이 명백하지 않음

  orders.forEach(order -> {
    // user 접근 시 Lazy Loading → SELECT 1건 실행
    System.out.println(order.getUser().getName());
  });
  // 코드만 보면 전혀 문제 없어 보임
```

---

## 😱 잘못된 이해

### Before: "Lazy Loading이 기본값이고 필요할 때만 로드하니까 효율적이다"

```
잘못된 믿음:
  "EAGER Loading은 항상 JOIN해서 무거우니까
   LAZY Loading이 더 효율적이다"

실제:
  단건 조회 시: LAZY가 맞음
    Order 1개만 필요 → User는 사용 안 하면 SELECT 안 함
  
  목록 조회 시: LAZY = N+1 위험
    Orders 100개 → User가 필요 → 100번 추가 SELECT
    vs
    JOIN 1번으로 한 번에 가져옴 (훨씬 효율적)

잘못된 진단:
  개발자: "DB가 느린 것 같아요"
  DBA:   "CPU, I/O 정상인데요"
  사실:  초당 1000번의 작은 쿼리가 Connection Pool을 소진
         작은 쿼리 수천 번 > 큰 쿼리 1번 (Connection 오버헤드 때문)

잘못된 해결:
  "EAGER Loading으로 바꾸자"
  → 단건 조회에서 불필요한 JOIN까지 항상 실행
  → N+1은 해결되지만 불필요한 데이터 로드 문제 발생
  → 올바른 해결: 필요한 경우에만 Fetch Join / @EntityGraph 사용
```

---

## ✨ 올바른 이해

### After: N+1 = "1번 조회 후 N번의 추가 쿼리"를 DB 레벨에서 파악

```
N+1이 DB에서 보이는 형태:

Slow Query Log에서:
  # Query_time: 0.003  ... Rows_examined: 1
  SELECT * FROM users WHERE id = 1;
  # Query_time: 0.003  ... Rows_examined: 1
  SELECT * FROM users WHERE id = 2;
  # Query_time: 0.003  ... Rows_examined: 1
  SELECT * FROM users WHERE id = 3;
  ... (100번 반복)
  
  → 개별 쿼리는 3ms로 빠름
  → 동일 패턴이 N번 반복 = N+1 시그니처

performance_schema에서:
  DIGEST_TEXT: SELECT * FROM users WHERE id = ?
  COUNT_STAR: 10,000  ← 같은 패턴이 1만 번!
  AVG_TIMER_WAIT: 3ms
  SUM_TIMER_WAIT: 30,000ms = 30초

해결 전략:
  1. Fetch Join (JPQL): JOIN FETCH → 1번의 JOIN 쿼리로 해결
  2. @EntityGraph: Fetch Join의 어노테이션 방식
  3. Batch Size: IN 절로 묶어서 SELECT (N→1 또는 N→N/batchSize)
  4. 즉시 로딩: 항상 필요한 경우에만 @ManyToOne(fetch=EAGER)

선택 기준:
  1:1 또는 N:1 관계 → Fetch Join 또는 @EntityGraph
  1:N 관계 (컬렉션) → Batch Size 또는 별도 쿼리
  여러 컬렉션 → Batch Size (MultipleBagFetchException 방지)
```

---

## 🔬 내부 동작 원리

### 1. N+1 발생 메커니즘 상세

```java
// N+1이 발생하는 전형적 패턴:

@Entity
public class Order {
    @Id Long id;
    BigDecimal amount;
    
    @ManyToOne(fetch = FetchType.LAZY)  // 기본값: Lazy
    User user;
    
    @OneToMany(fetch = FetchType.LAZY)
    List<OrderItem> items;
}

// 문제 코드:
@Transactional(readOnly = true)
public List<OrderDto> getOrderList() {
    List<Order> orders = orderRepository.findAll();
    // SQL: SELECT * FROM orders  → 1번 (100건 반환)
    
    return orders.stream()
        .map(order -> new OrderDto(
            order.getId(),
            order.getUser().getName(),  // Lazy → 1번 per order
            // SQL: SELECT * FROM users WHERE id = ?  ← 여기서 추가 쿼리!
            order.getAmount()
        ))
        .toList();
    // 총 SQL: 1 + N = 1 + 100 = 101번!
}

// DB 레벨에서 일어나는 일:
// T=0ms: SELECT * FROM orders (100건)
// T=1ms: SELECT * FROM users WHERE id = 1 (Connection A)
// T=2ms: SELECT * FROM users WHERE id = 2 (Connection A 재사용 또는 B)
// T=3ms: SELECT * FROM users WHERE id = 3
// ...
// T=100ms: SELECT * FROM users WHERE id = 100
// 총 시간: ~100ms (각 3ms × 100번은 아님, Connection 재사용)

// Connection Pool과의 관계:
// 각 쿼리는 Connection을 잠깐 점유 → 해제
// 하지만 동시 요청이 많으면:
// 요청 10개 × 101 쿼리 = 1010 DB 접근 (순차)
// 최대 10 Connection (HikariCP) → 요청 큐 발생
```

### 2. Slow Query Log에서 N+1 감지

```
N+1 시그니처:
  1. 동일한 쿼리 패턴이 짧은 시간 내 반복:
     # Time: 14:23:45.001  Query_time: 0.003  Rows_examined: 1
     SELECT * FROM users WHERE id = 1;
     # Time: 14:23:45.004  Query_time: 0.003  Rows_examined: 1
     SELECT * FROM users WHERE id = 2;
     ... (동일 패턴, 연속된 타임스탬프)

  2. mysqldumpslow에서:
     Count: 10000  Time=0.003s (30s)
     SELECT * FROM users WHERE id = N
     → 같은 패턴 1만 번 → N+1!

  3. performance_schema에서:
     COUNT_STAR = 10,000 (매우 높음)
     AVG_TIMER_WAIT = 3ms (매우 낮음)
     SUM_TIMER_WAIT = 30,000ms (높음)
     → 빠른 쿼리가 엄청나게 반복 = N+1 패턴

  4. DB Connection 로그에서:
     같은 Connection에서 반복 쿼리 또는
     Connection 수가 주기적으로 maxPoolSize에 도달
```

### 3. p6spy / datasource-proxy로 쿼리 계측

```groovy
// 의존성 추가 (build.gradle):
implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0'
```

```properties
# application.properties:
# p6spy 설정
decorator.datasource.p6spy.enable-logging=true
decorator.datasource.p6spy.logging=slf4j

# spy.properties (resources/ 아래):
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
executionThreshold=0
```

```java
// 테스트에서 쿼리 수 검증:
@DataJpaTest
class OrderRepositoryTest {
    
    @Test
    void n_plus_one_detection() {
        // p6spy 카운터 초기화
        StatementInspectorHolder.reset();
        
        // 목록 조회
        List<Order> orders = orderRepository.findAll();
        orders.forEach(o -> o.getUser().getName()); // Lazy Loading 트리거
        
        // 실행된 쿼리 수 확인
        long queryCount = StatementInspectorHolder.getCount();
        System.out.println("쿼리 수: " + queryCount);
        // 1 (orders) + N (users) = N+1 확인
        
        assertThat(queryCount).isLessThanOrEqualTo(2); // 목표: 2건 이하
    }
}

// datasource-proxy 방식 (더 세밀한 제어):
@Bean
public DataSource dataSource() {
    DataSource realDataSource = hikariDataSource();
    return ProxyDataSourceBuilder.create(realDataSource)
        .name("my-ds")
        .logQueryBySlf4j(SLF4JLogLevel.INFO)
        .countQuery()   // 쿼리 수 집계
        .build();
}
```

### 4. Fetch Join vs @EntityGraph 비교

```java
// 방법 1: JPQL Fetch Join
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
    List<Order> findWithUser(@Param("status") String status);
    // 생성 SQL:
    // SELECT DISTINCT o.*, u.*
    // FROM orders o
    // INNER JOIN users u ON o.user_id = u.id
    // WHERE o.status = ?
    // → 1번의 JOIN으로 Order + User 모두 로드
    // → 100 orders + 100 users = 1번의 쿼리
}

// 방법 2: @EntityGraph
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @EntityGraph(attributePaths = {"user"})
    List<Order> findByStatus(String status);
    // 생성 SQL:
    // SELECT o.*, u.*
    // FROM orders o
    // LEFT OUTER JOIN users u ON o.user_id = u.id
    // WHERE o.status = ?
    // → Fetch Join과 유사하지만 LEFT OUTER JOIN 사용
}

// Fetch Join vs @EntityGraph 주요 차이:
// Fetch Join:
//   INNER JOIN (명시적) → 관계가 없는 Row 제외
//   JPQL 작성 필요
//   DISTINCT 필요 (1:N Fetch Join 시 중복 Row)
//   페이징 쿼리에서 경고: HibernateJpaDialect.warnAboutHibernateDialect
//     "HHH90003004: firstResult/maxResults specified with collection fetch"
//     → COUNT 쿼리 없이 전체 JOIN 후 메모리에서 페이징 (위험!)

// @EntityGraph:
//   LEFT OUTER JOIN → 관계 없어도 포함
//   어노테이션으로 간단
//   역시 1:N에서 HibernateJpaDialect 경고 동일
```

### 5. Batch Size로 N+1 완화 (컬렉션)

```java
// 1:N 컬렉션에서의 해결책:

// 설정:
// application.properties:
// spring.jpa.properties.hibernate.default_batch_fetch_size=100
// 또는 개별 필드:
@OneToMany(fetch = FetchType.LAZY)
@BatchSize(size = 100)
List<OrderItem> items;

// 동작 원리:
// 기존 (N+1):
//   SELECT * FROM orders LIMIT 100          → orders 100건
//   SELECT * FROM order_items WHERE order_id = 1   → 1건
//   SELECT * FROM order_items WHERE order_id = 2   → 1건
//   ... (100번)

// BatchSize=100 적용:
//   SELECT * FROM orders LIMIT 100          → orders 100건
//   SELECT * FROM order_items WHERE order_id IN (1, 2, 3, ..., 100)  → 1번!
//   → N+1 → 2번으로 감소!

// BatchSize 선택 기준:
// 한 번에 가져올 부모 Row 수 = BatchSize
// 목록이 100건이면 BatchSize = 100
// 목록이 1000건이면 BatchSize = 1000 (또는 여러 번 나눔)

// MultipleBagFetchException 해결:
// 오류 발생 케이스:
@Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.coupons")
// → 두 컬렉션(items, coupons)을 동시에 Fetch Join → 예외!

// 해결 1: Batch Size + 별도 조회
// Order Fetch Join items만 → coupons는 Batch Size로
@Query("SELECT o FROM Order o JOIN FETCH o.items")
// @BatchSize(size=100) on coupons

// 해결 2: List → Set 변환
@OneToMany
Set<OrderItem> items;  // Set은 MultipleBag 예외 없음
// 단, 순서 보장 없음
```

---

## 💻 실전 실험

### 실험 1: N+1 발생 확인 (SQL 로그로)

```sql
-- Slow Query Log로 N+1 패턴 확인
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0;  -- 모든 쿼리 기록

-- JPA에서 N+1 발생 코드 실행 후 로그 확인:
-- $ grep "SELECT" /var/log/mysql/slow.log | grep "users" | wc -l
-- → 100이면 100번의 user 조회 = N+1

-- performance_schema에서 N+1 탐지:
SELECT
    SUBSTR(DIGEST_TEXT, 1, 80) AS query,
    COUNT_STAR AS exec_count,
    ROUND(AVG_TIMER_WAIT/1e9, 3) AS avg_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = DATABASE()
  AND COUNT_STAR > 50  -- 50번 이상 실행된 쿼리
  AND DIGEST_TEXT LIKE '%users%'
ORDER BY COUNT_STAR DESC;
-- COUNT_STAR가 목록 건수와 비슷하면 N+1
```

### 실험 2: Fetch Join 전후 쿼리 수 비교

```java
@Test
void compare_fetch_strategies() {
    // N+1 발생
    QueryCountHolder.reset();
    List<Order> ordersLazy = orderRepository.findAll();
    ordersLazy.forEach(o -> o.getUser().getName()); // Lazy 트리거
    long lazyCount = QueryCountHolder.getCount();
    System.out.println("Lazy(N+1) 쿼리 수: " + lazyCount); // 1+N
    
    // Fetch Join 사용
    QueryCountHolder.reset();
    List<Order> ordersFetch = orderRepository.findAllWithUser();
    ordersFetch.forEach(o -> o.getUser().getName()); // 이미 로드됨
    long fetchCount = QueryCountHolder.getCount();
    System.out.println("Fetch Join 쿼리 수: " + fetchCount); // 1
    
    assertThat(fetchCount).isEqualTo(1);
}
```

### 실험 3: Batch Size 효과 측정

```sql
-- Batch Size 없음 (N+1):
-- SELECT * FROM orders LIMIT 100; → 1번
-- SELECT * FROM order_items WHERE order_id = 1; → N번
-- ... (100번)

-- Batch Size = 100 설정 후:
-- SELECT * FROM orders LIMIT 100; → 1번
-- SELECT * FROM order_items WHERE order_id IN (1,2,...,100); → 1번
-- 총 2번으로 감소

-- IN 절 쿼리 확인:
EXPLAIN SELECT * FROM order_items
WHERE order_id IN (1, 2, 3, 4, 5, 10, 20, 30, 50, 100);
-- type: range (order_id 인덱스 있으면)
-- key: idx_order_id
-- rows: 예상 건수

-- Batch Size 크기에 따른 쿼리 수:
-- 목록 100건, Batch Size = 10:
--   100 / 10 = 10번의 IN 쿼리
-- 목록 100건, Batch Size = 100:
--   100 / 100 = 1번의 IN 쿼리 (최적)
```

### 실험 4: HikariCP Connection 소진 시뮬레이션

```java
// Connection Pool이 소진되는 상황 시뮬레이션
@SpringBootTest
class ConnectionPoolExhaustionTest {
    
    @Test
    void simulate_pool_exhaustion() throws Exception {
        int maxPoolSize = 10;
        int concurrentRequests = 20;
        CountDownLatch latch = new CountDownLatch(concurrentRequests);
        
        // 20개의 동시 요청 (각각 N+1로 101개 쿼리)
        for (int i = 0; i < concurrentRequests; i++) {
            executor.submit(() -> {
                try {
                    orderService.getOrderList();  // N+1 발생
                } catch (Exception e) {
                    System.out.println("에러: " + e.getMessage());
                    // "HikariPool-1 - Connection is not available,
                    //  request timed out after 30000ms"
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await();
        // 일부 요청이 Connection Timeout으로 실패
        
        // Fetch Join 버전으로 동일 테스트 → 모든 요청 성공
    }
}
```

---

## 📊 성능 비교

```
동일 쿼리 (주문 100건 + 사용자 정보):

방법 1: Lazy Loading (N+1):
  쿼리 수: 101번 (1 + 100)
  DB 실행 시간: 1×5ms + 100×3ms = 305ms
  Connection 점유: 101회
  데이터 전송: 100건 × 소량 × 101번 왕복

방법 2: Fetch Join:
  쿼리 수: 1번
  DB 실행 시간: 1×20ms (JOIN 포함)
  Connection 점유: 1회
  데이터 전송: 100건 × 사용자 포함 × 1번 왕복
  → 15배 빠름 (Connection 오버헤드 제거)

방법 3: Batch Size (IN 절):
  쿼리 수: 2번 (orders 1번 + users IN 1번)
  DB 실행 시간: 5ms + 8ms = 13ms
  Connection 점유: 2회
  → Fetch Join과 유사, 컬렉션에서 MultipleBag 없음

HikariCP 관점:
  Pool Size = 10, 동시 요청 10개:
    N+1 방식: 10 × 101 = 1010 connection 사용 (순차지만 큐 발생)
    Fetch Join: 10 × 1 = 10 connection 사용 (여유 있음)
  
  N+1 → 요청 큐 발생 → P99 응답 시간 급증
  Fetch Join → Connection Pool 여유 → 안정적
```

---

## ⚖️ 트레이드오프

```
Lazy Loading:
  ✅ 필요한 경우에만 데이터 로드 (단건 조회 최적)
  ✅ 설정 간단 (JPA 기본값)
  ❌ 목록 조회에서 N+1 위험
  ❌ 트랜잭션 밖에서 접근 시 LazyInitializationException
  적합: 단건 조회, 해당 관계를 사용하지 않는 경우

Fetch Join:
  ✅ 1번의 쿼리로 연관 데이터 모두 로드
  ✅ 명확한 성능 (N+1 없음)
  ❌ 1:N 관계에서 페이징 불가 (메모리 페이징 위험)
  ❌ 여러 컬렉션 동시 Fetch Join → MultipleBagFetchException
  ❌ JPQL 작성 필요
  적합: N:1 또는 1:1 관계 목록 조회

@EntityGraph:
  ✅ 어노테이션으로 간단
  ✅ Spring Data JPA 메서드와 통합
  ❌ Fetch Join과 동일한 1:N 페이징 제약
  적합: 간단한 N:1 관계 즉시 로딩

Batch Size:
  ✅ 1:N 컬렉션에서 안전 (페이징 가능)
  ✅ MultipleBagFetchException 없음
  ✅ 설정만으로 적용 (코드 변경 최소)
  ❌ 2번 이상의 쿼리 (Fetch Join보다 쿼리 수 많음)
  ❌ 대량(>1000건) 목록에서 IN 절이 길어짐
  적합: 1:N 컬렉션, 여러 컬렉션 동시 로딩

Default_batch_fetch_size 전역 설정:
  spring.jpa.properties.hibernate.default_batch_fetch_size=100
  → 모든 Lazy Loading에 자동 적용
  → 별도 설정 없이 대부분의 N+1 완화
  → 우선 이것부터 적용 후 필요시 Fetch Join 추가
```

---

## 📌 핵심 정리

```
N+1 핵심:

발생 원인:
  Lazy Loading 관계를 목록에서 개별 접근
  1번 목록 조회 + N번 연관 데이터 조회

DB에서의 시그니처:
  Slow Log: 동일 쿼리 패턴이 연속으로 N번 반복
  perf_schema: COUNT_STAR 매우 높음, AVG 낮음
  HikariCP: Connection Pool 소진, Connection timeout

탐지 도구:
  p6spy: 실행 쿼리 수 로깅 (개발환경)
  datasource-proxy: 쿼리 카운터 (테스트)
  performance_schema: 프로덕션 모니터링

해결책:
  N:1, 1:1 관계: Fetch Join 또는 @EntityGraph
  1:N 컬렉션: Batch Size (default_batch_fetch_size=100)
  여러 컬렉션: 각각 Batch Size (MultipleBagFetchException 방지)
  
우선 적용:
  spring.jpa.properties.hibernate.default_batch_fetch_size=100
  → 코드 수정 없이 대부분의 N+1 완화
  → 이후 Fetch Join으로 세밀하게 최적화
```

---

## 🤔 생각해볼 문제

**Q1.** `default_batch_fetch_size=100`을 설정했다. 주문 목록 API가 100건을 반환하고, 각 주문에는 평균 5개의 `OrderItem`이 있다. 설정 전과 후의 쿼리 수를 계산하고 총 데이터 전송량을 비교하라.

<details>
<summary>해설 보기</summary>

**설정 전 (N+1)**:
- `SELECT * FROM orders LIMIT 100` → 1번
- `SELECT * FROM order_items WHERE order_id = ?` → 100번 (각 주문마다)
- 총 쿼리: **101번**
- 데이터: 100건 orders + 100회 왕복으로 각 5건 = 총 100×5=500건 items
- Connection 점유: 101회

**설정 후 (Batch Size=100)**:
- `SELECT * FROM orders LIMIT 100` → 1번
- `SELECT * FROM order_items WHERE order_id IN (1,2,...,100)` → **1번**
- 총 쿼리: **2번** (99.0% 감소)
- 데이터: 동일 500건 items, 하지만 1번의 왕복
- Connection 점유: 2회

**차이**:
- 쿼리 수: 101 → 2 (50배 감소)
- Network round-trip: 101 → 2
- DB Connection 재사용: 101회 → 2회

단, IN 절의 주문 ID 목록이 길어지면 (1,2,...,100) 쿼리 자체가 약간 무거워집니다. 하지만 이는 101번의 왕복 비용에 비하면 미미합니다.

100건이 아닌 1000건 목록이라면: Batch Size=100이면 `IN (1..100)`, `IN (101..200)`, ..., `IN (901..1000)` = 10번의 IN 쿼리. 여전히 1001번보다 훨씬 낫습니다.

</details>

---

**Q2.** 다음 코드에서 Fetch Join을 사용했음에도 불구하고 HibernateJpaDialect 경고가 발생하고 페이징이 올바르지 않은 이유는?

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAllWithItems(Pageable pageable);
```

<details>
<summary>해설 보기</summary>

**1:N Fetch Join + Pageable의 충돌** 문제입니다.

발생 경고:
```
HHH90003004: firstResult/maxResults specified with collection fetch;
applying in memory!
```

**이유**:
- `JOIN FETCH o.items`는 `orders`와 `order_items`를 JOIN합니다
- 주문 1건에 items 5개 → JOIN 결과는 5행 (데이터 뻥튀기)
- `LIMIT 10`을 SQL에 적용하면 → ORDER 10건이 아닌 "JOIN 결과 행 10개" = Order 2건(각 5item)
- 이를 알기 때문에 Hibernate는 LIMIT 없이 전체를 가져와 **메모리에서 페이징**
- 100만 건이면 100만 건을 모두 메모리에 올린 후 10건만 반환 → OOM 위험!

**올바른 해결책**:
```java
// 방법 1: orders만 페이징 후, items는 Batch Size로
@Query("SELECT o FROM Order o")
Page<Order> findAll(Pageable pageable);
// + @BatchSize(size=100) on items → 2번의 쿼리로 해결

// 방법 2: 카운트 쿼리 분리 + 연관 별도 조회
@Query(value = "SELECT o FROM Order o",
       countQuery = "SELECT COUNT(o) FROM Order o")
Page<Order> findAll(Pageable pageable);
// 페이징 후 Batch Size로 items 로드

// 방법 3: DTO 프로젝션 (JOIN 결과를 DTO로)
@Query("SELECT new OrderDto(o.id, o.amount, COUNT(i)) " +
       "FROM Order o LEFT JOIN o.items i GROUP BY o.id, o.amount")
Page<OrderDto> findAllAsDto(Pageable pageable);
// items를 집계해서 DTO로 → 1:N 문제 없음
```

핵심: **1:N Fetch Join과 Pageable은 함께 쓰면 안 됩니다.** Batch Size가 올바른 대안입니다.

</details>

---

**Q3.** N+1 문제를 해결하기 위해 `@EntityGraph`를 전역으로 설정했더니 오히려 성능이 저하됐다. 어떤 상황에서 이런 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**단건 조회에서 불필요한 JOIN이 항상 실행되는 경우**입니다.

```java
// @EntityGraph를 모든 조회에 적용:
@EntityGraph(attributePaths = {"user", "items", "coupons"})
@Override
List<Order> findAll();

// 단건 조회도 동일하게:
@EntityGraph(attributePaths = {"user", "items", "coupons"})
Optional<Order> findById(Long id);
// → 단건 조회에서도 users, order_items, coupons 모두 JOIN!
// → 실제로는 order.getUser()만 필요한 경우에도 items, coupons도 로드
```

**발생하는 문제**:
1. 불필요한 데이터 로드: items나 coupons를 사용하지 않아도 항상 JOIN
2. JOIN 결과 뻥튀기: items 1만 건 × coupons 5천 건 → 엄청난 JOIN 결과
3. 메모리 사용 증가: 사용 안 하는 데이터도 Hibernate 1차 캐시에 적재
4. 특히 관리자 페이지에서 단건 상세조회 + 복잡한 연관관계 → JOIN이 더 비쌈

**올바른 적용 방식**:
```java
// 목록 조회 (N+1 위험 있는 곳)에만 적용:
@EntityGraph(attributePaths = {"user"})
List<Order> findByStatus(String status);

// 단건 조회는 기본 Lazy:
Optional<Order> findById(Long id);  // @EntityGraph 없음

// 필요한 경우 동적으로:
@EntityGraph(attributePaths = {"user", "items"})
Optional<Order> findWithDetailsById(Long id);  // 상세 조회 전용
```

결론: `@EntityGraph`는 "이 쿼리에서 이 관계가 항상 필요하다"는 것이 확실할 때만 적용해야 합니다. 전역 적용은 단건 조회에서의 불필요한 JOIN을 초래합니다.

</details>

---

<div align="center">

**[⬅️ Slow Query 분석](./01-slow-query-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: 페이징 최적화 ➡️](./03-pagination-optimization.md)**

</div>
