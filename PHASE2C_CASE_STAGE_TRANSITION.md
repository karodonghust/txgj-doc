# 阶段 2C：案件阶段推进与回退纵向切片

| 项目 | 当前值 |
| --- | --- |
| 日期 | 2026-08-18 |
| 状态 | `accepted_local` |
| 数据范围 | 仅本地确定性合成数据 |
| 代码状态 | 提交 `c22c04e` 已推送至 `origin/main` |
| 云端状态 | 未操作 AWS、Vercel、Cloudflare 或生产数据库 |

## 1. 本阶段解决的问题

阶段 2B 可以把 Assessment 推进到 `background_complete`，但 ServiceCase 仍停在 `signed`。阶段 2C 增加独立的案件阶段命令，使案件只能在满足权限、版本和资料前置条件时推进，并保留不可改写的回退原因与操作历史。

本阶段只实现两条方向：

| 从 | 到 | 操作者 | 前置条件 | 原因 |
| --- | --- | --- | --- | --- |
| `signed` | `background_collection` | 当前 Primary Advisor；若 Founder 本身是当前 Primary，也可代行 | 批准的 Manifest、Assessment 为 `background_complete`、背景 blocker 全部有答案 | 不需要 |
| `background_collection` | `signed` | Founder | 只能回退到紧邻上一阶段 | 必须非空 |

关闭、暂停、取消、其他阶段推进，以及 SchoolTarget、Task、Document 都不在本阶段范围。

## 2. 实现链路

```text
案件详情 CaseStageControls
          |
          v
POST /api/v1/cases/[caseId]/transitions
          |
          v
CaseTransitionService
          |
          v
PostgresqlCaseTransitionRepository
          |
          +--> 受控数据库阶段函数
          +--> 不可变阶段事实
          +--> ServiceCase 版本更新
          +--> AuditEvent + Outbox
          +--> IdempotencyReceipt
```

页面不是权限边界。按钮是否显示只改善操作体验；API、Repository 和数据库函数都会重新读取并验证当前角色、Primary 绑定、案件版本、案件阶段、Manifest、Assessment 和 blocker。

## 3. 数据库与安全边界

追加迁移 `024` 完成以下变化：

- 新增 `cases_service_case_transition_facts`，阶段历史只允许追加，不允许修改或删除。
- 新增 tenant-bound 的 `cases_apply_service_case_transition` 数据库函数。
- 撤销 `tianxing_app` 对 `cases_service_cases` 的普通 `UPDATE` 权限。
- 应用只能调用受控函数改变阶段，不能直接更新主表绕过规则。
- 阶段事实、案件版本、审计、Outbox 和幂等结果处于同一数据库事务；任何一步失败都不应留下部分结果。

追加迁移 `025` 和 `026` 完成安全加固与纠正：

- `025` 限制回退原因长度和操作时间窗口，并在最终更新前锁定、重查操作者权限及 Assessment 证据。
- `026` 修复 `025` 触发器中 Assessment/Manifest 变量与数据库字段同名造成的歧义；由于 `025` 已在本机应用，因此通过新增纠正迁移处理，没有改写迁移历史。

生产运行时没有因此启用。非 `local-synthetic` 模式继续 fail closed，生产迁移和 RDS 组合需要独立批准。

## 4. 本地验证结果

- 业务规则与 PostgreSQL Repository 聚焦测试：8/8 通过。
- 迁移边界与本地 runner 测试：11/11 通过。
- 模块架构测试：15/15 通过。
- 迁移 `024` apply 后 ledger 从 22 变为 23，public 表从 61 变为 62；加固与纠正 dry-run 分别只选择 `025`、`026`，apply 后 ledger 为 25，public 表仍为 62。
- Advisor 页面在 Assessment 已完成时显示并执行正向推进。
- Founder 页面要求填写原因后才能执行回退。
- 回退后再由 Primary Advisor 推进，刷新后状态和版本保持。
- 最终合成案件为 `background_collection`、记录版本 6，重新打开页面后状态保持。
- 数据库存在连续五条历史：版本 1→2、2→3、3→4、4→5、5→6；同时存在 5 条 AuditEvent、5 条 pending Outbox 和 5 条 completed IdempotencyReceipt。
- 使用 `tianxing_app` 直接更新 `cases_service_cases` 被 PostgreSQL 拒绝。

未运行 `pnpm lint`、`pnpm build` 或完整测试。仓库全量严格 TypeScript 检查仍有既有无关错误，本阶段不把清理全仓基线当作阶段 2C 的扩展任务。

## 5. 验收与下一步

用户已于 2026-08-18 确认验收通过。验收结论包括：

1. 接受由当前 Primary Advisor 执行推进，并在页面显示前置条件和权限提示。
2. 接受 Founder 回退时必须填写非空原因。
3. 接受 Assessment 完成与 ServiceCase 推进保持为两个独立动作。
4. 接受本阶段只实现第一组相邻阶段，后续阶段另开票据。

代码提交 `c22c04e` 已推送至 `origin/main`。下一张候选票据是 SchoolTarget 最小纵向切片，但必须再次单独确认，不因本次验收自动开始。
