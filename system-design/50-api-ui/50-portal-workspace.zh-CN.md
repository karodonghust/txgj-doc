# External Portal 工作区 API 与页面

状态：`approved`  
确认依据：项目负责人于 2026-08-25 确认 Portal 仅兑换/只读/logout、raw key 保护、请求时重验、字段 allowlist、统一拒绝和与内部通知/Billing 隔离  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[API 与页面交互设计索引](README.md)。

业务依据：[External Portal 流程与状态机](../30-business-flows/60-external-portal.zh-CN.md)。  
安全依据：[数据保留、备份恢复与运行时验证](../40-permissions-security/30-data-protection-runtime.zh-CN.md)。

## 1. 先看页面目标

Portal 工作区只让被明确选择的 Guardian 查看一个 Case 的有限对客信息：

- 当前对客阶段和更新时间；
- 已批准且已确认名单中的学校进度；
- Advisor 标记为对客可见的消息和行动项。

Portal 没有内部账号、角色切换或业务写入。Guardian 不能在 Portal 确认选校、决定 offer、上传文件或回复消息。

## 2. 入口兑换流程

```text
Primary Advisor 在内部生成 Grant
  -> raw bearer key 只显示一次
  -> Guardian 打开入口 fragment
  -> 页面立即清除 fragment
  -> POST /api/v1/portal/access/redeem
  -> 服务端限流 + hash 校验 + Grant/Viewer/Case 重验
  -> 设置独立 PortalSession Cookie
  -> GET /api/v1/portal/workspace
```

raw key 不进入 query、服务器日志、Referer、localStorage、analytics、错误、Audit 或页面 HTML。Portal Session 使用独立 `__Host-`、HttpOnly、Secure、SameSite=Strict、Path=/ Cookie。

## 3. 正式 API 路径

| Method | 路径 | 类型 | 用途 |
| --- | --- | --- | --- |
| `POST` | `/api/v1/portal/access/redeem` | command | 兑换一次性 Grant key，创建独立 PortalSession |
| `GET` | `/api/v1/portal/workspace` | query | 读取当前 Session 对应 Case 的白名单 DTO |
| `POST` | `/api/v1/portal/session/logout` | command | 当前 PortalSession 退出 |

Portal 不提供以下 API：

- confirm candidate list；
- decide offer；
- update Student/Guardian/Case；
- create/complete/cancel Task；
- upload/download Document；
- reply Message 或 complete ActionItem。

## 4. 兑换接口规则

`POST /api/v1/portal/access/redeem` 只接受浏览器当前入口提交的受控数据，不接受 organization、Case、Viewer 或 Guardian ID。

服务端必须：

1. 对 raw key 做 keyed hash/fingerprint；
2. 独立限流并防止枚举；
3. 检查 Grant active、7 天期限、Viewer/GuardianRelationship、Case 和 Organization；
4. 检查当前签发人仍是 Case 的 active Primary Advisor；
5. 检查当前 Grant 的 active Session 数不超过 3；
6. 成功写 AuditEvent 后创建 PortalSession。

任何失败统一返回：

```text
PORTAL_ACCESS_INVALID
```

不区分 key 错误、Grant 到期、Viewer 失效、Case 不存在或 Session 数量超限。

## 5. Workspace 响应 DTO

成功响应只包含对客 allowlist：

```json
{
  "api_version": "v1",
  "request_id": "opaque-id",
  "data": {
    "case": {
      "customer_stage": "application_in_progress",
      "last_customer_visible_update_at": "UTC timestamp"
    },
    "schools": [],
    "messages": [],
    "action_items": []
  }
}
```

允许字段：

| 区域 | 字段 |
| --- | --- |
| Case | customer-facing stage code/label、对客更新时间 |
| 学校 | 已批准展示的学校名称、customer-facing application status |
| 消息 | Advisor 标记可见的纯文本 body、published_at |
| 行动项 | 标题、deadline、completed |

禁止字段：

- Case number；
- Student/Guardian 姓名、邮箱、电话；
- Assessment、内部备注、内部 Task/Assignee；
- 文件、文件名、下载/预览入口；
- 候选草稿、驳回理由、AuditEvent、员工信息；
- 任何未在 allowlist 中的字段。

每次 workspace 读取都从 Cases 的公开查询构建结果，不保存 PortalProjection 或复制业务状态。

## 6. Session 和页面状态

PortalSession 固定边界：

- idle timeout 15 分钟；
- absolute timeout 8 小时；
- 任一期限不得晚于 Grant expires_at；
- Grant 撤销/到期、Viewer/关系失效、Case 进入 termination_pending/closed 后下一个请求立即失效。

页面状态：

| 状态 | 页面行为 |
| --- | --- |
| `loading` | 显示安全加载状态，不猜测 Case 内容 |
| `success` | 只显示白名单 DTO |
| `empty` | 显示暂无可见进度/消息/行动项，不泄露内部数据 |
| `invalid_access` | 显示统一入口无效，不说明具体原因 |
| `expired` | 对用户仍显示统一入口无效；内部只记录受控审计原因 |
| `unavailable` | 依赖不可用时显示服务暂不可用，不伪造页面数据 |
| `error` | 不显示内部 stack、SQL、路径、provider output 或 secret |

Case 暂停时 Portal 仍可按 Grant 原期限只读查看；不会因为暂停自动延长入口。

## 7. 页面安全边界

- 所有响应 `Cache-Control: no-store`，敏感页面禁止被搜索引擎索引或第三方 frame 嵌入。
- 使用严格 CSP、`Referrer-Policy: no-referrer`，不加载第三方追踪脚本。
- 不接受任意 redirect/return URL；只跳转固定 Portal route code。
- Portal 没有内部 Notification 列表，也不能通过通知 target 进入 ERP。
- Portal 不返回文件 capability、Object key、Token 或内部错误。
- Frontend 隐藏写按钮不是控制；服务端没有对应写命令。

## 8. 审计和失效

必须审计：

- access key 成功/失败兑换和限流；
- PortalViewer 创建/失效；
- Grant 创建、重新生成、撤销和到期；
- Session 创建、logout 和失效；
- workspace 读取的允许/拒绝结果。

Audit 只保存 Portal actor kind、organization、Case/Viewer/Grant opaque ID、action、outcome、request ID、reason code 和版本，不保存 raw key、Session secret、Guardian 资料、消息正文或学校名称。

## 9. 本模块待确认内容

请确认以下 7 点：

1. Portal 只提供兑换、白名单 workspace 查询和 logout 三类入口。
2. raw key 只显示一次，不进 query、日志、Referer、localStorage 或 analytics。
3. 每次兑换和 workspace 读取都重新检查 Grant、Viewer、关系、Case、签发人和期限。
4. Workspace 不展示 Case number、姓名、联系方式、Assessment、文件或内部 Task 信息。
5. Portal 没有确认、决定、回复、上传、下载或其他业务写 API。
6. Grant/Session/关系/Case 失效后下一个请求统一返回 `PORTAL_ACCESS_INVALID` 或 unavailable，不泄露具体原因。
7. Portal 只读、单 Case，不接收内部 Notifications，也不依赖 Platform Billing。

本纵向切片已确认。API 与页面交互设计阶段全部完成，下一步进入非功能设计、评审与开发拆分。
