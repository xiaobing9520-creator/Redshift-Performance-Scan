# Redshift Performance Improvement

## Project Purpose
GenAI-powered Amazon Redshift performance scanning and tuning recommendation system.

## Architecture
- Uses `awslabs-redshift-mcp-server` MCP server for read-only Redshift access
- Executes diagnostic SQL queries against system tables (STL_*, STV_*, SVV_*, SVL_*)
- Evaluates results against 40 codified performance rules
- Generates prioritized report with AWS documentation citations

## Key Constraints
- All Redshift access is READ-ONLY (enforced by MCP server)
- Target: Provisioned cluster in us-east-1
- Recommendations include remediation SQL for manual DBA execution

## MCP Server
Configured in `.mcp.json` - provides `list_clusters`, `list_databases`, `list_schemas`, `list_tables`, `list_columns`, `execute_query` tools.

## Rules Location
Performance rules are in `.claude/skills/redshift-scan/references/rules-*.md` files.
Diagnostic queries are in `.claude/skills/redshift-scan/references/diagnostic-queries.md`.
