# P0-ARCH-02 API 与入口契约 Handoff

状态：`approved`  
Owner：`architect`  
确认依据：项目负责人于 2026-08-25 确认通过 P0 边界  
业务依据：`BR-012`、`BR-070`、`BR-071`

返回[开发票据索引](README.md)。

## 业务结果

Backend 与 Frontend 使用同一套 `/api/v1/**`、请求上下文、错误 envelope 和 DTO allowlist，不再由页面、Route 或模块各自发明响应格式。

## 范围

- 正式业务 API 仅 `/api/v1/**`。
- 成功 envelope：`api_version + request_id + data`；失败 envelope：`api_version + error`。
- 私有 API 默认 `no-store`、`X-Request-Id`。
- 写命令使用 `Idempotency-Key` 和 expected version。
- `public.ts/server.ts/client.ts` 与 Identity `web.ts` 的允许依赖。
- 稳定错误、details allowlist、未知错误脱敏和 client runtime decoder。

## 不在范围

- 不修改 Route、client、component 或测试源码。
- 不决定后续每个 endpoint 的全部业务字段。
- 不接真实云 provider。

## 产物

- Backend/Frontend 共同 handoff。
- `P0-BE-00` 和后续 `FE-01` 的契约引用。

## 验收证据

- 设计可明确回答：身份从何解析、错误如何映射、DTO 如何 allowlist、client 如何拒绝错误 shape。
- `changed: none`（产品仓库）。
- `not_run`：API contract tests、typecheck、browser、database、cloud。

## 停止条件

- 业务 endpoint 需要传入客户端 organization/role 才能工作。
- 某个正式入口无法迁移到 `/api/v1/**` 且缺少明确退出策略。

## 回滚

仅撤回本设计票据状态；无产品或数据回滚。

## 下一门禁

项目负责人确认后，Backend 在 `P0-BE-00/BE-01` 实现，Frontend 在后续 `FE-01` 消费。
