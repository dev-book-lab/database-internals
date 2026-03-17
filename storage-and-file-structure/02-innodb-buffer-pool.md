# InnoDB Buffer Pool — 메모리와 디스크 사이 캐시 레이어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Buffer Pool이 LRU를 단순 적용하지 않고 Young/Old 서브리스트로 나눈 이유는 무엇인가?
- Full Table Scan이 Buffer Pool을 오염시키는 문제를 InnoDB가 어떻게 해결했는가?
- Dirty Page가 디스크에 flush되는 시점과 조건은 무엇인가?
- `innodb_buffer_pool_size`를 얼마로 설정해야 하는가?
- Buffer Pool Warming이란 무엇이고, 재시작 후 성능이 회복되는 데 왜 시간이 걸리는가?
- JPA의 쓰기 지연(Write-Behind)과 InnoDB Buffer Pool의 Dirty Page 관리는 어떤 관계인가?

---

## 🔍 왜 이 개념이 중요한가

### DB 성능의 첫 번째 변수 — "이 데이터가 메모리에 있는가?"

```
InnoDB 처리 흐름:
  
  쿼리 실행 → Buffer Pool에 Page 있음? ─Yes→ 메모리에서 즉시 반환 (나노초)
                          │ No
                          ▼
                  디스크에서 Page 읽기 (밀리초)
                          │
                          ▼
                  Buffer Pool에 Page 저장
                          │
                          ▼
                  메모리에서 데이터 반환

Buffer Pool Hit Rate의 중요성:
  Hit Rate 99%: 100개 요청 중 1개만 디스크 I/O
  Hit Rate 90%: 100개 요청 중 10개가 디스크 I/O
  → 10% 차이가 응답 시간을 수십 배 차이나게 만들 수 있음

실무에서 자주 나타나는 패턴:
  새벽 배치 작업이 Full Table Scan으로 Buffer Pool을 오염
  → 아침 출근 시간에 Cache Miss 폭발 → 응답 지연
  
  서버 재시작 후 수 분간 느린 응답
  → Buffer Pool이 비어있어 모든 요청이 Disk I/O
  → 이것이 Buffer Pool Warming이 필요한 이유
```

---

## 😱 잘못된 이해

### Before: Buffer Pool을 단순한 캐시로만 이해하는 접근

```
잘못된 이해 ①: "Buffer Pool이 크면 무조건 좋다"
  서버 메모리 64GB → innodb_buffer_pool_size = 60GB 설정
  
  문제:
    OS 파일 시스템 캐시, 연결당 스레드 스택, 정렬 버퍼 등 필요
    → 메모리 부족으로 Swap 발생
    → Swap I/O가 Disk I/O보다 훨씬 느릴 수 있음
  
  권장: 물리 메모리의 70~80% 이하로 설정

잘못된 이해 ②: "Buffer Pool은 읽기 캐시다"
  실제로는 읽기 + 쓰기 모두 Buffer Pool 경유
  
  UPDATE orders SET status = 'PAID' WHERE id = 1:
    ① Buffer Pool에서 해당 Page 찾기 (없으면 디스크에서 로드)
    ② Buffer Pool의 Page를 수정 → Dirty Page 생성
    ③ Redo Log에 변경 기록 (WAL)
    ④ Buffer Pool → 디스크 flush는 나중에 비동기로 처리
  
  쓰기는 Buffer Pool을 통해 지연(Buffered) 처리됨
  → JPA의 쓰기 지연(1차 캐시)과 구조적으로 유사

잘못된 이해 ③: "최근에 사용한 Page만 캐시된다"
  단순 LRU라면:
    Full Table Scan → 수천 개의 새 Page 로드
    → 실제로 자주 쓰이는 Page들이 밀려남
    → Hit Rate 폭락
  
  InnoDB는 이 문제를 알고 있어서 단순 LRU를 쓰지 않음
```

---

## ✨ 올바른 이해

### After: Midpoint Insertion Strategy (Young/Old LRU)

```
InnoDB Buffer Pool LRU 구조:

  ┌────────────────────────────────────────┐
  │              Buffer Pool               │
  │                                        │
  │  ┌──────────────────┐ ┌─────────────┐  │
  │  │   Young 서브리스트  │ │ Old 서브리스트 │  │
  │  │   (기본 5/8 = 63%)│ │  (기본 3/8)  │  │
  │  │                  │ │             │  │
  │  │  자주 접근되는      │ │  새로 로드된   │  │
  │  │  "Hot" Page      │ │  Page 진입점  │  │
  │  │                  │ │             │  │
  │  │  Head ← 최근      │ │  Head ← 진입 │  │
  │  │          접근     │ │  Tail → 교체 │  │
  │  └──────────────────┘ └─────────────┘  │
  └────────────────────────────────────────┘
         Old 서브리스트 Head =
         "Midpoint" (전체 LRU의 중간)

새 Page 로드 시:
  Old 서브리스트 Head(Midpoint)에 삽입
  → Full Table Scan이 수천 Page를 로드해도
    Young 서브리스트의 Hot Page는 영향 없음

Young으로 승급 조건:
  Old 서브리스트에 들어온 지 innodb_old_blocks_time(기본 1000ms = 1초)
  경과 후 다시 접근되면 → Young 서브리스트 Head로 이동
  
  1초 이내 재접근은 Young으로 이동 안 함:
    → Full Table Scan이 같은 Page를 짧은 시간에 여러 번 접근해도
      Young으로 승급되지 않음 (Hot Page 보호)
```

---

## 🔬 내부 동작 원리

### 1. Buffer Pool 내부 구조

```
Buffer Pool의 실제 구성 요소:

① Page 프레임 (실제 데이터 영역):
   각 16KB Page를 담는 메모리 블록
   innodb_buffer_pool_size로 설정한 크기가 대부분 여기에 사용

② Buffer Pool 인스턴스:
   멀티 인스턴스 지원 (innodb_buffer_pool_instances, 기본 8)
   
   이유: 단일 인스턴스 → Buffer Pool 접근 시 Mutex 경합
          8개 인스턴스 → 각 인스턴스별 Mutex → 병렬 접근 가능
   
   Page는 space_id + page_no를 해시해서 인스턴스 분배
   → 같은 Page는 항상 같은 인스턴스에서 관리

③ LRU List:
   Young + Old 서브리스트 (앞에서 설명한 Midpoint 구조)

④ Free List:
   아직 사용되지 않은 빈 Page 프레임 목록
   Buffer Pool 초기화 시 모든 프레임이 Free List에 있음
   새 Page 로드 시: Free List에서 프레임 할당
   Free List 고갈 시: LRU Tail에서 퇴출(evict)

⑤ Flush List:
   Dirty Page(수정되었지만 디스크에 아직 안 쓴 Page) 목록
   LSN 순서로 정렬
   → Checkpoint 시 이 목록을 기준으로 flush

⑥ Hash Table (Page Hash):
   space_id + page_no → Buffer Pool 내 위치 매핑
   Page 탐색: O(1)
```

### 2. Dirty Page Flush 메커니즘

```
Dirty Page가 디스크에 쓰이는 4가지 경로:

① Checkpoint (주기적 Flush):
   Redo Log가 일정 크기 이상 채워지면 자동 트리거
   
   이유: Crash Recovery 시간 단축
     - Dirty Page가 많을수록 Redo Log를 더 많이 재생해야 함
     - Checkpoint = "여기까지는 디스크에 확실히 반영됨" 표시
   
   innodb_log_file_size가 작으면:
     Checkpoint가 자주 발생 → 쓰기 I/O 증가
     But Crash Recovery 시간 단축
   
   innodb_log_file_size가 크면:
     Checkpoint 빈도 감소 → 쓰기 I/O 감소
     But Crash Recovery 시간 증가

② LRU Flush (공간 확보):
   Free List가 고갈될 때 LRU Tail의 Dirty Page를 flush
   → Page Cleaner Thread가 백그라운드로 수행

③ Adaptive Flush (InnoDB 자동 조절):
   Redo Log 사용률이 임계값에 가까워지면 자동으로 flush 속도 조절
   innodb_adaptive_flushing = ON (기본)

④ COMMIT 시 flush (innodb_flush_log_at_trx_commit 설정에 따라):
   Redo Log는 COMMIT 시 즉시 flush (기본값 = 1)
   But Buffer Pool의 Dirty Page는 바로 flush하지 않음
   → WAL(Write-Ahead Logging) 원칙: Redo Log만 flush하면 내구성 보장

Page Cleaner Thread:
  백그라운드에서 주기적으로 Dirty Page를 디스크에 쓰는 전담 스레드
  innodb_page_cleaners: Page Cleaner Thread 수 (기본 4)
  → innodb_buffer_pool_instances 수와 맞추는 것이 권장
```

### 3. Buffer Pool Warming

```
서버 재시작 후 Buffer Pool 상태:
  모든 Page가 Free List에 있음
  → 처음 몇 분~수십 분간 모든 쿼리가 Disk I/O 발생
  → Hit Rate가 낮은 상태 → 응답 시간 급등

InnoDB Buffer Pool Dump/Restore (MySQL 5.6+):
  COMMIT 이전 상태를 파일로 저장해 재시작 후 복원

  -- 자동 Dump 설정 (my.cnf):
  innodb_buffer_pool_dump_at_shutdown = ON  -- 종료 시 자동 dump
  innodb_buffer_pool_load_at_startup  = ON  -- 시작 시 자동 load
  
  실제로 저장하는 것:
    Buffer Pool의 실제 데이터가 아닌
    space_id + page_no 목록만 저장 (빠른 I/O)
  
  복원 시:
    목록의 Page를 백그라운드에서 디스크에서 읽어 Buffer Pool 채움
    → 진행 중에도 서버는 정상 동작
    → 완료 전까지는 Hit Rate가 완전히 회복되지 않음

  -- 진행률 확인:
  SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';
  -- "Loaded 32003/49781 pages" 형태로 표시

  -- 수동으로 즉시 dump:
  SET GLOBAL innodb_buffer_pool_dump_now = ON;
  
  -- 수동으로 즉시 load:
  SET GLOBAL innodb_buffer_pool_load_now = ON;
```

### 4. Change Buffer (구 Insert Buffer)

```
Change Buffer: Secondary Index 쓰기 최적화

문제:
  UPDATE orders SET status = 'PAID' WHERE id = 1;
  → orders 테이블에는 status 컬럼의 Secondary Index가 있음
  → 해당 Secondary Index Page가 Buffer Pool에 없다면
    디스크에서 읽어야 함 → Random I/O 발생
  
  Primary Key 업데이트는 Clustered Index Page를 수정 (어차피 읽음)
  But Secondary Index Page는 별도 I/O 필요

Change Buffer 해결책:
  Secondary Index Page가 Buffer Pool에 없을 때:
    실제 Index Page 수정 대신
    Change Buffer(ibdata1 내 특별 영역)에 "이 Index Page를 나중에 수정해야 함" 기록
    → 해당 Index Page가 다른 이유로 Buffer Pool에 로드될 때 병합(Merge)
  
  효과:
    Random I/O를 Sequential로 변환 (Change Buffer는 Sequential 쓰기)
    특히 INSERT/UPDATE/DELETE가 많고 Secondary Index가 많은 경우 효과적

  적합한 경우:
    ✅ 쓰기 집약적 워크로드
    ✅ Secondary Index가 많은 테이블
    ✅ 배치 삽입
  
  적합하지 않은 경우:
    ❌ 쓰기 후 즉시 읽는 패턴 (Merge I/O가 오히려 추가됨)
    ❌ 읽기 집약적 워크로드 (이점 없음)

  -- Change Buffer 크기 설정 (Buffer Pool의 비율):
  SHOW VARIABLES LIKE 'innodb_change_buffer_max_size';
  -- 기본값: 25 (Buffer Pool의 25% 최대 사용)
```

---

## 💻 실전 실험

### 실험 1: Buffer Pool Hit Rate 모니터링

```sql
-- 실시간 Buffer Pool 상태 확인
SELECT
    pool_id,
    pool_size AS total_pages,
    free_buffers AS free_pages,
    database_pages AS used_pages,
    old_database_pages AS old_pages,
    modified_db_pages AS dirty_pages,
    ROUND(hit_rate / 1000 * 100, 2) AS hit_rate_pct,
    ROUND(modified_db_pages / database_pages * 100, 2) AS dirty_pct
FROM information_schema.INNODB_BUFFER_POOL_STATS;

-- hit_rate 목표: 990 이상 (99.0% 이상)
-- dirty_pct 높으면: flush 작업 지연 또는 쓰기 집약적 워크로드
```

### 실험 2: Full Table Scan의 Buffer Pool 오염 실험

```sql
-- 실험 전 Buffer Pool 상태 저장
SELECT pool_id, hit_rate, modified_db_pages
FROM information_schema.INNODB_BUFFER_POOL_STATS;

-- 의도적으로 Full Table Scan 발생 (인덱스 없는 컬럼)
SELECT SQL_NO_CACHE COUNT(*) FROM orders WHERE description LIKE '%test%';

-- 실험 후 Buffer Pool 상태 확인
-- old_database_pages가 증가했는지 확인
-- Young pages의 비율 변화 확인
SELECT pool_id, hit_rate, modified_db_pages
FROM information_schema.INNODB_BUFFER_POOL_STATS;

-- innodb_old_blocks_time 조정 실험:
-- 기본 1000ms → 0으로 설정하면 단순 LRU처럼 동작
SHOW VARIABLES LIKE 'innodb_old_blocks_time';
-- Full Scan 후 Hot Page 교체 여부 변화 관찰
```

### 실험 3: Dirty Page Flush 타이밍 확인

```sql
-- Dirty Page 수 실시간 확인
SELECT modified_db_pages AS dirty_pages
FROM information_schema.INNODB_BUFFER_POOL_STATS;

-- 대량 UPDATE 실행 (Dirty Page 급증 유발)
UPDATE orders SET amount = amount * 1.1 WHERE status = 'PENDING';

-- Dirty Page 변화 추적
SELECT modified_db_pages FROM information_schema.INNODB_BUFFER_POOL_STATS;
-- 급증 후 시간이 지나면 Page Cleaner가 flush → 감소

-- Flush 관련 상태 변수
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_flushed';  -- 총 flush된 Page 수
SHOW STATUS LIKE 'Innodb_os_log_fsyncs';              -- Redo Log fsync 횟수

-- 강제 Checkpoint 발생시키기 (실험용)
-- 주의: 프로덕션에서 사용 금지
-- FLUSH TABLES; -- 모든 테이블 닫고 Dirty Page flush
```

### 실험 4: Buffer Pool Dump/Load

```bash
# Docker 환경에서 Buffer Pool Dump 실험
# MySQL 컨테이너 내에서:

# 1. 현재 Buffer Pool 상태 확인
mysql -u root -proot -e "
SELECT pool_size, database_pages, hit_rate
FROM information_schema.INNODB_BUFFER_POOL_STATS;"

# 2. Buffer Pool Dump (ib_buffer_pool 파일 생성)
mysql -u root -proot -e "SET GLOBAL innodb_buffer_pool_dump_now = ON;"

# 3. Dump 파일 확인
ls -lh /var/lib/mysql/ib_buffer_pool
# space_id:page_no 쌍의 목록

# 4. MySQL 재시작
docker-compose restart mysql

# 5. Load 진행 상황 확인
mysql -u root -proot -e "
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';"
# "Loaded 12034/49781 pages" → 진행 중
# "Buffer pool(s) load completed at ..." → 완료
```

---

## 📊 성능 비교

```sql
-- Buffer Pool 크기별 Hit Rate 비교 시뮬레이션
-- (동일 워크로드에서 Buffer Pool 크기 변화 효과)

/*
워크로드: 100만 건 orders 테이블 랜덤 조회 1000번

innodb_buffer_pool_size = 128MB (테이블보다 훨씬 작음):
  → 자주 사용하는 Page만 캐시 가능
  → Hit Rate: ~60-70%
  → 평균 응답 시간: ~5ms (디스크 I/O 빈번)

innodb_buffer_pool_size = 512MB (테이블과 비슷):
  → 대부분의 활성 Page 캐시 가능
  → Hit Rate: ~90-95%
  → 평균 응답 시간: ~1ms

innodb_buffer_pool_size = 2GB (테이블보다 훨씬 큼):
  → 테이블 전체 + 인덱스 모두 캐시 가능
  → Hit Rate: ~99%+
  → 평균 응답 시간: ~0.1ms
*/

-- 현재 Buffer Pool 활용률 확인
SELECT
    ROUND(database_pages * 16 / 1024, 2) AS used_mb,
    ROUND(pool_size * 16 / 1024, 2) AS total_mb,
    ROUND(database_pages / pool_size * 100, 2) AS utilization_pct
FROM information_schema.INNODB_BUFFER_POOL_STATS;
-- utilization_pct가 90% 이상이면 Buffer Pool 증가 검토
```

---

## ⚖️ 트레이드오프

```
innodb_buffer_pool_size 선택:
  너무 작게:
    ❌ Disk I/O 빈번 → 응답 시간 증가
    ❌ Hit Rate 저하
    ✅ 메모리 여유 → 다른 프로세스, OS 캐시 사용 가능

  너무 크게:
    ❌ OS Swap 발생 가능 → 오히려 성능 저하
    ❌ Buffer Pool 관리 오버헤드 증가
    ✅ Hit Rate 최대화
  
  일반 권장:
    데이터셋이 작으면: 전체 데이터 크기의 110% (인덱스 포함)
    데이터셋이 크면: 물리 메모리의 70~80%

innodb_buffer_pool_instances:
  적게 (1개):
    ❌ Mutex 경합 증가 (다중 CPU 환경에서)
    ✅ 관리 단순

  많이 (8~16개):
    ✅ Mutex 경합 감소 → 병렬 처리 향상
    ❌ 인스턴스당 최소 1GB 이상 권장 (너무 작으면 효과 없음)
  
  권장: Buffer Pool Size / 1GB 개수 (최대 64)

innodb_old_blocks_time (기본 1000ms):
  낮추면 (0에 가까울수록):
    Full Scan 후 Hot Page 교체 위험 증가
    OLTP에서는 증가 고려 (3000~5000ms)
  
  높이면:
    Full Scan이 Young 서브리스트 오염 못 함
    단, 진짜 새로운 Hot Page의 Young 진입이 지연
```

---

## 📌 핵심 정리

```
Buffer Pool 핵심:

역할:
  읽기: 디스크 → Buffer Pool → 쿼리 결과 (Disk I/O 최소화)
  쓰기: 쿼리 수정 → Buffer Pool(Dirty Page) → 나중에 디스크 flush
  → JPA 쓰기 지연과 구조적으로 유사
    JPA: 영속성 컨텍스트에 모아서 flush
    InnoDB: Buffer Pool에 모아서 비동기 flush

Midpoint Insertion (Young/Old LRU):
  새 Page → Old 서브리스트 Head(Midpoint) 삽입
  1초(innodb_old_blocks_time) 후 재접근 → Young으로 승급
  → Full Table Scan이 Hot Page를 교체하는 "Buffer Pool 오염" 방지

Dirty Page Flush 시점:
  Checkpoint (Redo Log 가득 차면)
  LRU Flush (Free List 고갈 시)
  Adaptive Flush (자동 조절)
  → COMMIT 즉시 Data Page flush는 기본값에서 발생하지 않음
  → Redo Log flush만이 내구성 보장 (WAL)

운영 핵심:
  Hit Rate 99% 이상 유지 목표
  Buffer Pool Size = 물리 메모리의 70~80%
  재시작 후: dump/load로 Warming 시간 단축
  새벽 배치: innodb_old_blocks_time 늘려 Hot Page 보호
```

---

## 🤔 생각해볼 문제

**Q1.** JPA의 `@Transactional` 내에서 1000개의 엔티티를 저장(`save()`)했다. JPA는 flush 전까지 1차 캐시(영속성 컨텍스트)에 모은다. InnoDB Buffer Pool 관점에서 flush 시점은 언제이며, COMMIT 전에 서버 크래시가 발생하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

JPA flush → SQL이 JDBC로 MySQL 전송 → InnoDB가 Buffer Pool 업데이트 + Undo Log 기록 + Redo Log 기록 순서로 진행됩니다.

COMMIT 전 크래시 발생 시:
1. Buffer Pool의 Dirty Page들은 사라집니다 (메모리이므로)
2. Redo Log에 해당 트랜잭션의 변경이 기록되어 있으나, COMMIT 마커가 없습니다
3. Crash Recovery 시: Redo Log를 재생하지만, COMMIT 마커가 없는 트랜잭션은 Undo Log를 사용해 롤백합니다
4. 결과: 1000개의 변경사항은 모두 취소됩니다 — JPA와 InnoDB 모두 원자성을 보장합니다.

이것이 `innodb_flush_log_at_trx_commit = 1`(기본값)이 중요한 이유입니다. Redo Log가 COMMIT 시 즉시 flush되기 때문에 크래시 후에도 커밋된 트랜잭션은 복구 가능합니다.

</details>

---

**Q2.** `innodb_old_blocks_time = 0`으로 설정하고 Full Table Scan을 실행하면 어떤 일이 발생하는가? 왜 기본값이 0이 아닌 1000ms인가?

<details>
<summary>해설 보기</summary>

`innodb_old_blocks_time = 0`이면 단순 LRU로 동작합니다. Full Table Scan이 수천 개의 Page를 Old 서브리스트에 넣고, 다시 접근(Scan이 같은 Page를 재접근)하면 즉시 Young 서브리스트로 이동합니다. Full Scan은 모든 Page를 한 번씩 Sequential하게 접근하므로 Young 서브리스트의 Hot Page들이 모두 교체됩니다.

기본값 1000ms의 의미: Full Table Scan은 Extent를 순서대로 읽으면서 매 Page를 보통 1초 이내에 재방문합니다. 1초 이내의 재접근은 Young으로 승급시키지 않으므로, Scan이 아무리 많은 Page를 읽어도 기존 Hot Page는 Young 서브리스트에 안전하게 유지됩니다.

OLTP 서비스에서 야간 배치가 Buffer Pool을 오염시키는 문제를 막는 핵심 설계입니다.

</details>

---

**Q3.** MySQL 서버를 재시작하지 않고 Buffer Pool 크기를 `innodb_buffer_pool_size = 2G`에서 `4G`로 늘리면 어떻게 동작하는가? 즉시 적용되는가?

<details>
<summary>해설 보기</summary>

MySQL 5.7.5+에서는 온라인으로 Buffer Pool 크기를 변경할 수 있습니다.

```sql
SET GLOBAL innodb_buffer_pool_size = 4294967296; -- 4GB
```

즉시 메모리 할당이 일어나지 않고, InnoDB는 Chunk 단위(`innodb_buffer_pool_chunk_size`, 기본 128MB)로 점진적으로 메모리를 추가 할당합니다. 이 과정에서:
- 서버는 계속 정상 동작합니다 (무중단)
- 진행 상황: `SHOW STATUS LIKE 'Innodb_buffer_pool_resize_status'`로 확인 가능
- 완료까지 수 초~수십 초 소요될 수 있습니다

`innodb_buffer_pool_instances`는 재시작 없이 변경 불가입니다. Buffer Pool 크기를 `innodb_buffer_pool_instances × innodb_buffer_pool_chunk_size`의 배수로 설정해야 자동 조정 없이 의도한 크기가 적용됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Page·Block·Extent](./01-page-block-extent.md)** | **[홈으로 🏠](../README.md)** | **[다음: Row Format ➡️](./03-row-format.md)**

</div>
