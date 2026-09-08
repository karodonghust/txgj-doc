# 业务基线变更记录

返回[唯一事实源索引](README.md)。

| 基线 | 日期 | 内容 |
| --- | --- | --- |
| `BR-BASELINE-20260908-v37` | 2026-09-08 | 新增仅 Admin 可用的邮件模板管理；Release 1 只管理内部使用者注册邀请模板，允许修改主题和正文说明并预览；一次性链接、按钮和有效期继续由系统固定安全生成，正文以纯文本保存并在 HTML 中转义 |
| `BR-BASELINE-20260908-v36` | 2026-09-08 | 确认 Resend 通过项目代码依赖集成，不使用 Vercel Marketplace；新增仅 Admin 可用的组织级邮件设置，允许在系统中保存或轮换 Resend API Key 与发件人资料；API Key 加密保存且任何读取、日志和审计均不回显秘密 |
| `BR-BASELINE-20260907-v35` | 2026-09-07 | 甲方确认不开放用户自助注册和第三方身份登录；仅由 Founder 邀请内部用户，系统发送一次性确认链接，受邀人设置初始密码和个人资料后激活；新增独立 Email 模块负责邀请邮件和投递回执 |
| `BR-BASELINE-20260829-v34` | 2026-08-29 | 确认 EmployeeProfile `display_name` 的业务含义和页面名称统一为“昵称”；工作区默认头像显示昵称首字；Release 1 不保存员工真实姓名，登录邮箱继续由 Identity 保存 |
| `BR-BASELINE-20260828-v33` | 2026-08-28 | 确认内部员工本人可以修改自己的 EmployeeProfile `display_name`，但不能修改登录邮箱、员工类型或角色；Founder/Admin 继续通过受控成员管理命令维护员工资料和角色，并受资格、兼容性、最后一个 Founder、并发版本和审计约束 |
| `BR-BASELINE-20260825-v32` | 2026-08-25 | 项目负责人改定客户来源模型：保留独立 ReferralSource 来源目录，并由 Case 建立可追溯关联；取代 v10/v11 中“不建立独立来源实体、直接保存 Case 来源字段”的结构决定，11 个来源类型继续有效 |
| `BR-BASELINE-20260825-v31` | 2026-08-25 | 依据已确认的 `BR-013` 消除 `BR-032` 旧冲突：Admin 基础角色默认不可查看 Case/Assessment；只有同时取得 Advisor 角色并满足案件关系时才按对应 Advisor 权限访问 |
| `BR-BASELINE-20260825-v30` | 2026-08-25 | 确认 ServiceCase 不允许删除或软删除，只能取消服务并人工结案，数据库永久保留案件及相关历史 |
| `BR-BASELINE-20260825-v29` | 2026-08-25 | 确认 Future 范围：销售收费、外部渠道与 AI、非 K12、多组织、Data Reviewer、资料合并和删除恢复均不在 Release 1，且不预建业务实体 |
| `BR-BASELINE-20260825-v28` | 2026-08-25 | 确认 Audit & Operations 模块业务审阅完成；关键操作审计、事务一致性、重复请求和敏感信息边界作为 Release 1 规则 |
| `BR-BASELINE-20260825-v27` | 2026-08-25 | 确认 Notifications 模块业务审阅完成；站内通知范围、触发节点、接收人、去重和隐私边界作为 Release 1 规则 |
| `BR-BASELINE-20260825-v26` | 2026-08-25 | 确认 Documents 模块业务审阅完成；私有存储、版本与扫描、下载权限和软删除边界作为 Release 1 规则 |
| `BR-BASELINE-20260825-v25` | 2026-08-25 | 确认 Schools 模块业务审阅完成；provisional School、Founder 统一审核、版本历史和发布边界作为 Release 1 规则 |
| `BR-BASELINE-20260825-v24` | 2026-08-25 | 确认 Release 1 不设置 Data Reviewer；所有学校资料变更统一由 Founder 审核，后续确有工作量再增加该角色 |
| `BR-BASELINE-20260825-v23` | 2026-08-25 | 确认 Tasks 模块业务审阅完成；自动任务范围、分派生命周期、逾期和完成规则作为 Release 1 业务边界，其余归技术设计 |
| `BR-BASELINE-20260825-v22` | 2026-08-25 | 确认兼容多角色与权限包含关系、Contractor 单角色限制、Admin 默认不可查看 Case/Assessment；Shared 只保留幂等及组织/操作人业务要求 |
| `BR-BASELINE-20260825-v21` | 2026-08-25 | 确认 Platform Billing 整体移出 Release 1；推进中 Case 计数、合同参考值、订阅状态和平台财务角色旧规则标记为 `superseded` |
| `BR-BASELINE-20260825-v20` | 2026-08-25 | 确认 Portal 仅由 Primary Advisor 创建、Primary Advisor 或 Founder 撤销，有效 7 天且到期需重建；会话和 Token 形式归安全技术设计 |
| `BR-BASELINE-20260825-v19` | 2026-08-25 | 确认 Guardian Portal 仅展示对客阶段、更新时间、批准学校进度及家长可见消息和行动项；未列字段默认拒绝 |
| `BR-BASELINE-20260825-v18` | 2026-08-25 | 重新确认 Guardian Portal 保留临时、单 Case、只读核心边界；精确授权参数与 Platform Billing 旧规则重新标为 `pending_review`，不再计为已完成 |
| `BR-BASELINE-20260825-v17` | 2026-08-25 | 确认 `deleted` Student/Guardian 在 Release 1 中不可通过业务页面或接口查看且不可恢复；取代 v15 的“已删除筛选查看”规则 |
| `BR-BASELINE-20260825-v16` | 2026-08-25 | 确认 Student 有未结案 Case、Guardian 有当前 Student 关系时禁止软删除；Founder 批准时必须重新检查 |
| `BR-BASELINE-20260825-v15` | 2026-08-25 | 确认 `pending_delete` 仍显示并标记；`deleted` 默认隐藏、可专项筛选查看，且不得用于新 Case 或新 Guardian 关系 |
| `BR-BASELINE-20260825-v14` | 2026-08-25 | 确认 Advisor 可申请、Founder 决定 Student/Guardian 删除；只做 `deleted` 软删除，数据库禁止物理删除；旧 `purged` 和最终 purge 规则被取代 |
| `BR-BASELINE-20260825-v13` | 2026-08-25 | 确认 Release 1 只提供疑似重复警告，不提供 Student 或 Guardian 的人工合并和撤销合并 |
| `BR-BASELINE-20260825-v12` | 2026-08-25 | 确认仅在姓名、Email 或电话任一非空值相同时显示疑似重复警告；出生日期不触发，系统不得自动关联或合并 |
| `BR-BASELINE-20260825-v11` | 2026-08-25 | 确认客户来源的 11 个受控选项；选择 `other` 时来源说明必填 |
| `BR-BASELINE-20260825-v10` | 2026-08-25 | 确认客户来源是每个 Case 上的可选来源类型和说明，不建立独立 ReferralSource 实体；具体受控选项仍待确认 |
| `BR-BASELINE-20260825-v9` | 2026-08-25 | 确认 Guardian 关系解除只结束当前关联并保留历史；主要联系人必须先完成交接 |
| `BR-BASELINE-20260825-v8` | 2026-08-25 | 确认主要联系人从现有关联 Guardian 中原子交接并保留历史；旧主要联系人继续保留关联关系 |
| `BR-BASELINE-20260825-v7` | 2026-08-25 | 确认添加关联 Guardian 可新建或手动选择已有对象；关系类型必填，新增关系默认不改变当前主要联系人 |
| `BR-BASELINE-20260825-v6` | 2026-08-25 | 确认 CRM 不设置次要联系人身份或排序；除唯一主要联系人外均为关联 Guardian，各类职责使用独立标志 |
| `BR-BASELINE-20260825-v5` | 2026-08-25 | 确认主要联系人只表示 Student 日常沟通的优先联系人，不自动取得法定监护、紧急联系、账单、最终选校确认、通知或 Portal 权限 |
| `BR-BASELINE-20260824-v4` | 2026-08-24 | 确认创建 Student 时必须在同一原子操作中建立唯一当前主要联系人；允许手动选择已有 Guardian，禁止自动关联 |
| `BR-BASELINE-20260824-v3` | 2026-08-24 | 确认 Student-Guardian 多对多关系、唯一当前主要联系人、关系版本历史及完整 `relationship_type` 受控选项 |
| `BR-BASELINE-20260824-v2` | 2026-08-24 | 确认 Guardian 最小资料；增加可空 `gender` 和 `date_of_birth`，年龄按当前日期计算整岁且不持久化；关系规则继续保持 `pending_review` |
| `BR-BASELINE-20260824-v1` | 2026-08-24 | 建立唯一事实源；汇总本对话已确认的角色、授权、EmployeeProfile、Student/gender、K12 案件、逐校申请、Task、通知、文件、Portal、平台计数和审计边界；未完成的 CRM 与模块设计明确标为 `pending_review` |
