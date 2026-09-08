# 产品边界现状分析

返回[现状分析总览](README.md)。业务依据：[产品边界](../business-requirements/00-product-scope.zh-CN.md)。

## 结论

| BR | 状态 | 当前实现摘要 |
| --- | --- | --- |
| `BR-001` | `部分符合` | 已形成 K12 内部 ERP 的模块化骨架，但尚未形成可操作的完整 Release 1 闭环 |
| `BR-002` | `部分符合` | 目标模块基本都有代码、迁移或页面，但多处仍是旧规则、mock 或不可用运行时 |
| `BR-003` | `冲突` | 已排除的 Data Reviewer、CRM merge 和 Platform Billing 仍存在活跃实现；ReferralSource 已由最新业务基线恢复为 Release 1 正式实体 |

## 主要证据

- `modules/` 已包含 identity、access、crm、cases、tasks、schools、documents、notifications、audit、operations、external-portal 等目标模块。
- `app/(erp)/today/page.tsx` 仍明确使用 `previewCaseWorkspaceAdapter` 和 synthetic case data。
- `modules/schools/infrastructure/runtime.ts`、`modules/notifications/infrastructure/runtime.ts`、`modules/external-portal/infrastructure/runtime.ts` 等仍固定抛出运行时不可用。
- `modules/access/domain/contract.ts`、登录页、邀请 API 和数据库角色约束仍包含 `data_reviewer`。
- `modules/crm/application/duplicate-review-service.ts` 与 `db/migrations/202608230020_030_expand_crm_duplicate_review.sql` 仍提供 merge/correction 模型。
- `modules/crm/application/referral-source-service.ts` 与 `db/migrations/202608230030_032_expand_case_referral_source_assignments.sql` 仍使用独立来源实体。
- `modules/platform-billing/`、`app/(erp)/platform/billing/` 和 `db/migrations/202608130020_012_expand_platform_billing.sql` 仍存在。

## 差异清单

| 优先级 | 差异 | 建议动作 |
| --- | --- | --- |
| `P0` | 排除项仍能进入类型、路由、页面或数据库 | 先冻结入口与依赖，再用追加迁移纠正数据库；历史迁移不重写 |
| `P0` | 核心 Case 流程没有端到端闭环 | 以选校确认、申请 Task、逐校结果、人工结案为第一条纵向流程 |
| `P1` | 多个模块只有契约或测试资产，没有目标环境运行组合 | 每个开发票据明确区分源码完成、本地 HTTP、浏览器、Preview 和生产验证 |
| `P2` | preview/mock 与权威读写页面并存 | 真实闭环完成后移除或明确隔离演示入口 |

## 完成标准

- Release 1 导航、API、服务和数据库不再依赖已排除业务。
- 一个合成案件可按唯一事实源完成内部端到端流程。
- 页面上不再把 preview/mock 数据呈现为正式业务状态。
