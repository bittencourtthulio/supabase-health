# supabase-health — Claude Code Skill

> **A battle-tested 6-layer production audit for any Supabase project.**  
> Every check maps to a real class of database crash. No theory — pure incident patterns.

---

## What Is This?

`supabase-health` is a [Claude Code skill](https://agentskills.io/specification) that performs a systematic audit of your Supabase project, covering the patterns that *actually* crash production databases.

This skill was built from real production incidents. Every check represents a confirmed crash pattern with a root cause and a verified fix. The DB layers (3–6) are fully framework-agnostic. The frontend layers (1–2) cover React, Next.js, SvelteKit, and Vue.

---

## Install

```bash
# Install globally (available in all your projects)
npx skills add bittencourtthulio/supabase-health
```

Or install manually:

```bash
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

### Layer 1 — Thundering Herd: Auth State + DB Queries

The silent killer of Supabase projects. Supabase Auth triggers 2–4 sequential state changes on load. If any effect or subscription has the full `user` object or `session` in its dependency list and also queries the DB, it re-runs on every auth tick — multiplying query counts by 40–80× under concurrent users.

✅ **Checks for:**
- Effects/subscriptions with `user` or `session` objects (not just IDs) in their dependency list that also query Supabase
- Auth state listeners that query the DB on `TOKEN_REFRESHED` events
- Missing concurrency guards (`useRef`, flags, or abort controllers)

---

### Layer 2 — Data Fetching Without Cache TTL

Without a staleness time, every page navigation re-fetches all data — even if the user navigated back 5 seconds ago. In SPAs with frequent back/forward navigation, each transition fires 3–10 unnecessary queries.

✅ **Checks for:**
- `useQuery` / `useSWR` / equivalent calls missing a cache TTL (`staleTime`, `dedupingInterval`)
- Polling intervals (`refetchInterval`) on queries with multi-table JOINs — use Supabase Realtime instead

---

### Layer 3 — RLS Policies That Kill Performance

RLS policies are evaluated **per row** by default. Poorly written policies — especially those calling `auth.uid()` directly — run a function call for every row scanned. With large tables and concurrent users, this multiplies CPU and RAM cost until saturation.

✅ **Checks for:**
- `auth.uid()` without `(SELECT auth.uid())` wrapper — the wrapper forces single evaluation per query (InitPlan vs RescanCond)
- Multiple PERMISSIVE policies for the same command on the same table — PostgreSQL evaluates all of them in OR with no short-circuit
- Foreign key columns without B-tree covering indexes — causes seq scans on every JOIN and CASCADE

---

### Layer 4 — Database Configuration

Supabase's default configuration is not tuned for high-concurrency SaaS workloads. Two defaults in particular cause production incidents at scale.

✅ **Checks for:**
- `work_mem` × `max_connections` exceeding safe RAM limits — includes a per-tier reference table
- `idle_in_transaction_session_timeout = 0` (Supabase default) — connections stuck in open transactions accumulate indefinitely, causing periodic OOM
- Active zombie connections in `idle in transaction` state

---

### Layer 5 — Planner Stats After Restart

Every DB restart resets `pg_stat_user_tables.n_live_tup` to zero. With zeroed stats, the PostgreSQL planner doesn't know table sizes and may choose seq scan over index scan — even with correct indexes in place. This is invisible until load increases.

✅ **Checks for:**
- `n_live_tup = 0` on tables that should have data (autovacuum hasn't run post-restart)
- High `avg_rows_per_seqscan` on indexed tables, indicating the planner is ignoring indexes

---

### Layer 6 — Heaviest Production Queries

Uses `pg_stat_statements` to surface the actual most expensive queries running in production — not synthetic benchmarks.

✅ **Checks for:**
- Same query with many calls in a short window (thundering herd fingerprint)
- Queries with `mean_ms > 100` (candidates for `EXPLAIN ANALYZE`)
- RPCs with call counts disproportionate to their expected usage (N+1 fingerprint)

---

## Compatibility

| | Support |
|---|---|
| **Supabase** | Any project with PostgREST + Auth + RLS |
| **DB layers (3–6)** | Any Supabase project, any frontend |
| **React** | 18+ with any data fetching library |
| **Next.js** | App Router and Pages Router (notes on SSR differences included) |
| **SvelteKit** | `$effect`, Svelte stores |
| **Vue** | `watchEffect`, Pinia |
| **TanStack Query** | v4 and v5 |
| **SWR** | Any version |

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

Found a new pattern that causes production crashes? Open a PR adding the pattern to the relevant layer with:

- The symptom (what you observed)
- The root cause (what actually happened)
- The fix (what resolved it)
- The check (grep/SQL to detect it in other projects)

This skill grows from real incidents — production scars welcome.

---

## License

MIT
