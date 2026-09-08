# CRM

返回[唯一事实源索引](README.md)。

## BR-020 Student

状态：`confirmed`

`Student` 表示学生客户主档，不是登录账号，也不是某一次申请或案件。同一 Student 可以有多个 ServiceCase；已取消或结案后重新签约时创建新 Case，不重开或覆盖旧 Case。

最小业务资料：

| 字段 | 规则 |
| --- | --- |
| `display_name` | 必填 |
| `date_of_birth` | 可空 |
| `contact_email` | 可空 |
| `contact_phone` | 可空 |
| `gender` | 可空；`male`、`female`、`other`、`not_disclosed` |
| `status` | `active`、`pending_delete`、`deleted`（软删除） |

`gender` 规则：

- `null` 表示尚未收集；`not_disclosed` 表示当事人明确不提供。
- 不得根据姓名、照片、学校或其他资料推断。
- 它属于 CRM，不得重复保存为 Assessment answer。
- 建档时可不填；目标学校或申请表要求时，必须在进入该校申请准备前补齐可用值。
- `other` 无法映射到学校选项时必须人工处理，不能静默改成 `male` 或 `female`。
- 学校性别政策可以产生提示或人工复核，不能自动移除 SchoolTarget 或自动拒绝申请。

Student 使用不可变 UUID 识别。姓名、生日、邮箱、电话和性别都不是身份键、唯一条件或自动合并依据。Release 1 不在 Student profile 保存 HKID、内地身份证、护照号码或证件影像。

新建 Student 时，只有当现有 Student 的非空姓名、Email 或电话至少一项与新资料相同时，才显示疑似重复警告；出生日期或其他字段相同不触发警告。该警告只供人工判断，不得自动关联或合并记录。

Release 1 不提供 Student 的人工合并或撤销合并。

## BR-021 Guardian 基础资料

状态：`confirmed`

`Guardian` 表示家长或监护人本人，是独立客户对象，不是内部 User，也不把“父亲／母亲”等与特定 Student 的关系直接保存为本人属性。

最小业务资料：

| 字段 | 规则 |
| --- | --- |
| `display_name` | 必填 |
| `email` | 可空 |
| `phone` | 可空 |
| `gender` | 可空；`male`、`female`、`other`、`not_disclosed` |
| `date_of_birth` | 可空，保存出生日期而不是年龄 |
| `status` | `active`、`pending_delete`、`deleted`（软删除） |

- `email` 和 `phone` 至少填写一个。
- `gender` 的 `null` 表示尚未收集，`not_disclosed` 表示本人明确不提供；不得根据姓名、称谓、照片或其他资料推断。
- 年龄仅在需要显示时，根据 `date_of_birth` 和当前日期计算整岁，不建立或保存 `age` 字段。
- Guardian 使用不可变 UUID 识别。姓名、出生日期、年龄、性别、Email 和电话都不是身份键、唯一条件或自动合并依据。
- 新建 Guardian 时，只有当现有 Guardian 的非空姓名、Email 或电话至少一项与新资料相同时，才显示疑似重复警告；出生日期或其他字段相同不触发警告。该警告只供人工判断，不得自动关联、建立 Student 关系或合并 Guardian。
- Release 1 不提供 Guardian 的人工合并或撤销合并。

## BR-022 Student 与 Guardian 关系

状态：`confirmed`

Student 与 Guardian 使用独立关系记录连接：

| 字段 | 规则 |
| --- | --- |
| `student_id` | 关联 Student |
| `guardian_id` | 关联 Guardian |
| `relationship_type` | 使用下方受控选项 |
| `relationship_description` | 可空；选择 `other` 时必填简短说明 |
| `is_legal_guardian` | 是否为法定监护人 |
| `is_primary_contact` | 是否为当前主要联系人 |
| `is_emergency_contact` | 是否为紧急联系人 |
| `is_billing_contact` | 是否为账单联系人 |
| `notification_consent` | 是否同意接收允许范围内的通知 |
| `starts_at` | 关系版本生效时间 |
| `ends_at` | 关系版本结束时间，可空 |

`relationship_type` 受控选项：

```text
parent
father
mother
step_parent
stepfather
stepmother
adoptive_parent
adoptive_father
adoptive_mother
foster_parent
foster_father
foster_mother
grandparent
paternal_grandfather
paternal_grandmother
maternal_grandfather
maternal_grandmother
adult_sibling
adult_brother
adult_sister
uncle
aunt
court_appointed_guardian
institutional_guardian
other_relative
non_relative_guardian
other
```

关系规则：

- 一个 Student 可以关联多个 Guardian；一个 Guardian 也可以关联多个 Student，以支持兄弟姐妹共享同一 Guardian。
- 主要联系人是机构处理该 Student 相关事务时，日常优先沟通的 Guardian；`is_primary_contact` 只表示联系优先级。
- 主要联系人身份不自动代表法定监护权、紧急联系人、账单联系人、最终选校确认权、通知接收同意或 Portal 访问权；这些资格必须分别按照各自规则判断和记录。
- CRM 不设置“次要联系人”身份或排序。除唯一主要联系人外，其余均为关联 Guardian；法定监护、紧急联系、账单和通知等职责继续使用各自独立标志。
- 每个 Student 同一时间必须且只能有一个有效的主要联系人。
- 更换主要联系人时，关闭旧的当前关系版本并建立新版本；Guardian 关系和历史记录不得删除或覆盖。
- Guardian 的性别不能自动决定 `relationship_type`；关系类型必须由有权员工明确选择。
- `is_legal_guardian` 与 `relationship_type` 分开记录；父亲、母亲或其他亲属身份不自动等于当前法定监护权。

## BR-023 Student 与主要联系人建档

状态：`confirmed`

- 创建 Student 时，必须在同一业务操作中建立一名当前主要联系人，不允许留下已建档但没有主要联系人的 Student。
- 主要联系人可以是本次新建的 Guardian，也可以由有权员工手动选择已有 Guardian，以支持兄弟姐妹共享同一 Guardian。
- 系统不得根据姓名、Email、电话、出生日期或其他资料自动选择或关联已有 Guardian。
- Student、Guardian（如本次新建）及 Student-Guardian 关系必须以单一原子操作保存；任一部分失败时全部回滚。
- 首次建档完成后，可以再为 Student 添加其他 Guardian 关系。

## BR-024 添加关联 Guardian

状态：`confirmed`

- Student 完成首次建档后，可以新建 Guardian 并建立关系，也可以由有权员工手动选择已有 Guardian 建立关系。
- 新增关系必须明确填写 `relationship_type`；选择 `other` 时仍须遵守 `BR-022` 的说明填写规则。
- 新增关联 Guardian 默认不是主要联系人，不得改变当前主要联系人；如需更换主要联系人，必须另行执行主要联系人交接。

## BR-025 主要联系人交接

状态：`confirmed`

- 新主要联系人必须从该 Student 当前有效的关联 Guardian 中选择；尚未关联的 Guardian 必须先按 `BR-024` 建立关系。
- 主要联系人交接必须以单一原子操作完成，任何失败都不得造成同时存在多个主要联系人或没有主要联系人。
- 交接必须保留新旧关系版本历史，不得覆盖原记录。
- 旧主要联系人在交接后继续作为关联 Guardian，不自动解除或终止其与 Student 的关系。

## BR-026 Guardian 关系解除

状态：`confirmed`

- 解除关系只结束 Guardian 与该 Student 的当前关联，不删除 Guardian 本人资料。
- 关系结束时间和历史版本必须保留，不得物理删除或覆盖。
- 当前主要联系人不能直接解除关系；必须先按 `BR-025` 完成主要联系人交接。
- 关系解除后，该 Guardian 不再作为该 Student 的当前关联 Guardian 或联系人。

## BR-027 客户来源

状态：`confirmed`

- Release 1 保留独立 `ReferralSource` 来源目录，用于统一保存公司实际使用的各种客户来源。
- ReferralSource 由 CRM 管理；每条来源至少保存显示名称、`BR-028` 的受控来源类型、可空说明和 active/inactive 状态。
- 每个 ServiceCase 最多关联一个当前 ReferralSource；同一个 ReferralSource 可以关联多个 Case。
- 客户来源不保存在 Student 上；同一 Student 的不同 Case 可以关联不同来源。
- 只有 active ReferralSource 可以用于新建或变更 Case 来源；来源停用后，既有 Case 关联和历史不得删除或改写。
- Case 更换来源时必须保留旧关联历史，不能直接覆盖而丢失原来源。
- ReferralSource 使用不可变 UUID 关联；显示名称、类型和说明不能充当关联键。
- 来源类型必须使用 `BR-028` 的受控选项，不得由开发自行增加枚举值。

已取代规则：此前“Release 1 不建立独立 ReferralSource、直接在 ServiceCase 保存来源类型和说明”的决定状态为 `superseded`，由本规则取代。

## BR-028 客户来源类型

状态：`confirmed`

客户来源使用以下受控选项：

| 值 | 含义 |
| --- | --- |
| `customer_referral` | 客户转介绍 |
| `employee_referral` | 员工介绍 |
| `school_referral` | 学校推荐 |
| `partner_referral` | 合作方推荐 |
| `website` | 官网 |
| `social_media` | 社交媒体 |
| `paid_advertising` | 付费广告 |
| `event` | 活动 |
| `walk_in` | 主动到访 |
| `other` | 其他 |
| `unknown` | 未知 |

每条 ReferralSource 必须有非空显示名称；选择 `other` 时，来源说明必填。

## BR-029 客户资料软删除

状态：`confirmed`

- 有权处理该客户资料的 Advisor 可以提交 Student 或 Guardian 删除申请；申请后状态变为 `pending_delete`。
- Founder 负责批准或驳回；Founder 也可以直接发起申请并作出决定。
- Admin 和 Contractor 不参与客户资料删除申请或决定。
- Student 仍有未结案 ServiceCase 时，不允许提交或批准软删除。
- Guardian 仍有任何当前有效的 Student 关系时，不允许提交或批准软删除；主要联系人必须先完成交接，其他当前关系必须先解除。
- Founder 批准时必须重新检查上述条件；条件已不满足时不得把状态改为 `deleted`。
- Founder 驳回后，状态恢复为 `active`；Founder 批准后，状态变为 `deleted`。
- `deleted` 只表示软删除。Student、Guardian 及其关系记录必须继续保存在数据库中，业务代码和数据库均不得对这些记录执行物理删除或最终 purge。
- 本规则明确取代此前使用的 `purged` 状态和“满足条件后最终 purge”规则；后续开发不得再使用旧规则。
- `pending_delete` 记录仍显示在正常客户列表和搜索结果中，但必须明确标记为“待删除审批”。
- `deleted` 记录在 Release 1 中完全不可供业务查看；业务页面、列表、搜索、筛选、详情和业务接口均不得返回这些记录，也不提供“已删除”列表或筛选。
- Release 1 不提供 Student 或 Guardian 的软删除恢复功能。
- `deleted` Student 不得用于新建 Case；`deleted` Student 或 Guardian 不得用于建立新的 Student-Guardian 关系。
- 删除申请、批准、驳回和状态转换必须保留审计记录。
