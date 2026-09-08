# 产品边界

返回[唯一事实源索引](README.md)。

## BR-001 Release 1 目标

状态：`confirmed`

Release 1 是天星内部使用的 K12 教育服务运营系统，用来统一管理学生、家长、案件、选校、逐校申请、任务、文件、进度、权限、审批和审计。负责人应能在系统中看到每个案件的当前阶段、下一步、负责人、截止时间和异常。

## BR-002 Release 1 包含

状态：`confirmed`

- 邀请制内部账号、会话、组织成员、角色和案件级授权。
- Student、Guardian 及其关系。
- K12 ServiceCase、正式 Assessment、候选学校名单和逐校 SchoolTarget。
- Task、分派、拒绝、重派、完成、取消和计算型逾期。
- 私有文件、版本、扫描、下载授权、软删除和恢复。
- 学校公开数据快照、人工变更审核、overlay 和 resolved view。
- 站内通知、业务审计和运营看板。
- 单案件、限时、字段白名单的只读 Guardian Portal。

## BR-003 Release 1 不包含

状态：`confirmed`

- 签约前 Lead、咨询、Quote 和销售 Contract 生命周期；不新增这些销售实体。
- 非 K12 的正式流程；大学、硕士等只可显示“正在开发中”，不能正式建案。
- 合作渠道登录；银行、保险等只作为 ReferralSource。
- Guardian 或 Student 的外部业务 Email，以及 SMS、WhatsApp 自动通知。
- Portal 写入、公开注册、留言、上传、文件查看或下载。
- AI 对客报告、AI 自动业务决定和知识库正式业务流程。
- 正式 Excel/CSV 导入。
- Student 或 Guardian 的人工合并和撤销合并。
- Student、Guardian 及其关系记录的数据库物理删除或最终 purge。
- ServiceCase 的软删除、物理删除或历史清除。
- 已软删除 Student 或 Guardian 的业务查看、列表筛选和恢复功能。
- `Data Reviewer` 基础角色；Release 1 的学校资料变更统一由 Founder 审核，后续确有工作量再增加该角色。
- Platform Billing，包括推进中案件计数、合同参考值、平台合同审批、订阅状态和相关平台财务角色。
- 收费公式、应付金额、税务发票、付款、退款、催收、总账和收入确认。
- 第二个正式组织和正式多租户商业启用。
