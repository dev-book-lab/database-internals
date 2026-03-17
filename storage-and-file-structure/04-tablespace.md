# Tablespace와 파일 구성 — InnoDB 데이터가 디스크에 배치되는 방식

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- System Tablespace(ibdata1)와 File-Per-Table Tablespace(.ibd)의 차이는 무엇인가?
- `innodb_file_per_table`을 OFF로 설정하면 어떤 문제가 생기는가?
- Undo Tablespace는 MVCC와 어떤 관계인가?
- `TRUNCATE TABLE`과 `DELETE FROM` 이후 디스크 공간이 회수되는 조건은 무엇인가?
- Tablespace 파일이 손상되면 InnoDB는 어떻게 감지하고 대응하는가?
- `ALTER TABLE`이 대용량 테이블에서 느린 이유와 Online DDL의 동작 원리는?

---

## 🔍 왜 이 개념이 중요한가

### "왜 DB 서버 디스크가 꽉 찼지?" — 운영에서 마주치는 현실

```
실무 시나리오 ①: ibdata1이 끝없이 커진다
  
  innodb_file_per_table = OFF 상태에서 대용량 작업:
    DELETE FROM orders WHERE created_at < '2023-01-01';  -- 수천만 건 삭제
    → ibdata1에서 공간이 "해제"되지 않음
    → ibdata1은 한 번 커지면 줄어들지 않음
    → OPTIMIZE TABLE을 실행해도 ibdata1 크기 변화 없음
  
  결과: 디스크 가득 참 → MySQL 쓰기 실패 → 서비스 장애

실무 시나리오 ②: ALTER TABLE이 예상보다 훨씬 오래 걸린다
  
  ALTER TABLE orders ADD COLUMN memo TEXT;
    → 테이블 크기에 비례한 시간
    → 100GB 테이블 → 수십 분 ~ 수 시간
    → 이 시간 동안 테이블 잠김 → 서비스 영향
  
  Online DDL vs 일반 DDL의 차이를 모르면
  배포 중 의도치 않은 서비스 다운이 발생

실무 시나리오 ③: Undo Log가 Tablespace를 점유
  
  오래 실행되는 트랜잭션:
    MVCC를 위해 Undo Log가 계속 쌓임
    → undo_001, undo_002 파일이 수GB로 성장
    → 트랜잭션 종료 후에야 삭제(Purge) 가능
```

---

## 😱 잘못된 이해

### Before: 테이블 삭제/Truncate하면 디스크 공간이 즉시 회수된다는 생각

```sql
-- 잘못된 기대:
DELETE FROM big_table WHERE year < 2020;
-- "이제 디스크에서 수십 GB가 줄었겠지?"

-- 실제: innodb_file_per_table = OFF (또는 ibdata1 사용) 환경에서
--   페이지 내부에서 Row에 '삭제 표시'만 함
--   InnoDB는 해당 공간을 재사용하지만 OS에 반환하지 않음
--   ibdata1은 절대 줄어들지 않음

-- 그나마 낫지만 여전히 문제:
TRUNCATE TABLE big_table;
-- innodb_file_per_table = ON 환경에서:
--   .ibd 파일 삭제 후 새로 생성 → 디스크 공간 즉시 회수 ✅
-- innodb_file_per_table = OFF 환경에서:
--   ibdata1 내부에서 초기화 → 파일 크기 변화 없음 ❌

-- DROP TABLE:
DROP TABLE big_table;
-- innodb_file_per_table = ON: .ibd 파일 삭제 → 즉시 회수 ✅
-- innodb_file_per_table = OFF: ibdata1 내부 공간 표시만 변경 ❌
```

---

## ✨ 올바른 이해

### After: Tablespace 종류별 공간 관리 방식

```
InnoDB Tablespace 종류:

① System Tablespace (ibdata1):
   - InnoDB 내부 자료구조 (Data Dictionary — MySQL 8.0 이전)
   - Change Buffer (Secondary Index 쓰기 최적화 버퍼)
   - Undo Log 일부 (MySQL 8.0에서 별도 분리됨)
   
   특징:
   - 자동으로만 커짐, 줄어들지 않음
   - ibdata1 하나에 모든 테이블 데이터가 들어감 (file_per_table=OFF 시)
   - innodb_data_file_path 로 설정

② File-Per-Table Tablespace (.ibd 파일):
   - 테이블당 독립 파일
   - innodb_file_per_table = ON (MySQL 5.6.6+ 기본값)
   - DROP TABLE, TRUNCATE TABLE 시 OS에 공간 즉시 반환
   - 개별 파일 백업/복원 가능
   - 운영상의 이유로 대부분 ON으로 사용

③ Undo Tablespace (undo_001, undo_002):
   - MVCC를 위한 이전 버전 데이터 저장
   - MySQL 8.0부터 System Tablespace에서 분리
   - 기본 2개, 최대 127개
   - 오래된 트랜잭션 종료 후 Purge Thread가 정리

④ Temporary Tablespace (ibtmp1):
   - 임시 테이블, ORDER BY/GROUP BY의 정렬 임시 결과
   - 서버 재시작 시 자동 삭제 후 재생성
   - 서버 실행 중 급격히 커지면 → 복잡한 쿼리의 정렬/해시 조인 임시 결과

⑤ General Tablespace:
   - 여러 테이블이 하나의 .ibd 파일을 공유 (수동 생성)
   - 잘 쓰이지 않음
```

---

## 🔬 내부 동작 원리

### 1. .ibd 파일 내부 구조

```
orders.ibd 파일 내부:

페이지 0:    File Space Header Page
             → Tablespace의 메타데이터
             → Extent 상태 목록 (FREE/FULL/PARTIALLY_FREE)
             → Segment Header 목록

페이지 1:    Insert Buffer Bitmap
             → Change Buffer 사용 비트맵

페이지 2:    Inode Page (Segment 관리)
             → 각 Segment의 Extent 목록, Free Page 목록

페이지 3+:   실제 데이터 페이지 (B-Tree Root Node)
             → Clustered Index (PK 기준)
             → Secondary Index (각 인덱스마다 별도 세그먼트)

.ibd 구조 확인 (innodb_ruby):
  innodb_space -f orders.ibd space-page-type-summary
  
  출력 예시:
  #    type                count   percent   description
  0    FSP_HDR             1       0.02%     File space header
  1    IBUF_BITMAP         1       0.02%     Insert buffer bitmap
  2    INODE               1       0.02%     File segment inode
  3    INDEX               8143    99.89%    B-tree index
  9    ALLOCATED           5       0.06%     Freshly allocated
```

### 2. innodb_file_per_table ON vs OFF 비교

```sql
-- 현재 설정 확인
SHOW VARIABLES LIKE 'innodb_file_per_table';
-- 기본값: ON (MySQL 5.6.6+)

-- innodb_file_per_table = ON 환경:
-- 각 테이블별 .ibd 파일 존재
-- /var/lib/mysql/deep_dive/
--   orders.ibd    -- orders 테이블 데이터 + 인덱스
--   products.ibd  -- products 테이블 데이터 + 인덱스

-- 공간 회수:
DROP TABLE orders;
-- → orders.ibd 파일 삭제 → OS에 즉시 공간 반환

TRUNCATE TABLE orders;
-- → orders.ibd 재생성 → 빈 파일로 초기화

-- innodb_file_per_table = OFF 환경:
-- 모든 테이블이 ibdata1에 저장
-- /var/lib/mysql/ibdata1 (점점 커짐)

-- 공간 회수 시 필요한 작업:
-- 1. mysqldump로 전체 백업
-- 2. MySQL 중지
-- 3. ibdata1 삭제
-- 4. MySQL 재시작 (ibdata1 재생성)
-- 5. 백업 데이터 복원
-- → 서비스 중단 없이는 불가능, 대규모 작업 필요
```

### 3. Undo Tablespace와 MVCC 연결

```
Undo Tablespace가 커지는 이유:

  MVCC 동작:
    UPDATE orders SET status = 'PAID' WHERE id = 1;
    → 변경 전 데이터를 Undo Log에 기록
    → Undo Log는 Undo Tablespace에 저장
    → 다른 트랜잭션이 이 Row의 이전 버전을 볼 수 있어야 하는 동안 유지

  언제 Undo Log를 삭제할 수 있는가:
    "이전 버전을 볼 수 있어야 하는 트랜잭션이 없을 때"
    = 현재 활성 중인 트랜잭션 중 가장 오래된 트랜잭션보다 이전 Undo Log

  문제 시나리오:
    트랜잭션 A: BEGIN → (1시간 동안 커밋하지 않음)
    그 동안 다른 트랜잭션들의 UPDATE → Undo Log 계속 쌓임
    → 트랜잭션 A가 "나는 1시간 전 시점의 데이터가 필요해"라고 요구하므로
    → 1시간치 Undo Log 모두 유지 필요
    → Undo Tablespace 급격히 증가

  -- 현재 활성 트랜잭션 확인
  SELECT
      trx_id,
      trx_state,
      TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS seconds_active,
      trx_query
  FROM information_schema.INNODB_TRX
  ORDER BY trx_started;
  -- seconds_active가 큰 트랜잭션이 있으면 Undo Log 축적 원인

  -- Undo Log 크기 확인
  SELECT
      name,
      ROUND(file_size / 1024 / 1024, 2) AS size_mb
  FROM information_schema.FILES
  WHERE file_type = 'UNDO LOG';

Purge Thread:
  트랜잭션이 종료되면 Purge Thread가 Undo Log를 정리
  SHOW STATUS LIKE 'Innodb_purge_trx_id_age';  -- Purge 지연 트랜잭션 수
  innodb_purge_threads: Purge 작업 병렬화 스레드 수 (기본 4)
```

### 4. Online DDL — ALTER TABLE과 Tablespace

```
DDL 작업이 Tablespace에 미치는 영향:

전통적 ALTER TABLE (MySQL 5.6 이전):
  1. 새 테이블 생성
  2. 기존 데이터 전체 복사
  3. 인덱스 재생성
  4. 기존 테이블 삭제, 새 테이블로 교체
  
  → 원본 테이블 크기만큼 추가 디스크 공간 필요
  → 작업 중 테이블 쓰기 잠금
  → 100GB 테이블 = 100GB 추가 필요 + 수 시간 잠금

Online DDL (MySQL 5.6+):
  ALGORITHM = INPLACE:
    새 파일 없이 기존 .ibd를 수정
    작업 중 DML(SELECT/INSERT/UPDATE/DELETE) 허용
    DDL 중 발생한 DML 변경사항 → Row Log에 임시 기록
    DDL 완료 후 Row Log를 실제 테이블에 적용
    
    추가 디스크: Row Log 크기만큼만 필요 (훨씬 작음)

  ALGORITHM = COPY:
    전통적 방식 그대로 (공간 2배 필요, 잠금 발생)
    불가피한 경우에만 사용

  -- Online DDL 예시
  ALTER TABLE orders
      ADD COLUMN memo TEXT,
      ALGORITHM = INPLACE,
      LOCK = NONE;  -- 잠금 없이 DDL 수행
  
  -- 어떤 DDL이 INPLACE 가능한지 확인:
  -- EXPLAIN ALTER TABLE orders ADD COLUMN memo TEXT;
  -- (MySQL 8.0.27+에서 지원)

  지원하는 INPLACE DDL:
    ✅ ADD/DROP COLUMN (일부)
    ✅ ADD INDEX (Secondary)
    ✅ RENAME COLUMN
    ✅ RENAME INDEX
  
  COPY가 필요한 DDL:
    ❌ 컬럼 타입 변경 (INT → BIGINT 등)
    ❌ PRIMARY KEY 변경
    ❌ CHARACTER SET 변경

pt-online-schema-change (대안):
  Percona Toolkit의 도구
  COPY 방식도 무중단으로 수행 가능
  트리거로 기존 테이블 변경사항을 새 테이블에 동기화
  완료 후 테이블명 교체 (RENAME)
```

### 5. Tablespace 파일 손상 감지

```
InnoDB의 Checksum 기반 손상 감지:

각 Page의 File Trailer (마지막 8 bytes):
  FIL_PAGE_END_LSN_OLD_CHKSUM (4 bytes): Checksum 값
  → 페이지 데이터로 계산한 해시값

페이지 읽기 시:
  ① 디스크에서 Page 읽음
  ② Checksum 재계산
  ③ 저장된 Checksum과 비교
  ④ 불일치 → "Corruption detected" 에러

-- Checksum 알고리즘 설정
SHOW VARIABLES LIKE 'innodb_checksum_algorithm';
-- 기본값: crc32 (MySQL 5.6.3+)
-- 옵션: crc32, innodb, none, strict_crc32 등

-- 테이블 무결성 검사
CHECK TABLE orders;
-- 출력:
-- deep_dive.orders  check  status  OK

-- 손상된 Page 발견 시 MySQL 에러 로그:
-- InnoDB: Database page corruption on disk or a failed
-- file read of tablespace orders/orders page [page id: page_no=12345].

-- innochecksum으로 오프라인 검사:
# MySQL 서비스 중지 후
innochecksum --page-type-summary /var/lib/mysql/deep_dive/orders.ibd
```

---

## 💻 실전 실험

### 실험 1: Tablespace 파일 확인

```bash
# Docker 컨테이너 내 MySQL 데이터 디렉토리 확인
docker exec -it mysql ls -lh /var/lib/mysql/

# 출력:
# ibdata1         -- System Tablespace
# ib_logfile0     -- Redo Log 파일 0
# ib_logfile1     -- Redo Log 파일 1
# ib_buffer_pool  -- Buffer Pool Dump (있다면)
# #ib_16384_0.dblwr -- Double Write Buffer
# deep_dive/      -- 데이터베이스 디렉토리

docker exec -it mysql ls -lh /var/lib/mysql/deep_dive/
# orders.ibd      -- orders 테이블 Tablespace
# orders_XXX.sdi  -- 테이블 정의 (MySQL 8.0)
```

```sql
-- information_schema로 파일 정보 확인
SELECT
    file_name,
    file_type,
    ROUND(file_size / 1024 / 1024, 2) AS size_mb,
    ROUND(free_extents * 1024 * 1024 / 1024 / 1024, 2) AS free_mb
FROM information_schema.FILES
WHERE table_schema = 'deep_dive'
ORDER BY file_size DESC;
```

### 실험 2: 대량 DELETE 후 공간 회수 실험

```sql
-- orders 테이블 크기 확인 (삭제 전)
SELECT
    ROUND(data_length / 1024 / 1024, 2) AS data_mb_before
FROM information_schema.tables
WHERE table_schema = 'deep_dive' AND table_name = 'orders';

-- 절반 삭제
DELETE FROM orders WHERE id % 2 = 0;

-- 삭제 후 크기 확인 → 거의 변화 없음!
SELECT
    ROUND(data_length / 1024 / 1024, 2) AS data_mb_after_delete
FROM information_schema.tables
WHERE table_schema = 'deep_dive' AND table_name = 'orders';
-- 내부적으로 Page가 비워졌지만 파일 크기는 그대로
-- InnoDB는 이 빈 공간을 향후 INSERT에 재사용

-- OPTIMIZE TABLE으로 공간 회수 (주의: 테이블 전체 재빌드)
OPTIMIZE TABLE orders;
-- = ALTER TABLE orders ENGINE=InnoDB 와 동일
-- 테이블 재빌드 → 빈 공간 제거 → .ibd 파일 크기 감소
-- 단, 대용량 테이블에서는 오래 걸림 (Online DDL INPLACE 사용 권장)

SELECT
    ROUND(data_length / 1024 / 1024, 2) AS data_mb_after_optimize
FROM information_schema.tables
WHERE table_schema = 'deep_dive' AND table_name = 'orders';
-- 이제 실제로 줄어든 크기 반영
```

### 실험 3: Online DDL 실험

```sql
-- 큰 테이블에 컬럼 추가 (INPLACE vs COPY 시간 비교)
-- 먼저 실행 계획 확인 (MySQL 8.0.27+)
-- EXPLAIN FORMAT=TREE ALTER TABLE orders ADD COLUMN memo TEXT;

-- INPLACE 방식 (권장)
ALTER TABLE orders
    ADD COLUMN memo TEXT COMMENT '메모',
    ALGORITHM=INPLACE,
    LOCK=NONE;
-- Lock=NONE: 작업 중 DML 허용
-- 소요 시간과 디스크 사용량 관찰

-- 작업 진행 중 모니터링
SELECT
    stage,
    event_name,
    work_completed,
    work_estimated,
    ROUND(work_completed / work_estimated * 100, 1) AS pct
FROM performance_schema.events_stages_current
WHERE event_name LIKE '%alter%';

-- 컬럼 제거 (원복)
ALTER TABLE orders DROP COLUMN memo, ALGORITHM=INPLACE, LOCK=NONE;
```

### 실험 4: Undo Tablespace 모니터링

```sql
-- Undo Tablespace 크기 확인
SELECT
    space_name,
    ROUND(file_size / 1024 / 1024, 2) AS size_mb,
    ROUND(free_extents * 16384 / 1024 / 1024, 2) AS free_mb
FROM information_schema.FILES
WHERE file_type = 'UNDO LOG';

-- 오래 실행 중인 트랜잭션 찾기 (Undo Log 축적 원인)
SELECT
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS elapsed_secs,
    trx_rows_modified,
    trx_query
FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60  -- 1분 이상 트랜잭션
ORDER BY trx_started;

-- Undo Log 오버플로우 체크
SHOW STATUS LIKE 'Innodb_undo_tablespace%';
```

---

## 📊 성능 비교

```
파일 구성 방식별 운영 비교:

innodb_file_per_table = ON (권장):
  장점:
    ✅ DROP TABLE, TRUNCATE TABLE 시 즉시 공간 회수
    ✅ 테이블별 물리적 I/O 분리 가능 (다른 디스크로 이동)
    ✅ 테이블별 개별 백업/복원 (transport tablespace)
    ✅ 파일 크기로 테이블 크기 직관적 파악
  단점:
    ❌ 파일 수 = 테이블 수 → 수천 개 테이블이면 파일 수천 개
    ❌ OS의 파일 핸들 제한 주의 (ulimit -n)

innodb_file_per_table = OFF:
  장점:
    ✅ 파일 수 최소화 (ibdata1 하나)
  단점:
    ❌ 공간 회수 불가 (인스턴스 재생성 외 방법 없음)
    ❌ 테이블별 이동/개별 백업 불가
    ❌ ibdata1 무한 성장
  
  사용 권장 케이스: 없음 (레거시 환경 외)

Online DDL 방식 비교:
  ALGORITHM=INPLACE, LOCK=NONE:
    서비스 중 무중단 DDL
    추가 디스크: Row Log만큼 (수십~수백 MB)
    소요 시간: 데이터 크기에 비례 (단, DML 허용 상태)
  
  ALGORITHM=COPY:
    테이블 전체 복사 → 원본 크기만큼 추가 공간 필요
    잠금 발생 → 서비스 영향
    타입 변경 등 불가피한 경우에만 사용
```

---

## ⚖️ 트레이드오프

```
Tablespace 설계 결정사항:

File-Per-Table vs Shared:
  현재(MySQL 8.0): file_per_table=ON이 기본 및 권장
  → 특별한 이유 없으면 변경하지 말 것

Undo Tablespace 크기 관리:
  오래 실행되는 트랜잭션 → Undo Log 축적 → 디스크 부족
  
  예방:
    ① 트랜잭션은 짧게 유지 (JPA @Transactional 범위 최소화)
    ② 배치 처리 시 주기적 COMMIT
    ③ 슬로우 트랜잭션 모니터링 알림 설정
  
  innodb_max_undo_log_size: Undo Tablespace 최대 크기 (기본 1GB)
  innodb_undo_log_truncate = ON: 최대 크기 초과 시 자동 정리 시도

Temporary Tablespace:
  ibtmp1이 갑자기 수십 GB → 복잡한 쿼리의 임시 결과물
  
  원인 진단:
    SHOW STATUS LIKE 'Created_tmp_disk_tables';  -- 디스크 임시 테이블 수
    EXPLAIN으로 "Using filesort", "Using temporary" 확인
  
  해결:
    tmp_table_size / max_heap_table_size 증가 (메모리 임시 테이블 사용)
    쿼리 최적화로 정렬/그룹화 최소화
```

---

## 📌 핵심 정리

```
Tablespace 핵심:

종류:
  System Tablespace (ibdata1): Change Buffer, 레거시 메타데이터
  File-Per-Table (.ibd): 테이블마다 독립 파일 (권장, 기본값)
  Undo Tablespace (undo_00N): MVCC 이전 버전 저장
  Temporary Tablespace (ibtmp1): 정렬/임시 테이블

File-Per-Table = ON 반드시 사용:
  이유: DROP/TRUNCATE 시 즉시 공간 회수 가능
        OFF이면 ibdata1이 무한히 커지며 줄일 수 없음

DELETE vs TRUNCATE 공간 회수:
  DELETE: 페이지에서 Row 삭제 표시만 → 공간 재사용은 하지만 OS 반환 안 함
  TRUNCATE (file_per_table=ON): .ibd 재생성 → 즉시 OS 공간 반환
  대량 데이터 삭제 후 공간 회수: OPTIMIZE TABLE (또는 ALTER TABLE ENGINE=InnoDB)

Online DDL:
  ALGORITHM=INPLACE, LOCK=NONE → 서비스 중단 없이 스키마 변경
  ALGORITHM=COPY → 불가피한 경우만 (타입 변경 등)
  대용량 테이블 DDL은 pt-online-schema-change 검토

Undo Log 관리:
  오래 실행되는 트랜잭션 → Undo Log 무한 축적
  → JPA 트랜잭션 범위 최소화가 DB 디스크 관리에도 중요
  → 배치 처리 시 주기적 COMMIT으로 Purge 가능하게 유지
```

---

## 🤔 생각해볼 문제

**Q1.** `innodb_file_per_table = ON` 상태에서 1억 건의 오래된 로그 데이터를 삭제하고 싶다. `DELETE FROM logs WHERE created_at < '2022-01-01'` vs `CREATE TABLE logs_new SELECT * FROM logs WHERE created_at >= '2022-01-01'; RENAME TABLE logs TO logs_old, logs_new TO logs; DROP TABLE logs_old;` 두 방법의 디스크 공간, 서비스 영향, 실행 시간 차이를 비교하라.

<details>
<summary>해설 보기</summary>

**방법 1: DELETE**
- 실행 시간: 매우 느림 (1억 건 Row별 Undo Log 생성, 개별 삭제)
- 디스크 공간: 삭제 중 Undo Log로 임시 증가, 삭제 완료 후 Page 내부 빈 공간(fragmentation)이 남음
- 서비스 영향: 트랜잭션이 길어질수록 Lock contention 증가
- 최종 공간 회수: OPTIMIZE TABLE 추가로 필요 (또는 영구 fragmentation)

**방법 2: CREATE + RENAME (Ghost Table 패턴)**
- 실행 시간: `CREATE TABLE logs_new SELECT ...`가 더 빠를 수 있음 (INSERT 최적화)
- 디스크 공간: 새 테이블 생성 중 원본과 새 테이블이 동시에 존재 → 최대 2배
- 서비스 영향: RENAME TABLE은 메타데이터만 변경 (거의 즉시), 그 전까지는 원본 접근 가능
- 최종 공간 회수: `DROP TABLE logs_old` → .ibd 파일 삭제 → 즉시 회수

**권장**: 방법 2 (Ghost Table 패턴)이 대용량 데이터 정리에 훨씬 효율적입니다. 단, 새 테이블 생성 중 원본 크기만큼의 추가 공간이 필요합니다. 이를 자동화한 것이 `pt-archiver` 도구입니다.

</details>

---

**Q2.** MySQL 서버를 재시작하면 `ibtmp1`(Temporary Tablespace) 파일이 사라지고 새로 생성된다. 재시작 전 `ibtmp1`이 50GB였다면 어떤 작업이 원인이었을 가능성이 높고, 어떻게 예방할 수 있는가?

<details>
<summary>해설 보기</summary>

**가능한 원인들**:
1. **대용량 임시 테이블**: `GROUP BY`, `ORDER BY`, `DISTINCT`가 메모리 임시 테이블 크기(`tmp_table_size`)를 초과하면 디스크 임시 테이블로 전환
2. **복잡한 JOIN**: Hash Join에서 빌드 사이드가 너무 크면 디스크로 스필
3. **큰 쿼리의 중간 결과**: `SELECT ... INTO ...` 또는 서브쿼리가 대용량 중간 결과 생성

**진단 방법**:
```sql
SHOW STATUS LIKE 'Created_tmp_disk_tables';
-- EXPLAIN 으로 "Using filesort", "Using temporary" 확인
-- performance_schema.events_statements_history_long 에서 슬로우 쿼리 찾기
```

**예방**:
1. `tmp_table_size`와 `max_heap_table_size` 값 증가 (메모리 여유 있으면)
2. EXPLAIN으로 임시 테이블 유발 쿼리 찾아 인덱스 추가 또는 쿼리 재작성
3. `innodb_temp_data_file_path = ibtmp1:12M:autoextend:max:5G`로 최대 크기 제한
   (제한에 걸리면 쿼리 실패 → 서비스 장애보다 디스크 full 방지 목적으로 설정)

</details>

---

**Q3.** `ALTER TABLE orders MODIFY COLUMN amount DECIMAL(10,2) NOT NULL DEFAULT 0`을 100GB 테이블에 실행하려고 한다. Online DDL(INPLACE)로 가능한가? 가능하지 않다면 무중단으로 스키마를 변경하는 방법은?

<details>
<summary>해설 보기</summary>

**INPLACE 불가**: 컬럼 데이터 타입 변경이 아닌 DEFAULT 값 변경만이라면 MySQL 8.0에서 INPLACE 가능합니다. 그러나 타입 자체를 변경하는 경우(예: `INT` → `BIGINT`, `VARCHAR(100)` → `VARCHAR(200)`)는 데이터 재포맷이 필요하므로 COPY 방식만 가능합니다.

**DECIMAL(10,2) → DECIMAL(10,2)는 타입 변경 없음**: DEFAULT 추가만이라면 INPLACE로 가능합니다.

타입을 실제로 변경해야 하는 경우 무중단 방법:

**pt-online-schema-change 사용**:
```bash
pt-online-schema-change \
  --alter "MODIFY COLUMN amount DECIMAL(12,4) NOT NULL DEFAULT 0" \
  --execute \
  D=deep_dive,t=orders
```
내부 동작:
1. 새 테이블 `_orders_new` 생성 (새 스키마)
2. 트리거로 원본 테이블 변경 사항을 새 테이블에 동기화
3. 원본 데이터를 청크 단위로 새 테이블에 복사
4. 완료 후 `RENAME TABLE`으로 교체
5. 원본 테이블 삭제

100GB 테이블이어도 서비스 중단 없이 처리 가능합니다. (단, 추가 디스크 100GB 필요)

</details>

---

<div align="center">

**[⬅️ 이전: Row Format](./03-row-format.md)** | **[홈으로 🏠](../README.md)** | **[다음: WAL — Write-Ahead Logging ➡️](./05-wal-write-ahead-logging.md)**

</div>
