# P5-FE-06 Guardian Portal 工作区

状态：`ready_after_F5_approval_and_P5-BE-08`  
Owner：`frontend`  
依赖：P1-FE-01、F5 线框批准、P5-BE-08 API 可用

返回[开发票据索引](README.md)。

## 业务结果

家长通过临时入口进入安静、清晰的单 Case 只读页面；无论入口为何失效，都得到一致且不泄露信息的结果。

## 范围

- `/portal/access` fragment 读取后立即清理、一次兑换、独立 Portal Cookie。
- `/portal/workspace` 只渲染 allowlist 的阶段、学校进度、消息和行动项。
- invalid/expired/revoked/denied/unavailable 使用同一外部页面；logout 清理 Portal Session。
- 内部 `/cases/:id/access` 提供 Primary Advisor 创建/重签和 Advisor/Founder 撤销控制。

## 不在范围

- 不显示 Case number、Guardian/Student 联系方式、文件、内部备注、Task assignee、Billing 或业务写入；不提供内部导航。

## 验收

- raw secret 不进入 history、analytics、持久浏览器存储、错误或截图证据。
- 刷新、返回、多个 tab、过期、撤销、第四 Session 和 offline/unavailable 行为安全。
- 桌面/移动/键盘/屏幕阅读器通过；focused tests、typecheck、新增 `test:p5-fe-06` 和独立 Local browser 验收通过。

## 停止与回滚

F5/API 未冻结、页面需要持久化 raw key、DTO 含非白名单字段或与内部 Session 混用时停止。回滚 Portal UI/client，并由 Backend 撤销新 Grant/Session；不删除历史。

