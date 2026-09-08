# Cases 案件工作台 API 与页面

状态：`approved`  
确认依据：项目负责人于 2026-08-25 确认案件工作台只显示授权摘要、统一 `/api/v1/**` 契约、服务端重验、页面边界状态和 Contractor 独立工作区  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[API 与页面交互设计索引](README.md)。

业务依据：[Cases 案件流程与状态机](../30-business-flows/10-cases.zh-CN.md)。  
权限依据：[内部认证与授权模型](../40-permissions-security/10-authorization-model.zh-CN.md)。  
入口依据：[Shared 与入口适配流程](../30-business-flows/70-shared-entry-adapters.zh-CN.md)。

## 1. 先看页面目标

案件工作台只需要让内部员工回答：

1. 这个 Case 当前走到哪一步？
2. 下一步是什么，谁负责，是否有异常？
3. 我当前有权执行哪些动作？

页面不自行推进状态，也不直接读取数据库。所有数据来自 Cases、Tasks、Documents、Notifications 的正式查询。

## 2. 页面结构

```text
案件工作台
  ├─ 案件摘要：学生、当前阶段、workflow status、Primary Advisor
  ├─ 下一步：当前待处理事项、负责人、due_at、异常
  ├─ Assessment：按当前授权显示 blocker/完成状态
  ├─ 候选学校名单：版本、Founder 审核、Guardian 确认
  ├─ 逐校申请：每所学校的 Target 状态、Assignee、Task、证据摘要
  ├─ 文件：当前用户可访问的文件 metadata 和扫描状态
  ├─ 通知：当前用户的站内待办入口
  └─ 历史：案件状态、审批、确认、Task 和审计摘要
```

页面不显示：

- 当前用户无权访问的模块或字段；
- Contractor 的完整 Case 内容；
- Portal 对客内容以外的 Guardian/Student 信息；
- 文件对象 key、下载 URL、Token 或扫描原始输出；
- 用 projection/cache 推测出的未确认业务状态。

## 3. 正式 API 路径

正式 API 统一使用 `/api/v1/**`，以下是本纵向切片的目标接口：

| Method | 路径 | 类型 | 用途 |
| --- | --- | --- | --- |
| `GET` | `/api/v1/cases` | query | 按当前授权列出 Case 摘要和风险摘要 |
| `GET` | `/api/v1/cases/:caseId/workspace` | query | 读取案件工作台聚合视图；服务端按模块查询并组装受控 DTO |
| `GET` | `/api/v1/cases/:caseId/assessment` | query | 读取当前用户允许的 Assessment/blocker |
| `GET` | `/api/v1/cases/:caseId/candidate-lists` | query | 读取候选名单版本和确认状态 |
| `GET` | `/api/v1/cases/:caseId/school-targets` | query | 读取逐校状态、Assignee 和受控证据摘要 |
| `POST` | `/api/v1/cases/:caseId/candidate-lists` | command | Primary Advisor 建立新名单版本 |
| `POST` | `/api/v1/cases/:caseId/candidate-lists/:versionId/review` | command | Founder 批准或驳回名单版本 |
| `POST` | `/api/v1/cases/:caseId/candidate-lists/:versionId/guardian-decision` | command | Primary Advisor 代录 Guardian 确认 |
| `POST` | `/api/v1/cases/:caseId/lifecycle` | command | 暂停、恢复、终止或 Founder 结案 |

说明：实际逐字段 DTO、分页和业务 error code 在本切片确认后才进入开发票据。

## 4. 查询响应 DTO 原则

成功统一 envelope：

```json
{
  "api_version": "v1",
  "request_id": "opaque-id",
  "data": {}
}
```

`GET /cases/:caseId/workspace` 的 `data` 只包含受控摘要：

```text
case: id, stage, workflow_status, primary_advisor, record_version
next_step: code, owner_kind, due_at, risk_code
assessment: allowed summary + blocker codes
candidate_lists: version/status/approval/guardian_decision summary
school_targets: school opaque/display reference, status, assignee summary
tasks: current user authorized task summary
documents: current user authorized metadata/scan summary
```

不直接 spread 任意模块对象；每个字段由 owning module 明确 allowlist。

## 5. 写命令通用输入

所有写命令至少包含：

```json
{
  "expected_record_version": 12,
  "reason_code": "controlled-code"
}
```

同时要求请求头：

```text
Idempotency-Key: opaque-retry-key
```

规则：

- `organization_id`、actor、role、Case scope 不从 body 读取；
- `expected_record_version` 不匹配返回 `STALE_VERSION`，页面提示用户刷新后重新确认；
- 同一 key + 同一 payload 重试返回第一次结果；不同 payload 使用同 key 返回冲突；
- Founder 审批、Guardian 代录、终止和结案写入 AuditEvent；
- 写入成功前必须完成当前权限和业务前置条件重验。

## 6. 页面操作与授权

| 页面操作 | 可见条件 | 服务端最终检查 |
| --- | --- | --- |
| 建立候选名单 | 当前 Primary Advisor、Assessment ready | Cases 当前关系、manifest blocker、学校引用和版本 |
| Founder 审批/驳回 | 当前 Founder | Founder RoleBinding、名单版本、expected version |
| 代录 Guardian 确认 | 当前 Primary Advisor | 同一 approved version、Guardian 关系、渠道和确认记录 |
| 暂停/恢复 | Primary Advisor 或 Founder | 没有已提交或后续 Target、Case status、expected version |
| 终止服务 | 有权员工 | 所有进行中 Target 变 withdrawn、取消 Task 请求 |
| 人工结案 | Founder | 全部 Target 终态、无未完成 Task、结案结果和原因 |

前端即使隐藏按钮，服务端仍必须执行同样检查；无权操作统一显示 denied，不泄露资源是否存在。

## 7. 页面状态

每个页面和区块必须分别处理：

| 状态 | 页面行为 |
| --- | --- |
| `loading` | 显示稳定骨架或加载提示；不显示猜测数据 |
| `success` | 显示当前授权范围内的 DTO |
| `empty` | 明确表示没有 Case、没有名单版本或没有 Task，不等同于 denied |
| `denied` | 显示统一无权限提示，不暴露具体原因 |
| `not_found` | 只在服务端允许调用者区分目标不存在时显示 |
| `stale_version` | 提示数据已变化，重新加载并要求用户重新确认 |
| `unavailable` | 依赖或运行时未接通，禁止假装写入成功 |
| `error` | 显示安全错误 code；不显示 stack、SQL 或 provider 内容 |

## 8. 跨模块查询边界

- `getCaseWorkspace` 可以由 server-side application coordinator 组合 Cases、Tasks、Documents、Notifications 的公开查询。
- coordinator 不写其他模块私有表；写操作仍调用 owning module command。
- Operations projection 只能提供风险/新鲜度摘要，不能作为授权或状态机依据。
- 文件列表必须按 Documents 当前资源级授权过滤；不能因用户能看 Case 就自动看全部文件。
- Contractor 使用 Tasks 单独的脱敏 workspace，不进入完整 Case workspace。

## 9. 本模块待确认内容

请确认以下 6 点：

1. 案件工作台只展示当前用户有权看到的 Case、名单、Target、Task、文件和通知摘要。
2. 正式 API 只使用 `/api/v1/**`、统一 envelope、`Idempotency-Key` 和 expected version。
3. 候选名单建立、Founder 审批、Guardian 代录、暂停/恢复、终止和人工结案都由服务端重新授权并审计。
4. 前端隐藏按钮不能代替服务端检查；无权、目标不存在、版本冲突和依赖不可用要区分安全的页面状态。
5. `getCaseWorkspace` 可以组合公开查询，但不跨模块写表、不用投影替代权威事实。
6. Contractor 使用单独脱敏 Task workspace，不进入完整 Case workspace。

本纵向切片已确认。下一步进入 Task 工作台 API/UI 设计。
