---
name: redshift-scan
description: >
  Scan an Amazon Redshift provisioned cluster for performance issues and generate
  prioritized tuning recommendations. Use this skill when investigating Redshift
  query latency, table design problems, WLM contention, maintenance gaps, or
  data loading inefficiencies. Executes 42 read-only diagnostic queries against
  system tables and evaluates results against 51 AWS best-practice rules.
license: MIT-0
compatibility: >
  Requires awslabs.redshift-mcp-server MCP server configured with IAM credentials
  that have redshift-data:ExecuteStatement, DescribeStatement, GetStatementResult,
  redshift:DescribeClusters, and redshift:GetClusterCredentialsWithIAM permissions.
metadata:
  author: ChemExpress
  version: "1.0"
  aws-services: Amazon Redshift
---

# Redshift Performance Scan

Scan an Amazon Redshift provisioned cluster for performance issues and generate prioritized tuning recommendations based on AWS best practices.

## Arguments

- `cluster_identifier` (required): The Redshift cluster identifier to scan
- `database_name` (optional, default: "dev"): The database to analyze

## Workflow

### Phase 1: Discovery

1. Call `list_clusters` to verify the target cluster exists and get metadata (node type, node count, status)
2. Call `list_databases` on the target cluster to confirm the database exists
3. Call `list_schemas` to enumerate user schemas (skip pg_catalog, information_schema, pg_internal)

### Phase 2: Data Collection

Execute diagnostic queries from `references/diagnostic-queries.md` in this order:

1. **Category A** (Cluster Config): 4 queries — slice topology, disk usage, parameters
2. **Category B** (Table Design): 8 queries — distribution, sort keys, encoding, skew
3. **Category C** (Query Performance): 7 queries — slow queries, alerts, disk spill, skew
4. **Category D** (Workload Management): 4 queries — WLM config, throughput, QMR violations
5. **Category E** (Maintenance): 5 queries — vacuum candidates, stale stats, auto-optimization
6. **Category F** (Data Loading): 3 queries — COPY performance, load errors
7. **Category G** (Advisor): 1 query — built-in Redshift Advisor recommendations
8. **Category H** (Enhanced from awslabs/amazon-redshift-utils): 10 queries — per-table alerts, block skew, predicate columns, unscanned tables, DK mismatch, fragmentation, QMR candidates, COPY throughput, missing stats in EXPLAIN

Use `execute_query` for each. If a query fails (e.g., permission denied on a system table), log the error and continue with the remaining queries.

### Phase 3: Rule Evaluation

Read the rule reference files and evaluate each rule against the collected data:

- `references/rules-table-design.md` (17 rules: TD-01 through TD-17)
- `references/rules-query-performance.md` (14 rules: QP-01 through QP-14)
- `references/rules-workload-management.md` (6 rules: WL-01 through WL-06)
- `references/rules-maintenance.md` (7 rules: MT-01 through MT-07)
- `references/rules-cluster-config.md` (3 rules: CC-01 through CC-03)
- `references/rules-data-loading.md` (4 rules: DL-01 through DL-04)

For each rule, check if the trigger condition is met based on the diagnostic data. If triggered, record the finding with severity, affected objects, and evidence.

### Phase 4: Report Generation

Generate the report following `references/report-template.md`:

1. Executive Summary with overall health score and finding counts by severity
2. Findings grouped by severity (CRITICAL > HIGH > MEDIUM > LOW)
3. Each finding includes: rule ID, evidence data, recommendation, remediation SQL, and AWS documentation URL
4. Redshift Advisor recommendations (from query G1)
5. Automatic optimization status summary
6. Next steps prioritized by impact

Output the report as markdown to the terminal.

## Important Notes

- All queries are READ-ONLY. The MCP server enforces this.
- Remediation SQL in recommendations is for the DBA to execute manually.
- If context becomes too large, summarize intermediate results before generating the final report.
- For tables with many findings, group them and show top 5 examples with a count of remaining affected tables.
