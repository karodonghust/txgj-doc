# P0-ARCH-01 Release 1 模块边界 Handoff

状态：`approved`  
Owner：`architect`  
确认依据：项目负责人于 2026-08-25 确认通过 P0 边界  
业务依据：`BR-003`、`BR-070`、`BR-071`

返回[开发票据索引](README.md)。

## 业务结果

Backend 可以在不猜测产品范围的情况下，实现 Release 1 active registry 和架构门禁；Platform Billing、Future、旧 DEC/Subscription/Merge/reconstruction 不再作为正式运行模块或入口。

## 范围

- 冻结 active module：`shared`、`identity`、`access`、`crm`、`schools`、`cases`、`tasks`、`documents`、`notifications`、`audit`、`operations`、`external_portal`。
- 冻结 `app/**`、`components/**`、`workers/**` 为 adapter，不是业务 owner。
- 冻结跨模块只能使用根级 `public.ts/server.ts/client.ts/web.ts`。
- 冻结架构门禁：禁止深层跨模块 import、非 owner 写表、browser 导入 server、Route 直连 repository、Shared 拥有业务状态。

## 不在范围

- 不修改产品源码、测试、migration 或数据库。
- 不删除历史 Platform Billing/legacy 数据。
- 不实现任何新业务流程。

## 产物

- 本票据及批准系统设计的精确引用。
- 给 `P0-BE-00` 的 owner/范围/门禁/验收 handoff。

## 验收证据

- `changed: none`（产品仓库）。
- Architect review 确认 active/excluded 清单无缺项。
- `not_run`：typecheck、architecture tests、browser、database、cloud。

## 停止条件

- 发现 confirmed `BR-*` 与 active registry 冲突。
- 无法区分历史保留与正式运行入口。

## 回滚

仅撤回本设计票据状态；无产品或数据回滚。

## 下一门禁

项目负责人确认后，交给 Backend 执行 `P0-BE-00`，Architect 不直接修改后端源码。
