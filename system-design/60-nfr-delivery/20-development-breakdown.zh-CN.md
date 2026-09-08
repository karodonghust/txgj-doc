# 开发拆分与交付门禁

状态：`approved`  
确认依据：项目负责人于 2026-08-25 同意 P0–P6 拆分、责任方边界、票据验收要求，并确认云环境暂缓、仅允许显式 `cloud-synthetic` fake adapter 做本地联调  
设计日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[非功能设计、评审与开发拆分索引](README.md)。

依据：[非功能基线与验收证据](10-nfr-baseline.zh-CN.md)、[系统设计索引](../README.md)。

## 1. 拆分原则

开发不按“先把所有表做完”拆分，而按可验证的纵向能力拆分：

```text
契约/边界冻结
  -> Shared + Identity/Access 基础能力
  -> CRM/Case 主链路
  -> Candidate List / SchoolTarget / Tasks
  -> Documents / Notifications
  -> External Portal
  -> 安全、运行时和生产证据
```

每张票据必须明确：

- owner 和不在范围内的内容；
- 依赖和允许的模块入口；
- 修改文件/数据库范围；
- 验收场景和命令；
- expected evidence、`not_run`、停止条件和回滚方式；
- 是否需要独立测试会话验收。

## 2. 交付阶段总览

| 阶段 | 目标 | 前置条件 | 通过门槛 |
| --- | --- | --- | --- |
| P0 契约基线 | 清理旧 registry/DEC/legacy 边界，固定公开入口 | 本系统设计全部 approved | architecture/static checks 通过 |
| P1 本地平台基础 | PostgreSQL、RLS、迁移、幂等、Audit/Outbox、LocalStack/ClamAV 组合 | P0 | 本地 readiness 和失败注入通过 |
| P2 内部 Case 主链路 | Identity/Access、CRM、Cases、Assessment、名单两层确认 | P1 | Case vertical slice 本地闭环通过 |
| P3 逐校申请与 Task | SchoolTarget、申请/面试 Task、完成证据和取消/结案 | P2 | Task/Cases 事件闭环通过 |
| P4 文件与站内通知 | Documents 扫描/下载授权、Notifications effect/scheduler | P3 | 文件与通知本地闭环通过 |
| P5 Guardian Portal | Grant/Session/allowlist 只读工作区 | P4 | Portal 撤销/过期/枚举防护通过 |
| P6 生产适配与独立验收 | AWS 香港 adapter、备份恢复、部署证据 | 当前环境具备后再开始；目前暂缓 | 生产 gate 明确 passed/failed/not_run |

阶段不能跳过前置门槛；Local 通过不自动推进到 Preview 或 Production。

## 3. Architect/契约票据

### ARCH-01：Release 1 模块边界与 registry

- 范围：冻结 11 个批准模块、排除项、公开入口和架构门禁的精确目标，并产出 Backend handoff。
- 不做：Architect 不修改 registry、测试或其他产品源码；不修改历史 migration。
- 验收：目标 registry、排除项、门禁和 Backend 验收命令无歧义。
- 证据：批准设计引用、handoff 文件、产品源码 `changed: none`。

### ARCH-02：公开入口和错误契约

- 范围：冻结 `public.ts/server.ts/client.ts` 边界、`/api/v1/**` envelope、稳定错误和 DTO allowlist，并产出 Backend/Frontend handoff。
- 不做：Architect 不修改 Route Handler、client 或测试源码；不冻结每个业务 endpoint 的最终业务字段。
- 验收：入口、错误、DTO 和客户端 decoder 的实现要求无歧义。

### ARCH-03：旧规则隔离与迁移输入

- 范围：记录旧 `actor_user_id`、DEC、portal local idempotency、billing/legacy route 的停用/保留/追加迁移输入。
- 不做：不在本票据执行 destructive cleanup。
- 验收：每项差异有 owner、corrective migration 方向和回滚说明。

## 4. Backend 票据

### BE-00：Release 1 registry、入口与架构门禁实现

- 依赖：ARCH-01、ARCH-02、ARCH-03。
- 范围：实现 active registry、模块根入口和架构门禁；隔离 Platform Billing/Future/旧 DEC/legacy 正式入口。
- 验收：`test:architecture`、`typecheck` 和相关 contract tests 通过；旧入口不在 Release 1 runtime/navigation/API。
- 不做：不修改历史 migration，不实现后续业务状态机。

### BE-01：Shared 幂等与事务基础

- 依赖：ARCH-01、ARCH-02。
- 范围：actor kind + opaque actor scope、canonical request hash、in-progress/replay/conflict、transaction runner。
- 验收：相同重试不重复副作用；不同 payload 稳定冲突；RLS context 归还连接前清理。

### BE-02：Identity/Access 认证与授权

- 依赖：BE-01、ARCH-02。
- 范围：Session/Principal 分离、四角色兼容性、EmployeeProfile、Capability 包含关系、ScopeGrant、Contractor 互斥。
- 验收：Admin 单独拒绝客户数据；Contractor 只能当前面试 Task；角色/成员撤销即时失效。

### BE-03：CRM 与 Case 建案/Assessment

- 依赖：BE-02。
- 范围：Student/Guardian/Relationship、ReferralSource、ServiceCase、Assessment blocker、Primary Advisor assignment。
- 验收：建案后进入 background；软删除/永久保留规则、RLS、审计和并发冲突通过。

### BE-04：Cases 选校与逐校状态机

- 依赖：BE-03、Schools contract。
- 范围：候选名单版本、Founder 审核、Guardian 代录、SchoolTarget 状态、人工结案双分支。
- 验收：全部学校拒绝不自动结案；同版本两层确认；状态非法转换稳定拒绝。

### BE-05：Tasks 事件闭环

- 依赖：BE-04。
- 范围：申请/面试自动 Task、Assignment、完成回执、重派、取消、逾期事实、Cases 重验。
- 验收：preparing 只产生一条准备并提交 Task；面试 Task 不推进学校结果；重复事件幂等。

### BE-06：Documents 上传/扫描/授权

- 依赖：BE-05、平台本地文件依赖。
- 范围：Document/Version/ScanResult、隔离/clean/rejected、Application Assignee 学校范围、短时 capability、30 天恢复。
- 验收：未 clean 不可下载；文件引用不跨 Case/SchoolTarget；扫描失败有界重试和告警。

### BE-07：Notifications effect 与 scheduler

- 依赖：BE-05、Audit/Outbox。
- 范围：九类 effect、接收人实时解析、3 天/1 天/逾期提醒、去重、suppressed/dead-letter。
- 验收：同 recipient/effect/source/date 只一条；暂停不停止逾期提醒；不产生外部 Email。

### BE-08：External Portal

- 依赖：BE-03、BE-04、BE-06、BE-07。
- 范围：Viewer/Grant/Session、固定 7 天、3 Session、15 分钟 idle/8 小时 absolute、allowlist DTO。
- 验收：撤销/改派/Case 终止即时失效；无 Case number/文件/业务写入；统一防枚举错误。

### BE-09：Production AWS adapter 边界

- 依赖：P0-P5 Local Release 1 accepted；可先完成 source/static gate，不需要真实 credential。
- 范围：生产 server adapter、配置 schema、区域/adapter kind fail-closed、provider 错误脱敏和 Platform Operations handoff。
- 验收：`production-aws` 拒绝 local/mock/preview/cloud-synthetic/legacy；fake 只能标记 `simulated`；真实 AWS 保持 `not_run`。
- 不做：Backend 不创建云资源、不写 secret、不执行 migration/部署/恢复。

## 5. Frontend 票据

### FE-01：Shared typed client 与 workspace shell

- 依赖：ARCH-02、BE-01/02。
- 范围：统一 envelope decoder、loading/empty/denied/stale/unavailable/error 状态、通知入口壳。
- 验收：页面不直接手写 fetch shape，不把客户端缓存当授权来源。

### FE-02：CRM、建案与 Cases 案件工作台

- 依赖：BE-03/04、FE-01。
- 范围：Student/Guardian/ReferralSource、建案向导、Case 摘要、Assessment、候选名单、SchoolTarget、操作确认和版本冲突提示。
- 验收：Founder/Advisor/Admin/Collaborator 显示边界正确；结案双分支明确。

### FE-03：Task 工作台与 Contractor workspace

- 依赖：BE-05、FE-01。
- 范围：Task 列表、Assignment 操作、三类完成表单、Contractor 脱敏工作区。
- 验收：allowed_actions 驱动按钮；Contractor 看不到完整 Case；完成资料校验错误可理解。

### FE-04：Documents 文件工作台

- 依赖：BE-06、FE-01。
- 范围：上传进度、扫描状态、下载 capability、提交凭证引用、删除/恢复提示。
- 验收：quarantined/scanning/rejected/unavailable 不显示成功下载；不出现对象 key。

### FE-05：Notifications 入口

- 依赖：BE-07、FE-01。
- 范围：列表、未读数、已读、目标重新授权和统一文案。
- 验收：不显示客户/学校/Task/文件信息；suppressed/failed 不出现在用户页面。

### FE-06：External Portal 工作区

- 依赖：BE-08、FE-01。
- 范围：fragment 清理、兑换、独立 Session、allowlist 内容、统一 invalid access 页面。
- 验收：无业务写入口；Case number/文件/联系方式不出现；过期/撤销不枚举。

## 6. Platform Operations 票据

### OPS-01：本地组合与 readiness

- 范围：Node 22/pnpm、PostgreSQL、RLS、LocalStack、ClamAV、queue、synthetic identity、显式 cloud-synthetic fake adapter 和 composition root。
- 验收：local-synthetic 完整 readiness；缺依赖 fail closed；fake adapter 结果明确标记 `simulated`；不加载 production secrets。

### OPS-02：测试数据库与失败注入

- 范围：隔离测试数据库、迁移顺序、并发冲突、重复事件、扫描失败、审计失败和权限撤销场景。
- 验收：结果标记 passed/failed/not_run；不把历史 Compose/Preview 输出当当前证据。

### OPS-03：Production AWS 环境、恢复与区域证据（暂缓）

- 范围：连接 Backend 已批准 adapters，执行香港 AWS/Cognito/RDS/S3/Queue/logging 环境、IAM、备份和恢复验证。
- 当前状态：没有真实云环境时不执行；可以先保留接口和配置结构，但不得声称已接通。
- 验收：production-aws 启动拒绝 local/mock/preview/cloud-synthetic/legacy；真实环境具备后再记录区域、权限、备份和恢复证据。
- 前置：P0–P5 本地验收通过；本票据不代表已部署或已通过。

## 7. 独立验收与交付门禁

独立测试继续使用现有独立测试工作流，不新增业务角色或永久测试实体。每个纵向票据在交给独立测试前必须提供：

- 已实现范围和未实现范围；
- 本地命令/场景和实际输出摘要；
- passed/failed/not_run；
- 脱敏失败证据、停止条件和回滚说明；
- Preview/Production 是否未运行。

开发者自测不等于独立验收；独立测试默认只读，不替代 Architect 的契约审查。

## 8. 确认结果

项目负责人已确认以下 6 点：

1. 采用 P0–P6 阶段顺序，不跳过本地基础和纵向闭环直接进入 Production。
2. Backend、Frontend、Platform Operations、Architect 和独立测试按上述边界协作。
3. Backend 先完成 Shared/Identity/Access，再推进 Case → Tasks → Documents/Notifications → Portal。
4. Frontend 按 Cases、Tasks、Documents、Notifications、Portal 纵向切片交付。
5. 每张票据必须记录范围、依赖、验收证据、`not_run`、停止条件和回滚方式。
6. 当前没有真实云环境时，允许使用显式 `cloud-synthetic` fake adapter 做本地页面/接口联调，但结果必须标记 `simulated/not_run`；`production-aws` 必须拒绝该 adapter，真实云验证暂缓。

本拆分及 P0-P6 可分派票据已完成。票据设计完成不等于实施完成；当前实施事实仍为 P0 已合并、P1-P5 待实施、P6 无环境暂缓。
