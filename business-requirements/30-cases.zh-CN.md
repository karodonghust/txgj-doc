# Cases

返回[唯一事实源索引](README.md)。

## BR-030 ServiceCase 边界与里程碑

状态：`confirmed`

Release 1 从客户在线下完成签约开始。签约后建立或关联 Student、Guardian 和 ServiceCase，并指定 Primary Advisor。已签合同文件如需保存，复用 Case Document。

`ServiceCase` 只表达案件全局里程碑：

```text
signed
  -> background_collection
  -> school_selection_confirmed
  -> application_in_progress
  -> closed
```

- `signed`：保存签约事实和起始里程碑。
- 建案并指定 Primary Advisor 后自动进入 `background_collection`，不要求人工阶段命令。
- Assessment 从 `draft` 开始；背景 blocker 完成后记录 `background_complete`，允许建立候选学校名单。
- Founder 批准名单且 Guardian 确认同一版本后达到 `school_selection_confirmed`。
- 至少一所已确认学校开始申请处理后进入 `application_in_progress`。
- `closed` 只能由 Founder 人工执行。

逐校准备、提交、面试、候补、录取、拒绝和撤回只由 SchoolTarget 表达，不作为整个案件的线性阶段。

## BR-031 案件暂停与恢复

状态：`confirmed`

- 仅在没有 SchoolTarget 达到 `submitted` 或后续状态时允许案件级暂停。
- Primary Advisor 可暂停或恢复自己负责且符合条件的案件；Founder 可操作任何符合条件的案件。
- Guardian 只能提出请求，由有权员工执行。
- 暂停必须填写简短自由文字原因并保存操作者和时间。
- 暂停不是结案，不覆盖暂停前里程碑；暂停期间不得按正常流程继续推进。
- 恢复后回到暂停前里程碑，已有 Task 使用原记录，不自动完成、取消或重建。
- 外部申请截止日期和内部 Task 截止时间继续计算，不自动顺延；逾期提醒继续。
- 已有学校正式提交后，停止某校申请必须记录该 SchoolTarget 为 `withdrawn`，不能用案件暂停代替。

## BR-032 Assessment

状态：`confirmed`

正式 K12 Assessment 使用不可变的 15 字段 v1 manifest：

| 分组 | 字段和取值 | `background_complete` blocker | `school_selection_confirmed` blocker |
| --- | --- | --- | --- |
| `student_profile` | `date_of_birth: date`；`residency_status: hk_permanent_resident/hk_non_permanent_resident/dependent_visa/other`；`primary_languages: cantonese/mandarin/english/other`（多选） | 3 项全部 | `date_of_birth`、`residency_status` |
| `education_profile` | `current_stage: kindergarten/primary/secondary`；`current_year_level: text`；`current_curriculum: hk_local/ib/cambridge/other` | 3 项全部 | `current_stage`、`current_year_level` |
| `school_preferences` | `target_stage: kindergarten/primary/secondary`；`preferred_systems: hk_local/hk_international`（多选）；`preferred_districts: hong_kong_island/kowloon/new_territories/any`（多选）；`preferred_admission_route: entry/transfer`；`fee_band: government_aided/private/international/undecided` | `target_stage`、`preferred_admission_route` | 5 项全部 |
| `family_context` | `primary_contact_language: cantonese/mandarin/english/other`；`education_priority: academic/balanced/language_immersion/supportive_environment/other`；`transport_arrangement: family_transport/school_bus/public_transport/undecided`；`fee_preference: government_aided/private/international/undecided` | `primary_contact_language`、`education_priority` | 4 项全部 |

- Primary Advisor 可读写。
- 有 `education_profile:view/edit` 明确授权的 Case Collaborator 按 capability 只读或编辑。
- Founder 只读。Admin 基础角色默认不可查看；Admin 只有同时取得 Advisor 角色并满足 Primary Advisor 或 Case Collaborator 等案件关系时，才按对应 Advisor 权限访问。
- 其他 Advisor、Contractor、Guardian、Student 和 Portal 默认不可查看原始 Assessment。
- `unknown` 与 `declined_to_provide` 不满足 blocker。
- `not_applicable` 仅在 manifest 明确允许时满足；当前 v1 没有 blocker 例外。
- 字段或枚举变化必须建立新 manifest version，不能静默改变旧案件。

## BR-033 候选学校名单确认

状态：`confirmed`

固定顺序：

```text
Primary Advisor 建立候选名单
  -> Founder 批准或驳回修改
  -> Guardian 确认 Founder 已批准的同一版本
  -> 进入逐校申请处理
```

- Founder 驳回或 Guardian 不确认时退回修改；修改版本必须重新经过 Founder 批准和 Guardian 确认。
- Guardian 确认必须保存名单版本、学校集合、确认人、结果、时间、渠道、代录人和 Founder 对同一版本的批准记录。
- Release 1 由 Primary Advisor 通过平台外人工电话、微信或面谈取得确认后代录。
- 有书面截图或文件时关联 Case；附件不是完成确认的强制条件。
- Portal 保持只读，不承担确认写入。
- 已确认名单变化时建立新版本；不变 SchoolTarget 和 Task 原样保留，新增学校确认后才进入 `preparing` 并创建申请 Task。
- 移除仍在进行中的已确认学校时记为 `withdrawn`，取消其未完成 Task 但保留历史；已终态学校保留原结果。

## BR-034 SchoolTarget 状态

状态：`confirmed`

```text
candidate -> preparing -> submitted
submitted -> interview                         （需要面试）
submitted / interview -> waitlisted / accepted / rejected
waitlisted -> accepted / rejected
accepted -> offer_confirmed / offer_declined
preparing / submitted / interview / waitlisted -> withdrawn
```

| 状态 | 含义 | 终态 |
| --- | --- | --- |
| `candidate` | 候选，未开始正式申请 | 否 |
| `preparing` | Guardian 已确认，正在准备 | 否 |
| `submitted` | 已提交且提交事实和凭证完整 | 否 |
| `interview` | 学校要求面试 | 否 |
| `waitlisted` | 候补，仍可能变化 | 否 |
| `accepted` | 学校发出 offer，等待 Guardian 决定 | 否 |
| `offer_confirmed` | Guardian 接受 offer | 是 |
| `offer_declined` | Guardian 拒绝 offer | 是 |
| `rejected` | 学校拒绝 | 是 |
| `withdrawn` | Student/Guardian 主动停止该校申请 | 是 |

不需要面试的学校跳过 `interview`。Guardian 对 offer 的决定也由 Primary Advisor 通过人工渠道取得后代录并保存确认记录。

## BR-039 人工结案与重新签约

状态：`confirmed`

结案必须同时满足：

1. 所有已确认 SchoolTarget 都处于 `offer_confirmed`、`offer_declined`、`rejected` 或 `withdrawn`。
2. 不存在 `preparing`、`submitted`、`interview`、`waitlisted` 或 `accepted`。
3. 不存在未完成 Case Task。
4. Founder 明确结案并保存结案结果、原因、操作者和时间。

满足条件也不得自动结案。所有学校都拒绝时必须保留两条分支：新增候选学校并重新走两层确认，或由 Founder 明确无 offer 结案。有 `offer_confirmed` 可记录成功结案，没有时仍可人工结案，但不得改写逐校结果。

客户终止整体服务时，进行中的 SchoolTarget 记为 `withdrawn`，相关未完成 Task 取消并保留历史，再由 Founder 人工结案。以后重新签约必须创建新的 ServiceCase。

ServiceCase 不允许删除或软删除，只能按照业务结果取消服务并人工结案。ServiceCase 及其状态、SchoolTarget、Task 和相关历史记录必须永久保留在数据库中，不得执行物理删除。
