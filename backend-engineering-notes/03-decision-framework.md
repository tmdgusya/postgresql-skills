# PostgreSQL 기술 선택 의사결정 프레임워크

백엔드 엔지니어를 위한 실전 기술 선택 가이드

---

## 1. 기본 키(Primary Key) 전략 선택 가이드

### 의사결정 트리

```
데이터베이스 기본 키 전략 선택
│
├─ Q1: 분산 시스템 환경인가?
│   ├─ NO → Q2로
│   └─ YES → Q3로
│
├─ Q2: 테이블 크기가 21억 row를 초과할 가능성?
│   ├─ NO → ✅ INT (4 bytes) - 가장 효율적
│   └─ YES → ✅ BIGINT (8 bytes) - 안전하고 단순
│
└─ Q3: 시간 기반 정렬이 필수인가?
    ├─ YES → Q4로
    └─ NO → ✅ UUID v4 - 완전한 무작위성, 보안성 우수

    Q4: 인덱스 성능이 매우 중요한가?
    ├─ YES → ✅ UUID v7 - 시간 기반 정렬 + 높은 인덱스 효율
    └─ NO → UUID v4와 v7 중 선택 가능
```

### 비교표

| 전략 | 크기 | 인덱스 효율 | 분산 환경 | 정렬 가능 | 보안성 | 권장 사용처 |
|------|------|------------|----------|----------|--------|------------|
| **INT** | 4 bytes | ⭐⭐⭐⭐⭐ | ❌ | ✅ | ⭐ | 소규모, 단일 DB |
| **BIGINT** | 8 bytes | ⭐⭐⭐⭐⭐ | ❌ | ✅ | ⭐ | 대규모, 단일 DB |
| **UUID v4** | 16 bytes | ⭐⭐ | ✅ | ❌ | ⭐⭐⭐⭐⭐ | 분산 시스템, 보안 중요 |
| **UUID v7** | 16 bytes | ⭐⭐⭐⭐ | ✅ | ✅ | ⭐⭐⭐⭐ | 분산 + 시계열 데이터 |

### 트레이드오프 분석

**BIGINT (연속 숫자)**
- ✅ 장점: 최고의 인덱스 성능, 작은 크기, 간단한 디버깅
- ❌ 단점: 중앙 집중식 시퀀스 필요, URL 노출 시 보안 위험, 샤딩 복잡
- 💡 최적: 모놀리식 아키텍처, 대규모 단일 DB

**UUID v4 (완전 무작위)**
- ✅ 장점: 진정한 분산 생성, 충돌 거의 없음, URL 추측 불가
- ❌ 단점: 무작위 인덱스 삽입으로 B-tree 파편화, 큰 크기
- 💡 최적: 마이크로서비스, 멀티 리전, 보안 중요 API

**UUID v7 (시간 기반)**
- ✅ 장점: 시간순 정렬, 인덱스 효율 개선, 분산 생성 가능
- ❌ 단점: 아직 표준화 진행 중, 같은 밀리초 내 약한 무작위성
- 💡 최적: 시계열 데이터, 이벤트 소싱, 분산 로깅

### 실전 SQL 예제

```sql
-- BIGINT (SERIAL)
CREATE TABLE users_bigint (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);

-- UUID v4
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE TABLE users_uuid4 (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) NOT NULL
);

-- UUID v7 (PostgreSQL 13+에서 커스텀 함수 필요)
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE OR REPLACE FUNCTION uuid_generate_v7()
RETURNS UUID AS $$
DECLARE
    unix_ts_ms BIGINT;
    uuid_bytes BYTEA;
BEGIN
    unix_ts_ms := (EXTRACT(EPOCH FROM clock_timestamp()) * 1000)::BIGINT;
    uuid_bytes := E'\\x' || lpad(to_hex(unix_ts_ms), 12, '0') ||
                  encode(gen_random_bytes(10), 'hex');
    RETURN uuid_bytes::UUID;
END;
$$ LANGUAGE plpgsql;

CREATE TABLE users_uuid7 (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    email VARCHAR(255) NOT NULL
);
```

---

## 2. 인덱스 타입 선택 플로우차트

### 의사결정 플로우

```
인덱스가 필요한 쿼리 패턴 분석
│
├─ Q1: 어떤 연산을 주로 하는가?
│
├─ [단일 값 동등 비교: WHERE col = value]
│   └─ 데이터 분포는?
│       ├─ 고유값 많음 (cardinality 높음) → ✅ B-TREE (기본)
│       └─ 고유값 적음 (cardinality 낮음) → ✅ HASH (= 만 사용 시)
│
├─ [범위 검색: WHERE col BETWEEN / > / < ]
│   └─ 데이터 특성은?
│       ├─ 일반 데이터 → ✅ B-TREE
│       ├─ 시계열, 대용량 (수억 row) → ✅ BRIN (Block Range Index)
│       └─ 시간/날짜 범위 + 물리적 정렬 → ✅ BRIN (50-1000배 작음)
│
├─ [전문 검색: LIKE '%keyword%', tsvector]
│   └─ ✅ GIN (Generalized Inverted Index)
│
├─ [JSON/JSONB 검색: jsonb @> query]
│   └─ 쿼리 패턴은?
│       ├─ 포함 여부 체크 (@>, ?, ?&) → ✅ GIN (기본)
│       └─ 특정 경로만 (jsonb->'key') → ✅ B-TREE (표현식 인덱스)
│
├─ [배열 연산: ARRAY @> element, && overlap]
│   └─ ✅ GIN
│
└─ [복합 조건: WHERE a = x AND b = y]
    └─ 쿼리 다양성은?
        ├─ 고정된 조합 → ✅ B-TREE (a, b) 복합 인덱스
        └─ 다양한 조합 → ✅ GIN (다중 컬럼 GIN 또는 여러 단일 인덱스)
```

### 인덱스 타입 비교표

| 인덱스 타입 | 지원 연산자 | 크기 | 구축 속도 | 검색 속도 | 최적 사용처 |
|------------|------------|------|-----------|-----------|------------|
| **B-TREE** | `=, <, >, <=, >=, BETWEEN, IN, IS NULL` | 보통 | 빠름 | 빠름 | 범위 검색, 정렬, 대부분의 경우 |
| **HASH** | `=` 만 | 작음 | 매우 빠름 | 매우 빠름 | 동등 비교만 필요한 경우 |
| **GIN** | `@>, <@, &&, ?` (JSONB, array, tsvector) | 큼 | 느림 | 매우 빠름 | 전문 검색, JSON, 배열 |
| **BRIN** | `=, <, >, <=, >=, BETWEEN` | 매우 작음 | 매우 빠름 | 보통 | 시계열, 물리적 정렬된 대용량 |
| **GiST** | 공간 데이터, 범위 타입 | 큼 | 보통 | 보통 | 지리 정보(PostGIS), 범위 겹침 |

### 트레이드오프 분석

**B-TREE (기본 선택)**
- ✅ 범용성 최고, 정렬 지원, 유니크 제약 가능
- ❌ JSONB/배열에는 비효율적
- 💡 **모든 경우의 안전한 기본값**

**HASH**
- ✅ 동등 비교에서 B-TREE보다 약간 빠름, 작은 크기
- ❌ 범위 검색 불가, 정렬 불가, 복제 불가 (pg < 10)
- 💡 **사용 권장 안 함** (B-TREE로 충분)

**GIN (역인덱스)**
- ✅ 전문 검색 필수, JSONB 쿼리 10-100배 빠름
- ❌ 인덱스 크기 2-3배, INSERT 느림, VACUUM 부담
- 💡 **읽기 >> 쓰기 인 전문 검색/JSON 필드**

**BRIN (블록 범위 인덱스)**
- ✅ 인덱스 크기 50-1000배 작음, 시계열에 이상적
- ❌ 물리적 정렬 필요, 무작위 데이터에 비효율
- 💡 **시계열, 로그, 센서 데이터 등 append-only 테이블**

### 실전 예제

```sql
-- B-TREE: 범위 검색 + 정렬
CREATE INDEX idx_orders_created ON orders(created_at);
-- 쿼리: WHERE created_at > NOW() - INTERVAL '7 days' ORDER BY created_at

-- GIN: JSONB 검색
CREATE INDEX idx_events_data ON events USING GIN(data jsonb_path_ops);
-- 쿼리: WHERE data @> '{"type": "purchase", "amount": 100}'

-- BRIN: 시계열 대용량 (테이블 물리적 정렬 필수)
CREATE INDEX idx_logs_timestamp ON logs USING BRIN(timestamp) WITH (pages_per_range = 128);
-- 전제조건: INSERT만 발생하거나, timestamp 순으로 클러스터링
-- 쿼리: WHERE timestamp BETWEEN '2024-01-01' AND '2024-01-31'

-- 복합 B-TREE: 다중 조건
CREATE INDEX idx_users_status_created ON users(status, created_at);
-- 쿼리: WHERE status = 'active' AND created_at > '2024-01-01'

-- 표현식 인덱스: 함수 결과
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- 쿼리: WHERE LOWER(email) = 'user@example.com'
```

### 인덱스 선택 체크리스트

```
□ 쿼리 로그에서 실제 slow query 확인했는가?
□ EXPLAIN ANALYZE로 시퀀셜 스캔 확인했는가?
□ 컬럼의 cardinality가 충분히 높은가? (> 1000 고유값)
□ 테이블 크기가 인덱스를 정당화하는가? (> 10,000 rows)
□ 쓰기 성능 저하를 감수할 수 있는가?
□ 인덱스 크기가 메모리에 맞는가? (shared_buffers 고려)
```

---

## 3. 페이지네이션 방식 선택

### 의사결정 트리

```
페이지네이션 요구사항 분석
│
├─ Q1: 데이터 크기는?
│   ├─ < 10,000 rows → Q2로
│   └─ > 10,000 rows → Q3로
│
├─ Q2: 페이지 번호 UI가 필수인가?
│   ├─ YES → ✅ OFFSET/LIMIT (작은 테이블은 괜찮음)
│   └─ NO → ✅ CURSOR (더 빠름)
│
└─ Q3: 데이터가 실시간으로 변경되는가?
    ├─ NO (정적/준정적) → ✅ OFFSET/LIMIT (캐싱으로 보완)
    └─ YES → Q4로

    Q4: 사용자가 깊은 페이지를 요청하는가?
    ├─ NO (처음 몇 페이지만) → ✅ OFFSET/LIMIT (제한된 max page)
    └─ YES → Q5로

    Q5: 정렬 기준은?
    ├─ 고유 컬럼 (id, created_at) → ✅ KEYSET (최고 성능)
    └─ 복합/비고유 컬럼 → ✅ CURSOR (복잡하지만 안정적)
```

### 비교표

| 방식 | 구현 난이도 | 성능 (대규모) | 일관성 | 페이지 번호 | 양방향 | 권장 사용처 |
|------|-----------|-------------|--------|-----------|--------|------------|
| **OFFSET/LIMIT** | ⭐ (쉬움) | ❌ O(n) | ❌ 불안정 | ✅ | ✅ | 소규모, 관리자 패널 |
| **CURSOR** | ⭐⭐⭐ (보통) | ✅ O(log n) | ✅ 안정 | ❌ | ✅ (복잡) | 모바일 무한 스크롤 |
| **KEYSET** | ⭐⭐ (쉬움) | ✅ O(log n) | ✅ 안정 | ❌ | ⭐⭐ (가능) | API, 피드, 대규모 |

### 트레이드오프 분석

**OFFSET/LIMIT**
```sql
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 100;
```

- ✅ 장점: 구현 간단, 페이지 번호, 양방향 탐색 쉬움
- ❌ 단점:
  - 깊은 페이지일수록 느림 (DB가 앞 row를 모두 스캔 후 버림)
  - 데이터 변경 시 중복/누락 발생
  - OFFSET 10,000 이상에서 성능 급락
- 💡 최적: 총 페이지 < 100, 데이터 변경 적음, 작은 테이블

**CURSOR (Opaque Cursor)**
```sql
-- 첫 페이지
SELECT * FROM posts
ORDER BY id DESC
LIMIT 20;
-- cursor = base64({last_id: 980, last_created_at: "2024-01-15"})

-- 다음 페이지
SELECT * FROM posts
WHERE id < 980
ORDER BY id DESC
LIMIT 20;
```

- ✅ 장점:
  - 인덱스 활용으로 일정한 성능 (O(log n))
  - 데이터 변경에 안정적
  - 상태 캡슐화 (내부 구조 숨김)
- ❌ 단점:
  - 페이지 번호 없음 (상대적 이동만 가능)
  - 복잡한 정렬 시 구현 복잡
  - 양방향 탐색 시 추가 로직 필요
- 💡 최적: 무한 스크롤, 모바일 앱, 실시간 피드

**KEYSET (Seek Method)**
```sql
-- 다음 페이지
SELECT * FROM posts
WHERE created_at < '2024-01-15 10:30:00'
   OR (created_at = '2024-01-15 10:30:00' AND id < 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

- ✅ 장점:
  - 최고 성능 (인덱스 seek)
  - 단순한 WHERE 조건
  - 데이터 일관성 보장
- ❌ 단점:
  - 고유 정렬 키 필요 (또는 복합 키)
  - 페이지 번호 불가
  - 역방향 탐색 쿼리 변경 필요
- 💡 최적: 대규모 API, 시계열 데이터, 성능 중요

### 실전 구현 가이드

#### OFFSET/LIMIT (간단한 경우)
```typescript
// API: GET /posts?page=5&per_page=20
async function getPostsOffset(page: number, perPage: number) {
    const offset = (page - 1) * perPage;
    const [posts, total] = await Promise.all([
        db.query(
            'SELECT * FROM posts ORDER BY created_at DESC LIMIT $1 OFFSET $2',
            [perPage, offset]
        ),
        db.query('SELECT COUNT(*) FROM posts')
    ]);

    return {
        data: posts.rows,
        pagination: {
            page,
            per_page: perPage,
            total: total.rows[0].count,
            total_pages: Math.ceil(total.rows[0].count / perPage)
        }
    };
}
```

#### KEYSET (고성능)
```typescript
// API: GET /posts?after=2024-01-15T10:30:00Z_12345&limit=20
async function getPostsKeyset(after?: string, limit = 20) {
    let query = 'SELECT * FROM posts';
    const params: any[] = [limit];

    if (after) {
        const [timestamp, id] = after.split('_');
        query += ` WHERE created_at < $2 OR (created_at = $2 AND id < $3)`;
        params.push(timestamp, id);
    }

    query += ' ORDER BY created_at DESC, id DESC LIMIT $1';

    const posts = await db.query(query, params);
    const lastPost = posts.rows[posts.rows.length - 1];

    return {
        data: posts.rows,
        next_cursor: lastPost
            ? `${lastPost.created_at}_${lastPost.id}`
            : null
    };
}

-- 필수 인덱스
CREATE INDEX idx_posts_created_id ON posts(created_at DESC, id DESC);
```

#### CURSOR (범용적)
```typescript
// 복잡한 정렬에도 대응
type Cursor = {
    last_value: any;
    last_id: number;
    direction: 'next' | 'prev';
};

async function getPostsCursor(cursor?: string, limit = 20) {
    const decoded: Cursor | null = cursor
        ? JSON.parse(Buffer.from(cursor, 'base64').toString())
        : null;

    let query = 'SELECT * FROM posts';
    const params: any[] = [limit];

    if (decoded) {
        const op = decoded.direction === 'next' ? '<' : '>';
        query += ` WHERE (created_at, id) ${op} ($2, $3)`;
        params.push(decoded.last_value, decoded.last_id);
    }

    query += ' ORDER BY created_at DESC, id DESC LIMIT $1';

    const posts = await db.query(query, params);
    const lastPost = posts.rows[posts.rows.length - 1];

    const nextCursor = lastPost ? Buffer.from(JSON.stringify({
        last_value: lastPost.created_at,
        last_id: lastPost.id,
        direction: 'next'
    })).toString('base64') : null;

    return { data: posts.rows, next_cursor: nextCursor };
}
```

### 성능 벤치마크 (1M rows 테이블)

| 페이지 위치 | OFFSET/LIMIT | KEYSET | 개선율 |
|-----------|-------------|--------|--------|
| 1-10 페이지 | 5ms | 2ms | 2.5배 |
| 100 페이지 | 45ms | 2ms | 22배 |
| 1,000 페이지 | 380ms | 2ms | 190배 |
| 10,000 페이지 | 3,200ms | 2ms | 1600배 |

---

## 4. ORM vs Raw SQL vs Query Builder 선택 기준

### 의사결정 트리

```
프로젝트 요구사항 분석
│
├─ Q1: 팀의 SQL 숙련도는?
│   ├─ 초급 (SQL 생소) → Q2로
│   ├─ 중급 (기본 쿼리 가능) → Q3로
│   └─ 고급 (최적화 경험) → ✅ RAW SQL + Query Builder 혼용
│
├─ Q2: 프로젝트 복잡도는?
│   ├─ 단순 CRUD → ✅ ORM (빠른 개발)
│   └─ 복잡한 비즈니스 로직 → ⚠️ ORM (나중에 병목 가능성)
│
└─ Q3: 성능 요구사항은?
    ├─ 일반적 (< 1000 req/s) → Q4로
    └─ 높음 (> 1000 req/s) → ✅ Query Builder + Raw SQL

    Q4: 데이터베이스 독립성이 필요한가?
    ├─ YES (MySQL/PostgreSQL 전환 가능성) → ✅ ORM
    └─ NO (PostgreSQL 전용 기능 사용) → ✅ Query Builder
```

### 비교표

| 접근법 | 생산성 | 성능 | 유지보수 | 타입 안정성 | PostgreSQL 기능 | 권장 사용처 |
|-------|--------|------|----------|-----------|---------------|------------|
| **ORM** (TypeORM, Prisma) | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | 스타트업, CRUD 앱 |
| **Query Builder** (Kysely) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 대부분의 경우 |
| **Raw SQL** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | 복잡한 분석, 최적화 |

### 트레이드오프 분석

**ORM (Object-Relational Mapping)**
```typescript
// Prisma 예시
const user = await prisma.user.create({
    data: {
        email: 'user@example.com',
        posts: {
            create: [
                { title: 'First Post' },
                { title: 'Second Post' }
            ]
        }
    },
    include: { posts: true }
});
```

- ✅ 장점:
  - 마이그레이션 자동 생성
  - 타입 안정성 완벽
  - 관계 자동 처리
  - 빠른 프로토타이핑
- ❌ 단점:
  - N+1 쿼리 문제 (즉시 로딩 필수)
  - 복잡한 JOIN 시 비효율적 쿼리 생성
  - PostgreSQL 고급 기능 사용 어려움 (CTE, Window Functions)
  - 학습 곡선 (ORM 특유의 quirks)
- 💡 최적: MVP, 관리자 대시보드, 간단한 API

**Query Builder (Type-safe SQL Builder)**
```typescript
// Kysely 예시
const result = await db
    .selectFrom('user')
    .leftJoin('post', 'post.user_id', 'user.id')
    .select(['user.email', 'post.title'])
    .where('user.created_at', '>', new Date('2024-01-01'))
    .execute();
```

- ✅ 장점:
  - SQL과 1:1 대응 (예측 가능)
  - 타입 안정성 유지
  - 복잡한 쿼리도 직관적
  - PostgreSQL 모든 기능 사용 가능
  - 성능 튜닝 가능
- ❌ 단점:
  - ORM보다 코드 양 많음
  - 마이그레이션 수동 작성
  - 관계 수동 관리
- 💡 최적: 프로덕션 API, 복잡한 비즈니스 로직, 성능 중요

**Raw SQL**
```typescript
const result = await pool.query<User>(`
    SELECT u.email, array_agg(p.title) as post_titles
    FROM users u
    LEFT JOIN posts p ON p.user_id = u.id
    WHERE u.created_at > $1
    GROUP BY u.id
`, [new Date('2024-01-01')]);
```

- ✅ 장점:
  - 완벽한 제어
  - 최고 성능
  - PostgreSQL 모든 기능
  - 복잡한 분석 쿼리
- ❌ 단점:
  - SQL 인젝션 위험
  - 타입 안정성 없음 (수동 타입 정의)
  - 리팩토링 어려움
  - 데이터베이스 종속적
- 💡 최적: 리포트, 복잡한 분석, 배치 작업, 성능 최적화

### 실전 혼용 전략 (권장)

```typescript
// 1. 간단한 CRUD: Query Builder
async function getUser(id: number) {
    return db
        .selectFrom('users')
        .selectAll()
        .where('id', '=', id)
        .executeTakeFirst();
}

// 2. 복잡한 관계: Query Builder + JSON 집계
async function getUserWithPosts(id: number) {
    return db
        .selectFrom('users')
        .leftJoin('posts', 'posts.user_id', 'users.id')
        .select([
            'users.email',
            db.fn.jsonAgg('posts').as('posts')
        ])
        .where('users.id', '=', id)
        .groupBy('users.id')
        .executeTakeFirst();
}

// 3. 매우 복잡한 분석: Raw SQL
async function getMonthlyRevenue(year: number) {
    return pool.query(`
        WITH monthly_orders AS (
            SELECT
                date_trunc('month', created_at) as month,
                SUM(total) as revenue,
                COUNT(*) as order_count
            FROM orders
            WHERE EXTRACT(YEAR FROM created_at) = $1
            GROUP BY 1
        ),
        growth AS (
            SELECT
                month,
                revenue,
                LAG(revenue) OVER (ORDER BY month) as prev_revenue,
                revenue - LAG(revenue) OVER (ORDER BY month) as growth
            FROM monthly_orders
        )
        SELECT
            month,
            revenue,
            COALESCE(growth / NULLIF(prev_revenue, 0) * 100, 0) as growth_rate
        FROM growth
        ORDER BY month
    `, [year]);
}
```

### 선택 가이드라인

| 시나리오 | 추천 도구 | 이유 |
|---------|----------|------|
| 신규 스타트업 MVP | Prisma (ORM) | 빠른 개발, 타입 안전 |
| 프로덕션 API | Kysely (Query Builder) | 성능 + 타입 안전 균형 |
| 복잡한 분석 쿼리 | Raw SQL | 최고 성능, 모든 기능 |
| 마이크로서비스 | Query Builder | 서비스별 최적화 가능 |
| 레거시 마이그레이션 | Raw SQL → Query Builder | 점진적 전환 |

---

## 5. 연결 풀러(Connection Pooler) 선택

### 의사결정 트리

```
연결 풀링 요구사항 분석
│
├─ Q1: 서버 아키텍처는?
│   ├─ 서버리스 (Lambda, Cloud Functions) → ✅ PgBouncer (Transaction Mode)
│   ├─ 컨테이너 다수 (K8s, ECS) → Q2로
│   └─ 단일/소수 서버 → ✅ 애플리케이션 레벨 풀 (pg, node-postgres)
│
├─ Q2: PostgreSQL 동시 연결 제한은?
│   ├─ 넉넉함 (> 500) → ✅ 애플리케이션 레벨 풀
│   └─ 부족 (< 200) → Q3로
│
└─ Q3: 고급 기능이 필요한가?
    ├─ YES (로드 밸런싱, 읽기 복제 라우팅) → ✅ Pgpool-II
    ├─ NO (단순 풀링만) → ✅ PgBouncer
    └─ 클라우드 관리형 선호 → ✅ AWS RDS Proxy / Supabase Pooler
```

### 비교표

| 솔루션 | 모드 | 성능 | 고급 기능 | 설정 복잡도 | 메모리 사용 | 권장 사용처 |
|--------|------|------|----------|-----------|-----------|------------|
| **PgBouncer** | Transaction/Session | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | 매우 적음 | 서버리스, 높은 동시성 |
| **Pgpool-II** | Session | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 보통 | 로드 밸런싱, 캐싱 |
| **앱 레벨** (pg pool) | - | ⭐⭐⭐⭐ | ⭐ | ⭐ | 보통 | 단일 앱, 간단한 경우 |
| **RDS Proxy** | - | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ (관리형) | - | AWS 환경 |

### 트레이드오프 분석

**PgBouncer (경량 연결 풀러)**
```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=production

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
pool_mode = transaction
max_client_conn = 10000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3
```

- ✅ 장점:
  - 초경량 (C로 작성, 메모리 < 10MB)
  - Transaction Mode: 수천 개 클라이언트 → 수십 개 DB 연결
  - 서버리스에 이상적
  - 설정 간단
- ❌ 단점:
  - Transaction Mode 제약 (prepared statements 불가, session 변수 유지 안 됨)
  - 로드 밸런싱 없음
  - 쿼리 캐싱 없음
- 💡 최적: 서버리스, 마이크로서비스, 높은 동시성 (1000+ 연결)

**Pgpool-II (미들웨어 + 풀러)**
```conf
# pgpool.conf
backend_hostname0 = 'primary.db'
backend_port0 = 5432
backend_weight0 = 1

backend_hostname1 = 'replica.db'
backend_port1 = 5432
backend_weight1 = 1

load_balance_mode = on
master_slave_mode = on
connection_cache = on
```

- ✅ 장점:
  - 로드 밸런싱 (읽기를 복제본으로 자동 라우팅)
  - 쿼리 캐싱
  - 자동 페일오버
  - 연결 풀링 + 미들웨어 기능
- ❌ 단점:
  - 복잡한 설정
  - 높은 메모리 사용
  - 디버깅 어려움
  - 단일 장애점 (HA 구성 필요)
- 💡 최적: 읽기 복제본 활용, 쿼리 캐싱 필요, 레거시 HA 구성

**애플리케이션 레벨 풀 (pg, node-postgres)**
```typescript
import { Pool } from 'pg';

const pool = new Pool({
    host: 'localhost',
    port: 5432,
    database: 'mydb',
    user: 'user',
    password: 'password',
    max: 20,                    // 최대 연결 수
    idleTimeoutMillis: 30000,   // 30초 idle 후 해제
    connectionTimeoutMillis: 2000,
});

// 사용
const client = await pool.connect();
try {
    const result = await client.query('SELECT * FROM users');
    return result.rows;
} finally {
    client.release();
}
```

- ✅ 장점:
  - 설정 매우 간단
  - 애플리케이션과 같은 프로세스 (레이턴시 없음)
  - 디버깅 쉬움
  - 추가 인프라 불필요
- ❌ 단점:
  - 각 앱 인스턴스마다 별도 풀 (총 연결 = 인스턴스 수 × pool size)
  - 서버리스에 부적합 (cold start마다 연결)
  - 전역 연결 제한 관리 어려움
- 💡 최적: 단일 서버, 소규모 앱, 컨테이너 < 10개

**AWS RDS Proxy (관리형)**
```typescript
// RDS Proxy 엔드포인트 사용
const pool = new Pool({
    host: 'mydb-proxy.proxy-xxx.us-east-1.rds.amazonaws.com',
    port: 5432,
    database: 'mydb',
    // IAM 인증 가능
});
```

- ✅ 장점:
  - 완전 관리형 (설정/유지보수 없음)
  - IAM 인증 통합
  - 자동 페일오버
  - 서버리스 최적화
- ❌ 단점:
  - AWS 종속
  - 추가 비용
  - PgBouncer보다 약간 느림
- 💡 최적: AWS Lambda, ECS, 완전 관리형 선호

### 연결 수 계산 공식

```
총 필요 연결 수 = (애플리케이션 인스턴스 수) × (인스턴스당 풀 크기)

예시 1: 단일 서버
- 앱 인스턴스: 1
- 풀 크기: 20
- 총 연결: 20 → 애플리케이션 레벨 풀 적합

예시 2: 마이크로서비스 (K8s)
- 서비스 A: 10 pod × 10 = 100
- 서비스 B: 5 pod × 10 = 50
- 서비스 C: 20 pod × 10 = 200
- 총 연결: 350 → PgBouncer 필수 (25개로 축소 가능)

예시 3: 서버리스
- Lambda 동시 실행: 1000
- 각각 1 연결: 1000 → PgBouncer Transaction Mode 필수
```

### 실전 아키텍처 예시

#### 서버리스 (Lambda + PgBouncer)
```
[1000 Lambda instances]
    ↓ (각 1 연결, 총 1000)
[PgBouncer Transaction Mode]
    ↓ (25 연결로 축소)
[PostgreSQL RDS]
```

#### 마이크로서비스 (K8s + PgBouncer)
```
[Service A: 10 pods]  [Service B: 5 pods]  [Service C: 20 pods]
        ↓                    ↓                    ↓
[PgBouncer Sidecar]  [PgBouncer Sidecar]  [PgBouncer Sidecar]
        ↓                    ↓                    ↓
                    [PostgreSQL Cluster]
```

#### 읽기 복제 (Pgpool-II)
```
[애플리케이션]
    ↓
[Pgpool-II]
    ↓ (write)      ↓ (read, 로드 밸런싱)
[Primary DB]   [Replica 1]  [Replica 2]
```

---

## 6. 파티셔닝 도입 시점 판단 기준

### 의사결정 트리

```
테이블 파티셔닝 필요성 검토
│
├─ Q1: 테이블 크기는?
│   ├─ < 100GB → ❌ 파티셔닝 불필요 (인덱스 최적화로 충분)
│   ├─ 100GB - 1TB → Q2로
│   └─ > 1TB → Q3로
│
├─ Q2: 쿼리 패턴이 시간 범위 기반인가?
│   ├─ YES (주로 최근 데이터만 조회) → ✅ 시간 기반 파티셔닝 고려
│   └─ NO (전체 범위 조회) → ❌ 파티셔닝 불필요
│
└─ Q3: 다음 중 하나라도 해당하는가?
    ├─ 오래된 데이터 삭제/아카이빙 필요 → ✅ Range Partitioning
    ├─ 특정 고객/지역별 격리 필요 → ✅ List Partitioning
    ├─ 균등 분산 필요 (해시) → ✅ Hash Partitioning
    └─ 해당 없음 → Q4로

    Q4: 성능 문제가 심각한가?
    ├─ YES → ✅ 파티셔닝 (하지만 운영 복잡도 증가)
    └─ NO → ⚠️ 인덱스/쿼리 최적화 먼저 시도
```

### 도입 기준 체크리스트

```
파티셔닝을 고려해야 하는 시점:

□ 테이블 크기 > 100GB
□ 단일 테이블 스캔이 1분 이상 소요
□ 인덱스 크기가 메모리(shared_buffers)를 초과
□ VACUUM이 수 시간 이상 소요
□ 주요 쿼리가 시간 범위 기반 (최근 1개월 데이터만)
□ 오래된 데이터 삭제가 빈번 (DELETE 대신 DROP PARTITION 사용)
□ 병렬 쿼리 성능 향상 필요
□ 특정 파티션만 백업/복원 필요

⚠️ 3개 이상 체크 → 파티셔닝 도입 검토
⚠️ 5개 이상 체크 → 파티셔닝 강력 권장
```

### 파티셔닝 타입 선택

| 타입 | 사용 시나리오 | 키 예시 | 장점 |
|------|------------|--------|------|
| **Range** | 시계열 데이터, 로그, 센서 데이터 | `created_at`, `order_date` | 오래된 파티션 삭제 용이, 쿼리 프루닝 |
| **List** | 지역/국가별, 카테고리별 격리 | `country`, `status` | 특정 그룹 격리, 규제 준수 |
| **Hash** | 균등 분산, 핫스팟 방지 | `user_id` | 자동 로드 밸런싱 |

### 트레이드오프 분석

**파티셔닝의 장점**
- ✅ 쿼리 성능 향상 (파티션 프루닝)
- ✅ 유지보수 간소화 (오래된 파티션 DROP)
- ✅ 병렬 쿼리 성능 개선
- ✅ 인덱스 크기 감소 (파티션별 작은 인덱스)
- ✅ VACUUM 시간 단축

**파티셔닝의 단점**
- ❌ 설계/구현 복잡도 증가
- ❌ 파티션 관리 자동화 필요 (새 파티션 생성)
- ❌ 파티션 키 변경 불가 (재구축 필요)
- ❌ Global Index 없음 (파티션별 개별 인덱스)
- ❌ Foreign Key 제약 (파티션 테이블에서 제한적)
- ❌ 파티션 간 조인 성능 저하 가능

### 실전 예제

#### Range Partitioning (시계열 데이터)
```sql
-- 월별 파티셔닝 (로그 테이블)
CREATE TABLE logs (
    id BIGSERIAL,
    created_at TIMESTAMP NOT NULL,
    level VARCHAR(10),
    message TEXT
) PARTITION BY RANGE (created_at);

-- 파티션 생성
CREATE TABLE logs_2024_01 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE logs_2024_02 PARTITION OF logs
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- 인덱스 (자동으로 각 파티션에 생성)
CREATE INDEX idx_logs_created_at ON logs(created_at);

-- 쿼리 (자동 파티션 프루닝)
EXPLAIN SELECT * FROM logs
WHERE created_at BETWEEN '2024-01-15' AND '2024-01-20';
-- → logs_2024_01 파티션만 스캔

-- 오래된 데이터 삭제 (초고속)
DROP TABLE logs_2023_01;  -- DELETE 대신 DROP (밀리초)
```

#### List Partitioning (지역별 격리)
```sql
-- 국가별 파티셔닝
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT,
    country VARCHAR(2),
    total NUMERIC(10, 2)
) PARTITION BY LIST (country);

CREATE TABLE orders_us PARTITION OF orders
    FOR VALUES IN ('US');

CREATE TABLE orders_eu PARTITION OF orders
    FOR VALUES IN ('DE', 'FR', 'GB', 'IT');

CREATE TABLE orders_asia PARTITION OF orders
    FOR VALUES IN ('JP', 'KR', 'CN');

-- 기본 파티션 (나머지 모든 국가)
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

#### Hash Partitioning (균등 분산)
```sql
-- 사용자 ID 기반 해시 파티셔닝
CREATE TABLE user_events (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    event_type VARCHAR(50),
    created_at TIMESTAMP
) PARTITION BY HASH (user_id);

-- 8개 파티션 생성 (2의 거듭제곱 권장)
CREATE TABLE user_events_0 PARTITION OF user_events
    FOR VALUES WITH (MODULUS 8, REMAINDER 0);

CREATE TABLE user_events_1 PARTITION OF user_events
    FOR VALUES WITH (MODULUS 8, REMAINDER 1);

-- ... 나머지 6개 파티션 ...
```

### 파티션 자동 관리 (pg_partman)

```sql
-- pg_partman 확장 설치
CREATE EXTENSION pg_partman;

-- 자동 파티션 생성 설정
SELECT partman.create_parent(
    p_parent_table := 'public.logs',
    p_control := 'created_at',
    p_type := 'native',
    p_interval := 'monthly',
    p_premake := 3  -- 3개월 미리 생성
);

-- 자동 유지보수 (cron으로 실행)
-- 1. 새 파티션 생성
-- 2. 오래된 파티션 삭제 (보관 기간 초과)
SELECT partman.run_maintenance_proc();
```

### 성능 비교 (1억 row 테이블)

| 작업 | 파티셔닝 전 | 파티셔닝 후 | 개선율 |
|------|-----------|-----------|--------|
| 최근 1개월 조회 | 12초 | 0.8초 | 15배 |
| 오래된 데이터 삭제 | 2시간 | 0.1초 (DROP) | 72,000배 |
| VACUUM | 45분 | 5분 (파티션별) | 9배 |
| 인덱스 크기 | 8GB | 1GB (활성 파티션) | 8배 |

### 파티셔닝 안티패턴

```
❌ 피해야 할 경우:
1. 테이블 크기 < 100GB (오버엔지니어링)
2. 쿼리가 전체 범위를 스캔 (파티션 프루닝 효과 없음)
3. 파티션 키를 자주 업데이트 (row 이동 발생)
4. 파티션 수 > 1000 (관리 복잡도 폭발)
5. 파티션 키가 쿼리에 포함되지 않음 (모든 파티션 스캔)

✅ 대신 시도할 것:
- 인덱스 최적화 (BRIN, partial index)
- 테이블 파티셔닝 대신 샤딩 고려
- 오래된 데이터를 별도 아카이브 테이블로 이동
- Materialized View로 집계 데이터 미리 계산
```

---

## 7. 종합 의사결정 플로우차트

### 새 프로젝트 시작 시 체크리스트

```
Phase 1: 스키마 설계
├─ [ ] 기본 키 전략 결정
│   └─ 분산 환경? → UUID v7, 단일 DB? → BIGINT
├─ [ ] 인덱스 전략 수립
│   └─ 주요 쿼리 패턴 분석 → B-tree/GIN/BRIN 선택
└─ [ ] 외래 키 제약 조건 정의

Phase 2: 애플리케이션 레이어
├─ [ ] 데이터 액세스 도구 선택
│   └─ 복잡도 낮음 → ORM, 높음 → Query Builder
├─ [ ] 페이지네이션 방식 결정
│   └─ 소규모 → OFFSET, 대규모 → KEYSET
└─ [ ] 연결 풀링 설정
    └─ 서버리스 → PgBouncer, 일반 → 앱 레벨 풀

Phase 3: 프로덕션 준비
├─ [ ] 모니터링 설정 (pg_stat_statements)
├─ [ ] 백업 전략 수립
└─ [ ] 성능 테스트 (부하 테스트)

Phase 4: 스케일링 (필요 시)
├─ [ ] 읽기 복제본 추가
├─ [ ] 파티셔닝 도입 (테이블 > 100GB)
└─ [ ] 캐싱 레이어 (Redis)
```

### 성능 문제 해결 플로우

```
PostgreSQL 성능 저하 감지
│
├─ Step 1: 원인 식별
│   ├─ slow query log 확인
│   ├─ EXPLAIN ANALYZE 실행
│   └─ pg_stat_statements 분석
│
├─ Step 2: 쿼리 최적화
│   ├─ 시퀀셜 스캔? → 인덱스 추가
│   ├─ 인덱스 있는데 안 씀? → 통계 업데이트 (ANALYZE)
│   └─ 복잡한 JOIN? → 쿼리 재작성 또는 Materialized View
│
├─ Step 3: 인덱스 최적화
│   ├─ 사용 안 하는 인덱스 삭제
│   ├─ 중복 인덱스 제거
│   └─ 부분 인덱스 고려
│
├─ Step 4: 스키마 재검토
│   ├─ 정규화/역정규화 균형
│   ├─ JSONB → 개별 컬럼 분리 고려
│   └─ 파티셔닝 도입 검토
│
└─ Step 5: 인프라 스케일링
    ├─ 수직 확장 (CPU/메모리 증설)
    ├─ 읽기 복제본 추가
    └─ 연결 풀러 도입/튜닝
```

---

## 요약: 핵심 의사결정 매트릭스

| 고려 사항 | 소규모 (<10K rows) | 중규모 (10K-1M) | 대규모 (>1M) |
|----------|------------------|----------------|------------|
| **기본 키** | INT | BIGINT | UUID v7 (분산 시) |
| **인덱스** | B-tree | B-tree + 선택적 GIN | BRIN (시계열) + GIN |
| **페이지네이션** | OFFSET | OFFSET (캐싱) | KEYSET 필수 |
| **쿼리 도구** | ORM | Query Builder | Query Builder + Raw SQL |
| **연결 풀** | 앱 레벨 | 앱 레벨 | PgBouncer |
| **파티셔닝** | 불필요 | 선택 | 필수 (>100GB) |

### 아키텍처별 권장 구성

**스타트업 MVP**
```
- 기본 키: BIGINT (단순)
- 인덱스: B-tree (기본)
- 쿼리: Prisma (빠른 개발)
- 페이지네이션: OFFSET (간단)
- 연결: 앱 레벨 풀
- 파티셔닝: 불필요
```

**프로덕션 API (모놀리식)**
```
- 기본 키: BIGINT
- 인덱스: B-tree + GIN (JSONB)
- 쿼리: Kysely (성능 + 타입 안전)
- 페이지네이션: KEYSET
- 연결: 앱 레벨 풀 (인스턴스 < 10)
- 파티셔닝: 100GB 초과 시
```

**마이크로서비스 (분산)**
```
- 기본 키: UUID v7
- 인덱스: B-tree + GIN
- 쿼리: Query Builder + Raw SQL
- 페이지네이션: KEYSET
- 연결: PgBouncer (필수)
- 파티셔닝: 서비스별 판단
```

**서버리스 (Lambda)**
```
- 기본 키: UUID v7
- 인덱스: B-tree (최소화)
- 쿼리: Raw SQL (cold start 최소화)
- 페이지네이션: KEYSET
- 연결: PgBouncer Transaction Mode (필수)
- 파티셔닝: 필요 시
```

---

## 추가 리소스

- [PostgreSQL 공식 문서 - Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [Use The Index, Luke!](https://use-the-index-luke.com/) - 인덱스 최적화 가이드
- [PgBouncer 공식 문서](https://www.pgbouncer.org/)
- [pg_partman GitHub](https://github.com/pgpartman/pg_partman)
- [Kysely (타입 안전 Query Builder)](https://kysely.dev/)

---

**문서 버전**: 1.0
**최종 업데이트**: 2024-01-28
**작성자**: PostgreSQL Backend Study Group
