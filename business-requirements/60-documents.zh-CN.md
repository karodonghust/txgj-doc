# Documents

返回[唯一事实源索引](README.md)。

## BR-050 文件

状态：`confirmed`

- 文件内容保存在私有对象存储，数据库保存 metadata、关联、权限、版本和状态。
- 文件没有公开 URL；上传、预览、下载、导出、删除和恢复前都重新检查服务端权限并记录审计。
- 新版本不得覆盖旧版本；只有扫描通过且未撤销的版本可以使用。
- 文件流程：`pending_upload -> quarantined -> scanning -> available`，扫描可进入 `rejected` 或 `scan_failed` 后有界重试。
- 文件默认 30 天 soft-delete 恢复窗口；legal hold 期间不得清除。
- Founder、当前 Primary Advisor 和该校当前 Application Assignee 可以下载该校申请必需的已扫描文件。
- Application Assignee 访问只限所负责学校，职责或案件授权失效后立即失去访问。
- 面试辅助人、Guardian、Student 和 Portal 不可下载。

## BR-015 试用等级对文件规则的取代范围

状态：`confirmed`（试用实施）

试用等级替代 BR-050 的旧基础角色判断：Founder/L1 可在全部业务范围操作文件，L2 限获授权分类，L3 只限当前任务明确关联并允许该动作的文件，不得枚举全案文件。明确关联不是下载自动授权；下载必须同时满足任务用途许可和版本扫描通过且未撤销。面试辅助任务不授予下载。撤销/重派后收回文件访问；已完成任务只读，不可上传。原私有存储、版本、安全扫描、软删除和审计约束继续适用，禁止新增批量下载/导出。
