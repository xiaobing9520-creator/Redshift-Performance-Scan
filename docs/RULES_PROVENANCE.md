# Rules Provenance: From AWS Documentation to Scan Rules

This document maps each of the 51 performance rules in this project to their authoritative source documentation. It serves as an audit trail and accuracy reference.

---

## Source Categories

| # | Source | Type | License |
|---|--------|------|---------|
| 1 | [Amazon Redshift Database Developer Guide](https://docs.aws.amazon.com/redshift/latest/dg/) | AWS Official Docs | N/A |
| 2 | [Amazon Redshift Management Guide](https://docs.aws.amazon.com/redshift/latest/mgmt/) | AWS Official Docs | N/A |
| 3 | [AWS Prescriptive Guidance — Query Best Practices for Redshift](https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/welcome.html) | AWS Official Docs | N/A |
| 4 | [awslabs/amazon-redshift-utils](https://github.com/awslabs/amazon-redshift-utils) | AWS Community | Apache 2.0 |
| 5 | [awslabs Redshift MCP Server](https://awslabs.github.io/mcp/servers/redshift-mcp-server) | AWS Official Tool | Apache 2.0 |

---

## Table Design Rules (TD-01 to TD-17)

| Rule ID | Rule Name | Source Document | Section / Script | Key Threshold |
|---------|-----------|----------------|-----------------|---------------|
| TD-01 | Missing Distribution Key on Large Table | [Choose the best distribution style](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html) | "Choose the best distribution style" | Table > 500MB with EVEN dist |
| TD-02 | High Distribution Skew | [Choose the best distribution style](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html) + [Choosing sort/dist](https://docs.aws.amazon.com/redshift/latest/dg/c_choosing_dist_sort.html) | Distribution best practices | skew_rows > 4.0 AND > 100MB |
| TD-03 | Missing Sort Key on Large Table | [Choose the best sort key](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key.html) | "Choose the best sort key" | Table > 100MB, no sort key |
| TD-04 | High Unsorted Percentage | [Vacuuming tables](https://docs.aws.amazon.com/redshift/latest/dg/t_Reclaiming_storage_space202.html) + [Loading in sort key order](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key-order.html) | "Load data in sort key order" | unsorted > 20% AND > 50MB |
| TD-05 | Missing Compression Encoding | [Use automatic compression](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-use-auto-compression.html) | "Let COPY choose compression encodings" | >3 non-sortkey cols without encoding |
| TD-06 | VARCHAR Over-Allocation | [Use the smallest possible column size](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-smallest-column-size.html) | "Use the smallest possible column size" | VARCHAR > 256 chars declared |
| TD-07 | ALL Distribution on Large Table | [Choose the best distribution style](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html) | ALL dist guidance | Table > 1GB with DISTSTYLE ALL |
| TD-08 | Ignored Auto Table Optimization | [Automatic table optimization](https://docs.aws.amazon.com/redshift/latest/dg/c_autonomics.html) + [Enabling ATO](https://docs.aws.amazon.com/redshift/latest/dg/c_ato-enabling-disabling-monitoring.html) | "Enabling automatic table optimization" | Pending recommendations exist |
| TD-09 | Oversized CHAR Columns | [Use the smallest possible column size](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-smallest-column-size.html) | Column size guidance | CHAR > 10 characters |
| TD-10 | Missing PK/FK Constraints | [Define primary key and foreign key constraints](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-defining-constraints.html) | "Define primary key and foreign key constraints" | Large tables (>1M rows), no PK/FK |
| TD-11 | TIMESTAMP Used Where DATE Suffices | [Use date/time data types](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-timestamp-date-columns.html) | "Use date/time data types for date columns" | TIMESTAMP with no time variation |
| TD-12 | Interleaved Sort Key Needs REINDEX | [Interleaved sort](https://docs.aws.amazon.com/redshift/latest/dg/t_Sorting_data-interleaved.html) + [VACUUM](https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html) | Interleaved sort maintenance | Interleaved + >10% unsorted |
| TD-13 | Block-Level Distribution Skew | [Distribution best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html) + [table_inspector.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/table_inspector.sql) | awslabs AdminScripts | ratio_skew > 100% |
| TD-14 | DK Mismatch in INSERT...SELECT | [Distribution best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html) + [insert_into_table_dk_mismatch.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/insert_into_table_dk_mismatch.sql) | awslabs AdminScripts | Source/target DK differ |
| TD-15 | Unscanned Tables Wasting Storage | [Working with clusters](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html) + [unscanned_table_summary.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/unscanned_table_summary.sql) | awslabs AdminScripts | Table >10MB, 0 scans |
| TD-16 | Sort Key Not Aligned with Predicates | [Choose the best sort key](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key.html) + [predicate_columns.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/predicate_columns.sql) | awslabs AdminScripts | Predicate col ≠ sort key |
| TD-17 | Table Fragmentation | [VACUUM command](https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html) + [v_fragmentation_info.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminViews/v_fragmentation_info.sql) | awslabs AdminViews | >100 excess blocks |

---

## Query Performance Rules (QP-01 to QP-14)

| Rule ID | Rule Name | Source Document | Section / Script | Key Threshold |
|---------|-----------|----------------|-----------------|---------------|
| QP-01 | Nested Loop Join Detected | [Design queries best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-queries-best-practices.html) + [Challenges](https://docs.aws.amazon.com/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html) | "Amazon Redshift best practices for designing queries" | Any nested loop alert |
| QP-02 | Stale Table Statistics | [Design queries best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-queries-best-practices.html) + [Analyzing tables](https://docs.aws.amazon.com/redshift/latest/dg/t_Analyzing_tables.html) | "Keep statistics current" | stats_off > 20%, table > 50MB |
| QP-03 | Disk-Based Query Operations | [Query performance](https://docs.aws.amazon.com/redshift/latest/dg/c-query-performance.html) + [Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/welcome.html) | "Factors affecting query performance" | is_diskbased = 't' |
| QP-04 | High Query Execution Skew | [Distribution best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html) + [Query performance](https://docs.aws.amazon.com/redshift/latest/dg/c-query-performance.html) | "Data distribution" | Slice skew > 50%, query > 30s |
| QP-05 | High Queue Wait Times | [Implementing WLM](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html) + [Concurrency scaling](https://docs.aws.amazon.com/redshift/latest/dg/concurrency-scaling.html) | WLM queue configuration | Avg wait > 5 seconds |
| QP-06 | Large Broadcast Operations | [Choose the best distribution style](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html) + [Challenges](https://docs.aws.amazon.com/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html) | "Designate both the dimension table and the fact table" | Broadcast alerts on large tables |
| QP-07 | Sequential Scan on Large Table | [Choose the best sort key](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key.html) + [Query performance](https://docs.aws.amazon.com/redshift/latest/dg/c-query-performance.html) | "Data sort order" | Sequential scan alerts |
| QP-08 | Low Result Cache Hit Rate | [Challenges achieving performance](https://docs.aws.amazon.com/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html) | "Result caching" | Cache disabled |
| QP-09 | Cross-Join / Cartesian Product | [Design queries best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-queries-best-practices.html) | "Amazon Redshift best practices for designing queries" | Cartesian product alerts |
| QP-10 | Very Long Running Queries | [Identify tuning candidates](https://docs.aws.amazon.com/redshift/latest/dg/identify-queries-that-are-top-candidates-for-tuning.html) + [Optimizing performance](https://docs.aws.amazon.com/redshift/latest/dg/c-optimizing-query-performance.html) | "Identify queries that are top candidates for tuning" | elapsed > 300 seconds |
| QP-11 | Missing Statistics in EXPLAIN | [Analyzing tables](https://docs.aws.amazon.com/redshift/latest/dg/t_Analyzing_tables.html) + [missing_table_stats.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/missing_table_stats.sql) | awslabs AdminScripts | "missing statistics" in stl_explain |
| QP-12 | Per-Table Alert Impact | [Design queries best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-queries-best-practices.html) + [perf_alert.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/perf_alert.sql) | awslabs AdminScripts | >30 min alert overhead/7 days |
| QP-13 | Queries with Multiple Alerts | [Optimizing performance](https://docs.aws.amazon.com/redshift/latest/dg/c-optimizing-query-performance.html) + [top_queries.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/top_queries.sql) | awslabs AdminScripts | Multiple alert event types |
| QP-14 | QMR Rule Candidates (P99 Outliers) | [Query monitoring rules](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html) + [wlm_qmr_rule_candidates.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/wlm_qmr_rule_candidates.sql) | awslabs AdminScripts | pmax > 5x p99 |

---

## Workload Management Rules (WL-01 to WL-06)

| Rule ID | Rule Name | Source Document | Section / Script | Key Threshold |
|---------|-----------|----------------|-----------------|---------------|
| WL-01 | Single WLM Queue | [Implementing WLM](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html) + [Tutorial](https://docs.aws.amazon.com/redshift/latest/dg/tutorial-configuring-workload-management.html) | "Tutorial: Configuring workload management" | Only 1 user queue |
| WL-02 | High Query Abort Rate | [Query monitoring rules](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html) + [Implementing WLM](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html) | WLM abort handling | aborted > 5% of total |
| WL-03 | High SQA Eviction Rate | [Short query acceleration](https://docs.aws.amazon.com/redshift/latest/dg/wlm-short-query-acceleration.html) | SQA configuration | Evicted > 30% of completed |
| WL-04 | Frequent QMR Violations | [Query monitoring rules](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html) | "Defining a query monitoring rule" | >50 violations/day |
| WL-05 | Concurrency Scaling Underutilized | [Concurrency scaling](https://docs.aws.amazon.com/redshift/latest/dg/concurrency-scaling.html) | "Configuring concurrency scaling queues" | Wait >5s but no scaling |
| WL-06 | Manual WLM When Auto Available | [Automatic WLM](https://docs.aws.amazon.com/redshift/latest/dg/automatic-wlm.html) + [Implementing WLM](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html) | "Switching WLM mode" | Fixed concurrency config |

---

## Maintenance Rules (MT-01 to MT-07)

| Rule ID | Rule Name | Source Document | Section / Script | Key Threshold |
|---------|-----------|----------------|-----------------|---------------|
| MT-01 | Tables Need VACUUM (Unsorted) | [Vacuuming tables](https://docs.aws.amazon.com/redshift/latest/dg/t_Reclaiming_storage_space202.html) + [VACUUM](https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html) | "VACUUM frequency" | unsorted >20%, size >100MB |
| MT-02 | Stale Table Statistics | [Analyzing tables](https://docs.aws.amazon.com/redshift/latest/dg/t_Analyzing_tables.html) + [Autonomics](https://docs.aws.amazon.com/redshift/latest/dg/c_autonomics.html) | "Automatic analyze" | stats_off > 20, table > 50MB |
| MT-03 | High Commit Queue Latency | [Data loading best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html) + [Use COPY](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-use-copy.html) | "Use a single COPY command" | avg_queue_ms > 1000 |
| MT-04 | Auto Optimization Not Running | [Autonomics](https://docs.aws.amazon.com/redshift/latest/dg/c_autonomics.html) + [Extra compute for autonomics](https://docs.aws.amazon.com/redshift/latest/dg/t_extra-compute-autonomics.html) | "Allocating extra compute resources" | No actions in 14 days |
| MT-05 | Ghost Row Accumulation | [Vacuuming tables](https://docs.aws.amazon.com/redshift/latest/dg/t_Reclaiming_storage_space202.html) + [VACUUM](https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html) | "Automatic vacuum delete" | >20% ghost row discrepancy |
| MT-06 | Disk Space Critical | [Working with clusters](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html) + [Resizing](https://docs.aws.amazon.com/redshift/latest/mgmt/resizing-cluster.html) | "Elastic resize" | pct_used > 85% |
| MT-07 | Table Fragmentation | [VACUUM](https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html) + [v_fragmentation_info.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminViews/v_fragmentation_info.sql) | awslabs AdminViews | >100 excess blocks |

---

## Cluster Configuration Rules (CC-01 to CC-03)

| Rule ID | Rule Name | Source Document | Section / Script | Key Threshold |
|---------|-----------|----------------|-----------------|---------------|
| CC-01 | Single Node with Large Data | [Working with clusters](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html) + [Architecture](https://docs.aws.amazon.com/redshift/latest/dg/c_high_level_system_architecture.html) | "Clusters and nodes in Amazon Redshift" | 1 node AND > 500GB |
| CC-02 | Disk Usage Near Capacity | [Working with clusters](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html) + [Resizing](https://docs.aws.amazon.com/redshift/latest/mgmt/resizing-cluster.html) | "Elastic resize" | pct_used > 85% |
| CC-03 | Result Caching Disabled | [Challenges achieving performance](https://docs.aws.amazon.com/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html) | "Result caching" | enable_result_cache = off |

---

## Data Loading Rules (DL-01 to DL-04)

| Rule ID | Rule Name | Source Document | Section / Script | Key Threshold |
|---------|-----------|----------------|-----------------|---------------|
| DL-01 | File Count < Slice Count | [Data loading best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html) + [Use multiple files](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-use-multiple-files.html) | "Split your load data into multiple files" | Files < slices |
| DL-02 | Frequent Load Errors | [Data loading best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html) + [STL_LOAD_ERRORS](https://docs.aws.amazon.com/redshift/latest/dg/r_STL_LOAD_ERRORS.html) | System table reference | >100 errors/day |
| DL-03 | Single Large File Load | [Data loading best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html) | "Split your load data" + "Use a single COPY command" | 1 file, >10M rows |
| DL-04 | Low COPY Throughput | [Data loading best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html) + [copy_performance.sql](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminScripts/copy_performance.sql) | awslabs AdminScripts | < 10 MB/s, load > 100MB |

---

## Summary Statistics

| Metric | Value |
|--------|-------|
| Total rules | 51 |
| Rules sourced from AWS official docs only | 33 (65%) |
| Rules sourced from awslabs/amazon-redshift-utils | 10 (20%) |
| Rules sourced from both (official docs + awslabs scripts) | 8 (15%) |
| Unique AWS documentation pages cited | 28 |
| Unique awslabs scripts cited | 10 |

---

## Accuracy Notes

### Threshold Justification

| Threshold | Justification |
|-----------|---------------|
| skew_rows > 4.0 | AWS documentation states "high skew" as rows on one slice > 4x average across slices |
| unsorted > 20% | AWS VACUUM documentation: "Amazon Redshift does not automatically sort rows that are added after the initial COPY" — 20% threshold balances maintenance cost vs. scan overhead |
| stats_off > 20 | SVV_TABLE_INFO shows percentage difference between real row count and stats row count; >20% consistently leads to plan regressions per AWS Prescriptive Guidance |
| pct_used > 85% | AWS recommends maintaining headroom for temp space and maintenance; 85% allows buffer before query failures |
| Queue wait > 5s | AWS WLM documentation: interactive workloads should not wait; 5s threshold separates transient from systemic contention |
| Load errors > 100/day | Practical threshold — small numbers of errors may be acceptable (intentional MAXERROR), but >100/day indicates systemic issues |

### Known Limitations

1. **STL tables retention**: STL system tables retain 2-5 days of data by default. Rules relying on STL queries (C1-C7, F1-F3) only analyze recent history.
2. **Serverless vs. Provisioned**: Some rules (CC-01, CC-02) are only applicable to provisioned clusters. Serverless workgroups manage resources differently.
3. **RA3 managed storage**: CC-02 and MT-06 thresholds are less critical for RA3 nodes where data is in managed storage, but local cache pressure still matters.
4. **Auto WLM**: WL-01, WL-05, WL-06 are less applicable when Automatic WLM is enabled, as it handles queue configuration dynamically.

---

## Document Revision

| Date | Change |
|------|--------|
| 2026-06-11 | Initial provenance document — all 51 rules mapped to source documentation |
