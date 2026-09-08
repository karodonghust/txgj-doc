# API 与页面交互设计索引

状态：`approved`  
开始日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[系统设计索引](../README.md)。

## 1. 本阶段要冻结什么

本阶段把已经确认的流程、数据和权限转换为用户可以实际操作的纵向切片：

- 正式 API 路径、method、输入和响应 DTO；
- 页面/工作台的信息层级和状态；
- loading、empty、error、denied、unavailable 等边界；
- 幂等、expected version、权限和错误如何在 UI 中表现。

本阶段不新增业务规则；页面不能绕过 owning module，也不能用本地状态代替服务端事实。

## 2. 审查顺序

| 顺序 | 纵向切片 | 文档 | 状态 |
| ---: | --- | --- | --- |
| 1 | Cases 案件工作台 | [Cases 案件工作台 API 与页面](10-cases-workspace.zh-CN.md) | `approved` |
| 2 | Task 工作台 | [Task 工作台 API 与页面](20-task-workspace.zh-CN.md) | `approved` |
| 3 | Documents 文件工作台 | [Documents 文件工作台 API 与页面](30-documents-workspace.zh-CN.md) | `approved` |
| 4 | Notifications 入口 | [Notifications 入口 API 与页面](40-notifications-ui.zh-CN.md) | `approved` |
| 5 | External Portal 工作区 | [External Portal 工作区 API 与页面](50-portal-workspace.zh-CN.md) | `approved` |

## 3. 前端设计批次

| 批次 | 文档 | 状态 |
| --- | --- | --- |
| F0 | [Release 1 前端信息架构与页面范围](frontend-design/10-information-architecture.zh-CN.md) | `approved` |
| F1 | [Today、Cases 列表和案件工作区线框](frontend-design/20-f1-today-cases-wireframes.zh-CN.md) | `approved` |
| F2 | [CRM 与建案向导线框](frontend-design/30-f2-crm-case-intake-wireframes.zh-CN.md) | `approved` |
| F3 | [选校、逐校申请与面试线框](frontend-design/40-f3-school-selection-application-interview-wireframes.zh-CN.md) | `approved` |
| F4 | [Documents 与 Notifications 线框](frontend-design/50-f4-documents-notifications-wireframes.zh-CN.md) | `approved` |
| F5 | [Guardian Portal 工作区线框](frontend-design/60-f5-guardian-portal-wireframes.zh-CN.md) | `approved` |

F1/F2/F3/F4/F5 只产出线框和交互状态，不修改产品源码；通过 Architect 验收后才进入对应 Frontend 实现。

## 4. 固定审查方式

```text
用户目标
  -> 页面状态和操作
  -> API 命令/查询
  -> DTO 与错误
  -> 权限/幂等/版本
  -> loading/empty/denied/unavailable
  -> 项目负责人确认
```

本阶段已完成：Cases、Tasks、Documents、Notifications 和 External Portal 的 API、DTO、页面状态、权限、幂等和错误边界均已确认。下一步进入非功能设计与开发拆分。
