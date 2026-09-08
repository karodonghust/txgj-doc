# P3-BE-05 逐校申请与 Task 事件闭环

状态：`ready_after_P2`  
Owner：`backend`  
依赖：P2 全部 Local gate 通过

返回[开发票据索引](README.md)。

## 业务结果

每所确认学校独立推进；进入 preparing 时为指定 Advisor 生成“准备并提交申请”Task，需要面试时生成独立面试辅助 Task。

## 范围

- SchoolTarget 逐校状态、结果、受控转换和 Cases milestone 重验。
- `preparing` 原子创建一条 application Task，由指定 Advisor 自己准备并提交申请。
- 需要面试时由 Advisor 选择辅助面试人，可由 Advisor 或 Contractor 担任，并创建 interview Task。
- Task/Assignment 生命周期、接受/拒绝/重派/取消/逾期事实和三类完成回执。
- application Task 完成必须有提交结果与证据引用；interview Task 完成不推进学校结果。
- 重复事件、Worker 重启和并发 claim 幂等；Case 暂停继续计算截止与风险。

## 不在范围

- 不单独建立 Application/Interview 实体；不处理文件字节、通知投递或外部 Email。

## 验收

- 每个 SchoolTarget 同类型 active Task 最多一个；重试不重复创建。
- Application Assignee 只能是 Primary Advisor 或有案件授权的 Advisor Collaborator；Contractor 只能承担 interview Task。
- 完成、拒绝、重派和取消保留 Assignment/receipt 历史；非法跨学校证据拒绝。
- 全部学校拒绝仍不自动结案；添加学校与 Founder 人工结案分支保持可用。
- Local PostgreSQL/HTTP/失败注入新增 `test:p3-be-05`，复用现有 Task workflow tests；typecheck/architecture 通过。

## 证据、停止与回滚

数据库/HTTP 是必需证据，browser 交独立测试，Preview/AWS `not_run`。发现重复 Task、Contractor 取得 Case 权限、面试完成推进学校状态或暂停停止风险计算时停止。已完成 Task/Assignment 不删除、不倒退，只用新事件纠正。

