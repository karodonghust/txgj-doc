# 权限与安全设计索引

状态：`approved`  
开始日期：2026-08-25  
业务基线：`BR-BASELINE-20260825-v32`

返回[系统设计索引](../README.md)。

## 1. 本阶段要冻结什么

本阶段回答：

- 用户如何认证，系统如何确认“你是谁”；
- 基础角色、Capability 和业务关系如何叠加；
- 谁能访问哪个 Case、SchoolTarget、Task 和文件；
- Portal、Contractor、Admin、Founder、Advisor 的边界如何落实；
- 哪些请求必须审计、限流、拒绝或 fail closed。

本阶段不新增业务角色，不修改已确认的业务流程，也不直接执行 migration。

## 2. 审查顺序

| 顺序 | 模块 | 文档 | 状态 |
| ---: | --- | --- | --- |
| 1 | 内部认证与授权模型 | [内部认证与授权模型](10-authorization-model.zh-CN.md) | `approved` |
| 2 | 高风险操作与安全控制 | [高风险操作与安全控制](20-security-controls.zh-CN.md) | `approved` |
| 3 | 数据保留、备份恢复与运行时验证 | [数据保留、备份恢复与运行时验证](30-data-protection-runtime.zh-CN.md) | `approved` |

先冻结认证、RBAC、Capability 和案件关系的组合规则，再分别审查文件、Portal、Contractor 和高风险操作的安全边界。

## 3. 固定审查方式

```text
认证边界
  -> 基础角色
  -> capability
  -> 业务关系/资源范围
  -> deny 优先级
  -> 高风险操作
  -> 审计与 fail closed
  -> 项目负责人确认
```

本阶段已完成：认证、角色、Capability、资源关系、高风险操作、审计、限流、数据保留和运行时安全边界均已确认。
