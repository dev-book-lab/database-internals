# 데드락 발생 원리와 감지 알고리즘

---

## 🎯 핵심 질문

- 데드락이 발생하기 위한 4가지 필요 조건(Coffman 조건)은?
- InnoDB의 Wait-for Graph 기반 데드락 감지는 어떻게 동작하는가?
- 롤백될 트랜잭션은 어떤 기준으로 선택되는가?
- `innodb_deadlock_detect = OFF`가 성능을 향상시키는 이유와 위험은?
- Lock Wait Timeout과 데드락 감지의 차이는?

---

## 🔬 내부 동작 원리

### 1. 데드락 발생 조건

```
Coffman 4가지 조건 (모두 동시 충족 시 데드락):

1. Mutual Exclusion (상호 배제):
   X Lock은 한 번에 하나의 트랜잭션만 보유 가능

2. Hold and Wait (보유하며 대기):
   Lock을 보유하면서 다른 Lock을 기다림

3. No Preemption (선점 불가):
   Lock을 강제로 빼앗을 수 없음 (ROLLBACK으로 해제만 가능)

4. Circular Wait (순환 대기):
   TRX A → TRX B 기다림, TRX B → TRX A 기다림

가장 흔한 패턴:
  TRX A: Lock(Row1) 보유 → Lock(Row2) 대기
  TRX B: Lock(Row2) 보유 → Lock(Row1) 대기
  → 순환 → 데드락

두 번째로 흔한 패턴 (Lock 업그레이드):
  TRX A: S Lock(Row1) 보유 → X Lock(Row1) 업그레이드 대기
  TRX B: S Lock(Row1) 보유 → X Lock(Row1) 업그레이드 대기
  → 둘 다 상대방의 S Lock 때문에 X Lock 획득 불가
  → 데드락
```

### 2. Wait-for Graph 알고리즘

```
InnoDB의 데드락 감지:

Wait-for Graph:
  노드: 각 트랜잭션
  간선: TRX A → TRX B (A가 B를 기다림)
  순환(Cycle) 존재 = 데드락!

실시간 사이클 감지:
  새로운 Lock 대기가 발생할 때마다 그래프 업데이트
  DFS(깊이 우선 탐색)로 사이클 확인
  사이클 발견 시 즉시 하나를 ROLLBACK

예시:
  TRX 100 → (Row1 X Lock 대기) → TRX 200 보유 중
  TRX 200 → (Row2 X Lock 대기) → TRX 100 보유 중
  
  Wait-for Graph:
    100 → 200 → 100  ← Cycle!
  → 데드락 감지 → 하나 롤백

탐지 시점:
  Lock 대기 큐에 새 항목이 추가될 때마다 검사
  → 데드락 발생 즉시 감지 (몇 밀리초 내)
  → innodb_deadlock_detect = ON (기본값)
```

### 3. 롤백 대상 선택 (Victim Selection)

```
ROLLBACK될 트랜잭션 선택 기준:
  기본: "Undo Log가 적은 트랜잭션" = 롤백 비용이 가장 낮은 것
  
  비용 = 롤백해야 할 변경 건수 (trx_weight)
       = 변경한 Row 수 + 보유한 Lock 수
  
  최저 비용 트랜잭션 → ROLLBACK (Victim)
  나머지 트랜잭션 → 계속 실행

에러:
  Victim 트랜잭션에게:
    ERROR 1213 (40001): Deadlock found when trying to get lock;
    try restarting transaction
  
  애플리케이션 처리:
    이 에러를 catch하여 트랜잭션 재시도 로직 필요

Spring에서 데드락 처리:
  @Retryable(value = DeadlockLoserDataAccessException.class, maxAttempts = 3)
  @Transactional
  public void processOrder() {
    // 데드락 발생 시 자동 재시도 (최대 3회)
  }
```

### 4. innodb_deadlock_detect

```
innodb_deadlock_detect = ON (기본):
  모든 Lock 대기 시마다 Wait-for Graph 탐색
  사이클 감지 → 즉시 롤백
  
  비용:
    다수의 트랜잭션이 동시에 대기 중이면
    매번 전체 그래프를 DFS 탐색 → CPU 부하
    고동시성 환경(수천 TPS)에서 latch 경합 발생 가능

innodb_deadlock_detect = OFF:
  사이클 탐색 없음 → CPU 절약
  대신: innodb_lock_wait_timeout(기본 50초)으로 타임아웃 처리
  → 데드락 발생 시 50초 후 Timeout에러로 자동 롤백
  
  적합한 환경:
    데드락이 매우 드문 환경
    고동시성(수만 TPS)에서 탐지 오버헤드가 문제일 때
  
  위험:
    데드락 발생 시 50초 동안 두 트랜잭션 모두 차단
    서비스 지연 급증

innodb_lock_wait_timeout (기본: 50초):
  데드락 여부와 무관하게 Lock 대기가 이 시간 초과 시 롤백
  프로덕션 권장: 5~10초
  (50초는 너무 길어 장애 시 대기 큐 폭증)
  
  설정:
  SET GLOBAL innodb_lock_wait_timeout = 10;
```

---

## 💻 실전 실험

### 실험: 의도적 데드락 발생

```sql
-- 세션 1
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- id=1에 X Lock 획득

-- 세션 2 (별도 창)
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;
-- id=2에 X Lock 획득

-- 세션 1 (계속)
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- id=2 X Lock 요청 → 세션 2가 보유 중 → 대기

-- 세션 2 (계속)
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
-- id=1 X Lock 요청 → 세션 1이 보유 중 → 데드락!
-- 즉시: ERROR 1213 Deadlock found
-- 비용 낮은 쪽이 롤백됨

-- 데드락 로그 확인
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 섹션 확인
```

---

## 📌 핵심 정리

```
데드락 감지 핵심:

발생 조건:
  두 트랜잭션이 서로 상대방의 Lock을 기다리는 순환 구조

Wait-for Graph:
  트랜잭션을 노드, 대기를 간선으로 표현
  DFS로 사이클 감지 = 데드락
  감지 즉시 최저 비용 트랜잭션 ROLLBACK

에러: ERROR 1213 → 재시도 로직 필요
Spring: @Retryable(DeadlockLoserDataAccessException.class)

innodb_deadlock_detect:
  ON (기본): 즉시 감지, 고동시성에서 CPU 부하
  OFF: 타임아웃(50초)으로 처리, 데드락 드문 환경에서만

innodb_lock_wait_timeout:
  기본 50초 → 프로덕션에서 5~10초 권장
```

---

## 🤔 생각해볼 문제

**Q1.** 세 트랜잭션 A → B → C → A의 순환 대기가 발생했다. 이 경우 InnoDB는 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

Wait-for Graph에서 A → B → C → A 사이클이 형성됩니다. InnoDB의 DFS 탐색이 이 사이클을 감지하면, **세 트랜잭션 중 가장 Undo Log가 적은(변경이 가장 적은) 트랜잭션 하나**를 선택하여 ROLLBACK합니다.

예를 들어 C가 가장 변경이 적다면 C가 롤백됩니다. 그러면 B는 C의 Lock을 기다리던 상태에서 해방되어 진행하고, A도 마찬가지입니다.

실제로 세 트랜잭션 데드락은 두 트랜잭션 데드락보다 덜 빈번하지만, 더 복잡한 잠금 순서에서 발생할 수 있습니다. 근본 해결책은 항상 동일합니다: **모든 트랜잭션에서 같은 순서로 Lock을 획득** (예: 항상 낮은 ID → 높은 ID 순서).

</details>

---

<div align="center">

**[⬅️ Gap Lock](./03-gap-lock-next-key-lock.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데드락 로그 분석 ➡️](./05-deadlock-log-analysis.md)**

</div>
