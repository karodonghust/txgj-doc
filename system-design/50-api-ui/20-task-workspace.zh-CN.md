# Task 工作台 API 与页面

状态：`approved`  
确认依据：项目负责人于 2026-08-25 确认内部/Contractor 双工作区、三类完成表单、申请提交证据、Cases 重验和服务端 allowed actions 规则  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[API 与页面交互设计索引](README.md)。

业务依据：[Tasks 任务流程与状态机](../30-business-flows/20-tasks.zh-CN.md)。  
权限依据：[内部认证与授权模型](../40-permissions-security/10-authorization-model.zh-CN.md)。

## 1. 先看页面目标

Task 工作台只回答：

1. 当前用户有哪些授权任务？
2. 每个任务谁负责、何时到期、当前状态是什么？
3. 当前用户可以接受、拒绝、完成、重派还是取消？
4. 完成任务需要哪些资料？

页面不能用本地状态直接推进 Case 或 SchoolTarget。Task 完成后由 Tasks 发布事实，Cases 再次校验后推进业务。

## 2. 两种工作区

### 2.1 内部 Task 工作台

适用于 Founder、Primary Advisor、Advisor Assignee 和 Case Collaborator Advisor：

- 可显示当前授权范围内的 Case/SchoolTarget 关联摘要；
- Application Assignee 可查看准备/提交所需资料和受控 Document 引用；
- Interview Support Assignee 可查看完成面试辅助所需的最小背景摘要；
- 不能因为能看到 Task 就自动获得整个 Case 或所有文件权限。

### 2.2 Contractor 单任务工作区

Contractor 只能进入当前有效 `interview_support` Task：

- 目标学校；
- 面试时间、方式、语言；
- Primary Advisor 提供的必要辅导要求；
- due_at、当前 Assignment 和允许动作。

Contractor 不显示 Student/Guardian 联系方式、完整 Assessment、Case 文件、其他学校、其他 Task 或其他 Case。

## 3. 正式 API 路径

| Method | 路径 | 类型 | 用途 |
| --- | --- | --- | --- |
| `GET` | `/api/v1/tasks` | query | 按当前授权列出内部 Task 摘要 |
| `GET` | `/api/v1/tasks/:taskId` | query | 读取当前用户可访问的 Task 详情 |
| `GET` | `/api/v1/tasks/assigned` | query | 读取当前 User 当前 Assignment 的任务 |
| `GET` | `/api/v1/contractor/tasks/:taskId/workspace` | query | Contractor 单任务脱敏 DTO |
| `POST` | `/api/v1/tasks/:taskId/assignment/accept` | command | 当前 Assignee 接受 Assignment |
| `POST` | `/api/v1/tasks/:taskId/assignment/reject` | command | 当前 Assignee 拒绝 Assignment |
| `POST` | `/api/v1/tasks/:taskId/complete` | command | 当前 accepted Assignee 提交完成资料 |
| `POST` | `/api/v1/tasks` | command | 当前 Primary Advisor 创建 manual Task |
| `POST` | `/api/v1/tasks/:taskId/reassign` | command | 当前 Primary Advisor 重派 Task |
| `POST` | `/api/v1/tasks/:taskId/cancel` | command | 当前 Primary Advisor 或 Cases 受信事件取消 Task |

正式 API 只使用 `/api/v1/**`，页面通过 typed `client.ts` 调用，不能直接手写不同响应 shape。

## 4. 查询 DTO

内部 Task 摘要只返回当前授权所需字段：

```text
task: id, task_type, title, state, due_at, is_overdue, record_version
case: opaque id + 受控摘要（按当前 Case scope）
school_target: opaque id + 受控学校引用（有授权时）
assignment: assignee role、assignment status、assigned_at、accepted_at
allowed_actions: 当前用户可执行的固定 action code
completion_summary: 已完成时的受控摘要，不返回文件正文
```

Contractor DTO 进一步收敛为：

```text
task: id, title, task_type=interview_support, state, due_at
school: 目标学校的必要引用
interview: 时间、方式、语言
brief: 脱敏辅导要求
assignment: 当前 Assignment 和允许动作
```

不返回任意数据库 row、Assessment 原文、Guardian 资料、内部备注、文件 URL、对象 key、其他任务或其他案件。

## 5. Task 状态与页面动作

| 当前状态 | 当前用户 | 页面允许动作 | 成功后 |
| --- | --- | --- | --- |
| `assigned` | 当前 Assignee | 接受、拒绝 | `accepted` 或 `awaiting_reassignment` |
| `assigned` | Primary Advisor | 查看、重派、取消 | 新 Assignment 或 `cancelled` |
| `accepted` | 当前 Assignee | 完成 | `completed`，发布完成事实 |
| `accepted` | Primary Advisor | 重派、取消 | 旧 Assignment 结束或 `cancelled` |
| `awaiting_reassignment` | Primary Advisor | 重派、取消 | 新 Assignment 或 `cancelled` |
| `completed` | 所有人 | 只读历史 | 终态 |
| `cancelled` | 所有人 | 只读历史 | 终态 |

页面根据服务端返回的 `allowed_actions` 渲染按钮；按钮隐藏不替代服务端权限检查。

## 6. 完成表单

### 6.1 `application_prepare_submit`

完成表单必须包含：

- 正式提交时间；
- 提交渠道；
- 实际提交人；
- 当期材料清单完成状态/快照；
- 学校参考号，或“学校未提供参考号”声明；
- 无参考号时至少一个 Case Document opaque ID 凭证引用。

Documents 和 Tasks 重新验证文件归属、SchoolTarget 用途、clean/available/未 revoked 和当前 Assignee 权限。

完成成功只表示 Tasks 已接受提交事实；Cases 接收 `tasks.application_submission_completed` 后重新检查并推进 Target。

### 6.2 `interview_support`

完成表单只保存：

- 辅助完成时间；
- 面试方式/语言确认；
- 必要辅导完成摘要。

不保存面试结果，也不允许通过完成该 Task 直接改变 SchoolTarget。

### 6.3 `manual`

保存完成时间和简短完成说明，不允许包含自动推进 Case 或 SchoolTarget 的字段。

## 7. 写命令通用规则

所有命令必须同时使用：

```text
Idempotency-Key: opaque-retry-key
expected_record_version: current version
```

- 服务端重新检查 User、Membership、RoleBinding、Case、Task、Assignment 和 Task 类型。
- 同一 key + 同一 payload 重试返回原结果；不同 payload 返回 `IDEMPOTENCY_KEY_REUSED`。
- 版本冲突返回 `STALE_VERSION`，提示页面刷新，不覆盖其他人的修改。
- 完成、重派、取消、拒绝写入 AuditEvent；必要异步事实写入 Outbox。
- Task 不直接写 Cases；Cases 不直接写 Tasks。

## 8. 页面边界状态

| 状态 | 页面行为 |
| --- | --- |
| `loading` | 显示加载状态，不猜测 Task 状态 |
| `success` | 显示当前授权 DTO 和 allowed actions |
| `empty` | 当前没有任务，不等于无权 |
| `denied` | 统一无权限提示，不泄露其他 Task 是否存在 |
| `not_found` | 仅在服务端允许区分时显示 |
| `stale_version` | 重新加载并要求用户重新提交动作 |
| `unavailable` | Tasks runtime/依赖不可用，不显示成功 |
| `error` | 只显示安全错误 code，不显示 stack/SQL/provider 内容 |

## 9. Notifications 与 Operations

- Task 创建、重派、拒绝、到期、逾期和结案选择 effect 由 Notifications 消费；页面不直接创建通知。
- due_at 和 overdue 从 Tasks 当前事实计算，不能由页面本地计算后写回。
- Operations 只能展示 Task backlog/overdue projection，不能成为 Task 授权或状态来源。
- Task 页面显示“有待办事项”通知时，点击仍要重新授权目标 Task。

## 10. 本模块待确认内容

请确认以下 6 点：

1. 内部 Task 工作台和 Contractor 单任务工作区分开，Contractor 不能进入完整 Case 工作台。
2. 正式 API 只使用 `/api/v1/**`、typed client、Idempotency-Key 和 expected version。
3. Task 完成表单按三种 Task 类型分别校验；申请提交必须有参考号或替代凭证。
4. Task 完成只发布事实，由 Cases 重新校验后推进 SchoolTarget。
5. 页面根据服务端 allowed actions 和边界状态渲染，不用本地状态替代权限或业务状态。
6. Task 的通知、逾期和 Operations 信息都通过对应模块公开查询，不跨模块直接写表。

本纵向切片已确认。下一步进入 Documents 文件工作台 API/UI 设计。
