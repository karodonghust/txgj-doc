# Notifications 领域模型与数据设计

状态：`approved`  
确认依据：项目负责人于 2026-08-25 接受 Notifications 的两张核心表、站内渠道、九类 effect、接收人、去重和抑制规则  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[领域模型与数据设计索引](README.md)。

业务依据：`BR-038`，并引用 `BR-032`、`BR-033`、`BR-035` 至 `BR-039`、`BR-070`、`BR-071`。  
模块依据：[Notifications 模块契约](../10-module-contracts/80-notifications.zh-CN.md)。  
现状依据：[Notifications、Audit 与隐私现状分析](../../current-state-analysis/70-notifications-audit.zh-CN.md)。  
代码参考：`modules/notifications/**`、migration `007`。

## 1. 先看结论

Notifications Release 1 只保留两张核心表：

| 对象 | 目标表 | 负责的事实 |
| --- | --- | --- |
| `Notification` | `notifications_notifications` | 某个内部 User 的站内待办提醒、目标引用和已读状态 |
| `DeliveryReceipt` | `notifications_delivery_receipts` | 一次 effect 的接收人解析、投递、抑制、失败和重试结果 |

不建立以下表：

- `ReminderSchedule`：由 Tasks 的 `due_at`、状态和当前时间计算。
- `NotificationPreference`：Release 1 的必需站内提醒不能被个人关闭。
- Email、SMS、WhatsApp、Push delivery 表：Release 1 不发送外部业务消息。
- `NotificationTarget`：目标只是 Notification 内的固定类型 + opaque ID，不是独立授权实体。

## 2. 通知生命周期

```text
committed outbox effect
  -> Worker claim lease
  -> 按 effect 解析当前接收人
  -> 重新检查 User / Membership / RoleBinding / 业务关系
  -> Notification + DeliveryReceipt 同事务写入
```

用户可见状态只有：

```text
unread -> read
```

投递被抑制或最终失败时：

- 只写 `DeliveryReceipt`。
- 不创建用户可见 Notification。
- 不修改产生该 effect 的 Case、SchoolTarget、Task 或审批事实。

## 3. 业务 effect 目录

| Effect | 触发事实 | 接收人 |
| --- | --- | --- |
| `task_assigned` | Task 首次分配 | 当前实际 Assignee |
| `task_reassigned` | Task 产生新 Assignment | 新 Assignee |
| `task_rejected` | Assignee 拒绝当前 Assignment | 当前 Primary Advisor |
| `candidate_list_review_requested` | 候选名单提交 Founder 审批 | 全部 active Founder |
| `candidate_list_reviewed` | Founder 批准或驳回名单 | 当前 Primary Advisor |
| `task_due_in_3_days` | 距 due_at 3 天 | 实际 Assignee、当前 Primary Advisor |
| `task_due_in_1_day` | 距 due_at 1 天 | 实际 Assignee、当前 Primary Advisor |
| `task_overdue_daily` | 已逾期且未完成/取消 | 实际 Assignee、Primary Advisor、全部 active Founder |
| `case_closure_choice_required` | 所有学校终态且没有未完成 Task | 当前 Primary Advisor、全部 active Founder |

规则：

- Case 暂停不停止逾期计算和提醒。
- Founder 审批、Guardian 确认、等待学校结果、offer 决定和人工结案不额外创建 Task。
- 通知不是 Task；Notifications 不能为了跟进而自动创建 Task。
- 同一 User 同时具有多个接收角色时只收到一条。

## 4. Notification 表

表：`notifications_notifications`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | Notification opaque UUID 主键 |
| `organization_id` | 是 | 已有 | 所属 Organization 和 RLS 边界 |
| `recipient_user_id` | 是 | 已有 | 实际接收通知的内部 User；不能由客户端传入任意 User ID |
| `outbox_id` | 是 | 已有 | 对应已提交的 Audit Outbox effect |
| `effect_type` | 是 | 已有 | 业务 effect 类型，例如 `task_assigned` |
| `effect_idempotency_key` | 是 | 已有 | effect 稳定幂等身份；必须区分来源对象和业务日期/版本 |
| `target_kind` | 条件必填 | 新增 | `task`、`case`、`candidate_list_version` 等固定目标类型 |
| `target_id` | 条件必填 | 新增 | 目标对象 opaque UUID；不保存名称或自由 URL |
| `target_action` | 条件必填 | 新增 | 固定动作，例如 `work`、`review`、`choose_closure` |
| `channel` | 是 | 已有 | 固定为 `in_app` |
| `content_code` | 是 | 已有 | 固定为 `PENDING_ITEM`，界面通过 i18n 展示最小文案 |
| `status` | 是 | 已有，需收敛 | 目标只允许 `unread`、`read`；`suppressed` 不生成用户可见行 |
| `read_at` | 条件必填 | 新增 | 标记已读的时间；未读为空 |
| `record_version` | 是 | 已有 | 已读状态乐观锁 |
| `created_at` | 是 | 已有 | 通知创建时间 |
| `updated_at` | 是 | 已有 | 最近一次已读状态变化时间 |

关键约束：

- `(organization_id, recipient_user_id, effect_type, effect_idempotency_key)` 唯一；不同接收人不能被错误合并。
- `target_kind`、`target_id`、`target_action` 要么全部为空，要么全部为固定白名单值。
- Notification 文案来自 `content_code + i18n`，不接受生产者传入自由文字。
- 只有 recipient User 可以执行 `unread -> read`；不能删除、撤回、静音或物理清理通知。
- 点击目标时必须重新授权；Notification 本身不是业务资源权限。

## 5. DeliveryReceipt 表

表：`notifications_delivery_receipts`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | 投递回执 opaque UUID 主键 |
| `organization_id` | 是 | 已有 | 所属 Organization 和 RLS 边界 |
| `outbox_id` | 是 | 已有 | 对应原始 effect 的 Outbox 记录 |
| `recipient_user_id` | 是 | 新增 | 本次解析出的接收人；即使被抑制也要留证 |
| `notification_id` | 否 | 已有，需放宽 | delivered 时关联 Notification；suppressed/failed 可为空 |
| `effect_type` | 是 | 已有 | effect 类型 |
| `effect_idempotency_key` | 是 | 已有 | effect 幂等身份 |
| `outcome` | 是 | 已有，需纠正 | `delivered`、`suppressed` 或 `failed` |
| `attempt_count` | 是 | 已有 | 当前尝试次数，最多 3 次 |
| `failure_code` | 条件必填 | 已有 | 最终失败时的稳定技术错误码 |
| `suppression_code` | 条件必填 | 新增 | 权限失效、目标失效或 effect 不再适用的受控原因 |
| `record_version` | 是 | 已有 | 回执重试状态的乐观锁 |
| `created_at` | 是 | 已有 | 回执创建时间 |
| `updated_at` | 是 | 已有 | 回执重试/最终状态更新时间 |

关键约束：

- `(organization_id, recipient_user_id, effect_type, effect_idempotency_key)` 唯一。
- `delivered` 必须有 Notification；`suppressed` 和最终 `failed` 不要求有 Notification。
- `failure_code` 与 `suppression_code` 使用固定 code，不保存自由文字或业务正文。
- `attempt_count` 单调递增，最多 3 次；最终失败进入 dead letter 并通知 Operations。
- 回执 identity 不可改写；重复 Worker 投递返回原回执，不产生第二条业务事实。

## 6. 去重规则

```text
recipient_user_id
  + effect_type
  + source_opaque_id
  + effect_version 或 Asia/Hong_Kong business_date
```

实现上可以将以上内容编码为 `effect_idempotency_key`，但最终唯一约束必须包含 `recipient_user_id`。

- 即时 effect 使用 Assignment ID、名单版本 ID 等稳定来源。
- 定时 effect 使用 Task ID + 香港业务日期。
- 标记已读不会停止未来日期的提醒。
- Worker 重启、重复 Outbox 或并发 claim 都必须幂等。

## 7. 权限和数据边界

| 访问者 | 权限 |
| --- | --- |
| 内部 User | 只能查看自己的 Notification，并标记自己的通知已读 |
| Founder | 作为当前 active Founder 接收指定 effect；不拥有全部通知的特殊读取权限 |
| Primary Advisor / Assignee | 按当前 Case、Task、Assignment 关系接收通知 |
| Admin 基础角色 | 不因 Admin 身份自动获得客户业务通知 |
| Guardian、Student、Portal | 不接收内部 Notifications，也不能读取通知列表 |

接收人解析必须重新检查 User、Membership、RoleBinding 和当前业务关系。失去权限后，投递写 `suppressed`，不能通过旧通知进入业务对象。

## 8. 与其他表的关系

Notifications 不拥有以下事实：

| 内容 | Owner |
| --- | --- |
| Task 状态、Assignment、due_at | Tasks |
| 候选学校名单审批和 Case 结案选择 | Cases |
| 当前角色、Membership、RoleBinding | Access / Identity |
| Outbox effect | Audit / Shared |
| 幂等记录 | Shared |

Notifications 只消费这些模块发布的公开 effect，不直接写入它们的私有表。

## 9. 当前实现与目标差异

| 优先级 | 当前实现 | 目标处理 |
| --- | --- | --- |
| `P0` | 只有通用 Task/Case transition effect | 建立已确认的九类 effect 和接收人解析规则 |
| `P0` | 没有 3 天、1 天和每日逾期调度 | 增加可重放 reminder evaluator，按香港业务日期去重 |
| `P0` | 没有所有学校终态且无未完成 Task 的提醒 | Cases 发布 `case_closure_choice_required` effect |
| `P0` | 当前唯一键没有包含 recipient | 改为 recipient + effect + effect key，避免多接收人互相覆盖 |
| `P0` | suppressed 仍可能创建不可见 Notification | suppressed 只保留 DeliveryReceipt |
| `P0` | DeliveryReceipt 强制要求 Notification | 允许 suppressed/failed receipt 的 `notification_id` 为空 |
| `P0` | runtime 固定 unavailable | 接通正式 PostgreSQL repository/worker；未接通继续 fail closed |
| `P1` | target 引用尚未进入 Notification | 增加固定 target kind/id/action，并在点击时重新授权 |
| `P1` | 可见文案仍为英文常量 | 接入固定 content code 的繁体中文 i18n，不增加业务信息 |

本次只冻结目标设计，不修改产品代码、migration 或数据库。

## 10. 设计决策

| ID | 决策 |
| --- | --- |
| `SD-NOTIF-001` | Release 1 只保留 Notification 和 DeliveryReceipt |
| `SD-NOTIF-002` | 唯一渠道是站内通知，不发送外部业务消息 |
| `SD-NOTIF-003` | 可见文案固定为最小“有待办事项”，不保存自由文字 |
| `SD-NOTIF-004` | 接收人由当前关系事实解析，不能由生产者任意指定 |
| `SD-NOTIF-005` | 去重必须区分 recipient、effect、来源对象和业务日期/版本 |
| `SD-NOTIF-006` | suppressed 只生成 DeliveryReceipt，不生成用户可见 Notification |
| `SD-NOTIF-007` | 通知点击目标时重新授权，不把通知当成业务权限 |

## 11. 本模块确认结果

项目负责人已一次确认下面 7 点：

1. 接受只有 Notification 和 DeliveryReceipt 两张核心表，不增加 Schedule、Preference 或外部渠道表。
2. 接受 Release 1 只有站内通知，不发送 Email、SMS、WhatsApp 或 Push。
3. 接受通知文案只显示最小“有待办事项”，业务信息通过重新授权后的页面查询获取。
4. 接受九类 effect、对应接收人和 3 天/1 天/每日逾期规则。
5. 接受去重键必须包含 recipient，不能把同一 effect 错误合并给不同 User。
6. 接受权限失效或 effect 失效时只写 suppressed DeliveryReceipt，不创建用户可见通知。
7. 接受通知不改变 Case、SchoolTarget、Task 或审批事实，也不自动创建 Task。

Notifications 已标记为 `approved`，下一步进入 Audit 与 Operations。
