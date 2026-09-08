# 数据保留、备份恢复与运行时安全验证

状态：`approved`  
确认依据：项目负责人于 2026-08-25 确认数据保留边界、文件清理门禁、备份恢复隔离、环境证据区分和依赖缺失时停止相关功能的 fail-closed 规则  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[权限与安全设计索引](README.md)。

业务依据：`BR-039`、`BR-050`、`BR-070`、`BR-071`，并引用 Shared、Documents、Audit、Operations 和 External Portal 设计。

## 1. 先看结论

本模块冻结三件事：

1. 哪些业务历史永远保留，哪些内容可以按规则清理；
2. 备份、恢复和灾备不能破坏软删除、审计和租户边界；
3. Local、Test、Preview、Production 的安全证据必须分开验证。

RPO/RTO、自动备份和演练频率已在[非功能基线第 5 节](../60-nfr-delivery/10-nfr-baseline.zh-CN.md)冻结；具体 S3 bucket、价格和客户长期业务 retention 年限仍须在 P6 生产部署时以真实环境和业务决定验证。retention 未配置时继续 fail closed。

## 2. 数据保留矩阵

| 数据类别 | Release 1 规则 | 是否允许物理删除 |
| --- | --- | --- |
| Student / Guardian 主档 | 只允许 soft-delete；数据库永久保留历史 | 不允许 |
| ServiceCase、SchoolTarget、Task 及状态历史 | 永久保留；结案不是删除 | 不允许 |
| AuditEvent | 追加式永久证据；禁止修改和删除 | 不允许 |
| Outbox delivered/dead-letter 及投递历史 | 作为异步证据保留；旧记录不可改写 | 不允许直接删除 |
| Notification | 用户不能删除；Release 1 不执行物理清理 | Release 1 不提供删除命令 |
| PortalViewer/Grant/Session | 失效、撤销和到期只改变状态；历史保留 | 不物理删除 |
| Document metadata、Version、ScanResult | metadata、版本、扫描和审计历史保留 | 不允许删除历史 |
| Document 对象内容 | soft-delete 后默认 30 天恢复窗口；满足 retention、无 hold、无引用和 Founder 批准后才可清理 | 只清对象内容 |
| OperationsProjection | 可 truncate/rebuild；不是业务证据 | 可以重建/替换 |
| IdempotencyRecord | Release 1 不执行物理清理；后续清理策略不得短于重试、Audit 和 Outbox 关联窗口 | Release 1 不提供清理命令 |
| 日志/telemetry 临时数据 | 只保留必要脱敏技术字段，按批准的安全 retention 管理 | 依保留策略清理 |

Student/Guardian 的永久保留规则优先于旧 migration 中的 purge 路径；历史 migration 不重写，后续用 corrective migration 禁止新 purge。

## 3. 文件清理门禁

```text
active
  -> pending_delete
  -> 30 天恢复窗口
  -> retention 已到期
  -> 无 legal hold
  -> 无有效业务引用
  -> Founder 明确批准
  -> 只清对象内容
```

任何条件未满足都必须 fail closed：

- 不删除对象内容；
- 保留 Document、Version、ScanResult metadata；
- 保留删除请求、批准、失败和审计历史；
- 不改变 Case、Task 或 SchoolTarget 状态。

未配置有效 retention 年限时，系统不得猜测，也不得沿用旧代码中的年限自动清理。

## 4. 备份规则

备份必须满足：

- 位于批准的香港生产数据边界；
- 传输中和静态加密；
- 使用独立最小权限的备份身份，不与应用写入身份混用；
- 记录备份版本、时间、来源环境、校验结果和恢复测试关联 ID；
- 不把备份下载到个人电脑、非批准区域或普通临时目录；
- 不能绕过 organization/RLS 或把 tenant 数据混入 platform audit 查询。

备份不是业务删除机制。即使某个文件对象内容被依法清理，也要按审计和备份治理规则处理其历史 metadata；不能以“删了主库”推断备份已删除。

## 5. 恢复流程

```text
发现故障
  -> 确认影响范围和恢复授权
  -> 选择经过校验的备份版本
  -> 恢复到隔离恢复环境
  -> 校验 schema、RLS、租户边界、Audit 链和版本一致性
  -> 只读验证关键业务查询
  -> Founder/Platform Operations 共同批准切换
  -> 记录恢复结果和新的 source watermark
```

恢复环境必须先隔离，不能直接覆盖生产数据库。恢复验证至少包括：

- User/Session 撤销和 Membership 状态仍然有效；
- Case、Task、Document、Portal 和 Audit 的 organization 边界没有串租户；
- `record_version`、Idempotency、Outbox 和 Audit 关联没有倒退；
- Document active version 仍要求 clean/available/未 revoked；
- Operations projection 能从权威来源重建；
- 恢复后的敏感查询仍先审计再返回。

恢复测试产生脱敏证据，不输出客户原始数据、连接串或 secret。

## 6. 运行时安全门

### 6.1 启动门

每个环境的 composition root 必须显式声明：

- `local-synthetic`、`test-database` 或 `production-aws`；
- Identity、Access、Cases、Tasks、Documents、Notifications、Audit、Outbox 和 Portal 所需 adapter；
- 数据库、队列、对象存储、扫描器和认证 provider readiness。

缺配置、区域不符、adapter 混用或关键依赖 unavailable 时启动失败，禁止静默切换 memory、JSON、mock、preview、legacy 或其他 region。

### 6.2 Readiness 门

Readiness 只返回受控 dependency code，例如：

```text
postgresql
postgresql_application
localstack_s3
localstack_sqs
clamav
identity_provider
```

不返回 hostname、连接串、bucket、queue、账号、路径、provider 原始错误或 secret。

### 6.3 运行时证据

| 环境 | 可以证明什么 | 不能替代什么 |
| --- | --- | --- |
| Local synthetic | 本地业务闭环、RLS/审计/幂等、失败路径和显式 `cloud-synthetic` fake adapter 联调 | 不能证明 AWS、香港区域或生产密钥配置 |
| Test database | 隔离测试数据和迁移兼容性 | 不能证明 Preview 或 Production |
| Preview | 明确声明的 Preview adapter/页面行为 | 不能证明生产对象存储、备份、区域或真实身份 provider |
| Production AWS | 实际部署、区域、权限、备份和恢复证据；必须拒绝 `cloud-synthetic` | 不能由源码或本地测试推断 |

## 7. 安全验证清单

每次发布或重大配置变更至少验证：

- 禁用 User、Membership、RoleBinding、ScopeGrant、TaskAssignment 和 PortalGrant 后下一个请求立即失效；
- Admin 单独登录不能读取客户业务；Contractor 不能离开当前面试 Task；
- Case、SchoolTarget、Document、Portal 和 Audit 不串 organization；
- 重复请求不重复产生业务事实、Task、通知、文件版本或 Portal Session；
- 高风险读取在 Audit 失败时拒绝返回；
- 文件未扫描 clean 时不能下载或作为提交证据；
- Portal 失效原因统一，不发生对象枚举；
- 日志、错误、Notification、Outbox、dead-letter 和 telemetry 无 PII、文件内容、Token 或 secret；
- Projection stale/rebuild 不改变权威业务事实；
- 生产运行模式不会加载 local/mock/preview/legacy adapter。

验证结果必须区分 `passed`、`failed` 和 `not_run`，并记录环境和脱敏证据；Local 通过不自动标记 Production 通过。

## 8. 本模块待确认内容

请确认以下 7 点：

1. Student、Guardian、ServiceCase、Task、Audit 和 Portal 历史按规则永久保留，不能通过 purge 删除。
2. Document 对象内容默认 30 天恢复窗口；retention、legal hold、引用或 Founder 批准任一缺失时不清理。
3. OperationsProjection 可重建；业务权威表和 AuditEvent 不因重建被覆盖。
4. 备份必须位于批准的香港边界，恢复先进入隔离环境并经过租户/RLS/Audit/版本校验。
5. 运行模式显式选择，缺依赖或 adapter 混用时 fail closed。
6. Local、Test、Preview、Production 证据严格分开，不能互相代替。
7. 安全验证统一报告 passed、failed、not_run，不把未验证的云或生产能力写成已完成。

本文件已确认。权限与安全设计阶段全部完成，下一步进入 API 与页面交互设计。
