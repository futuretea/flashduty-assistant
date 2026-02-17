# FlashDuty Assistant for Claude Code

一套用于 Claude Code 的 FlashDuty 集成技能，采用 **Sub-Agent + Skill** 架构，支持并行处理和独立上下文。

## 架构特点

```
┌─────────────────────────────────────────────────────────────┐
│  Skill Layer (触发器)                                         │
│  - 识别用户意图                                               │
│  - 决定调用哪个 Sub-Agent                                    │
│  - 汇总结果                                                   │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Sub-Agent 1 │   │  Sub-Agent 2 │   │  Sub-Agent N │
│  (独立上下文) │   │  (独立上下文) │   │  (独立上下文) │
└──────────────┘   └──────────────┘   └──────────────┘
```

### 核心优势

1. **独立上下文** - 每个 Sub-Agent 有自己的对话历史，避免上下文污染
2. **并行执行** - 同时分析多个事件、多个时间段、多个维度
3. **职责分离** - Skill 负责"何时触发"，Agent 负责"如何执行"
4. **性能提升** - 并行数据获取，减少用户等待时间

## 安装

### 1. 添加插件市场

```bash
/plugin marketplace add futuretea/flashduty-assistant
```

### 2. 安装插件

```bash
/plugin install flashduty-assistant@flashduty-assistant
```

### 3. 验证安装

```bash
/plugin list
```

安装成功后，你应该能看到 `flashduty-assistant` 在已安装插件列表中。

## 包含的技能

### 1. Incident Management（事件管理）

**触发词**: "create incident", "acknowledge incident", "close incident", "query incidents"

**执行方式**:
- 简单操作（ack/resolve/comment）直接执行
- 复杂查询委托给 `flashduty-incident-analyzer`
- 批量操作并行处理
- **SRE 集成**: 支持 SLO 影响和错误预算追踪

### 2. Incident Diagnosis（事件诊断）

**触发词**: "diagnose incident", "investigate incident", "analyze incident", "find root cause"

**执行方式**:
- 委托给 `flashduty-diagnosis-engine`
- 引擎内部并行获取：事件详情 + 时间线 + 关联告警
- 多事件诊断可并行启动多个引擎
- **SRE 最佳实践**: 考虑四大黄金信号（延迟、流量、错误、饱和度）

### 3. Team Collaboration（团队协作）

**触发词**: "assign incident", "find team member", "query teams", "who is on call"

**执行方式**:
- 委托给 `flashduty-team-resolver`
- 支持并行查询多个团队值班状态
- **SRE 集成**: 清晰的升级路径，响应时间目标

### 4. Incident Analytics（故障统计与分析）

**触发词**: "统计故障", "incident statistics", "故障分析", "MTTR", "MTTA", "故障趋势"

**执行方式**:
- 委托给 `flashduty-stats-collector` 或 `flashduty-incident-analyzer`
- **多维度并行分析**: 同时统计 severity、channel、趋势
- **多时间段对比**: 并行获取各月数据
- **多协作空间对比**: 并行分析各 channel

### 5. SRE Practices（SRE 最佳实践）

**触发词**: "postmortem", "事后分析", "error budget", "错误预算", "SLO", "SLI", "toil", "琐事", "reliability", "可靠性"

**执行方式**:
- **事后分析**: `flashduty-postmortem-generator` 生成无责备分析报告
- **错误预算**: `flashduty-error-budget-tracker` 追踪 SLO 合规性
- **琐事分析**: `flashduty-toil-analyzer` 识别自动化机会
- **发布决策**: 基于错误预算状态建议是否发布

**SRE 核心功能**:
- 错误预算计算和燃烧率分析
- 无责备事后分析（5 Whys 根因分析）
- 琐事识别和自动化路线图
- 多服务 SLO 对比仪表板
- 发布安全评估（基于错误预算）

## 项目结构

```
flashduty-assistant/
├── .claude/
│   └── settings.local.json              # 工具权限配置
├── .claude-plugin/
│   ├── marketplace.json                 # 插件市场元数据
│   └── plugin.json                      # 插件元数据
├── agents/                              # Sub-Agent 定义
│   ├── stats-collector/AGENT.md         # 统计收集 Agent
│   ├── diagnosis-engine/AGENT.md        # 诊断引擎 Agent
│   ├── team-resolver/AGENT.md           # 团队解析 Agent
│   ├── incident-analyzer/AGENT.md       # 通用分析 Agent
│   ├── postmortem-generator/AGENT.md    # 事后分析 Agent (SRE)
│   ├── error-budget-tracker/AGENT.md    # 错误预算 Agent (SRE)
│   └── toil-analyzer/AGENT.md           # 琐事分析 Agent (SRE)
├── skills/                              # Skill 触发器
│   ├── incident-management/SKILL.md
│   ├── incident-diagnosis/SKILL.md
│   ├── incident-analytics/SKILL.md
│   ├── team-collaboration/SKILL.md
│   └── sre-practices/SKILL.md           # SRE 最佳实践
├── .gitignore
├── CLAUDE.md
├── LICENSE
└── README.md
```

## 使用示例

### 基础操作（直接执行）

```
# 创建事件
"创建一个严重级别的事件，标题是数据库连接失败"

# 确认事件
"确认事件 FD123456"

# 关闭事件
"关闭事件 FD123456，根因是连接池耗尽"
```

### 诊断分析（Sub-Agent 执行）

```
# 单事件诊断
"诊断事件 FD123456"
→ 启动 diagnosis-engine
→ 并行获取详情/时间线/告警
→ 生成诊断报告

# 多事件对比诊断
"对比事件 A、B、C 的根因"
→ 并行启动 3 个 diagnosis-engine
→ 每引擎分析一个事件
→ 汇总对比结果
```

### 统计分析（并行 Sub-Agent）

```
# 单维度统计
"统计本周故障"
→ 启动 stats-collector
→ 使用 time_range: "7d"
→ 返回统计数据

# 多维度并行分析
"全面分析本周故障"
→ 并行启动：
   Agent 1: Severity 分布分析 (time_range: "7d")
   Agent 2: Channel 分布分析 (time_range: "7d")
   Agent 3: MTTR/MTTA 计算 (time_range: "7d")
   Agent 4: 趋势分析 (time_range: "30d")
→ 汇总生成综合报告

# 多时间段对比
"过去3个月的趋势"
→ 并行启动：
   Agent 1: 月份1统计 (time_range: "30d")
   Agent 2: 月份2统计 (time_range: "30d")
   Agent 3: 月份3统计 (time_range: "30d")
→ 合并趋势数据
```

### 团队协作

```
# 查找成员
"查找名叫John的团队成员"
→ 启动 team-resolver

# 多团队值班查询
"哪些团队有人在值班？"
→ 并行查询各团队 on-call

# 事件指派建议
"事件 FD123456 应该指派给谁？"
→ 并行获取：事件详情 + 值班表
→ 推荐最佳指派对象
```

### SRE 最佳实践（新增）

```
# 事后分析（无责备文化）
"生成事件 FD123456 的事后分析报告"
- 启动 flashduty-postmortem-generator
→ 包含：时间线、5 Whys 根因分析、行动项、经验教训
→ 确保无责备语言，聚焦系统性改进

# 错误预算追踪
"查看支付服务的错误预算"
→ 启动 flashduty-error-budget-tracker
→ 返回：已消耗 %、燃烧率、发布建议
→ "✅ 安全发布" 或 "⚠️ 建议暂停功能发布"

# 多服务 SLO 对比
"对比所有核心服务的可靠性"
→ 并行启动 flashduty-error-budget-tracker × N
→ 生成 SLO 合规性仪表板
→ 标识风险服务（预算消耗 > 50%）

# 琐事分析
"分析我们团队的琐事工作量"
→ 启动 flashduty-toil-analyzer
→ 识别：告警噪音、重复性问题、手动操作
→ 返回：琐事评分、自动化路线图、ROI 分析

# 发布安全评估（基于错误预算）
"这周可以发布新版本吗？"
→ 检查相关服务错误预算
→ 分析近期稳定性趋势
→ "✅ 安全发布" / "⚠️ 减少发布范围" / "🛑 建议延迟"

# 月度 SRE 报告
"生成本月 SRE 运营报告"
→ 并行启动：
   error-budget-tracker（所有服务）
   toil-analyzer（运营效率）
   stats-collector（MTTR/MTTA 趋势）
→ 综合报告：可靠性、效率、性能、优先事项
```

## Sub-Agent 并行执行模式

### 模式 1: 多事件并行诊断

```javascript
User: "分析事件 A、B、C"
→ Parallel:
   Task({ agent: "diagnosis-engine", incident: "A" })
   Task({ agent: "diagnosis-engine", incident: "B" })
   Task({ agent: "diagnosis-engine", incident: "C" })
→ 汇总发现
```

### 模式 2: 多协作空间对比

```javascript
User: "对比基础架构和数据库的故障"
→ Parallel:
   Task({ agent: "stats-collector", channel: "基础架构", time_range: "7d" })
   Task({ agent: "stats-collector", channel: "数据库", time_range: "7d" })
→ 生成对比表格
```

### 模式 3: 多维度并行分析

```javascript
User: "全面分析故障"
→ Parallel:
   Task({ agent: "stats-collector", dimension: "severity", time_range: "7d" })
   Task({ agent: "stats-collector", dimension: "channel", time_range: "7d" })
   Task({ agent: "stats-collector", dimension: "trend", time_range: "30d" })
→ 综合报告
```

### 模式 4: SRE 事后分析

```javascript
User: "分析事件并生成事后报告"
→ Parallel:
   Task({ agent: "flashduty-diagnosis-engine", incident: "FD123" })
   Task({ agent: "flashduty-postmortem-generator", incident: "FD123" })
→ 诊断报告 + 无责备事后分析
```

### 模式 5: 多服务错误预算追踪

```javascript
User: "检查所有服务的 SLO"
→ Parallel:
   Task({ agent: "flashduty-error-budget-tracker", service: "api-gateway" })
   Task({ agent: "flashduty-error-budget-tracker", service: "payment" })
   Task({ agent: "flashduty-error-budget-tracker", service: "user-service" })
→ 综合 SLO 仪表板
```

### 模式 6: SRE 月度评审

```javascript
User: "月度 SRE 报告"
→ Parallel:
   Task({ agent: "flashduty-error-budget-tracker", scope: "all-services", time_range: "30d" })
   Task({ agent: "flashduty-toil-analyzer", time_range: "30d" })
   Task({ agent: "flashduty-stats-collector", metrics: ["MTTR", "MTTA"], time_range: "30d" })
→ 可靠性 + 效率 + 性能 综合报告
```

## 依赖

- Claude Code 插件系统
- FlashDuty MCP 服务器

### 时间参数

FlashDuty MCP 服务器的 `list_incidents`、`list_alerts`、`get_incident_stats` 工具支持两种时间参数方式：

### `time_range` 参数（推荐）

- **时长格式**：`'1h'`、`'24h'`、`'7d'`、`'30d'`、`'1w'`、`'6M'`（结束时间 = 当前时间）
- **命名范围**：`'last_day'`（昨天 00:00:00 到 23:59:59）、`'last_week'`（上周一到周日）

### `start_time` + `end_time`

使用 Unix 秒级时间戳，当 `time_range` 未设置时为必填。

### Token 优化：`brief` 参数

`list_incidents` 和 `list_alerts` 支持 `brief: true` 参数：
- 仅返回关键字段，大幅减少响应大小
- **推荐在初始查询时使用**，然后对重点项获取完整详情

### 趋势分析：`aggregate_unit` 参数

`get_incident_stats` 支持 `aggregate_unit` 参数：
- `day`：按天分解
- `week`：按周分解
- `month`：按月分解

## 开发

### 添加 Sub-Agent

1. 创建 `agents/<agent-name>/AGENT.md`
2. 定义 agent 名称、工具集、并行能力
3. 说明输入输出格式
4. 从 skill 中引用

**SRE Agent 模板**:
```markdown
---
name: flashduty-sre-agent-name
description: Specific SRE task this agent handles
tools: ["mcp__flashduty__xxx"]
parallel: true
---

# SRE Agent Title

## SRE Principles Applied
- Error Budgets
- Blameless Culture
- Automation First

## Input Format
Parameters specific to SRE analysis

## Output Format
Structured SRE report
```

### 添加 Skill

1. 创建 `skills/<skill-name>/SKILL.md`
2. 定义触发条件（description）
3. 说明何时调用哪个 sub-agent
4. 提供并行执行模式示例

### SRE 事后分析（新增）

当添加新功能时，考虑：
- 是否支持错误预算追踪？
- 是否有助于减少琐事（toil）？
- 是否促进无责备文化？
- 是否可自动化？

### Agent 文件格式

```markdown
---
name: flashduty-agent-name
description: When to use this agent
tools: ["mcp__flashduty__xxx"]
parallel: true
---

# Agent Title

## Responsibilities
What this agent does

## Input Format
Parameters it accepts

## Output Format
Structured response format

## Execution Strategy
How to parallelize operations
```

## License

MIT
