---
name: supabase-health
description: Use when auditing a React + Supabase project for production performance issues, database crashes, or connection pool saturation. Triggers on symptoms like repeated DB crashes, high query counts, slow RLS evaluation, or post-restart planner issues.
---

# /supabase-health — Supabase Production Audit

Systematic 6-layer audit for React 18 + TanStack Query + Supabase projects. Every check maps to a real production incident. Run layers in order — earlier layers are the most critical.

---

## Layer 1 — Thundering Herd: useEffect + useAuth

**Why this crashes the DB:** `useAuth` triggers 2–4 sequential renders on load (loading → user → roles). Any `useEffect` with `user` (whole object) or `session` in its deps re-runs 2–4× per user per navigation. With 10 concurrent users: 10× 4× = 40–80× the expected query count. Connection pool saturates in seconds.

**Real incident:** `checkPremiumAccess` in `useCourseLogic.ts` fired 16 queries/user (expected: 3–4). DB crashed 4× in one afternoon with fewer than 20 simultaneous users.

### Check 1.1 — useEffect querying the DB

```bash
# Find useEffects that query Supabase (look for files with both)
grep -rn "useEffect" src/ --include="*.tsx" --include="*.ts" -l
```

For each file, look for:

```typescript
// DANGEROUS — whole user object in deps:
useEffect(() => {
  supabase.from("table").select(...)
}, [..., user, ...]);  // ← re-runs on every render

// DANGEROUS — session in deps:
useEffect(() => {
  supabase.from("table").select(...)
}, [..., session, ...]);  // ← re-runs on every token refresh
```

**Required fix:**

```typescript
// Replace user → user?.id in deps + add concurrency guard:
const checkInFlight = useRef(false);
const checkedForKey = useRef<string | null>(null);

useEffect(() => {
  const key = `${param1}:${param2}:${user?.id}:${relevantFlag}`;
  if (checkedForKey.current === key) return;
  if (checkInFlight.current) return;

  const run = async () => {
    checkInFlight.current = true;
    try {
      // DB queries here
      checkedForKey.current = key;
    } finally {
      checkInFlight.current = false;
    }
  };
  run();
}, [param1, param2, user?.id, relevantFlag]);  // ← user?.id, not user
```

### Check 1.2 — onAuthStateChange querying the DB

```bash
grep -rn "onAuthStateChange" src/ --include="*.tsx" --include="*.ts"
```

Verify:
- `TOKEN_REFRESHED` with valid session does **not** query the DB (only updates memory state)
- Role/profile queries only fire on `SIGNED_IN` and `INITIAL_SESSION`
- All listener functions that query the DB have `useRef` concurrency guards

```typescript
// CORRECT:
if (event === "TOKEN_REFRESHED" && session) {
  setSession(session);
  setUser(session.user);
  return;  // ← NO DB queries here
}
```

---

## Layer 2 — React Query Without staleTime

**Why this crashes the DB:** Without `staleTime`, every page navigation re-runs all queries — even if the user navigated back 5 seconds ago. In SPAs with frequent back/forward (portals, courses), each transition fires 3–10 unnecessary queries.

**Real incident:** `PortalPage` without `staleTime` caused portals/portal_courses query ratio of 167× — meaning portals was queried 167× more than expected per navigation. Fixed with `staleTime: 5 * 60 * 1000`.

### Check 2.1 — useQuery without staleTime

```bash
grep -rn "useQuery" src/ --include="*.tsx" --include="*.ts" | grep -v "staleTime" | grep -v "//.*useQuery" | grep -v ".test."
```

Every result is a candidate for `staleTime`. Legitimate exceptions:
- Real-time data that MUST always be fresh (unread notifications, cart)
- Queries with `enabled: false` (lazy — only run on user action)

**Required pattern:**
```typescript
const { data } = useQuery({
  queryKey: ['resource', id, user?.id],
  queryFn: () => fetchData(id!),
  enabled: !!id && !!user,
  staleTime: 5 * 60 * 1000,  // 5 minutes — safe default for content data
});
```

### Check 2.2 — refetchInterval with heavy queries

```bash
grep -rn "refetchInterval" src/ --include="*.tsx" --include="*.ts"
```

**Rule:** `refetchInterval` is forbidden on queries that JOIN 2+ tables beyond the main table. Use Supabase Realtime for live data.

```typescript
// FORBIDDEN — polling with heavy query:
useQuery({
  queryFn: () => fetchWithMultipleJoins(),
  refetchInterval: 30000,  // 30s × N users = DB saturated
});

// CORRECT — Realtime for badge counter, lazy query for payload:
// 1. useEffect with supabase.channel() to increment counter
// 2. useQuery with enabled: false + manual refetch() on click
```

---

## Layer 3 — RLS: Policies That Kill Performance

**Why this crashes the DB:** Poorly written RLS policies are evaluated per row, not once per query. With large tables and concurrent users, this multiplies CPU and RAM cost until saturation.

### Check 3.1 — auth.uid() directly in policies (RescanCond)

```sql
-- Find policies with direct auth.uid() (evaluated per row):
SELECT schemaname, tablename, policyname, qual
FROM pg_policies
WHERE qual LIKE '%auth.uid()%'
  AND qual NOT LIKE '%(SELECT auth.uid())%';
```

**Every result must be fixed:**

```sql
-- WRONG — auth.uid() evaluated per row:
CREATE POLICY "x" ON table USING (user_id = auth.uid());

-- CORRECT — evaluated once per query (InitPlan):
CREATE POLICY "x" ON table USING (user_id = (SELECT auth.uid()));
```

Same rule applies to `auth.jwt()` and `auth.role()`.

### Check 3.2 — Multiple PERMISSIVE policies for same command

```sql
-- Find tables with multiple SELECT policies:
SELECT tablename, COUNT(*) as policy_count, array_agg(policyname) as policies
FROM pg_policies
WHERE cmd IN ('SELECT', 'ALL')
GROUP BY tablename
HAVING COUNT(*) > 1
ORDER BY policy_count DESC;
```

**Any result with count > 1 is a problem:** PostgreSQL evaluates ALL PERMISSIVE policies in OR without short-circuit.

```sql
-- WRONG — two SELECT policies on the same table:
CREATE POLICY "admins" ON courses FOR ALL USING (has_role(..., 'admin'));
CREATE POLICY "published" ON courses FOR SELECT USING (published = true);

-- CORRECT — one unified SELECT policy:
CREATE POLICY "courses_select" ON courses FOR SELECT
  USING (has_role((SELECT auth.uid()), 'admin') OR published = true);
-- Separate write policies per command:
CREATE POLICY "courses_insert" ON courses FOR INSERT WITH CHECK (...);
```

### Check 3.3 — Tables without indexes on FKs

```sql
-- Find FKs without a covering index:
SELECT
  tc.table_name,
  kcu.column_name,
  ccu.table_name AS foreign_table
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
  ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
  ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND NOT EXISTS (
    SELECT 1 FROM pg_indexes
    WHERE tablename = tc.table_name
      AND indexdef LIKE '%' || kcu.column_name || '%'
  )
ORDER BY tc.table_name;
```

**Every result is a potential seq scan on JOINs and CASCADEs.** Fix:

```sql
CREATE INDEX IF NOT EXISTS idx_table_fk_col ON public.table(fk_col);
```

---

## Layer 4 — Database Configuration

Run in Supabase Dashboard SQL Editor (requires superuser — does not work via MCP):

```sql
SELECT name, setting, unit, source
FROM pg_settings
WHERE name IN (
  'work_mem', 'max_connections', 'shared_buffers',
  'idle_in_transaction_session_timeout',
  'statement_timeout', 'max_parallel_workers_per_gather'
);
```

### Check 4.1 — work_mem × max_connections

**Safety formula:** `work_mem × max_connections × 3 sort nodes ≤ available RAM × 0.75`

| Supabase Tier | RAM | max_connections | Safe max work_mem |
|---|---|---|---|
| Nano/Micro | 1 GB | 60 | ~4 MB |
| Small | 2 GB | 90 | ~5 MB |
| Medium | 4 GB | 160 | ~6 MB |
| Large | 8 GB | 300 | ~6 MB |

If `work_mem` exceeds the table values, fix in SQL Editor:
```sql
ALTER SYSTEM SET work_mem = '4MB';
SELECT pg_reload_conf();
```

### Check 4.2 — idle_in_transaction_session_timeout

Must be ≤ 30s. If at 0 (Supabase default):
```sql
ALTER SYSTEM SET idle_in_transaction_session_timeout = '30s';
SELECT pg_reload_conf();
```

Value 0 means connections stuck in open transactions are never terminated — they accumulate indefinitely and cause periodic OOM.

### Check 4.3 — Active zombie connections

```sql
SELECT state, count(*) as qty,
  MAX(EXTRACT(EPOCH FROM (now() - state_change))::int) as max_secs,
  SUM(CASE WHEN state = 'idle in transaction' THEN 1 ELSE 0 END) as idle_in_tx
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state ORDER BY qty DESC;
```

If `idle_in_tx > 0`, investigate:
```sql
SELECT pid, usename, application_name,
  EXTRACT(EPOCH FROM (now() - state_change))::int as secs_idle_in_tx,
  left(query, 100) as last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY secs_idle_in_tx DESC;
```

---

## Layer 5 — Planner Stats (post-restart)

**Critical context:** Every time the DB restarts (crash, PITR backup/restore, or maintenance), `pg_stat_user_tables.n_live_tup` resets to zero. With zeroed stats, the planner doesn't know table sizes and may choose seq scan over index scan — even with correct indexes.

### Check 5.1 — Verify n_live_tup

```sql
SELECT relname, n_live_tup, n_dead_tup, seq_scan, idx_scan
FROM pg_stat_user_tables
WHERE relname IN (
  -- list your project's main tables:
  'portals','courses','modules','lessons','users','profiles',
  'lesson_progress','portal_members','portal_subscriptions'
)
ORDER BY seq_scan DESC;
```

If `n_live_tup = 0` on tables that should have data, autovacuum ANALYZE hasn't run yet. **Force it manually in SQL Editor:**

```sql
VACUUM ANALYZE public.lessons;
VACUUM ANALYZE public.modules;
VACUUM ANALYZE public.courses;
VACUUM ANALYZE public.portals;
-- repeat for all main tables
```

> **Note:** `VACUUM ANALYZE` cannot run in a transaction — it does not work via MCP `execute_sql`. Use the Dashboard SQL Editor.

### Check 5.2 — Accumulated seq scans

```sql
SELECT relname, seq_scan, seq_tup_read,
  idx_scan, round(seq_tup_read::numeric / NULLIF(seq_scan, 0)) as avg_rows_per_seqscan
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC;
```

If `avg_rows_per_seqscan` is high (e.g. 3,000+ on a table that has an index), the planner is choosing seq scan — likely due to zeroed stats. Run `VACUUM ANALYZE` on that table.

---

## Layer 6 — Bonus: Heaviest Production Queries

```sql
SELECT round(total_exec_time::numeric, 1) as total_ms,
  round(mean_exec_time::numeric, 1) as mean_ms,
  calls,
  left(query, 150) as query
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_timezone%'
  AND query NOT LIKE '%base_types%'
  AND query NOT LIKE '%schema_migrations%'
ORDER BY total_exec_time DESC
LIMIT 20;
```

**Warning signs:**
- Same query with dozens of calls in minutes → thundering herd (Layer 1)
- `mean_ms > 100` → heavy query, check plan with `EXPLAIN ANALYZE`
- Any RPC with disproportionate calls → N+1 not fixed

---

## Production Incident Reference

| Date | Problem | Root Cause | Fix |
|------|---------|-----------|-----|
| 2026-05-11 | DB crash in 30s | `NotificationBell` with `refetchInterval: 30000` + 3-level JOIN query | Realtime + lazy query |
| 2026-05-14 | Periodic OOM | `work_mem = 12MB` × 160 connections = 5.7 GB potential | `work_mem = 4MB` |
| 2026-05-14 | CPU/RAM saturation | 691 policies with direct `auth.uid()` (RescanCond) | `(SELECT auth.uid())` everywhere |
| 2026-05-14 | Query degradation | 534 warnings from multiple PERMISSIVE policies across 27 tables | Unified SELECT policies with OR |
| 2026-05-14 | Seq scans on JOINs | 26 FKs without B-tree index | `CREATE INDEX` on FKs |
| 2026-05-17 | DB crash in 2–3 min | `checkAdminRole` called on `TOKEN_REFRESHED` without guard → exponential cascade | `useRef` guard + skip TOKEN_REFRESHED |
| 2026-05-18 | DB crashed 4× in one afternoon | `checkPremiumAccess` with whole `user` object in deps → 16 queries/user | `useRef` guard + `user?.id` in deps |
| 2026-05-18 | `portal_has_stripe` 167× more calls than expected | `PortalPage` without `staleTime` | `staleTime: 5 * 60 * 1000` |

---

## Compatibility

Designed for:
- React 18 + TanStack Query v5
- Supabase with PostgREST + Auth + RLS
- SPA (no SSR)

For Next.js/SSR projects, adjust Layer 1 (server-side auth patterns differ).

## Important: Commands That Do NOT Work via MCP

The following must run in the **Supabase Dashboard SQL Editor** (not via MCP `execute_sql`):
- `ALTER SYSTEM SET ...` — requires superuser outside a transaction
- `VACUUM ANALYZE` — cannot run inside a transaction
- `SELECT pg_reload_conf()` — requires superuser
