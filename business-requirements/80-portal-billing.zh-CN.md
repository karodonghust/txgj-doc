# External Portal 与 Platform Billing

返回[唯一事实源索引](README.md)。

## BR-060 Guardian Portal 核心边界

状态：`confirmed`

- 家长不注册内部系统账号；Portal 查看者不是内部 User，也不取得内部角色。
- 当前 Primary Advisor 可以为家长生成临时、只读、仅对应一个 ServiceCase 的查看入口。
- 家长只能查看机构已批准对客公开的申请进度，不能因同一 Guardian 关联多个 Student 而自动看到其他 Case。
- 家长不能通过 Portal 修改资料、上传文件或直接确认选校；确认仍由 Advisor 沟通后代录。
- Portal 不承担 Guardian 的名单确认、offer 决定或其他写入；由 Primary Advisor 取得确认后代录。
- Portal 仅可展示：Case 当前对客阶段和对客更新时间、已批准展示的学校名称及申请进度、由 Advisor 明确标记为家长可见的消息和行动项。
- Portal 不得展示文件、Assessment、联系方式、内部备注或审计记录；任何未列入白名单的字段默认不可见。

## BR-063 Guardian Portal 授权

状态：`confirmed`

- 只有当前 Primary Advisor 可以创建 Portal 授权；当前 Primary Advisor 可以撤销，Founder 可以紧急撤销。
- 每个 Portal 入口有效期为 7 天，不能在原入口上延期；到期后如仍需访问，由当前 Primary Advisor 重新生成。
- 授权被撤销或到期后，访问必须立即失效。
- 会话数量、空闲超时、Token 形式及安全传递方式属于安全技术设计，不作为客户业务规则；技术设计不得削弱单 Case 绑定、有效期和可撤销性。

## BR-061 推进中案件计数

状态：`superseded`

以下旧设计已被 `BR-003` 的 Release 1 排除决定取代，只保留历史追溯；Release 1 不得实现或依赖该计数：

`advancing_case_count_v1`：

- 计入 `background_collection`、`school_selection_confirmed`、`application_in_progress`。
- 暂停案件以及所有现有学校均拒绝但尚未结案的案件继续计入。
- 排除瞬时 `signed`、已记录整体终止但尚待结案、`pending_delete` 和 `closed`。
- 一个案件在香港月末 cutoff 最多计 0 或 1，不按天数或学校数折算，不 prorate。
- 迟到或修正事件创建新 policy/snapshot version，不静默改写已批准快照。
- 该计数不能生成应付金额。

## BR-062 合同参考值与订阅

状态：`superseded`

以下旧设计已被 `BR-003` 的 Release 1 排除决定取代，只保留历史追溯；Release 1 不得实现 Platform Billing：

- `contract_value_minor` 是带 ISO 4217 币种的不透明合同参考值，不是月费、单案价格、税基或应付金额。
- Release 1 不计算、编号、渲染、批准或发送发票和收费通知。
- `platform_finance` 可建立合同草稿；不同的 `platform_billing_approver` 才能激活、取代或终止，创建人与批准人不能相同。
- 平台角色不得读取租户 Student、Guardian、Case、文件或内部备注。
- `past_due` 只作信息展示，不影响登录、CRM、Portal、文件、Task 或导出权限。
- `suspended` 和 `terminated` 的业务效果保持不可用。
