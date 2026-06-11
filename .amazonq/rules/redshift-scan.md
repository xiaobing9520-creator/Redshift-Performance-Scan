# Redshift Performance Scan Rules

When working with Amazon Redshift in this project, follow these guidelines:

## MCP Server

This project uses `awslabs.redshift-mcp-server` for read-only access to Redshift clusters.
The MCP configuration is in `.mcp.json`. All Redshift access is READ-ONLY.

## Diagnostic Approach

When asked to scan or diagnose a Redshift cluster:

1. Use the skill defined in `.agents/skills/redshift-scan/SKILL.md`
2. Reference diagnostic queries from `.claude/skills/redshift-scan/references/diagnostic-queries.md`
3. Evaluate results against rules in `.claude/skills/redshift-scan/references/rules-*.md`
4. Generate reports per `.claude/skills/redshift-scan/references/report-template.md`

## Key Constraints

- Never execute DDL or DML against the cluster — all access is read-only
- Remediation SQL is advisory only — output for manual DBA execution
- Use LIMIT clauses to prevent context overflow
- Group findings by severity: CRITICAL > HIGH > MEDIUM > LOW
- Always cite AWS documentation URLs in recommendations

## Rule Categories

- **Table Design** (TD-01 to TD-17): Distribution keys, sort keys, encoding, VARCHAR sizing, block skew, DK mismatch, predicate alignment, fragmentation, unscanned tables
- **Query Performance** (QP-01 to QP-14): Nested loops, disk spill, execution skew, queue waits, missing stats, per-table alert impact, QMR candidates
- **Workload Management** (WL-01 to WL-06): Queue config, abort rates, SQA eviction, concurrency scaling
- **Maintenance** (MT-01 to MT-07): VACUUM, ANALYZE, ghost rows, disk space, fragmentation
- **Cluster Config** (CC-01 to CC-03): Node count, disk capacity, result caching
- **Data Loading** (DL-01 to DL-04): File parallelism, load errors, single-file loads, S3 throughput
