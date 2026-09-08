# F2 CRM 与建案向导低保真线框及交互契约

> 状态：`approved`
> Owner：Frontend Design
> 范围：Release 1 / F2（Student、Guardian、ReferralSource、Case Intake）
> 日期：2026-08-26
> 非范围：产品源码、API 实现、测试、数据库、migration、云、F3

## 0. 一页总览

### 0.1 本轮结果

本文件定义以下目标页面的低保真结构、进入与返回路径、权限边界、主要操作反馈和异常状态：

- `/students`：Student 列表、搜索和状态识别。
- `/students/new`：原子创建 Student 与当前唯一 Primary Guardian；或由员工明确选择已有 Guardian。
- `/students/:studentId`：Student 资料、由 DOB 即时计算的年龄、Guardian 与 Case 摘要、编辑和删除申请入口。
- `/students/:studentId/guardians`：Guardian 新建/关联、关系维护、主要联系人交接、解除关系和历史。
- `/students/deletion-requests`：Founder 审批删除申请。
- `/referral-sources`、`/referral-sources/:sourceId`：来源查看和 Founder 管理。
- `/cases/new`：选择或先建 Student、指定 Primary Advisor、可选 ReferralSource、复核并创建 Case。

本轮只定义前端目标体验。Architect 已冻结本文件所需的 endpoint、query、DTO、错误码和 capability；旧实现只作为差异证据，不反推目标契约。

### 0.2 目标连续性

~~~text
/students
  |-- 新建 Student --------------------------> /students/new
  |                                             |-- 新 Guardian + 主关系（同一 CRM 原子操作）
  |                                             `-- 明确搜索并选择已有 Guardian
  |-- 打开 Student --------------------------> /students/:studentId
  |                                             |-- Guardian 管理 -> /guardians
  |                                             |-- 删除申请
  |                                             `-- 新建 Case -> /cases/new?studentId=...
  `-- Founder 删除审批 ----------------------> /students/deletion-requests

/referral-sources ---------------------------> /referral-sources/:sourceId

/cases/new
  |-- 选择已有 active Student
  `-- 先完成 Student + Primary Guardian 建档 -> 回到向导并锁定该 Student
       -> 指定 Primary Advisor
       -> 可选 active ReferralSource
       -> Review / Create
       -> Case 自动进入 background_collection
       -> /cases/:caseId/assessment
~~~

CRM 建档与 Cases 建案是两个模块边界清晰的提交：Student 建档成功后，即使后续 Case 创建失败，Student 仍保留；向导提供“重试创建 Case”和“查看 Student”，不伪装成跨模块全局事务。

### 0.3 明确排除

- 不提供 DuplicateCandidate 持久化、重复工作台、merge、undo、自动关联或自动选择 Guardian。
- 不因 DOB 或 gender 触发重复提示。
- 不把 `father/mother/other_guardian` 当作完整目标枚举。
- 不允许 Admin 单独进入客户资料或管理 ReferralSource。
- 不显示或检索 `deleted` Student，不提供恢复、purge 或物理删除。
- 不把年龄作为字段录入或保存。
- 不引入 Platform Billing、Data Reviewer、Lead/Quote/Contract、AI、非 K12、多组织或其他 Future 页面。
- 不制作高保真视觉、颜色方案或营销落地页。

## 1. 事实来源与约束

### 1.1 规范性来源

- [本地执行约束](/Users/karo/Documents/Tianxingguoji/AGENT.md)
- [BR-Identity / Access](../../../business-requirements/10-identity-access.zh-CN.md)
- [BR-CRM](../../../business-requirements/20-crm.zh-CN.md)
- [BR-Cases](../../../business-requirements/30-cases.zh-CN.md)
- [CRM 模块契约](../../10-module-contracts/30-crm.zh-CN.md)
- [Cases 模块契约](../../10-module-contracts/50-cases.zh-CN.md)
- [CRM 数据模型](../../20-domain-data-models/30-crm.zh-CN.md)
- [Cases 数据模型](../../20-domain-data-models/50-cases.zh-CN.md)
- [授权模型](../../40-permissions-security/10-authorization-model.zh-CN.md)
- [F0 信息架构](./10-information-architecture.zh-CN.md)
- [F1 Today / Cases 线框](./20-f1-today-cases-wireframes.zh-CN.md)
- [Cases Workspace API/UI](../10-cases-workspace.zh-CN.md)

### 1.2 证据层级

| 层级 | 本文件用法 |
|---|---|
| confirmed BR | 唯一业务规则事实源 |
| approved 系统设计 | 模块边界、权限、目标路由和已冻结交互契约 |
| `docs/implementation/CRM-*`、`P1-05`、当前 `app/api/v1` 与 UI | 仅用于“现状差异”，不得覆盖目标 |
| Architect 冻结的 F2 transport contract | 本文件的可实现 API/UI 基线；仍须按 approved BR、模块 public port 和服务端授权实现 |

## 2. 路由与权限边界

### 2.1 目标路由矩阵

| 路由 | Founder | Advisor | Admin 单独 | Contractor | 主要返回路径 |
|---|---|---|---|---|---|
| `/students` | 可查看授权范围、搜索、新建 | 可查看授权范围、搜索、新建 | denied | denied | ERP 上一级入口 |
| `/students/new` | 可建档 | 可建档 | denied | denied | 返回 Student 列表或调用方 |
| `/students/:studentId` | 授权范围内查看/编辑/申请删除 | 业务关系范围内查看/编辑/申请删除 | denied | denied | 保留筛选返回 `/students` |
| `/students/:studentId/guardians` | 授权范围内维护 | 业务关系范围内维护 | denied | denied | 返回 Student 详情 |
| `/students/deletion-requests` | 查看并审批 | 不显示审批队列 | denied | denied | 返回 Student 列表 |
| `/referral-sources` | 查看全部并创建 | 只读 active | denied | denied | ERP 上一级入口 |
| `/referral-sources/:sourceId` | 查看/修改/停用 | 仅可读 active；inactive 直达 denied/unavailable | denied | denied | 保留筛选返回来源列表 |
| `/cases/new` | Founder 单独 denied；同时具备 Advisor 角色/capability 时按 Advisor 可创建 | 可创建 | denied | denied | 返回 `/cases` 或原 Student 详情 |

说明：Founder + Admin 组合身份按 capability 并集合并；Founder 的 CRM capability 保留，Admin 身份不会减去它。Admin 单独和 Contractor 不获得任何本轮客户资料入口。所有页面与每次写操作均由服务端重新授权，隐藏导航不是授权机制。

建案权限按已批准 F0 角色矩阵落地：`建案` 是 Advisor 的主要操作，不是 Founder 基础角色的操作。Founder 单独、Founder + Admin 均不能进入或提交 `/cases/new`；Founder 同时拥有有效 Advisor 角色/capability 时，按多角色并集以 Advisor 身份建案。该边界不影响 Founder 查看组织授权范围内 Case、审批名单或人工结案。

### 2.2 入口和拒绝边界

| 来源 | 入口 | 行为 |
|---|---|---|
| 主导航“客户” | `/students` | Founder/Advisor 可见；Admin 单独、Contractor 不显示 |
| Student 详情“新建案件” | `/cases/new?studentId=:studentId` | 只预选 active Student；服务端仍重验状态与权限 |
| Cases 列表“新建案件” | `/cases/new` | 进入完整向导 |
| 主导航“来源” | `/referral-sources` | Founder/Advisor 可见；Advisor 为只读 active 视图 |
| 删除申请入口 | `/students/deletion-requests` | 仅 Founder 显示 |
| 无权限直达 | 任一上述 URL | 统一 denied 页面；不得泄露对象是否存在，提供安全返回链接 |

## 3. 目标字段和受控值

### 3.1 Student 与 Guardian

| 对象 | 字段 | 前端规则 |
|---|---|---|
| Student | `display_name` | 必填；参与疑似重复比较 |
| Student | `date_of_birth` | 可空；只保存日期；年龄按当前日期即时计算 |
| Student | `gender` | 可空：`male/female/other/not_disclosed` |
| Student | `contact_email`、`contact_phone` | 可空；非空时参与疑似重复比较 |
| Student | `status` | 只显示 `active`、`pending_delete`；`deleted` 在业务 UI/API 完全不可见 |
| Guardian | `display_name` | 必填；参与疑似重复比较 |
| Guardian | `date_of_birth` | 可空；年龄只派生、不保存 |
| Guardian | `gender` | 可空：`male/female/other/not_disclosed` |
| Guardian | `email`、`phone` | 至少一个非空；非空时参与疑似重复比较 |

Gender 与 relationship_type 相互独立，不从称谓、关系或姓名推断。

### 3.2 Guardian relationship_type

前端选项必须完整来自 BR-022，不折叠成旧实现的三个值：

~~~text
parent                 father                 mother
step_parent            stepfather             stepmother
adoptive_parent        adoptive_father        adoptive_mother
foster_parent          foster_father          foster_mother
grandparent            paternal_grandfather   paternal_grandmother
maternal_grandfather   maternal_grandmother   adult_sibling
adult_brother          adult_sister           uncle
aunt                   court_appointed_guardian
institutional_guardian other_relative         non_relative_guardian
other
~~~

选择 `other` 时，relationship description 必填。主要联系人只是沟通优先级；与法定代理、紧急联系人、账单联系人、通知接收等标记彼此独立。每名 Student 在任一时点恰好一个 current Primary Guardian。

### 3.3 ReferralSource

| 字段 | 前端规则 |
|---|---|
| `display_name` | 必填 |
| `source_type` | `customer_referral`、`employee_referral`、`school_referral`、`partner_referral`、`website`、`social_media`、`paid_advertising`、`event`、`walk_in`、`other`、`unknown` |
| `description` | 可空；`source_type=other` 时必填 |
| `status` | `active` / `inactive`；新建或变更 Case 只能选择 active |

### 3.4 Case Intake

| 字段/选择 | 前端规则 |
|---|---|
| Student | 必须是 existing active Student；可在向导内先完成 CRM 建档再返回选择 |
| Primary Advisor | 必须选择有效的 active Advisor role binding |
| ReferralSource | 可空；只列 active；当前 Primary Advisor 仅可为自己的 Case 选择/变更 |
| `intake_year` | 必填目标入学年份；字段错误由冻结的 `VALIDATION_FAILED` envelope 返回 |
| `admission_type` | `entry` / `transfer` |
| `signed_at` | 必填；记录已完成线下签约的实际时间 |
| K12 Assessment manifest | 员工不可选择；服务端绑定创建时当前 approved K12 manifest，Review 只读显示版本摘要 |
| 创建后状态 | 服务端自动进入 `background_collection`；成功跳转 canonical assessment 子路由 |

## 4. Student 列表

### 4.1 桌面线框 W01

~~~text
+----------------------------------------------------------------------------------+
| 客户 / Student                                              [新建 Student]      |
|----------------------------------------------------------------------------------|
| [搜索姓名、Email 或电话________________] [状态: Active v] [清除]                |
| Founder only: [删除申请 (待处理徽标)]                                            |
|----------------------------------------------------------------------------------|
| Student                  联系方式              状态             最近更新          |
| Lin Chen                 l***@... / ***1234    Active           2026-08-26       >|
| Mei Wong                 m***@...              Pending delete   2026-08-25       >|
|----------------------------------------------------------------------------------|
| 已加载 25 条                                      [加载更多]                     |
+----------------------------------------------------------------------------------+
~~~

- 顶部右侧主按钮进入 `/students/new`；Founder 的审批入口与新建按钮分离。
- 搜索只使用目标查询契约允许的字段；空搜索显示当前授权范围，不暗示全组织权限。
- `active` 与 `pending_delete` 均可见且使用明确文字；没有 `deleted` 筛选项、计数或结果。
- 行点击进入 `/students/:studentId`，并携带列表搜索、状态和游标位置作为返回上下文。
- “加载更多”使用冻结的 opaque cursor；筛选变化时清除旧 cursor 并从首批加载，不显示 total count。

### 4.2 移动线框 W02

~~~text
+--------------------------------------+
| < ERP              Student       [+] |
| [搜索姓名、Email 或电话_________]     |
| [Active v]                 [清除]     |
|--------------------------------------|
| Lin Chen                         >    |
| l***@... · ***1234                   |
| Active · 更新 08-26                  |
|--------------------------------------|
| Mei Wong                        >     |
| m***@...                            |
| Pending delete · 更新 08-25          |
|--------------------------------------|
|             [加载更多]               |
+--------------------------------------+
~~~

移动端使用单列可扫描条目，不横向压缩桌面表格。搜索和状态筛选固定占用稳定高度；结果加载不会令按钮跳位。Founder 审批入口放在页面操作菜单中，图标配 tooltip/accessible label。

## 5. 新建 Student 与 Primary Guardian

### 5.1 桌面线框 W03

~~~text
+----------------------------------------------------------------------------------+
| < 返回 Student                    新建 Student                                    |
|----------------------------------------------------------------------------------|
| Student                                                                         |
| 姓名 * [____________________]  DOB [yyyy-mm-dd]  Gender [未提供 v]               |
| Email  [____________________]  电话 [________________________]                    |
|                                                                                  |
| Primary Guardian                                                                |
| [新建 Guardian] [选择已有 Guardian]                                              |
|                                                                                  |
| 新建模式：                                                                       |
| 姓名 * [____________________]  DOB [yyyy-mm-dd]  Gender [未提供 v]               |
| Email  [____________________]  电话 [________________________]                    |
| 至少填写 Email 或电话                                                            |
|                                                                                  |
| Relationship                                                                    |
| 类型 * [请选择 v]  说明 [____________________] (选择 other 时必填)               |
| [ ] Legal  [x] Primary  [ ] Emergency  [ ] Billing  [ ] Notification            |
|----------------------------------------------------------------------------------|
| [取消]                                             [创建 Student 与主联系人]      |
+----------------------------------------------------------------------------------+
~~~

- 页面只允许一个 Primary Guardian 配置，并固定 `Primary=true`；其余关系标记不由 Primary 自动推断。
- 切换“选择已有 Guardian”后，Guardian 表单变为显式搜索、结果列表和当前选择摘要；不会预选第一项。
- 提交为一个 CRM 原子操作：Student、Guardian（新建模式）和 current primary relationship 全部成功或全部失败。
- “取消”回到调用方；无调用方时返回 `/students`。离开有未保存内容时使用普通未保存更改确认。

### 5.2 移动线框 W04

~~~text
+--------------------------------------+
| < Student        新建 Student        |
|--------------------------------------|
| 1 Student                            |
| 姓名 * [________________________]     |
| DOB    [yyyy-mm-dd]                   |
| Gender [未提供 v]                    |
| Email  [________________________]     |
| 电话   [________________________]     |
|--------------------------------------|
| 2 Primary Guardian                   |
| [新建] [选择已有]                    |
| 姓名 * [________________________]     |
| Email  [________________________]     |
| 电话   [________________________]     |
| ...                                  |
|--------------------------------------|
| 3 Relationship                       |
| 类型 * [请选择 v]                    |
| [x] Primary  [ ] Legal ...           |
|--------------------------------------|
| [取消]        [创建 Student 与主联系人]|
+--------------------------------------+
~~~

移动端保持一个连续表单，不使用隐藏字段的横向 stepper。底部操作区可粘附，但必须为验证提示和系统键盘留出空间；提交时按钮宽度稳定并显示进行中状态。

### 5.3 疑似重复警告桌面线框 W05

~~~text
+----------------------------------------------------------------------------------+
| 疑似已有记录                                                       [关闭]        |
|----------------------------------------------------------------------------------|
| 非空姓名、Email 或电话中至少一项相同。请人工判断；系统不会自动关联或合并。         |
|                                                                                  |
| 类型       姓名             命中字段         辅助信息              操作          |
| Guardian   Chan Mei         Email            ***1234               [选择此人]    |
| Guardian   May Chan         电话             m***@...               [选择此人]    |
| Student    Lin Chen         姓名              DOB 未作为匹配条件     [打开查看]    |
|----------------------------------------------------------------------------------|
| [返回修改]                         [确认仍为独立记录并继续]                       |
+----------------------------------------------------------------------------------+
~~~

- 警告可在字段完成后预检，也必须由提交端重新检查；DOB、gender 不显示为“命中原因”。
- Guardian 匹配结果允许明确“选择此人”，随后仍需填写与该 Student 的 relationship。
- “继续独立建档”必须是明确操作；它不创建 DuplicateCandidate，不进入复核工作台。
- Student 匹配只提供查看或返回修改，不自动把 `/students/new` 转成 Case 创建或关联流程。

### 5.4 疑似重复警告移动线框 W06

~~~text
+--------------------------------------+
| 疑似已有记录                    [x]   |
| 至少一项姓名/Email/电话相同。         |
| 系统不会自动关联或合并。              |
|--------------------------------------|
| Guardian · Chan Mei                  |
| 命中: Email · ***1234                |
| [选择此人]                           |
|--------------------------------------|
| Student · Lin Chen                   |
| 命中: 姓名                           |
| [打开查看]                           |
|--------------------------------------|
| [返回修改]                           |
| [确认独立建档并继续]                 |
+--------------------------------------+
~~~

移动端警告采用全屏 dialog，焦点进入标题，关闭后回到触发字段。命中字段以文字表达，不只靠图标。

## 6. Student 详情

### 6.1 桌面线框 W07

~~~text
+----------------------------------------------------------------------------------+
| < Student 列表      Lin Chen       [Active]        [编辑] [新建 Case] [更多 v]   |
|----------------------------------------------------------------------------------|
| 基本资料                                                                         |
| DOB 2012-05-10 · 年龄 14（即时计算） · Gender 未提供                             |
| Email l***@... · 电话 ***1234                                                    |
|----------------------------------------------------------------------------------|
| Guardian 摘要                                      [管理 Guardian]               |
| Primary  Chan Mei · mother · 通知接收                                          >|
| Other    Chen Kai · father · Legal                                             >|
|----------------------------------------------------------------------------------|
| Case 摘要                                                                         |
| 2027 Entry · background_collection · Primary Advisor: A. Lee                  >   |
| 无 Case 时：[新建 Case]                                                          |
|----------------------------------------------------------------------------------|
| 最近变更摘要                                               [查看审计入口]         |
+----------------------------------------------------------------------------------+
~~~

- 编辑在同一 canonical detail context 打开表单区或 dialog；不创建独立业务状态。
- DOB 为空时显示“未提供”，年龄显示“不可计算”；年龄永远不是输入框或 PATCH 字段。
- “管理 Guardian”进入 `/students/:studentId/guardians`。
- “新建 Case”仅对 active Student 和具备 Advisor create capability 的当前身份显示；预选 Student 后仍由服务端验证。
- “申请删除”位于“更多”中，只对 Advisor/Founder 和服务端判定可申请时启用。若存在 open Case，入口显示不可用原因，不绕过 BR 约束。
- `pending_delete` 使用页首状态条明确标识；`deleted` 直达按不可见语义处理，不暴露其历史存在。

### 6.2 移动线框 W08

~~~text
+--------------------------------------+
| < Student        Lin Chen       [⋮]  |
| [Active]                             |
|--------------------------------------|
| 基本资料                       [编辑] |
| DOB 2012-05-10 · 年龄 14             |
| Gender 未提供                        |
| l***@... · ***1234                   |
|--------------------------------------|
| Guardian                             |
| Primary · Chan Mei · mother      >   |
| Chen Kai · father                >   |
| [管理 Guardian]                      |
|--------------------------------------|
| Cases                                |
| 2027 Entry · Background collection > |
|--------------------------------------|
| [新建 Case]                          |
+--------------------------------------+
~~~

移动端将编辑、删除申请放入明确命名的操作菜单；主任务“新建 Case”保留为文字按钮。返回键恢复 Student 列表上下文，而不是清空搜索。

## 7. Guardian 关系工作区

### 7.1 桌面线框 W09

~~~text
+----------------------------------------------------------------------------------+
| < Lin Chen             Guardian                                      [新增关系]   |
|----------------------------------------------------------------------------------|
| 当前主要联系人                                                                  |
| Chan Mei · mother · Email + 电话                         [交接主要联系人]        |
|----------------------------------------------------------------------------------|
| [当前关系] [历史]                                                               |
| Guardian       Relationship       Flags                    Effective       操作  |
| Chan Mei       mother             Primary, Notification    2026-01-10 -    [⋮]   |
| Chen Kai       father             Legal                    2026-01-10 -    [⋮]   |
|----------------------------------------------------------------------------------|
| 选择“新增关系”后：                                                              |
| [新建 Guardian] [关联已有 Guardian]                                             |
| Guardian 字段 / 显式搜索结果                                                    |
| Relationship type * [v]  Description [...]                                      |
| [ ] Legal [ ] Emergency [ ] Billing [ ] Notification                            |
| 默认非 Primary                                                [取消] [建立关系]  |
+----------------------------------------------------------------------------------+
~~~

- 新增默认是非 Primary。若业务需要成为 Primary，先建立关系，再使用独立“交接主要联系人”操作。
- “关联已有 Guardian”必须由员工输入搜索条件并明确选择结果；不根据疑似重复结果自动关联。
- 关系类型提供 BR-022 全部值；`other` 的说明在客户端与服务端均必填校验。
- “交接主要联系人”只列该 Student 的 current related Guardians；一次提交原子关闭旧 primary 版本并创建新 primary 版本。旧 Guardian 的其他 current relationship 不被结束。
- “解除关系”结束指定 relationship，不删除 Guardian。当前 Primary 不可先解除，必须先完成交接。
- “历史”显示关系版本、开始/结束时间、操作者和审计入口；历史不可直接编辑。

### 7.2 移动线框 W10

~~~text
+--------------------------------------+
| < Lin Chen          Guardian     [+] |
|--------------------------------------|
| Primary                              |
| Chan Mei · mother                    |
| Email + 电话                         |
| [交接主要联系人]                     |
|--------------------------------------|
| [当前] [历史]                        |
| Chan Mei                         [⋮]  |
| mother · Primary · Notification      |
| 自 2026-01-10                        |
|--------------------------------------|
| Chen Kai                         [⋮]  |
| father · Legal · 自 2026-01-10       |
+--------------------------------------+
~~~

新增、交接和解除使用全屏 sheet；关闭后回到原列表位置。关系 flag 在窄屏换行，不能用一串无文字图标替代。

## 8. 删除申请与 Founder 审批

### 8.1 桌面线框 W11

~~~text
+----------------------------------------------------------------------------------+
| < Student                删除申请                                                |
|----------------------------------------------------------------------------------|
| [待处理] [已决定]                         [对象: 全部 v] [申请人 v]               |
|----------------------------------------------------------------------------------|
| 对象             类型       当前状态       申请人       申请时间          操作   |
| Lin Chen         Student    Pending        A. Lee       2026-08-26       [审查]  |
| Chan Mei         Guardian   Pending        Founder      2026-08-25       [审查]  |
|----------------------------------------------------------------------------------|
| 审查面板                                                                        |
| 对象摘要 · 约束检查结果 · 相关审计入口                                          |
| Student: 当前无 open Case / Guardian: 当前无 active student relationship         |
| [拒绝]                                                         [批准软删除]       |
+----------------------------------------------------------------------------------+
~~~

- 此队列只对 Founder 可见；Advisor 从对象上下文提交申请，但没有审批入口。
- Student 存在 open Case 时不可申请或批准；Guardian 存在任一 active student relationship 时不可申请或批准。
- 批准只执行软删除；拒绝恢复为 active。没有 restore、purge 或物理删除操作。
- 审查时显示服务端最新约束检查。打开页面后的状态变化按 stale/version conflict 处理，不允许用旧快照批准。
- Guardian 删除申请入口来自已结束关系的历史/详情上下文，并仅在无 active relationship 时可用。

### 8.2 移动线框 W12

~~~text
+--------------------------------------+
| < Student          删除申请          |
| [待处理] [已决定]                    |
| [对象: 全部 v]                       |
|--------------------------------------|
| Student · Lin Chen               >   |
| Pending · A. Lee · 08-26             |
|--------------------------------------|
| Guardian · Chan Mei              >   |
| Pending · Founder · 08-25            |
|--------------------------------------|
| 审查 Lin Chen                        |
| 当前无 open Case                     |
| [拒绝]          [批准软删除]         |
+--------------------------------------+
~~~

移动端先进入单条审查页/全屏 sheet，再出现决定按钮，避免列表滑动时误触。批准按钮始终使用“软删除”文字，不显示垃圾桶图标作为唯一含义。

## 9. ReferralSource

### 9.1 列表桌面线框 W13

~~~text
+----------------------------------------------------------------------------------+
| 来源                                                  Founder: [新建来源]         |
|----------------------------------------------------------------------------------|
| [搜索名称________________________] [状态: Active v] [类型: 全部 v] [清除]        |
|----------------------------------------------------------------------------------|
| 名称                       类型                    状态             最近更新       |
| Customer referral - Wong   Customer referral       Active           2026-08-20  > |
| School partner - ABC       School referral         Inactive         2026-07-15  > |
|----------------------------------------------------------------------------------|
| 已加载 25 条                                      [加载更多]                     |
+----------------------------------------------------------------------------------+
~~~

- Founder 可查看 active/inactive、创建并进入管理；Advisor 只收到 active 结果，页面不呈现 inactive 筛选或数量。
- Advisor 在本页只读；其为自己 Case 选择来源发生在 Case intake/workspace，不在 CRM 中建立 Case 归属。
- Admin 单独与 Contractor denied。Founder+Admin 使用 Founder capability。

### 9.2 列表移动线框 W14

~~~text
+--------------------------------------+
| < ERP              来源          [+] |
| [搜索名称_______________________]     |
| [Active v] [类型 v]                  |
|--------------------------------------|
| Customer referral - Wong         >   |
| Customer referral · Active           |
|--------------------------------------|
| School partner - ABC             >   |
| School referral · Inactive           |
|--------------------------------------|
|             [加载更多]               |
+--------------------------------------+
~~~

Advisor 看不到 `+`、inactive 行和状态筛选。Founder 的新增按钮使用图标并提供 accessible label/tooltip。

### 9.3 详情桌面线框 W15

~~~text
+----------------------------------------------------------------------------------+
| < 来源列表          Customer referral - Wong      [Active]      [编辑] [停用]    |
|----------------------------------------------------------------------------------|
| 名称          Customer referral - Wong                                           |
| 类型          customer_referral                                                  |
| 说明          Wong family referral                                               |
| 状态          Active                                                             |
| 创建/更新     时间与操作者摘要                                  [审计入口]       |
|----------------------------------------------------------------------------------|
| 编辑模式（Founder）                                                             |
| 名称 * [________________________]  类型 * [________________ v]                   |
| 说明   [____________________________________________________]                     |
|                                              [取消] [保存]                       |
+----------------------------------------------------------------------------------+
~~~

- Founder 创建/编辑时，选择 `other` 后说明立即变为必填。
- 停用使用确认 dialog，并说明停用后不能用于新建或变更 Case；不把它描述为删除。
- Advisor 只读 active 来源；若来源在打开后变为 inactive，刷新/写入路径显示 unavailable，不泄露额外管理信息。

### 9.4 详情移动线框 W16

~~~text
+--------------------------------------+
| < 来源列表    来源详情          [⋮]  |
| [Active]                             |
|--------------------------------------|
| Customer referral - Wong             |
| Customer referral                    |
|--------------------------------------|
| 说明                                 |
| Wong family referral                 |
|--------------------------------------|
| 更新 2026-08-20                      |
| [编辑]                               |
+--------------------------------------+
~~~

Founder 的“编辑/停用”在操作菜单中均使用文字；Advisor 不渲染该菜单。返回时保留来源列表筛选。

## 10. `/cases/new` 连续建案向导

### 10.1 桌面线框 W17

~~~text
+----------------------------------------------------------------------------------+
| < Cases                 新建 Case                                                |
| 1 Student  -----  2 Case details  -----  3 Review                                |
|----------------------------------------------------------------------------------|
| STEP 1 · Student                                                                 |
| [选择已有 active Student] [先建立 Student + Primary Guardian]                    |
| 搜索 [姓名、Email 或电话____________________]                                    |
| ( ) Lin Chen · DOB 2012-05-10 · Active                                           |
| ( ) Mei Wong · DOB 未提供 · Active                                               |
| 新建模式复用 W03 的完整字段、Guardian 模式、关系字段和 W05 重复警告；            |
| CRM 原子提交成功后回到本 step，并明确选中刚创建的 active Student。                |
|                                                       [取消] [下一步]             |
|----------------------------------------------------------------------------------|
| STEP 2 · Case details                                                            |
| Student         Lin Chen                                          [更换]          |
| Primary Advisor * [请选择 active Advisor v]                                       |
| ReferralSource    [无 / 搜索 active 来源 v]                                      |
| Intake year *     [____]    Admission type * [entry / transfer]                  |
| 线下签约时间 *     [2026-08-26 10:30 Asia/Hong_Kong]                              |
| Assessment manifest  服务端绑定当前 approved K12 版本（不可选择）                 |
|                                             [上一步] [下一步]                     |
|----------------------------------------------------------------------------------|
| STEP 3 · Review                                                                  |
| Student           Lin Chen                                                       |
| Primary Guardian  Chan Mei · mother                                              |
| Primary Advisor   A. Lee                                                         |
| ReferralSource    Customer referral - Wong / 无                                  |
| Intake            2027 · entry                                                   |
| Signed at         2026-08-26 10:30 Asia/Hong_Kong                                |
| Assessment        K12 manifest v2026.08（服务端只读摘要）                         |
| 创建后            自动进入 background_collection                                |
|                               [上一步] [创建 Case]                                |
+----------------------------------------------------------------------------------+
~~~

- Step 1 只允许 active Student；`pending_delete` 和 `deleted` 不出现在可选结果。
- 整个向导只对 Advisor create capability 开放；Founder 单独、Founder + Admin、Admin 单独和 Contractor 直达均 denied，Founder + Advisor 按 Advisor capability 开放。
- 新建 Student 子流程在 `/cases/new` 内保持向导连续性，但调用 CRM 原子建档契约；成功 receipt 只回填 `student_id` 和必要显示摘要。
- Step 2 必须指定 Primary Advisor；ReferralSource 可空且只列 active。Advisor 仅能在符合自己 Case capability 的条件下选择。签约时间以 `Asia/Hong_Kong` 显示，提交为带时区的 ISO-8601 `signed_at`，服务端规范化并保存 UTC。
- Step 3 只复核，不允许以文字摘要绕过服务端最终验证。
- 创建 Case 成功后由服务端自动设为 `background_collection`，前端跳转 `/cases/:caseId/assessment`。
- Case 创建失败时，已成功建立的 Student 不回滚；保留向导输入，显示“重试创建 Case”和“查看 Student”。
- 浏览器后退按 step 返回；从第一步离开时回到 `/cases` 或发起该流程的 Student 详情。F2 不设计持久化向导草稿；刷新后重新开始 Case 输入，但已提交成功的 CRM Student 不受影响。

### 10.2 移动线框 W18

~~~text
+--------------------------------------+
| < Cases          新建 Case           |
| Step 1 of 3 · Student                |
|--------------------------------------|
| [选择已有] [先建立 Student]          |
| [搜索姓名、Email 或电话_________]     |
| ( ) Lin Chen · Active                |
| ( ) Mei Wong · Active                |
|--------------------------------------|
| Step 2                               |
| Primary Advisor * [选择 v]           |
| ReferralSource    [无 / 选择 v]      |
| Intake year *     [____]             |
| Admission type *  [entry v]          |
| Signed at *       [2026-08-26 10:30 HKT]|
|--------------------------------------|
| Step 3 · Review                      |
| Student: Lin Chen                    |
| Guardian: Chan Mei                   |
| Advisor: A. Lee                      |
| Source: 无                           |
| Intake: 2027 entry                   |
| Signed at: 2026-08-26 10:30 HKT       |
| Assessment: K12 v2026.08 (只读)      |
| 创建后: Background collection        |
|--------------------------------------|
| [上一步]              [创建 Case]    |
+--------------------------------------+
~~~

移动端一次只展示当前 step；上方使用文字进度，不用可误点的紧凑标签。Review 摘要纵向排列，提交按钮在请求期间锁定且不改变布局尺寸。

## 11. 主要操作交互契约

| 主要操作 | 按钮位置 | 前置条件 | 成功反馈 | 失败反馈 | stale / version conflict |
|---|---|---|---|---|---|
| 搜索/加载 Student | 列表筛选栏、列表尾部 | 页面可访问；query 合法 | 原位更新结果并保留筛选 | 筛选栏下方 error + 重试；保留已输入 query | cursor 失效时从首批重载并明确提示列表已更新 |
| 创建 Student + Primary Guardian | `/students/new` 底部主按钮 | 必填字段合法；Guardian email/phone 至少一项；关系合法；重复警告已人工处理 | standalone 进入 Student 详情；Case 向导返回并选中新 Student；toast/inline receipt | 字段错误就地关联；系统错误保留输入并提供重试 | 服务端重查重复与唯一 primary；不静默覆盖，返回表单并重新确认 |
| 选择已有 Guardian | 搜索结果行“选择此人” | 员工主动搜索；结果仍可访问 | 显示已选 Guardian 摘要，继续填写 relationship | 结果消失/无权时清除选择并给安全提示 | 提交时重新验证版本和 active relationship，不自动换选其他人 |
| 确认独立建档 | duplicate warning 底部主按钮 | 员工已看到具体命中字段 | 回到提交路径；本次确认随请求送交 | 警告无法重新验证时保持 dialog 并可重试 | 新命中出现时重新展示完整 warning，不沿用过期确认 |
| 编辑 Student | 详情页标题区“编辑” | 对象可见且服务端 capability 允许 | 返回只读态、刷新派生年龄与摘要、toast | 字段错误就地显示；系统错误保留草稿 | 禁止 last-write-wins；加载最新版并允许重新应用本地输入 |
| 申请软删除 | Student/Guardian 对象菜单 | Student 无 open Case；Guardian 无 active relationship；Advisor/Founder | 对象显示 `pending_delete`，入口变为“申请已提交” | 明确显示当前阻止条件；不改变状态 | 服务端重验约束；冲突时刷新对象和申请状态 |
| 新增 Guardian 关系 | Guardian 页右上“新增关系” | Student 可访问；新/已有 Guardian 明确；relationship 合法 | 关闭表单、列表出现 current relationship、toast | 字段/重复警告就地；系统错误保留输入 | 重载 current relations；不会自动变更 Primary |
| 交接主要联系人 | 当前 Primary 区按钮 | 目标是同一 Student 的 current related Guardian | 页首 Primary 原位更新；历史出现版本；toast | 不完整/无权/目标失效时 dialog 内报错 | 原子命令失败则旧 Primary 保持；刷新候选后重选 |
| 解除关系 | relationship 行菜单 | 关系 current 且不是当前 Primary | 行移入历史、显示结束时间、toast | 当前 Primary 明确要求先交接；其他失败不改 UI | 服务端版本不符时刷新该关系，不猜测结束结果 |
| 批准/拒绝删除 | Founder 审查面板底部 | Founder；申请 pending；业务约束仍满足 | 批准后对象不可见；拒绝后对象 active；队列更新 | 显示约束变化或系统错误；申请仍留在当前列表 | 强制重新加载申请和依赖状态后再决定 |
| 创建/保存 ReferralSource | 列表右上或详情编辑底部 | Founder；类型合法；`other` 有说明 | 进入/返回详情并 toast | 字段错误就地；系统错误保留草稿 | 不覆盖新版本；刷新详情后由 Founder 再提交 |
| 停用 ReferralSource | 详情页标题区/操作菜单 | Founder；来源当前 active | 状态改为 inactive；toast；不再出现在 Advisor/new Case 选择器 | dialog 内错误；状态不变 | 已被他人停用时刷新为 inactive 并结束操作 |
| Case 向导“下一步” | 每个 step 底部右侧 | 当前 step 客户端验证通过 | 进入下一 step，保留已选摘要 | 字段级提示并聚焦首个问题 | 对象失效时回到对应 step 并清除失效选择 |
| 创建 Case | Review 底部主按钮 | Advisor create capability；active Student、有效 Primary Advisor、可选 active 来源及 Cases 字段均通过服务端重验 | receipt 后 replace 到 `/cases/:caseId/assessment`；显示创建成功反馈 | `VALIDATION_FAILED` 保留向导并显示 allowlisted field_errors；其他错误可重试或查看已建 Student；不制造半成 Case | `STALE_VERSION` 或业务 `CONFLICT` 不自动重放；Idempotency-Key 重试只接受同一明确 receipt |

所有提交按钮在请求期间只允许一次激活，并以稳定宽度显示 loading。错误摘要置于表单顶部，同时字段旁保留具体错误；焦点移至首个可处理问题。成功反馈不能只靠颜色，且不会在 canonical 跳转前丢失。

## 12. 页面状态矩阵

| 页面 | loading | empty | validation | duplicate warning | denied | unavailable | stale/conflict | error | success 与返回 |
|---|---|---|---|---|---|---|---|---|---|
| `/students` | 表头和稳定行骨架 | “无符合条件的 Student”，保留清除筛选/新建入口 | 非法 query 就地修正 | 不适用 | 统一 denied；返回安全 ERP 入口 | 服务暂不可用，保留筛选并重试 | cursor 失效后提示并首批重载 | 列表区 error，不清空已有结果 | 搜索原位更新；详情返回恢复上下文 |
| `/students/new` | 首载受控值骨架；提交锁定 | 不适用 | 字段旁 + 顶部摘要 | W05/W06；只因非空姓名/Email/电话相同 | 不渲染表单；安全返回 `/students` | 受控值或搜索不可用时禁止提交 | 重新预检；不得沿用旧 duplicate acknowledgement | 保留全部输入并重试 | standalone 到详情；Case 向导回填 Student |
| `/students/:studentId` | 标题与 section 骨架 | Guardian/Case 分区各自 empty CTA | 编辑态字段错误 | 编辑产生匹配时同样警告；查看态不适用 | 不区分不存在和无权 | `pending_delete` 明示；`deleted` 不可见；依赖不可用分区降级 | 禁止覆盖，刷新最新版本 | 分区 error 可独立重试 | toast 后刷新详情；返回保留列表上下文 |
| `/students/:studentId/guardians` | Primary 和关系行骨架 | current 为空是数据不变量异常，不提供绕过；历史可为空 | 关系类型/other 说明/联系方式就地校验 | 新建/搜索 Guardian 时启用 | 统一 denied；返回 `/students` | 目标 Guardian/关系失效时清除选择 | 交接/解除失败后重载 current relationships | 表单保留输入；列表可重试 | 更新列表和历史；返回 Student 详情 |
| `/students/deletion-requests` | 队列与审查面板骨架 | “没有待处理申请” | 决定命令参数由冻结 DTO 校验 | 不适用 | 非 Founder 统一 denied | 约束不满足时说明不能批准 | 决定前强制刷新；冲突不提交旧决定 | 保留选中申请并重试 | 队列移除/更新；返回 `/students` |
| `/referral-sources` | 筛选和行骨架 | Founder 有新建 CTA；Advisor 只显示空结果 | query 非法就地处理 | 不适用 | Admin/Contractor denied | Advisor 永不接收 inactive；服务不可用可重试 | cursor 失效提示并重载 | 保留筛选和已有结果 | 详情返回恢复筛选 |
| `/referral-sources/:sourceId` | 标题与字段骨架 | 不适用 | `other` 说明必填等字段错误 | 不适用 | 对象级统一 denied | Advisor 访问已 inactive 来源显示 unavailable/安全返回 | 编辑/停用不覆盖新版本 | 保留 Founder 编辑草稿 | toast + 最新详情；返回来源列表 |
| `/cases/new` | options 分区骨架；提交锁定 | 无 active Student/Advisor/来源分别给对应可行动提示；来源为空允许继续 | 每 step 字段旁 + Review 摘要 | 新建 Student 子流程使用 W05/W06 | 无 Advisor create capability 统一 denied | 选择项失效则退回对应 step；来源失效可清除后继续 | 服务端重验全部引用；不静默替换 Student/Advisor | 保留可恢复的内存态输入；Student 已建则给详情入口 | 成功到 canonical assessment；取消回 `/cases` 或 Student 详情 |

### 12.1 通用 denied 与 unavailable

~~~text
+----------------------------------------------------------+
| 无法访问此页面                                            |
| 当前身份没有所需权限，或该记录不可用。                    |
| [返回 Student / Cases / 我的任务]                         |
+----------------------------------------------------------+
~~~

拒绝页不显示姓名、Email、电话、对象 ID 对应的人或“曾被删除”等敏感事实。Admin 单独和 Contractor 直达客户 URL 时遵循同一规则。

## 13. 桌面与移动交互原则

| 主题 | 桌面 | 移动 |
|---|---|---|
| 导航 | ERP 主导航 + 页面面包屑；详情保留列表上下文 | 顶部返回 + 单列内容；操作收进清晰命名菜单 |
| 列表 | 稳定列宽、可扫描表格、筛选栏不因结果改变高度 | 列表条目纵向呈现，不横向压缩表格 |
| 表单 | 相关字段分组，两列仅用于短字段 | 单列输入；键盘出现时不遮挡错误与主按钮 |
| 向导 | 可见 step 状态但只能按规则前进 | 每屏一个 step，固定文字进度 |
| Dialog | 仅用于确认、重复警告和短命令 | 复杂表单/警告使用全屏 sheet |
| 反馈 | section 级 loading/error + 页面级 receipt | 相同语义；反馈靠文字、图标与焦点，不只靠颜色 |

整体应是安静、实用、工作导向的 ERP：信息密度服务于扫描和重复操作；不使用营销 hero、装饰性卡片堆叠、无关插画或颜色方案。页面 section 为无框架布局，只有重复记录、dialog 和确需边界的工具区域可使用容器。

## 14. API、DTO 与权限追溯

### 14.1 追溯矩阵

HTTP、query、DTO、错误码和 receipt 已由 Architect 以 D1–D5 冻结；下表把页面操作追溯到 confirmed BR、approved module public port 和冻结 transport。

| 页面/操作 | Approved public query/command | 页面所需最小 DTO | 服务端权限边界 | 事实来源 |
|---|---|---|---|---|
| Student 列表 | `GET /api/v1/students?q=&status=&limit=&cursor=` (`listStudents`) | `id`、`display_name`、脱敏联系提示、`status`、`updated_at`、`record_version`、opaque `next_cursor`；q trim + Unicode/大小写规范化 contains；status 仅 active/pending_delete；limit 默认25/最大100；排序 `updated_at DESC,id ASC` | Founder 组织授权范围；Advisor capability + 业务关系；Admin/Contractor denied | BR-020、BR-029；approved CRM §10–12；Architect D1 |
| Student 详情 | `GET /api/v1/students/:studentId` (`getStudent`) | allowlisted Student 字段、`record_version`、current primary Guardian 摘要、Guardian/Case 受控摘要、`allowed_actions`；各 owner 独立授权 | 每个 owner 独立重验；摘要不扩大 CRM 或 Cases 访问 | BR-020；approved CRM §11、Cases §11；Architect D1 |
| Student 编辑 | `PATCH /api/v1/students/:studentId` (`updateStudent`) | 可编辑字段 + `expected_record_version`；统一 envelope | Founder；Advisor capability + 业务关系 | BR-020；approved CRM §12；Architect D1 |
| Student + Primary Guardian | `POST /api/v1/crm/potential-duplicates`、`POST /api/v1/students` | precheck 接收 kind + 非空 name/email/phone；返回命中字段、最小脱敏候选、10 分钟 opaque `warning_token`；create 使用 new|existing Guardian tagged union + 完整 relationship；receipt 为 Student/Guardian/relationship 各自 id+record_version | Founder/Advisor；已有 Guardian 搜索返回最小提示；不建 DuplicateCandidate | BR-020–BR-023；approved CRM §6、§7、§11–12；Architect D2 |
| Guardian current/history | `GET/POST /api/v1/students/:studentId/guardians`、`GET /api/v1/students/:studentId/guardian-relationships/history` | Guardian 目标字段、relationship version、27 种 type、description、flags、starts/ends、allowed actions；current/history 分开 | Founder；Advisor capability + 业务关系 | BR-021、BR-022；approved CRM §5、§11；Architect D3 |
| 新增/交接/解除关系 | `POST /api/v1/students/:studentId/guardians/primary-handoffs`、`POST /api/v1/students/:studentId/guardian-relationships/:relationshipId/end` | new/existing Guardian union、relationship input、successor relationship ID；相关 expected_record_version | Founder；Advisor capability + 业务关系；逐命令重验唯一 Primary | BR-024–BR-026；approved CRM §12；Architect D3 |
| 删除申请/决定 | `POST /api/v1/students/:id/deletion-requests` 或 `/guardians/:id/deletion-requests`；`GET /api/v1/crm/deletion-requests`；`POST /api/v1/crm/deletion-requests/:requestId/decisions` | decision body `decision=approve|reject` + `expected_record_version`；receipt 仅受影响 opaque ids、最新 status/record_versions、`occurred_at` | Advisor/Founder 申请；仅 Founder 决定；Admin/Contractor denied | BR-029；approved CRM §9–12、Cases `getCustomerDeletionGuard`；Architect D3 |
| ReferralSource 列表/详情 | CRM `listReferralSources`、`getReferralSource` | UUID、名称、11 种 type、description、status、version、受控审计摘要 | Founder 可见/管理；Advisor 只读 active；Admin/Contractor denied | BR-027、BR-028；approved CRM §8/§10–11；Architect D4 |
| ReferralSource 写操作 | `POST/GET/PATCH /api/v1/referral-sources`、`POST /api/v1/referral-sources/:sourceId/deactivate` | query q 规范化 display_name contains；status/source_type 精确匹配；默认25/最大100；排序 `display_name ASC,id ASC`；write body 带 description；update/deactivate 带 expected_record_version；receipt `referral_source{id,status,record_version,updated_at}` | Founder only；Advisor 服务端 active-only，inactive 直达通用 `NOT_FOUND` | BR-027、BR-028；approved CRM §12；Architect D4 |
| Case intake options | `GET /api/v1/cases/intake-options?student_q=&advisor_q=&source_q=` | 单一只读 coordinator；每类最多20条；allowlisted active Student/Advisor RoleBinding/ReferralSource option；无 organization/actor/total count | 仅 Advisor create capability；coordinator 只组合 owner public query，不直读私表 | BR-020、BR-027、BR-030；approved F0、authorization、Cases §4；Architect D5 |
| 创建 K12 Case | `POST /api/v1/cases` (`createK12Case`) | `student_id`、`primary_advisor_role_binding_id`、可空 `referral_source_id`、`intake_year`、`admission_type`、必填 `signed_at`；manifest 不由客户端传 | Advisor-only；Founder 单独/Founder+Admin/Admin/Contractor denied；Founder+Advisor 按 Advisor 允许 | BR-010、BR-030；approved F0、authorization、Cases §4/§12；Architect A1/A2/D5 |
| Case 创建 receipt | `POST /api/v1/cases` result | `case_id`、`stage=background_collection`、`workflow_status=active`、`record_version`、`assessment_manifest{id,version}`、`assessment_url=/cases/:caseId/assessment` | 与创建命令同一次授权和幂等结果 | BR-030；approved Cases SD-CASE-002；Architect D5 |

### 14.2 写命令通用 transport 要求

Approved CRM/Cases module contract 与 Architect D1–D5 已冻结所有写命令的 envelope、幂等、版本和返回约束。因此 F2 前端统一需要：

- 每次用户意图生成并复用同一个 `Idempotency-Key`，直到收到确定结果或用户明确重新发起。
- update、关系命令、删除决定、来源修改携带服务端上次返回的 expected record/version；`STALE_VERSION` 不自动重放写入。
- error DTO 只消费稳定 code、field path、safe message、current version/allowed actions 等 allowlisted 字段；字段错误统一为 `VALIDATION_FAILED` + `field_errors`，版本冲突为 `STALE_VERSION`，重复业务约束为 `CONFLICT`；不显示日志、表名、原始异常或 PII。
- 所有上述查询/命令使用统一 envelope、`no-store` 响应和 `X-Request-Id`；POST 写命令使用 `Idempotency-Key`。
- Case receipt 的 assessment URL 必须是 canonical `/cases/:caseId/assessment`，前端不根据 ID 拼出其他 legacy workspace。

## 15. 当前实现证据与目标差异

以下只描述 2026-08-26 的现状证据，不把旧代码或 implementation ticket 当作目标事实。

| 现状证据 | 当前行为 | F2 目标处理 |
|---|---|---|
| [CRM-01](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/docs/implementation/CRM-01_STUDENT_PRIMARY_GUARDIAN_CREATE_PLAN.md) | 初始关系只覆盖旧 `father/mother/other_guardian`，偏向新建 Guardian | 使用 BR-022 全量 27 种类型；同时支持员工明确选择已有 Guardian；保持一个 CRM 原子操作 |
| [StudentCreateForm.tsx](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/crm/StudentCreateForm.tsx:24) | 表单缺 Student/Guardian gender、Guardian DOB；只提供新 Guardian，旧关系枚举位于同一表单 | redesign 为 W03/W04，并加入 manual existing Guardian 模式和目标字段 |
| [students route-contract.ts](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/students/route-contract.ts:6) | create DTO 与目标字段/关系枚举不一致 | 不沿用为目标；按 approved CRM public command + D2 冻结 transport DTO |
| [StudentsDirectory.tsx](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/crm/StudentsDirectory.tsx:48) | 客户端本地过滤，且展示 duplicate review 正式入口 | redesign 为授权后的目标列表；duplicate 工作台从 Release 1 隔离 |
| [profile-maintenance-handler.ts](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/profile-maintenance-handler.ts:20) | 更新 DTO 未覆盖目标 Student gender、Guardian gender/DOB | redesign；年龄只由 DOB 派生，补齐目标可空字段，不新增 age |
| [Guardian attach handler](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/students/[studentId]/guardians/handler.ts:14) | 只接受已有 Guardian ID 和旧关系类型，缺完整说明/flags/历史动作 | redesign 为 W09/W10；新增、新关联、交接、解除和版本历史按 CRM public port |
| [CRM-04](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/docs/implementation/CRM-04_DUPLICATE_REVIEW_MERGE_UNDO_PLAN.md) 与 [duplicates page](</Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/(erp)/students/duplicates/page.tsx>) | 持久化 DuplicateCandidate、merge/undo、Data Reviewer 工作台 | `isolate_from_release1`；只保留瞬时 warning、明确选已有或独立继续 |
| [deletion requests API](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/crm/deletion-requests/route.ts:10) | 当前主要是 queue/request，缺完整 Founder approve/reject 目标命令 | 保留 route 位置但 redesign W11/W12；只软删除，不恢复、不 purge |
| [CRM-06](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/docs/implementation/CRM-06_REFERRAL_SOURCE_CASE_LINK_PLAN.md) 与 [referral handler](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/referral-sources/handler.ts:16) | 旧规则让 Admin 管理来源，类型/description 也与 BR-028 不一致 | Founder 管理；Advisor 只读 active 并为自己的 Case 选择；Admin/Contractor denied |
| [P1-05](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/docs/implementation/P1-05_CASE_CREATION_PLAN.md) | 旧描述把 Student + Case 当联合建案事务，未满足 Primary Guardian 目标 | 改为 CRM 建档先提交、Cases 创建后提交的连续 UX；模块分别保持自身原子性 |
| [CaseCreateForm.tsx](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/components/cases/CaseCreateForm.tsx:110) | 仅选已有 Student、无来源、旧 admission 值、员工手选 manifest，成功到 Case summary | redesign W17/W18：可先建 Student；`entry/transfer`；来源可选；`signed_at` 必填；manifest 服务端绑定；成功到 assessment |
| [cases route-contract.ts](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/cases/route-contract.ts:8) | 当前 body/response 未实现 F2 冻结的来源、签约时间和 manifest 只读契约 | 不冒充目标；按 approved `createK12Case` 与 D5 更新 transport contract 后才能实现 |

### 15.1 旧正式入口处理

- `/students/duplicates`、`/students/duplicates/:candidateId` 继续按 approved F0 标记 `isolate_from_release1`，不出现在导航、列表空态、警告或成功 receipt 中。
- 旧 merge、undo、corrections 和 Data Reviewer API 即使仍在源码中，也不是 F2 页面依赖。
- `/students/deletion-requests`、`/referral-sources` 和 `/referral-sources/:sourceId` 的 route 可以保留，但其内容和权限必须按本文件 redesign，不能以 `keep` 理解为保留旧业务规则。
- 当前代码中的 Admin 来源入口、旧关系类型和员工 manifest picker 不得进入 Release 1 正式入口。

## 16. Architect 决策

### 16.1 已冻结，直接进入实现契约

| ID | 冻结决定 | F2 落点 |
|---|---|---|
| A1 | 只有 Advisor capability 可创建 Case。Founder 单独不能创建；Founder + Advisor 因 Advisor 角色可创建；Admin 单独、Contractor 均拒绝。依据 approved F0 与 BR-010。 | §2 权限矩阵、Student 详情 CTA、`/cases/new` denied、§14 trace 全部统一为 Advisor-only |
| A2 | 建案必须填写线下签约时间 `signed_at`；Assessment manifest 不让员工手选，由服务端绑定当前 approved K12 manifest，Review 只显示只读版本摘要。 | §3.4、W17/W18、Case create DTO/receipt 和现状差异已明确 |

### 16.2 D1 Student 列表、详情和更新 HTTP 契约（已冻结）

~~~text
GET   /api/v1/students?q=&status=active|pending_delete&limit=&cursor=
GET   /api/v1/students/:studentId
PATCH /api/v1/students/:studentId
~~~

- `q` 先 trim，再对非空姓名、Email、电话做 Unicode/大小写规范化后的 `contains` 搜索；不执行重复判断。
- `status` 只允许 `active|pending_delete`；`deleted` 不可查询。
- `limit` 默认 25、最大 100；排序固定为 `updated_at DESC,id ASC`；opaque cursor 绑定规范化后的 q、status 和固定排序，筛选变化即失效；不返回 total count。
- detail 返回 allowlisted Student 字段、`record_version`、current primary Guardian 摘要、Guardian/Case 受控摘要和 `allowed_actions`，各 owner 独立授权。
- PATCH body 为可编辑字段 + `expected_record_version`；统一 envelope、`no-store`、`X-Request-Id`。

### 16.3 D2 原子建档与瞬时 duplicate warning 协议（已冻结）

~~~text
POST /api/v1/crm/potential-duplicates
POST /api/v1/students
~~~

- precheck 只接收 `kind` + 非空 name/email/phone；返回命中字段、最小脱敏候选和 opaque `warning_token`。
- token 不建实体、不落业务表；服务端签名或受控短期缓存均可。token 绑定 org、actor、kind、规范化字段 hash 和候选版本，10 分钟失效。
- create 的 `primary_guardian` 使用 `new|existing` tagged union，并带完整 relationship input；服务端最终复检。
- 需要人工继续时返回稳定 `DUPLICATE_WARNING_REQUIRED`（HTTP 409）和安全 allowlisted details；只有回传有效 `warning_token` 才能继续独立建档。
- receipt 精确包含 `student{id,record_version}`、`primary_guardian{id,record_version}`、`relationship{id,record_version}`，仍由统一 envelope 包裹；POST 使用 `Idempotency-Key`。

### 16.4 D3 Guardian 关系与删除决定 HTTP 契约（已冻结）

~~~text
GET  /api/v1/students/:studentId/guardians
GET  /api/v1/students/:studentId/guardian-relationships/history
POST /api/v1/students/:studentId/guardians
POST /api/v1/students/:studentId/guardians/primary-handoffs
POST /api/v1/students/:studentId/guardian-relationships/:relationshipId/end
POST /api/v1/students/:id/deletion-requests 或 /api/v1/guardians/:id/deletion-requests
GET  /api/v1/crm/deletion-requests
POST /api/v1/crm/deletion-requests/:requestId/decisions
~~~

- current 与 history 分开；新增/关联使用 `POST /api/v1/students/:studentId/guardians`。
- 交接复用 `POST /api/v1/students/:studentId/guardians/primary-handoffs`；解除使用 relationship ID end 命令。
- 删除申请沿用 Student/Guardian 对象路径；队列为 CRM GET；决定 body 为 `decision approve|reject` + `expected_record_version`。
- 所有修改、交接、解除和决定携带相关 `expected_record_version`；receipt 只返回受影响 opaque ids、最新 status/record_versions、`occurred_at`，统一 envelope、`no-store`、`X-Request-Id`。

### 16.5 D4 ReferralSource HTTP/query/DTO 契约（已冻结）

~~~text
GET   /api/v1/referral-sources?q=&status=&source_type=&limit=&cursor=
POST  /api/v1/referral-sources
GET   /api/v1/referral-sources/:sourceId
PATCH /api/v1/referral-sources/:sourceId
POST  /api/v1/referral-sources/:sourceId/deactivate
~~~

- `q` 只对规范化 display_name 做 `contains`；`source_type`、`status` 精确匹配。
- `limit` 默认 25、最大 100；排序固定为 `display_name ASC,id ASC`；opaque cursor 绑定 filters/sort；不返回 total count。
- Advisor 由服务端强制 active-only；直达 inactive source 返回通用 `NOT_FOUND`；Founder 可见 active/inactive。
- POST/GET/PATCH 路径按上方冻结；write body 带 description；update/deactivate 带 `expected_record_version`。
- receipt 为 `referral_source{id,status,record_version,updated_at}`，统一 envelope、`no-store`、`X-Request-Id`；POST 使用 `Idempotency-Key`。

### 16.6 D5 Case intake options、create 和 receipt HTTP 契约（已冻结）

~~~text
GET  /api/v1/cases/intake-options?student_q=&advisor_q=&source_q=
POST /api/v1/cases
~~~

create body：

~~~text
{
  student_id,
  primary_advisor_role_binding_id,
  referral_source_id?,
  intake_year,
  admission_type: "entry" | "transfer",
  signed_at
}
~~~

- 采用单一只读 coordinator；每类最多 20 条，分别返回 allowlisted active Student、Advisor RoleBinding、ReferralSource option；不返回 organization、actor 或 total count。coordinator 只组合 owner public query，不直读私表。
- POST body 正式冻结为 `student_id`、`primary_advisor_role_binding_id`、可选 `referral_source_id`、`intake_year`、`admission_type`、`signed_at`；不接收 `manifest_id`、`case_number`、`organization_id` 或 actor 字段。
- `signed_at` 接收带时区 ISO-8601，服务端规范化并保存 UTC；UI 以 `Asia/Hong_Kong` 显示。
- receipt 精确为 `case_id`、`stage=background_collection`、`workflow_status=active`、`record_version`、`assessment_manifest{id,version}`、`assessment_url=/cases/:caseId/assessment`。
- 统一 envelope、`no-store`、`X-Request-Id`；POST 使用 `Idempotency-Key`；字段错误为 `VALIDATION_FAILED` + allowlisted `field_errors`，版本冲突为 `STALE_VERSION`，重复业务约束为 `CONFLICT`。

## 17. 验收证据

| 门禁 | 结果 | 证据 |
|---|---|---|
| 整个 F2 页面范围已覆盖 | 满足 | 目标 8 类页面/流程，W01–W18 覆盖桌面和移动 |
| Student 状态和 deleted 边界 | 满足 | §3.1、§4、§6、§12：只显示 active/pending_delete；deleted 完全不可见 |
| 原子 Student + Primary Guardian | 满足 | §5、§14：新/已有 Guardian tagged union，不自动关联 |
| Guardian 全量关系和历史 | 满足 | §3.2、§7：BR-022 27 种 type、other 说明、交接、解除、历史 |
| 软删除 | 满足 | §8、§12：Advisor/Founder 申请、Founder 决定，无恢复/purge |
| ReferralSource 权限 | 满足 | §2、§9、§14：Founder 管理、Advisor active-only、Admin/Contractor denied |
| Case intake 连续性 | 满足 | §0.2、§10：CRM 先提交、Cases 后提交，成功到 canonical assessment |
| 建案角色和冻结字段 | 满足 | A1/A2：Advisor-only；`signed_at` 必填；manifest 服务端绑定且只读 |
| duplicate 规则 | 满足 | §5.3/5.4：只用非空姓名/Email/电话，不使用 DOB/gender，无持久化/merge/undo |
| 状态完整 | 满足 | §11–12：loading、empty、validation、duplicate、denied、unavailable、stale、error、success、返回路径 |
| API/DTO/权限可追溯 | 满足 | §14 追溯 confirmed BR、approved public port 与 Architect 冻结 D1–D5 transport |
| 现状与目标分离 | 满足 | §15 明示旧实现仅为证据，并列出冲突处理 |
| 无新增业务事实 | 满足 | 未新增实体、角色或业务状态；5 项只决定 transport shape |
| 非代码范围 | 满足 | 只新增本设计 Markdown；未修改产品源码、测试、数据库、migration 或云 |

已执行静态校验：本文件本地 Markdown 链接存在性、围栏配对、`git diff --check` 和 docs/product worktree 状态复核；实际结果记录在 §18。

## 18. 交付摘要

| 字段 | 值 |
|---|---|
| document_status | `approved` |
| review_readiness | `approved` |
| owner | Frontend Design |
| scope | F2 only；Student、Guardian、ReferralSource、Case Intake |
| changed | `system-design/50-api-ui/frontend-design/30-f2-crm-case-intake-wireframes.zh-CN.md` |
| wireframe_count | 18（W01–W18） |
| decisions_needed | 0 |
| browser | `not_run` |
| database | `not_run` |
| cloud | `not_run` |
| product_tests | `not_run` |
| F3 | `ready_to_start_after_approval` |
| evidence | 本地链接存在；54 个 Markdown 围栏配对；W01–W18 连续；D1–D5 均为 Architect approved transport contract；无尾随空格；产品 worktree 状态与开始时一致 |

剩余风险：产品实现必须严格使用 D1–D5 的 allowlist、版本、幂等和权限约束；browser、database、cloud、product tests 均为 `not_run`。
