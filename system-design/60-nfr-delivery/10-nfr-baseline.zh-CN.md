# 非功能基线与验收证据

状态：`approved`  
确认依据：项目负责人于 2026-08-25 确认多维质量标准、环境证据边界、失败/恢复验收、纵向开发分工和票据必备信息  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[非功能设计、评审与开发拆分索引](README.md)。

依据：[权限与安全设计](../40-permissions-security/README.md)、[API 与页面交互设计](../50-api-ui/README.md)。

## 1. 先看结论

Release 1 的“完成”不能只看页面能否打开，必须同时证明：

```text
业务正确
  + 权限正确
  + 数据不泄露
  + 重试不重复副作用
  + 失败时安全停止
  + 审计/告警可追踪
  + 环境证据没有混用
```

## 2. 非功能质量目标

| 质量维度 | Release 1 基线 |
| --- | --- |
| 安全 | 默认拒绝；高风险操作审计、限流、CSRF/来源防护和短时 capability |
| 隐私 | 日志、通知、Outbox、telemetry 和错误不含 PII、文件内容、Token 或 secret |
| 一致性 | expected version、actor-scoped idempotency、业务/Audit/Outbox 原子提交 |
| 可靠性 | Outbox/Worker 至少一次、消费者幂等、有界重试和 dead-letter |
| 可恢复性 | 备份恢复先进入隔离环境，验证 RLS、Audit、版本和租户边界 |
| 可观测性 | request/correlation/causation opaque ID、稳定 error code、Projection freshness、告警 |
| 可用性 | 依赖缺失或环境混用时 fail closed，页面显示 unavailable，不伪造成功 |
| 性能 | 采用本文件第 3 节的 Release 1 验收下限；测试负载、p95、异步延迟和错误率必须同时记录 |
| 可维护性 | 模块 owner、根级入口、公开 contract 和架构门禁可检查 |

## 3. 定量性能与容量基线

以下数值是 Release 1 的工程验收下限，不是客户规模承诺，也不是当前代码已达成的事实。若生产预测超过本基线，必须在 P6 前重新容量评审。

### 3.1 验收负载

| 项目 | 最低验收负载 |
| --- | --- |
| 并发交互 | 30 个内部 Session + 30 个 PortalSession + 5 个 Worker consumer |
| 代表性数据 | 5,000 Student、5,000 Guardian、10,000 Case、100,000 Task、100,000 Notification、50,000 Document metadata、500,000 AuditEvent |
| 文件 | 1 至 10 MiB；PDF/JPEG/PNG；扫描性能按 10 MiB 样本计 |
| 测试方式 | 固定合成数据、预热后至少 15 分钟稳定负载；记录版本、硬件/环境、样本数和 p50/p95/p99 |

### 3.2 响应与异步目标

| 路径 | Release 1 目标 |
| --- | --- |
| 授权后的列表/详情 API | p95 <= 1,000 ms |
| 普通业务 command | p95 <= 1,500 ms；不含文件字节传输和异步扫描 |
| 跨模块高风险 command | p95 <= 2,000 ms；Audit/Outbox 必须在事务内完成 |
| 内部主要页面可操作 | 预热后 p95 <= 3,000 ms；不得用假数据提前显示成功 |
| readiness | p95 <= 2,000 ms；依赖不完整返回稳定 unavailable code |
| Outbox 到 Notification/Task effect | 95% <= 60 秒；5 分钟仍未完成触发 backlog 告警 |
| 10 MiB 本地文件收据到扫描终态 | 95% <= 120 秒；超时进入可观察重试/失败，不伪造 clean |
| 非预期服务端错误 | 稳定负载期间 < 1%；业务拒绝、限流和故障注入不计入非预期错误 |

任何一项未测量都标记 `not_run`，不能以平均值替代 p95，也不能把 Local 数值写成 Production SLO 证据。

## 4. Session、Capability 与限流基线

### 4.1 时限

| 对象 | 冻结值 |
| --- | --- |
| 内部 Session | 每 User 最多 3 个 active；idle 8 小时；absolute 24 小时 |
| 敏感操作重新认证 | 最近强认证不超过 5 分钟 |
| 内部 Invite | 签发后 72 小时失效；只能使用一次 |
| PortalGrant / PortalSession | Grant 固定 7 天；每 Grant 最多 3 Session；idle 15 分钟；absolute 8 小时 |
| 上传/下载 capability | 单资源、单用途、最长 5 分钟；不得刷新原 capability |
| WarningManifestApproval | Release 1 不使用时间失效；manifest/warnings hash 改变即不再适用 |

### 4.2 默认限流

限流同时使用 actor/token keyed fingerprint 和短期来源 fingerprint；不保存 raw IP、Token、Portal key 或完整 User-Agent。

| 入口 | 默认阈值 |
| --- | --- |
| 登录 | 每账号 fingerprint 5 次/5 分钟；每来源 20 次/5 分钟；15 分钟内连续失败 10 次锁定 15 分钟 |
| Invite/Portal key 兑换 | 每 credential fingerprint 5 次/15 分钟；每来源 30 次/15 分钟 |
| Portal workspace 读取 | 每 PortalSession 120 次/分钟 |
| 上传 intent/receipt/扫描重试 | 每 actor 10 次/分钟；每 organization 100 次/分钟 |
| 下载/预览 capability | 每 actor 30 次/分钟 |
| 导出 | 每 actor 2 次/分钟；仍须逐资源授权和审计 |
| 其他高风险写入 | 每 actor 30 次/分钟；幂等重放仍计入口次数 |
| Outbox 重放/Projection 重建 | 每 actor 5 次/15 分钟；只允许受信运维入口 |

超过阈值返回稳定 `429 RATE_LIMITED`，不得改变业务状态。阈值只能通过受版本管理的环境策略收紧；放宽必须经过安全评审和项目负责人批准。

## 5. 保留、备份与恢复基线

- Release 1 不自动物理删除 Student、Guardian、Case、Task、Audit、Portal、Notification、Idempotency 或 Document metadata 历史。
- Document soft-delete 后 30 天内允许按授权恢复；30 天后普通恢复关闭。客户长期 retention 年限未确认前，不执行对象内容最终清理。
- Session/Invite 终态后立即使 verifier 和可用 provider credential 失效或清空；受控 metadata 与 Audit 继续保留，Release 1 不提供清理命令。
- 生产脱敏应用日志默认保留 30 天，聚合指标默认保留 90 天；AuditEvent 不进入普通日志 retention。
- 生产数据库目标：RPO <= 5 分钟、RTO <= 4 小时；Document 对象目标：RPO <= 24 小时、RTO <= 8 小时。
- 生产自动备份至少保留 7 天；每月执行一次隔离数据库恢复演练，每季度执行一次数据库 + Document 联合恢复演练。
- 上述恢复目标是 P6 验收目标，不是当前云环境已通过的证据；真实云环境未建立时保持 `not_run`。

## 6. 并发与锁顺序

高风险跨模块事务使用 owner 提供的公开锁定/校验入口，不允许一个模块直接操作另一个模块的私有表。统一顺序为：

```text
Organization/Access 上下文重验
  -> CRM（Student -> Guardian -> Relationship；同类按 UUID）
  -> Cases（ServiceCase -> CandidateListVersion -> SchoolTarget；同类按 UUID）
  -> Tasks（Task -> Assignment；同类按 UUID）
  -> Documents（Document -> Version；同类按 UUID）
  -> Audit -> Outbox -> Idempotency result
```

- Student 删除决定固定先锁 Student，再由 Cases owner 按 UUID 锁定该 Student 的未结案 Case 并重验；不得反向取锁。
- 同一层多记录统一按 UUID 升序锁定；禁止无界表锁和请求内动态改变顺序。
- 锁等待超过 2 秒返回稳定 `CONFLICT/RETRY_LATER`，事务回滚；不得绕过 expected version 强行提交。
- Membership、RoleBinding、Grant 或 Assignment 在事务开始后发生变化时，提交前必须重新验证必要授权事实。

## 7. 关键用户路径验收

至少覆盖以下纵向路径：

1. 内部登录 → Access 多角色解析 → Cases 工作台；
2. 建立候选名单 → Founder 审批 → Guardian 代录确认；
3. SchoolTarget `preparing` → 申请 Task → 提交证据 → Cases 重验；
4. 需要面试 → 面试辅助 Task → Contractor 脱敏工作区；
5. 文件上传 → 隔离 → 扫描 → 可用 → 精确下载 capability；
6. Task 分配/逾期 → Notifications 站内提醒 → 点击重新授权；
7. Portal key 兑换 → PortalSession → 白名单 workspace → 失效；
8. Founder 结案 → Audit → Outbox/Operations projection；
9. 重试、版本冲突、权限撤销、依赖不可用和 dead-letter。

## 8. 证据边界

| 环境 | 可证明 | 不能证明 |
| --- | --- | --- |
| Local synthetic | 本地 PostgreSQL、RLS、审计、幂等、上传/扫描模拟、显式 `cloud-synthetic` fake adapter 和页面基本行为 | AWS 区域、生产身份 provider、真实备份和生产权限 |
| Test database | 隔离数据、migration、并发、失败注入和回归 | Preview/Production 的部署和区域事实 |
| Preview | 明确声明的 Preview 页面/adapter 行为 | AWS S3/RDS/Cognito/备份、香港生产边界 |
| Production AWS | 实际区域、IAM、数据库、对象存储、队列、日志、备份和恢复证据 | 未运行或未记录的场景 |

任何报告都必须标记 `passed`、`failed`、`not_run`，并写明环境、时间、版本和脱敏证据。`cloud-synthetic` 的结果只能标记 `simulated`，不能写成云环境 passed。

## 9. 失败与恢复验收

必须验证：

- User/Membership/RoleBinding/Assignment/Grant 撤销后下一个请求立即失效；
- expected version 冲突不覆盖他人修改；
- 相同幂等请求不重复创建业务事实；
- 不同 payload 复用 key 返回稳定冲突；
- Audit 写失败时高风险写入回滚；
- Outbox 重复投递不重复创建 Task/Notification/DocumentVersion；
- 文件未 clean 时不允许下载或作为 Task 证据；
- Portal 失效统一拒绝且不枚举；
- Projection stale/rebuild 不改变权威事实；
- 依赖缺失、环境混用或区域不符时功能停止，不静默回退。

## 10. 安全与隐私验收

- Cookie 写请求有 CSRF/same-origin 防护；
- 登录、Portal 兑换、文件 capability、上传、导出和运维重放独立限流；
- API/页面默认 no-store，DTO 使用正向 allowlist；
- 日志、Audit 可见摘要、Notification、Outbox、dead-letter 和 telemetry 不含 PII、文件内容、Token 或 secret；
- tenant 与 platform audit/operations 路径分离；
- Student、Guardian、Case 历史不通过 purge 删除；
- 文件对象清理遵守 30 天恢复、retention、legal hold、引用和 Founder 批准门禁。

## 11. 开发拆分原则

确认本基线后，开发票据按纵向能力拆分，不按“把所有表做完”拆分：

| 责任方 | 主要范围 |
| --- | --- |
| Backend | module domain/application/repository、migration、API server、幂等、Audit/Outbox、focused tests |
| Frontend | typed client、页面状态、表单、错误/权限/unavailable 状态、browser contract tests |
| Platform Operations | PostgreSQL/LocalStack/ClamAV/queue、composition root、readiness、备份/恢复和环境证据 |
| 独立 QA | 只读验证、回归、失败注入、浏览器验收；报告 passed/failed/not_run |
| Architect | 契约冻结、跨模块协调、证据审查和用户确认 |

每张开发票据必须写：owner、范围、依赖、验收命令/场景、expected evidence、not_run、停止条件和回滚方式。

## 12. 确认结果

项目负责人已确认完成第 7 阶段；本文件同时冻结以下内容：

1. Release 1 的完成标准同时包含业务、权限、隐私、一致性、可靠性、可恢复性和可观测性。
2. Local、Test、Preview、Production 证据严格分开，所有结果标记 `passed/failed/not_run`。
3. 失败、重试、撤销、版本冲突、未扫描文件、Portal 失效和依赖不可用都必须有明确安全行为。
4. 高风险写入、读取、文件和 Portal 路径都必须通过安全与审计验收。
5. 开发按纵向能力拆分，由 Backend、Frontend、Platform Operations、独立 QA 和 Architect 分工。
6. 每张开发票据必须包含范围、依赖、验收证据、not_run、停止条件和回滚方式。
7. 第 3 至 6 节的性能、容量、Session、限流、保留、恢复和锁顺序作为 Release 1 技术基线。

本文件设计已完成；这些目标是否达成仍由各开发票据和环境门禁分别提供证据。
