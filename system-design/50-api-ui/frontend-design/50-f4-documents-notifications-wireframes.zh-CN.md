# F4 Documents 与 Notifications 低保真线框及交互契约

> 状态：`approved`
> Owner：`frontend-design`
> 范围：Release 1 内部 ERP；Documents 文件工作台与 Notifications 入口
> 前置：F0/F1/F2/F3 approved；BR-050、BR-038、BR-070、BR-071 confirmed；Documents/Notifications API/UI approved
> 本轮不做：产品源码、React/TSX、API 实现、数据库、migration、测试、浏览器、对象存储 provider 选择、外部 Email/SMS/WhatsApp、Guardian/Student/Portal 文件权限、F5 Portal

## 0. 一页总览

### 0.1 本轮目标

F4 定义两个内部 ERP 入口的低保真结构和可执行交互边界：

- `/documents`：按当前用户授权的 Case、SchoolTarget、Application Assignee 资源过滤文件 metadata；完成注册、版本上传、upload capability、object receipt、扫描结果、预览/下载 capability、soft-delete、restore 和 legal hold 状态表达。
- `/cases/:caseId/documents`：共享 Case layout 下的案件文件入口；从 Applications/Task 进入时保留 Case/SchoolTarget/Assignee 上下文，不扩大资源权限。
- `/notifications`：当前 Session User 的站内通知列表、unread count、read 和 resolve-target；只显示最小文案，必需通知没有关闭/静音操作。
- Workspace shell：Founder、Advisor 和有相应 capability 的组合身份显示通知 unread count；Contractor 不进入全局通知列表，Task 页面仍由 Tasks owner 显示其分派事实。

Documents 和 Notifications 都是 owning module 的投影/命令入口：页面不能因文件存在自动推进 SchoolTarget/Task，也不能因通知存在自动授予 Case、Task 或文件权限。

### 0.2 F3、Task 与 F4 连续性

```text
/cases/:caseId/applications
  |-- application_prepare_submit Task
  |     `-- 打开 /cases/:caseId/documents（SchoolTarget + Assignee scope）
  |           -> 选择 clean/available/未撤销 Version 作为 opaque evidence ref
  |           -> 返回 /tasks/:taskId 或 Applications
  |
  |-- /cases/:caseId/interviews
  |     `-- interview_support Task 不进入 Documents；Contractor 永远无文件入口
  |
  `-- /cases/:caseId/documents
        -> 注册 Document/Version
        -> upload capability -> object receipt
        -> quarantined -> scanning -> available/rejected/scan_failed
        -> 只有 available 才能预览/下载/激活/作为 Task 证据

已提交 Cases/Tasks effect
  -> Notifications Worker 重新解析 recipient
  -> /notifications 显示最小“有待办事项需要处理”
  -> 点击 resolve-target -> 服务端重新授权 -> 固定内部 route
```

### 0.3 Release 1 明确排除

- 不向 Guardian、Student 或 Portal 暴露内部文件列表、预览、下载、上传、删除或恢复。
- 不给 Contractor 文件权限；面试辅助人只使用单一脱敏 Task DTO。
- 不显示对象存储 key、bucket、公开 URL、capability secret、扫描原始输出、provider 错误或自由文字 PII。
- 不在页面新增 Document、Version、ScanResult、LegalHold、Notification 或 DeliveryReceipt 实体；不新增文件或通知状态。
- 不提供通知删除、撤回、静音、关闭必需通知或外部发送渠道。
- 不设计 Documents 资料治理、学校资料治理、Portal 页面、Operations/Audit 详细页面；F4 仅显示其 approved 入口边界和受控状态。

## 1. 路由、入口和返回路径

### 1.1 目标路由树

```text
/today
  `-- 通知图标 + unread count -> /notifications

/documents
  |-- Case 过滤器 -> /cases/:caseId/documents
  |-- SchoolTarget 过滤器（仅已授权 scope） -> /cases/:caseId/documents
  `-- Assignee 过滤器（仅返回 owner 允许的值） -> /documents

/cases/:caseId（共享 Case layout）
  |-- /documents             Case 文件列表与上传/版本操作
  |-- /applications          SchoolTarget 行的文件证据入口
  `-- /interviews            不显示文件入口给 Contractor

/tasks/:taskId
  `-- application_prepare_submit -> 当前 SchoolTarget 的 /documents scope

/notifications
  `-- read / resolve-target -> owning module canonical route
```

### 1.2 页面与入口矩阵

| 页面 | 进入 | 主要内容 | 返回/失败路径 |
| --- | --- | --- | --- |
| `/documents` | Sidebar 的文件入口、Today/Case 的“查看文件”链接 | 授权文件 metadata、Case/SchoolTarget/Assignee scope 筛选、版本/扫描/生命周期状态 | 保留筛选上下文；无权统一 denied，不确认目标是否存在 |
| `/cases/:caseId/documents` | Case layout、Applications 文件证据链接、Task 来源上下文 | 当前 Case 文件；若有 SchoolTarget/Assignee scope，只显示该 scope | 返回来源 Case 子路由或 `/tasks/:taskId`；Case 重新授权 |
| `/notifications` | Today shell 通知入口、TopBar 通知入口 | 当前 Session User 的 unread/read 通知、最小文案、允许动作 | resolve 成功进入固定内部 route；目标失效显示 unavailable 并留在列表 |
| shell unread count | `/today`、内部任务壳层的授权通知入口 | 当前 User 的 unread count；不作为权限或数据数量依据 | 查询失败显示“暂不可用”状态，不用旧 count 冒充当前值 |
| `/tasks/:taskId` 文件证据区 | Applications 申请 Task 行 | 仅选择已授权 SchoolTarget 文件 Version 作为 opaque evidence ref | 文件不可用/失权回 Task，不能由 Documents 改 Task 状态 |

页面 URL 与 API path 分离：Case 页面使用 `/cases/:caseId/documents`，查询仍调用 approved Documents API；不得为了过滤新增 query tab、客户端私有表或对象直链。

### 1.3 当前入口保护

Contractor 登录后仍从 `/tasks` 开始；`/documents`、`/cases/:caseId/documents`、`/notifications` 由服务端按 capability/resource relation 重新授权。Admin 单独角色不能通过 URL 或通知进入客户文件；Founder+Admin 组合按 Founder/Advisor 等实际 capability 合并，不能简化为 Admin 放行。

## 2. 角色矩阵与拒绝边界

| 身份/关系 | Documents 可见范围 | Documents 操作 | Notifications 可见范围 | 明确拒绝 |
| --- | --- | --- | --- | --- |
| Founder | 组织内当前授权 Case 的 metadata、版本、扫描和治理状态 | 按 `allowed_actions` 下载/预览；可执行 Founder legal hold 治理；不因 Founder 身份自动取得每个 Case 上传/删除权 | 当前 User 可见的组织审批、逾期、结案选择等最小通知 | 不能绕过 Case/SchoolTarget/Document owner；不能看到原始扫描输出 |
| Admin 单独 | 无客户文件 | 无上传、下载、删除、恢复、legal hold | 不因 Admin 身份自动收到客户业务通知；仅可见自身合法系统通知（若 owning contract 返回） | Case、Student/Guardian、SchoolTarget、客户文件和业务通知均不因 Admin 放行 |
| Advisor / Primary Advisor | 自己负责或明确授权 Case 的文件；Primary Advisor 可到 Case scope | 注册/新版本/上传；按 Case 关系 soft-delete/restore；当前 Application Assignee 只操作负责学校用途 | 自己 Case、Task Assignment、名单审核结果、到期/逾期和结案选择等最小通知 | 不能看无关系 Case；不能用通知扩大权限 |
| Advisor Collaborator / Application Assignee | 仅获授权 Case；Application Assignee 仅当前 SchoolTarget 必要材料和 `submission_evidence` | 可上传/引用自己学校材料；不能扩展到其他学校、不能直接改 SchoolTarget | 当前 Assignment/Case 关系允许的最小通知 | 不能成为 Contractor 越权入口；失去 Assignment 立即失权 |
| Contractor | 无文件列表、无预览/下载/上传/删除/恢复 | 无 | 不进入 `/notifications` 全局列表；Task owner 处理分派事实 | 不得进入 Case、Documents 或读取 Guardian/Assessment/其他 Task |
| Guardian Viewer | 无内部文件 | 无 | 无内部通知 | Portal 与内部 Notifications/Documents 完全隔离 |

Founder+Admin、Founder+Advisor 等组合取有效 capability 并集，但每个请求仍由 Identity、Access、Case/SchoolTarget Assignment 和 Documents/Notifications owner 重新判断。`allowed_actions` 是按钮显示依据，永远不是授权来源。

## 3. Documents 页面信息架构

### 3.1 资源级过滤

| 过滤维度 | UI 控件 | 服务器边界 | 空/失权表现 |
| --- | --- | --- | --- |
| Case | `/documents` 顶部 Case 选择；Case 内页锁定当前 Case | 只返回当前 User 可访问 Case 的 metadata；选择变化重新读取，不在浏览器本地拼接全局数据 | 没有授权 Case 显示空态；失权统一 denied，不确认 Case 是否存在 |
| SchoolTarget | Case scope 内的学校目标选择；Applications 进入时锁定 | 只返回当前 Application Assignee 或获授权 Advisor 的该校必要文件 | 不适用或无目标显示“没有可用学校范围”；Contractor 永久 denied |
| Application Assignee | Founder/Primary Advisor 可见的受控 assignee 选择；不可任意输入 User ID | owner 依据当前 Assignment/Case 关系过滤；不返回组织成员全集 | 无可选值显示空态；Admin/Contractor 不显示控件 |
| Version/扫描 | 状态筛选：pending_upload、quarantined、scanning、available、rejected、scan_failed、abandoned | 以服务端 `active_version`、scan 和 lifecycle 返回为准 | 过滤变化不改变 Document 事实；无结果是 empty，不是 denied |
| 生命周期 | active、pending_delete、deleted；legal hold 作为受控旗标 | 仅返回当前用户允许看到的 lifecycle metadata；legal hold 不等于新状态 | `deleted` 仅显示受控 tombstone；不提供对象恢复以外的直接清理 |

列表不承诺 total count；任何筛选、排序或分页语义以 approved Documents owner DTO 为准。F4 不发明未冻结的 query key，不把 `case_id`、`target_id` 或 `assignee_id` 当作越权凭证。

### 3.2 Documents 行字段

每行只显示可安全返回的字段：受控 opaque Document ID/用途 code、Case/SchoolTarget scope 摘要、active Version ID/状态、scan safe result code、lifecycle、legal hold 旗标、record version、updated_at 和 `allowed_actions`。目标设计不显示对象 key、永久 URL、Token、扫描原文或未经批准的原始文件名。

### 3.3 W01：Documents 桌面目录

~~~text
+--------------------------------------------------------------------------------+
| [≡] 文件工作台                         [通知 unread] [账号]                    |
+--------------------------------------------------------------------------------+
| 文件                                                                     |
| 授权范围内的文件 metadata、版本和安全状态                                     |
|                                                                              |
| Case [全部授权案件 v]  SchoolTarget [全部 v]  Assignee [与我有关 v]           |
| 版本 [全部 v]  生命周期 [全部 v]                              [重新载入]       |
|                                                                              |
| Document opaque-••21   用途 submission_evidence   Case scope                 |
| Version v3 · available · clean · active        版本 8   [预览] [下载]         |
|                                                                              |
| Document opaque-••77   用途 identity_and_case_evidence                       |
| Version v1 · scanning · 不可用                  版本 2   [查看状态]            |
|                                                                              |
| Document opaque-••95   用途 submission_evidence   SchoolTarget scope          |
| Version v2 · pending_delete · legal hold        版本 4   [恢复] [查看]         |
|                                                                              |
| 空态：当前授权范围没有文件。Denied：无法查看文件。Unavailable：[重试]       |
+--------------------------------------------------------------------------------+
~~~

桌面主操作固定在过滤栏和每行右侧；下载/预览只在 `allowed_actions` 且 Version clean/available/未 revoked 时出现。通知文案不把文件行内容带入通知。

### 3.4 W02：Documents 移动目录

~~~text
+----------------------------------+
| [☰] 文件                 [通知]  |
| 文件范围 [选择 Case v]           |
| [SchoolTarget v] [Assignee v]    |
| [状态 v]              [刷新]     |
|                                  |
| opaque-••21                       |
| submission_evidence              |
| Case scope · v3                  |
| available · clean · active       |
| [预览] [下载]                    |
|                                  |
| opaque-••77                       |
| identity_and_case_evidence       |
| v1 · scanning · 不可用           |
| [查看状态]                       |
|                                  |
| [暂无文件 / 无权 / 暂不可用]     |
+----------------------------------+
~~~

移动端过滤器采用顺序 sheet；每行按钮垂直排列，状态文本与按钮不重叠。返回键回到原 Case/Task 来源，不将 scope 写入长期权限。

## 4. Case/SchoolTarget 文件入口与上传

### 4.1 Case 页面结构

`/cases/:caseId/documents` 使用 F1 共享 Case layout，保留 Case 标题、阶段、返回路径和独立子路由导航。进入 Applications 的 SchoolTarget 文件入口后，页面显示受控 `Case + SchoolTarget + current Application Assignee` scope 摘要；若当前用户只有 Case 级权限，则不显示该校专属文件。

### 4.2 W03：Case Documents 桌面

~~~text
+--------------------------------------------------------------------------------+
| < 返回 Applications   Case opaque-••42   [摘要][申请][面试][文件]              |
| 文件 / 当前 Case scope                                                        |
| SchoolTarget scope：SchoolReferencePin-••8   Assignee：当前授权 Advisor       |
|                                                                              |
| [注册文件]   [过滤：用途 v] [版本/扫描 v]                    record_version 12 |
|                                                                              |
| submission_evidence · opaque-••31 · v2 · available · clean · active           |
| 用途：申请提交凭证   允许：预览/下载/引用 Task                             |
| [预览] [下载] [引用到申请 Task]                                               |
|                                                                              |
| identity_and_case_evidence · opaque-••18 · v1 · rejected                      |
| 文件不可用；不显示扫描原文。                                                  |
| [新建版本]                                                                    |
|                                                                              |
| [返回 Applications]                         [刷新授权/版本]                  |
+--------------------------------------------------------------------------------+
~~~

“注册文件”只打开用途/学校范围表单；文件名、对象存储地址和 provider 细节不由页面强行扩展。`引用到申请 Task` 仅回传 opaque Version ID，Tasks/Cases 仍重验。

### 4.3 W04：Case Documents 移动

~~~text
+----------------------------------+
| < 申请   Case ••42    [文件]     |
| Case scope                         |
| SchoolTarget ••8 / Assignee      |
| [注册文件] [过滤 v]               |
|                                  |
| submission_evidence               |
| opaque-••31 · v2                 |
| available · clean · active       |
| [预览] [下载] [引用 Task]         |
|                                  |
| rejected · opaque-••18           |
| 不可预览/下载        [新建版本]  |
|                                  |
| [返回申请] [刷新授权]            |
+----------------------------------+
~~~

### 4.4 W05：上传注册与传输桌面

~~~text
+--------------------------------------------------------------------------------+
| 注册 Document / 新建不可变 Version                                             |
| Scope：Case opaque-••42  SchoolTarget ••8  用途 [submission_evidence v]       |
|                                                                              |
| 文件选择 [选择文件]     checksum/size/content_type（服务端复核）              |
|                                                                              |
| [取消]                                             [注册并取得上传 capability] |
|                                                                              |
| 1  pending_upload   已注册，等待上传                                          |
| 2  quarantined      已收到对象，等待安全检查                                  |
| 3  scanning         安全检查中                                                |
| 4  available        clean，可预览/下载/激活                                   |
|                                                                              |
| rejected / scan_failed：文件不可用；[建立新 Version] [稍后重试]              |
| unavailable：结果暂时无法确认；保持当前状态，不显示成功。                   |
+--------------------------------------------------------------------------------+
~~~

前置条件：当前用户有 Document register/upload capability、Case/SchoolTarget scope 有效、用途受控、文件输入通过客户端基础校验。服务端仍复核 actor、scope、checksum、size、content type、expected version，并通过 upload capability 绑定精确 Version。

### 4.5 W06：上传传输移动

~~~text
+----------------------------------+
| < 文件   上传 Version            |
| 用途 submission_evidence         |
| SchoolTarget ••8                 |
| [选择文件]                       |
|                                  |
| [注册] -> pending_upload         |
| [上传] -> quarantined             |
| 安全检查中 -> scanning           |
|                                  |
| available / clean                |
| 可预览、下载或引用 Task          |
| rejected / scan_failed            |
| 不可用 [建立新 Version]          |
|                                  |
| [取消] [重新载入状态]             |
+----------------------------------+
~~~

上传中的按钮固定禁用重复提交；同一 Idempotency-Key 重试只重放原 receipt。网络中断时显示“结果暂时无法确认”，不将浏览器 PUT 成功当作服务端 available。

## 5. 文件版本治理、预览下载与生命周期

### 5.1 状态与允许动作

| Version / Document 状态 | 页面显示 | 允许动作 | 禁止动作 |
| --- | --- | --- | --- |
| `pending_upload` | 等待上传 | 继续上传、按授权 abandon | 预览、下载、激活、Task 证据 |
| `quarantined` | 已接收，等待安全检查 | 只读刷新状态 | 预览、下载、激活 |
| `scanning` | 安全检查中 | 只读刷新状态 | 预览、下载、激活 |
| `available` + clean + 未 revoked | 可用 | 按授权预览、下载、激活、引用 Task | 绕过授权或改写旧 Version |
| `rejected` | 文件不可用 | 查看安全失败 code、建立新 Version | 预览、下载、激活、Task 证据 |
| `scan_failed` | 扫描暂时失败/可重试 | 刷新、按 owning policy 建新 Version | 假装 clean、预览、下载、激活 |
| `abandoned` | 上传已放弃 | 查看受控历史 | 继续使用该 Version |
| Document `active` | 当前生命周期有效 | 按 allowed actions 操作 Version | 无 expected version 的覆盖 |
| Document `pending_delete` | 进入 30 天恢复窗口 | 受权 restore；显示 hold/引用阻断 | 新建无关对象、直接清理 |
| Document `deleted` | 对象内容已清理的 tombstone | 查看受控 metadata/历史 | 预览、下载、恢复为 active 内容 |
| `legal_hold=true` | Legal hold 旗标 | Founder 按治理 capability 查看/解除 | soft-delete 或最终清理；普通用户不得更改 |

`legal_hold` 是现有治理旗标，不是新增生命周期状态。页面只在 DTO 的 `allowed_actions` 明确包含对应动作时显示按钮；approved Documents API/UI 未冻结的命令路径不得由前端自行拼接，缺少路径时只显示只读旗标和“由治理流程处理”。

### 5.2 W07：版本治理、预览与下载桌面

~~~text
+--------------------------------------------------------------------------------+
| Document opaque-••31 · submission_evidence                                    |
| Scope：Case ••42 / SchoolTarget ••8 / Assignee Advisor                       |
| Lifecycle：active    Legal hold：是    record_version：12                    |
|                                                                              |
| Version history                                                               |
| v3  available · clean · active · 2026-08-26       [预览] [下载] [引用 Task]   |
| v2  available · clean · 保留历史                         [查看摘要]           |
| v1  rejected · 安全失败 code                       [查看状态] [新建版本]       |
|                                                                              |
| 治理                                                                      |
| [soft-delete]（legal hold 时禁用） [restore]（仅 pending_delete）              |
| [设置/解除 legal hold]（仅 Founder + allowed_actions）                       |
|                                                                              |
| 成功：显示 server receipt、最新 record_version 和下一步；                   |
| stale：重新读取后要求再次确认；denied/unavailable：不暴露对象内容。          |
+--------------------------------------------------------------------------------+
~~~

预览和下载都先调用精确 Version capability 命令，再由浏览器消费短时结果；列表不保存永久 URL。预览不降低下载权限，不能从 preview fallback 到对象直链。

### 5.3 W08：版本治理移动

~~~text
+----------------------------------+
| < 文件   opaque-••31             |
| submission_evidence              |
| Case ••42 / SchoolTarget ••8     |
| active · legal hold：是          |
|                                  |
| v3 · available · clean           |
| [预览] [下载] [引用 Task]        |
| v2 · available · 历史            |
| v1 · rejected                    |
| [查看状态] [新建 Version]        |
|                                  |
| [soft-delete]（hold 禁用）       |
| [restore]（pending_delete 时）   |
| [治理旗标]（Founder 才显示）     |
|                                  |
| [重新读取] [返回来源]             |
+----------------------------------+
~~~

### 5.4 文件操作矩阵

| 操作 | 按钮位置 | 前置条件 | 成功反馈 | 失败/冲突反馈 |
| --- | --- | --- | --- | --- |
| 注册 Document | Case Documents 顶部 `[注册文件]` | Case 关系、用途和 scope 有效；当前用户有 create | receipt 的 Document/Version opaque IDs、`pending_upload`、最新 record_version | `VALIDATION_FAILED` 就地字段错误；`FORBIDDEN/NOT_FOUND` 通用拒绝；重复 Idempotency-Key 重放原 receipt |
| 创建新 Version | 文件行 `[新建版本]` | Document active、当前关系有效、用途不扩大 scope | 新 Version `pending_upload`，旧 Version 不覆盖 | `STALE_VERSION` 刷新 Document；legal hold/生命周期冲突显示安全原因 |
| 取得 upload capability | 上传表单主按钮后 | 精确 Document/Version、expected version、文件输入 | 只保存短时 capability 到当前内存上传流程；显示上传阶段 | capability 失效/服务不可用显示 retry；不写入 URL、日志或持久页面状态 |
| 提交 object receipt | 浏览器上传完成后自动/明确 `[确认上传]` | capability、opaque object receipt、checksum/size/version 绑定 | `quarantined`，随后 `scanning`；显示 request id | receipt 重复返回原结果；绑定不符 `CONFLICT/VALIDATION_FAILED`；不显示 provider 细节 |
| 激活 Version | available 行 `[激活]`（若 allowed） | clean、available、未 revoked、expected record version | 显示 active Version 和旧 Version 保留；Tasks 可重新引用 | `STALE_VERSION` 重载；扫描/引用/权限不满足 `CONFLICT`；不伪造 active |
| 预览/下载 | available 行右侧 `[预览]`/`[下载]` | 当前授权、精确 Version available/clean/未 revoked | capability 成功后打开受控结果；页面只保留 receipt 摘要 | denied/not_found 统一提示；失效/不可用可重试；不显示公开 URL |
| soft-delete | 文件治理操作区 `[soft-delete]` | Primary Advisor Case 权限、active Document、理由、expected version、无 legal hold | lifecycle=`pending_delete`，显示恢复截止窗口和 receipt | legal hold/非 active/stale 返回 `CONFLICT/STALE_VERSION`；不删除 metadata |
| restore | pending_delete 行 `[restore]` | 当前授权、30 天窗口、clean Version、expected version | lifecycle=`active`，Version/扫描历史不改写 | 窗口过期、无 clean Version、失权或 stale 显示安全失败；不假装恢复 |
| legal hold | Founder 治理区 | Founder + owner 返回的治理 action、理由、expected version、Audit | legal hold 旗标与最新 record version | 非 Founder/缺 route/版本冲突只读或 denied；不提供对象直接清理 |
| 引用到申请 Task | available 行 `[引用 Task]` | 当前 Application Assignee、用途/SchoolTarget 匹配、未 revoked | 只回传 opaque Version ID；Task 显示引用待 Cases/Tasks 重验 | scope 不匹配/文件不可用/stale 显示 validation 或 conflict；不改 SchoolTarget |

## 6. Documents 页面状态与交互反馈

| 页面/操作 | loading | empty | validation | denied | unavailable/error | stale/version conflict | success |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `/documents` 目录 | 标题、筛选骨架、按钮禁用 | 当前 scope 无文件；保留“清除筛选/回 Case” | 不接受非法 scope/用途组合 | 通用“无法查看文件”，不泄露对象存在 | 服务暂不可用 + 重试；无 provider/key/stack | 重新读取当前列表，不覆盖本地筛选意图 | 以 server DTO 更新行和状态 |
| `/cases/:caseId/documents` | Case layout 保留标题和返回 | Case 或 SchoolTarget scope 无文件；不等于无权 | 用途、学校范围、文件输入错误 | 子路由统一 denied；不显示文件行 | Documents/Case runtime error 与 empty 分开 | expected version 不符，局部刷新 Document/Case | 新注册/新 Version receipt 后回当前 Case 子路由 |
| upload capability/object receipt | 按阶段显示 pending/processing | 没有已注册 Version 时显示注册入口 | content type、size、checksum、scope 由 server 复核 | 无 upload capability 时隐藏主按钮 | 结果未知显示“稍后重试”，不显示已完成 | Version 被其他人更新时重新读取并重选文件 | `quarantined`/`scanning` receipt；最终由 server 状态驱动 |
| scanning/rejected/scan_failed | 轮询或手动刷新，不假定 clean | 无 active Version 不等于无权 | 不把原始扫描输出放入表单 | 无下载按钮 | `scan_failed` 显示有限重试/建立新 Version；`rejected` 安全失败 | 扫描状态改变时刷新行，不覆盖新 Version | `available` 才显示预览/下载/激活 |
| preview/download | capability 请求 loading，按钮防重复 | 无 available Version 显示不可用 | 版本/动作组合不符 | 通用无法访问 | capability/对象 runtime 不可用可重试 | Version revoked/changed 时要求重新读取 | 只显示短时结果已签发，不保存永久 URL |
| soft-delete/restore/legal hold | 操作按钮进入 pending | 无可治理行显示说明 | 理由、expected version 缺失 | Founder/Primary Advisor boundary enforced server-side | 受控错误 + request id | 重新读取 lifecycle/legal_hold 后再次确认 | receipt + 最新 lifecycle/record_version |
| Task evidence 引用 | 读取 Task/Document scope | 无可用证据显示“扫描通过后可选择” | 用途、SchoolTarget、Version 不匹配 | 非 Application Assignee 隐藏引用按钮 | Tasks/Documents 任一 runtime 不可用，停在当前 Task | Task 或 Document 版本冲突分别刷新 owner DTO | opaque Version ID 写入 Task completion payload；Cases 重验 |

所有页面还遵循 F0 状态基线：`not_found` 只有服务端允许区分时才展示；否则使用通用 unavailable/denied；错误不得显示 PII、object key、Token、provider、SQL 或堆栈。

## 7. Notifications 页面信息架构与线框

### 7.1 通知可见事实

Notifications 只显示当前 Session User 的站内通知。可见状态只有 `unread -> read`；`suppressed`、`failed`、dead-letter 和投递内部错误不创建或不展示用户可见 Notification。Release 1 文案固定为“有待办事项需要处理”，不显示 Student、Guardian、Case、学校、Task 标题、文件名、原因或自由文字。

通知来源仍由 BR-038/approved flow 决定：Task 分配/重派/拒绝、候选名单 Founder 审批请求/结果、到期前 3 天/1 天、逾期每日、全部学校终态且无未完成 Task 的结案选择提醒。通知不是 Task、业务事实或权限。

### 7.2 W09：Notifications 桌面

~~~text
+--------------------------------------------------------------------------------+
| [≡] 通知                         未读 3                         [账号]          |
+--------------------------------------------------------------------------------+
| 站内通知                                                                       |
| [全部] [未读]                                             [标为已读（逐条）]   |
|                                                                              |
| ● 有待办事项需要处理                                  未读   [打开]           |
|   2026-08-26 09:10 · allowed_actions: read, resolve_target                    |
|                                                                              |
| ○ 有待办事项需要处理                                  已读   [打开]           |
|   2026-08-25 17:40 · allowed_actions: read, resolve_target                    |
|                                                                              |
| ○ 有待办事项需要处理                                  已读   [打开]           |
|   2026-08-25 10:00 · allowed_actions: read                                    |
|                                                                              |
| 空态：暂无待办事项                                                             |
| unavailable/error：通知服务暂不可用 [重试]；不显示 suppressed/failed            |
+--------------------------------------------------------------------------------+
~~~

“打开”先调用 `resolve-target`，服务端以当前 Session User 重新授权并返回固定 route code；客户端不读取或拼接 target token。已读按钮只能作用于当前 recipient，必需通知没有关闭、删除、静音按钮。

### 7.3 W10：Notifications 移动

~~~text
+----------------------------------+
| < 返回        通知        [账号] |
| 未读 3   [全部] [未读]            |
|                                  |
| ● 有待办事项需要处理             |
|   未读 · 09:10       [打开]      |
|                                  |
| ○ 有待办事项需要处理             |
|   已读 · 昨日        [打开]      |
|                                  |
| 暂无待办事项 / 暂不可用           |
| [重试]                           |
+----------------------------------+
~~~

移动端点击通知不直接跳转未知 URL：先显示短暂 resolving 状态，成功后进入固定路由；目标失效显示 unavailable 并留在列表。窄屏不显示隐藏的业务对象摘要。

### 7.4 通知操作矩阵

| 操作 | 位置 | 前置条件 | 成功反馈 | 失败/冲突反馈 |
| --- | --- | --- | --- | --- |
| 读取列表 | `/notifications` 首屏/重试 | 当前 Session User | 只显示属于当前 recipient 的最小通知 | unauthenticated/denied 通用；不接受 client user id |
| 读取 unread count | shell 通知图标 | 当前 User 查询授权 | 显示当前投影 count；count 不作为权限 | runtime unavailable 显示暂不可用，不用旧值冒充 |
| 标记已读 | 每条通知的 `[标为已读]` | 当前 recipient、状态 unread、expected record version | `read_at`/最新 record version 返回，行更新为已读 | stale 要求重新读取；重复 Idempotency-Key 重放原结果；不停止未来提醒 |
| resolve-target | 每条 `[打开]` | notification 有 target token 和 `resolve_target` allowed action | server 返回固定 route code；客户端导航后 owning module 再读 | 失权/过期/Case 关闭统一 unavailable；不泄露 target 是否存在 |
| 关闭/删除/静音 | 不提供按钮 | 必需通知不能关闭 | 无 | UI 不发请求；suppressed/failed 不在页面显示 |

## 8. 统一 API、DTO、错误、幂等和版本契约

### 8.1 Approved Documents public API/DTO/权限追溯

| 页面动作 | approved endpoint | 请求/响应最小契约 | UI 权限边界 | 事实来源 |
| --- | --- | --- | --- | --- |
| Case 文件列表 | `GET /api/v1/cases/:caseId/documents` | Document metadata、active Version、scan、lifecycle、legal_hold、allowed_actions；无 object key/公开 URL/token/文件内容 | Case owner/Primary Advisor/Founder 按资源关系；Application Assignee 只看该校必要文件 | BR-050；approved Documents UI |
| SchoolTarget 文件列表 | `GET /api/v1/school-targets/:targetId/documents` | 当前 Application Assignee 的该校文件 scope 和版本摘要；不得扩大 Case | Primary Advisor/授权 Advisor/Application Assignee；Contractor/Guardian/Student/Portal denied | BR-050；approved Documents UI |
| 注册 Document/第一 Version | `POST /api/v1/cases/:caseId/documents` | 服务端注册 metadata/用途/scope，返回 opaque receipt；`Idempotency-Key`、统一 envelope、`no-store`、`X-Request-Id` | 有 Case create/upload capability 且用途合法 | approved Documents UI；BR-050/070 |
| 创建不可变 Version | `POST /api/v1/documents/:documentId/versions` | 新 Version receipt；不得覆盖旧 Version；带 expected record version | 当前 Document owner/允许的 Application Assignee scope | approved Documents UI；BR-050 |
| upload capability | `POST /api/v1/documents/:documentId/versions/:versionId/upload-intent` | 短时 capability 绑定精确 Version、对象引用、checksum/size/content type；secret 不进入页面持久状态 | 当前资源授权；UI 只调用，不直连 provider | approved Documents UI；BR-050 |
| object receipt | `POST /api/v1/documents/:documentId/versions/:versionId/object-receipt` | 校验 opaque object receipt、Version、checksum/size；结果进入 quarantined/scanning | 仅当前上传流程；重复 receipt 幂等 | approved Documents UI；BR-050/070 |
| 激活 Version | `POST /api/v1/documents/:documentId/activate` | 只接受 clean、available、未 revoked、expected version；返回最新 active pointer | 资源授权 + allowed action；不由客户端标 clean | approved Documents UI；BR-050 |
| preview/download capability | `POST /api/v1/documents/:documentId/versions/:versionId/download-capability` | 精确 Version 的短时结果；预览不降低下载权限；无永久 URL | Founder/Primary Advisor/Application Assignee 依资源规则 | approved Documents UI；BR-050/071 |
| soft-delete | `POST /api/v1/documents/:documentId/soft-delete` | 理由、expected version、Idempotency-Key；返回 lifecycle/active version/record version | 当前 Primary Advisor；无 legal hold | approved Documents UI；BR-050/070 |
| restore | `POST /api/v1/documents/:documentId/restore` | 30 天窗口、clean Version、expected version；返回 active lifecycle receipt | 当前 Primary Advisor；pending_delete 内 | approved Documents UI；BR-050/070 |

Approved Documents UI 将具体上传 provider 隔离在 server-only adapter；F4 不复制对象存储/扫描器 DTO，也不在浏览器使用 object key。

### 8.2 Approved Notifications public API/DTO/权限追溯

| 页面动作 | approved endpoint | 请求/响应最小契约 | UI 权限边界 | 事实来源 |
| --- | --- | --- | --- | --- |
| 通知列表 | `GET /api/v1/notifications` | 当前 recipient 的 `id, content_code, status, created_at, read_at, record_version, target_kind/token, allowed_actions`；不接受 User ID | 当前 Session User；Admin 不因基础角色自动获得客户业务通知；Contractor 不进全局列表 | BR-038/070/071；approved Notifications UI |
| unread count | `GET /api/v1/notifications/unread-count` | 当前 User 的投影 count；不作为权限来源 | 只显示在有通知入口 capability 的 shell | BR-038；approved Notifications UI |
| mark read | `POST /api/v1/notifications/:notificationId/read` | `expected_record_version`；`Idempotency-Key`；返回最新 status/read_at/record_version | 只有当前 recipient；必需提醒不可关闭但可读 | approved Notifications UI；BR-070 |
| resolve target | `POST /api/v1/notifications/:notificationId/resolve-target` | 服务端读取 Session、重新授权，返回固定 route code；不返回任意 URL | 目标 owner 再次授权；失效为 unavailable | approved Notifications UI；BR-038/071 |

### 8.3 统一错误和反馈

所有 F4 command 采用 approved shared envelope、`X-Request-Id`、`no-store`，写入带 `Idempotency-Key`；客户端只消费稳定 allowlisted code。页面至少处理：

| 稳定结果 | 页面处理 |
| --- | --- |
| `UNAUTHENTICATED` | 显示工作阶段失效，提供重新登录；不保留敏感内容 |
| `FORBIDDEN` / 通用 denied | 通用无权页面；不确认资源存在 |
| `NOT_FOUND` | 只有 owner 允许区分时显示；否则统一 denied/unavailable |
| `VALIDATION_FAILED` | 就地字段/文件输入错误；不清空用户可重试的非敏感输入 |
| `STALE_VERSION` | 重新读取 owner DTO，显示版本已变化；不静默覆盖 |
| `CONFLICT` | 显示业务前置、legal hold、生命周期或幂等冲突的受控文案 |
| `SERVICE_UNAVAILABLE` / unavailable | 显示暂不可用 + 重试；不显示 provider/数据库细节，不伪造成功 |
| 成功 receipt | 以 server status/record_version 为准，更新当前行并给出返回路径 |

### 8.4 版本、幂等和缓存原则

- Document 注册、Version 创建、upload intent、object receipt、activate、soft-delete、restore 和 Notification read 都以服务端 receipt 为成功依据。
- 所有重复写入使用同一 `Idempotency-Key` 重放原结果；key 重用不同 payload 返回受控 `CONFLICT`，不得重复 Version、通知或审计。
- 文件和通知 API 查询使用 `no-store`；浏览器缓存、unread count、列表空态和通知 target 都不是授权来源。
- `record_version` 不匹配返回 `STALE_VERSION`；客户端先重读，再由用户重新确认高风险操作。
- 审计与业务命令同事务；F4 页面不显示审计原文，通知不复制 PII。

## 9. 当前源码路线盘点与目标差异

以下只记录当前 Tianxingguoji worktree 的实现证据，不覆盖 confirmed BR 或 approved Documents/Notifications 目标契约。

| 当前路由/源码证据 | 当前实现 | F4 目标分类 |
| --- | --- | --- |
| [`/documents`](</Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/(erp)/documents/page.tsx:1>) | 真实页面挂载 `DocumentsDirectory`，可按 classification/lifecycle/version 在客户端过滤 | `redesign`：改为服务端 Case/SchoolTarget/Assignee 资源过滤，补齐版本治理和安全状态 |
| [`DocumentsDirectory.tsx`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/documents/DocumentsDirectory.tsx:31) | 读取 `listDocuments`，显示本地筛选、loading/unauthenticated/denied/unavailable/empty | `redesign`：不把客户端过滤当授权；补齐 cursor/owner DTO、stale/error/success 和 scope |
| [`CaseDocumentsPanel.tsx`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/documents/CaseDocumentsPanel.tsx:24) | Case 页面已有登记、上传、下载 capability 调用和基础冲突提示 | `redesign`：对齐 approved endpoint 命名、object receipt、scan 状态、SchoolTarget/Assignee scope、soft-delete/restore/legal hold |
| [`document-ui.tsx`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/documents/document-ui.tsx:27) | 当前行显示 `display_name`、Case number、classification/lifecycle/version pill | `redesign`：目标 DTO 不泄露未经批准的原始文件名、Case number 或 provider 信息；改受控 opaque/purpose/scope 摘要 |
| [`app/api/v1/cases/[caseId]/documents/route.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/cases/[caseId]/documents/route.ts:20) | 当前 Case list/register route；实现使用 shared handler 和 runtime | `keep/redesign`：保留 owner 边界，按 approved DTO/receipt、统一 headers、resource authorization 复核 |
| [`app/api/v1/cases/[caseId]/documents/[documentId]/versions/[versionId]/upload-intents/route.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/cases/[caseId]/documents/[documentId]/versions/[versionId]/upload-intents/route.ts:1) | 当前 plural `upload-intents` runtime route | `redirect/isolate_from_release1`：目标 UI/API 使用 approved `.../upload-intent` 命令；不在 F4 修改源码 |
| [`app/api/v1/cases/[caseId]/documents/[documentId]/download-intents/route.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/cases/[caseId]/documents/[documentId]/download-intents/route.ts:1) | 当前 download intent route；返回 bytes 流程由 client 消费 | `redesign`：目标为精确 Version `download-capability`，每次重新授权、无永久 URL |
| [`deletions/route.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/cases/[caseId]/documents/[documentId]/deletions/route.ts:17) / [`restorations/route.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/cases/[caseId]/documents/[documentId]/restorations/route.ts:17) | 当前 route 已解析 expected version/Idempotency-Key，并映射 stale/conflict | `redesign`：目标 Documents command 使用 approved soft-delete/restore path 和 legal hold/30 天窗口 DTO |
| [`navigation-registry.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/layout/navigation-registry.ts:27) | 当前有 `/documents` nav，required capability `documents.read`；没有 `/notifications` nav item | `/documents` `keep/redesign`；`/notifications` `missing`，由 TopBar/新入口按 approved capability 暴露 |
| [`TopBar.tsx`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/layout/TopBar.tsx:101) | 当前通知按钮只打开 `notifications_unavailable` 静态 popover，没有通知 list/read/resolve API | `redesign`：unread count、最小文案、read/resolve-target 和 safe unavailable/error |
| `app/**` 通知页面/API 文件盘点 | 当前没有正式 `/notifications` page，也没有 `/api/v1/notifications` route；`modules/notifications` 仅有 domain/server/runtime/repository | `/notifications`、四个 Notifications API 入口 `missing`（设计文件不实现） |
| [`modules/notifications/domain/contract.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/modules/notifications/domain/contract.ts:1) | 当前域契约已有 `in_app`、minimal content code、unread/read/suppressed 和 delivery effect idempotency | `keep as evidence`：UI 只展示 approved user-visible unread/read；suppressed/failed 不显示 |

旧实现中的 `display_name`、Case number、plural intent route、客户端 filtering、静态 unavailable popover 和通用 `/documents` list 都只是现状证据；它们不能覆盖 BR-050、BR-038 或 approved API/UI 的安全边界。

## 10. 桌面、移动和可访问性原则

- 桌面端保留左侧一级导航、稳定的 Case context bar 和列表筛选栏；文件行的状态、版本和操作按固定列顺序展示，避免卡片套卡片。
- 移动端将筛选收进单一 sheet，文件/通知行改为纵向字段组；主操作固定在当前上下文底部，但不遮挡错误、键盘焦点或返回链接。
- 状态不能只靠颜色表达；`available`、`scanning`、`rejected`、`scan_failed`、`pending_delete` 和 `legal_hold` 同时使用文字、图标/标签和允许动作。
- 所有 icon-only 控件提供可读 aria-label/tooltip；通知、下载、恢复和删除等高风险动作使用明确文字按钮。
- 加载、空、拒绝、不可用、冲突和成功反馈保留标题与上下文；不把“无数据”与“无权限”混用，不在页面显示 PII、Token、object key 或 provider 错误。
- 断点、筛选和 scope 变化不改变服务端授权；浏览器返回只恢复导航上下文，不恢复过期 capability 或旧 record version。

## 11. 追溯、验收和交付摘要

### 11.1 事实来源

- [confirmed Documents BR-050](../../../business-requirements/60-documents.zh-CN.md)
- [confirmed Notifications/Audit BR-038、BR-070、BR-071](../../../business-requirements/70-notifications-audit.zh-CN.md)
- [approved Documents 文件工作台 API/UI](../30-documents-workspace.zh-CN.md)
- [approved Notifications 入口 API/UI](../40-notifications-ui.zh-CN.md)
- [approved Authorization model](../../40-permissions-security/10-authorization-model.zh-CN.md)
- [approved F0 信息架构](./10-information-architecture.zh-CN.md)
- [approved F1 Today/Cases/Workspace](./20-f1-today-cases-wireframes.zh-CN.md)
- [approved F2 CRM/Case intake](./30-f2-crm-case-intake-wireframes.zh-CN.md)
- [approved F3 Schools/Applications/Interviews](./40-f3-school-selection-application-interview-wireframes.zh-CN.md)

### 11.2 Architect 验收对照

| 门禁 | 本文证据 | 结果 |
| --- | --- | --- |
| 只覆盖 Documents 与 Notifications | §0.1、§0.3；未设计 Portal/F5/外部渠道 | 满足 |
| 文件资源级过滤 | §1.2、§3.1、W01–W04；Case/SchoolTarget/Assignee 由 owner 重验 | 满足 |
| 上传到扫描完整链路 | W05–W06、§5.1、§6；pending_upload/quarantined/scanning/available/rejected/scan_failed/abandoned | 满足 |
| 预览/下载 capability 与安全边界 | §5.2、§5.4、§8.1；不使用公开 URL/object key | 满足 |
| soft-delete/restore/legal hold | §5.1、W07–W08、§5.4；30 天窗口、hold 阻断、Founder/Primary Advisor 边界 | 满足 |
| Notifications 当前 User/unread/read/resolve | §7、W09–W10、§7.4、§8.2；suppressed/failed 不展示 | 满足 |
| 角色与拒绝边界 | §2；Founder/Admin/Advisor/Contractor/Guardian Viewer、组合身份和 Contractor 隔离 | 满足 |
| loading/empty/denied/unavailable/stale/error/success | §6 与 §7.4；写操作均有 receipt/重试/冲突语义 | 满足 |
| F3/Task 连续性 | §0.2、§1.2、§4.1、Task evidence 行 | 满足 |
| 当前实现证据与目标区分 | §9；只读源码路径，未修改产品仓库 | 满足 |

### 11.3 交付字段

| 字段 | 值 |
| --- | --- |
| `status` | `approved` |
| `owner` | `frontend-design` |
| `scope` | F4 Documents 与 Notifications 低保真线框及交互契约 |
| `changed_files` | 仅新增本文件；产品源码、测试、数据库、migration、云配置均未改 |
| `wireframe_count` | 10（W01–W10，桌面/移动覆盖 Documents 目录、Case 文件、上传传输、版本治理、Notifications） |
| `decision_count` | 0（未新增业务实体、角色、状态或渠道；复用 approved transport） |
| `review_readiness` | `ready_for_architect_review` |
| `browser` | `not_run` |
| `database` | `not_run` |
| `cloud` | `not_run` |
| `product_tests` | `not_run` |

### 11.4 实现门禁与风险

- F4 已通过 Architect 复审，可按本文件进入前端实现；实现仍不得绕过 owning module，也不得把未冻结的 legal hold command path 自行发明为浏览器 endpoint。
- legal hold 的具体 command path/DTO 未在 approved Documents API/UI 正式路径表中列出；本文因此只设计旗标、前置条件和 `allowed_actions` 驱动的入口，不发明浏览器 endpoint。实现前由 Documents owner/Architect 提供已批准 transport，缺失时该按钮保持只读/隐藏。
- `/notifications` 页面和 Notifications 四个 `/api/v1` 入口在当前源码中缺失；本文件将它们标为目标 `missing`，不把现有静态 unavailable popover 当作运行证据。
- 当前 Documents 代码和路由可能与 approved target path 命名不同；本文件只记录差异，不在本轮修复或重命名。

本文件完成 F4 设计交付，状态为 `approved`；已允许按本契约进入前端实现，F5 仍需单独完成设计审核。
