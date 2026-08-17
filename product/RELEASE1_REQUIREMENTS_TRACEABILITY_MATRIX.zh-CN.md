# Tianxingguoji Release 1 需求追踪矩阵

| 字段 | 内容 |
| --- | --- |
| 文档状态 | `accepted_stage1_input` |
| 阶段 | 接手阶段 0.2 已完成；当前作为阶段 1 输入 |
| 审计日期 | 2026-08-17；2026-08-18 刷新身份和模块路径进度 |
| 需求依据 | 已确认的 `RELEASE1_REQUIREMENTS_READER.zh-CN.md` 与 accepted/amended 决策 |
| 代码基线 | 阶段 0.2 初始审计为 `3410b8f366de946546d9f4958010febb734c1216`；2026-08-18 模块路径刷新对应 `e51440e7892e50ea115a35274dff31e56fe00828` |
| 文档基线 | 本地 `txgj-doc` 提交 `84a3826b18d32b81c88cde831ce251ce4944538c` |
| 证据范围 | 静态审计后已补充本地 Compose、15 份空库迁移、61 张 public 表、真实 PostgreSQL 约束测试和应用 readiness 证据；未运行 lint、build、云端或真实数据验证 |

## 1. 用途

本文把已经确认的需求连接到当前实现位置，回答六个问题：

1. 用户从哪个页面操作？
2. 页面应调用哪个 API？
3. 业务规则属于哪个模块？
4. 数据应保存在哪些表？
5. 目前有哪些测试证据？
6. 这条能力现在到底能不能在本地端到端操作？

文件存在只说明设计或代码骨架存在，不等于功能已经可运行。

## 2. 状态说明

| 状态 | 含义 |
| --- | --- |
| `mock_only` | 页面只读取硬编码或 preview 数据，没有连到目标 API 和数据库 |
| `legacy_partial` | 旧路径可以读取或写入，但仍依赖 Neon、请求时建表或非 `/api/v1` 接口，不是目标实现 |
| `contract_only` | API、领域服务、迁移或测试已经存在，但默认 runtime 明确拒绝运行 |
| `mixed_partial` | 同一能力同时存在 Mock、旧路径和新契约，尚未形成一条端到端链路 |
| `foundation_runtime_partial` | 本地依赖底座和数据库 schema 已实机运行，但合成身份或领域 runtime 尚未接通 |
| `out_of_scope_visible` | Release 1 不包含该功能，但当前仍有可访问页面或导航入口 |
| `local_operable` | 在当前本地环境完成页面、API、模块、数据库和聚焦测试验证；本次审计没有任何核心能力达到此状态 |

## 3. 总体追踪矩阵

| ID | 需求与决策来源 | 页面 | 目标 API | 领域模块 | 主要数据库表 | 现有测试证据 | 当前状态与主要缺口 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| R1-01 | 邀请制账号、角色、会话、案件授权；第 5 章，`DEC-007` 至 `DEC-011`、`DEC-020`、`DEC-029` | `/login`、`/login/activate`、`/admin/access` | `/api/v1/auth/*`、`/api/v1/cases/[caseId]/collaborators*` | `modules/identity`、`modules/access` | `identity_users`、`identity_sessions`、`identity_invites`、`access_*` | `identity-access-schema`、`identity-onboarding`、`identity-revoke-workflow`、`collaborator-scope-workflow`、`auth/mode`、`auth/runtime` | `foundation_runtime_partial`：`AUTH_MODE` 已隔离本地角色登录与 Cognito 登录，本地 opaque session 和统一 actor 校验可用；会话仍在进程内，合成身份尚未持久化到 PostgreSQL，Access 业务 runtime 仍未接通 |
| R1-02 | Student 与 ServiceCase 分离、Guardian 独立、主联系人关系；第 4 章，`DEC-004` 至 `DEC-006` | `/students`、`/students/[studentId]`、`/students/[studentId]/guardians` | `/api/v1/students/[studentId]/guardians`、`.../primary-handoffs`；当前没有 v1 Student 列表/详情 API | `modules/crm` | `crm_students`、`crm_guardians`、`crm_student_guardian_relationships`、`crm_referral_sources` | `crm-schema`、`guardian-relationship-workflow`、`duplicate-merge-workflow` | `mixed_partial`：学生列表和详情读取 `modules/crm/infrastructure/mock-students.ts`；Guardian 命令契约存在但 runtime 不可用；缺少 Student 查询和编辑链路 |
| R1-03 | K12 ServiceCase、八阶段流程、版本冲突、幂等与受控回退；第 6 章，`DEC-027`、`DEC-032`、`DEC-044` | `/cases`、`/cases/new`、`/cases/[caseId]`、`/cases/[caseId]/workspace` | 目标为 `/api/v1/cases`、`/api/v1/cases/[caseId]/transitions`；现页面还使用 `/api/cases` | `modules/cases` | `cases_service_cases`、`cases_assessments`、`shared_idempotency_records`、`audit_events`、`audit_outbox` | `case-creation-workflow`、`case-transition-workflow`、`case-workspace-model`、`case-api-json-types` | `mixed_partial`：列表/新建仍走旧 Neon service，详情仍有 preview adapter；v1 只有创建和命令路由，缺少可用的列表/详情 read model；默认 cases runtime 不可用 |
| R1-04 | 版本化四层 K12 评估、明确语义状态、阶段 blocker；第 7 章，`DEC-012`、`DEC-013` | 当前 16 字段显示位于 `/students/[studentId]`；正式编辑器用于案件 workspace | `/api/v1/cases/[caseId]/assessment` | `modules/cases/application/assessment-service.ts`、`modules/cases/domain/schema-resolver.ts`、`schema/k12` | `cases_schema_manifests`、`cases_schema_manifest_fields`、`cases_assessments`、`cases_assessment_answers` | `case-schema`、`k12-catalogue`、`assessment-workflow` | `mixed_partial`：正式 schema、服务和测试存在，但 runtime 不可用；现有页面 16 字段来自 Mock，未按正式 manifest 渲染；16 个旧字段与 15 个正式字段没有一一映射 |
| R1-05 | SchoolTarget 独立状态、证据、终态结果和案件摘要；第 8 章，`DEC-027`、`DEC-058` | 案件详情和 workspace 中的学校目标区域 | `/api/v1/cases/[caseId]/school-targets*` | `modules/cases/application/school-target-service.ts`、`modules/cases/application/outcome-service.ts` | `cases_school_targets`、`cases_case_outcomes`、`schools_resolved_revisions` | `school-target-workflow`、`case-target-outcome-workflow` | `contract_only`：命令路由、服务、表和测试存在；页面仍展示 preview 数据，school-target 和 outcome runtime 默认不可用 |
| R1-06 | Task 独立工作流、执行人完成、不同 Founder 审批；第 9 章，`DEC-028` | `/tasks`、`/cases/[caseId]#tasks`、`/contractor/tasks/[taskId]` | `/api/v1/tasks/[taskId]/transitions`、`/api/v1/contractor/tasks/[taskId]` | `modules/tasks` | `tasks_transition_policies`、`tasks_transition_rules`、`tasks_tasks`、`tasks_task_assignments`、`tasks_task_transition_receipts` | `task-workflow`、`contractor-task-workspace`、`tasks/transitions` | `mixed_partial`：任务总页读取 preview adapter 且 mutation 被禁用；承包人 workspace 和命令契约存在，但 runtime 不可用；缺少内部任务列表/创建 API |
| R1-07 | 私有文件、隔离扫描、版本、软删除和恢复；第 10 章，`DEC-017` | `/documents`、`/cases/[caseId]#documents` | `/api/v1/cases/[caseId]/documents/upload-intents`、`.../deletions`、`.../restorations`、`.../version-rollbacks` | `modules/documents`、`modules/documents/infrastructure/object-store.ts`、`workers/scan-document.ts`、`workers/reconcile-documents.ts` | `documents_documents`、`documents_document_versions`、`documents_scan_results` | `document-upload-workflow`、`document-scan-workflow`、`document-version-workflow`、`document-store` | `mixed_partial`：本地 S3/SQS/ClamAV 底座已健康运行，但页面仍是 preview，文档 runtime、扫描 runtime 和版本 runtime 默认不可用，也没有文件列表 read model |
| R1-08 | 站内通知与追加式审计，同事务失败即拒绝写入；第 11 章，`DEC-032`、`DEC-062` | 当前没有通知中心或审计查看页 | 当前没有独立 v1 通知/审计查询 API；由业务命令和 worker 产生副作用 | `modules/notifications`、`modules/audit`、`workers/deliver-in-app.ts` | `audit_events`、`audit_outbox`、`notifications_notifications`、`notifications_delivery_receipts` | `outbox-audit`、`in-app-notification-delivery` | `contract_only`：原子效果契约和 worker 测试存在；默认通知 runtime 不可用；缺少本地 worker 运行入口、通知读取 API 和用户页面 |
| R1-09 | Founder 看板显示案件阶段、责任人、下一任务、截止和异常；第 11 章，`DEC-052`、`DEC-057` | `/dashboard`、`/today` | `/api/v1/dashboard/cases` | `modules/operations/domain/case-dashboard-projection.ts`、`modules/operations/infrastructure/case-dashboard-route.ts` | 从 `cases_service_cases`、`tasks_tasks` 等权威表生成投影视图 | `case-dashboard-projection`、`case-dashboard-route` | `mixed_partial`：dashboard 已调用 v1 API但默认 runtime 不可用；today 仍读取 preview adapter；尚未验证空库和合成数据页面 |
| R1-10 | 学校快照、筛选、人工审核、provisional School 和 overlay 治理；第 8、14 章，`DEC-014` 至 `DEC-016` | `/selector`、`/schools`、`/admin/crawler`、`/admin/schools` | 旧 `/api/crawler/*`；目标 `/api/v1/schools/*`、`/api/v1/admin/schools/*` | `modules/schools/infrastructure/crawler`、`modules/schools` | 旧 `crawler_*` 请求时建表；目标 `schools_schools`、`schools_snapshots`、`schools_snapshot_records`、`schools_overlay_*`、`schools_resolved_revisions` | `crawler/snapshot-manifest`、`school-change-workflow`、`school-governance-workflow`、`school-target-workflow`、`schools/resolver` | `legacy_partial`：已提交快照可由旧 API 读取，审核决定和工单仍走 Neon 且请求时建表；当前 manifest 是旧 warning 格式；v1 学校 runtime 默认不可用 |
| R1-11 | 单案件、限时、字段白名单的只读家长 Portal；第 12 章，`DEC-064`、`DEC-065` | `/portal/access`、`/portal/workspace`、`/cases/[caseId]/access` | `/api/v1/portal/sessions`、`/api/v1/portal/workspace`、`/api/v1/cases/[caseId]/portal-grants*` | `modules/external-portal` | `portal_viewers`、`portal_access_grants`、`portal_sessions`、`portal_security_events`、`portal_idempotency_records` | `portal-api-routes`、`portal-schema-contract`、`portal-repository-contract`、`portal/contract-policy`、`portal-pages` | `contract_only`：页面、allowlist、路由和测试存在；默认路由显式返回 runtime unavailable，尚无本地持久化组合和浏览器验收 |
| R1-12 | 平台只统计推进中案件数量并显示合同参考值，不计算金额；第 13 章，`DP-06`、`DP-09`、`DP-11` | `/platform/billing` | `/api/v1/platform/billing/overview` | `modules/platform-billing` | `platform_billing_contract_versions`、`platform_billing_metric_snapshots`、`platform_billing_subscription_projections`、`cases_billing_projection_events` | `platform-billing-persistence`、`platform-billing-overview-route`、`platform-billing-schema-contract`、`platform-billing/contract-policy` | `contract_only`：计数和权限规则、页面模型及表存在；默认 runtime 不可用；尚未从本地案件事件生成月底快照 |
| R1-13 | 使用受控事件逐案重建合成/未来既有案件；第 3、16 章，`DEC-061`、`DEC-067` | `/cases/reconstructions/new`、`/cases/reconstructions/[reconstructionId]` | `/api/v1/cases/reconstructions` | `modules/cases/domain/reconstruction`、`modules/cases/application/reconstruction`、`modules/cases/infrastructure/reconstruction` | `cases_reconstructions`、`cases_reconstruction_versions`、`cases_reconstruction_events`、`cases_reconstruction_gaps`、`cases_reconstruction_activations` | `case-reconstruction-workflow`、`case-reconstruction-route`、`case-reconstruction-schema`、`case-reconstruction-ui-model` | `contract_only`：事件契约、UI 模型、路由和表存在；默认 runtime 不可用；当前阶段只允许合成数据，不授权真实案件 |
| R1-14 | 明确互斥的 `local-synthetic` 运行组合；阶段 0.1 当前范围 | 所有 Release 1 页面 | 为上述 v1 路由提供本地依赖；新增 `/api/v1/local/readiness` | 已有 fail-closed 本地配置、迁移 runner、依赖探测和身份 mode adapter；尚未连接其他领域公开 runtime | PostgreSQL 17；LocalStack 模拟 S3/SQS；ClamAV | 15 条 migration ledger、61 张 public 表；迁移与真实 PostgreSQL 约束测试通过；应用 readiness 全 ready；本地身份聚焦测试通过 | `foundation_runtime_partial`：本地依赖、schema 和进程内合成角色登录已验证；身份持久化、其他领域 runtime 和 Worker 证据仍缺失，因此整体业务能力尚不属于 `local_operable` |
| R1-X01 | AI/知识库、外部 AI 流程不在 Release 1；第 3 章 | `/ai`、`/admin/knowledge` | `/api/knowledge` | `modules/future/domain/feature-contracts.ts`、`modules/future/infrastructure/knowledge-db.ts` | 当前组合不创建知识库表 | `future-scope` 架构测试验证 Sidebar 只渲染不可点击占位；模块边界测试验证知识库 adapter fail closed | `contract_only`：未来功能在 Release 1 导航中不可点击，Knowledge adapter 拒绝执行且不再请求时建表；直接页面路由的统一拒绝边界仍需后续核验 |

## 4. 16 字段评估差异

当前 `/students/[studentId]` 展示的 16 个字段来自 `types/index.ts` 和 `modules/crm/infrastructure/mock-students.ts`。正式评估契约来自四个已批准 schema，共 15 个结构化字段：

- `student-profile.v1.json`：3 个；
- `education-profile.v1.json`：3 个；
- `family-context.v1.json`：4 个；
- `school-preferences.v1.json`：5 个。

两组字段数量和语义不同，不能通过改名直接迁移：

| 当前 16 字段 | 正式归属或候选映射 | 结论 |
| --- | --- | --- |
| `applicant_name` | `crm_students.display_name` | 属于 Student 身份，不应重复保存为评估答案 |
| `age` | 由 `crm_students.date_of_birth` 在查看时计算 | 不应保存会随时间失真的年龄文本 |
| `highest_education_institution` | 当前正式 schema 无“在读学校”字段 | `gap`：需要产品决定是否新增结构化字段或独立就读记录 |
| `prior_education` | 与 `education_profile.current_stage/current_year_level/current_curriculum` 只有部分重合 | `gap`：历史教育经历不能被当前阶段字段替代 |
| `major` | K12 当前范围不适用 | 候选移出 Release 1 评估 |
| `gpa` | 当前正式 schema 无直接字段 | `gap`：需要决定成绩表达方式、量表和证据 |
| `english_level` | `student_profile.primary_languages` 不是语言能力 | `gap`：不能错误映射 |
| `english_test_score` | 当前正式 schema 无直接字段 | `gap`：需要决定考试类型、分数、日期和证据结构 |
| `work_experience` | K12 当前范围通常不适用 | 候选移出 Release 1 评估 |
| `industry` | K12 当前范围不适用 | 候选移出 Release 1 评估 |
| `awards_or_experiences` | 当前正式 schema 无直接字段 | `gap`：自由文本还是版本化经历记录需决定 |
| `study_abroad_purpose` | 与 `family_context.education_priority` 只有部分重合 | `gap`：两者语义不能直接合并 |
| `target_institutions` | `cases_school_targets` | 应由逐校 SchoolTarget 表示，不应保存为逗号分隔评估文本 |
| `preferred_major` | K12 当前范围不适用 | 候选移出 Release 1 评估 |
| `other_requirements` | 当前正式 schema 无万能备注字段 | `gap`：需决定是否允许受控备注以及可见性、审计和保留规则 |
| `planned_enrollment_time` | `cases_service_cases.intake_year` 只能覆盖年份 | `partial`：若业务需要月份或学期，需新增明确字段，不能只保留显示字符串 |

当前结论：保留旧页面作为 Mock 证据，但在字段决策完成前，不把这 16 个字段写入正式 migration，也不把它们直接接到 `/api/v1/cases/[caseId]/assessment`。

## 5. 静态审计识别的主要缺口

| Gap | 优先级 | 影响 | 退出条件 |
| --- | --- | --- | --- |
| GAP-01：没有 `local-synthetic` composition root | P0 | 几乎所有 v1 业务路由默认不可用 | 本地模式能显式装配 Postgres、对象存储/队列、扫描器和开发身份；非本地模式继续 fail closed |
| GAP-02：Compose/Colima 与空库迁移底座 | 已关闭 | PostgreSQL 17、LocalStack、ClamAV 已健康运行，15 份迁移已从空库重放 | 保持 runbook、manifest、ledger、权限和 readiness 回归证据 |
| GAP-03：合成身份尚未持久化 | P0 | 进程重启会丢失本地 session，无法形成可重复的数据库身份基线 | 合成组织、用户、membership、role binding 和 session 由本地 PostgreSQL 管理；生产 Cognito 保护不被削弱 |
| GAP-04：缺少核心 read model | P0 | Student、Case、Task、Document 无法从目标 API 列表/查看 | 为首个纵向切片定义最小查询 API，页面不再读 Mock/legacy service |
| GAP-05：16 字段与正式 15 字段 schema 冲突 | P0 产品决策 | 直接开发会固化错误数据模型 | 对 16 个字段逐项作保留、改名、迁移或移出决定，并形成版本化 schema |
| GAP-06：学校数据存在 legacy 与目标两套写入路径 | P1 | 审核决定和工单仍依赖 Neon 请求时建表 | 定义快照转换/重新发布规则，并把可写状态迁入受迁移管理的本地 Postgres |
| GAP-07：未来功能直接路由边界仍需核验 | P1 | 导航已不可点击，但仍需证明直接 URL 不能进入 Release 1 功能 | 知识库/AI 直接路由在当前组合中统一拒绝访问，并保留架构与浏览器证据 |
| GAP-08：测试主要使用 fake，没有当前本地端到端证据 | P1 | 无法证明页面、API、数据库和 worker 共同可用 | 首个开发切片完成聚焦单元/集成、真实本地 Postgres 和浏览器证据 |

## 6. 阶段 0.2 结论与下一确认点

当前仓库不是“什么都没做”：领域契约、迁移、命令服务和聚焦测试覆盖面较广。但它也不是“已经基本可用”：页面仍大量依赖 Mock/旧路径，目标 runtime 尚未装配，本轮没有发现可以认定为 `local_operable` 的核心端到端能力。

因此不建议从某个页面样式或孤立功能直接开工。阶段 1 本地运行底座当前进度为：

1. `local-synthetic` 配置边界已完成；
2. PostgreSQL 17、LocalStack 和 ClamAV 的 Compose 环境已完成；
3. 15 份迁移的空库重放与权限验证已完成；
4. 可切换的本地角色登录和 Cognito 登录边界已完成；
5. 业务模块已统一按 `domain / application / infrastructure` 分层，并建立跨模块公开入口门禁；
6. 下一步把确定性合成身份和 session 持久化到本地 PostgreSQL。

`GAP-05` 的字段冲突仍未解决，不影响先完成身份入口，但在开发正式 Assessment 写入前必须决策。
