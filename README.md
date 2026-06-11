# Redshift Performance Scan Skill

An open, cross-platform AI agent skill that connects to Amazon Redshift via MCP (Model Context Protocol), executes diagnostic queries against system tables, and generates prioritized tuning recommendations based on AWS best practices.

## Features

- **42 diagnostic SQL queries** across 8 categories (table design, query performance, WLM, maintenance, cluster config, data loading, advisor, enhanced diagnostics from awslabs/amazon-redshift-utils)
- **51 performance rules** with severity levels (CRITICAL/HIGH/MEDIUM/LOW)
- **AWS documentation citations** for every recommendation
- **Remediation SQL** examples for each finding
- **Cross-platform**: Works with Kiro, Amazon Quick Desktop, Claude Code, and any [Agent Skills](https://agentskills.io)-compatible assistant

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  AI Coding Assistant (Kiro / Amazon Quick / Claude Code / ...)   │
│                                                                 │
│  ┌──────────┐  ┌──────────────────┐  ┌───────────────────┐     │
│  │ SKILL.md │  │ diagnostic-      │  │ rules-*.md        │     │
│  │ (流程编排) │  │ queries.md       │  │ (51 条规则)        │     │
│  │          │  │ (42 条诊断SQL)    │  │                   │     │
│  └────┬─────┘  └────────┬─────────┘  └────────┬──────────┘     │
│       │                  │                     │                 │
│       ▼                  ▼                     ▼                 │
│  Phase 1: Discovery → Phase 2: Collection → Phase 3: Evaluation │
│                                              │                   │
│                                              ▼                   │
│                                    Phase 4: Report Generation    │
└────────────────────────┬────────────────────────────────────────┘
                         │ MCP Protocol (stdio)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  awslabs-redshift-mcp-server (Redshift Data API)                │
└────────────────────────┬────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Amazon Redshift Cluster (STL_* / STV_* / SVV_* / SVL_*)       │
└─────────────────────────────────────────────────────────────────┘
```

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

### Kiro

Open this project in Kiro. The skill is available via `.kiro/skills/redshift-scan/`.
Invoke via natural language: "Scan my Redshift cluster `my-cluster-id` database `dev` for performance issues."

### Amazon Quick Desktop

With MCP configured, ask in Amazon Quick Desktop:
"Run a performance scan on Redshift cluster `my-cluster-id`."

### Claude Code

```bash
cd Redshift_Performance_Improvement
claude
/redshift-scan my-cluster-id dev
```

### Any Agent Skills-Compatible Assistant

The skill at `.agents/skills/redshift-scan/SKILL.md` follows the [Agent Skills](https://agentskills.io) open standard and works with any compatible client.

## Configuration

### MCP Server (.mcp.json)

This project uses the official [awslabs Redshift MCP Server](https://awslabs.github.io/mcp/servers/redshift-mcp-server). Pre-configured for us-east-1:

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

For full configuration options (Serverless, cross-account, custom endpoints), see the [official documentation](https://awslabs.github.io/mcp/servers/redshift-mcp-server).

## Project Structure

```
.
├── .agents/skills/redshift-scan/       # Agent Skills standard (cross-platform)
│   └── SKILL.md
├── .amazonq/rules/                     # Amazon Quick Desktop project rules
│   └── redshift-scan.md
├── .claude/skills/redshift-scan/       # Claude Code skill + canonical references
│   ├── SKILL.md
│   └── references/
│       ├── diagnostic-queries.md       # 42 SQL queries (Categories A-H)
│       ├── rules-table-design.md       # 17 rules (TD-01..TD-17)
│       ├── rules-query-performance.md  # 14 rules (QP-01..QP-14)
│       ├── rules-workload-management.md # 6 rules (WL-01..WL-06)
│       ├── rules-maintenance.md        # 7 rules (MT-01..MT-07)
│       ├── rules-cluster-config.md     # 3 rules (CC-01..CC-03)
│       ├── rules-data-loading.md       # 4 rules (DL-01..DL-04)
│       └── report-template.md          # Output format specification
├── .kiro/                              # Kiro IDE support
│   ├── steering.md
│   └── skills/redshift-scan/skill.md
├── .mcp.json                           # MCP server configuration (shared)
├── docs/
│   └── RULES_PROVENANCE.md            # Rule-to-documentation mapping
├── CLAUDE.md                           # Claude Code project context
├── LICENSE                             # MIT-0
├── NOTICE                              # Copyright attribution
├── CODE_OF_CONDUCT.md                  # Amazon Open Source Code of Conduct
├── CONTRIBUTING.md                     # Contribution guidelines
└── README.md
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

## Rule Sources and Provenance

All 51 rules are derived from authoritative AWS sources. For the complete mapping of each rule to its source documentation, see **[docs/RULES_PROVENANCE.md](docs/RULES_PROVENANCE.md)**.

### 1. AWS Official Documentation (33 rules, 65%)

Primary source for rules TD-01 through TD-12, QP-01 through QP-10, WL-01 through WL-06, MT-01 through MT-06, CC-01 through CC-03, and DL-01 through DL-03.

Key documentation pages:

| Document | Rules Derived |
|----------|--------------|
| [Choose the best distribution style](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html) | TD-01, TD-02, TD-07, QP-04, QP-06 |
| [Choose the best sort key](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key.html) | TD-03, QP-07, TD-16 |
| [Vacuuming tables](https://docs.aws.amazon.com/redshift/latest/dg/t_Reclaiming_storage_space202.html) | TD-04, MT-01, MT-05 |
| [Use automatic compression](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-use-auto-compression.html) | TD-05 |
| [Use the smallest possible column size](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-smallest-column-size.html) | TD-06, TD-09 |
| [Define PK and FK constraints](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-defining-constraints.html) | TD-10 |
| [Automatic table optimization](https://docs.aws.amazon.com/redshift/latest/dg/c_autonomics.html) | TD-08, MT-04 |
| [Design queries best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-queries-best-practices.html) | QP-01, QP-02, QP-09, QP-12 |
| [Implementing WLM](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html) | WL-01, WL-02, WL-06, QP-05 |
| [Query monitoring rules](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html) | WL-02, WL-04, QP-14 |
| [Data loading best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html) | DL-01, DL-02, DL-03, DL-04, MT-03 |
| [Prescriptive Guidance — Query Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/welcome.html) | QP-03 |

### 2. awslabs/amazon-redshift-utils (10 rules, 20%)

Source for enhanced diagnostics (Category H queries) that power rules TD-13 through TD-17 and QP-11 through QP-14.

- Repository: https://github.com/awslabs/amazon-redshift-utils (Apache 2.0, 2800+ stars)
- Scripts used:

| Script | Rules |
|--------|-------|
| `src/AdminScripts/table_inspector.sql` | TD-13 (block skew) |
| `src/AdminScripts/insert_into_table_dk_mismatch.sql` | TD-14 (DK mismatch) |
| `src/AdminScripts/unscanned_table_summary.sql` | TD-15 (unscanned tables) |
| `src/AdminScripts/predicate_columns.sql` | TD-16 (predicate alignment) |
| `src/AdminViews/v_fragmentation_info.sql` | TD-17, MT-07 (fragmentation) |
| `src/AdminScripts/perf_alert.sql` | QP-12 (per-table alert impact) |
| `src/AdminScripts/missing_table_stats.sql` | QP-11 (missing stats in EXPLAIN) |
| `src/AdminScripts/top_queries.sql` | QP-13 (multi-alert queries) |
| `src/AdminScripts/wlm_qmr_rule_candidates.sql` | QP-14 (QMR candidates) |
| `src/AdminScripts/copy_performance.sql` | DL-04 (COPY throughput) |

### 3. awslabs Redshift MCP Server

- Documentation: https://awslabs.github.io/mcp/servers/redshift-mcp-server
- Source: https://github.com/awslabs/mcp
- Provides the `execute_query`, `list_clusters`, `list_databases`, `list_schemas`, `list_tables`, `list_columns` tools used to collect diagnostic data

## Extending Rules

To add a new rule:

1. Edit the appropriate `rules-*.md` file in `.claude/skills/redshift-scan/references/`
2. Follow the existing rule format (Rule ID, Severity, Trigger Condition, Diagnostic Queries, Observation Template, Recommendation, Remediation SQL, Documentation Source)
3. If the rule needs new diagnostic data, add a query to `diagnostic-queries.md`

No code changes needed — the AI agent reads and applies rules from markdown at runtime.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.

----
Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.

SPDX-License-Identifier: MIT-0
