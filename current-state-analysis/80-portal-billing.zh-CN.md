# External Portal 与 Platform Billing 现状分析

返回[现状分析总览](README.md)。业务依据：[External Portal 与 Platform Billing](../business-requirements/80-portal-billing.zh-CN.md)。

## 结论

| BR | 状态 | 当前实现摘要 |
| --- | --- | --- |
| `BR-060` | `部分符合` | 单 Case、只读、字段过滤和禁用文件/编辑的契约存在；运行时不可用，且输出多了 case number |
| `BR-063` | `冲突` | 7 天、撤销、到期和 Session 安全基础存在，但旧规则允许 Founder 创建授权，所有 Portal 路由仍不可用 |
| `BR-061` | `超范围残留` | 已被取代的 advancing case count 仍有 migration、domain、测试和平台页面依赖 |
| `BR-062` | `超范围残留` | 已被取代的合同参考值、订阅投影和平台角色仍有完整数据库与代码结构 |

## Portal 已有基础

- Portal viewer 不是内部 User，使用独立 viewer、grant 和 session 表。
- grant 绑定一个 organization 和一个 ServiceCase，最大 TTL 为 7 天。
- 撤销、到期、viewer relationship、issuer 当前授权和 Case 状态会在访问时重新检查。
- workspace builder 只投影 customer-visible SchoolTarget、action item 和 message。
- contract 明确禁止 document、download、export、comment、edit 和 delete。
- Session 数量、idle/absolute timeout 和错误收敛已作为技术安全设计存在。

## Portal 关键差异

| 优先级 | 当前代码 | 与业务基线的差异 | 建议动作 |
| --- | --- | --- | --- |
| `P0` | `evaluatePortalGrantAuthorization` 允许 Founder 和当前 Primary Advisor创建 | 只有当前 Primary Advisor 可以创建；Founder 只能紧急撤销 | 拆分 issue/revoke authorization；issue 只允许当前 Primary Advisor |
| `P0` | Portal grant/session/workspace routes 注入固定抛出 `PortalRuntimeUnavailable` 的实现 | 页面只能显示“暂时不可用” | 接通 viewer、grant、session 和只读 projection repository |
| `P0` | `PortalCaseReadV1` 输出 `caseNumber` | BR-060 的“仅可展示”白名单未包含案件编号 | 从外部 DTO 移除，除非以后重新取得客户确认 |
| `P0` | domain actor roles 仍含 Data Reviewer；effective access 仍接受 subscription status | 新角色集和 R1 无 Billing 的边界未落地 | 移除 Data Reviewer 与 Billing 类型依赖；Portal 授权不得依赖订阅投影 |
| `P1` | 旧授权模型有 rotate 操作 | 到期后应由 Primary Advisor 重新生成，不能在原入口延期 | 明确 rotate 只能撤销旧 grant 并创建全新 grant/secret，不延长原记录 |

## Platform Billing 残留范围

- `modules/platform-billing/` 包含合同、月度计数、订阅、平台角色、审计和 runtime。
- `app/(erp)/platform/billing/page.tsx` 与 `/api/v1/platform/billing/overview` 仍是活跃页面/API。
- migration 012 创建 `platform_finance`、`platform_billing_approver`、合同版本、`advancing_case_count`、subscription projection 和平台审计。
- migration 013 及后续投影逻辑仍把 Case 阶段映射到 Billing 计数。
- Portal domain 仍带 `active/past_due` subscription status，虽然当前 policy 不因 past_due 阻断访问。

## Platform Billing 处理建议

`P0` 目标是“不再属于 Release 1 的可执行范围”，不是立即物理删除历史数据库对象：

1. 移除 R1 导航和公开 API 入口。
2. 切断 Portal、Case 与 Billing 的运行依赖。
3. 禁止 R1 migration/seed 主路径创建或依赖新 Billing 数据。
4. 保留历史 migration 文件，不重写 Git 历史。
5. 将未来 Billing 重新立项所需资料放入归档/未来范围，不作为当前验收项。

## 完成标准

- 只有当前 Primary Advisor 能创建全新 7 天授权；其本人和 Founder 可撤销。
- Portal 严格只返回 BR-060 白名单，不能确认、上传或下载。
- 撤销、过期或 Primary Advisor 关系失效后，下一个请求立即失败。
- Release 1 导航、API、runtime、Case 和 Portal 均不依赖 Platform Billing。
