# P1-BE-01 Shared 幂等与事务基础

状态：`ready`  
Owner：`backend`  
依赖：P0 已合并、ARCH-02 已批准

返回[开发票据索引](README.md)。

## 业务结果

所有可重试写入共享同一套 actor-scoped 幂等、事务和错误语义；重复请求不再产生第二份业务事实。

## 范围

- 先盘点现有 Shared 实现，仅修正相对批准设计的差异。
- 实现/收敛 actor kind + opaque actor scope、canonical request hash、in-progress/replay/conflict。
- 提供 owning module 可用的 transaction runner、RLS request context 和提交前清理。
- 业务事实、Audit、Outbox 和 Idempotency result 在同一 PostgreSQL 事务提交。
- 正式 API 使用统一 envelope、`X-Request-Id`、`no-store` 和稳定错误。

## 不在范围

- 不实现 CRM/Case 等业务状态机；不修改历史 migration；不接云数据库。

## 验收

- 相同 actor/scope/key/hash 返回首次结果；不同 hash 返回稳定冲突；in-progress 不并发执行。
- 事务任一步失败无部分业务/Audit/Outbox/Idempotency 写入。
- 两个 PostgreSQL 连接验证 organization 隔离，连接归还后无 RLS context 泄露。
- `pnpm check:toolchain`、`pnpm test:architecture`、`pnpm typecheck` 及新增 `test:p1-be-01` 通过。

## 证据与停止条件

记录 source/static/PostgreSQL Local Dev 的 `passed/failed/not_run`；browser、Preview、AWS 默认 `not_run`。发现需改写历史 migration、跨模块私表写入或事务无法原子提交时停止。

## 回滚

未执行 migration 时撤回本票据变更；已执行 migration 只允许 forward corrective migration。关闭新入口前保留旧数据与脱敏失败证据。

