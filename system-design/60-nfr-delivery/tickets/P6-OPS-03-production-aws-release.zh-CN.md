# P6-OPS-03 香港生产适配、恢复与发布验收

状态：`deferred_no_environment`  
Owner：`platform-operations`  
依赖：P0-P5 Local Release 1 已全部 accepted、P6-BE-09 source gate 通过；真实香港环境、账号、预算和单独执行批准

返回[开发票据索引](README.md)。

## 业务结果

只有真实香港生产 adapter、迁移、备份恢复、安全和运行证据全部通过后，Production Release 1 才可能 go。

## 当前边界

目前没有环境，本票据不执行。Backend 可保留 production port/config shape，Local 联调可使用显式 `cloud-synthetic` fake adapter；结果只能标记 `simulated/not_run`，不能写成 AWS 已接通。

## 获批后的范围

- 连接 Backend 已批准的 `production-aws` adapters，配置 Cognito、RDS PostgreSQL、S3、Queue、KMS、日志、告警和备份环境。
- 强制 `ap-east-1`，拒绝 local/mock/preview/cloud-synthetic/legacy/其他区域。
- 生产 migration dry-run/执行、兼容前一应用镜像、forward corrective migration 和 deployment rollback。
- 隔离数据库及 Document 恢复演练，验证 RPO/RTO、RLS、Audit、版本、对象链接和租户边界。
- 性能/容量、安全/隐私/dead-letter/撤销/告警验收与同一 evidence manifest 的人工 go/no-go。

## 不在范围

- 本票据不修改 application、adapter 或 migration 源码，也不自行申请预算、创建账号、写入 secret、运行 Terraform、迁移数据、部署或切换 endpoint；每项执行都是另一个精确批准门。

## Production 通过门槛

- [交付、发布与回滚门禁](../30-release-readiness.zh-CN.md)第 5 节全部 `passed`。
- 数据库 RPO <= 5 分钟/RTO <= 4 小时；Document RPO <= 24 小时/RTO <= 8 小时。
- Platform Operations、Architect、Founder 和项目负责人记录同一 manifest 的 go；任一 `failed/not_run` 即 `no_go`。

## 停止与回滚

区域/账号/目标不精确、secret/PII 暴露、migration checksum drift、Audit/RLS/对象链接不一致或缺少人工批准即停止。应用使用已验证兼容前一镜像；migration 只 forward correction；恢复只进隔离目标；云资源清理需保存证据后的单独批准。
