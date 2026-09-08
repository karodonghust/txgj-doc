# 天星国际 Release 1 需求阅读版

> 本文把现有已接受或已修订的需求决策重新编排为便于业务审阅的简体中文版本。
> 它不新增需求、不替代决策台账，也不证明功能已经实现。

| 字段 | 内容 |
| --- | --- |
| 文档状态 | `reader_copy_amended` |
| 阅读对象 | 项目接手人、业务负责人、后续开发人员 |
| 用户确认日期 | 2026-08-17 初次确认；2026-08-24 采纳 `DEC-071` K12 案件流程，由 `DEC-073` 恢复只提供站内通知，由 `DEC-074` 收口通知、Assessment 和推进中案件计数，并由 `DEC-075` 补充 Student 性别资料规则 |
| 当前目标 | K12 案件流程已确认；继续逐项审阅公共与业务模块，并记录新需求与旧实现的差异 |
| 当前执行边界 | 只开发本地、确定性合成数据版本；不操作真实数据、云端或生产环境 |
| 需求口径 | 按用户决定，以当前已存在且状态为 accepted/amended 的文档为需求基线 |
| 不包含 | Java 对照、实现教程、代码完成度声明、未经批准的新业务规则 |

> `DEC-071`/`DEC-073`/`DEC-074` 的完整流程、状态定义、Task 触发、通知、Assessment 和计数规则见 `K12_CASE_AND_SCHOOL_APPLICATION_WORKFLOW.zh-CN.md`；`DEC-075` 的 Student 性别资料规则以决策台账为准。本阅读版只提供摘要；冲突时以流程文档和决策台账为准。

## 1. 这份文档怎么读

正式决策文档按 `DEC-001`、`DEC-002` 等编号记录规则，适合追踪和开发，但不适合第一次了解产品。本文按业务使用顺序重新组织这些规则，同时保留原决策编号，方便以后追到代码和测试。

建议一次只审阅一个章节。每章可以给出以下三种结论：

- `确认`：描述符合你对业务的理解。
- `需要修改`：业务实际做法与文档不一致，需要先修改需求决策。
- `需要解释`：概念或流程尚未理解，先讨论，不进入开发。

本文中的三类信息需要区分：

| 类型 | 含义 |
| --- | --- |
| 已批准需求 | 产品应具备的业务行为，开发时必须遵守 |
| 当前本地边界 | 眼下允许开发和验证的范围，不代表未来生产范围被取消 |
| 未来或受门禁约束 | 方向可能已确定，但真实启用、云端操作或不可逆行为仍未获授权 |

主要来源：

- `../PRD_IMPLEMENTATION_DECISIONS.md`：当前产品与实施决策台账。
- `../PRD_PHASE_IMPLEMENTATION_PLAN.md`：需求依赖、状态机和验收计划。
- `../PHASE3_EMPTY_TENANT_PILOT_REVISION_PLAN.md`：空租户、历史案件重建和试点修订。
- `../decisions/R1X-DECISION-BASELINE-20260812.md`：Portal 与平台计数的当前本地实现基线。
- `../TAKEOVER_PHASE0_SCOPE_BASELINE.zh-CN.md`：2026-08-17 接手工作的本地执行边界。

当本文与上述来源冲突时，以用户最新明确决定及文档索引规定的权威顺序为准，并暂停受影响的开发。

## 2. 产品目标与成功标准

### 2.1 Release 1 要解决的问题

天星国际需要一个内部 K12 留学服务运营系统，把原本可能分散在表格、聊天和个人记忆中的学生、案件、选校、任务、文件、进度、权限和审批统一起来。

Release 1 的核心结果是：负责人无需在通信软件中逐人追问，也能看到每个案件目前处于什么阶段、下一步做什么、由谁负责、截止时间是什么，以及有哪些异常或待批准事项。顾问能在受控权限下推进案件，所有重要修改都有版本、来源和审计记录。`[DEC-001, DEC-040, DEC-041, DEC-057]`

### 2.2 面向不同人员的结果

| 人员 | Release 1 应提供的结果 |
| --- | --- |
| Founder / 管理员 | 查看所有试点案件的状态、责任人、下一任务、截止时间和异常；批准高风险操作；撤销权限；恢复到已知正确状态 |
| Primary Advisor / 主责顾问 | 创建并推进自己负责的 K12 案件，维护评估、学校目标、任务和文件 |
| Case Collaborator / 案件协作顾问 | 只在明确授权的案件、数据范围、操作能力和有效期内协作 |
| Contractor / 兼职或外包人员 | 只可处理明确分配给自己的面试辅助 Task，并只看到完成该单一任务必需的脱敏信息；不能因此查看完整案件或文件 |
| Data Reviewer / 数据审核员 | 根据证据审核学校资料变更，不能直接修改爬虫原始快照 |
| Guardian / Applicant / 家长或申请人 | 通过可撤销、会过期的只读入口查看一个被明确授权的案件；确认最终学校名单和 offer 决定 |
| Platform Finance / 平台财务角色 | 只查看不含个人信息的案件数量和合同参考数据，不因此取得租户业务数据权限 |

### 2.3 最终验收方向

未来真实交付必须先用合成场景证明行为确定、权限隔离、并发安全、可恢复；再交付不含业务数据的生产租户；之后由获授权客户用户逐案重建 1 至 3 个历史案件，观察至少 5 个营业日，经人工决定后才可扩大到总计 5 至 10 个受监控案件。`[DEC-031, DEC-057, DEC-061, DEC-062]`

当前接手阶段只交付本地合成数据版本，不执行上述真实数据和生产试点。

## 3. Release 1 做什么、不做什么

### 3.1 当前产品范围

- 邀请制内部账号、登录、会话、角色和案件级授权。
- Student、Guardian 及两者关系。
- K12 ServiceCase、版本化评估、案件阶段和逐校 SchoolTarget。
- Task、分派、受控流转、完成记录和审计；不要求 Founder 逐条验收。
- 私有文件上传、隔离扫描、版本、恢复和删除流程。
- 学校公开数据快照、临时学校、学校资料变更申请、审核后覆盖层和回滚。
- 站内通知、仪表盘投影和追加式审计。
- 受限的单案件只读 Portal。
- 不含个人信息的“推进中案件数量”和不透明合同参考值。
- 确定性合成场景、失败场景、迁移、回滚和恢复证据。`[DEC-001, DEC-003-DEC-046, DEC-050, DEC-064-DEC-066]`

### 3.2 明确不在 Release 1 中

- 非 K12 的正式业务流程。大学、硕士等只显示“正在开发中”，不能建案。
- 合作渠道登录。银行、保险等只作为 ReferralSource 记录。
- Portal 公开注册、写入、留言、上传、文件查看或下载。
- 发给 Guardian 或 Student 的外部业务 Email，以及所有 SMS、WhatsApp 自动通知。
- AI 对客报告、AI 自动决策和知识库业务流程。
- Excel/CSV 正式导入功能。
- 收费公式、应付金额、税务发票、付款、退款、催收、会计总账和收入确认。
- 自动把爬虫结果同步并部署到前端。
- 第二个正式组织、多租户商业启用和跨租户知识共享。
- 跨区域灾备和长期双写。`[DEC-002, DEC-003, DEC-025, DEC-033, DEC-047-DEC-049, DEC-060, DEC-064]`

### 3.3 当前只允许本地合成版本

眼下开发目标是可实际操作的本地 Release 1，使用合成身份、合成案件和非个人学校数据。本地环境可以模拟 PostgreSQL、对象存储、消息队列、文件扫描和角色登录，但不能接触真实 Student、Guardian、案件、文件或生产凭据。

未来 AWS 香港架构仍然是生产目标和安全约束，只是当前不执行云端创建、迁移、部署或生产启用。

## 4. 核心业务对象

### 4.1 Student 与 ServiceCase 必须分开

`Student` 表示一个真实的人，`ServiceCase` 表示一次具体服务委托。一个学生可以有多个案件，例如某次入学申请与下一学年的插班申请是不同案件。已取消或已结案后重新签约时，创建新案件，不重开或覆盖旧案件。`[DEC-004]`

同一 `学生 + 入学年份 + 申请类型` 最多只能存在一个未结束案件。每个案件同时拥有系统 UUID 和便于人工识别的案件编号。

Student 基础资料包含可空的 `gender`：`male`、`female`、`other`、`not_disclosed`。`null` 表示尚未收集，`not_disclosed` 表示明确不提供。建档时可以暂不填写；目标学校或申请表要求时，必须在进入该校申请准备前由有权员工补齐或人工处理无法直接映射的选项。系统不得根据姓名等资料推断，也不得仅凭学校性别政策自动移除候选学校。该字段属于 CRM，不重复进入正式 15 字段 Assessment。`[DEC-075]`

### 4.2 Guardian 是独立对象

一个 Guardian 可以关联多个兄弟姐妹，一个 Student 也可以关联多个 Guardian。关系中要记录法定监护状态、主要联系人、紧急联系人、账单联系人、通知同意和有效时间。`[DEC-005]`

每个学生至少有一名有效主要联系人，同一时间最多一名。邮箱或电话可以重复，只能提示疑似重复，不能据此自动合并人员。

### 4.3 不保存法定证件号码和影像

Release 1 不保存 HKID、内地身份证、护照号码或证件影像，只记录证件类型、是否已提供、到期日和材料状态。未来若确实需要保存，必须另行设计加密、遮罩、下载权限、审计、保留和删除流程。`[DEC-006]`

### 4.4 案件责任关系

每个活跃案件必须且只能有一个 Primary Advisor。案件转交必须一次性完成，并记录转出人、转入人、原因、时间和版本。Founder 可以暂代。`[DEC-008]`

协作顾问不是共同所有人，只通过限时授权获得指定范围。兼职或外包人员不能成为案件协作顾问，只能通过 TaskAssignment 获得单一任务所需的信息。

## 5. 用户、角色和权限

### 5.1 内部账号规则

- 不开放自行注册，由管理员邀请。
- 生产账号必须使用 MFA；身份系统只证明“这个人是谁”。
- 角色、案件归属、数据范围、操作能力和有效期由业务数据库在每次请求时判断。
- 停用账号必须立即停止业务访问，并撤销或使现有会话失效。
- 生产会话规则为 15 分钟空闲过期、8 小时绝对过期、最多 3 个活跃会话；第 4 次登录拒绝，不自动挤掉旧会话。高风险操作要求 5 分钟内重新完成 TOTP。`[DEC-007, DEC-020; OD-08]`

本地合成登录可以采用开发专用实现，但不得改变以上生产规则，也必须在非本地模式下拒绝启用。

### 5.2 协作顾问的数据范围

可授权范围只有以下预定义类别：

- `case_summary`：案件摘要。
- `education_profile`：教育背景。
- `school_targets`：目标学校。
- `task_workspace`：任务工作区。
- `communications`：沟通信息。
- `identity_contact`：身份和联系信息，默认不开放。
- `internal_notes`：内部备注，默认不开放。

每个范围分别授予 `view`、`comment` 或 `edit`。即使获得 `edit`，协作顾问也不能转交主责顾问、邀请其他协作者、推进或回退案件、批准最终选校、导出敏感资料、发布对客内容或删除正式历史。`[DEC-009, DEC-010]`

普通协作授权默认最长 7 天，不能超过案件结束时间。`identity_contact` 和 `internal_notes` 还必须由 Founder 批准并填写原因。到期、账号停用、移除协作者或案件结案时立即失效；续期必须创建新的授权决定。`[DEC-011, DEC-029]`

### 5.3 高风险操作

敏感导出需要 Founder 批准，使用一次性短效下载、watermark、香港存储和完整审计。Student、Guardian、ServiceCase 的删除先进入 `pending_delete`，通过批准、保留期、legal hold 和引用检查后才可清除。`[DEC-045, DEC-046]`

## 6. K12 案件主流程

Release 1 从客户已经在线下完成签约开始，不管理签约前的 Lead（潜在客户）、咨询和 Quote（报价），也不新增销售 Lead、Quote 或 Contract 实体。签约后才建立或关联 Student、Guardian 和 ServiceCase；需要留存的已签合同文件复用案件 Document。这里不指 Platform Billing 的合同参考版本。`[DEC-054, DEC-071]`

### 6.1 案件级里程碑

`ServiceCase` 不再使用旧八阶段线性流程，只记录整个案件的里程碑：

```text
signed
  -> background_collection
  -> school_selection_confirmed
  -> application_in_progress
  -> closed
```

- `signed`：保存已签约事实和起始里程碑；建立案件并指定 Primary Advisor 后不需要人工推进。
- `background_collection`：建案成功后自动进入。Assessment 从 `draft` 开始，由 Primary Advisor 维护；这一过程不自动创建 Task。
- `school_selection_confirmed`：Founder 已批准候选名单，Guardian 已确认同一版本并保存确认记录。
- `application_in_progress`：逐校申请正在处理；准备、提交、面试、候补和学校结果都属于 SchoolTarget。
- `closed`：只可由 Founder 在满足结案前提后人工执行。`[DEC-027, DEC-071]`

Assessment 的背景阻断项完成后记录 `background_complete`，此时允许 Advisor 建立候选学校名单；它是背景收集的完成门槛，不是进入 `background_collection` 的条件。现有“背景完成后才进入该状态”的实现与当前需求冲突。

进行中的案件必须能够标记为暂停，用于家长要求暂缓等暂时无法继续但不应结案的情况。只有尚未向任何学校正式提交申请，即没有 SchoolTarget 达到 `submitted` 或后续状态时，才允许暂停整个案件；一旦已有学校提交，申请方停止该校须按具体 SchoolTarget 记录 `withdrawn`。Primary Advisor 可直接暂停或恢复自己负责且符合条件的案件，不需要 Founder 事前审批；Founder 可暂停或恢复任何符合条件的案件。Guardian 只可提出请求，由有权员工执行。每次暂停必须填写简短自由文字原因，并保存操作者和操作时间；Release 1 不设固定原因分类。暂停不是结案，不覆盖暂停前里程碑，且暂停期间不得按正常流程继续推进。恢复后回到暂停前里程碑，已有未完成 Task 继续原记录，不自动取消、完成或重建。学校规定的申请截止日期和内部 Task 截止时间都继续计算且不得自动顺延，Task 可在暂停期间逾期；暂停期间继续向 Primary Advisor、Founder 和 Task 实际 assignee 发送内部逾期提醒，不发送给 Guardian。暂停的数据表示仍待技术设计。

### 6.2 选校、例外和结案

选校必须依次经过 Advisor 建立候选名单、Founder 批准或驳回修改、Guardian 确认同一版本并留档。名单变化后重新走 Founder 和 Guardian 两层确认，之后才进入申请处理。

Release 1 由案件 Primary Advisor 通过平台外人工电话、微信或面谈取得 Guardian 确认，再代录确认结果、时间、方式、操作人和对应名单版本。若有截图或文件则关联到案件；书面附件不是完成确认的强制条件。只读 Portal 不承担确认写入。

确认后修改学校名单必须创建新版本并重新走 Founder 批准和 Guardian 确认。不变 SchoolTarget 和 Task 保留；新增学校在新版确认后才创建申请 Task；移除且仍在进行中的学校记为 `withdrawn`，未完成 Task 取消但保留历史；已终态学校保留原结果。

所有 SchoolTarget 都处于终态且没有未完成 Task，只是允许结案的前提，不会自动结案。此时可以新增学校并重新选校，也可以由 Founder 明确结案并保存结果与原因。

客户终止整体服务时，进行中的 SchoolTarget 记为 `withdrawn`，相关未完成 Task 取消但保留历史，再由 Founder 人工结案并保存原因。以后重新签约必须建立新 ServiceCase，不重开旧案。`[DEC-004, DEC-071; amended OD-04]`

### 6.3 并发和重复请求

所有重要写入都携带当前版本。若两个人同时修改，后提交者得到 `409` 冲突，页面应展示当前版本与自己的修改差异，由用户重新确认，不能静默覆盖。相同请求被重复提交时，也不能重复创建案件、任务、通知或其他副作用。`[DEC-032, DEC-044]`

## 7. K12 正式 15 字段评估与旧 16 字段页面

### 7.1 正式评估规则

正式 K12 评估不是一张永远不变的表，而是由四层版本化模块组合：

1. K12 基础模块。
2. 教育阶段模块，例如幼儿园、小学或中学。
3. 学校体系模块，例如香港本地或香港国际学校。
4. 申请路径模块，例如起始年级或插班。`[DEC-012, DEC-026]`

建案时固定所用的评估清单和模块版本。以后升级规则时，不能静默改变旧案件的答案或必填条件。

### 7.2 字段值与完整度

每个答案要保存字段和模块版本、更新人、更新时间、来源和可见性。缺失信息必须区分：

- `unknown`：当前不知道。
- `not_applicable`：此案件不适用。
- `declined_to_provide`：当事人拒绝提供。

不能用空字符串、“暂无”或普通 `null` 混淆这些业务含义。`[DEC-013, DEC-043]`

Draft 阶段只要求最小身份、申请分类、目标入学年份和主责顾问。进入“背景收集完成”和“选校确认”时分别检查明确的阻断字段，不能只看一个完成百分比。

### 7.3 正式采用 15 字段 v1

旧页面的 16 字段展示不再作为业务契约。Release 1 正式采用现有已批准 manifest 的 15 字段：学生资料 3 项、教育背景 3 项、选校偏好 5 项、家庭情况 4 项。精确 field ID、类型、enum 和 blocker 清单见完整流程文档第 4 节；字段规则变更必须建立新 manifest version，不能静默改变旧案件。

Primary Advisor 可读写；有 `education_profile:view/edit` 明确 grant 的 CaseCollaborator 按 capability 只读或编辑；Founder 和 Admin 只读。其他 Advisor、Contractor、Data Reviewer、Guardian、Student 和 Portal 默认不可查看原始 Assessment。`unknown` 与 `declined_to_provide` 不满足 blocker；`not_applicable` 只有 manifest 明确允许时才满足，当前 v1 没有此类 blocker 例外。`[DEC-012, DEC-013, DEC-074]`

## 8. 学校资料与逐校申请

### 8.1 SchoolTarget 与案件阶段分离

一个案件可以申请多所学校。先由 Advisor 建立候选名单，Founder 审批，再由 Guardian 确认同一版本；确认后每所学校用独立 SchoolTarget 跟踪：

```text
candidate -> preparing -> submitted
submitted -> interview（需要时）
submitted / interview -> waitlisted（非终态）/ accepted / rejected
waitlisted -> accepted / rejected
accepted -> offer_confirmed / offer_declined
preparing / submitted / interview / waitlisted -> withdrawn
```

不需要面试的学校可以跳过 `interview`。`waitlisted` 仍可能变化，不能结案；`accepted` 只表示学校发出 offer，必须等待 Guardian 接受或拒绝。只有 `offer_confirmed`、`offer_declined`、`rejected` 和 `withdrawn` 是终态。

案件总阶段只是摘要，不能虚构或覆盖逐校事实。同一案件可能同时有录取、拒绝、候补和撤回结果。每个学校结果和 Guardian offer 决定都要保存受控代码、日期、确认方式、来源、操作人和版本；offer 决定同样由 Primary Advisor 在通过平台外人工电话、微信或面谈取得确认后代录，有书面证据时关联到案件。`[DEC-027, DEC-058, DEC-071]`

标准 K12 路径中：

- 每所学校按当期官方要求维护自己的材料清单，不设置全校通用固定清单。
- `preparing -> submitted` 必须保存提交时间、渠道、提交人和清单完成状态，并保存学校参考号或至少一份其他提交凭证；学校不提供参考号时要明确记录并保留确认页、确认邮件 PDF、回执或邮寄凭证等证据。
- `submitted -> interview` 只在学校要求面试时发生，并需要邀请证据和面试时间。
- 学校结果和 Guardian offer 决定需要相应证据或可审计确认记录。
- 缺少规定证据时，状态转换必须被阻止。`[OD-05]`

### 8.2 临时学校

若学校不在现有目录，Advisor 可以创建 provisional School，填写最小身份、地区、体系、阶段和原因。状态依次为：

```text
provisional -> under_review -> verified -> retired
```

临时学校可以关联案件和资料搜集任务，但必须显示“未验证”，不能作为已核实事实对客展示，也不能推测官方 URL。`[DEC-014]`

### 8.3 学校资料变更

所有学校字段都可以提出变更申请，但不能直接改爬虫快照。申请要保存原值、建议值、理由、官方 URL、证据、申请人和当时的学校版本。普通字段由 Data Reviewer 审核；学校身份、合并、拆分、停用和主要官网变更由 Founder 审核；提交者不能审核自己。`[DEC-015]`

爬虫快照保持不可变，批准的人工修改作为 overlay 生效。选校方案固定引用当时看到的 resolved version，后续快照不能静默改变已批准方案。错误修改通过停用 revision 回滚，不改写历史。`[DEC-016]`

## 9. Task 任务工作流

Task 与案件阶段分开管理。本轮确认的自动创建规则是：

- SchoolTarget 进入 `preparing` 时，逐校指派 Primary Advisor 或具明确案件授权的 Case Collaborator，并自动创建一条与该校绑定的“准备并提交申请”Task。
- 该 Task 只有在申请已经提交，并保存提交时间、渠道、提交人、清单完成状态，以及参考号或其他提交凭证后才能完成。
- 学校要求面试时，指派面试辅助人，并自动创建一条与该校绑定的面试辅助 Task；默认由 Primary Advisor 负责，也可以指派已成为 CaseCollaborator 的正式 Advisor 或通过单一 TaskAssignment 指派 Contractor。
- 背景资料收集不生成 Task；没有面试要求也不生成面试 Task。
- 自动创建必须幂等，不能因为重试为同一学校和同一轮申请重复建 Task。`[DEC-071]`

除申请和面试两类外，不增加其他自动 Task。Primary Advisor 可以按真实需要为自己负责的案件人工创建临时 Task，但人工 Task 不自动推进 ServiceCase 或 SchoolTarget。名单审批、Guardian 确认、等待结果、offer 决定和结案均不自动生成 Task；对应站内提醒按第 11.1 节执行，通知不等于 Task。

已确认名单移除学校或客户终止整体服务时，受影响的未完成 Task 取消但保留全部历史；暂停案件则继续保留原 Task，两种情形不得混淆。

通用 Task 生命周期已修订：

- 新任务从 `assigned` 开始，当前 Assignee 可以接受或填写原因拒绝；拒绝结束当前 TaskAssignment，Task 本身保留并等待重派。
- 拒绝或换人时，Primary Advisor 在同一条 Task 上重新分派；保留 Assignment 历史，新负责人重新接受，不另建 Task。
- `overdue` 是根据未完成 Task 的 `due_at` 计算的提示和提醒标记，不是阻断流程的独立状态；逾期后仍可完成。
- Assignee 保存对应完成记录和证据后即可完成；Founder 不逐条验收，`approved` 不再是完成后的必经状态。
- 所有命令仍检查权限和预期版本。拒绝、重新分派、取消和完成都保留操作人、原因或完成记录、时间和审计。`[DEC-028, DEC-071; amended OD-06]`

非 Primary Advisor 的面试辅助人只可查看面试 Task、目标学校、时间／方式／语言、辅导要求和 Primary Advisor 整理的必要背景摘要。不得查看完整 Assessment、Guardian／联系方式、内部备注、案件文件、下载／导出、其他学校或其他案件。Contractor 不因此成为 CaseCollaborator；拒绝 Task，或 Task 完成、取消、重派、案件结束即失去访问。

## 10. 文件工作流

### 10.1 上传、扫描和版本

文件内容放在私有对象存储，数据库只保存 metadata、关联、权限、版本和状态。文件没有公开 URL；下载或上传前必须重新检查服务端权限。`[DEC-017]`

文件版本的主要流程为：

```text
pending_upload -> quarantined -> scanning -> available
                                      -> rejected
                                      -> scan_failed -> bounded retry
```

替换文件时创建新版本，不能覆盖旧版本。只有已经扫描通过且未撤销的版本可以成为当前版本或被恢复。预览、下载、导出、删除和恢复都写入审计。`[DEC-017, DEC-024, DEC-030]`

### 10.2 删除和保留

- 文件默认有 30 天 soft-delete 恢复窗口。
- legal hold 期间不能清除文件，且 legal hold 不自动过期。
- `identity_and_case_evidence` 在案件结案后保留 7 年。
- `operational_attachment` 在案件结案后保留 2 年。
- 未关联案件的 `temporary_upload` 30 天后到期。
- 未分类文件拒绝进入正式保留流程。`[DEC-030; OD-02]`

Portal 在 Release 1 中不能查看或下载文件。

申请文件只能由 Founder、案件当前 Primary Advisor 和该校 Application Assignee 下载，并且 Application Assignee 只可访问该校申请必需的已扫描可用文件。仅负责面试辅助的人不能下载。提交凭证复用现有 SchoolTarget、Task、Case Document 和 audit／version，不新增 Material 或 SubmissionEvidence 实体。`[DEC-017, DEC-028, DEC-071]`

## 11. 通知、审计和仪表盘

### 11.1 站内通知

Release 1 只提供站内通知，不向 Guardian 或 Student 发送外部业务 Email，也不发送 SMS 或 WhatsApp。Primary Advisor 通过平台外人工电话、微信或面谈取得 Guardian 的名单或 offer 确认后在系统代录；Email 和只读 Portal 都不是 Release 1 的确认写入渠道。

Task 分配／重派通知 Assignee，拒绝通知 Primary Advisor；候选名单待审通知 Founder，审批结果和待 Guardian 确认通知 Primary Advisor；Task 到期前 3 天和 1 天提醒 Assignee 与 Primary Advisor，逾期后每天提醒 Assignee、Primary Advisor 与 Founder；所有学校终态且没有未完成 Task 时提醒 Primary Advisor 与 Founder 选择新增学校或人工结案。暂停期间提醒继续，同一接收人／effect／日期去重。文案只表达“有待办事项”，不披露学生、家长、案件、学校或文件信息。`[DEC-023, DEC-033, DEC-071, DEC-073, DEC-074]`

### 11.2 审计

状态转换、授权、批准、导出、删除、恢复和高风险读取都写入追加式审计。业务修改与必需审计必须在同一数据库事务中完成：若审计无法保存，修改也必须失败。`[DEC-032, DEC-062]`

租户业务审计与平台财务审计分开保存。平台人员不能因为需要审计或统计而获得 Student、Guardian、案件内容、文件或内部备注的读取权。`[DEC-066]`

审计日志保留 1 年，一般应用日志和产品技术遥测基线保留 30 天。日志与遥测不得包含个人资料、文件内容、评估答案、原始密钥、Cookie、URL 查询、表单自由文本或屏幕录制。`[DEC-039, DEC-062]`

### 11.3 仪表盘

Founder 应能查看案件阶段、下一任务、负责人、截止日期、异常和待批准项。仪表盘是从权威业务数据生成的可重建视图，不能反过来成为修改业务真相的快捷方式。`[DEC-052, DEC-057]`

## 12. 家长／申请人只读 Portal

### 12.1 Portal 的定位

PortalViewer 不是内部员工账号，也不是内部 Cognito 角色。一个访问授权只绑定一个组织、一个查看者和一个 ServiceCase，不能因为同一 Guardian 有多个孩子而自动看到兄弟姐妹或其他案件。`[DEC-064, DEC-065; DP-01-DP-05]`

Founder 或该案件当前 Primary Advisor 可以创建、轮换或撤销授权。原始访问密钥只显示一次，不能进入 URL、日志、遥测、截图或证据文件；数据库只保存安全哈希和 fingerprint。

授权必须指定到期时间，最长 7 天，不能原地延期。一个授权最多产生 3 个活跃 Portal 会话；会话 15 分钟空闲过期，绝对有效期不超过 8 小时或授权到期时间中的较早者。第 4 个会话拒绝。撤销、轮换或到期会立即使相关会话失效。

### 12.2 可以看到的内容

`portal_case_read_v1` 只允许：

- 案件编号。
- 已批准的对客案件阶段。
- 最近一次对客可见更新时间。
- 已批准对客显示的学校名称和 SchoolTarget 状态。
- 对客行动项的标题、截止时间和完成状态。
- 明确标记为 `customer_visible` 的消息。

默认拒绝：文件和下载、评估答案、联系信息、内部备注、审计、协作者、Advisor 私人资料、价格合同、导出、评论、编辑和删除。新增字段必须经过批准并升级 allowlist 版本，旧授权不会自动扩大。`[DP-04]`

### 12.3 自动拒绝条件

每次读取都重新检查授权、会话、Guardian/Student 关系、案件、组织、签发者权限、能力版本和当前时间。案件结案或待删除、关系失效、签发者失去权限、组织失效、授权撤销或到期时立即拒绝；恢复关系或重开案件也不会自动复活旧授权。`[DP-03]`

## 13. 平台案件计数与合同参考值

### 13.1 当前只统计，不计费

当前本地 Release 1 只计划保存版本化的“推进中案件数量”，不实现收费公式。旧 `DP-06` 的六阶段映射已经被取代：`advancing_case_count_v1` 计入 `background_collection`、`school_selection_confirmed` 和 `application_in_progress`；暂停案件以及所有学校均拒绝但尚未结案的案件继续计入。`signed`、已记录整案终止但尚待结案、`pending_delete` 和 `closed` 不计入。

一个案件在香港月末 cutoff 最多计 0 或 1，不按天数或学校数折算、不 prorate；迟到或修正事件通过新 policy/snapshot version 重建，不能静默改写已批准快照，也不能由该计数生成应付金额。`[DP-06, DEC-071, DEC-074]`

### 13.2 合同值不是应付金额

系统可以保存使用整数最小货币单位和 ISO 4217 币种的不可变合同版本，但 `contract_value_minor` 只是“不透明合同参考值”，不能解释为月费、单案价格、税基或可计算收费。`[DP-07]`

当前不生成 payable amount，不编号、不渲染、不批准、不发送发票或收费通知。系统最多提供 `calculation_unavailable` 的内部预览，显示所引用的合同版本和案件计数。

### 13.3 平台角色与业务隔离

`platform_finance` 可以创建合同草稿和非金额月度预览；不同的 `platform_billing_approver` 才能激活、取代或终止合同版本。创建人与批准人不能相同。平台角色不能读取租户业务明细或把自己加入租户。`[DP-08, DEC-066]`

订阅状态在 Release 1 中只作信息展示。`past_due` 不得影响 CRM、Portal、文件、任务、登录或导出权限；`suspended` 和 `terminated` 的业务效果不可用。第二个正式组织仍被明确阻止。`[DP-09, DP-11, DEC-060]`

## 14. 爬虫学校数据的产品边界

前端消费的学校发布包必须同时包含以下四个文件：

- `records.json`
- `review_queue.json`
- `run_summary.json`
- `publish_manifest.json`

候选发布必须验证 manifest、schema、文件集合、记录数量、哈希和 warnings。有 warning 时不能自动接受、同步或发布，需要 Data Reviewer 建议和 Founder 对精确 manifest 及 warning 列表进行一次性批准。`[DEC-050; OD-12]`

爬虫发布、复制快照到前端、Git 提交/推送、应用部署是四个独立动作，任何一个动作的批准都不能自动授权其他动作。当前 Release 1 不做自动同步。

## 15. 不可削弱的质量和隐私规则

- 所有敏感数据的未来生产存储、处理、日志、临时副本和备份都必须位于香港；不能用香港对象存储掩盖其他环节在区外处理。`[DEC-018-DEC-024]`
- 任何缓存、搜索、统计、AI 索引和仪表盘都只是可重建派生数据，不能成为授权来源或业务真相。`[DEC-052]`
- 所有模块只能通过自己的公开接口修改所拥有的数据，不能跨模块直接写表。`[DEC-056]`
- 重要操作需要稳定错误代码、request ID、明确是否可以重试、幂等键、版本冲突和追加式审计。`[DEC-032, DEC-044]`
- 权限测试必须覆盖页面、直接 API、猜 ID、搜索、导出、文件、后台任务和缓存，不能只依赖“页面上没有按钮”。`[DEC-053, DEC-057]`
- 必须测试空数据、权限不足、长文本、加载、错误、并发、重复请求、部分失败、回滚和恢复。
- 产品遥测故障可以进入明确降级状态继续业务，但强制审计故障时业务写入必须失败。`[DEC-062]`

当前本地模式使用合成数据验证这些规则。它不能为了方便而放松生产约束，也不能在非本地模式下自动回退到 Mock、Neon 或本地服务。

## 16. 一条完整业务流程示例

下面的例子用于把前述需求串起来，不新增规则：

1. Founder 邀请 Advisor，Advisor 完成登录和 MFA；本地开发阶段使用明确隔离的合成登录。
2. Advisor 创建 Student，并为其建立一个 K12 ServiceCase，填写入学年份、申请类型并成为 Primary Advisor。
3. Advisor 关联 Guardian，确认当前主要联系人。
4. 建案成功后案件自动进入 `background_collection`。系统按案件类型固定评估 manifest，Assessment 从 `draft` 开始，Advisor 直接填写背景信息；这一步不生成 Task。
5. Advisor 建立候选学校名单。若学校不在目录中，先建立 provisional School，并提交证据等待审核。
6. Founder 批准或驳回名单；批准后，Primary Advisor 通过平台外人工电话、微信或面谈取得 Guardian 对同一版本的确认并代录。名单修改后重新走两层确认。
7. Guardian 确认后，案件进入 `school_selection_confirmed` 和申请处理。每所确认学校独立进入 `preparing`，系统为其指派负责人并自动创建一条“准备并提交申请”Task。
8. Application Assignee 按该校官方要求维护材料清单并上传申请材料。文件先隔离并扫描；正式提交后保存时间、渠道、提交人、清单完成状态，以及参考号或其他凭证，申请 Task 才能完成，SchoolTarget 进入 `submitted`。
9. 学校要求面试时，系统指派面试辅助人并自动创建辅助 Task；不需要面试时跳过。默认由 Primary Advisor 辅助，也可指派获案件授权的 Advisor 或只取得该 Task 脱敏工作区的 Contractor。
10. 学校给出候补时继续等待；给出 offer 后由 Guardian 决定接受或拒绝，Primary Advisor 取得确认后代录；拒绝和申请方撤回也分别保存，不合并成案件结果。
11. 如需让家长查看进度，Founder 或当前 Primary Advisor 创建最长 7 天的只读 Portal 授权。当前只读 Portal 不承担名单或 offer 确认写入。
12. 所有状态变化、确认、授权、文件操作和高风险读取都进入审计。重复请求不产生重复副作用，并发冲突不会覆盖他人修改。
13. 当所有学校都处于终态且没有开放 Task 后，团队选择新增学校，或由 Founder 人工结案；系统永不自动结案。
14. 平台按 `background_collection`、`school_selection_confirmed`、`application_in_progress` 统计推进中案件；暂停和全部学校已拒绝但尚未结案仍计入，其余明确排除状态不计入。系统不计算或发送应付金额。

## 17. 开发前需要重点审阅的事项

现有文档已经给出大量规则，但以下内容最容易与真实业务理解发生偏差，应在开始对应功能前明确确认：

| 审阅项 | 当前文档口径 | 为什么需要你确认 |
| --- | --- | --- |
| 产品边界 | Release 1 是内部 K12 运营核心，外加受限只读 Portal 和平台案件计数 | 决定我们第一版究竟服务哪些人、哪些流程 |
| Student 性别资料 | CRM 保存可空 `gender`；建档可不填，学校要求时在该校申请准备前补齐或人工处理；不自动推断或自动排除学校 | 业务规则已确认；当前表、API、页面和测试尚未实现 |
| 新案件里程碑 | 建案后自动进入 `background_collection`；Assessment 完成后允许建立候选名单，再进入后续里程碑 | 业务口径已确认；旧状态和既有数据迁移仍需设计 |
| 案件暂停与恢复 | 仅在尚无 SchoolTarget 正式提交时允许暂停；Primary Advisor 可直接暂停或恢复自己负责且符合条件的案件，Founder 可暂停或恢复任何符合条件的案件，Guardian 只可提出请求；暂停填写自由文字原因；恢复回到暂停前里程碑并继续原 Task；截止日期不顺延 | 数据表示仍待技术设计，业务规则已确认 |
| 15 字段正式评估 | 采用现有已批准 v1 manifest；Primary Advisor／获授权 Collaborator 可编辑，Founder／Admin 只读；阻断语义按 `DEC-074` | 业务规则已确认；旧 16 字段展示、当前角色写权限和 blocker 判定仍需差异修改 |
| SchoolTarget 与选校确认 | Founder 批准后由 Guardian 确认同一名单版本；Primary Advisor 代录；确认后改校建立新版本，不变目标/Task 保留，新增目标确认后建 Task，进行中的移除目标 `withdrawn` 并取消未完成 Task，已终态目标保留原结果 | 业务规则已确认，数据表示仍待技术设计 |
| 自动 Task 与验收 | 申请准备/提交和必要面试各自动产生一条逐校 Task；背景收集不产生；Assignee 可接受/拒绝，同 Task 重派，逾期只作提示，完成不再逐条等待 Founder 审批；Primary Advisor 可人工建临时 Task | 业务规则已确认，旧状态机和页面仍需差异修改 |
| 申请材料与下载 | 每校按官方要求维护清单；提交保存时间、渠道、提交人、清单状态和参考号或其他凭证；Founder、Primary Advisor、该校 Application Assignee 可下载必要文件，面试辅助人和 Portal 不可下载 | 业务规则已确认，精确字段和授权实现仍需差异设计 |
| 外部 Email | Release 1 不向 Guardian 或 Student 发送；SMS/WhatsApp 也不包含 | 若未来需要启用，必须作为新范围重新确认，开发不得自行接通 |
| Portal 可见内容 | 只显示案件阶段、学校进度、行动项和对客消息，不显示文件 | 决定家长入口是否有实际价值，同时影响隐私风险 |
| 平台案件计数 | 计入三个进行中里程碑，暂停和全拒未结案继续计入；排除 signed、整案终止待结案、pending_delete、closed；不计算金额 | 业务映射已确认；旧 policy 和月底快照实现仍需差异修改 |
| 学校快照 warning | 必须人工审核后才能接受 | 决定数据更新速度与风险承担方式 |

这些审阅不是重新开展一次客户调研，也不是要求你现在设计全部细节；目的是确认现有文档没有明显偏离你准备接手的业务方向。

## 18. 逐章确认记录

| 轮次 | 审阅范围 | 当前状态 | 结论或修改入口 |
| ---: | --- | --- | --- |
| 1 | 第 2-3 章：产品目标与 Release 1 边界 | `已确认` | 2026-08-17，未提出修改 |
| 2 | 第 4-5 章：核心对象、用户和权限 | `已确认` | 2026-08-17，未提出修改 |
| 3 | 第 6-7 章：案件阶段与评估 | `后被部分修订` | 2026-08-17 初次确认；案件阶段于 2026-08-24 被第 7 轮修订，评估结构不因此自动改变 |
| 4 | 第 8-10 章：学校、任务和文件 | `后被部分修订` | 2026-08-17 初次确认；SchoolTarget、Task 流转、面试 Contractor 脱敏工作区、逐校材料/提交证据和申请文件下载角色于 2026-08-24 被第 7 轮修订 |
| 5 | 第 11-14 章：通知、Portal、平台计数和爬虫 | `后被部分修订并已确认` | 2026-08-17 初次确认；2026-08-24 由 `DEC-073` 固定站内通知边界，`DEC-074` 固定通知触发／节奏和新案件计数；Portal 和爬虫规则未在本轮修改 |
| 6 | 第 15-17 章：质量规则、完整示例和重点问题 | `已确认` | 2026-08-17，确认作为后续工作依据 |
| 7 | K12 案件、选校、逐校申请、自动 Task、offer 决定、结案、通知、Assessment 与计数 | `已确认` | 2026-08-24 形成 `DEC-071`、`DEC-073`、`DEC-074`；流程文档第 12 节列出的 K12 客户业务问题已全部确认，后续只剩实现差异设计与验证 |
| 8 | 公共与业务模块：CRM Student 基础资料 | `审阅中` | 2026-08-24 形成 `DEC-075`，确认 Student 性别字段及申请前校验；CRM 其余对象继续逐项审阅 |

第 7 轮已修订第 3、4、5 轮中的部分旧口径。第 8 轮开始逐项审阅公共与业务模块。本文仍保持“阅读版”身份，不替代决策台账和完整流程文档；在相应差异计划完成前，不选择受影响开发切片。
