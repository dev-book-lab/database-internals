<div align="center">

# 🗄️ Database Internals Deep Dive

**"SQL을 쓰는 것과, DB가 내부에서 무슨 일을 하는지 아는 것은 다르다"**

<br/>

> *"인덱스를 걸었는데 쿼리가 느리고, 격리 수준을 설정했는데 Phantom Read가 발생하고, 데드락 로그를 보고도 원인을 모른다면 — DB를 블랙박스로 두고 있는 것이다"*

InnoDB Buffer Pool의 Page 교체 원리부터 B-Tree 인덱스가 Binary Tree가 아닌 이유, MVCC가 Undo Log로 이전 버전을 보관하는 방식, Gap Lock이 Phantom Read를 막는 메커니즘까지  
**왜 이렇게 설계됐는가** 라는 질문으로 MySQL InnoDB 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=flat-square&logo=mysql&logoColor=white)](https://dev.mysql.com/doc/)
[![InnoDB](https://img.shields.io/badge/InnoDB-Storage_Engine-orange?style=flat-square&logo=mysql&logoColor=white)](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)
[![Docs](https://img.shields.io/badge/Docs-40개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

DB에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "인덱스를 걸면 빠릅니다" | B-Tree가 Binary Tree 대신 선택된 이유, 디스크 I/O 최소화를 위한 높은 Fan-out 구조, Clustered Index가 PK 순서로 데이터를 물리적으로 배치하는 원리 |
| "EXPLAIN으로 실행계획을 확인하세요" | Cost-Based Optimizer가 통계 정보(rows, cardinality)를 기반으로 인덱스 선택 비용을 계산하는 방법, `type: ALL` vs `type: ref` 차이의 실제 의미 |
| "격리 수준을 REPEATABLE_READ로 설정하세요" | MVCC가 Undo Log로 스냅샷 시점의 이전 버전을 보관하는 방식, `trx_id`와 `Read View`로 어느 버전을 읽을지 결정하는 원리 |
| "데드락이 발생하면 재시도하세요" | Next-Key Lock의 범위가 결정되는 조건, InnoDB 데드락 감지 알고리즘(Wait-for Graph), 데드락 로그에서 어느 트랜잭션이 어떤 락을 잡고 있었는지 분석하는 방법 |
| "N+1 문제는 Fetch Join으로 해결합니다" | N+1이 발생할 때 Slow Query Log에 어떻게 기록되는지, DB 관점에서 수십 개의 독립 쿼리가 Connection Pool에 미치는 영향 |
| 이론 나열 | 실행 가능한 EXPLAIN 분석 + Slow Query Log 재현 + 락 확인 쿼리 + Docker Compose 실험 환경 + Spring Data JPA 연결 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Storage](https://img.shields.io/badge/🔹_Storage-Page·Block·Extent_구조-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./storage-and-file-structure/01-page-block-extent.md)
[![Index](https://img.shields.io/badge/🔹_Index-왜_B--Tree인가-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./index-internals/01-btree-why-not-binary-tree.md)
[![Query](https://img.shields.io/badge/🔹_Query-쿼리_실행_단계-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./query-execution-optimizer/01-query-execution-stages.md)
[![MVCC](https://img.shields.io/badge/🔹_Transaction-ACID_실제_구현-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./transaction-mvcc/01-acid-implementation.md)
[![Lock](https://img.shields.io/badge/🔹_Lock-락의_종류와_구조-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./lock-and-concurrency/01-lock-types.md)
[![Perf](https://img.shields.io/badge/🔹_Performance-Slow_Query_분석-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./performance-tuning/01-slow-query-analysis.md)
[![Repl](https://img.shields.io/badge/🔹_Replication-Binary_Log_원리-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./replication-high-availability/01-mysql-replication-binlog.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Storage & File Structure

> **핵심 질문:** MySQL은 데이터를 디스크에 어떻게 저장하고, 메모리에서 어떻게 관리하며, 장애 시 어떻게 복구하는가?

<details>
<summary><b>Page·Block 구조부터 WAL(Write-Ahead Logging)까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Page·Block·Extent — InnoDB 물리 저장 구조](./storage-and-file-structure/01-page-block-extent.md) | InnoDB가 데이터를 16KB Page 단위로 저장하는 이유, Page → Extent → Segment 계층 구조, 디스크 I/O가 Page 단위로 발생하는 원리와 Random I/O vs Sequential I/O 비용 차이 |
| [02. InnoDB Buffer Pool — 메모리와 디스크 사이 캐시 레이어](./storage-and-file-structure/02-innodb-buffer-pool.md) | Buffer Pool이 LRU 알고리즘으로 Page를 교체하는 방식, Young/Old 서브리스트 구조, `innodb_buffer_pool_size` 튜닝 기준, Dirty Page가 디스크에 플러시되는 시점 — JPA 쓰기 지연과의 연결 |
| [03. Row Format — 데이터가 Page 안에 저장되는 구조](./storage-and-file-structure/03-row-format.md) | COMPACT / DYNAMIC / COMPRESSED Row Format 비교, NULL 비트맵이 저장 공간을 절약하는 방식, VARCHAR와 TEXT의 오프-페이지 저장(External Storage) 조건, 실제 Row 크기 계산 방법 |
| [04. Tablespace와 파일 구성](./storage-and-file-structure/04-tablespace.md) | System Tablespace vs File-Per-Table Tablespace 차이, `.ibd` 파일 내부 구조, Undo Tablespace와 Temporary Tablespace의 역할, `innodb_file_per_table` 설정 선택 기준 |
| [05. WAL — Write-Ahead Logging과 장애 복구](./storage-and-file-structure/05-wal-write-ahead-logging.md) | WAL이 "디스크에 먼저 쓰지 않아도 되는 이유"를 가능하게 하는 원리, Redo Log로 커밋된 트랜잭션을 재현하는 Crash Recovery 과정, `innodb_flush_log_at_trx_commit` 설정과 내구성 트레이드오프 |

</details>

<br/>

### 🔹 Chapter 2: Index 완전 분해

> **핵심 질문:** 인덱스를 걸었는데 쿼리가 왜 느리고, Optimizer는 언제 인덱스를 타지 않으며, Covering Index는 왜 빠른가?

<details>
<summary><b>B-Tree 원리부터 인덱스 설계 전략까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. B-Tree 인덱스 — 왜 Binary Tree가 아닌 B-Tree인가](./index-internals/01-btree-why-not-binary-tree.md) | 디스크 I/O 횟수를 최소화하기 위해 Node 하나에 수백 개의 키를 담는 높은 Fan-out 구조, Binary Tree vs B-Tree 탐색 비용 비교, B+Tree에서 데이터가 Leaf Node에만 존재하는 이유, 범위 스캔이 Leaf 레벨 연결 리스트를 순회하는 방식 |
| [02. Clustered Index vs Non-Clustered Index](./index-internals/02-clustered-vs-non-clustered.md) | Clustered Index가 PK 순서로 데이터를 물리적으로 배치하는 원리, Secondary Index가 PK를 포인터로 저장하는 이유, Secondary Index로 조회 시 발생하는 이중 탐색(Double Lookup), PK 선택이 삽입 성능에 미치는 영향 — JPA `@Id` 전략과의 연결 |
| [03. Covering Index와 Index-Only Scan](./index-internals/03-covering-index-index-only-scan.md) | Extra 컬럼에 `Using index`가 표시되는 조건, Covering Index가 테이블 접근 없이 인덱스만으로 결과를 반환하는 원리, SELECT 컬럼에 따라 Covering 여부가 달라지는 이유, JPA `@Query`에서 Covering Index를 활용하는 패턴 |
| [04. Composite Index — 컬럼 순서가 왜 중요한가](./index-internals/04-composite-index-column-order.md) | 좌측 접두사 규칙(Leftmost Prefix Rule)이 B-Tree 구조에서 도출되는 원리, `(a, b, c)` 인덱스에서 `WHERE b = ?`가 인덱스를 타지 않는 이유, 카디널리티 순서 vs 쿼리 패턴 순서 선택 기준, EXPLAIN의 `key_len`으로 인덱스 사용 범위 확인하는 방법 |
| [05. Index Selectivity와 Cardinality](./index-internals/05-index-selectivity-cardinality.md) | Cardinality가 낮은 컬럼(`성별`, `상태`)에 인덱스를 걸어도 Full Scan보다 느릴 수 있는 이유, Optimizer가 통계 정보를 기반으로 인덱스 사용 여부를 결정하는 비용 모델, `ANALYZE TABLE`로 통계를 갱신해야 하는 시점 |
| [06. 인덱스를 타지 않는 쿼리 패턴](./index-internals/06-index-skip-patterns.md) | 함수 적용(`WHERE YEAR(created_at) = 2024`), 형변환(`WHERE user_id = '1'`), `LIKE '%keyword'`, `OR` 조건이 인덱스를 무력화하는 내부 원리, JPA에서 자주 발생하는 인덱스 미사용 패턴 |
| [07. 인덱스 설계 전략 — 언제 걸고 언제 걸지 않는가](./index-internals/07-index-design-strategy.md) | 인덱스 추가가 `INSERT` / `UPDATE` / `DELETE` 성능을 저하시키는 이유(B-Tree 재균형), 인덱스 유지 비용 vs 조회 이득의 트레이드오프 계산, 복합 인덱스 vs 단일 인덱스 선택 기준, 실무에서 인덱스를 제거해야 하는 신호 |

</details>

<br/>

### 🔹 Chapter 3: Query Execution & Optimizer

> **핵심 질문:** SQL이 실행되는 순간 Parse → Optimize → Execute 각 단계에서 무슨 일이 일어나고, Optimizer는 어떤 기준으로 실행계획을 선택하는가?

<details>
<summary><b>쿼리 실행 단계부터 힌트·강제 인덱스까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 쿼리 실행 단계 — Parse → Optimize → Execute](./query-execution-optimizer/01-query-execution-stages.md) | SQL 문자열이 Parse Tree로 변환되는 과정, Logical Plan을 Physical Plan으로 변환하는 Optimization 단계, `Handler_read_*` 상태 변수로 실제 실행 경로를 확인하는 방법 |
| [02. EXPLAIN과 EXPLAIN ANALYZE — 실행계획 읽는 법](./query-execution-optimizer/02-explain-analyze.md) | `type`, `key`, `rows`, `Extra` 컬럼의 의미를 InnoDB 내부 동작과 연결해서 해석하는 방법, `EXPLAIN ANALYZE`의 실제 실행 비용과 예측 비용 비교, `Using filesort` / `Using temporary` 가 발생하는 조건 |
| [03. Cost-Based Optimizer — 인덱스 선택 원리](./query-execution-optimizer/03-cost-based-optimizer.md) | Optimizer가 `rows × cost_per_row`로 각 실행계획의 비용을 계산하는 방식, `mysql.server_cost` / `mysql.engine_cost` 테이블의 비용 상수, 통계 정보(페이지 수, 카디널리티)가 부정확할 때 잘못된 실행계획이 선택되는 시나리오 |
| [04. Join 알고리즘 — Nested Loop, Hash Join, Sort Merge Join](./query-execution-optimizer/04-join-algorithms.md) | Nested Loop Join이 인덱스 없이 O(n²)이 되는 원리, MySQL 8.0에서 추가된 Hash Join의 동작 방식과 적합한 상황, Join 순서(Driving Table)가 성능에 미치는 영향, JPA Fetch Join이 실제로 생성하는 SQL과 실행계획 |
| [05. Statistics — Optimizer의 판단 근거](./query-execution-optimizer/05-statistics.md) | `information_schema.INNODB_TABLE_STATS`의 `n_rows`, `clustered_index_size` 등이 어떻게 수집되는지, 통계 자동 갱신 조건과 `innodb_stats_auto_recalc` 설정, 대량 데이터 변경 후 실행계획이 달라지는 원인 분석 |
| [06. 힌트와 강제 인덱스 사용](./query-execution-optimizer/06-hints-force-index.md) | `FORCE INDEX` / `USE INDEX` / `IGNORE INDEX` 문법과 Optimizer가 무시하는 경우, MySQL 8.0의 Optimizer Hint(`/*+ INDEX(...) */`) 방식, JPA에서 힌트를 적용하는 `@QueryHints` 패턴, 힌트가 필요 없도록 통계를 정비하는 근본적 해결책 |

</details>

<br/>

### 🔹 Chapter 4: Transaction & MVCC

> **핵심 질문:** `REPEATABLE_READ`와 `READ_COMMITTED`가 다르게 동작하는 이유는 무엇이고, MVCC는 락 없이 어떻게 일관된 읽기를 보장하는가?

<details>
<summary><b>ACID 구현 원리부터 Redo Log 내구성까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. ACID — 교과서 정의와 실제 구현 사이](./transaction-mvcc/01-acid-implementation.md) | Atomicity를 Undo Log로, Durability를 Redo Log로 구현하는 방식, Consistency가 DB 엔진이 아닌 애플리케이션 책임인 이유, Isolation이 격리 수준 선택 문제로 환원되는 구조 — Spring `@Transactional`이 이 중 어느 부분을 담당하는가 |
| [02. MVCC — 락 없이 읽기 일관성을 보장하는 원리](./transaction-mvcc/02-mvcc-internals.md) | 각 Row에 `DB_TRX_ID` / `DB_ROLL_PTR` 숨겨진 컬럼이 저장되는 방식, `Read View`가 생성 시점의 활성 트랜잭션 목록을 기록해 어느 버전을 읽을지 결정하는 원리, 읽기가 쓰기를 차단하지 않는 이유 |
| [03. Undo Log — 이전 버전을 보관하는 방법](./transaction-mvcc/03-undo-log.md) | Undo Log가 `DB_ROLL_PTR`로 연결된 버전 체인(Version Chain)을 형성하는 구조, 롤백 시 Undo Log를 역방향으로 적용하는 과정, 오래된 Read View가 남아있을 때 Undo Log가 삭제되지 않아 Tablespace가 팽창하는 문제 |
| [04. 트랜잭션 격리 수준 4가지 — 구현 원리까지](./transaction-mvcc/04-isolation-levels.md) | `READ_UNCOMMITTED` / `READ_COMMITTED` / `REPEATABLE_READ` / `SERIALIZABLE` 각각이 Read View를 언제 생성하는지의 차이, `READ_COMMITTED`에서 Non-Repeatable Read가 발생하는 구체적 원리, JPA `@Transactional(isolation = ...)` 설정과 실제 DB 격리 수준의 관계 |
| [05. Phantom Read — 발생 조건과 방지 방법](./transaction-mvcc/05-phantom-read.md) | Phantom Read가 같은 범위 쿼리에서 다른 행이 나타나는 이유, `REPEATABLE_READ`에서 Consistent Read는 막지만 Current Read(잠금 읽기)에서 발생하는 Phantom Read, Gap Lock이 삽입을 차단해 Phantom Read를 방지하는 메커니즘 |
| [06. Consistent Read vs Current Read](./transaction-mvcc/06-consistent-read-vs-current-read.md) | `SELECT`(스냅샷 읽기)와 `SELECT ... FOR UPDATE` / `SELECT ... FOR SHARE`(잠금 읽기)가 내부적으로 다른 경로를 타는 이유, Consistent Read는 Undo Log 버전을 읽지만 Current Read는 최신 버전을 읽고 락을 거는 차이 — JPA `@Lock` 어노테이션과의 연결 |
| [07. Redo Log와 트랜잭션 내구성](./transaction-mvcc/07-redo-log-durability.md) | COMMIT 시 Redo Log가 먼저 디스크에 flush되는 이유(WAL), Redo Log Buffer → Redo Log File 플러시 시점, `innodb_flush_log_at_trx_commit` 0/1/2 설정의 내구성 vs 성능 트레이드오프, Group Commit으로 IOPS를 줄이는 원리 |

</details>

<br/>

### 🔹 Chapter 5: Lock & Concurrency

> **핵심 질문:** InnoDB는 언제 Row Lock을, 언제 Gap Lock을 걸며, 데드락은 어떤 순서로 발생하고 InnoDB는 어떻게 감지하는가?

<details>
<summary><b>락의 종류부터 Optimistic vs Pessimistic 선택 기준까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 락의 종류 — Shared, Exclusive, Intention Lock](./lock-and-concurrency/01-lock-types.md) | S Lock(공유 잠금)과 X Lock(배타 잠금)의 호환 행렬, IS / IX Intention Lock이 테이블-레벨에서 Row Lock 의도를 표시하는 이유, `performance_schema.data_locks`로 현재 락 상태를 확인하는 방법 |
| [02. Row Lock vs Table Lock — InnoDB가 락을 거는 단위](./lock-and-concurrency/02-row-lock-vs-table-lock.md) | InnoDB가 인덱스 레코드에 락을 거는 원리(인덱스가 없으면 Full Table Lock), `LOCK TABLES` vs InnoDB Row Lock 비교, `information_schema.INNODB_TRX`로 트랜잭션별 락 보유 현황 조회 방법 |
| [03. Gap Lock과 Next-Key Lock — Phantom Read 방지 메커니즘](./lock-and-concurrency/03-gap-lock-next-key-lock.md) | Gap Lock이 인덱스 레코드 사이의 "간격"을 잠가 새로운 삽입을 차단하는 방식, Next-Key Lock = Record Lock + Gap Lock의 범위 계산 방법, `READ_COMMITTED`에서 Gap Lock이 비활성화되는 이유와 그 결과 |
| [04. 데드락 발생 원리와 감지 알고리즘](./lock-and-concurrency/04-deadlock-detection.md) | 두 트랜잭션이 서로의 락을 기다리는 순환 의존(Circular Wait) 형성 과정, InnoDB의 Wait-for Graph 기반 데드락 감지 알고리즘, 롤백될 트랜잭션을 선택하는 `innodb_deadlock_detect` 비용 계산 |
| [05. 데드락 로그 분석과 해결 전략](./lock-and-concurrency/05-deadlock-log-analysis.md) | `SHOW ENGINE INNODB STATUS`의 `LATEST DETECTED DEADLOCK` 섹션을 단계별로 해석하는 방법, 어느 트랜잭션이 어느 인덱스 레코드에 어떤 락을 보유·대기 중인지 읽는 방법, 데드락 재현 → 로그 확인 → 락 순서 통일로 해결하는 실전 패턴 |
| [06. Optimistic Lock vs Pessimistic Lock — 선택 기준](./lock-and-concurrency/06-optimistic-vs-pessimistic-lock.md) | Pessimistic Lock(`SELECT ... FOR UPDATE`)이 충돌을 사전 차단하는 방식과 처리량 저하 대가, Optimistic Lock(버전 컬럼 CAS)이 충돌을 사후 감지하는 방식과 재시도 비용, JPA `@Version`이 UPDATE WHERE version = ? 로 변환되는 원리, 쓰기 충돌 빈도에 따른 선택 기준 |

</details>

<br/>

### 🔹 Chapter 6: Performance Tuning

> **핵심 질문:** 느린 쿼리를 어떻게 찾고, N+1은 DB 관점에서 왜 문제이며, OFFSET 기반 페이징은 왜 데이터가 많을수록 느려지는가?

<details>
<summary><b>Slow Query 분석부터 HikariCP 튜닝까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Slow Query 분석 방법론](./performance-tuning/01-slow-query-analysis.md) | `slow_query_log` 설정과 `long_query_time` 임계값 선택 기준, `mysqldumpslow`로 패턴별 슬로우 쿼리를 집계하는 방법, `performance_schema`의 `events_statements_summary_by_digest`로 상위 쿼리를 찾는 방법 |
| [02. N+1 문제 — DB 관점에서의 근본 원인](./performance-tuning/02-n-plus-one-db-perspective.md) | N+1이 발생할 때 Slow Query Log에 동일 쿼리가 N번 반복 기록되는 패턴, 각 쿼리가 독립 Connection을 사용할 때 Connection Pool 소진과의 관계, `p6spy` / `datasource-proxy`로 JPA가 날리는 쿼리 수를 계측하는 방법, Fetch Join과 `@EntityGraph`가 생성하는 실행계획 비교 |
| [03. 페이징 최적화 — OFFSET의 함정과 커서 기반 페이징](./performance-tuning/03-pagination-optimization.md) | `LIMIT 10 OFFSET 100000`이 실제로 100010행을 읽고 100000행을 버리는 원리, No-Offset(커서 기반) 페이징이 `WHERE id > last_id`로 B-Tree 탐색을 재시작하는 방식, JPA에서 커서 기반 페이징을 구현하는 패턴 |
| [04. 대용량 테이블 최적화 — 파티셔닝과 샤딩](./performance-tuning/04-large-table-optimization.md) | Range / List / Hash Partition이 파티션 프루닝(Partition Pruning)으로 스캔 범위를 줄이는 원리, 파티션 키 선택이 잘못됐을 때 전체 파티션을 스캔하는 문제, 수직 샤딩(컬럼 분리)과 수평 샤딩(행 분리)의 트레이드오프 |
| [05. Connection Pool 튜닝 — HikariCP와 DB 연결 관리](./performance-tuning/05-connection-pool-hikaricp.md) | HikariCP가 Connection을 미리 생성해두는 방식, `maximumPoolSize` 계산 공식(CPU 코어 수 × 2 + 유효 디스크 수), Connection 대기로 인한 `HikariPool-1 - Connection is not available` 에러 분석, Spring Boot에서 HikariCP 설정과 모니터링 방법 |

</details>

<br/>

### 🔹 Chapter 7: Replication & High Availability

> **핵심 질문:** Primary에 쓰인 데이터가 Replica에 어떻게 전파되고, Read Replica 사용 시 정합성은 어떻게 보장되는가?

<details>
<summary><b>Binary Log 복제 원리부터 Spring Read/Write 분리까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. MySQL Replication 원리 — Binary Log 기반 복제](./replication-high-availability/01-mysql-replication-binlog.md) | Primary가 Binary Log에 변경 이벤트를 기록하고, Replica의 IO Thread가 Relay Log로 수신하며, SQL Thread가 적용하는 3단계 구조, Statement-Based vs Row-Based vs Mixed 바이너리 로그 포맷 비교 |
| [02. Read Replica — 읽기 분산과 정합성 트레이드오프](./replication-high-availability/02-read-replica-consistency.md) | Replication Lag이 발생하는 이유와 측정 방법(`Seconds_Behind_Source`), 방금 쓴 데이터를 Read Replica에서 읽었을 때 보이지 않는 Stale Read 문제, 쓰기 직후 읽기는 Primary로, 이후 읽기는 Replica로 라우팅하는 전략 |
| [03. GTID 기반 복제와 자동 페일오버](./replication-high-availability/03-gtid-auto-failover.md) | GTID(Global Transaction ID)가 `server_uuid:transaction_id` 형태로 트랜잭션을 전역 고유 식별하는 방식, Replica가 어디까지 적용했는지 추적하는 `gtid_executed`, MHA / Orchestrator로 자동 페일오버 시 GTID가 중요한 이유 |
| [04. Spring에서 Read/Write 분리 구현](./replication-high-availability/04-spring-read-write-separation.md) | `AbstractRoutingDataSource`로 트랜잭션 속성(`readOnly = true`)에 따라 DataSource를 동적으로 선택하는 패턴, `@Transactional(readOnly = true)`가 MySQL에 `SET TRANSACTION READ ONLY`를 전송해 MVCC 최적화를 활성화하는 원리, LazyConnectionDataSourceProxy와의 조합 |

</details>

---

## 🔬 핵심 분석 대상 — 쿼리 실행 전체 흐름

이 레포의 모든 챕터는 아래 흐름을 완전히 이해하는 것을 목표로 합니다.

```
애플리케이션 (JPA / JDBC)
  │  SQL 문자열 전송
  ▼
┌─────────────────────────────────────────┐
│  MySQL Server Layer                     │
│  ① Parser      SQL → Parse Tree         │
│  ② Optimizer   실행계획 수립 (Ch3)         │
│    ├── 통계 정보 조회 (rows, cardinality)  │
│    ├── 인덱스 선택 비용 계산                 │
│    └── Join 순서 결정                     │
│  ③ Executor    실행계획 실행               │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│  InnoDB Storage Engine                  │
│                                         │
│  Buffer Pool  (Ch1)                     │
│    ├── Page Cache (16KB 단위)            │
│    └── Dirty Page → Disk (비동기)        │
│                                         │
│  B-Tree Index 탐색  (Ch2)                │
│    ├── Clustered Index (PK → Row)       │
│    └── Secondary Index → PK → Row       │
│                                         │
│  MVCC  (Ch4)                            │
│    ├── Read View 생성 (격리 수준별)         │
│    ├── Undo Log 버전 체인 탐색             │
│    └── Consistent Read / Current Read   │
│                                         │
│  Lock Manager  (Ch5)                    │
│    ├── Record Lock / Gap Lock 획득       │
│    └── Wait-for Graph 데드락 감지          │
│                                         │
│  Disk I/O                               │
│    ├── Redo Log flush (COMMIT 시)        │
│    └── Page 읽기/쓰기 (Page 단위)          │
└─────────────────────────────────────────┘

트랜잭션 생명주기:
  BEGIN
    → Read View 생성 (REPEATABLE_READ 기준)
    → 쿼리 실행: B-Tree 탐색 → Lock 획득
    → 변경: Undo Log 기록 → Buffer Pool 갱신
  COMMIT
    → Redo Log flush → Lock 해제
    → Buffer Pool → Disk (비동기, Checkpoint)

장애 시나리오:
  ① 인덱스 없는 쿼리     → Full Table Scan   → Slow Query Log 기록
  ② 격리 수준별 이상현상  → Dirty / Phantom Read 재현 → MVCC로 분석
  ③ 데드락 발생          → Wait-for Graph 감지 → ROLLBACK → 로그 분석
  ④ N+1 발생            → Slow Query Log N번 반복 → Fetch Join으로 해결
```

---

## 🐳 실험 환경 (Docker Compose)

모든 챕터의 실험은 아래 환경에서 재현 가능합니다.

```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: deep_dive
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # 실험용 테스트 데이터
    ports:
      - "3306:3306"
    command: >
      --slow_query_log=ON
      --slow_query_log_file=/var/log/mysql/slow.log
      --long_query_time=0.1
      --innodb_monitor_enable=all
      --general_log=ON
      --general_log_file=/var/log/mysql/general.log
```

```sql
-- init.sql: 공통 실험 데이터
-- 인덱스 실험: 100만 건 주문 데이터
-- 트랜잭션 실험: 재고 감소 동시성 시나리오
-- 데드락 실험: 두 트랜잭션의 교차 업데이트 시나리오
-- 페이징 실험: OFFSET vs 커서 기반 성능 비교
```

각 문서에서 해당 시나리오에 필요한 쿼리와 실험 순서를 안내합니다.

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **잘못된 이해** | Before — DB를 블랙박스로 두는 접근과 그 결과 |
| ✨ **올바른 이해** | After — 원리를 알고 난 후의 접근과 해석 |
| 🔬 **내부 동작 원리** | MySQL InnoDB 소스 레벨 분석 + ASCII 구조도 |
| 💻 **실전 실험** | EXPLAIN 분석, 슬로우 쿼리 재현, 락 확인 쿼리 |
| 📊 **성능 비교** | 인덱스 유무, 격리 수준별, 락 전략별 수치 비교 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "느린 쿼리를 당장 분석해야 한다" — 실전 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch3-02  EXPLAIN 읽는 법 → 실행계획 해석
Day 2  Ch2-01  B-Tree 인덱스 원리 → 인덱스 선택 이유 이해
       Ch2-06  인덱스를 타지 않는 패턴 → 현재 쿼리 점검
Day 3  Ch6-01  Slow Query 분석 방법론 → 실제 서비스에 적용
```

</details>

<details>
<summary><b>🟡 "트랜잭션과 락을 제대로 이해하고 싶다" — 핵심 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch4-01  ACID 실제 구현 방법
Day 2  Ch4-02  MVCC 동작 원리 → Read View 이해
       Ch4-03  Undo Log 버전 체인
Day 3  Ch4-04  격리 수준 4가지 — 구현 원리까지
Day 4  Ch5-01  락의 종류 (S/X/IS/IX)
       Ch5-03  Gap Lock과 Next-Key Lock
Day 5  Ch5-04  데드락 발생 원리
       Ch5-05  데드락 로그 분석 실습
Day 6  Ch6-02  N+1 문제 — DB 관점 분석
Day 7  Ch7-04  Spring Read/Write 분리 구현
```

</details>

<details>
<summary><b>🔴 "MySQL InnoDB 소스코드를 직접 읽고 내부를 완전히 이해하고 싶다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — InnoDB 물리 저장 구조
        → Buffer Pool의 LRU 교체 정책을 performance_schema로 실시간 관찰

2주차  Chapter 2 전체 — Index 완전 분해
        → EXPLAIN으로 각 인덱스 전략별 type, key_len 변화 확인

3주차  Chapter 3 전체 — Query Execution & Optimizer
        → EXPLAIN ANALYZE로 예측 비용 vs 실제 비용 오차 분석

4주차  Chapter 4 전체 — Transaction & MVCC
        → 격리 수준별 Phantom Read 재현 실험 → Undo Log 버전 체인 추적

5주차  Chapter 5 전체 — Lock & Concurrency
        → 데드락 의도적으로 발생 → SHOW ENGINE INNODB STATUS로 로그 분석

6주차  Chapter 6 전체 — Performance Tuning
        → N+1을 Slow Query Log로 포착 → Fetch Join으로 해결 → 비교

7주차  Chapter 7 전체 — Replication & High Availability
        → Docker Compose로 Primary/Replica 구성 → Replication Lag 측정
```

</details>

---

## 🔗 선행 학습 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-data-transaction-deep-dive](https://github.com/dev-book-lab/spring-data-transaction-deep-dive) | JPA 영속성 컨텍스트, `@Transactional` 전파, JPQL | Ch4(MVCC와 JPA 격리 수준 매핑), Ch5(Optimistic Lock `@Version`), Ch6(N+1 해결) |
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | AOP Proxy, Bean 생명주기 | Ch5(`@Transactional`이 Proxy로 트랜잭션 경계를 만드는 원리) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Auto-configuration, Actuator | Ch6(HikariCP Auto-configuration, `spring.datasource` 설정) |

> 💡 이 레포는 **MySQL InnoDB 내부 동작**에 집중합니다. JPA/Spring을 모르더라도 순수 DB 관점으로 학습 가능합니다. 단, Chapter 6의 N+1 분석과 Chapter 7의 Read/Write 분리는 Spring Data JPA 선행 지식이 있을 때 더 깊이 연결됩니다.

---

## 🙏 Reference

- [MySQL 8.0 Reference Manual — InnoDB Storage Engine](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)
- [MySQL 8.0 Reference Manual — EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [InnoDB Source Code (GitHub)](https://github.com/mysql/mysql-server/tree/trunk/storage/innobase)
- [High Performance MySQL, 4th Edition](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)
- [Use The Index, Luke — SQL Indexing and Tuning](https://use-the-index-luke.com/)
- [MySQL InnoDB Internals (Percona Blog)](https://www.percona.com/blog/)
- [Jeremy Cole — InnoDB Ruby (Page 구조 분석 도구)](https://github.com/jeremycole/innodb_ruby)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"SQL을 쓰는 것과, DB가 내부에서 무슨 일을 하는지 아는 것은 다르다"*

</div>
