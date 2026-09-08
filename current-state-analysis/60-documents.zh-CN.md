# Documents 现状分析

返回[现状分析总览](README.md)。业务依据：[Documents](../business-requirements/60-documents.zh-CN.md)。

## 结论

| BR | 状态 | 当前实现摘要 |
| --- | --- | --- |
| `BR-050` | `部分符合` | 私有对象、不可变版本、扫描、有界重试、30 天恢复、legal hold、审计与短期 URL 基础较完整；下载授权和生产组合仍不完整 |

## 已有基础

- 对象键为 opaque UUID 路径，文件 metadata 和版本保存在 PostgreSQL。
- 上传只允许 PDF/JPEG/PNG，最大 10 MiB；上传 intent 10 分钟，下载 intent 5 分钟。
- 状态包含 `pending_upload -> quarantined -> scanning -> available`，也支持 rejected、scan_failed 和最多 3 次扫描尝试。
- clean 且未 revoked 的 available version 才能激活或下载；新版本不会覆盖旧版本。
- 文件支持 30 天 soft-delete/restore 窗口；legal hold 阻止清除。
- 注册、版本、上传 intent、下载 intent、删除、恢复和回滚路径已写审计/outbox。
- DOC-01 与 DOC-02 已有 PostgreSQL、HTTP/浏览器测试资产；本轮未重新运行。

## 关键差异

| 优先级 | 当前代码 | 与业务基线的差异 | 建议动作 |
| --- | --- | --- | --- |
| `P0` | download repository 只允许 Founder 或当前 Primary Advisor | 缺“该校当前 Application Assignee”按学校范围下载 | 将文件与 SchoolTarget/申请用途显式关联，事务内重验当前 assignment 和文件必需性 |
| `P0` | 通用 capability `documents.download` 先按基础角色判断 | 仅有 capability 不能证明具体学校职责 | 下载授权最终必须同时满足角色/关系、Case、SchoolTarget、assignment 和文件分类 |
| `P0` | `DocumentUploadRuntime` 固定 unavailable；transfer/scan 只支持 local-synthetic | 正式对象存储、扫描 worker 和生产数据库未组合 | 先完成本地闭环，再分别验证 Vercel Preview 能力边界和 AWS 香港生产组合 |
| `P1` | policy 支持 retention 到期后 Founder purge | BR-050 允许文件在窗口和 legal hold 规则下清理，但需避免与 CRM 禁止 purge 混淆 | 保持文件生命周期独立；明确文件清理不代表删除 Student/Guardian/Case 记录 |
| `P1` | 未见 Portal/面试辅助下载入口，但需靠拒绝测试证明 | 默认不可见不等于服务端永远禁止 | 增加 Guardian、Student、Portal、Interview Assignee 的稳定拒绝测试 |

## 完成标准

- Founder、当前 Primary Advisor、该校当前 Application Assignee 能按精确业务范围下载。
- Application Assignee 失去学校职责或案件授权后，下一个请求立即失败。
- Interview Assignee、Guardian、Student 和 Portal 无法取得下载 intent。
- 本地对象存储、队列、ClamAV 和 PostgreSQL 完成一次真实上传、扫描、下载、删除与恢复闭环。
- Preview 与 AWS 验证分别记录，不能用本地结果代替。
