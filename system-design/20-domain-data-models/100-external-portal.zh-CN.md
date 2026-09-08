# External Portal 领域模型与数据设计

状态：`approved`  
确认依据：项目负责人于 2026-08-25 接受 PortalViewer、PortalGrant、PortalSession、7 天只读入口、Guardian 边界和与 Platform Billing 解耦设计  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[领域模型与数据设计索引](README.md)。

业务依据：`BR-060`、`BR-063`，并引用 `BR-029`、`BR-033`、`BR-034`、`BR-039`、`BR-050`、`BR-070`、`BR-071`。  
模块依据：[External Portal 模块契约](../10-module-contracts/100-external-portal.zh-CN.md)。  
现状依据：[External Portal 与 Platform Billing 现状分析](../../current-state-analysis/80-portal-billing.zh-CN.md)。  
代码参考：`modules/external-portal/**`、migration `011`。

## 1. 先看结论

External Portal Release 1 只保留 3 个核心对象：

| 对象 | 目标表 | 负责的事实 |
| --- | --- | --- |
| `PortalViewer` | `portal_viewers` | 一个 Case 中被明确选择的 Guardian 查看者 |
| `PortalGrant` | `portal_access_grants` | 一次固定 7 天、可撤销的只读入口 |
| `PortalSession` | `portal_sessions` | 入口兑换后的独立浏览器会话 |

不建立以下实体：

- Portal User、Guardian 账号、密码、OTP 或内部 Identity。
- PortalProjection：每次请求从 Cases 的公开查询生成白名单 DTO。
- PortalSecurityEvent：复用 AuditEvent。
- PortalIdempotencyRecord：复用 Shared IdempotencyRecord。
- Student/Applicant viewer：Release 1 只有 Guardian viewer。
- Message、ActionItem、File、Assessment 表：事实由 Cases 或 Documents 拥有，Portal 只读。

## 2. Portal 访问链路

```text
当前 Primary Advisor 选择 GuardianRelationship
  -> 创建 PortalViewer
  -> 生成一次性 256-bit bearer secret
  -> 只保存 keyed hash + fingerprint
  -> 创建固定 7 天 PortalGrant
  -> Guardian 兑换独立 PortalSession
  -> 每次请求重新检查权限并返回白名单 DTO
```

入口生命周期：

```text
PortalGrant:   active -> revoked / expired
PortalSession: active -> revoked / expired
PortalViewer:  active -> inactive
```

## 3. PortalViewer 表

表：`portal_viewers`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | Viewer opaque UUID 主键 |
| `organization_id` | 是 | 已有 | Organization 和 RLS 边界 |
| `service_case_id` | 是 | 已有 | 只绑定一个 Case |
| `guardian_relationship_id` | 是 | 已有 | 对应该 Case Student 的 active GuardianRelationship |
| `status` | 是 | 已有 | `active` 或 `inactive` |
| `record_version` | 是 | 已有 | Viewer 状态乐观锁 |
| `created_at` | 是 | 已有 | 创建时间 |
| `updated_at` | 是 | 已有 | 最近一次状态变化时间 |

目标模型移除：

| 字段 | 处理 |
| --- | --- |
| `subject_type` | 停止新写入；Release 1 不再支持 applicant viewer |
| `applicant_student_id` | 停止新写入；Student 不能作为 Portal viewer |

关键约束：

- `guardian_relationship_id` 必须属于 `service_case_id` 对应 Student。
- 同一 Guardian 关联多个 Case 时，每个 Case 单独建立 Viewer，不自动扩展访问。
- GuardianRelationship 结束、Guardian/Student 删除、Case 失效时，Viewer 下一个请求立即失效。
- Viewer 历史不物理删除；只能由 active 变为 inactive。

## 4. PortalGrant 表

表：`portal_access_grants`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | Grant opaque UUID 主键 |
| `lifecycle_id` | 是 | 已有 | 同一入口生命周期关联；重新生成时创建新 lifecycle |
| `organization_id` | 是 | 已有 | 租户边界 |
| `service_case_id` | 是 | 已有 | Grant 的唯一 Case 范围 |
| `portal_viewer_id` | 是 | 已有 | 对应 PortalViewer |
| `keyed_secret_hash` | 条件必填 | 已有 | 入口创建时保存 keyed hash；撤销时只能清空 |
| `secret_fingerprint` | 是 | 已有 | 非敏感 fingerprint，用于审计和限流 |
| `capability_set_version` | 是 | 已有 | 固定为 `portal_case_read_v1` |
| `status` | 是 | 已有 | `active`、`revoked`、`expired` |
| `issued_by_user_id` | 是 | 已有 | 签发入口的实际 User |
| `issued_at` | 是 | 已有 | 签发时间 |
| `expires_at` | 是 | 已有 | 固定为签发时间 + 7 天以内 |
| `revoked_by_user_id` | 条件必填 | 已有 | 撤销人；Primary Advisor 或 Founder |
| `revoked_at` | 条件必填 | 已有 | 撤销时间 |
| `revoke_reason_code` | 条件必填 | 已有 | 受控撤销原因 |
| `record_version` | 是 | 已有 | Grant 状态乐观锁 |
| `created_at` | 是 | 已有 | 创建时间 |
| `updated_at` | 是 | 已有 | 最近一次状态变化时间 |

关键约束：

- 只有当前 Primary Advisor 可以创建或重新生成 Grant。
- Founder 只能紧急撤销，不能创建或延期入口。
- `expires_at` 创建后不可修改；原入口不能延期。
- 同一 Case + Viewer 最多一个 active Grant；重新生成时旧 Grant 和全部 Session 原子撤销。
- Grant 撤销或到期时，全部 active Session 同时失效。
- 明文 bearer secret 只返回一次，不进入数据库、日志、Audit、URL query 或 localStorage。

## 5. PortalSession 表

表：`portal_sessions`

| 字段 | 必填 | 现状 | 用途 |
| --- | --- | --- | --- |
| `id` | 是 | 已有 | Session opaque UUID 主键 |
| `organization_id` | 是 | 已有 | 租户边界 |
| `service_case_id` | 是 | 已有 | 与 Grant 绑定同一 Case |
| `grant_id` | 是 | 已有 | 对应 PortalGrant |
| `session_slot` | 是 | 已有 | 1 至 3；每个 Grant 最多 3 个 active Session |
| `keyed_session_hash` | 条件必填 | 已有 | Session Cookie 对应的 keyed hash；失效时清空 |
| `status` | 是 | 已有 | `active`、`revoked`、`expired` |
| `grant_expires_at` | 是 | 已有 | 创建时复制 Grant 过期时间，防止 Session 超出入口期限 |
| `last_seen_at` | 是 | 已有 | 最近一次成功请求时间 |
| `idle_expires_at` | 是 | 已有 | 最长 15 分钟 idle timeout |
| `absolute_expires_at` | 是 | 已有 | 最长 8 小时 absolute timeout |
| `revoked_at` | 条件必填 | 已有 | Session 撤销时间 |
| `revoke_reason_code` | 条件必填 | 已有 | logout、grant_revoked、grant_expired 等原因 |
| `record_version` | 是 | 已有 | Session 状态乐观锁 |
| `created_at` | 是 | 已有 | 创建时间 |
| `updated_at` | 是 | 已有 | 最近一次请求或状态变化时间 |

关键约束：

- Session 的 idle 和 absolute deadline 都不能晚于 Grant `expires_at`。
- Session Cookie 使用 `__Host-`、`HttpOnly`、`Secure`、`SameSite=Strict`、`Path=/`。
- 不复用内部 ERP Session/Cookie，也不能兑换成内部 User。
- Session 失效后，下一个请求统一返回 `PORTAL_ACCESS_INVALID`，不区分是 Viewer、Grant 还是 Session 失效。

## 6. Portal 对客内容边界

Portal 不保存对客内容，只读取 Cases 的公开查询结果：

| 内容 | 处理 |
| --- | --- |
| 当前对客阶段和更新时间 | Cases 提供白名单字段 |
| Founder 已批准且 Guardian 已确认的学校进度 | Cases 提供固定 customer-facing mapping |
| Advisor 标记为 customer-visible 的消息 | Cases 提供只读结果 |
| Advisor 标记为 customer-visible 的行动项 | Cases 提供只读结果 |

明确禁止返回：

- Case number。
- Student/Guardian 姓名和联系方式。
- Assessment、内部备注、内部 Task/Assignee。
- 文件、文件名、下载/预览入口。
- 候选草稿、驳回理由、AuditEvent 和员工信息。

Portal 不能确认选校、决定 offer、回复消息、完成行动项、上传文件或修改任何业务数据。

## 7. 权限边界

| 操作者 | 权限 |
| --- | --- |
| 当前 Primary Advisor | 创建、重新生成、查看摘要和撤销自己 Case 的 Grant |
| Founder | 查看组织授权摘要、紧急撤销；不能创建或延期 |
| 其他 Advisor/Collaborator | 无 PortalGrant 操作权限 |
| Admin 基础角色 | 无 PortalGrant 操作权限 |
| Contractor | 无 |
| Guardian | 只有已兑换的当前 PortalSession 只读访问 |
| Student、Portal 外部访问者 | 无 |

每次 Portal workspace 请求重新检查：Session、Grant、Viewer、GuardianRelationship、Case、Organization，以及签发人是否仍为当前 Primary Advisor。

## 8. 当前实现与目标差异

| 优先级 | 当前实现 | 目标处理 |
| --- | --- | --- |
| `P0` | Founder 和 Primary Advisor 都可创建 Grant | 只有当前 Primary Advisor 可创建；Founder 只紧急撤销 |
| `P0` | expires_at 可由客户端选择 | 服务端固定签发后 7 天，客户端不传期限 |
| `P0` | 支持 applicant Student viewer | Release 1 只保留 Guardian viewer |
| `P0` | Portal DTO/代码仍含 Case number | 从对客白名单和公开 DTO 中删除 |
| `P0` | policy 接受 Data Reviewer、Subscription/past_due | 移除旧角色和 Billing 依赖；past_due 不参与访问判断 |
| `P0` | runtime 固定 unavailable | 接通正式 repository/service；未配置时继续 fail closed |
| `P1` | `rotate` 容易被理解为延期 | 改为 reissue：撤销旧 Grant，创建全新 secret 和 7 天期限 |
| `P1` | PortalSecurityEvent、PortalIdempotencyRecord 重复平台能力 | 停止新写入，收敛到 AuditEvent 和 Shared IdempotencyRecord |
| `P1` | 页面/API 骨架已有但 workspace runtime 不可用 | 接通严格白名单 DTO，并覆盖撤销、过期、改派即时失效 |

Platform Billing 不属于 Release 1；历史 migration 不修改，只从入口、runtime 和 module registry 隔离。

本次只冻结目标设计，不修改产品代码、migration 或数据库。

## 9. 设计决策

| ID | 决策 |
| --- | --- |
| `SD-PORTAL-001` | 核心对象只保留 PortalViewer、PortalGrant、PortalSession |
| `SD-PORTAL-002` | PortalViewer 只代表一个 Case 中被明确选择的 Guardian |
| `SD-PORTAL-003` | Grant 只有当前 Primary Advisor 可以创建，固定 7 天且不可延期 |
| `SD-PORTAL-004` | bearer secret 只显示一次，兑换独立 PortalSession |
| `SD-PORTAL-005` | 每 Grant 最多 3 个 Session，15 分钟 idle、8 小时 absolute |
| `SD-PORTAL-006` | Portal 每次请求实时重验，使用正向字段 allowlist |
| `SD-PORTAL-007` | Portal 只读、单 Case，不支持文件、确认、回复或任何业务写入 |
| `SD-PORTAL-008` | External Portal 与 Platform Billing 完全解耦 |

## 10. 本模块确认结果

项目负责人已一次确认下面 8 点：

1. 接受 Portal 只有 Viewer、Grant、Session 三个核心对象；Guardian 不是内部 User。
2. 接受只有当前 Primary Advisor 能创建 7 天入口，Founder 只能撤销。
3. 接受重新分享必须撤销旧入口并创建全新 secret，不能延期旧入口。
4. 接受每个 Grant 最多 3 个 Session，15 分钟 idle、8 小时 absolute。
5. 接受 Portal 只展示对客阶段、学校进度、可见消息和行动项，不展示 Case number。
6. 接受 Portal 不能确认、回复、修改、上传、查看文件或访问其他 Case。
7. 接受任何 Viewer、Grant、Session、关系、Case 或签发人失效都会立即拒绝访问。
8. 接受 External Portal 与 Platform Billing 完全解耦，Platform Billing 不进入 Release 1。

External Portal 已标记为 `approved`，下一步进入最后一个模块：Shared 与入口适配层。
