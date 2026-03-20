# Connection Pool 튜닝 — HikariCP와 DB 연결 관리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HikariCP가 Connection을 미리 생성해두는 방식과 Pool 크기 결정 알고리즘은?
- `maximumPoolSize` 계산 공식 `Ncpu × 2 + Neff_disk`의 의미는?
- `HikariPool-1 - Connection is not available, request timed out after Nms` 에러가 발생하는 정확한 조건은?
- `minimumIdle`과 `maximumPoolSize`를 같게 설정하는 것이 권장되는 이유는?
- Connection이 Dead 상태인지 감지하는 HikariCP의 메커니즘은?
- Micrometer + Prometheus로 HikariCP Pool 상태를 모니터링하는 방법은?

---

## 🔍 왜 이 개념이 중요한가

### "DB는 멀쩡한데 애플리케이션에서 DB 에러가 난다"

```
가장 흔한 Connection Pool 관련 장애:

에러 메시지:
  HikariPool-1 - Connection is not available, request timed out after 30000ms
  com.zaxxer.hikari.pool.HikariPool$PoolInitializationException
  
원인이 아닌 것:
  DB 서버 다운 → DB 에러는 즉시 발생, 30초 기다리지 않음
  쿼리가 느린 것 → 느린 쿼리는 별도 증상
  
실제 원인:
  Pool의 모든 Connection이 사용 중 → 새 요청이 대기
  30초(connectionTimeout) 초과 → 위 에러 발생

왜 모든 Connection이 사용 중인가?
  시나리오 A: 갑작스런 트래픽 증가
    평상시 TPS=100, Pool=10, 쿼리 평균 10ms
    → 초당 Connection 사용량 = 100 × 0.01s = 1 Connection (여유)
    
    트래픽 10배 증가: TPS=1000
    → 초당 Connection 사용량 = 1000 × 0.01s = 10 Connection = Pool 포화!
    → 대기 큐 발생 → connectionTimeout 초과
    
  시나리오 B: 느린 쿼리로 인한 Connection 점유
    느린 쿼리 (10초): 10개 요청 × 10초 = 100 Connection-seconds
    Pool=10이면 10초 동안 10개 모두 사용 중 → 새 요청 대기

이해하면:
  Pool 크기 계산 공식으로 적절한 값 설정
  Slow Query 개선으로 Connection 점유 시간 단축
  모니터링으로 Pool 포화 상태 사전 감지
```

---

## 😱 잘못된 이해

### Before: "Connection Pool 크기가 클수록 좋다"

```
잘못된 믿음:
  "maximumPoolSize를 100으로 늘리면 더 많은 요청을 처리할 수 있다"
  
실제:
  DB 서버는 Connection당 메모리와 스레드 사용
  MySQL: 기본적으로 Connection당 ~1MB 메모리
  Connection 100개 = ~100MB (부담이 아님)
  하지만:
    Connection이 많다 ≠ 처리량이 높다
    DB CPU/디스크가 병목이면 Connection이 더 많아도 동일한 TPS
    오히려 Context Switching 오버헤드 증가
    Connection 100개가 모두 동시에 쿼리하면
    → DB가 100개의 스레드를 동시 관리 → 오히려 느려짐

HikariCP 공식 문서 인용:
  "Want high performance? Keep a small pool."
  "우리 분석 결과: 9600 Connection > 2048 Connection이 더 빠름"
  (Pool 크기가 작을 때 더 빠른 이유: 경합 감소, 캐시 효율)

잘못된 또 다른 믿음:
  "minimumIdle = 0이면 필요할 때만 Connection 생성 → 효율적"
  
실제:
  트래픽 급증 시 Connection 생성 시간(~50ms) 추가
  → 초기 요청들이 50ms씩 더 기다림 → 응답 시간 급증
  → minimumIdle = maximumPoolSize (상시 유지)가 더 안정적
```

---

## ✨ 올바른 이해

### After: Pool 크기 = DB가 효율적으로 처리할 수 있는 동시 Connection 수

```
Connection Pool의 역할:
  DB Connection 생성 비용: ~50ms (TCP 연결 + 인증 + 세션 초기화)
  Pool 없이: 모든 요청마다 50ms 추가 오버헤드
  Pool 있이: 미리 생성 → 요청 시 즉시 반환 (~0ms)
  
적절한 Pool 크기 공식 (HikariCP 권장):
  Pool Size = Ncpu × 2 + Neff_disk
  
  Ncpu: DB 서버의 CPU 코어 수
  Neff_disk: 유효 스피들 디스크 수 (SSD면 보통 1로 처리)
  
  AWS RDS db.m5.large (2 vCPU):
    Pool Size = 2 × 2 + 1 = 5
  
  AWS RDS db.r6g.xlarge (4 vCPU):
    Pool Size = 4 × 2 + 1 = 9
  
  이것이 놀랍게 작아 보이는 이유:
    DB CPU가 병목 → Connection이 더 있어도 처리량 같음
    오히려 Context Switching으로 느려짐
    
  애플리케이션 서버가 여러 개인 경우:
    총 Connection = Pool Size × 서버 수
    서버 10대 × Pool 5 = 50 Connection
    DB가 허용 가능한지 확인 (max_connections 설정)
    
  실무 조정:
    공식은 시작점, 실제 모니터링으로 조정
    Pool 포화 시 Pool 크기 증가보다 Slow Query 개선이 우선
```

---

## 🔬 내부 동작 원리

### 1. HikariCP Connection 생명주기

```
HikariCP 초기화 (애플리케이션 시작 시):

1. minimumIdle개의 Connection 즉시 생성
   (minimumIdle = maximumPoolSize 권장 → 모두 즉시 생성)
   
2. 각 Connection: TCP 연결 → MySQL 인증 → 세션 설정
   소요 시간: ~50ms per connection
   
3. ConnectionTestQuery 실행 (idle connection 검증)
   또는 isValid() 확인
   
4. Pool에 등록 → Ready 상태

Connection 대여 (Request 처리):
  1. 요청: dataSource.getConnection()
  2. Pool에 여유 Connection 있으면: 즉시 반환 (~0ms)
  3. Pool이 가득 차면: connectionTimeout 동안 대기
  4. connectionTimeout 초과: HikariTimeoutException

Connection 반납 (Request 완료):
  1. connection.close() 호출 (실제 연결 닫힘 아님!)
  2. HikariCP가 Connection 상태 초기화 (auto-commit 등)
  3. Pool에 반납 → 다음 요청에 재사용

Dead Connection 감지:
  maxLifetime (기본 30분): 이 시간 초과한 Connection 교체
  keepaliveTime (기본 0): 주기적으로 idle Connection에 ping
  connectionTestQuery: idle Connection 검증 쿼리
  → MySQL의 wait_timeout(기본 8시간)과 충돌 방지
    DB가 Connection을 끊어도 Pool은 모름 → 사용 시 에러
    → maxLifetime < MySQL wait_timeout 설정 필요
```

### 2. 주요 설정과 의미

```yaml
# application.yml:
spring:
  datasource:
    hikari:
      # 연결 설정
      maximum-pool-size: 10        # 최대 Connection 수
      minimum-idle: 10             # 유지할 최소 idle Connection (=maximum 권장)
      connection-timeout: 30000    # Connection 대기 최대 시간 (30초, 기본값)
                                   # → 이 시간 초과 시 "Connection is not available" 에러
      
      # Connection 유효성
      max-lifetime: 1800000        # Connection 최대 수명 (30분, 기본값)
                                   # → 반드시 DB의 wait_timeout보다 짧게 설정
      keepalive-time: 60000        # idle Connection ping 주기 (60초)
                                   # → DB가 끊기 전에 먼저 확인
      connection-test-query: SELECT 1  # DB 연결 검증 쿼리 (기본값 없음)
                                       # → JDBC4+는 isValid() 자동 사용
      
      # 쿼리 설정
      initialization-fail-timeout: 1  # 시작 시 Connection 생성 실패 시 종료 (-1: 계속 시도)
      
      # Pool 이름 (로그/메트릭 식별)
      pool-name: MyServicePool

# MySQL 연동 시 필수 설정:
# max-lifetime (1800000) < MySQL wait_timeout (28800000 기본 = 8시간)
# → MySQL이 Connection 끊기 전에 HikariCP가 먼저 교체
```

### 3. Connection 부족 원인과 진단

```java
// HikariPool 모니터링으로 Connection 부족 감지

// 방법 1: Micrometer + Prometheus (Spring Boot Actuator)
// build.gradle:
// implementation 'io.micrometer:micrometer-registry-prometheus'
// implementation 'org.springframework.boot:spring-boot-starter-actuator'

// 자동으로 노출되는 메트릭:
// hikaricp_connections_active        ← 현재 사용 중인 Connection 수
// hikaricp_connections_idle          ← 대기 중인 Connection 수
// hikaricp_connections_pending       ← 대기 큐의 요청 수 (> 0이면 Pool 포화!)
// hikaricp_connections_timeout_total ← 타임아웃 발생 횟수 (증가하면 위험!)
// hikaricp_connections_usage_seconds ← Connection 사용 시간 분포

// Prometheus 쿼리 (Grafana에서):
// Pool 포화 여부: hikaricp_connections_pending > 0
// 사용률: hikaricp_connections_active / hikaricp_connections_total * 100

// 방법 2: HikariDataSource 직접 조회
@Autowired
HikariDataSource dataSource;

public void monitorPool() {
    HikariPoolMXBean pool = dataSource.getHikariPoolMXBean();
    log.info("Active: {}, Idle: {}, Pending: {}, Total: {}",
        pool.getActiveConnections(),
        pool.getIdleConnections(),
        pool.getThreadsAwaitingConnection(),  // 대기 중인 요청 수!
        pool.getTotalConnections());
}

// 방법 3: 로그 레벨 설정 (개발환경)
# application.yml:
logging:
  level:
    com.zaxxer.hikari: DEBUG
    com.zaxxer.hikari.HikariConfig: DEBUG
    com.zaxxer.hikari.pool.HikariPool: DEBUG
```

### 4. Connection 부족 해결 전략

```java
// 상황별 해결 전략:

// 문제 1: 단순히 트래픽이 늘었을 때
// Pool 크기 증가 (단기 대응)
spring.datasource.hikari.maximum-pool-size=20
// 하지만 DB 서버 용량 확인 필요 (max_connections)

// SHOW VARIABLES LIKE 'max_connections';  -- DB의 최대 Connection 수
// SHOW STATUS LIKE 'Threads_connected';   -- 현재 연결된 Connection 수
// Threads_connected / max_connections * 100 = 사용률 (80% 이상이면 위험)

// 문제 2: Slow Query로 인한 Connection 점유
// → Pool 크기 증가보다 Slow Query 개선이 우선!
// 1초 쿼리를 100ms로 개선 → 동일한 Pool로 10배 처리량
// 10초 쿼리를 1초로 개선 → 동일한 Pool로 10배 처리량

// 문제 3: 트랜잭션 내 외부 HTTP 호출
@Transactional
public void badPattern() {
    Order order = orderRepo.findById(id);  // Connection 획득 (Pool에서 차감)
    
    externalApiCall();  // 2초 소요 → 2초 동안 Connection 점유!
    
    orderRepo.save(order);
    // COMMIT → Connection 반납
}
// 2초 동안 Connection을 잡고 있음 → Pool 포화 원인!

// 올바른 패턴:
public void goodPattern() {
    Order order = fetchOrder(id);  // 별도 트랜잭션, 즉시 Connection 반납
    
    externalApiCall();  // Connection 없음 → Pool에 영향 없음
    
    updateOrder(order);  // 별도 트랜잭션, 빠르게 반납
}

// 문제 4: Read Replica 분리로 Pool 부하 분산
@Transactional(readOnly = true)
public List<Order> getOrders() { ... }  // Read Replica Pool에서 Connection
// spring.datasource.read.url=jdbc:mysql://read-replica-host/...
// spring.datasource.read.hikari.maximum-pool-size=10
```

### 5. MySQL 서버 사이드 설정과의 연동

```sql
-- MySQL 연결 설정 확인
SHOW VARIABLES LIKE 'max_connections';
-- 기본값: 151
-- 권장: 앱 서버 수 × Pool Size + 여유 (DBA, 모니터링용)
-- 예: 서버 5대 × Pool 10 + 20 = 70 설정

SHOW VARIABLES LIKE 'wait_timeout';
-- 기본값: 28800 (8시간)
-- HikariCP max-lifetime: 1800000ms (30분) < 8시간 → OK
-- 하지만 wait_timeout이 60초로 낮은 환경에서는:
-- max-lifetime: 55000ms (55초) < 60초로 조정 필요

SHOW VARIABLES LIKE 'interactive_timeout';
-- 인터랙티브 클라이언트의 timeout (CLI 등)
-- 일반 Connection Pool은 wait_timeout 적용

-- 현재 연결 상태 확인
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';   -- 현재 쿼리 실행 중인 Connection
SHOW STATUS LIKE 'Threads_cached';    -- MySQL 내부 Connection 캐시

-- Connection 별 상태
SHOW FULL PROCESSLIST;
-- Command: Sleep    → idle Connection
-- Command: Query    → 쿼리 실행 중
-- Time: 30         → 30초 동안 실행 중 (Slow Query!)
-- State: Waiting for table metadata lock → MDL 대기 중

-- 오래된 Sleep Connection 확인:
SELECT id, user, host, db, command, time, state
FROM information_schema.PROCESSLIST
WHERE command = 'Sleep' AND time > 600  -- 10분 이상 sleep
ORDER BY time DESC;
```

---

## 💻 실전 실험

### 실험 1: Connection Pool 포화 시뮬레이션

```java
@Test
void simulate_pool_exhaustion() throws Exception {
    // Pool Size = 5, connectionTimeout = 1000ms (1초)로 설정
    
    List<CompletableFuture<Void>> futures = new ArrayList<>();
    
    // 10개의 동시 요청 (Pool=5이므로 5개는 대기)
    for (int i = 0; i < 10; i++) {
        final int idx = i;
        futures.add(CompletableFuture.runAsync(() -> {
            try {
                orderService.slowQuery(idx);  // 3초 걸리는 쿼리
            } catch (Exception e) {
                System.out.println("Request " + idx + " 실패: " + e.getMessage());
                // "Connection is not available, request timed out after 1000ms"
            }
        }));
    }
    
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    // 5개는 성공, 5개는 connectionTimeout으로 실패
}
```

### 실험 2: HikariCP 메트릭 확인

```bash
# Actuator 엔드포인트로 Pool 상태 확인
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active
curl http://localhost:8080/actuator/metrics/hikaricp.connections.idle
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending

# Prometheus 메트릭 확인
curl http://localhost:8080/actuator/prometheus | grep hikari
# hikaricp_connections_active{pool="HikariPool-1"} 3.0
# hikaricp_connections_idle{pool="HikariPool-1"} 7.0
# hikaricp_connections_pending{pool="HikariPool-1"} 0.0  ← 0이면 정상
# hikaricp_connections_timeout_total{pool="HikariPool-1"} 0.0  ← 0이면 정상
```

### 실험 3: max-lifetime과 MySQL wait_timeout 연동

```sql
-- MySQL에서 wait_timeout 낮게 설정 (실험용)
SET GLOBAL wait_timeout = 10;  -- 10초

-- HikariCP max-lifetime = 10000ms (10초)와 동일하게 설정
-- → Race condition 발생 가능!

-- 올바른 설정:
-- max-lifetime < wait_timeout
-- max-lifetime = wait_timeout - 수초 (여유분)
-- 예: wait_timeout=60 → max-lifetime=55000ms

-- 데드 Connection 감지:
SET GLOBAL wait_timeout = 10;
-- 10초 후 MySQL이 Connection 끊음
-- HikariCP: keepaliveTime=5000이면 5초마다 ping → 죽은 Connection 발견 → 교체

SET GLOBAL wait_timeout = 28800;  -- 원복
```

---

## 📊 성능 비교

```
Pool 크기별 처리량 (이론적):

공식: Pool Size = Ncpu × 2 + Neff_disk (DB 서버 기준)

DB 서버: 4 vCPU, SSD
계산: 4 × 2 + 1 = 9

Pool=5 (너무 작음):
  최대 동시 쿼리: 5개
  병목: Pool 대기 발생
  
Pool=9 (공식 기준):
  최대 동시 쿼리: 9개
  DB CPU 활용률: ~100%
  처리량: 최대
  
Pool=20 (너무 큼):
  최대 동시 쿼리: 20개 (이론)
  하지만 DB CPU는 9개 이상 처리 못함
  → Context Switching 오버헤드
  → 오히려 처리량 감소 가능

Connection 사용 시간과 Pool 크기 관계:
  평균 쿼리 시간: 10ms
  TPS 목표: 1000
  필요 Connection = TPS × 평균 쿼리 시간 / 1000
    = 1000 × 0.01 = 10 Connection
  
  느린 쿼리 있을 때 (평균 100ms):
    필요 Connection = 1000 × 0.1 = 100 Connection!
  → Pool 크기 보다 Slow Query 개선이 우선

HikariCP vs C3P0 vs Commons DBCP 비교:
  HikariCP: 초당 수만 건 Connection 대여/반납
  C3P0:     HikariCP 대비 약 3~5배 느림
  Commons DBCP: HikariCP 대비 약 2~3배 느림
  → Spring Boot 기본값이 HikariCP인 이유
```

---

## ⚖️ 트레이드오프

```
Pool 크기:

너무 작음:
  ❌ 트래픽 증가 시 Pool 포화 → connectionTimeout 에러
  ❌ 처리량 제한
  ✅ DB 서버 부하 낮음
  ✅ 메모리 사용 낮음

너무 큼:
  ❌ DB 서버 Context Switching 오버헤드
  ❌ DB 메모리 사용 증가
  ❌ max_connections 초과 위험 (여러 앱 서버 합산)
  ✅ Pool 대기 없음 (단기적으로 안전해 보임)

권장:
  공식(Ncpu×2+1)으로 시작 → 모니터링 → 조정
  pending > 0 자주 발생 시 → Pool 크기 증가 또는 Slow Query 개선

minimumIdle 설정:
  minimumIdle = 0:
    ✅ 트래픽 없을 때 Connection 자원 절약
    ❌ 트래픽 증가 시 Connection 생성 시간(~50ms) 추가
    ❌ 콜드 스타트 지연
  
  minimumIdle = maximumPoolSize:
    ✅ 항상 준비된 Connection → 빠른 응답
    ❌ 트래픽 없어도 Connection 유지 (DB 서버 리소스)
  
  권장: minimumIdle = maximumPoolSize (안정적인 서비스)
  예외: 개발/테스트 환경에서만 minimumIdle = 1~2

connectionTimeout vs Slow Query:
  connectionTimeout 늘리기 (30초 → 60초):
    응급 처방, 근본 해결 아님
    사용자는 60초를 기다려야 함 (더 나쁜 UX)
  
  Slow Query 개선:
    근본 해결
    Connection 점유 시간 줄이기 → 같은 Pool로 더 많은 처리
```

---

## 📌 핵심 정리

```
HikariCP 핵심:

Connection Pool 역할:
  Connection 생성 비용(~50ms) 절약
  재사용으로 고성능 달성

Pool 크기 공식:
  Ncpu × 2 + Neff_disk (DB 서버 CPU 기준)
  AWS db.m5.large (2 CPU): 5개
  모니터링으로 fine-tuning

핵심 설정:
  maximum-pool-size: 공식 기반 (과도하게 크게 하지 말 것)
  minimum-idle: maximum-pool-size와 동일 (안정성)
  connection-timeout: 30000ms (기본, 30초)
  max-lifetime: MySQL wait_timeout보다 수초 짧게
  keepalive-time: 60000ms (60초마다 idle Connection 검증)

에러 진단:
  "Connection is not available": Pool 포화
  원인 1: 트래픽 증가 → Pool 크기 증가 (단기)
  원인 2: Slow Query → 쿼리 최적화 (근본)
  원인 3: 트랜잭션 내 외부 IO → 트랜잭션 범위 축소

모니터링:
  hikaricp_connections_pending > 0: Pool 포화 신호
  hikaricp_connections_timeout_total 증가: 타임아웃 발생
  SHOW STATUS LIKE 'Threads_connected': DB 서버 연결 수
```

---

## 🤔 생각해볼 문제

**Q1.** 서버 10대가 각각 `maximumPoolSize=10`으로 설정돼 있다. MySQL 서버의 `max_connections=151`(기본값)이면 어떤 문제가 생기는가? 해결책은?

<details>
<summary>해설 보기</summary>

**문제**:
10대 × Pool 10 = 최대 100 Connection. 정상적으로는 151 미만이므로 OK입니다.

하지만 **DBA 직접 접속, 모니터링 도구, 슬레이브 복제 Connection** 등을 합산하면:
- 앱 서버 10대 × 10 = 100
- 데이터베이스 관리자 직접 접속: 5
- 모니터링 도구(Percona PMM 등): 3
- 슬레이브 복제: 1
- 합계: ~109 (아직 여유 있음)

하지만 **서버 수가 늘면** 또는 **Pool이 과도하게 크면** `max_connections` 초과 가능:
```
Too many connections (error 1040)
```

**해결책**:

1. **max_connections 증가**:
```sql
SET GLOBAL max_connections = 500;
-- my.cnf: max_connections = 500
-- 단, Connection당 메모리 사용 증가 (스레드, 세션 변수 등)
-- DB 서버 메모리 충분한지 확인
```

2. **Pool 크기 줄이기**:
```yaml
# 공식 적용: DB CPU 4코어이면 Pool = 9
maximum-pool-size: 9  # 10 → 9
# 10대 × 9 = 90 + 여유 = 안전
```

3. **Connection 과금 아키텍처 (ProxySQL)**:
```
앱 서버 10대 (각 Pool=10) → ProxySQL → MySQL
ProxySQL이 Connection을 재사용하여 실제 MySQL Connection 수 줄임
앱 서버 100개 Connection → ProxySQL이 20개 MySQL Connection으로 처리
```

4. **Read Replica 분산**:
```
Write Pool: 앱 × 5 → Primary MySQL
Read Pool:  앱 × 5 → Read Replica MySQL
Primary 부하 절반으로 감소
```

</details>

---

**Q2.** 다음 코드에서 Connection Pool이 빠르게 소진되는 이유를 설명하고, Connection 점유 시간을 줄이는 방법을 제안하라.

```java
@Transactional
public OrderDto placeOrder(OrderRequest request) {
    // 1. 재고 확인
    Product product = productRepo.findByIdWithLock(request.getProductId());
    if (product.getStock() < request.getQuantity()) {
        throw new InsufficientStockException();
    }
    
    // 2. 재고 차감
    product.decreaseStock(request.getQuantity());
    
    // 3. 결제 API 호출 (평균 3초)
    PaymentResult payment = paymentGateway.pay(request.getUserId(), request.getAmount());
    
    // 4. 주문 생성
    Order order = orderRepo.save(new Order(product, payment));
    
    return toDto(order);
}
```

<details>
<summary>해설 보기</summary>

**Connection 소진 이유**:

`@Transactional`로 메서드 시작 시 Connection을 획득하고, 메서드 종료(COMMIT)까지 유지합니다. 이 메서드에 **평균 3초 걸리는 외부 결제 API 호출**이 포함되어 있어:
- 평균 Connection 점유 시간 = DB 쿼리 시간(~50ms) + 결제 API 시간(~3000ms) ≈ 3초
- Pool 10개 × 3초 = 초당 최대 3.3 TPS만 처리 가능!
- 트래픽이 이보다 높으면 Pool 포화 → connectionTimeout 에러

**해결책: 트랜잭션 범위 축소**

```java
// 분리된 패턴:
public OrderDto placeOrder(OrderRequest request) {
    // 단계 1: 재고 예약 (짧은 트랜잭션)
    Long reservationId = reserveStock(request);  // ~50ms, Connection 즉시 반납
    
    // 단계 2: 결제 (트랜잭션 밖, Connection 없음)
    PaymentResult payment;
    try {
        payment = paymentGateway.pay(request.getUserId(), request.getAmount());
    } catch (Exception e) {
        releaseReservation(reservationId);  // 예약 취소 (짧은 트랜잭션)
        throw new PaymentFailedException();
    }
    
    // 단계 3: 주문 확정 (짧은 트랜잭션)
    return confirmOrder(reservationId, payment);  // ~50ms, Connection 즉시 반납
}

@Transactional
private Long reserveStock(OrderRequest request) { ... }  // 50ms만 Connection 사용

@Transactional
private OrderDto confirmOrder(Long reservationId, PaymentResult payment) { ... }

// 이제 Connection 점유 시간 = 50ms (3000ms → 50ms, 60배 개선)
// Pool 10개 × 50ms = 초당 최대 200 TPS 처리 가능!
```

단, 분리된 패턴은 결제 성공 후 주문 저장 실패 시 보상 트랜잭션(결제 취소)이 필요합니다. Outbox 패턴이나 Saga 패턴을 함께 고려해야 합니다.

</details>

---

**Q3.** HikariCP `hikaricp_connections_timeout_total` 메트릭이 갑자기 증가하기 시작했다. 진단 순서를 단계별로 설명하라.

<details>
<summary>해설 보기</summary>

**진단 단계**:

**1단계: 현재 Pool 상태 확인**
```
hikaricp_connections_active: 10/10 (100% → 포화!)
hikaricp_connections_pending: 50 (대기 큐 50개)
```

**2단계: DB에서 현재 실행 중인 쿼리 확인**
```sql
SHOW FULL PROCESSLIST;
-- Time 컬럼이 높은 것 → Slow Query!
-- State = 'Locked' → Lock 대기!
-- 많은 Sleep Connection → 트랜잭션이 끝나지 않음

SELECT * FROM information_schema.INNODB_TRX ORDER BY trx_started;
-- 장기 실행 트랜잭션 확인
```

**3단계: Slow Query 확인**
```sql
SELECT SUM_TIMER_WAIT/1e9/COUNT_STAR AS avg_ms, DIGEST_TEXT
FROM performance_schema.events_statements_summary_by_digest
WHERE COUNT_STAR > 10
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
-- 최근에 느려진 쿼리 있는지 확인
```

**4단계: 트래픽 변화 확인**
```
APM / 모니터링: TPS가 갑자기 증가했는가?
→ 증가했으면: Pool 크기 임시 증가 (단기) + Slow Query 최적화 (장기)
→ 변화 없으면: 특정 Slow Query가 새로 생겼을 가능성
```

**5단계: 즉각 조치**
```sql
-- 장시간 실행 중인 쿼리 KILL
KILL [process_id];  -- SHOW PROCESSLIST에서 확인한 ID

-- Pool 크기 임시 증가 (재시작 없이)
-- application.yml 수정 후 서버 재시작 필요 (런타임 변경 불가)
-- 또는 Read Replica로 읽기 부하 분산
```

**6단계: 근본 원인 해결**
```
원인 A: 새로운 Slow Query → EXPLAIN으로 확인 → 인덱스 추가
원인 B: 트랜잭션 내 외부 호출 → 트랜잭션 범위 축소
원인 C: 영구 트래픽 증가 → DB 스케일 업 또는 Read Replica 추가
```

</details>

---

<div align="center">

**[⬅️ 대용량 최적화](./04-large-table-optimization.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Replication & HA ➡️](../replication-high-availability/01-mysql-replication-binlog.md)**

</div>
