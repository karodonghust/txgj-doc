# Documents 模块契约

状态：`approved`  
确认依据：项目负责人于 2026-08-25 指示继续进入下一模块  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v31`

返回[模块契约索引](README.md)。

业务依据：`BR-050`，并引用 `BR-012`、`BR-013`、`BR-035`、`BR-036`、`BR-039`。  
现状依据：[Documents 现状分析](../../current-state-analysis/60-documents.zh-CN.md)。

## 1. 一句话职责

Documents 只回答：

> 文件属于哪个 Case、当前可用版本是什么、扫描是否通过，以及当前访问者能否获得一次短时上传或下载能力？

Documents 不决定 Case、SchoolTarget 或 Task 是否可以推进。

## 2. 负责与不负责

| Documents 负责 | Documents 不负责 |
| --- | --- |
| 文件 metadata、业务用途和关联范围 | Student、Case、SchoolTarget 或 Task 主状态 |
| 不可变文件版本和当前可用版本 | 申请材料清单是否齐全 |
| 上传收据、隔离、扫描结果和有界重试 | Application Assignee 或 TaskAssignment 关系本身 |
| 文件软删除、恢复、legal hold 和清理门禁 | CRM/Case 业务记录的删除规则 |
| 每次请求的文件级授权和短时 capability | 公开 URL、永久分享链接或 Portal 文件区 |

## 3. 核心对象

Release 1 只保留三个核心对象：

| 对象 | 含义 | 关键约束 |
| --- | --- | --- |
| `Document` | 一个 Case 文件及其用途、关联范围和生命周期 | 必须归属一个 Case；可以关联多个 SchoolTarget 用途；metadata 保留历史 |
| `DocumentVersion` | 一次实际上传的不可变内容版本 | 对象引用、checksum、大小和内容类型上传后不可改写 |
| `ScanResult` | 一个版本的一次受控扫描工作及最终结果 | 记录 policy version、attempt、结果和时间；不可删除或改写历史 |

以下概念不单独建业务实体：

- Case/SchoolTarget/Task 关联：作为 Document 聚合内的受控关系表达；数据库可使用关联表，但不形成新的业务聚合。
- upload/download intent：短时 capability 和审计事实，不是长期授权实体。
- active version：Document 指向一个通过扫描且未撤销的 DocumentVersion。
- legal hold：Document 的治理状态和记录，不建立独立工作流实体。
- Application Assignee 文件工作区：按当前 Case/SchoolTarget 关系计算的查询结果。

## 4. 文件业务范围

- Release 1 的业务文件统一以 Case 为授权根，不建立脱离 Case 的客户文件库。
- Document 可以标记为 Case 通用，或关联一至多个 SchoolTarget 用途。
- Task 只保存受控 Document 引用；不复制文件内容、对象键或下载 URL。
- 一个已扫描文件可以被多个学校申请复用，不要求复制文件或创建重复 Document。
- SchoolTarget 用途变化保留历史；移除用途不会改写既有 Task completion 或审计记录。

建议使用固定用途分类：

| 分类 | 典型内容 |
| --- | --- |
| `case_evidence` | 已签合同、身份或案件证明 |
| `application_material` | 学校申请需要的材料 |
| `submission_evidence` | 确认页、确认邮件 PDF、回执或邮寄凭证 |
| `operational_attachment` | 其他内部业务附件 |

分类只帮助授权、保留和检索，不能代替具体 Case/SchoolTarget 关系检查。

## 5. Document 生命周期

```text
active -> pending_delete -> active       （30 天窗口内恢复）
active -> pending_delete -> deleted      （窗口结束且全部清理条件满足）
```

规则：

- 删除只进入 `pending_delete`，不立即删除对象内容或数据库记录。
- 默认恢复窗口为 soft-delete 后 30 天。
- 30 天到期只表示不能普通恢复，不等于自动授权清理。
- `deleted` 表示对象内容已按治理规则清理；Document、Version 和 ScanResult metadata 保留为不可变 tombstone。
- legal hold 期间不能新发起 soft-delete 或进入最终清理；已经 pending_delete 的文件被冻结，直到 hold 解除或恢复。
- 文件删除、恢复或清理不改变 Student、Case、Task 或 SchoolTarget。

## 6. DocumentVersion 与扫描生命周期

```text
pending_upload -> quarantined -> scanning -> available
pending_upload -> abandoned
scanning -> rejected
scanning -> scan_failed -> scanning       （有界重试）
```

规则：

- 新版本永远创建新对象，不覆盖旧版本。
- 上传完成后先进入隔离区；未扫描文件不能预览、下载、导出或成为 active version。
- 只有 clean、available 且未 revoked 的版本可以激活。
- malicious 结果进入 rejected，不向普通用户返回文件内容。
- scan_failed 按固定技术策略有界重试；达到上限后保留失败状态并进入 Operations 告警。
- 新的 clean 版本激活后，旧的 clean 版本仍保持 available，但不再是 active version。
- 可回滚到仍为 clean、available、未 revoked 的旧版本；回滚只移动 active pointer，不改写版本历史。

## 7. 上传闭环

```text
注册 Document
  -> 创建 pending_upload Version
  -> 服务端签发短时私有上传 capability
  -> 对象存储确认收据
  -> quarantined
  -> Worker 扫描
  -> clean 后 available 并受控激活
```

- 上传前重验 actor、organization、Case、Document 范围和 expected version。
- capability 绑定固定对象键、版本、content type、checksum、size 和短有效期。
- 对象键只使用 opaque ID，不包含姓名、学校名、Case number 或原始文件名。
- 客户端声称“上传成功”不能推进状态；必须由服务端读取对象存储收据。
- 重复对象通知和 Worker 投递必须幂等，不能重复创建 Version、ScanResult 或激活事件。
- 未完成的 pending upload 可以显式 abandoned；不得当成可用文件。

## 8. 下载、预览与导出

- 文件没有公开 URL，也不把私有对象键返回给普通业务查询。
- 预览与下载使用同一文件级授权；差别只在响应方式，不降低权限。
- 每次签发短时 download capability 前重新检查 actor、Case、SchoolTarget 用途、active version、scan 和 revoke 状态。
- capability 只能指向授权时确认的精确版本；过期后必须重新申请。
- 批量导出不得扩大单文件权限；每个文件都必须重新检查，失败项不能被打包绕过。
- 访问 capability、Cookie、对象键和文件内容不得进入日志、通知或普通 Audit 摘要。

## 9. 角色和关系授权

| 访问者 | Release 1 文件权限 |
| --- | --- |
| Founder | 查看并下载组织内授权 Case 文件；不因 Founder 身份自动修改文件 |
| 当前 Primary Advisor | 查看、注册、上传新版本、关联用途、下载、软删除和恢复自己 Case 的文件 |
| 当前 Application Assignee | 只查看/下载自己负责 SchoolTarget 所需文件；可上传该校提交凭证 |
| 其他 Case Collaborator | 默认不可访问；仅成为当前 Application Assignee 时获得该校范围 |
| Admin 基础角色 | 默认不可查看客户文件 |
| Contractor/Interview Assignee | 不可查看、预览、下载或导出 Case 文件 |
| Guardian、Student、Portal | 不可查看、上传或下载内部文件 |

Application Assignee 的访问必须同时满足：

1. Membership、Advisor RoleBinding 和 User 当前 active。
2. Case 当前有效且与 actor 的案件授权一致。
3. actor 是请求 SchoolTarget 的当前 Application Assignee。
4. Document 明确关联该 SchoolTarget，或被 Primary Advisor 标记为该校可复用材料。
5. 文件和 active version 通过生命周期与扫描检查。

职责、名单版本、SchoolTarget 或案件授权失效后，下一个请求立即拒绝。

## 10. 删除、恢复与最终清理

- 当前 Primary Advisor 可以对自己 Case 的 active Document 发起 soft-delete；理由和 expected version 必填。
- 30 天窗口内，当前 Primary Advisor 可以恢复到仍有效的 clean version。
- legal hold 的设置和解除只允许 Founder，并记录理由、操作者、时间和版本。
- 最终清理必须同时满足：恢复窗口已过、适用 retention 已到期、无 legal hold、无有效业务引用、Founder 明确批准。
- 业务基线尚未确定各分类的长期 retention 年限；未配置有效 retention 时必须 fail closed，不得清理。
- 最终清理只删除对象内容；数据库 metadata、版本、扫描、批准和审计记录不得物理删除。

## 11. 对外查询契约

| 查询 | 主要调用方 | 返回 |
| --- | --- | --- |
| `listCaseDocuments` / `getDocument` | 内部文件页面 | 授权范围内 metadata、用途、生命周期和版本摘要 |
| `getApplicationDocumentWorkspace` | Application Assignee 工作区 | 当前 SchoolTarget 的最小文件列表和允许动作 |
| `validateEvidenceReferences` | Tasks completion coordinator | 引用是否属于当前 Case/SchoolTarget 且为 clean active version |
| `getDocumentLifecycleFacts` | Operations | opaque ID、状态、重试、retention 和 legal hold 摘要 |
| `getDocumentAuditFacts` | Audit | 操作类型、opaque ID、版本和受控 reason code |

普通查询不返回对象 bucket/key、预签名 URL、checksum、扫描内部错误或文件内容。

## 12. 对外命令契约

| 命令 | 关键规则 |
| --- | --- |
| `registerCaseDocument` | 当前 Primary Advisor；Application Assignee 仅限自己的 SchoolTarget 提交凭证 |
| `createDocumentVersion` | 授权范围内新建不可变 Version；业务幂等 identity |
| `issueUploadCapability` | 精确 Version、类型、checksum、大小和短 TTL |
| `acceptObjectReceipt` | 只信任对象存储收据；幂等进入 quarantined |
| `recordScanResult` | 只允许受信 Worker；匹配 scan policy/version/attempt |
| `activateCleanVersion` | clean、available、未 revoked；expected version |
| `issueDownloadCapability` | 请求时重新授权；只签发精确 active version |
| `rollbackActiveVersion` | 当前 Primary Advisor；目标旧版本仍 clean/available |
| `softDeleteDocument` | 当前无 legal hold；理由、引用和 expected version 门禁 |
| `restoreDocument` | 30 天窗口内；目标 clean version 和 expected version 门禁 |
| `setLegalHold` / `releaseLegalHold` | Founder；理由必填；保留完整历史 |
| `purgeDocumentContent` | 窗口、retention、hold、引用和 Founder 批准全部满足 |

所有业务写命令都必须幂等，并与 metadata、AuditEvent、Outbox 和 Idempotency result 原子提交。对象存储和扫描副作用通过 outbox/worker 协调，不使用跨系统数据库事务。

## 13. 发布的事实

| 事实 | 主要消费者 |
| --- | --- |
| `documents.document_registered` / `version_created` | Audit、Operations |
| `documents.object_quarantined` | 扫描 Worker、Audit、Operations |
| `documents.version_available` / `version_rejected` | Primary Advisor、Tasks、Audit、Operations |
| `documents.scan_retry_exhausted` | Primary Advisor、Founder、Operations |
| `documents.active_version_changed` | Tasks、Audit |
| `documents.soft_deleted` / `restored` | Tasks、Audit、Operations |
| `documents.legal_hold_changed` / `content_purged` | Founder、Audit、Operations |

事件只携带 opaque ID、状态、版本、受控 reason code 和必要关联 ID；不携带文件名、姓名、Case number、对象键、URL、文件内容或扫描原始输出。

## 14. 依赖规则

| 类型 | 允许 |
| --- | --- |
| 业务依赖 | `Access`、`Cases` 的公开授权与 Case/SchoolTarget 关系契约 |
| 平台依赖 | `Shared`、`Audit` 的公开契约 |
| 外部端口 | 私有对象存储、队列和病毒扫描器的显式 adapter |
| 允许消费者 | Tasks、内部 Documents 入口、Operations Worker |

明确禁止：

- Documents 写 Access、Cases、Tasks 或 Notifications 私有表。
- Tasks、页面或 Worker 直接写 Document/Version/ScanResult 表。
- 文件 available 或 Task 引用有效时直接推进 SchoolTarget 或 Case。
- Portal、Contractor 或 Admin 基础角色通过通用 capability 绕过资源级授权。
- local/mock adapter 静默进入 production-aws；运行时未接通必须 fail closed。

## 15. 安全与一致性不变量

- 每次读写重验 organization、actor、Case、SchoolTarget 用途、Document、active Version 和 expected version。
- 任何文件内容在扫描 clean 前都不能离开隔离处理链路。
- DocumentVersion 内容、对象 identity、checksum 和扫描历史不可改写。
- 私有对象存储默认拒绝公开访问，生产对象固定在香港批准区域。
- 数据库 RLS、最小权限和应用授权必须同时存在；任一层不能替代其他层。
- 旧 Primary Advisor/Assignee binding 被撤销后，即使同一用户另有 active role，也不能继承旧文件权限。
- 日志、错误、通知和 outbox 不得泄露 PII、文件内容、私有 URL、对象键、token 或扫描原始输出。
- 本地上传/扫描闭环通过不代表 Vercel 或 AWS 生产已接通。

## 16. 与当前代码的差异

| 优先级 | 当前实现 | 目标契约 |
| --- | --- | --- |
| `P0` | 下载只允许 Founder 或当前 Primary Advisor | 增加按精确 SchoolTarget 限定的 Application Assignee 下载 |
| `P0` | Document owner 仅在 Student/Case/Task 三选一 | Release 1 以 Case 为授权根，并支持多个 SchoolTarget 用途关联 |
| `P0` | 通用 `documents.*` capability 先按单一基础角色授权 | 改为 Access 多角色 context + Case/SchoolTarget/Document 资源级重验 |
| `P0` | Application Assignee 无学校范围上传提交凭证路径 | 只允许为自己 SchoolTarget 上传 `submission_evidence` |
| `P1` | 分类与申请材料/提交凭证用途不完全匹配 | 收敛固定分类并迁移既有 metadata |
| `P1` | 旧 clean 版本可转为 `superseded`，与回滚条件不一致 | clean 旧版本保持 available；active pointer 单独表示当前版本 |
| `P1` | purge policy 含旧 retention 规则，但业务基线未确认具体年限 | retention 未确认时 fail closed，不把旧年限当业务事实 |
| `P1` | Portal/面试辅助默认无入口，但需稳定拒绝证明 | 服务端明确永久拒绝并覆盖测试 |
| `P1` | production runtime 未完整接通；部分页面仍是 preview | 接通显式 adapter；未接通前继续 fail closed |

这些差异进入后续开发拆分；本环节不修改产品代码或数据库。

## 17. 设计决策

| ID | 决策 |
| --- | --- |
| `SD-DOC-001` | Release 1 核心对象只保留 Document、DocumentVersion、ScanResult |
| `SD-DOC-002` | 所有客户业务文件以 Case 为授权根，可关联多个 SchoolTarget 用途 |
| `SD-DOC-003` | 版本不可变，只有 clean/available/未 revoked 版本可以激活 |
| `SD-DOC-004` | 没有公开 URL；每次操作签发短时、精确、重新授权的 capability |
| `SD-DOC-005` | Application Assignee 只访问自己 SchoolTarget 的必要文件和提交凭证 |
| `SD-DOC-006` | 30 天是恢复窗口，不是自动清理日期；retention 不明时禁止清理 |
| `SD-DOC-007` | 最终清理只清对象内容，数据库保留不可变 tombstone 和审计历史 |

## 18. 本模块验收标准

项目负责人需要确认：

1. Release 1 只保留 Document、DocumentVersion、ScanResult 三个核心对象。
2. 文件以 Case 为授权根，可被多个学校申请复用，不复制文件内容。
3. 只有扫描通过且未撤销的版本可以激活、预览、下载或作为 Task 证据。
4. Application Assignee 只访问自己负责学校的必要材料，并可上传该校提交凭证。
5. Contractor、Admin 基础角色、Guardian、Student 和 Portal 均不能访问内部文件。
6. 删除先进入 30 天恢复窗口；legal hold 或 retention 不明确时禁止最终清理。
7. 最终清理只清对象内容，数据库 metadata、版本、扫描和审计历史永久保留。

确认后进入下一个模块：Notifications。
