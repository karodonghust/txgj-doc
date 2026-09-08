# Tianxingguoji 文档入口

## 唯一业务事实源

未来开发只能引用：

[`business-requirements/README.md`](business-requirements/README.md)

本 Codex 对话负责产生和确认业务决定；上述目录负责把确认结果分类持久化。模块文件中的 `confirmed` 规则才可以进入开发，`pending_review` 不得由开发自行补全。

除本 README 外，活跃文档区只保留这一个业务需求目录。其他需求、决策、阶段记录、技术设计、领域说明和研究材料均已移入 `archive/`。

## 其他文档的地位

| 位置 | 用途 | 能否定义业务逻辑 |
| --- | --- | --- |
| `business-requirements/` | 当前分类业务基线 | 可以，唯一来源 |
| `archive/pre-canonical-business-baseline-2026-08-24/` | 唯一基线建立前的旧业务文档原文 | 不可以，只作本对话核对材料 |
| `archive/reference-materials-2026-08-24/` | 阶段记录、技术设计、领域说明、研究及旧路径提示 | 不可以，只作历史参考 |
| Tianxingguoji 代码、迁移和测试 | 当前实现证据 | 不可以反向定义需求 |

## 更新规则

1. 项目负责人在本对话明确确认业务规则。
2. 同一轮更新唯一业务事实源和变更记录。
3. 新开发票据引用具体 `BR-*` 编号。
4. 发现代码、旧文档或测试与 `BR-*` 冲突时，记录实现差异，不修改业务规则迁就旧实现。
5. 没有 `confirmed` 规则时，停止相关业务实现并回到本对话确认。

## 当前审阅位置

- 已确认：Identity、Access、EmployeeProfile、K12 Cases、CRM Student、Guardian 及其关系、External Portal、Platform Billing。
- 当前模块：CRM。
- 下一项：Student 与 Guardian 的建档方式。

归档索引：

- [`archive/pre-canonical-business-baseline-2026-08-24/README.md`](archive/pre-canonical-business-baseline-2026-08-24/README.md)
- [`archive/reference-materials-2026-08-24/README.md`](archive/reference-materials-2026-08-24/README.md)
