# Diagnostic Queries

All queries are SELECT-only and safe for production execution. Each includes LIMIT clauses to manage result size.

## Sources

- Queries A1-G1: Original diagnostic queries based on AWS Redshift documentation
- Queries H1-H10: Adapted from [awslabs/amazon-redshift-utils](https://github.com/awslabs/amazon-redshift-utils) (Apache 2.0 License)
  - `src/AdminScripts/perf_alert.sql`
  - `src/AdminScripts/table_inspector.sql`
  - `src/AdminScripts/top_queries.sql`
  - `src/AdminScripts/copy_performance.sql`
  - `src/AdminScripts/missing_table_stats.sql`
  - `src/AdminScripts/filter_used.sql`
  - `src/AdminScripts/predicate_columns.sql`
  - `src/AdminScripts/unscanned_table_summary.sql`
  - `src/AdminScripts/wlm_qmr_rule_candidates.sql`
  - `src/AdminScripts/insert_into_table_dk_mismatch.sql`
  - `src/AdminViews/v_check_data_distribution.sql`
  - `src/AdminViews/v_fragmentation_info.sql`
  - `src/AdminViews/v_get_tbl_scan_frequency.sql`
  - `src/AdminViews/v_extended_table_info.sql`

---

## Category A: Cluster Configuration

### A1 - Slice Topology
```sql
SELECT node, slice, type
FROM stv_slices
ORDER BY node, slice;
```

### A2 - Disk Space Utilization
```sql
SELECT owner AS node,
       used AS used_mb,
       capacity AS capacity_mb,
       ROUND(used::float / capacity * 100, 2) AS pct_used
FROM stv_node_storage_capacity
ORDER BY node;
```

### A3 - Key Parameter Settings
```sql
SELECT name, setting, unit, short_desc
FROM pg_settings
WHERE name IN (
  'max_concurrency_scaling_clusters',
  'wlm_json_configuration',
  'statement_timeout',
  'max_cursor_result_set_size',
  'enable_result_cache_for_session',
  'query_group',
  'search_path'
)
ORDER BY name;
```

### A4 - Cluster Version and Uptime
```sql
SELECT version() AS cluster_version;
```

---

## Category B: Table Design

### B1 - Table Storage and Distribution Overview
```sql
SELECT database, schema, "table", diststyle, sortkey1,
       max_varchar, sortkey1_enc, sortkey_num,
       size AS size_mb, pct_used, unsorted, stats_off,
       tbl_rows, skew_sortkey1, skew_rows
FROM svv_table_info
WHERE schema NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
ORDER BY size DESC
LIMIT 100;
```

### B2 - Tables Missing Sort Keys (large tables)
```sql
SELECT database, schema, "table", size, tbl_rows
FROM svv_table_info
WHERE sortkey1 IS NULL
  AND size > 100
  AND schema NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
ORDER BY size DESC;
```

### B3 - Tables with High Unsorted Percentage
```sql
SELECT database, schema, "table", size, unsorted, tbl_rows
FROM svv_table_info
WHERE unsorted > 20
  AND size > 50
  AND schema NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
ORDER BY unsorted DESC;
```

### B4 - Column Encoding Analysis
```sql
SELECT schemaname, tablename, "column", type, encoding, distkey, sortkey
FROM pg_table_def
WHERE schemaname NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
ORDER BY tablename, sortkey
LIMIT 500;
```

### B5 - Tables with Uncompressed Non-Sortkey Columns
```sql
SELECT tablename, COUNT(*) AS raw_columns
FROM pg_table_def
WHERE schemaname NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
  AND encoding = 'none'
  AND sortkey != 1
GROUP BY tablename
HAVING COUNT(*) > 3
ORDER BY raw_columns DESC
LIMIT 50;
```

### B6 - Distribution Skew Detection
```sql
SELECT schema, "table", skew_rows, diststyle, size AS size_mb
FROM svv_table_info
WHERE skew_rows > 4.0
  AND size > 100
  AND schema NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
ORDER BY skew_rows DESC;
```

### B7 - Automatic Table Optimization Recommendations
```sql
SELECT database, schema, "table", type, ranking, benefit, command
FROM svv_alter_table_recommendations
ORDER BY benefit DESC
LIMIT 50;
```

### B8 - VARCHAR Over-Allocation Detection
```sql
SELECT schemaname, tablename, "column", type,
       CAST(SUBSTRING(type FROM '\\(([0-9]+)\\)') AS INT) AS declared_size
FROM pg_table_def
WHERE schemaname NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
  AND type LIKE 'character varying%'
  AND CAST(SUBSTRING(type FROM '\\(([0-9]+)\\)') AS INT) > 256
ORDER BY declared_size DESC
LIMIT 50;
```

---

## Category C: Query Performance

### C1 - Top Slow Queries (last 7 days)
```sql
SELECT query, userid, elapsed/1000000.0 AS elapsed_sec,
       SUBSTRING(querytxt, 1, 100) AS query_text,
       starttime
FROM stl_query
WHERE starttime > DATEADD(day, -7, GETDATE())
  AND userid > 1
  AND elapsed > 60000000
ORDER BY elapsed DESC
LIMIT 25;
```

### C2 - Query Alert Events Summary
```sql
SELECT event, solution, COUNT(*) AS occurrence_count,
       MIN(event_time) AS first_seen, MAX(event_time) AS last_seen
FROM stl_alert_event_log
WHERE event_time > DATEADD(day, -7, GETDATE())
GROUP BY event, solution
ORDER BY occurrence_count DESC
LIMIT 30;
```

### C3 - Nested Loop Join Alerts
```sql
SELECT query, COUNT(*) AS nested_loop_count
FROM stl_alert_event_log
WHERE event ILIKE '%nested loop%'
  AND event_time > DATEADD(day, -7, GETDATE())
GROUP BY query
ORDER BY nested_loop_count DESC
LIMIT 20;
```

### C4 - Disk-Based Query Operations
```sql
SELECT query, segment, step, label,
       rows, bytes, workmem, is_diskbased
FROM svl_query_summary
WHERE is_diskbased = 't'
  AND query IN (
    SELECT query FROM stl_query
    WHERE starttime > DATEADD(day, -3, GETDATE())
      AND userid > 1
  )
ORDER BY bytes DESC
LIMIT 30;
```

### C5 - Query Execution Skew
```sql
SELECT query, segment, step,
       MAX(rows) AS max_rows, MIN(rows) AS min_rows,
       CASE WHEN MAX(rows) > 0
            THEN ROUND(1.0 * (MAX(rows) - AVG(rows)) / MAX(rows) * 100, 2)
            ELSE 0 END AS skew_pct
FROM svl_query_report
WHERE query IN (
  SELECT query FROM stl_query
  WHERE starttime > DATEADD(day, -3, GETDATE())
    AND userid > 1
    AND elapsed > 30000000
)
GROUP BY query, segment, step
HAVING MAX(rows) > 1000
  AND (CASE WHEN MAX(rows) > 0
       THEN 1.0 * (MAX(rows) - AVG(rows)) / MAX(rows) ELSE 0 END) > 0.5
ORDER BY skew_pct DESC
LIMIT 30;
```

### C6 - Queue Wait Times
```sql
SELECT service_class, COUNT(*) AS query_count,
       AVG(total_queue_time)/1000000.0 AS avg_queue_sec,
       MAX(total_queue_time)/1000000.0 AS max_queue_sec
FROM stl_wlm_query
WHERE service_class > 4
  AND queue_start_time > DATEADD(day, -7, GETDATE())
GROUP BY service_class
ORDER BY avg_queue_sec DESC;
```

### C7 - Concurrency Scaling Usage
```sql
SELECT TRUNC(starttime) AS day,
       COUNT(*) AS queries_on_scaling,
       SUM(elapsed)/1000000.0 AS total_scaling_sec
FROM stl_query
WHERE starttime > DATEADD(day, -14, GETDATE())
  AND concurrency_scaling_status = 1
GROUP BY TRUNC(starttime)
ORDER BY day DESC;
```

---

## Category D: Workload Management

### D1 - WLM Queue Configuration
```sql
SELECT service_class, name, num_query_tasks,
       query_working_mem, max_execution_time,
       user_group_wild_card, query_group_wild_card
FROM stv_wm_service_class_config
WHERE service_class > 4
ORDER BY service_class;
```

### D2 - WLM Queue Throughput
```sql
SELECT service_class,
       COUNT(*) AS total_queries,
       AVG(total_exec_time)/1000000.0 AS avg_exec_sec,
       MAX(total_exec_time)/1000000.0 AS max_exec_sec,
       SUM(CASE WHEN aborted = 1 THEN 1 ELSE 0 END) AS aborted_queries
FROM stl_wlm_query
WHERE queue_start_time > DATEADD(day, -7, GETDATE())
  AND service_class > 4
GROUP BY service_class
ORDER BY service_class;
```

### D3 - Query Monitoring Rule Violations
```sql
SELECT rule, action, COUNT(*) AS violation_count,
       MIN(eventtime) AS first_violation,
       MAX(eventtime) AS last_violation
FROM stl_wlm_rule_action
WHERE eventtime > DATEADD(day, -7, GETDATE())
GROUP BY rule, action
ORDER BY violation_count DESC;
```

### D4 - Short Query Acceleration Metrics
```sql
SELECT TRUNC(starttime) AS day,
       COUNT(CASE WHEN final_state = 'Completed' THEN 1 END) AS sqa_completed,
       COUNT(CASE WHEN final_state = 'Evicted' THEN 1 END) AS sqa_evicted,
       AVG(CASE WHEN final_state = 'Completed' THEN total_exec_time END)/1000000.0 AS avg_sqa_sec
FROM stl_wlm_query
WHERE starttime > DATEADD(day, -7, GETDATE())
  AND service_class = 14
GROUP BY TRUNC(starttime)
ORDER BY day DESC;
```

---

## Category E: Maintenance and Health

### E1 - VACUUM Candidates
```sql
SELECT schema, "table", size, tbl_rows, unsorted,
       COALESCE(stats_off, 0) AS stats_off
FROM svv_table_info
WHERE (unsorted > 10 OR stats_off > 10)
  AND size > 50
  AND schema NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
ORDER BY unsorted DESC
LIMIT 30;
```

### E2 - Tables with Ghost/Deleted Rows
```sql
SELECT name AS tablename,
       rows AS live_rows,
       sorted_rows,
       CASE WHEN rows > 0
            THEN ROUND(100.0 * (rows - sorted_rows) / rows, 2)
            ELSE 0 END AS pct_unsorted
FROM stv_tbl_perm
WHERE rows > 100000
  AND slice = 0
ORDER BY pct_unsorted DESC
LIMIT 30;
```

### E3 - Automatic Optimization Actions (last 14 days)
```sql
SELECT type, status, tbl_name,
       TRUNC(eventtime) AS event_date,
       COUNT(*) AS action_count
FROM svl_auto_worker_action
WHERE eventtime > DATEADD(day, -14, GETDATE())
GROUP BY type, status, tbl_name, TRUNC(eventtime)
ORDER BY event_date DESC
LIMIT 50;
```

### E4 - Table Statistics Staleness
```sql
SELECT database, schema, "table", stats_off, size, tbl_rows
FROM svv_table_info
WHERE stats_off > 20
  AND size > 50
  AND schema NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
ORDER BY stats_off DESC
LIMIT 30;
```

### E5 - Commit Queue Latency
```sql
SELECT TRUNC(startqueue) AS day,
       COUNT(*) AS commits,
       AVG(DATEDIFF(ms, startqueue, startwork)) AS avg_queue_ms,
       MAX(DATEDIFF(ms, startqueue, startwork)) AS max_queue_ms
FROM stl_commit_stats
WHERE startqueue > DATEADD(day, -7, GETDATE())
GROUP BY TRUNC(startqueue)
ORDER BY day DESC;
```

---

## Category F: Data Loading

### F1 - Recent COPY Operations Performance
```sql
SELECT q.query, lc.lines_scanned,
       DATEDIFF(sec, q.starttime, q.endtime) AS duration_sec,
       SUBSTRING(lc.filename, 1, 80) AS file_path
FROM stl_load_commits lc
JOIN stl_query q ON q.query = lc.query
WHERE q.starttime > DATEADD(day, -7, GETDATE())
ORDER BY lc.lines_scanned DESC
LIMIT 30;
```

### F2 - Load Error Summary
```sql
SELECT err_reason, COUNT(*) AS error_count,
       MIN(starttime) AS first_error, MAX(starttime) AS last_error
FROM stl_load_errors
WHERE starttime > DATEADD(day, -7, GETDATE())
GROUP BY err_reason
ORDER BY error_count DESC
LIMIT 20;
```

### F3 - COPY File Count vs Slice Count
```sql
SELECT q.query,
       COUNT(DISTINCT lc.filename) AS num_files,
       (SELECT COUNT(DISTINCT slice) FROM stv_slices) AS num_slices,
       SUM(lc.lines_scanned) AS total_rows
FROM stl_load_commits lc
JOIN stl_query q ON q.query = lc.query
WHERE q.starttime > DATEADD(day, -3, GETDATE())
GROUP BY q.query
HAVING COUNT(DISTINCT lc.filename) < (SELECT COUNT(DISTINCT slice) FROM stv_slices)
ORDER BY total_rows DESC
LIMIT 20;
```

---

## Category G: Redshift Advisor

### G1 - Advisor Recommendations
```sql
SELECT feature_name, recommendation_type, ranking,
       SUBSTRING(observation, 1, 200) AS observation,
       SUBSTRING(recommendation_text, 1, 200) AS recommendation
FROM svv_redshift_advisor
ORDER BY ranking
LIMIT 30;
```

---

## Category H: Enhanced Diagnostics (from awslabs/amazon-redshift-utils)

> Source: https://github.com/awslabs/amazon-redshift-utils (Apache 2.0 License)
> These queries are adapted from the AdminScripts and AdminViews directories.

### H1 - Performance Alerts with Table Context
> Adapted from: `src/AdminScripts/perf_alert.sql`
```sql
SELECT trim(s.perm_table_name) AS "table",
       (SUM(ABS(DATEDIFF(seconds,
           COALESCE(b.starttime, d.starttime, s.starttime),
           CASE WHEN COALESCE(b.endtime, d.endtime, s.endtime) > COALESCE(b.starttime, d.starttime, s.starttime)
                THEN COALESCE(b.endtime, d.endtime, s.endtime)
                ELSE COALESCE(b.starttime, d.starttime, s.starttime) END
       )))/60)::NUMERIC(24,0) AS minutes,
       SUM(COALESCE(b.rows, d.rows, s.rows)) AS rows,
       trim(split_part(l.event, ':', 1)) AS event,
       SUBSTRING(trim(l.solution), 1, 60) AS solution,
       MAX(l.query) AS sample_query,
       COUNT(DISTINCT l.query) AS occurrence_count
FROM stl_alert_event_log AS l
LEFT JOIN stl_scan AS s ON s.query = l.query AND s.slice = l.slice AND s.segment = l.segment
LEFT JOIN stl_dist AS d ON d.query = l.query AND d.slice = l.slice AND d.segment = l.segment
LEFT JOIN stl_bcast AS b ON b.query = l.query AND b.slice = l.slice AND b.segment = l.segment
WHERE l.userid > 1
  AND l.event_time >= DATEADD(day, -7, CURRENT_DATE)
  AND s.perm_table_name NOT LIKE 'volt_tt%'
  AND s.perm_table_name NOT LIKE 'Internal Worktable'
GROUP BY 1, 4, 5
ORDER BY 2 DESC, 7 DESC
LIMIT 50;
```

### H2 - Table Skew Inspector (Block-Level Distribution)
> Adapted from: `src/AdminScripts/table_inspector.sql`
```sql
SELECT ti.schema AS schemaname,
       ti."table" AS tablename,
       ti.table_id AS tableid,
       ti.size AS size_in_mb,
       CASE WHEN ti.diststyle ILIKE 'KEY%' THEN 1 ELSE 0 END AS has_dist_key,
       CASE WHEN ti.sortkey1 IS NOT NULL THEN 1 ELSE 0 END AS has_sort_key,
       CASE WHEN ti.encoded = 'Y' THEN 1 ELSE 0 END AS has_col_encoding,
       ROUND(100 * CAST(iq.max_blocks_per_slice - iq.min_blocks_per_slice AS FLOAT)
             / GREATEST(NVL(iq.min_blocks_per_slice, 0)::INT, 1), 2) AS ratio_skew_across_slices,
       ROUND(CAST(100 * iq.dist_slice AS FLOAT)
             / (SELECT COUNT(DISTINCT slice) FROM stv_slices WHERE type = 'D'), 2) AS pct_slices_populated
FROM svv_table_info ti
JOIN (SELECT tbl,
             MIN(c) AS min_blocks_per_slice,
             MAX(c) AS max_blocks_per_slice,
             COUNT(DISTINCT slice) AS dist_slice
      FROM (SELECT b.tbl, b.slice, COUNT(*) AS c
            FROM stv_blocklist b GROUP BY b.tbl, b.slice)
      WHERE tbl IN (SELECT table_id FROM svv_table_info)
      GROUP BY tbl) iq ON iq.tbl = ti.table_id
WHERE ti.schema NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
ORDER BY ratio_skew_across_slices DESC
LIMIT 50;
```

### H3 - Top Queries with Alert Classification
> Adapted from: `src/AdminScripts/top_queries.sql`
```sql
SELECT trim(database) AS db, COUNT(query) AS n_qry,
       MAX(SUBSTRING(qrytext, 1, 80)) AS qrytext,
       MIN(run_seconds) AS "min", MAX(run_seconds) AS "max",
       AVG(run_seconds) AS "avg", SUM(run_seconds) AS total,
       MAX(query) AS max_query_id,
       MAX(starttime)::date AS last_run, aborted,
       LISTAGG(event, ', ') WITHIN GROUP (ORDER BY query) AS events
FROM (
  SELECT userid, label, stl_query.query, trim(database) AS database,
         trim(querytxt) AS qrytext, md5(trim(querytxt)) AS qry_md5,
         starttime, endtime,
         DATEDIFF(seconds, starttime, endtime)::NUMERIC(12,2) AS run_seconds,
         aborted,
         DECODE(alrt.event,
           'Very selective query filter','Filter',
           'Scanned a large number of deleted rows','Deleted',
           'Nested Loop Join in the query plan','Nested Loop',
           'Distributed a large number of rows across the network','Distributed',
           'Broadcasted a large number of rows across the network','Broadcast',
           'Missing query planner statistics','Stats',
           alrt.event) AS event
  FROM stl_query
  LEFT OUTER JOIN (
    SELECT query, trim(split_part(event, ':', 1)) AS event
    FROM stl_alert_event_log
    WHERE event_time >= DATEADD(day, -7, CURRENT_DATE)
    GROUP BY query, trim(split_part(event, ':', 1))
  ) AS alrt ON alrt.query = stl_query.query
  WHERE userid <> 1
    AND starttime >= DATEADD(day, -7, CURRENT_DATE)
)
GROUP BY database, label, qry_md5, aborted
ORDER BY total DESC
LIMIT 50;
```

### H4 - COPY Performance (S3 Transfer Throughput)
> Adapted from: `src/AdminScripts/copy_performance.sql`
```sql
SELECT q.starttime, s.query,
       SUBSTRING(q.querytxt, 1, 120) AS querytxt,
       s.n_files, s.size_mb, s.time_seconds,
       s.size_mb / DECODE(s.time_seconds, 0, 1, s.time_seconds) AS mb_per_s
FROM (SELECT query, COUNT(*) AS n_files,
             SUM(transfer_size/(1024*1024)) AS size_mb,
             (MAX(end_time) - MIN(start_time))/(1000000) AS time_seconds,
             MAX(end_time) AS end_time
      FROM stl_s3client
      WHERE http_method = 'GET' AND query > 0 AND transfer_time > 0
      GROUP BY query) AS s
LEFT JOIN stl_query AS q ON q.query = s.query
WHERE s.end_time >= DATEADD(day, -7, CURRENT_DATE)
ORDER BY s.time_seconds DESC, s.size_mb DESC
LIMIT 50;
```

### H5 - Missing Statistics in EXPLAIN Plans
> Adapted from: `src/AdminScripts/missing_table_stats.sql`
```sql
SELECT SUBSTRING(trim(plannode), 1, 100) AS plannode,
       COUNT(*) AS occurrence_count
FROM stl_explain
WHERE plannode LIKE '%missing statistics%'
  AND plannode NOT LIKE '%redshift_auto_health_check_%'
GROUP BY plannode
ORDER BY 2 DESC
LIMIT 20;
```

### H6 - Predicate Columns (Sort Key Candidates)
> Adapted from: `src/AdminScripts/predicate_columns.sql`
```sql
WITH predicate_column_info AS (
  SELECT ns.nspname AS schema_name, c.relname AS table_name,
         a.attnum AS col_num, a.attname AS col_name,
         a.attisdistkey, a.attsortkeyord,
         CASE
           WHEN 10002 = s.stakind1 THEN array_to_string(stavalues1, '||')
           WHEN 10002 = s.stakind2 THEN array_to_string(stavalues2, '||')
           WHEN 10002 = s.stakind3 THEN array_to_string(stavalues3, '||')
           WHEN 10002 = s.stakind4 THEN array_to_string(stavalues4, '||')
           ELSE NULL::VARCHAR
         END AS pred_ts
  FROM pg_statistic s
  JOIN pg_class c ON c.oid = s.starelid
  JOIN pg_namespace ns ON c.relnamespace = ns.oid
  JOIN pg_attribute a ON c.oid = a.attrelid AND a.attnum = s.staattnum
)
SELECT schema_name, table_name, col_num, col_name,
       CASE WHEN pred_ts NOT LIKE '2000-01-01%' THEN (split_part(pred_ts,'||',1))::TIMESTAMP ELSE NULL::TIMESTAMP END AS first_predicate_use,
       attisdistkey AS is_distkey, attsortkeyord AS is_sortkey
FROM predicate_column_info
WHERE pred_ts NOT LIKE '2000-01-01%'
ORDER BY schema_name, table_name, col_num
LIMIT 100;
```

### H7 - Unscanned Tables (Wasted Storage)
> Adapted from: `src/AdminScripts/unscanned_table_summary.sql`
```sql
SELECT t.database, t.schema, t."table", t.size AS size_mb, t.tbl_rows,
       NVL(s.num_qs, 0) AS num_scans
FROM svv_table_info t
LEFT JOIN (
  SELECT tbl, COUNT(DISTINCT query) AS num_qs
  FROM stl_scan
  WHERE userid > 1 AND type = 2
  GROUP BY tbl
) s ON s.tbl = t.table_id
WHERE t.schema NOT IN ('pg_catalog', 'information_schema', 'pg_internal')
  AND NVL(s.num_qs, 0) = 0
  AND t.size > 10
ORDER BY t.size DESC
LIMIT 30;
```

### H8 - Distribution Key Mismatch in INSERT...SELECT
> Adapted from: `src/AdminScripts/insert_into_table_dk_mismatch.sql`
```sql
SELECT DISTINCT trim(pgn.nspname) || '.' || trim(pgc.relname) AS target,
       tt.distkey AS target_dk,
       trim(pgn2.nspname) || '.' || trim(pgc2.relname) AS source,
       ts.distkey AS source_dk
FROM stl_insert i
JOIN stl_scan s ON i.query = s.query
JOIN pg_class AS pgc ON pgc.oid = i.tbl
JOIN pg_namespace AS pgn ON pgn.oid = pgc.relnamespace
JOIN pg_class AS pgc2 ON pgc2.oid = s.tbl
JOIN pg_namespace AS pgn2 ON pgn2.oid = pgc2.relnamespace
LEFT JOIN (SELECT attrelid, MIN(CASE attisdistkey WHEN 't' THEN attname ELSE NULL END) AS distkey
           FROM pg_attribute GROUP BY 1) AS tt ON tt.attrelid = i.tbl
LEFT JOIN (SELECT attrelid, MIN(CASE attisdistkey WHEN 't' THEN attname ELSE NULL END) AS distkey
           FROM pg_attribute GROUP BY 1) AS ts ON ts.attrelid = s.tbl
WHERE i.tbl <> s.tbl
  AND s.perm_table_name <> 'Internal Worktable'
  AND i.slice = 0 AND s.slice = 0
  AND (tt.distkey <> ts.distkey OR tt.distkey IS NULL OR ts.distkey IS NULL)
ORDER BY 1
LIMIT 30;
```

### H9 - Table Fragmentation Estimate
> Adapted from: `src/AdminViews/v_fragmentation_info.sql`
```sql
SELECT tbl, tablename, dbname, SUM(t_excess_blks) AS est_space_gain_blocks
FROM (
  SELECT tbl, col, node, tablename, trim(datname) AS dbname,
         SUM(excess_blks) * (col + 1) AS t_excess_blks
  FROM (SELECT tbl, slice, col, COUNT(*) AS total_blks FROM stv_blocklist WHERE num_values > 0 GROUP BY 1,2,3) a
  JOIN (SELECT tbl, slice, MAX(col) AS col FROM stv_blocklist GROUP BY 1,2) b USING (tbl, slice, col)
  JOIN (SELECT tbl, slice, col, COUNT(*) - CEIL(SUM(num_values)/130994.0) AS excess_blks
        FROM stv_blocklist WHERE num_values > 0 AND num_values < 130994 GROUP BY 1,2,3) c USING (tbl, slice, col)
  JOIN stv_slices d USING (slice)
  JOIN (SELECT id, trim(name) AS tablename, db_id FROM stv_tbl_perm WHERE slice = 0) f ON b.tbl = f.id
  JOIN pg_database g ON f.db_id = g.oid
  WHERE excess_blks > 1
  GROUP BY 1,2,3,4,5
)
WHERE tbl > 1
GROUP BY 1,2,3
HAVING SUM(t_excess_blks) > 100
ORDER BY 4 DESC
LIMIT 30;
```

### H10 - WLM QMR Rule Candidates (P99 Outlier Analysis)
> Adapted from: `src/AdminScripts/wlm_qmr_rule_candidates.sql`
```sql
WITH qmr AS (
  SELECT service_class, 'query_cpu_time'::VARCHAR(30) AS qmr_metric,
         MEDIAN(query_cpu_time) AS p50,
         PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY query_cpu_time) AS p99,
         MAX(query_cpu_time) AS pmax
  FROM svl_query_metrics_summary WHERE userid > 1 GROUP BY 1
  UNION ALL
  SELECT service_class, 'query_execution_time'::VARCHAR(30),
         MEDIAN(query_execution_time),
         PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY query_execution_time),
         MAX(query_execution_time)
  FROM svl_query_metrics_summary WHERE userid > 1 GROUP BY 1
  UNION ALL
  SELECT service_class, 'query_temp_blocks_to_disk'::VARCHAR(30),
         MEDIAN(query_temp_blocks_to_disk),
         PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY query_temp_blocks_to_disk),
         MAX(query_temp_blocks_to_disk)
  FROM svl_query_metrics_summary WHERE userid > 1 GROUP BY 1
  UNION ALL
  SELECT service_class, 'cpu_skew'::VARCHAR(30),
         MEDIAN(cpu_skew),
         PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY cpu_skew),
         MAX(cpu_skew)
  FROM svl_query_metrics_summary WHERE userid > 1 GROUP BY 1
  UNION ALL
  SELECT service_class, 'io_skew'::VARCHAR(30),
         MEDIAN(io_skew),
         PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY io_skew),
         MAX(io_skew)
  FROM svl_query_metrics_summary WHERE userid > 1 GROUP BY 1
  UNION ALL
  SELECT service_class, 'nested_loop_join_row_count'::VARCHAR(30),
         MEDIAN(nested_loop_join_row_count),
         PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY nested_loop_join_row_count),
         MAX(nested_loop_join_row_count)
  FROM svl_query_metrics_summary WHERE userid > 1 GROUP BY 1
)
SELECT service_class, qmr_metric, p50, p99, pmax,
       (LEFT(p99,1)::INT+1)*POWER(10, LENGTH((p99/10)::BIGINT)) AS candidate_rule,
       ROUND(pmax/((LEFT(p99,1)::INT+1)*POWER(10, LENGTH((p99/10)::BIGINT))), 2) AS pmax_magnitude
FROM qmr
WHERE NVL(p99, 0) >= 10
  AND (NVL(p50, 0) + NVL(p99, 0)) < NVL(pmax, 0)
  AND ((LEFT(p99,1)::INT+1)*POWER(10, LENGTH((p99/10)::BIGINT))) < NVL(pmax, 0)
ORDER BY service_class, pmax_magnitude DESC
LIMIT 30;
```
