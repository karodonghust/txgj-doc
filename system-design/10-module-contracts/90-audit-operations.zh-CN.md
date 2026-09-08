# Audit 与 Operations 模块契约

状态：`approved`  
确认依据：项目负责人于 2026-08-25 指示继续进入下一模块  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v31`

返回[模块契约索引](README.md)。

业务依据：`BR-070`、`BR-071`，并引用所有产生状态转换、授权、批准、确认、导出、删除、恢复和高风险读取的 `BR-*`。  
现状依据：[Notifications、Audit 与隐私现状分析](../../current-state-analysis/70-notifications-audit.zh-CN.md)。

## 1. 一句话职责

- **Audit** 保存“谁在什么时候对什么做了什么，结果如何”的不可改写证据。
- **Operations** 根据权威事件生成可重建看板和告警，帮助发现风险，但不能决定业务结果。

二者边界：Audit 是证据；Operations 是投影。Operations 丢失可以重建，Audit 丢失不能接受。

## 2. 负责与不负责

| 模块 | 负责 | 不负责 |
| --- | --- | --- |
| Audit | AuditEvent、OutboxMessage、追加式证据和安全 payload | 保存业务正文、代替业务表或判断授权 |
| Operations | Case 风险投影、technical health、OperationalAlert、projection health | 修改 Case/Task/Document，作为授权或结案依据 |

共同禁止：

- 不复制 Student/Guardian 资料、Assessment answer、文件内容或自由文字业务正文。
- 不因看板、缓存、日志或统计显示某状态，就绕过权威模块重新校验。
- 不把平台审计与租户业务审计混在同一读取边界。

## 3. 最小核心对象

| 模块 | 核心对象 | 含义 |
| --- | --- | --- |
| Audit | `AuditEvent` | 一个追加式、不可修改或删除的审计事实 |
| Audit | `OutboxMessage` | 与业务提交绑定的异步 effect 信封及投递状态 |
| Operations | `OperationsProjection` | 可删除、可重建、带 source version/checkpoint 的只读投影 |
| Operations | `OperationalAlert` | 一个已触发的运行或隐私风险告警及处理状态 |

以下概念不单独建业务实体：

- before/after snapshot：只保存受控字段 hash、版本和状态 code。
- Dashboard、Search、Unread count、Metric：都是 OperationsProjection 的不同视图。
- AlertDefinition：版本化代码目录，不是业务用户可编辑实体。
- ProjectionCheckpoint：投影内部技术状态，不是业务对象。
- Log/Trace：可观测性记录，不是 AuditEvent，也不是业务事实。

## 4. 必须审计的动作

以下动作必须产生追加式 AuditEvent：

- 业务状态转换和重要写入。
- RoleBinding、Membership、CaseCollaborator、ScopeGrant 和敏感授权变化。
- Founder 批准/驳回、Guardian 代录确认、offer 决定和人工结案。
- Task 分配、接受、拒绝、重派、完成和取消。
- 文件上传 capability、下载/预览/导出、版本激活、删除、恢复、legal hold 和最终清理。
- Student/Guardian soft-delete、恢复、重复记录处理和主要联系人变更。
- Assessment、客户文件、导出等高风险读取，包括允许和拒绝结果。
- 受信 Worker 的扫描、通知投递、dead-letter 和人工重放。

普通低风险列表浏览和纯 UI 操作不逐条形成 AuditEvent；仍可产生不含业务内容的 telemetry。

## 5. AuditEvent 最小内容

AuditEvent 只保存：

- organization、actor kind、实际 actor user opaque ID。
- event/action/resource type 和 resource opaque ID。
- succeeded、denied 或 failed 结果。
- occurred_at、request_id、event version、record version。
- 受控 reason code、before/after hash 和 causation/correlation opaque ID。

规则：

- user 动作必须保存实际 User ID，不能只写角色名称或“管理员”。
- system/worker 动作使用明确 actor kind，并引用导致它发生的 AuditEvent/OutboxMessage 或受控 job/run ID；不能伪造用户。
- 角色、Membership 和业务关系只作为当时授权证据的 opaque 引用，不复制员工或客户资料。
- 自由文字理由保存在拥有该业务事实的模块；Audit 只保存其记录 ID、hash 或受控 reason code。
- 审计查询中的可见摘要也遵守同样的字段白名单。

## 6. 原子提交规则

```text
业务命令
  -> 校验当前授权与 expected version
  -> 写业务模块权威事实
  -> 写必需 AuditEvent
  -> 如有异步 effect，再写 OutboxMessage
  -> 写 Idempotency result
  -> 同一 PostgreSQL 事务提交
```

- 必需 AuditEvent 保存失败时，业务修改必须回滚。
- 需要跨模块/外部副作用时，OutboxMessage 保存失败也必须回滚。
- 没有异步 effect 的写入只需 AuditEvent，不制造无意义 OutboxMessage。
- 高风险读取必须在返回资料或 capability 前成功保存 AuditEvent；失败时拒绝读取。
- 已被拒绝的敏感请求即使审计暂时失败也不能变成允许；返回安全失败并触发 critical alert。
- 普通 telemetry sink 失败不能回滚已提交业务；Audit 与 telemetry 不可混为一谈。

## 7. 并发与幂等

- 重要写入使用 `expectedRecordVersion`；不匹配时返回稳定 conflict，绝不 last-write-wins。
- 每个可重试命令使用 organization-scoped idempotency key 和 canonical request hash。
- 相同 key、相同 payload 返回第一次结果；相同 key、不同 payload 返回稳定冲突。
- in-progress 请求不并发执行第二次副作用。
- 重放必须复用第一次业务结果对应的同一组 AuditEvent、OutboxMessage（如有）和 Idempotency result，不能追加重复证据或副作用。
- 重复请求不得重复创建 Case、SchoolTarget、Task、Notification、DocumentVersion 或其他副作用。

IdempotencyRecord 仍由 Shared 的技术原语负责；Audit 只要求其与业务证据在同一事务使用。

## 8. Outbox 生命周期

```text
pending -> processing -> delivered
pending -> processing -> pending          （有界重试）
pending -> processing -> dead_letter
```

- Worker 使用 lease、attempt count 和 expected version，崩溃后可以安全重领。
- 每个消费者独立幂等；Outbox delivered 不代表消费者可以跳过自己的 DeliveryReceipt。
- dead_letter 保留原 effect、attempt、时间和受控错误 code，并产生 OperationalAlert。
- delivered 和 dead_letter 是终态，不物理删除或改写历史。
- 人工重放只能创建新的 replay attempt/reference，不能把旧 dead-letter 改成成功。
- Outbox payload 只允许 opaque ID、状态、版本、effect code 和受控标量。

## 9. Audit 读取授权

| 访问者 | Audit 权限 |
| --- | --- |
| Founder | 查看本组织租户业务审计的脱敏摘要 |
| 当前 Primary Advisor | 不读取全局 Audit；Case 时间线由各业务模块公开历史组成 |
| Admin 基础角色 | 只看 technical health，不读取租户客户业务审计 |
| Contractor | 不读取 Audit |
| Guardian、Student、Portal | 不读取 Audit |
| Platform operator | 只读取独立 platform audit/technical telemetry，不读取租户业务审计 |

- Audit 查询按 organization、时间、event/action/resource type 和 opaque ID 筛选。
- 不支持按姓名、邮箱、电话、文件名或自由文字搜索审计。
- Audit 导出属于高风险动作：Founder 专属、短时能力、再次审计，并继续保持字段白名单。

## 10. 租户与平台边界

- tenant AuditEvent 必须带 organization ID，并受 tenant RLS 和独立读取权限保护。
- platform audit 使用独立 schema/role/入口，不与 tenant AuditEvent 共享业务查询。
- Platform operator、供应商支持或基础设施人员不能借日志、统计、备份或审计读取租户内容。
- tenant Founder 也不能通过租户入口读取 platform credential、secret 或其他平台内部记录。
- Platform Billing 不进入 Release 1；其旧 audit、actor 和 database role 不得接入 Release 1 runtime。

## 11. OperationsProjection

Operations 可以维护：

- Case 当前阶段、下一步 code、负责人 opaque ID、截止时间和异常 count。
- open/overdue Task count、SchoolTarget 状态 count 和提醒/文件处理异常 count。
- Outbox backlog、Notification delivery、Document scan、runtime health 和 projection lag。

投影规则：

- 只消费 committed outbox/event，按 event identity 幂等更新。
- 保存 source version、checkpoint、content hash 和 freshness/stale 标记。
- 投影可以 truncate/rebuild；重建结果必须与同一 source watermark 一致。
- 投影不保存 Student/Guardian 姓名、联系方式、Assessment answer、文件名或自由文字 next action。
- 页面需要显示姓名等业务资料时，先按当前授权取得 projection 中的 opaque ID，再由 CRM/Cases 权威查询返回允许字段。
- 投影 stale 时必须明确显示；审批、提交、结案等写操作仍直接查询权威模块。

## 12. Operations 访问边界

| 访问者 | Operations 权限 |
| --- | --- |
| Founder | 组织级 Case 风险摘要、业务告警和 technical health |
| 当前 Primary Advisor | 自己 Case 的工作摘要和风险，不扩大 Case 权限 |
| Case Collaborator | 只在当前授权 scope 内查看最小投影 |
| Admin 基础角色 | technical health 和无客户内容的运行告警 |
| Contractor | 不访问 Operations dashboard |
| Platform operator | 平台 technical health；不读取租户 Case projection |

所有 dashboard 查询都必须使用 Access 多角色 AuthorizationContext，再以 Cases/Tasks 当前关系过滤。投影中的旧关系不能继续授权。

## 13. OperationalAlert

Release 1 告警目录只保留：

| 类别 | 典型告警 |
| --- | --- |
| Identity/Access | 登录失败突增、撤销积压、异常授权拒绝 |
| Documents | scan stuck、scan dead-letter、隔离对象积压 |
| Audit/Outbox | 必需审计失败、outbox stuck/dead-letter |
| Notifications | delivery dead-letter、提醒 scheduler 落后 |
| Privacy | PII canary 命中、敏感字段进入日志/telemetry |
| Region/Runtime | 香港依赖 unhealthy、错误 runtime adapter、关键服务 unavailable |
| Projection | lag、hash mismatch、rebuild mismatch |

预算、计费、正式数据导入和 backfill 告警不进入 Release 1。

告警生命周期：

```text
firing -> acknowledged -> mitigated -> closed
firing / acknowledged / mitigated -> needs_human
needs_human -> acknowledged / mitigated
```

- 告警按 alert definition + organization/platform scope + occurrence window 去重。
- acknowledge 只表示有人接手，不代表风险消失。
- mitigated 必须引用 runbook step/result；closed 必须先达到 detector recovery 条件。
- Operations 不能自动修改业务状态来“修复”告警。
- 告警不会绕过 Notifications 的九类业务提醒目录；Release 1 在 Operations 页面处理技术告警。

## 14. Telemetry 与日志

- Telemetry 只保存 route/command/job、结果、duration、retryable、稳定 error code 和必要 opaque scope。
- 日志和 telemetry 不得包含姓名、联系方式、Assessment answer、文件内容、Cookie、token、secret、完整表单、URL query 或自由文字 PII。
- request_id/trace_id 必须是服务端生成或清洗后的安全 opaque 值。
- telemetry sink 失败记录降级告警，但不能让普通业务成功变成失败。
- security/audit 必需路径失败时必须 fail closed，不能降级为普通 telemetry。
- 生产日志、临时副本、备份、审计和 telemetry 必须位于批准的香港数据边界。

## 15. 对外查询契约

| 查询 | 调用方 | 返回 |
| --- | --- | --- |
| `listTenantAuditEvents` | Founder 审计页 | 字段白名单内的 tenant AuditEvent 摘要 |
| `exportTenantAudit` | Founder 审计导出 | 重新授权后的短时脱敏导出能力 |
| `getCaseOperationsDashboard` | Founder/Advisor workspace | 当前授权 Case 的投影、freshness 和风险摘要 |
| `getTechnicalHealth` | Founder/Admin/Platform Operations | 无客户内容的健康、lag、queue 和 alert 摘要 |
| `listOperationalAlerts` | 按访问范围的内部页面 | 告警 code、severity、state、时间和 runbook reference |

## 16. 对外命令契约

| 命令 | 关键规则 |
| --- | --- |
| `appendRequiredAudit` | 只供业务 repository 在当前事务内调用；无普通 HTTP 入口 |
| `recordHighRiskRead` | 返回敏感资料/capability 前调用；失败即拒绝读取 |
| `claimOutbox` / `completeOutbox` / `failOutbox` | 受信 Worker；lease、expected version、有界重试 |
| `applyProjectionEvent` | 受信 projector；event identity 幂等 |
| `rebuildProjection` | 受信 Operations 命令；明确 source watermark 和 hash 校验 |
| `acknowledgeAlert` / `mitigateAlert` / `closeAlert` | Founder/Admin 按 alert scope；reason code 和 expected version |

Operations 命令只修改投影和告警状态，不能写任何业务模块权威表。

## 17. 依赖规则

| 模块 | 允许依赖 |
| --- | --- |
| Audit | Shared 技术原语；查询入口可使用 Access 授权契约 |
| Operations | Audit committed event、Access 和业务模块公开查询 |
| 业务模块 | Audit 的 runtime-neutral builder/transaction port |

明确禁止：

- Audit 或 Operations 直接写 Identity、Access、CRM、Cases、Tasks、Schools、Documents、Notifications 私有表。
- 业务模块从 Operations projection 读取授权或状态转换前置条件。
- Audit 依赖业务模块私有对象并复制完整 before/after payload。
- Operations 因 projection mismatch 自动纠正权威业务数据。
- 未配置的 runtime 回退到 memory、preview、legacy 或非香港生产 adapter。

## 18. 安全与一致性不变量

- AuditEvent、已提交 OutboxMessage 和 Delivery history 追加式保留，禁止物理删除。
- OperationsProjection 可重建，但 rebuild 不能改写 AuditEvent 或业务事实。
- tenant 与 platform 数据路径、数据库角色、查询接口和导出能力严格分离。
- 任何审计/告警/telemetry payload 都经过字段 allowlist 和敏感 key/value 拒绝。
- 所有组织级查询受 organization scope、RLS 和当前 Access 关系共同约束。
- 所有高风险读写在返回成功前已有必需审计证据。
- 香港边界必须以实际部署、存储、日志、备份和恢复证据验证；源码或本地测试不能代替。
- local-synthetic 通过只证明 Local Dev；AWS production 仍是独立 gate。

## 19. 与当前代码的差异

| 优先级 | 当前实现 | 目标契约 |
| --- | --- | --- |
| `P0` | 新业务命令尚未完整接入 Audit/Outbox | 每个已确认高风险动作都有原子审计证据 |
| `P0` | `MutationEffectBundle` 默认 Audit 与 Outbox 一对一 | 无异步 effect 时只写 Audit，不制造空 Outbox |
| `P0` | CRM migration 仍存在最终 purge 路径 | Student/Guardian 只软删除，ServiceCase 永久保留 |
| `P0` | Platform Billing audit/actor/role 仍在活跃源码 | 从 Release 1 runtime、registry 和查询边界隔离 |
| `P1` | Case dashboard projection 保存姓名、Case number 和自由文字 next action | 投影只保存 opaque ID、code、版本、计数和截止时间 |
| `P1` | dashboard 使用单一 actor role 和旧 Case stage | 使用 Access 多角色 context 和已确认 Case/Task 状态 |
| `P1` | alert catalogue 含月度预算告警 | Release 1 删除 budget/billing 告警，只保留运行和隐私风险 |
| `P1` | import/backfill ledger 仍由 Operations 导出 | 正式导入不在 Release 1；从公开入口移除 |
| `P1` | dashboard runtime 尚未接通，部分 telemetry/alert 只有契约 | 接通显式本地/生产 adapter；未配置时 fail closed |

这些差异进入后续开发拆分；本环节不修改产品代码或数据库。

## 20. 设计决策

| ID | 决策 |
| --- | --- |
| `SD-AUD-001` | AuditEvent 是不可改写证据；OperationsProjection 是可重建缓存 |
| `SD-AUD-002` | 必需审计与业务修改同事务；无异步 effect 时不强制创建 Outbox |
| `SD-AUD-003` | 高风险读取先成功审计再返回；普通 telemetry 失败不回滚业务 |
| `SD-AUD-004` | tenant audit 与 platform audit 分离，互相不能借入口读取 |
| `SD-OPS-001` | Operations projection 不保存客户 PII，不成为授权或状态机依据 |
| `SD-OPS-002` | Release 1 alert catalogue 移除 budget、billing、import 和 backfill |
| `SD-OPS-003` | Admin 只看 technical health；Founder/Advisor 按当前 Case 关系看业务投影 |
| `SD-OPS-004` | 香港数据边界必须用部署证据验证，不能由源码或本地测试推断 |

## 21. 本模块验收标准

项目负责人需要确认：

1. Audit 只保留 AuditEvent、OutboxMessage；Operations 只保留 Projection、OperationalAlert。
2. 必需审计失败时业务写入回滚；高风险读取在审计失败时拒绝返回。
3. 只有确有异步 effect 时才创建 OutboxMessage。
4. 看板、搜索、缓存和统计均可重建，不能作为授权或业务事实。
5. Operations projection 不保存客户姓名、联系方式、Assessment 或文件信息。
6. Founder 看租户审计；Admin 只看无客户内容的 technical health；平台人员不能读取租户内容。
7. Release 1 移除 budget/billing/import/backfill 告警，只保留运行、隐私和投影风险告警。
8. 香港数据边界必须在 AWS 阶段独立验证，不能以本地代码或测试代替。

确认后进入下一个模块：External Portal。
