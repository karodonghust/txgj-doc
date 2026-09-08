# Release 1 交付、发布与回滚门禁

状态：`approved`  
确认依据：项目负责人于 2026-08-26 指示完成系统设计第 7 阶段  
设计日期：2026-08-26  
业务基线：`BR-BASELINE-20260825-v32`

返回[非功能设计、评审与开发拆分索引](README.md)。

## 1. 两种“完成”必须分开

| 结果 | 完成条件 |
| --- | --- |
| Local Release 1 | P0-P5 的相关 Backend/Frontend/Platform Operations 票据完成，自测与现有独立测试工作流的 Local Dev 验收通过，Architect 接受证据 |
| Production Release 1 | Local Release 1 已通过，且 P6 香港生产 adapter、迁移、备份恢复、部署、安全和人工 go/no-go 全部通过 |

当前没有真实云环境，因此 P6 可以保持 `deferred/not_run`，不阻塞本地开发与演示；但任何人不得把 Local、fake adapter 或 Vercel check 写成 Production 已就绪。

## 2. 每批固定门禁

```text
批准的设计/票据
  -> Owner 盘点现状与差异
  -> 精确实现 + focused self-test
  -> Architect 契约/集成 readiness review
  -> 现有独立测试工作流执行 Local Dev 验收
  -> Architect 最终审查
  -> 项目负责人决定 commit/push/PR/merge
  -> 如需部署，另开 Platform Operations 授权门
```

阶段依赖为 P0 -> P1 -> P2 -> P3 -> P4 -> P5；不能用后续页面或 mock 掩盖前置运行时未接通。P6 只在真实香港生产环境、账号、预算和执行授权具备后开始。

## 3. 票据完成定义

每张票据只有同时满足以下条件才可标记 `accepted`：

- 只修改票据授权范围，现有合规实现优先复用；
- 业务、权限、租户、幂等、Audit/Outbox 和失败路径符合批准设计；
- focused static/test/runtime evidence 实际运行并记录；
- 独立测试需要的场景、环境和命令完整，结果为 `passed/failed/not_run`；
- 未运行的 browser、database、Preview、AWS 或 migration 明确写成 `not_run`；
- 停止条件与回滚路径真实可执行，不把历史输出当当前证据；
- Architect 接受，且外部 Git/部署动作取得项目负责人单独批准。

## 4. Local Release 1 验收清单

- Node 22/pnpm 版本、architecture、typecheck 和 API envelope 门禁通过；
- PostgreSQL 17 从批准 migration/baseline 重建，RLS、权限、租户隔离和连接归还清理通过；
- Identity/Access、CRM/Case、名单两层确认、逐校申请、Tasks、Documents、Notifications、Portal 纵向路径通过；
- loading、empty、denied、stale、unavailable、failure、responsive 和 accessibility 场景通过；
- 重试、冲突、撤销、审计失败、重复投递、扫描失败和依赖不可用均 fail closed；
- 第 10 号非功能基线中的 Local 性能目标已测量；未达标项有 owner 和阻塞结论；
- 合成数据、日志、证据中不含真实 PII、secret、连接串、对象 key 或文件内容。

## 5. Production go/no-go

Production 只有全部为 `passed` 才能 go：

- `production-aws` 拒绝 local/mock/preview/cloud-synthetic/legacy adapter；
- Cognito、RDS、S3、Queue、KMS、日志和备份均提供 `ap-east-1` 运行证据；
- 生产 migration dry-run、执行计划、兼容旧应用镜像和 forward corrective migration 路径已评审；
- 隔离恢复演练满足数据库与 Document 的 RPO/RTO，并通过 RLS/Audit/版本/租户校验；
- 安全、隐私、性能、告警、dead-letter、审计和访问撤销验收通过；
- Platform Operations、Architect、Founder 和项目负责人对同一 evidence manifest 做出明确决定。

任一项为 `failed` 或 `not_run`，Production 结论只能是 `no_go`。

## 6. 回滚规则

| 类型 | 回滚方式 |
| --- | --- |
| 未合并源码 | 只撤回票据范围内变更；保留用户无关改动 |
| 已合并应用 | 使用已验证的兼容前一镜像/提交；部署切换需单独批准 |
| 已执行 migration | 不 down、不改历史；新增 forward corrective migration |
| 数据恢复 | 恢复到新的隔离实例，校验通过并再次批准后才允许切换 |
| Document 对象 | 不覆盖历史 Version；用新审计指针或新 Version 修正 |
| Projection | 可停止消费者并重建；不得回写权威业务表 |
| 云资源/隔离环境 | 保存证据后，以精确目标和单独批准执行清理 |

出现跨租户、PII/secret 泄露、Audit 不连续、migration checksum 不符、对象版本不符或区域错误时立即停止；保持入口 fail closed，不边运行边手工修数据。

## 7. 最小证据清单

每次验收记录：ticket、commit、schema/policy version、环境、时间、owner、命令/场景、passed/failed/not_run、脱敏摘要、已知风险、回滚目标和 Architect 决定。Production 额外记录部署 ID、区域、备份/恢复 run ID 和人工 go/no-go。

本文件是交付标准，不是运行收据，也不授权 commit、push、PR、migration、部署、云资源或生产数据操作。
