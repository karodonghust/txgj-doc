# Tasks 现状分析

返回[现状分析总览](README.md)。业务依据：[Tasks](../business-requirements/40-tasks.zh-CN.md)。

## 结论

| BR | 状态 | 当前实现摘要 |
| --- | --- | --- |
| `BR-035` | `缺失` | 只有人工通用 Task；未找到 SchoolTarget 进入 preparing 时自动创建申请 Task 的实现 |
| `BR-036` | `缺失` | Contractor 脱敏工作区契约存在，但没有“需要面试时自动创建并指派辅助 Task”的业务触发 |
| `BR-037` | `冲突` | 接受、拒绝、重派、取消和完成基础存在，但旧模型仍有 `approved`、Founder 审批和独立 `overdue` 状态 |

## 已有基础

- 新 Task 已从 `assigned` 开始。
- Assignee 可以接受或带原因拒绝；Primary Advisor/Founder 可以重派或取消；Assignment 历史由数据库保留。
- 当前 Task workspace 提供创建、列表、详情和状态转换 API；非 production-aws 模式可组合 PostgreSQL repository。
- `due_at` 已进入 migration 033。
- Contractor DTO 已限制字段并有失效测试资产，但 `contractor-workspace-runtime.ts` 仍固定不可用。
- Task 修改使用 expected version、幂等记录、审计和 outbox。

## 关键差异

| 优先级 | 当前代码 | 与业务基线的差异 | 建议动作 |
| --- | --- | --- | --- |
| `P0` | 没有 application preparation/submission Task 类型或自动触发 | Guardian 确认后不会产生逐校申请工作 | 以 SchoolTarget + application round 建立幂等键，自动创建一条“准备并提交申请”Task |
| `P0` | 没有 interview support Task 触发 | 学校要求面试后不会产生辅助工作 | submitted -> interview 时创建；默认 Primary Advisor，可改派合规 Advisor/Contractor |
| `P0` | `TASK_STATES` 含 `approved`，Release 1 policy 有 `completed -> approved` 且只允许 Founder | 已明确不需要逐条 Founder 验收 | 删除活跃审批转换；完成记录满足后由 Assignee 直接完成 |
| `P0` | `TASK_STATES` 含 `overdue` | overdue 应为 `due_at` 计算标记，不是业务状态 | 从状态机移除，查询时计算 `is_overdue`；完成/取消后停止提醒 |
| `P0` | `TASK_STATES` 还含 `created`、`reassigned` 等过渡状态 | 可能把动作事实当成长期状态 | 状态模型收敛为业务可停留状态；重派保留为 assignment/history 事件 |
| `P0` | 申请 Task 无提交字段和凭证校验 | Task 可以在未记录提交事实时完成 | 完成命令校验提交时间、渠道、提交人、清单状态及参考号或替代凭证 |
| `P1` | Contractor workspace runtime 不可用 | 面试 Contractor 无法实际进入单任务工作区 | 在授权、脱敏 DTO 和即时失效规则完成后接通运行时 |

## 自动 Task 边界

Release 1 只自动创建两类：

1. 每校“准备并提交申请”Task。
2. 学校明确要求面试时的“面试辅助”Task。

背景收集、Founder 审批、Guardian 确认、等待学校结果、offer 决定和结案都不自动创建 Task。Primary Advisor 仍可按需要创建普通临时 Task。

## 完成标准

- 同一 SchoolTarget、同一申请轮次不会重复创建申请 Task。
- 面试 Task 只在需要面试时创建，Contractor 只看到允许的脱敏字段。
- Task 完成后即为 completed，不再等待 Founder approved。
- overdue 只由时间计算，暂停不顺延 due_at，提醒持续到完成或取消。
