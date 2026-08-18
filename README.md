# Tianxingguoji 项目文档索引

## 1. 用途

本目录保存跨版本的产品需求、实施决策、阶段计划、生产架构研究和历史设计。它与代码仓库中的 `docs/` 分工如下：

- 本目录回答“为什么做、做什么、哪些决策已批准”。
- 代码仓库 `docs/` 回答“某个代码版本如何实现、验证和运维”。
- 代码、迁移、测试和 release evidence 证明实际完成度；文档存在本身不代表实现已经可运行。

## 2. 当前执行边界

截至 2026-08-18，接手阶段 0 和阶段 1 已完成。阶段 2A 已在本地贯通 CRM Student 读取和“从既有 Student 建立 K12 ServiceCase”的首个 API v1 纵向切片，并通过用户人工验收。阶段 2B 已贯通 15 字段 Assessment 读取、逐字段保存、明确语义状态和 `background_complete` 门禁，并于 2026-08-18 获得用户验收，代码提交为 `9ef226a`。阶段 2C 已完成 Primary Advisor 案件推进和 Founder 带原因回退的本地实现、端到端验证及用户验收，代码提交 `c22c04e` 已推送至 `origin/main`。当前范围基线为本仓库根目录的 `TAKEOVER_PHASE0_SCOPE_BASELINE.md`，中文阅读版为 `TAKEOVER_PHASE0_SCOPE_BASELINE.zh-CN.md`；代码、迁移和测试仍是实际完成度的最终证据。

当前不授权：

- AWS、Cloudflare、Vercel、DNS、Terraform 或生产数据库操作；
- 真实 Student、Guardian、ServiceCase、文件或其他 PII 的导入和处理；
- 自动爬虫调度和生产 snapshot 发布；
- 生产首案、Portal 首次真实授权或第二组织启用。

产品文档中的 AWS 香港架构仍是未来生产目标，并未被本地交付目标取代；只是其执行整体延后。

## 3. 权威顺序

出现冲突时按以下顺序处理：

1. 用户最新的明确决定。
2. `PRD_IMPLEMENTATION_DECISIONS.md` 中状态为 accepted/amended 的决策。
3. 已采纳的阶段修订和技术决策。
4. `PRD_PHASE_IMPLEMENTATION_PLAN.md` 的 dependency graph 与验收契约。
5. 代码仓库中的版本绑定实现记录、契约、迁移、测试和证据。
6. research 文档。
7. archive 历史文档。

低优先级来源不得覆盖高优先级来源。发现冲突时记录并停止受影响的实现，不自行选择方便的版本。

## 4. 当前核心文档

| 文档 | 类型 | 状态与用途 |
|---|---|---|
| `TAKEOVER_PHASE0_SCOPE_BASELINE.zh-CN.md` | 中文接手范围基线 | `accepted_current_scope`；2026-08-17 已由用户确认，固定当前本地 Release 1 范围、证据顺序和执行约束 |
| `TAKEOVER_PHASE0_SCOPE_BASELINE.md` | 英文接手范围基线 | `accepted_current_scope`；与中文阅读版保持相同语义 |
| `product/RELEASE1_REQUIREMENTS_READER.zh-CN.md` | 中文需求阅读版 | `reader_copy_confirmed`；用户已确认现有需求方向，本文仍不替代决策台账 |
| `product/RELEASE1_REQUIREMENTS_TRACEABILITY_MATRIX.zh-CN.md` | 阶段 0.2 需求追踪矩阵 | `accepted_stage1_input`；连接需求、页面、API、模块、数据库和测试，作为阶段 1 的缺口基线 |
| `PHASE2A_CRM_CASE_VERTICAL_SLICE.md` | 阶段 2A 实施记录 | `accepted_local`；记录 Student 读取与既有 Student 建案纵向切片的边界、实现和验证结果 |
| `PHASE2B_ASSESSMENT_VERTICAL_SLICE.md` | 阶段 2B 实施记录 | `accepted_local`；记录 15 字段 Assessment、答案保存和背景收集门禁的边界与本地证据 |
| `PHASE2C_CASE_STAGE_TRANSITION.md` | 阶段 2C 实施记录 | `accepted_local`；记录首个案件阶段推进、回退、数据库边界和本地证据 |
| `product/IDENTITY_CONTEXT.md` | Identity 领域术语 | `active_domain_language`；区分内部 Account Disable、Cognito Revoke Effect、Reconciliation Attempt 和 Revoke Receipt |
| `decisions/R1X-DECISION-BASELINE-20260812.md` | Portal/Billing 决策基线 | `implementation_baseline_selected`；DP-01 至 DP-12 可约束本地实现，但真实启用仍需独立 gate |
| `PRD_IMPLEMENTATION_DECISIONS.md` | 决策台账 | `authoritative_active`；保存已批准、修订、取代和待决事项 |
| `PRD_PHASE_IMPLEMENTATION_PLAN.md` | 阶段实施计划 | `authoritative_active`；Release 1 ticket graph、依赖和验收证据基线 |
| `PHASE3_EMPTY_TENANT_PILOT_REVISION_PLAN.md` | 阶段修订 | `adopted_amendment`；修订 Phase 3/4 的空租户、重建和试点规则 |
| `IMPLEMENTATION_PLAN_AWS_HK_RELEASE1_PORTAL_BILLING.md` | 范围扩展方案 | `active_with_later_constraints`；Portal/Billing/AWS 范围受后续 DEC、R1X baseline 和当前本地边界约束 |
| `TECHNICAL_DECISION_PRODUCTION_PLATFORM.md` | 技术决策 | `accepted_future_production_target`；AWS 香港敏感平面和 Cloudflare 公开平面 |
| `TECHNICAL_DESIGN_AWS_CLOUDFLARE_PRODUCTION_DEPLOYMENT.md` | 生产部署设计 | `approved_design_execution_deferred`；不构成 plan/apply/deploy 权限 |
| `product/BUSINESS_CONTEXT_AND_FUTURE_BACKLOG.md` | 产品输入 | `candidate_input`；历史业务背景和 Release 1 之后的候选能力，不是批准需求 |

首次接手建议先阅读中文需求阅读版，再按其中的 `DEC-*` 编号查阅决策台账；当前开发排序先参考阶段 0.2 追踪矩阵，再进入阶段实施计划中的具体票据。

### 待权威同步事项

部分核心文档的页首状态早于 2026-08-17 的当前工作决定。进入开发前需用单独的文档票完成同步：

- 当前只交付本地 Release 1，云端执行全部延后；
- 本地使用确定性合成数据，不导入真实业务数据；
- 本地计划采用 Colima、Docker Compose、PostgreSQL 17、LocalStack、ClamAV 和开发专用角色登录；
- Platform Billing 当前交付只包含推进中个案计数与合同参考值，不计算金额、账单或付款；
- `decisions/R1X-DECISION-BASELINE-20260812.md` 的 DP-01 至 DP-12 已被用户选择为本地实现基线，但不授权任何真实或生产使用。

在同步完成前，用户最新明确决定优先，旧状态不得被解释为云端执行授权。

## 5. Research 索引

以下文件均为 `research_only`。它们提供比较、证据或历史评估，不自动形成供应商、预算或部署决策。

| 文档 | 主题 |
|---|---|
| `research/2026-08-11_ALTERNATIVE_PRODUCTION_PLATFORMS_DEEP_DIVE.md` | 替代生产平台比较 |
| `research/2026-08-11_AWS_CLOUDFLARE_ACCOUNT_REGISTRATION_AND_VERCEL_MIGRATION.md` | 账号注册与 Vercel 迁移研究 |
| `research/2026-08-11_AWS_CLOUDFLARE_HYBRID_COST_RESIDENCY_CHECK.md` | 混合架构成本与数据驻留 |
| `research/2026-08-11_AWS_CLOUDFLARE_VS_CHINA_CLOUD_HONG_KONG.md` | 香港区域云平台比较 |
| `research/2026-08-11_AWS_PRODUCTION_PLATFORM_DEEP_DIVE.md` | AWS 生产平台研究 |
| `research/2026-08-11_CRM_CLOUD_DESIGN_REVISION_REVIEW.md` | CRM 云设计修订审阅 |
| `research/2026-08-11_DATABASE_AND_MULTITENANCY_DEEP_DIVE.md` | 数据库与多租户研究 |
| `research/2026-08-11_PRODUCTION_HOSTING_AUTH_OPTIONS.md` | 托管与身份选项 |
| `research/2026-08-11_PRODUCTION_PLATFORM_DATABASE_MULTITENANCY_SYNTHESIS.md` | 生产平台与多租户综合结论 |
| `research/aws-identity-hong-kong-assessment.md` | 香港身份服务评估 |
| `research/aws-postgresql-hong-kong-assessment.md` | 香港 PostgreSQL 评估 |
| `research/aws-s3-hong-kong-assessment.md` | 香港 S3 评估 |
| `research/mdn-learn-web-development-programming-insights.md` | 通用 Web 工程参考 |

研究文档中若包含旧价格、产品能力、区域可用性或法规描述，实际使用前必须重新核实。

## 6. 历史归档

`archive/legacy-design/` 保存 2026 年 6 月的早期学校平台和 ERP 构想。

- 状态统一为 `historical_non_authoritative`。
- 不得依据归档文档直接开发、采购或部署。
- 仍有价值的业务输入已经提炼到 `product/BUSINESS_CONTEXT_AND_FUTURE_BACKLOG.md`。
- 归档详情见 `archive/legacy-design/README.md`。

## 7. 状态词汇

| 状态 | 含义 |
|---|---|
| `accepted_current_scope` | 用户已确认的当前接手范围；不自动授权提交、推送、外部操作或部署 |
| `reader_copy_confirmed` | 中文阅读版中的需求方向已经确认；权威规则仍以决策台账为准 |
| `draft_for_user_review` | 已完成初稿或静态审计，等待用户核对后才能进入下一步骤 |
| `accepted_stage1_input` | 阶段 0.2 审计已确认，可作为阶段 1 排序输入；仍需随代码进度更新 |
| `implemented_local_pending_acceptance` | 已在本地完成实现和聚焦验证，等待用户人工验收；不自动授权提交、推送或进入下一阶段 |
| `authoritative_active` | 当前权威来源，仍需遵守其内部 gate |
| `adopted_amendment` | 已采纳的局部修订，覆盖被明确点名的旧内容 |
| `active_with_later_constraints` | 内容仍有用，但必须结合后续决策读取 |
| `accepted_future_production_target` | 生产方向已接受，不代表获准执行 |
| `approved_design_execution_deferred` | 设计可以作为未来输入，当前不得操作外部系统 |
| `candidate_input` | 候选产品输入，尚未批准为需求 |
| `active_domain_language` | 当前统一使用的领域术语；不等同于运行实现或发布证据 |
| `implementation_baseline_selected` | 可约束本地实现的决策基线；保留文档内列出的人工和生产 gate |
| `research_only` | 研究证据，不是决定 |
| `historical_non_authoritative` | 仅供追溯，不得作为现行前提 |

## 8. 维护规则

- 新决定写入决策台账，不只写在聊天或研究报告中。
- 被取代的决定保留原文并标注 superseded，不静默删除。
- 研究与批准分开；推荐不等于授权。
- 状态变化必须写明日期、决策来源和影响范围。
- 不在索引中复制完整技术方案，避免多个版本漂移。
- 版本绑定的实现细节、运行手册和测试证据留在代码仓库。
