# Query Performance Rules

## QP-01: Nested Loop Join Detected

**Severity**: CRITICAL
**Category**: Query Performance
**Trigger Condition**: Any queries with nested loop join alerts in stl_alert_event_log
**Diagnostic Queries**: C3

**Observation Template**:
> {count} queries triggered nested loop join alerts in the last 7 days. Nested loops are extremely expensive in Redshift's columnar architecture and typically indicate missing join conditions or Cartesian products.

**Recommendation**:
Review the flagged queries for missing or incorrect JOIN conditions. Ensure that join columns have matching data types and that tables have appropriate distribution keys to co-locate joined data.

**Remediation SQL**:
```sql
-- Identify the problematic query
SELECT query, SUBSTRING(querytxt, 1, 500)
FROM stl_query WHERE query = {query_id};

-- Check EXPLAIN plan for the nested loop
EXPLAIN SELECT ... (the problematic query);
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_designing-queries-best-practices.html
- https://docs.aws.amazon.com/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html

---

## QP-02: Stale Table Statistics

**Severity**: HIGH
**Category**: Query Performance
**Trigger Condition**: stats_off > 20% for tables > 50MB
**Diagnostic Queries**: E4

**Observation Template**:
> {count} tables have statistics more than 20% stale. The query planner may generate suboptimal execution plans with outdated statistics.

**Recommendation**:
Run ANALYZE on affected tables. Verify that automatic ANALYZE is enabled. For frequently-updated tables, consider running ANALYZE after major loads.

**Remediation SQL**:
```sql
ANALYZE {schema}.{table};
-- Or analyze all tables:
ANALYZE;
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_designing-queries-best-practices.html
- https://docs.aws.amazon.com/redshift/latest/dg/t_Analyzing_tables.html

---

## QP-03: Disk-Based Query Operations (Disk Spill)

**Severity**: HIGH
**Category**: Query Performance
**Trigger Condition**: is_diskbased = 't' in svl_query_summary for recent queries
**Diagnostic Queries**: C4

**Observation Template**:
> {count} query steps spilled to disk in the last 3 days. Disk-based operations are orders of magnitude slower than in-memory operations.

**Recommendation**:
- Increase WLM memory allocation for the queue running these queries
- Reduce concurrency to give each query more memory
- Optimize the query to reduce intermediate result sizes (add filters earlier, reduce GROUP BY columns)
- Consider resizing the cluster for more memory per node

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c-query-performance.html
- Section: "Factors affecting query performance — Number of nodes"
- https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/welcome.html

---

## QP-04: High Query Execution Skew

**Severity**: MEDIUM
**Category**: Query Performance
**Trigger Condition**: Per-slice skew > 50% for queries running > 30 seconds
**Diagnostic Queries**: C5

**Observation Template**:
> {count} long-running queries show significant execution skew (>50% imbalance across slices). Some slices process far more data than others.

**Recommendation**:
Execution skew typically indicates a distribution key problem. The data involved in the query is concentrated on fewer slices. Review the distribution keys of tables in the query and consider redistribution.

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html
- https://docs.aws.amazon.com/redshift/latest/dg/c-query-performance.html
- Section: "Data distribution"

---

## QP-05: High Queue Wait Times

**Severity**: HIGH
**Category**: Query Performance
**Trigger Condition**: Average queue wait time > 5 seconds for any WLM service class
**Diagnostic Queries**: C6

**Observation Template**:
> WLM service class {service_class} has average queue wait time of {avg_queue_sec} seconds (max: {max_queue_sec}s). Queries are waiting too long for available slots.

**Recommendation**:
- Increase concurrency for the affected queue
- Enable concurrency scaling to handle bursts
- Review if queries can be routed to different queues
- Consider Short Query Acceleration (SQA) for quick queries stuck behind long ones

**Remediation SQL**:
```sql
-- Check current WLM config
SELECT * FROM stv_wm_service_class_config WHERE service_class > 4;
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html
- https://docs.aws.amazon.com/redshift/latest/dg/concurrency-scaling.html

---

## QP-06: Large Broadcast Operations

**Severity**: MEDIUM
**Category**: Query Performance
**Trigger Condition**: Alert events containing 'broadcast' or 'distribution' with large row counts
**Diagnostic Queries**: C2

**Observation Template**:
> Broadcast/redistribution alerts detected for queries involving large datasets. Broadcasting a large table to all nodes is expensive.

**Recommendation**:
Co-locate large joined tables by choosing the same distribution key (the join column). Use KEY distribution on the join column for both tables to eliminate broadcast.

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html
- Section: "Designate both the dimension table and the fact table"
- https://docs.aws.amazon.com/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html

---

## QP-07: Sequential Scan on Large Table

**Severity**: MEDIUM
**Category**: Query Performance
**Trigger Condition**: Alert events indicating sequential scan where zone map could help
**Diagnostic Queries**: C2

**Observation Template**:
> Sequential scan alerts detected. Queries are scanning entire tables when zone maps could eliminate blocks.

**Recommendation**:
Ensure the table has a sort key on the column used in the WHERE clause. Zone maps (min/max metadata per block) are only effective when data is sorted.

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key.html
- https://docs.aws.amazon.com/redshift/latest/dg/c-query-performance.html
- Section: "Data sort order"

---

## QP-08: Low Result Cache Hit Rate

**Severity**: LOW
**Category**: Query Performance
**Trigger Condition**: enable_result_cache_for_session is 'off' OR repeated identical queries not benefiting from cache
**Diagnostic Queries**: A3

**Observation Template**:
> Result caching is disabled for this session/cluster. Repeated identical queries cannot benefit from cached results.

**Recommendation**:
Enable result caching unless your workload requires always-fresh results. Result caching can dramatically improve response times for repeated analytical queries.

**Remediation SQL**:
```sql
SET enable_result_cache_for_session = on;
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html
- Section: "Result caching"

---

## QP-09: Cross-Join / Cartesian Product Detected

**Severity**: HIGH
**Category**: Query Performance
**Trigger Condition**: Alert events containing 'Cartesian product' or cross-join indicators
**Diagnostic Queries**: C2

**Observation Template**:
> Cartesian product alerts detected. Queries are performing cross-joins, which generate massive intermediate result sets.

**Recommendation**:
Review queries for missing JOIN conditions. Ensure all table relationships are properly expressed with ON clauses. Cross-joins should only be used intentionally for small datasets.

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_designing-queries-best-practices.html
- Section: "Amazon Redshift best practices for designing queries"

---

## QP-10: Very Long Running Queries

**Severity**: MEDIUM
**Category**: Query Performance
**Trigger Condition**: Queries with elapsed > 300 seconds (5 minutes)
**Diagnostic Queries**: C1

**Observation Template**:
> {count} queries took longer than 5 minutes in the last 7 days. The longest query took {max_sec} seconds.

**Recommendation**:
Review the top slow queries for optimization opportunities:
1. Check EXPLAIN plans for inefficient steps
2. Verify table design (sort keys, distribution)
3. Look for missing statistics
4. Consider materialized views for complex aggregations

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/identify-queries-that-are-top-candidates-for-tuning.html
- https://docs.aws.amazon.com/redshift/latest/dg/c-optimizing-query-performance.html

---

## QP-11: Missing Statistics Flagged in EXPLAIN Plans

**Severity**: HIGH
**Category**: Query Performance
**Trigger Condition**: stl_explain contains nodes with 'missing statistics' text
**Diagnostic Queries**: H5

**Observation Template**:
> {occurrence_count} EXPLAIN plan nodes flagged "missing statistics" for tables. The optimizer cannot generate efficient plans without current statistics.

**Recommendation**:
Run ANALYZE on all tables flagged with missing statistics. This is more reliable than checking stats_off in svv_table_info because it shows tables where the optimizer actually hit the problem during planning.

**Remediation SQL**:
```sql
-- Analyze all tables (safest approach)
ANALYZE;

-- Or target specific tables from the plannode output
ANALYZE {schema}.{table};
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/t_Analyzing_tables.html
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/missing_table_stats.sql

---

## QP-12: Per-Table Alert Impact Analysis

**Severity**: HIGH
**Category**: Query Performance
**Trigger Condition**: Any table accumulating >30 minutes of alert-related overhead in 7 days
**Diagnostic Queries**: H1

**Observation Template**:
> Table `{table}` has accumulated {minutes} minutes of performance alert overhead. Events: {event} ({occurrence_count} occurrences). Solution: {solution}

**Recommendation**:
This query shows the actual time cost of each performance alert per table. Focus optimization on tables with the highest accumulated alert minutes — these represent the biggest potential time savings.

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c_designing-queries-best-practices.html
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/perf_alert.sql

---

## QP-13: Queries with Multiple Alert Types

**Severity**: MEDIUM
**Category**: Query Performance
**Trigger Condition**: Queries appearing in H3 with multiple alert event types (Filter, Deleted, Nested Loop, Distributed, Broadcast, Stats)
**Diagnostic Queries**: H3

**Observation Template**:
> Query (md5: {qry_md5}) runs {n_qry} times, total {total} seconds. Alert types: {events}. Multiple alert types indicate compound performance issues.

**Recommendation**:
Queries with multiple alert types need comprehensive redesign, not just one fix:
- **Stats** + **Distributed**: Missing stats causes bad distribution decisions
- **Nested Loop** + **Broadcast**: Missing join conditions or type mismatches
- **Filter** + **Deleted**: Stale data + missing sort keys

Address alerts in order: Stats → Distribution → Sort → Query rewrite.

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/c-optimizing-query-performance.html
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/top_queries.sql

---

## QP-14: QMR Rule Candidate Identification (P99 Outliers)

**Severity**: LOW
**Category**: Query Performance
**Trigger Condition**: Metrics where max value significantly exceeds p99 (pmax_magnitude > 5x)
**Diagnostic Queries**: H10

**Observation Template**:
> Service class {service_class}: metric `{qmr_metric}` has p99={p99} but max={pmax} ({pmax_magnitude}x beyond candidate threshold). A QMR rule at {candidate_rule} would catch extreme outliers without affecting normal queries.

**Recommendation**:
Implement Query Monitoring Rules (QMR) to catch runaway queries before they consume excessive resources. Start with LOG action to assess impact, then escalate to HOP or ABORT.

**Remediation SQL**:
```sql
-- Via parameter group WLM JSON configuration:
-- Add rule to queue: {"query_{qmr_metric}": {candidate_rule}, "action": "log"}
```

**Documentation Source**:
- https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html
- https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/wlm_qmr_rule_candidates.sql
