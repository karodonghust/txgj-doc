# F5 Guardian Portal 工作区低保真线框及交互契约

> 状态：`approved`
> Owner：`frontend-design`
> 范围：Release 1 外部 Guardian Portal；单 Case、临时 Grant/PortalSession、白名单只读 workspace
> 前置：F0–F4 approved；BR-060、BR-063 confirmed；External Portal API/UI approved；Authorization model approved
> 本轮不做：产品源码、React/TSX、API 实现、数据库、migration、测试、Platform Billing、外部 Email/SMS/WhatsApp、Portal 写操作、Documents/Notifications 页面

## 0. 一页总览

### 0.1 本轮目标

F5 只定义 Guardian Viewer 通过已批准 Grant/PortalSession 进入单一 ServiceCase 的低保真只读页面：

- `/portal/access`：输入顾问提供的一次性访问入口，兑换独立 PortalSession；失败统一，不泄露 key、Case、Viewer 或 Guardian 事实。
- `/portal/workspace`：读取当前 PortalSession 绑定的一个 Case 白名单 DTO，显示对客阶段、更新时间、已批准展示的学校申请进度、Advisor 标记为可见的消息和行动项。
- Portal session：独立 `__Host-`、HttpOnly、Secure、SameSite=Strict Cookie；请求时重验 Grant、Viewer/Guardian relationship、Case、组织、签发人和 Session 期限。
- 退出：`/api/v1/portal/session/logout` 清除当前 PortalSession，返回 `/portal/access`；不链接到内部 ERP、通知或文件页面。

Portal 只显示“已批准且允许对客公开”的事实。家长看到的行动项可以包含需要家长查看/确认的明确文字，但没有完成、确认、上传、回复或决定按钮；F3 规定的名单确认和 offer 决定仍由 Primary Advisor 通过电话、微信或面谈取得后在内部 ERP 代录。

### 0.2 Portal 连续性

```text
内部 ERP
  /cases/:caseId/access
    Primary Advisor 创建 Grant（7 天、单 Case）
    Founder 可紧急撤销；Primary Advisor 可撤销/重新生成
            |
            | raw key 只显示一次；Guardian 从受控入口打开
            v
Portal /portal/access
  -> POST /api/v1/portal/access/redeem
  -> 独立 PortalSession Cookie
  -> GET /api/v1/portal/workspace
            |
            v
Portal /portal/workspace
  Case 对客阶段 + 更新时间
  学校名称 + customer-facing application status
  对客消息 + 只读行动项
  [刷新] [退出]

任何失效/撤销/过期/关系变化
  -> 下一个请求统一 PORTAL_ACCESS_INVALID 或 PORTAL_UNAVAILABLE
  -> /portal/access，不解释具体原因
```

### 0.3 Portal 与内部 ERP 的硬边界

| 范围 | Portal Guardian Viewer | 内部 ERP Founder/Admin/Advisor/Contractor |
| --- | --- | --- |
| 身份 | PortalViewer + PortalGrant + PortalSession；不是内部 User | Identity User + Membership + RoleBinding + capability |
| 数据 | 单一 Case 的对客 allowlist | 组织内按 Case/Task/Document 关系读取 |
| 写入 | 永久拒绝业务写入 | 由 Cases/Tasks/Documents owner 命令决定 |
| 名单确认/offer | 只读显示 Advisor 标记的公开信息；无确认/决定命令 | F3 Primary Advisor 代录 Guardian confirmation/offer decision |
| 文件 | 不列出、不预览、不下载、不上传 | F4 Documents 按 Case/SchoolTarget/Assignee 授权 |
| Task/通知 | 不读取内部 Task 或 Notifications | F1–F4 内部 Task/Notifications 入口 |
| 返回/退出 | 只能 Portal 内刷新、退出到 `/portal/access` | 内部 ERP 按 F0–F4 canonical route 返回 |

Guardian、Student 不能通过 Portal 写入 Student、Guardian、Case、Candidate List、Application、Task、Document 或 Notifications；不得因同一 Guardian 关联多个 Student 自动看到其他 Case。

## 1. 入口、路由与会话

### 1.1 目标路由树

```text
/portal/access
  |-- raw key 兑换 -> POST /api/v1/portal/access/redeem
  `-- 失效/退出 -> 保留在本页

/portal/workspace
  |-- GET /api/v1/portal/workspace
  |-- [刷新] 重新读取同一 PortalSession
  `-- [退出] POST /api/v1/portal/session/logout -> /portal/access
```

Portal 路由不出现在内部 ERP 一级导航、角色选择器或 Notifications target 中。内部员工访问 `/portal/*` 不会把内部 Session 转换为 PortalViewer；服务端按 PortalSession 单独判断。

### 1.2 Grant 与 Session 事实

| 事实 | 页面表达 | 约束 |
| --- | --- | --- |
| Portal Grant | 由当前 Primary Advisor 为单一 Case 生成的临时入口 | 有效期 7 天；不能在原入口上延期；需重新生成 |
| Founder 撤销 | 只在内部 `/cases/:caseId/access` 执行 | Portal 下一个请求立即失效 |
| Guardian/Viewer relationship | 兑换和每次 workspace read 都服务端重验 | relationship 失效统一访问无效 |
| PortalSession idle timeout | 不在页面展示 secret；接近失效时下一请求重新判断 | approved security contract：15 分钟 idle |
| PortalSession absolute timeout | 可在 access 失效文案中提示重新兑换 | approved security contract：8 小时，且不晚于 Grant expires_at |
| active sessions | 不展示数量和其他 Session | 最多 3 个；超限统一访问失败 |
| Case binding | workspace 永远只对应一个 Case | 不接受客户端 Case ID 切换，不返回其他 Case |
| session cookie | 页面不可读 secret | 独立 `__Host-` HttpOnly/Secure/SameSite=Strict/Path=/ Cookie |

### 1.3 兑换流程与返回路径

```text
打开 /portal/access
  -> 粘贴/输入顾问提供的入口数据
  -> 客户端立即清除入口 fragment（若存在）
  -> [进入案件进度]
  -> POST redeem（只发送受控入口数据，不传 organization/Case/Viewer/Guardian ID）
  -> 201 active + absolute expiry
  -> replace /portal/workspace

redeem 失败
  -> 统一 access invalid / rate limited / unavailable
  -> 不回显输入，不说明 key、Grant、Viewer、Case 或 Session 具体原因

workspace 失效
  -> 401 PORTAL_ACCESS_INVALID
  -> 清除本地 workspace state
  -> [返回访问页] -> /portal/access
```

### 1.4 W01：Portal Access 桌面

~~~text
+--------------------------------------------------------------------------------+
|                           天星案件门户                                        |
|                                                                                |
|                 查看案件进度                                                   |
|                 输入顾问提供的临时访问入口。                                   |
|                                                                                |
|                 访问入口 [____________________________]                       |
|                                                                                |
|                 入口不会加入网址或保存在此装置。                               |
|                                                                                |
|                 [进入案件进度]                                                 |
|                                                                                |
|                 loading：正在验证入口…                                         |
|                 invalid/expired/revoked：目前无法使用此案件门户。              |
|                 rate limited：请稍后再试。                                     |
|                 unavailable：案件门户暂时无法使用。                           |
+--------------------------------------------------------------------------------+
~~~

Access 页面不显示 Case number、Guardian/Student 姓名、学校、Grant 到期原始原因或内部错误。提交期间按钮禁用；失败后保留页面结构但清空敏感输入。

### 1.5 W02：Portal Access 移动

~~~text
+----------------------------------+
|          天星案件门户            |
|                                  |
| 查看案件进度                     |
| 输入顾问提供的临时访问入口。     |
|                                  |
| 访问入口                         |
| [________________________]       |
|                                  |
| 不会加入网址或保存在此装置。     |
|                                  |
| [进入案件进度]                   |
|                                  |
| 正在验证 / 无效 / 暂不可用       |
| [重新输入]                       |
+----------------------------------+
~~~

移动端使用单列、可聚焦输入和明确返回/重试文字；不使用内部 ERP 侧栏、通知图标或第三方追踪控件。

## 2. Workspace 信息架构与最小数据

### 2.1 Workspace 区域

| 区域 | 允许显示 | 不得显示 |
| --- | --- | --- |
| Case 摘要 | `customer-facing stage/label`、对客更新时间 | Case number、Student/Guardian 姓名、联系方式、Assessment、内部备注 |
| 学校进度 | 已批准展示的学校名称、`customer-facing application status` | crawler 原文、内部学校资料、候选草稿、驳回理由、内部 Target/Assignee |
| 消息 | Advisor 明确标记可见的纯文本 body、published_at | 内部通知、自由文字 PII、文件链接、回复输入 |
| 行动项 | 标题、deadline、completed；可表达需要家长查看/确认的信息 | 完成/确认按钮、Task ID、内部 Task、Assignee、Case 写命令 |
| 页面控制 | 刷新、退出、返回访问页 | 任意 Case 切换、外部 URL、ERP/Notifications/Documents 链接 |

每次 workspace read 都由 Cases 对客查询构建白名单结果，不建立 PortalProjection 或复制业务状态。`customer_visible=false` 的 school/message/action item 不进入 DTO；空数组是合法成功结果，不等于 denied。

### 2.2 W03：Workspace 桌面

~~~text
+--------------------------------------------------------------------------------+
| 天星案件门户                                                [刷新] [退出]      |
+--------------------------------------------------------------------------------+
| 案件进度                                                                       |
| 当前阶段：申请处理中                                                          |
| 最近更新：2026-08-26 10:30（香港时间）                                        |
|                                                                                |
| 学校申请进度                                                                   |
| 学校 A                                  申请处理中                             |
| 学校 B                                  等待结果                               |
|                                                                                |
| 家长需要查看的信息                                                             |
| [只读] 材料准备提醒 · 截止 2026-09-01                                          |
| [只读] 顾问已发布最新进度                                                     |
|                                                                                |
| 最新消息                                                                       |
| 2026-08-26 · 顾问发布的对客可见消息正文                                       |
|                                                                                |
| 空态：暂无可见进度/消息/行动项。                                               |
| 失效：目前无法使用此案件门户。 [返回访问页]                                   |
+--------------------------------------------------------------------------------+
~~~

桌面端不显示内部 Case 标识或员工信息；所有值来自当前 PortalSession 的白名单 DTO。行动项没有完成按钮，`completed` 只作为事实标签。

### 2.3 W04：Workspace 移动

~~~text
+----------------------------------+
| 天星案件门户       [刷新] [退出] |
|                                  |
| 案件进度                         |
| 申请处理中                       |
| 最近更新：08-26 10:30            |
|                                  |
| 学校申请进度                     |
| 学校 A                           |
| 申请处理中                       |
| 学校 B                           |
| 等待结果                         |
|                                  |
| 家长需要查看的信息               |
| [只读] 材料准备提醒              |
| 截止：09-01                      |
|                                  |
| 最新消息                         |
| 顾问发布的对客可见消息正文       |
|                                  |
| 暂无可见内容 / 无法使用          |
| [返回访问页]                     |
+----------------------------------+
~~~

移动端按 Case 摘要、学校、行动项、消息的顺序纵向排列；不横向堆叠字段，不让消息正文遮盖退出/返回控件。

### 2.4 W05：Workspace 失效/不可用桌面

~~~text
+--------------------------------------------------------------------------------+
| 天星案件门户                                                                  |
|                                                                                |
|                         目前无法使用此案件门户                               |
|                                                                                |
| 访问入口可能已失效，或服务暂时不可用。                                        |
| 不显示具体 key、Grant、Viewer、Case 或 Session 原因。                         |
|                                                                                |
| [返回访问页]                                      [重新载入]                  |
|                                                                                |
| expired/revoked/session invalid -> 清除 workspace state                       |
| unavailable/runtime -> 保留安全状态，允许重试，不伪造旧数据                 |
+--------------------------------------------------------------------------------+
~~~

Portal 不区分向 Guardian 展示 expired、revoked、relationship inactive 或 Case closed 的具体原因；内部 Audit 可以保存受控 reason code，页面统一为入口无效。

### 2.5 W06：Workspace 失效/不可用移动

~~~text
+----------------------------------+
|          天星案件门户            |
|                                  |
| 目前无法使用此案件门户           |
| 入口可能已失效，或服务暂不可用。 |
|                                  |
| [返回访问页]                     |
| [重新载入]                       |
+----------------------------------+
~~~

移动失效页不提供“切换 Case”“联系内部员工查看详情”或任意外部跳转；Guardian 只能重新兑换顾问提供的新入口。

## 3. Portal API、DTO、错误和会话契约

### 3.1 Approved public API 与页面动作

| 页面动作 | approved endpoint | 最小请求/响应 | 页面边界 |
| --- | --- | --- | --- |
| 兑换 Grant | `POST /api/v1/portal/access/redeem` | 只提交受控入口数据；成功创建独立 PortalSession，返回 active/absolute expiry；不传 organization、Case、Viewer、Guardian ID | 入口原文只在当前提交中使用；成功后清理 fragment/input，不显示 raw key |
| 读取 workspace | `GET /api/v1/portal/workspace` | `case{customer_stage,last_customer_visible_update_at}`、`schools[]`、`messages[]`、`action_items[]`；仅 allowlisted customer-visible 字段 | 当前 PortalSession 绑定的单一 Case；每次请求重新授权；`Cache-Control: private, no-store` |
| 刷新 workspace | 同一 `GET /api/v1/portal/workspace` | 不新增客户端状态或查询参数；以最新白名单 DTO 替换当前内容 | 不接受 Case ID 切换；错误时不使用旧内容伪装成功 |
| 退出 | `POST /api/v1/portal/session/logout` | 当前 PortalSession 失效并清除 Cookie；返回 signed_out | 退出后回 `/portal/access`；不进入 ERP |

Portal 不提供以下 endpoint 或页面命令：

```text
confirm candidate list
decide offer
update Student / Guardian / Case
create / complete / cancel / reassign Task
upload / preview / download / delete / restore Document
read internal Notifications
reply Message / complete ActionItem
```

### 3.2 Workspace DTO allowlist

目标 DTO 只允许以下结构；字段名以 approved Portal API/UI 为准，页面不得自行扩展：

```json
{
  "api_version": "v1",
  "request_id": "opaque-id",
  "data": {
    "case": {
      "customer_stage": "application_in_progress",
      "last_customer_visible_update_at": "UTC timestamp"
    },
    "schools": [
      { "name": "approved school display", "status": "customer-facing status" }
    ],
    "messages": [
      { "body": "Advisor-approved customer-visible text", "published_at": "UTC timestamp" }
    ],
    "action_items": [
      { "title": "customer-visible item", "deadline": "UTC timestamp or null", "completed": false }
    ]
  }
}
```

禁止字段包括 Case number、Student/Guardian 姓名、email、phone、Assessment、内部备注、内部 Task/Assignee、文件/文件名/capability、候选草稿、驳回理由、AuditEvent、员工信息、Portal key、Session secret 和任何未列入 allowlist 的字段。旧实现返回 `case_number` 只作为差异证据，不能进入目标 DTO。

### 3.3 Grant/Session 请求时重验

兑换和 workspace read 均重新检查：

1. Grant/PortalViewer/Guardian relationship 与当前组织仍有效；
2. Grant 绑定同一 Case、状态 active、expires_at 未到期；
3. 签发人仍是当前 active Primary Advisor（Founder 可撤销不改变 Guardian 角色）；
4. Case 仍为可对客的 active Case；
5. PortalSession status、idle expiry、absolute expiry 和 capability set version 有效；
6. active session 数量不超过 3；
7. 对客查询只返回 customer-visible allowlist，不复制内部 projection。

Case 暂停不自动延长 Grant；Case closed/cancelled/pending_delete、Grant revoked/expired、Viewer relationship 失效、签发人失权或 Session 到期都会在下一请求失效。Platform Billing 的 `past_due` 不得成为 Portal 页面或权限条件。

### 3.4 Public error mapping

| HTTP/公开 code | 页面文案与动作 | 不得泄露 |
| --- | --- | --- |
| `400 PORTAL_REQUEST_INVALID` | 输入无效；返回 access 页面重新输入 | 解析细节、字段栈、Grant/Case 信息 |
| `401 PORTAL_ACCESS_INVALID` | “目前无法使用此案件门户”；提供返回访问页 | key 错误、过期、撤销、Session invalid、relationship 或 Case 原因 |
| `403 PORTAL_ACCESS_DENIED` | 统一无权/无法使用；返回访问页 | 目标是否存在、内部角色、组织信息 |
| `409 PORTAL_CONFLICT` | 访问状态已变化；重新载入/重新兑换 | 内部版本、审计、授权记录 |
| `429 PORTAL_RATE_LIMITED` | 请稍后再试；不增加输入尝试 | 限流算法、计数、账户存在性 |
| `503 PORTAL_UNAVAILABLE` | 服务暂不可用；重试或回访问页 | provider、数据库、stack、cookie secret |

所有响应 `no-store`；错误页面不保留 workspace payload、raw key 或任意长效 token。`PORTAL_ACCESS_INVALID` 既用于 access redeem 失败，也用于 workspace 读取时的失效 Session，保持统一失败语义。

### 3.5 Loading、empty、denied、stale、error、success

| 页面/状态 | loading | empty | denied/expired/revoked | unavailable/error | stale/conflict | success |
| --- | --- | --- | --- | --- | --- | --- |
| `/portal/access` | 禁用提交按钮，显示正在验证 | 初始空表单；不代表无 Case | 统一入口无效，清空输入，允许重新输入 | rate limited/503 显示安全重试 | 不在客户端解释版本；重新兑换 | 只显示 active/expiry 摘要或直接进入 workspace |
| `/portal/workspace` | 只显示安全骨架，不猜 Case 内容 | 白名单 DTO 成功但 schools/messages/action_items 均空；显示暂无可见进度 | 统一入口无效，清除内容，返回 `/portal/access` | 503/未知错误显示暂不可用；不使用旧缓存 | 无 Portal 写入；若返回 `PORTAL_CONFLICT` 重新 GET，不本地合并 | 只显示最新 allowlist、香港时区格式化时间和退出/刷新 |
| workspace refresh | 按钮进入 loading，避免重复请求 | 新 DTO 仍为空即 empty | Session 失效转 access invalid | 保留安全空壳，不保留敏感旧内容 | 完整替换 DTO，不按数组 index 合并 | 更新时间和列表以 server response 为准 |
| logout | 按钮禁用，等待 cookie clear | 已退出回 access | 已失效也清 Cookie 并回 access | logout 503 仍清理本地页面状态，显示可重新进入 | 不重放 workspace | `signed_out` 后 `/portal/access` |

## 4. F4、F3 与内部 ERP 连续性

### 4.1 Grant 管理回到内部 ERP

```text
内部员工登录
  -> /cases/:caseId/access
  -> Primary Advisor 生成单 Case Grant
  -> raw key 只显示一次，交给 Guardian
  -> Guardian /portal/access 兑换
  -> Portal /portal/workspace 只读
  -> logout/失效 -> /portal/access
```

F5 不设计 `/cases/:caseId/access` 的内部按钮或 Grant API；它只保证 Portal 入口的返回路径和失效语义与 F0/F1 一致。Primary Advisor 重新生成时创建新入口，不延长旧 Grant；Founder 可紧急撤销。

### 4.2 F3 业务事实不可被 Portal 改写

| F3 事实 | Portal 表现 | 真正写入入口 |
| --- | --- | --- |
| Founder-approved candidate list | 只可在对客学校进度中显示已批准展示的学校/状态 | 内部 Cases；Primary Advisor 代录 Guardian list decision |
| Guardian list confirmation | Portal 不提供 confirm/not_confirmed 按钮 | Primary Advisor 通过电话/微信/面谈在 `/cases/:caseId/schools` 代录 |
| SchoolTarget application result | 只显示 customer-facing application status | Cases/Tasks 内部命令和重验 |
| accepted offer | Portal 不提供 offer_confirmed/offer_declined | Primary Advisor 在 Applications 代录实际 Guardian 决定 |
| action item | 只显示标题、deadline、completed | 若有业务确认，由内部 Advisor/owning module 按已批准规则处理；Portal 无 complete API |

### 4.3 F4 文件和通知边界

- Documents 工作台只在内部 ERP `/documents` 或 `/cases/:caseId/documents`；Portal 不返回文件、预览、下载、上传、删除、restore 或 legal hold 控件。
- Notifications 只服务内部 User；Portal 没有 `/notifications`、unread count、read 或 resolve-target。
- Portal 不能使用内部 Notification target、内部 Cookie、Access role、ScopeGrant 或 Contractor Task DTO。
- F4 的“家长需要查看的信息”只使用 Portal `action_items` allowlist；不把内部 Task 或通知正文复制到 Portal。

## 5. 当前源码盘点与目标差异

以下只描述当前 Tianxingguoji worktree 的源码证据，不覆盖 confirmed BR 或 approved Portal API/UI。

| 当前路由/源码证据 | 当前实现 | F5 目标分类 |
| --- | --- | --- |
| [`/portal/access`](</Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/(portal)/portal/access/page.tsx:1>) | 输入 `access_key`，调用旧 `/api/v1/portal/sessions` POST；区分 401/503，并跳转 `/portal/workspace` | `redesign`：改用 approved `/api/v1/portal/access/redeem`，fragment 清理、统一 invalid/rate-limit/unavailable、最小 copy |
| [`/portal/workspace`](</Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/(portal)/portal/workspace/page.tsx:1>) | 读取旧 DTO，显示 `case_number`、stage、学校、action items、messages，并使用 `/api/v1/portal/sessions` DELETE 退出 | `redesign`：目标 DTO 移除 Case number/PII，使用 approved logout path，补齐 expired/revoked/stale/error 与只读边界 |
| [`app/api/v1/portal/sessions/route.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/portal/sessions/route.ts:1) | 当前 POST/DELETE 合并兑换和退出，设置 Portal cookie；是旧路径证据 | `redirect/isolate_from_release1`：目标分离为 approved access/redeem 与 session/logout；本轮不改源码 |
| [`app/api/v1/portal/workspace/route.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/portal/workspace/route.ts:1) | 当前 GET 使用 `case_number` 等旧 response 字段和 runtime adapter | `redesign`：目标只返回 Portal allowlist，统一 envelope/no-store/error mapping、请求时重验 |
| [`app/api/v1/portal/workspace/handler.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/portal/workspace/handler.ts:1) | 当前 handler 已有 no-store、session cookie 读取和 503/401 映射，但直接序列化旧 DTO | `redesign`：保持 no-store，按 approved `data.case/schools/messages/action_items` 白名单输出 |
| [`app/api/v1/portal/sessions/handler.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/api/v1/portal/sessions/handler.ts:1) | 当前访问输入最小长度、Cookie 清理和 signed_out receipt；公开 code 与 approved path 不完全一致 | `redesign`：以 approved public error `PORTAL_ACCESS_INVALID/PORTAL_RATE_LIMITED/PORTAL_UNAVAILABLE` 对齐 |
| [`modules/external-portal/domain/contract.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/modules/external-portal/domain/contract.ts:1) | 当前 policy 已有 Grant/Session/Case/issuer/relationship/TTL/active-session 规则，但类型仍含历史 `data_reviewer`、billing status 和 `caseNumber` | `keep as evidence`：F5 不新增角色/状态；目标 UI 只采用 approved PortalViewer allowlist，Platform Billing 排除 |
| [`modules/external-portal/domain/policy.ts`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/modules/external-portal/domain/policy.ts:69) | 当前 policy 请求时检查 Grant、Session、Viewer、Case、issuer、organization，并明确 past_due 不影响 access | `keep as evidence`：目标继续 fail closed；past_due 不进入 Portal 页面或授权条件 |
| [`app/globals.css`](/Users/karo/.codex/worktrees/a09c/Tianxingguoji/app/globals.css:24) | 当前 Portal 使用独立 shell/panel/grid 和响应式 CSS | `redesign`：F5 只保留独立 Portal layout 原则，不复制视觉色板或营销卡片堆叠 |
| `app/**` Portal route inventory | 当前有 `/portal/access`、`/portal/workspace`，没有 approved `/portal/access/redeem` 页面路由（它是 API command） | 页面 `/portal/access`、`/portal/workspace` `keep/redesign`；API path 对齐由后续实现处理 |
| `modules/platform-billing/infrastructure/portal-billing-production.ts` | 当前仓库有历史 Portal/Billing 适配器文件 | `isolate_from_release1`：不读取、不导航、不显示、不作为 Portal access predicate |

当前源码中的 `case_number`、`/portal/sessions`、旧 DTO 字段和历史 `data_reviewer` 仅是现状证据；不能覆盖 BR-060/063 或 approved Portal contract。

## 6. 桌面、移动和可访问性原则

- Portal 不使用内部 ERP 侧栏、角色切换、通知图标或任意外部跳转；页面只保留清晰的刷新、返回访问页和退出操作。
- 桌面端以单一受约束 workspace 布局呈现 Case 摘要、学校、行动项、消息；不做营销落地页，不使用装饰性卡片堆叠。
- 移动端单列显示，操作项固定在可见区域；长消息换行，不能遮挡退出、返回或错误文案。
- 状态不只靠颜色：loading、empty、invalid、expired/revoked、unavailable、success 使用文字与可读标签；Portal 不展示内部 error code 原文。
- 所有按钮和链接有可读文字/aria-label；访问入口输入可键盘操作；失败后焦点回到入口字段或错误摘要。
- `no-store`、CSP、`Referrer-Policy: no-referrer`、禁止第三方追踪和 frame 约束属于 approved Portal 安全边界；本轮不实现配置。

## 7. 追溯、验收与交付摘要

### 7.1 事实来源

- [confirmed BR-060/BR-063 Portal](../../../business-requirements/80-portal-billing.zh-CN.md)
- [approved External Portal API/UI](../50-portal-workspace.zh-CN.md)
- [approved Authorization model](../../40-permissions-security/10-authorization-model.zh-CN.md)
- [approved F0 信息架构](./10-information-architecture.zh-CN.md)
- [approved F1 Today/Cases/Workspace](./20-f1-today-cases-wireframes.zh-CN.md)
- [approved F2 CRM/Case intake](./30-f2-crm-case-intake-wireframes.zh-CN.md)
- [approved F3 Schools/Applications/Interviews](./40-f3-school-selection-application-interview-wireframes.zh-CN.md)
- [approved F4 Documents/Notifications](./50-f4-documents-notifications-wireframes.zh-CN.md)

### 7.2 Architect 验收对照

| 门禁 | 本文证据 | 结果 |
| --- | --- | --- |
| Portal 单 Case、只读、白名单 | §0.1、§0.3、§2.1、§3.2、W03–W04 | 满足 |
| Grant/PortalSession 入口与失效 | §1.2–§1.4、§3.3–§3.4、W01–W02/W05–W06 | 满足 |
| expired/revoked/unavailable | W05–W06、§3.4、§3.5；具体原因统一不泄露 | 满足 |
| Guardian/Student 不写内部业务 | §0.3、§3.1、§4.2；没有任何 Portal write API | 满足 |
| F3 内部代录连续性 | §4.2；名单确认和 offer 决定仍由 Primary Advisor 代录 | 满足 |
| F4 文件/通知隔离 | §4.3；无 Documents/Notifications/Portal capability | 满足 |
| 最小数据与隐私 | §2.1、§3.2；不显示 PII、文件、内部 Task/备注 | 满足 |
| loading/empty/denied/stale/error/success | §3.5；Portal 无写入，stale 以重读/冲突安全处理 | 满足 |
| 当前源码与目标区分 | §5；旧 path/DTO 只作现状证据 | 满足 |
| Platform Billing 排除 | §0.3、§3.3、§5；`past_due` 不影响 Portal access | 满足 |

### 7.3 交付字段

| 字段 | 值 |
| --- | --- |
| `status` | `approved` |
| `owner` | `frontend-design` |
| `scope` | F5 Guardian Portal workspace 低保真线框及交互契约 |
| `changed_files` | 仅新增本文件；产品源码、测试、数据库、migration、云配置均未改 |
| `wireframe_count` | 6（W01–W06：Access/Workspace/失效态，桌面与移动） |
| `decision_count` | 0（未新增角色、实体、状态、API 或业务规则；复用 approved Portal contract） |
| `review_readiness` | `ready_for_architect_review` |
| `browser` | `not_run` |
| `database` | `not_run` |
| `cloud` | `not_run` |
| `product_tests` | `not_run` |

F5 文档完成，状态为 `approved`；允许按本契约进入前端实现，不扩展超出 Portal workspace 范围的设计。
