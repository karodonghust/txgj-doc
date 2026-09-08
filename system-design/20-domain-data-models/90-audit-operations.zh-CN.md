# Audit 与 Operations 领域模型与数据设计

状态：`approved`  
确认依据：项目负责人于 2026-08-25 接受 AuditEvent、OutboxMessage、OperationsProjection、OperationalAlert、原子审计、重放和租户/平台边界设计  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[领域模型与数据设计索引](README.md)。

业务依据：`BR-070`、`BR-071`，并引用所有产生状态转换、授权、批准、确认、导出、删除、恢复和高风险读取的 `BR-*`。  
模块依据：[Audit 与 Operations 模块契约](../10-module-contracts/90-audit-operations.zh-CN.md)。  
现状依据：[Notifications、Audit 与隐私现状分析](../../current-state-analysis/70-notifications-audit.zh-CN.md)。  
代码参考：`modules/audit/**`、`modules/operations/**`、migration `007`。

## 1. 先看结论

Audit 与 Operations Release 1 使用 4 个核心对象：

| 模块 | 对象 | 目标表 | 负责的事实 |
| --- | --- | --- | --- |
| Audit | `AuditEvent` | `audit_events` | 谁在什么时候对什么做了什么，结果如何 |
| Audit | `OutboxMessage` | `audit_outbox` | 与业务提交绑定的异步 effect、lease、重试和终态 |
| Operations | `OperationsProjection` | `operations_projections` | 可重建的风险/健康投影和 source checkpoint |
| Operations | `OperationalAlert` | `operations_operational_alerts` | 运行、隐私、区域和投影风险告警及处理状态 |

一句话区分：

```text
Audit = 永久证据，不能改、不能删
Operations = 可重建投影，丢失后可以重算
```

不建立以下实体：

- `BeforeAfterSnapshot`：只保存受控字段 hash。
- `Dashboard`、`Search`、`Metric`、`UnreadCount`：都是投影的视图。
- `AlertDefinition`：固定在版本化代码目录中。
- `ProjectionCheckpoint`：作为 Projection 的技术字段，不单独建业务表。
- `TelemetryEvent`、Log、Trace：进入受控 telemetry sink，不是 AuditEvent。

## 2. 原子提交流程

```text
业务命令
  -> 检查当前授权和 expected version
  -> 写业务模块权威事实
  -> 写必需 AuditEvent
  -> 如有异步副作用，再写 OutboxMessage
  -> 写 Shared IdempotencyRecord
  -> 同一 PostgreSQL 事务提交
```

- 必需 AuditEvent 写失败，业务修改必须回滚。
- 没有异步副作用时，只写 AuditEvent，不制造空 Outbox。
- 高风险读取必须先成功记录 AuditEvent，再返回资料或 capability。
- 普通 telemetry 失败不能回滚已经成功的业务写入。

## 3. AuditEvent 表

表：`audit_events`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | AuditEvent opaque UUID 主键 |
| `organization_id` | 是 | 已有 | 租户业务审计边界 |
| `actor_user_id` | 条件必填 | 已有 | `actor_kind=user` 时保存实际 User UUID；system/worker 为空 |
| `actor_kind` | 是 | 已有 | `user`、`system` 或 `worker` |
| `event_type` | 是 | 已有 | 业务事件类型 |
| `event_version` | 是 | 已有 | 事件契约版本 |
| `action` | 是 | 已有 | `create`、`update`、`approve`、`read`、`delete` 等受控动作 |
| `resource_type` | 是 | 已有 | 被操作对象类型 |
| `resource_id` | 是 | 已有 | 被操作对象 opaque UUID |
| `outcome` | 是 | 已有 | `succeeded`、`denied` 或 `failed` |
| `request_id` | 是 | 已有 | 请求或 Worker run 的安全 opaque ID |
| `occurred_at` | 是 | 已有 | 业务动作发生时间，UTC |
| `before_hash_sha256` | 否 | 已有 | 变更前受控字段快照 hash，不保存正文 |
| `after_hash_sha256` | 否 | 已有 | 变更后受控字段快照 hash，不保存正文 |
| `causation_id` | 否 | 新增 | 导致本事件的上游 Audit/Outbox opaque ID |
| `correlation_id` | 否 | 新增 | 同一业务流程的关联 opaque ID |
| `metadata` | 是 | 已有 | 严格 allowlist 的标量 metadata：版本、reason code、effect、状态、重试次数等 |

关键约束：

- AuditEvent 只允许 INSERT；禁止 UPDATE 和 DELETE。
- `actor_kind=user` 必须有 `actor_user_id`；不能只写“Founder”或“管理员”。
- metadata 不保存姓名、邮箱、电话、文件内容、URL、Cookie、token、secret 或自由文字。
- before/after 只保存 hash、版本和受控状态 code。
- 审计查询只能按 Organization、时间、事件/资源类型和 opaque ID 查询，不能按客户姓名搜索。

## 4. OutboxMessage 表

表：`audit_outbox`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | Outbox UUID 主键 |
| `audit_event_id` | 是 | 已有 | 对应同事务写入的 AuditEvent |
| `organization_id` | 是 | 已有 | 租户边界；平台 effect 使用独立路径 |
| `aggregate_type` | 是 | 已有 | 业务聚合类型 |
| `aggregate_id` | 是 | 已有 | 业务聚合 opaque UUID |
| `event_type` | 是 | 已有 | 要发布的 effect 类型 |
| `event_version` | 是 | 已有 | effect 契约版本 |
| `idempotency_key` | 是 | 已有 | Outbox 唯一幂等身份 |
| `request_id` | 是 | 已有 | 原始请求/Worker run ID |
| `payload` | 是 | 已有 | 只含 opaque ID、状态、版本、effect code 和受控标量 |
| `status` | 是 | 已有 | `pending`、`processing`、`delivered`、`dead_letter` |
| `attempt_count` | 是 | 已有 | 投递尝试次数，最多 3 次 |
| `available_at` | 是 | 已有 | 最早可领取时间 |
| `leased_until` | 否 | 已有 | Worker lease 结束时间 |
| `lease_version` | 是 | 已有 | lease 乐观锁版本 |
| `delivered_at` | 条件必填 | 已有 | delivered 终态时间 |
| `dead_lettered_at` | 条件必填 | 已有 | dead_letter 终态时间 |
| `last_error_code` | 否 | 已有 | 最后一次受控技术错误码 |
| `replay_of_outbox_id` | 条件必填 | 新增 | 人工重放时指向原 dead-letter Outbox；原记录不改写 |
| `replay_reason_code` | 条件必填 | 新增 | 人工重放的受控原因 |
| `record_version` | 是 | 已有 | Outbox 状态/lease 乐观锁 |
| `created_at` | 是 | 已有 | 创建时间 |
| `updated_at` | 是 | 已有 | 最近一次状态变化时间 |

关键约束：

- Outbox identity、payload 和幂等键不可改写。
- 状态只能按 `pending -> processing -> delivered/dead_letter` 推进；Worker 崩溃可回到 pending。
- delivered 和 dead_letter 是终态，不能改回成功或 pending。
- 人工重放创建新的 Outbox 记录，不能把旧 dead-letter 改成 delivered。
- `payload` 只能保存 allowlist 字段，不保存业务正文和 PII。

## 5. OperationsProjection 表

表：`operations_projections`

这是可重建的技术投影，不是业务事实表。`content_json` 必须按 `projection_kind + projection_version` 使用严格 schema，不能作为万能业务表。

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 新增 | Projection UUID 主键 |
| `scope_kind` | 是 | 新增 | `tenant` 或 `platform`；决定读取边界 |
| `organization_id` | 条件必填 | 新增 | tenant projection 必填；platform projection 为空 |
| `projection_kind` | 是 | 新增 | `case_dashboard`、`technical_health`、`notification_health`、`document_health` 等固定类型 |
| `source_schema_version` | 是 | 新增 | 上游事实来源 schema 版本 |
| `projection_version` | 是 | 新增 | 当前投影结构版本 |
| `source_snapshot_id` | 是 | 新增 | 本次重建使用的 source snapshot opaque ID |
| `source_captured_at` | 是 | 新增 | source snapshot 采集时间 |
| `last_source_event_id` | 否 | 新增 | 已应用的最后一个事件 opaque ID |
| `last_source_event_version` | 否 | 新增 | 已应用的最后一个事件版本 |
| `content_json` | 是 | 新增 | 严格版本化的投影内容；只保存 opaque ID、状态 code、计数、截止时间和健康标记 |
| `content_hash_sha256` | 是 | 新增 | 投影内容 hash，用于重建一致性校验 |
| `freshness_state` | 是 | 新增 | `fresh`、`stale` 或 `rebuilding` |
| `generated_at` | 是 | 新增 | 投影生成时间 |
| `record_version` | 是 | 新增 | 投影版本乐观锁 |
| `created_at` | 是 | 新增 | 创建时间 |
| `updated_at` | 是 | 新增 | 最近一次投影更新时间 |

投影不得保存：

- Student/Guardian 姓名、邮箱、电话。
- Assessment answer、文件名、文件内容。
- 自由文字 next action 或内部备注。

如果页面需要姓名或 Case number，先从投影取得 opaque ID，再向 CRM/Cases 权威查询当前允许字段。

## 6. OperationalAlert 表

表：`operations_operational_alerts`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 新增 | 告警记录 UUID |
| `alert_id` | 是 | 新增 | 固定 Alert Catalogue ID，例如 `scan.stuck` |
| `catalogue_version` | 是 | 新增 | 告警目录版本 |
| `occurrence_id` | 是 | 新增 | 一次告警发生的 opaque ID |
| `scope_kind` | 是 | 新增 | `tenant` 或 `platform` |
| `organization_id` | 条件必填 | 新增 | tenant 告警必填；platform 告警为空 |
| `detected_at` | 是 | 新增 | 检测时间 |
| `metric_name` | 是 | 新增 | 受控指标名称 |
| `observed_value` | 是 | 新增 | 检测到的数值 |
| `threshold_value` | 是 | 新增 | 触发阈值 |
| `window_seconds` | 是 | 新增 | 观察窗口 |
| `severity` | 是 | 新增 | `warning`、`high`、`critical` |
| `state` | 是 | 新增 | `firing`、`acknowledged`、`mitigated`、`closed`、`needs_human` |
| `acknowledged_by_user_id` | 条件必填 | 新增 | 接手告警的内部 User |
| `acknowledged_at` | 条件必填 | 新增 | 接手时间 |
| `mitigated_by_user_id` | 条件必填 | 新增 | 完成缓解动作的内部 User |
| `mitigated_at` | 条件必填 | 新增 | 缓解时间 |
| `closed_by_user_id` | 条件必填 | 新增 | 关闭告警的内部 User |
| `closed_at` | 条件必填 | 新增 | 关闭时间 |
| `reason_code` | 条件必填 | 新增 | 受控处理原因；不保存自由文字 |
| `runbook_ref` | 是 | 新增 | 固定 runbook 路径或 code |
| `record_version` | 是 | 新增 | 告警状态乐观锁 |
| `created_at` | 是 | 新增 | 创建时间 |
| `updated_at` | 是 | 新增 | 最近一次状态变化时间 |

告警目录只保留：

- Identity/Access 风险。
- Document 扫描积压和 dead-letter。
- Audit/Outbox 积压和 dead-letter。
- Notifications 投递失败和 scheduler 落后。
- PII canary 和隐私违规。
- 香港区域/运行时故障。
- Projection lag 或 hash mismatch。

Release 1 不包含 budget、billing、import 或 backfill 告警。

## 7. 权限边界

| 访问者 | Audit 权限 | Operations 权限 |
| --- | --- | --- |
| Founder | 本组织租户审计脱敏摘要 | 组织 Case 风险、告警和 technical health |
| 当前 Primary Advisor | 不看全局 Audit | 只看当前授权 Case 的工作摘要 |
| Case Collaborator | 不直接看 Audit | 只看当前 scope 的最小投影 |
| Admin 基础角色 | 只看无客户内容的 technical health | 只看 technical health 和运行告警 |
| Contractor | 无 | 无 |
| Guardian、Student、Portal | 无 | 无 |
| Platform operator | 只能看独立 platform audit/telemetry | 只能看 platform technical health，不能看租户内容 |

投影、缓存和告警都不能作为授权来源；每次查询必须重新检查 Access 和业务模块当前关系。

## 8. 当前实现与目标差异

| 优先级 | 当前实现 | 目标处理 |
| --- | --- | --- |
| `P0` | Audit/Outbox 尚未覆盖全部已确认业务动作 | 每个状态转换、授权、批准、确认、导出、删除、恢复和高风险读取都产生必需 AuditEvent |
| `P0` | `MutationEffectBundle` 默认 Audit 与 Outbox 一对一 | 无异步 effect 时只写 Audit，不创建空 Outbox |
| `P0` | Operations 没有正式持久化 Projection/Alert 表 | 新增可重建 Projection 和 OperationalAlert 存储 |
| `P0` | Case dashboard projection 含姓名、Case number 和自由文字 | 投影只保存 opaque ID、状态 code、计数、截止时间和 hash |
| `P0` | tenant 与 platform 旧路径仍有混用风险 | 用 scope、schema、role 和查询入口分离两类数据 |
| `P1` | Outbox 没有明确 replay reference | 新增 `replay_of_outbox_id` 和受控 replay reason |
| `P1` | Alert catalogue 仍含 budget/billing 告警 | Release 1 移除预算、计费、导入和 backfill 告警 |
| `P1` | dashboard/telemetry runtime 尚未接通 | 接通正式 adapter；未配置时 fail closed 或明确 stale |

本次只冻结目标设计，不修改产品代码、migration 或数据库。

## 9. 设计决策

| ID | 决策 |
| --- | --- |
| `SD-AUD-001` | AuditEvent 是不可改写证据，禁止物理删除 |
| `SD-AUD-002` | 必需 AuditEvent 与业务修改同事务；没有异步 effect 时不创建空 Outbox |
| `SD-AUD-003` | Outbox 终态不可逆；人工重放创建新记录并引用旧记录 |
| `SD-OPS-001` | OperationsProjection 可重建，不能作为授权或业务状态机依据 |
| `SD-OPS-002` | Projection 不保存客户 PII、Assessment answer、文件内容或自由文字 |
| `SD-OPS-003` | OperationalAlert 只表达技术/隐私/区域/投影风险，不自动修改业务事实 |
| `SD-OPS-004` | tenant 与 platform 的审计、投影、查询权限和数据路径分离 |

## 10. 本模块确认结果

项目负责人已一次确认下面 8 点：

1. 接受 AuditEvent、OutboxMessage、OperationsProjection、OperationalAlert 四个核心对象。
2. 接受 Audit 是永久证据，Operations 是可重建投影。
3. 接受必需审计与业务修改同事务；没有异步 effect 时不创建空 Outbox。
4. 接受 Outbox dead-letter 不改写，人工重放创建新记录。
5. 接受 Projection 不保存客户姓名、联系方式、Assessment answer、文件内容或自由文字。
6. 接受 Founder 查看租户审计；Admin 只看 technical health；平台人员不能读取租户内容。
7. 接受 Release 1 移除 budget、billing、import 和 backfill 告警。
8. 接受 Operations 不能修改 Case、Task、Document 或其他业务权威表。

Audit 与 Operations 已标记为 `approved`，下一步进入 External Portal。
