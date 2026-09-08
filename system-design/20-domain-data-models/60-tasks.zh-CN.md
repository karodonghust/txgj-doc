# Tasks 领域模型与数据设计

状态：`approved`  
确认依据：项目负责人于 2026-08-25 接受 Tasks 三张目标表、自动任务边界和完成回执设计  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[领域模型与数据设计索引](README.md)。

业务依据：`BR-035` 至 `BR-037`。  
模块依据：[Tasks 模块契约](../10-module-contracts/60-tasks.zh-CN.md)。  
现状依据：[Tasks 现状分析](../../current-state-analysis/40-tasks.zh-CN.md)。  
代码参考：`modules/tasks/**`、migration `005`、`033`。

## 1. 设计结论

Tasks Release 1 使用三张目标逻辑表：

| 对象 | 目标表 | 负责的事实 |
| --- | --- | --- |
| `Task` | `tasks_tasks` | 工作内容、类型、截止时间、当前状态和业务幂等身份 |
| `TaskAssignment` | `tasks_task_assignments` | 每次明确分派、接受、拒绝、重派和结束的关系 |
| `TaskTransitionReceipt` | `tasks_task_transition_receipts` | Task 状态变化、完成记录和受控证据引用 |

不建立以下表：

- `overdue` 表：逾期由 `due_at` 和当前时间计算。
- `Material` 或 `SubmissionEvidence` 表：提交材料清单和凭证引用放在完成回执中，文件正文归 Documents。
- `TaskApproval` 表：Release 1 没有 Founder 逐条验收，完成即 completed。
- 动态 `TaskTransitionPolicy` 表：规则固定在版本化代码契约中；旧 policy/rule 表停止新写入，仅保留历史。

## 2. 任务来源与生命周期

自动任务只有两类：

| 类型 | 创建节点 | 完成后 |
| --- | --- | --- |
| `application_prepare_submit` | SchoolTarget 进入 `preparing` | 发布提交事实，Cases 重验后推进 Target |
| `interview_support` | 学校明确要求面试，Target 进入 `interview` | 只表示辅助完成，不推进学校结果 |

Primary Advisor 还可以人工创建 `manual` Task；它不自动推进 Case 或 SchoolTarget。

```text
Task:
assigned -> accepted -> completed
assigned / accepted -> awaiting_reassignment -> assigned
assigned / accepted / awaiting_reassignment -> cancelled

Assignment:
assigned -> accepted -> ended
assigned -> rejected -> ended
```

- 拒绝结束当前 Assignment，Task 进入 `awaiting_reassignment`。
- 重派复用同一 Task，新增 Assignment，不创建替代 Task。
- `overdue` 不是状态：`now > due_at` 且 Task 未 completed/cancelled 时计算为 true。
- Task 和 Assignment 历史永久保留，不物理删除。

## 3. Task 目标逻辑表

表：`tasks_tasks`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | Task UUID 主键 |
| `organization_id` | 是 | 已有 | 所属 Organization，也是 RLS 边界 |
| `service_case_id` | 是 | 已有 | 对应 Cases ServiceCase |
| `school_target_id` | 条件必填 | 新增 | 申请/面试 Task 对应的 SchoolTarget；manual 可空 |
| `task_type` | 是 | 新增 | `application_prepare_submit`、`interview_support` 或 `manual` |
| `task_key` | 是 | 新增 | 业务幂等身份；自动任务由 Target + 申请轮次/面试事件组成 |
| `creation_trigger` | 是 | 新增 | `case_event` 或 `advisor_manual` |
| `source_event_id` | 条件必填 | 新增 | 自动任务对应的 Cases 事件 ID；manual 为空 |
| `title` | 是 | 已有 | 任务标题 |
| `task_brief` | 是 | 已有 | 工作说明；Contractor 只收到脱敏版本 |
| `due_at` | 是 | 已有 | 截止时间；用于计算 overdue |
| `state` | 是 | 需纠正 | `assigned`、`accepted`、`awaiting_reassignment`、`completed` 或 `cancelled` |
| `owner_user_id` | 是 | 已有 | 负责该 Case 的 Primary Advisor，用于任务治理和重派边界 |
| `current_assignment_id` | 条件必填 | 新增 | 当前有效 TaskAssignment；创建事务完成后必填 |
| `completed_at` | 条件必填 | 新增 | Task 进入 completed 的时间 |
| `cancelled_at` | 条件必填 | 新增 | Task 进入 cancelled 的时间 |
| `cancelled_by_user_id` | 条件必填 | 新增 | 取消操作者；系统事件可空并由 Audit 留证 |
| `cancellation_reason` | 条件必填 | 新增 | 取消原因 |
| `last_transition_receipt_id` | 条件必填 | 已有 | 当前状态对应的最后回执 |
| `last_transition_actor_user_id` | 否 | 已有 | 最后一次 User 操作者；系统动作可空 |
| `last_transition_reason` | 否 | 已有 | 最后一次受控原因 |
| `record_version` | 是 | 已有 | 乐观锁；任何状态/当前 Assignment 变化均检查 expected version |
| `created_at` | 是 | 已有 | 创建时间 |
| `updated_at` | 是 | 已有 | 更新时间 |

关键约束：

- `(organization_id, task_key)` 唯一；网络重试不能重复创建同一自动 Task。
- `application_prepare_submit` 必须有 `school_target_id` 和 `source_event_id`。
- `interview_support` 必须有 `school_target_id` 和面试事件来源。
- `manual` 必须由当前 Primary Advisor 创建，并且不能绑定会自动推进业务的 completion 语义。
- `completed` 必须有完成回执；`cancelled` 必须有取消操作者、时间和原因。
- Task 不直接修改 Cases 或 SchoolTarget；完成只发布事件。

## 4. TaskAssignment 目标逻辑表

表：`tasks_task_assignments`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | Assignment UUID；每次重派新建 |
| `organization_id` | 是 | 已有 | 所属 Organization |
| `task_id` | 是 | 已有 | 对应 Task |
| `assignee_user_id` | 是 | 已有 | 实际执行人 |
| `assignee_membership_id` | 是 | 新增 | 当时的 active Membership |
| `assignee_role_binding_id` | 是 | 新增 | 当时的精确 RoleBinding |
| `assignee_role` | 是 | 已有 | `advisor` 或 `contractor` |
| `case_collaborator_id` | 条件必填 | 新增 | Advisor 非 Primary 时的明确 CaseCollaborator；Contractor 必须为空 |
| `redaction_profile` | 是 | 已有，需纠正 | Advisor 为 `case_task`；Contractor 固定为 `task_only` |
| `assigned_by_actor_kind` | 是 | 新增 | `user` 或 `service` |
| `assigned_by_actor_id` | 是 | 由旧 assigned_by_user_id 扩展 | 分派主体 opaque ID |
| `assignment_reason` | 是 | 由旧 reason 改名 | 初次分派、拒绝后重派或更换原因 |
| `status` | 是 | 需纠正 | `assigned`、`accepted`、`rejected`、`reassigned` 或 `cancelled` |
| `assigned_at` | 是 | 新增 | 分派时间 |
| `accepted_at` | 条件必填 | 新增 | Assignee 接受时间 |
| `ended_at` | 条件必填 | 新增 | Assignment 结束时间 |
| `ended_by_user_id` | 条件必填 | 新增 | 结束 Assignment 的 User；系统取消可空 |
| `end_reason` | 条件必填 | 新增 | 拒绝、重派或取消原因 |
| `record_version` | 是 | 新增 | Assignment 生命周期乐观锁 |
| `created_at` | 是 | 已有 | 创建时间 |
| `updated_at` | 是 | 新增 | 更新时间 |

关键约束：

- 一个 Task 同时最多一条未结束的 Assignment。
- 新 Task 创建时必须和第一条 `assigned` Assignment 同一事务写入。
- 只有当前 Assignment 的 Assignee 可以接受或拒绝自己的分派。
- 只有 Primary Advisor（或 Cases 的受信取消事件）可以重派、取消 Assignment。
- Contractor 只能被分派 `interview_support`，且只能读取当前 Assignment 的脱敏 DTO。
- Assignment 结束后，下一次请求立即重新检查并拒绝访问；不依赖后台清理。
- Assignment 历史不删除；重派不复活旧 Assignment。

## 5. TaskTransitionReceipt 目标逻辑表

表：`tasks_task_transition_receipts`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | 状态变化回执 UUID |
| `organization_id` | 是 | 已有 | 所属 Organization |
| `task_id` | 是 | 已有 | 对应 Task |
| `assignment_id` | 条件必填 | 新增 | 执行动作时的当前 Assignment；系统取消可空 |
| `from_state` | 是 | 已有 | 原 Task 状态 |
| `to_state` | 是 | 已有 | 新 Task 状态 |
| `actor_kind` | 是 | 新增 | `user` 或 `service` |
| `actor_user_id` | 条件必填 | 已有，需允许系统动作为空 | 实际 User；service 动作由 actor_kind 和 Audit 表达 |
| `actor_role` | 条件必填 | 已有，移除 data_reviewer | founder / admin / advisor / contractor；service 可空 |
| `expected_record_version` | 是 | 已有 | 写入前版本 |
| `result_record_version` | 是 | 已有 | 写入后版本，必须 +1 |
| `reason` | 条件必填 | 已有 | 拒绝、重派、取消或异常原因 |
| `completion_record_json` | 条件必填 | 新增 | completed 时的严格 schema 完成记录和证据引用 |
| `source_event_id` | 否 | 新增 | Cases 取消或自动协调事件 ID |
| `occurred_at` | 是 | 新增 | 业务动作发生时间 |
| `created_at` | 是 | 已有 | 回执写入时间 |

`completion_record_json` 只允许三种严格结构：

| Task 类型 | 完成记录内容 |
| --- | --- |
| `application_prepare_submit` | 提交时间、渠道、提交人、材料清单完成快照、学校参考号或无参考号声明、至少一个替代凭证 Document ID |
| `interview_support` | 辅助完成时间、面试方式/语言确认、必要辅导完成摘要；不保存面试结果 |
| `manual` | 完成时间和简短完成说明；不得包含自动推进 Case 的字段 |

关键约束：

- Receipt append-only；任何 Task 状态变化必须有一条匹配回执。
- `completed` 必须满足对应 task_type 的完成 schema；`approved` 不再是合法目标状态。
- `application_prepare_submit` 必须满足“官方参考号”或“明确无参考号 + 替代凭证”二选一。
- Receipt 只保存 opaque ID、hash 和受控摘要，不保存文件正文、Guardian 联系方式或 Assessment 原文。
- 失败重试使用 Shared IdempotencyRecord；相同命令只返回原结果，不产生第二条业务事实。

## 6. 授权与 Contractor 边界

| 访问者 | 可做什么 |
| --- | --- |
| 当前 Primary Advisor | 查看 Case Task、创建 manual、重派、取消和处理自己可执行的任务 |
| Founder | 查看组织内 Task；不逐条批准 completed |
| Advisor Assignee | 读取并处理自己当前 Assignment |
| Case Collaborator Advisor | 只按 Case scope 和当前 Assignment 访问 |
| Contractor | 只读取并操作当前 `interview_support` Assignment 的 `task_only` DTO |
| Admin 基础角色 | 默认不能查看客户 Task |
| Guardian、Student、Portal | 不直接访问内部 Task |

每次请求重新检查 Organization、Case、Task、Assignment、Membership、RoleBinding 和 expected version。客户端 role、Task ID 或缓存结果都不是授权依据。

## 7. 事件与跨模块边界

```text
Cases: SchoolTarget preparing
  -> tasks.application_task_requested
  -> Task + Assignment

Cases: SchoolTarget interview required
  -> tasks.interview_task_requested
  -> Task + Assignment

Task completed
  -> tasks.application_submission_completed / interview_support_completed
  -> Cases 重新校验后推进或保留 Target
```

- Cases 不写 Tasks 私有表；Tasks 不写 Cases 私有表。
- 名单移除学校或整体终止服务时，Cases 发布取消请求；Tasks 取消未完成 Task 并保留历史。
- Case 暂停不自动取消、完成、重建或顺延 Task；`due_at` 继续计算。
- Notifications 只消费 Task reminder facts；不会改变 Task 状态。

## 8. RLS、索引与数据分类

- 三张表都显式保存 `organization_id`，启用并 FORCE RLS。
- 关键索引：`(organization_id, task_key)` 唯一；`(organization_id, service_case_id, state, due_at)`；当前 Assignment 部分唯一索引；`(organization_id, task_id, result_record_version)` 唯一。
- Task brief、completion summary、evidence Document ID 属于内部业务数据；不进入普通日志、Outbox payload 或错误正文。
- Contractor 返回固定脱敏 DTO：Task 标题、学校、面试时间/方式/语言、辅导要求、due_at、当前状态和允许动作。
- 不返回 Assessment、Guardian 联系方式、内部备注、其他学校、其他 Case 或文件下载能力。

## 9. 当前 schema 与目标差异

| 优先级 | 当前实现 | 目标处理 |
| --- | --- | --- |
| `P0` | Task policy/rules 是动态数据库审批模型 | Release 1 改为固定版本化代码规则；旧表停止新写 |
| `P0` | Task 状态含 created/rejected/reassigned/approved/overdue | 收敛为 assigned/accepted/awaiting_reassignment/completed/cancelled |
| `P0` | 没有 application/interview 自动任务触发 | 由 Cases 事件 + task_key 幂等创建 |
| `P0` | Task 没有 SchoolTarget、task_type 和业务幂等字段 | 增加对应字段 |
| `P0` | Assignment 缺精确 Membership/RoleBinding、接受/结束回执 | 增加授权快照和生命周期字段 |
| `P0` | TransitionReceipt 无完成资料 | 增加严格 completion_record_json 和证据校验 |
| `P0` | 完成需要 Founder approved | 删除 approved 状态，Assignee 满足条件直接 completed |
| `P1` | Contractor runtime 固定 unavailable | 接通显式 PostgreSQL adapter 和单任务脱敏 DTO |

历史 migration `005` 不修改。后续只新增 corrective migration，并先处理旧 policy、旧状态和既有 Task 的只读分类；本阶段不执行数据库变更。

## 10. 设计决策

| ID | 决策 |
| --- | --- |
| `DD-TASK-001` | Tasks 只保留 Task、TaskAssignment、TaskTransitionReceipt 三张目标表 |
| `DD-TASK-002` | 自动任务只有申请准备/提交和面试辅助两类 |
| `DD-TASK-003` | 重派复用同一 Task，追加 Assignment，不创建替代 Task |
| `DD-TASK-004` | overdue 是 due_at 计算值；approved 不存在 |
| `DD-TASK-005` | 申请 Task 完成回执保存提交资料和参考号/替代凭证 |
| `DD-TASK-006` | Task 完成只发布事实，Cases 重新校验后推进 Target |
| `DD-TASK-007` | Contractor 只能通过当前 Assignment 获得单任务脱敏工作区 |

## 11. 本模块验收标准

项目负责人需要确认：

1. 接受三张目标逻辑表，不增加 Material、SubmissionEvidence、Approval 或 Overdue 表。
2. 接受自动任务只有申请准备/提交和必要面试辅助两类。
3. 接受拒绝结束当前 Assignment，重派保留同一 Task 和全部历史。
4. 接受 overdue 只是计算标记，completed 不需要 Founder approved。
5. 接受申请 Task 完成必须保存提交资料和参考号/替代凭证。
6. 接受 Contractor 只访问当前面试辅助 Task 的严格脱敏工作区。
7. 接受 Task 完成不能直接修改 Case 或 SchoolTarget。

确认后进入下一个模块：Documents。
