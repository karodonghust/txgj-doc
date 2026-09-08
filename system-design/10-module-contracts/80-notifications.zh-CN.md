# Notifications 模块契约

状态：`approved`  
确认依据：项目负责人于 2026-08-25 指示继续进入下一模块  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v31`

返回[模块契约索引](README.md)。

业务依据：`BR-038`，并引用 `BR-032`、`BR-033`、`BR-035` 至 `BR-039`、`BR-070`、`BR-071`。  
现状依据：[Notifications、Audit 与隐私现状分析](../../current-state-analysis/70-notifications-audit.zh-CN.md)。

## 1. 一句话职责

Notifications 只回答：

> 哪个内部用户现在应看到一条“有待办事项”的站内提醒，这个提醒是否已投递、已读、被抑制或投递失败？

通知不能创建 Task，也不能修改 Case、SchoolTarget 或审批结果。

## 2. 负责与不负责

| Notifications 负责 | Notifications 不负责 |
| --- | --- |
| 站内 Notification、已读状态和最小跳转引用 | Task、Case、SchoolTarget 或审批事实 |
| DeliveryReceipt、重试、抑制和 dead-letter 结果 | 决定谁是 Primary Advisor、Assignee 或 Founder |
| 业务 effect 到接收人的解析和请求时重验 | 外部 Email、SMS、WhatsApp 或 Push |
| 3 天、1 天和每日逾期提醒调度 | 修改 due_at、暂停 Case 或自动结案 |
| 接收人 + effect + 业务日期去重 | 用户自定义通知偏好、静音或频率设置 |

## 3. 核心对象

Release 1 只保留两个核心对象：

| 对象 | 含义 | 关键约束 |
| --- | --- | --- |
| `Notification` | 一个内部用户可见的站内待办提醒 | 只有 `unread`、`read`；可见文案不含业务 PII |
| `DeliveryReceipt` | 一次 effect 投递、抑制或失败的最终记录 | 保存 recipient、effect identity、attempt 和结果；不可改写或删除 |

以下概念不单独建实体：

- ReminderSchedule：由 Tasks 的 due_at 和当前状态按时计算。
- NotificationPreference：Release 1 的必需站内提醒不可由个人关闭。
- Email/SMS/WhatsApp delivery：不进入 Release 1。
- suppressed notification：只保存 DeliveryReceipt，不生成用户可见 Notification。
- notification target：Notification 内的 opaque 引用和请求时解析结果，不是新的授权实体。

## 4. 唯一渠道与可见内容

- Release 1 只有 `in_app`，所有内部和外部收件人都不发送业务 Email、SMS 或 WhatsApp。
- Guardian、Student 和 Portal 不接收内部 Notification，也不拥有通知列表。
- 可见标题和正文只能表达“有待办事项”，不得出现姓名、联系方式、Case number、学校名、Task 标题、文件名或自由文字原因。
- 页面可以显示统一的本地化文案，例如“有待办事项需要处理”。
- Notification 可以保存 opaque target reference；点击时由服务端重新授权后跳转，不能把私有信息写进 URL。
- 目标已失效或当前用户失去权限时，返回统一不可用结果，不泄露资源是否存在。

## 5. 业务 effect 目录

| Effect | 触发事实 | 接收人 |
| --- | --- | --- |
| `task_assigned` | Task 首次分配 | 当前实际 Assignee |
| `task_reassigned` | 同一 Task 产生新 Assignment | 新 Assignee |
| `task_rejected` | Assignee 拒绝当前 Assignment | 当前 Primary Advisor |
| `candidate_list_review_requested` | 候选名单版本提交 Founder 审批 | 全部 active Founder 用户 |
| `candidate_list_reviewed` | Founder 批准或驳回名单版本 | 当前 Primary Advisor |
| `task_due_in_3_days` | Task 距 due_at 3 天 | 实际 Assignee、当前 Primary Advisor |
| `task_due_in_1_day` | Task 距 due_at 1 天 | 实际 Assignee、当前 Primary Advisor |
| `task_overdue_daily` | Task 已逾期且未完成/取消 | 实际 Assignee、当前 Primary Advisor、全部 active Founder 用户 |
| `case_closure_choice_required` | Cases 确认全部 Target 终态且无未完成 Task | 当前 Primary Advisor、全部 active Founder 用户 |

规则：

- Founder 批准后待 Guardian 确认，由 `candidate_list_reviewed` 提醒 Primary Advisor 跟进，不再创建第二条同义提醒。
- Guardian 确认、等待学校结果、offer 决定和人工结案本身不生成额外 Notification，除非上表已有明确 effect。
- 通知不是 Task；任何 effect 都不能为了“方便跟进”自动创建 Task。
- “通知 Founder”表示所有 active Founder；不新增 designated Founder 实体。
- 同一人同时是 Assignee、Primary Advisor 或 Founder 时，按去重规则只收到一条。

## 6. 截止时间提醒

- 调度器按组织业务时区 `Asia/Hong_Kong` 计算业务日期。
- 到期前 3 天和 1 天各检查一次；只对未 completed/cancelled 的 Task 产生 effect。
- `is_overdue = now > due_at`；逾期后每天产生一次提醒，直到 Task completed/cancelled。
- Case 暂停不修改 due_at，也不停止逾期计算或提醒。
- 逾期 Task 仍可接受、完成、拒绝、重派或取消；Notifications 不改变这些动作。
- due_at 或 Assignment 发生有权变更后，下一次调度使用 Tasks 返回的当前事实。
- 标记已读不停止未来日期的必需提醒。

## 7. 去重 identity

```text
notification_dedupe_key =
  organization_id
  + recipient_user_id
  + effect_type
  + source_opaque_id
  + effect_version_or_business_date
```

- 即时 effect 使用生产者的稳定业务版本，例如 Assignment ID 或名单版本 ID。
- 定时 effect 使用 Task ID + `Asia/Hong_Kong` 业务日期。
- 同一接收人、同一 effect、同一来源、同一天只生成一条。
- 不同 Task 或不同名单版本不能被错误合并。
- 同一消息重复投递、Worker 重启或并发 claim 必须返回原 DeliveryReceipt，不创建第二条 Notification。

## 8. 接收人与权限重验

- 生产者只发布业务 effect 和 opaque source ID，不直接授予通知访问权。
- Notifications 通过 Access、Cases 或 Tasks 的公开查询解析当前接收人。
- 创建 Notification 前必须重验 recipient User、Membership、RoleBinding 和业务关系仍 active。
- Task 分配通知只给当前 Assignment；旧 Assignment 在投递前已结束时写 suppressed receipt。
- Task 拒绝、名单审批结果和结案选择提醒使用投递时的当前 Primary Advisor。
- Founder 接收人只取当前 active Founder RoleBinding；被撤销者不再投递。
- 点击 target 时再次重验；收到通知不等于永久获得目标资源权限。

## 9. 投递、重试与抑制

```text
outbox effect
  -> claim lease
  -> resolve recipients
  -> reauthorize
  -> create Notification + delivered receipt
```

- Worker 使用短 lease，并允许崩溃后安全重领。
- 技术失败有界重试；Release 1 默认最多 3 次。
- 最终失败写 failed DeliveryReceipt、进入 dead letter，并通知 Operations；不能修改原业务事实。
- 接收人失去权限、目标已失效或 effect 不再适用时写 suppressed DeliveryReceipt，用户不可见。
- delivered Notification 与 DeliveryReceipt 在同一数据库事务创建。
- 投递失败不回滚已经提交的 Task、Case 或审批操作。

## 10. 已读生命周期

```text
unread -> read
```

- 只有 Notification 的 recipient 可以标记已读。
- 已读时间和 expected version 必须保存，重复标记返回原结果。
- Release 1 不提供删除、撤回、静音或物理清理 Notification 的用户命令。
- 权限失效后，历史 Notification 不得继续作为进入业务对象的入口；target 解析仍会拒绝。

## 11. 对外查询契约

| 查询 | 调用方 | 返回 |
| --- | --- | --- |
| `listMyNotifications` | 内部通知列表 | 当前用户的最小文案、unread/read、时间和 opaque target token |
| `getMyUnreadCount` | Workspace shell | 当前用户未读数量 |
| `resolveNotificationTarget` | 通知点击入口 | 重新授权后的内部路由或统一 unavailable |
| `getDeliveryHealth` | Operations | effect、attempt、delivered/suppressed/failed 计数，不含业务内容 |

通知查询永远以 Session 当前 user 为 recipient，不接受客户端传入其他 user ID。

## 12. 对外命令契约

| 命令 | 关键规则 |
| --- | --- |
| `deliverBusinessEffect` | 仅受信 Worker；按 effect 目录解析接收人并幂等投递 |
| `evaluateTaskReminders` | 调度器；读取 Tasks 当前 reminder facts，按业务日期生成 effect |
| `markNotificationRead` | 当前 recipient；expected version；幂等 |

普通 Route Handler 不能直接创建 Notification，也不能传入任意 recipient、正文或 target URL。

## 13. 消费与发布

Notifications 消费：

- Cases 的名单审批请求、名单审批结果和结案选择 effect。
- Tasks 的分配、重派、拒绝、完成、取消和 due_at/Assignment 当前事实。

Notifications 发布：

| 事实 | 消费者 |
| --- | --- |
| `notifications.notification_created` / `read` | Audit、Operations 投影 |
| `notifications.delivery_suppressed` | Operations |
| `notifications.delivery_failed` | Operations 告警 |

事件只携带 opaque ID、effect code、状态和版本，不携带业务正文或 PII。

## 14. 依赖规则

| 类型 | 允许 |
| --- | --- |
| 业务依赖 | `Access`、`Cases`、`Tasks` 的公开查询契约 |
| 平台依赖 | `Shared`、`Audit` 的公开契约 |
| 异步输入 | Cases、Tasks 的 committed outbox event |
| 允许消费者 | 内部 Workspace shell、Today 页面、Operations |

明确禁止：

- Notifications 写 Cases、Tasks、Access 或 Identity 私有表。
- 业务模块、页面或 Route Handler 直接写 Notification/DeliveryReceipt 表。
- 使用 Notification、未读数或 target 缓存作为业务授权来源。
- 添加外部邮件、短信、WhatsApp、Push 或用户自定义渠道。
- 将 Guardian、Student 或 Portal 身份映射成内部通知 recipient。

## 15. 安全与一致性不变量

- Notification 只能属于一个 organization 和一个内部 User recipient。
- 接收人解析、创建 Notification、DeliveryReceipt 和 dedupe 结果必须原子提交。
- 任何可见文案都来自固定 content code + i18n，不接受生产者自由文字。
- 通知 target 使用 opaque 引用并在点击时重验，不能包含 PII 或未授权业务参数。
- 日志、outbox、dead letter、telemetry 和错误不包含姓名、联系方式、Case/School/File 内容、Cookie、token 或 secret。
- scheduler 和 Worker 至少一次执行；唯一约束和 idempotency 保证最终不重复。
- runtime 未接通时 fail closed；不得用 preview notification 假装已投递。

## 16. 与当前代码的差异

| 优先级 | 当前实现 | 目标契约 |
| --- | --- | --- |
| `P0` | eligible event 只有通用 Task/Case transition | 实现已确认的九类业务 effect 及接收人规则 |
| `P0` | 没有 3 天、1 天和每日逾期 scheduler | 增加可重放 reminder evaluator 和日期去重 |
| `P0` | 没有所有 Target 终态且无未完成 Task 的提醒 | 消费 Cases 的 `case_closure_choice_required` effect |
| `P0` | 单一 actor role 与通用 capability 语义仍存在于上游 | 使用 Access 多角色 context 和实时业务关系解析 recipient |
| `P1` | suppressed 同时创建不可见 Notification 记录 | suppressed 只保留 DeliveryReceipt，不创建 Notification |
| `P1` | 固定英文文案 | 使用固定最小 content code 的繁体中文界面文案；仍不披露业务对象 |
| `P1` | runtime 固定 unavailable | 接通显式 PostgreSQL repository/worker；未配置时继续 fail closed |
| `P1` | Today/通知入口仍含 preview 路径 | 只读取正式 Notifications 查询契约 |

这些差异进入后续开发拆分；本环节不修改产品代码或数据库。

## 17. 设计决策

| ID | 决策 |
| --- | --- |
| `SD-NOTIF-001` | Release 1 只保留 Notification 和 DeliveryReceipt 两个核心对象 |
| `SD-NOTIF-002` | 唯一渠道是站内通知；不发送任何外部业务消息 |
| `SD-NOTIF-003` | 可见文案固定为最小“有待办事项”，target 点击时重新授权 |
| `SD-NOTIF-004` | 提醒接收人由当前关系事实解析，不信任生产者传入的任意 recipient |
| `SD-NOTIF-005` | 去重区分来源对象，并满足 recipient + effect + business date 唯一 |
| `SD-NOTIF-006` | 暂停不停止逾期提醒；已读不停止未来日期提醒 |
| `SD-NOTIF-007` | suppressed 只形成 DeliveryReceipt，不产生用户可见 Notification |

## 18. 本模块验收标准

项目负责人需要确认：

1. Release 1 只有站内通知，不发送外部业务 Email、SMS、WhatsApp 或 Push。
2. 核心对象只有 Notification 和 DeliveryReceipt，不新增 Schedule/Preference 实体。
3. Task、名单审批和人工结案选择只按本契约九类 effect 提醒。
4. 到期前 3 天、1 天及逾期每日提醒遵循已确认接收人，Case 暂停期间继续。
5. 通知文案只显示“有待办事项”，点击目标时重新授权。
6. 同一用户兼任多个接收角色时只收一条；不同 Task 不被错误合并。
7. 失去权限或 effect 失效时抑制投递，不泄露资源存在性。

确认后进入下一个模块：Audit 与 Operations。
