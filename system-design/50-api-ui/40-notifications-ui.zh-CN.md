# Notifications 入口 API 与页面

状态：`approved`  
确认依据：项目负责人于 2026-08-25 确认最小站内文案、当前 User 查询、点击重新授权、必需提醒不可关闭和 suppressed/failed 不对用户展示  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[API 与页面交互设计索引](README.md)。

业务依据：[Notifications 通知流程与状态机](../30-business-flows/40-notifications.zh-CN.md)。  
权限依据：[高风险操作与安全控制](../40-permissions-security/20-security-controls.zh-CN.md)。

## 1. 先看页面目标

通知入口只回答：

1. 当前内部用户有没有待办提醒？
2. 这条提醒是否已读？
3. 点击后当前用户是否仍有权进入目标页面？

通知不是业务事实，不是 Task，也不是权限。页面不能因为收到通知就直接显示或操作目标资源。

## 2. 正式 API 路径

| Method | 路径 | 类型 | 用途 |
| --- | --- | --- | --- |
| `GET` | `/api/v1/notifications` | query | 当前 User 的站内通知列表 |
| `GET` | `/api/v1/notifications/unread-count` | query | Workspace shell 未读数 |
| `POST` | `/api/v1/notifications/:notificationId/read` | command | 当前 recipient 标记已读 |
| `POST` | `/api/v1/notifications/:notificationId/resolve-target` | command/query | 点击时重新授权并返回固定内部路由 |

正式 API 只使用 `/api/v1/**`；所有查询以当前 Session User 为 recipient，不接受客户端传入其他 User ID。

## 3. 通知列表 DTO

每一项只返回最小字段：

```text
notification: id, content_code, status, created_at, read_at, record_version
target: fixed target_kind + opaque target token（可为空）
allowed_actions: read / resolve_target
```

Release 1 `content_code` 固定使用最小“有待办事项”文案，例如：

```text
PENDING_ITEM -> 有待办事项需要处理
```

不返回：

- Student、Guardian、Case、学校、Task、文件名称；
- 自由文字原因、Assessment、联系方式；
- 私有 URL、对象 key、Token、Cookie 或完整业务 payload。

## 4. 通知来源与页面展示

页面不需要显示 effect 的具体业务正文，只需按统一文案显示待办提醒。通知来源可能包括：

- Task 分配、重派、拒绝；
- 候选名单提交 Founder 审批或审批结果；
- Task 到期前 3 天、1 天和逾期每日提醒；
- 全部学校终态且无未完成 Task 时的结案选择提醒。

不同来源仍必须按 recipient + effect + source + 业务日期/版本去重。页面不能把多条不同 Task 合并成一条，也不能因为一个人同时拥有多个角色而显示多条同义提醒。

## 5. 已读流程

```text
unread -> read
```

标记已读命令必须包含：

```text
Idempotency-Key: opaque-retry-key
expected_record_version: current version
```

- 只有当前 recipient 可以标记已读。
- 重复点击返回原结果，不创建新的通知或业务事实。
- 标记已读不停止未来日期的到期/逾期提醒。
- Release 1 不提供删除、撤回、静音或关闭必需通知的用户命令。

## 6. 点击目标流程

```text
点击通知
  -> POST resolve-target
  -> 服务端读取当前 Session User
  -> 重新检查目标资源授权
  -> 返回固定内部 route code
  -> 页面跳转并再次读取 owning module DTO
```

目标失效、权限撤销、Case 结束或通知已过期时，返回统一 unavailable，不泄露目标是否存在。

Notification 中的 target token 不能被当成长期权限，也不能直接拼接任意 URL。

## 7. 页面状态

| 状态 | 页面行为 |
| --- | --- |
| `loading` | 显示加载状态，不猜测未读数 |
| `success` | 显示最小文案、已读状态和允许动作 |
| `empty` | 显示“暂无待办事项” |
| `denied` | 统一拒绝，不显示其他用户的通知 |
| `unavailable` | 通知 runtime 不可用，显示服务暂不可用，不伪造已读/投递 |
| `error` | 只显示安全错误 code，不显示 Worker/provider 细节 |

Unread count 是当前 User 的投影查询，不是权限来源，也不能决定 Case 是否可以结案。

## 8. 投递失败与抑制

页面不直接处理投递重试。Notifications Worker 负责：

- 重新解析 recipient；
- 检查 User、Membership、RoleBinding 和业务关系；
- 重复 effect 返回原 DeliveryReceipt；
- 失去权限或 effect 失效写 suppressed receipt，不创建可见 Notification；
- 最终失败进入 dead-letter，由 Operations 处理。

页面只会看到成功创建且属于当前 User 的 Notification；不展示 suppressed、failed 或 dead-letter 的内部细节。

## 9. 渠道和边界

- Release 1 只有 `in_app`。
- 不发送业务 Email、SMS、WhatsApp 或 Push。
- Guardian、Student、Portal 不读取内部通知列表。
- 通知点击后仍需通过目标模块的正常授权；通知不能扩大 Case、Task、Document 或 Portal 权限。

## 10. 本模块待确认内容

请确认以下 6 点：

1. 通知入口只显示最小“有待办事项”文案，不显示客户、学校、Task 或文件信息。
2. 通知列表、未读数、已读和目标解析都使用 `/api/v1/**`，以当前 Session User 为准。
3. 点击通知目标时重新授权；Notification 不是长期权限或业务事实。
4. 已读不停止未来提醒；用户不能删除、静音或关闭必需通知。
5. suppressed/failed/dead-letter 不在用户页面展示，交给 Notifications/Operations 处理。
6. Release 1 只有站内通知，Guardian、Student 和 Portal 不接收内部 Notifications。

本纵向切片已确认。下一步进入 External Portal 工作区 API/UI 设计。
