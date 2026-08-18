# 阶段 2B：Assessment 背景收集纵向切片

| 项目 | 状态 |
|---|---|
| 日期 | 2026-08-18 |
| 状态 | `accepted_local`；2026-08-18 用户确认提交并推送 |
| 数据范围 | 仅本地确定性合成数据 |
| 代码状态 | `9ef226a`（`feat: complete local assessment background slice`） |

## 1. 本阶段解决的问题

阶段 2A 已经能从既有 Student 建立 K12 ServiceCase 和 draft Assessment，但 Assessment 还不能真实读取、保存或完成背景收集。阶段 2B 贯通案件详情页面、API v1、应用服务、PostgreSQL Repository 和数据库事务，使内部人员可以操作建案时固定的正式 15 字段评估。

## 2. 已实现范围

- 案件详情按绑定的不可变 manifest 渲染 15 个正式 K12 字段。
- 每个字段独立保存，并使用答案 `record_version` 做乐观并发控制。
- 支持 `provided`、`unknown`、`not_applicable` 和 `declined_to_provide` 四种明确语义状态。
- 10 个 `background_collection` 阻塞字段都有答案后，允许 Assessment 从 `draft` 进入 `background_complete`。
- 完成背景收集不会自动修改 ServiceCase 阶段；案件阶段仍由独立命令推进。
- 每次答案保存和背景完成都在同一事务写入业务数据、AuditEvent、Outbox 和 IdempotencyReceipt。
- Founder、Admin 可操作组织内案件；Advisor 只可操作自己负责的案件；其他角色被拒绝。
- 新建案件默认使用 `local-release1-v2`；已被旧案件引用的 v1 保留，不改写、不删除。

## 3. 架构与数据边界

```text
案件详情 / AssessmentEditor
            |
            v
/api/v1/cases/{caseId}/assessment
            |
            v
AssessmentService
            |
            v
PostgresqlAssessmentRepository
            |
            +--> Assessment / Answer
            +--> AuditEvent / Outbox
            +--> IdempotencyReceipt
```

- 页面只负责收集输入和展示冲突，不直接拼写数据库规则。
- 应用服务负责角色、字段类型、语义状态、blocker 和命令契约。
- Repository 在一个租户事务内重读权限、锁定版本并提交原子副作用。
- 应用账号不直接读取 manifest 底层表，只能调用受组织边界保护的数据库函数。
- 数据库存储的 blocker 名称使用 Assessment 状态；应用目录使用案件阶段名称，映射只发生在持久化边界。

## 4. 追加迁移

- `021`：允许受控的 Assessment 状态推进，并继续验证 manifest、版本、时间和阻塞项。
- `022`：提供绑定案件和组织的 manifest/field 读取函数，并补充运行时 blocker 校验。
- `023`：强化答案校验触发器的执行身份；不向应用账号开放 manifest 字段表。

本地迁移账本为 22/22，61 张 public 表数量不变。已执行迁移保持不可变，后续问题只能通过新迁移修正。

## 5. 明确未包含

- `background_complete -> selection_ready` 的用户流程。
- ServiceCase 阶段推进、回退或审批。
- SchoolTarget、Task、Document 或 Guardian 编辑。
- 旧 v1 manifest 的删除或静默升级。
- AWS、Vercel、Cloudflare、Cognito 或任何生产环境操作。
- 真实学生、家长、案件、评估答案或文件。

## 6. 本地验证结果

- 聚焦迁移与权限边界测试 19 项通过。
- Assessment 服务与 K12 目录测试 11 项通过；本地种子安全测试 1 项通过。
- 迁移 022 和 023 均先 dry-run 再 apply，本地 ledger 最终为 22。
- v2 种子验证结果为 2 名 Student、2 名 Guardian、2 条关系、1 个可见 v2 manifest 和 15 个字段。
- 浏览器建立 v2 合成案件，保存 10 个背景阻塞字段并完成背景收集；刷新后状态和答案仍存在。
- 数据库证据：测试案件保持 `signed`，Assessment 为 `background_complete`、版本 2；存在 10 个答案、11 个 AuditEvent、11 个 Outbox 和 11 个完成态 IdempotencyReceipt。
- 新建案件页面在服务重启后默认选择 v2，同时继续显示可追溯的 v1。
- 仓库全量严格 TypeScript 检查仍有既有无关错误；本阶段文件的聚焦检查没有新增错误。完整基线清零不是 2B 的完成声明。

## 7. 验收结论

2026-08-18，用户确认提交并推送阶段 2B，视为以下本地行为验收通过：15 字段来源清楚、语义状态可理解、逐字段保存符合预期、10 个背景阻塞项完成后按钮解锁、刷新后数据保留、Assessment 完成不自动改变案件阶段。

代码提交为 `9ef226a`。下一步在代码推送完成后再决定下一张票，当前候选为 SchoolTarget 最小纵向切片；本次验收不自动授权开始该模块。
