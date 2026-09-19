# Tasks

返回[唯一事实源索引](README.md)。

## BR-035 申请准备与提交 Task

状态：`confirmed`

- SchoolTarget 进入 `preparing` 时，逐校指定 Application Assignee。
- Application Assignee 可为 Primary Advisor 或有明确案件授权的 Case Collaborator。
- 系统为该校自动创建一条“准备并提交申请”Task；同一学校、同一申请轮次必须幂等。
- Assignee 同时负责准备和提交，不拆成两个 Task。
- 每所学校按当期官方要求维护自己的材料清单，不设全校通用固定清单。
- Task 完成前必须保存提交时间、渠道、提交人、清单完成状态，以及学校参考号或至少一份其他提交凭证。
- 学校不提供参考号时必须明确记录，并保存确认页、确认邮件 PDF、回执或邮寄凭证等其他证据。
- 提交凭证复用 SchoolTarget、Task、Case Document 和审计能力，不新增 Material 或 SubmissionEvidence 实体。

## BR-036 面试辅助 Task

状态：`confirmed`

- 仅在学校明确要求面试时创建。
- 默认由 Primary Advisor 负责，也可指派已作为 Case Collaborator 的正式 Advisor，或通过单一 TaskAssignment 指派 Contractor。
- 非 Primary Advisor 只能看到该 Task、目标学校、面试时间／方式／语言、辅导要求和 Primary Advisor 整理的必要背景摘要。
- 不得查看完整 Assessment、Guardian/联系方式、内部备注、案件文件、下载/导出、其他学校或其他案件。
- Task 完成不代表学校结果，只表示面试辅助工作完成。
- Contractor 不因此成为 CaseCollaborator、申请提交负责人或案件成员；拒绝、完成、取消、重派或案件结束后立即失去工作区访问。

## BR-037 Task 通用规则

状态：`confirmed`

- 背景资料收集不自动创建 Task。
- 自动 Task 只有申请准备/提交和必要面试两类。
- Primary Advisor 可按真实需要为自己负责的案件人工创建临时 Task；人工 Task 不自动推进 Case 或 SchoolTarget。
- 新 Task 从 `assigned` 开始，Assignee 可接受或填写原因拒绝。
- 拒绝结束当前 TaskAssignment，Task 保留并等待 Primary Advisor 在同一 Task 上重派。
- 重派保留 Assignment 历史，新 Assignee 重新接受，不创建替代 Task。
- `overdue` 由 `due_at` 计算，是提示与提醒标记，不是独立业务状态；逾期后仍可完成。
- Assignee 保存该 Task 要求的完成记录和证据后即可完成；Founder 不逐条验收，也不经过 `approved`。
- 拒绝、重派、取消和完成保存操作人、时间、原因或完成记录和审计。
- 暂停时保留原 Task；已确认名单移除学校或整体终止服务时，受影响未完成 Task 取消但保留历史。

## BR-015 试用等级对任务规则的取代范围

状态：`confirmed`（试用实施）

显式试用等级按 BR-015 授权：Founder/L1 可管理全部案件任务，L2 只可管理获授权分类任务，均可指派 L3。L3 仅可读取当前指派任务的必要上下文并执行该任务，不因此获得其他任务或完整案件权限。完成后只读至撤销；拒绝、撤销、重派、取消立即失去旧任务工作区权限。BR-036 旧完成即失权及 BR-035/037 旧基础角色/负责人限制在该试用范围为 `superseded`；任务完成条件、提交证据、历史与审计要求继续适用。
