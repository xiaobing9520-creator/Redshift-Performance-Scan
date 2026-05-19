# Diagnostic Queries

All queries are SELECT-only and safe for production execution. Each includes LIMIT clauses to manage result size.

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
