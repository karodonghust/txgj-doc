# P1-OPS-01 Local 组合与 Readiness 验收

状态：`ready_after_P1-BE-02_self_test`  
Owner：`platform-operations`  
依赖：P1-BE-01、P1-BE-02 自测完成

返回[开发票据索引](README.md)。

## 业务结果

开发和验收使用同一套明确的 Local Dev：PostgreSQL 17、LocalStack、ClamAV、queue 和 synthetic identity，依赖缺失时安全停止。

## 范围

- 以只含合成数据的 Colima/Docker Compose 启动本地依赖并记录当前证据。
- 执行批准 migration/baseline、seed、readiness 和运行模式检查。
- 验证 `local-synthetic` 不读取 production secret；显式 `cloud-synthetic` 只返回 `simulated`。
- 验证缺失 PostgreSQL/S3/SQS/ClamAV/identity provider 时相关入口 fail closed。

## 所有权边界

Platform Operations 不修改 application、migration 或测试源码；源码缺陷退回 Backend。数据库执行、容器和环境证据不由 Backend 代签。

## 验收

- `pnpm check:toolchain`、`pnpm local:up`、`pnpm local:ps`、migration dry-run/apply、synthetic seed、`pnpm test:local-foundation` 按批准环境成功。
- readiness 仅返回受控 code，不泄露连接串、hostname、bucket、queue、路径或 provider output。
- 重启/重建后可从空环境确定性恢复；Production/Preview/AWS 均保持 `not_run`。

## 停止与回滚

发现非合成数据、远程数据库、生产 secret、adapter 混用或 migration checksum drift 立即停止。回滚只关闭精确 Compose 项目；数据卷清理属于单独破坏性授权。

