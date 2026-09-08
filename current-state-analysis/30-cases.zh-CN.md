# Cases 现状分析

返回[现状分析总览](README.md)。业务依据：[Cases](../business-requirements/30-cases.zh-CN.md)。

## 结论

| BR | 状态 | 当前实现摘要 |
| --- | --- | --- |
| `BR-030` | `部分符合` | 五阶段、建案和 signed 后自动进入 background_collection 的数据库基础已存在；后续里程碑未闭环 |
| `BR-031` | `部分符合` | 暂停/恢复、原因、权限和已提交后禁止暂停已实现；目标环境与提醒联动未完成 |
| `BR-032` | `部分符合` | 15 字段 manifest、blocker、版本化答案和读写服务基本存在；完整授权/浏览器闭环未重新验收 |
| `BR-033` | `缺失` | 候选 SchoolTarget 可创建，但 Founder 审批、Guardian 同版本确认和名单版本模型不存在 |
| `BR-034` | `冲突` | 缺 `offer_confirmed/offer_declined`；旧实现把 waitlisted/accepted 当终态并要求 Outcome |
| `BR-039` | `部分符合` | 数据库禁止删除 Case；人工终止/结案命令和结案条件尚未实现 |

## 已有基础

- `ServiceCaseStage` 已收敛为 `signed -> background_collection -> school_selection_confirmed -> application_in_progress -> closed`。
- migration 036 要求 Case 不能以 `signed` 状态提交事务，并提供原子进入 `background_collection` 的数据库函数。
- 15 字段 K12 v1 manifest 已进入 migration 036，并有 Assessment repository、API、页面和测试资产。
- 暂停/恢复会记录 lifecycle fact、原因、操作者、版本、审计和 outbox；存在 `submitted` 或更后状态时数据库拒绝暂停。
- SchoolTarget 已有 candidate 创建、状态转换、证据校验、Outcome 与审计基础。
- migration 003 为 ServiceCase、Assessment、SchoolTarget 和 Outcome 设置拒绝 DELETE 的触发器。

## 关键差异

| 优先级 | 当前代码 | 与业务基线的差异 | 建议动作 |
| --- | --- | --- | --- |
| `P0` | `SchoolTargetService.getSchoolTargets()` 固定返回 `canCreate: false` 和 `selection_workflow_required` | 代码承认选校确认流程未实现 | 新增候选名单聚合与不可变版本，先完成 Founder 决策，再完成 Guardian 代录确认 |
| `P0` | 无 Founder 同版本批准、Guardian 同版本确认、学校集合快照 | 无法证明申请学校经过两层确认 | 记录 list version、school set、批准与确认 receipt；修改即产生新版本并重新确认 |
| `P0` | SchoolTarget 类型和 DB 只有 candidate/preparing/submitted/interview/waitlisted/accepted/rejected/withdrawn | 缺 offer 接受/拒绝两个终态 | 追加 `offer_confirmed/offer_declined`，更新 transition、UI、Outcome 和测试 |
| `P0` | `TARGET_TERMINAL_STATES` 包含 waitlisted、accepted | 会过早允许完成或结案判断 | 终态只保留 offer_confirmed、offer_declined、rejected、withdrawn |
| `P0` | preparing -> submitted 强制 official reference | 学校无参考号时应允许其他凭证 | 改为“参考号或明确无参考号 + 至少一份其他凭证”的组合校验 |
| `P0` | `evaluateCaseTransitionPolicy` 对 terminate/close 固定拒绝；UI 测试也确保没有 close 控件 | 无法执行整体终止和 Founder 人工结案 | 实现原子终止/结案命令，重验所有目标终态、未完成 Task、操作者及原因 |
| `P0` | 无“全部终态后新增学校或结案”的业务分支 | 全拒后流程无落点 | 保持 Case 开放，允许新名单版本；只有 Founder 明确操作才 closed |
| `P1` | `getCaseRuntime()` 在所有环境固定不可用，workspace 仅非 production-aws 接 PostgreSQL | 建案与完整写路径尚未形成目标环境组合 | 统一 `/api/v1` runtime composition，再做本地 HTTP 与浏览器验收 |

## 已确认主流程的目标形态

```text
建案并指定 Primary Advisor
  -> 自动进入背景收集
  -> Assessment background_complete
  -> 候选名单版本
  -> Founder 批准
  -> Guardian 确认同一版本
  -> 每校进入 preparing 并创建申请 Task
  -> submitted / interview / offer 决策
  -> Founder 人工结案，或建立新候选名单版本
```

## 完成标准

- 每次进入申请处理都能追溯到同一名单版本的 Founder 批准和 Guardian 确认。
- SchoolTarget 状态和终态判断与 BR-034 完全一致。
- 申请准备/提交、面试与 Task 在同一业务事务或可靠 outbox 流程中形成闭环。
- 所有学校结束也不会自动结案；只有 Founder 在满足条件后人工结案。
- 重新签约创建新 Case，旧 Case 及全部历史永久保留。
