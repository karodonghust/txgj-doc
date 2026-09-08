# Schools 现状分析

返回[现状分析总览](README.md)。业务依据：[Schools](../business-requirements/50-schools.zh-CN.md)。

## 结论

| BR | 状态 | 当前实现摘要 |
| --- | --- | --- |
| `BR-051` | `冲突` | provisional、ChangeRequest、overlay、历史回滚和 warning receipt 已建模，但 Data Reviewer 仍能审核普通字段，且正式运行时未接通 |

## 已有基础

- Advisor 创建 provisional School 的服务和 API 已存在。
- 学校修改通过 ChangeRequest 和 candidate overlay 表达，不直接覆盖爬虫快照。
- overlay revision 不可变；批准后形成 resolved view；停用 revision 可回退，不改写历史。
- 领域契约禁止提交者自审。
- crawler snapshot 对 warning manifest 有 hash、精确 warning 集合、有效期和 Founder receipt 校验。
- 学校读取、ChangeRequest、审核、resolved view 和 overlay disable/reconcile 都已有路由与测试资产。

## 关键差异

| 优先级 | 当前代码 | 与业务基线的差异 | 建议动作 |
| --- | --- | --- | --- |
| `P0` | `SchoolReviewRole` 和 governance service 允许 `data_reviewer` | Release 1 所有学校变更统一由 Founder 审核 | 移除 Data Reviewer 审核路径；普通字段与 identity 字段都只允许 Founder |
| `P0` | Admin 具有 `schools.manage`/`crawler.manage` | Admin 是否可操作学校业务与已确认 Founder 审核边界混杂 | Admin 可保留技术配置入口，但不得批准、驳回或发布学校业务变更 |
| `P0` | `runtime.ts`、`school-governance-runtime.ts`、`resolved-view-runtime.ts` 固定抛出 unavailable | API 存在但业务无法运行 | 接通同一 PostgreSQL 权威数据源，避免页面继续落到 mock/legacy crawler 路径 |
| `P1` | `mock-schools.ts` 和旧 crawler API 仍被页面/兼容入口使用 | 用户可能看到与权威 overlay 不一致的数据 | 统一目录读取到 resolved view；mock 只允许测试夹具使用 |
| `P1` | warning receipt 仍要求 reviewer recommendation + Founder accept | Release 1 不设 Data Reviewer | 将精确 manifest 的一次性批准简化为 Founder 独立决策与审计记录 |

## 完成标准

- Advisor 可以提交 provisional 和 ChangeRequest，但不能审核自己的请求。
- 所有学校变更、合并、拆分、停用、官网和 warning manifest 只由 Founder 决定。
- 学校页面、候选学校选择和 SchoolTarget pin 都读取同一 resolved view。
- 快照发布、Git 发布和应用部署保持独立，不互相推导授权。
