# 历史设计归档

## 归档目的

本目录保存 Tianxingguoji 在 2026 年 6 月形成的早期学校信息平台和 ERP 构想。它们用于解释产品演进过程，不再是当前需求、架构、开发计划或成本决策的权威来源。

归档日期：2026-08-17。

## 文件状态

| 文件 | 历史主题 | 当前状态 | 当前替代来源 |
|---|---|---|---|
| `2026-06-03-hk-school-platform-design.md` | 学校爬虫、筛选、管理后台和早期技术栈 | 已替代 | `../../PRD_IMPLEMENTATION_DECISIONS.md`、`../../PRD_PHASE_IMPLEMENTATION_PLAN.md` |
| `2026-06-03-hk-school-platform-design-REVIEW.md` | 对早期公开学校搜索 MVP 的审阅 | 已替代且与内部 ERP 定位部分冲突 | 当前 PRD、阶段计划及代码仓库测试契约 |
| `2026-06-03-hk-school-platform-design-driven-development-plan.md` | 早期 Contract First、纵向切片和四周 MVP 计划 | 流程思想已吸收，计划已替代 | `../../PRD_PHASE_IMPLEMENTATION_PLAN.md` |
| `留學仲介ERP系統架構評估.md` | 行业痛点、低代码、自动化、报告和 AI 构想 | 仅作研究输入 | `../../product/BUSINESS_CONTEXT_AND_FUTURE_BACKLOG.md` 及当前技术决策 |

## 使用规则

- 不得依据本目录直接实现功能、选择供应商、购买服务或操作生产环境。
- 历史文档中的数字、价格、市场描述和第三方能力没有因归档而获得验证。
- 历史技术栈包括 Next.js 15、FastAPI、Celery、Supabase/Neon、低代码、n8n、Dify 等，与当前批准架构不同。
- 有价值的业务想法必须先进入当前产品文档，获得明确决策后才能开发。
- 文件内部可能保留已经失效的相对链接和本机路径，保留它们是为了维持原始记录，不代表链接仍可使用。

## 当前权威文档

- `../../PRD_IMPLEMENTATION_DECISIONS.md`
- `../../PRD_PHASE_IMPLEMENTATION_PLAN.md`
- `../../IMPLEMENTATION_PLAN_AWS_HK_RELEASE1_PORTAL_BILLING.md`
- `../../TECHNICAL_DECISION_PRODUCTION_PLATFORM.md`
- `../../TECHNICAL_DESIGN_AWS_CLOUDFLARE_PRODUCTION_DEPLOYMENT.md`

代码版本绑定的实现计划、运行手册、安全模型和发布证据保留在 Tianxingguoji 代码仓库的 `docs/` 与 `evidence/` 中。
