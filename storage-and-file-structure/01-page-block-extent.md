# Page · Block · Extent — InnoDB 물리 저장 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- InnoDB는 왜 데이터를 1행 단위가 아닌 16KB Page 단위로 읽고 쓰는가?
- Page → Extent → Segment → Tablespace 계층 구조는 왜 이 순서로 설계됐는가?
- Random I/O와 Sequential I/O의 비용 차이가 인덱스 설계에 어떤 영향을 주는가?
- `innodb_page_size`를 변경하면 어떤 트레이드오프가 생기는가?
- 하나의 Row가 16KB를 넘으면 InnoDB는 어떻게 저장하는가?
- `SHOW ENGINE INNODB STATUS`에서 물리 저장 구조와 관련된 정보를 어떻게 읽는가?

---

## 🔍 왜 이 개념이 중요한가

### DB 성능의 90%는 I/O에서 결정된다

```
애플리케이션 개발자가 보는 세계:
  SELECT * FROM orders WHERE user_id = 1;
  → 즉시 결과 반환

실제로 일어나는 일:
  ① CPU: SQL 파싱 → 실행계획 수립      (나노초)
  ② Memory: Buffer Pool 확인           (나노초)
  ③ Disk I/O: 없으면 디스크에서 로드   (밀리초!)

  나노초 vs 밀리초 = 10만 배 차이

  쿼리가 느린 이유의 90%:
    Disk I/O가 발생하기 때문
    → Disk I/O가 발생하는 단위 = Page
    → Page 구조를 이해하면 I/O 비용을 예측할 수 있다

왜 JPA/ORM 개발자에게도 중요한가:
  JPA가 SELECT를 날릴 때:
    → InnoDB는 그 Row가 속한 Page를 통째로 메모리에 올림
    → Lazy Loading이 N+1이 되면 N개의 Page I/O 발생 가능
    → 인덱스가 없는 조회는 테이블의 모든 Page를 읽음
  
  Page 구조를 모르면:
    "왜 1개 Row를 읽었는데 이렇게 느리지?" 를 설명할 수 없다
```

---

## 😱 잘못된 이해

### Before: Row 단위로 읽는다고 생각하는 접근

```
잘못된 멘탈 모델:
  SELECT * FROM orders WHERE id = 42;
  → InnoDB: "id = 42인 row 하나를 디스크에서 읽어옴"
  → 1 row = 1 disk read

실제:
  id = 42인 row가 속한 16KB Page 전체를 읽음
  그 Page에 포함된 수십 ~ 수백 개의 다른 row도 함께 메모리에 올라옴

잘못된 결론:
  "인덱스가 없어도 1개 row를 읽는 건 빠르겠지?"
  → 실제로는 Full Table Scan = 테이블의 모든 Page를 순서대로 읽음
  → 100만 row 테이블 = 수천 개의 Page = 수천 번의 I/O

잘못된 최적화:
  "자주 읽는 컬럼만 SELECT해서 row 크기를 줄이면 빠르겠지?"
  → 컬럼을 적게 선택해도 Page 단위로 읽으므로 I/O 횟수는 동일
  → (단, Covering Index 사용 시에는 의미 있음 — Ch2에서 설명)
```

---

## ✨ 올바른 이해

### After: Page 단위 I/O의 실제 의미

```
InnoDB I/O 단위:
  최소 단위 = Page (기본 16KB)
  디스크에서 메모리(Buffer Pool)로 옮길 때 항상 Page 단위
  메모리에서 디스크로 쓸 때도 항상 Page 단위

Page 크기가 16KB인 이유:
  HDD 섹터 크기: 512B ~ 4KB
  SSD 페이지 크기: 4KB ~ 16KB
  OS 파일시스템 블록: 4KB ~ 8KB
  
  16KB = 한 번의 I/O로 실용적인 양의 데이터를 올릴 수 있는 크기
  너무 작으면 → 잦은 I/O (오버헤드 증가)
  너무 크면 → 필요 없는 데이터를 많이 올림 (메모리 낭비, 쓰기 비용 증가)

인덱스 설계와의 연결:
  인덱스 B-Tree의 각 Node = 1 Page (16KB)
  → Node 하나에 수백~수천 개의 키 저장 가능
  → 트리 높이가 낮아짐 → I/O 횟수 최소화
  (이 내용은 Ch2에서 상세 분석)
```

---

## 🔬 내부 동작 원리

### 1. InnoDB 물리 저장 계층 구조

```
Tablespace (.ibd 파일)
  └── Segment (논리 단위 — 인덱스 Leaf / Non-Leaf / Rollback 등)
        └── Extent (1MB = 64개 Page)
              └── Page (16KB — I/O의 최소 단위)
                    └── Row (실제 데이터)
                          └── Column Value

각 계층의 역할:

Page (16KB):
  InnoDB의 기본 I/O 단위
  Buffer Pool에서 관리되는 캐시 단위
  종류: Data Page, Index Page, Undo Log Page, System Page 등

Extent (1MB = 64 × 16KB):
  연속된 64개의 Page 묶음
  InnoDB가 디스크 공간을 할당할 때 Extent 단위로 요청
  → 이유: 연속 공간 확보 → Sequential I/O 가능
  
  새 테이블 생성 시:
    처음에는 1개 Page씩 할당 (소규모 테이블)
    32개 Page 초과 시 1개 Extent씩 할당 (대규모 확장)

Segment:
  논리적 단위 — 각 인덱스는 2개 Segment 보유
    ① Leaf Segment: 실제 데이터 Row (B-Tree Leaf Node)
    ② Non-Leaf Segment: 인덱스의 내부 Node
  
  Segment = 여러 Extent의 묶음
  Segment 헤더: 할당된 Extent 목록, 현재 크기 등 관리

Tablespace:
  하나 이상의 .ibd 파일로 구성
  innodb_file_per_table=ON (기본): 테이블마다 .ibd 파일
  System Tablespace (ibdata1): 공유 자원 저장
```

### 2. Page 내부 구조

```
InnoDB Data Page (16KB) 내부 레이아웃:

┌─────────────────────────────┐  Offset 0
│   File Header (38 bytes)    │  ← Page 번호, 이전/다음 Page 포인터,
│                             │     Page 타입, LSN(Log Sequence Number)
├─────────────────────────────┤  Offset 38
│   Page Header (56 bytes)    │  ← Record 수, Free Space 시작 위치,
│                             │     삭제된 Record 목록, 마지막 삽입 위치
├─────────────────────────────┤  Offset 94
│   Infimum Record (13 bytes) │  ← 페이지 내 최솟값 가상 레코드
├─────────────────────────────┤
│   Supremum Record (13 bytes)│  ← 페이지 내 최댓값 가상 레코드
├─────────────────────────────┤
│                             │
│   User Records              │  ← 실제 Row 데이터
│   (가변 크기)                 │     링크드 리스트로 정렬 순서 유지
│                             │     새 Record는 Free Space에서 할당
├─────────────────────────────┤
│   Free Space                │  ← 빈 공간
│   (가변 크기)                 │
├─────────────────────────────┤
│   Page Directory            │  ← Record 그룹의 오프셋 슬롯 배열
│   (슬롯당 2 bytes)           │     Binary Search로 Record 빠르게 탐색
├─────────────────────────────┤
│   File Trailer (8 bytes)    │  ← Checksum (Page 손상 감지용)
└─────────────────────────────┘  Offset 16383

핵심 포인트:
  User Records는 삽입 순서가 아닌 키 기준 정렬 순서로 논리적 링크드 리스트
  물리적 위치는 삽입 순서, 논리적 순서는 포인터로 관리
  → 삽입 시 전체 재정렬 불필요
  
  Page Directory:
    Page를 4~8개 Record 그룹으로 나누고, 각 그룹의 마지막 Record 오프셋 저장
    Binary Search로 O(log n) 탐색 → O(n) 선형 스캔 불필요
```

### 3. File Header — Page 간 연결 구조

```c
// InnoDB Page File Header (InnoDB 소스: fil0fil.h 참고)
// Page 38바이트 헤더의 핵심 필드:

FIL_PAGE_SPACE_OR_CHKSUM  (4 bytes): Checksum 또는 Space ID
FIL_PAGE_OFFSET           (4 bytes): 이 Page의 번호 (Tablespace 내 위치)
FIL_PAGE_PREV             (4 bytes): 이전 Page 번호 (B-Tree Leaf용 링크드 리스트)
FIL_PAGE_NEXT             (4 bytes): 다음 Page 번호
FIL_PAGE_LSN              (8 bytes): 마지막으로 수정된 Log Sequence Number
FIL_PAGE_TYPE             (2 bytes): Page 타입
  FIL_PAGE_INDEX     = 0x45BF  (B-Tree 데이터 페이지)
  FIL_PAGE_UNDO_LOG  = 0x0002  (Undo Log 페이지)
  FIL_PAGE_INODE     = 0x0003  (Segment 관리)
  FIL_PAGE_IBUF_FREE_LIST = 0x0004 (Change Buffer)

FIL_PAGE_PREV / FIL_PAGE_NEXT의 의미:
  B-Tree의 Leaf Node Page들이 이 포인터로 연결됨
  → 범위 스캔 시 다음 Page를 포인터로 즉시 접근
  → Sequential I/O 가능 (디스크 탐색 없이 다음 Page 읽기)

  예: WHERE id BETWEEN 100 AND 200 실행 시
    id=100이 있는 Leaf Page 찾기 (B-Tree 탐색)
    → FIL_PAGE_NEXT 포인터로 다음 Page 순회
    → id=200을 넘는 순간 중단
```

### 4. Random I/O vs Sequential I/O

```
디스크 I/O 비용 비교:

HDD (자기 디스크):
  Sequential I/O: 100~200 MB/s
  Random I/O: 0.5~2 MB/s  ← 탐색 시간(Seek Time) 때문
  
  Random I/O가 느린 이유:
    디스크 암(Arm)이 물리적으로 이동해야 함
    평균 탐색 시간: ~8ms
    1초에 Random I/O 125개 = 8ms × 125 = 1000ms
    
SSD (플래시 메모리):
  Sequential I/O: 500~3000 MB/s
  Random I/O: 50~500 MB/s
  
  SSD에서도 Sequential이 4~10배 빠름
  (Flash Translation Layer 오버헤드, 내부 페이지 정렬)

InnoDB에서의 실제 영향:

① Full Table Scan (Sequential I/O):
   테이블의 모든 Extent를 순서대로 읽음
   → HDD에서도 Sequential I/O이므로 상대적으로 빠름
   → Optimizer가 작은 테이블에서 Full Scan을 선택하는 이유

② Secondary Index를 통한 조회 (Random I/O):
   Secondary Index Leaf에서 PK 획득 (Index Page 접근)
   → PK로 Clustered Index에서 실제 Row 접근 (Data Page 접근)
   → 2번의 Page I/O = 2번의 Random I/O
   → 결과 행이 많으면 수천 번의 Random I/O
   
   Optimizer가 Full Scan vs Index Scan을 선택하는 기준:
     결과 행 비율이 20~30% 이상이면 Full Scan이 더 빠를 수 있음
     (Random I/O 수 > Full Scan의 Sequential I/O 수)
   
   EXPLAIN에서 이 결정이 type: ALL vs type: ref로 나타남
```

### 5. Extent 단위 공간 할당이 Sequential I/O를 만드는 이유

```
공간 할당 없이 1 Page씩 할당한다면:

  INSERT 10000 rows:
    Page 1: 디스크의 섹터 0~31 (16KB)
    Page 2: 디스크의 섹터 10000~10031  ← 멀리 떨어진 위치
    Page 3: 디스크의 섹터 50000~50031  ← 또 다른 위치
    ...
    → Sequential Scan이어도 물리적으로 Random I/O 발생

InnoDB의 Extent 단위 할당:
  Extent = 64 Page = 1MB = 연속된 64 × 16KB 블록
  
  INSERT 10000 rows:
    Extent 1: 섹터 0~2047 (1MB, 연속)
    Extent 2: 섹터 2048~4095 (1MB, 연속)
    ...
    → Full Table Scan = 각 Extent 내부는 Sequential I/O
    → 디스크 암 이동이 Extent 경계에서만 발생

innodb_ruby로 실제 Extent 분포 확인 가능:
  gem install innodb_ruby
  innodb_space -f orders.ibd space-extents
  → 어느 Extent가 어느 Segment에 속하는지 시각화
```

---

## 💻 실전 실험

### 실험 1: Page 크기 확인

```sql
-- InnoDB Page 크기 확인
SHOW VARIABLES LIKE 'innodb_page_size';
-- 결과: 16384 (16KB = 16 × 1024)

-- Page 크기는 인스턴스 생성 시에만 설정 가능 (이후 변경 불가)
-- my.cnf: innodb_page_size = 16384 (기본값, 4096/8192/16384/32768/65536 선택 가능)
```

### 실험 2: 테이블 실제 크기와 Page 수 계산

```sql
-- 실험용 테이블 생성
CREATE TABLE orders (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    amount     DECIMAL(10,2) NOT NULL,
    status     VARCHAR(20) NOT NULL,
    created_at DATETIME NOT NULL,
    description TEXT
) ENGINE=InnoDB;

-- 100만 건 데이터 삽입 (docker-compose 환경 기준)
INSERT INTO orders (user_id, amount, status, created_at, description)
SELECT
    FLOOR(RAND() * 100000) + 1,
    ROUND(RAND() * 99999 + 1, 2),
    ELT(FLOOR(RAND() * 3) + 1, 'PENDING', 'PAID', 'CANCELLED'),
    DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 365) DAY),
    REPEAT('x', FLOOR(RAND() * 100))
FROM information_schema.columns a
CROSS JOIN information_schema.columns b
LIMIT 1000000;

-- 테이블 크기와 Page 수 확인
SELECT
    table_name,
    ROUND(data_length / 1024 / 1024, 2)            AS data_mb,
    ROUND(index_length / 1024 / 1024, 2)           AS index_mb,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS total_mb,
    ROUND(data_length / 16384)                     AS estimated_data_pages,
    ROUND(index_length / 16384)                    AS estimated_index_pages
FROM information_schema.tables
WHERE table_schema = 'deep_dive' AND table_name = 'orders';

-- 예상 결과:
-- data_mb: ~150-200MB
-- estimated_data_pages: ~10000-13000
-- → 100만 row = 10000개 이상의 Page로 구성
```

### 실험 3: Page 레벨 I/O 모니터링

```sql
-- Buffer Pool Hit Rate 계산
SELECT
    (1 - (
        (SELECT variable_value FROM performance_schema.global_status
         WHERE variable_name = 'Innodb_buffer_pool_reads') /
        (SELECT variable_value FROM performance_schema.global_status
         WHERE variable_name = 'Innodb_buffer_pool_read_requests')
    )) * 100 AS buffer_pool_hit_rate_pct;
-- 99% 이상이 이상적, 95% 미만이면 Buffer Pool 크기 증가 검토

-- Full Scan vs Index Scan I/O 비교 실험
-- Page Read 카운터 스냅샷
SELECT variable_value INTO @before_reads
FROM performance_schema.global_status
WHERE variable_name = 'Innodb_pages_read';

-- Full Table Scan 유발
SELECT COUNT(*) FROM orders WHERE amount > 50000;

SELECT variable_value INTO @after_reads
FROM performance_schema.global_status
WHERE variable_name = 'Innodb_pages_read';

SELECT @after_reads - @before_reads AS pages_read_for_full_scan;
-- 수천 개의 Page 읽음

-- 인덱스 추가 후 동일 실험
CREATE INDEX idx_amount ON orders(amount);
-- 동일 쿼리 실행 → 훨씬 적은 Page 읽음
```

### 실험 4: SHOW ENGINE INNODB STATUS로 Page 정보 확인

```sql
SHOW ENGINE INNODB STATUS\G

-- 출력 중 BUFFER POOL AND MEMORY 섹션 해석:
-- -------
-- BUFFER POOL AND MEMORY
-- -------
-- Buffer pool size   131072       ← 총 Page 수 (131072 × 16KB = 2GB)
-- Free buffers       124908       ← 빈 Page 수
-- Database pages     5720         ← 사용 중인 Page 수
-- Old database pages 1696         ← LRU Old 서브리스트의 Page 수
-- Modified db pages  910          ← Dirty Page 수 (아직 디스크에 안 씀)
-- Pending reads      0            ← 디스크 읽기 대기 중인 요청
-- Pages made young 1919           ← Old → Young으로 이동한 Page 수
-- Pages read 2343, created 734, written 2988
-- Buffer pool hit rate 1000 / 1000 ← 1000/1000 = 100% Hit Rate
```

---

## 📊 성능 비교

```sql
-- Row 크기 vs Page 당 Row 수 계산
SELECT
    table_name,
    avg_row_length,
    ROUND(16384 / avg_row_length) AS rows_per_page,   -- 한 Page에 몇 Row
    table_rows,
    ROUND(table_rows / (16384 / avg_row_length)) AS estimated_pages
FROM information_schema.tables
WHERE table_schema = 'deep_dive' AND table_name = 'orders';
-- rows_per_page가 낮을수록 → 더 많은 Page I/O 필요

/*
innodb_page_size 별 트레이드오프 요약:

  4KB:  Random I/O 시 불필요한 데이터 최소화
        BUT B-Tree 높이 증가 → 탐색 I/O 증가

  16KB: B-Tree Node당 수백 개 키 → 낮은 트리 높이 (기본값, 균형점)

  64KB: BLOB/TEXT 많은 경우 오프-페이지 저장 최소화
        BUT Buffer Pool 낭비, Random I/O 비용 증가
*/
```

---

## ⚖️ 트레이드오프

```
Page 크기 선택:
  작게 (4KB/8KB):
    ✅ OLTP: 소량 Row 조회 시 불필요한 데이터 로드 최소화
    ✅ Random I/O 패턴에서 메모리 효율적
    ❌ B-Tree 높이 증가 → 인덱스 탐색 I/O 증가
    ❌ 인스턴스 생성 후 변경 불가

  크게 (32KB/64KB):
    ✅ DW/OLAP: 대용량 Sequential Scan에 유리
    ✅ TEXT/BLOB 컬럼 많은 경우 오프-페이지 저장 감소
    ❌ Buffer Pool 낭비 증가
    ❌ 소량 조회 시 오버헤드

Extent 크기 (고정 1MB):
  Extent가 클수록:
    ✅ Sequential I/O 구간 길어짐 → HDD에서 유리
    ❌ 소규모 테이블에서 공간 낭비
  
  InnoDB는 이를 고려해 소규모(32 Page 미만)는 1 Page씩 할당

Row가 Page를 초과하는 경우:
  DYNAMIC Format(기본): 오프-페이지(External Page)에 나머지 저장
  → 조회 시 추가 Page I/O 발생
  → 가능하면 Row 크기를 8KB 이하로 유지 권장
  → TEXT/BLOB은 자주 읽히지 않는다면 별도 테이블로 분리 고려
```

---

## 📌 핵심 정리

```
InnoDB 물리 저장 구조 핵심:

계층 구조:
  Tablespace → Segment → Extent(1MB) → Page(16KB) → Row

Page(16KB)가 핵심인 이유:
  모든 I/O의 최소 단위
  Buffer Pool 캐시의 관리 단위
  B-Tree Node 크기 (Fan-out에 직결)
  1 Row를 읽어도 16KB Page 전체가 메모리에 올라옴

Extent(1MB)가 필요한 이유:
  연속 공간 보장 → Sequential I/O 가능
  HDD에서 디스크 탐색 횟수 감소
  공간 할당 효율화

Random I/O vs Sequential I/O:
  Random: Secondary Index → PK → Row 조회 (페이지 위치가 불연속)
  Sequential: Full Table Scan, Clustered Index Range Scan
  HDD에서 Random이 100배 이상 느릴 수 있음
  
  Optimizer가 Full Scan을 선택하는 이유:
    결과 비율 20~30% 초과 시 Random I/O × N > Sequential I/O 전체

개발자가 챙겨야 할 것:
  인덱스 없는 대용량 테이블 조회 = 수천 Page의 Sequential I/O
  N+1 쿼리 = N × (몇 Page의 Random I/O)
  Covering Index = 데이터 Page 없이 Index Page만으로 응답
    → Page I/O 횟수 감소 (Ch2-03에서 상세 분석)
```

---

## 🤔 생각해볼 문제

**Q1.** `orders` 테이블에 100만 건이 있고, 평균 Row 크기가 200 bytes이다. 인덱스 없이 `SELECT * FROM orders WHERE description LIKE '%배송%'`을 실행하면 최소 몇 번의 Page I/O가 발생하는가?

<details>
<summary>해설 보기</summary>

계산 과정:
1. 한 Page(16KB = 16,384 bytes)에 들어가는 Row 수: `16,384 / 200 ≈ 81개` (헤더 오버헤드 제외)
2. 전체 Page 수: `1,000,000 / 81 ≈ 12,346개`
3. Buffer Pool에 없다면 최소 **12,346번의 Disk I/O** 발생

이것이 Full Table Scan이 무서운 이유입니다. `LIKE '%배송%'`는 앞에 와일드카드가 있어 인덱스도 사용할 수 없습니다. 이런 경우 Elasticsearch 같은 Full-Text Search 엔진을 검토해야 합니다.

</details>

---

**Q2.** `innodb_page_size`를 64KB로 늘리면 B-Tree 높이가 낮아질 텐데, InnoDB가 기본값을 16KB로 설정한 이유는?

<details>
<summary>해설 보기</summary>

B-Tree 높이만 놓고 보면 Page 크기가 클수록 유리합니다. 하지만 다음 트레이드오프가 있습니다:

1. **Buffer Pool 낭비**: Row 1개를 읽어도 64KB가 올라옵니다. 같은 메모리에서 캐시할 수 있는 엔트리 수가 4배 줄어듭니다.

2. **Random I/O 비용 증가**: Random Access 패턴에서 불필요한 64KB가 I/O됩니다. SSD 기준으로도 4배 더 큰 데이터를 읽게 됩니다.

3. **쓰기 증폭**: Dirty Page를 flush할 때 64KB 전체를 씁니다. 1 byte만 변경해도 64KB 쓰기가 발생합니다.

4. **대부분의 Row가 1~2KB 수준**: 16KB면 한 Page에 수십 개의 Row가 들어갑니다.

16KB는 B-Tree 높이, Buffer Pool 효율, I/O 비용의 균형점으로 설계된 값입니다.

</details>

---

**Q3.** InnoDB는 테이블 생성 초기에 Extent(1MB) 단위가 아닌 1 Page씩 할당한다. 왜 그런 설계인가?

<details>
<summary>해설 보기</summary>

`CREATE TABLE` 후 데이터를 거의 넣지 않는 소규모 테이블이 많기 때문입니다.

처음부터 Extent(1MB) 단위로 할당하면:
- 10개 Row만 있는 테이블도 1MB의 디스크 공간 점유
- 서버에 수천 개의 테이블이 있다면 수 GB의 공간이 빈 Extent로 낭비

그래서 InnoDB는 "Fragment Page"라는 개념을 사용합니다:
- 처음 32 Page는 System Tablespace의 공유 Fragment 영역에서 1 Page씩 할당
- 33번째 Page부터 해당 테이블 전용 Extent를 통째로 할당

이 설계로 소규모 테이블은 디스크 낭비 없고, 대규모 테이블은 Sequential I/O의 이점을 얻습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: InnoDB Buffer Pool ➡️](./02-innodb-buffer-pool.md)**

</div>
