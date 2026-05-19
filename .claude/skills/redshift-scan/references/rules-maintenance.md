# Maintenance Rules

## MT-01: Tables Need VACUUM (High Unsorted)

**Severity**: HIGH
**Category**: Maintenance
**Trigger Condition**: unsorted > 20% AND size > 100MB
**Diagnostic Queries**: E1

**Observation Template**:
> {count} tables have >20% unsorted rows. Table `{schema}.{table}` is {unsorted}% unsorted ({size_mb} MB). This forces additional block scans during queries.

**Recommendation**:
Run VACUUM SORT on affected tables. Schedule regular VACUUM operations during off-peak hours. Verify automatic vacuum is enabled and running (check E3 results).

**Remediation SQL**:
```sql
-- Sort only (faster, no delete reclaim)
VACUUM SORT ONLY {schema}.{table};

-- Full vacuum (sort + reclaim deleted rows)
VACUUM FULL {schema}.{table};

-- With BOOST for faster execution (uses all resources)
VACUUM FULL {schema}.{table} BOOST;
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/t_Reclaiming_storage_space202.html
- https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html
- Section: "VACUUM frequency"

---

## MT-02: Stale Table Statistics

**Severity**: MEDIUM
**Category**: Maintenance
**Trigger Condition**: stats_off > 20 for tables > 50MB
**Diagnostic Queries**: E4

**Observation Template**:
> {count} tables have statistics that are >20% stale. Table `{schema}.{table}` has stats_off={stats_off}. The query planner may generate suboptimal plans.

**Recommendation**:
Run ANALYZE on affected tables. Verify automatic ANALYZE is enabled. For tables with frequent DML, schedule ANALYZE after major data changes.

**Remediation SQL**:
```sql
ANALYZE {schema}.{table};

-- Or analyze specific columns
ANALYZE {schema}.{table}({column1}, {column2});

-- Analyze all tables in schema
ANALYZE {schema};
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/t_Analyzing_tables.html
- https://docs.aws.amazon.com/redshift/latest/dg/c_autonomics.html
- Section: "Automatic analyze"

---

## MT-03: High Commit Queue Latency

**Severity**: HIGH
**Category**: Maintenance
**Trigger Condition**: avg_queue_ms > 1000 (average commit queue wait > 1 second)
**Diagnostic Queries**: E5

**Observation Template**:
> Average commit queue latency is {avg_queue_ms}ms (max: {max_queue_ms}ms). High commit queue times indicate write contention from too many concurrent write operations.

**Recommendation**:
Reduce the frequency of COMMIT operations:
- Batch small writes into larger transactions
- Avoid single-row INSERTs; use COPY for bulk loads
- Reduce the number of concurrent write sessions
- Schedule ETL jobs to avoid overlapping write windows

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-use-copy.html

---

## MT-04: Automatic Optimization Not Running

**Severity**: MEDIUM
**Category**: Maintenance
**Trigger Condition**: No entries in svl_auto_worker_action in the last 14 days
**Diagnostic Queries**: E3

**Observation Template**:
> No automatic optimization actions (vacuum, analyze, sort key, dist key) have run in the past 14 days. The cluster may not have enough idle time for background maintenance.

**Recommendation**:
Automatic maintenance runs during low-traffic periods. If the cluster is consistently busy:
1. Consider allocating extra compute resources for autonomics
2. Schedule manual VACUUM/ANALYZE during known quiet windows
3. Review if workload can be shifted to create maintenance windows

**Remediation SQL**:
```sql
-- Check if automatic table optimization is enabled
SELECT relname, reldiststyle, releffectivediststyle
FROM pg_class_info
WHERE relnamespace NOT IN (SELECT oid FROM pg_namespace WHERE nspname IN ('pg_catalog', 'information_schema'));
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_autonomics.html
- https://docs.aws.amazon.com/redshift/latest/dg/t_extra-compute-autonomics.html
- Section: "Allocating extra compute resources for automatic database optimization"

---

## MT-05: Ghost Row Accumulation

**Severity**: MEDIUM
**Category**: Maintenance
**Trigger Condition**: > 20% difference between live rows and sorted rows (indicating many deleted/ghost rows)
**Diagnostic Queries**: E2

**Observation Template**:
> Table `{tablename}` has significant ghost row accumulation ({pct_unsorted}% discrepancy). Deleted rows still occupy disk space and are scanned during queries.

**Recommendation**:
Run VACUUM DELETE on affected tables to reclaim space from deleted rows. If the table has both unsorted data and ghost rows, use VACUUM FULL.

**Remediation SQL**:
```sql
-- Reclaim space from deleted rows only
VACUUM DELETE ONLY {schema}.{tablename};

-- Full vacuum (sort + delete reclaim)
VACUUM FULL {schema}.{tablename};
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/t_Reclaiming_storage_space202.html
- Section: "Automatic vacuum delete"
- https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html

---

## MT-06: Disk Space Critical

**Severity**: CRITICAL
**Category**: Maintenance
**Trigger Condition**: pct_used > 85% on any node
**Diagnostic Queries**: A2

**Observation Template**:
> Node {node} disk usage is at {pct_used}% ({used_mb}MB / {capacity_mb}MB). Critical disk usage can cause query failures and prevent maintenance operations.

**Recommendation**:
Immediate actions:
1. Run VACUUM DELETE to reclaim ghost rows
2. Drop unnecessary tables or old data
3. Archive cold data to S3 (Redshift Spectrum or UNLOAD)
4. Consider resizing the cluster (elastic resize to add nodes)

For RA3 nodes, data is stored in managed storage which auto-scales, but excessive usage still impacts performance.

**Remediation SQL**:
```sql
-- Find largest tables
SELECT schema, "table", size AS size_mb
FROM svv_table_info ORDER BY size DESC LIMIT 20;

-- Unload cold data to S3
UNLOAD ('SELECT * FROM {schema}.{table} WHERE date < ''2023-01-01''')
TO 's3://bucket/archive/{table}/'
IAM_ROLE 'arn:aws:iam::account:role/RedshiftUnloadRole'
PARQUET;
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html
- https://docs.aws.amazon.com/redshift/latest/mgmt/resizing-cluster.html
