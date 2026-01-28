# PostgreSQL 기반 애플리케이션 테스팅 전략

백엔드 엔지니어를 위한 PostgreSQL 테스팅 완벽 가이드

---

## 목차

1. [데이터베이스 테스팅의 종류](#1-데이터베이스-테스팅의-종류)
2. [각 테스트 레벨별 검증 항목](#2-각-테스트-레벨별-검증-항목)
3. [테스트 환경 구성 방법](#3-테스트-환경-구성-방법)
4. [성능 테스트 도구 및 방법론](#4-성능-테스트-도구-및-방법론)
5. [CI/CD 파이프라인 통합](#5-cicd-파이프라인-통합)
6. [구체적인 테스트 코드 예시](#6-구체적인-테스트-코드-예시)

---

## 1. 데이터베이스 테스팅의 종류

### 1.1 단위 테스트 (Unit Tests)

**목적**: 개별 SQL 쿼리, 함수, 프로시저의 정확성 검증

**특징**:
- 격리된 환경에서 실행
- 빠른 실행 속도 (밀리초 단위)
- 모킹/스텁 활용 가능
- 트랜잭션 롤백으로 데이터 초기화

**테스트 대상**:
- SQL 함수의 입출력 검증
- 저장 프로시저 로직
- 트리거 동작
- 뷰(View) 결과

```sql
-- 예: PostgreSQL 함수 단위 테스트
CREATE OR REPLACE FUNCTION test_calculate_discount()
RETURNS void AS $$
BEGIN
    ASSERT (SELECT calculate_discount(100, 'PREMIUM')) = 20;
    ASSERT (SELECT calculate_discount(100, 'STANDARD')) = 10;
END;
$$ LANGUAGE plpgsql;
```

### 1.2 통합 테스트 (Integration Tests)

**목적**: 애플리케이션과 데이터베이스 간 상호작용 검증

**특징**:
- 실제 DB 연결 사용
- 애플리케이션 레이어 포함
- 트랜잭션 경계 테스트
- ORM/쿼리 빌더 동작 검증

**테스트 대상**:
- API 엔드포인트 → DB 쿼리 플로우
- 복잡한 조인 쿼리
- 트랜잭션 ACID 속성
- 외래 키 제약 조건
- 데이터 마이그레이션

### 1.3 성능 테스트 (Performance Tests)

**목적**: 쿼리 실행 시간 및 리소스 사용량 측정

**특징**:
- 쿼리 실행 계획 분석
- 인덱스 효율성 검증
- N+1 쿼리 문제 탐지
- 메모리/CPU 사용량 모니터링

**측정 지표**:
- 쿼리 실행 시간 (ms)
- 디스크 I/O
- 버퍼 히트율
- 인덱스 스캔 vs 순차 스캔 비율

### 1.4 부하 테스트 (Load/Stress Tests)

**목적**: 높은 동시성 환경에서의 안정성 검증

**특징**:
- 다수의 동시 연결 시뮬레이션
- 데드락 탐지
- 연결 풀 한계 테스트
- 레플리케이션 지연 측정

**테스트 시나리오**:
- 동시 사용자 100/1000/10000명
- 쓰기 집약적 워크로드
- 읽기 집약적 워크로드
- 혼합 워크로드

---

## 2. 각 테스트 레벨별 검증 항목

### 2.1 인덱스 사용 검증 (EXPLAIN ANALYZE)

**왜 중요한가?**
- 잘못된 인덱스 → 순차 스캔 → 성능 저하
- 프로덕션 데이터량에서는 밀리초 vs 초 단위 차이

**검증 방법**:

```sql
-- 1. 실행 계획 확인
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 123
  AND created_at > NOW() - INTERVAL '30 days';

-- ✅ 좋은 패턴: Index Scan
-- Index Scan using idx_orders_user_created on orders
-- (cost=0.42..8.44 rows=1 width=100) (actual time=0.025..0.026 rows=1 loops=1)

-- ❌ 나쁜 패턴: Seq Scan
-- Seq Scan on orders
-- (cost=0.00..35000.00 rows=1000 width=100) (actual time=150.123..850.456 rows=1000 loops=1)
```

**자동화된 검증**:

```typescript
// TypeScript 예시
async function assertIndexUsed(query: string, expectedIndexName: string) {
    const result = await db.query(`EXPLAIN (FORMAT JSON) ${query}`);
    const plan = result.rows[0]['QUERY PLAN'][0]['Plan'];

    function findIndexScan(node: any): boolean {
        if (node['Node Type'] === 'Index Scan' &&
            node['Index Name'] === expectedIndexName) {
            return true;
        }
        if (node.Plans) {
            return node.Plans.some(findIndexScan);
        }
        return false;
    }

    assert(findIndexScan(plan),
           `Expected index ${expectedIndexName} not used`);
}

// 테스트 케이스
it('should use idx_orders_user_created index', async () => {
    const query = `
        SELECT * FROM orders
        WHERE user_id = 123 AND created_at > '2024-01-01'
    `;
    await assertIndexUsed(query, 'idx_orders_user_created');
});
```

### 2.2 RLS (Row Level Security) 정책 검증

**RLS란?**
- PostgreSQL의 행 수준 보안 정책
- 사용자별로 접근 가능한 데이터 제한
- Supabase, multi-tenant 앱에서 필수

**테스트해야 할 시나리오**:

```sql
-- RLS 정책 예시
CREATE POLICY "Users can only see their own data"
ON user_profiles
FOR SELECT
USING (auth.uid() = user_id);

CREATE POLICY "Users can update their own data"
ON user_profiles
FOR UPDATE
USING (auth.uid() = user_id);
```

**검증 코드**:

```typescript
describe('RLS Policy Tests', () => {
    let userAClient: SupabaseClient;
    let userBClient: SupabaseClient;

    beforeEach(async () => {
        // 각 사용자의 세션으로 별도 클라이언트 생성
        userAClient = createClient(URL, KEY, {
            headers: { Authorization: `Bearer ${userAToken}` }
        });
        userBClient = createClient(URL, KEY, {
            headers: { Authorization: `Bearer ${userBToken}` }
        });
    });

    it('should prevent users from seeing other users data', async () => {
        // User A가 자신의 데이터만 볼 수 있는지 확인
        const { data: userAData } = await userAClient
            .from('user_profiles')
            .select('*');

        expect(userAData.every(row => row.user_id === userA.id)).toBe(true);

        // User B의 데이터가 포함되지 않았는지 확인
        expect(userAData.find(row => row.user_id === userB.id)).toBeUndefined();
    });

    it('should prevent users from updating other users data', async () => {
        // User A가 User B의 데이터를 수정하려 시도
        const { error } = await userAClient
            .from('user_profiles')
            .update({ name: 'Hacked!' })
            .eq('user_id', userB.id);

        expect(error).toBeDefined();
        expect(error.code).toBe('42501'); // insufficient_privilege
    });

    it('should allow users to update their own data', async () => {
        const { data, error } = await userAClient
            .from('user_profiles')
            .update({ name: 'Updated Name' })
            .eq('user_id', userA.id)
            .select();

        expect(error).toBeNull();
        expect(data[0].name).toBe('Updated Name');
    });
});
```

### 2.3 동시성 처리 (데드락, 레이스 컨디션)

**데드락 탐지 테스트**:

```typescript
async function testDeadlock() {
    const client1 = await pool.connect();
    const client2 = await pool.connect();

    try {
        // Transaction 1: orders → inventory 순서로 잠금
        await client1.query('BEGIN');
        await client1.query('UPDATE orders SET status = $1 WHERE id = $2',
                            ['processing', 1]);

        // Transaction 2: inventory → orders 순서로 잠금 (역순)
        await client2.query('BEGIN');
        await client2.query('UPDATE inventory SET quantity = quantity - 1 WHERE id = $1',
                            [100]);

        // 데드락 발생 시도
        const promise1 = client1.query('UPDATE inventory SET quantity = quantity - 1 WHERE id = $1',
                                       [100]);
        const promise2 = client2.query('UPDATE orders SET status = $1 WHERE id = $2',
                                       ['processing', 1]);

        await Promise.all([promise1, promise2]);

        // 여기 도달하면 데드락 발생하지 않음 (좋음)
        await client1.query('COMMIT');
        await client2.query('COMMIT');

    } catch (error) {
        // 40P01 = deadlock_detected
        if (error.code === '40P01') {
            console.log('Deadlock detected as expected');
            await client1.query('ROLLBACK');
            await client2.query('ROLLBACK');
            throw error;
        }
    } finally {
        client1.release();
        client2.release();
    }
}

// 데드락 방지 검증
it('should prevent deadlocks with proper locking order', async () => {
    // 모든 트랜잭션이 동일한 순서로 테이블 잠금
    await expect(testDeadlock()).rejects.toThrow('deadlock_detected');

    // 해결책: 일관된 잠금 순서 사용
    // 항상 orders → inventory 순서로 잠금
});
```

**레이스 컨디션 테스트**:

```typescript
describe('Race Condition Tests', () => {
    it('should handle concurrent inventory updates correctly', async () => {
        // 초기 재고 설정
        await db.query('UPDATE inventory SET quantity = 100 WHERE id = 1');

        // 100명이 동시에 1개씩 구매
        const purchases = Array(100).fill(null).map(() =>
            db.query(`
                UPDATE inventory
                SET quantity = quantity - 1
                WHERE id = 1 AND quantity > 0
            `)
        );

        await Promise.all(purchases);

        // 재고가 정확히 0이어야 함
        const result = await db.query('SELECT quantity FROM inventory WHERE id = 1');
        expect(result.rows[0].quantity).toBe(0);
    });

    it('should prevent overselling with pessimistic locking', async () => {
        await db.query('UPDATE inventory SET quantity = 10 WHERE id = 1');

        // 20명이 동시에 구매 시도
        const purchases = Array(20).fill(null).map(async () => {
            const client = await pool.connect();
            try {
                await client.query('BEGIN');

                // FOR UPDATE로 행 잠금
                const result = await client.query(`
                    SELECT quantity FROM inventory
                    WHERE id = 1
                    FOR UPDATE
                `);

                if (result.rows[0].quantity > 0) {
                    await client.query(`
                        UPDATE inventory
                        SET quantity = quantity - 1
                        WHERE id = 1
                    `);
                    await client.query('COMMIT');
                    return 'success';
                } else {
                    await client.query('ROLLBACK');
                    return 'out_of_stock';
                }
            } finally {
                client.release();
            }
        });

        const results = await Promise.all(purchases);

        // 정확히 10개만 판매되어야 함
        expect(results.filter(r => r === 'success').length).toBe(10);
        expect(results.filter(r => r === 'out_of_stock').length).toBe(10);

        const finalStock = await db.query('SELECT quantity FROM inventory WHERE id = 1');
        expect(finalStock.rows[0].quantity).toBe(0);
    });
});
```

### 2.4 연결 풀링 동작 검증

```typescript
describe('Connection Pool Tests', () => {
    it('should respect max pool size', async () => {
        const pool = new Pool({
            max: 5, // 최대 5개 연결
            connectionTimeoutMillis: 1000
        });

        // 5개 연결 획득
        const clients = await Promise.all([
            pool.connect(),
            pool.connect(),
            pool.connect(),
            pool.connect(),
            pool.connect()
        ]);

        // 6번째 연결은 타임아웃되어야 함
        await expect(pool.connect()).rejects.toThrow('timeout');

        // 연결 해제 후 다시 획득 가능
        clients[0].release();
        const newClient = await pool.connect();
        expect(newClient).toBeDefined();

        clients.forEach(c => c.release());
        newClient.release();
        await pool.end();
    });

    it('should reuse idle connections', async () => {
        const pool = new Pool({ max: 10 });

        // 연결 획득 후 해제
        const client1 = await pool.connect();
        const pid1 = await client1.query('SELECT pg_backend_pid()');
        client1.release();

        // 같은 연결이 재사용되는지 확인
        const client2 = await pool.connect();
        const pid2 = await client2.query('SELECT pg_backend_pid()');
        client2.release();

        expect(pid1.rows[0].pg_backend_pid).toBe(pid2.rows[0].pg_backend_pid);

        await pool.end();
    });

    it('should handle connection errors gracefully', async () => {
        const pool = new Pool({
            host: 'localhost',
            port: 5432,
            database: 'testdb'
        });

        // 쿼리 중 연결 끊김 시뮬레이션
        const client = await pool.connect();

        // 강제로 연결 종료
        await client.query('SELECT pg_terminate_backend(pg_backend_pid())');

        // 새 쿼리는 자동으로 재연결되어야 함
        const result = await pool.query('SELECT 1');
        expect(result.rows[0]['?column?']).toBe(1);

        await pool.end();
    });
});
```

---

## 3. 테스트 환경 구성 방법

### 3.1 Docker를 사용한 로컬 환경

**장점**:
- 빠른 설정
- 일관된 환경
- 격리된 테스트

**docker-compose.yml**:

```yaml
version: '3.8'

services:
  postgres-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
      POSTGRES_INITDB_ARGS: "-c shared_buffers=256MB -c max_connections=200"
    ports:
      - "5433:5432"
    volumes:
      - ./init-scripts:/docker-entrypoint-initdb.d
      - postgres-test-data:/var/lib/postgresql/data
    tmpfs:
      - /var/lib/postgresql/data:rw,noexec,nosuid,size=512m
    command:
      - "postgres"
      - "-c"
      - "fsync=off"  # 테스트 환경에서만! 성능 향상
      - "-c"
      - "synchronous_commit=off"
      - "-c"
      - "full_page_writes=off"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres-test-data:
```

**테스트 스크립트**:

```bash
#!/bin/bash
# test-setup.sh

# Docker 컨테이너 시작
docker-compose up -d postgres-test

# DB 준비될 때까지 대기
echo "Waiting for PostgreSQL..."
until docker-compose exec -T postgres-test pg_isready -U testuser -d testdb; do
    sleep 1
done

echo "Running migrations..."
npm run migrate:test

echo "Running tests..."
npm test

# 정리
docker-compose down -v
```

### 3.2 Testcontainers (프로그래매틱 컨테이너 관리)

**장점**:
- 테스트 코드에서 직접 제어
- 병렬 테스트 격리
- CI/CD 통합 용이

```typescript
import { GenericContainer, StartedTestContainer } from 'testcontainers';
import { Client } from 'pg';

describe('Integration Tests with Testcontainers', () => {
    let container: StartedTestContainer;
    let client: Client;

    beforeAll(async () => {
        // PostgreSQL 컨테이너 시작
        container = await new GenericContainer('postgres:16-alpine')
            .withEnvironment({
                POSTGRES_DB: 'testdb',
                POSTGRES_USER: 'testuser',
                POSTGRES_PASSWORD: 'testpass'
            })
            .withExposedPorts(5432)
            .withTmpFs({ '/var/lib/postgresql/data': 'rw' })
            .start();

        // 클라이언트 연결
        client = new Client({
            host: container.getHost(),
            port: container.getMappedPort(5432),
            database: 'testdb',
            user: 'testuser',
            password: 'testpass'
        });

        await client.connect();

        // 스키마 초기화
        await client.query(`
            CREATE TABLE users (
                id SERIAL PRIMARY KEY,
                email VARCHAR(255) UNIQUE NOT NULL,
                created_at TIMESTAMP DEFAULT NOW()
            );
        `);
    }, 60000); // 컨테이너 시작 타임아웃

    afterAll(async () => {
        await client.end();
        await container.stop();
    });

    beforeEach(async () => {
        // 각 테스트 전 데이터 초기화
        await client.query('TRUNCATE users RESTART IDENTITY CASCADE');
    });

    it('should insert and retrieve user', async () => {
        const email = 'test@example.com';

        await client.query(
            'INSERT INTO users (email) VALUES ($1)',
            [email]
        );

        const result = await client.query(
            'SELECT * FROM users WHERE email = $1',
            [email]
        );

        expect(result.rows[0].email).toBe(email);
    });
});
```

### 3.3 실제 DB 복제본 (Staging/QA 환경)

**전략**:

```typescript
// config/database.ts
export function getDatabaseConfig(env: string) {
    switch (env) {
        case 'test':
            return {
                host: 'localhost',
                port: 5433,
                database: 'testdb',
                // 빠른 테스트를 위한 최적화
                poolSize: 5,
                ssl: false
            };

        case 'qa':
            return {
                host: process.env.QA_DB_HOST,
                port: 5432,
                database: 'qa_db',
                // 프로덕션과 유사한 설정
                poolSize: 20,
                ssl: true,
                replication: {
                    master: process.env.QA_DB_MASTER,
                    slaves: [process.env.QA_DB_SLAVE_1]
                }
            };

        case 'production':
            return {
                host: process.env.PROD_DB_HOST,
                port: 5432,
                database: 'prod_db',
                poolSize: 100,
                ssl: true,
                replication: {
                    master: process.env.PROD_DB_MASTER,
                    slaves: [
                        process.env.PROD_DB_SLAVE_1,
                        process.env.PROD_DB_SLAVE_2
                    ]
                }
            };
    }
}
```

**데이터 익명화 스크립트**:

```sql
-- 프로덕션 데이터 복제 후 개인정보 익명화
UPDATE users
SET
    email = CONCAT('user', id, '@example.com'),
    name = CONCAT('User ', id),
    phone = NULL,
    address = NULL;

UPDATE payment_methods
SET
    card_number = '****-****-****-1234',
    cvv = NULL;
```

---

## 4. 성능 테스트 도구 및 방법론

### 4.1 pgbench (내장 벤치마크 도구)

**기본 사용법**:

```bash
# 1. 테스트 데이터 초기화 (100만 행)
pgbench -i -s 10 testdb

# 2. 기본 TPC-B 벤치마크 (10개 클라이언트, 60초)
pgbench -c 10 -j 2 -T 60 testdb

# 출력 예시:
# transaction type: <builtin: TPC-B (sort of)>
# scaling factor: 10
# number of clients: 10
# number of threads: 2
# duration: 60 s
# number of transactions actually processed: 12345
# latency average = 48.6 ms
# tps = 205.749921 (including connections establishing)
# tps = 205.876543 (excluding connections establishing)
```

**커스텀 쿼리 벤치마크**:

```sql
-- queries.sql
\set user_id random(1, 100000)
SELECT * FROM users WHERE id = :user_id;
SELECT * FROM orders WHERE user_id = :user_id ORDER BY created_at DESC LIMIT 10;
```

```bash
# 커스텀 쿼리로 벤치마크
pgbench -c 50 -j 4 -T 120 -f queries.sql testdb
```

**읽기/쓰기 혼합 테스트**:

```sql
-- mixed-workload.sql
\set user_id random(1, 100000)
\set action random(1, 10)

-- 70% 읽기
BEGIN;
SELECT * FROM users WHERE id = :user_id;
-- 30% 쓰기
\if :action <= 3
    UPDATE users SET last_login = NOW() WHERE id = :user_id;
\endif
COMMIT;
```

### 4.2 k6 (모던 부하 테스트 도구)

**설치 및 기본 사용**:

```bash
# 설치 (macOS)
brew install k6

# 실행
k6 run load-test.js
```

**PostgreSQL 부하 테스트 스크립트**:

```javascript
// load-test.js
import sql from 'k6/x/sql';
import { check } from 'k6';

const db = sql.open('postgres', 'postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable');

export const options = {
    stages: [
        { duration: '1m', target: 50 },   // 1분 동안 50명으로 증가
        { duration: '3m', target: 50 },   // 3분 동안 50명 유지
        { duration: '1m', target: 100 },  // 1분 동안 100명으로 증가
        { duration: '3m', target: 100 },  // 3분 동안 100명 유지
        { duration: '1m', target: 0 },    // 1분 동안 0명으로 감소
    ],
    thresholds: {
        'http_req_duration': ['p(95)<500'],  // 95%가 500ms 이하
        'http_req_failed': ['rate<0.01'],    // 에러율 1% 미만
    },
};

export default function () {
    // 1. 사용자 조회
    const userResult = sql.query(db, `
        SELECT * FROM users
        WHERE id = $1
    `, Math.floor(Math.random() * 100000) + 1);

    check(userResult, {
        'user found': (r) => r.length > 0,
    });

    // 2. 주문 목록 조회
    const ordersResult = sql.query(db, `
        SELECT * FROM orders
        WHERE user_id = $1
        ORDER BY created_at DESC
        LIMIT 10
    `, Math.floor(Math.random() * 100000) + 1);

    check(ordersResult, {
        'orders retrieved': (r) => r !== null,
    });

    // 3. 쓰기 작업 (30% 확률)
    if (Math.random() < 0.3) {
        sql.exec(db, `
            INSERT INTO logs (user_id, action, created_at)
            VALUES ($1, $2, NOW())
        `,
            Math.floor(Math.random() * 100000) + 1,
            'page_view'
        );
    }
}

export function teardown() {
    sql.close(db);
}
```

**실행 및 결과 분석**:

```bash
k6 run --out json=results.json load-test.js

# 결과 요약
# ✓ user found
# ✓ orders retrieved
#
# checks.........................: 100.00% ✓ 15234      ✗ 0
# data_received..................: 4.2 MB  70 kB/s
# data_sent......................: 2.1 MB  35 kB/s
# iteration_duration.............: avg=245ms min=12ms med=198ms max=1.2s p(90)=456ms p(95)=578ms
# iterations.....................: 7617    127/s
# vus............................: 100     min=0       max=100
# vus_max........................: 100     min=100     max=100
```

### 4.3 커스텀 Node.js 성능 테스트

```typescript
// performance-test.ts
import { Pool } from 'pg';
import { performance } from 'perf_hooks';

interface BenchmarkResult {
    queryName: string;
    avgTime: number;
    minTime: number;
    maxTime: number;
    p95Time: number;
    p99Time: number;
    totalQueries: number;
}

async function benchmarkQuery(
    pool: Pool,
    queryName: string,
    query: string,
    params: any[],
    iterations: number = 1000
): Promise<BenchmarkResult> {
    const times: number[] = [];

    for (let i = 0; i < iterations; i++) {
        const start = performance.now();
        await pool.query(query, params);
        const end = performance.now();
        times.push(end - start);
    }

    times.sort((a, b) => a - b);

    return {
        queryName,
        avgTime: times.reduce((a, b) => a + b) / times.length,
        minTime: times[0],
        maxTime: times[times.length - 1],
        p95Time: times[Math.floor(times.length * 0.95)],
        p99Time: times[Math.floor(times.length * 0.99)],
        totalQueries: iterations
    };
}

async function runPerformanceTests() {
    const pool = new Pool({
        host: 'localhost',
        port: 5432,
        database: 'testdb',
        user: 'testuser',
        password: 'testpass',
        max: 20
    });

    console.log('Starting performance tests...\n');

    // 1. 인덱스 있는 쿼리
    const indexedResult = await benchmarkQuery(
        pool,
        'Indexed query (user_id)',
        'SELECT * FROM orders WHERE user_id = $1',
        [12345],
        1000
    );

    // 2. 인덱스 없는 쿼리
    const nonIndexedResult = await benchmarkQuery(
        pool,
        'Non-indexed query (status)',
        'SELECT * FROM orders WHERE status = $1',
        ['completed'],
        1000
    );

    // 3. 복잡한 조인 쿼리
    const joinResult = await benchmarkQuery(
        pool,
        'Complex join',
        `SELECT o.*, u.email, p.total
         FROM orders o
         JOIN users u ON o.user_id = u.id
         JOIN payments p ON o.id = p.order_id
         WHERE o.created_at > $1`,
        [new Date('2024-01-01')],
        500
    );

    // 결과 출력
    const results = [indexedResult, nonIndexedResult, joinResult];

    console.table(results.map(r => ({
        'Query': r.queryName,
        'Avg (ms)': r.avgTime.toFixed(2),
        'Min (ms)': r.minTime.toFixed(2),
        'Max (ms)': r.maxTime.toFixed(2),
        'P95 (ms)': r.p95Time.toFixed(2),
        'P99 (ms)': r.p99Time.toFixed(2),
    })));

    // 성능 기준 검증
    if (indexedResult.p95Time > 10) {
        console.error('❌ Indexed query P95 exceeds 10ms threshold');
    } else {
        console.log('✅ Indexed query performance acceptable');
    }

    await pool.end();
}

runPerformanceTests();
```

**실행 결과 예시**:

```
Starting performance tests...

┌─────────┬──────────────────────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ (index) │          Query           │ Avg (ms) │ Min (ms) │ Max (ms) │ P95 (ms) │ P99 (ms) │
├─────────┼──────────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│    0    │ 'Indexed query (user_id)'│  '2.34'  │  '1.12'  │  '8.45'  │  '3.21'  │  '4.56'  │
│    1    │'Non-indexed query (st...'│ '156.78' │ '98.23'  │ '234.56' │ '189.34' │ '210.12' │
│    2    │    'Complex join'        │ '12.45'  │  '8.90'  │ '25.67'  │ '18.23'  │ '21.34'  │
└─────────┴──────────────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘

✅ Indexed query performance acceptable
```

---

## 5. CI/CD 파이프라인 통합

### 5.1 GitHub Actions 예시

```yaml
# .github/workflows/test.yml
name: PostgreSQL Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run migrations
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
        run: npm run migrate

      - name: Run unit tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
        run: npm run test:unit

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
        run: npm run test:integration

      - name: Run performance tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
        run: npm run test:performance

      - name: Check test coverage
        run: npm run test:coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  load-test:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: loadtest
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v3

      - name: Setup k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6

      - name: Seed test data
        run: |
          PGPASSWORD=testpass psql -h localhost -U testuser -d loadtest < ./test-data/seed.sql

      - name: Run load tests
        run: k6 run ./tests/load/load-test.js

      - name: Upload load test results
        uses: actions/upload-artifact@v3
        with:
          name: load-test-results
          path: ./test-results/
```

### 5.2 GitLab CI 예시

```yaml
# .gitlab-ci.yml
stages:
  - test
  - performance
  - deploy

variables:
  POSTGRES_DB: testdb
  POSTGRES_USER: testuser
  POSTGRES_PASSWORD: testpass
  DATABASE_URL: postgresql://testuser:testpass@postgres:5432/testdb

test:unit:
  stage: test
  image: node:20-alpine
  services:
    - postgres:16-alpine
  script:
    - npm ci
    - npm run migrate
    - npm run test:unit
  coverage: '/Statements\s+:\s+(\d+\.\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

test:integration:
  stage: test
  image: node:20-alpine
  services:
    - postgres:16-alpine
  script:
    - npm ci
    - npm run migrate
    - npm run test:integration
  only:
    - merge_requests
    - main

test:performance:
  stage: performance
  image: node:20-alpine
  services:
    - postgres:16-alpine
  script:
    - npm ci
    - npm run migrate
    - npm run test:performance
    - npm run analyze-performance
  artifacts:
    paths:
      - performance-results/
    expire_in: 30 days
  only:
    - merge_requests

test:load:
  stage: performance
  image: grafana/k6:latest
  services:
    - postgres:16-alpine
  script:
    - k6 run --out json=results.json tests/load/load-test.js
  artifacts:
    paths:
      - results.json
  only:
    - schedules
```

### 5.3 테스트 결과 리포팅

```typescript
// test-reporter.ts
import fs from 'fs';
import path from 'path';

interface TestResult {
    suite: string;
    passed: number;
    failed: number;
    duration: number;
    coverage?: number;
}

interface PerformanceMetric {
    query: string;
    p95: number;
    p99: number;
    threshold: number;
    passed: boolean;
}

function generateTestReport(
    testResults: TestResult[],
    performanceMetrics: PerformanceMetric[]
) {
    const totalPassed = testResults.reduce((sum, r) => sum + r.passed, 0);
    const totalFailed = testResults.reduce((sum, r) => sum + r.failed, 0);
    const totalDuration = testResults.reduce((sum, r) => sum + r.duration, 0);

    const report = `
# PostgreSQL Test Report

## Summary
- **Total Tests**: ${totalPassed + totalFailed}
- **Passed**: ${totalPassed} ✅
- **Failed**: ${totalFailed} ${totalFailed > 0 ? '❌' : ''}
- **Duration**: ${totalDuration.toFixed(2)}s

## Test Suites

${testResults.map(r => `
### ${r.suite}
- Passed: ${r.passed}
- Failed: ${r.failed}
- Duration: ${r.duration.toFixed(2)}s
${r.coverage ? `- Coverage: ${r.coverage}%` : ''}
`).join('\n')}

## Performance Metrics

${performanceMetrics.map(m => `
### ${m.query}
- **P95**: ${m.p95.toFixed(2)}ms
- **P99**: ${m.p99.toFixed(2)}ms
- **Threshold**: ${m.threshold}ms
- **Status**: ${m.passed ? '✅ PASS' : '❌ FAIL'}
`).join('\n')}

## Recommendations

${performanceMetrics
    .filter(m => !m.passed)
    .map(m => `- ⚠️ ${m.query} exceeds threshold (${m.p95.toFixed(2)}ms > ${m.threshold}ms)`)
    .join('\n')}

---
Generated on: ${new Date().toISOString()}
`;

    fs.writeFileSync(
        path.join(process.cwd(), 'test-report.md'),
        report
    );

    console.log('Test report generated: test-report.md');
}

// 사용 예시
const testResults: TestResult[] = [
    { suite: 'Unit Tests', passed: 145, failed: 0, duration: 12.3, coverage: 87.5 },
    { suite: 'Integration Tests', passed: 67, failed: 2, duration: 45.6 },
    { suite: 'RLS Tests', passed: 23, failed: 0, duration: 8.9 }
];

const performanceMetrics: PerformanceMetric[] = [
    { query: 'SELECT users by ID', p95: 2.3, p99: 4.5, threshold: 10, passed: true },
    { query: 'Complex join query', p95: 15.6, p99: 23.4, threshold: 20, passed: true },
    { query: 'Non-indexed search', p95: 156.7, p99: 234.5, threshold: 50, passed: false }
];

generateTestReport(testResults, performanceMetrics);
```

---

## 6. 구체적인 테스트 코드 예시

### 6.1 TypeScript/Node.js + Jest

**환경 설정**:

```typescript
// jest.config.js
module.exports = {
    preset: 'ts-jest',
    testEnvironment: 'node',
    setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],
    testMatch: ['**/*.test.ts'],
    collectCoverageFrom: [
        'src/**/*.ts',
        '!src/**/*.d.ts'
    ],
    coverageThresholds: {
        global: {
            branches: 80,
            functions: 80,
            lines: 80,
            statements: 80
        }
    }
};
```

```typescript
// tests/setup.ts
import { Pool } from 'pg';

let pool: Pool;

beforeAll(async () => {
    pool = new Pool({
        connectionString: process.env.DATABASE_URL
    });

    // 테스트 스키마 생성
    await pool.query(`
        CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            email VARCHAR(255) UNIQUE NOT NULL,
            name VARCHAR(100),
            created_at TIMESTAMP DEFAULT NOW()
        );

        CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
    `);
});

afterAll(async () => {
    await pool.end();
});

beforeEach(async () => {
    // 각 테스트 전 데이터 초기화
    await pool.query('TRUNCATE users RESTART IDENTITY CASCADE');
});

export { pool };
```

**CRUD 테스트**:

```typescript
// tests/user.test.ts
import { pool } from './setup';

describe('User CRUD Operations', () => {
    describe('CREATE', () => {
        it('should insert a new user', async () => {
            const result = await pool.query(
                'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *',
                ['test@example.com', 'Test User']
            );

            expect(result.rows[0]).toMatchObject({
                email: 'test@example.com',
                name: 'Test User'
            });
            expect(result.rows[0].id).toBeDefined();
            expect(result.rows[0].created_at).toBeInstanceOf(Date);
        });

        it('should fail on duplicate email', async () => {
            await pool.query(
                'INSERT INTO users (email, name) VALUES ($1, $2)',
                ['test@example.com', 'User 1']
            );

            await expect(
                pool.query(
                    'INSERT INTO users (email, name) VALUES ($1, $2)',
                    ['test@example.com', 'User 2']
                )
            ).rejects.toThrow(/duplicate key value/);
        });
    });

    describe('READ', () => {
        beforeEach(async () => {
            await pool.query(
                'INSERT INTO users (email, name) VALUES ($1, $2), ($3, $4)',
                ['user1@example.com', 'User 1', 'user2@example.com', 'User 2']
            );
        });

        it('should retrieve user by email', async () => {
            const result = await pool.query(
                'SELECT * FROM users WHERE email = $1',
                ['user1@example.com']
            );

            expect(result.rows).toHaveLength(1);
            expect(result.rows[0].name).toBe('User 1');
        });

        it('should use index for email lookup', async () => {
            const plan = await pool.query(
                'EXPLAIN (FORMAT JSON) SELECT * FROM users WHERE email = $1',
                ['user1@example.com']
            );

            const planText = JSON.stringify(plan.rows[0]);
            expect(planText).toContain('Index Scan');
            expect(planText).toContain('idx_users_email');
        });
    });

    describe('UPDATE', () => {
        let userId: number;

        beforeEach(async () => {
            const result = await pool.query(
                'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING id',
                ['test@example.com', 'Old Name']
            );
            userId = result.rows[0].id;
        });

        it('should update user name', async () => {
            await pool.query(
                'UPDATE users SET name = $1 WHERE id = $2',
                ['New Name', userId]
            );

            const result = await pool.query(
                'SELECT name FROM users WHERE id = $1',
                [userId]
            );

            expect(result.rows[0].name).toBe('New Name');
        });
    });

    describe('DELETE', () => {
        it('should delete user', async () => {
            const insertResult = await pool.query(
                'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING id',
                ['delete@example.com', 'To Delete']
            );
            const userId = insertResult.rows[0].id;

            await pool.query('DELETE FROM users WHERE id = $1', [userId]);

            const selectResult = await pool.query(
                'SELECT * FROM users WHERE id = $1',
                [userId]
            );

            expect(selectResult.rows).toHaveLength(0);
        });
    });
});
```

### 6.2 Python + pytest

**환경 설정**:

```python
# conftest.py
import pytest
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_AUTOCOMMIT

@pytest.fixture(scope="session")
def db_connection():
    """세션 레벨 DB 연결"""
    conn = psycopg2.connect(
        dbname="testdb",
        user="testuser",
        password="testpass",
        host="localhost",
        port=5432
    )

    # 테스트 스키마 생성
    with conn.cursor() as cur:
        cur.execute("""
            CREATE TABLE IF NOT EXISTS products (
                id SERIAL PRIMARY KEY,
                name VARCHAR(255) NOT NULL,
                price DECIMAL(10, 2) NOT NULL,
                stock INT DEFAULT 0,
                created_at TIMESTAMP DEFAULT NOW()
            );

            CREATE INDEX IF NOT EXISTS idx_products_name
            ON products(name);
        """)
        conn.commit()

    yield conn

    conn.close()

@pytest.fixture(autouse=True)
def clean_db(db_connection):
    """각 테스트 후 데이터 초기화"""
    yield

    with db_connection.cursor() as cur:
        cur.execute("TRUNCATE products RESTART IDENTITY CASCADE")
        db_connection.commit()
```

**테스트 코드**:

```python
# test_products.py
import pytest
import psycopg2

def test_insert_product(db_connection):
    """제품 삽입 테스트"""
    with db_connection.cursor() as cur:
        cur.execute(
            "INSERT INTO products (name, price, stock) VALUES (%s, %s, %s) RETURNING id",
            ("Laptop", 1299.99, 10)
        )
        product_id = cur.fetchone()[0]
        db_connection.commit()

        assert product_id is not None

        # 삽입된 데이터 검증
        cur.execute("SELECT name, price, stock FROM products WHERE id = %s", (product_id,))
        result = cur.fetchone()

        assert result[0] == "Laptop"
        assert float(result[1]) == 1299.99
        assert result[2] == 10

def test_concurrent_stock_update(db_connection):
    """동시성 재고 업데이트 테스트"""
    # 초기 제품 생성
    with db_connection.cursor() as cur:
        cur.execute(
            "INSERT INTO products (name, price, stock) VALUES (%s, %s, %s) RETURNING id",
            ("Phone", 999.99, 100)
        )
        product_id = cur.fetchone()[0]
        db_connection.commit()

    # 두 개의 독립적인 연결 생성
    conn1 = psycopg2.connect(
        dbname="testdb", user="testuser",
        password="testpass", host="localhost"
    )
    conn2 = psycopg2.connect(
        dbname="testdb", user="testuser",
        password="testpass", host="localhost"
    )

    try:
        # Transaction 1: FOR UPDATE로 행 잠금
        cur1 = conn1.cursor()
        cur1.execute("BEGIN")
        cur1.execute(
            "SELECT stock FROM products WHERE id = %s FOR UPDATE",
            (product_id,)
        )
        stock1 = cur1.fetchone()[0]

        # Transaction 2: 동일한 행에 접근 시도 (블로킹됨)
        cur2 = conn2.cursor()
        cur2.execute("BEGIN")

        # 이 쿼리는 Transaction 1이 커밋될 때까지 대기
        cur2.execute(
            "UPDATE products SET stock = stock - 5 WHERE id = %s",
            (product_id,)
        )

        # Transaction 1 완료
        cur1.execute(
            "UPDATE products SET stock = %s WHERE id = %s",
            (stock1 - 10, product_id)
        )
        conn1.commit()

        # Transaction 2 완료
        conn2.commit()

        # 최종 재고 확인
        with db_connection.cursor() as cur:
            cur.execute("SELECT stock FROM products WHERE id = %s", (product_id,))
            final_stock = cur.fetchone()[0]

            # 100 - 10 - 5 = 85
            assert final_stock == 85

    finally:
        conn1.close()
        conn2.close()

def test_index_usage(db_connection):
    """인덱스 사용 검증"""
    # 테스트 데이터 삽입
    with db_connection.cursor() as cur:
        for i in range(100):
            cur.execute(
                "INSERT INTO products (name, price, stock) VALUES (%s, %s, %s)",
                (f"Product {i}", 100 + i, i)
            )
        db_connection.commit()

        # EXPLAIN ANALYZE 실행
        cur.execute("""
            EXPLAIN (FORMAT JSON)
            SELECT * FROM products WHERE name = 'Product 50'
        """)
        plan = cur.fetchone()[0]

        # 인덱스 스캔이 사용되었는지 확인
        assert 'Index Scan' in str(plan)
        assert 'idx_products_name' in str(plan)

def test_transaction_rollback(db_connection):
    """트랜잭션 롤백 테스트"""
    with db_connection.cursor() as cur:
        cur.execute("BEGIN")

        cur.execute(
            "INSERT INTO products (name, price, stock) VALUES (%s, %s, %s)",
            ("Temp Product", 50, 5)
        )

        # 명시적 롤백
        cur.execute("ROLLBACK")

        # 데이터가 롤백되었는지 확인
        cur.execute("SELECT COUNT(*) FROM products WHERE name = 'Temp Product'")
        count = cur.fetchone()[0]

        assert count == 0
```

### 6.3 E2E 테스트 (Supabase 클라이언트)

```typescript
// tests/e2e/auth-flow.test.ts
import { createClient, SupabaseClient } from '@supabase/supabase-js';
import { faker } from '@faker-js/faker';

describe('E2E: Authentication Flow with RLS', () => {
    let supabase: SupabaseClient;
    let testUser: { email: string; password: string };

    beforeAll(() => {
        supabase = createClient(
            process.env.SUPABASE_URL!,
            process.env.SUPABASE_ANON_KEY!
        );
    });

    beforeEach(() => {
        testUser = {
            email: faker.internet.email(),
            password: faker.internet.password({ length: 12 })
        };
    });

    it('should complete full user lifecycle', async () => {
        // 1. 회원가입
        const { data: signUpData, error: signUpError } = await supabase.auth.signUp({
            email: testUser.email,
            password: testUser.password
        });

        expect(signUpError).toBeNull();
        expect(signUpData.user).toBeDefined();

        const userId = signUpData.user!.id;

        // 2. 프로필 생성 (RLS로 자동으로 자신의 프로필만 생성 가능)
        const { data: profileData, error: profileError } = await supabase
            .from('profiles')
            .insert({
                user_id: userId,
                username: faker.internet.userName(),
                bio: faker.lorem.sentence()
            })
            .select()
            .single();

        expect(profileError).toBeNull();
        expect(profileData.user_id).toBe(userId);

        // 3. 로그아웃
        await supabase.auth.signOut();

        // 4. 로그인
        const { data: signInData, error: signInError } = await supabase.auth.signInWithPassword({
            email: testUser.email,
            password: testUser.password
        });

        expect(signInError).toBeNull();
        expect(signInData.user!.id).toBe(userId);

        // 5. 프로필 조회 (RLS로 자신의 프로필만 조회 가능)
        const { data: fetchedProfile, error: fetchError } = await supabase
            .from('profiles')
            .select('*')
            .eq('user_id', userId)
            .single();

        expect(fetchError).toBeNull();
        expect(fetchedProfile.user_id).toBe(userId);

        // 6. 정리
        await supabase.auth.signOut();
    });

    it('should enforce RLS policies', async () => {
        // User A 생성 및 로그인
        const userA = await createTestUser(supabase);
        const clientA = createClient(
            process.env.SUPABASE_URL!,
            process.env.SUPABASE_ANON_KEY!,
            {
                global: {
                    headers: {
                        Authorization: `Bearer ${userA.session.access_token}`
                    }
                }
            }
        );

        // User B 생성
        const userB = await createTestUser(supabase);

        // User A가 User B의 프로필을 조회하려 시도
        const { data, error } = await clientA
            .from('profiles')
            .select('*')
            .eq('user_id', userB.user.id);

        // RLS 정책에 의해 접근 불가
        expect(data).toHaveLength(0);

        // 정리
        await supabase.auth.signOut();
    });
});

async function createTestUser(supabase: SupabaseClient) {
    const { data, error } = await supabase.auth.signUp({
        email: faker.internet.email(),
        password: faker.internet.password({ length: 12 })
    });

    if (error) throw error;
    return data;
}
```

---

## 마치며

PostgreSQL 기반 애플리케이션의 테스팅은 단순한 기능 검증을 넘어 **성능, 보안, 동시성**까지 포괄해야 합니다.

### 핵심 체크리스트

- [ ] 모든 주요 쿼리에 대해 `EXPLAIN ANALYZE` 검증
- [ ] RLS 정책이 있다면 각 역할별 접근 테스트
- [ ] 동시성 시나리오 (재고 차감, 좌석 예약 등) 테스트
- [ ] 연결 풀 한계 테스트
- [ ] 성능 회귀 방지를 위한 벤치마크 자동화
- [ ] CI/CD 파이프라인에서 모든 테스트 실행
- [ ] 프로덕션과 유사한 데이터 볼륨으로 QA 환경 테스트

### 권장 테스트 피라미드

```
        /\
       /  \  E2E (5%)
      /    \
     /------\ Integration (20%)
    /        \
   /----------\ Unit (75%)
  /--------------\
```

**테스트 실행 빈도**:
- 단위 테스트: 매 커밋
- 통합 테스트: 매 PR
- 성능 테스트: 매일 밤 scheduled job
- 부하 테스트: 매 릴리스 전

### 참고 자료

- [PostgreSQL Official Docs - Testing](https://www.postgresql.org/docs/current/regress.html)
- [pgbench Documentation](https://www.postgresql.org/docs/current/pgbench.html)
- [k6 Documentation](https://k6.io/docs/)
- [Testcontainers](https://www.testcontainers.org/)
- [Supabase Testing Guide](https://supabase.com/docs/guides/getting-started/testing)
