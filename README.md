# PharmApp FAERS — Quickstart (Ubuntu & Windows 11)

**Repository:** https://github.com/nghiencuuthuoc/pharmapp-faers  
**Reference repo (style & conventions):** https://github.com/nghiencuuthuoc/pharmapp-chembl-35  
**Prebuilt dump (.dump):** https://drive.google.com/drive/folders/1x8U87NpS3uRuxpHBRsxZa7LvhdlGeTCC?usp=sharing

This guide helps you **restore the FAERS PostgreSQL database** from a prebuilt dump on both **Ubuntu** and **Windows 11**, and provides sanity‑check queries and optional tuning/indexing. It follows the same AI/ML Ops flavor used across PharmApp repos (folder conventions, reproducibility, scripts).

> **Why a `.dump` (custom format)?** It’s compressed and supports **parallel restore** with `pg_restore -j` → much faster and safer than raw SQL.

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Folder Layout](#folder-layout)
- [Windows 11 — Restore from `.dump` (Recommended)](#windows-11--restore-from-dump-recommended)
- [Ubuntu — Restore from `.dump`](#ubuntu--restore-from-dump)
- [Sanity Checks & First Queries](#sanity-checks--first-queries)
- [Optional Indexes](#optional-indexes)
- [Performance Tips (Optional)](#performance-tips-optional)
- [Troubleshooting](#troubleshooting)
- [Create Your Own Dump (Optional)](#create-your-own-dump-optional)
- [License & Notes](#license--notes)

---

## Prerequisites

- **PostgreSQL 17.x** (server + client tools) on your machine.
  - Verify with:
    ```bash
    psql --version
    pg_restore --version
    ```
- (Optional) **Conda env** you use for PharmApp, e.g. `pa3`.
- Enough disk space: FAERS can be **tens of GB** when restored.

---

## Folder Layout

Suggested layout following other PharmApp repos:
```
pharmapp-faers/
├─ README.md
├─ scripts/
│  ├─ restore_from_dump.bat      # Windows helper (optional)
│  └─ restore_from_dump.sh       # Ubuntu helper (optional)
└─ docs/
   └─ queries.sql                # Sample sanity queries (optional)
```

You may keep the `.dump` anywhere (e.g. `E:\PharmAppDev\database\FAERS\faers_db_YYYYMMDD.dump`).

---

## Windows 11 — Restore from `.dump` (Recommended)

1) **Download** the latest `.dump` from Google Drive:  
   https://drive.google.com/drive/folders/1x8U87NpS3uRuxpHBRsxZa7LvhdlGeTCC?usp=sharing

2) **Open CMD** (or *Anaconda Prompt* if you use Conda) and run:

```cmd
REM 1) Create role nct (if missing)
psql -h localhost -U postgres -c "DO $$ BEGIN IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='nct') THEN CREATE ROLE nct LOGIN PASSWORD 'nct'; ALTER ROLE nct CREATEDB; END IF; END $$;"

REM 2) Recreate database (COLLATE/CTYPE C for portability; if it errors, omit them)
psql -h localhost -U postgres -c "DROP DATABASE IF EXISTS faers_db;"
psql -h localhost -U postgres -c "CREATE DATABASE faers_db OWNER nct TEMPLATE template0 ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C';"
REM If above fails on collations, use:
REM psql -h localhost -U postgres -c "CREATE DATABASE faers_db OWNER nct TEMPLATE template0 ENCODING 'UTF8';"

REM 3) Parallel restore (edit the path to your dump)
pg_restore -h localhost -U nct -d faers_db --clean --if-exists --no-owner --no-privileges --role=nct -j 6 "E:\PharmAppDev\database\FAERS\faers_db_YYYYMMDD.dump"

REM 4) Sanity check
psql -h localhost -U nct -d faers_db -c "\dt+"
psql -h localhost -U nct -d faers_db -c "SELECT current_user, current_database();"
```

> **Notes**
> - Change `-j 6` to fit your CPU/SSD (4–12 is typical).
> - If `pg_restore` asks for a password, enter `nct` (or what you set).
> - If you see extension errors (e.g., `pgcrypto`), create them:
>   ```cmd
>   psql -h localhost -U postgres -d faers_db -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
>   psql -h localhost -U postgres -d faers_db -c "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";"
>   ```

---

## Ubuntu — Restore from `.dump`

1) **Install PostgreSQL 17** (or compatible) and client tools.  
2) Run the following (adjust the path to your `.dump` file):

```bash
# 1) Create role nct (if missing)
psql -h localhost -U postgres -c "DO \$\$ BEGIN IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='nct') THEN CREATE ROLE nct LOGIN PASSWORD 'nct'; ALTER ROLE nct CREATEDB; END IF; END \$\$;"

# 2) Recreate database
psql -h localhost -U postgres -c "DROP DATABASE IF EXISTS faers_db;"
psql -h localhost -U postgres -c "CREATE DATABASE faers_db OWNER nct TEMPLATE template0 ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C';"
# If locale C fails on your system, use:
# psql -h localhost -U postgres -c "CREATE DATABASE faers_db OWNER nct TEMPLATE template0 ENCODING 'UTF8';"

# 3) Parallel restore (edit dump path)
pg_restore -h localhost -U nct -d faers_db --clean --if-exists --no-owner --no-privileges --role=nct -j 6 "/path/to/faers_db_YYYYMMDD.dump"

# 4) Sanity check
psql -h localhost -U nct -d faers_db -c "\dt+"
psql -h localhost -U nct -d faers_db -c "SELECT current_user, current_database();"
```

> For non-interactive password, you can use `~/.pgpass`:
> ```bash
> echo "localhost:5432:faers_db:nct:nct" >> ~/.pgpass
> chmod 600 ~/.pgpass
> ```

---

## Sanity Checks & First Queries

Inside `psql` as `nct`:
```sql
-- Connect
\c faers_db nct

-- List tables & sizes
\dt+

-- Top heavy tables (approx rows via stats)
SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       n_live_tup AS approx_rows
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;

-- Total size of schema public
SELECT pg_size_pretty(SUM(pg_total_relation_size(c.oid))) AS public_total_size
FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
WHERE n.nspname='public' AND c.relkind='r';

-- Basic FAERS checks (adjust column names if needed)
SELECT COUNT(*) FROM demo;                  -- total rows in DEMO
SELECT COUNT(DISTINCT primaryid) FROM demo; -- number of unique cases

-- Top reported suspected drugs
SELECT UPPER(drugname) AS drug, COUNT(*) AS n
FROM drug
WHERE role_cod IN ('PS','SS')   -- Primary / Secondary suspect
GROUP BY 1
ORDER BY n DESC
LIMIT 20;

-- Top reactions (Preferred Terms)
SELECT pt, COUNT(*) AS n
FROM reac
GROUP BY pt
ORDER BY n DESC
LIMIT 20;

-- Pair (drug × reaction)
SELECT UPPER(d.drugname) AS drug, r.pt, COUNT(*) AS n
FROM drug d
JOIN reac r ON r.primaryid = d.primaryid
WHERE d.role_cod IN ('PS','SS')
GROUP BY 1,2
ORDER BY n DESC
LIMIT 20;

-- Yearly trend (event_dt stored as YYYYMMDD text)
SELECT SUBSTRING(event_dt,1,4) AS year, COUNT(*) AS n
FROM demo
GROUP BY 1
ORDER BY 1;
```

> **Tip:** Run `VACUUM (ANALYZE, VERBOSE);` once after restore to refresh statistics.

---

## Optional Indexes

These speed up common lookups (id joins, case-insensitive search):
```sql
-- IDs used for joins
CREATE INDEX IF NOT EXISTS idx_drug_primaryid ON drug(primaryid);
CREATE INDEX IF NOT EXISTS idx_reac_primaryid ON reac(primaryid);
CREATE INDEX IF NOT EXISTS idx_ther_primaryid ON ther(primaryid);

-- Case-insensitive lookups
CREATE INDEX IF NOT EXISTS idx_drug_drugname_lower ON drug (LOWER(drugname));
CREATE INDEX IF NOT EXISTS idx_reac_pt_lower       ON reac (LOWER(pt));
```

> Build indexes during off-hours; they can take time on large tables.

---

## Performance Tips (Optional)

- **Parallel restore:** `pg_restore -j N` where `N` ≈ number of CPU cores (4–12 typical).
- **WAL/maintenance settings** (superuser only):
  ```sql
  ALTER SYSTEM SET maintenance_work_mem='1GB';
  ALTER SYSTEM SET max_wal_size='8GB';
  ALTER SYSTEM SET checkpoint_timeout='30min';
  SELECT pg_reload_conf();
  ```
  Revert after restore if needed.
- **Storage:** SSD/NVMe drastically improves restore and query speed.

---

## Troubleshooting

- **`permission denied for relation ...`**  
  Use a role that owns objects (`nct`). Ensure restore used `--no-owner --role=nct`.

- **`could not create extension ...`**  
  Create required extensions as superuser (e.g., `postgres`):  
  `CREATE EXTENSION IF NOT EXISTS pgcrypto;`

- **Collation/CType errors on CREATE DATABASE**  
  Drop `LC_COLLATE/LC_CTYPE` from the `CREATE DATABASE` command and use system defaults.

- **Version mismatch warnings**  
  Prefer `pg_dump/pg_restore` from the same or newer major version than the server.

---

## Create Your Own Dump (Optional)

If you build FAERS on **Ubuntu** and want a portable dump:
```bash
pg_dump -h localhost -U rd -d faers_db -Fc -Z 9 -f faers_db_$(date +%Y%m%d).dump
```
Then restore on Windows using the steps above.

---

## License & Notes

- FAERS is a public dataset released by the U.S. FDA; always review original documentation for context and limitations of spontaneous reporting data.
- This repository follows the PharmApp style for organization and reproducibility.  
- **Repo:** https://github.com/nghiencuuthuoc/pharmapp-faers

> Issues/PRs welcome for scripts (batch/shell) and SQL helpers to streamline routine restores and analytics.
