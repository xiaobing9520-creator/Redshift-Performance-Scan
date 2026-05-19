# Redshift Performance Scan

## Description
Scan an Amazon Redshift provisioned cluster for performance issues and generate prioritized tuning recommendations based on AWS best practices. Uses the Redshift MCP server for read-only diagnostic queries against system tables.

## Arguments
- `cluster_identifier` (required): The Redshift cluster identifier to scan
- `database_name` (optional, default: "dev"): The database to analyze

## Instructions

### Phase 1: Discovery
1. Use `list_clusters` MCP tool to verify the target cluster exists, get node type and count
2. Use `list_databases` to confirm the target database
3. Use `list_schemas` to enumerate user schemas (exclude pg_catalog, information_schema, pg_internal)

### Phase 2: Data Collection
Execute all diagnostic queries from `.claude/skills/redshift-scan/references/diagnostic-queries.md` using the `execute_query` MCP tool:

- **Category A** (4 queries): Cluster topology, disk usage, parameters
- **Category B** (8 queries): Table design — distribution, sort keys, encoding, skew
- **Category C** (7 queries): Query performance — slow queries, alerts, disk spill
- **Category D** (4 queries): WLM — configuration, throughput, violations
- **Category E** (5 queries): Maintenance — vacuum candidates, stats, auto-optimization
- **Category F** (3 queries): Data loading — COPY performance, errors
- **Category G** (1 query): Redshift Advisor recommendations
- **Category H** (10 queries): Enhanced diagnostics from awslabs/amazon-redshift-utils — per-table alerts, block skew, predicate columns, unscanned tables, DK mismatch, fragmentation, QMR candidates

If a query fails, log the error and continue with remaining queries.

### Phase 3: Rule Evaluation
Read and evaluate rules from these reference files:
- `.claude/skills/redshift-scan/references/rules-table-design.md` (TD-01 to TD-17)
- `.claude/skills/redshift-scan/references/rules-query-performance.md` (QP-01 to QP-14)
- `.claude/skills/redshift-scan/references/rules-workload-management.md` (WL-01 to WL-06)
- `.claude/skills/redshift-scan/references/rules-maintenance.md` (MT-01 to MT-07)
- `.claude/skills/redshift-scan/references/rules-cluster-config.md` (CC-01 to CC-03)
- `.claude/skills/redshift-scan/references/rules-data-loading.md` (DL-01 to DL-04)

For each rule, check if the trigger condition threshold is met in the collected data. Record triggered findings with: rule ID, severity, affected objects, evidence data.

### Phase 4: Report Generation
Generate the report per `.claude/skills/redshift-scan/references/report-template.md`:
1. Executive summary with health score
2. Findings by severity (CRITICAL → HIGH → MEDIUM → LOW)
3. Each finding: evidence, recommendation, remediation SQL, AWS documentation link
4. Advisor recommendations and auto-optimization status
5. Prioritized next steps

## Notes
- All operations are read-only; remediation SQL is advisory for DBA execution
- Limit query results to avoid context overflow (LIMIT clauses are built into queries)
- Group multiple tables triggering the same rule; show top 5 examples
