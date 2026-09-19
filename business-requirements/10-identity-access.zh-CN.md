# Identity 与 Access

返回[唯一事实源索引](README.md)。

## BR-010 内部角色

状态：`confirmed`

| 角色 | 业务职责 |
| --- | --- |
| `Founder` | 公司级决策者；审批候选学校名单、高风险授权和敏感操作；在满足条件后人工结案 |
| `Admin` | 管理内部账号、组织成员和日常系统配置；不自动取得 Founder 专属批准权 |
| `Advisor` | 负责学生和案件日常工作；成为案件负责人时称 Primary Advisor |
| `Contractor` | 只处理明确分配的单一任务，并只读取该任务所需的脱敏信息 |

`Primary Advisor`、`Case Collaborator`、`Application Assignee` 和 `Interview Support Assignee` 是案件或任务中的责任关系，不是新的基础角色。

## BR-011 员工资料

状态：`confirmed`

内部员工使用以下组合表示：

```text
User
  -> OrganizationMembership
       -> RoleBinding
       -> EmployeeProfile
```

- `User`：登录账号和账号状态。
- `OrganizationMembership`：账号是否属于该公司。
- `RoleBinding`：基础业务角色。
- `EmployeeProfile`：员工在公司中的显示资料，不决定权限。

`EmployeeProfile` 最小业务字段：

| 字段 | 规则 |
| --- | --- |
| `membership_id` | 关联公司成员身份，一对一 |
| `display_name` | 必填，业务含义为员工“昵称”，用于负责人、审批人和员工列表展示 |
| `employment_type` | `FULL_TIME` 或 `PART_TIME` |
| `avatar_key` | 可空，只保存受控头像对象引用 |
| `email` | 员工资料接口必须返回，但权威值读取 `User.normalized_email`，不在 EmployeeProfile 重复保存 |

员工资料维护规则：

- 登录邮箱是 Identity 的账号标识，不作为可编辑“用户名”。
- Release 1 页面统一将员工 `display_name` 标示为“昵称”；右上角及侧栏默认头像显示昵称的第一个字符。
- Release 1 不保存员工真实姓名；未来若有合同、人事或合规需要，应新增独立真实姓名字段，不得把昵称改作真实姓名。
- 员工本人可以修改自己的昵称（`display_name`）；该操作不得修改登录邮箱、员工类型或任何角色。
- Founder 或 Admin 可以维护组织成员的昵称（`display_name`）、`employment_type` 和基础角色。
- 本人即使同时具有 Founder/Admin，也必须通过同一受控成员管理命令修改自己的员工类型或角色，不能绕过资格、角色兼容性、最后一个 Founder、并发版本和审计规则。

角色资格限制：

- `FULL_TIME` 可以担任 `Founder`、`Advisor`。
- `PART_TIME` 可以担任 `Contractor`。
- 员工类型不自动赋予角色，只限制可分配角色。
- 修改员工类型后若现有角色不兼容，应拒绝修改，不能静默删角色。
- `Admin` 暂不增加员工类型限制。
- Release 1 不设置 `Data Reviewer` 基础角色；学校资料变更统一由 Founder 审核，后续确有工作量再增加该角色。

## BR-012 授权模型

状态：`confirmed`

授权按以下顺序判断：

```text
有效 User/Session
  -> 有效 OrganizationMembership
  -> RoleBinding 的 Workspace Capability
  -> 具体案件关系
  -> Scope + Capability 或 TaskAssignment
  -> 业务资源自身规则
```

- 前端隐藏按钮不等于授权；每个服务端请求都必须重新判断。
- Primary Advisor 按案件负责人权限工作。
- Case Collaborator 只在指定案件、指定 Scope、指定 Capability 和有效期内工作。
- Contractor 只通过当前 TaskAssignment 取得单任务脱敏工作区。
- Scope 使用 `case_summary`、`education_profile`、`school_targets`、`task_workspace`、`communications`、`identity_contact`、`internal_notes`。
- Capability 使用 `view`、`comment`、`edit`；导出默认禁止。
- 普通授权默认最长 7 天；`identity_contact` 与 `internal_notes` 需要 Founder 批准且禁止自批。

## BR-013 多角色与权限包含关系

状态：`confirmed`

- 同一个 OrganizationMembership 可以同时具有兼容的多个基础角色；权限自动合并，不要求员工在登录时选择或切换角色。
- `Founder` 与 `Advisor` 可以组合；`Contractor` 不能与任何其他基础角色组合。
- `edit` 自动包含 `comment` 和 `view`；`comment` 自动包含 `view`。
- `Admin` 只负责账号、成员和日常系统配置，默认不能查看 ServiceCase 或 Assessment。Admin 如需参与客户业务，必须另外取得兼容的 `Advisor` 角色。

## BR-014 内部账号邀请与激活

状态：`confirmed`

- Release 1 不开放公开注册、自助注册或第三方身份登录。
- 只有 `Founder` 可以创建内部用户邀请，并在邀请时指定员工类型和基础角色。
- 邀请先创建 `invited` 用户、组织成员和待启用的角色事实；受邀人未完成激活前不能登录或访问业务数据。
- 系统向受邀邮箱发送一次性确认链接。链接只允许使用一次，并且必须有明确过期时间；重发会使旧链接失效。
- 受邀人点击链接后自行设置初始密码和最小个人资料（当前必填昵称）；不能修改登录邮箱、组织、员工类型或角色。头像沿用员工资料能力，后续可在个人资料功能中维护，不阻塞账号激活。
- 初始化成功后，用户和组织成员从 `invited` 变为 `active`，再创建内部会话。
- 邮箱地址是 Identity 的登录标识；创建后不可由用户自行修改。
- 邀请邮件属于内部账号管理，不属于面向 Guardian 或 Student 的业务通知。

## BR-015 K12 试用等级与授权（2026-09-19）

状态：`confirmed`（项目负责人授权试用开发；客户未逐项验收）

依据：项目负责人授权按 `ACCESS-TRIAL-01_PERMISSION_MATRIX_AND_PLAN.md` 完成开发、测试、合并 main 并推送。以下是可实施的试用规则，不表示客户正式验收或生产迁移授权。

- 单一机构天星教育，K12 按国际学校、本地学校分类；每个试用案件只属一个明确分类，未知分类默认拒绝。
- 每位试用成员一个等级：Founder、L1、L2、L3。L2 可有多个分类授权；L201/L202 是显示编码，不是独立基础角色。
- Founder 全部业务及员工管理权限；L1 全部业务权限，不能维护员工等级、分类授权、邀请、停用或系统凭据；L2 只处理已授权分类的案件及关联资料，不审批候选名单、不结案；L3 只处理当前指派任务和必要上下文，可跨分类接受任务，不获得完整案件、Assessment、联系人及内部备注。
- 只有 Founder 可设置等级、分类、邀请和停用账号；任何人可修改本人昵称。员工类型不授予或限制试用等级。
- L2 及以上可在自己业务范围内指派 L3；L3 不得转派。撤销或重派立即收回原受派人的权限；完成后只读至指派撤销。
- 试用授权采用明确等级、分类和任务关系，不与历史角色权限取并集；旧用户不自动映射或升权。映射真实用户必须经项目负责人审阅；本地只用合成账号。
- 禁止新增批量导出授权。所有等级遵守业务校验、扫描、并发版本、审计和幂等性，Founder 不绕过这些约束。
- 与本条冲突的 BR-010、BR-011 角色资格、BR-012 旧责任关系、BR-013 多角色并集，在显式启用新等级的成员范围内标记为 `superseded`；上文保留历史追溯。无新等级的历史账号不得隐式获得新权限。
- T01–T07 试用假设登记及具体矩阵见实施计划；变更仍由项目负责人确认。部署及真实员工迁移不属于此次代码合并授权。
