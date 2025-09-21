# PharmApp FAERS — Setup & Data Restore Guide (Ubuntu & Windows 11)

This README walks you through setting up a **conda environment**, initializing **PostgreSQL**, and restoring the **FAERS** database from a prebuilt dump. It covers both **Ubuntu** and **Windows 11**.

> Tested with PostgreSQL **17.x** and Python **3.11**.  
> Recommended conda environment name: **`pharmapp_3`** (you can keep using your historical `py311_rdkit_psql` if you prefer).

---

## 0) Repositories & Data
- FAERS project repo: **https://github.com/nghiencuuthuoc/pharmapp-faers**
- Reference (ChEMBL): **https://github.com/nghiencuuthuoc/pharmapp-chembl-35**
- FAERS dump (Google Drive): **https://drive.google.com/drive/folders/1x8U87NpS3uRuxpHBRsxZa7LvhdlGeTCC?usp=sharing**

> Tip: put the downloaded dump somewhere easy to find, e.g.  
> - Ubuntu: `/mnt/hgfs/E/PharmAppDev/database/FAERS/faers_db_YYYYMMDD.dump`  
> - Windows: `E:\PharmAppDev\database\FAERS\faers_db_YYYYMMDD.dump`

---

## 1) Create & Activate the Conda Environment

### 1.1 Quick Start (Recommended)
```bash
# Create a fresh conda env named 'pharmapp_3' with Python 3.11
conda create -n pharmapp_3 python=3.11 -y
conda activate pharmapp_3

# Core scientific stack + PostgreSQL client/binaries
conda install -c conda-forge rdkit rdkit-postgresql -y
conda install -c conda-forge matplotlib postgresql -y

# Platform-specific (choose one block)
# Ubuntu:
conda install -c conda-forge gxx_linux-64 -y
# Windows 11:
conda install -c conda-forge vs2015_runtime -y

# Optional build & imaging deps often useful in PharmApp modules
conda install -c conda-forge -c anaconda cmake cairo pillow eigen pkg-config -y
conda install -c conda-forge -c anaconda boost-cpp boost py-boost -y
```

### 1.2 Historical recipe (as of 2025-09-20)
```bash
conda create -n py311_rdkit_psql python=3.11 -y
conda activate py311_rdkit_psql
conda install -c conda-forge rdkit rdkit-postgresql -y
conda install -c conda-forge matplotlib gxx_linux-64 postgresql -y  # ubuntu
conda install -c conda-forge matplotlib vs2015_runtime postgresql -y # windows 11
conda install -c conda-forge -c anaconda cmake cairo pillow eigen pkg-config -y
conda install -c conda-forge -c anaconda boost-cpp boost py-boost -y
```

> **Note**: On Windows, you may also have the official PostgreSQL installed system-wide. Either is fine as long as `psql`, `pg_restore`, and `pg_ctl` are on `PATH`.

---

## 2) Initialize PostgreSQL Cluster

> You only need to initialize once per machine (per data directory).

### Ubuntu
```bash
# Example data directory
initdb /home/rd/postgresdb
pg_ctl -D /home/rd/postgresdb -l /home/rd/postgresdb/logfile start
```

### Windows 11 (CMD)
```bat
initdb -D "E:\PharmAppDev\pgdata"
pg_ctl -D ^"E^:^\PharmAppDev^\pgdata^" -l logfile start
pg_ctl -D ^"E^:^\PharmAppDev^\pgdata^" -l logfile restart
```

> If the server is already running as a Windows service or via another data directory, adjust paths/ports accordingly.

---

## 3) Create Role & Database, then Restore FAERS

We will create a role **`nct`** (password `nct`) and a database **`faers_db`** owned by that role.

### 3.1 Create role & database (run as `postgres`)
**Ubuntu / Windows (psql):**
```bash
psql -h localhost -U postgres -c "DO $$ BEGIN IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='nct') THEN CREATE ROLE nct LOGIN PASSWORD 'nct'; ALTER ROLE nct CREATEDB; END IF; END $$;"
psql -h localhost -U postgres -c "DROP DATABASE IF EXISTS faers_db;"
psql -h localhost -U postgres -c "CREATE DATABASE faers_db OWNER nct TEMPLATE template0 ENCODING 'UTF8';"
```

> If you require specific collations on Windows and see errors, recreate the database without LC settings (as above) or with appropriate `LC_COLLATE/LC_CTYPE` available on your system.

### 3.2 Restore from the `.dump` (parallel)
```bash
# Replace the path to your dump file accordingly
pg_restore -h localhost -U nct -d faers_db \
  --clean --if-exists --no-owner --no-privileges --role=nct \
  -j 6 "/path/to/faers_db_YYYYMMDD.dump"
```

- `-j 6` uses 6 parallel workers; tune based on your CPU/SSD (e.g., 4–12).
- `--no-owner --role=nct` ensures restored objects belong to `nct`.
- `--clean --if-exists` drops existing objects before recreating them.

> On Windows CMD, remember to quote Windows paths:  
> `pg_restore -h localhost -U nct -d faers_db -j 6 "E:\PharmAppDev\database\FAERS\faers_db_20250921.dump"`

---

## 4) Verify the Restore

```bash
# List tables
psql -h localhost -U nct -d faers_db -c "\dt+"

# Show current user & DB
psql -h localhost -U nct -d faers_db -c "SELECT current_user, current_database();"

# Optional: sizes & rough row counts
psql -h localhost -U nct -d faers_db -c \
"SELECT relname AS table_name, pg_size_pretty(pg_total_relation_size(relid)) AS total_size, n_live_tup AS approx_rows
 FROM pg_stat_user_tables ORDER BY total_size DESC LIMIT 20;"
```

### (Optional) Analyze for better planner stats
```bash
psql -h localhost -U nct -d faers_db -c "VACUUM (ANALYZE);"
```

---

## 5) Useful Indexes & Starter Queries

### 5.1 Indexes (create only once)
```sql
-- On linking IDs
CREATE INDEX IF NOT EXISTS idx_drug_primaryid ON drug(primaryid);
CREATE INDEX IF NOT EXISTS idx_reac_primaryid ON reac(primaryid);
CREATE INDEX IF NOT EXISTS idx_ther_primaryid ON ther(primaryid);

-- Case-insensitive searches
CREATE INDEX IF NOT EXISTS idx_drug_drugname_lower ON drug (LOWER(drugname));
CREATE INDEX IF NOT EXISTS idx_reac_pt_lower      ON reac (LOWER(pt));
```

### 5.2 Quick sanity queries
```sql
-- Top 20 suspect drugs
SELECT UPPER(drugname) AS drug, COUNT(*) AS n
FROM drug
WHERE role_cod IN ('PS','SS')
GROUP BY 1 ORDER BY n DESC LIMIT 20;

-- Top 20 reactions (PT)
SELECT pt, COUNT(*) AS n
FROM reac
GROUP BY pt ORDER BY n DESC LIMIT 20;

-- Frequent (drug × reaction) pairs
SELECT UPPER(d.drugname) AS drug, r.pt, COUNT(*) AS n
FROM drug d
JOIN reac r ON r.primaryid = d.primaryid
WHERE d.role_cod IN ('PS','SS')
GROUP BY 1,2 ORDER BY n DESC LIMIT 20;

-- Trend by year (assuming event_dt as 'YYYYMMDD')
SELECT SUBSTRING(event_dt,1,4) AS year, COUNT(*) AS n
FROM demo
GROUP BY 1 ORDER BY 1;
```

---

## 6) Tips & Troubleshooting

- **Versions**: Prefer `pg_dump` from the **same version** as the source server. `pg_restore` may be same or slightly newer.
- **Extensions**: If the dump expects extensions (e.g., `pgcrypto`, `"uuid-ossp"`), create them before or after restore:
  ```sql
  CREATE EXTENSION IF NOT EXISTS pgcrypto;
  CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
  ```
- **Authentication**:
  - Ubuntu: you can set `~/.pgpass` to avoid typing passwords (`chmod 600 ~/.pgpass`).
  - Windows: use `SET PGPASSWORD=...` before commands (or rely on interactive prompts).
- **Performance**:
  - Use `-j` with `pg_restore` for parallelism.
  - Ensure your data directory is on a fast SSD.
  - Consider increasing `maintenance_work_mem`, `max_wal_size`, and `checkpoint_timeout` temporarily for large restores.

---

## 7) Contributing / Structure

- File issues and PRs at **pharmapp-faers**. Keep scripts cross-platform when possible.
- Prefer environment name `pharmapp_3` across modules for consistency.

---

## 8) License

See the repository’s LICENSE file.

---

**Happy analyzing!**  
If you hit a snag, capture the full error output and open an issue on the repo.
