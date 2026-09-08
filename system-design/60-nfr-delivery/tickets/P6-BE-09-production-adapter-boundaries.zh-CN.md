# P6-BE-09 Production AWS Adapter 边界

状态：`ready_after_P5_local_no_cloud_claim`  
Owner：`backend`  
依赖：P0-P5 Local Release 1 已 accepted；P6 Platform 环境仍可暂缓

返回[开发票据索引](README.md)。

## 业务结果

产品源码具备明确的 `production-aws` adapter 接口与 fail-closed 组合边界，同时在没有云环境时绝不伪造 AWS 已接通。

## 范围

- 由 Backend 实现 Cognito/RDS/S3/Queue/KMS/telemetry 所需 server adapter 与配置 schema；Platform Operations 不修改这些源码。
- `production-aws` 启动时强制批准区域和 adapter kind，拒绝 local/mock/preview/cloud-synthetic/legacy。
- 显式 `cloud-synthetic` fake adapter 只用于 Local UI/API 联调，返回结果和 evidence 必须为 `simulated`。
- 对 provider 错误做稳定 code/allowlist 映射，不记录 connection string、bucket、queue、object key、credential 或原始响应。
- 提供后续 P6-OPS-03 可执行的 readiness、health、migration 和 rollback 输入契约。

## 不在范围

- 不创建/修改 AWS 资源、IAM、secret、DNS、数据库或对象；不运行 Terraform、migration、部署或恢复；不声称区域和权限已验证。

## 验收

- source/static/contract tests 证明 runtime mode 与 adapter allowlist fail closed。
- fake adapter 只能在非 production 模式加载，所有结果标记 `simulated`。
- 缺配置、区域不符和 provider unavailable 返回稳定错误，不静默回退。
- focused tests、`pnpm test:architecture`、`pnpm typecheck` 和新增 `test:p6-be-09` 通过；真实 AWS 全部 `not_run`。

## 停止与回滚

需要真实 credential 才能完成源码测试、出现静默 fallback、provider detail 泄露或越过 Backend 所有权执行环境操作时停止。回滚只撤回 adapter/composition 源码；任何云资源都不在本票据授权范围。

