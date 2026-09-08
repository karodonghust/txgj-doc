# CRM 现状分析

返回[现状分析总览](README.md)。业务依据：[CRM](../business-requirements/20-crm.zh-CN.md)。

## 结论

| BR | 状态 | 当前实现摘要 |
| --- | --- | --- |
| `BR-020` | `冲突` | Student 主档存在，但无 gender，状态仍是 `purged`，重复检测含 DOB，且保留 merge |
| `BR-021` | `冲突` | Guardian 主档存在，但无 gender/DOB，状态和 merge 仍按旧规则 |
| `BR-022` | `冲突` | 关系版本和联系人标志存在，但类型只有 3 个，缺 `relationship_description` |
| `BR-023` | `部分符合` | 新 Student + 新 Guardian + 主要关系可原子建档；首次建档不能选择已有 Guardian |
| `BR-024` | `部分符合` | 可关联已有 Guardian；不能在同一操作中新建 Guardian 后关联 |
| `BR-025` | `部分符合` | 主要联系人原子交接和历史保留已实现，但受旧关系类型限制 |
| `BR-026` | `缺失` | 未找到结束当前 Guardian 关系的服务或 API |
| `BR-027` | `部分符合` | 独立 ReferralSource 和 Case 关联历史已存在，但目录字段和来源类型仍按旧规则 |
| `BR-028` | `冲突` | 当前来源枚举为 bank/insurance/other_partner 等旧值 |
| `BR-029` | `冲突` | 只有提交 pending delete 的路径；旧设计会 purge，缺 Founder 批准/驳回到 `deleted/active` 的闭环 |

## 已有基础

- `crm_students`、`crm_guardians`、`crm_student_guardian_relationships` 已使用 UUID、组织边界、版本号和审计/幂等基础。
- 数据库已有“每个 active Student 恰好一名当前主要联系人”的延迟约束。
- `StudentCreateService` 和 PostgreSQL repository 可原子创建 Student、新 Guardian 与主要关系。
- `GuardianRelationshipService` 已支持关联已有 Guardian 和主要联系人交接。
- 正常读取服务只返回 `active`、`pending_delete`，不会返回旧 `purged` 记录。
- CRM 多个 local/database-test 运行时已接 PostgreSQL；production-aws 仍主动不可用。

## 关键差异

| 优先级 | 当前代码证据 | 差异与风险 | 建议动作 |
| --- | --- | --- | --- |
| `P0` | `db/migrations/202608021630_002_expand_crm.sql` 使用 `active/pending_delete/purged` 并清除 PII | 与永久保留的 `deleted` 软删除规则相反 | 追加纠正迁移：引入 `deleted`，禁止新 purge；保留旧表事实供迁移审计 |
| `P0` | `DeletionReviewService` 只有 request/list；没有 approve/reject | Founder 无法完成删除审批 | 新增批准/驳回命令，事务内重验 Case/关系条件并写审计 |
| `P0` | `duplicate-review-service.ts` 和 migration 030 支持 merge/correction，DOB 也是匹配信号 | Release 1 只警告，不合并；DOB 不触发警告 | 停用 merge/correction API 与 capability；匹配信号只保留非空姓名/email/电话 |
| `P0` | Student、Guardian 表和 DTO 缺 gender；Guardian 缺 DOB | 资料结构不满足已确认最小字段 | 追加字段、枚举检查、API 和页面；年龄只计算不保存 |
| `P0` | 关系类型只有 `father/mother/other_guardian` | 与 27 个受控选项冲突 | 统一 domain、API、DB check 和 UI；增加 `relationship_description` 及 other 必填规则 |
| `P0` | 无 relationship termination | 非主要 Guardian 无法按业务解除关系 | 新增结束关系命令；主要联系人必须先交接；只写 `ends_at` 和新版本 |
| `P0` | `referral-source-service.ts`、独立表和 Case assignment 表已存在，但类型仍为 bank/insurance/other_partner | 最新规则保留来源目录和 Case 关联，但必须采用 BR-028 的 11 个类型 | 保留实体与关联历史；纠正来源字段、枚举、other 说明和 active-only 新关联规则 |
| `P1` | 首次建档只接受新 Guardian；后续 attach 只接受已有 Guardian | 两条路径都只完成了一半 | 将“新建或选择已有”作为显式 union command，保持单事务与禁止自动关联 |

## 数据纠正注意事项

- 历史 migration 不重写，只增加纠正 migration。
- 旧 `purged` 数据是否已经存在必须在迁移前做只读计数；不能假设为空。
- 停用 merge 不等于删除历史表；先阻断新写入和 UI/API，再决定历史数据保留方式。
- `pending_delete` 仍应出现在普通列表并带明确标记；`deleted` 在 Release 1 的业务接口中完全不可见。

## 完成标准

- Student/Guardian 字段、状态、重复警告和关系类型与 BR 完全一致。
- 首次建档与后续关联都支持“新 Guardian 或手动选择已有 Guardian”，且不会自动关联。
- Guardian 关系可结束，主要联系人必须先完成交接。
- ReferralSource 目录归 CRM，Case 当前来源和更换历史归 Cases；两侧都使用 BR-028 的受控类型。
- 软删除只有 `active -> pending_delete -> active/deleted`，业务和数据库都无最终 purge。
