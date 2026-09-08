# P4-FE-04 Documents 工作台

状态：`ready_after_F4_approval_and_P4-BE-06`  
Owner：`frontend`  
依赖：P1-FE-01、F4 线框批准、P4-BE-06 API 可用

返回[开发票据索引](README.md)。

## 业务结果

授权员工能看清上传与扫描进度，且不会把未 clean 文件误认为可用。

## 范围

- `/documents` 和 Case documents 视图、上传进度、扫描状态、download capability、soft-delete/restore 提示。
- quarantined/scanning/rejected/scan_failed/unavailable/pending_delete/deleted 状态完整。
- 页面不显示 bucket、object key、预签名 URL、扫描原始输出或客户信息通知正文。

## 不在范围

- 不生成外部 Email；不让 Portal/Contractor/Admin 单独浏览文件；不在客户端判定 clean。

## 验收

- 未 clean 文件不显示成功下载/证据选项；capability 过期可安全重取。
- 上传失败、收据失败、扫描失败和 runtime unavailable 不显示成功。
- 桌面/移动/键盘/焦点/进度稳定性通过；focused tests、typecheck、新增 `test:p4-fe-04` 和 Local browser 独立验收通过。

## 停止与回滚

F4/API 未冻结、页面需持久化 raw capability 或客户端自行把文件标成 clean 时停止。回滚 UI/client，不撤销或删除 Document 历史。
