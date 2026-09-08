# P2-BE-04 候选名单两层确认与 Case 流程

状态：`ready_after_P2-BE-03`  
Owner：`backend`  
依赖：P2-BE-03、Schools 已批准公开契约

返回[开发票据索引](README.md)。

## 业务结果

Primary Advisor 建立候选学校名单，Founder 先审核，Advisor 再代录家长确认；只有同一版本完成两层确认后才进入逐校申请准备。

## 范围

- CandidateListVersion + CandidateListItem，不另建“当前名单”实体。
- Primary Advisor 建立/修改新版本；Founder approve/reject；Primary Advisor 保存 Guardian 确认记录。
- Guardian 确认绑定同一 approved version，保存 actor、时间和受控确认记录；家长不直接登录内部系统操作。
- approved snapshot/revision pin、SchoolTarget 初始化与 Case milestone 重验。
- paused/termination_pending/closed 和人工结案门禁；全部学校拒绝不自动结案。

## 不在范围

- 不生成 Tasks，不提交学校申请，不实现面试、文件、Portal 或外部 Email。

## 验收

- 未经 Founder approve 不接受 Guardian 确认；版本变更后旧确认不适用。
- reject 后只能创建/修改新版本；历史版本、决定和确认不可改写。
- 重试只生成一组 SchoolTarget；非法状态转换和 stale version 稳定拒绝。
- 全部 Target 终态时 Case 保持开放，返回“添加新学校或 Founder 人工结案”两个分支。
- Local PostgreSQL、HTTP、并发与 RLS 测试新增 `test:p2-be-04`；architecture/typecheck 通过。

## 证据、停止与回滚

Local database/HTTP 为必需证据；browser/Preview/AWS 保持 `not_run`。发现家长确认未留记录、跨版本复用确认或自动结案时停止。纠正只追加新版本/迁移，不改历史决定。

