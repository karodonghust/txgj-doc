# P0-ARCH-03 旧规则与迁移输入清单

状态：`approved`  
Owner：`architect`  
确认依据：项目负责人于 2026-08-25 确认通过 P0 边界  
业务依据：`BR-003`、`BR-010`、`BR-029`、`BR-030`、`BR-039`、`BR-060`、`BR-070`、`BR-071`

返回[开发票据索引](README.md)。

## 业务结果

后续 Backend 可以使用追加式 corrective migration 和受控入口迁移纠正旧实现，不修改历史 migration，不误删历史数据。

## 范围

清单至少覆盖：

- `shared_idempotency_records.actor_user_id` 向 `actor_kind + actor_opaque_id` 的目标迁移。
- 旧 Portal 本地 security/idempotency 结构停止新写并保留历史。
- 旧 Data Reviewer、DEC、Subscription、Platform Billing、Merge/reconstruction 正式入口隔离。
- 产品仓库本地 `AGENT.md` 和 `PROJECT_PLAN.local.md` 曾将案件计数、合同参考值或 Platform Billing 写入 Release 1；已于 2026-08-26 按 confirmed `BR-003` 同步为整体排除，旧实现只保留历史追溯。
- Student/Guardian purge 路径停止新写；ServiceCase 永久保留。
- Admin 旧客户数据 capability、单一 role Session 和 Contractor 全局 Task capability 纠正。
- legacy API、preview/mock route 的调用方和退出门禁。

## 不在范围

- 不编写或执行 migration。
- 不删除表、文件、route 或历史数据。
- 不连接 Local、Test、Preview、Production 数据库。

## 产物

每项差异必须记录：现状证据、目标 owner、corrective migration/入口动作、历史保留方式、依赖、风险和回滚。

## 验收证据

- 清单与当前代码/migration 的路径可追踪。
- 每项明确分类为 `retain`、`correct`、`stop_new_writes` 或 `isolate_from_release1`。
- 工程指引同步 `changed`：产品仓库本地忽略文件 `AGENT.md`、`PROJECT_PLAN.local.md`；未修改产品源码、测试、migration 或数据库。
- `not_run`：migration、database、browser、cloud。

## 停止条件

- 无法证明旧字段/表是否已有数据。
- 纠正方案要求重写历史 migration 或物理删除数据。
- 发现与 confirmed `BR-*` 冲突。

## 回滚

仅撤回清单状态；无产品或数据回滚。

## 下一门禁

项目负责人确认清单后，Backend 在独立 migration 票据逐项实现；执行数据库仍需 Platform Operations 单独批准。
