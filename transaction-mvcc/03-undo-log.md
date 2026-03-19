# Undo Log — 이전 버전을 보관하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Undo Log가 물리적으로 어느 파일에 어떤 구조로 저장되는가?
- Insert Undo와 Update Undo가 생명주기가 다른 이유는?
- 롤백 시 Undo Log를 "역방향으로 적용"한다는 것의 구체적 과정은?
- Purge Thread는 어떤 조건이 충족됐을 때 Undo Log를 삭제하는가?
- 오래된 Read View가 Undo Tablespace를 팽창시키는 연쇄 과정은?
- `History list length`가 높아지면 SELECT 성능에도 영향을 미치는 이유는?

---

## 🔍 왜 이 개념이 중요한가

### Undo Log를 이해해야 "왜 디스크가 갑자기 꽉 찼지?"를 설명할 수 있다

```
실제 장애 패턴:
  서비스가 정상 운영 중인데 DB 서버 디스크가 90%→100%→장애

  원인 추적:
    df -h → /var/lib/mysql 가 갑자기 수십 GB 증가
    ls -lh /var/lib/mysql/undo_001 → 갑자기 30GB!

  근본 원인:
    어딘가 장기 실행 트랜잭션(배치, 분석 쿼리, 미커밋 트랜잭션)이 있음
    그 트랜잭션의 Read View가 살아있는 동안
    다른 모든 UPDATE/DELETE의 Undo Log를 삭제 못 함
    → Undo Tablespace 무제한 팽창

  왜 개발자도 알아야 하는가:
    JPA의 @Transactional 메서드가 외부 HTTP 호출을 포함하면
    → 응답 대기 시간만큼 트랜잭션이 열려있음
    → 그동안 쌓이는 Undo Log
    초당 1000건의 UPDATE가 있고 HTTP 호출이 30초라면
    → 30,000건의 Undo Log가 삭제 불가 상태로 쌓임

이것을 이해하면:
  트랜잭션을 짧게 유지해야 하는 이유를 내부 원리로 설명 가능
  배치 처리를 작은 단위로 커밋해야 하는 이유 이해
  분석 쿼리를 Read Replica로 분리해야 하는 이유 이해
```

---

## 😱 잘못된 이해

### Before: Undo Log는 롤백에만 쓰이고 COMMIT하면 즉시 삭제된다

```
잘못된 멘탈 모델:

  트랜잭션 실행:
    변경 → Undo Log 기록
  
  COMMIT:
    변경 확정 → Undo Log 삭제 (더 이상 필요 없으니까)
  
  ROLLBACK:
    Undo Log 사용 → 원상 복구 → Undo Log 삭제

실제:
  COMMIT 후에도 Undo Log는 즉시 삭제되지 않음!
  이유:
    다른 트랜잭션이 이 Row의 이전 버전을 읽어야 할 수 있음 (MVCC)
    Read View가 남아있는 한 삭제 불가

실질적 영향 (오해에서 비롯된 문제):
  "COMMIT 했으니까 Undo Log 공간은 즉시 돌아온다" → 아님
  "트랜잭션이 짧아도 Undo Log는 금방 정리된다" → 다른 장기 트랜잭션 있으면 아님
  "배치를 한 트랜잭션으로 처리하면 오히려 더 효율적이다" → 반대
    → 큰 트랜잭션 = 큰 Undo Log = 롤백 비용 증가 + Purge 지연
```

---

## ✨ 올바른 이해

### After: Undo Log는 Atomicity와 MVCC 두 가지 목적으로 사용된다

```
Undo Log의 이중 역할:

목적 1 — Atomicity (롤백):
  ROLLBACK 발생 시 Undo Log를 역방향으로 적용
  → 트랜잭션의 모든 변경을 취소
  이 역할이 끝나면 Insert Undo는 바로 삭제 가능

목적 2 — MVCC (이전 버전 읽기):
  다른 트랜잭션의 Read View가 이 Row의 이전 버전을 필요로 할 때
  → Undo Log에서 그 버전을 제공
  이 역할이 끝날 때까지 (모든 Read View가 불필요해질 때까지) 삭제 불가

두 역할의 생명주기 차이:

  INSERT undo:
    목적: 삽입된 Row를 롤백 시 삭제하기 위해
    삭제 시점: COMMIT 직후 (다른 트랜잭션이 "없던 Row의 이전 버전"을 볼 필요 없음)
    → Insert Undo는 빠르게 Purge됨

  UPDATE / DELETE undo:
    목적 1: 롤백 시 이전 값으로 복원 (COMMIT 후 목적 1 완료)
    목적 2: 다른 트랜잭션의 MVCC 이전 버전 제공 (목적 2는 계속 유지)
    삭제 시점: 가장 오래된 활성 Read View보다 오래된 TRX_ID의 Undo만 삭제 가능
    → Update/Delete Undo는 오래 남을 수 있음

Purge Thread:
  백그라운드에서 주기적으로 실행
  "현재 가장 오래된 Read View의 m_up_limit_id = X"
  → TRX_ID < X인 Undo Log는 어떤 Read View도 참조 안 함 → 삭제 가능
  → TRX_ID >= X인 Undo Log는 아직 누군가 볼 수 있음 → 삭제 불가
```

---

## 🔬 내부 동작 원리

### 1. Undo Log 물리 저장 구조

```
MySQL 8.0 Undo Tablespace 구조:

파일 위치: datadir/undo_001, datadir/undo_002 (기본)
  innodb_undo_tablespaces: 파일 수 (기본값 2, 최대 127)
  innodb_undo_directory: 저장 경로 (기본 datadir)

계층 구조:
  Undo Tablespace (파일)
    └── Rollback Segment (최대 128개)
          └── Undo Log Segment
                └── Undo Log Records (각 변경 기록)

Rollback Segment 역할:
  동시 트랜잭션의 Undo Log를 분리하여 경합 감소
  MySQL 8.0: 최대 128 Rollback Segments × 2 Tablespace = 256개 병렬 처리 가능
  → 고동시성 쓰기에서 Undo Log 기록 경합 최소화

Undo Record 구조:
  [헤더: 타입, 이전 Undo Record 위치]
  [테이블 ID]
  [TRX_ID: 이 레코드를 만든 트랜잭션 ID]
  [변경 전 컬럼 값들]
  
  INSERT undo: [테이블ID] + [PK 값만]
  UPDATE undo: [테이블ID] + [변경된 컬럼의 이전 값 목록]
  DELETE undo: [테이블ID] + [삭제된 Row 전체]
```

### 2. 롤백 과정 상세 — 역방향 적용

```
시나리오: TRX 500이 3번의 변경 후 ROLLBACK

TRX 500 실행 순서:
  1. UPDATE accounts SET balance=8000 WHERE id=1  → Undo_A 생성 (이전: 10000)
  2. UPDATE accounts SET balance=6000 WHERE id=1  → Undo_B 생성 (이전: 8000)
  3. UPDATE products SET stock=5 WHERE id=1       → Undo_C 생성 (이전: 10)

Undo Log 체인:
  Undo_C → Undo_B → Undo_A → NULL

ROLLBACK 실행 (역방향):
  Step 1: Undo_C 적용
    → products.stock = 10으로 복원 (5 → 10)
    → DB_TRX_ID 이전 값으로 복원
    → Undo_C 삭제 가능 상태로 마킹

  Step 2: Undo_B 적용
    → accounts.balance = 8000으로 복원 (6000 → 8000)

  Step 3: Undo_A 적용
    → accounts.balance = 10000으로 복원 (8000 → 10000)

최종 상태: 트랜잭션 500 이전과 완전히 동일

대량 변경 롤백 비용:
  100만 건 UPDATE 후 ROLLBACK:
    → Undo_1 ~ Undo_1,000,000을 역순으로 적용
    → 100만 건의 UPDATE와 동일한 I/O 발생
    → UPDATE가 5분 걸렸다면 ROLLBACK도 ~5분

  실무 교훈:
    배치 UPDATE는 소량씩 커밋 (만약 실수해도 롤백 부담 감소)
    "한 번에 100만 건 UPDATE + 확인 후 커밋"은 위험한 패턴
```

### 3. Purge Thread — Undo Log 삭제 메커니즘

```
Purge Thread 동작:

설정:
  innodb_purge_threads: 병렬 Purge Thread 수 (기본 4)
  innodb_purge_batch_size: 한 번에 처리할 Undo Log 레코드 수 (기본 300)

삭제 판단 기준:
  전체 활성 Read View 중 가장 오래된 것의 m_up_limit_id = X
  → TRX_ID < X 인 UPDATE/DELETE Undo Log = 삭제 가능

실제 삭제 프로세스:
  ① Purge Thread: "현재 가장 오래된 Read View의 TRX 시작 시점 = TRX 1000"
  ② TRX_ID < 1000인 Undo Log 후보들을 Purge 큐에 추가
  ③ 배치로 물리적 삭제 실행
  ④ 삭제된 공간을 Undo Segment의 Free List에 반환

Delete Mark와 Purge의 관계:
  DELETE 실행 시: Row에 "삭제 표시(Delete Mark)"만 함 (물리적 삭제 아님)
  이유: MVCC에서 다른 트랜잭션이 이 Row의 이전 버전을 읽어야 할 수 있음
  Purge Thread가 나중에 물리적으로 제거:
    ① Delete Mark된 Row에 대한 Undo Log Purge 가능 판단
    ② Row 물리적 삭제
    ③ 인덱스에서 해당 엔트리 제거

History List Length:
  Purge 대기 중인 Undo Log 레코드 수
  SHOW ENGINE INNODB STATUS\G 에서 확인:
    "History list length: N"
  정상: N < 1,000
  주의: N > 10,000
  위험: N > 100,000 → Purge가 쓰기 속도를 따라가지 못함
```

### 4. Undo Tablespace 팽창 시나리오

```
팽창 발생 메커니즘:

T=0:    분석 쿼리 시작 → Read View 생성 (m_up_limit_id = 1000)
T=0~30분: 운영 DB에서 매초 2000건의 UPDATE 발생
          → 매초 2000개의 Update Undo 레코드 생성
          → TRX_ID = 1001 ~ 1,800,001 (30분 × 60초 × 1000 TRX)

Purge Thread 판단:
  "가장 오래된 Read View: m_up_limit_id = 1000"
  "TRX_ID 1001부터의 Undo Log는 아직 이 Read View가 볼 수 있음"
  → 30분치 Undo Log 전부 삭제 불가
  → 1,800,000개 Undo Record × 평균 200 bytes = ~360MB (단순 계산)
  → 실제: Row 크기가 크면 수 GB ~ 수십 GB 팽창 가능

innodb_undo_log_truncate (MySQL 8.0, 기본 ON):
  Undo Tablespace가 innodb_max_undo_log_size(기본 1GB) 초과 시
  → Truncate 실행 (파일 실제 크기 감소)
  단, Truncate 중 약간의 성능 영향 있음
  → 크게 팽창하기 전에 근본 원인(장기 트랜잭션) 해결이 우선

팽창 방지 전략:
  1. 트랜잭션을 짧게: @Transactional 내에서 외부 IO 금지
  2. 배치는 소량 커밋: LIMIT 1000씩 나눠 처리
  3. 분석 쿼리는 Read Replica로: 운영 DB의 Undo Log에 영향 없음
  4. innodb_max_undo_log_size 모니터링 설정
```

### 5. SAVEPOINT와 부분 롤백

```sql
-- SAVEPOINT를 이용한 부분 롤백

START TRANSACTION;

INSERT INTO orders (user_id, amount) VALUES (1, 5000);  -- Undo_A
-- Undo_A: "id=? 삽입됨, 롤백 시 이 PK를 DELETE"

SAVEPOINT sp1;
-- sp1 이후의 변경만 롤백 가능하게 체크포인트 생성

UPDATE accounts SET balance = balance - 5000 WHERE id = 1;  -- Undo_B
UPDATE inventory SET stock = stock - 1 WHERE product_id = 1; -- Undo_C

-- 재고 확인 후 문제 발견
SELECT stock FROM inventory WHERE product_id = 1;  -- -1이면 문제!

ROLLBACK TO SAVEPOINT sp1;
-- Undo_C 역방향 적용 → inventory 복원
-- Undo_B 역방향 적용 → accounts 복원
-- Undo_A는 그대로 (sp1 이전이므로 유지)

-- ORDER는 여전히 INSERT된 상태
SELECT * FROM orders WHERE user_id = 1;  -- 방금 INSERT한 order가 있음

COMMIT;
-- 또는 ROLLBACK; (모두 취소)
```

---

## 💻 실전 실험

### 실험 1: 롤백 비용 — 변경 건수에 비례

```sql
-- 실험: 변경 건수에 따른 롤백 시간 비교

-- 소량 변경 롤백
START TRANSACTION;
UPDATE orders SET memo = 'test' WHERE id <= 100;  -- 100건
SET @start = NOW(6);
ROLLBACK;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS rollback_100_ms;

-- 대량 변경 롤백
START TRANSACTION;
UPDATE orders SET memo = 'test' WHERE id <= 100000;  -- 10만 건
SET @start = NOW(6);
ROLLBACK;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS rollback_100k_ms;

-- 예상: 10만 건 롤백이 100건의 약 1000배 느림
-- 이유: Undo Log 레코드가 100,000개 → 역방향 적용 100,000번
```

### 실험 2: History List Length — 장기 트랜잭션 영향

```sql
-- 초기 상태 확인
SHOW ENGINE INNODB STATUS\G
-- "History list length: N" 초기값 기록

-- 세션 1: 장기 트랜잭션 시뮬레이션 (Read View 고정)
START TRANSACTION;
SELECT COUNT(*) FROM orders;  -- Read View 생성

-- 세션 2: 많은 UPDATE 실행
UPDATE orders SET memo = CONCAT('v', id)
WHERE id BETWEEN 1 AND 50000;  -- 5만 건
COMMIT;

-- History List Length 확인
SHOW ENGINE INNODB STATUS\G
-- "History list length: 증가" → 세션 1 때문에 Purge 불가

-- 세션 1 커밋
COMMIT;  -- Read View 해제

-- Purge Thread 작동 후 Length 감소 확인 (수초 후)
SHOW ENGINE INNODB STATUS\G
```

### 실험 3: Undo Tablespace 크기 모니터링

```sql
-- Undo Tablespace 크기 확인
SELECT
    space,
    name,
    ROUND(file_size / 1024 / 1024, 2) AS size_mb,
    ROUND(allocated_size / 1024 / 1024, 2) AS allocated_mb
FROM information_schema.INNODB_TABLESPACES
WHERE name LIKE 'innodb_undo%' OR name LIKE 'undo%'
ORDER BY file_size DESC;

-- Purge 설정 확인
SHOW VARIABLES LIKE 'innodb_undo%';
-- innodb_undo_log_truncate: ON/OFF
-- innodb_max_undo_log_size: 팽창 임계값

-- 장기 트랜잭션 감지
SELECT
    trx_id,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS seconds_running,
    trx_rows_modified,
    trx_state,
    LEFT(trx_query, 100) AS current_query
FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 30
ORDER BY trx_started;
-- 30초 이상 실행 중인 트랜잭션 → 즉시 조사 필요
```

### 실험 4: SAVEPOINT 부분 롤백

```sql
CREATE TABLE log_actions (
    id     BIGINT AUTO_INCREMENT PRIMARY KEY,
    action VARCHAR(100),
    ts     DATETIME DEFAULT NOW()
) ENGINE=InnoDB;

START TRANSACTION;

INSERT INTO orders (user_id, amount, status) VALUES (1, 5000, 'PENDING');
INSERT INTO log_actions (action) VALUES ('ORDER_CREATED');

SAVEPOINT after_order;

UPDATE accounts SET balance = balance - 5000 WHERE id = 1;
INSERT INTO log_actions (action) VALUES ('BALANCE_DECREASED');

-- 잔액이 충분한지 확인
SELECT balance FROM accounts WHERE id = 1;
-- 만약 음수라면:
ROLLBACK TO SAVEPOINT after_order;
-- → UPDATE accounts, 두 번째 log_actions INSERT만 롤백
-- → order INSERT, 첫 번째 log_actions INSERT는 유지

SELECT * FROM orders ORDER BY id DESC LIMIT 1;  -- order 남아있음
SELECT * FROM log_actions ORDER BY id DESC LIMIT 3;  -- 첫 log만 남아있음

ROLLBACK;  -- 또는 COMMIT;
```

---

## 📊 성능 비교

```
Undo Log 관련 성능 영향:

롤백 비용 vs 변경 건수:
  100건:   ~1ms
  1만건:   ~100ms
  100만건: ~10초
  → 대형 트랜잭션 롤백은 서비스 중단 수준의 비용 발생

버전 체인 탐색 vs SELECT 성능:
  체인 길이 1 (최신 버전 바로 보임): 0 I/O 추가
  체인 길이 10: ~10번의 Undo Log 랜덤 읽기
  체인 길이 1000 (장기 트랜잭션 존재 시): ~1000번 Undo Log 읽기
  → 장기 트랜잭션 1개가 다른 모든 세션의 SELECT를 느리게 만듦

Insert Undo vs Update Undo 처리 속도:
  Insert Undo: COMMIT 직후 삭제 가능 → Purge 부하 낮음
  Update Undo: 모든 Read View가 완료될 때까지 보관 → Purge 부하 높음
  → UPDATE가 많은 시스템에서 Purge Thread 부하 모니터링 필요
```

---

## ⚖️ 트레이드오프

```
Undo Log 관련 설계 결정:

배치 처리: 한 트랜잭션 vs 소량 커밋

  한 트랜잭션으로 100만 건:
    ✅ 완전한 원자성 (전체 성공 or 전체 실패)
    ❌ 100만 건의 Undo Log 축적
    ❌ 실패 시 롤백 비용 ~10초
    ❌ 그동안 모든 UPDATE Undo Log 삭제 불가 (History Length 급증)
    ❌ 다른 세션의 SELECT 성능 저하

  1000건씩 나눠 커밋:
    ✅ Undo Log 즉시 정리 (작은 트랜잭션)
    ✅ 실패 시 롤백 비용 1/1000
    ❌ 완전한 원자성 없음 (500번째 커밋 후 실패 → 이미 500 × 1000건 반영)
    → 멱등성(idempotency) 보장 필요 또는 별도 보상 트랜잭션 설계

innodb_undo_tablespaces:
  2 (기본): 최소 구성
  더 많으면: Purge 병렬화 → 고쓰기 환경에서 유리
  너무 많으면: 파일 관리 복잡성 증가

innodb_purge_threads:
  4 (기본): 대부분 환경 충분
  더 높이면: Purge 처리량 증가 (History Length 관리)
  주의: 너무 높으면 오히려 경합 발생 (보통 4 이하 권장)
```

---

## 📌 핵심 정리

```
Undo Log 핵심:

이중 역할:
  ① Atomicity: 롤백 시 변경을 역방향으로 취소
  ② MVCC: 이전 버전을 제공하여 스냅샷 읽기 지원

종류별 생명주기:
  Insert Undo: COMMIT 직후 삭제 가능
  Update/Delete Undo: 모든 활성 Read View가 불필요해질 때 삭제 가능

Purge Thread:
  가장 오래된 Read View보다 오래된 Undo Log만 삭제
  History List Length = Purge 대기 중인 레코드 수
  높은 History Length → 버전 체인 탐색 증가 → SELECT 성능 저하

장기 트랜잭션 = 시스템 적
  Undo Log 삭제 차단 → Tablespace 팽창
  다른 세션의 SELECT 성능 저하 (버전 체인 길어짐)
  진단: SHOW ENGINE INNODB STATUS의 History list length

대량 배치 처리 권장:
  소량(1000~10000건) 단위로 커밋
  멱등성 보장으로 중간 실패 대응
  분석 쿼리는 Read Replica로 분리
```

---

## 🤔 생각해볼 문제

**Q1.** 서비스가 운영 중에 갑자기 `SELECT COUNT(*) FROM orders` 가 3배 느려졌다. EXPLAIN에는 아무 변화가 없고 인덱스도 동일하다. Undo Log와 관련하여 어떤 원인을 의심해야 하는가?

<details>
<summary>해설 보기</summary>

**버전 체인 탐색 비용 증가**가 원인일 가능성이 높습니다.

진단 순서:
1. `SHOW ENGINE INNODB STATUS\G`에서 "History list length" 확인
   - 수만 이상이면 Purge가 따라가지 못하는 상태
2. `SELECT * FROM information_schema.INNODB_TRX ORDER BY trx_started;`로 장기 트랜잭션 확인
   - 수분 이상 실행 중인 트랜잭션이 있는지

발생 메커니즘:
- 장기 트랜잭션이 Read View를 오래 고정
- 그동안 orders 테이블의 UPDATE가 쌓임
- 각 orders Row의 버전 체인이 길어짐
- `SELECT COUNT(*)`가 각 Row를 읽을 때 긴 버전 체인을 탐색
- → SELECT 성능 저하

해결:
- 장기 트랜잭션 종료 (KILL 또는 COMMIT 유도)
- Purge Thread가 정리한 후 자연스럽게 성능 회복
- 근본 원인: `@Transactional` 내 외부 IO 제거, 배치 분리

</details>

---

**Q2.** DELETE 10만 건과 UPDATE 10만 건 중 어느 쪽이 Undo Log를 더 많이 차지하는가? 그 이유는?

<details>
<summary>해설 보기</summary>

**DELETE가 더 많이 차지합니다.**

- **DELETE Undo**: 삭제된 Row의 **전체 데이터**를 기록 (언제든 완전히 복원할 수 있도록)
- **UPDATE Undo**: **변경된 컬럼의 이전 값만** 기록 (변경되지 않은 컬럼은 제외)

예시:
- Row 크기 1KB인 테이블, 20바이트짜리 컬럼 1개만 UPDATE
  - UPDATE Undo: ~20 bytes × 10만 = 2MB
  - DELETE Undo: ~1KB × 10만 = 100MB (50배 차이)

추가 차이:
- DELETE는 "Delete Mark"만 표시 후 Purge Thread가 나중에 물리적으로 제거
- UPDATE는 새 버전을 현재 위치에 기록하고 이전 버전을 Undo Log에 보관
- 인덱스 측면: DELETE는 모든 인덱스에서 해당 항목 Purge 필요 (UPDATE보다 많은 인덱스 갱신)

실무 교훈: 대량 DELETE는 Undo Log와 Purge 부하 모두에서 UPDATE보다 비쌉니다. 가능하면 소량씩 나눠서 실행하고, Undo Tablespace 여유 공간을 확인한 후 실행하는 것이 안전합니다.

</details>

---

**Q3.** 다음 코드에서 Undo Log 관점의 문제점과 개선 방법을 설명하라.

```java
@Transactional
public void monthlyDataCleanup() {
    List<Long> oldOrderIds = orderRepository.findIdsOlderThan(90); // 100만 건
    for (Long id : oldOrderIds) {
        orderRepository.deleteById(id);  // 건건이 DELETE
    }
}
```

<details>
<summary>해설 보기</summary>

**문제점**:
1. **100만 건을 한 트랜잭션에서 처리** → 100만 건의 Delete Undo 기록 (각 Row 전체 크기)
2. 트랜잭션 완료 전까지 100만 건의 Undo Log가 모두 삭제 불가
3. 실패 시 롤백 비용 = 100만 건 DELETE 비용 (수분)
4. 트랜잭션이 긴 시간 열려있어 다른 세션의 SELECT 성능 저하

**개선 방법**:
```java
// @Transactional 없음 (각 배치가 독립 트랜잭션)
public void monthlyDataCleanup() {
    int page = 0;
    int batchSize = 1000;
    
    while (true) {
        // @Transactional이 붙은 별도 메서드 호출 (Spring 프록시 동작을 위해)
        int deleted = cleanupBatch(batchSize);
        if (deleted == 0) break;
        
        // 배치 사이 짧은 휴식 (다른 쿼리 기회 제공)
        Thread.sleep(10);
    }
}

@Transactional  // 1000건씩 독립 트랜잭션
public int cleanupBatch(int size) {
    return orderRepository.deleteOlderThan90DaysLimit(size);
    // DELETE FROM orders WHERE created_at < ? LIMIT 1000
}
```

장점:
- 각 1000건 커밋 후 Undo Log 즉시 Purge 가능
- 실패 시 최대 1000건만 롤백 (Undo Log 부담 1/1000)
- 다른 서비스 요청 처리 기회 제공
- 전체 작업 시간은 비슷하지만 시스템 영향 대폭 감소

단점: 완전한 원자성 없음 → 멱등성 보장 필요 (재실행해도 같은 결과)

</details>

---

<div align="center">

**[⬅️ MVCC](./02-mvcc-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: 격리 수준 ➡️](./04-isolation-levels.md)**

</div>
