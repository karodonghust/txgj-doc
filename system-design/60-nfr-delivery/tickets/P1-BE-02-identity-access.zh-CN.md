# P1-BE-02 Identity 与 Access 认证授权

状态：`passed`  
Owner：`backend`  
依赖：P1-BE-01

返回[开发票据索引](README.md)。

## 业务结果

内部登录只证明 User 身份；Access 在每个请求解析有效 Membership、兼容多角色和资源关系，角色撤销后立即失权。

## 范围

- 收敛 User/Invite/ProviderBinding/Session 与 EmployeeProfile/Membership/RoleBinding/ScopeGrant 边界。
- 支持 Founder+Admin、Founder+Advisor、Advisor+Contractor；拒绝 Founder+Contractor、Admin+Contractor。
- 冻结 Founder/Admin/Advisor/Contractor 四角色；不创建 Data Reviewer。
- 实现 Session、Invite、reauthentication 和限流参数，值来自[非功能基线](../10-nfr-baseline.zh-CN.md)。
- Admin 单独无客户数据权限；Contractor 只通过当前面试 TaskAssignment 取得脱敏能力。
- 撤销 User/Membership/RoleBinding/Grant 后下一个请求立即拒绝。

## 不在范围

- 不实现 Guardian Portal Session；不发送外部客户 Email；不创建新的员工/角色实体。

## 验收

- 多角色 capability 取允许集合，deny/互斥规则优先；登录不显示 role selector。
- Session 并发、idle/absolute、Invite 一次性、敏感操作 reauth 和锁定策略通过。
- 两个 organization 的 RLS/授权隔离、缓存失效、撤销并发与 Audit 失败回滚通过。
- `pnpm test:auth`、`pnpm test:identity-access-schema`、`pnpm test:architecture`、`pnpm typecheck` 及新增 `test:p1-be-02` 通过。

## 证据与停止条件

数据库与 Local Dev 必须实际运行；browser、Preview、Cognito/AWS 未运行写 `not_run`。发现 Session 固化单一角色、UI 代替服务端授权或生产静默使用 synthetic identity 时停止。

## 回滚

应用回到兼容前一实现；数据库只追加纠正迁移。终态 Session/Invite 不恢复 active，不恢复已撤销 capability。

## Unified Release 1 交付证据（2026-08-27）

| Gate | 状态 | 脱敏证据/边界 |
| --- | --- | --- |
| PostgreSQL 17 | `passed` | 用户在统一主工作区验证：10 tests, 10 pass, 0 fail |
| Browser | `not_run` | 未提供浏览器验收证据 |
| Deployment | `not_run` | 未提供部署验收证据 |
| Cloud capabilities | `not_run` | 未提供云能力验收证据 |
| Unified main-directory SHA | `pending` | 等待统一主目录后续收口绑定 |

本记录只更新交付清单状态，不扩展为浏览器、部署或云端通过声明；上述未运行项仍保持独立门禁。
