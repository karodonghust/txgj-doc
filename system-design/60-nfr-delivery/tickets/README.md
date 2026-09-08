# Release 1 开发票据索引

状态：`approved`  
建立日期：2026-08-25  
设计完成日期：2026-08-26  
P0 确认依据：项目负责人于 2026-08-25 确认通过 P0 边界  
业务基线：`BR-BASELINE-20260825-v32`

返回[非功能设计、评审与开发拆分索引](../README.md)。

## 1. 状态说明

- `approved/ready` 表示票据设计完整，可以按依赖分派，不表示已经开发。
- 当前实施事实包括 P0 已合并，以及 P1-BE-02 PostgreSQL 17 gate 已通过；其余 P1-P5 交付项仍按各自证据状态记录，P6 因无真实云环境暂缓。
- Frontend 票据必须等待对应 F1-F5 线框通过 Architect 与项目负责人确认。

## 2. P0 契约基线

| 顺序 | 票据 | Owner | 状态 |
| ---: | --- | --- | --- |
| 1 | [ARCH-01 模块边界 handoff](P0-ARCH-01-module-boundaries.zh-CN.md) | Architect | `approved` |
| 2 | [ARCH-02 API 与入口契约 handoff](P0-ARCH-02-api-entry-contract.zh-CN.md) | Architect | `approved` |
| 3 | [ARCH-03 旧规则与迁移输入清单](P0-ARCH-03-legacy-migration-inputs.zh-CN.md) | Architect | `approved` |
| 4 | [BE-00 Registry、入口与架构门禁实现](P0-BE-00-registry-boundaries.zh-CN.md) | Backend | `merged` |

## 3. P1 本地平台基础

| 票据 | Owner | 状态 |
| --- | --- | --- |
| [BE-01 Shared 幂等与事务基础](P1-BE-01-shared-transaction-foundation.zh-CN.md) | Backend | `passed` |
| [BE-02 Identity 与 Access](P1-BE-02-identity-access.zh-CN.md) | Backend | `passed` |
| [FE-01 Shared client 与工作台外壳](P1-FE-01-shared-client-shell.zh-CN.md) | Frontend | `passed`（focused） |
| [OPS-01 Local 组合与 readiness](P1-OPS-01-local-composition-readiness.zh-CN.md) | Platform Operations | `ready_after_BE_self_test` |
| [OPS-02 测试数据库与失败注入](P1-OPS-02-test-db-failure-injection.zh-CN.md) | Platform Operations | `ready_after_OPS-01` |

## 4. P2 CRM、Case 与候选名单

| 票据 | Owner | 状态 |
| --- | --- | --- |
| [BE-03 CRM、建案与 Assessment](P2-BE-03-crm-case-assessment.zh-CN.md) | Backend | `passed` |
| [BE-04 候选名单两层确认](P2-BE-04-candidate-list-case-flow.zh-CN.md) | Backend | `partial` |
| [FE-02 CRM、建案与 Cases 工作台](P2-FE-02-crm-cases-workspace.zh-CN.md) | Frontend | `passed`（focused） |

## 5. P3 逐校申请与 Tasks

| 票据 | Owner | 状态 |
| --- | --- | --- |
| [BE-05 SchoolTarget 与 Task 闭环](P3-BE-05-school-target-tasks.zh-CN.md) | Backend | `partial` |
| [FE-03 逐校申请、Task 与 Contractor 工作区](P3-FE-03-tasks-applications.zh-CN.md) | Frontend | `passed`（focused） |

## 6. P4 Documents 与 Notifications

| 票据 | Owner | 状态 |
| --- | --- | --- |
| [BE-06 Documents 上传、扫描与授权](P4-BE-06-documents.zh-CN.md) | Backend | `passed` |
| [BE-07 站内通知与提醒调度](P4-BE-07-notifications.zh-CN.md) | Backend | `passed` |
| [FE-04 Documents 工作台](P4-FE-04-documents.zh-CN.md) | Frontend | `passed`（focused） |
| [FE-05 Notifications 入口](P4-FE-05-notifications.zh-CN.md) | Frontend | `passed`（focused） |

## 7. P5 Guardian Portal

| 票据 | Owner | 状态 |
| --- | --- | --- |
| [BE-08 Guardian External Portal](P5-BE-08-external-portal.zh-CN.md) | Backend | `passed` |
| [FE-06 Guardian Portal 工作区](P5-FE-06-external-portal.zh-CN.md) | Frontend | `passed`（focused） |

## 8. P6 Production

| 票据 | Owner | 状态 |
| --- | --- | --- |
| [BE-09 Production AWS Adapter 边界](P6-BE-09-production-adapter-boundaries.zh-CN.md) | Backend | `passed`（source/focused） |
| [OPS-03 香港生产适配、恢复与发布](P6-OPS-03-production-aws-release.zh-CN.md) | Platform Operations | `deferred_no_environment` |

## 9. 云环境约定

- 当前没有真实云环境，P0–P5 可以使用显式 `cloud-synthetic` fake adapter 做本地联调。
- fake 结果只标记 `simulated`；云/生产检查保持 `not_run`。
- `production-aws` 必须拒绝 local、mock、preview、cloud-synthetic 和 legacy adapter。

## 10. 固定交付顺序

```text
Architect 契约确认
  -> Backend 实现与自测
  -> Architect readiness review
  -> 现有独立测试工作流 Local 验收
  -> Architect 最终审查
  -> 项目负责人决定 Git/PR
```

本索引不创建新的 QA 角色、任务或实体。

## 11. Unified Release 1 delivery evidence

更新时间：2026-08-27。统一目录：`/Users/karo/Documents/Tianxingguoji`。统一主目录 SHA：`pending`。

`QA_STATUS=blocked/partial`。以下是独立 QA 最终复核的分项结果；该状态不得解释为 Release 1 总体验收通过。

| Ticket / gate | Status | Evidence |
| --- | --- | --- |
| P1-BE-01 PostgreSQL 17 | `passed` | 10 tests, 10 pass, 0 fail |
| P1-BE-02 PostgreSQL 17 | `passed` | 10 tests, 10 pass, 0 fail |
| P2-BE-03 focused + CASE-01/CRM-05 Local Next Dev HTTP + PostgreSQL | `passed` | 修复后 16/16；CASE-01、CRM-05 真实本地 HTTP+PG 通过 |
| P2-BE-04 | `partial` | 24 pass, 1 skipped；`P2_BE04_HTTP_URL`、`P2_BE04_FIXTURE` 未配置 |
| P3-BE-05 | `partial` | 14 pass, 2 skipped；隔离 PostgreSQL URL、`P3_HTTP_BASE_URL` 未配置 |
| P4-BE-06 | `passed` | 9 tests, 9 pass, 0 fail |
| P4-BE-07 | `passed` | 13 tests, 13 pass, 0 fail |
| P5-BE-08 | `passed` | 23 tests, 23 pass, 0 fail |
| P6-BE-09 | `passed` | 1 test, 1 pass, 0 fail；仅 source/focused boundary evidence |
| FE-01–06 focused suites | `passed` | 全部 focused 通过 |
| typecheck | `passed` | 全量 TypeScript 检查通过 |
| architecture | `passed` | 25/25 |
| baseline | `passed` | `pnpm db:baseline:check` 通过 |
| diff-check | `passed` | `git diff --check` 通过 |
| Browser | `not_run` | 未运行浏览器验收 |
| Worker | `not_run` | 未运行 worker 验收 |
| Deployment | `not_run` | 未运行部署验收 |
| Vercel / Neon / AWS | `not_run` | 未运行对应远端或云能力验收 |
| Unified main-directory SHA binding | `pending` | 等待统一主目录后续收口 |

P2-BE-04、P3-BE-05 的 skipped 项是环境配置缺失，不计为通过；Local、browser、worker、deployment、Vercel、Neon 与 AWS 证据继续保持独立边界。

独立验收继续复用现有“独立测试”工作流；每批在 owner 自测与 Architect readiness review 后，才交付对应 Local Dev 场景。
