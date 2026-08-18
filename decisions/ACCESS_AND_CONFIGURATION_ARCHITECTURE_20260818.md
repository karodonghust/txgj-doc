# 统一权限与配置治理架构

| 项目 | 决策 |
| --- | --- |
| 状态 | `accepted_for_planning` |
| 日期 | 2026-08-18 |
| 决策编号 | `DEC-069` |
| 实现票据 | `P2-13`、`P2-14`、`P2-15` |
| 当前授权 | 允许设计、开发计划和本地源码实现；不授权生产配置、云资源、部署或真实数据 |

## 1. 问题

当前系统已经把 User、Membership、RoleBinding、Primary Advisor、TaskAssignment、
CaseCollaborator 和 ScopeGrant 保存为权威数据，但仍有三类分散的硬编码：

1. `OrganizationRole -> WorkspaceCapability` 矩阵以及不少 API/Service 的允许角色写在代码中；
2. 页面导航没有消费 capability，页面登录检查和服务端授权也没有统一入口；
3. session 时长、重试、区域、时区、展示字典和日期等配置分散在领域、基础设施和组件中。

结果是同一权限变更可能同时修改菜单、页面、Route Handler 和 Service，而且其中任何遗漏
都会造成界面与服务端行为不一致。把所有常量直接放进一个可任意编辑的后台表同样不可接受，
因为租户隔离、资源归属和数据库权限不能成为普通运行配置。

## 2. 决策

系统采用四层配置治理，并为每个配置项登记 owner、source、validation、reload 和 audit 规则。

| 层级 | 例子 | 权威来源 | 变更方式 |
| --- | --- | --- | --- |
| 安全不变量 | organization/RLS、资源归属、Primary Advisor、敏感 scope、错误隐藏 | 代码 + PostgreSQL constraint/function/RLS | 追加 migration、代码审查、负向测试 |
| 版本化业务策略 | role-capability 关系、流程模板、表单 manifest | PostgreSQL immutable versioned policy | draft -> approved -> active；带 hash、actor、时间和 audit |
| 部署运行配置 | region、endpoint、timeout、retry、bucket、feature flag | 类型化环境配置；生产由受管 secret/config 注入 | 启动时校验，缺失或非法时 fail closed |
| 组织展示配置 | timezone、locale、品牌和受控展示字典 | PostgreSQL organization settings 或版本化 catalogue | 明确 owner、默认值、版本和 fallback |

不得使用单个通用 key-value 表承载以上四类配置，也不得让浏览器、JWT claim、菜单或缓存成为
授权真相。

## 3. 权限模型

### 3.1 固定词汇与可版本化关系

- `OrganizationRole`、`WorkspaceCapability`、资源类型、scope、action 和稳定错误码继续是代码契约；
  新词汇需要代码、数据库约束、测试和文档同步。
- role 与 workspace capability 的允许关系改为 immutable policy set。每个 organization 同一时间
  只有一个 active policy version；规则只表达显式 allow，缺失即 deny。
- Release 1 的首个 policy version 由追加 migration 按当前已批准矩阵建立并激活，以保持现有行为。
- Release 1 不提供任意在线编辑器。`/admin/access` 先提供只读 policy version、角色绑定和审计视图；
  后续若增加编辑器，必须作为独立票据设计 approval、lockout prevention 和 rollback。
- policy activation 必须验证 Founder 仍拥有 `access.manage`，manifest hash 匹配，所有 capability 均在
  代码 registry 内，并在同一事务写 activation fact、AuditEvent 和 Outbox。

建议的持久化边界为 `access_policy_sets` 与 `access_role_capability_rules`。已 approved/active 的内容
不可原地修改；替换只能建立新版本。回滚是激活一个已批准的旧版本或纠正版本，不删除历史。

### 3.2 单一服务端决策入口

Access module 提供运行时无关的请求和结果契约，以及 server-only evaluator：

```ts
type AuthorizationRequest = {
  capability: WorkspaceCapability
  resource?: { type: string; id: string }
}

type AuthorizationDecision =
  | { allowed: true; policyVersion: string }
  | { allowed: false; code: AuthorizationDenialCode }
```

- workspace capability 由 active policy version + active RoleBinding 计算。
- case/task/document 等资源权限仍由 owning repository 在同一 transaction 读取当前组织、分配、
  scope、expiry 和 record version 后决定；通用 capability 不替代资源级授权。
- Route Handler 和 Service 不再维护各自的角色数组；迁移后只请求 capability，并继续调用 owning
  repository 的资源授权。
- 已认证但没有 workspace capability 返回稳定 `403`；为防止枚举而隐藏的资源继续返回 `404`。
- 每次敏感请求重新校验 session、membership、role binding 和 active policy version。缓存只能以
  organization + role + policy version 为 key，并且不能延迟 revoke 或 policy activation 生效。

### 3.3 页面与导航

- 建立唯一的 navigation registry，每个项目声明 route、label key、icon 和 required capability。
- `/api/v1/auth/me` 返回服务端计算的 capability snapshot 和 policy version；Sidebar 只使用它隐藏
  无权限入口，不自行推导角色。
- 每个受保护页面或 route group 使用 server-side page guard；客户端 `AppFrame` 只负责交互体验，
  不能作为安全边界。
- 直接 URL、API、猜测 ID、搜索、导出、后台任务和缓存都必须有负向测试；没有按钮不构成证据。

## 4. 运行和展示配置

`P2-14` 建立类型化 `RuntimeConfig` composition root，首批收口 session/invite policy、API timeout、
重试次数、upload intent TTL、object-store region/bucket 和 organization default timezone。领域服务只接收
经过校验的 policy object，不直接读取 `process.env`。安全上限继续由代码约束，环境值只能在批准范围内。

`P2-15` 退出 Today 页固定日期和 preview 数据，把 role/stage/admission/assessment label 迁入受控的
presentation catalogue，并为 organization timezone/locale 提供权威设置。Schema 拥有的字段 label
随 manifest 版本提供；品牌是否可配置不影响授权或业务事实。

## 5. 迁移顺序

1. `P2-13A` 冻结 capability registry、authorization contract、deny/error 语义和当前矩阵 fixture。
2. `P2-13B` 增加 policy tables、bootstrap version、repository 和 activation invariants。
3. `P2-13C` 先迁移 `/api/v1/auth/me`、Sidebar 和 page guard，再逐个迁移 API/Service 角色数组。
4. `P2-13D` 完成五角色页面/API/资源负向矩阵和只读 Access 管理页。
5. `P2-14` 收口运行配置；每个常量迁移都必须证明默认行为不变。
6. `P2-15` 收口时区、日期、preview 数据和展示字典。

在 `P2-13` 完成前，不新增第六个 organization role，也不新增另一套菜单权限判断。

## 6. 验收和回滚

- 五角色的当前允许/拒绝行为在 bootstrap policy 下保持不变。
- UI 隐藏、server page guard、API capability guard 和 repository resource guard 分层测试全部通过。
- 没有 active policy、未知 capability、非法 policy hash、inactive membership/role 或过期 grant 均 fail closed。
- activation、rollback、grant use 和高风险 denial 均可审计且不记录敏感内容。
- `P2-14` 非法或缺失 production 配置阻止启动；local synthetic 只接受明确的本地配置。
- `P2-15` 使用当前 organization timezone 计算日期，页面不再包含固定 2026-08 过滤或 preview 权威数据。

回滚不得恢复分散角色数组。权限策略通过激活上一批准版本回滚；代码问题通过 feature gate 暂停新
policy resolver，但仍必须 fail closed。数据库 migration 保持追加式，不删除 policy history。
