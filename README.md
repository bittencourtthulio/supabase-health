# supabase-health — Claude Code Skill

> **A battle-tested 6-layer production audit for React + Supabase projects.**  
> Every check maps to a real database crash. No theory — pure incident response.

---

## What Is This?

`supabase-health` is a [Claude Code skill](https://agentskills.io/specification) that performs a systematic audit of your Supabase project, covering the patterns that *actually* crash production databases in React SaaS apps.

This skill was built from **3 weeks of production incidents** on a live React 18 + Supabase + TanStack Query platform. Every single check represents a real crash with a timestamp, root cause, and confirmed fix.

---

## Install

```bash
# Install globally (available in all projects)
npx skills add bittencourtthulio/supabase-health
```

Or install manually:

```bash
# Copy the skill file to your Claude skills directory
curl -o ~/.claude/skills/supabase-health.md \
  https://raw.githubusercontent.com/bittencourtthulio/supabase-health/main/supabase-health.md
```

---

## Usage

Once installed, type in Claude Code:

```
/supabase-health
```

Claude will run the full audit, layer by layer, checking both your codebase and database.

---

## What Gets Audited

### Layer 1 — Thundering Herd: useEffect + useAuth

The silent killer of Supabase projects. `useAuth` triggers 2–4 sequential renders on load. If a `useEffect` has `user` (the whole object) or `session` in its deps, it re-runs on every render — multiplying DB queries by 40–80× under concurrent users.

**Real incident:** `checkPremiumAccess` fired 16 queries/user (expected: 3–4). DB crashed 4× in one afternoon with under 20 users.

✅ **Checks for:**
- `useEffect` with `user` or `session` in deps that also query Supabase
- `onAuthStateChange` listeners that query the DB on `TOKEN_REFRESHED`
- Missing `useRef` concurrency guards

---

### Layer 2 — React Query Without `staleTime`

Without `staleTime`, every page navigation re-executes all queries. In SPAs with frequent back/forward navigation, each transition fires 3–10 unnecessary queries.

**Real incident:** `PortalPage` without `staleTime` caused a 167× query ratio — portals was queried 167× more than expected per navigation.

✅ **Checks for:**
- `useQuery` calls missing `staleTime`
- `refetchInterval` on queries with 2+ JOIN levels (use Realtime instead)

---

### Layer 3 — RLS Policies That Kill Performance

Poorly written RLS policies are evaluated **per row**, not once per query. With large tables and concurrent users, this multiplies CPU and RAM cost until saturation.

**Real incident:** 691 policies with direct `auth.uid()` (RescanCond) caused CPU/RAM saturation. Fixed by wrapping with `(SELECT auth.uid())` to force InitPlan evaluation.

✅ **Checks for:**
- `auth.uid()` without `(SELECT ...)` wrapper in policies
- Multiple PERMISSIVE policies for the same command on the same table
- Foreign keys without B-tree covering indexes

---

### Layer 4 — Database Configuration

Supabase's default configuration is not optimized for production SaaS workloads. The defaults can cause periodic OOM and connection accumulation under load.

**Real incident:** `work_mem = 12MB` × 160 connections = 5.7 GB potential memory usage on a 4 GB instance → periodic OOM crashes.

✅ **Checks for:**
- `work_mem` × `max_connections` exceeding safe RAM limits (per-tier table included)
- `idle_in_transaction_session_timeout = 0` (Supabase default — zombie connection accumulation)
- Active zombie connections (`idle in transaction` state)

---

### Layer 5 — Planner Stats After Restart

Every DB restart (crash, PITR restore, maintenance) resets `pg_stat_user_tables.n_live_tup` to zero. With zeroed stats, the PostgreSQL planner can't estimate table sizes and may choose seq scan over index scan — even with correct indexes in place.

✅ **Checks for:**
- `n_live_tup = 0` on tables that should have data (autovacuum hasn't run yet)
- High `avg_rows_per_seqscan` indicating the planner is ignoring indexes

---

### Layer 6 — Heaviest Production Queries

Uses `pg_stat_statements` to surface the actual most expensive queries running in production.

✅ **Checks for:**
- Thundering herd patterns (same query with dozens of calls in minutes)
- Queries with `mean_ms > 100`
- N+1 RPCs with disproportionate call counts

---

## Production Incident Timeline

| Date | Problem | Root Cause | Fix |
|------|---------|-----------|-----|
| 2026-05-11 | DB crash in 30s | `NotificationBell` with `refetchInterval: 30000` + 3-level JOIN | Realtime + lazy query |
| 2026-05-14 | Periodic OOM | `work_mem = 12MB` × 160 connections | `work_mem = 4MB` |
| 2026-05-14 | CPU/RAM saturation | 691 policies with direct `auth.uid()` (RescanCond) | `(SELECT auth.uid())` everywhere |
| 2026-05-14 | Query degradation | 534 warnings, multiple PERMISSIVE policies on 27 tables | Unified SELECT policies with OR |
| 2026-05-14 | Seq scans on JOINs | 26 FKs without B-tree index | `CREATE INDEX` on FKs |
| 2026-05-17 | DB crash in 2–3 min | `checkAdminRole` on `TOKEN_REFRESHED` without guard | `useRef` guard + skip TOKEN_REFRESHED |
| 2026-05-18 | DB crashed 4× in one afternoon | `checkPremiumAccess` with `user` object in deps → 16 queries/user | `useRef` guard + `user?.id` in deps |
| 2026-05-18 | `portal_has_stripe` 167× more calls than expected | `PortalPage` without `staleTime` | `staleTime: 5 * 60 * 1000` |

---

## Compatibility

| Technology | Version |
|---|---|
| React | 18+ |
| TanStack Query | v5 |
| Supabase | Any (PostgREST + Auth + RLS) |
| App type | SPA (no SSR) |

> For **Next.js/SSR** projects, Layer 1 patterns differ (server-side auth). The DB layers (3–6) apply universally.

---

## Important Limitations

Some checks **cannot run via MCP `execute_sql`** and must be executed in the **Supabase Dashboard SQL Editor**:

| Command | Why |
|---|---|
| `ALTER SYSTEM SET ...` | Requires superuser outside a transaction |
| `VACUUM ANALYZE` | Cannot run inside a transaction |
| `SELECT pg_reload_conf()` | Requires superuser |

The skill explicitly tells you when to switch to the Dashboard.

---

## Contributing

Found a new production pattern that causes crashes? Open a PR adding it to the incident table with:
- Date (approximate)
- Problem description
- Root cause
- Fix applied

This skill lives and grows from real incidents.

---

## License

MIT — use it, fork it, deploy it.

---

*Built from real crashes on a live SaaS platform. Every check has a body count.*
