# P4-BE-06 Documents 上传、扫描与授权

状态：`ready_after_P3`  
Owner：`backend`  
依赖：P3 Local gate、P1-OPS-01 的 LocalStack/ClamAV/queue

返回[开发票据索引](README.md)。

## 业务结果

Case 文件从登记、私有上传、隔离扫描到可用下载形成闭环；未 clean 的文件绝不用于申请证据。

## 范围

- Document/DocumentVersion/ScanResult/用途关联和 active version。
- 1-10 MiB，PDF/JPEG/PNG；对象 key 只含 opaque Document/Version ID。
- 短时单用途 upload/download capability、对象收据、checksum/size/content-type 重验。
- quarantined/scanning/available/rejected/scan_failed/abandoned、重试、dead-letter 和告警。
- Primary Advisor Case 范围；Application Assignee 只访问自己 SchoolTarget 必需文件并上传 submission evidence。
- soft-delete、30 天普通恢复、legal hold 和 final purge fail-closed；业务 retention 未确认前不清对象内容。

## 不在范围

- 不接真实 S3/GuardDuty；不向 Portal/Contractor/Admin 暴露文件；不执行最终对象清理。

## 验收

- 未 clean/revoked/deleted 文件不可下载、预览或作为 Task evidence。
- 重复收据/扫描投递只产生一份事实；receipt/claim/Audit/Outbox 失败回滚一致。
- 跨 Case/SchoolTarget/organization 引用拒绝；capability 过期、撤销和重放安全失败。
- 10 MiB 本地文件 95% 在 120 秒内到扫描终态；超时可观察且不伪造 clean。
- 复用 `pnpm test:doc-01-dev-http`、`pnpm test:doc-02-dev-http`，新增 `test:p4-be-06`；typecheck/architecture 通过。

## 证据、停止与回滚

Local PostgreSQL/LocalStack/ClamAV/HTTP 必须实际运行；S3/GuardDuty/AWS `not_run`。发现公开 URL、对象 key/PII 泄露、未扫描下载或静默 provider fallback 时停止。历史 Version 不覆盖；用新 Version/审计指针/forward migration 修正。

