# Tianxingguoji Release 1 需求追踪矩阵

| 字段 | 内容 |
| --- | --- |
| 文档状态 | `accepted_stage1_input_amended` |
| 阶段 | 接手阶段 0.2 已完成；2026-08-24 工作流修订后作为差异规划输入 |
| 审计日期 | 2026-08-17；2026-08-18 刷新身份和模块路径进度；2026-08-24 同步 `DEC-071`/`DEC-073`/`DEC-074`/`DEC-075` |
| 需求依据 | `K12_CASE_AND_SCHOOL_APPLICATION_WORKFLOW.zh-CN.md`、已修订的 `RELEASE1_REQUIREMENTS_READER.zh-CN.md` 与 accepted/amended 决策 |
| 代码基线 | 阶段 0.2 初始审计为 `3410b8f366de946546d9f4958010febb734c1216`；2026-08-18 模块路径刷新对应 `e51440e7892e50ea115a35274dff31e56fe00828` |
| 文档基线 | 本地 `txgj-doc` 提交 `84a3826b18d32b81c88cde831ce251ce4944538c` |
| 证据范围 | 静态审计后已补充本地 Compose、25 条增量迁移 ledger、62 张 public 表、真实 PostgreSQL 身份/约束测试、应用 readiness，以及阶段 2A 至 2C 的浏览器与数据库证据；未运行 lint、build、云端、真实数据或 25 份迁移全量空库恢复验证 |

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
| `local_operable` | 在当前本地环境完成页面、API、模块、数据库和聚焦测试验证；只描述已明确验收或记录的纵向切片，不外推到整个需求 |
| `requirements_amended_not_implemented` | 业务需求已被后续决策修订；现有代码或测试仍证明旧合同，必须先完成差异设计再实施和重新验收 |

## 3. 总体追踪矩阵

| ID | 需求与决策来源 | 页面 | 目标 API | 领域模块 | 主要数据库表 | 现有测试证据 | 当前状态与主要缺口 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| R1-01 | 邀请制账号、角色、会话、案件授权；第 5 章，`DEC-007` 至 `DEC-011`、`DEC-020`、`DEC-029` | `/login`、`/login/activate`、`/admin/access` | `/api/v1/auth/*`、`/api/v1/cases/[caseId]/collaborators*` | `modules/identity`、`modules/access` | `identity_users`、`identity_sessions`、`identity_invites`、`access_*` | `identity-access-schema`、`identity-onboarding`、`identity-revoke-workflow`、`collaborator-scope-workflow`、`auth/mode`、`auth/runtime`、`local-synthetic-identity-postgresql` | `foundation_runtime_partial`：本地五角色身份和 opaque Session 已由受限 PostgreSQL Repository 管理，并通过 Next.js 重启、`/auth/me` 和登出验证；正式邀请激活和 Access 业务 runtime 仍未接通，真实 Cognito/RDS 也未执行 |
| R1-02 | Student 与 ServiceCase 分离、Guardian 独立、主联系人关系，以及 Student 可空性别资料与逐校申请前校验；第 4 章，`DEC-004` 至 `DEC-006`、`DEC-075` | `/students`、`/students/[studentId]`、`/students/[studentId]/guardians` | `/api/v1/students`、`/api/v1/students/[studentId]`、`/api/v1/students/[studentId]/guardians`、`.../primary-handoffs` | `modules/crm` | `crm_students`、`crm_guardians`、`crm_student_guardian_relationships`、`crm_referral_sources` | `crm-schema`、`guardian-relationship-workflow`、`duplicate-merge-workflow`、`local-synthetic-crm-postgresql` | `requirements_amended_partial`：阶段 2A 已让 Student 列表和详情通过 API v1 读取 PostgreSQL；当前 `crm_students`、DTO、页面和测试尚无 `gender`，须追加可空字段、四值验证、旧资料兼容和逐校申请前校验；新增/编辑 Student、Guardian 页面和主联系人交接 runtime 仍未接通 |
| R1-03 | K12 ServiceCase 从线下签约完成后开始，不含 Lead／Quote／销售 Contract 实体；案件里程碑、首校提交前暂停/恢复、Founder 后 Guardian 选校确认、Primary Advisor 代录、确认后改校、终止/重签、人工结案、版本冲突与幂等；第 6 章及完整流程文档，`DEC-004`、`DEC-027`、`DEC-032`、`DEC-044`、`DEC-054`、`DEC-071` | `/cases`、`/cases/new`、`/cases/[caseId]`、`/cases/[caseId]/workspace` | `/api/v1/cases`、`/api/v1/cases/[caseId]`、`/api/v1/cases/[caseId]/transitions`；暂停/恢复、名单审批/改版、终止及 Primary Advisor 代录确认 API 待差异设计 | `modules/cases`；已签合同文件复用 `modules/documents` | `cases_service_cases`、`cases_service_case_transition_facts`、`cases_assessments`、`shared_idempotency_records`、`audit_events`、`audit_outbox`；暂停/恢复、名单版本、终止和确认记录表示法待设计，优先复用现有聚合/版本/audit；不新增销售 Lead／Quote／Contract 表 | `case-creation-workflow`、旧 `case-transition-workflow`、`postgresql-case-transition-repository`、`case-workspace-service`、`PHASE2A_CRM_CASE_VERTICAL_SLICE.md`、`PHASE2C_CASE_STAGE_TRANSITION.md` | `requirements_amended_not_implemented`：旧实现要求 Assessment 完成后才进入 `background_collection`；还缺少新里程碑、暂停/恢复、自由文字原因、顺序确认/代录、确认后改校保留/新增/撤回、整体终止、重签新案、两分支人工结案和迁移测试。签约前销售流程已明确排除；其余业务语义已确认，剩余表示法属于技术设计 |
| R1-04 | 版本化四层 K12 评估、正式 15 字段、明确语义状态、阶段 blocker 与角色可见性；第 7 章，`DEC-012`、`DEC-013`、`DEC-074` | `/cases/[caseId]` 的正式 AssessmentEditor；旧 16 字段 Student preview 不作为正式契约 | `/api/v1/cases/[caseId]/assessment`、`.../assessment/background-completion` | `modules/cases/application/assessment-service.ts`、`modules/cases/infrastructure/postgresql-assessment-repository.ts`、`schema/k12` | `cases_schema_manifests`、`cases_schema_manifest_fields`、`cases_assessments`、`cases_assessment_answers` | `k12-catalogue`、`assessment-workflow`、`assessment-runtime-boundary`、`PHASE2B_ASSESSMENT_VERTICAL_SLICE.md` | `requirements_amended_partial`：15 字段和 enum receipt 已存在，但当前完成门禁把任何已有答案都视为满足 blocker，且 PostgreSQL 实现允许 Founder/Admin 写入、未接通 Collaborator grant；须改为 unknown/declined 不通过、仅 manifest 明确允许的 not_applicable 可通过、Primary Advisor/获授权 Collaborator 可写、Founder/Admin 只读 |
| R1-05 | SchoolTarget 独立流程、可选面试、候补/offer 语义、Guardian offer 决定、Primary Advisor 代录、确认后改校和人工结案前提；第 8 章及完整流程文档，`DEC-027`、`DEC-058`、`DEC-071` | 案件详情和 workspace 中的学校目标区域 | `/api/v1/cases/[caseId]/school-targets*`；Primary Advisor 代录 Guardian offer 决定及名单改版 API 待差异设计 | `modules/cases/application/school-target-service.ts`、`modules/cases/application/outcome-service.ts` | `cases_school_targets`、`cases_case_outcomes`、`schools_resolved_revisions`；确认方式/时间/操作人、名单版本和可选证据表示法待设计 | 旧 `school-target-workflow`、`case-target-outcome-workflow` | `requirements_amended_not_implemented`：缺少新终态、非终态候补/录取、跳过面试、Guardian 决定代录，以及确认后保留不变目标、新增目标、进行中移除目标 `withdrawn` 且已终态结果不可改写的合同；runtime 默认不可用 |
| R1-06 | 两类逐校自动 Task、Primary Advisor 人工临时 Task、暂停时保留原 Task、改校/终止时取消并保留历史、同 Task 拒绝/重派、计算型逾期提示、无逐项 Founder 验收、Advisor/Contractor 面试辅助脱敏工作区及提交完成门禁；第 9 章及完整流程文档，修订后的 `DEC-028`、`DEC-071` | `/tasks`、`/cases/[caseId]#tasks`、`/contractor/tasks/[taskId]` | `/api/v1/tasks/[taskId]/transitions`、`/api/v1/contractor/tasks/[taskId]`；SchoolTarget 触发、名单改版/终止、暂停联动和申请提交事实接口待差异设计 | `modules/tasks` 与 `modules/cases` 的公开边界 | 复用 `tasks_transition_policies`、`tasks_transition_rules`、`tasks_tasks`、`tasks_task_assignments`、`tasks_task_transition_receipts`；不新增面试角色/实体 | 旧 `task-workflow`、`contractor-task-workspace`、`tasks/transitions` | `requirements_amended_not_implemented`：缺少两类自动 Task、幂等触发、提交证据门禁及各联动；旧 policy 把 `reassigned`、`approved`、`overdue` 当状态，重派后无后续转移且要求 Founder 逐条审批，与新规则冲突。Contractor 仅可担任面试辅助并使用 Task 脱敏 DTO；runtime 仍需差异实现和验证 |
| R1-07 | 私有文件、隔离扫描、版本、软删除和恢复；逐校材料清单与提交凭证复用现有 SchoolTarget/Task/Case Document；Founder、当前 Primary Advisor 和该校当前 Application Assignee 可下载必要文件，面试辅助人和 Portal 不可下载；第 10 章，`DEC-017`、`DEC-071` | `/documents`、`/cases/[caseId]#documents` | `/api/v1/cases/[caseId]/documents/upload-intents`、`.../deletions`、`.../restorations`、`.../version-rollbacks`；Application Assignee 限校下载授权待差异设计 | `modules/documents`、`modules/tasks`、`modules/cases`、`workers/scan-document.ts`、`workers/reconcile-documents.ts` | 复用 `documents_documents`、`documents_document_versions`、`documents_scan_results`；不新增 Material／SubmissionEvidence 表 | `document-upload-workflow`、`document-scan-workflow`、`document-version-workflow`、`document-store` | `requirements_amended_not_implemented`：本地 S3/SQS/ClamAV 底座已健康运行，但页面仍是 preview，Document runtime 与文件列表 read model 未完整接通；旧权限只覆盖 Founder/Primary Advisor，尚缺 Application Assignee 限校下载、随负责人／既有授权失效撤权和提交凭证门禁 |
| R1-08 | 站内通知、事件收件人、到期前 3 天／1 天与每日逾期提醒、暂停期继续提醒及追加式审计；第 9、11 章，`DEC-032`、`DEC-033`、`DEC-062`、`DEC-071`、`DEC-073`、`DEC-074` | 当前没有通知中心或审计查看页 | 当前没有独立 v1 通知/审计查询 API | `modules/notifications`、`modules/audit`；现有 `workers/deliver-in-app.ts` 只覆盖站内路径，符合当前外部通知边界 | `audit_events`、`audit_outbox`、`notifications_notifications`、`notifications_delivery_receipts` | `outbox-audit`、`in-app-notification-delivery` | `requirements_amended_not_implemented`：原子效果契约和站内 worker 测试存在，但尚缺 DEC-074 的 Task／名单／结案分支触发、精确收件人、3 天／1 天提醒、每日逾期与按接收人/effect/date 去重；默认通知 runtime 仍不可用，且不得新增 Guardian/Student 外部通知 |
| R1-09 | Founder 看板显示案件阶段、责任人、下一任务、截止和异常；第 11 章，`DEC-052`、`DEC-057` | `/dashboard`、`/today` | `/api/v1/dashboard/cases` | `modules/operations/domain/case-dashboard-projection.ts`、`modules/operations/infrastructure/case-dashboard-route.ts` | 从 `cases_service_cases`、`tasks_tasks` 等权威表生成投影视图 | `case-dashboard-projection`、`case-dashboard-route` | `mixed_partial`：dashboard 已调用 v1 API但默认 runtime 不可用；today 仍读取 preview adapter；尚未验证空库和合成数据页面 |
| R1-10 | 学校快照、筛选、人工审核、provisional School 和 overlay 治理；第 8、14 章，`DEC-014` 至 `DEC-016` | `/selector`、`/schools`、`/admin/crawler`、`/admin/schools` | 旧 `/api/crawler/*`；目标 `/api/v1/schools/*`、`/api/v1/admin/schools/*` | `modules/schools/infrastructure/crawler`、`modules/schools` | 旧 `crawler_*` 请求时建表；目标 `schools_schools`、`schools_snapshots`、`schools_snapshot_records`、`schools_overlay_*`、`schools_resolved_revisions` | `crawler/snapshot-manifest`、`school-change-workflow`、`school-governance-workflow`、`school-target-workflow`、`schools/resolver` | `legacy_partial`：已提交快照可由旧 API 读取，审核决定和工单仍走 Neon 且请求时建表；当前 manifest 是旧 warning 格式；v1 学校 runtime 默认不可用 |
| R1-11 | 单案件、限时、字段白名单的只读家长 Portal；第 12 章，`DEC-064`、`DEC-065` | `/portal/access`、`/portal/workspace`、`/cases/[caseId]/access` | `/api/v1/portal/sessions`、`/api/v1/portal/workspace`、`/api/v1/cases/[caseId]/portal-grants*` | `modules/external-portal` | `portal_viewers`、`portal_access_grants`、`portal_sessions`、`portal_security_events`、`portal_idempotency_records` | `portal-api-routes`、`portal-schema-contract`、`portal-repository-contract`、`portal/contract-policy`、`portal-pages` | `contract_only`：页面、allowlist、路由和测试存在；默认路由显式返回 runtime unavailable，尚无本地持久化组合和浏览器验收 |
| R1-12 | 平台只统计推进中案件数量并显示合同参考值，不计算金额；第 13 章，`DP-06`、`DP-09`、`DP-11`、`DEC-071`、`DEC-074` | `/platform/billing` | `/api/v1/platform/billing/overview` | `modules/platform-billing` | `platform_billing_contract_versions`、`platform_billing_metric_snapshots`、`platform_billing_subscription_projections`、`cases_billing_projection_events` | `platform-billing-persistence`、`platform-billing-overview-route`、`platform-billing-schema-contract`、`platform-billing/contract-policy` | `requirements_amended_not_implemented`：新映射已确认计入三个进行中里程碑（含暂停和全拒未结案），排除 signed、整案终止待结案、pending_delete、closed；旧 policy 未实现该映射，默认 runtime 不可用，也未从本地案件事件生成月底快照 |
| R1-13 | 使用受控事件逐案重建合成/未来既有案件；第 3、16 章，`DEC-061`、`DEC-067` | `/cases/reconstructions/new`、`/cases/reconstructions/[reconstructionId]` | `/api/v1/cases/reconstructions` | `modules/cases/domain/reconstruction`、`modules/cases/application/reconstruction`、`modules/cases/infrastructure/reconstruction` | `cases_reconstructions`、`cases_reconstruction_versions`、`cases_reconstruction_events`、`cases_reconstruction_gaps`、`cases_reconstruction_activations` | `case-reconstruction-workflow`、`case-reconstruction-route`、`case-reconstruction-schema`、`case-reconstruction-ui-model` | `contract_only`：事件契约、UI 模型、路由和表存在；默认 runtime 不可用；当前阶段只允许合成数据，不授权真实案件 |
| R1-14 | 明确互斥的 `local-synthetic` 运行组合；阶段 0.1 当前范围 | 所有 Release 1 页面 | 为上述 v1 路由提供本地依赖；`/api/v1/local/readiness` | fail-closed 本地配置、迁移 runner、依赖探测、身份 mode adapter，以及 Identity、CRM、Case/Assessment/首个 Case transition 的 PostgreSQL 本地 Repository | PostgreSQL 17；LocalStack 模拟 S3/SQS；ClamAV | 25 条 migration ledger、62 张 public 表；五角色 Session、CRM/Case 2A、Assessment 2B 和 Case transition 2C 本地证据；应用 readiness 五项全 ready | `foundation_runtime_partial`：底座和首批内部纵向切片可运行；SchoolTarget、Task、Document、Worker、Portal 和 Platform Billing 等其他 runtime 仍缺失 |
| R1-X01 | AI/知识库、外部 AI 流程不在 Release 1；第 3 章 | 无功能页面；导航只允许不可点击占位 | 无 Release 1 API | `modules/future/domain/feature-contracts.ts` | 无知识库表或 persistence adapter | `future-scope` 验证导航占位；`module-boundaries` 验证 AI/Knowledge 页面、Route Handler 和 persistence adapter 不存在 | `contract_only`：只保留运行时无关的禁用契约与导航占位元数据；Release 1 不暴露页面、API、job、credential 或 data-write 表面；浏览器直接 URL 的 404/不可用证据待里程碑验收补充 |

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
| GAP-02：Compose/Colima 与空库迁移底座 | 已关闭 | PostgreSQL 17、LocalStack、ClamAV 已健康运行；最初 15 份迁移已从空库重放，当前增量 ledger 为 25 | 保持 runbook、manifest、ledger、权限和 readiness 回归证据；恢复里程碑再补 25 份全量空库重放 |
| GAP-03：合成身份尚未持久化 | 已关闭 | 固定组织、五个用户、membership、role binding 和 Session 已由本地 PostgreSQL 管理，并通过进程重启验证 | 保持最小权限、RLS、Cognito token 边界、幂等 seed 和聚焦回归证据 |
| GAP-04：缺少核心 read model | 部分关闭 | Student 与 Case 已由 API v1/PostgreSQL 提供列表和详情；Task、Document 等模块仍不可用 | 各后续模块逐一建立最小查询 API，页面退出 Mock/legacy service |
| GAP-05：16 字段与正式 15 字段 schema 冲突 | 正式路径已关闭 | 阶段 2B 明确采用四层 15 字段版本化 schema，旧 16 字段 preview 不写入正式 Assessment | 后续单独退出旧 preview；新增业务字段必须走版本化 schema 决策，不直接映射旧字段 |
| GAP-06：学校数据存在 legacy 与目标两套写入路径 | P1 | 审核决定和工单仍依赖 Neon 请求时建表 | 定义快照转换/重新发布规则，并把可写状态迁入受迁移管理的本地 Postgres |
| GAP-07：未来功能直接路由边界 | 架构边界已关闭；浏览器证据待补 | 页面、API 和 persistence adapter 已从活跃源码移除，架构测试阻止重新引入；尚未执行里程碑浏览器 404/不可用验证 | 保持源码缺失与架构测试，并在获批的浏览器验收批次补充直接 URL 证据 |
| GAP-08：测试主要使用 fake，没有当前本地端到端证据 | P1 | 无法证明页面、API、数据库和 worker 共同可用 | 首个开发切片完成聚焦单元/集成、真实本地 Postgres 和浏览器证据 |
| GAP-09：`DEC-071`／`DEC-074` 新业务流程尚未进入实现 | P0 | 当前案件状态、SchoolTarget 终态、选校确认、自动 Task、offer 决定、人工结案、Assessment 权限／blocker、站内通知、Dashboard 和平台计数仍可能执行或展示旧合同 | 客户业务问题已全部确认；下一步先批准完整差异计划，再以追加 migration/兼容方案更新领域、API、权限、审计、站内通知、UI 和测试，并用合成数据完成本地端到端验证 |
| GAP-10：`DEC-075` Student 性别资料尚未实现 | P1 | CRM 无法保存学校申请可能要求的性别资料，也无法在逐校申请准备前识别缺失或无法映射的值 | 以追加 migration 为旧 Student 保留 `null`，同步更新建档／编辑 DTO、验证、页面、授权、审计 redaction、逐校准备门禁和合成测试；不得自动推断或仅凭此字段移除 SchoolTarget |

## 6. 阶段 0.2 结论与下一确认点

当前仓库已经形成首批可本地操作的纵向切片：身份与 Session、CRM Student 读取、既有 Student 建案、正式 Assessment 背景收集，以及第一组旧合同下的相邻案件阶段推进/回退。`DEC-071` 之后，这些案件阶段证据只能证明历史实现，不能证明新流程完成。Guardian 编辑、新案件里程碑、选校双重确认、SchoolTarget、自动 Task、offer 决定、人工结案、Document、Worker、Portal 和 Platform Billing 等仍未贯通。

因此不建议从某个页面样式或孤立功能直接开工。阶段 1 本地运行底座当前进度为：

1. `local-synthetic` 配置边界已完成；
2. PostgreSQL 17、LocalStack 和 ClamAV 的 Compose 环境已完成；
3. 最初 15 份迁移的空库重放已完成，当前 25 条增量 ledger 与身份权限验证已完成；
4. 可切换的本地角色登录和 Cognito 登录边界已完成；
5. 业务模块已统一按 `domain / application / infrastructure` 分层，并建立跨模块公开入口门禁；
6. 确定性合成身份和 Session 已持久化到本地 PostgreSQL，并通过 Next.js 重启验证；
7. 阶段 2A CRM/Case、阶段 2B Assessment 背景收集和阶段 2C 首个案件阶段命令均已验收。

`GAP-05` 已通过采用正式 15 字段版本化 schema 关闭；旧 16 字段页面只作为待退出 preview，不得反向成为正式数据契约。
