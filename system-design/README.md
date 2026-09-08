# 系统设计索引

业务事实来源：[`business-requirements/`](../business-requirements/README.md)  
现状差异来源：[`current-state-analysis/`](../current-state-analysis/README.md)

## 使用规则

- 业务需求回答“系统必须做什么”，优先级最高。
- 系统设计回答“准备怎样实现”，不得覆盖已确认的 `BR-*`。
- 当前代码和旧文档只能证明现状，不能反向改变业务需求。
- 设计状态为 `draft_for_review` 时，不据此开始业务实现。
- 设计经项目负责人确认后改为 `approved`，再拆分开发票据。

## 设计进度

| 顺序 | 环节 | 文档 | 状态 |
| ---: | --- | --- | --- |
| 1 | 总体架构与系统边界 | [总体架构](00-overall-architecture.zh-CN.md) | `approved` |
| 2 | 模块职责与依赖契约 | [模块契约索引](10-module-contracts/README.md) | `approved` |
| 3 | 领域模型与数据设计 | [领域模型与数据设计索引](20-domain-data-models/README.md) | `approved` |
| 4 | 业务流程与状态机 | [业务流程与状态机索引](30-business-flows/README.md) | `approved` |
| 5 | 权限与安全设计 | [权限与安全设计索引](40-permissions-security/README.md) | `approved` |
| 6 | API 与页面交互设计 | [API 与页面交互设计索引](50-api-ui/README.md) | `approved` |
| 7 | 非功能设计、评审与开发拆分 | [非功能设计与开发拆分索引](60-nfr-delivery/README.md) | `approved` |

## 设计顺序约束

```text
业务基线
  -> 现状差异
  -> 总体架构
  -> 模块契约
  -> 数据与流程
  -> 权限
  -> API/UI
  -> 非功能与评审
  -> 开发票据
```

不能跳过模块所有权直接设计共享表，也不能在状态机未确定前冻结 API。

系统设计 7 个阶段已全部完成。`approved` 表示设计基线已冻结，不表示 P1-P6 已开发、测试或部署。
