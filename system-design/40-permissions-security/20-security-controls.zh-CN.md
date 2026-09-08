# 高风险操作与安全控制

状态：`approved`  
确认依据：项目负责人于 2026-08-25 确认高风险操作的审计原子性、CSRF、限流、防枚举、PII/secret 脱敏、文件/Portal 安全边界和香港运行 fail-closed 规则  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[权限与安全设计索引](README.md)。

业务依据：`BR-070`、`BR-071`，并引用已确认的 Identity、Cases、Tasks、Documents、Notifications、Portal 和 Shared 流程。

## 1. 先看结论

安全控制不是某一个角色的工作，而是每个请求的固定顺序：

```text
身份有效
  -> 组织/角色/业务关系有效
  -> 输入和 expected version 有效
  -> 高风险动作限流/防 CSRF/重新认证（如适用）
  -> 必需 AuditEvent 成功
  -> 写业务事实 + Outbox + Idempotency 原子提交
  -> 返回最小 DTO
```

任何关键条件无法确认，系统默认拒绝或 fail closed；不能用页面隐藏、缓存、日志或投影替代服务端检查。

## 2. 高风险操作分类

| 类别 | 操作示例 | 必须控制 |
| --- | --- | --- |
| 身份/成员 | 邀请、激活、禁用 User、撤销全部 Session | 强身份、限流、Audit、幂等 |
| 角色/授权 | 分配/撤销角色、创建/批准/撤销敏感 ScopeGrant | Founder/Admin 资格、禁止自批、expected version、Audit |
| 案件决策 | Founder 名单审批、Guardian 代录确认、offer 决定、Founder 结案 | owning module 规则、版本、Audit、幂等 |
| 任务/申请 | 重派、完成申请提交 Task、取消 Task | 当前 Assignment、完成证据、Audit、幂等 |
| 文件 | 上传、下载/预览、导出、版本激活、删除、恢复、legal hold、最终清理 | 文件级授权、扫描、短时 capability、Audit、限流 |
| Portal | Grant 创建/撤销、key 兑换、Session 建立 | 单 Case、固定期限、限流、防枚举、Audit |
| 技术运维 | Outbox 重放、Projection 重建、告警关闭、运行模式切换 | 受信 server-only command、scope、reason、Audit |
| 高风险读取 | 客户文件、Assessment、联系方式、审计导出 | 读前 Audit 成功、最小字段、限流、no-store |

## 3. 审计与原子性

高风险写入固定执行：

```text
重验授权和 expected version
  -> 写 owning module 权威事实
  -> 写必需 AuditEvent
  -> 如有异步 effect，写 OutboxMessage
  -> 写 Shared Idempotency result
  -> 同一事务提交
```

- AuditEvent 写失败，业务写入回滚。
- Outbox 写失败，相关异步业务写入回滚。
- 高风险读取先成功写 AuditEvent，再返回资料、下载 capability 或导出能力。
- 审计只保存实际 actor、资源 opaque ID、结果、版本、受控 reason code 和 hash；不保存 PII、文件内容、Cookie、Token、secret 或完整表单。
- 被拒绝的敏感请求也要留下脱敏审计事实；审计失败时不能把拒绝变成允许。

## 4. 并发、幂等与重放

- 所有高风险写命令使用 expected record version；版本不匹配返回 `STALE_VERSION`，禁止静默覆盖。
- 所有可重试写命令使用 Shared actor-scoped idempotency key 和 canonical request hash。
- 同一 scope/key + 同一 hash 返回第一次结果；不同 hash 返回 `IDEMPOTENCY_KEY_REUSED`。
- in-progress 命令不并发产生第二次副作用。
- Outbox dead-letter 人工重放创建新 replay reference，不能修改旧证据。
- 重试不重复创建 Case、SchoolTarget、Task、Notification、DocumentVersion、PortalGrant 或 Session。

## 5. CSRF、来源和 Cookie

内部 ERP 使用 Cookie 身份时：

- 所有改变状态的请求必须通过 same-origin/Origin 校验和 CSRF 防护；`SameSite` 不能作为唯一控制。
- CSRF token 不能放进 URL、日志、Notification 或普通 DTO。
- Cookie 使用 HttpOnly；生产使用 Secure 和明确 SameSite 策略。
- Portal Session 使用独立 `__Host-`、HttpOnly、Secure、SameSite=Strict、Path=/ Cookie。
- logout、Portal key 兑换、成员邀请兑换和敏感授权入口分别设置限流。
- redirect/return URL 只能使用固定内部 route code，不接受任意外部 URL。

## 6. 限流与防枚举

限流是入口控制，不改变业务事实。以下入口必须有独立的 keyed rate limit：

| 入口 | 防护目标 |
| --- | --- |
| 登录、邀请兑换、Session 建立 | 防凭证猜测和账号枚举 |
| Portal access key 兑换、Session 读取 | 防 bearer key 猜测和 Case 枚举 |
| 下载/预览 capability、批量导出 | 防文件内容滥用 |
| 文件上传、对象收据、扫描重试 | 防资源消耗和重复副作用 |
| 高风险写命令 | 防重复提交和自动化滥用 |
| Audit 导出、Projection 重建、Outbox 重放 | 防运维权限滥用 |

限流 key 只保存短期 keyed fingerprint 或受控 opaque scope；不记录 raw key、完整 IP、User-Agent、URL 或 Token。

具体阈值、封禁时长和 organization 配额已在[非功能基线第 4 节](../60-nfr-delivery/10-nfr-baseline.zh-CN.md)冻结；实现不得沿用与该基线冲突的旧代码数值。

所有认证失败、Portal 失效、目标不存在或授权不足尽量收敛到稳定错误，避免通过响应差异枚举 User、Guardian、Case、Grant、Document 或 Task。

## 7. 输入、DTO 与错误防泄露

- Route 只接受正向字段 allowlist；未知字段拒绝或丢弃并记录稳定错误。
- 不直接 spread domain object、database row、provider response 或 Error。
- 正式 API 使用 `/api/v1/**` 统一 envelope；私有响应默认 `Cache-Control: no-store`。
- 错误不返回 stack、SQL、provider output、环境变量、文件路径、对象 key、bucket、queue、Cookie、Token、secret 或 PII。
- `details` 只允许安全版本、diff token、稳定 readiness code 和明确 allowlist 字段。
- Portal 使用统一 `PORTAL_ACCESS_INVALID`；外部不区分失败原因。
- 下载/导出 DTO 不返回公开 URL；只返回短时 capability 或固定内部动作结果。

## 8. 数据泄露防护

以下内容不得进入日志、Audit 可见摘要、Notification、Outbox payload、dead letter、telemetry 或错误：

- Student/Guardian 姓名、生日、邮箱、电话；
- Assessment answer、内部备注、自由文字业务原因；
- 文件内容、文件名、对象 key、预签名 URL、扫描原始输出；
- Cookie、Token、raw Portal key、Session secret、邀请 secret；
- 完整表单、完整 query/body、第三方 provider 响应。

允许的公共技术字段：

- organization/actor/resource opaque ID；
- event/effect code、状态 code、版本、计数、截止时间；
- request/correlation/causation opaque ID；
- hash、attempt count、稳定错误码和脱敏运行指标。

## 9. 文件和外部入口专项规则

### 9.1 Documents

- 文件必须在隔离和扫描 clean 前不可预览、下载、导出或作为提交证据。
- 每次上传、下载、预览、导出、删除、恢复和清理都重新执行资源级授权。
- Application Assignee 只访问自己负责学校的必要文件；Contractor、Guardian、Student、Portal 永久拒绝。
- 对象存储必须私有；对象引用使用 opaque ID，不含姓名、学校名、Case number 或原始文件名。

### 9.2 External Portal

- Portal 使用独立 Viewer/Grant/Session，不兑换内部 User。
- Grant 固定 7 天；每个 Grant 最多 3 个 Session，idle 15 分钟、absolute 8 小时。
- raw bearer secret 只显示一次，不进入 query、日志、Audit、analytics 或持久浏览器存储。
- Portal 只读、单 Case、字段 allowlist；不提供文件或任何业务写命令。

## 10. 香港数据边界与运行模式

生产敏感数据、日志、临时副本、备份、Audit、telemetry 和对象存储必须使用批准的香港生产边界。证明方式必须是部署和运行证据，而不是源码注释或本地测试。

运行模式显式选择：

| 模式 | 规则 |
| --- | --- |
| `local-synthetic` | 只能证明本地开发闭环，不得进入生产构建 |
| `test-database` | 只使用隔离测试数据和依赖 |
| `production-aws` | 只允许批准的香港生产 adapters，检测到 mock/local/preview/legacy 即启动失败 |

未配置、依赖不可用或区域不符合时 fail closed；不能静默使用 memory、JSON、preview、mock、其他 region 或开发凭证。

## 11. 本模块待确认内容

请确认以下 7 点：

1. 高风险写入必须经过授权、版本、幂等、Audit，并与业务事实原子提交。
2. 高风险读取必须先成功审计，再返回资料、文件 capability 或导出能力。
3. Cookie 写请求必须有 CSRF/same-origin 防护；Portal 使用独立安全 Cookie。
4. 登录、Portal 兑换、文件 capability、上传、导出、重放等入口分别限流并防枚举。
5. 日志、通知、审计摘要、Outbox、dead letter 和 telemetry 不得包含 PII、文件内容、Token 或 secret。
6. 文件和 Portal 继续执行独立的资源级授权和短时 capability 规则。
7. 生产香港数据边界必须用部署证据验证；运行模式不完整时 fail closed。

本文件已确认。下一步进入数据保留、备份、恢复和运行时安全验证设计。
