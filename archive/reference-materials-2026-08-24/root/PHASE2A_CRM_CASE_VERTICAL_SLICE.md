# 阶段 2A：CRM Student 与 Case 最小纵向切片

| 项目 | 状态 |
|---|---|
| 日期 | 2026-08-18 |
| 状态 | `accepted_local`；2026-08-18 用户人工验收通过 |
| 数据范围 | 仅本地确定性合成数据 |
| 代码状态 | 本地未提交、未推送 |

## 1. 本阶段解决的问题

阶段 1 已经让本地身份、Session、PostgreSQL 和模块边界能够运行，但学生和案件页面仍没有形成一条真实业务链。阶段 2A 选择最小且可验证的切片：内部 Founder、Admin 或 Advisor 登录后，可以读取 CRM Student，并从一个既有 Student 建立 K12 ServiceCase。

这个切片只证明页面、API、应用服务、Repository、RLS 和数据库事务已经贯通，不代表整个案件办理流程已经完成。

## 2. 已实现范围

- 固定种子建立 2 名 Student、2 名 Guardian、2 条有效主要监护关系。
- 使用四个已批准的 K12 schema module 组成 1 个不可变的 15 字段 manifest。
- `/students` 和 Student 详情从 PostgreSQL 读取，不再使用 `mockStudents`。
- `/api/v1/students` 提供组织范围内的 Student 列表和详情读取。
- `/api/v1/cases/options` 提供建案所需的 Student、Primary Advisor 和已批准 manifest 选项。
- `/api/v1/cases` 支持案件列表和“既有 Student 建案”。
- `/api/v1/cases/{caseId}` 提供案件与 draft Assessment 详情。
- 建案事务同时写入 ServiceCase、draft Assessment、AuditEvent、Outbox 和 IdempotencyReceipt。
- 使用 `tianxing_app` 最小权限连接和 PostgreSQL RLS；应用不使用 migration owner 连接执行业务查询。
- 本地 readiness 新增应用数据库角色与合成业务基线检查。

## 3. 权限与数据边界

- Founder 和 Admin 可以读取组织内 Student/Case，并可选择有效 Primary Advisor 建案。
- Advisor 只能以自己作为 Primary Advisor 建案，案件列表和详情只显示自己负责的案件。
- Contractor 和其他未授权角色在进入 Repository 前被拒绝。
- Student 与 ServiceCase 继续是两个独立聚合：Student 保存长期 CRM 身份，ServiceCase 保存一次申请服务的年份、申请类型、负责人和流程状态。
- 幂等键由客户端为一次建案操作生成；重复请求不会建立第二个案件。

## 4. 明确未包含

- 新增或编辑 Student、Guardian，以及主要监护人交接。
- Assessment 答案录入、15 字段校验和 `background_complete` 门禁。
- 案件阶段变更、回退、关闭和 Founder 审批。
- SchoolTarget、Task、Document、Notification 的业务页面与写入链路。
- AWS Cognito、RDS、S3、SQS 或任何生产环境操作。
- 真实学生、家长、案件或文件数据。

## 5. 本地验证结果

- 聚焦单元、契约、迁移、集成和架构测试：47 项通过。
- 种子脚本重复执行成功，结果保持 2 名 Student、2 名 Guardian、2 条关系、1 个 manifest、15 个字段。
- 浏览器以 Founder 本地身份完成 Student 列表、Student 详情、建案、案件列表和案件详情流程。
- 建立 2 个合成测试案件；数据库中对应存在 2 个 Assessment、2 个 AuditEvent、2 个 Outbox 和 2 个 IdempotencyReceipt。
- Next.js 重启后 Session 仍有效，两个案件仍可读取，证明身份与业务数据均非进程内临时状态。
- 仓库全量严格 TypeScript 检查仍受既有无关错误影响；本阶段新增文件已排除在当前错误清单之外。完整基线清零不是 2A 的完成声明。

## 6. 验收结论与下一确认点

2026-08-18，用户完成并通过本地人工验收。验收范围包括：Student 与案件清楚分开、从既有 Student 建案、案件列表及详情，以及尚未接通的数据未被伪造展示。

下一步先提交 2A 代码和文档变更，再单独确认 2B。当前建议优先贯通 Assessment 15 字段答案、草稿保存和 `background_complete` 门禁；未经确认，不开始这一范围。
