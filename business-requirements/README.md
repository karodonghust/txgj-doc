# 天星国际业务需求唯一事实源

| 属性 | 内容 |
| --- | --- |
| 文档状态 | `active_single_source_of_truth` |
| 基线版本 | `BR-BASELINE-20260920-v55` |
| 生效日期 | 2026-09-20 |
| 决策人 | 项目负责人（本 Codex 对话用户） |
| 适用范围 | Tianxingguoji Release 1 业务逻辑 |
| 不代表 | 代码已经实现、数据库已经迁移、测试已经通过或生产已经上线 |

## 唯一事实源规则

1. 本 Codex 对话是业务决策的产生渠道；本目录是未来开发唯一允许引用的业务事实源。
2. 只有项目负责人在本对话中明确表示“同意”“确认”“接受”“按此执行”或同等含义的内容，才能写为 `confirmed`。
3. 助手建议、旧文档、当前代码、测试、页面和历史实现都不能自行成为业务需求。
4. `pending_review` 只表示待讨论，开发不得据此自行补全、建表、增加状态、增加角色或产生外部副作用。
5. 后续决定与本目录冲突时，以项目负责人最新确认的决定为准；同一轮必须更新对应模块文件和 `CHANGELOG.md`，并把旧规则标记为 `superseded`，不得静默覆盖历史。
6. 未来需求票、实现计划、API、DDL、测试和验收必须引用本目录中的 `BR-*` 编号。未找到对应 `confirmed` 规则时停止业务实现并回到本对话确认。
7. `/Users/karo/Documents/txgj-doc/archive/` 中的材料只用于本对话核对背景和发现问题，不得直接作为开发依据。
8. 开发对话默认只读取本索引、当前模块文件和被明确引用的跨模块 `BR-*`，不需要加载整个目录。

状态定义：

| 状态 | 含义 |
| --- | --- |
| `confirmed` | 已由项目负责人确认，可以进入后续差异设计和开发 |
| `pending_review` | 来自旧资料或尚未完成讨论，不得实现为正式业务规则 |
| `superseded` | 已被更新决定取代，只保留追溯用途 |

## 模块索引

| 文件 | BR 范围 | 内容 |
| --- | --- | --- |
| [`00-product-scope.zh-CN.md`](00-product-scope.zh-CN.md) | `BR-001`–`BR-003` | Release 1 目标、包含和排除范围 |
| [`10-identity-access.zh-CN.md`](10-identity-access.zh-CN.md) | `BR-010`–`BR-015` | 角色、员工资料、授权模型和内部账号邀请 |
| [`20-crm.zh-CN.md`](20-crm.zh-CN.md) | `BR-020`–`BR-029` | Student、Guardian、关系、建档和客户来源 |
| [`30-cases.zh-CN.md`](30-cases.zh-CN.md) | `BR-030`–`BR-034`、`BR-039` | 案件、Assessment、选校和结案 |
| [`40-tasks.zh-CN.md`](40-tasks.zh-CN.md) | `BR-035`–`BR-037` | 申请、面试和 Task 通用规则 |
| [`50-schools.zh-CN.md`](50-schools.zh-CN.md) | `BR-051` | 学校目录与数据变更 |
| [`60-documents.zh-CN.md`](60-documents.zh-CN.md) | `BR-050` | 文件、扫描、版本和下载权限 |
| [`70-notifications-audit.zh-CN.md`](70-notifications-audit.zh-CN.md) | `BR-038`、`BR-070`–`BR-071` | 通知、审计、隐私和一致性 |
| [`75-email.zh-CN.md`](75-email.zh-CN.md) | `BR-075` | 内部账号邀请邮件与投递回执 |
| [`80-portal-billing.zh-CN.md`](80-portal-billing.zh-CN.md) | `BR-060`–`BR-063` | Guardian Portal 和平台计数边界 |
| [`CHANGELOG.md`](CHANGELOG.md) | 基线版本 | 每次确认和取代记录 |

## 模块审阅进度

| 模块组 | 状态 | 说明 |
| --- | --- | --- |
| Identity | `confirmed` | 登录账号、邀请、Session 边界已确认 |
| Email | `confirmed` | 内部账号邀请邮件、一次性确认链接、投递回执及 Admin 管理 Resend 配置和邀请模板已确认 |
| Access | `confirmed` | Founder、Admin、Advisor、Contractor 四个 Release 1 基础角色及多角色授权边界已确认；Data Reviewer 延后 |
| Cases | `confirmed` | K12 案件、选校、逐校申请、Task、人工结案及 ServiceCase 永久保留规则已确认 |
| CRM / Student | `confirmed` | Student 边界、最小字段和 gender 已确认 |
| CRM / Guardian | `confirmed` | Guardian 最小字段、gender、出生日期和计算年龄规则已确认 |
| CRM / Guardian 关系 | `confirmed` | 多对多关系、唯一主要联系人、无次要联系人排序、独立职责标志、通知同意、有效期和关系类型已确认 |
| CRM / Student 建档 | `confirmed` | Student 与唯一主要联系人原子建档规则已确认 |
| CRM / 添加关联 Guardian | `confirmed` | 支持新建或手动选择已有 Guardian；关系类型必填且默认不改变主要联系人 |
| CRM / 主要联系人交接 | `confirmed` | 从现有关联 Guardian 中原子切换并保留历史；旧联系人不自动解除关系 |
| CRM / Guardian 关系解除 | `confirmed` | 结束当前关联但不删除 Guardian 或历史；主要联系人必须先交接 |
| CRM / 客户来源 | `confirmed` | 保留独立 ReferralSource 来源目录；每个 Case 最多关联一个当前来源，同一来源可关联多个 Case；11 个受控类型已确认 |
| CRM / 重复处理 | `confirmed` | 仅姓名、Email 或电话任一非空值相同时提示；不自动关联，Release 1 不提供人工合并或撤销合并 |
| CRM / 软删除 | `confirmed` | Advisor 申请、Founder 决定；有未结案 Case 或当前 Guardian 关系时禁止删除；`deleted` 不可业务查看且 Release 1 不可恢复 |
| Shared | `confirmed` | 不重复产生副作用、写入识别组织和操作人的业务要求已确认；实现方式归技术设计 |
| Tasks 模块通用设计 | `confirmed` | 自动任务范围、分派生命周期、逾期和完成规则已确认；其余归技术设计 |
| Schools 模块通用设计 | `confirmed` | 首次 EDB 全量导入、逐校周期抓取（最快每天一次）、人工/爬虫学校免额外审批进入可用状态、不显示未验证标记已确认；v49 仅豁免首次 EDB 自动带入网址，后续人工来源变更仍须 Founder 审批；停用/退场机制及与 v52 手动补充的交界仍为 `pending_review`，详见 BR-051 |
| Documents 模块通用设计 | `confirmed` | 私有存储、版本与扫描、下载权限和软删除边界已确认 |
| Notifications 模块通用设计 | `confirmed` | 站内通知范围、触发节点、接收人、去重和隐私边界已确认 |
| Audit & Operations | `confirmed` | 关键操作审计、事务一致性、重复请求和敏感信息边界已确认 |
| External Portal | `confirmed` | 单 Case 只读边界、字段白名单、7 天授权和撤销规则已重新确认；会话与 Token 归技术设计 |
| Platform Billing | `confirmed` | Release 1 整体排除；旧计数和合同参考值规则标记为 `superseded` |
| Future | `confirmed` | 销售收费、外部渠道与 AI、非 K12 及延后角色和资料能力均不在 Release 1，不预建业务实体；多组织按 BR-001 在整个系统范围排除 |

试用授权补充：BR-015 已获项目负责人授权实施，按 Founder/L1/L2/L3 等级、分类及任务范围授权；上表旧基础角色描述仅保留历史账号兼容语义，不与新等级取并集。客户未逐项验收，生产迁移及部署未获授权。
