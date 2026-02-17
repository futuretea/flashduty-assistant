---
name: flashduty-incident-analytics
description: This skill should be used when the user asks to "统计故障", "统计事件", "incident statistics", "故障分析", "MTTR", "MTTA", "故障报告", "incident trend", "故障趋势", "analyze incidents", "incident metrics", "SLO report", or discusses incident statistics, operational metrics, and reliability analysis (事件统计、运营指标和可靠性分析).
version: 3.0.0
---

# FlashDuty 事件分析

本技能帮助你分析事件数据并生成统计信息。对于复杂分析，它委托给具有独立上下文的专用 Sub-Agent。

## 可用 Sub-Agent

### SRE 聚焦分析
当用户查询与可靠性工程相关时，可与以下 SRE Agent 并行启动：

- `flashduty-error-budget-tracker`：错误预算追踪、SLO 合规性分析
- `flashduty-toil-analyzer`：琐事识别、自动化机会

**与统计分析并行启动**：
```
用户: "分析本周故障并生成报告"
→ 并行：
   Sub-Agent 1: flashduty-stats-collector (基础指标)
   Sub-Agent 2: flashduty-error-budget-tracker (可靠性)
   Sub-Agent 3: flashduty-toil-analyzer (效率)
→ 生成综合 SRE 报告
```

### 1. `flashduty-stats-collector`
**用于**: 聚合统计、MTTR/MTTA 计算、趋势分析

**何时调用**：
- 用户要求 "本周故障统计"
- 用户要求 "MTTR趋势"
- 用户要求 "故障率"
- 任何带有时间范围和指标的请求

**传递参数**：
```json
{
  "time_range": "7d",
  "channel_ids": [可选],
  "team_ids": [可选],
  "severities": [可选],
  "aggregate_unit": "day" | "week" | "month",
  "dimensions": ["severity", "channel", "trend"],
  "query_type": "quick_stats" | "trend_analysis" | "detailed_list"
}
```

> **注意**：优先使用 `time_range`（如 `'7d'`、`'30d'`、`'last_week'`）而非手动计算 `start_time`/`end_time`。趋势分析时使用 `aggregate_unit` 参数。

### 2. `flashduty-incident-analyzer`
**用于**: 复杂过滤、基于标签的分组、自定义查询

**何时调用**：
- 用户要求 "按namespace统计"
- 用户要求 "production环境的故障"
- 需要多条件过滤

## 并行执行模式

### 模式 1: 多 Channel 对比
```
用户: "对比各协作空间的故障情况"

→ 并行启动：
  Sub-Agent 1: Channel A 的统计
  Sub-Agent 2: Channel B 的统计
  Sub-Agent 3: Channel C 的统计
  ...
→ 聚合结果并展示对比
```

### 模式 2: 多时间段趋势
```
用户: "过去3个月的趋势"

→ 并行启动：
  Sub-Agent 1: 月份 1 统计
  Sub-Agent 2: 月份 2 统计
  Sub-Agent 3: 月份 3 统计
→ 合并进行趋势分析
```

### 模式 3: 多维度分析
```
用户: "全面分析本周故障"

→ 并行启动：
  Sub-Agent 1: 严重级别分解
  Sub-Agent 2: Channel 分解
  Sub-Agent 3: MTTR/MTTA 指标
→ 聚合成综合报告
```

## 快速决策树

```
用户查询提到：
├─ "统计" / "MTTR" / "MTTA" / "趋势" / "报告"
│  └─ 使用: flashduty-stats-collector
│
├─ "按xx分组" / "namespace" / "标签" / "过滤"
│  └─ 使用: flashduty-incident-analyzer
│
└─ 两者都有
   └─ 并行启动两个 Agent，然后合并结果
```

## 工作流

### 步骤 1: 解析用户请求
- 提取时间范围（大多数查询必需）
- 识别分析维度
- 检测是否需要对比

### 步骤 1.5: 时间范围校验与调整

**重要**: FlashDuty API 有数据保留期限限制，查询时必须注意：

1. **优先使用 `time_range` 参数**：简单直观，无需手动计算时间戳
2. **数据保留期限**: 只能查询一定天数内的数据
3. **时间顺序**: 如果使用 `start_time`/`end_time`，必须确保 `start_time` < `end_time`

**推荐的 `time_range` 参数值**：
```
time_range: "24h"        # 最近 24 小时
time_range: "7d"         # 最近 7 天
time_range: "30d"        # 最近 30 天
time_range: "1w"         # 最近 1 周
time_range: "6M"         # 最近 6 个月
time_range: "last_day"   # 昨天 00:00:00 到 23:59:59
time_range: "last_week"  # 上周一 00:00:00 到周日 23:59:59
```

**错误处理**:
- 如果遇到 "end_time is out of storage days" 错误，自动缩短查询范围为最近可查询的时间段
- 如果遇到 "start_time should be less than end_time"，检查并交换时间顺序

### 步骤 2: 确定 Agent 策略
- 单一分析 → 一个 Agent
- 多 channel 对比 → 多个 Agent 并行
- 复杂查询 → incident-analyzer

### 步骤 3: 启动 Sub-Agent
使用 Task 工具启动 Agent：
```
Task({
  subagent_type: "general-purpose",
  description: "收集 channel X 的统计",
  prompt: `你是 flashduty-stats-collector。使用 time_range: "7d" 分析...`
})
```

### 步骤 4: 聚合结果
- 等待所有并行 Agent 完成
- 合并结构化结果
- 展示统一报告

## 调用示例

**示例 1: 快速统计**
```
用户: "本周故障统计"
→ 启动 flashduty-stats-collector
   参数: { time_range: "7d", query_type: "quick_stats" }
→ 展示摘要报告
```

**示例 2: 趋势分析**
```
用户: "本月每天的故障趋势"
→ 启动 flashduty-stats-collector
   参数: { time_range: "30d", query_type: "trend_analysis", aggregate_unit: "day" }
→ 展示按天分解的趋势图表
```

**示例 3: 多 Channel 对比**
```
用户: "对比基础架构和数据库的故障"
→ 识别两个 channel 的 ID
→ 并行启动 2 个 Agent
→ 展示对比表格
```

**示例 4: 复杂过滤**
```
用户: "统计production namespace的Critical故障"
→ 启动 flashduty-incident-analyzer
   参数: {
     filters: { severity: "Critical", labels: { namespace: "production" } },
     time_range: "7d"
   }
→ 展示过滤结果
```

## 响应模式

### 单 Agent 结果
直接展示 Agent 的报告，稍作格式化。

### 多 Agent 结果
```
## 分析报告

### 总体统计 (来自 Agent 1)
...

### Channel 分解 (来自 Agent 2)
...

### 趋势 (来自 Agent 3)
...

### 摘要
合并各 Agent 结果，给出整体结论...
```
