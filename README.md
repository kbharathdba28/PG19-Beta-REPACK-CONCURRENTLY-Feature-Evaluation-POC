# PG19-Beta-REPACK-CONCURRENTLY-Feature-Evaluation-POC
# PostgreSQL 19 Beta – REPACK (CONCURRENTLY) Feature Evaluation and Bloat Removal Analysis

## Executive Summary

PostgreSQL 19 introduces a built-in `REPACK` command with `CONCURRENTLY` support, providing an online method to reclaim table bloat without requiring third-party tools such as `pg_repack`.

This Proof of Concept (PoC) evaluates the PostgreSQL 19 Beta `REPACK (CONCURRENTLY)` feature and compares its effectiveness against `VACUUM ANALYZE` for reclaiming table and index bloat.

### Objectives

* Install PostgreSQL 19 Beta from source.
* Create a large test table.
* Generate significant table bloat.
* Compare the impact of `VACUUM ANALYZE` and `REPACK (CONCURRENTLY)`.
* Validate that REPACK allows concurrent application activity during execution.

### Key Finding

`VACUUM ANALYZE` removes dead tuples and updates optimizer statistics but does not reclaim disk space. `REPACK (CONCURRENTLY)` successfully removes both table and index bloat while maintaining application availability.

---

## Environment Details

| Component           | Value                |
| ------------------- | -------------------- |
| PostgreSQL Version  | PostgreSQL 19 Beta 1 |
| Operating System    | Amazon Linux         |
| Installation Method | Source Compilation   |
| Data Directory      | /pgdata/19beta1      |
| Database Port       | 5433                 |
| Test Database       | poc                  |
| Test Table          | sales                |

---

## PostgreSQL 19 Beta Installation

### Download Source Package

```bash
wget https://ftp.postgresql.org/pub/source/v19beta1/postgresql-19beta1.tar.gz
tar -xvf postgresql-19beta1.tar.gz
```

### Install Build Dependencies

```bash
dnf install gcc
dnf install -y libicu-devel pkgconf-pkg-config
dnf install bison flex
```

### Configure and Build PostgreSQL

```bash
./configure --without-readline --without-zlib
make
make install
```

---

## Cluster Initialization

### Create Data Directory

```bash
mkdir -p /pgdata/19beta1
chown postgres:postgres /pgdata/19beta1
```

### Initialize Database Cluster

```bash
su - postgres
/usr/local/pgsql/bin/initdb -D /pgdata/19beta1
```

Initialization completed successfully with:

* UTF8 Encoding
* Locale: C.UTF-8
* Checksums Enabled

---

## PostgreSQL Startup

Modify `postgresql.conf`:

```conf
port = 5433
```

Start PostgreSQL:

```bash
/usr/local/pgsql/bin/pg_ctl -D /pgdata/19beta1 -l logfile start
```

Connect:

```bash
/usr/local/pgsql/bin/psql -h localhost -p 5433 -d postgres
```

Create database:

```sql
CREATE DATABASE poc;
```

---

## Test Table Creation

```sql
CREATE TABLE sales (
    id BIGSERIAL PRIMARY KEY,
    customer_id INT,
    amount NUMERIC(10,2),
    created_at TIMESTAMP
);
```

---

## Populate Test Data

Insert 30 million rows:

```sql
INSERT INTO sales(customer_id, amount, created_at)
SELECT
    gs % 100000,
    round((random()*1000)::numeric,2),
    now()
FROM generate_series(1,30000000) gs;
```

Update statistics:

```sql
ANALYZE sales;
```

---

## Initial Storage Utilization

| Metric     | Size    |
| ---------- | ------- |
| Table Size | 1723 MB |
| Index Size | 643 MB  |
| Total Size | 2366 MB |

The table occupied approximately 2.3 GB.

---

## Creating Table Bloat

Delete 16 million rows:

```sql
DELETE
FROM sales
WHERE id <= 16000000;
```

More than 50% of the table data was removed.

---

## Storage After Delete Operation (Before VACUUM)

| Metric     | Size    |
| ---------- | ------- |
| Table Size | 1723 MB |
| Index Size | 643 MB  |
| Total Size | 2366 MB |

### Observation

Although more than half of the rows were deleted, the physical table and index sizes remained unchanged because PostgreSQL marks deleted rows as dead tuples and does not automatically shrink table files.

---

## VACUUM ANALYZE Test

```sql
VACUUM ANALYZE sales;
```

### Storage After VACUUM ANALYZE

| Metric     | Size    |
| ---------- | ------- |
| Table Size | 1723 MB |
| Index Size | 643 MB  |
| Total Size | 2366 MB |

### Observation

VACUUM ANALYZE:

* Removed dead tuples from PostgreSQL visibility.
* Updated optimizer statistics.
* Made free space reusable for future operations.

However:

* No disk space was returned to the operating system.
* Table size remained unchanged.
* Index size remained unchanged.

---

## REPACK (CONCURRENTLY) Test

```sql
REPACK (CONCURRENTLY) sales;
```

REPACK reorganizes the table and indexes online while allowing concurrent user activity.

### Storage After REPACK (CONCURRENTLY)

| Metric     | Before Repack | After Repack |
| ---------- | ------------- | ------------ |
| Table Size | 1723 MB       | 862 MB       |
| Index Size | 643 MB        | 321 MB       |
| Total Size | 2366 MB       | 1183 MB      |

---

## Storage Savings Achieved

| Component  | Before  | After   | Reduction |
| ---------- | ------- | ------- | --------- |
| Table Size | 1723 MB | 862 MB  | 50%       |
| Index Size | 643 MB  | 321 MB  | 50%       |
| Total Size | 2366 MB | 1183 MB | 50%       |

Approximately **1.18 GB** of storage was reclaimed.

---

## Concurrent Activity Validation

Four separate terminal sessions were used to validate concurrent activity during REPACK execution.

| Session   | Activity                   | Result                                      |
| --------- | -------------------------- | ------------------------------------------- |
| Session 1 | REPACK Execution           | Repack executed successfully                |
| Session 2 | Concurrent INSERT Activity | Write operations continued without blocking |
| Session 3 | Concurrent SELECT Activity | Read operations continued successfully      |
| Session 4 | Lock Monitoring            | Only lightweight locks observed             |

### Lock Monitoring Query

```sql
SELECT
    pid,
    state,
    mode,
    granted
FROM pg_locks pl
JOIN pg_stat_activity psa
ON pl.pid = psa.pid;
```

Observed lock types:

* AccessShareLock
* RowExclusiveLock
* ExclusiveLock

REPACK required only a brief ACCESS EXCLUSIVE lock during the final table swap phase.

---

## Feature Comparison

| Feature                         | VACUUM ANALYZE | VACUUM FULL | REPACK (CONCURRENTLY) |
| ------------------------------- | -------------- | ----------- | --------------------- |
| Removes Table Bloat             | Partially      | Yes         | Yes                   |
| Removes Index Bloat             | No             | Yes         | Yes                   |
| Updates Statistics              | Yes            | No          | No                    |
| Table Rewrite                   | No             | Yes         | Yes                   |
| DML Allowed During Operation    | Yes            | No          | Yes                   |
| SELECT Allowed During Operation | Yes            | No          | Yes                   |
| Reclaims OS Disk Space          | No             | Yes         | Yes                   |
| Concurrent Execution            | Yes            | No          | Yes                   |
| Suitable for 24x7 Systems       | Yes            | No          | Yes                   |
| Risk of Blocking Applications   | Very Low       | Very High   | Very Low              |

---

## Conclusion

The PostgreSQL 19 Beta `REPACK (CONCURRENTLY)` feature successfully demonstrated online table and index reorganization with minimal application impact.

### Key Findings

* VACUUM ANALYZE removes dead tuples but does not reduce physical storage.
* REPACK (CONCURRENTLY) reclaimed approximately 50% of consumed storage.
* Both table and index bloat were eliminated.
* Concurrent INSERT and SELECT operations continued successfully.
* Lock contention was minimal.
* Application availability was maintained throughout the maintenance activity.

Based on the results of this PoC, `REPACK (CONCURRENTLY)` is a highly effective option for reclaiming space and removing bloat in large PostgreSQL environments where downtime must be minimized.
