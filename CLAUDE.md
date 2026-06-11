# Redshift Performance Improvement

This project provides a GenAI-powered Redshift performance scanning skill that:
- Connects to Amazon Redshift via the `awslabs-redshift-mcp-server` MCP server
- Executes 42 diagnostic SQL queries against system tables
- Evaluates results against 51 codified AWS best practice rules
- Generates a prioritized performance tuning report with documentation citations

## MCP Server

The project uses `awslabs.redshift-mcp-server` for read-only access to Redshift clusters.
Configuration is in `.mcp.json`. Required IAM permissions:
- `redshift:DescribeClusters`
- `redshift-data:ExecuteStatement`
- `redshift-data:DescribeStatement`
- `redshift-data:GetStatementResult`
- `redshift:GetClusterCredentialsWithIAM`

## Skills

- `/redshift-scan [cluster-id] [database]` — Full performance scan with report generation

## Rules

Performance rules are in `.claude/skills/redshift-scan/references/rules-*.md` files.
Each rule includes severity, threshold, diagnostic query reference, and AWS documentation citation.
