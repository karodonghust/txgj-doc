# P3-FE-03 逐校申请、Task 与 Contractor 工作区

状态：`ready_after_F3_approval_and_P3-BE-05`  
Owner：`frontend`  
依赖：P1-FE-01、F3 线框批准、P3-BE-05 API 可用

返回[开发票据索引](README.md)。

## 业务结果

Advisor 能逐校查看申请和完成任务；Contractor 只看到自己当前面试 Task 的脱敏工作区。

## 范围

- `/tasks`、`/tasks/:taskId`、Case applications/interviews/close 视图。
- application/interview/case-completion 三类完成表单，以及拒绝、重派、取消、逾期展示。
- `allowed_actions` 驱动按钮；stale/withdrawn/cancelled/denied/unavailable 安全状态。
- Contractor 入口直接进入当前授权 Task，不渲染 Case/Student/Guardian/文件目录内容。
- 所有学校终态后显示“添加学校”与“Founder 结案”两个明确命令。

## 不在范围

- 不实现 Document 上传字节或 Notification 投递；不单建 Application/Interview 页面实体。

## 验收

- Advisor/Founder/Contractor 的列表范围、表单和返回路径正确；Admin 单独拒绝客户 Task。
- 任务完成缺少必要证据时保留输入并显示稳定校验错误，不假装成功。
- 刷新后状态来自服务端；桌面/移动/键盘/焦点和表单错误通过。
- focused UI tests、`pnpm typecheck`、新增 `test:p3-fe-03` 及 Local Dev browser 独立验收通过。

## 停止与回滚

F3/API 未冻结、Contractor DTO 含客户身份或页面自行推进 SchoolTarget/Case 时停止。回滚 UI/client；不删除已完成 Task 或历史回执。

