# P1-OPS-02 测试数据库与失败注入

状态：`ready_after_P1-OPS-01`  
Owner：`platform-operations`  
依赖：P1-OPS-01、相关 Backend focused tests 已提供

返回[开发票据索引](README.md)。

## 业务结果

并发、撤销、重复投递和基础依赖失败能在隔离环境重复验证，不依赖历史输出或手工修改数据。

## 范围

- 建立每次可重建的隔离 PostgreSQL 17 测试目标和合成 fixture。
- 执行 stale version、同/异 payload 幂等、Audit/Outbox 失败、RLS pool reuse、权限撤销和锁超时场景。
- 为后续 P2-P5 提供同一失败注入入口、证据格式和清理边界。

## 所有权边界

Platform Operations 只执行批准脚本并保存脱敏证据；测试/应用/migration 源码缺陷返回对应 owner，不现场修补。

## 验收

- 每个场景可从空数据库重复运行，成功/失败状态和回滚后的权威表计数确定。
- 两连接并发、organization 隔离和连接归还清理通过。
- 结果逐项标记 `passed/failed/not_run`；browser、Preview、AWS 不由本票据证明。

## 停止与回滚

目标不是隔离合成数据库、清理目标不精确、发现 PII/secret 或失败后状态未知时停止。清理隔离数据库需使用票据记录的精确目标，不得使用宽泛目录/实例范围。

