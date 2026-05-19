# Redshift Performance Scan

A GenAI-powered performance scanning skill that connects to Amazon Redshift via MCP, executes diagnostic queries against system tables, and generates prioritized tuning recommendations based on AWS best practices.

## Features

- **32 diagnostic SQL queries** across 7 categories (table design, query performance, WLM, maintenance, cluster config, data loading, advisor)
- **40 performance rules** with severity levels (CRITICAL/HIGH/MEDIUM/LOW)
- **AWS documentation citations** for every recommendation
- **Remediation SQL** examples for each finding
- Works with both **Claude Code** (`/redshift-scan`) and **Kiro**

## Prerequisites

### 1. Install uv (Python package manager)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. AWS Credentials

Configure AWS CLI with credentials that have access to your Redshift cluster:

```bash
aws configure --profile default
```

### 3. IAM Permissions

The IAM user/role needs these permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "redshift:DescribeClusters",
        "redshift:GetClusterCredentialsWithIAM",
        "redshift:GetClusterCredentials",
        "redshift-data:ExecuteStatement",
        "redshift-data:DescribeStatement",
        "redshift-data:GetStatementResult",
        "redshift-serverless:ListWorkgroups",
        "redshift-serverless:GetWorkgroup",
        "redshift-serverless:GetCredentials"
      ],
      "Resource": "*"
    }
  ]
}
```

### 4. Database Permissions

The database user connecting via IAM needs:
- `SELECT` on system tables (STL_*, STV_*, SVV_*, SVL_*)
- `USAGE` on schemas to explore
- Typically, a superuser or a user with `pg_monitor` role equivalent

## Usage

### Claude Code

```bash
# Navigate to this project directory
cd Redshift_Performance_Improvement

# Start Claude Code (MCP server auto-starts via .mcp.json)
claude

# Run the scan
/redshift-scan my-cluster-id dev
```

### Kiro

Open this project in Kiro. The skill is available in `.kiro/skills/redshift-scan/`.
Invoke it with your cluster identifier and database name.

## Configuration

### MCP Server (.mcp.json)

The Redshift MCP server is pre-configured for us-east-1:

```json
{
  "mcpServers": {
    "awslabs-redshift-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.redshift-mcp-server@latest"],
      "env": {
        "AWS_PROFILE": "default",
        "AWS_DEFAULT_REGION": "us-east-1",
        "FASTMCP_LOG_LEVEL": "INFO"
      }
    }
  }
}
```

To change region or profile, edit `.mcp.json`.

## Project Structure

```
.claude/skills/redshift-scan/
├── SKILL.md                    # Skill definition (workflow instructions)
└── references/
    ├── diagnostic-queries.md   # 42 SQL queries (A-G original + H from awslabs/amazon-redshift-utils)
    ├── rules-table-design.md   # 17 rules (TD-01..TD-17)
    ├── rules-query-performance.md  # 14 rules (QP-01..QP-14)
    ├── rules-workload-management.md # 6 rules (WL-01..WL-06)
    ├── rules-maintenance.md    # 7 rules (MT-01..MT-07)
    ├── rules-cluster-config.md # 3 rules (CC-01..CC-03)
    ├── rules-data-loading.md   # 4 rules (DL-01..DL-04)
    └── report-template.md      # Output format specification
```

## Rule Categories

| Category | Rules | Covers |
|----------|-------|--------|
| Table Design | TD-01 to TD-17 | Distribution keys, sort keys, encoding, VARCHAR sizing, constraints, block skew, DK mismatch, predicate alignment, fragmentation, unscanned tables |
| Query Performance | QP-01 to QP-14 | Nested loops, disk spill, skew, queue waits, caching, missing stats in EXPLAIN, per-table alert impact, QMR candidates |
| Workload Management | WL-01 to WL-06 | Queue config, abort rates, SQA, concurrency scaling |
| Maintenance | MT-01 to MT-07 | VACUUM, ANALYZE, ghost rows, disk space, fragmentation |
| Cluster Config | CC-01 to CC-03 | Node count, disk capacity, result caching |
| Data Loading | DL-01 to DL-04 | File parallelism, load errors, single-file loads, S3 transfer throughput |

## Rule Sources

Rules are derived from two primary sources:

### 1. AWS Official Documentation
- [Redshift Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/best-practices.html)
- [Table Design Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html)
- [Query Performance Tuning](https://docs.aws.amazon.com/redshift/latest/dg/c-optimizing-query-performance.html)
- [Workload Management](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html)
- [Automatic Optimization](https://docs.aws.amazon.com/redshift/latest/dg/c_autonomics.html)
- [Data Loading Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html)
- [Query Best Practices (Prescriptive Guidance)](https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/welcome.html)
- [Monitoring Performance](https://docs.aws.amazon.com/redshift/latest/mgmt/metrics.html)
- [Redshift Spectrum Performance](https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-performance.html)

### 2. awslabs/amazon-redshift-utils (Apache 2.0)
- Repository: https://github.com/awslabs/amazon-redshift-utils (2800+ stars)
- Enhanced diagnostic queries (Category H) adapted from:
  - `src/AdminScripts/perf_alert.sql` — Per-table performance alert aggregation
  - `src/AdminScripts/table_inspector.sql` — Block-level distribution skew analysis
  - `src/AdminScripts/top_queries.sql` — Top queries with alert classification
  - `src/AdminScripts/copy_performance.sql` — S3 transfer throughput analysis
  - `src/AdminScripts/missing_table_stats.sql` — Missing stats in EXPLAIN plans
  - `src/AdminScripts/predicate_columns.sql` — Sort key candidate identification
  - `src/AdminScripts/unscanned_table_summary.sql` — Wasted storage detection
  - `src/AdminScripts/insert_into_table_dk_mismatch.sql` — DK mismatch in ETL
  - `src/AdminScripts/wlm_qmr_rule_candidates.sql` — P99 outlier QMR candidates
  - `src/AdminViews/v_fragmentation_info.sql` — Table fragmentation estimation

## Extending Rules

To add a new rule:

1. Edit the appropriate `rules-*.md` file
2. Follow the existing rule format:
   - Rule ID, Severity, Trigger Condition, Threshold
   - Diagnostic Queries Used (reference IDs from diagnostic-queries.md)
   - Observation Template, Recommendation, Remediation SQL
   - Documentation Source (AWS docs URL and/or github.com/awslabs/amazon-redshift-utils path)
3. If the rule needs new diagnostic data, add a query to `diagnostic-queries.md`

No code changes needed — the LLM reads and applies rules from markdown.
