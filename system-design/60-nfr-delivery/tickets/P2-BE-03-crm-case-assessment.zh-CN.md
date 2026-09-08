# P2-BE-03 CRM、建案与 Assessment

状态：`ready_after_P1`  
Owner：`backend`  
依赖：P1 全部 Local gate 通过

返回[开发票据索引](README.md)。

## 业务结果

Advisor 可以建立/维护 Student、Guardian 和关系，选择客户来源并建立 Case；Assessment 不完整时明确阻塞后续选校。

## 范围

- Student/Guardian/Profile、唯一 Primary Guardian、关系版本与手工关联。
- 姓名/邮箱/电话相同时只警告，不自动关联、不建立合并工作台。
- ReferralSource 独立目录：Founder 管理；Primary Advisor 为自己 Case 选择/变更并保留历史。
- Student/Guardian soft-delete 审批：Advisor/Founder 申请，Founder 决定；deleted 永久隐藏且不 purge。
- ServiceCase 建立、Primary Advisor assignment、Assessment 15 字段与 blocker。
- 所有 command 使用 expected version、Shared 幂等、Audit/Outbox 和 organization RLS。

## 不在范围

- 不实现候选名单/SchoolTarget、Task、Document 或 Portal；不创建 Lead/Quote/Contract 实体。

## 验收

- 原子建立 Student + Primary Guardian；关系交接/解除保留完整历史。
- 重复警告只匹配姓名、邮箱或电话，用户可继续建立独立记录。
- 同一 Case 最多一个当前来源；inactive 来源不可新关联；旧关联保留。
- 有未结案 Case 的 Student、仍有有效关系的 Guardian 不能删除；跨模块锁顺序符合 NFR。
- Case 建立进入 background，Assessment blocker/版本冲突/租户隔离/Audit 回滚通过。
- 复用现有 CRM/CASE focused HTTP/PostgreSQL tests，并新增 `test:p2-be-03`；`pnpm typecheck` 通过。

## 证据、停止与回滚

Local PostgreSQL/HTTP 必须 `passed`；browser 交独立测试，Preview/AWS `not_run`。发现自动关联、物理删除、跨模块私表写入或来源历史覆盖时停止。已用 migration 只通过 forward corrective migration 回滚。

