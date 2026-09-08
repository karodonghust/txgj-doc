# P5-BE-08 Guardian External Portal

状态：`ready_after_P4`  
Owner：`backend`  
依赖：P4 Local gate 通过

返回[开发票据索引](README.md)。

## 业务结果

当前 Primary Advisor 可为 Guardian 创建一个单 Case、固定 7 天、可撤销的只读入口；家长无需内部账号即可查看白名单进度。

## 范围

- PortalViewer/PortalGrant/PortalSession、opaque key + raw secret 只显示一次、fragment 清理后兑换。
- 仅当前 Primary Advisor 签发/重签；Primary Advisor 或 Founder 撤销；固定 7 天不可延期。
- 每 Grant 最多 3 Session，idle 15 分钟、absolute 8 小时，独立安全 Cookie。
- 每次请求重验 Grant、Viewer/GuardianRelationship、Case、Organization 和 allowlist。
- 只返回对客阶段、学校进度、已发布消息/行动项；统一 `PORTAL_ACCESS_INVALID` 防枚举。
- 限流、Audit、幂等、撤销/改派/Case termination/closed 即时失效。

## 不在范围

- 不创建 Guardian 内部 User；不提供业务写入、文件、Case number、联系方式、内部备注或 Billing；不发外部业务 Email。

## 验收

- raw secret 不进入 URL query、日志、Audit、analytics、数据库明文或普通 DTO。
- 第 4 个并发 Session 拒绝；idle/absolute/Grant 到期、撤销和重签原子失效。
- 对不存在/过期/撤销/未授权目标的外部响应不可区分；内部 Audit 保留稳定原因。
- 跨 Case/organization、改派、关系失效和状态变更在下一请求拒绝。
- 新增 `test:p5-be-08`，复用 Portal contract/persistence tests；Local PostgreSQL/HTTP、typecheck/architecture 通过。

## 证据、停止与回滚

Local database/HTTP 为必需证据；真实外部传递、Preview、AWS `not_run`。出现 raw secret/PII 泄露、文件可见、业务写入口或旧 Session 继续有效时停止。撤销新 Grant/Session 即可关闭入口，历史不删除。

