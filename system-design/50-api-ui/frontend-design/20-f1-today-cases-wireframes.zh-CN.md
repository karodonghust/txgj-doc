# F1 Today、Cases 列表与案件工作区低保真线框

状态：`approved`
Owner：frontend-design
Architect 复审：2026-08-26 通过；四项阻塞决定已完整落地，无遗留 F1 设计阻塞
前置基线：F0 信息架构已由项目负责人于 2026-08-26 确认；业务基线 BR-BASELINE-20260825-v32

返回：[F0 前端信息架构](./10-information-architecture.zh-CN.md)

## 0. 本轮范围与不变约束

本轮只冻结 Today、Cases 列表和案件工作区的低保真结构、交互状态、入口连续性和权限追溯。没有页面源码、React/TSX、API、DTO 实现、数据库、migration 或测试改动；不定义颜色、品牌、动效或高保真视觉。

线框只使用已批准事实：

- [confirmed Cases BR-030 至 BR-034、BR-039](../../../business-requirements/30-cases.zh-CN.md)；
- [confirmed Task BR-035 至 BR-037](../../../business-requirements/40-tasks.zh-CN.md)；
- [confirmed Notifications/Audit BR-038、BR-070、BR-071](../../../business-requirements/70-notifications-audit.zh-CN.md)；
- [approved Cases 案件工作台 API/UI](../10-cases-workspace.zh-CN.md)；
- [approved Task 工作台 API/UI](../20-task-workspace.zh-CN.md)；
- approved Notifications 入口 API/UI；
- F0 已批准的信息架构和角色边界。

页面不能自行推进 Case、SchoolTarget 或 Task 状态；页面只能根据 owning module 返回的 DTO、allowed_actions、record version 和安全 error code 展示或提交操作。

## 1. F1 页面边界与入口连续性

### 1.1 页面入口

| 页面 | 目的 | 进入 | 返回 |
| --- | --- | --- | --- |
| /today | Founder、Admin、Advisor 的授权工作摘要、风险、即将到期任务和通知入口 | 非 Contractor 登录成功、根路径、Sidebar | 进入 /cases、/tasks、/cases/:caseId；返回由浏览器历史或明确的“查看全部”链接完成 |
| /today（Contractor 显式访问） | 统一 denied 页面；不查询或展示 Case 内容 | Contractor 输入 URL 或旧书签 | 只提供“返回我的任务”链接到 /tasks；不重定向到单个 Task |
| /cases | 授权 Case 列表和下一步摘要 | Sidebar、Today 的“我的案件” | 返回 /today；保留用户当前筛选/排序作为页面查询上下文，不改变授权 |
| /cases/new | 建立或关联 Student、指定 Primary Advisor 的入口 | /cases、/today、Student 详情 | 成功进入 canonical /cases/:caseId；取消返回来源列表 |
| /cases/:caseId | Case workspace canonical summary；承载共享 Case layout | /cases 行、Today 行、通知目标 | 返回 /cases 或原始授权来源；子路由导航不改变 Case 事实 |
| /cases/:caseId/workspace | 旧兼容 URL，不渲染工作区 | 外部/旧链接 | 308 重定向到 /cases/:caseId，保留 canonical 语义 |
| /tasks/:taskId | 内部 Task 详情和完成表单 | Today、Case workspace、/tasks | 完成/拒绝/重派后返回来源；保留 Task 历史 |
| /contractor/tasks/:taskId | F0 兼容入口 | 外部/旧链接 | 只能进入 Contractor 脱敏工作区；不得落入完整 Case 或内部 Task DTO |

### 1.2 目标页面关系

~~~text
登录
  |-- Contractor --------------------> /tasks
  |                                      显式 /today -> denied -> [返回我的任务]
  |
  +-- Founder / Admin / Advisor -----> /today
                                         |-- 我的案件 ------------> /cases
                                         |-- 风险案件/下一步 ------> /cases/:caseId
                                         |-- 即将到期任务 --------> /tasks/:taskId
                                         +-- 通知 --resolve-target-> 固定授权 route

/cases
  |-- 新建 --------------------------> /cases/new -> /cases/:caseId
  |-- Case 行 -----------------------> /cases/:caseId（canonical summary）
  +-- 旧 /cases/:caseId/workspace ---> 308 -> /cases/:caseId

/cases/:caseId（共享 Case layout）
  |-- /assessment
  |-- /schools
  |-- /applications
  |-- /interviews
  |-- /documents
  |-- /access
  +-- /close
~~~

Case workspace 的 tab 是共享 Case layout 内的导航链接，每个 tab 对应独立子路由，不使用 query tab。每次进入 summary 或子路由都重新执行服务端资源授权；deep link、浏览器返回和列表查询上下文可保留，但都不是授权来源。

### 1.3 Canonical 与 Case 子路由矩阵

页面 URL 与聚合 API 是两个契约：页面 canonical 是 /cases/:caseId；共享 layout 的 summary 仍调用 GET /api/v1/cases/:caseId/workspace。旧页面 URL /cases/:caseId/workspace 只执行 308 重定向，不渲染第二套 workspace，也不改变聚合 API 路径。

| 导航标签 | 页面路由 | 主要读取/操作 | Deep link 与返回 | 授权边界 |
| --- | --- | --- | --- | --- |
| 摘要 | /cases/:caseId | workspace 聚合 API；阶段、下一步、风险、负责人 | Cases/Today/通知可直接进入；返回保留来源查询上下文 | 进入 canonical 时重新授权 |
| Assessment | /cases/:caseId/assessment | Assessment 查询与背景完成命令 | 可直接收藏；返回 Case summary 或原来源 | 路由加载和每次写入均重新授权 |
| 候选学校 | /cases/:caseId/schools | candidate lists、Founder review、Guardian decision、SchoolTarget 摘要 | summary 或 next step 进入；返回同一 Case | 不因 Case summary 可见而自动取得写权限 |
| 逐校申请 | /cases/:caseId/applications | SchoolTarget、申请 Task 和提交凭证摘要 | 学校行/Task 返回该路由 | Cases、Tasks、Documents 各自重验资源权限 |
| 面试 | /cases/:caseId/interviews | interview_support Task、分派与受控摘要 | 申请或 Task 入口进入；内部用户返回该 Case | Contractor 不进入此路由，只进脱敏 Task workspace |
| 文件 | /cases/:caseId/documents | Case 文件 metadata、扫描状态和 allowed actions | 申请/Task 可带受控来源上下文 | Documents owning module 逐资源重验 |
| Portal 授权 | /cases/:caseId/access | Grant 创建、撤销、重新生成和状态摘要 | summary 进入；完成后返回同一路由 | Primary Advisor/Founder 按 approved 授权规则重验 |
| 人工结案 | /cases/:caseId/close | 结案前置、结果与原因提交 | summary/所有学校拒绝提示进入；成功回 summary | 仅 Founder 且所有 BR-039 前置成立 |

共享 layout 只提供 Case 标题、阶段、返回链接和上述导航，不缓存跨子路由写权限。子路由 denied 时保留通用页面框架但不展示受保护内容；not_found 与 denied 的区分仍由服务端安全契约决定。

## 2. Today 工作台

### 2.1 信息分区

Today 只回答四个问题：

1. 我现在有哪些授权待办？
2. 哪些授权案件有 blocker、版本冲突、逾期或需要人工选择？
3. 哪些 Task 接近截止或已经 overdue？
4. 我是否有新的站内待办提醒？

| 区域 | 显示 | 不显示 |
| --- | --- | --- |
| 我的待办 | 当前 User 可执行的 Task、Case action 或固定通知目标；显示受控标题、owner kind、due_at、allowed action | 未授权 Case、完整通知正文、任意 URL |
| 案件风险 | 当前授权 Case 的风险 code、下一步 code、负责人摘要、freshness/stale 标记 | 从 projection 推断的授权、Assessment 原文、未授权客户资料 |
| 即将到期任务 | Tasks 返回的 due_at、is_overdue、Task 类型和 allowed actions；按正式事实排序 | 页面本地计算的新状态、暂停后顺延的假日期 |
| 通知入口 | 当前 User 的 unread count 和最小“有待办事项需要处理”文案 | Student、Guardian、学校、Case number、Task 名称、文件名或自由文字 |
| 页面状态 | 当前数据时间、重试、空状态、不可用原因 | provider、stack、token、数据库信息 |

### 2.2 角色可见范围

| 角色 | Today 可见 | Today 不可见/入口边界 |
| --- | --- | --- |
| Founder | 组织授权范围内的 Case 风险、结案选择提醒、逾期提醒、自己的 Task、Operations/Audit 入口 | 不因 Founder 自动获得每个文件下载权；仍按 owning module 资源规则 |
| Admin | 仅技术健康、无客户内容的运行提示和账号管理入口 | 默认不显示 Case、Assessment、Student/Guardian、客户 Task、客户文件或租户业务审计 |
| Advisor | 自己负责或明确授权 Case 的风险、下一步、Task、名单审批跟进和通知 | 不显示其他 Case；不把通知或投影当作额外权限 |
| Contractor | 登录后默认进入 /tasks；显式访问 /today 时只见统一 denied 和“返回我的任务”链接 | 不显示全局 Today、Case、Student/Guardian、其他 Task、文件或通知列表；不重定向到某个单 Task |

Contractor 不拥有 Today 工作台。denied 页面不发起 Cases 查询、不显示 Case 是否存在，也不从 /today 推断或选择某个 Assignment；唯一操作位于主内容区，为“返回我的任务”链接到 /tasks。

### 2.3 桌面线框：Today

~~~text
+--------------------------------------------------------------------------------+
| [≡] 今日工作       香港组织 / Release 1              [通知 3] [账号]           |
+----------------------+---------------------------------------------------------+
| 工作台               | 今日工作                                               |
|   今日工作           | 授权范围内的下一步和异常                                |
|   案件               |                                                         |
|   学生与监护人       | +-------------------+ +-------------------+ +---------+ |
|   任务               | | 我的待办  4       | | 案件风险  2       | | 到期 1 | |
|   文件               | | [打开任务]        | | [查看案件]        | | [打开] | |
|   学校资料           | +-------------------+ +-------------------+ +---------+ |
|                      |                                                         |
| 管理                 | 我的待办                                               |
|   账号与成员         | [Task/Case 受控摘要]   负责人  due_at  [查看]        |
|   学校治理           | [Task/Case 受控摘要]   负责人  due_at  [查看]        |
|   运营与审计         |                                                         |
|                      | 案件风险                                               |
|                      | [risk code] [下一步 code] [负责人] [freshness] [查看]  |
|                      |                                                         |
|                      | 即将到期任务                                           |
|                      | [Task type] [due_at] [overdue?] [allowed action]       |
|                      |                                                         |
|                      | 通知入口                                               |
|                      | 有待办事项需要处理 (3)                     [查看通知]   |
+----------------------+---------------------------------------------------------+
| Session / 角色摘要；不显示敏感业务内容；退出                         |
+--------------------------------------------------------------------------------+
~~~

### 2.4 移动线框：Today

~~~text
+--------------------------------------+
| [☰] 今日工作              [通知 3]  |
+--------------------------------------+
| 授权范围内的下一步                    |
|                                      |
| 我的待办 4                 [查看全部]|
| [短摘要]  due_at          >          |
| [短摘要]  due_at          >          |
|                                      |
| 案件风险 2                 [查看全部]|
| [risk code] [下一步]       >          |
|                                      |
| 即将到期任务 1                       |
| [Task type] [due_at]      [打开]     |
|                                      |
| 有待办事项需要处理 (3)   [查看通知]  |
+--------------------------------------+
| [菜单抽屉：工作台 / 案件 / 任务 / ...] |
+--------------------------------------+
~~~

移动端只保留一个打开导航控制；通知文案仍不披露业务对象。

## 3. Cases 列表

### 3.1 列表字段和筛选

Cases 列表只显示 approved Cases query 返回的受控摘要，字段基线如下；未在 approved DTO 中冻结的字段不得由前端自行增加。

| 字段/控件 | 用途 | 边界 |
| --- | --- | --- |
| Case 受控标识 | 进入 Case；内部可显示 approved case reference/摘要 | 不向 Portal 复用；不从 URL 推导权限 |
| 当前阶段 | signed、background_collection、school_selection_confirmed、application_in_progress、closed | 暂停/终止作为 workflow status 或风险摘要，不新增 Case 阶段 |
| workflow status | 显示暂停、待处理、终止等已批准摘要 | 不由页面本地转换阶段 |
| Primary Advisor/负责人摘要 | 帮助排序和下一步 | 只显示当前用户获授权的受控值 |
| next_step / risk | 案件风险、下一步 code、due_at | projection stale 时明确标记；不能直接执行写入 |
| 搜索 q | 仅搜索 API 允许的 Case 受控摘要字段 | 不支持姓名/电话等未批准的越权搜索 |
| stage | 按 approved Case stage 过滤 | 空筛选结果不是 denied |
| workflow_status | 按 approved workflow status 过滤 | 不新增 Case 阶段 |
| risk_code | 按服务端受控风险 code 过滤 | 不由页面计算或自由输入 code |
| advisor_user_id | 按 API 返回的可选 Advisor 过滤；“与我有关”可作为当前 Advisor 快捷项 | 不传 organization/actor，不扩大当前授权集合 |
| 排序 | 固定为 next_step.due_at ASC NULLS LAST、updated_at DESC、id ASC | 不提供浏览器本地排序或未批准 sort 参数 |
| 加载更多 | cursor 分页；limit 默认 25、最大 100 | 不解析 cursor，不要求 total count，不显示“共 N 条” |

#### 3.1.1 GET /api/v1/cases query 契约

| 参数 | 语义 | 客户端责任 |
| --- | --- | --- |
| q | 受控摘要搜索 | 输入变化后清除旧 cursor，从第一页重新请求 |
| stage | Case stage 过滤 | 只发送 approved 枚举；“全部”时省略 |
| workflow_status | workflow status 过滤 | 只发送 approved 值；不映射为 stage |
| risk_code | 风险 code 过滤 | 使用服务端提供或设计冻结的受控选项 |
| advisor_user_id | Advisor 过滤 | 只发送当前用户可选值；不能作为越权读取条件 |
| limit | 单次返回上限 | 省略即 25；客户端不得请求超过 100 |
| cursor | 服务端 opaque token | 原样回传；不得解析、拼接、持久化为业务标识或跨筛选复用 |

服务端默认稳定排序固定为 next_step.due_at ASC NULLS LAST、updated_at DESC、id ASC。cursor 与生成它时的 q、stage、workflow_status、risk_code、advisor_user_id 和固定排序绑定；任一筛选改变时，客户端丢弃当前 cursor，从首批结果重新加载。只有响应提供后续 cursor 时才显示“加载更多”；没有 cursor 即表示当前筛选已加载完毕，但不代表客户端知道 total count。

### 3.2 桌面线框：Cases 列表

~~~text
+--------------------------------------------------------------------------------+
| [≡] 案件                     [搜索授权案件] [通知] [账号]                    |
+----------------------+---------------------------------------------------------+
| Workspace            | 案件                                                   |
|   今日工作           | K12 ServiceCase；只显示当前授权范围                   |
|   案件               |                                                         |
|   学生与监护人       | [新建案件]                                             |
|   任务               |                                                         |
|   文件               | 筛选：[阶段 v] [流程 v] [风险 v] [Advisor v]           |
|   学校资料           | 搜索：[受控摘要________________] [清除]                |
|                      | 排序：下一步到期升序（空值最后）/ 更新降序 / ID 升序   |
|                      |                                                         |
|                      | +---------------------------------------------------+   |
|                      | | Case 摘要 | 阶段 | 下一步/风险 | 负责人 | 更新 | > |   |
|                      | | C-opaque   | 背景 | 补齐资料    | Advisor | 时间 | > |   |
|                      | | C-opaque   | 选校 | 待 Founder  | Founder | 时间 | > |   |
|                      | +---------------------------------------------------+   |
|                      | 当前已加载 25 条                         [加载更多]     |
|                      | 空状态：没有符合筛选的授权案件；[清除筛选]             |
+----------------------+---------------------------------------------------------+
~~~

### 3.3 移动线框：Cases 列表

~~~text
+--------------------------------------+
| [☰] 案件                 [新建]     |
+--------------------------------------+
| 搜索授权案件                         |
| [________________________] [筛选]   |
| [阶段 v] [流程 v] [风险 v] [Advisor v]|
+--------------------------------------+
| 当前已加载 2 条                      |
|                                      |
| Case 受控摘要                         |
| 阶段：背景收集                       |
| 下一步：补齐资料 · due_at            |
| 负责人：Advisor             [打开 >] |
|                                      |
| Case 受控摘要                         |
| 阶段：选校确认                       |
| 风险：待 Founder 处理       [打开 >] |
+--------------------------------------+
| [加载更多]                           |
+--------------------------------------+
~~~

### 3.4 Cases 进入、返回和失败

- 点击 Case 行直接进入 canonical /cases/:caseId；共享 layout 和 summary 聚合 API 分别按契约加载，服务端再次授权。
- /cases/new 成功后进入新 Case 的 canonical /cases/:caseId；失败保留已填输入和安全 error code。
- 旧 /cases/:caseId/workspace 只执行 308 到 /cases/:caseId，不渲染过渡页、不保留 query tab。
- 返回 /cases 使用明确返回链接；若保留筛选上下文，只作为 UI 查询上下文，不成为授权参数。
- 403/denied 不区分 Case 不存在和无权；服务端允许区分时才显示 not_found。
- 列表 stale 只显示 freshness 提示；点入详情仍以 Case 权威查询为准。
- 空列表不等于 denied；依赖不可用不显示“已加载 0 条”。
- 加载更多时原样回传服务端 cursor 和同一组筛选；筛选变化立即废弃旧 cursor。客户端可防止重复点击，但不解析 token 或以本地去重掩盖服务端分页错误。

## 4. 案件工作区

### 4.1 标题区与阶段时间线

canonical /cases/:caseId 及其独立子路由共享同一 Case layout。标题区必须回答：当前 Case 是谁的、处于哪个阶段、谁负责、下一步是什么、当前用户能做什么。

~~~text
+--------------------------------------------------------------------------------+
| 案件 > Case 受控摘要                                                        |
| Student 受控显示名 · Case reference（内部）                                  |
| Stage: background_collection   Workflow: active   Primary: Advisor           |
| 下一步：完成 Assessment blocker       更新：时间       版本：record_version  |
| [返回案件] [打开 Student]                         [允许的主要操作 v]          |
+--------------------------------------------------------------------------------+
| signed -------- background_collection --- school_selection_confirmed          |
|    ✓                         ●                         ○                       |
|                         application_in_progress -------- closed               |
|                                  ○                         ○                  |
| 说明：阶段由服务端事实决定；暂停/终止/风险是 workflow/摘要，不改写里程碑。   |
+--------------------------------------------------------------------------------+
| [摘要] [Assessment] [候选学校] [逐校申请] [面试] [文件] [Portal 授权] [结案] |
+--------------------------------------------------------------------------------+
~~~

移动端把标题、阶段当前节点、下一步和主要 allowed action 放在顶部；时间线横向可滚动但不隐藏当前节点。

### 4.2 桌面线框：案件工作区总览

~~~text
+--------------------------------------------------------------------------------+
| [返回案件] Case 受控摘要                                  [允许操作 v]       |
| 阶段 timeline / workflow status / Primary Advisor / record_version             |
+----------------------+---------------------------------------------------------+
| Case 工作区          | 摘要                                                   |
|   摘要               | [下一步] [风险] [负责人] [更新时间]                    |
|   Assessment         |                                                         |
|   候选学校           | Assessment                                             |
|   逐校申请           | blocker: 2 / 当前授权完成度摘要             [打开]     |
|   面试               |                                                         |
|   文件               | 候选学校名单                                           |
|   Portal 授权        | v3 · Founder 审批：待处理 · Guardian 确认：未有记录      |
|   结案               | [Primary Advisor 建名单] [Founder 审批] [代录确认]      |
|                      |                                                         |
|                      | 逐校申请                                               |
|                      | School opaque/display · 状态 · Assignee · Task · 风险   |
|                      | [打开逐校申请]                                          |
|                      |                                                         |
|                      | Task / 文件 / 通知摘要                                 |
|                      | [查看 Task] [查看文件] [查看通知]                      |
|                      |                                                         |
|                      | Portal 授权 / 审计入口                                 |
|                      | [管理 Portal Grant] [查看允许的历史/审计摘要]           |
+----------------------+---------------------------------------------------------+
~~~

### 4.3 移动线框：案件工作区

~~~text
+--------------------------------------+
| [返回] Case 摘要           [操作 v] |
| 阶段：背景收集                       |
| 下一步：完成 blocker     版本：12   |
+--------------------------------------+
| [摘要] [Assessment] [学校] [申请]   |
| [面试] [文件] [Portal] [结案]       |
| <导航链接横向滚动；每项是独立路由>   |
+--------------------------------------+
| 当前区块标题                         |
| 状态/摘要/允许操作                   |
|                                      |
| [主要操作]                           |
|                                      |
| 关联区块：下一步 / 风险 / 通知       |
+--------------------------------------+
| [返回案件列表]                       |
+--------------------------------------+
~~~

### 4.4 工作区区块范围

| 区块 | 读取 | 操作入口 | 不允许 |
| --- | --- | --- | --- |
| 摘要 | Case stage、workflow status、Primary Advisor、next step、risk、record version | 打开下一步、暂停/恢复/终止/结案（按 allowed actions） | 本地推进阶段 |
| Assessment | 当前用户允许的 blocker/完成摘要 | Primary Advisor 或获 scope 的 Collaborator 编辑；背景完成命令 | Founder 写入；Admin 默认查看 |
| 候选名单 | 版本、学校集合、Founder 审批、Guardian 决定摘要 | Primary Advisor 建版本；Founder approve/reject；Primary Advisor 代录同版确认 | Portal 写入；跳过 Founder/Guardian 顺序 |
| 逐校申请 | SchoolTarget 状态、Assignee、Task、凭证摘要 | 打开 Task/文件；按 allowed actions 处理 | 以 Case 阶段替代 Target 状态 |
| 面试 Task | 目标学校、时间/方式/语言、必要摘要、Assignment | 内部 Task 处理或 Contractor 脱敏入口 | Contractor 查看完整 Assessment/文件/联系方式 |
| 文件 | metadata、版本、扫描状态、用途和 allowed actions | 上传/收据/扫描后预览下载；Application Assignee 选择提交凭证 | 扫描前下载；泄露 object key/URL |
| 通知 | 当前 User 最小待办摘要 | 打开通知列表，resolve-target 后再授权 | 把通知当长期权限或 Task |
| Portal 授权 | Grant 状态、viewer/期限摘要 | Primary Advisor 创建/撤销/重新生成；Founder 紧急撤销 | Guardian 自己创建；延期原 Grant |
| 人工结案 | BR-039 结案前置、允许结果/原因和缺口摘要 | Founder 在 /cases/:caseId/close 明确提交 | 自动结案；条件不足时提交 |
| 历史/审计入口 | Case history、审批、确认、Task 和受控审计摘要 | Founder 进入审计查询；其他角色仅按范围看业务历史 | Operations projection 代替 Audit 或授权 |

通知和历史/审计是 summary 内的受控入口，不新增 Case 子路由。Task 详情使用 /tasks/:taskId；从 applications 或 interviews 返回时保持来源 Case 子路由，但重新授权。

## 5. 核心流程约束在工作区中的表现

### 5.1 Assessment 到候选名单

- Assessment 从 draft 开始；背景 blocker 完成后显示 background_complete，允许 Primary Advisor 建立候选名单。
- Candidate list 建立按钮位于“候选名单”区块右上或区块底部；前置条件是当前 Primary Advisor、Assessment ready、学校引用和版本可用。
- Founder 审批/驳回显示在同一版本行内；驳回必须回到修改，不创建隐式新版本。
- Founder 批准后，Guardian 确认按钮只由 Primary Advisor 看到；记录必须绑定同一 approved version、学校集合、确认渠道、代录人、时间和结果。
- Guardian 确认未完成时，Case 不显示 school_selection_confirmed；Portal 不承担确认写入。

### 5.2 所有学校拒绝后的两条分支

工作区显示一个“结案选择”提示，但不自动结案、不自动创建 Task：

~~~text
所有已确认学校均为 rejected
  |
  +--> 新增候选学校
  |      新名单版本 -> Founder 批准 -> Guardian 确认同一版本
  |
  +--> Founder 人工结案：无 offer
         记录结果、原因、操作者、时间；保留所有 Target 历史
~~~

若存在 accepted、submitted、interview、waitlisted、preparing 或未完成 Task，结案按钮不可用并显示前置条件。新增学校不覆盖原结果；移除进行中的学校使用 withdrawn 并取消未完成 Task、保留历史。

### 5.3 Preparing、申请和面试 Task

- SchoolTarget 进入 preparing 时，页面显示 Application Assignee 和“准备并提交申请” Task；同一学校同一轮次只显示一条自动 Task。
- Application Task 完成表单要求提交时间、渠道、提交人、材料清单完成状态，以及学校参考号或至少一份 clean/available/未 revoked 的替代凭证。
- 学校明确要求面试时，显示 interview_support Task 和 Assignee 入口；可指派 Primary Advisor、授权 Advisor Collaborator 或单一 Contractor。
- Interview Support Assignee 只看到目标学校、面试时间/方式/语言、辅导要求、due_at 和必要背景摘要；完成只表示辅助完成，不改变 SchoolTarget 结果。
- Task 完成发布事实，由 Cases 重新校验后才推进 SchoolTarget；页面不能直接写 Case 或 Target。

## 6. 主要操作交互表

| 操作 | 按钮位置 | 前置条件 | 成功反馈 | 失败/冲突反馈 |
| --- | --- | --- | --- | --- |
| 打开 Case | Today/Cases 行右侧；Case 标题 | 当前 Session 和资源授权有效 | 进入 workspace summary | denied/not_found/unavailable；不泄露存在性 |
| 保存 Assessment | Assessment 区底部 | Primary Advisor 或明确 education_profile edit；当前字段/版本有效 | 显示已保存、最新版本和 blocker 更新 | validation 保留输入；STALE_VERSION 刷新并重新确认；unavailable 不显示成功 |
| 标记 background_complete | Assessment 区“完成背景资料” | blocker 全部满足、当前 Primary Advisor、expected version | 显示 background_complete 和“可建立候选名单” | blocker 未满足列出受控 code；冲突要求刷新；审计失败则失败关闭 |
| 建立候选名单 | 候选名单区右上/空状态 CTA | Assessment ready、Primary Advisor、学校引用有效 | 显示新版本和待 Founder 审批 | denied/validation/stale/idempotency；不创建重复名单 |
| Founder approve/reject | 候选版本行内 | 当前 Founder、非自审、精确 version | 显示 approved 或退回修改；通知 Primary Advisor | denied、expected version 冲突、审计失败；要求重新加载 |
| 代录 Guardian 确认 | approved 版本行内 | 当前 Primary Advisor、同版 approved、渠道/结果/代录人字段齐全 | 显示 Guardian decision record；允许进入下一步检查 | 版本不一致、关系失效、重复 key 返回原结果或冲突 |
| 选择/新增学校 | 候选名单或结案选择区 | 新名单版本流程；active 学校引用；不覆盖终态 Target | 显示待 Founder 审批的版本 | 不允许绕过审批；基线冲突/学校不可用安全失败 |
| 进入 preparing | SchoolTarget 行内 allowed action | Guardian 已确认同版名单、当前授权 | 显示 preparing，并显示幂等创建的申请 Task | 版本冲突、Task 创建不可用；不显示假成功 |
| 接受/拒绝 Task | Task 卡/Task 详情主操作区 | 当前 Assignment、Task assigned | accepted 或 awaiting_reassignment | 拒绝必须填写原因；denied/stale/unavailable 安全提示 |
| 重派/取消 Task | Task 详情二级操作 | Primary Advisor 或 Cases 受信动作；当前 Task 未终态 | 新 Assignment 或 cancelled；保留历史 | allowed_actions 不含时隐藏/禁用；冲突要求刷新 |
| 完成申请 Task | Task 详情完成表单 | accepted；提交事实和凭证校验通过 | Task completed；等待 Cases 重新校验 Target | 缺参考号和替代凭证、文件不可用、STALE_VERSION；不推进 Target |
| 完成面试辅助 | Interview Task 完成表单 | accepted；仅保存辅助完成摘要 | Task completed；SchoolTarget 不变 | denied/stale/unavailable；不接收面试结果字段 |
| 暂停/恢复/终止 | 标题区“允许操作”菜单 | 暂停无 submitted 或后续 Target；恢复/终止符合 Cases 前置 | 显示新 workflow status 和审计完成 | 不满足前置、冲突、审计失败；不能顺延 due_at |
| Founder 人工结案 | 结案选择区或标题区 | 所有 Target 终态、无未完成 Case Task、Founder | 显示 closed、结果/原因/时间；通知结案选择完成 | 条件不足列出缺口；STALE_VERSION 要求重新确认；不自动结案 |
| Portal Grant | Portal 授权区 | 当前 Primary Advisor 创建；Founder 可紧急撤销；Case/Guardian 关系有效 | 显示 7 天 Grant 状态；raw key 只显示一次 | invalid/unavailable/撤销；不能延期原 Grant |
| 通知点击 | TopBar/通知区 | 当前 recipient；resolve-target 再授权 | 跳转固定内部 route | target 失效统一 unavailable；通知不扩大权限 |

所有写操作都带 Idempotency-Key 和 expected_record_version；成功必须同时满足业务事实、AuditEvent 和必要 Outbox 的提交约束。

## 7. 页面状态与反馈基线

| 页面/区块 | loading | empty | error | denied | unavailable | stale/version conflict | success |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Today | skeleton；不猜计数 | “暂无待办/风险/到期任务” | 安全 code + 重试 | Contractor 固定 denied +“返回我的任务”；其他角色按权限拒绝 | projection/Notifications unavailable，不伪造 | 标 freshness；点击仍读权威模块 | 显示数据时间、来源和固定下一步 |
| Cases 列表 | 表头/行 skeleton | 无授权案件或无匹配筛选 | 保留筛选；重试 | 不显示其他 Case | API/runtime 不可用 | query stale 只提示；详情重新授权 | 显示当前授权摘要和分页/cursor 控件 |
| 旧 workspace URL | 不渲染 skeleton | 不适用 | 重定向基础设施失败使用安全 error | 不在旧 URL 判断 Case 权限 | 路由层不可用 | 不适用 | 308 到 canonical /cases/:caseId |
| Case 标题/时间线 | 标题和时间线骨架 | 允许的摘要为空时说明 | 区块级安全错误 | 统一 denied | Case runtime unavailable | record_version 冲突后重新加载 | 阶段、workflow、负责人、下一步和更新时间 |
| Assessment/名单/Target | 当前 tab skeleton | 未开始/无版本/无学校 | 保留输入或回到只读 | 按 Scope 拒绝，不扩大 Case | owning module unavailable | 写操作要求重新确认 | 服务端结果、版本、审计提示 |
| Task 区块 | Task skeleton | 无当前授权 Task | 安全 error code | Contractor/无 Assignment 拒绝 | Tasks unavailable | allowed action 重新获取 | accepted/rejected/completed 等服务端结果 |
| Documents 区块 | metadata skeleton | 当前 scope 无文件 | scan_failed/rejected 安全提示 | 资源级拒绝 | scanner/object unavailable | Version 冲突刷新 | clean+available 才显示预览/下载 |
| Portal/通知/审计入口 | 按目标模块安全加载 | 无通知/无 Grant/无可见历史 | 固定 error code | 统一拒绝 | unavailable，不伪造写入 | target/Grant/version 重新校验 | 跳固定路由或显示受控结果 |

## 8. API、DTO、权限追溯表

| UI 面 | Approved API/查询 | 受控 DTO/状态 | 权限边界 | BR/设计依据 |
| --- | --- | --- | --- | --- |
| Today 我的待办/风险 | GET /api/v1/cases；GET /api/v1/tasks；Operations 公开查询；GET /api/v1/notifications/unread-count | Case summary、risk/next_step、Task summary、unread count、freshness | 当前 Session User；Case/Task 关系；Operations 不是授权来源 | BR-001、BR-038、BR-070；approved Cases/Task/Notifications |
| Cases 列表 | GET /api/v1/cases；q、stage、workflow_status、risk_code、advisor_user_id、limit、cursor | id/受控摘要、stage、workflow、risk、record version、opaque cursor；默认 limit 25，最大 100；固定稳定排序 | 当前授权 Case；不能传 organization/actor；cursor 绑定筛选与排序且客户端不解析 | BR-030、BR-070；approved Cases；Architect F1 决策 |
| Case workspace | canonical 页面 /cases/:caseId；GET /api/v1/cases/:caseId/workspace | case、next_step、assessment summary、candidate list、school target、Task、Document summary | 页面 URL 与 API 路径分离；server-side coordinator 组合公开查询；owner 最终重验 | BR-030–BR-034、BR-039；approved Cases；Architect F1 决策 |
| Assessment | GET /api/v1/cases/:caseId/assessment；背景完成命令 | allowed blocker/manifest summary、record version、safe error | Primary Advisor/明确 education_profile Scope；Founder 只读；Admin 默认拒绝 | BR-032；approved Cases |
| Candidate list | GET/POST /api/v1/cases/:caseId/candidate-lists；review；guardian-decision | version/status/approval/guardian decision summary | Advisor 建立；Founder approve/reject；Primary Advisor 代录同版确认 | BR-033；approved Cases |
| SchoolTarget | GET /api/v1/cases/:caseId/school-targets | target status、school reference、Assignee、受控证据摘要 | Case/SchoolTarget 关系；不由 UI 改 Case 阶段 | BR-034；approved Cases |
| Task 列表/详情 | GET /api/v1/tasks、GET /api/v1/tasks/:taskId、GET /api/v1/tasks/assigned | task、assignment、due_at、is_overdue、allowed_actions、completion summary | 当前 Assignee/Primary Advisor/Case Collaborator；Contractor 仅脱敏接口 | BR-035–BR-037；approved Task |
| Task 写操作 | POST accept/reject/complete/reassign/cancel | 服务端结果、Audit/Outbox 事实 | Idempotency-Key、expected version、TaskAssignment 重验 | BR-035–BR-037、BR-070；approved Task |
| Contractor | GET /api/v1/contractor/tasks/:taskId/workspace | task、school、interview、brief、assignment | active Contractor + 当前 interview_support Assignment；不进完整 Case | BR-036；approved Task/authorization |
| 通知入口 | GET notifications/unread-count；POST read/resolve-target | 最小 content_code、read 状态、opaque target | 当前 recipient；点击再次授权 | BR-038；approved Notifications |
| 生命周期 | POST /api/v1/cases/:caseId/lifecycle | workflow result、record version、reason code | Primary Advisor/Founder 按 pause/resume/terminate/close 条件 | BR-031、BR-039；approved Cases |
| 文件摘要 | Case workspace Documents 查询；具体上传/下载命令由 approved Documents API | metadata、scan/status、allowed actions；不返回 key/URL/token | Founder/Primary Advisor/Application Assignee 资源级权限 | BR-050；approved Documents |
| 审计/运营入口 | Audit/Operations 公开查询契约 | 脱敏 AuditEvent、projection freshness、alert code | Founder 租户审计；Admin technical health；不能授权/改业务 | BR-070、BR-071；Audit/Operations contract |

## 9. 桌面和移动端交互原则

- 桌面保留左 Sidebar、Case 上下文标题、主内容和 allowed action 区；tab 是共享 Case layout 内的独立子路由导航链接，不变成一级导航。
- 移动端只保留一个导航按钮；Case 标题、当前阶段、下一步和主要操作位于首屏；子路由导航横向滚动，当前项有明确文本和页面语义。
- 表格在移动端转为分组行，保持 Case 摘要、阶段、下一步、负责人和进入按钮；不要求横向滚动完成主动作。
- 所有按钮必须有文字或可读标签；disabled 必须说明原因；颜色不能独自承载状态。
- 异步写入成功后保留当前上下文，显示下一步链接；失败保留输入，不用 toast 掩盖安全或版本错误。
- Contractor 视图密度和字段比内部 Case workspace 更低；Portal 不使用内部导航。

## 10. Architect 决策记录与可实现契约

Architect 于 2026-08-26 解决 F1 的四个阻塞项。本节是路由与查询实现契约，不新增业务实体、角色、状态或业务规则。

| 决策 | 可实现契约 |
| --- | --- |
| Contractor 默认入口 | 登录后进入 /tasks。显式访问 /today 返回统一 denied 页面，主内容只提供“返回我的任务”到 /tasks；不重定向到单个 Task，不查询或暴露 Case。 |
| Cases cursor query | GET /api/v1/cases 接受 q、stage、workflow_status、risk_code、advisor_user_id、limit、cursor。limit 默认 25、最大 100；cursor 是服务端 opaque token，绑定筛选与固定排序，客户端原样回传且不解析；不要求 total count。 |
| Cases 稳定排序 | next_step.due_at ASC NULLS LAST、updated_at DESC、id ASC。客户端不做本地重排，不提供未批准的 sort 参数。 |
| 页面 canonical | /cases/:caseId 是 Case workspace canonical；旧 /cases/:caseId/workspace 以 308 重定向。聚合 API 保持 GET /api/v1/cases/:caseId/workspace。 |
| 共享 layout 子路由 | summary=/cases/:caseId；另有 assessment、schools、applications、interviews、documents、access、close。tab 使用导航链接，不使用 query tab；保持 deep link、返回路径和逐路由重新授权。 |

当前没有遗留 F1 设计阻塞问题；任何业务语义变化仍须回到 confirmed BR 和 owning approved API/UI 文档。

## 11. 交付状态与验证边界

~~~text
status: approved
owner: frontend-design
scope: F1 low-fidelity Today, Cases list, Case workspace wireframes and interaction states
changed: system-design/50-api-ui/frontend-design/20-f1-today-cases-wireframes.zh-CN.md; system-design/50-api-ui/frontend-design/10-information-architecture.zh-CN.md
evidence: confirmed BR and approved API/UI traceability; six Markdown/ASCII wireframes; cursor/canonical/subroute contracts; local Markdown link and consistency checks
not_run: browser, database, cloud, deployment, migration, product source changes, product tests
risks_or_stop_conditions: no remaining F1 design blocker recorded; Architect re-review is required before approved status or implementation
~~~
