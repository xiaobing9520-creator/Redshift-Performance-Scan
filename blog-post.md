# 基于 Agent Skills 开放标准构建 Amazon Redshift 智能性能巡检——跨平台 AI 技能的实践

**作者**：陈醒（ChemExpress 数据平台团队）
**日期**：2026 年 6 月

---

## 摘要

Amazon Redshift 的性能调优长期依赖 DBA 的经验积累与手动排查。本文介绍一种基于 AI Agent Skill 的 Redshift 性能巡检方案：遵循 [Agent Skills](https://agentskills.io) 开放标准定义技能，通过 MCP（Model Context Protocol）协议安全连接 Redshift Data API，将 42 条诊断 SQL 和 51 条最佳实践规则编码为可复用的知识资产。该技能支持 Kiro、Amazon Q Developer、Claude Code 及任何兼容 Agent Skills 标准的 AI 编程助手，DBA 可在自己熟悉的工具中一键触发巡检，数分钟内获得覆盖六大维度的优先级报告。

---

## 目录

一、引言：Redshift 性能调优的痛点
二、方案设计思路
三、架构概览
四、跨平台技能设计
五、核心实现
六、规则体系
七、效果展示
八、安全与只读保障
九、总结与下一步

---

## 一、引言：Redshift 性能调优的痛点

如果你管理过 Amazon Redshift 集群，你大概率经历过这样的场景：

- **排查效率低**：报表查询突然变慢，需要逐一检查 STL_ALERT_EVENT_LOG、SVV_TABLE_INFO、STL_WLM_QUERY 等十余张系统表，拼凑问题全貌。
- **知识碎片化**：分布键选择、排序键对齐、VACUUM 策略、WLM 队列配置……最佳实践散落在文档不同章节，需结合数据规模做判断。
- **人工成本高**：一次完整巡检需资深 DBA 花费半天到一天，且依赖个人经验决定检查范围和优先级。
- **工具碎片化**：团队使用不同的 AI 编程助手（Kiro、Amazon Q、Cursor 等），希望性能巡检能力在任何工具中都能使用。

**核心问题**：能否将 DBA 的巡检知识编码为跨平台可复用的 AI 技能，在任何支持 MCP 的 AI 助手中一键触发？

---

## 二、方案设计思路

### 2.1 为什么选择 Agent Skills + MCP？

| 方案 | 优势 | 不足 |
|------|------|------|
| Shell/Python 脚本 | 执行确定性高 | 规则硬编码，新增需改代码；不可跨 AI 平台复用 |
| CloudWatch 告警 | 实时性好 | 仅覆盖指标层，无法做表级/查询级深度分析 |
| Redshift Advisor | 零部署 | 建议范围有限，无法自定义阈值和优先级 |
| **Agent Skill + MCP** | 规则即文档；一次编写，跨平台运行 | 需要 AI 工具链 |

关键设计决策：

1. **Agent Skills 开放标准**：技能以 `SKILL.md`（YAML frontmatter + Markdown 指令）定义，被 Kiro、Amazon Q Developer、Claude Code 等多个 AI 助手原生支持
2. **MCP 作为标准化数据层**：通过 `awslabs-redshift-mcp-server` 实现安全、只读的数据库访问，同一 `.mcp.json` 配置被所有工具共享
3. **规则即文档**：51 条规则以 Markdown 描述，新增规则 = 追加文本，无需编译部署

### 2.2 跨平台兼容矩阵

| AI 助手 | 技能位置 | 项目上下文 | MCP 配置 |
|---------|---------|-----------|---------|
| **Kiro** | `.kiro/skills/redshift-scan/skill.md` | `.kiro/steering.md` | `.mcp.json` |
| **Amazon Q Developer** | `.amazonq/rules/redshift-scan.md` | 同上 | `.amazonq/default.json` 或 `.mcp.json` |
| **Agent Skills 标准** | `.agents/skills/redshift-scan/SKILL.md` | — | `.mcp.json` |
| **Claude Code** | `.claude/skills/redshift-scan/SKILL.md` | `CLAUDE.md` | `.mcp.json` |

所有平台共享同一套规则知识库（`.claude/skills/redshift-scan/references/`），通过路径引用实现 "一处维护，多处使用"。

---

## 三、架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│  AI 编程助手 (Kiro / Amazon Q Developer / ...)                   │
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
│  awslabs-redshift-mcp-server                                    │
│  (uvx awslabs.redshift-mcp-server@latest)                       │
│                                                                 │
│  Tools: list_clusters | list_databases | list_schemas           │
│         list_tables   | list_columns   | execute_query          │
└────────────────────────┬────────────────────────────────────────┘
                         │ Redshift Data API + IAM 临时凭证
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Amazon Redshift Cluster                                        │
│  系统表: STL_* / STV_* / SVV_* / SVL_*                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、跨平台技能设计

### 4.1 Agent Skills 标准格式

技能遵循 [Agent Skills 开放标准](https://agentskills.io/specification)，核心是一个带 YAML frontmatter 的 Markdown 文件：

```yaml
---
name: redshift-scan
description: >
  Scan an Amazon Redshift provisioned cluster for performance issues and generate
  prioritized tuning recommendations. Use this skill when investigating Redshift
  query latency, table design problems, WLM contention, maintenance gaps, or
  data loading inefficiencies.
license: MIT-0
compatibility: >
  Requires awslabs.redshift-mcp-server MCP server with IAM credentials.
metadata:
  author: ChemExpress
  version: "1.0"
  aws-services: Amazon Redshift
---

# Redshift Performance Scan
[四阶段工作流指令...]
```

`description` 字段是 AI 助手决定是否激活技能的关键——当用户请求匹配描述语义时，助手自动加载完整指令。

### 4.2 项目目录结构

```
.
├── .agents/skills/redshift-scan/SKILL.md   # 跨平台标准入口
├── .amazonq/rules/redshift-scan.md         # Amazon Q Developer 规则
├── .claude/skills/redshift-scan/
│   ├── SKILL.md                            # 含完整 frontmatter
│   └── references/                         # 规范知识库（单一数据源）
│       ├── diagnostic-queries.md           # 42 条 SQL
│       ├── rules-*.md                      # 51 条规则
│       └── report-template.md             # 输出模板
├── .kiro/
│   ├── steering.md                         # 项目级上下文
│   └── skills/redshift-scan/skill.md       # Kiro 技能入口
└── .mcp.json                               # 共享 MCP 配置
```

**关键设计**：规则和查询知识仅在 `references/` 目录维护一份，各平台的技能文件通过路径引用，避免内容漂移。

### 4.3 在 Kiro 中使用

Kiro 通过 `.kiro/skills/` 目录发现技能，结合 `.kiro/steering.md` 提供项目持久上下文。使用时只需自然语言描述意图：

> "扫描我的 Redshift 集群 `chemexpress-prod`，数据库 `analytics`，找出性能问题并给出优先级建议。"

Kiro 自动匹配技能描述、加载完整指令、通过 MCP 执行诊断查询、评估规则、生成报告。

### 4.4 在 Amazon Q Developer 中使用

Amazon Q Developer 通过 `.amazonq/rules/` 获取项目上下文，通过 MCP 配置连接 Redshift 服务器。在 IDE 聊天中：

> "@workspace 对集群 `chemexpress-prod` 执行 Redshift 性能巡检"

---

## 五、核心实现

### 5.1 MCP Server 配置

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

认证采用 IAM 临时凭证（`GetClusterCredentialsWithIAM`），无需存储数据库密码。

### 5.2 诊断查询（42 条 SQL）

| 类别 | 查询数 | 覆盖内容 | 数据来源 |
|------|--------|----------|----------|
| A - 集群配置 | 4 | Slice 拓扑、磁盘使用、参数设置 | stv_slices, stv_node_storage_capacity |
| B - 表设计 | 8 | 分布策略、排序键、编码、倾斜 | svv_table_info, pg_table_def |
| C - 查询性能 | 7 | 慢查询、告警、磁盘溢出、执行倾斜 | stl_query, stl_alert_event_log |
| D - 负载管理 | 4 | WLM 配置、吞吐量、QMR 违规 | stv_wm_service_class_config |
| E - 维护健康 | 5 | VACUUM 候选、统计信息、自动优化 | svl_auto_worker_action |
| F - 数据加载 | 3 | COPY 性能、加载错误 | stl_load_commits, stl_load_errors |
| G - Advisor | 1 | 内置建议 | svv_redshift_advisor |
| H - 增强诊断 | 10 | 块倾斜、谓词列、碎片化、QMR 候选 | 改编自 awslabs/amazon-redshift-utils |

**设计原则**：
- 全部 SELECT-ONLY，MCP Server 层面强制只读
- 内置 LIMIT 子句防止上下文溢出
- 容错执行：单条失败不阻断整体流程

### 5.3 四阶段工作流

```
Phase 1: Discovery    → 验证集群存在、获取拓扑
Phase 2: Collection   → 执行 42 条诊断 SQL
Phase 3: Evaluation   → 对照 51 条规则逐一评估
Phase 4: Report       → 生成优先级报告（Markdown）
```

---

## 六、规则体系

### 6.1 规则结构（示例）

```markdown
## TD-02: High Distribution Skew

**Severity**: HIGH
**Trigger Condition**: skew_rows > 4.0 AND size > 100MB
**Diagnostic Queries**: B6

**Observation Template**:
> Table `{schema}.{table}` has row skew ratio of {skew_rows}x...

**Recommendation**: Choose a column with higher cardinality...
**Remediation SQL**: ALTER TABLE ... ALTER DISTSTYLE KEY DISTKEY (...);
**Documentation Source**: https://docs.aws.amazon.com/redshift/latest/dg/...
```

### 6.2 规则分布（51 条）

| 类别 | 规则数 | 严重度分布 |
|------|--------|-----------|
| 表设计 | 17 | 4 HIGH, 7 MEDIUM, 6 LOW |
| 查询性能 | 14 | 1 CRITICAL, 6 HIGH, 5 MEDIUM, 2 LOW |
| 负载管理 | 6 | 1 HIGH, 4 MEDIUM, 1 LOW |
| 维护 | 7 | 1 CRITICAL, 2 HIGH, 3 MEDIUM, 1 LOW |
| 集群配置 | 3 | 1 CRITICAL, 1 MEDIUM, 1 LOW |
| 数据加载 | 4 | 1 HIGH, 3 MEDIUM |

### 6.3 规则来源

- **AWS 官方文档**：[Redshift Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/best-practices.html)、[Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/welcome.html)
- **awslabs/amazon-redshift-utils**（2800+ stars，Apache 2.0）：`perf_alert.sql`、`table_inspector.sql`、`predicate_columns.sql`、`wlm_qmr_rule_candidates.sql` 等

---

## 七、效果展示

### 7.1 典型输出（节选）

```markdown
# Redshift Performance Scan Report

**Cluster**: chemexpress-prod | **Database**: analytics
**Node Type**: ra3.xlplus x 4 | **Disk Usage**: 62%

## Executive Summary
**Overall Health**: Needs Attention
**Total Findings**: 18 (Critical: 0, High: 5, Medium: 8, Low: 5)

## High Severity Findings

### [TD-02] High Distribution Skew
**Affected**: orders (12.3x), shipment_details (8.7x), inventory_snapshot (6.2x)
**Remediation**: ALTER TABLE public.orders ALTER DISTSTYLE KEY DISTKEY (customer_id);
```

### 7.2 执行耗时

| 阶段 | 耗时 | 对比人工 |
|------|------|---------|
| Discovery | ~5s | — |
| Collection | 2-4 min | — |
| Evaluation | ~10s | — |
| Report | ~15s | — |
| **合计** | **3-5 分钟** | **人工：4-8 小时** |

---

## 八、安全与只读保障

| 层级 | 控制措施 |
|------|---------|
| IAM | 仅授予 `redshift-data:Execute/Describe/GetResult` + `GetClusterCredentialsWithIAM` |
| MCP Server | awslabs-redshift-mcp-server 强制只读 |
| SQL | 全部 SELECT，无 DDL/DML |
| 输出 | 修复 SQL 仅作为文本建议，不自动执行 |
| 凭证 | IAM 临时凭证，不存储长期密码 |

---

## 九、总结与下一步

本方案展示了 **Agent Skills 开放标准 + MCP 协议** 在数据库运维领域的实用模式：

1. **一次编写，跨平台运行**：遵循 Agent Skills 标准，同一技能在 Kiro、Amazon Q Developer、Claude Code 等工具中均可使用
2. **知识编码而非代码编码**：51 条规则以 Markdown 描述，降低维护门槛
3. **MCP 作为标准化接入层**：通过 `awslabs-redshift-mcp-server` 实现安全数据访问，无需自建 API
4. **aws-samples 质量标准**：MIT-0 License、标准化 README、CONTRIBUTING.md、CODE_OF_CONDUCT.md

---

### 下一步行动

- **快速开始**：克隆项目，配置 AWS 凭证，在你使用的 AI 助手中触发巡检
- **自定义规则**：在 `references/rules-*.md` 中添加适合你业务的阈值
- **贡献社区**：参考 CONTRIBUTING.md 提交新规则或诊断查询

---

### 参考资源

- [Agent Skills 开放标准](https://agentskills.io) — 跨平台 AI 技能规范
- [awslabs/mcp](https://github.com/awslabs/mcp) — AWS 官方 MCP Server 集合（含 Redshift）
- [awslabs/amazon-redshift-utils](https://github.com/awslabs/amazon-redshift-utils) — 社区诊断脚本（Apache 2.0）
- [Amazon Redshift Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/best-practices.html)
- [Kiro](https://kiro.dev) — AWS AI 编程助手
- [Amazon Q Developer](https://aws.amazon.com/q/developer/) — AI 开发者工具

### 相关产品

- [Amazon Redshift](https://aws.amazon.com/redshift/)
- [Kiro](https://kiro.dev)
- [Amazon Q Developer](https://aws.amazon.com/q/developer/)
- [AWS Agent Toolkit](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/)

---

*本文代码和规则以 MIT-0 许可开源，欢迎根据自身集群特征扩展和改进。*
