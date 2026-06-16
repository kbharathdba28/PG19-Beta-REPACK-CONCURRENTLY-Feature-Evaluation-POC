# PG19-Beta-REPACK-CONCURRENTLY-Feature-Evaluation-POC
PostgreSQL 19 Beta – REPACK (CONCURRENTLY) Feature Evaluation and Bloat Removal Analysis
1. Executive Summary
PostgreSQL 19 introduces a built-in REPACK command with CONCURRENTLY support, providing an online method to reclaim table bloat without requiring third-party tools such as pg_repack. 
This Proof of Concept (PoC) evaluates the PostgreSQL 19 Beta REPACK (CONCURRENTLY) feature and compares its effectiveness against VACUUM ANALYZE for reclaiming table and index bloat.
The objective was to:
•	Install PostgreSQL 19 Beta from source.
•	Create a large test table.
•	Generate significant table bloat.
•	Compare the impact of VACUUM ANALYZE and REPACK (CONCURRENTLY).
•	Validate that REPACK allows concurrent application activity during execution.
The test demonstrates that VACUUM ANALYZE removes dead tuples and updates statistics but does not reclaim disk space, whereas REPACK (CONCURRENTLY) successfully removes both table and index bloat while maintaining application availability.
________________________________________
</>
2. Environment Details
Component	Value
PostgreSQL Version	PostgreSQL 19 Beta 1
Operating System	Amazon Linux
Installation Method	Source Compilation
Data Directory	/pgdata/19beta1
Database Port	5433
Test Database	poc
Test Table	sales
________________________________________
3. PostgreSQL 19 Beta Installation
Download Source Package
wget https://ftp.postgresql.org/pub/source/v19beta1/postgresql-19beta1.tar.gz
tar -xvf postgresql-19beta1.tar.gz
Install Build Dependencies
dnf install gcc
dnf install -y libicu-devel pkgconf-pkg-config
dnf install bison flex
Configure PostgreSQL
./configure --without-readline --without-zlib
make
make install
________________________________________
4. Cluster Initialization
Create Data Directory
mkdir -p /pgdata/19beta1
chown postgres:postgres /pgdata/19beta1
Initialize Database Cluster
su - postgres
/usr/local/pgsql/bin/initdb -D /pgdata/19beta1
Initialization completed successfully with:
•	UTF8 Encoding
•	Locale: C.UTF-8
•	Checksums Enabled
________________________________________
5. PostgreSQL Startup
Modify postgresql.conf:
port = 5433
Start PostgreSQL:
/usr/local/pgsql/bin/pg_ctl -D /pgdata/19beta1 -l logfile start
Connect to PostgreSQL:
/usr/local/pgsql/bin/psql -h localhost -p 5433 -d postgres
Create test database:
CREATE DATABASE poc;
________________________________________
6. Test Table Creation
CREATE TABLE sales (
    id BIGSERIAL PRIMARY KEY,
    customer_id INT,
    amount NUMERIC(10,2),
    created_at TIMESTAMP
);
________________________________________
7. Populate Test Data
Insert 30 million rows:
INSERT INTO sales(customer_id, amount, created_at)
SELECT
    gs % 100000,
    round((random()*1000)::numeric,2),
    now()
FROM generate_series(1,30000000) gs;
Update statistics:
ANALYZE sales;
________________________________________
8. Initial Storage Utilization
SELECT
    pg_size_pretty(pg_relation_size('sales'))       AS table_size,
    pg_size_pretty(pg_indexes_size('sales'))        AS index_size,
    pg_size_pretty(pg_total_relation_size('sales')) AS total_size;
Before generating bloat:
Metric	Size
Table Size	1723 MB
Index Size	643 MB
Total Size	2366 MB
The table occupied approximately 2.3 GB.
________________________________________
9. Creating Table Bloat
Delete 16 million rows:
DELETE
FROM sales
WHERE id <= 16000000;
This removed more than 50% of the table data.
________________________________________
10. Storage After Delete Operation(Before VACUUM)
Metric	Size
Table Size	1723 MB
Index Size	643 MB
Total Size	2366 MB
Observation
Despite deleting over half of the rows, the physical table and index sizes remained unchanged.
This occurred because PostgreSQL marks rows as dead tuples but does not automatically shrink the table files.
________________________________________
11. VACUUM ANALYZE Test
Execute:
VACUUM ANALYZE sales;
________________________________________
12. Storage After VACUUM ANALYZE
Metric	Size
Table Size	1723 MB
Index Size	643 MB
Total Size	2366 MB
Observation
VACUUM ANALYZE:
•	Removed dead tuples from PostgreSQL visibility.
•	Updated optimizer statistics.
•	Made free space reusable for future operations.
However:
•	No disk space was returned to the operating system.
•	Table size remained unchanged.
•	Index size remained unchanged.
________________________________________
13. REPACK (CONCURRENTLY) Test
Execute:
REPACK (CONCURRENTLY) sales;
REPACK reorganizes the table and indexes online while allowing user activity.
________________________________________
14. Storage After REPACK (CONCURRENTLY)
Metric	Before Repack	After Repack
Table Size	1723 MB	862 MB
Index Size	643 MB	321 MB
Total Size	2366 MB	1183 MB
________________________________________
15. Storage Savings Achieved
Component	Before	After	Reduction
Table Size	1723 MB	862 MB	50%
Index Size	643 MB	321 MB	50%
Total Size	2366 MB	1183 MB	50%
Approximately 1.18 GB of storage was reclaimed.
________________________________________
16. Concurrent Activity Validation
During REPACK execution, four separate terminal sessions were used to validate concurrent activity.

 
Session 1 – REPACK Execution
Purpose:
•	Start REPACK
Result:
•	Repack is in progress.
________________________________________
Session 2 – Concurrent INSERT Activity
Purpose:
•	Simulate application writes.
•	Verify INSERT operations continue during REPACK.
Observation:
•	INSERT operations completed successfully.
•	No blocking observed.
Result:
REPACK supported concurrent write activity.
________________________________________
Session 3 – Concurrent SELECT Activity
Purpose:
•	Simulate application read workload.
•	Verify user access remains available.
Observation:
•	SELECT queries continued successfully.
•	No interruptions occurred.
Result:
REPACK supported concurrent read activity.
________________________________________
Session 4 – Lock Monitoring
Purpose:
Monitor lock behavior during REPACK.
Sample Query:
SELECT
    pid,
    state,
    mode,
    granted
FROM pg_locks pl
JOIN pg_stat_activity psa
ON pl.pid = psa.pid;
Observed Locks:
No long-running
•	AccessShareLock
•	RowExclusiveLock
•	ExclusiveLock
Observation:
REPACK primarily used lightweight locks and only required a brief ACCESS EXCLUSIVE lock during the final table swap.
Result:
Minimal impact on concurrent database activity.
________________________________________17. Feature Comparison
________________________________________
18. Conclusion
The PostgreSQL 19 Beta REPACK (CONCURRENTLY) feature successfully demonstrated online table and index reorganization with minimal application impact.
Key findings:
•	VACUUM ANALYZE removes dead tuples but does not reduce physical storage.
•	REPACK (CONCURRENTLY) reclaimed approximately 50% of consumed storage.
•	Both table and index bloat were eliminated.
•	Concurrent INSERT and SELECT operations continued successfully.
•	Lock contention was minimal.
•	Application availability was maintained throughout the maintenance activity.
Based on the results of this PoC, REPACK (CONCURRENTLY) is a highly effective option for reclaiming space and removing bloat in large production PostgreSQL environments where downtime must be minimized.
