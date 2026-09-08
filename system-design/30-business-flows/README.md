# 业务流程与状态机设计索引

状态：`approved`  
开始日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[系统设计索引](../README.md)。

## 1. 本阶段要冻结什么

本阶段把已确认的业务需求转换为可执行的流程和状态机，明确：

- 哪些状态是全局案件状态，哪些状态只属于逐校申请。
- 哪个角色可以触发每次转换。
- 前置条件、必需证据、任务和跨模块事件。
- 暂停、终止、结案和“全部学校拒绝”时的分支。

本阶段不冻结 API URL、页面布局或数据库 migration；这些在流程确认后处理。

## 2. 审查顺序

| 顺序 | 模块 | 文档 | 状态 |
| ---: | --- | --- | --- |
| 1 | Cases | [Cases 案件流程与状态机](10-cases.zh-CN.md) | `approved` |
| 2 | Tasks | [Tasks 任务流程与状态机](20-tasks.zh-CN.md) | `approved` |
| 3 | Documents | [Documents 文件流程与状态机](30-documents.zh-CN.md) | `approved` |
| 4 | Notifications | [Notifications 通知流程与状态机](40-notifications.zh-CN.md) | `approved` |
| 5 | Audit 与 Operations | [Audit 与 Operations 流程与状态机](50-audit-operations.zh-CN.md) | `approved` |
| 6 | External Portal | [External Portal 流程与状态机](60-external-portal.zh-CN.md) | `approved` |
| 7 | Shared 与入口适配层 | [Shared 与入口适配流程](70-shared-entry-adapters.zh-CN.md) | `approved` |

先冻结 Cases，再依次审查 Tasks、Documents、Notifications、External Portal 和 Shared 受到流程影响的入口规则。

## 3. 固定审查方式

```text
流程总览
  -> 全局状态机
  -> 逐校状态机
  -> 任务产生点
  -> 角色与前置条件
  -> 异常/分支
  -> 项目负责人确认
```

本阶段已完成：7 个模块的流程、状态、任务、入口、异常分支和跨模块边界均已确认。后续实现仍需遵守权限、安全和 API/UI 阶段的冻结结果。
