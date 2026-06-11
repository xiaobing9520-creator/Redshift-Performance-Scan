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

## Rule Sources

### 1. AWS Official Documentation
- [Redshift Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/best-practices.html)
- [Table Design Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html)
- [Query Performance Tuning](https://docs.aws.amazon.com/redshift/latest/dg/c-optimizing-query-performance.html)
- [Workload Management](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-implementing-workload-management.html)
- [Data Loading Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/c_loading-data-best-practices.html)
- [Query Best Practices (Prescriptive Guidance)](https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/welcome.html)

### 2. awslabs Redshift MCP Server
- Documentation: https://awslabs.github.io/mcp/servers/redshift-mcp-server
- Source: https://github.com/awslabs/mcp

### 3. awslabs/amazon-redshift-utils (Apache 2.0)
- Repository: https://github.com/awslabs/amazon-redshift-utils
- Enhanced diagnostic queries (Category H) adapted from AdminScripts and AdminViews

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
