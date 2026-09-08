# Notifications、Audit 与隐私现状分析

返回[现状分析总览](README.md)。业务依据：[Notifications、Audit 与 Operations](../business-requirements/70-notifications-audit.zh-CN.md)。

## 结论

| BR | 状态 | 当前实现摘要 |
| --- | --- | --- |
| `BR-038` | `缺失` | 只有通用站内“有待办”投递骨架；触发事件、接收人、定时提醒和运行时远未覆盖已确认规则 |
| `BR-070` | `部分符合` | 追加式审计、outbox、expected version、幂等和组织/操作者上下文广泛存在，但并非所有业务动作已闭环 |
| `BR-071` | `冲突` | 香港边界与脱敏契约已有基础，但 CRM purge、Platform Billing 残留和部分旧/preview 路径仍违反新边界 |

## Notifications 已有基础

- 仅定义 `in_app` channel，没有业务 Email、SMS 或 WhatsApp channel。
- 通知固定为 `PENDING_ITEM / A pending item needs attention.`，不暴露业务对象名称。
- 投递采用 outbox、lease、最多 3 次尝试、delivery receipt、suppressed 和 dead-letter 模型。
- 完成投递前 repository 被要求重新检查接收人的当前权限。

## Notifications 关键差异

| 优先级 | 当前代码 | 与业务基线的差异 | 建议动作 |
| --- | --- | --- | --- |
| `P0` | eligible event 只有 `tasks.task_transitioned` 和 `cases.service_case_stage_transitioned` | 缺 Task 分配/重派/拒绝、选校审批、可结案等精确事件 | 建立业务 effect catalogue，并为每类 effect 明确接收人解析规则 |
| `P0` | 没有 3 天、1 天、每日逾期调度 | 截止时间提醒完全缺失 | 增加可重放 scheduler/worker，按 recipient + effect + business day 去重 |
| `P0` | 没有“所有目标终态且无未完成 Task”通知 | 全拒或全部结束后不会提醒新增学校/人工结案 | 由 Case 投影产生 readiness effect，通知 Primary Advisor 与 Founder |
| `P0` | `getInAppNotificationRuntime()` 固定 unavailable | 通知 worker 无法实际投递 | 接通 PostgreSQL repository 与 worker 入口，验证权限失效后的 suppress |
| `P1` | 可见文案固定英文 | 业务基线只限制最小泄露，但产品界面需一致 | 保持不泄露前提下接入 i18n；不要加入 Student/School 等名称 |

## Audit 与一致性已有基础

- `audit_events` 和 `audit_outbox` 是追加式结构，多数业务服务用 `buildAtomicMutationEffects` 一并提交。
- CRM、Case、Task、Document、School 等写入普遍带 `expectedRecordVersion` 和稳定冲突错误。
- `shared_idempotency_records` 与各模块 repository 支持同 key 重放、payload hash 冲突和 in-progress 结果。
- 审计 payload 使用 effect type、状态、版本、hash 或受控 reason code，已避免在通用构造器中直接放完整表单。
- Operations 已有 telemetry、alert catalogue 和可重建 Case dashboard projection 契约；dashboard runtime 尚未接通。

## Audit 与隐私关键差异

| 优先级 | 差异 | 建议动作 |
| --- | --- | --- |
| `P0` | 新业务动作尚不存在，因此 Founder 选校审批、Guardian 代录确认、offer 决定和结案也没有审计事实 | 每个新 command 与必需审计、outbox 在同一事务落库 |
| `P0` | CRM 迁移仍允许最终 purge，直接违反 BR-071 | 采用 CRM 纠正迁移，永久保留软删除记录 |
| `P0` | Platform Billing 平台审计和租户外数据库角色仍作为活跃 R1 结构存在 | 从 R1 入口和运行依赖中隔离，不让其成为租户业务授权来源 |
| `P1` | 许多生产 runtime 未配置，香港边界只存在于契约/基础设施源码 | 在 AWS 阶段单独验证 RDS、S3、日志、备份和临时数据的实际区域 |
| `P1` | legacy crawler、preview/mock 和兼容 API 仍并存 | 逐入口核对日志与响应字段，避免绕过统一审计和脱敏层 |
| `P1` | Operations dashboard runtime 不可用 | 接通只读投影；继续禁止用统计/缓存替代业务授权事实 |

## 完成标准

- 所有 BR-038 通知触发、接收人、频率和去重规则都有测试和可运行 worker。
- 每个高风险动作都能证明“业务事实与必需审计同事务成功或同事务失败”。
- 日志、通知、审计摘要和 telemetry 不含 PII、自由文字正文、token 或 secret。
- 香港数据边界使用部署证据验证，不以 Terraform 源码或本地测试代替。
