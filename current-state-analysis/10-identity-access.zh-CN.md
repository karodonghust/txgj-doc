# Identity 与 Access 现状分析

返回[现状分析总览](README.md)。业务依据：[Identity 与 Access](../business-requirements/10-identity-access.zh-CN.md)。

## 结论

| BR | 状态 | 当前实现摘要 |
| --- | --- | --- |
| `BR-010` | `冲突` | RBAC 骨架存在，但仍有 Data Reviewer，且 Admin 权限越过已确认边界 |
| `BR-011` | `缺失` | 未找到 EmployeeProfile 表、服务或管理页面 |
| `BR-012` | `部分符合` | Scope、Capability、7 天授权和敏感 Scope 审批已建模，但部分运行时未接通 |
| `BR-013` | `冲突` | 数据库允许多 RoleBinding，但 Session 只挑一个角色；权限不合并，Capability 也不继承 |

## 已有基础

- `identity_users`、`access_organization_memberships`、`access_role_bindings` 和 Session 模型已存在。
- `modules/access/domain/contract.ts` 已定义 7 个 Scope、`view/comment/edit`、7 天期限、敏感 Scope Founder 审批及禁止自批。
- 服务端业务服务普遍调用 `evaluateBootstrapAuthorization`；前端按钮不是唯一授权来源。
- `ContractorTaskWorkspaceService` 已定义任务级脱敏 DTO 和失效检查，但正式运行时仍不可用。

## 关键差异

| 优先级 | 当前代码 | 与业务基线的差异 | 建议动作 |
| --- | --- | --- | --- |
| `P0` | `ORGANIZATION_ROLES`、登录页、邀请 API、迁移均包含 `data_reviewer` | Release 1 只有 Founder、Admin、Advisor、Contractor | 从活跃角色、邀请、登录、授权和学校审核路径移除；数据库用新迁移纠正 |
| `P0` | Admin 拥有 Assessment、Student、ReferralSource、School/Crawler 管理能力 | Admin 默认不能读取 ServiceCase 或 Assessment，也不应自动参与客户业务 | 重建 Admin capability 白名单；客户业务必须通过额外 Advisor 角色取得 |
| `P0` | `postgresql-session-service.ts` 按优先级选一个 RoleBinding 并 `LIMIT 1` | 多个兼容角色应自动合并权限 | Session 返回有效角色集合；授权按 capability union 判断 |
| `P0` | `evaluateScopeGrant` 要求请求 capability 与 grant capability 完全相等 | `edit` 应包含 `comment/view`，`comment` 应包含 `view` | 建立明确的 Capability 包含关系并覆盖服务端测试 |
| `P0` | 未见 Contractor 与其他角色互斥约束 | Contractor 不能与任何其他基础角色组合 | 在服务和数据库层同时拒绝不兼容组合 |
| `P1` | 无 EmployeeProfile | 缺 display name、employment type、avatar 和角色资格约束 | 新增一对一 EmployeeProfile 及只读权威 email 投影 |
| `P1` | `modules/access/infrastructure/runtime.ts` 和 Contractor workspace 未完整接通 | 授权模型不能只停留在领域契约 | 接通目标环境 adapter，并以实际请求验证授权即时失效 |

## EmployeeProfile 最小落地范围

只实现 `membership_id`、`display_name`、`employment_type`、`avatar_key`，接口从 `User.normalized_email` 返回 email。修改 employment type 时先校验现有角色，不自动撤销角色。

## 完成标准

- 四种基础角色成为代码、API 和数据库的唯一可分配集合。
- Founder + Advisor 同时存在时无需切换角色，权限自动合并。
- Admin 单独登录不能读取 Case、Assessment 或客户资料。
- Contractor 无法与其他角色共存，且只能读取当前 Task 的脱敏工作区。
