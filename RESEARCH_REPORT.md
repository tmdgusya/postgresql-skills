# Supabase PostgreSQL Best Practices Skill Set 분석 보고서

## 개요

이 보고서는 Supabase에서 작성한 Claude Code용 PostgreSQL Best Practices Skill Set을 분석하고, 각 카테고리가 왜 이렇게 구성되었는지 PostgreSQL 공식 문서에 근거하여 기술적으로 설명합니다.

**분석 대상**: https://github.com/supabase/agent-skills/tree/main/skills/supabase-postgres-best-practices

---

## 1. Skill Set 구조 및 우선순위 설계 근거

### 1.1 8개 카테고리 우선순위 체계

| 우선순위 | 카테고리 | 영향도 | 성능 개선 범위 |
|----------|----------|--------|----------------|
| 1 | Query Performance | CRITICAL | 10-1000x |
| 2 | Connection Management | CRITICAL | 시스템 안정성 |
| 3 | Security & RLS | CRITICAL | 보안 필수 |
| 4 | Schema Design | HIGH | 5-20x |
| 5 | Concurrency & Locking | MEDIUM-HIGH | 3-10x |
| 6 | Data Access Patterns | MEDIUM | 10-100x |
| 7 | Monitoring & Diagnostics | LOW-MEDIUM | 진단 도구 |
| 8 | Advanced Features | LOW | 점진적 개선 |

**설계 근거**:
- CRITICAL 등급은 시스템 전체에 즉각적인 영향을 미치는 항목
- 쿼리 성능과 연결 관리가 최상위인 이유는 대부분의 성능 문제가 여기서 발생
- 보안(RLS)이 3위인 이유는 Supabase의 멀티테넌트 아키텍처 특성

---

## 2. Query Performance (CRITICAL)

### 2.1 query-missing-indexes: 인덱스 누락 문제

**Skill 내용**: WHERE 및 JOIN 컬럼에 인덱스 추가로 100-1000x 성능 향상

**PostgreSQL 공식 문서 근거**:

PostgreSQL의 쿼리 실행 엔진은 인덱스가 없으면 **Sequential Scan**을 수행합니다:

> "A sequential scan reads every row in the table and checks each row against the scan condition."
> — [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

**기술적 설명**:

```
Sequential Scan 비용 = (디스크 페이지 수 × seq_page_cost) + (행 수 × cpu_tuple_cost)
Index Scan 비용 = (인덱스 탐색 비용) + (선택된 행 × random_page_cost)
```

인덱스가 없으면:
- **O(n)** 시간 복잡도로 모든 행 스캔
- 대용량 테이블에서 기하급수적 성능 저하
- Disk I/O 병목 발생

### 2.2 query-index-types: 인덱스 타입 선택

**Skill 내용**: B-tree, GIN, BRIN, Hash 인덱스의 적절한 선택

**PostgreSQL 공식 문서 근거**:

#### B-tree (기본값)
> "B-trees can handle equality and range queries on data that can be sorted into some ordering."
> — [PostgreSQL: Index Types](https://www.postgresql.org/docs/current/indexes-types.html)

- 지원 연산자: `<`, `<=`, `=`, `>=`, `>`, `BETWEEN`, `IN`
- 정렬된 데이터 구조로 이진 탐색 가능
- **O(log n)** 탐색 복잡도

#### GIN (Generalized Inverted Index)
> "GIN is designed for handling cases where the items to be indexed are composite values, and the queries to be handled by the index need to search for element values that appear within the composite items."
> — [PostgreSQL: GIN Indexes](https://www.postgresql.org/docs/current/gin-intro.html)

**내부 구조**:
```
JSONB {"tags": ["sql", "postgres"]}
  ↓ GIN 인덱스
"sql" → [row1, row5, row12]
"postgres" → [row1, row3, row7]
```

- 역 인덱스(inverted index) 구조
- JSONB, 배열, 전문 검색에 최적
- `@>`, `?`, `&&` 연산자 지원

#### BRIN (Block Range Index)
> "BRIN indexes store summaries about the values stored in consecutive physical block ranges of a table."
> — [PostgreSQL: BRIN Indexes](https://www.postgresql.org/docs/current/brin-intro.html)

- 블록 범위별 min/max만 저장
- **B-tree 대비 100-1000배 작은 크기**
- 시계열 데이터(물리적 순서 = 시간 순서)에 최적

### 2.3 query-composite-indexes: 복합 인덱스

**Skill 내용**: "equality columns first, range columns last" 규칙

**PostgreSQL 공식 문서 근거**:

> "The exact rule is that equality constraints on leading columns, plus any inequality constraints on the first column that does not have an equality constraint, will be used to limit the portion of the index that is scanned."
> — [PostgreSQL: Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)

**기술적 설명**:

```sql
CREATE INDEX idx ON orders (status, created_at);

-- 효율적: status에서 즉시 점프 후 created_at 범위 스캔
WHERE status = 'pending' AND created_at > '2024-01-01'

-- 비효율적: status 전체 스캔 필요
WHERE created_at > '2024-01-01'
```

**Leftmost Prefix Rule**:
- B-tree는 첫 번째 컬럼으로 물리적 정렬
- 선두 컬럼의 등호 조건이 스캔 범위를 가장 효과적으로 제한
- 중간 컬럼 건너뛰기는 Skip Scan 최적화로 특정 경우만 가능

### 2.4 query-covering-indexes: 커버링 인덱스

**Skill 내용**: INCLUDE 절로 heap fetch 회피, 2-5x 성능 향상

**PostgreSQL 공식 문서 근거**:

> "An index-only scan can answer queries from the index without reference to the heap."
> — [PostgreSQL: Index-Only Scans](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)

**기술적 메커니즘**:

1. **Visibility Map**: 모든 행이 모든 트랜잭션에 가시적인 페이지 추적
2. **Index-Only Scan 조건**:
   - 쿼리가 참조하는 모든 컬럼이 인덱스에 포함
   - Visibility Map 비트가 설정된 페이지

```sql
CREATE INDEX idx ON users (email) INCLUDE (name, created_at);

-- Index-Only Scan 가능 (heap 접근 불필요)
SELECT name, created_at FROM users WHERE email = 'test@example.com';
```

**성능 이점**:
- 랜덤 디스크 I/O 제거 (heap 미접근)
- 인덱스는 힙보다 작아 캐시 효율성 증가
- Suffix truncation으로 상위 B-tree 레벨 최적화

### 2.5 query-partial-indexes: 부분 인덱스

**Skill 내용**: WHERE 조건으로 인덱스 크기 5-20x 감소

**PostgreSQL 공식 문서 근거**:

> "A partial index is an index built over a subset of a table; the subset is defined by a conditional expression (called the predicate of the partial index)."
> — [PostgreSQL: Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)

**기술적 제약**:

> "PostgreSQL cannot recognize mathematically equivalent expressions; the predicate must match the query's WHERE condition or be implied by simple inequalities."

```sql
-- 인덱스
CREATE INDEX idx ON users (email) WHERE deleted_at IS NULL;

-- 사용 가능 (정확히 일치)
SELECT * FROM users WHERE email = 'test@example.com' AND deleted_at IS NULL;

-- 사용 불가 (prepared statement)
SELECT * FROM users WHERE email = $1 AND deleted_at IS NULL;
```

---

## 3. Connection Management (CRITICAL)

### 3.1 conn-pooling: 연결 풀링

**Skill 내용**: 연결당 1-3MB 메모리 소비, 풀 크기 공식 제공

**PostgreSQL 공식 문서 근거**:

> "A complex query might perform several sort and hash operations at the same time, with each operation generally being allowed to use as much memory as work_mem specifies."
> — [PostgreSQL: Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html)

**연결당 메모리 구성**:
- `work_mem`: 쿼리별 정렬/해시 작업용 (기본 4MB)
- `maintenance_work_mem`: 유지보수 작업용
- `temp_buffers`: 임시 테이블용
- 백엔드 프로세스 기본 메모리: 약 2-3MB

**풀 크기 공식** (PostgreSQL Wiki):
```
connections = (core_count × 2) + effective_spindle_count
```

- `core_count × 2`: I/O 대기 시간 동안의 지연 관리
- `effective_spindle_count`: 디스크 I/O 용량 반영

### 3.2 conn-limits: 연결 제한

**Skill 내용**: `work_mem × max_connections < 25% RAM`

**기술적 근거**:

메모리 할당 구조:
```
Total RAM
├── shared_buffers (25-40% RAM) - PostgreSQL 권장
├── OS/기타 (15% RAM)
└── work_mem pool (나머지)
    └── work_mem × max_connections × 복수 작업
```

> "The total memory used could be many times the value of work_mem; it is necessary to keep this fact in mind when choosing the value."
> — [PostgreSQL: work_mem](https://www.postgresql.org/docs/current/runtime-config-resource.html)

**보수적 접근 이유**:
- 복잡한 쿼리는 여러 work_mem 할당을 동시 사용
- 동시 세션 수 증가 시 메모리 급증
- OOM Killer 방지

### 3.3 conn-idle-timeout: 유휴 연결 타임아웃

**Skill 내용**: 30-50% 연결 슬롯 회수 가능

**PostgreSQL 공식 문서 근거**:

> "This option can be used to ensure that idle sessions do not hold locks for an unreasonable amount of time. Even when no significant locks are held, an open transaction prevents vacuuming away recently-dead tuples."
> — [PostgreSQL: idle_in_transaction_session_timeout](https://www.postgresql.org/docs/current/runtime-config-client.html)

**유휴 연결이 보유하는 리소스**:
1. **락(Locks)**: 다른 트랜잭션 차단
2. **MVCC 스냅샷**: dead tuple 정리 방해
3. **메모리**: 백엔드 프로세스 유지
4. **연결 슬롯**: 새 연결 차단

### 3.4 conn-prepared-statements: Prepared Statements와 풀링

**Skill 내용**: Transaction mode에서 prepared statement 충돌 문제

**기술적 설명**:

Prepared statements는 **세션 메모리 컨텍스트**에 저장됩니다:
- Parse tree 캐싱
- Plan caching
- 연결별로 격리

**Transaction Mode 충돌**:
```sql
-- Connection A (Transaction 1)
PREPARE my_stmt AS SELECT * FROM users WHERE id = $1;

-- Connection이 풀로 반환 후 다른 클라이언트에 할당

-- Connection A (Transaction 2, 다른 클라이언트)
PREPARE my_stmt AS SELECT * FROM orders WHERE id = $1;
-- ERROR: prepared statement "my_stmt" already exists
```

**해결책**:
1. Session mode 사용 (확장성 감소)
2. Prepared statements 비활성화 (`prepareThreshold=0`)
3. PgBouncer 1.21+ 사용 (`max_prepared_statements` 지원)

---

## 4. Security & RLS (CRITICAL)

### 4.1 security-rls-basics: Row Level Security 기본

**Skill 내용**: 애플리케이션 레벨 필터링 대신 데이터베이스 강제

**PostgreSQL 공식 문서 근거**:

> "In addition to the SQL-standard privilege system available through GRANT, tables can have row security policies that restrict, on a per-user basis, which rows can be returned by normal queries or inserted, updated, or deleted by data modification commands."
> — [PostgreSQL: Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

**USING vs WITH CHECK**:

| 절 | 목적 | 적용 시점 |
|---|------|----------|
| USING | 기존 행 가시성 제어 | SELECT, UPDATE, DELETE |
| WITH CHECK | 새/수정 행 유효성 검증 | INSERT, UPDATE |

```sql
-- USING: 어떤 행을 볼 수 있는지
CREATE POLICY select_own ON documents
    FOR SELECT USING (owner_id = auth.uid());

-- WITH CHECK: 어떤 행을 생성할 수 있는지
CREATE POLICY insert_own ON documents
    FOR INSERT WITH CHECK (owner_id = auth.uid());
```

### 4.2 security-rls-performance: RLS 성능 최적화

**Skill 내용**: 함수 캐싱으로 100x+ 성능 향상

**기술적 문제**:

```sql
-- 나쁜 예: auth.uid()가 매 행마다 호출
CREATE POLICY bad ON documents
    USING (owner_id = auth.uid());

-- 좋은 예: 서브쿼리로 initPlan 실행, 결과 캐시
CREATE POLICY good ON documents
    USING (owner_id = (SELECT auth.uid()));
```

**최적화 기법**:

1. **인덱스 추가**: RLS 정책 컬럼에 인덱스 (99.94% 개선)
2. **함수 래핑**: `(SELECT auth.uid())` 패턴 (99.993% 개선)
3. **SECURITY DEFINER**: 복잡한 권한 로직 캡슐화
4. **역할 명시**: `TO authenticated`로 불필요한 평가 방지

### 4.3 security-privileges: 최소 권한 원칙

**PostgreSQL 공식 문서 근거**:

> "The right to modify or destroy an object is inherent in being the object's owner, and cannot be granted or revoked in itself."
> — [PostgreSQL: Privileges](https://www.postgresql.org/docs/current/ddl-priv.html)

**기본 PUBLIC 권한** (보안 위험):
- 함수/프로시저: EXECUTE 기본 부여
- 데이터베이스: CONNECT, TEMPORARY 기본 부여

**권장 설정**:
```sql
-- PUBLIC 권한 제거
REVOKE ALL ON SCHEMA public FROM PUBLIC;

-- 명시적 역할 부여
GRANT USAGE ON SCHEMA public TO authenticated;
GRANT SELECT ON users TO authenticated;
```

---

## 5. Schema Design (HIGH)

### 5.1 schema-primary-keys: 기본 키 전략

**Skill 내용**: `bigint identity` 권장, random UUID v4 회피

**기술적 근거**:

**Random UUID v4 문제**:
- B-tree 인덱스에서 무작위 삽입 → 페이지 분할 증가
- 인덱스 단편화 (fragmentation)
- 캐시 효율성 저하

**대안**:
```sql
-- 순차적 bigint (최적)
id bigint generated always as identity primary key

-- 시간 순서 UUID (분산 시스템용)
id uuid default uuid_generate_v7() primary key  -- pg_uuidv7 확장
```

### 5.2 schema-data-types: 데이터 타입 선택

**Skill 내용**: 적절한 타입으로 50% 스토리지 감소

**PostgreSQL 공식 문서 근거**:

| 잘못된 선택 | 올바른 선택 | 이유 |
|------------|------------|------|
| `int` | `bigint` | 21억 오버플로우 방지 |
| `varchar(255)` | `text` | PostgreSQL에서 동일 성능, 제약 불필요 |
| `timestamp` | `timestamptz` | 타임존 정보 보존 |
| `float` | `numeric(10,2)` | 금액의 정확한 소수점 연산 |

> "The storage requirement for a short string (up to 126 bytes) is 1 byte plus the actual string. There is no performance difference among these three types."
> — [PostgreSQL: Character Types](https://www.postgresql.org/docs/current/datatype-character.html)

### 5.3 schema-foreign-key-indexes: 외래 키 인덱스

**Skill 내용**: PostgreSQL은 FK에 자동 인덱스를 생성하지 않음

**기술적 영향**:

```sql
-- FK 정의만으로는 인덱스 생성 안됨
ALTER TABLE orders ADD FOREIGN KEY (customer_id) REFERENCES customers(id);

-- 명시적 인덱스 필요
CREATE INDEX orders_customer_id_idx ON orders (customer_id);
```

**인덱스 없는 FK 문제**:
1. JOIN 시 전체 테이블 스캔
2. ON DELETE CASCADE 시 테이블 전체 락 + 스캔
3. 10-100x 성능 저하

### 5.4 schema-partitioning: 테이블 파티셔닝

**Skill 내용**: 1억 행 이상 테이블에서 5-20x 성능 향상

**PostgreSQL 공식 문서 근거**:

> "Partitioning refers to splitting what is logically one large table into smaller physical pieces."
> — [PostgreSQL: Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)

**이점**:
- **Partition Pruning**: 쿼리가 관련 파티션만 스캔
- **빠른 데이터 삭제**: `DROP PARTITION`은 즉시 완료
- **병렬 처리**: 파티션별 독립 VACUUM/ANALYZE

```sql
CREATE TABLE events (
    id bigint,
    created_at timestamptz,
    data jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

---

## 6. Concurrency & Locking (MEDIUM-HIGH)

### 6.1 lock-deadlock-prevention: 데드락 방지

**PostgreSQL 공식 문서 근거**:

> "The best defense against deadlocks is generally to avoid them by being certain that all applications using a database acquire locks on multiple objects in a consistent order."
> — [PostgreSQL: Deadlocks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-DEADLOCKS)

**데드락 시나리오**:
```sql
-- Transaction 1
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
-- 그 후 id = 2를 업데이트하려 함

-- Transaction 2
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- 데드락: T1은 T2를 기다리고, T2는 T1을 기다림
```

**해결책**: 일관된 락 순서 (항상 id = 1 먼저)

### 6.2 lock-advisory: Advisory Locks

**PostgreSQL 공식 문서 근거**:

> "PostgreSQL provides a means for creating locks that have application-defined meanings. These are called advisory locks, because the system does not enforce their use."
> — [PostgreSQL: Advisory Locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)

**Session vs Transaction Level**:

| 특성 | Session-Level | Transaction-Level |
|------|---------------|-------------------|
| 함수 | `pg_advisory_lock()` | `pg_advisory_xact_lock()` |
| 지속 | 명시적 해제까지 | 트랜잭션 종료까지 |
| 해제 | 수동 필요 | 자동 |
| 롤백 시 | 유지됨 | 자동 해제 |

### 6.3 lock-skip-locked: SKIP LOCKED

**Skill 내용**: 큐 처리에서 10x 처리량 향상

**PostgreSQL 공식 문서 근거**:

> "With SKIP LOCKED, any selected rows that cannot be immediately locked are skipped."
> — [PostgreSQL: SELECT](https://www.postgresql.org/docs/current/sql-select.html)

```sql
-- 여러 워커가 동시에 실행 가능
UPDATE jobs
SET status = 'processing', worker_id = $1
WHERE id = (
    SELECT id FROM jobs
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
RETURNING *;
```

### 6.4 lock-short-transactions: 짧은 트랜잭션

**PostgreSQL 공식 문서 근거**:

> "A transaction that runs for a long time can contribute to table bloat because VACUUM cannot remove recently-dead tuples that may be visible to this transaction."
> — [PostgreSQL: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)

**긴 트랜잭션의 문제**:
1. VACUUM 작동 방해 → 테이블 bloat
2. 락 보유 시간 증가 → 다른 트랜잭션 대기
3. XID Wraparound 위험 증가
4. 연결 풀 고갈

---

## 7. Data Access Patterns (MEDIUM)

### 7.1 data-n-plus-one: N+1 쿼리 제거

**Skill 내용**: 10-100x 라운드 트립 감소

**문제**:
```javascript
// N+1: 101번의 쿼리
const users = await db.query('SELECT * FROM users WHERE active = true'); // 1번
for (const user of users) {
    const orders = await db.query('SELECT * FROM orders WHERE user_id = $1', [user.id]); // N번
}
```

**해결책**:
```sql
-- JOIN으로 1번의 쿼리
SELECT u.id, u.name, o.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.active = true;

-- 또는 배열 파라미터
SELECT * FROM orders WHERE user_id = ANY($1);
```

### 7.2 data-pagination: 커서 기반 페이지네이션

**Skill 내용**: OFFSET은 O(n), 커서는 O(1)

**기술적 비교**:

```sql
-- OFFSET: 10000페이지에서 200,000행 스캔 후 버림
SELECT * FROM posts ORDER BY id OFFSET 200000 LIMIT 20;

-- 커서: 인덱스로 즉시 점프
SELECT * FROM posts WHERE id > $last_id ORDER BY id LIMIT 20;
```

**OFFSET 문제**:
- 깊은 페이지일수록 선형적으로 느려짐
- 동일 데이터를 반복 스캔
- 메모리 낭비

### 7.3 data-batch-inserts: 배치 삽입

**Skill 내용**: 10-50x 빠른 대량 삽입

**PostgreSQL 공식 문서 근거**:

> "COPY is optimized for loading large numbers of rows; it is less flexible than INSERT, but incurs significantly less overhead for large data loads."
> — [PostgreSQL: COPY](https://www.postgresql.org/docs/current/sql-copy.html)

```sql
-- 개별 INSERT (느림)
INSERT INTO users (name) VALUES ('Alice');
INSERT INTO users (name) VALUES ('Bob');

-- 배치 INSERT (빠름)
INSERT INTO users (name) VALUES ('Alice'), ('Bob'), ('Charlie');

-- COPY (가장 빠름)
COPY users (name) FROM STDIN;
```

### 7.4 data-upsert: UPSERT 패턴

**Skill 내용**: SELECT-then-INSERT의 레이스 컨디션 방지

**PostgreSQL 공식 문서 근거**:

> "ON CONFLICT DO UPDATE guarantees an atomic INSERT or UPDATE outcome; provided there is no independent error, one of those two outcomes is guaranteed."
> — [PostgreSQL: INSERT](https://www.postgresql.org/docs/current/sql-insert.html)

```sql
-- 원자적 UPSERT
INSERT INTO user_stats (user_id, login_count)
VALUES ($1, 1)
ON CONFLICT (user_id)
DO UPDATE SET login_count = user_stats.login_count + 1
RETURNING *;
```

---

## 8. Monitoring & Diagnostics (LOW-MEDIUM)

### 8.1 monitor-explain-analyze: 실행 계획 분석

**핵심 지표**:

| 위험 신호 | 의미 | 해결책 |
|----------|------|--------|
| Sequential Scan | 전체 테이블 스캔 | 인덱스 추가 |
| Rows Removed by Filter 높음 | 선택도 낮음 | 인덱스 조건 개선 |
| External Merge Sort | work_mem 부족 | work_mem 증가 |
| Nested Loop (많은 반복) | 비효율적 조인 | 조인 전략 변경 |

### 8.2 monitor-pg-stat-statements: 쿼리 통계

**PostgreSQL 공식 문서 근거**:

> "The pg_stat_statements module provides a means for tracking planning and execution statistics of all SQL statements executed by a server."
> — [PostgreSQL: pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)

**주요 메트릭**:
- `total_exec_time`: 총 실행 시간
- `calls`: 실행 횟수
- `shared_blks_hit/read`: 캐시 히트율
- `rows`: 반환/영향 행 수

### 8.3 monitor-vacuum-analyze: VACUUM과 ANALYZE

**PostgreSQL 공식 문서 근거**:

> "PostgreSQL databases require periodic maintenance known as vacuuming. For many installations, it is sufficient to let vacuuming be performed by the autovacuum daemon."
> — [PostgreSQL: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)

**Autovacuum 트리거 공식**:
```
vacuum_threshold = autovacuum_vacuum_threshold +
                   autovacuum_vacuum_scale_factor × reltuples
```

기본값: 50 + (0.2 × 테이블 크기) = 20%의 dead tuple

**High-churn 테이블 권장**:
```sql
ALTER TABLE high_activity SET (
    autovacuum_vacuum_scale_factor = 0.02,  -- 2%
    autovacuum_analyze_scale_factor = 0.01  -- 1%
);
```

---

## 9. Advanced Features (LOW)

### 9.1 advanced-full-text-search: 전문 검색

**Skill 내용**: LIKE '%keyword%' 대비 100x 빠름

**PostgreSQL 공식 문서 근거**:

> "Full Text Searching provides the capability to identify natural-language documents that satisfy a query."
> — [PostgreSQL: Full Text Search](https://www.postgresql.org/docs/current/textsearch-intro.html)

```sql
-- tsvector 컬럼과 GIN 인덱스
ALTER TABLE articles ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || content)) STORED;

CREATE INDEX articles_search_idx ON articles USING GIN (search_vector);

-- 전문 검색 쿼리
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql & performance');
```

### 9.2 advanced-jsonb-indexing: JSONB 인덱싱

**Skill 내용**: 10-100x 빠른 JSONB 쿼리

**인덱스 전략**:

```sql
-- GIN 인덱스 (포함 연산자용)
CREATE INDEX idx_attrs ON products USING GIN (attributes);
-- WHERE attributes @> '{"color": "red"}'

-- Expression 인덱스 (특정 키 검색용)
CREATE INDEX idx_brand ON products ((attributes->>'brand'));
-- WHERE attributes->>'brand' = 'Apple'
```

**Operator Class 선택**:
- `jsonb_ops` (기본): 모든 연산자 지원, 큰 인덱스
- `jsonb_path_ops`: `@>` 전용, 2-3x 작은 인덱스

---

## 10. 결론: Skill Set 설계 철학

### 10.1 우선순위 기반 설계

Supabase의 skill set은 **영향도 기반**으로 구성되었습니다:

1. **CRITICAL**: 시스템 전체에 즉각적 영향 (쿼리 성능, 연결, 보안)
2. **HIGH**: 장기적 확장성에 영향 (스키마 설계)
3. **MEDIUM**: 특정 상황에서 중요 (동시성, 데이터 패턴)
4. **LOW**: 점진적 개선 (모니터링, 고급 기능)

### 10.2 AI 에이전트 최적화

각 reference 파일은 AI가 즉시 적용할 수 있도록:
- **Before/After** 패턴으로 구조화
- **정량적 개선** 수치 제공 (10x, 100x 등)
- **PostgreSQL 공식 문서** 참조 링크 포함
- **Copy-paste 가능한** SQL 예제

### 10.3 Supabase 특화

일반 PostgreSQL best practices에 더해:
- **RLS (Row Level Security)** 강조 - 멀티테넌트 필수
- **auth.uid()** 함수 최적화 - Supabase 인증 통합
- **Connection Pooling** 중시 - Serverless 환경 대응

---

## 참고 문서

### PostgreSQL 공식 문서
- [Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
- [Index-Only Scans](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)
- [Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)
- [Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html)
- [Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)

### PostgreSQL Wiki
- [Number Of Database Connections](https://wiki.postgresql.org/wiki/Number_Of_Database_Connections)
- [Tuning Your PostgreSQL Server](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)

### Supabase 문서
- [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [RLS Performance Best Practices](https://supabase.com/docs/guides/troubleshooting/rls-performance-and-best-practices-Z5Jjwv)
