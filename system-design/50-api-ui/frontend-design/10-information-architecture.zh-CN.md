# Release 1 前端信息架构与页面范围

状态：`approved`
Owner：`frontend-design`
Architect：当前业务事实源对话
确认依据：项目负责人于 2026-08-26 明确确认 F0
业务基线：`BR-BASELINE-20260825-v32`

返回 [API 与页面交互设计索引](../README.md)。

## 1. 先看结论

本轮只冻结前端信息架构，不写产品源码、不画高保真视觉稿。

目标是一个面向内部员工的安静、实用、工作导向 ERP：

- 以案件和待办为工作入口；
- 以 Student、Guardian、Case、SchoolTarget、Task、Document 为业务对象；
- 所有页面操作受服务端身份、组织范围、角色和 capability 约束；
- Guardian Portal 是独立的单案件只读入口，不是内部 ERP 的一个角色页面。

Release 1 明确不出现：Platform Billing、推进中案件计数、合同参考值、订阅/平台财务角色、Lead/Quote/Contract 销售生命周期、Data Reviewer、AI、非 K12、多组织、Portal 写入和外部业务 Email。

## 2. 页面边界

| 区域 | 使用者 | 能做什么 | 不能做什么 |
| --- | --- | --- | --- |
| 内部 ERP | Founder、Admin、Advisor、Contractor | 处理组织内 CRM、Case、学校、Task、文件、通知、审计和运营 | 不能绕过服务端授权；Admin 默认不查看 Case/Assessment；Contractor 只处理被分派 Task |
| Guardian Portal | Guardian Viewer（临时授权查看者） | 查看单个 Case 的允许阶段、学校进度、家长可见消息和行动项 | 不注册内部账号；不写入、不上传、不查看文件、不查看内部备注；授权 7 天且可撤销 |

## 3. 一级导航与目标路由树

### 3.1 内部 ERP

```text
/today                         今日工作台（个人待办、风险、即将到期）
/cases                         案件
  /cases/new                   新建 K12 案件
  /cases/:caseId               案件概览
  /cases/:caseId/assessment    Assessment 与背景资料
  /cases/:caseId/schools       候选学校名单、版本、Founder 审批、家长确认
  /cases/:caseId/applications  逐校申请、申请状态、准备/提交任务
  /cases/:caseId/interviews    面试安排、辅助面试人和面试任务
  /cases/:caseId/documents     案件文件
  /cases/:caseId/access        Guardian Portal 授权
  /cases/:caseId/close         人工结案
/students                      Student 与 Guardian
  /students/new                原子建立 Student + 主要 Guardian
  /students/:studentId         Student 详情
  /students/:studentId/guardians 关系、主要联系人交接、解除关系
  /referral-sources            来源目录
/schools                       学校目录、公开快照和已审核资料
/tasks                        Task 工作台
  /tasks/:taskId               Task 详情
/documents                     文件工作台
/notifications                 站内通知
/operations                    运营看板、技术健康和可重建告警
/audit                         业务审计查询（Founder）
/admin/access                 员工资料、角色和 capability 管理（Admin）
```

F1 路由决策补充：Contractor 登录后默认进入 `/tasks`；显式访问 `/today` 时返回统一 denied 页面和“返回我的任务”链接。Case workspace 的页面 canonical 为 `/cases/:caseId`，其余 Case tab 使用上列独立子路由并共享 Case layout；旧 `/cases/:caseId/workspace` 以 308 重定向到 canonical。页面路由不改变聚合 API `GET /api/v1/cases/:caseId/workspace`。

### 3.2 Guardian Portal

```text
/portal/access                 输入临时访问凭据
/portal/workspace              单 Case 只读工作区
```

Portal 路由不得显示在内部 ERP 一级导航中，也不得通过内部 role selector 进入。

## 4. 角色入口与主要操作

| 角色 | 默认入口 | 允许的主要操作 | 明确拒绝 |
| --- | --- | --- | --- |
| Founder | `/today`、`/cases`、`/schools`、`/audit`、`/operations` | 审批候选学校名单和学校资料变更；查看组织案件和审计；人工结案；管理 Portal 撤销 | 不因 Founder 身份自动获得文件修改权或 Portal 外部身份 |
| Admin | `/today`、`/admin/access`、技术健康 | 管理员工资料、邀请/停用、角色与 capability；查看 technical health | 默认不可查看 Case、Assessment、Student/Guardian 内容或文件 |
| Advisor | `/today`、`/cases`、`/students`、`/tasks` | 建档、建案、填写 Assessment；建立候选学校名单；执行申请准备/提交；创建 Portal 授权；处理自己案件 | 不能批准自己的候选名单；不能绕过 Founder 审批；不能查看不在范围内的 Case |
| Contractor | `/tasks` | 查看被分派的 Task；完成或拒绝；上传被任务授权的结果 | 不能浏览全局 Task、Case、Student、Guardian、文件目录或管理分派 |
| Guardian Viewer | `/portal/access` | 查看单 Case 允许字段和家长行动项 | 无内部角色；不能写入、上传、下载文件、留言或查看内部备注 |

Founder + Admin 是同一员工的多角色组合：导航和按钮按当前 capability 合并显示，服务端逐操作检查；不能把组合身份简化成一个 role 字符串。

## 5. 当前路由盘点与目标处理

当前 `app/**` 共 34 个非 API 页面路由；导航 registry 当前有 9 个正式入口和 4 个 Future 占位。以下分类只描述目标处理，不代表当前实现已经满足目标。

| 当前路由 | 源码证据 | 分类 | 目标处理 |
| --- | --- | --- | --- |
| `/` | `app/page.tsx` | `redirect` | 登录后按 capability 跳转：Contractor 到 `/tasks`，其他内部角色到 `/today`；未登录到 `/login` |
| `/login` | `app/(auth)/login/page.tsx` | `keep` | 统一内部登录；不显示 role selector |
| `/login/activate` | `app/(auth)/login/activate/page.tsx` | `keep` | 邀请激活和首次凭据设置 |
| `/today` | `app/(erp)/today/page.tsx` | `redesign` | 去除 preview adapter，改为真实 Today 查询；Contractor 显式访问返回 denied + `/tasks` 链接 |
| `/dashboard` | `app/(erp)/dashboard/page.tsx` | `redesign` | 合并到 `/operations` 或明确为个人工作台，避免重复入口 |
| `/cases` | `app/(erp)/cases/page.tsx` | `redesign` | 案件列表按授权范围过滤 |
| `/cases/new` | `app/(erp)/cases/new/page.tsx` | `redesign` | Student/Guardian/Assessment 建案连续流程 |
| `/cases/[caseId]` | `app/(erp)/cases/[caseId]/page.tsx` | `redesign` | 案件概览与阶段事实 |
| `/cases/[caseId]/workspace` | `app/(erp)/cases/[caseId]/workspace/page.tsx` | `redirect` | 以 308 重定向到 canonical `/cases/:caseId`；不渲染第二套工作区 |
| `/cases/[caseId]/access` | `app/(erp)/cases/[caseId]/access/page.tsx` | `keep` | Primary Advisor 创建、Founder/Advisor 撤销 Portal 授权 |
| `/cases/reconstructions/new` | `app/(erp)/cases/reconstructions/new/page.tsx` | `isolate_from_release1` | 旧重建入口不得出现在正式导航；保留历史追溯 |
| `/cases/reconstructions/[reconstructionId]` | `app/(erp)/cases/reconstructions/[reconstructionId]/page.tsx` | `isolate_from_release1` | 旧重建详情不作为 Release 1 正式页面 |
| `/students` | `app/(erp)/students/page.tsx` | `redesign` | Student/Guardian CRM 列表 |
| `/students/new` | `app/(erp)/students/new/page.tsx` | `redesign` | 原子建立 Student + 唯一主要 Guardian |
| `/students/[studentId]` | `app/(erp)/students/[studentId]/page.tsx` | `redesign` | 主档、关系、案件摘要；deleted 不返回 |
| `/students/[studentId]/guardians` | `app/(erp)/students/[studentId]/guardians/page.tsx` | `redesign` | 关系版本、主要联系人交接、解除 |
| `/students/deletion-requests` | `app/(erp)/students/deletion-requests/page.tsx` | `keep` | Advisor 申请、Founder 决定；只软删除 |
| `/students/duplicates` | `app/(erp)/students/duplicates/page.tsx` | `isolate_from_release1` | 只保留即时重复警告，不提供合并工作台 |
| `/students/duplicates/[candidateId]` | `app/(erp)/students/duplicates/[candidateId]/page.tsx` | `isolate_from_release1` | 旧重复详情不进入正式入口 |
| `/referral-sources` | `app/(erp)/referral-sources/page.tsx` | `keep` | 来源目录管理 |
| `/referral-sources/[sourceId]` | `app/(erp)/referral-sources/[sourceId]/page.tsx` | `keep` | 来源详情和关联 Case |
| `/schools` | `app/(erp)/schools/page.tsx` | `redesign` | 学校目录、快照和审核 |
| `/admin/schools` | `app/(erp)/admin/schools/page.tsx` | `redirect` | 学校资料审核归 Founder；不保留 Admin 专属业务审批入口 |
| `/admin/access` | `app/(erp)/admin/access/page.tsx` | `redesign` | 员工资料、角色和 capability 管理；只显示 Admin 可操作内容 |
| `/admin/crawler` | `app/(erp)/admin/crawler/page.tsx` | `isolate_from_release1` | 调度/爬虫运维不进入当前 Release 1 正式导航 |
| `/selector` | `app/(erp)/selector/page.tsx` | `redesign` | 改为案件内候选学校名单工作区，不作为全局选校页 |
| `/tasks` | `app/(erp)/tasks/page.tsx` | `redesign` | Advisor/Contractor 按 capability 和 assignment 过滤 |
| `/tasks/[taskId]` | `app/(erp)/tasks/[taskId]/page.tsx` | `redesign` | 任务生命周期、拒绝、重派、完成和逾期 |
| `/contractor/tasks/[taskId]` | `app/(erp)/contractor/tasks/[taskId]/page.tsx` | `redirect` | 统一到 `/tasks/:taskId`，服务端保留 Contractor 资源边界 |
| `/documents` | `app/(erp)/documents/page.tsx` | `redesign` | 文件工作台；Case 是授权根，扫描未通过不可预览/下载 |
| `/platform/billing` | `app/(erp)/platform/billing/page.tsx` | `isolate_from_release1` | Platform Billing 整体排除；不得导航、读写或作为权限依据 |
| `/portal/access` | `app/(portal)/portal/access/page.tsx` | `keep` | 临时访问凭据；通用失败文案 |
| `/portal/workspace` | `app/(portal)/portal/workspace/page.tsx` | `keep` | 单 Case 只读白名单；不得展示文件或内部备注 |
| `/cases/__fixtures/workspace` | `app/(fixtures)/cases/__fixtures/workspace/page.tsx` | `isolate_from_release1` | 仅开发 fixture，不是正式入口 |

### 5.1 目标页面但当前缺失

以下是业务流程需要、但当前 `app/**` 尚无对应正式页面的目标入口，标记为 `missing`，不得由前端自行发明业务字段：

| 目标路由 | 来源 | 下一步 |
| --- | --- | --- |
| `/cases/:caseId/assessment` | `BR-030`、Cases API/UI | F1/F2 与案件工作区一起设计 |
| `/cases/:caseId/schools` | `BR-031`、`BR-032` | F3 |
| `/cases/:caseId/applications` | `BR-033`、`BR-035` | F3 |
| `/cases/:caseId/interviews` | `BR-036` | F3 |
| `/cases/:caseId/documents` | `BR-050` | F4；可先由 `/documents` 提供案件筛选 |
| `/cases/:caseId/close` | `BR-034`、`BR-039` | F3 |
| `/notifications` | `BR-038` | F4 |
| `/operations` | `BR-070` | F4 |
| `/audit` | `BR-071` | F4 |

另外，当前 navigation registry 的 `nav.dataReview` 和 4 个 Future 占位只作为隔离/不可点击元数据；Data Reviewer 不属于 Release 1，不生成页面或角色入口。

## 6. 核心流程的页面连续性

| 流程 | 页面顺序 | 不能跳过的事实 |
| --- | --- | --- |
| 员工与权限 | `/login` → `/admin/access` | 身份由服务端解析；Founder+Admin 可组合；Admin 默认不看业务内容 |
| CRM 建档 | `/students/new` → Student 详情 → Guardians | 主要 Guardian 必须唯一；重复只警告，不自动关联 |
| 建案评估 | `/students/:id` → `/cases/new` → `/cases/:id/assessment` | K12、正式 15 字段和 Assessment 版本由 Case 固定 |
| 候选学校 | `/cases/:id/schools` | Advisor 建名单 → Founder 批准/驳回修改 → 家长确认并留记录 |
| 逐校申请 | `/cases/:id/applications` | 准备申请产生 Task；指定 Advisor 负责准备和提交 |
| 面试 | `/cases/:id/interviews` → `/tasks/:taskId` | 需要面试时分配辅助面试人并产生任务；当前 Advisor 可自己完成 |
| 文件 | `/cases/:id/documents` 或 `/documents` | 上传、扫描、版本和用途授权；未通过扫描不可下载 |
| 学校资料 | `/schools` → Founder 审核 | 学校快照、人工变更和发布版本可追溯 |
| 通知审计 | `/notifications`、`/audit`、`/operations` | 通知去重；审计不可改写；Operations 不能改业务事实 |
| 结案 | `/cases/:id/close` | 人工结案；所有学校拒绝后仍允许新增学校或明确结案 |
| Portal | `/cases/:id/access` → `/portal/access` → `/portal/workspace` | 单 Case、7 天、可撤销、只读；不发外部业务 Email |

## 7. 页面状态基线

所有目标页面至少定义以下状态，且状态文案不能泄露无权访问对象是否存在：

| 状态 | UI 要求 |
| --- | --- |
| `loading` | 保留页面标题和结构骨架，避免跳动；写操作按钮防重复 |
| `empty` | 说明“还没有数据”和下一步动作；不把空数据当错误 |
| `error` | 显示 request id 和可重试动作；不显示堆栈、token 或 PII |
| `denied` | 通用无权提示；不暴露对象/角色存在性 |
| `unavailable` | 明确服务暂不可用；不静默切换到 memory/mock/preview |
| `stale/version conflict` | 显示当前版本已变化，要求重新加载；不得覆盖他人更新 |
| `success` | 显示已保存/已提交/已撤销的结果和下一步链接；以服务端返回为准 |

## 8. 桌面与移动端原则

- 桌面端：左侧一级导航，案件工作区采用固定上下文栏；列表筛选和详情操作保持稳定位置。
- 移动端：一级导航折叠为可访问菜单；案件关键操作固定在底部操作区；表格改为分组列表，不横向堆叠全部字段。
- 所有页面优先文本层级、状态、待办和下一步；不做营销落地页，不使用装饰性卡片堆叠。
- 颜色不能单独承担状态含义；按钮必须有文字或可读标签。

## 9. 后续设计批次

1. F1：Today、Cases 列表和案件工作区线框。
2. F2：Student/Guardian/ReferralSource 和建案向导线框。
3. F3：候选学校、家长确认、逐校申请和面试 Task 线框。
4. F4：Documents、Schools、Notifications、Audit/Operations 线框。
5. F5：Portal 两页线框与移动端状态稿。

每批只在本文件通过 Architect 验收后开始；不直接进入页面编码。

## 10. 返回 Architect 确认的问题

仅保留真正阻塞后续线框的问题：

1. `/dashboard` 是 `/today` 的兼容重定向，还是保留独立的 Operations 看板？
2. Founder 是否需要一个独立的“学校审核”入口，还是只从 `/schools` 和案件上下文进入？
3. 案件中的 Assessment、学校、申请、面试、文件是否采用标签页，还是采用纵向步骤导航？

以上问题不阻塞当前信息架构；其余页面行为可由 confirmed BR 和已批准 API/UI 文档确定。

## 11. 本轮证据与验收状态

- 当前路由盘点：34 个非 API `page.tsx`。
- 当前导航证据：9 个正式入口、4 个 Future 占位。
- 目标页面/入口：内部 ERP 10 个一级工作区 + 约 25 个上下文页面；Portal 2 个入口。
- 产品源码、测试、migration、数据库、云配置：`changed: none`。
- browser/database/cloud：`not_run`。
- Architect 验收：通过。34 个现有路由全部分类；角色边界、核心流程、页面状态和旧入口隔离均已覆盖。
- 本文档状态：`approved`；进入 F1 线框设计。
