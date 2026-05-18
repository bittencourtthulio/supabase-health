---
name: supabase-health
description: Use when auditing any Supabase project for production performance issues, database crashes, connection pool saturation, or slow queries. Triggers on symptoms like repeated DB crashes, query count spikes, slow RLS evaluation, or post-restart planner degradation.
---

# /supabase-health — Supabase Production Audit

Systematic 6-layer audit for Supabase projects. Every check maps to a real production incident class. Run layers in order — earlier layers are the most critical.

Works with any frontend (React, Next.js, SvelteKit, Vue) and any data fetching library. DB layers (3–6) are framework-agnostic.

---

## Layer 1 — Thundering Herd: Auth State + DB Queries

**Why this crashes the DB:** Supabase Auth triggers 2–4 sequential state changes on load (loading → user → roles). Any subscription or effect that has the full `user` object or `session` in its dependency list re-runs 2–4× per user per navigation. With 10 concurrent users: 10× 4 = 40–80× the expected query count. Connection pool saturates in seconds.

**Pattern class:** Any hook, effect, or listener that (a) queries the DB and (b) re-runs on every auth state change rather than only on meaningful changes.

### Check 1.1 — Effects querying the DB on auth state changes

Search for effects that combine DB calls with auth object dependencies:

```bash
# Find all files that use both auth state and effects
grep -rn "useEffect\|createEffect\|$effect" src/ --include="*.tsx" --include="*.ts" --include="*.svelte" --include="*.vue" -l
```

For each file, look for this dangerous pattern:

```typescript
// DANGEROUS — whole user/session object in dependency list:
useEffect(() => {
  db.from("table").select(...)   // or any DB call
}, [..., user, ...]);            // user object changes on every auth tick

useEffect(() => {
  db.from("table").select(...)
}, [..., session, ...]);         // session changes on every token refresh
```

**Required fix — use a stable scalar ID, not the whole object:**

```typescript
// Add a concurrency guard + use user?.id (stable scalar) instead of user:
const inFlight = useRef(false);
const lastKey = useRef<string | null>(null);

useEffect(() => {
  const key = `${stableParam1}:${stableParam2}:${user?.id}`;
  if (lastKey.current === key) return;   // already ran for these params
  if (inFlight.current) return;          // already running

  const run = async () => {
    inFlight.current = true;
    try {
      // DB queries here
      lastKey.current = key;
    } finally {
      inFlight.current = false;
    }
  };
  run();
}, [stableParam1, stableParam2, user?.id]);  // ← scalar ID, not object
```

### Check 1.2 — Auth state listeners querying the DB on every event

```bash
grep -rn "onAuthStateChange\|onAuthChange\|authStateChange" src/ --include="*.tsx" --include="*.ts" --include="*.svelte" -l
```

For each listener, verify:
- `TOKEN_REFRESHED` with a valid session does **not** query the DB — it only updates in-memory state
- DB queries (roles, profile, permissions) only fire on `SIGNED_IN` and `INITIAL_SESSION`
- Any function inside a listener that queries the DB has a concurrency guard

```typescript
// CORRECT — skip DB on token refresh:
if (event === "TOKEN_REFRESHED" && session) {
  setSession(session);
  setUser(session.user);
  return;  // ← NO DB queries here
}
```

---

## Layer 2 — Data Fetching Without Cache TTL

**Why this crashes the DB:** Without a staleness time (cache TTL), every navigation re-fetches all data — even if the user returned 5 seconds ago. In SPAs with frequent back/forward navigation, each transition fires 3–10 unnecessary queries.

**Pattern class:** Any data fetching hook with no cache TTL configured. Applies to TanStack Query (`staleTime`), SWR (`dedupingInterval`), Apollo (`fetchPolicy`), and custom fetch hooks.

### Check 2.1 — Queries without staleness configuration

**TanStack Query:**
```bash
grep -rn "useQuery\|useSuspenseQuery" src/ --include="*.tsx" --include="*.ts" | grep -v "staleTime" | grep -v "//.*useQuery" | grep -v ".test."
```

**SWR:**
```bash
grep -rn "useSWR" src/ --include="*.tsx" --include="*.ts" | grep -v "dedupingInterval\|refreshInterval" | grep -v ".test."
```

Every result is a candidate for adding a cache TTL. Legitimate exceptions:
- Data that MUST always be fresh (unread notification counts, shopping cart totals)
- Lazy queries (`enabled: false`) that only run on explicit user action

**Safe defaults by data type:**

| Data type | Recommended TTL |
|---|---|
| User profile / roles | 5–10 min |
| Content (courses, articles, products) | 5–10 min |
| Lists / directories | 2–5 min |
| Counts / aggregates | 1–2 min |
| Unread counts / real-time badges | Use Realtime instead |

```typescript
// TanStack Query — safe pattern:
const { data } = useQuery({
  queryKey: ['resource', id, user?.id],
  queryFn: () => fetchResource(id!),
  enabled: !!id && !!user?.id,
  staleTime: 5 * 60 * 1000,
});
```

### Check 2.2 — Polling on heavy queries

```bash
grep -rn "refetchInterval\|refreshInterval\|pollingInterval" src/ --include="*.tsx" --include="*.ts" --include="*.svelte" --include="*.vue"
```

**Rule:** Polling is forbidden on queries that JOIN 2+ tables beyond the main table. Use Supabase Realtime instead.

```typescript
// FORBIDDEN — polling with a multi-join query:
useQuery({
  queryFn: () => fetchWithMultipleJoins(),
  refetchInterval: 30_000,  // 30s × N users = DB overwhelmed
});

// CORRECT pattern:
// 1. useEffect with supabase.channel() → increment an in-memory counter
// 2. useQuery with enabled: false + manual refetch() on user click
```

---

## Layer 3 — RLS: Policies That Kill Performance

**Why this crashes the DB:** Poorly written RLS policies are evaluated **per row**, not once per query. With large tables and concurrent users, this multiplies CPU and RAM cost until saturation.

### Check 3.1 — auth.uid() directly in policies (RescanCond)

```sql
-- Find policies evaluating auth.uid() per row:
SELECT schemaname, tablename, policyname, qual
FROM pg_policies
WHERE qual LIKE '%auth.uid()%'
  AND qual NOT LIKE '%(SELECT auth.uid())%';
```

**Every result must be fixed:**

```sql
-- WRONG — evaluated per row (RescanCond):
CREATE POLICY "user_rows" ON my_table
  USING (user_id = auth.uid());

-- CORRECT — evaluated once per query (InitPlan):
CREATE POLICY "user_rows" ON my_table
  USING (user_id = (SELECT auth.uid()));
```

The same applies to `auth.jwt()` and `auth.role()` — always wrap in `(SELECT ...)`.

### Check 3.2 — Multiple PERMISSIVE policies for the same command

```sql
-- Find tables with multiple SELECT/ALL policies:
SELECT tablename, COUNT(*) as policy_count, array_agg(policyname) as policies
FROM pg_policies
WHERE cmd IN ('SELECT', 'ALL')
GROUP BY tablename
HAVING COUNT(*) > 1
ORDER BY policy_count DESC;
```

**Any result with count > 1 is a problem:** PostgreSQL evaluates ALL PERMISSIVE policies with OR and no short-circuit — every policy runs for every row.

```sql
-- WRONG — two separate SELECT policies:
CREATE POLICY "admins" ON resources FOR ALL
  USING (is_admin((SELECT auth.uid())));
CREATE POLICY "published" ON resources FOR SELECT
  USING (is_published = true);

-- CORRECT — one unified SELECT policy with OR:
CREATE POLICY "resources_select" ON resources FOR SELECT
  USING (is_admin((SELECT auth.uid())) OR is_published = true);

-- Keep write policies separate per command:
CREATE POLICY "resources_insert" ON resources FOR INSERT WITH CHECK (...);
CREATE POLICY "resources_update" ON resources FOR UPDATE USING (...);
```

### Check 3.3 — Foreign keys without covering indexes

```sql
-- Find FK columns without a B-tree index:
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

**Every result is a potential seq scan on JOINs and CASCADE operations.**

```sql
-- Fix: create a B-tree index for each unindexed FK:
CREATE INDEX IF NOT EXISTS idx_my_table_fk_col ON public.my_table(fk_col);
```

---

## Layer 4 — Database Configuration

Run in **Supabase Dashboard SQL Editor** (requires superuser — does not work via MCP):

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

If `work_mem` exceeds the table value for your tier:
```sql
ALTER SYSTEM SET work_mem = '4MB';
SELECT pg_reload_conf();
```

### Check 4.2 — idle_in_transaction_session_timeout

Must be ≤ 30s. Supabase default is 0 (never times out):

```sql
ALTER SYSTEM SET idle_in_transaction_session_timeout = '30s';
SELECT pg_reload_conf();
```

Value 0 means connections stuck in open transactions accumulate indefinitely — a common cause of periodic OOM crashes.

### Check 4.3 — Active zombie connections

```sql
SELECT state, count(*) as qty,
  MAX(EXTRACT(EPOCH FROM (now() - state_change))::int) as max_secs,
  SUM(CASE WHEN state = 'idle in transaction' THEN 1 ELSE 0 END) as idle_in_tx
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state ORDER BY qty DESC;
```

If `idle_in_tx > 0`, investigate which queries are stuck:

```sql
SELECT pid, usename, application_name,
  EXTRACT(EPOCH FROM (now() - state_change))::int as secs_idle_in_tx,
  left(query, 120) as last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY secs_idle_in_tx DESC;
```

---

## Layer 5 — Planner Stats After Restart

**Critical context:** Every DB restart (crash, PITR restore, maintenance) resets `pg_stat_user_tables.n_live_tup` to zero. With zeroed stats, the PostgreSQL planner can't estimate table sizes and may choose seq scan over index scan — even with correct indexes in place.

### Check 5.1 — Verify n_live_tup for main tables

```sql
-- Replace table names with your project's main tables:
SELECT relname, n_live_tup, n_dead_tup, seq_scan, idx_scan
FROM pg_stat_user_tables
ORDER BY seq_scan DESC
LIMIT 20;
```

If `n_live_tup = 0` on tables that should have data, autovacuum hasn't run yet. Force it in the Dashboard SQL Editor:

```sql
-- Run for each table showing n_live_tup = 0:
VACUUM ANALYZE public.your_table_name;
```

> **Note:** `VACUUM ANALYZE` cannot run inside a transaction — it does not work via MCP `execute_sql`. Use the Dashboard SQL Editor.

### Check 5.2 — High seq scans despite having indexes

```sql
SELECT relname, seq_scan, seq_tup_read,
  idx_scan,
  round(seq_tup_read::numeric / NULLIF(seq_scan, 0)) as avg_rows_per_seqscan
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC;
```

If `avg_rows_per_seqscan` is high (3,000+) on a table that has indexes, the planner is ignoring them — likely due to zeroed stats. Run `VACUUM ANALYZE` on that table.

---

## Layer 6 — Heaviest Production Queries

```sql
SELECT
  round(total_exec_time::numeric, 1) as total_ms,
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
- Same query with dozens of calls in a short window → thundering herd (see Layer 1)
- `mean_ms > 100` → heavy query, inspect with `EXPLAIN (ANALYZE, BUFFERS)`
- Any RPC with disproportionately high call counts → N+1 not resolved
- Queries selecting `*` on large tables → add column projection

---

## Commands That Do NOT Work via MCP

The following require the **Supabase Dashboard SQL Editor** (not MCP `execute_sql`):

| Command | Why |
|---|---|
| `ALTER SYSTEM SET ...` | Requires superuser outside a transaction |
| `VACUUM ANALYZE` | Cannot run inside a transaction |
| `SELECT pg_reload_conf()` | Requires superuser |

---

## Compatibility Notes

- **DB layers (3–6):** Apply to any Supabase project regardless of frontend
- **Layer 1–2:** Examples use React/TanStack Query — adapt patterns to your framework:
  - SvelteKit: `$effect`, `$derived`, Svelte stores
  - Vue: `watchEffect`, `watch`, Pinia
  - Next.js/SSR: auth patterns differ on the server side (no `useEffect` anti-pattern, but watch for server components re-fetching on every request without `cache()`)
