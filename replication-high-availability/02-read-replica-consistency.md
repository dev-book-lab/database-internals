# Read Replica — 읽기 분산과 정합성 트레이드오프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Seconds_Behind_Source`가 0이어도 Replication Lag이 있을 수 있는 이유는?
- "방금 쓴 데이터를 Read Replica에서 못 읽는" 시나리오를 정확히 설명하면?
- 쓰기 직후 읽기를 Primary로 라우팅하는 "Read-Your-Writes" 패턴의 구현 방법은?
- Replica에서 읽어야 하는데 강한 일관성도 보장해야 할 때 사용하는 방법은?
- Lag을 모니터링하는 `Seconds_Behind_Source`의 정확한 의미와 한계는?
- `pt-heartbeat`가 Lag 측정에서 `Seconds_Behind_Source`보다 정확한 이유는?

---

## 🔍 왜 이 개념이 중요한가

### Read Replica를 잘못 사용하면 "데이터가 사라졌다"는 버그가 생긴다

```
실제 버그 시나리오:

  1. 사용자: 게시글 제목 수정 (Primary UPDATE)
  2. 즉시 목록 페이지로 리다이렉트 (Replica SELECT)
  3. 목록에서 이전 제목이 보임 → "수정이 안 됐다" 민원
  
  DB 로그 확인:
    Primary: UPDATE 완료 (커밋됨)
    Replica: 아직 반영 안 됨 (Lag 200ms)
    사용자: 200ms 내에 Replica에서 조회 → 이전 데이터 확인

  더 심각한 케이스:
    결제 완료 → 주문 상태를 PAID로 업데이트 (Primary)
    주문 확인 페이지 조회 (Replica)
    → 아직 PENDING 상태 → "결제가 안 됐다" 오류 메시지
    → 사용자가 다시 결제 시도 → 이중 결제!

  이 버그의 특징:
    로컬/스테이징: 재현 안 됨 (Replica 없음)
    프로덕션에서만: Lag이 발생하는 상황에서만
    간헐적: Lag이 짧을 때는 우연히 맞음
    → 가장 찾기 어려운 버그 유형

이것을 해결하려면:
  Read Replica 라우팅 전략을 아키텍처 레벨에서 설계해야 함
```

---

## 😱 잘못된 이해

### Before: "Read Replica에서 읽어도 항상 최신 데이터다"

```
잘못된 코드:
  @Transactional
  public void updateOrder(Long id, String status) {
    orderRepository.updateStatus(id, status);  // Primary에 UPDATE
  }
  
  // 같은 HTTP 요청에서:
  @Transactional(readOnly = true)  // Replica로 라우팅됨!
  public Order getOrder(Long id) {
    return orderRepository.findById(id);  // Replica에서 SELECT
    // 방금 업데이트했는데 이전 데이터 반환!
  }

잘못된 믿음:
  "readOnly=true면 Replica로 가고 최신 데이터를 읽는다"
  → readOnly=true는 Replica 라우팅 트리거일 뿐
  → Replica의 최신성은 Replication Lag에 달려 있음

또 다른 잘못된 믿음:
  "Seconds_Behind_Source = 0이면 완전히 동기화됐다"
  → Seconds_Behind_Source = 0 ≠ 완전히 동기화
  → 이 값은 "마지막으로 처리한 이벤트의 Primary 타임스탬프와 현재 시각의 차이"
  → Replica가 처리할 새 이벤트가 없어서 0으로 보일 수도 있음
  → pt-heartbeat 같은 도구가 더 정확한 이유
```

---

## ✨ 올바른 이해

### After: Read Replica = 최종 일관성, 쓰기 직후 읽기는 명시적 라우팅 필요

```
Replication Lag의 현실:
  일반 상황: 수 ms ~ 수십 ms (거의 실시간)
  부하 높은 상황: 수 초 ~ 수 분
  대형 트랜잭션: 수 분 ~ 수십 분 (적용 완료까지)
  
  → Read Replica는 "최종 일관성(Eventual Consistency)"
  → 언젠가는 Primary와 동일해지지만 즉시는 아님

일관성 수준별 전략:

강한 일관성 필요 (결제, 재고, 잔액):
  → 항상 Primary에서 읽기
  → @Transactional (readOnly=false 기본) → Primary 라우팅

약한 일관성 허용 (게시글 조회, 검색, 통계):
  → Replica에서 읽기 가능
  → @Transactional(readOnly=true) → Replica 라우팅

쓰기 직후 읽기 (Read-Your-Writes):
  → 쓰기 후 일정 시간 또는 조건까지 Primary에서 읽기
  → 또는 "최근 N초 내 쓰기가 있었으면 Primary로"
  → 세션 고정(Session Stickiness): 해당 세션은 항상 Primary

균형 전략 (실무 권장):
  1. @Transactional(readOnly=false): Primary
  2. @Transactional(readOnly=true): Replica
  3. 사용자 세션에 "최근 변경 있음" 플래그 → Primary 유지
  4. 중요 읽기는 @Transactional (readOnly=false)로 Primary 강제
```

---

## 🔬 내부 동작 원리

### 1. Replication Lag 발생 원인

```
Lag 발생 단계:

단계 1 (네트워크 전송 Lag):
  Primary COMMIT → Binary Log 기록
  IO Thread가 Primary에 Binary Log 요청
  네트워크로 Relay Log 수신
  → 일반: 수 ms
  → 네트워크 이슈: 수 초 ~ 수십 초

단계 2 (SQL Thread 처리 Lag):
  Relay Log 이벤트를 DB에 적용
  단일 스레드: 순서대로 처리
  → Primary의 쓰기 TPS > Replica의 적용 TPS → Lag 누적
  → 대형 트랜잭션: 적용 동안 다른 이벤트 대기

Lag이 증가하는 상황:
  Primary에 대량 쓰기 집중 (배치 작업, 대용량 Migration)
  Replica 하드웨어가 Primary보다 사양이 낮음
  단일 큰 트랜잭션 (1건이지만 100만 Row 변경)
  Replica에 다른 쿼리 부하 있음

Lag이 자연적으로 감소하는 상황:
  Primary 쓰기 감소 → SQL Thread가 따라잡기
  병렬 복제 활성화 → Relay Log 적용 속도 향상
```

### 2. Seconds_Behind_Source 측정 원리와 한계

```sql
-- Seconds_Behind_Source 계산 방식:
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source: 5

-- 계산 방법:
-- Seconds_Behind_Source = UNIX_TIMESTAMP() - [현재 적용 중인 이벤트의 Primary 타임스탬프]
-- 
-- 예:
-- Primary에서 14:00:00에 발생한 이벤트를
-- Replica가 14:00:05에 적용 중이라면:
-- Seconds_Behind_Source = 5

-- 한계 1: Replica가 처리할 이벤트가 없으면 0으로 표시
-- Replica가 완전히 따라잡았거나 Primary에 쓰기가 없으면
-- 마지막 이벤트 처리 후 새 이벤트 없음 → 0 표시
-- → 실제 Lag이 있어도 0으로 보일 수 있음

-- 한계 2: Network Lag은 포함 안 됨
-- IO Thread가 받지 못한 이벤트 = Relay Log에 없음
-- → SQL Thread 관점에서는 할 일이 없음 → 0으로 표시
-- 하지만 Binary Log에는 아직 전송 안 된 이벤트 있을 수 있음

-- 더 정확한 Lag 측정: pt-heartbeat (Percona Toolkit)
-- Primary에 주기적으로 현재 타임스탬프 기록:
-- pt-heartbeat --update --database percona --daemonize
-- 
-- Replica에서 실제 Lag 측정:
-- pt-heartbeat --monitor --database percona
-- → heartbeat 테이블의 타임스탬프 차이 = 실제 Lag
-- → IO Thread + SQL Thread 모두 포함한 End-to-End Lag

-- Binary Log 위치로 Lag 확인:
-- Primary:
SHOW BINARY LOG STATUS\G  -- File + Position

-- Replica:
SHOW REPLICA STATUS\G
-- Source_Log_File + Exec_Source_Log_Pos (SQL Thread 위치)
-- Source_Log_File + Read_Source_Log_Pos (IO Thread 위치)
-- 차이가 크면 큰 Lag
```

### 3. Read-Your-Writes 패턴 구현

```java
// 방법 1: 세션 플래그 기반 (Spring + ThreadLocal)

@Component
public class ReadWriteRoutingContext {
    
    // 쓰기 후 일정 시간 동안 Primary 강제 사용
    private static final ThreadLocal<Long> lastWriteTime = new ThreadLocal<>();
    private static final long PRIMARY_LOCK_DURATION_MS = 2000;  // 2초
    
    public static void markWrite() {
        lastWriteTime.set(System.currentTimeMillis());
    }
    
    public static boolean shouldUsePrimary() {
        Long writeTime = lastWriteTime.get();
        if (writeTime == null) return false;
        return System.currentTimeMillis() - writeTime < PRIMARY_LOCK_DURATION_MS;
    }
    
    public static void clear() {
        lastWriteTime.remove();
    }
}

// @Transactional(readOnly=false) 후 markWrite() 호출
@Transactional
public void updateOrder(Long id, String status) {
    orderRepository.updateStatus(id, status);
    ReadWriteRoutingContext.markWrite();  // 쓰기 완료 표시
}

// 라우팅 결정:
public DataSource getDataSource() {
    if (ReadWriteRoutingContext.shouldUsePrimary()) {
        return primaryDataSource;  // 쓰기 직후 2초는 Primary
    }
    if (isReadOnly()) {
        return replicaDataSource;   // 일반 읽기는 Replica
    }
    return primaryDataSource;       // 기본은 Primary
}

// 방법 2: GTID 기반 (가장 정확하지만 복잡)
// 쓰기 후 Primary의 현재 GTID 확인
// Replica에서 해당 GTID가 적용됐는지 확인
// → 적용됐으면 Replica 사용, 아니면 Primary 사용

// Primary에서:
// SELECT @@gtid_executed; → 현재까지 실행된 GTID 집합

// Replica에서:
// SELECT WAIT_FOR_EXECUTED_GTID_SET('uuid:1-1000', 1);
// → 1 초 내에 GTID 적용 완료되면 0 반환 (성공)
// → 타임아웃이면 1 반환 (아직 적용 안 됨)
```

### 4. Lag 모니터링과 임계값 설정

```sql
-- 프로덕션 모니터링 쿼리:
SELECT
    CHANNEL_NAME,
    SERVICE_STATE AS io_state,
    GTID_DIFFERENCE_WITH_PRIMARY AS gtid_behind
FROM performance_schema.replication_connection_status\G

SELECT
    CHANNEL_NAME,
    SERVICE_STATE AS sql_state
FROM performance_schema.replication_applier_status\G

-- Lag 임계값 알람 설정 (Prometheus 기준):
-- seconds_behind_source > 30 → WARNING
-- seconds_behind_source > 300 → CRITICAL
-- seconds_behind_source = NULL → Replica 중단! CRITICAL

-- Lag 관련 MySQL 메트릭:
SHOW STATUS LIKE 'Replica_heartbeat_period';  -- heartbeat 주기
SHOW STATUS LIKE 'Replica_last_heartbeat';    -- 마지막 heartbeat 시각

-- Relay Log 크기 (아직 적용 안 된 이벤트):
SHOW REPLICA STATUS\G
-- Relay_Log_Space: Relay Log 총 크기
-- Relay_Log_Pos vs Relay_Log_File 위치로 남은 양 계산

-- 병렬 복제 상태 확인:
SELECT * FROM performance_schema.replication_applier_status_by_worker\G
-- WORKER_ID: 1, 2, 3, ... (병렬 워커)
-- APPLYING_TRANSACTION: 현재 적용 중인 트랜잭션
-- LAST_APPLIED_TRANSACTION: 마지막 완료 트랜잭션
```

### 5. 강한 일관성이 필요한 경우 처리 패턴

```java
// 패턴 1: 중요 데이터는 항상 Primary에서 읽기
public class OrderService {
    
    // 결제, 재고, 잔액: 강한 일관성 필요 → Primary 필수
    @Transactional  // readOnly=false (기본) → Primary 라우팅
    public void processPayment(Long orderId) {
        Order order = orderRepository.findByIdForUpdate(orderId);  // FOR UPDATE
        // 최신 상태 확인 후 처리
    }
    
    // 게시글 조회, 검색: 약한 일관성 허용 → Replica 가능
    @Transactional(readOnly = true)  // → Replica 라우팅
    public List<Order> searchOrders(String keyword) {
        return orderRepository.search(keyword);
    }
}

// 패턴 2: Replica에서 읽되 WAIT_FOR_EXECUTED_GTID_SET으로 동기화 확인
@Transactional
public void writeAndRead(OrderDto dto) {
    orderRepository.save(toEntity(dto));
    // 현재 GTID 기록
    String currentGtid = getCurrentGtid();
    
    // 별도 메서드에서 Replica 조회
    // → WAIT_FOR_EXECUTED_GTID_SET(currentGtid, 2) 호출
    //   성공 시 Replica 사용, 타임아웃 시 Primary fallback
}

// 패턴 3: 캐시 무효화로 일관성 보완
@Transactional
public void updatePost(Long id, String title) {
    postRepository.updateTitle(id, title);
    cache.delete("post:" + id);  // Replica 캐시 무효화
    // 다음 조회 시 캐시 miss → Primary에서 재로드 → 캐시 저장
}
```

---

## 💻 실전 실험

### 실험 1: Replica Lag 측정 비교

```sql
-- Replica 서버에서:

-- 방법 1: SHOW REPLICA STATUS (빠르지만 부정확)
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source 확인

-- 방법 2: Performance Schema (더 세밀)
SELECT
    w.WORKER_ID,
    w.APPLYING_TRANSACTION,
    TIMESTAMPDIFF(SECOND,
        w.APPLYING_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP,
        NOW()) AS lag_sec
FROM performance_schema.replication_applier_status_by_worker w
WHERE w.APPLYING_TRANSACTION != '';

-- Primary에서 테스트 쓰기 후 Lag 측정:
-- Primary:
INSERT INTO lag_test VALUES (NOW());
SHOW BINARY LOG STATUS;  -- Position 기록

-- Replica:
-- Exec_Source_Log_Pos가 위에서 기록한 Position에 도달했는지 확인
SELECT Exec_Source_Log_Pos FROM SHOW REPLICA STATUS;
```

### 실험 2: 쓰기 직후 Replica 읽기 — Lag 재현

```python
# Python으로 Lag 재현:
import pymysql
import time

primary = pymysql.connect(host='primary', ...)
replica = pymysql.connect(host='replica', ...)

# Primary에 INSERT
cursor_p = primary.cursor()
cursor_p.execute("INSERT INTO posts (title) VALUES ('Test Post')")
primary.commit()
insert_id = cursor_p.lastrowid

# 즉시 Replica에서 조회
cursor_r = replica.cursor()
cursor_r.execute("SELECT * FROM posts WHERE id = %s", (insert_id,))
result = cursor_r.fetchone()

if result is None:
    print("Lag 재현! Replica에서 못 읽음")
    # Lag이 있으면 None 반환
    
# 100ms 후 재시도
time.sleep(0.1)
cursor_r.execute("SELECT * FROM posts WHERE id = %s", (insert_id,))
result = cursor_r.fetchone()
print("100ms 후:", result)  # 대부분의 경우 이제 보임
```

### 실험 3: 병렬 복제 Lag 개선 효과

```sql
-- Replica에서:

-- 단일 스레드 설정
SET GLOBAL replica_parallel_workers = 0;

-- Primary에서 대량 쓰기 후 Lag 측정
-- SHOW REPLICA STATUS → Seconds_Behind_Source 기록

-- 병렬 복제 활성화
SET GLOBAL replica_parallel_workers = 4;
SET GLOBAL replica_parallel_type = 'LOGICAL_CLOCK';

-- 같은 쓰기 워크로드 후 Lag 재측정
-- 차이 비교: 단일 스레드 vs 4 워커
```

---

## 📊 성능 비교

```
읽기 라우팅 전략별 특성:

모든 읽기 → Primary:
  일관성: 완전
  처리량: Primary 부하 높음 (Read Replica 이점 없음)
  적합: 소규모, 모든 읽기에 강한 일관성 필요

모든 읽기 → Replica:
  일관성: 최종 일관성 (Lag에 따라)
  처리량: 높음 (Primary 읽기 부하 감소)
  위험: 쓰기 직후 읽기 불일치

쓰기/읽기 분리 (readOnly 기반):
  @Transactional(readOnly=true) → Replica
  @Transactional → Primary
  일관성: 트랜잭션 내는 보장, 트랜잭션 간은 Lag 있음
  처리량: 균형 (일반 읽기는 Replica, 중요 읽기는 Primary)

Read-Your-Writes 추가:
  쓰기 직후 2초 = Primary
  이후 = Replica
  일관성: 사용자별 "내가 쓴 것은 보인다" 보장
  처리량: 쓰기가 많으면 Primary 부하 지속

Replica Lag 수치 (일반 환경):
  정상: 0~50ms
  주의: 50ms~1s
  위험: 1s~10s
  장애: >10s 또는 NULL (복제 중단)
```

---

## ⚖️ 트레이드오프

```
Read Replica 사용:

장점:
  ✅ Primary 읽기 부하 감소
  ✅ 읽기 처리량 선형 확장 (Replica 추가로)
  ✅ Primary 장애 시 Replica로 Failover 가능
  ✅ 보고서, 분석 쿼리 Primary 영향 없이 실행

단점/주의사항:
  ❌ 최종 일관성 → 쓰기 직후 읽기 불일치 가능
  ❌ Lag 관리 필요 (모니터링, 임계값 알람)
  ❌ 라우팅 로직 복잡도 증가
  ❌ 트랜잭션 간 일관성 없음

일관성 vs 성능 트레이드오프:
  강한 일관성: Primary만 사용 → 성능 제한
  약한 일관성: Replica 사용 → 성능 향상, 불일치 위험

실무 권장:
  1. 쓰기 후 읽기: Primary 또는 Read-Your-Writes
  2. 결제/재고/잔액: 항상 Primary
  3. 게시글/검색/통계: Replica 허용
  4. Lag 모니터링 필수 (30초 초과 시 알람)
  5. Replica 중단 시 Primary fallback 로직 구현
```

---

## 📌 핵심 정리

```
Read Replica 정합성 핵심:

Replication Lag:
  비동기 복제 → 쓰기 후 즉시 Replica에서 못 읽을 수 있음
  일반: 수 ms, 부하 시: 수 초 ~ 수 분

Seconds_Behind_Source:
  SQL Thread의 현재 적용 이벤트 기준
  0 ≠ 완전 동기화 (IO Thread Lag 미포함)
  pt-heartbeat가 더 정확한 End-to-End Lag 측정

일관성 전략:
  강한 일관성: Primary에서만 읽기
  Read-Your-Writes: 쓰기 후 일정 시간 Primary 유지
  GTID Wait: Replica에서 특정 GTID 적용 확인 후 읽기

라우팅 기준:
  @Transactional(readOnly=false) → Primary
  @Transactional(readOnly=true) → Replica
  쓰기 직후: Primary 유지 (ThreadLocal 플래그 또는 GTID)

모니터링:
  seconds_behind_source > 30: 경고
  Replica_SQL_Running = No: 복제 중단 (즉시 대응)
  hikari_connections_pending > 0 on primary: Primary 과부하
```

---

## 🤔 생각해볼 문제

**Q1.** SNS 서비스에서 "팔로우" 버튼을 누른 후 팔로워 목록을 즉시 조회했을 때 팔로우가 반영되지 않는 현상을 어떻게 처리해야 하는가? 두 가지 전략을 설명하라.

<details>
<summary>해설 보기</summary>

**전략 1: Read-Your-Writes — 쓰기 직후 Primary 유지**

```java
// 팔로우 API
@Transactional
public void follow(Long followerId, Long followeeId) {
    followRepository.save(new Follow(followerId, followeeId));
    ReadWriteContext.markWrite(followerId);  // 이 사용자는 2초간 Primary 읽기
}

// 팔로워 목록 조회 API
@Transactional(readOnly = true)
public List<User> getFollowers(Long userId, Long requesterId) {
    // requesterId가 최근 쓰기를 했으면 Primary 사용
    // → AbstractRoutingDataSource에서 자동 처리
    return followRepository.findFollowers(userId);
}
```
장점: 간단, 쓴 사람은 즉시 확인 가능  
단점: 쓰기가 많은 사용자는 계속 Primary 사용

**전략 2: UI 레벨에서 낙관적 업데이트 (Optimistic UI)**

```javascript
// 프론트엔드에서:
// 1. 팔로우 버튼 클릭 시 → 즉시 UI 업데이트 (API 응답 기다리지 않음)
followButton.setFollowing(true);  // 즉시 "팔로잉 중" 표시
followCount.increment();          // 즉시 카운트 증가

// 2. 백그라운드에서 API 호출
api.follow(followeeId).then(() => {
    // 성공: UI는 이미 업데이트됨
}).catch(() => {
    // 실패: 롤백
    followButton.setFollowing(false);
    followCount.decrement();
});
```
장점: 가장 빠른 UX, DB 라우팅 변경 불필요  
단점: 실제 실패 시 롤백 처리 필요, 복잡한 상황에서 버그 가능

**실무 선택**: 두 전략을 조합합니다. 팔로우/좋아요 같은 단순 상태는 Optimistic UI, 결제/주문 같은 중요 데이터는 Read-Your-Writes + Primary 강제 읽기.

</details>

---

**Q2.** Replica의 `Seconds_Behind_Source = 0`이고 Primary와 Replica의 데이터를 직접 비교했더니 Primary에는 있는 Row가 Replica에 없다. 어떤 원인인가?

<details>
<summary>해설 보기</summary>

**원인 1: IO Thread와 SQL Thread의 위치 차이**

`Seconds_Behind_Source = 0`은 SQL Thread가 현재 처리 중인 이벤트의 타임스탬프 기준입니다. 하지만:
- IO Thread가 아직 받지 못한 이벤트가 있을 수 있음
- SQL Thread가 방금 막 따라잡았지만 최신 이벤트는 아직 수신 전

**진단**:
```sql
SHOW REPLICA STATUS\G
-- Read_Source_Log_Pos (IO Thread 위치)
-- Exec_Source_Log_Pos (SQL Thread 위치)
-- 두 값과 Primary의 SHOW BINARY LOG STATUS Position 비교
```

**원인 2: Multi-Source 또는 Filtered 복제**

```sql
-- 특정 데이터베이스/테이블을 복제에서 제외했을 수 있음
SHOW REPLICA STATUS\G
-- Replicate_Ignore_Table: 제외된 테이블 목록
-- Replicate_Wild_Ignore_Table: 패턴으로 제외된 것
```

**원인 3: 복제 필터링**

```sql
-- my.cnf에서:
-- replicate-ignore-db = test_db
-- replicate-do-table = production.%
-- → 특정 테이블은 복제 제외 → Replica에 없음
```

**원인 4: 복제 오류 후 건너뜀**

과거에 `SQL_REPLICA_SKIP_COUNTER`로 에러 이벤트를 건너뛰면서 해당 데이터가 적용 안 됐을 가능성.

**진단 방법**:
```sql
-- Primary의 Binary Log에서 해당 Row INSERT 이벤트 찾기:
mysqlbinlog mysql-binlog.000023 | grep "해당 Row ID"
-- Replica의 Exec 위치보다 이전에 있으면 적용됐어야 함
-- 없으면 → 복제 필터 또는 Skip
```

</details>

---

**Q3.** `WAIT_FOR_EXECUTED_GTID_SET`을 사용해 Replica에서 강한 일관성 읽기를 구현했다. 타임아웃이 자주 발생한다면 이 방법의 한계와 대안을 설명하라.

<details>
<summary>해설 보기</summary>

**`WAIT_FOR_EXECUTED_GTID_SET`의 한계**:

```sql
-- Replica에서:
SELECT WAIT_FOR_EXECUTED_GTID_SET('source_uuid:1-12345', 1);
-- 1초 내에 GTID 1-12345까지 적용되면 0 반환 (성공)
-- 타임아웃이면 1 반환
```

타임아웃이 자주 발생하는 이유:
1. Replica Lag이 타임아웃값(1초)보다 큼
2. 대형 트랜잭션 적용 중 → 한동안 다른 이벤트 대기
3. 네트워크 이슈로 Binary Log 전달 지연

**한계**:
- 타임아웃 → Primary fallback 필요 → 결국 Primary 부하 증가
- 쓰기 후마다 GTID 전달 → API 응답에 GTID 포함 → 아키텍처 복잡도
- 세션 관리, 직렬화 오버헤드

**대안**:

1. **세션 고정(Session Stickiness) + TTL**:
```java
// 쓰기 후 2초간 Primary 사용 (GTID 추적 없음)
// 단순하고 효과적, 정확한 Lag 보장은 없지만 실용적
```

2. **중요 데이터는 항상 Primary**:
```java
// 결제, 재고 → Primary에서만 읽기
// 설계 레벨에서 해결 (GTID 타임아웃 없음)
```

3. **ProxySQL의 `causal_reads` 옵션**:
```sql
-- ProxySQL이 자동으로 GTID 추적 및 라우팅 처리
-- 애플리케이션 코드 변경 없이 Read-Your-Writes 보장
```

4. **MySQL InnoDB Cluster (Group Replication)**:
```
동기 복제 → Lag 없음 → WAIT 불필요
단, 쓰기 성능 약간 저하
```

**실무 결론**: GTID Wait은 이론적으로 깔끔하지만 타임아웃 처리 복잡도와 Replica Lag 의존성 때문에 실무에서는 단순한 "쓰기 후 X초간 Primary" 또는 "중요 읽기는 무조건 Primary" 전략이 더 자주 선택됩니다.

</details>

---

<div align="center">

**[⬅️ Binary Log 복제](./01-mysql-replication-binlog.md)** | **[홈으로 🏠](../README.md)** | **[다음: GTID와 페일오버 ➡️](./03-gtid-auto-failover.md)**

</div>
