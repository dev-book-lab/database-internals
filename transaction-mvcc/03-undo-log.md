# Undo Log — 이전 버전을 보관하는 방법

---

## 🎯 핵심 질문

- Undo Log와 버전 체인은 물리적으로 어디에 저장되는가?
- 롤백 시 Undo Log를 "역방향으로 적용"한다는 것은 구체적으로 어떤 의미인가?
- Purge Thread는 Undo Log를 언제 삭제하는가?
- 오래된 Read View가 Undo Tablespace 팽창을 일으키는 메커니즘은?
- `innodb_undo_log_truncate`는 무엇을 하는가?

---

## 🔬 내부 동작 원리

### 1. Undo Log 저장 구조

```
Undo Log 위치 (MySQL 8.0):
  별도의 Undo Tablespace 파일 (undo_001, undo_002, ...)
  기본 경로: datadir/undo_001, datadir/undo_002
  innodb_undo_tablespaces: Undo Tablespace 파일 수 (기본 2)

Undo Log 세그먼트 구조:
  Rollback Segment
    └── Undo Log Segment
          └── Undo Log Records (각 변경 기록)

Rollback Segment:
  동시 트랜잭션의 Undo Log를 분리하는 단위
  MySQL 8.0: 최대 128개 Rollback Segment
  → 동시 트랜잭션 수 증가에 따른 경합 감소

Undo Record 종류:
  Insert Undo:  INSERT된 Row의 PK만 기록
                롤백 시: 해당 PK의 Row 삭제
  Update Undo:  변경 전 컬럼 값 기록 (변경된 컬럼만)
                롤백 시: 이전 값으로 복원
                MVCC: 이전 버전 읽기에 재사용
  Delete Undo:  삭제 전 전체 Row 기록
                롤백 시: Row 재삽입

Insert Undo vs Update Undo 차이:
  Insert Undo:
    COMMIT 직후 삭제 가능 (다른 트랜잭션이 참조 불필요)
    이유: INSERT 이전 버전 = Row 없음 → Read View에서 볼 이유 없음
  
  Update/Delete Undo:
    모든 활성 Read View가 불필요해질 때까지 보관
    이유: 이전 버전을 보고 있는 트랜잭션이 있을 수 있음
```

### 2. 버전 체인 탐색 과정

```
SELECT amount FROM accounts WHERE id = 1 (Read View: TRX 150이 활성)

현재 Row:
  [id=1, amount=6000 | TRX_ID=300 | ROLL_PTR=→Undo_B]

탐색 과정:
  Step 1: 현재 Row 확인
    TRX_ID=300, Read View에서 활성 TRX = [150]
    300 > 150 (m_up_limit_id) → 더 확인 필요
    300 not in m_ids([150]) → 300은 이미 커밋됨
    300 < m_low_limit_id (Read View 생성 시 다음 TRX_ID=400) → 보임
    → 현재 Row (amount=6000) 반환!

만약 TRX_ID=300이 아직 활성이었다면:
  300 in m_ids → 안 보임 → ROLL_PTR 따라 이동
  
  Step 2: Undo Log B 읽기
    [id=1, amount=8000 | TRX_ID=200]
    200 < m_up_limit_id(150)? NO
    200 not in m_ids([150, 300]) → 커밋됨
    200 < m_low_limit_id → 보임
    → amount=8000 반환

  만약 200도 활성이었다면 계속 체인 따라 탐색...
  체인 끝(ROLL_PTR=NULL)에 도달해도 보이는 버전 없으면: Row가 존재하지 않음으로 처리

성능 영향:
  버전 체인이 길수록 SELECT가 느림
  오래된 트랜잭션이 있으면 버전 체인 탐색 비용 증가
  → 장시간 트랜잭션은 성능에 악영향
```

### 3. 롤백 과정 상세

```
UPDATE accounts SET amount = 6000 WHERE id = 1; (TRX=300)
현재 Row: [amount=6000, TRX_ID=300, ROLL_PTR=→Undo]
Undo Log: [이전 값: amount=10000, TRX_ID=이전값=100]

ROLLBACK 실행:
  1. Undo Log 읽기: 이전 amount=10000
  2. 현재 Row를 이전 값으로 업데이트: amount=10000
  3. TRX_ID를 이전 TRX_ID로 복원 (또는 적절한 값으로)
  4. Undo Log 마킹 (롤백 완료, 이후 Purge 가능)

여러 변경의 롤백 (역방향):
  TRX 300이 3개 Row를 순서대로 변경: A → B → C
  Undo Log: C_undo → B_undo → A_undo (생성 역순)
  
  ROLLBACK 시: C_undo 적용 → B_undo 적용 → A_undo 적용
  → 항상 역방향 (생성 역순) 적용
  → 최종적으로 TRX 300 이전 상태로 완전 복원

세이브포인트(SAVEPOINT):
  SAVEPOINT sp1;
  -- 여러 변경
  ROLLBACK TO SAVEPOINT sp1;
  → sp1 이후의 Undo Log만 역방향 적용
  → sp1 이전 변경은 유지
```

### 4. Purge Thread — Undo Log 삭제

```
Purge Thread의 역할:
  COMMIT된 트랜잭션의 Undo Log를 물리적으로 삭제
  삭제 조건: 어떤 활성 Read View도 해당 Undo Log 버전을 참조하지 않을 때

삭제 가능 여부 판단:
  가장 오래된 Read View의 m_up_limit_id = X
  TRX_ID < X인 Undo Log는 삭제 가능
  (X보다 오래된 트랜잭션은 모두 커밋됐고 더 이상 볼 필요 없음)

Undo Tablespace 팽창 시나리오:
  T1: 분석 쿼리 시작 (Read View: TRX_ID < 1000 커밋됨)
  T2~T100000: 매초 수백 건의 UPDATE 발생
  → 각 UPDATE의 Undo Log가 TRX_ID ≥ 1000
  → 가장 오래된 Read View = TRX 1000 기준
  → TRX 1000 이후의 모든 Undo Log 삭제 불가
  → 수십만 건의 Undo Log 누적 → Tablespace 수십 GB 팽창!

Undo Tablespace 모니터링:
  SELECT name, subsystem, count, comment
  FROM information_schema.INNODB_METRICS
  WHERE name LIKE '%undo%';
  
  또는 파일 크기 직접 확인:
  ls -lh /var/lib/mysql/undo_001 /var/lib/mysql/undo_002

innodb_undo_log_truncate (기본: ON in MySQL 8.0):
  Undo Tablespace가 innodb_max_undo_log_size 초과 시
  → 오래된 Undo Log를 물리적으로 truncate (파일 크기 감소)
  → 자동으로 팽창 방지
  단, truncate 중 약간의 성능 영향

문제 진단:
  SHOW ENGINE INNODB STATUS\G
  → "History list length: N" 확인
  → 수만 이상이면 Purge가 따라가지 못하는 신호
  → 장기 실행 트랜잭션 확인:
  SELECT trx_id, trx_started, trx_state, LEFT(trx_query, 50)
  FROM information_schema.INNODB_TRX
  ORDER BY trx_started LIMIT 10;
```

---

## 💻 실전 실험

### 실험 1: History List Length 모니터링

```sql
-- Undo Log 누적 상태 확인
SHOW ENGINE INNODB STATUS\G
-- "TRANSACTIONS" 섹션에서:
-- "History list length: N" → Purge 대기 중인 Undo Log 레코드 수

-- 정상: N < 1000
-- 주의: N > 10000
-- 위험: N > 100000 (장기 트랜잭션 또는 Purge 지연)

-- 장기 실행 트랜잭션 찾기
SELECT
    trx_id,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS seconds_running,
    trx_state,
    trx_rows_modified,
    LEFT(trx_query, 100) AS query_preview
FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 10
ORDER BY trx_started;
```

### 실험 2: 롤백 성능 측정

```sql
-- 대량 변경 후 롤백 비용 측정
START TRANSACTION;
-- 10만 건 UPDATE
UPDATE orders SET status = 'TEMP' WHERE status = 'PAID' LIMIT 100000;
-- Undo Log에 10만 건의 이전 상태 기록됨

SET @start = NOW(6);
ROLLBACK;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6))/1000 AS rollback_ms;
-- 롤백 시간 ∝ Undo Log 레코드 수
-- → 큰 트랜잭션의 롤백은 오래 걸림
```

---

## 📌 핵심 정리

```
Undo Log 핵심:

저장 위치: Undo Tablespace (undo_001, undo_002, ...)

종류:
  Insert Undo: PK만 기록, COMMIT 후 즉시 Purge 가능
  Update/Delete Undo: 이전 값 기록, MVCC에 재사용
                       모든 Read View가 불필요해질 때까지 보관

버전 체인:
  DB_ROLL_PTR로 현재 버전 → 이전 버전 → 그 이전 버전
  SELECT 시 Read View 기준으로 보이는 버전 탐색

롤백: Undo Log를 역방향으로 적용 (생성 역순)

Purge Thread:
  가장 오래된 Read View보다 오래된 Undo Log 삭제
  장기 트랜잭션 → Purge 차단 → Tablespace 팽창

진단:
  History list length (INNODB STATUS)
  information_schema.INNODB_TRX에서 장기 트랜잭션 확인
```

---

## 🤔 생각해볼 문제

**Q1.** `DELETE FROM orders WHERE status = 'OLD'` (100만 건) 롤백이 같은 건수의 `INSERT` 롤백보다 훨씬 느린 이유는?

<details>
<summary>해설 보기</summary>

DELETE의 Undo Log에는 삭제된 전체 Row 데이터가 저장됩니다. 100만 건 DELETE → 100만 건의 전체 Row를 Undo Log에 기록. 롤백 시 이 100만 건을 다시 테이블에 INSERT하는 과정이 필요합니다.

INSERT의 Undo Log에는 PK만 저장됩니다. 롤백 시 해당 PK의 Row를 DELETE하면 됩니다.

따라서 DELETE 롤백은:
1. 100만 건의 전체 Row 데이터를 Undo Log에서 읽음 (I/O)
2. 각 Row를 테이블에 재삽입 (B+Tree 삽입 + 인덱스 갱신)
3. 인덱스가 N개면 N × 100만 번의 인덱스 삽입

결론: 대량 DELETE + ROLLBACK은 매우 비용이 높습니다. 실수로 대량 DELETE를 실행했다면 롤백이 오래 걸리고 그동안 시스템에 부하가 집중됩니다. 배치 DELETE는 `LIMIT`으로 소량씩 나눠서 실행하는 것이 안전합니다.

</details>

---

<div align="center">

**[⬅️ MVCC](./02-mvcc-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: 격리 수준 ➡️](./04-isolation-levels.md)**

</div>
