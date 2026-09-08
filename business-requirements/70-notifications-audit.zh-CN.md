# Notifications、Audit 与 Operations

返回[唯一事实源索引](README.md)。

## BR-038 站内通知

状态：`confirmed`

- Release 1 只提供站内通知，不向 Guardian 或 Student 发送外部业务 Email、SMS 或 WhatsApp。
- Task 分配或重派通知新 Assignee；Task 被拒绝通知 Primary Advisor。
- 候选名单提交审批通知 Founder；Founder 批准或驳回通知 Primary Advisor；批准后待 Guardian 确认仍由 Primary Advisor 跟进。
- Task 到期前 3 天和 1 天提醒实际 Assignee 与 Primary Advisor。
- Task 逾期后每天提醒实际 Assignee、Primary Advisor 和 Founder，直至完成或取消；暂停期间继续。
- 所有 SchoolTarget 终态且无未完成 Task 时，通知 Primary Advisor 与 Founder 选择新增学校或人工结案。
- 同一接收人、同一 effect、同一天只生成一条。
- 可见文案只表达“有待办事项”，不披露 Student、Guardian、Case、SchoolTarget、学校或文件信息。
- Founder 审批、Guardian 确认、等待结果、offer 决定和结案本身不自动生成 Task；通知不等于 Task。

## 与 BR-075 内部账号邮件的边界

状态：`confirmed`

- 账号邀请确认邮件只发送给 Founder 指定的内部用户邮箱。
- 邮件由独立 Email 模块负责，不与站内通知共用业务实体或权限判断。
- 邮件内容只包含确认链接、有效期和最小操作说明；不得包含密码、角色以外的业务资料、学生或家长信息。
- 每次发送必须记录可重放的投递回执和发送状态；不得保存明文确认 token。
- 发送失败不得把邀请标记为已完成；Founder 可以在受控操作下重新发送并使旧链接失效。
- Email 模块可以使用经过配置的 SMTP 或事务邮件传输；认证本身不依赖第三方身份平台。

## BR-070 审计与一致性

状态：`confirmed`

- 状态转换、授权、批准、确认、导出、删除、恢复和高风险读取写入追加式审计。
- 业务修改与必需审计在同一事务完成；审计保存失败时业务修改也失败。
- 重要写入带预期版本；并发冲突不得静默覆盖，应返回稳定冲突结果供用户重新确认。
- 重复请求不得重复创建 Case、SchoolTarget、Task、通知、文件版本或其他副作用。
- 每项业务写入必须能够识别并记录所属组织和实际操作人；请求上下文、幂等键和传播方式属于技术设计。
- 看板、搜索、缓存和统计都是可重建投影，不是授权来源或业务事实。

## BR-071 隐私与数据边界

状态：`confirmed`

- 未来生产的敏感数据、日志、临时副本和备份必须位于批准的香港边界。
- 日志、审计可见摘要和通知不得包含姓名、联系信息、Assessment answer、文件内容、Cookie、token、secret、完整表单或自由文字 PII。
- 平台审计与租户业务审计分离；平台人员不能借审计或统计读取租户内容。
- Student 与 Guardian 按 `BR-029` 仅做软删除，数据库不得物理删除或最终 purge；该规则已取代此前的 purge 规则。ServiceCase 按 `BR-039` 不允许删除或软删除，只能结案并永久保留历史。
