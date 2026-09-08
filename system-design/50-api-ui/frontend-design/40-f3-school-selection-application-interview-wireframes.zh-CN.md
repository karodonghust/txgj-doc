# F3 选校、逐校申请与面试低保真线框及交互契约

> 状态：`approved`
> Owner：frontend-design
> Architect 复审：2026-08-26 通过；D1–D5 transport contract 已冻结，`decision_count=0`
> 范围：Release 1 / F3；内部 ERP 的候选学校名单、逐校申请、面试 Task
> 日期：2026-08-26
> 非范围：产品源码、API 实现、数据库、migration、测试、Documents 页面、Schools 资料治理、Notifications 页面、Guardian Portal、F4/F5

## 0. 一页总览

### 0.1 本轮目标

本文件定义共享 Case layout 下三个独立子路由的低保真结构、页面连续性、权限边界、操作反馈和最小 target contract：

- `/cases/:caseId/schools`：Primary Advisor 建立候选学校名单版本；Founder 审批/驳回；Primary Advisor 通过电话、微信或面谈代录 Guardian 对同一 approved version 的确认。
- `/cases/:caseId/applications`：逐校 SchoolTarget、Application Assignee、准备并提交申请 Task、提交凭证摘要和所有允许状态。
- `/cases/:caseId/interviews`：需要面试的 SchoolTarget、面试事实摘要、Interview Support Task 和 Advisor/Contractor 辅助分派。

所有页面只消费 Cases、Tasks、Schools 和 Documents 的已批准公开查询/命令。页面不写其他模块私表，不本地推断状态，不把 Task 完成直接当作 SchoolTarget 状态变化。

### 0.2 已确认的业务连续性

~~~text
/cases/:caseId
  |-- /schools
  |     |-- Primary Advisor 新建完整名单版本
  |     |-- Founder 批准 / 驳回修改
  |     |-- Primary Advisor 代录 Guardian 对同一版本确认
  |     `-- 确认名单变化 -> 保留/新增/撤回 SchoolTarget 与 Task
  |
  |-- /applications
  |     |-- preparing -> 一个幂等“准备并提交申请”Task
  |     |-- Assignee 完成并提交时间/渠道/凭证
  |     `-- Cases 重验后才可进入 submitted
  |
  `-- /interviews
        |-- 学校要求面试 -> submitted -> interview
        |-- 指派 Advisor Collaborator 或 Contractor Interview Support Task
        |-- Primary Advisor 可自己完成
        `-- Task 完成只发布事实；Cases 再重验 SchoolTarget
~~~

名单版本变化的差异规则：不变学校保留原 SchoolTarget、Application Assignee 和 Task；新增学校在同一版本完成两层确认后进入 `preparing` 并请求申请 Task；移除进行中的学校进入 `withdrawn` 并请求取消未完成 Task；已终态学校保留原结果。所有学校 `rejected` 时不自动结案，Founder 只能选择建立新名单版本或进入既有 `/cases/:caseId/close` 明确结案流程。

### 0.3 明确禁止

- 不新增 Application、Offer、Material、SubmissionEvidence、InterviewResult 或新的基础角色实体。
- 不把 `accepted` 当终态；只有 `offer_confirmed`、`offer_declined`、`rejected`、`withdrawn` 是终态。
- 不让 Guardian、Portal 或 Student 写入确认或 offer 决定；F3 只设计员工内部代录。
- 不让 Contractor 进入 Case、Schools、Applications 或 Interviews 页面；Contractor 只进入当前有效的单一 `interview_support` Task 脱敏工作区。
- 不在 F3 设计 Documents 上传/扫描/下载页面；申请凭证只显示已授权的 opaque Document 引用和安全状态。
- 不设计 Schools 资料变更、Crawler、学校治理、站内通知列表或 Portal 页面。

## 1. 事实来源、路由和入口

### 1.1 规范性来源

- [本地执行约束](/Users/karo/Documents/Tianxingguoji/AGENT.md)
- [BR-Cases（BR-030、031、032、033、034、039）](../../../business-requirements/30-cases.zh-CN.md)
- [BR-Tasks（BR-035、036、037）](../../../business-requirements/40-tasks.zh-CN.md)
- [BR-Notifications/Audit（BR-038、070、071）](../../../business-requirements/70-notifications-audit.zh-CN.md)
- [F1 Today / Cases / Workspace（approved）](./20-f1-today-cases-wireframes.zh-CN.md)
- [F2 CRM / Case Intake（approved）](./30-f2-crm-case-intake-wireframes.zh-CN.md)
- [Cases 案件工作台 API/UI（approved）](../10-cases-workspace.zh-CN.md)
- [Task 工作台 API/UI（approved）](../20-task-workspace.zh-CN.md)
- [Notifications 入口 API/UI（approved，F3 只引用副作用）](../40-notifications-ui.zh-CN.md)
- [授权模型（approved）](../../40-permissions-security/10-authorization-model.zh-CN.md)
- [Cases 模块契约（approved）](../../10-module-contracts/50-cases.zh-CN.md)

### 1.2 路由基线

| 路由 | 目标用途 | 进入 | 返回 |
|---|---|---|---|
| `/cases/:caseId/schools` | 名单版本、Founder 审批、Guardian 代录确认、名单差异 | Case summary 的 Schools tab、Today 风险/下一步 | 返回 Case summary；保留来源上下文 |
| `/cases/:caseId/applications` | 逐校 Target 和申请 Task | Schools 确认后的下一步、Case summary、Task 来源 | 返回 Case applications；Task 返回原 Target |
| `/cases/:caseId/interviews` | 面试 Target 和 Interview Support Task | Applications 的 interview 行、Case summary、内部 Task 来源 | 返回 Case interviews；Contractor 不进入此路由 |
| `/cases/:caseId/close` | Founder 既有人工结案页；F3 只提供入口连续性 | 所有学校拒绝选择区、Case summary | 结案成功回 `/cases/:caseId` |
| `/tasks/:taskId` | 内部 Task 工作台 | Applications/Interviews 行 | 完成/拒绝/重派后回原来源 |
| `/contractor/tasks/:taskId` | Contractor 单 Task 脱敏工作区 | Contractor `/tasks` 或受控 Task 入口 | 完成/拒绝后回 `/tasks` |

Case 页面仍使用 canonical `/cases/:caseId` 和独立子路由，不使用 query tab。旧 `/cases/:caseId/workspace` 仍由 F1 规则 308 重定向到 canonical；F3 不复制第二套 layout。

### 1.3 角色和入口矩阵

| 角色/组合 | `/schools` | `/applications` | `/interviews` | 明确拒绝 |
|---|---|---|---|---|
| Founder | 查看组织授权范围；approve/reject 当前名单版本；查看 Target/Task；所有学校拒绝时进入 close 分支 | 查看逐校结果和 Task 事实；不代替 Assignee 完成申请 | 查看授权 Case 的面试摘要；不获取 Contractor 专属额外字段 | 不因 Founder 自动获得文件下载或 Task 完成权 |
| Advisor / Primary Advisor | 建立名单版本；提交 Founder 审批；代录 Guardian 确认；查看/处理自己 Case | 作为 Application Assignee 准备/提交；有明确授权的 Advisor Collaborator 可完成被分派 Task | 处理自己 Case；可自己完成 Interview Support Task 或指派 Advisor/Contractor | 不能批准自己的名单；不能跳过同版本 Guardian 确认；不能直接改 Target 状态 |
| Advisor Collaborator | 默认不能创建名单或代录确认，除非被 owner capability 明确允许的页面动作返回 | 只能访问被授权 Case/Target 和当前 Assignment；可成为 Application Assignee | 只能访问被授权 Case/Task；可作为辅助人 | 不能扩大 Case、Assessment、文件或其他学校范围 |
| Admin 单独 | denied | denied | denied | 不因 Admin 账号管理能力读取客户、Case、Task 或学校申请 |
| Founder + Admin | 使用 Founder 的业务 capability，并保留 Admin 管理入口 | 同 Founder | 同 Founder | Admin capability 不削减 Founder，也不凭 Admin 增加业务范围 |
| Contractor | denied | denied | denied | 只能通过当前有效 `interview_support` Assignment 进入单 Task 脱敏 DTO |

服务端每次请求重新检查 User、Membership、全部 RoleBinding、Capability、Case 关系、ScopeGrant、SchoolTarget/Task Assignment 和 expected version。前端隐藏按钮不是授权证据。Portal、Guardian Viewer 不出现在 F3 内部 ERP 页面。

## 2. `/cases/:caseId/schools` 候选学校名单

### 2.1 页面结构和门槛

名单页面按四段展示：

1. Case header：Student 受控摘要、阶段、Primary Advisor、当前 record version。
2. Current version：当前完整学校集合、学校 pinned reference、版本和确认状态摘要。
3. Version history/diff：可打开旧版本和新旧集合差异；旧版本只读。
4. Allowed action area：由服务端 `allowed_actions` 决定新建、提交、Founder review、Guardian decision 或进入新名单分支。

创建或提交前置条件：Assessment 使用绑定的 approved manifest；background blocker 和 school selection blocker 满足；当前用户是该 Case 的 Primary Advisor。学校选择只消费 Schools owner 提供的 approved/active `SchoolReferencePin` 选项，不在 F3 改学校资料。

名单版本必须保存：版本号、完整学校集合、每校 SchoolReferencePin、创建人、创建时间、变更说明、Founder 对同一版本的 approve/reject 操作人/时间/理由/版本，以及 Guardian 确认人、结果、时间、渠道、代录人和对应 Founder approval 记录。

### 2.2 W01：Advisor 桌面名单编辑/版本页

~~~text
+----------------------------------------------------------------------------------+
| < Case 受控摘要       Schools / 候选学校名单                    [返回 Case]     |
| Stage: school_selection_confirmed?   Primary Advisor: A. Lee   Version: v4       |
|----------------------------------------------------------------------------------|
| 当前版本 v4 · [待 Founder 审批 / 已批准 / 待 Guardian 确认 / 已确认]              |
| 创建人 A. Lee · 创建时间 · 变更说明：新增两所九龙学校                               |
| Founder approval：记录摘要 / Guardian decision：记录摘要                        |
|----------------------------------------------------------------------------------|
| 学校集合（完整版本，不显示未经授权的学校字段）                       [新建版本]  |
| #  SchoolReferencePin       地区/层次摘要       版本差异        Target/Task     |
| 1  School A                 controlled ref       不变            保留            |
| 2  School B                 controlled ref       新增            Guardian 确认后准备 |
| 3  School C                 controlled ref       移除            withdrawn       |
|----------------------------------------------------------------------------------|
| [查看上一版本] [查看差异]                                                        |
|                            [取消] [提交 Founder 审批]                            |
+----------------------------------------------------------------------------------+
~~~

- 学校行只显示 Schools owner 的 allowlisted reference、必要申请摘要和差异标签，不显示 Crawler 原文。
- `新建版本` 复制当前集合为新编辑态；不得原地覆盖已保存版本。
- 提交后页面锁定该版本的集合编辑，等待 Founder 决定；失败不显示假成功。

### 2.3 W02：Advisor 移动名单编辑/版本页

~~~text
+--------------------------------------+
| < Case              Schools       ⋮  |
| Stage: 选校确认 · Primary: A. Lee    |
|--------------------------------------|
| 当前版本 v4                          |
| 待 Founder 审批                      |
| 创建人 A. Lee · 变更说明             |
|--------------------------------------|
| School A                         >   |
| controlled ref · 不变 · 保留         |
|--------------------------------------|
| School B                         >   |
| controlled ref · 新增 · 等待 Guardian 确认 |
|--------------------------------------|
| School C                         >   |
| 移除 · 将 withdrawn · 保留历史       |
|--------------------------------------|
| [查看差异] [新建版本]                |
| [提交 Founder 审批]                  |
+--------------------------------------+
~~~

窄屏不使用横向表格；版本、集合、差异和允许动作按纵向 section 呈现。底部主按钮在提交期间保持稳定宽度。

### 2.4 W03：Founder 审批桌面页

~~~text
+----------------------------------------------------------------------------------+
| < Case / Schools       Founder 审核名单版本 v4                                  |
| Student 受控摘要 · Primary Advisor: A. Lee · 提交时间 · record_version          |
|----------------------------------------------------------------------------------|
| 版本元数据                                                                       |
| 完整学校集合 3 所 · 变更说明 · 与 v3 差异                                        |
| School A  不变 / 保留      School B  新增      School C  移除并将 withdrawn       |
|----------------------------------------------------------------------------------|
| 审批记录（同一版本）                                                             |
| 当前 Founder approval：待决定                                                   |
| Guardian decision：尚未开始（批准后由 Primary Advisor 代录）                    |
| 理由（驳回时必填） [_______________________________________________]             |
|----------------------------------------------------------------------------------|
| [返回只读版本]                              [驳回修改] [批准此版本]              |
+----------------------------------------------------------------------------------+
~~~

- Founder 批准只绑定当前 immutable version 和 expected record version；不可批准页面显示的另一个版本。
- 驳回必须保存受控理由/必要说明，并回到 Advisor 新建版本路径；不修改原版本集合。
- 批准后显示“待 Guardian 确认”，不自动创建 Target、不自动结案、不由 Founder 代录 Guardian。

### 2.5 W04：Founder 审批移动页

~~~text
+--------------------------------------+
| < Case / Schools   审核 v4           |
| Lin Chen · A. Lee                    |
|--------------------------------------|
| 变更说明                             |
| 新增 School B，移除 School C         |
|--------------------------------------|
| 学校集合                             |
| A · 不变 · 保留                      |
| B · 新增 · 等待 Guardian 确认         |
| C · 移除 · withdrawn 后保留历史      |
|--------------------------------------|
| 理由（驳回必填）                     |
| [____________________________]       |
| [驳回修改]        [批准此版本]       |
+--------------------------------------+
~~~

驳回按钮先验证理由；批准/驳回结果显示在同一版本记录区。移动端返回仍回 Case Schools，而不是回到全局 Schools 治理页。

### 2.6 W05：Primary Advisor 代录 Guardian 确认桌面页

~~~text
+----------------------------------------------------------------------------------+
| < Case / Schools       代录 Guardian 确认 · 名单版本 v4                          |
| Founder approval：已批准 · approval record/version：同一 v4                    |
|----------------------------------------------------------------------------------|
| Guardian 确认事实（由 Primary Advisor 通过平台外渠道取得）                       |
| Guardian * [选择当前 related Guardian v]                                         |
| 结果 *    ( ) confirm   ( ) not_confirmed                                       |
| 渠道 *    ( ) 电话       ( ) 微信       ( ) 面谈                                 |
| 实际确认时间 * [yyyy-mm-dd hh:mm Asia/Hong_Kong]                                 |
| 代录人      A. Lee（当前 Primary Advisor，只读）                                |
| 备注/理由 [_______________________________________________]                     |
|----------------------------------------------------------------------------------|
| 绑定版本：v4 · 学校集合 hash/摘要 · Founder approval record                      |
| [取消]                                                  [保存 Guardian 决定]       |
+----------------------------------------------------------------------------------+
~~~

- 只列 Student 当前相关且服务端允许的 Guardian；主要联系人不自动等于最终确认人。
- `confirm` 必须绑定同一 Founder-approved v4；`not_confirmed` 退回修改路径，不产生学校申请 Task。
- 保存记录包含版本、学校集合、Guardian、结果、时间、渠道、代录人、Founder approval record；不调用 Portal 写入。

### 2.7 W06：Primary Advisor 代录 Guardian 确认移动页

~~~text
+--------------------------------------+
| < Schools          Guardian 确认     |
| Founder approved · v4                |
|--------------------------------------|
| Guardian * [选择 v]                  |
| 结果 *                              |
| ( ) 已确认    ( ) 未确认             |
| 渠道 *                              |
| ( ) 电话 ( ) 微信 ( ) 面谈           |
| 时间 * [2026-08-26 10:30 HKT]        |
| 代录人 A. Lee（只读）                |
| 说明 [________________________]      |
|--------------------------------------|
| 版本 v4 · Founder approval 已绑定    |
| [取消]          [保存 Guardian 决定] |
+--------------------------------------+
~~~

移动端确认页明确写“代录”，不出现 Portal 登录、Guardian 账号或外部发送按钮。保存后显示 receipt 和下一步：确认成功进入 Applications，未确认进入新名单版本。

## 3. `/cases/:caseId/applications` 逐校申请

### 3.1 SchoolTarget 状态和责任

F3 只显示 BR-034 已确认状态，不新增状态：

~~~text
candidate -> preparing -> submitted
submitted -> interview                         （学校明确要求面试）
submitted / interview -> waitlisted / accepted / rejected
waitlisted -> accepted / rejected
accepted -> offer_confirmed / offer_declined
preparing / submitted / interview / waitlisted -> withdrawn
~~~

终态为 `offer_confirmed`、`offer_declined`、`rejected`、`withdrawn`。`accepted` 仍等待 Primary Advisor 通过人工渠道取得 Guardian offer 决定；决定记录包含 Guardian、结果、时间、渠道和代录人。F3 不让 Guardian 或 Portal 直接写入。

Application Assignee 只能是当前 Primary Advisor 或有明确 Case 授权的 Advisor Collaborator。每所学校、每一申请轮次只有一条幂等“准备并提交申请” Task；Assignee 同时负责准备和提交，不拆成两个 Task。Contractor 不能成为 Application Assignee。

### 3.2 W07：Applications 桌面页

~~~text
+----------------------------------------------------------------------------------+
| < Case / Applications                                      [返回 Case]          |
| Student 受控摘要 · Stage: application_in_progress · Primary: A. Lee             |
|----------------------------------------------------------------------------------|
| 逐校申请（SchoolTarget 权威状态）                         [查看名单版本]         |
| 筛选：[全部状态 v] [Assignee v] [只看我的 Task]                                  |
|----------------------------------------------------------------------------------|
| 学校/Target       状态              Assignee          Task/截止          操作      |
| School A          preparing         A. Lee             assigned · due      [打开]  |
| School B          submitted         A. Lee             completed           [查看]  |
| School C          interview         K. Chan            assigned · due      [面试]  |
| School D          accepted          A. Lee             —                   [代录 offer]|
| School E          rejected          A. Lee             —                   [查看历史]|
|----------------------------------------------------------------------------------|
| Target 详情 / 允许动作                                                         |
| School A · preparing · SchoolReferencePin 受控摘要                              |
| 申请 Task：准备并提交申请 · current Assignment · allowed actions               |
| 材料清单：按学校当期要求的完成摘要（不展示 Documents 页面）                    |
| [打开 Task] [进入面试页] [记录 Guardian offer 决定]                             |
|----------------------------------------------------------------------------------|
| 全部学校拒绝时：不自动结案                                                       |
| Founder 可 [建立新名单版本] 或 [进入人工结案]                                   |
+----------------------------------------------------------------------------------+
~~~

- 行状态由 Cases `listSchoolTargets` 返回；Task 状态和 `allowed_actions` 由 Tasks 返回，页面不把 `assigned/accepted/completed` 映射为 SchoolTarget 状态。
- `preparing` 行必须显示申请 Task 的创建/当前 Assignment 摘要；没有 Task receipt 时显示 unavailable/冲突，不显示“准备中已完成”。
- `submitted` 只显示 Cases 已重验的提交事实摘要：提交时间、渠道、提交人、清单完成状态、学校参考号或无参考号声明和凭证引用状态。
- `accepted` 的主操作是进入 Guardian offer 代录，不是“接受 offer”按钮由页面自动推进。
- 所有学校为 `rejected` 时显示两条 Founder 分支；Advisor 只能查看或建立新名单（若服务端允许），不能代替 Founder 结案。

### 3.3 W08：Applications 移动页

~~~text
+--------------------------------------+
| < Case        Applications      ⋮    |
| Stage: Application in progress       |
|--------------------------------------|
| [全部状态 v] [Assignee v]            |
|--------------------------------------|
| School A                             |
| Preparing · A. Lee                   |
| 申请 Task · assigned · due           |
| [打开 Task]                       >  |
|--------------------------------------|
| School B                             |
| Submitted · 已完成提交事实           |
| [查看]                             >  |
|--------------------------------------|
| School C                             |
| Interview · K. Chan                  |
| [进入面试]                         >  |
|--------------------------------------|
| School D                             |
| Accepted · 等待 Guardian 决定        |
| [代录 offer 决定]                   >|
|--------------------------------------|
| 全部拒绝：Founder 分支                 |
| [新名单版本]       [人工结案]         |
+--------------------------------------+
~~~

移动端把 Target 详情作为可展开行或全屏详情，不横向压缩 Assignee、Task 和状态。提交凭证只显示受控摘要和允许动作，不显示对象 key、下载 URL 或文件内容。

### 3.4 申请 Task 完成连续性

~~~text
Guardian confirmed same list version
  -> Cases: new SchoolTarget enters preparing
  -> Cases: idempotent task request
  -> Tasks: one application_prepare_submit Task
  -> Assignee accepts / rejects / is reassigned
  -> Assignee completes time + channel + submitter + checklist + reference/evidence
  -> Tasks publishes completion fact
  -> Cases rechecks Target, Assignee, version and evidence
  -> submitted OR stable conflict/error; never local success
~~~

申请完成表单的最小字段来自 approved Task UI：提交时间、提交渠道、实际提交人、当期学校材料清单完成状态/快照、学校参考号或明确“无参考号”声明，以及至少一个其他提交凭证的 opaque Document ID。F3 不设计凭证上传或扫描；Documents owner 负责 clean/available/not revoked 检查。

## 4. `/cases/:caseId/interviews` 面试辅助

### 4.1 页面边界

只有学校明确要求面试时，Cases 才把 `submitted` 目标推进到 `interview` 并请求 Interview Support Task。Task 默认由 Primary Advisor 负责，也可以单一 Assignment 指派已授权 Advisor Collaborator 或 Contractor。

Interview Support Task 只暴露：目标学校、面试时间、方式、语言、Primary Advisor 必要辅导摘要、due_at、Assignment 和 allowed actions。Advisor 在 Case 内可看所授权的 Target；Contractor 不看完整 Assessment、Guardian 联系方式、Case 文件、其他学校、其他 Task 或其他 Case。

### 4.2 W09：Interviews 桌面页

~~~text
+----------------------------------------------------------------------------------+
| < Case / Interviews                                         [返回 Case]          |
| Student 受控摘要 · Primary Advisor: A. Lee                                      |
|----------------------------------------------------------------------------------|
| 面试目标（只显示需要面试的 SchoolTarget）                                       |
| 学校              Target 状态      时间/方式/语言        Support Assignee  Task  |
| School C          interview        09-10 · 面谈 · 粤语   K. Chan          assigned|
| School F          submitted        学校未要求面试         —                —      |
|----------------------------------------------------------------------------------|
| School C 面试详情                                                                |
| SchoolReferencePin：受控摘要 · Target record_version                              |
| 面试时间 [09-10 16:00 HKT] · 方式 [面谈] · 语言 [粤语]                           |
| 必要辅导摘要（按 Case/Task scope）                                               |
| Support Assignee [Advisor Collaborator / Contractor v]                            |
| 当前 Task：interview_support · assigned · due_at · allowed actions               |
| [打开内部 Task] [重派/取消（Primary Advisor）] [自己完成]                        |
|----------------------------------------------------------------------------------|
| 完成辅助不代表学校结果；结果由 Cases 后续事实重验，不在此按钮自动改变 Target。    |
+----------------------------------------------------------------------------------+
~~~

- 未要求面试的 `submitted` 目标不显示为可创建面试 Task 的“异常”；它只是跳过 interview。
- 指定 Contractor 时只创建/重派该 Interview Support Task，不授予 Case Collaborator、Application Assignee 或案件成员关系。
- Primary Advisor 选择“自己完成”时复用同一 Task，不另建角色或替代 Task。
- Task 完成后行显示“辅助完成事实已发布，等待 Cases 重验”，不把 Target 自动改成 `waitlisted`、`accepted` 或其他结果。

### 4.3 W10：Interviews 移动页

~~~text
+--------------------------------------+
| < Case          Interviews       ⋮   |
|--------------------------------------|
| School C                            |
| Target: Interview                   |
| 09-10 16:00 HKT · 面谈 · 粤语       |
|--------------------------------------|
| 必要辅导摘要                         |
| 受控、最小范围内容                   |
|--------------------------------------|
| Support Assignee                    |
| K. Chan · Advisor Collaborator       |
| Task: interview_support · assigned  |
| [打开 Task] [重派] [自己完成]        |
|--------------------------------------|
| 这不是学校结果录入。                 |
| Task 完成后由 Cases 重验。            |
+--------------------------------------+
~~~

Contractor 永远不从本页打开；其入口是 `/tasks` 到当前 `/contractor/tasks/:taskId`，返回 `/tasks`，并接受服务端的单 Task 脱敏 DTO。

## 5. 主要操作矩阵

所有写按钮由服务端返回的 `allowed_actions` 控制；每个写命令使用 `Idempotency-Key` 和相关 `expected_record_version`。成功反馈必须同时满足 owning module 事实、AuditEvent 和必要 Outbox 结果；页面不能只显示本地 toast。

| 操作 | 按钮位置 | 前置条件/授权 | 成功反馈 | 失败、冲突和返回 |
|---|---|---|---|---|
| 新建名单版本 | `/schools` 当前版本区 `[新建版本]` | Primary Advisor；Assessment background/school-selection blocker 满足；Case 未关闭 | 新版本编辑态；旧版本只读；显示版本和变更说明 | `VALIDATION_FAILED` 就地字段错误；denied 不泄露 Case；返回当前 Schools |
| 添加/移除学校到新版本 | 新版本编辑区集合操作 | 仅编辑中的新版本；学校来自 approved/active pin options；不覆盖旧版本 | 差异标签更新；保存草稿 receipt（若契约提供） | 学校 pin stale/unavailable 时保留草稿并要求刷新；不改旧 Target |
| 提交 Founder 审批 | Advisor 页面底部主按钮 | 当前 Primary Advisor；完整学校集合；版本未提交 | 显示等待 Founder 决定；通知由 Notifications 消费，不在 F3 直接发 | `STALE_VERSION` 重新加载；`CONFLICT` 保持编辑态；返回 Case Schools |
| Founder 批准 | Founder review 底部主按钮 | Founder；当前 version；expected version；名单合法 | 版本显示 approved；开放 Primary Advisor Guardian 代录入口 | 版本冲突要求刷新；审计/Outbox 失败显示 error；不创建 Target 或 Task 假成功 |
| Founder 驳回修改 | Founder review 底部次按钮 | Founder；驳回理由满足命令要求 | 原版本保留；显示修改理由；Advisor 可新建版本 | 理由校验失败聚焦字段；版本冲突刷新；返回只读 review |
| 代录 Guardian 确认 | `/schools` Guardian decision 区 | Primary Advisor；Founder approved 同一版本；当前 related Guardian；结果、渠道、时间齐全 | `confirm` 显示同版确认记录和进入 Applications；`not_confirmed` 显示新名单版本入口 | 版本不一致、Guardian 关系失效或 stale 要求刷新；不调用 Portal；返回 Schools |
| 应用已确认名单差异 | confirmed receipt 后 Cases owning action | 同一 approved version + Guardian confirmation；服务端计算差异 | 不变 Target/Task 保留；新增 Target preparing 并请求一条申请 Task；移除进行中 Target withdrawn 并请求取消 Task | 任一副作用 unavailable/partial 显示“处理中/需重试”安全状态；幂等重试不重复 Target/Task |
| 所有学校拒绝分支 | Applications 底部 Founder 区 | 所有当前相关 Target 为 rejected；没有自动结案 | Founder 选择新名单版本或进入 `/close`；保留逐校结果 | 非 Founder 隐藏/denied；状态变化重验；不自动 close |
| 接受/拒绝 Application Task | `/tasks/:taskId` 或 Applications Task 行 | 当前 Assignee；Task `assigned`；允许动作 | `accepted` 或 `awaiting_reassignment`；来源回 Applications | 拒绝要求原因；stale 刷新 Task；Contractor 不可访问 application Task |
| 完成申请提交 Task | Task 完成表单主按钮 | Task `accepted`；Assignee；时间、渠道、提交人、清单、参考号/替代凭证齐全 | Tasks `completed`；Cases 异步重验后显示 submitted | `VALIDATION_FAILED` 保留表单；凭证不可用拒绝；Cases 重验冲突不推进 Target |
| 进入 interview | Applications Target 允许动作 | 学校明确要求面试；Cases 当前 Target `submitted`；owner command | Target 显示 `interview`；请求一个 Interview Support Task | 不满足学校事实/版本冲突则不创建 Task；返回 Applications |
| 分派面试辅助人 | Interviews Assignee 下拉/重派菜单 | Primary Advisor 或受信 Cases action；Advisor Collaborator 有授权，Contractor 为当前合资格单 Task | 保留 Assignment 历史；新 Assignee 重新接受；同一 Task 不新建替代 Task | 无效角色/Assignment denied；stale 刷新；返回 Interviews |
| 完成 Interview Support Task | 内部 Task 或 Contractor Task 主按钮 | 当前 accepted Assignee；最小面试摘要；完成表单不含学校结果 | Tasks completed；显示辅助完成事实，等待 Cases 重验 | 失败不改 Target；Contractor 只回 `/tasks`；不可用不显示成功 |
| 代录 Guardian offer 决定 | Applications `accepted` 行 | Primary Advisor；Target accepted；通过电话/微信/面谈取得 Guardian 决定 | `offer_confirmed` 或 `offer_declined` 记录 Guardian、结果、时间、渠道、代录人 | stale/Guardian 关系变化要求刷新；不让 Guardian/Portal 直接写入 |
| 进入人工结案 | Applications 所有拒绝分支 | Founder；BR-039 前置由 `/close` owning API 重验 | F3 返回 `/close`；结案成功回 Case summary | F3 不自行 close；条件不足显示缺口和返回 Applications |

### 5.1 统一写入反馈

~~~text
loading  -> 主按钮锁定、保持尺寸、显示进行中
success  -> owning module receipt + 最新 DTO + 可识别的下一步链接
VALIDATION_FAILED -> 顶部摘要 + 字段级错误 + 聚焦首个问题
FORBIDDEN / NOT_FOUND -> 通用 denied/not_found，不泄露资源事实
STALE_VERSION -> 显示当前版本摘要，要求刷新后重新确认
CONFLICT -> 显示稳定业务冲突和可行动回退，不自动重放
SERVICE_UNAVAILABLE -> 显示暂不可用和安全重试，不显示假成功
~~~

## 6. 页面状态矩阵

| 页面/区块 | loading | empty | validation | denied | unavailable/error | stale/conflict | success/返回 |
|---|---|---|---|---|---|---|---|
| `/schools` Advisor | 版本、学校行和允许动作骨架 | 没有当前名单版本且具备 create capability：显示“新建名单版本”；无权限不是 empty | 学校集合不能为空；重复学校 pin、无效变更说明就地提示 | 非当前 Primary Advisor 显示通用 denied，不显示名单 | Schools/Cases 查询不可用，保留已读摘要并提供重试；不显示空集合假数据 | 版本变化后重载当前和编辑草稿；禁止静默覆盖 | 新版本/提交/确认后显示最新 receipt，返回同一 Schools 并保留 Case 来源 |
| Founder review | 版本元数据和集合骨架 | 没有待 Founder 决定版本：显示空态；不把无权当空态 | 驳回理由字段错误 | 非 Founder 统一 denied | 当前版本依赖不可用，不能显示批准按钮 | expected version 不符时重新读取版本和 approval record | approve/reject 记录后回 Schools，显示对应下一步 |
| Guardian 代录区 | Founder approval、Guardian options、表单骨架 | 未有 approved version 时不显示确认表单；无 related Guardian 显示阻断原因 | Guardian、结果、渠道、Asia/Hong_Kong 时间必填 | 非 Primary Advisor 不能代录 | Guardian 关系或 Cases 查询不可用，表单不提交 | approval version 或 Guardian relation 变化要求刷新 | confirm 进入 Applications；not_confirmed 进入新名单版本 |
| `/applications` | Target 行、Task summary、allowed actions 骨架 | 尚无 confirmed Target：显示“完成同版 Guardian 确认后生成”；没有 Task 不等于 denied | 完成 Task 字段错误；offer 决定字段错误 | 无 Case/Target 关系统一 denied；Contractor 不进此路由 | Tasks/Cases/Documents reference 不可用，区分服务错误和无 Task | Target/Task version 冲突分别刷新 owner DTO，不改另一个模块事实 | Task receipt/Target receipt 后回 Applications；Task 来源返回原行 |
| `/interviews` | 需要面试的 Target、Interview Task 骨架 | 没有学校要求面试：明确 empty；不把 submitted 当异常 | 时间、方式、语言、Assignee 必填（以冻结 DTO 为准） | 无 Case/Task scope denied；Contractor 使用单 Task 入口 | Task runtime 或 Assignment 查询不可用，不显示可完成按钮 | Target 或 Task 版本变化时刷新并重新选择 Assignee | Assignment/Task receipt 更新当前行；Task 完成显示等待 Cases 重验 |
| `/tasks/:taskId` application | Task detail/complete form 骨架 | 任务不存在或无当前 Assignment 按安全 not_found/denied | 时间、渠道、提交人、清单和凭证字段错误 | 非当前 Assignee/不合格 role denied | Tasks/Documents 依赖不可用，保留草稿 | Task record/assignment stale 要求重新加载 | completed 回 Applications，不宣称 Target 已 submitted |
| `/contractor/tasks/:taskId` | 脱敏 Task 骨架 | 非当前有效 interview_support Assignment 不显示任务 | 只校验 Interview Support 完成摘要 | 通用 404/denied，不确认 Case/Task 存在 | Contractor runtime 不可用时重试，不降级到完整 Case | Assignment 被撤销/重派后立即失效 | completed/拒绝回 `/tasks`；不进入 Case 页面 |

所有页面另有统一 `not_found` 安全语义：只有服务端允许区分目标不存在时才显示；否则使用通用“无法访问此页面”。F3 不显示原始错误、SQL、provider、token、文件 key 或 PII。

## 7. API、DTO 与权限追溯

### 7.1 已批准的 public port 和路径

| 页面/操作 | Approved endpoint / command | 最小页面 DTO | 服务端 owner/权限 | 事实追溯 |
|---|---|---|---|---|
| Case workspace 上下文 | `GET /api/v1/cases/:caseId/workspace` | Case stage、workflow、Primary Advisor、record_version、名单/Target/Task 摘要和 allowed actions | Cases coordinator 组合公开查询；owner 各自重验 | F1、approved Cases API/UI、BR-030 |
| 候选名单读取 | `GET /api/v1/cases/:caseId/candidate-lists?limit=&cursor=` | `version`、immutable version 摘要、状态、`record_version`、created_at、Founder approval、Guardian decision、opaque next cursor；默认25/最大100，版本号 DESC、id ASC，无 total | Primary Advisor/Founder 按 Case 关系；Admin/Contractor denied；cursor 绑定 case/filter/sort | BR-033；Cases public query；Architect D1 |
| 学校选项 | `GET /api/v1/schools/options?q=&limit=&cursor=` | approved/active `SchoolReferencePin` allowlist；不返回 crawler 原文；opaque cursor | 只返回当前可使用的学校选项；Schools owner 最终授权 | BR-033；Architect D1 |
| 新建候选名单 | `POST /api/v1/cases/:caseId/candidate-lists` | body `school_reference_pins` 完整集合、`change_reason`、`expected_case_record_version`；receipt `{id,version,status,record_version,created_at}` immutable | 当前 Primary Advisor；background/school blocker 重验 | BR-033；`createCandidateListVersion`；Architect D1 |
| Founder review | `POST /api/v1/cases/:caseId/candidate-lists/:versionId/review` | body `{decision: approve|reject, reason?, expected_record_version}`；receipt `{version_id,approval_id,status,record_version,occurred_at}` | Founder only；reject reason 必填；actor/time 从 session/server 记录 | BR-033、BR-070；`reviewCandidateList`；Architect D2 |
| Guardian decision | `POST /api/v1/cases/:caseId/candidate-lists/:versionId/guardian-decision` | body `{guardian_id,decision: confirm|not_confirmed,occurred_at,channel: phone|wechat|meeting,expected_record_version}`；receipt `{version_id,guardian_decision_id,status,record_version,occurred_at}` | Current Primary Advisor；recorder 从 session 解析；同一 Founder-approved version；Portal denied | BR-033、BR-070；`recordGuardianListDecision`；Architect D2 |
| Target 读取 | `GET /api/v1/cases/:caseId/school-targets` | 完整 BR-034 state、current Application Assignee、`application_prepare_submit` Task、record_version、allowed actions | Case/Target resource relation；Contractor denied | BR-034；`listSchoolTargets`；Architect D4 |
| Task 读取 | `GET /api/v1/tasks/:taskId`、`GET /api/v1/tasks/assigned` | task type/state/due_at/overdue/assignment/allowed_actions/completion summary | Current Assignee/Primary Advisor/Case Collaborator | BR-035–037；approved Task API/UI |
| Application Task 完成 | `POST /api/v1/tasks/:taskId/complete` | `application_prepare_submit` completion fields、opaque evidence ref、expected version | Current accepted Application Assignee；Tasks owner；Cases 后续重验 | BR-035、BR-070；approved Task API/UI |
| Interview Task 读取 | `GET /api/v1/contractor/tasks/:taskId/workspace`（Contractor）或内部 Task GET | 目标学校、时间/方式/语言、必要 brief、Assignment、allowed actions | 当前有效 `interview_support` Assignment；Contractor 脱敏 | BR-036；approved authorization/Task API/UI |
| Target event | `POST /api/v1/cases/:caseId/school-targets/:targetId/events` | body `{event_type: interview_required|result_recorded|withdrawn,evidence,expected_record_version}`；server 派生下一状态 | Cases owner；客户端不得传 `next_state` | BR-034；Architect D4 |
| Guardian offer decision | `POST /api/v1/cases/:caseId/school-targets/:targetId/offer-decisions` | body `{guardian_id,decision: offer_confirmed|offer_declined,occurred_at,channel: phone|wechat|meeting,expected_record_version}` | 仅 `accepted` -> `offer_*`；Primary Advisor 代录；保存实际 Guardian | BR-034、BR-070；Architect D4 |
| Application Assignee | `POST /api/v1/cases/:caseId/school-targets/:targetId/assignments` | body `{advisor_role_binding_id,expected_record_version}`；Assignment 历史追加 | 仅 Primary Advisor 或已授权 Advisor Collaborator；Contractor 永不担任 | BR-035；Architect D4 |
| Target / confirmed diff | `POST /api/v1/cases/:caseId/candidate-lists/:versionId/apply-confirmed` | body `{expected_record_version}`；receipt 分组 unchanged/new/withdrawn target IDs 与 task requests/cancellations | 同版 Founder approval + Guardian confirm；可靠幂等副作用 | BR-033/034/035/037；Architect D3 |
| Interview support | `POST /api/v1/cases/:caseId/school-targets/:targetId/interview-support` | body `{interview_at,mode,language,expected_record_version}`；Task/Interview fact receipt | 只允许 submitted 且学校事实要求面试；默认 Primary Advisor；一个幂等 Task | BR-034、BR-036；Architect D5 |
| Task Assignment | `POST /api/v1/tasks/:taskId/reassign` | approved Task reassign body/expected version；Assignment 历史保留 | 合资格 Advisor Collaborator 或 Contractor；Contractor 不能管理 Case | BR-036、BR-037；approved Task API/UI；Architect D5 |

### 7.2 跨模块副作用和数据边界

- Cases `applyConfirmedListChanges` 通过可靠事件/公开 port 请求 Tasks 创建或取消 Task；Cases 不写 Tasks 表。
- Tasks 完成后发布 `tasks.application_submission_completed` 或 Interview Support 完成事实；Cases 重新查询 Target、Assignee、版本和凭证，再推进状态。
- Documents 只返回当前用户可访问的 opaque reference、clean/available/revoked 摘要；F3 不设计上传、下载或扫描 UI。
- Notifications 消费名单审批、审批结果、Task 分派/重派/拒绝/到期/逾期和结案选择 effect；F3 不读取或写通知页面，也不把通知当作权限。
- AuditEvent 与业务命令同事务；日志、通知、outbox 不复制姓名、联系方式、Assessment answer、文件内容或自由文字 PII。

### 7.3 Architect 已冻结的 D1–D5 transport contract

以下五项已冻结为 F3 可实现契约；它们不新增业务实体、角色或状态。所有 POST 使用 `Idempotency-Key`、统一 envelope、`no-store`、`X-Request-Id`；时间输入使用带时区 RFC3339，服务端保存 UTC，UI 显示 `Asia/Hong_Kong`。

#### D1. Candidate List

~~~text
GET  /api/v1/cases/:caseId/candidate-lists?limit=&cursor=
POST /api/v1/cases/:caseId/candidate-lists
GET  /api/v1/schools/options?q=&limit=&cursor=
~~~

- 列表默认 `limit=25`、最大 100；版本号 DESC、id ASC；opaque cursor 绑定 case/filter/sort；不返回 total。
- POST body：`school_reference_pins`（完整集合，pin 由 Schools owner 提供）、`change_reason`、`expected_case_record_version`。
- 只有当前 Primary Advisor 且 background/school blocker 满足时可创建；服务端不接受客户端 actor/role/organization。
- receipt 为 immutable version `{id,version,status,record_version,created_at}`。
- 学校选项只来自 approved/active `SchoolReferencePin` allowlist；不提供 crawler 原文。

#### D2. Founder Review 与 Guardian Decision

~~~text
POST /api/v1/cases/:caseId/candidate-lists/:versionId/review
POST /api/v1/cases/:caseId/candidate-lists/:versionId/guardian-decision
~~~

- review body：`{decision: approve|reject, reason?, expected_record_version}`；reject 时 `reason` 必填；server 从 session/clock 记录 actor/time；receipt `{version_id,approval_id,status,record_version,occurred_at}`。
- Guardian body：`{guardian_id,decision: confirm|not_confirmed,occurred_at,channel: phone|wechat|meeting,expected_record_version}`；recorder 从 session 解析，客户端不得传；必须绑定同一 Founder-approved version；receipt `{version_id,guardian_decision_id,status,record_version,occurred_at}`。
- `confirm` 才能进入 confirmed diff；`not_confirmed` 只提供新名单版本路径，不创建申请 Task。

#### D3. Apply Confirmed Diff

~~~text
POST /api/v1/cases/:caseId/candidate-lists/:versionId/apply-confirmed
~~~

- body：`{expected_record_version}`；只允许同版 Founder approval + Guardian confirm。
- server 按 School pin + application round 计算 `unchanged/new/removed/terminal`。
- receipt 分组返回 `unchanged_target_ids`、`new_target_ids`、`withdrawn_target_ids`、`task_requests[{task_id,status}]`、`task_cancellations[{task_id,status}]`。
- 副作用可靠幂等，不能重复 Target/Task；`partial`/`unavailable` 明确为“处理中/可重试”，不得伪装完成。

#### D4. Target、Offer 与 Application Assignee

~~~text
POST /api/v1/cases/:caseId/school-targets/:targetId/events
POST /api/v1/cases/:caseId/school-targets/:targetId/offer-decisions
POST /api/v1/cases/:caseId/school-targets/:targetId/assignments
~~~

- events body：`{event_type: interview_required|result_recorded|withdrawn,evidence,expected_record_version}`；server 重验并派生下一状态，客户端不得传 `next_state`。
- offer body：`{guardian_id,decision: offer_confirmed|offer_declined,occurred_at,channel: phone|wechat|meeting,expected_record_version}`；只允许 `accepted -> offer_*`；Primary Advisor 代录并保存实际 Guardian。
- assignment body：`{advisor_role_binding_id,expected_record_version}`；只允许 Primary Advisor 或已授权 Advisor Collaborator；Contractor 永不担任 Application Assignee；Assignment 历史追加。
- Applications 读取必须返回完整 BR-034 state、current assignment、`application_prepare_submit` Task 和 `allowed_actions`。Task 完成发布事实，Cases 再推进 `submitted`。

#### D5. Interview Support

~~~text
POST /api/v1/cases/:caseId/school-targets/:targetId/interview-support
POST /api/v1/tasks/:taskId/reassign
GET  /api/v1/contractor/tasks/:taskId/workspace
~~~

- interview-support body：`{interview_at,mode,language,expected_record_version}`；只允许 `submitted` 且学校事实要求面试；Cases 记录 interview fact，并幂等请求一个 `interview_support` Task，默认 Primary Advisor。
- Task assignment 使用 approved reassign command；可重派至合资格 Advisor Collaborator 或 Contractor；Contractor 只能读取单 Task 脱敏 DTO。
- Contractor DTO 只含 target school/time/mode/language/brief/due_at/assignment/allowed_actions，不能进入 Case 或读取完整 Assessment、Guardian、文件、其他学校或其他 Task。
- Task 完成只走 approved complete command；Cases 重验后不自动改 SchoolTarget 结果。

## 8. 当前实现证据与目标差异

以下证据只描述当前产品仓库，不覆盖 confirmed BR 或 approved module contract。

| 当前实现证据 | 当前行为 | F3 目标处理 |
|---|---|---|
| [SchoolTargetsPanel.tsx](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/cases/SchoolTargetsPanel.tsx:17) | 只读加载 SchoolTarget，显示 items 和 `school_options`；按钮状态只显示 ready/forbidden/unavailable，没有名单版本、Founder 审批、Guardian 确认或申请 Task | `/schools` redesign 为 W01–W06；版本、两层确认和 allowed actions 由 Cases DTO 驱动 |
| [school-targets route](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/cases/[caseId]/school-targets/route.ts:24) | GET 返回旧 `case_stage/intake_year/admission_type/can_create/items/school_options`；POST 固定返回 `CONFLICT` | 目标读取仍由 approved `listSchoolTargets` 负责；创建/差异/Task 副作用按 D3 冻结，不沿用旧 `can_create` 或假写入 |
| [target transitions route](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/cases/[caseId]/school-targets/[targetId]/transitions/route.ts:9) | 只做 bootstrap capability 检查，随后固定 `CONFLICT`；不能表达 approved 状态机 | Applications/Interviews 只调用 owning Cases command；状态只能由服务端重验推进，按 D4 冻结 transport |
| [target outcomes route](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/cases/[caseId]/school-targets/[targetId]/outcomes/route.ts:9) | 只做角色检查后固定 `CONFLICT`，没有 Guardian offer 决定记录 | `accepted` 后由 Primary Advisor 代录 offer decision，记录同版 Guardian/渠道/时间/代录人；不新增 Offer 实体 |
| [selector/page.tsx](</Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/(erp)/selector/page.tsx:20>) | 全局 crawler 学校搜索、浏览器本地筛选、报告/PDF 和报错 Ticket；使用旧 `s1_admission` 等值 | `isolate_from_release1` 作为全局入口；F3 只在 Case 名单版本内消费 approved SchoolReferencePin，不设计 Schools 治理 |
| [workspace-model.ts](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/cases/workspace-model.ts:3) | workspace tabs 包含 overview/assessment/schools/tasks/documents/timeline，并用 `?tab=` 生成链接 | F1 canonical 子路由保持独立 `/schools`、`/applications`、`/interviews`；F3 不引入 query tab |
| [CaseWorkspace.tsx](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/cases/CaseWorkspace.tsx:217) | 当前 workspace 对 schools/tasks 仅渲染通用 rows/action，冲突 dialog 为通用摘要；没有 F3 版本差异和 Task/Target ownership 分区 | 使用独立子路由和 Cases/Tasks owner DTO；冲突按版本、名单、Target、Task 分别刷新 |
| [tasks route](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/tasks/route.ts:6) | 当前 Task list/create 是通用 runtime route，未表达 application_prepare_submit 与 interview_support 的 F3 页面契约 | 复用 approved Task API/UI；按 task_type、Assignment 和 allowed_actions 分流，不在 F3 新增 Task 状态 |
| [TaskDetailView.tsx](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/tasks/TaskDetailView.tsx:67) | 通用 Task 详情和 transition controls，当前没有按申请提交/面试辅助呈现完成字段的低保真结构 | `/applications` 连接申请提交表单，`/interviews` 连接最小辅助 Task；完成后回源路由，Cases 再重验 |
| [contractor task route](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/contractor/tasks/[taskId]/route.ts) | 当前存在 Contractor API 入口，但 F3 目标仍需严格限定 `interview_support` 脱敏投影 | 保留入口边界；Contractor 不进入三条 Case 子路由，不读取完整 Case/Assessment/Guardian/文件 |

### 8.1 现有正式入口分类

| 入口 | F3 处理 |
|---|---|
| `/cases/:caseId/schools` | `redesign`；不保留当前只读 panel 的业务边界 |
| `/cases/:caseId/applications` | `missing` 或从 Case workspace 新增独立子路由；不以通用 tasks tab 代替 |
| `/cases/:caseId/interviews` | `missing` 或从 Case workspace 新增独立子路由；Contractor 不共用 |
| `/selector` | `isolate_from_release1`；全局 crawler/filter/PDF/Ticket 不是 F3 选校入口 |
| `/schools`、`/admin/schools` | `isolate_from_release1` for F3；Schools 资料治理属于 F4，不从 F3 进入 |
| `/tasks/:taskId` | `keep/redesign` approved Task 工作台入口；Application/Interview 完成字段按 task_type 展示 |
| `/contractor/tasks/:taskId` | `keep/redesign` approved Contractor 单 Task 入口；只保留 interview_support 脱敏边界 |

## 9. 桌面、移动和可访问性原则

| 主题 | 桌面 | 移动 |
|---|---|---|
| Case 导航 | 共享 Case layout + 独立子路由；标题区保留阶段、负责人、版本和返回 | 顶部返回、单列 section；不使用 query tab |
| 版本比较 | 学校集合差异表、固定列宽、旧版本只读 | 逐校差异条目；每条显示不变/新增/移除及后果 |
| 操作区 | Founder/Advisor 允许动作集中在当前版本/Target 标题区底部 | 底部固定主操作；错误和键盘不被遮挡 |
| Task | Target 行显示 Task/Assignment 摘要，点击进入 owner Task 页面 | 每个 Target 纵向显示状态、Assignee、Task、due_at 和动作 |
| Contractor | 不进入 Case 页面 | 单 Task 脱敏页面；只显示最小 Interview Support 内容 |
| 反馈 | 稳定 loading、section error、冲突 dialog；错误有文字和焦点 | 全屏 sheet/dialog；不只用颜色表达状态 |

界面是安静、实用、工作导向的 ERP；不使用营销 hero、装饰性卡片堆叠或 Schools 资料展示墙。学校资料、申请凭证和面试摘要均按 allowlist 扫描，禁止把对象 key、原始 crawler 内容或 PII 放入页面。

## 10. 验收证据与交付摘要

| 门禁 | 结果 | 证据 |
|---|---|---|
| 三条 F3 路由范围完整 | 满足 | `/schools`、`/applications`、`/interviews` 及 shared Case layout 在 §1–4 覆盖 |
| 桌面/移动线框 | 满足 | W01–W10，共 10 个 Markdown/ASCII 线框 |
| 名单两层确认 | 满足 | W01–W06；版本、完整集合、Founder approval、Guardian 代录字段和同版绑定 |
| 名单变化连续性 | 满足 | §0.2、§5：不变保留、新增 preparing + application Task、移除 withdrawn + cancel Task、全拒绝两分支 |
| SchoolTarget 状态 | 满足 | §3.1 固定 BR-034 全状态和终态；accepted 不作终态 |
| 申请 Task | 满足 | §3.4、§5：一个幂等准备/提交 Task；提交证据字段；Cases 重验后 submitted |
| 面试 Task | 满足 | §4：需要面试才创建；Advisor/Contractor 辅助；完成不改变学校结果 |
| Guardian offer 决定 | 满足 | §3.1、§5：Primary Advisor 代录记录 Guardian/结果/时间/渠道/代录人；Portal 不写 |
| 角色拒绝边界 | 满足 | §1.3、§4：Founder/Admin/Advisor/Contractor、Founder+Admin、Contractor task_only |
| 状态完整 | 满足 | §5–6：loading、empty、validation、denied、unavailable、stale/conflict、error、success、返回 |
| API/DTO/权限追溯 | 满足 | §7：approved Cases/Tasks public port 与 Architect 已冻结 D1–D5 transport |
| 现状与目标分离 | 满足 | §8：当前源码证据与目标处理分栏；不覆盖 confirmed BR |
| 业务边界 | 满足 | 未新增实体、角色、状态；不设计 Documents、Schools governance、Notifications、Portal |
| 非代码范围 | 满足 | 只新增本设计 Markdown；未修改产品源码、测试、数据库、migration 或云 |

### 10.1 交付字段

| 字段 | 值 |
|---|---|
| status | `approved` |
| owner | `frontend-design` |
| changed file | `system-design/50-api-ui/frontend-design/40-f3-school-selection-application-interview-wireframes.zh-CN.md` |
| wireframe_count | 10（W01–W10） |
| decision_count | 0 |
| browser | `not_run` |
| database | `not_run` |
| cloud | `not_run` |
| product-tests | `not_run` |
| F4/F5 | `approved` |

Architect 已审核通过 F3 并冻结 D1–D5；本文件只授权按 F3 设计进入前端实现，不授权修改业务事实源或跳过后端 owning module。F4/F5 也已完成审核并标记 `approved`。
