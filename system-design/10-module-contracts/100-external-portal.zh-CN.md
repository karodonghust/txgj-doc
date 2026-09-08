# External Portal 模块契约

状态：`approved`  
确认依据：项目负责人于 2026-08-25 指示继续进入下一模块  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v31`

返回[模块契约索引](README.md)。

业务依据：`BR-060`、`BR-063`，并引用 `BR-029`、`BR-033`、`BR-034`、`BR-039`、`BR-050`、`BR-070`、`BR-071`。  
现状依据：[External Portal 与 Platform Billing 现状分析](../../current-state-analysis/80-portal-billing.zh-CN.md)。

## 1. 一句话职责

External Portal 只回答：

> 某位与一个 Case 关联的 Guardian，是否持有仍有效的 7 天只读入口和当前 Session；如果有效，只返回该 Case 的对客字段白名单。

Portal 不是家长版 ERP，也不是内部账号系统。

## 2. 负责与不负责

| External Portal 负责 | External Portal 不负责 |
| --- | --- |
| PortalViewer、PortalGrant、PortalSession | 内部 User、Membership 或 RoleBinding |
| 单 Case、7 天、可撤销的只读访问 | 修改 CRM、Case、Task、SchoolTarget 或 Document |
| bearer access key 兑换独立 Portal Session | Guardian 注册、密码、OTP 或内部登录 |
| 严格字段白名单和请求时授权 | 文件、Assessment、联系方式、内部备注或审计展示 |
| 通用拒绝、限流和安全审计 | 系统外 Email、SMS、WhatsApp 或链接发送 |

## 3. 最小核心对象

Release 1 只保留三个核心对象：

| 对象 | 含义 | 关键约束 |
| --- | --- | --- |
| `PortalViewer` | 一个 Case 中被明确选择的 Guardian 查看者 | 绑定 organization、Case 和当前 GuardianRelationship；不是 User |
| `PortalGrant` | 一次固定 7 天的只读入口 | 绑定一个 Viewer 和一个 Case；scope、有效期和 secret identity 不可修改 |
| `PortalSession` | 入口兑换后的浏览器会话 | 绑定一个 Grant/Case；独立于内部 Session；受 idle/absolute timeout 限制 |

以下概念不单独建实体：

- PortalProjection：每次读取时从 Cases 的公开查询构建白名单 DTO，不保存第二份业务真相。
- AccessKey：PortalGrant 的一次性显示 bearer secret；数据库只保存 keyed hash/fingerprint。
- PortalSecurityEvent：复用 AuditEvent，不再建立重复安全事件实体。
- PortalIdempotencyRecord：复用 Shared IdempotencyRecord。
- Applicant/Student viewer：Release 1 只有 Guardian viewer，不建立 Student Portal 身份。

## 4. PortalViewer

- 当前 Primary Advisor 从该 Case Student 的 active GuardianRelationship 中明确选择 Guardian。
- 同一个 Guardian 关联多个 Student 或 Case 时，每个 Case 分别建立 Viewer/Grant；绝不自动扩展访问。
- Viewer 只保存 GuardianRelationship opaque ID，不复制姓名、邮箱、电话或身份证明。
- GuardianRelationship 结束、Guardian/Student deleted、Case 不再可访问时，Viewer 立即失效。
- 更换主要联系人不会自动转移 Portal；新的 Guardian 必须被 Primary Advisor 明确选择。
- Viewer 只能从 active 变为 inactive，历史不物理删除。

## 5. PortalGrant

```text
active -> revoked
active -> expired
```

- 只有当前 Primary Advisor 可以创建 PortalGrant。
- Founder 不能创建或重新生成入口，只能紧急撤销。
- 每个 Grant 从服务端签发时起固定有效 7 天；客户端不能选择更短/更长日期。
- Grant 的 organization、Case、Viewer、capability version、issued_at 和 expires_at 创建后不可修改。
- 到期不能延期，也不能恢复 expired/revoked Grant。
- 需要继续访问时，当前 Primary Advisor 创建全新 Grant、ID、secret 和 7 天期限。
- 同一 Case + Viewer 最多一个 active Grant；重新生成时旧 Grant 及其全部 Session 先原子撤销。
- 撤销立即清除可验证 secret hash 并撤销全部 active Session，保留 fingerprint 和审计历史。

## 6. 创建与撤销授权

创建时必须重新检查：

1. actor User、Membership 和 Advisor RoleBinding 当前 active。
2. actor 是该 Case 当前 Primary Advisor。
3. Case 当前为 active 或 paused，不在 termination pending/closed。
4. Viewer 的 GuardianRelationship 仍 active，且属于该 Case Student。
5. organization active，且没有任何 Platform Billing/Subscription 前置条件。

撤销权限：

| 操作者 | 权限 |
| --- | --- |
| 当前 Primary Advisor | 创建、重新生成、查看授权摘要、撤销自己 Case 的 Grant |
| Founder | 查看组织内授权摘要并紧急撤销；不能创建或重新生成 |
| 其他 Advisor/Case Collaborator | 不可操作 PortalGrant |
| Admin 基础角色、Contractor | 不可操作 PortalGrant |

撤销 reason code、操作者、时间和 expected version 必填并审计。

## 7. Access key 形式与传递

- Grant 创建时生成至少 256-bit 随机 bearer secret，只向当前 Primary Advisor 显示一次。
- 数据库只保存使用独立 server pepper 计算的 keyed hash 和非敏感 fingerprint；不保存明文 secret。
- 系统不发送 Portal Email/SMS/WhatsApp；Advisor 在系统外把入口交给正确 Guardian。
- 建议入口使用 URL fragment 承载 raw key；fragment 不进入 HTTP request、服务端日志或 Referer。
- Portal access 页面立即用 `history.replaceState` 清除 fragment，再通过 same-origin POST 兑换 Session。
- raw key 不进入 query string、cookie、localStorage、analytics、日志、错误、Audit 或页面 HTML。
- 页面禁止第三方脚本/追踪资源，并使用严格 CSP、`Referrer-Policy: no-referrer` 和 `Cache-Control: no-store`。

bearer link 被转发即可能被他人使用；Release 1 不增加注册、密码或 OTP。怀疑泄露时由 Primary Advisor 或 Founder 立即撤销并重新生成。

## 8. PortalSession

```text
active -> revoked
active -> expired
```

固定安全参数：

- 每个 Grant 最多 3 个 active Session。
- idle timeout：15 分钟。
- absolute timeout：8 小时。
- Session 的任何期限都不得晚于 Grant expires_at。
- 每次成功读取可按上限刷新 idle expiry，但不能延长 absolute expiry 或 Grant。

Session secret：

- 使用独立高强度随机 secret，数据库只保存 keyed hash。
- 浏览器只通过 `__Host-`、`HttpOnly`、`Secure`、`SameSite=Strict`、`Path=/` Cookie 保存。
- 不复用内部 ERP Session/Cookie，也不能兑换成内部身份。
- Logout、Grant 撤销/到期、Viewer/关系失效或 Case 失效后，下一个请求立即失败。

## 9. 请求时有效性检查

每次 Portal workspace 请求都重新检查：

- raw Session hash 对应唯一 active PortalSession。
- Session idle/absolute deadline 未到，Grant 的 7 天 deadline 未到。
- Grant active，scope/version 与 Session 的 organization/Case/Viewer 完全一致。
- Viewer 和 GuardianRelationship 当前 active，并仍属于该 Case Student。
- 创建 Grant 的 Advisor 仍是当前 active Primary Advisor。
- Case 当前为 active 或 paused，organization active；termination pending/closed 立即拒绝。

任一条件失败都拒绝访问。旧 Primary Advisor 离任或 Case 改派后，旧 Grant 不自动转给新 Advisor；新 Primary Advisor 必须创建新入口。

## 10. 对客字段白名单

Portal 唯一允许返回：

| 区域 | 字段 |
| --- | --- |
| Case | customer-facing stage code/label、last customer-visible update time |
| 学校 | 已批准展示的学校名称、customer-facing application status code/label |
| 消息 | Advisor 明确标记 customer-visible 的纯文本 body、published_at |
| 行动项 | Advisor 明确标记 customer-visible 的标题、deadline、completed |

学校显示规则：

- 只显示属于 Founder 已批准且 Guardian 已确认名单版本、仍有效的 SchoolTarget。
- 内部 SchoolTarget 状态通过固定、版本化的 customer-facing mapping 输出，不直接暴露内部 reason、blocker 或操作记录。
- 不为 Portal 单独复制或修改 SchoolTarget 状态。

明确禁止返回：

- Case number、Student/Guardian 姓名或联系方式。
- Assessment、内部备注、内部 Task/Assignee、候选草稿或驳回理由。
- 文件、文件名、下载/预览/导出入口。
- AuditEvent、员工信息、内部时间线、School overlay/crawler 治理信息。
- 任何未在白名单中列出的字段；新增字段必须先更新业务基线和 capability version。

## 11. 消息与行动项

- Portal 不拥有消息/行动项的内部业务真相，只展示 Cases 公开查询返回的 approved customer-visible facts。
- 只有当前 Primary Advisor 能在内部 Case 工作区创建或修改对客消息/行动项并明确标记可见。
- 发布前必须是纯文本并经过长度、控制字符和危险链接/markup 校验。
- Portal Viewer 不能回复、评论、确认、完成行动项或上传证据。
- 将行动项标为 completed 仍由内部有权员工操作；Portal 只读结果。
- 取消 customer-visible 后，下一个 Portal 请求不再显示，但内部历史和 AuditEvent 保留。

## 12. 明确禁止的 Portal 动作

Portal 永久拒绝：

- 修改 Student、Guardian、Case 或申请资料。
- 确认/驳回候选名单、选择学校或确认/拒绝 offer。
- 创建、接受、完成或取消 Task。
- 上传、查看、下载、预览、导出或删除 Document。
- 评论、留言、发送消息或提交自由文字表单。
- 查看其他 Case，或根据同一 Guardian/Student 自动发现 Case。

前端隐藏不是安全控制；服务端没有对应写命令和文件接口。

## 13. 通用错误、限流与防枚举

- invalid、unknown、revoked、expired Grant/Session 和超过 Session 上限统一返回 `PORTAL_ACCESS_INVALID`。
- 不向外部响应区分 Guardian、Case、Viewer、Grant 或 Session 是否存在。
- Access key redemption、Session read 和失败尝试使用独立速率限制。
- 限流 key 使用短期、受控的 keyed fingerprint；日志不保存 raw key、完整 IP、User-Agent 或 URL。
- Portal 所有响应使用 `no-store`，敏感页面禁止被搜索引擎索引或第三方 frame 嵌入。
- CSRF/Origin 校验保护 Session/Logout 入口；Portal 没有其他写业务动作。

## 14. 审计

以下动作写入 tenant AuditEvent：

- Viewer 创建/失效。
- Grant 创建、重新生成、撤销和到期。
- Access key 成功/失败兑换、限流和 Session 创建/退出/失效。
- Portal workspace 高风险读取的允许/拒绝结果。

Audit 只保存 organization、Portal/Case/Viewer opaque ID、action、outcome、request ID、受控 reason code 和版本；不保存 raw key/session、Guardian 资料、Portal 消息正文或学校名称。

## 15. 对外查询契约

| 查询 | 调用方 | 返回 |
| --- | --- | --- |
| `listCasePortalViewers` | 当前 Primary Advisor | active GuardianRelationship 候选和现有 Viewer 的最小摘要 |
| `listCasePortalGrants` | 当前 Primary Advisor、Founder | fingerprint、状态、签发/到期时间和 Session count；不返回 secret |
| `readPortalWorkspace` | Portal Session 路由 | 请求时构建的严格 allowlist DTO |

Portal workspace 查询不接受客户端传入 organization、Case 或 Viewer ID；只从当前 Session/Grant scope 解析。

## 16. 对外命令契约

| 命令 | 关键规则 |
| --- | --- |
| `ensurePortalViewer` | 当前 Primary Advisor；选择当前 Case 的 active GuardianRelationship |
| `issuePortalGrant` | 当前 Primary Advisor；服务端固定 7 天；raw secret 只返回一次 |
| `reissuePortalGrant` | 原子撤销旧 Grant/Session，再创建全新 Grant/secret/7 天期限 |
| `revokePortalGrant` | 当前 Primary Advisor 或 Founder；reason、expected version、幂等 |
| `redeemPortalGrant` | 公共入口；hash discovery、限流、最多 3 个 Session、统一错误 |
| `revokePortalSession` | 当前 Portal Session logout 或 Grant 联动撤销 |
| `readPortalWorkspace` | 每次重新授权；Audit 成功后返回白名单 |

所有写入使用 expected version、idempotency、Audit 和必要 Outbox；Portal 不建立独立 idempotency/audit 数据孤岛。

## 17. 依赖规则

| 类型 | 允许 |
| --- | --- |
| 内部授权 | `Access` 的当前员工 AuthorizationContext |
| 客户关系 | `CRM` 的 GuardianRelationship eligibility 查询 |
| Case 投影 | `Cases` 的 portal-safe facts 和当前 Primary Advisor/Case 状态 |
| 平台能力 | `Shared`、`Audit` 的公开契约 |

明确禁止：

- ExternalPortal 直接写 Identity、Access、CRM、Cases、Tasks、Documents 或 Notifications 私有表。
- CRM/Cases/页面直接写 PortalViewer、Grant 或 Session 表。
- 依赖 Platform Billing、Subscription、Entitlement、Data Reviewer 或平台角色。
- 使用内部 Session、Portal projection、缓存或客户端参数作为授权来源。
- local/mock/preview adapter 静默进入 production-aws；未接通必须 fail closed。

## 18. 安全与一致性不变量

- Portal 身份与内部 User/Session/Cookie/Role 永久分离。
- Viewer、Grant、Session 都绑定同一个 organization 和 Case，任何 scope mismatch 立即拒绝。
- Grant 固定 7 天、不可延期；Session 期限不能超过 Grant。
- raw bearer secret 只出现一次，不进入数据库、URL query、日志、审计、analytics 或持久浏览器存储。
- 每次读取重新检查 Grant、Session、Viewer、GuardianRelationship、Case、issuer 和 organization。
- Portal DTO 使用正向 allowlist；未知字段不能因序列化对象扩展而自动暴露。
- 所有 grant/session/security 历史保留，不物理删除。
- Portal 不依赖 Billing，且永远没有文件和业务写入口。
- 香港生产数据边界必须由实际部署证据验证；Local Dev 不能代替。

## 19. 与当前代码的差异

| 优先级 | 当前实现 | 目标契约 |
| --- | --- | --- |
| `P0` | Founder 和 Primary Advisor 都可创建 Grant | 只有当前 Primary Advisor 可创建；Founder 只紧急撤销 |
| `P0` | expires_at 可由客户端选择，最多 7 天 | 服务端固定 issued_at + 7 天，客户端不传期限 |
| `P0` | Portal DTO 返回 Case number | 从 allowlist、domain DTO、route 和 UI 删除 |
| `P0` | Viewer 支持 guardian 和 applicant Student | Release 1 只保留 Case-scoped Guardian viewer |
| `P0` | policy 仍接受 Data Reviewer、Subscription/past_due | 移除旧角色和 Billing 依赖 |
| `P0` | grant/session/workspace runtime 固定 unavailable | 接通显式 repository/service；未配置时继续 fail closed |
| `P1` | `rotate` 名称容易被理解为延期旧入口 | 改为 reissue：撤销旧 Grant，创建全新 Grant/secret/期限 |
| `P1` | PortalSecurityEvent、PortalIdempotencyRecord 重复平台能力 | 收敛到 AuditEvent 和 Shared IdempotencyRecord |
| `P1` | 页面/API 骨架已有，但 workspace 依赖不可用 | 接通严格白名单 DTO，并覆盖撤销/过期/改派即时失效 |
| `P1` | Platform Billing 页面、API、migration 依赖仍在源码 | 从 Release 1 入口、runtime 和 module registry 隔离；历史 migration 不重写 |

这些差异进入后续开发拆分；本环节不修改产品代码或数据库。

## 20. 设计决策

| ID | 决策 |
| --- | --- |
| `SD-PORTAL-001` | 核心对象只保留 PortalViewer、PortalGrant、PortalSession |
| `SD-PORTAL-002` | PortalViewer 只代表一个 Case 中被明确选择的 Guardian，不是 User |
| `SD-PORTAL-003` | Grant 由当前 Primary Advisor 创建，固定 7 天且不可延期；Founder 只撤销 |
| `SD-PORTAL-004` | bearer key 只显示一次，以 URL fragment 传递，兑换独立安全 Session |
| `SD-PORTAL-005` | 每 Grant 最多 3 个 Session，15 分钟 idle、8 小时 absolute |
| `SD-PORTAL-006` | Portal 每次请求实时重验，DTO 使用严格正向字段 allowlist |
| `SD-PORTAL-007` | Portal 只读且单 Case，不支持文件、确认、回复、上传或任何业务写入 |
| `SD-PORTAL-008` | External Portal 与 Platform Billing 完全解耦 |

## 21. 本模块验收标准

项目负责人需要确认：

1. Portal 核心对象只有 Viewer、Grant、Session；Guardian 不是内部 User。
2. 只有当前 Primary Advisor 能创建全新 7 天入口；Founder 只能紧急撤销。
3. 原入口不能延期；重新分享必须撤销旧入口并生成全新 secret。
4. 每个 Grant 最多 3 个 Session，15 分钟 idle、8 小时 absolute。
5. Portal 只展示对客阶段/更新时间、已批准学校进度、可见消息和行动项，不展示 Case number。
6. Portal 不能确认、回复、修改、上传、查看文件或访问其他 Case。
7. Grant/Session/关系/Case/issuer 任一失效，下一个请求立即拒绝且不泄露原因。
8. Portal 不依赖 Platform Billing、Subscription 或内部员工角色模型。
9. Case 暂停时入口仍按原期限有效；termination pending 或 closed 后立即失效。

确认后进入最后一个模块：Shared 与入口适配层。
