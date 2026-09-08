# Shared 与入口适配层领域模型与数据设计

状态：`approved`  
确认依据：项目负责人于 2026-08-25 接受 Shared 唯一幂等表、受信 actor scope、统一入口 envelope 与 Release 1 registry 设计  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[领域模型与数据设计索引](README.md)。

业务依据：[Shared 与入口适配层契约](../10-module-contracts/110-shared-entry-adapters.zh-CN.md)。

## 1. 先看结论

Shared 只保存一类数据：`IdempotencyRecord`（幂等记录）。它记录“同一个已授权命令是否已经执行过”，防止浏览器重试、Worker 重复投递或 Portal 重复提交造成重复副作用。

入口适配层（Route Handler、页面 client、Portal 入口、Worker entrypoint）不拥有业务数据，因此不新增业务表。它们只做输入解析、身份上下文建立、调用业务模块公开入口和响应映射。

本模块明确不建立：通用 BusinessEntity、Workflow、Approval、Role、Capability、Permission、Decision、旧 `DEC-*` 表、跨模块状态表、Platform Billing 表。

## 2. 领域对象与 owner

| 对象 | 类型 | owner | 生命周期 |
| --- | --- | --- | --- |
| `IdempotencyRecord` | 持久化技术记录 | Shared | 创建后只允许从 `in_progress` 进入 `completed` 或 `failed`；不物理删除 |
| `RequestContext` | 运行时值对象 | Shared | 每次请求/事件创建；请求结束即失效，不落库 |
| `ApiSuccess` / `ApiError` | 运行时响应 envelope | Shared | 由入口创建并返回，不落库 |
| `ModuleDefinition` | 构建期架构 metadata | Shared | 随代码版本发布，不落库 |

Shared 不保存业务事实。Case、Task、Document、Notification、Portal 等事实仍由各自模块保存；Shared 只提供技术原语。

## 3. 关系与幂等边界

```text
受信入口
  -> RequestContext + 实际 actor scope
  -> Shared.IdempotencyRecord（锁定/重放/冲突）
  -> owning module 写业务事实
  -> AuditEvent / OutboxMessage（如适用）
  -> 完成 IdempotencyRecord
```

- 同一 `organization_id + actor_kind + actor_opaque_id + operation + idempotency_key` 只能有一条记录。
- 相同作用域和 key、相同 `request_hash`：返回第一次命令的稳定结果，不重复写入。
- 相同作用域和 key、不同 `request_hash`：返回 `IDEMPOTENCY_KEY_REUSED`，不执行第二次。
- 首次请求仍为 `in_progress`：返回 `IDEMPOTENCY_IN_PROGRESS`，不并行执行。
- 幂等记录与业务事实、必需 AuditEvent、OutboxMessage 在同一事务中提交。
- 幂等 key 只是重试标识，不是登录凭证或授权凭证；每次重试都要重新检查授权和业务前置条件。

## 4. 目标逻辑表：`shared_idempotency_records`

这是 Shared 的唯一目标持久化表。字段说明中的“现状”只描述当前代码/迁移证据，不代表目标已实现。

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` (`uuid`) | 是 | 已有 | 服务端生成的 opaque 主键；不能作为授权凭证 |
| `organization_id` (`uuid`) | 是 | 已有，外键指向 `access_organizations` | 租户边界；幂等记录必须归属一个业务组织 |
| `actor_kind` (`text`) | 是 | **新增目标字段**；当前没有 | 实际调用者类型：`user`、`portal`、`worker`、`system`；不能填写角色名 |
| `actor_opaque_id` (`text`) | 是 | **新增目标字段**；当前用 `actor_user_id` 代替 | 实际调用者的受控 opaque 引用：用户 UUID、Portal Viewer/Grant 引用或 Worker/Job/Run 引用；不保存 Guardian 姓名或联系方式 |
| `operation` (`text`) | 是 | 已有 | 稳定的命令/操作 code，例如 `students.create`；同一 operation 必须使用同一输入规范 |
| `idempotency_key` (`text`) | 是 | 已有 | 客户端或 Worker 生成的重试标识；应有长度和字符集上限，不能承载 token/PII |
| `request_hash` (`char(64)`) | 是 | 已有 | 对“已验证且标准化的 command”做 canonical JSON 后计算的 SHA-256；不同输入不得复用同一 key |
| `state` (`text`) | 是 | 已有 | 允许值：`in_progress`、`completed`、`failed`；只允许单向完成，禁止回退 |
| `result_reference` (`text`) | 否（完成时是） | 已有 | 第一次执行产生的 opaque receipt/结果引用；不保存完整 DTO、请求正文或 PII |
| `response_hash` (`char(64)`) | 否（完成时是） | 已有 | 对受控响应结果的 hash，用于稳定重放和完整性校验 |
| `record_version` (`bigint`) | 是 | 已有，默认 `1` | 乐观并发控制；每次合法更新递增 1 |
| `created_at` (`timestamptz`) | 是 | 已有 | UTC 创建时间；服务端生成 |
| `updated_at` (`timestamptz`) | 是 | 已有 | UTC 最后更新时间；服务端生成 |

### 4.1 目标约束与索引

- 唯一约束改为：`(organization_id, actor_kind, actor_opaque_id, operation, idempotency_key)`。
- `actor_kind` 只能取受控 machine code；`actor_opaque_id` 必须非空且不能是显示姓名、邮箱、电话、token 或完整请求内容。
- `state = completed/failed` 时必须有 `result_reference` 和 `response_hash`；`in_progress` 时二者必须为空。
- `request_hash`、`response_hash` 必须是固定 64 位十六进制值；`record_version >= 1`。
- `in_progress -> completed/failed` 只能由持有当前版本的事务完成；禁止删除记录和静默覆盖。
- 保留按唯一作用域查找的唯一索引；如后续需要清理窗口，再单独评估 `state/updated_at` 索引，不提前增加无依据索引。

### 4.2 当前实现差异

| 当前证据 | 目标处理 |
| --- | --- |
| 现有表使用 `actor_user_id`，只支持 User actor | 新增 `actor_kind + actor_opaque_id`，以后停止新写入 `actor_user_id`；历史值通过追加 corrective migration 映射为 `actor_kind=user` |
| Portal/Worker 仍有重复或局部幂等结构 | 统一使用 Shared 表；旧结构停止新入口和新写入，历史保留，迁移窗口另行设计 |
| 旧 Shared domain 仍有 `DEC-*` 和跨模块状态迁移辅助逻辑 | 从 active contract 移除；不迁入本表，也不新增替代表 |
| 当前实现/迁移可能没有覆盖所有终态和删除保护 | 用新的 corrective migration 补齐约束、触发器和唯一索引；不修改旧 migration |

## 5. 不落库的运行时对象

### 5.1 `RequestContext`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `requestId` | 是 | 已有服务端生成 | 本次 HTTP/事件的 opaque 关联 ID；返回 `request_id` 并用于脱敏审计关联 |
| `receivedAt` | 是 | 已有服务端 Clock | UTC 接收时间；客户端不能覆盖 |
| `correlationId` | 否 | 契约已定义，当前实现需补齐 | 跨 outbox/Worker 链路关联；只接受受信传播值 |
| `causationId` | 否 | 契约已定义，当前实现需补齐 | 标识产生当前命令/事件的受信来源；普通浏览器请求可为空 |

不保存 Cookie、token、IP、User-Agent、完整 URL/query/body、姓名、邮箱、电话或其他 PII。

### 5.2 API envelope

正式 `/api/v1/**` 只返回以下两种结构之一：

```json
{
  "api_version": "v1",
  "request_id": "opaque-id",
  "data": {}
}
```

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

`data` 和 `error` 不能同时出现。私有 API 默认 `Cache-Control: no-store` 并返回 `X-Request-Id`。错误详情只允许安全的当前版本、diff token 或 readiness 状态；不能返回 stack、SQL、provider 输出、路径、secret、token、PII 或完整请求。

通用 HTTP 分类为：`INVALID_REQUEST`、`UNAUTHENTICATED`、`FORBIDDEN`、`NOT_FOUND`、`METHOD_NOT_ALLOWED`、`CONFLICT`/`STALE_VERSION`、`VALIDATION_FAILED`、`RATE_LIMITED`、`INTERNAL_ERROR`、`SERVICE_UNAVAILABLE`。业务模块负责将自己的具体错误映射到这些公共分类。

## 6. 入口适配层：不新增持久化表

| 入口 | 身份/范围来源 | 允许做什么 | 明确禁止 |
| --- | --- | --- | --- |
| 内部 ERP 页面/API | Identity Session + Access 当前 AuthorizationContext | 解析请求，调用 owning module 的 query/command，映射 DTO | 信任前端 organization/role/actor；直接 SQL；自行推进业务状态 |
| Guardian Portal | 独立 Portal Session/Grant/Viewer scope | 调用 External Portal 白名单查询或 session command | 转换为内部 User；读取内部角色/Session；写 Case/CRM/文件 |
| 内部认证入口 | Identity 的 cookie/header adapter | 调用 Identity/Access 受控认证和 onboarding 用例 | 在 auth route 决定业务角色或权限 |
| Worker | 受信 worker identity + committed outbox/event | 验证消息 schema，按 effect/event identity 幂等调用指定消费者 | 信任消息中的 role/permission；直接写任意 repository；未提交就 ack |
| Health/Readiness | 独立 technical policy | 返回受控 dependency code | 返回客户数据、连接串、bucket、queue、账号或 secret |

所有入口都遵循：`RequestContext -> 结构解析 -> 身份/范围解析 -> owning module 重验 -> 显式 DTO/error -> API envelope`。入口只是适配器，不是业务 owner。

## 7. 模块根入口与运行模式

### 7.1 根级文件职责

| 文件 | 用途 | 限制 |
| --- | --- | --- |
| `public.ts` | runtime-neutral types、稳定 code、公开 command/query contract | 不得依赖数据库、Next.js、secret 或 `server-only` |
| `server.ts` | server-only application service、transaction port、composition factory | 浏览器不得导入；不得提供无授权通用 repository |
| `client.ts` | typed `/api/v1/**` client、输入 normalization、envelope/DTO decoder | 不含业务状态机、server secret 或数据库 |
| `app/**`、`components/**`、`workers/**` | HTTP、页面和 Worker 适配 | 只能调用模块根级公开入口，不拥有领域实体 |

### 7.2 Release 1 active registry

active registry 只包含：`shared`、`identity`、`access`、`crm`、`schools`、`cases`、`tasks`、`documents`、`notifications`、`audit`、`operations`、`external_portal`。`Platform Billing`、`Future` 及旧 Subscription/Merge/重建入口不进入 Release 1 runtime、权限或导航。

运行模式必须显式选择：`local-synthetic`、`test-database`、`production-aws`。组合不完整或依赖不可用时 fail closed，禁止静默回退到 memory、JSON、preview、mock、legacy API、其他 region 或开发凭证。

## 8. 验收标准

请确认以下 6 点：

1. Shared 只有 `shared_idempotency_records` 一张目标持久化表，入口适配层不建业务表。
2. 幂等唯一作用域采用 `organization_id + actor_kind + actor_opaque_id + operation + idempotency_key`，不再以角色或仅 `actor_user_id` 识别调用者。
3. 幂等记录不保存完整请求、响应、token 或 PII，并与业务事实/Audit/Outbox 原子提交。
4. `RequestContext` 只包含 `requestId`、`receivedAt`、`correlationId`、`causationId` 等技术上下文，不落库。
5. 正式入口统一 `/api/v1/**` envelope；页面、Portal、Worker 都只能通过所属模块公开入口访问业务。
6. active registry 和运行模式按 Release 1 范围执行，不包含 Platform Billing、Future 或旧 DEC/跨模块状态实体。

本模块确认后，11 个模块的数据设计全部完成。下一步按系统设计顺序进入“业务流程与状态机设计”，之后再冻结权限、安全和 API/UI 交互。
