# Documents 文件工作台 API 与页面

状态：`approved`  
确认依据：项目负责人于 2026-08-25 确认文件资源级过滤、服务端上传/收据/下载命令、扫描可用门禁、Application Assignee 学校范围和安全失败状态  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[API 与页面交互设计索引](README.md)。

业务依据：[Documents 文件流程与状态机](../30-business-flows/30-documents.zh-CN.md)。  
权限依据：[高风险操作与安全控制](../40-permissions-security/20-security-controls.zh-CN.md)。  
任务依据：[Task 工作台 API 与页面](20-task-workspace.zh-CN.md)。

## 1. 先看页面目标

文件工作台只回答：

1. 当前用户能看到哪些文件 metadata？
2. 文件版本是否已上传、扫描、可用或被拒绝？
3. 当前用户能否上传新版本、预览、下载、恢复或删除？
4. 哪个文件可以作为申请提交凭证？

页面永远不显示对象存储 key、公开 URL、Token、扫描原始输出或无授权文件。

## 2. 文件工作区范围

| 用户关系 | 页面范围 |
| --- | --- |
| Founder | 当前授权 Case 的文件 metadata、版本和治理动作 |
| Primary Advisor | 自己 Case 的文件管理和版本操作 |
| Application Assignee | 当前负责 SchoolTarget 的必要材料和提交凭证 |
| Case Collaborator | 默认无文件；只有当前 Application Assignee 才有该校范围 |
| Interview Assignee/Contractor | 无文件工作区 |
| Admin 基础角色 | 默认无客户文件 |
| Guardian、Student、Portal | 无内部文件入口 |

Application Assignee 的每次查询都必须重新验证 Case、SchoolTarget、Assignment、Document 用途和 Version 状态。

## 3. 正式 API 路径

| Method | 路径 | 类型 | 用途 |
| --- | --- | --- | --- |
| `GET` | `/api/v1/cases/:caseId/documents` | query | 当前用户授权 Case 文件 metadata 和版本摘要 |
| `GET` | `/api/v1/school-targets/:targetId/documents` | query | 当前 Application Assignee 可访问的该校文件 |
| `POST` | `/api/v1/cases/:caseId/documents` | command | 注册 Document 和第一份 Version |
| `POST` | `/api/v1/documents/:documentId/versions` | command | 创建不可变新 Version |
| `POST` | `/api/v1/documents/:documentId/versions/:versionId/upload-intent` | command | 签发短时私有上传 capability |
| `POST` | `/api/v1/documents/:documentId/versions/:versionId/object-receipt` | command | 接收并验证对象存储收据 |
| `POST` | `/api/v1/documents/:documentId/activate` | command | 激活 clean/available Version |
| `POST` | `/api/v1/documents/:documentId/versions/:versionId/download-capability` | command | 请求时重新授权并签发精确 Version capability |
| `POST` | `/api/v1/documents/:documentId/soft-delete` | command | 进入 30 天恢复窗口 |
| `POST` | `/api/v1/documents/:documentId/restore` | command | 窗口内恢复 Document |

具体对象存储 provider、bucket、queue 和扫描器不进入浏览器 DTO；由 server-only adapter 处理。

## 4. 查询 DTO

文件列表只返回：

```text
document: id, purpose_code, case_scope, school_target_scope
active_version: id, status, content_type, size, created_at
scan: status, attempt_count, safe_result_code
lifecycle: active/pending_delete/deleted, legal_hold flag
allowed_actions: 当前用户可执行的固定 action code
```

不返回：

- 对象 key、bucket、预签名 URL；
- 原始文件名（除非后续明确属于受控 metadata）；
- 扫描原始输出或 provider error；
- 文件内容、Token、Cookie 或 capability secret。

下载/预览不是列表字段中的永久 URL；用户点击后，客户端调用 download-capability 命令，服务端重新授权并返回一次性/短时结果。

## 5. 上传流程页面

```text
选择用途和学校范围
  -> 注册 Document/Version
  -> 取得短时 upload capability
  -> 浏览器上传
  -> 提交 object receipt
  -> 页面显示 quarantined/scanning
  -> Worker 完成扫描
  -> available 后才可激活/预览/下载
```

页面规则：

- 用户选择的文件类型、大小和 checksum 只是输入；服务端必须重新校验。
- 客户端不能直接把 Version 标记为 available 或 clean。
- 重复点击、网络重试和对象重复收据使用同一 Idempotency-Key，不重复创建 Version 或扫描记录。
- Application Assignee 只能选择自己 SchoolTarget 的 `submission_evidence` 或授权的申请材料用途。

## 6. 文件状态在页面上的表现

| 状态 | 页面显示 | 允许动作 |
| --- | --- | --- |
| `pending_upload` | 等待上传 | 继续上传、abandon（按授权） |
| `quarantined` | 已接收，等待安全检查 | 只读状态，不能下载 |
| `scanning` | 安全检查中 | 只读状态，不能下载 |
| `available` | 可用 | 按资源授权预览/下载/激活 |
| `rejected` | 文件不可用 | 不能预览/下载；显示安全失败提示 |
| `scan_failed` | 检查暂时失败 | 显示处理中/告警状态，不假装可用 |
| `abandoned` | 上传已放弃 | 只读历史 |
| `pending_delete` | 等待清理，可恢复 | 30 天窗口内按授权恢复 |
| `deleted` | 内容已清理 | 只读 metadata tombstone |

## 7. 申请提交凭证

Application Task 完成页面可以从当前 SchoolTarget 文件工作区选择 Document Version 引用，但服务端必须验证：

- 文件属于当前 Case；
- 用途属于当前 SchoolTarget；
- Version 是 clean、available、未 revoked；
- 当前用户仍是该校 Application Assignee；
- 若无学校参考号，至少选择一份替代凭证。

页面提交成功只表示完成资料已送入 Tasks；Cases 仍要消费完成事实并重新校验，不能由文件页面直接改变 SchoolTarget 状态。

## 8. 删除、恢复与 legal hold

| 操作 | 页面入口 | 服务端条件 |
| --- | --- | --- |
| soft-delete | Primary Advisor 文件操作 | 当前 Case 权限、无 legal hold、expected version、理由必填 |
| restore | Primary Advisor | 30 天窗口、当前授权、有效 clean Version、expected version |
| legal hold | Founder 治理入口 | Founder 权限、理由、Audit、expected version |
| 最终清理 | 受信 Operations/Founder 流程 | retention 已到期、无 hold、无引用、Founder 批准；条件不全 fail closed |

最终清理页面只展示受控状态和批准结果，不提供直接删除对象的浏览器命令。

## 9. 页面边界状态

| 状态 | 页面行为 |
| --- | --- |
| `loading` | 显示加载状态，不猜测文件可用性 |
| `empty` | 当前 scope 没有文件，不等于无权 |
| `success` | 只显示当前授权 metadata 和 allowed actions |
| `denied` | 统一无权限提示，不泄露文件存在性 |
| `not_found` | 仅在服务端允许区分时显示 |
| `scanning` | 明确文件尚不可用，不提供下载按钮 |
| `stale_version` | 重新加载并要求用户重新确认 |
| `unavailable` | 对象存储/扫描 runtime 不可用，不显示上传成功 |
| `error` | 只显示安全错误 code，不显示 provider/路径/secret |

## 10. 安全与跨模块边界

- 页面使用 Documents typed client；不直接访问对象存储、数据库或扫描器。
- 每次上传、下载、预览、导出、删除、恢复和激活都重新授权并写必需 AuditEvent。
- 文件未 clean 前不得成为 Task 证据；文件存在也不自动完成 Task。
- Operations projection 只能显示积压、扫描和 freshness，不作为文件授权来源。
- Contractor、Guardian、Student 和 Portal 不进入文件 API；服务端稳定拒绝。
- 私有 API `no-store`；不把 capability、object key 或文件内容写进 URL、日志、通知、Outbox 或错误。

## 11. 本模块待确认内容

请确认以下 6 点：

1. 文件工作台按 Case/SchoolTarget/Assignee 做资源级过滤，不能因能看 Case 就看全部文件。
2. 正式 API 统一 `/api/v1/**`，上传、收据、下载 capability 和状态变更都使用服务端命令。
3. 只有 clean、available、未撤销 Version 可以预览、下载或成为 Task 证据。
4. Application Assignee 可以为自己学校上传/引用提交凭证，但不能修改 SchoolTarget 状态。
5. Contractor、Guardian、Student、Portal 和基础 Admin 没有内部文件入口。
6. 上传、扫描、下载、删除和恢复失败时显示安全边界状态，不能假装已成功或泄露 provider 细节。

本纵向切片已确认。下一步进入 Notifications 入口 API/UI 设计。
