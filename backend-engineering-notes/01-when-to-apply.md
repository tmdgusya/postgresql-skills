# PostgreSQL 최적화 기법 적용 시점 가이드

백엔드 엔지니어를 위한 실무 중심 PostgreSQL 최적화 의사결정 가이드

---

## 목차

1. [프로젝트 규모별 최적화 로드맵](#1-프로젝트-규모별-최적화-로드맵)
2. [성능 문제 징후와 해결책 매핑](#2-성능-문제-징후와-해결책-매핑)
3. [카테고리별 적용 시나리오](#3-카테고리별-적용-시나리오)
4. [체크리스트](#4-체크리스트)

---

## 1. 프로젝트 규모별 최적화 로드맵

### Phase 1: 스타트업 MVP (DAU < 1,000)

**우선순위: 속도와 개발 생산성**

#### 필수 적용 사항
- **Schema Design**
  - Primary Key는 무조건 설정 (UUID vs Serial 선택은 글로벌 확장성 고려)
  - Foreign Key 제약조건 설정 (데이터 무결성)
  - 소문자 컬럼명 사용 (따옴표 지옥 방지)

- **Security**
  - 기본 권한 설정 (public schema 접근 제한)
  - 애플리케이션 전용 role 생성
  - 환경변수로 credential 관리

- **Connection Management**
  - 기본 connection pooling 설정 (pgBouncer or 애플리케이션 레벨)
  - `idle_in_transaction_session_timeout` 설정 (10초 권장)

#### 아직 필요 없는 것
- ❌ RLS (Row Level Security) - 애플리케이션 레벨에서 충분
- ❌ Composite Index - 단일 컬럼 인덱스로 시작
- ❌ Partitioning - 테이블 크기 < 10GB면 불필요
- ❌ 고급 모니터링 - 기본 로그로 충분

**예시 시나리오**
```
상황: SaaS MVP 개발 중, 사용자 테이블 생성
결정:
✅ id (UUID primary key)
✅ created_at (timestamp with time zone)
✅ email (unique index)
❌ (name, email) composite index - 아직 검색 패턴 불명확
❌ Partitioning - 사용자 < 10,000명 예상
```

---

### Phase 2: 성장 단계 (DAU 1,000 ~ 50,000)

**우선순위: 병목 지점 해소**

#### 이 시점에 추가 적용
- **Query Performance**
  - Missing Index 주기적 체크 (`pg_stat_user_tables`, `pg_stat_user_indexes`)
  - Slow Query 로깅 활성화 (`log_min_duration_statement = 1000`)
  - EXPLAIN ANALYZE 습관화

- **Connection Management**
  - Connection Pool 크기 최적화
  - Prepared Statement 활용 (같은 쿼리 반복 시)
  - Statement Timeout 설정 (30초 권장)

- **Data Access Patterns**
  - N+1 쿼리 탐지 및 해결 (JOIN or batch fetch)
  - Pagination 구현 (Cursor-based for infinite scroll)
  - Batch Insert 적용 (대량 데이터 입력 시)

#### 조건부 적용
- **Composite Index**: 복합 조건 쿼리 빈도 > 100회/분
- **Partial Index**: 특정 조건 레코드만 자주 조회 (예: status = 'active')
- **JSONB Indexing**: JSONB 컬럼 조회 > 50회/분

**예시 시나리오**
```
상황: 대시보드 로딩 5초 → 사용자 이탈 증가
분석: SELECT * FROM orders WHERE user_id = ? AND status = 'pending' (1초 소요)
해결:
✅ CREATE INDEX idx_orders_user_status ON orders(user_id, status)
✅ WHERE status = 'pending' 조건이 10%만 해당 → Partial Index 고려
결과: 1초 → 50ms
```

---

### Phase 3: 대규모 서비스 (DAU > 50,000)

**우선순위: 안정성과 확장성**

#### 이 시점에 필수
- **Security & RLS**
  - RLS 정책 구현 (멀티테넌트 환경)
  - RLS Performance 최적화 (인덱스 + Security Definer 함수)

- **Schema Design**
  - Partitioning (테이블 > 100GB or 연도별 데이터 분리)
  - 적절한 데이터 타입 선택 (text vs varchar, timestamp vs timestamptz)

- **Concurrency & Locking**
  - Advisory Lock (분산 잠금 필요 시)
  - FOR UPDATE SKIP LOCKED (작업 큐 구현)
  - 트랜잭션 최소화 (< 100ms 목표)

- **Monitoring**
  - pg_stat_statements 활성화
  - 자동 VACUUM/ANALYZE 튜닝
  - 복제 지연 모니터링 (Replica 사용 시)

- **Advanced Features**
  - Full-Text Search (검색 기능)
  - Covering Index (쿼리 최적화)
  - Materialized View (복잡한 집계 쿼리)

**예시 시나리오**
```
상황: 멀티테넌트 SaaS, 조직별 데이터 격리 필요
문제: 애플리케이션 레벨 필터링 → 개발자 실수로 데이터 노출 위험
해결:
✅ RLS 정책 구현
   CREATE POLICY tenant_isolation ON data_table
   USING (organization_id = current_setting('app.current_org_id')::uuid);
✅ organization_id 컬럼 인덱스 추가 (RLS 성능)
✅ Security Definer 함수로 복잡한 권한 로직 캡슐화
```

---

## 2. 성능 문제 징후와 해결책 매핑

### 🔴 응답 시간 급증 (1초 → 5초+)

#### 징후
- 특정 API endpoint 타임아웃
- 사용자 대시보드 로딩 지연
- CPU 사용률 정상, 디스크 I/O 급증

#### 진단
```sql
-- 1. Slow Query 확인
SELECT query, calls, mean_exec_time, max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- 2. Missing Index 확인
SELECT
  schemaname,
  tablename,
  seq_scan,
  seq_tup_read,
  idx_scan,
  seq_tup_read / seq_scan AS avg_seq_tup_read
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 10;
```

#### 해결책 우선순위
1. **Missing Index 추가** (가장 빠른 효과)
2. **N+1 쿼리 제거** (JOIN or eager loading)
3. **Partial Index** (조건부 조회가 많을 때)
4. **Query Rewrite** (서브쿼리 → JOIN)

---

### 🔴 Connection Pool 고갈

#### 징후
- "sorry, too many clients already" 에러
- Connection 획득 대기 시간 증가
- 정상 요청도 실패

#### 진단
```sql
-- 현재 Connection 상태
SELECT
  state,
  COUNT(*) as count,
  MAX(EXTRACT(EPOCH FROM (now() - state_change))) as max_duration_seconds
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state;

-- Idle in Transaction 찾기
SELECT pid, usename, state, query_start, state_change, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > interval '5 seconds';
```

#### 해결책
1. **idle_in_transaction_session_timeout** 설정 (10초)
2. **Connection Pool 크기** 증가 (CPU 코어 수 * 2 ~ 4)
3. **Long-running Query** 최적화 or 타임아웃 설정
4. **Prepared Statement** 재사용 활성화

**적용 시나리오**
```
상황: Kubernetes pod 증가 → Connection 부족
계산:
- Pod 10개 * Connection Pool 20개 = 200 connections
- Postgres max_connections = 100 (기본값) ❌
해결:
✅ max_connections = 300 (여유 포함)
✅ pgBouncer 도입 (Transaction pooling)
✅ Replica 추가 (읽기 부하 분산)
```

---

### 🔴 Deadlock 발생

#### 징후
- "deadlock detected" 에러
- 동시 트랜잭션 환경에서 간헐적 실패
- 특정 테이블 업데이트 시 타임아웃

#### 진단
```sql
-- Deadlock 로그 확인
-- postgresql.conf: log_lock_waits = on

-- 현재 Lock 대기 상황
SELECT
  blocked_locks.pid AS blocked_pid,
  blocked_activity.usename AS blocked_user,
  blocking_locks.pid AS blocking_pid,
  blocking_activity.usename AS blocking_user,
  blocked_activity.query AS blocked_statement,
  blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

#### 해결책
1. **트랜잭션 순서 통일** (항상 같은 테이블 순서로 UPDATE)
2. **트랜잭션 시간 최소화** (< 100ms)
3. **Advisory Lock** 사용 (명시적 잠금 순서)
4. **SELECT FOR UPDATE NOWAIT** (즉시 실패 후 재시도)

**적용 시나리오**
```
상황: 재고 차감 + 주문 생성 동시 처리 시 Deadlock
원인:
- Transaction A: UPDATE inventory → INSERT orders
- Transaction B: INSERT orders → UPDATE inventory
해결:
✅ 순서 통일: 항상 inventory → orders 순서
✅ Advisory Lock 적용
   SELECT pg_advisory_xact_lock(product_id);
   UPDATE inventory ...
   INSERT INTO orders ...
```

---

### 🔴 테이블 Bloat (디스크 용량 급증)

#### 징후
- 디스크 사용량 증가하지만 레코드 수는 정체
- SELECT 성능 저하 (Sequential Scan 시간 증가)
- VACUUM 실행해도 용량 회수 안됨

#### 진단
```sql
-- Bloat 확인
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
  n_dead_tup,
  n_live_tup,
  ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_tup_percent
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

#### 해결책
1. **VACUUM FULL** (일회성, 테이블 잠금 발생)
2. **autovacuum 튜닝** (더 자주 실행)
3. **UPDATE 패턴 변경** (불필요한 업데이트 제거)
4. **HOT Update** 활용 (같은 페이지 내 업데이트)

**적용 시나리오**
```
상황: 실시간 조회수 업데이트 → 1시간 만에 테이블 크기 2배
원인: UPDATE views = views + 1 (매번 새 row version 생성)
해결:
✅ Redis 캐시로 변경 (1시간마다 배치 업데이트)
✅ Partitioning (일별 테이블 분리 후 구 데이터 삭제)
✅ autovacuum_vacuum_scale_factor = 0.05 (기본 0.2 → 더 자주 실행)
```

---

### 🔴 복제 지연 (Replica Lag)

#### 징후
- Primary 쓰기 후 Replica에서 조회 시 데이터 없음
- Monitoring에서 Replication Lag > 5초
- Replica CPU 사용률 높음

#### 진단
```sql
-- Primary에서 실행
SELECT
  client_addr,
  state,
  sent_lsn,
  write_lsn,
  replay_lsn,
  sync_state,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;

-- Replica에서 실행
SELECT
  now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

#### 해결책
1. **Replica 스펙 업그레이드** (CPU, 디스크 I/O)
2. **Primary Write 부하 분산** (배치 작업 시간대 조정)
3. **Synchronous Replication 비활성화** (성능 > 일관성)
4. **Replica 인덱스 추가** (RLS 정책 실행 시 필요)

---

## 3. 카테고리별 적용 시나리오

### Query Performance

#### Composite Index
**언제 사용?**
- WHERE 절에 2개 이상 컬럼 조건이 **항상 함께** 사용됨
- 복합 정렬 필요 (ORDER BY col1, col2)
- 쿼리 빈도 > 100회/분

**적용 예시**
```sql
-- 시나리오: 사용자 활동 로그 조회
-- 쿼리: SELECT * FROM activity_logs WHERE user_id = ? AND created_at > ?
-- 빈도: 1,000회/분

-- 실행 계획 확인
EXPLAIN ANALYZE
SELECT * FROM activity_logs
WHERE user_id = 'abc123' AND created_at > now() - interval '7 days';

-- 개선 전: Seq Scan (800ms)
-- 개선 후: Index Scan (15ms)
CREATE INDEX idx_activity_user_time ON activity_logs(user_id, created_at DESC);
```

**주의사항**
- 컬럼 순서 중요: 선택도(Selectivity) 높은 컬럼이 먼저
- user_id (선택도 높음) → created_at (선택도 낮음)

---

#### Covering Index
**언제 사용?**
- SELECT 컬럼이 적고 고정적임
- 인덱스만으로 쿼리 완결 가능
- 테이블 크기 > 10GB

**적용 예시**
```sql
-- 시나리오: 주문 목록 API (id, status, created_at만 반환)
-- 문제: Index Scan 후 Heap Fetch 발생 (Disk I/O 2배)

-- 개선 전
CREATE INDEX idx_orders_user ON orders(user_id);
-- EXPLAIN: Index Scan + Heap Fetch

-- 개선 후 (Covering Index)
CREATE INDEX idx_orders_user_covering ON orders(user_id) INCLUDE (status, created_at);
-- EXPLAIN: Index Only Scan ✅
```

---

#### Partial Index
**언제 사용?**
- 특정 조건 레코드만 자주 조회 (예: active=true가 전체의 5%)
- 인덱스 크기 축소 필요
- 쓰기 성능 중요 (인덱스 업데이트 부하 감소)

**적용 예시**
```sql
-- 시나리오: 전체 주문 중 pending 상태만 대시보드 표시
-- 데이터: pending 5% (10만건), completed 95% (190만건)

-- 개선 전: 전체 인덱스 (2GB)
CREATE INDEX idx_orders_status ON orders(status);

-- 개선 후: Partial Index (100MB)
CREATE INDEX idx_orders_pending ON orders(user_id, created_at)
WHERE status = 'pending';

-- 쿼리에서 조건 명시 필수
SELECT * FROM orders WHERE status = 'pending' AND user_id = ?;
```

---

### Connection Management

#### Connection Pooling 크기 최적화
**언제 조정?**
- Connection wait time > 100ms
- "too many clients" 에러
- 애플리케이션 스케일아웃 시

**계산 공식**
```
적정 Pool Size = (CPU 코어 수 * 2) ~ (CPU 코어 수 * 4)

예시:
- RDS db.t3.medium (2 vCPU)
- 권장 Pool Size: 4 ~ 8 connections per application instance
- Application instances: 10개
- Total connections: 40 ~ 80
- Postgres max_connections: 100 이상 설정
```

**적용 예시**
```javascript
// Node.js - pg Pool 설정
const pool = new Pool({
  max: 20,                    // 최대 connection 수
  min: 2,                     // 최소 유지 connection
  idleTimeoutMillis: 30000,   // 30초 idle 후 반환
  connectionTimeoutMillis: 10000, // 10초 내 connection 획득 실패 시 에러
});
```

---

#### Prepared Statement
**언제 사용?**
- 같은 쿼리를 다른 파라미터로 반복 실행
- SQL Injection 방지 필요
- 파싱 오버헤드 제거 (쿼리 빈도 > 10회/초)

**적용 예시**
```javascript
// 개선 전: 매번 쿼리 파싱
for (let userId of userIds) {
  await pool.query(`SELECT * FROM users WHERE id = '${userId}'`); // ❌ SQL Injection 위험
}

// 개선 후: Prepared Statement
const stmt = await pool.query('PREPARE user_query AS SELECT * FROM users WHERE id = $1');
for (let userId of userIds) {
  await pool.query('EXECUTE user_query($1)', [userId]); // ✅ 파싱 1회, 실행 N회
}

// 또는 파라미터화 쿼리
for (let userId of userIds) {
  await pool.query('SELECT * FROM users WHERE id = $1', [userId]); // ✅ 라이브러리가 자동 prepare
}
```

---

### Security & RLS

#### Row Level Security
**언제 사용?**
- 멀티테넌트 환경 (조직/팀별 데이터 격리)
- 민감 데이터 접근 제어 (사용자 권한별)
- 애플리케이션 레벨 보안 불충분

**적용 시나리오**
```sql
-- 시나리오: SaaS 플랫폼, 조직별 데이터 격리

-- 1. RLS 활성화
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- 2. 정책 생성
CREATE POLICY tenant_isolation ON documents
  USING (organization_id = current_setting('app.current_org_id')::uuid);

-- 3. 애플리케이션에서 세션 변수 설정
-- 모든 쿼리 전에 실행
SET app.current_org_id = 'org-uuid-here';

-- 4. 성능 최적화: organization_id 인덱스 필수
CREATE INDEX idx_docs_org ON documents(organization_id);
```

**주의사항**
- RLS 정책은 모든 쿼리에 WHERE 절 추가 → 인덱스 필수
- `current_setting()` 호출 비용 → Security Definer 함수로 캐싱

---

### Schema Design

#### Partitioning
**언제 사용?**
- 테이블 크기 > 100GB
- 시계열 데이터 (연도/월별 분리)
- 구 데이터 정기 삭제 필요

**적용 예시**
```sql
-- 시나리오: 로그 테이블 (월 10GB 증가, 6개월 후 삭제)

-- 1. Partitioned 테이블 생성
CREATE TABLE logs (
  id BIGSERIAL,
  user_id UUID,
  event_type TEXT,
  created_at TIMESTAMPTZ NOT NULL,
  data JSONB
) PARTITION BY RANGE (created_at);

-- 2. 파티션 생성 (월별)
CREATE TABLE logs_2024_01 PARTITION OF logs
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE logs_2024_02 PARTITION OF logs
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- 3. 인덱스는 각 파티션에 자동 생성
CREATE INDEX idx_logs_user ON logs(user_id);

-- 4. 구 데이터 삭제 (DROP TABLE로 즉시 삭제)
DROP TABLE logs_2023_06; -- TRUNCATE보다 빠름
```

**효과**
- Query: WHERE created_at >= '2024-01-01' → 해당 파티션만 스캔 (Partition Pruning)
- DELETE: DROP TABLE로 즉시 삭제 (VACUUM 불필요)

---

#### 적절한 데이터 타입 선택
**언제 고려?**
- 스키마 설계 초기
- 디스크 용량 최적화 필요
- 인덱스 크기 최소화

**선택 가이드**
```sql
-- Text vs Varchar
-- ✅ 추천: TEXT (Postgres에서 성능 차이 없음, 길이 제한 불필요)
email TEXT NOT NULL;

-- ❌ 피하기: VARCHAR(255) (임의의 제한, 변경 시 마이그레이션)
email VARCHAR(255);

-- Timestamp
-- ✅ 추천: TIMESTAMPTZ (시간대 정보 포함, UTC 저장)
created_at TIMESTAMPTZ DEFAULT now();

-- ❌ 피하기: TIMESTAMP (시간대 정보 없음, 혼란 유발)
created_at TIMESTAMP;

-- Integer 타입
-- Small: SMALLINT (-32K ~ 32K) - 상태 코드, enum
-- Medium: INTEGER (-2B ~ 2B) - 일반 ID
-- Large: BIGINT (-9E18 ~ 9E18) - 자동 증가 ID
-- ✅ 추천: BIGSERIAL (자동 증가 + 64bit, 오버플로우 걱정 없음)
id BIGSERIAL PRIMARY KEY;
```

---

### Concurrency & Locking

#### Advisory Lock
**언제 사용?**
- 분산 환경에서 동시 실행 방지 (배치 작업)
- 애플리케이션 레벨 임계 영역 보호
- Redis 없이 간단한 분산 잠금 필요

**적용 예시**
```sql
-- 시나리오: 일일 리포트 생성 (중복 실행 방지)

-- 1. 잠금 시도 (즉시 반환)
SELECT pg_try_advisory_lock(12345);
-- true: 잠금 획득 성공 → 작업 진행
-- false: 이미 실행 중 → 종료

-- 2. 작업 실행
INSERT INTO daily_reports SELECT ...;

-- 3. 잠금 해제
SELECT pg_advisory_unlock(12345);

-- 트랜잭션 범위 잠금 (자동 해제)
BEGIN;
SELECT pg_advisory_xact_lock(12345); -- 트랜잭션 종료 시 자동 해제
-- 작업 수행
COMMIT;
```

**주의사항**
- Lock ID는 애플리케이션 전체에서 유일해야 함
- 세션 종료 시 자동 해제되지만 명시적 해제 권장

---

#### FOR UPDATE SKIP LOCKED
**언제 사용?**
- 작업 큐 구현 (여러 워커가 동시 처리)
- 동시성 높은 환경에서 레코드 선점
- Deadlock 방지

**적용 예시**
```sql
-- 시나리오: 이메일 발송 큐 (여러 워커가 동시 처리)

-- 개선 전: Deadlock 발생
BEGIN;
SELECT * FROM email_queue WHERE status = 'pending' LIMIT 10 FOR UPDATE;
-- 다른 워커와 충돌 → Deadlock

-- 개선 후: SKIP LOCKED
BEGIN;
SELECT * FROM email_queue
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED; -- 이미 잠긴 행은 건너뜀

-- 상태 업데이트
UPDATE email_queue SET status = 'processing' WHERE id = ANY(?);
COMMIT;

-- 이메일 발송 후
UPDATE email_queue SET status = 'sent' WHERE id = ANY(?);
```

**효과**
- 여러 워커가 동시 실행해도 서로 다른 레코드 처리
- Deadlock 제거
- 처리량 선형 증가

---

### Data Access Patterns

#### N+1 쿼리 해결
**언제 발생?**
- ORM 사용 시 관계 자동 로딩
- 반복문 내에서 쿼리 실행
- API 응답 시간 > 1초

**탐지 방법**
```javascript
// 로그에서 같은 패턴 쿼리 반복 발견
SELECT * FROM posts WHERE user_id = 1;
SELECT * FROM posts WHERE user_id = 2;
SELECT * FROM posts WHERE user_id = 3;
// ... 100번 반복
```

**해결 예시**
```javascript
// 개선 전: N+1 쿼리
const users = await db.query('SELECT * FROM users LIMIT 100');
for (let user of users) {
  user.posts = await db.query('SELECT * FROM posts WHERE user_id = $1', [user.id]);
}
// 총 쿼리: 1 + 100 = 101개

// 해결 1: JOIN
const result = await db.query(`
  SELECT u.*, p.*
  FROM users u
  LEFT JOIN posts p ON p.user_id = u.id
  WHERE u.id = ANY($1)
`, [userIds]);
// 총 쿼리: 1개

// 해결 2: Batch Fetch (DataLoader 패턴)
const users = await db.query('SELECT * FROM users LIMIT 100');
const userIds = users.map(u => u.id);
const posts = await db.query('SELECT * FROM posts WHERE user_id = ANY($1)', [userIds]);
// 총 쿼리: 2개

// posts를 user_id별로 그룹핑
const postsByUser = groupBy(posts, 'user_id');
users.forEach(user => {
  user.posts = postsByUser[user.id] || [];
});
```

---

#### Batch Insert
**언제 사용?**
- 대량 데이터 입력 (> 1,000건)
- CSV Import
- 초기 데이터 시딩

**적용 예시**
```sql
-- 개선 전: 개별 INSERT (10,000건 = 30초)
INSERT INTO products (name, price) VALUES ('Product 1', 100);
INSERT INTO products (name, price) VALUES ('Product 2', 200);
-- ... 10,000번 반복

-- 개선 후: 배치 INSERT (10,000건 = 1초)
INSERT INTO products (name, price) VALUES
  ('Product 1', 100),
  ('Product 2', 200),
  -- ... 1000개씩 배치
  ('Product 1000', 1000);

-- 또는 COPY (가장 빠름)
COPY products(name, price) FROM STDIN WITH CSV;
Product 1,100
Product 2,200
...
\.
```

**Node.js 예시**
```javascript
// pg-format 라이브러리 사용
const format = require('pg-format');

const values = products.map(p => [p.name, p.price]);
const sql = format('INSERT INTO products (name, price) VALUES %L', values);
await db.query(sql);

// 또는 직접 구현 (1000개씩 분할)
for (let i = 0; i < products.length; i += 1000) {
  const batch = products.slice(i, i + 1000);
  const placeholders = batch.map((_, idx) => `($${idx * 2 + 1}, $${idx * 2 + 2})`).join(',');
  const values = batch.flatMap(p => [p.name, p.price]);
  await db.query(`INSERT INTO products (name, price) VALUES ${placeholders}`, values);
}
```

---

## 4. 체크리스트

### 프로젝트 시작 시 (MVP)

#### Schema Design
- [ ] 모든 테이블에 Primary Key 설정 (UUID or BIGSERIAL)
- [ ] Foreign Key 제약조건 설정
- [ ] NOT NULL 제약조건 필수 컬럼에 설정
- [ ] 컬럼명 소문자 사용 (snake_case)
- [ ] TIMESTAMPTZ 사용 (TIMESTAMP 아님)
- [ ] TEXT 사용 (VARCHAR 대신)

#### Security
- [ ] 애플리케이션 전용 role 생성 (postgres 사용 금지)
- [ ] public schema 권한 제한
- [ ] SSL/TLS 연결 설정
- [ ] 환경변수로 credential 관리

#### Connection
- [ ] Connection pooling 설정 (최소값)
- [ ] `idle_in_transaction_session_timeout = 10s` 설정
- [ ] `statement_timeout = 30s` 설정

---

### 성장 단계 (트래픽 증가 시)

#### Monitoring
- [ ] Slow query 로깅 활성화 (`log_min_duration_statement = 1000`)
- [ ] pg_stat_statements 확장 설치
- [ ] 주간 missing index 리포트 자동화
- [ ] Connection pool 사용률 모니터링

#### Performance
- [ ] 자주 사용하는 WHERE 절 컬럼 인덱스 추가
- [ ] N+1 쿼리 탐지 및 해결
- [ ] Pagination 구현 (Cursor-based)
- [ ] EXPLAIN ANALYZE 습관화

#### Schema
- [ ] Foreign Key 컬럼 인덱스 추가
- [ ] Composite index 검토 (복합 조건 쿼리)
- [ ] Partial index 검토 (조건부 조회)

---

### 대규모 서비스 (안정성 중시)

#### Advanced Performance
- [ ] Covering index 적용 (자주 조회하는 컬럼)
- [ ] Partitioning 검토 (테이블 > 100GB)
- [ ] Materialized View 검토 (복잡한 집계)
- [ ] Connection Pool 크기 최적화

#### Security
- [ ] RLS 정책 구현 (멀티테넌트)
- [ ] RLS 성능 최적화 (인덱스 + Security Definer)
- [ ] Audit 로깅 설정

#### Concurrency
- [ ] Advisory Lock 적용 (배치 작업)
- [ ] FOR UPDATE SKIP LOCKED (작업 큐)
- [ ] 트랜잭션 시간 측정 및 최적화 (< 100ms)
- [ ] Deadlock 모니터링 및 방지

#### High Availability
- [ ] Read Replica 추가
- [ ] Replication lag 모니터링
- [ ] Failover 테스트
- [ ] Backup 자동화 및 복구 테스트

---

### 배포 전 체크리스트

#### Performance
- [ ] EXPLAIN ANALYZE 주요 쿼리 검증
- [ ] Missing index 확인
- [ ] N+1 쿼리 제거 확인
- [ ] Slow query 테스트 (부하 테스트)

#### Security
- [ ] SQL Injection 방어 확인 (Prepared statement)
- [ ] RLS 정책 테스트 (권한별)
- [ ] Credential rotation 테스트
- [ ] Backup 복구 테스트

#### Monitoring
- [ ] 알람 설정 (Connection, Replication lag, Disk)
- [ ] Dashboard 구축 (Grafana + prometheus-postgres-exporter)
- [ ] On-call 대응 매뉴얼 작성

---

## 참고 자료

### 성능 측정 쿼리

```sql
-- 1. 가장 느린 쿼리 TOP 10
SELECT
  query,
  calls,
  total_exec_time,
  mean_exec_time,
  max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- 2. 인덱스 사용률
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- 3. 테이블 크기
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
  pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- 4. Cache Hit Rate (> 99% 목표)
SELECT
  sum(heap_blks_read) as heap_read,
  sum(heap_blks_hit) as heap_hit,
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as cache_hit_ratio
FROM pg_statio_user_tables;

-- 5. Connection 상태
SELECT
  datname,
  usename,
  application_name,
  client_addr,
  state,
  COUNT(*)
FROM pg_stat_activity
GROUP BY datname, usename, application_name, client_addr, state;
```

---

## 결론

PostgreSQL 최적화는 **"언제 적용할 것인가"**가 핵심입니다.

### 의사결정 원칙
1. **측정 없이 최적화하지 말 것** - EXPLAIN ANALYZE, pg_stat_statements 활용
2. **조기 최적화 지양** - MVP는 단순하게, 문제 발생 시 대응
3. **병목 지점 우선 해결** - 가장 느린 쿼리부터 개선
4. **비용 대비 효과** - 10분 투자로 80% 개선 가능한 작업 우선

### 단계별 우선순위
```
Phase 1 (MVP): Schema 무결성 + 기본 보안
Phase 2 (성장): Index + N+1 해결 + Connection Pool
Phase 3 (대규모): RLS + Partitioning + Concurrency 최적화
```

이 가이드를 바탕으로 **프로젝트 단계에 맞는 최적화**를 적용하세요.
