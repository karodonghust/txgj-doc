# 领域模型与数据设计索引

状态：`approved`  
开始日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[系统设计索引](../README.md)。

## 1. 这一阶段解决什么

本阶段把已经确认的模块职责转换为可实现的数据结构，逐个回答：

1. 模块内有哪些领域对象和值对象。
2. 对象之间是一对一、一对多还是多对多。
3. 哪些字段和关系必须由数据库保证。
4. 哪些历史只能追加，哪些状态可以更新。
5. 当前表结构与目标模型有什么差异。
6. 后续需要哪些纠正 migration，但本阶段不执行 migration。

领域模型回答“业务事实如何组成”；数据设计回答“这些事实如何可靠保存”。

## 2. 每个模块的固定产物

每份模块文档都包含：

- 领域对象、值对象及唯一 owner。
- ER 关系图和基数。
- 对象 identity、生命周期和不可变量。
- 目标逻辑表的全部字段、字段含义、必填性、外键和唯一约束。
- organization 边界、RLS 与授权所需引用。
- PII/secret/文件引用的数据分类。
- 乐观锁、审计、幂等和历史保留方式。
- 查询所需索引方向；不提前做无依据的性能优化。
- 当前 schema 的保留、纠正、停用和新增清单。
- 验收标准与仍待后续状态机/API 阶段冻结的内容。

## 3. 统一设计规则

所有模块共同遵守：

1. 每项业务事实只有一个模块 owner；其他模块只保存必要 opaque reference。
2. 主键使用服务端生成的 opaque UUID；业务编号不能充当数据库主键或授权凭证。
3. 租户业务表显式保存 `organization_id`，并以 PostgreSQL RLS/FORCE RLS 作为纵深防御。
4. 可并发修改的聚合保存 `record_version`，重要写入使用 expected version，禁止静默覆盖。
5. 时间保存 UTC `timestamptz`；年龄等派生值不持久化，按权威日期计算。
6. 核心业务事实使用明确列、外键和约束；不使用通用 EAV 或无 schema 的 JSONB 代替领域模型。
7. JSON/JSONB 只用于版本化 manifest、不可变 snapshot 或严格 schema 的技术 payload。
8. Student、Guardian 只软删除且 Release 1 不可恢复；ServiceCase 永久保留，只能结案。
9. 审批、确认、分派、关系和重要状态变化保留版本历史，不能用覆盖当前行丢失证据。
10. AuditEvent 不是业务事实副本；业务理由和确认内容仍保存在 owning module。
11. Projection、缓存、搜索和统计可重建，不能成为授权或业务权威。
12. 历史 migration 不修改；所有纠正使用新的 append-only migration。

## 4. 约束分工

| 规则类型 | 主要位置 |
| --- | --- |
| 字段类型、非空、枚举、唯一性、外键、版本递增 | PostgreSQL constraint/trigger + domain |
| 当前角色、Case 关系、审批资格、流程前置条件 | owning module application/domain 每次重验 |
| 租户隔离 | application authorization + PostgreSQL RLS/FORCE RLS |
| 重复请求 | Shared IdempotencyRecord + owning module transaction |
| 必需证据 | owning module 业务历史 + AuditEvent |
| 跨模块副作用 | committed OutboxMessage + 消费者幂等 |
| 页面显示和交互提示 | 后续 API/UI 设计；不能代替服务端或数据库约束 |

数据库约束负责“不可能写出非法结构”；领域服务负责“当前业务条件下是否允许这次操作”。两者不能互相替代。

## 5. 跨模块关系规则

- 跨模块只引用对方公开的 opaque ID，不复制姓名、联系方式、状态正文或权限结果。
- 物理外键仅用于不会破坏 owner 边界、历史保留和独立演进的稳定 identity 关系；每条在模块文档中单独决定。
- 不能依赖跨模块外键级联删除；Release 1 业务数据默认禁止物理级联删除。
- 不能通过数据库 join 或 projection 绕过 owning module 的当前授权检查。
- 同步写入不能直接修改另一个业务模块的表；需要副作用时使用公开 command 或 outbox。

## 6. 当前数据库的处理方式

现有 migration 和表只作为“当前实现证据”，不自动成为目标设计：

| 分类 | 处理 |
| --- | --- |
| 与确认模型一致 | 保留，并在目标文档中明确 |
| 字段/约束不足 | 设计新的 corrective migration |
| 属于旧业务但有历史数据 | 停止新入口和新写入，保留历史；不重写旧 migration |
| Platform Billing、Future 等排除项 | 从 Release 1 runtime/权限/导航隔离；是否日后物理清理由单独数据治理决定 |
| preview/mock/test 数据 | 不作为生产业务权威或 migration 依据 |

本阶段只输出目标模型和差异，不连接或修改 Local、Neon、Vercel、AWS 数据库。

## 7. 设计顺序

| 顺序 | 模块 | 状态 |
| ---: | --- | --- |
| 1 | [Identity](10-identity.zh-CN.md) | `approved` |
| 2 | [Access](20-access.zh-CN.md) | `approved` |
| 3 | [CRM](30-crm.zh-CN.md) | `approved` |
| 4 | [Schools](40-schools.zh-CN.md) | `approved` |
| 5 | [Cases](50-cases.zh-CN.md) | `approved` |
| 6 | [Tasks](60-tasks.zh-CN.md) | `approved` |
| 7 | [Documents](70-documents.zh-CN.md) | `approved` |
| 8 | [Notifications](80-notifications.zh-CN.md) | `approved` |
| 9 | [Audit 与 Operations](90-audit-operations.zh-CN.md) | `approved` |
| 10 | [External Portal](100-external-portal.zh-CN.md) | `approved` |
| 11 | [Shared 与入口适配层](110-shared-entry-adapters.zh-CN.md) | `approved` |

顺序沿用已确认模块契约。Shared 最后审查，是为了先从实际模块需求验证公共字段，避免提前建立万能共享模型。

## 8. 每个模块的审查步骤

```text
已确认 BR-* 与模块契约
  -> 读取当前 domain/schema/migration
  -> 画目标领域关系
  -> 冻结对象与不可变量
  -> 逐表展示并确认全部目标字段
  -> 冻结逻辑表和约束
  -> 列出现状差异与 corrective migration 输入
  -> 项目负责人确认
  -> 进入下一模块
```

每次只审查一个完整模块；在同一次审阅中展示该模块的全部表，每张表都使用“字段、必填、现状、用途”矩阵。该模块整体确认前不进入下一模块，文档保持 `draft_for_review`，不能作为 migration 实现依据。

## 9. 本步骤验收标准

项目负责人只需确认：

1. 接受先领域对象、再逻辑表、最后 migration 的顺序。
2. 接受第 3 节的统一数据规则。
3. 接受逐模块审查，第一项从 Identity 开始。

本阶段已完成：11 个模块均已完成领域对象、目标表字段、约束方向和现状差异审查。下一阶段进入业务流程与状态机设计；本索引不直接进入 migration 实现。
