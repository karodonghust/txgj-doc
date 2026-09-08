# P4-BE-07 站内通知与提醒调度

状态：`ready_after_P3`  
Owner：`backend`  
依赖：P3-BE-05、Shared/Audit/Outbox 可用

返回[开发票据索引](README.md)。

## 业务结果

Advisor 与 Founder 在系统内收到必要任务和风险提醒；不向家长或学生发送业务 Email。

## 范围

- 九类已批准 effect、接收人在投递时实时解析、站内 Notification 与 DeliveryReceipt。
- Task 到期前 3 天、1 天和逾期每日提醒，按香港业务日期计算。
- 逾期接收人：实际 Assignee、Primary Advisor、Founder；暂停期间继续计算与提醒。
- 同 recipient/effect/source/业务日期或版本去重；suppressed/failed/dead-letter 状态。
- 点击通知只携带安全 route code/opaque target，目标页面重新授权。

## 不在范围

- 不发送外部 Email/SMS/WhatsApp；不在通知正文存姓名、学校、Task、文件或 Case 信息。

## 验收

- 重复投递/Worker 重启/并发 claim 返回原 receipt，不生成第二条 Notification。
- 角色重叠不重复提醒；撤销 Assignment/角色后旧接收人不再获得新通知。
- due-at 变更、完成、取消、暂停、时区边界和 dead-letter 重放通过。
- 95% effect 在 60 秒内投递，5 分钟 backlog 产生脱敏告警。
- 新增 `test:p4-be-07`，复用 notification/outbox tests；typecheck/architecture 通过。

## 证据、停止与回滚

Local Worker/PostgreSQL/HTTP 是必需证据；外部消息 provider、Preview、AWS `not_run`。发现外部 Email、PII payload、重复通知或暂停停止逾期计算时停止。已投递历史不删除；禁用新 producer/consumer 并用审计重放修正。

