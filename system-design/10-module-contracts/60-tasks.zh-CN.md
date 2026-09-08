# Tasks 模块契约

状态：`approved`  
确认依据：项目负责人于 2026-08-25 指示进入下一模块  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v31`

返回[模块契约索引](README.md)。

业务依据：`BR-035` 至 `BR-037`，并引用 `BR-012`、`BR-013`、`BR-033`、`BR-034`、`BR-039`、`BR-050`。  
现状依据：[Tasks 现状分析](../../current-state-analysis/40-tasks.zh-CN.md)。

## 1. 一句话职责

Tasks 只回答：

> 需要谁在什么时候完成哪项工作，当前有效分派是谁，工作是否已接受、完成、取消或逾期？

Task 完成不自行决定 Case 或 SchoolTarget 是否可以推进。

## 2. 负责与不负责

| Tasks 负责 | Tasks 不负责 |
| --- | --- |
| Task 内容、类型、due_at 和当前工作状态 | ServiceCase、SchoolTarget 或申请结果 |
| TaskAssignment 当前分派和完整历史 | Primary Advisor、CaseCollaborator、Application Assignee 关系本身 |
| 接受、拒绝、重派、取消和完成记录 | 文件内容、扫描结果或对象存储 |
| 申请提交 Task 的完成资料 | 独立 Material 或 SubmissionEvidence 实体 |
| 面试辅助 Task 的受限工作区 | Assessment、Guardian 联系资料或内部备注 |
| 计算型 overdue 与提醒所需事实 | Founder 逐条验收或 approved 状态 |

## 3. 核心对象

Release 1 只保留两个核心对象：

| 对象 | 含义 | 关键约束 |
| --- | --- | --- |
| `Task` | 一项需要完成的工作 | 关联一个 Case，可选关联一个 SchoolTarget；永久保留历史，不物理删除 |
| `TaskAssignment` | 一次明确的任务分派 | 记录 assignee、角色、分派/接受/结束时间和结束原因；重派只追加新 Assignment |

以下概念不单独建实体：

- overdue：由 due_at、当前时间和 Task 状态计算。
- rejection/reassignment：TaskAssignment 的结束事实和 Task 历史。
- completion/approval receipt：Task 的追加式完成记录；Release 1 没有 approval。
- Material/SubmissionEvidence：使用 Task completion、SchoolTarget 和 Case Document 引用。
- Contractor workspace：当前 Assignment 的脱敏查询结果。
- TaskTransitionPolicy：Release 1 使用固定、版本化领域规则，不建立可由业务用户动态审批的 policy 实体。

## 4. Task 类型

| 类型 | 创建来源 | 是否可推进业务 |
| --- | --- | --- |
| `application_prepare_submit` | Cases 的 SchoolTarget preparing 事件 | 完成后只发布提交事实，由 Cases 重验后决定 submitted |
| `interview_support` | Cases 的 interview required 事件 | 否；完成只表示辅助工作完成 |
| `manual` | 当前 Primary Advisor 人工创建 | 否；不自动推进 Case 或 SchoolTarget |

Release 1 自动 Task 只有前两类：

- background collection 不自动建 Task。
- Founder 名单审批、Guardian 确认、等待学校结果、offer 决定和结案不自动建 Task。
- 自动创建必须消费 Cases 的权威事件，不能由页面根据当前状态自行补建。

## 5. Task 与 Assignment 生命周期

Task 的可停留状态收敛为：

```text
assigned -> accepted -> completed
assigned -> awaiting_reassignment       （Assignee 拒绝）
accepted -> awaiting_reassignment       （Primary Advisor 主动重派）
awaiting_reassignment -> assigned       （同一 Task 创建新 Assignment）
assigned / accepted / awaiting_reassignment -> cancelled
```

规则：

- 新 Task 创建时同时建立第一条 active TaskAssignment，对外状态为 assigned。
- Assignee 可以接受，或在未接受时填写原因拒绝。
- 拒绝只结束当前 Assignment，Task 转为 awaiting_reassignment，不创建替代 Task。
- Primary Advisor 重派时结束旧 Assignment，并在同一 Task 上创建新 assigned Assignment。
- 新 Assignee 必须重新接受；旧 Assignment 历史不可修改或删除。
- completed 和 cancelled 是终态。
- created、rejected、reassigned 只作为动作/历史事实，不作为长期 Task 状态。
- Release 1 没有 approved；Assignee 满足完成条件后直接 completed。

## 6. 申请准备与提交 Task

- SchoolTarget 进入 preparing 时，由 Cases 提供当前 Application Assignee 和申请轮次。
- Application Assignee 只能是当前 Primary Advisor，或有明确案件授权的 Case Collaborator；Contractor 不能担任。
- 同一 SchoolTarget、同一申请轮次只创建一条 Task；幂等键不得依赖标题或学校名称。
- 一条 Task 同时负责准备和正式提交，不拆成两个 Task。
- 材料清单按该校当期官方要求保存为 Task 工作内容/完成快照，不建立全校通用固定清单。

完成前必须保存：

1. 提交时间。
2. 提交渠道。
3. 实际提交人。
4. 当期材料清单完成状态/快照。
5. 学校参考号，或明确“学校未提供参考号”。
6. 未提供参考号时，至少一份确认页、确认邮件 PDF、回执或邮寄凭证等 Case Document 引用。

Tasks 在完成事务中重新查询 Cases 的当前 Target/Application Assignee 事实，并通过受控证据查询确认引用有效。完成后发布 `tasks.application_submission_completed`；Cases 再次校验后才推进 SchoolTarget。

## 7. 面试辅助 Task

- 只消费 Cases 的 interview required 事件创建，不能因为学校类型或 UI 推测自动创建。
- 默认 Assignee 为当前 Primary Advisor。
- 也可指派当前 Case Collaborator Advisor，或通过单一 TaskAssignment 指派 Contractor。
- 同一 SchoolTarget、同一面试事件/version 必须幂等。
- 完成只表示面试辅助工作结束，不表示面试结果、录取或 SchoolTarget 推进。

非 Primary Advisor 的工作区只允许：

- Task 标题和辅导要求。
- 目标学校。
- 面试时间、方式和语言。
- Primary Advisor 编写的必要背景摘要。
- due_at、当前 Assignment 和允许动作。

明确禁止返回完整 Assessment、Guardian/联系方式、内部备注、Case 文件、下载/导出、其他学校或其他 Case。

## 8. Contractor 单任务边界

- Contractor 必须具有 active Membership、唯一 Contractor RoleBinding 和当前 active TaskAssignment。
- 每次请求都重新检查 Task、Assignment、Case 和角色状态；不能只信任 Session claim。
- Contractor 只可读取当前分配的 interview support Task 脱敏 DTO。
- Contractor 只能接受、带原因拒绝或按完成要求完成自己的 Task。
- 拒绝、完成、取消、重派、Assignment 结束或 Case 结束后，下一个请求立即失去访问。
- Contractor 不成为 CaseCollaborator、Application Assignee、Primary Advisor 或案件成员。
- Contractor 不得使用全局 tasks.read/tasks.transition 获得其他 Task 列表。

## 9. Manual Task

- 只有当前 Primary Advisor 可以为自己负责的 active Case 创建 manual Task。
- manual Task 必须有明确标题、工作说明、due_at 和合规 Assignee。
- Assignee 必须通过当前案件关系和数据最小化检查；向 Contractor 分派时必须存在明确的单任务脱敏 DTO。
- manual Task 的完成记录由创建时定义，但不得修改 Case stage、SchoolTarget 或学校结果。
- 不通过 manual Task 绕过 application/interview 自动 Task 的必填完成证据。

## 10. Overdue 与暂停

```text
is_overdue = 当前时间 > due_at
             且 Task 不是 completed/cancelled
```

- overdue 只是查询、显示和提醒标记，不是 Task 状态。
- 逾期后仍可接受、完成、拒绝、重派或取消。
- Case 暂停不修改 due_at，不自动完成、取消或重建 Task。
- 暂停期间 Task 保留；正常业务动作受 Cases 暂停规则限制。
- 逾期提醒继续，直到 Task completed/cancelled 或 due_at/Assignment 通过有权命令明确变更。

## 11. 取消规则

- 已确认名单移除学校时，Cases 请求取消该 SchoolTarget 的未完成 Task。
- 整体终止服务时，Cases 请求取消 Case 的全部未完成 Task。
- 取消保留 Task、Assignment、操作者、时间和原因历史。
- completed Task 不因名单修改或 Case 结案被回滚或删除。
- 重复取消事件必须返回原结果，不产生第二次副作用。

## 12. 授权边界

| 访问者 | Task 权限 |
| --- | --- |
| 当前 Primary Advisor | 查看 Case Task、创建 manual、重派和按规则取消 |
| Founder | 查看组织内 Case Task；不逐条审批 completed Task |
| Advisor Assignee | 查看并操作自己的完整授权 Task |
| Case Collaborator | 只在其 Case scope 和当前 Assignment 允许范围内访问 |
| Contractor | 只访问当前 interview support Task 的脱敏工作区 |
| Admin 基础角色 | 默认不可查看客户 Task |
| Guardian、Student、Portal | 不直接访问内部 Task |

所有入口先使用 Access 多角色 AuthorizationContext，再由 Tasks 重验 Cases 关系和当前 Assignment。

## 13. 对外查询契约

| 查询 | 主要调用方 | 返回 |
| --- | --- | --- |
| `listCaseTasks` / `getTask` | 内部任务页面、Cases | 授权范围内 Task、当前 Assignment、计算型 overdue 和受控完成摘要 |
| `getAssignedTasks` | Advisor 自己的工作台 | 当前分派给自己的 Task，不扩大 Case 权限 |
| `getContractorTaskWorkspace` | Contractor 单任务路由 | 固定脱敏 DTO 和当前允许动作 |
| `getOpenTaskGuard` | Cases 结案协调器 | Case 是否仍有未完成 Task |
| `getTaskDocumentFacts` | Documents | Task/Assignment/SchoolTarget 关联的最小授权事实 |
| `getTaskReminderFacts` | Notifications | due_at、overdue、Assignee opaque ID 和去重 identity |

## 14. 对外命令契约

| 命令 | 关键规则 |
| --- | --- |
| `ensureApplicationTask` | 只消费 Cases 事件；SchoolTarget + application round 幂等 |
| `ensureInterviewSupportTask` | 只消费 interview event；默认 Primary Advisor；事件/version 幂等 |
| `createManualTask` | 当前 Primary Advisor；不得自动推进业务 |
| `acceptAssignment` | 当前 Assignee；expected version |
| `rejectAssignment` | 当前未接受 Assignee；原因必填；结束 Assignment |
| `reassignTask` | 当前 Primary Advisor；同一 Task 新建 Assignment 并保留旧历史 |
| `completeTask` | 当前 accepted Assignee；按 Task 类型校验完成记录和证据 |
| `cancelTask` | 当前 Primary Advisor 或 Cases 受信事件；原因必填；保留历史 |

所有写命令都必须幂等，使用 expected version，并与 Task/Assignment、AuditEvent、Outbox 和 Idempotency result 原子提交。

## 15. 发布的事实

| 事实 | 主要消费者 |
| --- | --- |
| `tasks.task_created` / `assignment_created` | Assignee 站内通知、Audit、Operations |
| `tasks.assignment_accepted` / `rejected` / `reassigned` | Primary Advisor、Audit、Operations |
| `tasks.application_submission_completed` | Cases、Audit、Operations |
| `tasks.interview_support_completed` | Primary Advisor、Audit、Operations |
| `tasks.manual_task_completed` | Primary Advisor、Audit |
| `tasks.task_cancelled` | Cases、Notifications、Audit、Operations |

事件只携带 opaque ID、状态、版本、受控 reason code 和必要证据引用；不携带 Student/Guardian PII、Assessment、文件内容或 Contractor 背景摘要。

## 16. 依赖规则

| 类型 | 允许 |
| --- | --- |
| 业务依赖 | `Access`、`Cases` 的公开授权/关系契约 |
| 平台依赖 | `Shared`、`Audit` 的公开契约 |
| 证据协调 | 顶层 completion coordinator 可组合 Tasks、Cases、Documents 的公开查询 |
| 异步协作 | 消费 Cases 任务请求；向 Cases、Notifications 发布 Task 事实 |
| 允许消费者 | Cases、Documents、Notifications、内部 Task 入口、Contractor 单任务入口 |

明确禁止：

- Tasks 写 Cases、Access、Documents 或 Notifications 私有表。
- Cases、页面或 Worker 直接写 Task/Assignment 表。
- Task completion 直接修改 SchoolTarget 或 Case stage。
- Contractor 使用 Case workspace、文件接口或通用 Task 列表。
- mock/preview/legacy Task runtime 静默进入 production-aws。

## 17. 安全与一致性不变量

- Task 和 TaskAssignment 历史不可物理删除或改写。
- 任意操作重验 organization、Case、Task、current Assignment、actor 和 expected version。
- 自动 Task 使用业务幂等 identity，网络 idempotency key 不能替代业务去重约束。
- Assignment 失效后，Contractor/Advisor 的任务访问在下一请求立即消失。
- 完成记录和 Document 引用不复制文件内容；outbox、审计和错误不得包含 PII 或完整任务背景。
- 事件至少一次投递，消费者必须幂等；无法创建/推进时进入 Operations 告警。
- runtime 未配置时 fail closed；本地 PostgreSQL runtime 不证明 production-aws 已接通。

## 18. 与当前代码的差异

| 优先级 | 当前实现 | 目标契约 |
| --- | --- | --- |
| `P0` | 没有 application/interview Task 类型和 Cases 自动触发 | 只按两类 Cases 事件幂等创建自动 Task |
| `P0` | Task 状态包含 approved 和 Founder 审批 | completed 即完成，不再审批 |
| `P0` | overdue 是独立状态 | 改为 due_at 计算标记 |
| `P0` | created/rejected/reassigned 是长期 Task 状态 | 动作归历史；拒绝后 Task 停在 awaiting_reassignment |
| `P0` | 申请 Task 缺提交资料和替代凭证校验 | 按 BR-035 完整校验完成记录 |
| `P0` | 动态 TaskTransitionPolicy/OD-06 决定运行时状态机 | Release 1 使用固定版本化领域规则，不保留动态审批 policy |
| `P1` | Contractor DTO 只有标题、brief、due_at | 增加明确允许的学校/面试信息，仍严格排除其他 Case 数据 |
| `P1` | Contractor workspace runtime 固定 unavailable | 接通显式 PostgreSQL adapter，Assignment 失效即时拒绝 |
| `P1` | service 使用单一 IdentitySessionActor.role 和全局 task capability | 改用 Access 多角色 context、Case 关系和当前 Assignment |
| `P1` | production-aws runtime 主动 unavailable | 实现显式生产 adapter；未接通前保持 fail closed |

这些差异进入后续开发拆分；本环节不修改产品代码或数据库。

## 19. 设计决策

| ID | 决策 |
| --- | --- |
| `SD-TASK-001` | Tasks 只拥有 Task 与 TaskAssignment |
| `SD-TASK-002` | Release 1 自动任务仅申请准备/提交和面试辅助两类 |
| `SD-TASK-003` | 拒绝结束 Assignment，重派复用同一 Task 并追加新 Assignment |
| `SD-TASK-004` | overdue 是计算标记，approved 不存在 |
| `SD-TASK-005` | 申请 Task 完成必须保存完整提交事实和受控证据引用 |
| `SD-TASK-006` | Task 完成只发布事实，Cases 重验后决定 SchoolTarget 是否推进 |
| `SD-TASK-007` | Contractor 只通过当前 Assignment 获得单任务脱敏工作区 |

## 20. 本模块验收标准

项目负责人需要确认：

1. Tasks 只保留 Task 和 TaskAssignment，不增加其他业务实体。
2. 自动 Task 只有申请准备/提交和必要面试辅助两类。
3. 拒绝结束当前 Assignment，重派保留同一 Task 和全部 Assignment 历史。
4. overdue 只是计算标记；completed 不需要 Founder approved。
5. 申请 Task 完成必须保存提交资料和参考号/替代凭证。
6. Contractor 只看当前面试辅助 Task 的严格脱敏工作区。
7. Task 完成不能直接推进 Case 或 SchoolTarget。

确认后进入下一个模块：Documents。
