# 模块职责与依赖契约

状态：`approved`  
确认依据：项目负责人于 2026-08-25 确认最后一个模块并指示进入领域模型与数据设计

返回[系统设计索引](../README.md)。

## 本环节产物

每个模块单独冻结：

1. 模块拥有的业务事实。
2. 可以接收的命令和查询。
3. 可以对外发布的事实。
4. 允许依赖的模块。
5. 明确禁止的职责和依赖。

这里只确定模块契约，不提前确定表字段、HTTP payload 或页面布局。

## 设计顺序

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

顺序按依赖从基础到业务核心排列。Platform Billing 不进入 Release 1 模块契约。
