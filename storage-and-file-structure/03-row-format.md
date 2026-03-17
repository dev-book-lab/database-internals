# Row Format — 데이터가 Page 안에 저장되는 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- InnoDB의 4가지 Row Format(REDUNDANT / COMPACT / DYNAMIC / COMPRESSED) 차이는 무엇인가?
- NULL 비트맵이 실제로 어떻게 저장 공간을 절약하는가?
- VARCHAR(255)와 VARCHAR(1000)은 실제 저장 방식이 어떻게 다른가?
- TEXT/BLOB 컬럼이 "오프-페이지(External Page)"에 저장되는 조건은 무엇인가?
- 한 Row의 실제 크기를 어떻게 계산하는가?
- Row Format 선택이 쿼리 성능과 저장 공간에 어떤 영향을 주는가?

---

## 🔍 왜 이 개념이 중요한가

### Row 크기는 Page당 Row 수를 결정하고, Page당 Row 수는 I/O를 결정한다

```
Row 크기 → Page당 Row 수 → 쿼리 당 Page I/O 수 → 성능

예시: 100만 건 테이블, 평균 Row 크기 차이

Row 크기 200 bytes:
  한 Page(16KB)에 81개 Row
  100만 건 = 12,346 Pages = 풀 스캔 시 12,346 Page I/O

Row 크기 800 bytes:
  한 Page(16KB)에 20개 Row
  100만 건 = 50,000 Pages = 풀 스캔 시 50,000 Page I/O
  → 동일 데이터, 4배 더 많은 I/O

Row 크기를 줄이는 방법:
  ① NULL이 많은 컬럼 → NULL 비트맵으로 자동 절약
  ② VARCHAR를 실제 사용 길이에 맞게 선언
  ③ 자주 읽히지 않는 대용량 컬럼 → 별도 테이블 분리
  ④ COMPRESSED Row Format으로 압축

JPA/Hibernate 사용 시 함정:
  @Column(length = 1000)으로 선언한 VARCHAR
  → 실제 데이터가 짧아도 가변 길이 헤더 증가
  → 인덱스 Key 길이 제한 초과 가능
  → 오프-페이지 저장 조건 충족 가능 (의도치 않게)
```

---

## 😱 잘못된 이해

### Before: VARCHAR의 크기를 크게 잡아도 비용이 없다는 생각

```java
// JPA 엔티티 흔한 패턴
@Entity
public class Product {
    @Id
    @GeneratedValue
    private Long id;
    
    @Column(length = 1000)  // "넉넉하게 잡아두자"
    private String name;
    
    @Column(length = 5000)  // "여유 있게"
    private String description;
    
    @Column(length = 255)
    private String status;
}
```

```
잘못된 이해:
  "VARCHAR는 가변 길이니까 실제 데이터 크기만큼만 공간을 쓴다"
  "선언 크기는 상관없다"

실제 문제들:
  ① VARCHAR(1000) 컬럼은 길이 저장에 2 bytes 필요
     VARCHAR(255) 이하는 1 byte
     → 컬럼이 많으면 헤더 크기 차이 누적

  ② VARCHAR의 선언 크기가 인덱스 생성에 영향
     MySQL 기본 인덱스 최대 키 길이: 767 bytes (utf8mb4 기준)
     VARCHAR(1000) + utf8mb4 = 4000 bytes → 인덱스 생성 불가!
     CREATE INDEX idx_description ON products(description); -- ERROR

  ③ 오프-페이지 저장 조건:
     DYNAMIC Format에서 Row 크기가 페이지의 절반(8KB)에 가까우면
     → 긴 VARCHAR/TEXT 컬럼이 오프-페이지로 밀려남
     → 조회 시 추가 Page I/O
```

---

## ✨ 올바른 이해

### After: Row Format의 실제 저장 방식 이해

```
VARCHAR 실제 저장 구조 (DYNAMIC Format):

  저장 예시: VARCHAR(500)에 "Hello" 저장

  레코드 헤더 영역:
    가변 길이 필드 목록: [5]  ← "Hello"의 실제 길이 (1 byte, 255 이하)
    NULL 플래그 비트맵: [0]   ← NULL 아님
    레코드 헤더: [...]        ← 다음 레코드 포인터, deleted 플래그 등

  데이터 영역:
    [H][e][l][l][o]           ← 실제 데이터 5 bytes

  선언 크기(500)은 저장 공간에 영향 없음
  단, 길이 저장 bytes:
    실제 데이터가 255 bytes 이하: 1 byte로 길이 저장
    실제 데이터가 256 bytes 이상: 2 bytes로 길이 저장
    → VARCHAR(255)와 VARCHAR(256)의 구조적 차이!
```

---

## 🔬 내부 동작 원리

### 1. Row Format 종류와 특징

```
4가지 Row Format 비교:

① REDUNDANT (레거시, MySQL 5.0 이전):
   모든 컬럼의 오프셋(위치 포인터)을 레코드 헤더에 저장
   → 컬럼 수만큼 헤더 크기 증가
   → NULL 컬럼도 실제 공간 차지 (NULL 플래그만 저장하지 않음)
   거의 사용하지 않음 (레거시 호환용)

② COMPACT (MySQL 5.0+):
   가변 길이 컬럼의 실제 길이만 헤더에 저장
   NULL 비트맵으로 NULL 컬럼 공간 절약
   현재도 많이 사용됨

③ DYNAMIC (MySQL 5.7+ 기본값):
   COMPACT와 유사하지만 오프-페이지 저장 방식이 다름
   
   COMPACT의 오프-페이지:
     768 bytes를 레코드 내에 저장하고 나머지만 외부 페이지
   
   DYNAMIC의 오프-페이지:
     전체 데이터를 외부 페이지에 저장 (레코드에는 20 bytes 포인터만)
     → 레코드 내 공간 효율이 더 높음
     → 한 Page에 더 많은 Row 저장 가능 (긴 컬럼이 있는 경우)

④ COMPRESSED (압축):
   DYNAMIC + zlib 압축
   KEY_BLOCK_SIZE 설정으로 압축 페이지 크기 지정
   
   장점:
     저장 공간 30~70% 절약 (데이터에 따라 다름)
     I/O 감소 (더 많은 데이터를 같은 I/O로 읽음)
   
   단점:
     CPU 비용 증가 (압축/해제)
     Buffer Pool에는 압축 해제된 페이지 + 압축된 페이지 모두 유지
     → Buffer Pool 효율 저하 가능

-- 현재 테이블의 Row Format 확인
SHOW TABLE STATUS LIKE 'orders'\G
-- Row_format: Dynamic (기본값)

-- Row Format 변경
ALTER TABLE orders ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;
```

### 2. COMPACT Row Format 상세 구조

```
COMPACT Row Format 레코드 구조:

┌─────────────────────────────────────────────────────┐
│ 가변 길이 컬럼 목록 (역순)                                │
│  각 가변 길이 컬럼의 실제 저장 길이                         │
│  컬럼당 1 byte (≤255) 또는 2 bytes (>255)              │
├─────────────────────────────────────────────────────┤
│ NULL 플래그 비트맵                                     │
│  NULL 허용 컬럼 수 / 8 (올림) bytes                     │
│  비트가 1이면 해당 컬럼은 NULL                            │
├─────────────────────────────────────────────────────┤
│ 레코드 헤더 (5 bytes 고정)                              │
│  다음 레코드 오프셋 포인터 (2 bytes)                      │
│  레코드 타입 (일반/B-Tree 최솟값/최댓값 등)                 │
│  삭제 플래그, 힙 번호 등                                 │
├─────────────────────────────────────────────────────┤
│ 숨겨진 시스템 컬럼                                      │
│  DB_TRX_ID (6 bytes): 마지막 수정 트랜잭션 ID            │
│  DB_ROLL_PTR (7 bytes): Undo Log 포인터               │
│  [DB_ROW_ID (6 bytes): PK 없을 때 InnoDB 자동 생성]     │
├─────────────────────────────────────────────────────┤
│ 실제 컬럼 데이터 (고정 길이 컬럼 먼저, 가변 길이 후)           │
│  NULL 컬럼: 데이터 없음 (비트맵으로만 표시)                 │
└─────────────────────────────────────────────────────┘

숨겨진 시스템 컬럼의 중요성:
  DB_TRX_ID: MVCC에서 이 Row를 수정한 트랜잭션 식별
  DB_ROLL_PTR: Undo Log의 이전 버전 체인 탐색
  → 모든 Row에 고정 13 bytes 오버헤드
  → PK가 없으면 DB_ROW_ID 6 bytes 추가
  → "PK는 항상 명시적으로 정의해야 하는 이유" 중 하나
```

### 3. NULL 비트맵의 실제 절약 효과

```sql
-- 예시 테이블: NULL 허용 컬럼 8개
CREATE TABLE user_profile (
    id          BIGINT NOT NULL,
    nickname    VARCHAR(50) NULL,      -- 비트 0
    birth_date  DATE NULL,             -- 비트 1
    phone       VARCHAR(20) NULL,      -- 비트 2
    address     VARCHAR(200) NULL,     -- 비트 3
    job         VARCHAR(100) NULL,     -- 비트 4
    bio         TEXT NULL,             -- 비트 5
    website     VARCHAR(200) NULL,     -- 비트 6
    avatar_url  VARCHAR(500) NULL,     -- 비트 7
    PRIMARY KEY (id)
) ROW_FORMAT=DYNAMIC;

-- 모든 NULL 허용 컬럼이 NULL인 Row 삽입
INSERT INTO user_profile (id) VALUES (1);

-- REDUNDANT Format이었다면:
--   각 NULL 컬럼도 최소 1 byte씩 저장 = 8+ bytes
-- COMPACT/DYNAMIC Format:
--   NULL 비트맵 1 byte (8개 컬럼 → 1 byte에 8비트)
--   비트 모두 1로 세팅 → 실제 데이터 공간 0
--   절약: 수백 bytes (컬럼 타입에 따라)

-- 실제 Row 크기 확인
SELECT
    id,
    CHAR_LENGTH(nickname) AS nickname_len,
    -- INFORMATION_SCHEMA에서는 정확한 row 크기를 직접 확인하기 어려움
    -- innodb_ruby 또는 pt-online-schema-change 같은 도구 활용
    1 AS placeholder
FROM user_profile WHERE id = 1;
```

### 4. 오프-페이지(External Storage) 저장 조건

```
DYNAMIC Format의 오프-페이지 조건:

규칙: Row의 총 크기가 페이지의 절반(8KB)을 넘으면
      가장 긴 가변 길이 컬럼부터 오프-페이지로 이동
      (레코드에는 20 bytes 포인터만 남김)

예시:
  테이블: id(8) + name(50) + description(TEXT) + content(TEXT)
  
  Row 크기 계산:
    고정: id 8 bytes + 시스템 컬럼 13 bytes = 21 bytes
    가변: name 50 bytes + description 5000 bytes + content 3000 bytes
    합계: ~8071 bytes → 8KB(8192) 초과!
    
  InnoDB의 대응:
    content(3000) → External Page로 이동 (레코드에 20 bytes 포인터)
    → 레코드 크기: 21 + 50 + 5000 + 20 = 5091 bytes (8KB 이하)
    
    여전히 크다면 description도 오프-페이지로 이동

오프-페이지 조회 비용:
  Row 조회:
    Clustered Index Page 1 I/O
    + 오프-페이지 컬럼당 1~N I/O (컬럼 크기에 따라 여러 Page)
  
  -- 오프-페이지 여부 확인
  SELECT
      column_name,
      character_maximum_length,
      data_type
  FROM information_schema.COLUMNS
  WHERE table_schema = 'deep_dive'
    AND table_name = 'orders'
  ORDER BY character_maximum_length DESC NULLS LAST;
  -- TEXT/BLOB은 항상 오프-페이지 대상
  
COMPACT vs DYNAMIC 오프-페이지 비교:
  COMPACT:
    오프-페이지 전환 시 768 bytes를 레코드에 인라인 저장
    → 레코드에 768 bytes + 20 bytes 포인터 = 788 bytes 오버헤드
    → 인라인 데이터를 읽어도 나머지는 오프-페이지 접근 필요 (비효율)
  
  DYNAMIC:
    오프-페이지 전환 시 레코드에 20 bytes 포인터만 남김
    → 레코드가 더 작아짐 → Page당 더 많은 Row 저장
    → MySQL 5.7+ 기본값인 이유
```

### 5. 실제 Row 크기 계산

```
Row 크기 계산 공식 (DYNAMIC Format):

고정 오버헤드:
  - 레코드 헤더: 5 bytes
  - DB_TRX_ID: 6 bytes
  - DB_ROLL_PTR: 7 bytes
  - (PK 없으면) DB_ROW_ID: 6 bytes
  합계: 18 bytes (PK 있을 때)

컬럼별 크기:
  INT / BIGINT: 4 / 8 bytes
  DATETIME: 5 bytes (MySQL 5.6.4+, 마이크로초 없을 때)
  DATE: 3 bytes
  DECIMAL(10,2): 5 bytes
  VARCHAR(N): 실제 데이터 bytes + 길이 bytes (≤255 → 1, >255 → 2)
  CHAR(N): utf8mb4 기준 N × 4 bytes (최대)
            (실제로는 실제 문자 수에 따라 다름, COMPACT는 최소로 저장)
  TEXT / BLOB: 오프-페이지이면 20 bytes 포인터만
  NULL 컬럼: NULL 비트맵에 표시 (해당 컬럼 데이터 0 bytes)

가변 길이 헤더:
  NULL 허용 컬럼 수 / 8 (올림) bytes (NULL 비트맵)
  가변 길이 컬럼 수만큼 1~2 bytes (길이 저장)
```

---

## 💻 실전 실험

### 실험 1: Row Format 확인 및 변경

```sql
-- 현재 Row Format과 설정 확인
SELECT
    table_name,
    row_format,
    avg_row_length,
    data_length,
    index_length
FROM information_schema.tables
WHERE table_schema = 'deep_dive';

-- 기본 Row Format 설정 확인
SHOW VARIABLES LIKE 'innodb_default_row_format';
-- 결과: dynamic (MySQL 5.7.9+)

-- COMPRESSED 변환 실험
ALTER TABLE orders ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;

-- 압축 후 크기 비교
SELECT
    table_name,
    row_format,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb
FROM information_schema.tables
WHERE table_schema = 'deep_dive' AND table_name = 'orders';
-- COMPRESSED 후 data_mb가 감소했는지 확인

-- DYNAMIC으로 되돌리기
ALTER TABLE orders ROW_FORMAT=DYNAMIC;
```

### 실험 2: NULL 비트맵 효과 측정

```sql
-- NULL 컬럼이 많은 테이블 생성
CREATE TABLE test_null (
    id    BIGINT AUTO_INCREMENT PRIMARY KEY,
    col1  VARCHAR(100) NULL,
    col2  VARCHAR(100) NULL,
    col3  VARCHAR(100) NULL,
    col4  VARCHAR(100) NULL,
    col5  VARCHAR(100) NULL,
    col6  VARCHAR(100) NULL,
    col7  VARCHAR(100) NULL,
    col8  VARCHAR(100) NULL
) ROW_FORMAT=DYNAMIC;

-- NULL만 있는 Row와 값이 있는 Row 비교
INSERT INTO test_null (id) VALUES (1);  -- 모두 NULL
INSERT INTO test_null VALUES (2, REPEAT('A', 100), REPEAT('B', 100),
    REPEAT('C', 100), REPEAT('D', 100), REPEAT('E', 100),
    REPEAT('F', 100), REPEAT('G', 100), REPEAT('H', 100));  -- 모두 값

-- 테이블 크기로 간접 비교 (Row 수가 적어 차이 미미할 수 있음)
SELECT * FROM information_schema.INNODB_BUFFER_PAGE
WHERE TABLE_NAME LIKE '%test_null%'
LIMIT 5;
-- page_type, data_size, free_size 컬럼으로 페이지 채움 정도 확인
```

### 실험 3: 오프-페이지 저장 유발 실험

```sql
-- TEXT 컬럼이 있는 테이블
CREATE TABLE test_offpage (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,
    short   VARCHAR(100),
    medium  TEXT,
    large   MEDIUMTEXT
) ROW_FORMAT=DYNAMIC;

-- 작은 데이터 삽입 (인라인 저장)
INSERT INTO test_offpage VALUES (1, 'Hello', REPEAT('A', 100), REPEAT('B', 100));

-- 큰 데이터 삽입 (오프-페이지 유발)
INSERT INTO test_offpage VALUES (2, 'Hello',
    REPEAT('X', 8000),      -- 8KB 이상 → 오프-페이지 가능성
    REPEAT('Y', 65000));    -- 확실히 오프-페이지

-- 오프-페이지 여부는 performance_schema나 innodb_ruby로 확인
-- Buffer Pool에서 오프-페이지 참조 Page 개수로 간접 확인
SELECT
    page_type,
    COUNT(*) AS page_count
FROM information_schema.INNODB_BUFFER_PAGE
WHERE TABLE_NAME = '`deep_dive`.`test_offpage`'
GROUP BY page_type;
-- FIL_PAGE_INDEX: 일반 B-Tree 페이지
-- FIL_PAGE_TYPE_BLOB: 오프-페이지 (BLOB/TEXT External)
```

### 실험 4: VARCHAR 길이 선언이 인덱스에 미치는 영향

```sql
-- VARCHAR(1000) 컬럼에 인덱스 생성 시도
CREATE TABLE test_varchar (
    id    BIGINT AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(1000) CHARACTER SET utf8mb4
);

-- 전체 컬럼 인덱스 시도 → 실패
CREATE INDEX idx_name ON test_varchar(name);
-- ERROR 1071: Specified key was too long; max key length is 3072 bytes
-- utf8mb4: 1000 × 4 = 4000 bytes > 3072 bytes

-- Prefix 인덱스로 해결
CREATE INDEX idx_name_prefix ON test_varchar(name(191));
-- 191 × 4 = 764 bytes → 767 bytes 제한 (innodb_large_prefix OFF 시)
-- MySQL 8.0 기본: 3072 bytes, 191 불필요 → 더 길게 가능

-- 767 bytes 제한이 있던 이유:
-- 내부 인덱스 페이지의 레코드는 최소 2개 이상 들어가야 함
-- B-Tree 특성상 Node에 최소 2개 키 필요
-- 16KB / 2 = 8KB, 헤더 오버헤드 제외하면 ~767 bytes
```

---

## 📊 성능 비교

```sql
-- DYNAMIC vs COMPRESSED 성능 비교
-- 1. DYNAMIC으로 100만 건 삽입 시간 측정
DROP TABLE IF EXISTS perf_dynamic;
CREATE TABLE perf_dynamic LIKE orders;
ALTER TABLE perf_dynamic ROW_FORMAT=DYNAMIC;

SET @start = NOW(6);
INSERT INTO perf_dynamic SELECT * FROM orders;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000000 AS seconds_dynamic;

-- 2. COMPRESSED로 동일 작업
DROP TABLE IF EXISTS perf_compressed;
CREATE TABLE perf_compressed LIKE orders;
ALTER TABLE perf_compressed ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;

SET @start = NOW(6);
INSERT INTO perf_compressed SELECT * FROM orders;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000000 AS seconds_compressed;

-- 3. 크기 비교
SELECT
    table_name,
    row_format,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb
FROM information_schema.tables
WHERE table_schema = 'deep_dive'
  AND table_name IN ('perf_dynamic', 'perf_compressed');

-- 예상 결과:
-- perf_dynamic: 삽입 빠름, 크기 큼
-- perf_compressed: 삽입 20~30% 느림, 크기 30~60% 작음
```

---

## ⚖️ 트레이드오프

```
Row Format 선택 기준:

DYNAMIC (기본값, 대부분의 경우):
  ✅ 오프-페이지 처리 효율적 (포인터 20 bytes만)
  ✅ VARCHAR/TEXT 혼합 사용에 적합
  ✅ CPU 오버헤드 없음
  ❌ 디스크 공간 비압축 상태

COMPRESSED (공간이 중요한 경우):
  ✅ 디스크 공간 30~70% 절약
  ✅ 읽기 I/O 감소 (더 많은 데이터가 같은 크기 I/O로)
  ❌ CPU 비용 증가 (압축/해제)
  ❌ Buffer Pool 사용 증가 (압축/비압축 Page 모두 유지)
  ❌ 쓰기 집약적 워크로드에서 CPU 병목 가능
  
  적합한 케이스:
    읽기 집약적 + 스토리지 비용이 중요
    히스토리 데이터, 아카이브 테이블
    JSON/TEXT 등 압축률 높은 데이터

NULL 컬럼 설계:
  ✅ 자주 비어있는 컬럼은 NULL 허용으로 선언 → 공간 절약
  ❌ NOT NULL이어야 하는 비즈니스 규칙이 있다면 NULL 허용 불가
  
  NULL vs 빈 문자열:
    NULL: 비트맵에만 표시 (0 bytes)
    '': 실제 0 bytes 저장 + 길이 헤더 1 byte
    → NULL이 더 효율적이지만 애플리케이션 NULL 처리 필요

VARCHAR 길이 선언:
  실제 필요한 최대 길이로 선언 (넉넉하게 X)
  
  이유:
    ① 인덱스 키 길이 제한 영향
    ② Memory 임시 테이블 사용 시: CHAR로 변환되어 선언 크기만큼 할당
       ORDER BY, GROUP BY, DISTINCT → 임시 테이블 사용 가능
       VARCHAR(1000) → 임시 테이블에서 1000 bytes × row 수
```

---

## 📌 핵심 정리

```
Row Format 핵심:

기본값 DYNAMIC 사용 이유:
  오프-페이지 처리 효율 (포인터 20 bytes)
  COMPACT보다 Page당 Row 더 많이 저장 가능 (긴 컬럼 있을 때)

NULL 비트맵:
  NULL 허용 컬럼 수 / 8 bytes의 비트맵
  NULL인 컬럼은 데이터 공간 0 bytes
  → NULL이 많은 설계에서 공간 효율 극대화

VARCHAR 저장:
  선언 크기가 아닌 실제 데이터 크기만 저장
  단, 실제 길이 저장에 1~2 bytes 추가
  256 bytes 이상 실제 데이터 → 2 bytes 길이 저장

오프-페이지 (External Storage):
  Row 크기 > ~8KB → 가장 긴 가변 컬럼부터 외부 Page로 이동
  레코드에는 20 bytes 포인터만 남음
  → 추가 Page I/O 발생
  → 자주 읽히지 않는 대형 컬럼은 별도 테이블 분리 고려

숨겨진 시스템 컬럼 (13 bytes 고정):
  DB_TRX_ID (6 bytes): MVCC 트랜잭션 추적
  DB_ROLL_PTR (7 bytes): Undo Log 이전 버전 체인
  → 모든 Row에 최소 18 bytes 오버헤드 (PK 포함)

개발자가 챙겨야 할 것:
  VARCHAR는 실제 필요한 크기로 선언
  자주 NULL인 컬럼 → NULL 허용으로 공간 절약
  TEXT/BLOB이 자주 조회되면 오프-페이지 I/O 비용 고려
  PK는 항상 명시적으로 정의 (DB_ROW_ID 6 bytes 절약)
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 두 테이블 중 어느 것이 동일한 데이터에 대해 더 적은 Page I/O를 발생시키는가? 그 이유는?

```sql
-- 테이블 A
CREATE TABLE orders_a (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    status      VARCHAR(1000) NOT NULL,  -- 실제 값은 최대 10 chars
    amount      DECIMAL(10,2) NOT NULL
);

-- 테이블 B  
CREATE TABLE orders_b (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    status      VARCHAR(20) NOT NULL,   -- 실제 크기에 맞게 선언
    amount      DECIMAL(10,2) NOT NULL
);
```

<details>
<summary>해설 보기</summary>

두 테이블의 실제 저장 크기는 동일합니다. VARCHAR는 실제 데이터 길이만큼만 저장하므로, status에 "PENDING"(7 chars)을 저장하면 두 테이블 모두 7 bytes + 1 byte(길이) = 8 bytes를 씁니다. 따라서 **Page I/O는 동일**합니다.

하지만 테이블 A에는 다른 문제가 있습니다:
1. `status` 컬럼에 단순 인덱스 생성 불가 (utf8mb4 기준 1000 × 4 = 4000 bytes > 3072 bytes 제한)
2. `ORDER BY status`, `GROUP BY status` 실행 시 임시 테이블이 필요하면 메모리에서 VARCHAR(1000)으로 할당 → 메모리 낭비
3. 애플리케이션에서 실수로 1000 chars 이상의 데이터 삽입 허용

즉, Row Format 자체보다는 인덱스 가능 여부와 임시 테이블 비용에서 차이가 납니다. **선언 크기는 실제 저장 크기에는 영향 없지만, 인덱스와 임시 테이블에는 영향을 줍니다.**

</details>

---

**Q2.** PK를 정의하지 않은 테이블을 InnoDB가 어떻게 처리하며, 성능에 어떤 영향을 주는가?

<details>
<summary>해설 보기</summary>

InnoDB는 Clustered Index 기반 스토리지 엔진으로, 반드시 Clustered Index가 필요합니다.

PK가 없을 때 InnoDB의 동작:
1. **UNIQUE NOT NULL 컬럼이 있으면**: 첫 번째 UNIQUE NOT NULL 컬럼을 Clustered Index로 사용
2. **없으면**: InnoDB가 숨겨진 6 bytes `DB_ROW_ID` 컬럼을 자동 생성하고 이를 Clustered Index로 사용

`DB_ROW_ID`의 문제점:
- 모든 Row에 6 bytes 추가 (의도치 않은 공간 낭비)
- 애플리케이션에서 Row를 고유하게 참조할 방법이 없음
- JPA에서 `@Id` 없이 엔티티를 만들면 매핑 오류 발생
- 다른 테이블에서 FK로 참조 불가능

결론: **PK는 항상 명시적으로 정의**해야 합니다. AUTO_INCREMENT BIGINT를 PK로 사용하는 것이 Clustered Index 삽입 순서와 성능 면에서 가장 좋습니다. UUID를 PK로 사용하면 랜덤 삽입으로 B-Tree 페이지 분할이 잦아져 성능이 저하됩니다.

</details>

---

**Q3.** `TEXT` 컬럼과 `VARCHAR(65535)` 컬럼의 저장 방식 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

DYNAMIC Format 기준:

**TEXT 컬럼**: 항상 오프-페이지(External Page) 저장 대상입니다. Row 크기와 무관하게 데이터가 크면 외부 페이지로 이동합니다.

**VARCHAR(65535)**: 65,535 bytes가 최대값이지만, 실제로는 테이블의 Row 크기 제한(65,535 bytes)과 utf8mb4 멀티바이트 때문에 더 짧게 제한됩니다. Row 크기가 8KB 이하라면 인라인으로 저장되고, 초과하면 오프-페이지로 이동합니다.

실용적 차이:
- `TEXT`는 DDL에서 기본값(DEFAULT) 설정 불가, `VARCHAR`는 가능
- `TEXT`는 인덱스 생성 시 반드시 Prefix 길이 지정 필요
- `VARCHAR`는 조건이 맞으면 전체 컬럼 인덱스 가능

성능 면에서는: 실제로 자주 조회되는 짧은 내용이면 `VARCHAR`가 인라인 저장 가능성이 높아 더 효율적입니다. 대용량 내용이면 어차피 오프-페이지이므로 `TEXT`와 큰 차이 없습니다.

</details>

---

<div align="center">

**[⬅️ 이전: InnoDB Buffer Pool](./02-innodb-buffer-pool.md)** | **[홈으로 🏠](../README.md)** | **[다음: Tablespace와 파일 구성 ➡️](./04-tablespace.md)**

</div>
