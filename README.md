# Postgres MCP Server Setup 

This SOP walks you from zero to a working **PostgreSQL MCP server** wired into **Claude Desktop** with DBA‑grade visibility and optional load testing using **pgbench**. It assumes a local Postgres and one target database **`pagila`**.

---

## Prerequisites

* macOS with Postgres running locally and reachable on `127.0.0.1:5432`.
* A database to monitor (e.g., `pagila`).
* Admin/superuser access to run SQL grants and enable extensions.
* Python installed; you’ll create a virtualenv in **Step 3** with `python3 -m venv .venv && source .venv/bin/activate` (or use Docker as an alternative).
* Claude Desktop installed.

> **Tip:** If your password contains special characters, URL‑encode it in connection strings.

---

## 1) Create the Monitoring Role (Least‑Privilege)

Connect as an admin and run:

```sql
-- 1) Create a login for the MCP server
CREATE ROLE claude_ro LOGIN PASSWORD '**********';

-- 2) Allow it to connect to the databases you want to monitor (repeat per DB)
GRANT CONNECT ON DATABASE pagila TO claude_ro;

-- 3) Grant cluster-wide monitoring visibility
GRANT pg_monitor TO claude_ro;

-- 4) (Optional) If you want this user to read table data (not required for many checks)
GRANT USAGE ON SCHEMA public TO claude_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO claude_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO claude_ro;
```

> **Notes**
>
> * `ALTER DEFAULT PRIVILEGES` affects **future** objects created by the issuing role. Keep the explicit `GRANT SELECT ON ALL TABLES` to cover existing tables.
> * Repeat schema grants if you have non‑`public` schemas.

---

## 2) Enable Workload Visibility (`pg_stat_statements`)

**Self‑managed Postgres:**

```sql
-- Inspect current preload list
SHOW shared_preload_libraries;

-- If nothing is set yet:
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';

-- If you already have entries, append carefully (example):
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements,auto_explain';
```

Restart Postgres (pick one):

```bash
brew services restart postgresql@15   # or your version
pg_ctl -D /path/to/data restart
```

Then enable the extension in the target DB:

```sql
\c pagila
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

> **Managed services (RDS/Aurora/etc.)** use a DB parameter group to set `shared_preload_libraries`, apply it, **reboot**, then `CREATE EXTENSION` per DB.

---

## 3) Install & Run the Postgres MCP Server

```bash
git clone https://github.com/crystaldba/postgres-mcp.git
cd postgres-mcp

# create and activate a virtualenv (recommended)
python3 -m venv .venv && source .venv/bin/activate

# upgrade pip and install
python -m pip install --upgrade pip
pip install postgres-mcp

# verify install
python -m pip show postgres-mcp
which postgres-mcp    # may still be blank on some shells

# run it (two options)
postgres-mcp --access-mode=restricted
# or, if PATH still quirky, use the module form:
python -m postgres_mcp --access-mode=restricted

# set DB URL and run
export DATABASE_URI="postgresql://claude_ro:password@127.0.0.1:5432/pagila"
postgres-mcp --access-mode=restricted

# discover absolute binary path (use this for Claude config)
realpath .venv/bin/postgres-mcp
# -> /Users/MCP-Server/postgres-mcp/.venv/bin/postgres-mcp
```

> **Access mode:** Keep `--access-mode=restricted` for read‑only guardrails.

---

## 4) Wire Into Claude Desktop

Edit (or create) the config at:

```
~/Library/Application Support/Claude/claude_desktop_config.json
```

Insert/merge the following under a single top‑level `"mcpServers"` object:

```json
{
  "mcpServers": {
    "postgres": {
      "command": "/Users/MCP-Server/postgres-mcp/.venv/bin/postgres-mcp",
      "args": ["--access-mode=restricted"],
      "env": {
        "DATABASE_URI": "postgresql://claude_ro:*********@127.0.0.1:5432/pagila?application_name=postgres-mcp"
      }
    }
  }
}
```

Now **Quit Claude Desktop (⌘Q)** and reopen. Start a **new chat**.

---

## 5) Verify the Connection

In Claude, ask it to run (table output only):

```sql
SELECT current_database() AS db,
       current_user       AS usr,
       current_setting('application_name', true) AS app,
       inet_client_addr() AS client_ip,
       pg_backend_pid()   AS pid;
```

You should see `db=pagila`, `usr=claude_ro`, `app=postgres-mcp`, `client_ip=127.0.0.1`.

---

## 6) Load Testing with `pgbench`

> Ensure `pgbench` is installed (via Homebrew Postgres, Postgres.app, or Docker). The examples below use a separate test DB `loadtest`.

```bash
createdb loadtest || true
pgbench -i -s 10 loadtest
pgbench -S -M prepared -c 10 -j 4 -T 60 -r loadtest
pgbench -M prepared -c 64 -j 32 -T 300 -r loadtest
```

> **Tip:** After each batch, you can reset stats for clean analysis:
>
> ```sql
> SELECT pg_stat_statements_reset();
> ```

---

## 7) Ready‑Made DBA Prompts (Paste into Claude)

### Health & Overview

* "Run a database health check: uptime, version, connections vs max_connections, active vs idle, longest-running queries (>5m), locks, temp files usage, and any red flags. End with a 5-item action list."
* "Summarize top resource consumers by database and user (CPU/time from stats, connections, temp usage). Provide a table and quick recommendations."

### Sessions, Locks, Blocking

* "List sessions running > 5 minutes. For each, show pid, user, db, wait_event, lock type, and the normalized query text. Identify any blockers and the blocked tree."
* "Show current blocking ↔ blocked pairs with relation names and lock modes; include a ‘safe resolution’ note for each case."
* "Report on frequent lock wait events over the last hour and which tables are most affected."

### Workload (requires `pg_stat_statements`)

* "From pg_stat_statements, show top 15 query fingerprints by total_time and mean_time with rows, shared_blks_read/hit, and planning/execution time. Add index/plan hints per fingerprint."
* "Find ‘chatty’ queries: high calls, low mean time, but large total_time. Recommend batching/caching or statement redesign."
* "Find queries with high rows_removed_by_filter or mismatched row estimates; suggest selective indexes or predicate rewrites."

### Indexes (hypothetical + usage)

* "Using pg_stat_statements, pick the 5 slowest normalized fingerprints and simulate candidate indexes with hypopg, showing estimated plan cost before/after. Output only CREATE INDEX ... statements and a ranked recommendation. Do not create real indexes."
* "Find unused/rarely used indexes (low idx_scan, high size) and overlapping/duplicate indexes. Recommend drops/merges with justification."
* "For table public.<TABLE>, propose 3–5 composite indexes (different column orders) based on workload; evaluate with hypopg and rank them."

### Vacuum, Bloat & Storage

* "Estimate table and index bloat for the 20 largest relations; sort by wasted bytes. Propose VACUUM (FULL) / REINDEX candidates and safer alternatives with expected space reclaimed."
* "Autovacuum status: tables with high dead tuples, recent vacuums, last autovacuum times, freeze age. Recommend per-table autovacuum_* thresholds and cost settings."
* "Show relations with high HOT-update churn or TOAST activity and suggest schema/index adjustments to reduce heap bloat."

### Temp Files, Sorts & Work_mem

* "Identify queries causing temp file usage and external sorts/hash spills. Estimate a safe per-session work_mem strategy and suggest targeted bumps rather than global increases."
* "List top statements by temp bytes and propose fixes (indexes, query rewrite, memory tuning)."

### Configuration Review (read‑only)

* "Review key settings vs workload: shared_buffers, effective_cache_size, work_mem, maintenance_work_mem, max_wal_size, checkpoint_timeout, autovacuum_vacuum_cost_*. Suggest safe starting values with rationale based on DB size and observed stats."
* "Detect parameter mismatches for OLTP: checkpoints too frequent, WAL growth spikes, autovacuum lag indicators. Produce a JSON of parameter → suggested_value → reason."

### WAL, Replication & Backups

* "Show replication lag per standby in bytes and time, recent WAL generation rate per hour, and whether max_wal_size/wal_compression need tuning. Include recommendations for checkpoint cadence."
* "Produce a 24h table of WAL generated/hour and correlate with top workload windows."

### Security & Roles (read‑only)

* "List roles with superuser/createdb/createrole/bypassrls; flag risks and propose least-privilege changes (advice only)."
* "Show active connections by client_addr and ssl state; suggest IP allow-list tightening and TLS enforcement."

### App/ORM Patterns

* "Detect N+1 patterns: very frequent short queries per session/user. Recommend ORM eager loading/batching and caching strategies."
* "Find the hottest tables by write load and update patterns; suggest index/schema changes to reduce heap churn."

### Maintenance Planning

* "Generate a weekly maintenance plan: top bloat candidates, expected reclaim, REINDEX list, and a safe sequence with time windows. Include a rollback/abort note."
* "List dead-tuple hotspots and propose per-table autovacuum_vacuum_scale_factor/threshold overrides."

### Incident Triage (one‑shot)

* "Right now, diagnose performance: top blockers, longest runners, temp spillers, autovacuum interference, and immediate safe mitigations. End with a prioritized 5-item action list and owner suggestions."

### Sanity/Context Prompts

* "What’s the connected database/user/host? Show: current_database(), current_user, inet_server_addr(), current_setting('application_name', true)."
* "Since when were pg_stat_statements last reset? Are stats representative? If not, caveat recommendations."

---

## Troubleshooting

* **`nodename nor servname provided`** → The `DATABASE_URI` host is missing/typo. Use `127.0.0.1` for local. Verify with `psql "$DATABASE_URI" -c "select version();"`.
* **`must be loaded via shared_preload_libraries`** → You edited the setting but didn’t **restart**. Restart then `CREATE EXTENSION` again.
* **Claude doesn’t see the DB** → Confirm absolute path to `postgres-mcp` in config; ensure JSON is valid; **⌘Q** restart Claude; add `application_name=postgres-mcp` and verify in `pg_stat_activity`.
* **Password with special chars** → URL‑encode in the URI (e.g., `@` → `%40`).

---

## Security & Rollback

* Keep production sessions in `--access-mode=restricted`.
* To revoke access or remove the role:

```sql
REVOKE pg_monitor FROM claude_ro;
REVOKE USAGE ON SCHEMA public FROM claude_ro;
REVOKE SELECT ON ALL TABLES IN SCHEMA public FROM claude_ro;
-- (Revoke any other grants as needed)
DROP ROLE IF EXISTS claude_ro;
```

---
## Deployment Screenshot

## Appendix: Useful One‑liners

**Tag connections so you can spot them:**

```bash
export DATABASE_URI="postgresql://claude_ro:********@127.0.0.1:5432/pagila?application_name=postgres-mcp"
```

**Who’s connected (from the DB side):**

```sql
SELECT pid, datname, usename, application_name, client_addr, state
FROM pg_stat_activity
WHERE usename = 'claude_ro'
ORDER BY pid DESC;
```

**Reset workload stats before a test batch:**

```sql
SELECT pg_stat_statements_reset();
```

---

**You’re set.** Use the prompts above inside Claude to drive DBA analysis, and the pgbench commands to generate load for realistic tuning. If you want a variant for RDS/Aurora (parameter groups, SSL, SG rules), we can add a short section next.
