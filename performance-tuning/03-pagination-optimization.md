# 페이징 최적화 — OFFSET의 함정과 커서 기반 페이징

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `LIMIT 10 OFFSET 100000`이 왜 100010행을 읽어야 하는가?
- OFFSET 기반과 커서 기반 페이징의 실행계획(EXPLAIN)은 어떻게 다른가?
- 커서 기반 페이징이 "B-Tree를 재시작한다"는 것의 정확한 의미는?
- `WHERE id > last_id LIMIT 10`이 항상 옳은 커서 패턴인가? 예외는?
- 정렬 기준이 `id`가 아닌 `created_at`일 때 커서 기반 페이징을 구현하는 방법은?
- JPA `Slice`와 `Page`의 내부 차이와 커서 기반 페이징과의 관계는?

---

## 🔍 왜 이 개념이 중요한가

### "게시글 마지막 페이지 클릭하면 왜 이렇게 느리지?"

```
OFFSET 기반 페이징의 비밀:
  LIMIT 10 OFFSET 1000000
  
  MySQL 내부 처리:
    인덱스 순서대로 1,000,010 Row를 읽음
    1,000,000 Row를 버림
    나머지 10 Row를 반환
  
  비용:
    O(OFFSET) = 오프셋이 클수록 선형 증가
    OFFSET 1000: 빠름 (1010 Row 읽기)
    OFFSET 100000: 느림 (100010 Row 읽기)
    OFFSET 1000000: 매우 느림 (1000010 Row 읽기)

실제 측정 (100만 건 테이블):
  LIMIT 10 OFFSET 0:        0.001s
  LIMIT 10 OFFSET 10000:    0.05s
  LIMIT 10 OFFSET 100000:   0.5s
  LIMIT 10 OFFSET 1000000:  5s
  
  → 오프셋이 1000배 늘면 시간도 거의 1000배

왜 이것이 문제인가:
  무한 스크롤 UI: 사용자가 스크롤할수록 OFFSET 증가
  관리자 페이지: 마지막 페이지로 이동 = 최대 OFFSET
  배치 처리: OFFSET으로 전체 데이터 순회 = O(N²) 비용
```

---

## 😱 잘못된 이해

### Before: "LIMIT이 있으니까 빠르다"

```sql
-- 잘못된 믿음:
-- "LIMIT 10이면 10개만 읽으니까 빠르다"

SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 10 OFFSET 100000;

-- 기대: 10행만 읽음 → 빠름
-- 실제: 100010행을 created_at 기준으로 정렬 후 읽기
--       → 앞의 100000행 버리기 → 마지막 10행 반환

-- EXPLAIN 결과:
-- type: index     ← 인덱스 사용 (OK처럼 보임)
-- key: idx_created_at
-- rows: 100010    ← 10이 아니라 100010을 예측!
-- Extra: Backward index scan

-- 잘못된 최적화:
-- "인덱스가 있으니까 괜찮겠지" → 인덱스로 정렬은 해결
-- 하지만 OFFSET만큼 행을 읽고 버리는 것은 해결 안 됨
-- → 인덱스가 있어도 OFFSET이 크면 느림

-- 또 다른 잘못된 믿음:
-- "COUNT(*)로 전체 건수 조회 후 OFFSET 계산하면 된다"
-- → COUNT(*) 자체도 100만 건이면 느릴 수 있음
-- → 페이지가 많을수록 마지막 OFFSET이 커짐 → 느려짐
```

---

## ✨ 올바른 이해

### After: OFFSET은 버리는 행 수, 커서는 탐색 시작점을 직접 지정

```
OFFSET 기반 (비효율):
  "처음부터 N번째까지 읽고, 그것을 다 버려라"
  
  B-Tree 탐색:
    인덱스 Leaf Node의 첫 번째 Entry부터 탐색 시작
    OFFSET까지 모든 Entry를 순서대로 거쳐감 (버리더라도)
    OFFSET 위치에서 LIMIT개 반환

커서 기반 (효율):
  "id > last_id인 첫 번째 위치부터 읽어라"
  
  B-Tree 탐색:
    WHERE id > 100000 → B-Tree에서 100000 이후 위치로 직접 이동
    (B-Tree 탐색: O(log N))
    해당 위치에서 LIMIT개 반환
    → OFFSET 0으로 읽는 것과 동일한 속도!

핵심 차이:
  OFFSET: O(OFFSET + LIMIT) = OFFSET이 크면 느림
  커서:   O(log N + LIMIT) = 항상 일정한 속도!

제약사항:
  커서 기반은 이전/다음 탐색만 가능 (특정 페이지 번호 이동 불가)
  → "이전 / 다음" 스크롤 UI에 적합
  → "1, 2, 3 ... 100 페이지" UI에는 OFFSET이 필요하지만
     행 수가 많지 않은 경우에만 허용 (관리자 기능 등)
```

---

## 🔬 내부 동작 원리

### 1. OFFSET 기반 페이징의 실제 처리 경로

```sql
-- 테스트: 100만 건 posts 테이블
-- id: BIGINT PK, created_at: DATETIME, INDEX idx_created (created_at)

-- OFFSET 기반 페이징
EXPLAIN ANALYZE
SELECT id, title, created_at
FROM posts
ORDER BY created_at DESC
LIMIT 10 OFFSET 100000\G

-- 출력 분석:
-- -> Limit/Offset: 10/100000 row(s)
--    (actual rows=10 loops=1)
--    -> Index scan on posts using idx_created (reverse)
--       (actual rows=100010 loops=1)   ← 100010행 읽음!
-- 
-- cost: ~100010 × index_lookup_cost
-- 실행 시간: created_at 인덱스 있어도 O(OFFSET)

-- 왜 인덱스가 있어도 OFFSET만큼 읽어야 하는가:
-- 인덱스 Leaf Node는 연속된 데이터
-- [created_at=t1, pk=1] [t2, pk=2] ... [t100010, pk=100010] ...
-- OFFSET 100000 = 앞의 100000 Entry를 순서대로 읽고 건너뜀
-- 건너뛰는 것도 읽어야 함 (어디서 100000번째인지 알아야 하므로)

-- Covering Index가 도움이 되는 이유:
-- 인덱스만으로 결과 구성 가능 → Clustered Index 접근 없음
-- created_at + id + title이 모두 인덱스에 있으면 빠름
-- 하지만 OFFSET만큼 읽는 것은 여전히 해결 안 됨

-- 지연 JOIN으로 OFFSET 비용 최소화 (절충안):
SELECT p.*
FROM posts p
INNER JOIN (
    SELECT id FROM posts
    ORDER BY created_at DESC
    LIMIT 10 OFFSET 100000  -- 인덱스 전용 서브쿼리 (빠름)
) tmp USING(id);
-- 서브쿼리: 인덱스만으로 ID 목록 추출 (Covering Index)
-- 외부 쿼리: 10개 ID로 나머지 컬럼 조회
-- → 전체를 Clustered Index로 읽는 것보다 빠름
-- → 하지만 여전히 OFFSET만큼 인덱스를 읽는 비용은 존재
```

### 2. 커서 기반 페이징 — B-Tree 직접 탐색

```sql
-- 커서 기반 페이징:
-- 이전 페이지의 마지막 id = 100000

EXPLAIN ANALYZE
SELECT id, title, created_at
FROM posts
WHERE id > 100000        -- ← 커서: 이 위치부터 시작
ORDER BY id
LIMIT 10\G

-- 출력:
-- -> Limit: 10 row(s)
--    (actual rows=10 loops=1)
--    -> Index range scan on posts using PRIMARY (id > 100000)
--       (actual rows=10 loops=1)   ← 정확히 10행만 읽음!
--
-- 처리 경로:
-- B-Tree에서 id > 100000인 첫 번째 Entry로 직접 이동 (O(log N))
-- 그 위치에서 순서대로 10개 읽기
-- → OFFSET 0으로 읽는 것과 동일한 속도!
-- → 100만 번째 페이지나 1번째 페이지나 같은 속도

-- 커서 기반의 실행계획:
-- type: range
-- key: PRIMARY
-- rows: 10 (예측) ← OFFSET 방식의 100010과 완전히 다름!

-- id vs created_at 커서:

-- id 커서 (단순):
WHERE id > last_id ORDER BY id LIMIT 10
-- 장점: id는 유니크 → 커서 명확
-- 단점: 최신순 정렬이 created_at 기준이면 id만으로는 불완전

-- created_at 커서 (복합):
WHERE (created_at, id) < (last_created_at, last_id)
ORDER BY created_at DESC, id DESC
LIMIT 10
-- 이유: created_at이 동일한 Row가 있을 수 있음 (중복 타임스탬프)
-- → (created_at, id) 복합 커서로 유일성 보장
-- 인덱스: (created_at, id) 복합 인덱스 필요
```

### 3. 정렬 기준별 커서 구현 패턴

```java
// 패턴 1: id 기준 커서 (가장 단순)
@Query("SELECT p FROM Post p WHERE p.id > :lastId ORDER BY p.id ASC")
List<Post> findNext(@Param("lastId") Long lastId, Pageable pageable);
// 사용:
Long lastId = 0L;  // 초기값
List<Post> posts = repo.findNext(lastId, PageRequest.of(0, 10));
lastId = posts.get(posts.size() - 1).getId();  // 다음 커서
// ↑ 클라이언트가 이 lastId를 다음 요청에 포함

// 패턴 2: created_at 기준 커서 (타임스탬프 중복 대비)
@Query("""
    SELECT p FROM Post p
    WHERE (p.createdAt < :lastCreatedAt)
       OR (p.createdAt = :lastCreatedAt AND p.id < :lastId)
    ORDER BY p.createdAt DESC, p.id DESC
    """)
List<Post> findNextByCreatedAt(
    @Param("lastCreatedAt") LocalDateTime lastCreatedAt,
    @Param("lastId") Long lastId,
    Pageable pageable);
// 인덱스: (created_at DESC, id DESC) 복합 인덱스 필요

// 패턴 3: Spring Data의 Slice 활용
Slice<Post> findByIdGreaterThan(Long lastId, Pageable pageable);
// Slice: 다음 페이지 존재 여부만 (COUNT 쿼리 없음)
// Page: 전체 건수 포함 (COUNT 쿼리 추가)
// 커서 기반에서는 Slice가 적합

// 패턴 4: 커서 토큰 응답 API 설계
@GetMapping("/posts")
public CursorResponse<PostDto> getPosts(
    @RequestParam(required = false) String cursor) {  // Base64 인코딩된 커서
    
    CursorInfo cursorInfo = decodeCursor(cursor);
    List<Post> posts = repo.findNext(cursorInfo.getLastId(), PageRequest.of(0, 11));
    // 11개 요청 → 11개면 다음 페이지 있음, 10개면 마지막 페이지
    
    boolean hasNext = posts.size() > 10;
    if (hasNext) posts = posts.subList(0, 10);
    
    String nextCursor = hasNext ? encodeCursor(posts.get(9).getId()) : null;
    return new CursorResponse<>(toDtos(posts), nextCursor, hasNext);
}
```

### 4. OFFSET 기반 페이징이 필요한 경우 최적화

```sql
-- 관리자 페이지처럼 페이지 번호 이동이 필요한 경우:

-- 지연 JOIN (Deferred Join / Late Row Lookup):
-- 서브쿼리로 ID만 먼저 추출 → 나머지 컬럼은 JOIN

-- 일반 OFFSET (느림):
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 10 OFFSET 100000;
-- → 100010건의 모든 컬럼 읽기

-- 지연 JOIN (빠름):
SELECT p.*
FROM posts p
INNER JOIN (
    SELECT id
    FROM posts
    ORDER BY created_at DESC
    LIMIT 10 OFFSET 100000
    -- ↑ 커버링 인덱스로 ID만 추출 (created_at, id 인덱스 사용)
) sub ON p.id = sub.id;
-- → 서브쿼리: 인덱스만으로 ID 10개 추출 (Clustered Index 접근 없음)
-- → 외부 JOIN: PK로 10개만 추출 (매우 빠름)
-- → 약 2~5배 성능 향상 (OFFSET이 클수록 효과 큼)

-- 성능 비교 (100만 건, OFFSET 100000):
-- 일반: 1.2s
-- 지연 JOIN: 0.35s
-- 커버링 인덱스 + 지연 JOIN: 0.15s

-- 추가 최적화: 전체 건수 캐싱
-- COUNT(*) FROM posts → 1만건 이상이면 느릴 수 있음
-- 해결: Redis에 COUNT 캐싱 (1분 TTL)
-- 또는: 정확한 총 건수 포기 → "약 N건" 표시

-- 전체 건수 근사치 (빠름):
SELECT table_rows
FROM information_schema.tables
WHERE table_name = 'posts' AND table_schema = DATABASE();
-- 통계 기반 근사치 → ANALYZE TABLE 후 더 정확
-- 정확하지 않지만 "총 N만 건의 게시글" UI에는 충분
```

### 5. 배치 처리에서의 페이징 최적화

```java
// 전체 데이터 배치 처리 시 OFFSET 절대 금지:

// 나쁜 패턴 (O(N²)):
int page = 0;
List<Order> batch;
do {
    batch = orderRepo.findAll(PageRequest.of(page++, 1000));
    // 1페이지: OFFSET 0 → 1000건 읽기
    // 2페이지: OFFSET 1000 → 2000건 읽기
    // ...
    // 1000페이지: OFFSET 999000 → 1000000건 읽기!
    // 총 I/O: 1000 + 2000 + ... + 1000000 = O(N²)
    processBatch(batch);
} while (!batch.isEmpty());

// 좋은 패턴 (O(N)):
Long lastId = 0L;
List<Order> batch;
do {
    batch = orderRepo.findByIdGreaterThan(lastId, PageRequest.of(0, 1000));
    // 항상 OFFSET 0 → 항상 1000건만 읽기
    // 1회: WHERE id > 0 LIMIT 1000 → 1000건
    // 2회: WHERE id > 1000 LIMIT 1000 → 1000건
    // N회: WHERE id > N*1000 LIMIT 1000 → 1000건
    // 총 I/O: 1000 × N회 = O(N)
    if (!batch.isEmpty()) lastId = batch.get(batch.size() - 1).getId();
    processBatch(batch);
} while (!batch.isEmpty());

// Spring Batch ItemReader에서:
JpaCursorItemReaderBuilder<Order>()
    .name("orderReader")
    .entityManagerFactory(emf)
    .queryString("SELECT o FROM Order o WHERE o.status = 'PENDING' ORDER BY o.id")
    // ← 내부적으로 커서 기반 읽기
    .build();
```

---

## 💻 실전 실험

### 실험 1: OFFSET vs 커서 기반 실행 계획 비교

```sql
-- 테스트 데이터 생성
CREATE TABLE pagination_test (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200),
    created_at DATETIME,
    INDEX idx_created(created_at)
) ENGINE=InnoDB;

-- 100만 건 삽입 (실험용)
INSERT INTO pagination_test (title, created_at)
SELECT CONCAT('Post ', seq),
       DATE_ADD('2020-01-01', INTERVAL seq SECOND)
FROM (SELECT @rownum := @rownum + 1 AS seq FROM information_schema.columns
      CROSS JOIN (SELECT @rownum := 0) r
      LIMIT 1000000) t;

-- OFFSET 기반 비교
EXPLAIN ANALYZE SELECT id, title FROM pagination_test
ORDER BY created_at LIMIT 10 OFFSET 0\G
-- actual rows=10, 빠름

EXPLAIN ANALYZE SELECT id, title FROM pagination_test
ORDER BY created_at LIMIT 10 OFFSET 500000\G
-- actual rows=500010, 매우 느림!

-- 커서 기반
EXPLAIN ANALYZE SELECT id, title FROM pagination_test
WHERE id > 500000 ORDER BY id LIMIT 10\G
-- actual rows=10, 빠름! (OFFSET 0과 동일)
```

### 실험 2: 지연 JOIN 성능 비교

```sql
-- 직접 OFFSET:
EXPLAIN ANALYZE
SELECT id, title, created_at FROM pagination_test
ORDER BY created_at LIMIT 10 OFFSET 500000;

-- 지연 JOIN:
EXPLAIN ANALYZE
SELECT p.id, p.title, p.created_at
FROM pagination_test p
INNER JOIN (
    SELECT id FROM pagination_test
    ORDER BY created_at LIMIT 10 OFFSET 500000
) sub ON p.id = sub.id;

-- 실행 시간 비교:
-- 직접 OFFSET: ~1.5s
-- 지연 JOIN: ~0.4s
-- 차이: idx_created (created_at, id 포함)가 있으면 서브쿼리가 Covering Index 사용
```

### 실험 3: 복합 커서 (타임스탬프 중복 대비)

```sql
-- 동일 created_at이 있는 경우 id 추가로 유일성 보장
-- 커서: (created_at='2024-01-15 10:00:00', id=12345)

-- 다음 페이지:
SELECT id, title, created_at
FROM pagination_test
WHERE (created_at < '2024-01-15 10:00:00')
   OR (created_at = '2024-01-15 10:00:00' AND id < 12345)
ORDER BY created_at DESC, id DESC
LIMIT 10;

-- EXPLAIN으로 인덱스 사용 확인:
EXPLAIN SELECT id, title, created_at
FROM pagination_test
WHERE (created_at < '2024-01-15 10:00:00')
   OR (created_at = '2024-01-15 10:00:00' AND id < 12345)
ORDER BY created_at DESC, id DESC
LIMIT 10;
-- type: range, key: idx_created_id (복합 인덱스 필요)
```

---

## 📊 성능 비교

```
OFFSET vs 커서 기반 (100만 건 posts 테이블):

OFFSET 기반 (LIMIT 10):
  OFFSET 0:       0.001s  (10행 읽기)
  OFFSET 10000:   0.05s   (10010행 읽기)
  OFFSET 100000:  0.5s    (100010행 읽기)
  OFFSET 500000:  2.5s    (500010행 읽기)
  OFFSET 1000000: 5.0s    (1000010행 읽기)
  
  특성: O(OFFSET) → 선형 증가

커서 기반 (WHERE id > last_id LIMIT 10):
  id=0:       0.001s  (10행 읽기)
  id=10000:   0.001s  (10행 읽기)
  id=100000:  0.001s  (10행 읽기)
  id=500000:  0.001s  (10행 읽기)
  id=1000000: 0.001s  (10행 읽기)
  
  특성: O(log N + 10) → 항상 일정!

지연 JOIN (OFFSET 100000):
  직접 OFFSET: 0.5s
  지연 JOIN:   0.15s
  → 3배 향상 (Covering Index 활용)

배치 처리 비교 (100만 건, 1000건씩):
  OFFSET 방식: O(N²) = 총 5억 행 읽기
  커서 방식:   O(N)  = 총 100만 행 읽기
  차이: 500배!
```

---

## ⚖️ 트레이드오프

```
OFFSET 기반:
  ✅ 특정 페이지 번호로 바로 이동 가능 (1, 2, 3 ... N 페이지)
  ✅ 구현 단순 (Spring Data JPA의 Pageable)
  ✅ 전체 건수 파악 용이 (COUNT(*) 쿼리)
  ❌ OFFSET이 클수록 선형 성능 저하
  ❌ 데이터 삽입/삭제 시 페이지 경계 불안정
     (5번 페이지 보다가 새 데이터 추가되면 중복/누락)
  적합: 전체 페이지 수가 적음 (< 100페이지), 관리자 기능

커서 기반:
  ✅ O(log N + LIMIT) 항상 일정한 성능
  ✅ 데이터 삽입/삭제에 안정적 (커서 위치 고정)
  ✅ 무한 스크롤 UI에 이상적
  ❌ 특정 페이지 번호로 이동 불가 (이전/다음만)
  ❌ 커서 관리 필요 (클라이언트가 커서 보관)
  ❌ 정렬 기준이 유니크하지 않으면 복합 커서 필요
  ❌ 전체 건수 파악 어려움 (COUNT 쿼리 별도)
  적합: 무한 스크롤, SNS 피드, 배치 처리

선택 기준:
  UI가 "이전/다음" 또는 무한 스크롤: 커서 기반
  UI가 "1, 2, 3 페이지 번호": OFFSET (단, 페이지 수 제한)
  배치 처리: 반드시 커서 기반
  데이터 실시간 삽입 많음: 커서 기반 (중복/누락 방지)
  
  페이지 수 제한 권장 (OFFSET 사용 시):
    "최대 100페이지까지만 조회 가능" 정책 추가
    → OFFSET 최대 = 100 × pageSize (제한됨)
```

---

## 📌 핵심 정리

```
페이징 최적화 핵심:

OFFSET 문제:
  LIMIT N OFFSET M = M+N행을 읽고 M행 버림
  비용: O(M) → OFFSET 클수록 선형 증가
  배치 처리에서 O(N²) 위험

커서 기반:
  WHERE id > last_id ORDER BY id LIMIT N
  B-Tree에서 last_id 위치로 직접 이동 → N행만 읽음
  비용: O(log N + LIMIT) → 항상 일정!

타임스탬프 커서:
  created_at 중복 가능 → (created_at, id) 복합 커서
  WHERE (created_at < ?) OR (created_at = ? AND id < ?)
  복합 인덱스 (created_at, id) 필요

지연 JOIN (OFFSET 절충안):
  서브쿼리로 ID만 추출 (Covering Index)
  외부 쿼리로 나머지 컬럼 조회
  → OFFSET 비용 줄이되 페이지 번호 유지

Spring 구현:
  Pageable: OFFSET 기반 (Page = COUNT 포함, Slice = 다음 여부만)
  커서: @Query + WHERE id > :lastId + PageRequest.of(0, N)
  배치: JpaCursorItemReader (커서 기반 자동)

선택:
  무한 스크롤 / 배치: 커서 기반 (필수)
  페이지 번호 UI + 소규모: OFFSET + 지연 JOIN
  전체 순회 배치: 커서 기반 (OFFSET 금지)
```

---

## 🤔 생각해볼 문제

**Q1.** SNS 피드에서 "새 게시글이 추가됐을 때 OFFSET 기반 페이징이 중복/누락을 일으키는 시나리오"를 구체적으로 설명하고, 커서 기반이 이를 방지하는 원리를 설명하라.

<details>
<summary>해설 보기</summary>

**OFFSET 기반의 중복 문제**:

시간 순서:
1. 사용자가 1페이지 조회: `LIMIT 10 OFFSET 0` → 게시글 [10, 9, 8, 7, 6, 5, 4, 3, 2, 1] 반환
2. 다른 사용자가 새 게시글(id=11) 작성
3. 사용자가 2페이지 조회: `LIMIT 10 OFFSET 10` → 게시글 [1, ...] 이 반환

*문제*: `id=1`이 1페이지와 2페이지 모두 포함됨. 새 게시글이 맨 앞에 추가되면서 기존 게시글들이 한 칸씩 밀렸기 때문입니다.

반대로 **누락 문제**: 1페이지 조회 후 게시글이 삭제되면, 다음 페이지에서 그만큼 앞당겨져 사용자가 보지 못한 게시글이 건너뛰어집니다.

**커서 기반의 방지 원리**:

1. 사용자가 1페이지 조회: `WHERE id > 0 ORDER BY id DESC LIMIT 10` → [10...1], 커서=1
2. 새 게시글(id=11) 작성
3. 사용자가 2페이지 조회: `WHERE id < 1 ORDER BY id DESC LIMIT 10` → [... id < 1]

커서(id=1)는 변하지 않습니다. 새 게시글이 추가되어도 "id < 1인 게시글"의 집합은 동일합니다. 따라서 2페이지를 요청했을 때 정확히 커서 이전의 게시글만 반환됩니다.

단, 새 게시글(id=11)은 1페이지를 새로고침하지 않으면 보이지 않습니다. 이것은 커서 기반의 의도된 동작입니다 (세션 일관성).

</details>

---

**Q2.** 정렬 기준이 `amount` (DECIMAL, 중복 가능)이고 페이지당 10건을 커서 기반으로 구현한다면 어떻게 해야 하는가? SQL과 인덱스를 함께 설명하라.

<details>
<summary>해설 보기</summary>

`amount`는 중복될 수 있으므로 단독으로는 커서 역할을 할 수 없습니다. `(amount, id)` 복합 커서가 필요합니다.

**내림차순 정렬 구현**:
```sql
-- 첫 페이지:
SELECT id, amount, title
FROM orders
ORDER BY amount DESC, id DESC
LIMIT 10;
-- 결과 마지막: amount=5000, id=42

-- 다음 페이지 (커서: amount=5000, id=42):
SELECT id, amount, title
FROM orders
WHERE (amount < 5000)                  -- amount가 더 작은 것
   OR (amount = 5000 AND id < 42)      -- 같은 amount에서 id가 더 작은 것
ORDER BY amount DESC, id DESC
LIMIT 10;
```

**필요한 인덱스**:
```sql
CREATE INDEX idx_amount_id ON orders(amount DESC, id DESC);
-- 또는:
CREATE INDEX idx_amount_id ON orders(amount, id);
-- → EXPLAIN에서 type: range, key: idx_amount_id 확인
```

**주의사항**:
- `OR` 조건이 인덱스를 효율적으로 사용하려면 `(amount, id)` 복합 인덱스 필요
- `OR` 대신 `UNION` 방식도 가능하지만 복잡도 증가
- MySQL 8.0에서는 위 OR 패턴이 인덱스 최적화 가능

**API 설계**:
```java
// 커서를 Base64 인코딩:
String cursor = Base64.encode(amount + "," + id);

// 다음 요청:
CursorInfo info = Base64.decode(cursor);
List<Order> next = repo.findNextByAmount(info.amount, info.id, PageRequest.of(0, 10));
```

</details>

---

**Q3.** Spring Batch에서 `JpaPagingItemReader`와 `JpaCursorItemReader`의 내부 구현 차이를 설명하고, 대용량 데이터 처리 시 어느 것을 선택해야 하는지 이유를 설명하라.

<details>
<summary>해설 보기</summary>

**JpaPagingItemReader (OFFSET 기반)**:
```java
// 내부 구현 (단순화):
int page = 0;
while (true) {
    List<T> results = entityManager
        .createQuery(query)
        .setFirstResult(page * pageSize)  // OFFSET 계산
        .setMaxResults(pageSize)
        .getResultList();
    if (results.isEmpty()) break;
    process(results);
    page++;
}
// 문제: page가 커질수록 setFirstResult(큰 OFFSET) → O(N²) 비용
```

**JpaCursorItemReader (커서 기반, Spring Batch 4.3+)**:
```java
// 내부 구현 (단순화):
ScrollableResults<T> cursor = session
    .createQuery(query)
    .scroll(ScrollMode.FORWARD_ONLY);
// → DB에서 결과를 스트리밍으로 가져옴
// → 한 번에 전체를 메모리에 올리지 않음
// → 내부적으로 DB 커서 사용 (OFFSET 없음)
while (cursor.next()) {
    process(cursor.get());
}
```

**선택 기준**:
- **대용량(>100만 건)**: `JpaCursorItemReader` 필수
  - O(N) 비용, 메모리 효율적 (스트리밍)
  - 단, 트랜잭션이 처리 전체 동안 열려 있어야 함 (DB 커서 유지)
  - 장시간 실행 시 Lock 문제 가능

- **소~중규모(<10만 건)**: `JpaPagingItemReader` 허용
  - 페이지 단위 독립 트랜잭션 (각 페이지 후 COMMIT)
  - 재시작 가능성 있음 (page 번호로 재개)
  - Multi-thread 처리 가능

- **실무 권장**: 대용량 배치는 `WHERE id > lastId ORDER BY id LIMIT batchSize` 쿼리를 직접 사용하거나, `JpaCursorItemReader`를 사용합니다. `JpaPagingItemReader`는 OFFSET 문제로 대용량에서는 사용하지 않습니다.

</details>

---

<div align="center">

**[⬅️ N+1 문제](./02-n-plus-one-db-perspective.md)** | **[홈으로 🏠](../README.md)** | **[다음: 대용량 최적화 ➡️](./04-large-table-optimization.md)**

</div>
