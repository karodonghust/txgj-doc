# Shared 与入口适配层契约

状态：`approved`  
确认依据：项目负责人于 2026-08-25 指示继续进入领域模型与数据设计  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v31`

返回[模块契约索引](README.md)。

业务依据：`BR-001`、`BR-002`、`BR-003`、`BR-070`、`BR-071`。  
架构依据：[Release 1 总体架构设计](../00-overall-architecture.zh-CN.md)。  
代码参考：`modules/shared/**`、`app/**`、`components/**`、`workers/**`、`lib/api/client.ts`、`tests/architecture/module-boundaries.test.ts`。

## 1. 一句话职责

- **Shared** 提供不含具体业务含义的请求、响应、幂等和数据库事务技术原语。
- **入口适配层** 把 HTTP、页面或 Worker 输入翻译为已授权的模块查询/命令，再把结果翻译回对应入口。

两者都不拥有业务流程。Shared 不是“暂时不知道放哪里就放进去”的模块。

## 2. 负责与不负责

| Shared 负责 | Shared 不负责 |
| --- | --- |
| RequestContext、统一 API envelope 和受控错误基类 | User、角色、Capability 或 AuthorizationContext |
| IdempotencyRecord 及通用判定规则 | Case、Task、Document、Portal 等业务状态 |
| 通用 PostgreSQL transaction runner 和运行模式校验 | 任何业务表、业务 repository 或跨模块写入 |
| 模块 registry 和架构边界检查所需 metadata | 旧 DEC 清单、业务状态迁移或产品范围决定 |

| 入口适配层负责 | 入口适配层不负责 |
| --- | --- |
| HTTP method/path/header/body 的解析与响应映射 | 承载业务状态机或审批规则 |
| 调用 Identity、Access、Portal 或 Worker 的受信上下文解析 | 信任客户端传入的 organization、actor、role 或 scope |
| 调用模块根级公开入口 | 直接访问模块内部 repository 或业务表 |
| 页面交互、typed browser client 和 Worker composition | 创建第二份业务规则或绕过服务端授权 |

## 3. 最小对象

Shared 唯一持久化对象是：

| 对象 | 含义 | 关键约束 |
| --- | --- | --- |
| `IdempotencyRecord` | 一次可重试命令的技术执行记录 | 按组织、实际 actor、operation 和 key 隔离；不保存完整请求、响应或 PII |

以下只是运行时值或构建期 metadata，不是数据库业务实体：

- `RequestContext`：服务端 request ID 和接收时间。
- `ApiSuccess` / `ApiError`：统一 JSON envelope。
- `ModuleDefinition`：架构测试使用的模块 owner、source root 和公开入口清单。
- transaction runner、Clock、ID generator：可替换技术依赖，不持久化为业务对象。

明确不建立：

- 通用 BusinessEntity、通用 Workflow 或通用 Approval。
- Shared Role、Shared Capability 或 Shared Permission。
- Shared Decision、旧 `DEC-*` 状态表或跨模块状态迁移清单。
- 每个模块各自重复的 IdempotencyRecord。

业务状态迁移由拥有该事实的模块定义；产品范围由已确认 `BR-*` 和本系统设计决定。

## 4. 受信请求上下文

`RequestContext` 只包含技术事实：

| 字段 | 来源 | 规则 |
| --- | --- | --- |
| `requestId` | 服务端生成 | 安全 opaque 值；返回响应并用于审计关联 |
| `receivedAt` | 服务端 Clock | UTC 时间；客户端不能覆盖 |
| `correlationId` | 服务端创建或从受信 outbox 传播 | 只允许安全 opaque 值；不接受任意外部文本 |
| `causationId` | 当前命令/事件的受信来源 | Worker 关联原 outbox/event；普通请求可为空 |

`RequestContext` 不保存 Cookie、token、IP、User-Agent、完整 URL、query、body、姓名或联系方式。

业务调用同时携带但不由 Shared 拥有：

- 内部请求：Identity 的当前 Principal + Access 的最新 AuthorizationContext。
- Portal 请求：External Portal 当前 Session/Grant scope；绝不转换成内部 User。
- Worker 请求：受信 worker identity + committed outbox/event identity。

organization 和实际 actor 必须从以上受信事实解析，不能从浏览器 body、query 或自定义 header 接受。

## 5. 幂等规则

### 5.1 适用范围

- 所有可能产生业务写入或副作用、且可能被浏览器/网络重试的命令都必须幂等。
- 内部 HTTP 写命令使用 `Idempotency-Key` header；同一次用户提交的网络重试复用同一 key。
- Worker 使用 outbox/event identity 派生稳定 effect key。
- Portal 的 Grant 管理使用内部 User actor；Portal Session 兑换使用 Portal Viewer/Grant 的 opaque actor scope。
- 纯 GET/query 不创建 IdempotencyRecord。

### 5.2 唯一作用域

目标唯一作用域为：

```text
organization_id + actor_kind + actor_opaque_id + operation + idempotency_key
```

- `actor_kind=user` 时必须引用实际 User，不能只写角色。
- `actor_kind=portal` 时只保存 Portal Viewer/Grant opaque 引用，不保存 Guardian 资料。
- `actor_kind=worker/system` 时必须引用受控 worker/job/run identity 和 causation ID。
- key 只负责识别重试，不是授权凭证；每次重试仍重新判断当前授权和业务前置条件。

### 5.3 请求 hash 与结果

- 先把已验证、已标准化的 command 按稳定 JSON 规则 canonicalize，再计算 request hash。
- hash 包含 operation、目标 opaque ID、业务输入和 expected version；不包含 Cookie、token 或显示文案。
- 相同 scope/key + 相同 hash：不再次产生副作用，返回第一次的稳定结果。
- 相同 scope/key + 不同 hash：返回稳定 `IDEMPOTENCY_KEY_REUSED` conflict。
- 首次请求仍在执行：返回稳定 `IDEMPOTENCY_IN_PROGRESS` conflict，不并行执行第二次。
- record 只保存 result reference 和 response hash；业务 DTO/PII 不复制到 Shared。

### 5.4 原子性

```text
开始数据库事务
  -> 解析/锁定 IdempotencyRecord
  -> 重验授权、expected version 和业务前置条件
  -> 写本模块业务事实
  -> 写必需 AuditEvent
  -> 如有异步 effect，写 OutboxMessage
  -> 完成 IdempotencyRecord
提交
```

- 任一步失败，事务整体回滚；不能留下“业务已写入但幂等结果丢失”。
- 可重试的基础设施失败不保存为永久 terminal failure。
- 已提交后响应丢失，重试必须取得第一次 command receipt，不能重新执行。
- Release 1 不清理 IdempotencyRecord；后续若增加清理策略，不得早于允许重试和关联 Audit/Outbox 的保留窗口。

## 6. 通用数据库事务能力

Shared 可以提供 server-only transaction runner，但必须满足：

- 每个租户事务显式设置 organization、实际 actor 和 request/correlation context。
- PostgreSQL RLS context 只在当前事务有效，连接归还池前不能泄露到下一请求。
- Route Handler、页面、组件和 browser client 永远不能取得 Pool、Client 或原始 SQL 能力。
- 业务 repository 仍位于 owning module；Shared 不能通过“通用 repository”写任意业务表。
- 一个同步事务只修改一个业务 owner 的权威事实，并可同时写 Shared Idempotency、Audit 和 Outbox。
- 另一个业务模块的副作用通过其公开 command 或 outbox 消费者完成，不在 Shared 中做跨模块 SQL。

## 7. 统一 API envelope

正式 `/api/v1/**` JSON 响应统一为：

```json
{
  "api_version": "v1",
  "request_id": "opaque-id",
  "data": {}
}
```

失败统一为：

```json
{
  "api_version": "v1",
  "error": {
    "code": "STABLE_MACHINE_CODE",
    "message": "Safe generic message",
    "request_id": "opaque-id",
    "retryable": false,
    "details": {}
  }
}
```

规则：

- 成功只有 `data`，失败只有 `error`，不能两者并存。
- 私有业务 API 默认 `Cache-Control: no-store`，并返回 `X-Request-Id`。
- command 成功返回受控 receipt（opaque ID、record version、必要状态），不直接 spread domain/repository 对象。
- ID 是 opaque 值；时间使用 UTC ISO-8601；状态和枚举使用稳定 machine code。
- 可见中英文文案由 UI i18n 或明确 customer-facing mapping 生成，不把内部异常文字直接返回。
- 精确 endpoint DTO、分页和每项业务 error code 留到“API 与页面交互设计”按纵向流程冻结。

## 8. 稳定错误边界

Shared 只定义通用 HTTP/error category；业务模块负责把自己的受控失败映射到它：

| HTTP | category | 使用边界 |
| ---: | --- | --- |
| 400 | `INVALID_REQUEST` | JSON、header、path/query 结构无效 |
| 401 | `UNAUTHENTICATED` | 内部 Session 缺失/失效 |
| 403 | `FORBIDDEN` | 已认证但当前授权不足；防枚举场景可由模块收敛为 404 |
| 404 | `NOT_FOUND` | 目标不存在或调用者不可区分其存在性 |
| 405 | `METHOD_NOT_ALLOWED` | 路由不支持该 method |
| 409 | `CONFLICT` / `STALE_VERSION` | 幂等冲突、并发版本或当前状态冲突 |
| 422 | `VALIDATION_FAILED` | 已解析但不符合受控输入约束 |
| 429 | `RATE_LIMITED` | 入口限流 |
| 500 | `INTERNAL_ERROR` | 未分类故障；不泄露内部原因 |
| 503 | `SERVICE_UNAVAILABLE` | 目标 runtime/dependency 未接通或暂时不可用 |

- `details` 默认空；只有显式 allowlist 的当前版本、安全 diff token 或受控 readiness 状态可返回。
- stack、SQL、provider output、环境变量、文件路径、Cookie、token、secret、PII 和完整输入永不进入响应。
- 未知异常统一映射为 `INTERNAL_ERROR`，同时写脱敏 telemetry；不能把 `error.message` 直接返回。
- Portal 继续使用更严格的统一 `PORTAL_ACCESS_INVALID` 防枚举规则。

## 9. Release 1 入口类型

| 入口 | 身份/范围来源 | 允许调用 | 禁止 |
| --- | --- | --- | --- |
| 内部 ERP 页面/API | Identity Session + Access AuthorizationContext | 对应模块 application query/command | 信任前端角色、直接 SQL |
| Guardian Portal | Portal Session/Grant | External Portal 的白名单 query 和 session command | 内部 User、文件、业务写入 |
| 内部认证入口 | Identity cookie/header adapter | Identity + Access onboarding 的受控用例 | 在 auth route 决定业务角色 |
| Worker | 受信 worker + outbox/event | 指定消费者 use case | 任意调用模块内部 repository |
| Health/Readiness | 独立 technical policy | 无客户内容的 dependency state | 客户数据、业务写命令 |

Release 1 不存在 Platform Billing、Data Reviewer、外部销售、正式导入、AI 或自动 crawler 调度入口。

## 10. 内部 HTTP 请求流程

```text
Route Handler
  -> 创建 RequestContext
  -> 解析 method/path/header/body（只做结构校验）
  -> Identity 解析当前 Session
  -> Access 重新读取 Membership + roles[] + capability
  -> 调用 owning module application service
  -> 模块重验资源关系、expected version 和业务规则
  -> 映射显式 DTO / 稳定错误
  -> 返回统一 envelope
```

- Route Handler 不写 SQL、不发布业务事件、不创建 Notification，也不直接调用 infrastructure repository。
- route contract 可以检查类型、格式、长度和枚举，但业务前置条件必须由 owning module 再判断。
- organization、actor、role、capability 和 Case scope 不从请求 payload 读取。
- UI 隐藏按钮只改善体验，不能替代任何服务端检查。

## 11. Portal 请求流程

```text
Portal Route Handler
  -> 创建独立 RequestContext
  -> 兑换或读取 Portal Session
  -> External Portal 每次重验 Grant/Viewer/Case/issuer
  -> Cases/CRM 公开查询返回 allowlist facts
  -> 映射 Portal 专用 DTO
  -> no-store + 通用拒绝响应
```

Portal adapter 不调用内部 `requireIdentityActor`，也不复用内部 Session Cookie、Access role 或内部业务 DTO。

## 12. Worker 请求流程

```text
Worker entrypoint
  -> 验证 queue/outbox 来源和 schema version
  -> 建立 worker RequestContext + causation/correlation
  -> 按 event/effect identity 幂等 claim
  -> 调用唯一指定的消费者 application service
  -> 写 DeliveryReceipt / 新业务事实 / Audit（按 owner 契约）
  -> ack；失败按有界重试或 dead-letter
```

- Worker payload 只携带 opaque ID、版本、effect code 和受控标量。
- Worker 不信任消息中的角色/permission，也不能用过期 projection 代替当前授权/关系检查。
- `ack` 只在消费者事务提交后发生；重复投递不得重复创建 Task、Notification 或文件状态变化。

## 13. 模块根级入口

| 文件 | 可被谁使用 | 允许内容 | 禁止内容 |
| --- | --- | --- | --- |
| `public.ts` | 任何运行时、其他模块 | runtime-neutral type、受控 code、公开 query/command contract | `server-only`、数据库、Next.js、secret |
| `server.ts` | Route、Worker、其他模块 server code | application service、server-only port/factory | browser import、无授权通用 repository |
| `client.ts` | 页面和组件 | typed `/api/v1/**` client、输入 normalization、响应 decoder | domain 状态机、server secret、数据库 |
| Identity `web.ts` | Next.js auth 入口 | Cookie/header 与 Identity 的窄适配 | 角色/capability 判断 |

- 跨模块 import 只能指向目标模块根级公开入口。
- 同模块依赖保持 `domain <- application <- infrastructure`；domain/application 不反向依赖 infrastructure。
- `app/**`、`components/**` 和 `workers/**` 是入口适配层，不是业务 owner，也不向模块反向暴露契约。
- 浏览器模块需要 HTTP 时必须使用 owning module 的 `client.ts`；页面/组件不直接手写 `fetch` 和独立响应 shape。
- Server Component 可调用 `server.ts`，但仍必须先建立与 Route Handler 等价的身份和授权上下文。

## 14. API 路由与页面边界

- 正式业务 API 只使用 `/api/v1/**`。
- legacy `/api/**`、preview、mock 和兼容 route 不进入目标导航或正式运行时。
- 内部 ERP、Portal 和技术运维路由使用独立 layout/cookie/授权边界。
- 页面只负责显示和交互，不根据本地缓存自行推进 Case、Task、SchoolTarget 或授权状态。
- browser client 必须在运行时 decode envelope 和 DTO；TypeScript type 本身不代表外部响应可信。
- 私有页面必须实现 loading、empty、error、denied 和 unavailable 状态；具体交互在 API/UI 设计阶段冻结。
- 退出旧 route 时先迁移所有调用方和验收，再移除公开入口；历史 migration 不重写。

## 15. 入口安全

- Cookie 身份的业务写请求必须做 same-origin/CSRF 防护；`SameSite` 不是唯一控制。
- 登录、邀请兑换、Portal key 兑换、下载 capability 和其他敏感入口使用独立速率限制。
- route/body 设置明确大小与类型上限；文件内容不经过普通 JSON Route Handler，使用 Documents 的受控 upload intent。
- DTO 使用正向字段 allowlist；禁止 spread domain、database row、provider response 或 Error。
- redirect、return URL 和通知 target 使用受控内部 route code，不能接受任意外部 URL。
- 生产页面和 API 不输出 source map、secret、raw dependency detail 或客户 PII telemetry。

## 16. 运行时组合

| 模式 | 允许 adapter | 规则 |
| --- | --- | --- |
| `local-synthetic` | 本地 PostgreSQL、LocalStack、ClamAV、确定性合成数据 | 只证明 Local Dev |
| `test-database` | 明确测试数据库和测试依赖 | 只用于隔离验证，不代表生产 |
| `production-aws` | 批准的香港 AWS/Cognito/RDS/S3/Queue adapter | 任何 mock/local/preview adapter 都使启动失败 |

- composition root 显式选择模式和 adapter；未配置、组合不完整或依赖错误时 fail closed。
- 禁止自动回退到 memory、JSON、preview、mock、legacy API、其他 region 或开发凭证。
- runtime readiness 只返回受控 dependency code，不返回 hostname、连接串、bucket、queue、账号或 secret。
- Local、Vercel/Preview 和 AWS Production 证据分别记录，不能互相替代。

## 17. 模块 registry 与架构门禁

目标 registry 只登记本 Release 1 已批准模块、公开入口和准确 owner：

- `shared`、`identity`、`access`、`crm`、`schools`、`cases`、`tasks`、`documents`、`notifications`、`audit`、`operations`、`external_portal`。
- `app`、`components`、`workers` 标记为 adapters，不拥有业务实体。
- `platform_billing` 和 `future` 不属于 active Release 1 registry。

架构测试至少阻止：

1. 跨模块 import 内部文件。
2. domain/application 向外依赖 infrastructure。
3. 非 owner 模块写业务表。
4. browser code 导入 `server.ts`、数据库或 server-only 文件。
5. Route Handler 直接 import repository/infrastructure。
6. 正式页面直接调用 legacy/mock/preview API。
7. active registry 再出现已排除模块、角色或旧业务实体。
8. Shared 出现业务状态迁移、角色、审批或产品决定。

## 18. 模块依赖

| 类型 | 允许 |
| --- | --- |
| Shared 依赖 | Node/Web 标准原语、PostgreSQL driver 的 server-only adapter、运行配置 port |
| 业务模块依赖 Shared | Request/API/idempotency/transaction 技术契约 |
| 入口适配层依赖 | Identity/Access/External Portal 及目标业务模块的根级入口 |
| Shared 禁止依赖 | Identity、Access、CRM、Cases、Tasks、Schools、Documents、Notifications、Audit、Operations、Portal 的业务代码 |

Shared 不能因为被所有模块使用而反向 import 任何业务模块。Audit/Outbox 是独立平台模块，不合并进 Shared。

## 19. 当前代码与目标差异

本阶段只记录差异，不修改产品代码或数据库：

| 优先级 | 当前证据 | 目标 |
| --- | --- | --- |
| `P0` | `modules/shared/domain/decision-guards.ts` 保存旧 `DEC-*`、排除项和跨模块状态迁移 | 从 Shared active contract 移除；业务状态机回到 owner，范围以当前 `BR-*`/系统设计为准 |
| `P0` | module registry 仍登记 Platform Billing、Future、Subscription、MergeRevision、旧 Portal/Audit owner | 按 11 份批准模块契约重建 active registry；保留 CRM ReferralSource ownership |
| `P0` | `/api/v1/platform/billing`、重复/合并、reconstruction 等旧入口仍存在 | 从 Release 1 导航、正式 API 和 runtime 隔离；ReferralSource 入口按 CRM 新契约纠正后保留 |
| `P1` | Shared IdempotencyRecord 只支持 `actor_user_id`，Portal/Worker 另建重复幂等结构 | 统一为受控 actor kind + opaque actor scope；用追加迁移纠正，不改旧 migration |
| `P1` | 多个页面/组件直接 `fetch`，Portal 没有 typed `client.ts`，部分仍使用 `/api/auth/**` | 迁移到 owning module browser client 和唯一 `/api/v1/**` envelope |
| `P1` | 多个 runtime 固定 unavailable 或仍有 preview/mock/legacy 组合 | 显式接通目标 adapter；未接通继续 fail closed |
| `P1` | Route Handler 的 parse/auth/error 组合方式不完全一致，部分 DTO 使用对象 spread | 统一入口 pipeline 和显式响应 allowlist |
| `P1` | architecture tests 仍把排除模块视为 active owner | 门禁改为当前 Release 1 registry，并增加 route/browser/Shared 纯技术边界 |

## 20. 设计决策

| ID | 决策 |
| --- | --- |
| `SD-SHARED-001` | Shared 唯一持久化对象是 IdempotencyRecord，不拥有业务实体 |
| `SD-SHARED-002` | Shared 不保存 DEC、角色、Capability、审批或跨模块状态迁移 |
| `SD-SHARED-003` | organization 和实际 actor 只从受信身份/授权/Portal/Worker 上下文解析 |
| `SD-SHARED-004` | 所有可重试业务写命令使用统一幂等作用域，并与业务/Audit/Outbox 原子提交 |
| `SD-SHARED-005` | `/api/v1/**` 和统一 envelope 是 Release 1 唯一正式 HTTP 契约 |
| `SD-SHARED-006` | app/components/workers 只做适配，跨模块只使用根级公开入口 |
| `SD-SHARED-007` | browser client 负责 typed request/response decoding，页面不直接手写 API shape |
| `SD-SHARED-008` | runtime 显式组合且 fail closed，Local/Preview/Production 证据不能混用 |

## 21. 本模块验收标准

项目负责人需要确认：

1. Shared 不承载任何业务规则；唯一持久化对象是 IdempotencyRecord。
2. 所有业务写命令识别 organization 和实际 actor，且不能信任客户端传入这些值。
3. 重试使用统一幂等记录；相同请求不重复副作用，不同 payload 复用 key 必须冲突。
4. app/components/workers 只是入口适配器，不能直接写数据库或模块内部 repository。
5. 正式 API 只保留 `/api/v1/**`，页面通过 typed `client.ts` 或受权 server entrypoint 访问。
6. Shared 不再保存旧 DEC、状态迁移、角色或产品范围清单。
7. production-aws 不得回退到 local/mock/preview/legacy adapter。
8. active module registry 不包含 Platform Billing、Future 或其他 Release 1 排除项。

确认后，11 个模块职责契约全部完成；下一环节进入“领域模型与数据设计”。
