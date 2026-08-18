# 阶段 2D：候选学校目标纵向切片

| 项目 | 当前值 |
| --- | --- |
| 日期 | 2026-08-18 |
| 状态 | `static_verified_committed_push_pending` |
| 数据范围 | 仅本地确定性合成数据 |
| 代码基线 | `7df3b7f`；分支 `codex/permission-policy-architecture` |
| 云端状态 | 不操作 AWS、Vercel、Cloudflare 或生产数据库 |

## 1. 本阶段解决的问题

阶段 2C 已允许当前 Primary Advisor 把 ServiceCase 推进到
`background_collection`，但案件详情还不能读取或创建 PostgreSQL 权威的
SchoolTarget。阶段 2D 只贯通以下最小闭环：

1. Founder 读取组织内案件的 SchoolTarget；Advisor 只读取自己当前主责案件。
2. 当前 Primary Advisor 在 `background_collection` 阶段，从 3 所固定的本地
   synthetic 学校中选择一所，创建 `candidate` SchoolTarget。
3. 创建时固定学校当时的 resolved revision、hash 和显示名称；刷新后仍从
   PostgreSQL 读取同一事实。
4. Target、resolved revision、幂等结果、AuditEvent 和 Outbox 在同一事务提交。

本阶段不实现 SchoolTarget 状态推进、CaseOutcome、案件进入
`school_selection_confirmed`、provisional School、overlay 管理、crawler 数据导入、
Task、Document、真实数据或云端运行时。

## 2. 冻结 API 契约

### 2.1 查询

`GET /api/v1/cases/[caseId]/school-targets`

```json
{
  "case_id": "uuid",
  "case_stage": "background_collection",
  "intake_year": 2027,
  "admission_type": "hk_k12_standard_v1",
  "can_create": true,
  "create_blocked_reason": null,
  "items": [
    {
      "target_id": "uuid",
      "school_id": "uuid",
      "school_name": "Synthetic School",
      "state": "candidate",
      "intake_year": 2027,
      "admission_type": "hk_k12_standard_v1",
      "record_version": 1,
      "resolved_revision_id": "uuid",
      "resolution_sha256": "64-character lowercase hex",
      "created_at": "2026-08-18T00:00:00.000Z"
    }
  ],
  "school_options": [
    {
      "school_id": "uuid",
      "display_name": "Synthetic School",
      "resolution_sha256": "64-character lowercase hex"
    }
  ]
}
```

`create_blocked_reason` 只允许 `founder_read_only`、
`case_stage_not_allowed`、`no_school_options` 或 `null`。非当前主责 Advisor
不能通过该字段探测案件存在。

`items[].school_name` 必须来自目标固定的 resolved revision；
`school_options` 必须来自当前 active snapshot 和 approved overlay 的解析结果。
已成为本案目标的学校不再出现在选项中。列表按 `created_at`、`target_id`
稳定排序；本阶段最多返回 3 个固定选项，不增加搜索或分页契约。

### 2.2 创建

`POST /api/v1/cases/[caseId]/school-targets`

请求必须携带合法 `Idempotency-Key`，body 只允许：

```json
{
  "school_id": "uuid",
  "expected_resolution_sha256": "64-character lowercase hex"
}
```

`intake_year` 和 `admission_type` 由事务锁定后的 ServiceCase 提供。旧客户端若在
body 中提交这两个字段或其他未知字段，返回 `400 INVALID_REQUEST`，不得静默忽略。

成功继续使用当前项目约定的 HTTP 200，并返回：

```json
{
  "case_id": "uuid",
  "item": {}
}
```

其中 `item` 与 GET 的单项结构完全相同。

## 3. 权限和错误边界

- Founder 可读组织内案件，但本票始终只读。
- Advisor 只能读取自己当前主责案件。
- POST 只允许 Advisor 角色且为当前 Primary Advisor，并且案件阶段必须是
  `background_collection`。
- 页面上的按钮不是权限边界；POST 事务必须重新读取活动用户、Membership、Role
  binding、Primary 关系和案件阶段。
- 非法 path/body/hash 为 `400 INVALID_REQUEST` 或 `422 VALIDATION_FAILED`。
- 无会话为 `401 UNAUTHENTICATED`；不允许的角色为 `403 FORBIDDEN`。
- 不可见案件、非主责 Advisor 或不存在的学校为 `404 NOT_FOUND`。
- 非目标阶段、重复目标、幂等冲突为 `409 CONFLICT`。
- resolved hash 变化为 `409 STALE_VERSION`。
- resolved 数据不完整或 runtime 未配置为 `503 SERVICE_UNAVAILABLE`。

## 4. 模块和事务所有权

- Cases 拥有 ServiceCase、SchoolTarget、案件权限、命令编排和 API。
- Schools 拥有 current resolved view 的读取、确定性解析和
  `schools_resolved_revisions` 写入。
- Cases PostgreSQL Repository 不得包含 `INSERT INTO schools_*`；它通过 Schools
  的 server-only PostgreSQL collaborator 共享同一事务。
- 新 migration 撤销 `tianxing_app` 对 `cases_school_targets` 的直接 INSERT，只允许
  受保护数据库函数创建 candidate target。函数再次检查 actor、Primary、阶段，且从
  ServiceCase 复制 intake/admission。
- 不新建业务表；复用现有 School、Snapshot、SnapshotRecord、ResolvedRevision 和
  SchoolTarget 表。

事务顺序固定为：tenant/actor context -> 活动身份重查 -> 锁 Case/Primary ->
幂等重放检查 -> 阶段检查 -> 读取 Case 年度/类型 -> 锁并解析 School -> hash 比较 ->
重复检查 -> 追加或复用 resolved revision -> 受保护函数插入 target -> AuditEvent ->
Outbox -> 完成幂等记录 -> commit。任一步失败必须全部回滚。

## 5. 本地数据决策

扩展本地 Release 1 seed，使用固定 UUID 创建 3 所明确标记为 synthetic 的学校、
一个固定 active snapshot 和对应 snapshot records。字段包含中英文名、地区、固定
source key 和 `.invalid` 测试网址，hash 使用现有 canonical 逻辑确定性生成。

不导入或回退读取旧 crawler JSON。现有 585 条 crawler 数据的 manifest 为
`health: warn`，不能成为本票的权威数据。Seed 重跑只接受完全一致的数据；发现另一
active snapshot 或同 ID 不同内容时必须失败。

## 6. 前后端文件边界

后端负责：Service、Route Handler、PostgreSQL repositories、Schools transaction
collaborator、runtime、migration、seed、module registry、后端聚焦测试和数据库证据。

前端负责：`modules/cases/client.ts`、`SchoolTargetsPanel`、案件详情装配、API decoder、
交互状态和前端聚焦测试。前端不得修改 route、service、repository、migration、seed
或 module registry。

案件详情中的 Panel 位于 AssessmentEditor 之后。前端只服从服务端 `can_create`，
使用原生 select 呈现本票最多 3 个学校选项；不开发真实目录的搜索、分页或自定义
combobox。POST 成功后重新 GET，不在浏览器自行拼接权威列表。

## 7. 验收标准

1. Advisor 在 `background_collection` 创建目标，刷新后仍可见。
2. Founder 可读但没有创建控件；非主责 Advisor不能探测案件。
3. 服务端忽略不了浏览器伪造的年度/类型，因为这些字段被拒绝且数据库从 Case 复制。
4. stale、duplicate、双击和网络不确定重试不会产生重复副作用。
5. Target 固定名称和 hash 不随当前学校解析结果变化。
6. 数据库存在各一条 target、resolved revision、audit、outbox 和 completed
   idempotency；重放后计数不增加。
7. `tianxing_app` 直接 INSERT target 被 PostgreSQL 拒绝。
8. resolved revision、target、audit、outbox 或 idempotency 前的失败注入全部回滚。
9. 桌面和 390px 浏览器验收无横向溢出、无 console/page error，键盘可完成创建。

## 8. 开发门禁

后端和前端可以按第 6 节文件边界并行实现，但不得提交、推送、部署或操作云资源。
双方完成后先各自报告变更、测试和残余风险，由架构师执行集成审查；数据库迁移 apply
和浏览器验收仍需单独获得用户批准。

## 9. 集成审查记录

2026-08-18 已完成前后端实现与架构师静态集成审查。审查退回并修正了以下问题：

1. 后端在过滤和稳定排序后显式限制最多 3 个学校选项。
2. 只有 `cases_school_targets_identity_idx` 的 PostgreSQL 唯一冲突映射为重复目标；
   其他唯一冲突保持原始失败语义。
3. 本地 seed 同时校验 School 与 SnapshotRecord 的 `source_school_key`。
4. 前端移除面向业务用户的数据库、内部模型和合成数据实现术语。

架构师使用项目要求的 Node.js 22 执行项目级 TypeScript 检查，结果为 0 错误；契约、
事务、模块边界、迁移清单、种子及前端状态的聚焦测试共 53 项，全部通过；migration
`027` 的 SHA-256 与 manifest 一致，两个仓库的 `git diff --check` 均通过。

尚未执行：真实 PostgreSQL migration apply、Release 1 seed、数据库权限/回滚证据、
桌面与 390px 浏览器验收和推送。这些操作不属于静态验收，仍需单独批准。Phase 2D
代码已提交为 `7df3b7f`；权限与配置开发计划另提交为 `5b089b9`。
