# P4-FE-05 Notifications 入口

状态：`ready_after_F4_approval_and_P4-BE-07`  
Owner：`frontend`  
依赖：P1-FE-01、F4 线框批准、P4-BE-07 API 可用

返回[开发票据索引](README.md)。

## 业务结果

Advisor 与 Founder 能看到最小化站内待办提醒；点击后由目标模块重新授权，不从通知本身获得资料访问权。

## 范围

- Notification 列表、未读数、已读和点击后重新授权。
- 九类 effect 使用固定最小文案；不显示客户、学校、Task、Case 或文件资料。
- loading/empty/denied/unavailable/error 状态；suppressed/failed 不对用户展示。

## 不在范围

- 不发送 Email/SMS/WhatsApp；不在客户端计算接收人、逾期或去重。

## 验收

- 同一用户多角色不会显示重复同义提醒；目标失权后显示统一 denied。
- unread count 只是投影，不决定 Case/Task 权限或状态。
- 桌面/移动/键盘/屏幕阅读器通过；focused tests、typecheck、新增 `test:p4-fe-05` 和 Local browser 独立验收通过。

## 停止与回滚

F4/API 未冻结、DTO 包含 PII/业务正文或页面需要信任 route target 授权时停止。回滚 UI/client，不删除 Notification/DeliveryReceipt 历史。
