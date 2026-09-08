# P1-FE-01 Shared typed client 与工作台外壳

状态：`ready_after_F1_approval_and_P1-BE-02`  
Owner：`frontend`  
依赖：F0 已批准、F1 Architect/项目负责人确认、P1-BE-02 可用

返回[开发票据索引](README.md)。

## 业务结果

内部用户通过一致的导航、页面状态和错误反馈进入工作台；页面不再各自猜测 API shape 或权限。

## 范围

- 建立统一 typed client/envelope decoder、request-id 和安全错误映射。
- 落地 workspace shell、角色/capability 导航、Today 基础区和 Notifications 入口壳。
- 统一 loading、empty、denied、stale、unavailable、error、retry 状态。
- Admin/Contractor 导航按批准边界显示；服务端拒绝仍是最终依据。
- 使用现有 i18n 与 icon 体系，覆盖键盘、焦点、响应式和屏幕阅读器语义。

## 不在范围

- 不实现各业务完整表单；不发明 endpoint/DTO/权限；不以 fixture 作为正式成功状态。

## 验收

- 页面不直接手写业务 fetch shape，不从客户端 role/cache 推导授权。
- 401/403/404/409/429/503 和 malformed envelope 均有稳定且不泄密的状态。
- 桌面与移动 viewport 无重叠、溢出或不可操作控件；关键路径键盘可达。
- focused component/static check、`pnpm typecheck` 和新增 `test:p1-fe-01` 通过；Local Dev browser happy/denied/unavailable 交给现有独立测试工作流。

## 停止与回滚

F1 未批准、Backend DTO 不稳定或页面需要绕过服务端授权时停止。回滚仅撤回本票据 UI/client 变更，保留共享 API 契约和用户无关改动。

