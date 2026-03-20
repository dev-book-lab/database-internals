# Spring에서 Read/Write 분리 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AbstractRoutingDataSource`가 DataSource를 동적으로 선택하는 내부 메커니즘은?
- `@Transactional(readOnly=true)`가 MySQL에 전송하는 힌트와 그 효과는?
- `LazyConnectionDataSourceProxy`가 없으면 어떤 문제가 생기는가?
- 트랜잭션 전파 (Propagation)와 DataSource 라우팅이 충돌하는 경우는?
- Replica 장애 시 Primary로 자동 Fallback하는 구현 방법은?
- Read/Write 분리 구현 후 발생하는 세션 변수 문제와 해결책은?

---

## 🔍 왜 이 개념이 중요한가

### 잘못 구현된 R/W 분리 = 성능 향상도 없고 버그만 생긴다

```
흔한 잘못된 구현:
  1. @Transactional(readOnly=true)만 설정하고
     DataSource 라우팅 로직 없음
  → readOnly 힌트만 있고 실제로는 Primary에서 읽기

  2. LazyConnectionDataSourceProxy 없이 구현
  → @Transactional 진입 시 DataSource 결정 (readOnly 설정 전!)
  → 항상 Primary로 라우팅됨

  3. 트랜잭션 전파 고려 없음
  → 내부 메서드에서 readOnly=false로 바뀌어도 여전히 Replica
  → 쓰기가 Replica에 가려다 에러 (Replica는 READ ONLY)

  4. Replica 장애 처리 없음
  → Replica 다운 → 모든 읽기 요청 실패 → 서비스 장애
  → Primary로 Fallback 필요

올바른 구현을 위해 필요한 것:
  AbstractRoutingDataSource: 라우팅 로직
  LazyConnectionDataSourceProxy: 실제 연결을 늦게 획득
  TransactionSynchronizationManager: 현재 트랜잭션 상태 확인
  Health Check + Fallback: Replica 장애 대응
```

---

## 😱 잘못된 이해

### Before: "@Transactional(readOnly=true)만 붙이면 Replica로 간다"

```java
// 잘못된 믿음:
@Transactional(readOnly = true)
public List<Order> getOrders() {
    return orderRepository.findAll();
    // → "readOnly=true니까 자동으로 Replica에서 읽겠지?"
    // → 아님! readOnly는 힌트일 뿐, DataSource 라우팅 설정 없으면 Primary
}

// 실제로 필요한 것:
// 1. AbstractRoutingDataSource 구현
// 2. 라우팅 키(Primary/Replica) 결정 로직
// 3. @Transactional의 readOnly 속성을 라우팅 키로 사용
// 4. LazyConnectionDataSourceProxy로 감싸기

// 잘못된 결과 2:
@Transactional(readOnly = true)
public List<Order> getOrders() {
    // LazyConnectionDataSourceProxy 없이 구현 시:
    // → @Transactional AOP가 먼저 실행 = BEGIN 트랜잭션
    // → 이 시점의 readOnly = false (기본값)
    // → Routing: Primary 선택됨
    // → 그 다음에 readOnly = true로 변경됨 (하지만 이미 Primary 선택 완료)
    // → Replica로 라우팅 안 됨!
    return orderRepository.findAll();
}
```

---

## ✨ 올바른 이해

### After: 올바른 R/W 분리 = AbstractRoutingDataSource + LazyConnectionDataSourceProxy의 조합

```
핵심 구성 요소:

1. AbstractRoutingDataSource:
   여러 DataSource를 등록하고 런타임에 선택
   determineCurrentLookupKey()를 오버라이드하여 라우팅 결정

2. DataSourceContextHolder:
   ThreadLocal로 현재 스레드의 DataSource 키 보관
   Primary / Replica 전환

3. TransactionSynchronizationManager:
   현재 트랜잭션의 readOnly 여부 제공
   isCurrentTransactionReadOnly() → 라우팅 결정

4. LazyConnectionDataSourceProxy:
   실제 DB Connection 획득을 최대한 지연
   @Transactional 진입 시가 아닌 실제 쿼리 실행 시 Connection 획득
   → readOnly 속성이 설정된 후 Connection 획득 = 올바른 라우팅

5. AOP Order:
   @Transactional AOP보다 DataSource 라우팅 AOP가 먼저 실행
   → readOnly 설정 후 DataSource 결정

흐름:
  HTTP 요청 → @Transactional(readOnly=true) AOP 진입
  → readOnly=true 설정 → LazyConnectionDataSourceProxy
  → 실제 쿼리 실행 시점 → determineCurrentLookupKey() 호출
  → isCurrentTransactionReadOnly() = true → Replica 선택
  → Replica DataSource에서 Connection 획득
  → 쿼리 실행
```

---

## 🔬 내부 동작 원리

### 1. AbstractRoutingDataSource 구현

```java
// 라우팅 키 정의
public enum DataSourceType {
    PRIMARY, REPLICA
}

// ThreadLocal로 라우팅 키 관리
public class DataSourceContextHolder {
    private static final ThreadLocal<DataSourceType> contextHolder = new ThreadLocal<>();
    
    public static void setDataSourceType(DataSourceType type) {
        contextHolder.set(type);
    }
    
    public static DataSourceType getDataSourceType() {
        return contextHolder.get();
    }
    
    public static void clear() {
        contextHolder.remove();
    }
}

// 라우팅 DataSource 구현
public class RoutingDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        // 우선순위 1: 명시적으로 설정된 DataSource 타입
        DataSourceType type = DataSourceContextHolder.getDataSourceType();
        if (type != null) {
            return type;
        }
        
        // 우선순위 2: 현재 트랜잭션의 readOnly 속성
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        return isReadOnly ? DataSourceType.REPLICA : DataSourceType.PRIMARY;
    }
}
```

### 2. DataSource 설정 (Spring Boot)

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }
    
    @Bean
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {
        
        RoutingDataSource routing = new RoutingDataSource();
        
        Map<Object, Object> dataSources = new HashMap<>();
        dataSources.put(DataSourceType.PRIMARY, primary);
        dataSources.put(DataSourceType.REPLICA, replica);
        
        routing.setTargetDataSources(dataSources);
        routing.setDefaultTargetDataSource(primary);  // 기본값: Primary
        routing.afterPropertiesSet();
        
        return routing;
    }
    
    @Primary
    @Bean
    public DataSource dataSource(
            @Qualifier("routingDataSource") DataSource routing) {
        // LazyConnectionDataSourceProxy로 감싸기 (핵심!)
        return new LazyConnectionDataSourceProxy(routing);
        // → 실제 Connection 획득을 쿼리 시점까지 지연
        // → @Transactional의 readOnly가 설정된 후 라우팅 결정
    }
    
    // JPA EntityManagerFactory, TransactionManager도 위 DataSource 사용
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource, JpaVendorAdapter jpaVendorAdapter) {
        LocalContainerEntityManagerFactoryBean emf = new LocalContainerEntityManagerFactoryBean();
        emf.setDataSource(dataSource);  // LazyConnectionDataSourceProxy가 감싼 것
        emf.setJpaVendorAdapter(jpaVendorAdapter);
        emf.setPackagesToScan("com.example");
        return emf;
    }
    
    @Bean
    public PlatformTransactionManager transactionManager(
            LocalContainerEntityManagerFactoryBean emf) {
        return new JpaTransactionManager(emf.getObject());
    }
}
```

```yaml
# application.yml:
spring:
  datasource:
    primary:
      url: jdbc:mysql://primary-host:3306/db
      username: app_user
      password: secret
      hikari:
        maximum-pool-size: 10
        pool-name: primary-pool
    replica:
      url: jdbc:mysql://replica-host:3306/db
      username: app_user
      password: secret
      hikari:
        maximum-pool-size: 20  # 읽기가 많으므로 Replica Pool 더 크게
        pool-name: replica-pool
```

### 3. readOnly가 MySQL에 미치는 영향

```java
// @Transactional(readOnly=true)의 MySQL 레벨 동작:

// 1. Spring이 MySQL에 전송하는 힌트:
// SET TRANSACTION READ ONLY;
// (또는 Connection 레벨 SET SESSION TRANSACTION READ ONLY)

// 2. MySQL MVCC 최적화:
// 일반 SELECT: REPEATABLE READ = Read View 생성 (약간의 오버헤드)
// READ ONLY 트랜잭션:
//   → InnoDB가 Read View 생성을 최적화
//   → undo log 참조를 줄임
//   → 특히 큰 테이블에서 성능 향상

// 3. Undo Log 변경 불가:
// READ ONLY 트랜잭션에서 UPDATE/INSERT 시도:
// ERROR 1792: Cannot execute statement in a READ ONLY transaction.

// 4. Hibernate 1차 캐시 최적화:
// readOnly=true → FlushMode.NEVER
// → Dirty Checking(변경 감지) 비활성화
// → 메모리/CPU 절약

// 실제 확인:
@Transactional(readOnly = true)
public void checkTransactionState() {
    String level = jdbcTemplate.queryForObject(
        "SELECT @@tx_read_only", String.class);
    System.out.println("tx_read_only: " + level);  // 1 (읽기 전용)
    
    String sessionLevel = jdbcTemplate.queryForObject(
        "SELECT @@transaction_read_only", String.class);
    System.out.println("transaction_read_only: " + sessionLevel);  // 1
}
```

### 4. 트랜잭션 전파와 DataSource 충돌 방지

```java
// 주의: 중첩 트랜잭션에서 라우팅 일관성 보장

@Service
public class OrderService {
    
    // readOnly=true → Replica
    @Transactional(readOnly = true)
    public OrderDto getOrderWithUser(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        
        // 내부 메서드 호출 → REQUIRED 전파 = 기존 트랜잭션 참여
        UserDto user = userService.getUser(order.getUserId());
        // → 여기서도 Replica 사용 (기존 readOnly=true 트랜잭션에 참여)
        
        return new OrderDto(order, user);
    }
}

@Service
public class UserService {
    
    // 기존 트랜잭션 있으면 참여 (REQUIRED)
    @Transactional(readOnly = true)
    public UserDto getUser(Long userId) {
        // 기존 트랜잭션의 readOnly 속성 = true → Replica
        return userRepository.findById(userId)
            .map(UserDto::from)
            .orElseThrow();
    }
}

// 위험한 케이스: readOnly 속성 변경
@Transactional(readOnly = true)
public void getAndUpdate(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();  // Replica
    order.updateStatus("DONE");
    orderRepository.save(order);  // 쓰기 시도 → Replica는 READ ONLY → 에러!
    // ERROR 1792: Cannot execute statement in a READ ONLY transaction
}

// 올바른 패턴: 쓰기가 필요하면 readOnly=false (기본값)
@Transactional  // readOnly=false → Primary
public void updateOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();  // Primary에서 읽기
    order.updateStatus("DONE");
    orderRepository.save(order);  // Primary에 쓰기 → 정상
}
```

### 5. Replica 장애 시 Fallback

```java
// Replica 장애 감지 및 Primary Fallback

@Component
public class HealthAwareRoutingDataSource extends AbstractRoutingDataSource {
    
    private final AtomicBoolean replicaHealthy = new AtomicBoolean(true);
    
    // 주기적으로 Replica 상태 확인 (5초마다)
    @Scheduled(fixedDelay = 5000)
    public void checkReplicaHealth() {
        try (Connection conn = replicaDataSource.getConnection()) {
            conn.createStatement().execute("SELECT 1");
            replicaHealthy.set(true);
        } catch (Exception e) {
            log.warn("Replica unhealthy, falling back to primary: {}", e.getMessage());
            replicaHealthy.set(false);
        }
    }
    
    @Override
    protected Object determineCurrentLookupKey() {
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        
        if (isReadOnly && replicaHealthy.get()) {
            return DataSourceType.REPLICA;  // 정상: Replica 사용
        }
        return DataSourceType.PRIMARY;  // Replica 장애 시 Primary Fallback
    }
}

// 여러 Replica 중 라운드로빈:
@Component
public class MultiReplicaRoutingDataSource extends AbstractRoutingDataSource {
    
    private final List<DataSourceType> replicas = List.of(
        DataSourceType.REPLICA_1,
        DataSourceType.REPLICA_2,
        DataSourceType.REPLICA_3
    );
    private final AtomicInteger replicaIndex = new AtomicInteger(0);
    
    @Override
    protected Object determineCurrentLookupKey() {
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        
        if (isReadOnly) {
            // 라운드로빈으로 Replica 선택
            int idx = replicaIndex.getAndIncrement() % replicas.size();
            return replicas.get(idx);
        }
        return DataSourceType.PRIMARY;
    }
}
```

---

## 💻 실전 실험

### 실험 1: 라우팅 동작 확인

```java
@RestController
@RequestMapping("/test")
public class TestController {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @GetMapping("/connection-info")
    @Transactional(readOnly = true)
    public String testReadOnly() {
        // 실제로 어느 서버에 연결됐는지 확인
        String host = jdbcTemplate.queryForObject(
            "SELECT @@hostname", String.class);
        boolean readOnly = "1".equals(jdbcTemplate.queryForObject(
            "SELECT @@tx_read_only", String.class));
        
        return "Connected to: " + host + ", readOnly: " + readOnly;
        // readOnly=true면 Replica 호스트명 반환
    }
    
    @GetMapping("/write-test")
    @Transactional  // readOnly=false → Primary
    public String testWrite() {
        String host = jdbcTemplate.queryForObject(
            "SELECT @@hostname", String.class);
        return "Connected to: " + host;
        // Primary 호스트명 반환
    }
}
```

### 실험 2: LazyConnectionDataSourceProxy 없을 때 동작 확인

```java
// LazyConnectionDataSourceProxy 제거 후:
@Bean
public DataSource dataSource(DataSource routingDataSource) {
    // LazyConnectionDataSourceProxy 제거!
    return routingDataSource;  // 직접 사용
}

// 결과 확인:
@Transactional(readOnly = true)
public String testLazyConnection() {
    String host = jdbcTemplate.queryForObject("SELECT @@hostname", String.class);
    // → LazyConnectionDataSourceProxy 없으면 @Transactional 진입 시 
    //   (readOnly가 아직 false) Connection 획득 → Primary!
    return host;  // Replica 기대하지만 Primary 반환
}

// LazyConnectionDataSourceProxy 추가 후:
@Bean
public DataSource dataSource(DataSource routingDataSource) {
    return new LazyConnectionDataSourceProxy(routingDataSource);
}
// → readOnly=true 설정 후 Connection 획득 → Replica 정상 라우팅
```

---

## 📊 성능 비교

```
R/W 분리 적용 전후:

적용 전 (Primary만 사용):
  읽기 TPS: Primary 제한 (100%)
  쓰기 TPS: Primary (100%)
  Primary 부하: 100%

적용 후 (Replica 1대 추가):
  읽기 TPS: Replica로 분산 (Primary ~20%, Replica ~80% - 서비스 특성에 따라 다름)
  쓰기 TPS: Primary 동일
  Primary 부하: ~30% 감소 (읽기 비중 70%가 Replica로 이동한 경우)

Replica Pool 크기 설정:
  읽기:쓰기 = 8:2인 서비스:
    Primary Pool: 5 (쓰기 + 중요 읽기)
    Replica Pool: 15 (일반 읽기)
    → 읽기 부하를 Replica가 흡수

@Transactional(readOnly=true) 효과 측정:
  readOnly=false: Dirty Checking O(N), FlushMode.AUTO
  readOnly=true:  Dirty Checking X, FlushMode.NEVER
  대규모 조회 (1000건): readOnly=true가 ~5~15% 빠름
  (주로 1차 캐시 Dirty Checking 비용 절약)
```

---

## ⚖️ 트레이드오프

```
R/W 분리 구현:

장점:
  ✅ Primary 읽기 부하 감소
  ✅ 읽기 처리량 수평 확장 가능
  ✅ readOnly=true → Hibernate/MySQL 최적화

단점/복잡도:
  ❌ DataSource 설정 복잡성 증가
  ❌ Replication Lag으로 인한 일관성 문제 (별도 전략 필요)
  ❌ 트랜잭션 전파 버그 발생 가능 (readOnly 속성 주의)
  ❌ Replica 장애 처리 (Fallback 로직 필요)
  ❌ 테스트 복잡도 증가 (H2는 Replica 없음 → 테스트 설정 분리)

LazyConnectionDataSourceProxy 필요성:
  없으면: 항상 Primary로 라우팅됨 (R/W 분리 효과 없음)
  있으면: readOnly 설정 후 Connection 획득 → 올바른 라우팅

세션 변수 문제:
  일부 세션 변수(SET SESSION ...)는 Connection 레벨에 저장
  Connection Pool에서 Connection 재사용 시 이전 세션 변수 오염 가능
  HikariCP: connectionInitSql 또는 connectionTestQuery로 초기화
  
  예: SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED
  → 다른 요청이 같은 Connection을 받아 RC로 실행될 수 있음
  → @Transactional(isolation=...) 사용 후 반드시 Spring이 복원 처리 확인

ProxySQL 대안:
  애플리케이션 레벨 구현 없이 Proxy 레벨에서 R/W 분리
  SELECT → Replica, DML → Primary (SQL 파싱 기반)
  장점: 애플리케이션 코드 변경 없음
  단점: 트랜잭션 내 SELECT는 올바르게 라우팅 (Primary로) 필요 → 설정 주의
```

---

## 📌 핵심 정리

```
Spring R/W 분리 핵심:

필수 구성:
  1. AbstractRoutingDataSource.determineCurrentLookupKey():
     isCurrentTransactionReadOnly() → REPLICA or PRIMARY
  
  2. LazyConnectionDataSourceProxy:
     @Transactional 진입 시가 아닌 실제 쿼리 시 Connection 획득
     → readOnly 설정 후 라우팅 결정 가능
  
  3. Replica 장애 Fallback:
     Health Check + AtomicBoolean으로 Primary로 자동 전환

readOnly=true 효과:
  DB 레벨: SET TRANSACTION READ ONLY → 쓰기 불가, MVCC 최적화
  Hibernate: FlushMode.NEVER, Dirty Checking 비활성화

트랜잭션 전파 주의:
  REQUIRED 전파: 기존 트랜잭션 참여 → 기존 readOnly 유지
  쓰기가 필요한 메서드: @Transactional (readOnly=false, 기본값)
  Replica에서 쓰기 시도 → ERROR 1792

테스트:
  SELECT @@hostname → 어느 서버에 연결됐는지 확인
  SELECT @@tx_read_only → readOnly 트랜잭션 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 `readOnly=true`임에도 Replica가 아닌 Primary로 라우팅되는 이유를 설명하라.

```java
@Service
public class OrderService {
    
    @Transactional  // (1) Primary
    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.process();
        orderRepository.save(order);
        
        sendNotification(orderId);  // (2) 내부 호출
    }
    
    @Transactional(readOnly = true)  // Primary? Replica?
    public void sendNotification(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        // 알림 전송 처리
    }
}
```

<details>
<summary>해설 보기</summary>

**`sendNotification`은 Primary를 사용합니다.**

이유: **REQUIRED 전파(기본값)**

`processOrder`가 이미 `readOnly=false` 트랜잭션을 시작했습니다. `sendNotification`을 같은 클래스 내에서 직접 호출할 경우, Spring AOP 프록시를 거치지 않으므로 `@Transactional(readOnly=true)`가 무시됩니다.

설령 다른 빈에서 호출하더라도 `Propagation.REQUIRED`이므로:
- 현재 활성 트랜잭션(readOnly=false) 발견 → 그 트랜잭션에 참여
- `sendNotification`의 `readOnly=true`는 **새 트랜잭션이 생성될 때만** 적용
- 기존 트랜잭션에 참여하면 기존 readOnly 속성 유지 (false)

**해결 방법**:
1. `sendNotification`을 별도 클래스/빈으로 분리 + `REQUIRES_NEW` 전파
2. `sendNotification`을 트랜잭션 밖으로 이동 (읽기만 한다면 `@Transactional` 불필요할 수도)
3. 자기 자신 주입(Self-Injection) 후 프록시를 통해 호출 (비권장)

</details>

---

**Q2.** 테스트 환경에서는 H2 인메모리 DB를 사용하고 프로덕션에서는 MySQL R/W 분리를 사용한다. 테스트 설정을 어떻게 구성해야 하는가?

<details>
<summary>해설 보기</summary>

H2는 단일 DataSource이고 Read Replica가 없습니다. 다음 방법으로 테스트 환경을 구성합니다.

**방법 1: Profile로 분리**

```java
// 프로덕션 설정:
@Configuration
@Profile("production")
public class ProductionDataSourceConfig {
    // AbstractRoutingDataSource + LazyConnectionDataSourceProxy 설정
}

// 테스트 설정:
@Configuration
@Profile("test")
public class TestDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        // H2 단일 DataSource (R/W 분리 없음)
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}
```

```yaml
# application-test.yml:
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=MySQL
    driver-class-name: org.h2.Driver
```

**방법 2: 테스트용 RoutingDataSource (Primary = Replica = H2)**

```java
@TestConfiguration
public class TestDataSourceConfig {
    
    @Bean
    public DataSource dataSource() {
        DataSource h2 = createH2DataSource();
        
        // 테스트에서는 Primary와 Replica 모두 같은 H2 사용
        RoutingDataSource routing = new RoutingDataSource();
        Map<Object, Object> sources = new HashMap<>();
        sources.put(DataSourceType.PRIMARY, h2);
        sources.put(DataSourceType.REPLICA, h2);  // 동일!
        routing.setTargetDataSources(sources);
        routing.setDefaultTargetDataSource(h2);
        
        return new LazyConnectionDataSourceProxy(routing);
    }
}
```

**방법 3: Mock DataSource로 라우팅만 테스트**

```java
@SpringBootTest
class DataSourceRoutingTest {
    
    @MockBean
    DataSource replicaDataSource;
    
    @Test
    void readOnly_routes_to_replica() {
        // readOnly=true 트랜잭션에서 replicaDataSource.getConnection() 호출 확인
        verify(replicaDataSource, times(1)).getConnection();
    }
}
```

**실무 권장**: Profile 기반 분리가 가장 깔끔합니다. 테스트에서는 H2 단일 DataSource를 사용하고, R/W 분리 로직은 Integration Test (Testcontainers + 실제 MySQL)에서 별도로 검증합니다.

</details>

---

**Q3.** 다음 설정에서 발생하는 세션 변수 오염 문제를 설명하고 해결책을 제시하라.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void criticalOperation() {
    // READ COMMITTED 격리 수준으로 실행
    orderRepository.findAll();
}

@Transactional(readOnly = true)  // 기본 REPEATABLE READ
public List<Order> normalRead() {
    return orderRepository.findAll();
    // 이 메서드에서 실제 격리 수준은?
}
```

<details>
<summary>해설 보기</summary>

**세션 변수 오염 문제**:

1. `criticalOperation()` 실행: Spring이 `SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED` 실행
2. 메서드 종료: Spring이 이전 격리 수준 (REPEATABLE READ)으로 복원해야 함
3. Connection이 Pool에 반납됨

만약 Spring이 복원에 실패하거나 (예외 처리 오류, 버그), 또는 직접 JDBC로 세션 변수를 변경했다면:
4. 다른 요청이 같은 Connection 획득 → READ COMMITTED 격리 수준으로 실행!
5. `normalRead()`가 기대하는 REPEATABLE READ가 아닌 RC로 동작

**확인 방법**:
```java
@Transactional(readOnly = true)
public String checkIsolation() {
    return jdbcTemplate.queryForObject(
        "SELECT @@transaction_isolation", String.class);
    // 기대: REPEATABLE-READ, 오염 시: READ-COMMITTED
}
```

**해결책**:

1. **HikariCP connectionInitSql 설정** (Connection 생성 시 초기화):
```yaml
spring.datasource.hikari.connection-init-sql: |
  SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
```

2. **connectionTestQuery로 상태 리셋** (Connection 재사용 시):
```java
hikariConfig.setConnectionTestQuery("SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ");
```
단, connectionTestQuery는 isValid() 대신 사용하므로 성능 저하 가능

3. **직접 격리 수준 변경 금지** - 항상 `@Transactional(isolation=...)` 사용
Spring이 자동으로 복원 처리: 메서드 종료 시 이전 격리 수준으로 SET SESSION 재실행

4. **DataSource 레벨 기본값 설정**:
```java
hikariConfig.setTransactionIsolation("TRANSACTION_REPEATABLE_READ");
// Connection 생성 시 기본 격리 수준 설정
// Pool에서 꺼낼 때 이 값으로 초기화됨
```

Spring의 `@Transactional(isolation=...)` 메커니즘:
- 진입 시: 현재 격리 수준 저장 → 새 격리 수준 SET SESSION
- 종료 시: 저장한 격리 수준으로 복원 SET SESSION
- 이 복원이 정상적으로 동작하므로 일반적으로 문제 없음
- 단, 직접 JDBC로 세션 변수를 바꾸면 Spring이 추적 못 함 → 오염

</details>

---

<div align="center">

**[⬅️ GTID 페일오버](./03-gtid-auto-failover.md)** | **[홈으로 🏠](../README.md)**

</div>
