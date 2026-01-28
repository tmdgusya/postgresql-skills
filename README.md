# PostgreSQL Best Practices Study

Supabase의 Claude Code용 PostgreSQL Skills 분석 및 PostgreSQL 공식 문서 기반 기술적 근거 자료

## 📁 프로젝트 구조

```
postgresql-study/
├── README.md                          # 이 파일
├── RESEARCH_REPORT.md                 # 상세 분석 보고서 (한국어)
└── supabase-postgres-best-practices/  # Supabase Skills 다운로드
    ├── SKILL.md                       # 메인 스킬 파일
    └── references/                    # 개별 레퍼런스 문서들
```

## 📊 Skill Set 카테고리 요약

| 우선순위 | 카테고리 | 영향도 | 핵심 내용 |
|:--------:|----------|:------:|-----------|
| 1 | **Query Performance** | CRITICAL | 인덱스 전략 (B-tree, GIN, BRIN), 복합 인덱스, 커버링 인덱스 |
| 2 | **Connection Management** | CRITICAL | 연결 풀링, 메모리 관리, Prepared Statements |
| 3 | **Security & RLS** | CRITICAL | Row Level Security, 권한 관리 |
| 4 | **Schema Design** | HIGH | PK 전략, 데이터 타입, FK 인덱스, 파티셔닝 |
| 5 | **Concurrency & Locking** | MEDIUM-HIGH | 데드락 방지, Advisory Locks, SKIP LOCKED |
| 6 | **Data Access Patterns** | MEDIUM | N+1 문제, 페이지네이션, 배치 처리 |
| 7 | **Monitoring & Diagnostics** | LOW-MEDIUM | EXPLAIN ANALYZE, pg_stat_statements |
| 8 | **Advanced Features** | LOW | 전문 검색, JSONB 인덱싱 |

## 🔑 핵심 인사이트

### 1. 인덱스 타입별 사용 시나리오

```
┌─────────────────────────────────────────────────────────────┐
│  B-tree (기본)     │  =, <, >, BETWEEN, IN, ORDER BY        │
│  GIN              │  JSONB @>, 배열 &&, 전문검색 @@         │
│  BRIN             │  시계열 데이터 (물리적 순서 = 시간순서)  │
│  Hash             │  순수 등호 검색 (거의 사용 안함)         │
└─────────────────────────────────────────────────────────────┘
```

### 2. 복합 인덱스 컬럼 순서

```sql
-- ✅ 올바른 순서: 등호 컬럼 먼저, 범위 컬럼 나중
CREATE INDEX idx ON orders (status, created_at);
-- WHERE status = 'pending' AND created_at > '2024-01-01'

-- ❌ 잘못된 순서: 범위 컬럼이 먼저면 전체 스캔
CREATE INDEX idx ON orders (created_at, status);
```

### 3. 연결 풀 크기 공식

```
connections = (CPU 코어 수 × 2) + 디스크 스핀들 수

예: 4코어 서버 + SSD 1개 = (4 × 2) + 1 = 9개 연결
```

### 4. RLS 성능 최적화

```sql
-- ❌ 느림: auth.uid()가 매 행마다 호출
CREATE POLICY bad ON documents
    USING (owner_id = auth.uid());

-- ✅ 빠름: 서브쿼리로 initPlan 캐싱
CREATE POLICY good ON documents
    USING (owner_id = (SELECT auth.uid()));
```

### 5. OFFSET vs 커서 페이지네이션

```
OFFSET 방식:  O(n) - 페이지가 깊어질수록 선형적으로 느려짐
커서 방식:    O(1) - 모든 페이지에서 일정한 성능
```

## 📚 상세 문서

- **[RESEARCH_REPORT.md](./RESEARCH_REPORT.md)**: 전체 분석 보고서
- **[supabase-postgres-best-practices/SKILL.md](./supabase-postgres-best-practices/SKILL.md)**: Supabase 원본 스킬

## 🔗 참고 링크

### PostgreSQL 공식 문서
- [Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
- [Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

### Supabase 문서
- [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [RLS Performance Best Practices](https://supabase.com/docs/guides/troubleshooting/rls-performance-and-best-practices-Z5Jjwv)

### 원본 저장소
- [supabase/agent-skills](https://github.com/supabase/agent-skills/tree/main/skills/supabase-postgres-best-practices)
