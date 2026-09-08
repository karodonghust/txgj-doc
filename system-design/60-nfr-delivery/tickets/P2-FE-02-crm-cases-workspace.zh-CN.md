# P2-FE-02 CRM、建案与 Cases 工作台

状态：`ready_after_F1_F2_approval_and_P2-BE`  
Owner：`frontend`  
依赖：P1-FE-01、F1/F2 线框批准、P2-BE-03/04 API 可用

返回[开发票据索引](README.md)。

## 业务结果

Advisor 可以从客户资料进入建案和 Assessment；Founder/Advisor 在同一 Case 工作台完成名单两层确认并理解所有阻塞原因。

## 范围

- Student/Guardian/relationship/ReferralSource 列表、详情和受控表单。
- 建案向导、Assessment、Cases 列表、Case summary 与候选名单工作区。
- 重复警告明确显示命中字段，只允许人工继续/取消，不提供自动关联。
- Founder 审批、Guardian 确认代录、stale version 重新载入及全部学校终态双分支。
- 使用 `allowed_actions` 和服务端 DTO；完整 loading/empty/denied/stale/unavailable/error 状态。

## 不在范围

- 不实现逐校 Task/Document/Portal；不允许 Admin 单独浏览客户资料；不让 Guardian 登录内部页面确认。

## 验收

- Founder/Advisor/Admin/Collaborator/Contractor 页面边界和按钮正确，服务端拒绝可理解。
- deleted 主档不可查看；pending_delete 明确显示；来源停用不影响历史显示。
- 桌面/移动、键盘、焦点、表单错误和刷新后持久状态通过。
- focused UI tests、`pnpm typecheck` 和新增 `test:p2-fe-02` 通过；Local Dev browser happy/negative 流程由现有独立测试验收。

## 停止与回滚

线框/API/DTO 未冻结、页面需要 mock 冒充保存成功或授权只能靠隐藏按钮时停止。回滚 UI/client 变更，不回滚已提交业务历史。

