# Schools 现状差异复核（2026-10-10）

返回[现状分析总览](README.md)。唯一业务判定来源为 [BR-051（学校目录与数据变更）](/Users/karo/Documents/txgj-doc/business-requirements/50-schools.zh-CN.md)，现行基线 **BR-BASELINE-20261010-v60（基线）**。只使用仍有效的 `confirmed` 条文；历史原文、设计文档、旧 PRD 和既往分析不是判定标准。

分析日期：2026-10-10。业务仓库 main 快照：`032e02351f7549e72d2e1e52388bd8226e6596a9`。代码仓库 Tianxingguoji main 快照：`76a88eb2f853d8b4b6b5ea5d62a6aed93db915fc`，已核对远端 main 相同。分析对象是该提交的代码，不代表其已部署版本。

## 新旧关系与证据边界

[2026-08-25 学校分析](50-schools.zh-CN.md) 对应 v32、代码 `a0c862f34a88`，全文保留。本报告更新其**学校模块在现行基线下的结论**；不修改旧文、不更新其余八个模块，也不把本次子项数量加进旧总览的 35 条 BR 统计。

旧报告“学校正式入口均 unavailable”的概括不适用于当前全部路径：目录、最低建档、人工变更、人工审批、详情读取已有直接使用 PostgreSQL 仓储的 v1 路由。旧的治理/恢复 getter 仍固定抛出 unavailable；学校选项在 `production-aws` 模式下也仍拒绝组合。两者必须分别陈述。没有找到当前主目录直接读取 mock 的证据，不能沿用旧报告“学校页面使用 mock”的概括。

本轮是只读静态调查，没有运行应用、执行自动化测试、浏览器流程、Preview 或生产验证，也没有发起抓取。检查覆盖：`modules/schools` 的 38 个文件（领域、应用、基础设施及导出入口）；学校相关迁移及固定历史引用约束；学校/管理/兼容爬虫页面、组件和路由；相关单元、集成、迁移及浏览器断言资产；`contracts/school-crawl-handoff/v2` 的 schema、说明、样本与 TypeScript 校验器。契约目录 55 个文件包含 52 个样本（11 合法预期、41 非法预期）；**本轮没有执行其校验，不把预期或旧运行输出计为本轮结果**。不检查或修改 school-tracker 的现行实现。

沿用本目录状态词汇：`符合`、`部分符合`、`冲突`、`缺失`、`超范围残留`。下文“缺失”表示在上述检查范围内未找到对应的可执行业务路径；任意 JSON 能容纳字段、页面按钮名称相近或契约样本能表达，不足以证明运行路径存在。“冲突”和“超范围残留”是可定位的静态规则差异，不宣称已在生产触发。

验证层级继续区分：

```text
源码/迁移存在 → 自动化测试存在 → 本地运行验证
            → 浏览器流程验证 → Vercel Preview 验证 → AWS 生产验证
```

表中 `S` = 源码/迁移证据，`T` = 自动化测试资产存在。它们只是下文的阅读缩写，不新增验收状态。所有条目的后四层均为**证据不足，需补充运行验证**。因此对有结构、有测试但缺运行闭环的正向条目用“部分符合”，不写成“已实现”。设计阶段只在引用挂起项时沿用：阶段 A（契约与样本）、阶段 B（接入预检）、阶段 C（后台及数据库）、阶段 D（页面与 Cases）、阶段 E（真实小样本联调）；它们不构成本票的实施授权。

## 有效规则的核对口径

- v55 已允许首次一次性 EDB 全量导入、免额外逐校可用审批及逐校周期抓取；这不等于后续更新自动启用。人工添加/修改来源仍须审批，首次 EDB 自动带入网址是受限例外。“未验证”标记已不应在界面展示。
- D4（学校对象粒度）在 v56 确认：`scrn` 前六位界定学校，校址、时段、学段不拆校，稳定内部编号继续保留。N2（同校招生记录身份与跨批次对应）也已确认，不再把两者当挂起前置。
- v57 学年标准为 `YYYY/YY`；v58 以页面词汇执行本地年级归一化，取代学校级学制门槛，枚举/区间含义已确认。国际词汇保持原样且不与本地 P/S 映射。
- v59 已选择双语/版本受限例外；仅该范围修改 N2，普通 URL 编码、大小写及等价变化仍不是新身份。例外的证据和后续对应仍有四项挂起问题。
- 权限交叉核对引用 [BR-015（K12 试用等级与授权）](/Users/karo/Documents/txgj-doc/business-requirements/10-identity-access.zh-CN.md:109) 的 `confirmed` 试用成员边界及 [BR-011（员工资料）](/Users/karo/Documents/txgj-doc/business-requirements/10-identity-access.zh-CN.md:62) 的 Data Reviewer 排除规则：显式等级成员的 Founder/L1 业务权限不能仅据旧 Founder-only 文句判为错误；试用权限不与旧角色取并集。本报告不刷新 Identity 模块，不把一般业务等级视为每种特殊来源/退场操作已经获确认，也不替挂起的具体操作资格作决定。当前 v1 人工审批仓储限定 Founder/L1，不能据旧领域类型推断其正在允许 Data Reviewer 审批。

本报告把一条 BR-051 拆为 **110 个核对项**，用于逐条定位仍有效的确认规则及其排除边界；相同要求的历史重述只算一次。表内序号是报告行号，不是新 BR 编号。学校字段 V1 的 49 个字段在附表逐一核对，但已经归入第 14–18 项，不再重复进入总数。28 个挂起项单列，不分配五种结论、不进入完成度或验收分母。v60 的登记纪律作为本次调查边界，不虚构为需要新增运行功能的第 111 项。

| 状态 | 核对项数 |
| --- | ---: |
| 符合 | 0 |
| 部分符合 | 57 |
| 冲突 | 10 |
| 缺失 | 39 |
| 超范围残留 | 4 |
| 合计 | 110 |

## 可复核证据索引

下表短码只用于交叉引用源码位置，不是业务或设计编号。源码链说明见矩阵，避免把一个 migration 的存在当作完整能力。

| 索引 | 证据位置及本次能证明的范围 |
| --- | --- |
| E01 | [最低建档服务](/Users/karo/Documents/Tianxingguoji/modules/schools/application/service.ts:176)、[最低建档仓储](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-provisional-repository.ts:15)、[最低建档路由](/Users/karo/Documents/Tianxingguoji/app/api/v1/schools/provisionals/route.ts)、[建档迁移](/Users/karo/Documents/Tianxingguoji/db/migrations/202609190130_070_expand_provisional_school_intake.sql)。名称至少一个、生成 UUID、追加建档事实、审计/幂等及服务端资格检查。 |
| E02 | [目录路由](/Users/karo/Documents/Tianxingguoji/app/api/v1/schools/route.ts:13)、[目录仓储](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-directory-repository.ts:17)、[已解析资料读取事务](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-resolved-view-transaction.ts:147)、[详情路由](/Users/karo/Documents/Tianxingguoji/app/api/v1/schools/[schoolId]/resolved/route.ts)。主目录经权限校验读取当前 PostgreSQL 快照与修订，不回落磁盘或 mock；provisional 有单独列表。 |
| E03 | [人工变更仓储](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-change-repository.ts:77)、[人工变更路由](/Users/karo/Documents/Tianxingguoji/app/api/v1/schools/[schoolId]/change-requests/route.ts)、[有效值基线迁移](/Users/karo/Documents/Tianxingguoji/db/migrations/202609190140_071_capture_school_change_effective_baseline.sql)、[提交时旧值迁移](/Users/karo/Documents/Tianxingguoji/db/migrations/202609190160_073_capture_school_change_previous_value.sql)。字段候选、有效值 hash、旧值、来源摘录与审核前不生效。 |
| E04 | [人工审批仓储](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-review-repository.ts:18)、[人工审批路由](/Users/karo/Documents/Tianxingguoji/app/api/v1/admin/schools/change-requests/[changeRequestId]/reviews/route.ts)、[审批履历迁移](/Users/karo/Documents/Tianxingguoji/db/migrations/202609190150_072_record_school_change_reviews.sql)。当前 v1 路径复核 Founder/L1、禁止自审、事务内锁定和旧值检查、追加审批回执及解析版本。迁移仍保留旧 Data Reviewer 分支，见 E08。 |
| E05 | [快照及修订基础迁移](/Users/karo/Documents/Tianxingguoji/db/migrations/202608022030_004_expand_school_overlay.sql:1)、[领域解析器](/Users/karo/Documents/Tianxingguoji/modules/schools/domain/resolver.ts:78)、[固定版本事务](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-resolved-view-transaction.ts)。UUID、source key、不可变快照/字段/解析版本及 SchoolTarget 历史 pin 的基础；不是 EDB 对应、N2 或爬虫启用引擎。 |
| E06 | [详情四块及单套招生字段](/Users/karo/Documents/Tianxingguoji/components/schools/SchoolDetail.tsx:46)、[目录单条映射](</Users/karo/Documents/Tianxingguoji/app/(erp)/schools/page.tsx:105>)、[人工变更表单](/Users/karo/Documents/Tianxingguoji/components/schools/SchoolChangeForm.tsx)、[审核表单](/Users/karo/Documents/Tianxingguoji/components/schools/SchoolReviewForm.tsx)。四块和人工工作流组件存在；“重新載入”是查询刷新，不是抓取入口。 |
| E07 | [建档面板](/Users/karo/Documents/Tianxingguoji/components/schools/ProvisionalSchoolsPanel.tsx:50)。成功提示、标题、按钮及每校徽标仍显示“未验证”。 |
| E08 | [旧领域审核资格](/Users/karo/Documents/Tianxingguoji/modules/schools/domain/contract.ts:254)、[旧治理服务](/Users/karo/Documents/Tianxingguoji/modules/schools/application/governance-service.ts)、[旧恢复服务](/Users/karo/Documents/Tianxingguoji/modules/schools/application/resolved-view.ts:145)、[旧迁移审核/回滚资格](/Users/karo/Documents/Tianxingguoji/db/migrations/202609190150_072_record_school_change_reviews.sql:120)、[管理页旧说明](</Users/karo/Documents/Tianxingguoji/app/(erp)/admin/schools/page.tsx:64>)。普通字段 Data Reviewer、身份 Founder/L1、整 revision 回滚等旧分支仍在；旧 getter 不可用，与当前 v1 仓储分开核对。 |
| E09 | [整批 v1 快照装载](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/crawler/snapshot.ts:140)、[兼容映射](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/crawler/server.ts:39)、[旧学校 API](/Users/karo/Documents/Tianxingguoji/app/api/crawler/schools/route.ts)、[旧选校页面](</Users/karo/Documents/Tianxingguoji/app/(erp)/selector/page.tsx:36>)、[当前 main 保留的 v1 快照](/Users/karo/Documents/Tianxingguoji/data/crawler-source/latest/records.json)。单条 flat record、school_key、容量文字住宿 fallback、Boolean 特教、校验成功即成为内存 active。读取现有快照只用于核对字段含义，没有改动或重抓。 |
| E10 | [旧爬虫配置及可变决定](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/crawler/db.ts:6)、[兼容管理页面](</Users/karo/Documents/Tianxingguoji/app/(erp)/admin/crawler/page.tsx:88>)、[决定 API](/Users/karo/Documents/Tianxingguoji/app/api/crawler/review-decisions/route.ts)。全局 default 配置、单校 key 决定 UPSERT、旧 Admin/Data Reviewer 资格；没有对应的逐校任务/候选启用闭环。 |
| E11 | [单校 v2 草案说明](/Users/karo/Documents/Tianxingguoji/contracts/school-crawl-handoff/v2/README.md:5)、[schema](/Users/karo/Documents/Tianxingguoji/contracts/school-crawl-handoff/v2/contract.schema.json:1447)、[语义校验](/Users/karo/Documents/Tianxingguoji/scripts/validate-school-handoff-v2.ts:118)、[52 样本清单](/Users/karo/Documents/Tianxingguoji/contracts/school-crawl-handoff/v2/cases.json)、[双侧样本比较工具](/Users/karo/Documents/Tianxingguoji/scripts/test-school-handoff-v2.ts)。仍为 proposed、依据 v56；能表达多招生、未知/失败、逐字段证据与原文，未接入 Schools 运行时。 |
| E12 | [单元测试目录](/Users/karo/Documents/Tianxingguoji/tests/unit/schools)、[人工提交集成测试](/Users/karo/Documents/Tianxingguoji/tests/integration/school-change-workflow.test.ts)、[旧治理集成测试](/Users/karo/Documents/Tianxingguoji/tests/integration/school-governance-workflow.test.ts)、[解析事务测试](/Users/karo/Documents/Tianxingguoji/tests/integration/postgresql-resolved-school-transaction.test.ts)、[快照测试](/Users/karo/Documents/Tianxingguoji/tests/unit/crawler/snapshot-manifest.test.ts)。资产存在；其中事务测试使用测试协议/夹具，不能视为本轮真实数据库通过。 |
| E13 | [试用人工提交断言](/Users/karo/Documents/Tianxingguoji/tests/integration/trial-school-change-assertions.ts)、[试用审核断言](/Users/karo/Documents/Tianxingguoji/tests/integration/trial-school-review-assertions.ts)、[建档浏览器断言](/Users/karo/Documents/Tianxingguoji/tests/integration/trial-provisional-school-browser-assertions.ts)、[审核界面断言](/Users/karo/Documents/Tianxingguoji/tests/integration/trial-school-review-ui-assertions.ts)、[变更表单断言](/Users/karo/Documents/Tianxingguoji/tests/integration/trial-school-change-form-assertions.ts)。存在数据库/浏览器验证脚本，不是本轮已经运行的证据。 |
| E14 | [旧建档运行组合](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/runtime.ts:21)、[旧治理组合](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/school-governance-runtime.ts)、[旧恢复组合](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/resolved-view-runtime.ts)、[选项运行组合](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/school-options-runtime.ts:17)、[停用 revision 路由](/Users/karo/Documents/Tianxingguoji/app/api/v1/schools/[schoolId]/overlays/[overlayRevisionId]/disables/route.ts)、[协调路由](/Users/karo/Documents/Tianxingguoji/app/api/v1/admin/schools/[schoolId]/overlays/[overlayRevisionId]/reconciliations/route.ts)。旧 getter 抛错，不能将存在的路由当可运行恢复能力。 |
| E15 | [学校选项服务](/Users/karo/Documents/Tianxingguoji/modules/schools/application/school-options-service.ts)、[选项仓储](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-school-options-repository.ts)、[引用约束迁移](/Users/karo/Documents/Tianxingguoji/db/migrations/202608180110_027_enable_candidate_school_target.sql)、[引用边界测试](/Users/karo/Documents/Tianxingguoji/tests/migration/school-target-candidate-boundary.test.ts)。不可变版本引用基础；可选资格不据实现反推业务答案。 |

## 逐项差异矩阵

“出处”均指 BR-051 的现行条文章节及历史基线定位，不是以设计文档作授权。`S/T` 仅指证据到测试资产存在；`S` 后写“未找到”也是静态检查结论，不是运行验证。

### 建档、身份、人工治理与字段范围

| 序号 | 有效确认规则 / 出处 | 状态 | 依据层级与界限 |
| --- | --- | --- | --- |
| 001 | 目录无学校时允许 Advisor 建档；原始建档不可变（原规则、v55） | 部分符合 | S/T：E01、E12、E13 有建档路由、资格检查、追加记录；未运行。显示标记另见 082。 |
| 002 | 人工变更经 ChangeRequest，不改原始爬虫快照（原规则） | 部分符合 | S/T：E03–E05、E12、E13 有候选、审批及不可变触发器。 |
| 003 | 人工治理沿用审核边界，爬虫启用权限放宽不扩大人工治理（原规则、v39） | 部分符合 | S/T：E04 人工审批资格与 BR-015 试用范围有依据；合并、拆分及退场不能据类型补答案，依赖 D1（人工来源/纯外部键关联）、T2（发起与审批）、T6（停办与录错的差异）。旧资格残留见 110。 |
| 004 | 提交者不得自审（原规则、v49、v52） | 部分符合 | S/T：E04 同事务拒绝自审，E12/E13 有断言；旧 crawler 决定不是该路径。 |
| 005 | 批准修改以 overlay revision 生效（原规则） | 部分符合 | S/T：E03–E05 有候选转批准、追加解析版本。 |
| 006 | 错误修订通过停用 revision 回滚，不改写历史（原规则） | 部分符合 | S/T：E05/E08/E12 有领域回滚与历史约束，E14 路由仍依赖 unavailable；不等于逐字段恢复。 |
| 007 | 爬虫完成不能自动启用（原规则、v39、v55） | 冲突 | S/T：E09 校验成功立即将磁盘候选置为兼容内存 active，测试也要求此行为；没有普通用户独立启用步骤。这是兼容读取路径，不能声称当前 PostgreSQL 目录也自动启用。 |
| 008 | 页面可见用户可发起单校抓取并启用，服务端复核权限（v39） | 缺失 | S：E02 有读权限、E03 有人工变更权限，但未找到单校抓取/候选启用命令或路由。可用入口条件另依赖 D2（可用抓取入口组合）。 |
| 009 | 首版单校更新；除 v55 一次性初始导入外不扩大一般批量授权（v39、v55） | 缺失 | S：E06 查询刷新不发起抓取；E09 是整批装载，不是单校任务实现，不能据此声称已发生未经批准网络批量抓取。 |
| 010 | 发起与启用为独立动作（v39） | 缺失 | S：没有对应的两步命令、候选与启用回执；E11 包是观察而不是启用指令，仅技术表达存在。 |
| 011 | 原始数据、旧版本、逐字段旧新值、来源、时间、发起/启用人保留（v39） | 部分符合 | S/T：E03–E05 人工变更旧新值及操作审计存在；爬虫逐字段启用履历未找到。 |
| 012 | 已有申请固定引用历史资料，不随目录更新变化（v39、v45、v56） | 部分符合 | S/T：E05/E15 不可变解析版本、pin/hash 与引用约束/测试存在；未重跑跨流程。 |
| 013 | 发布、复制、Git 发布、部署不互相推导授权（原规则） | 部分符合 | S：E09/E11 未把包称作部署授权；目录里未找到完整发布授权流程证据，不能证明外部操作纪律。 |
| 014 | V1 学校身份七字段（v38） | 部分符合 | S：E01/E02/E05 有内部 UUID、source key、名称；E11 有 EDB/注册/校址表达。运行字段覆盖见附表，EDB 绑定无执行链。 |
| 015 | V1 基础资料十一字段（v38） | 部分符合 | S：E06/E09 仅映射/展示部分，E11 有结构；资料页、传真、授课时间等未接主页面。 |
| 016 | V1 学费住宿五字段（v38） | 部分符合 | S：E09 只有说明文字，E11 有三态住宿及学费字段；主运行路径没有完整五字段语义。错误容量 fallback 另见 020。 |
| 017 | V1 招生十五字段（v38） | 部分符合 | S：E06 有一套六字段、E09 部分 flat 字段，E11 有十五字段数组；没有多记录运行读取/启用。 |
| 018 | V1 来源履历十一字段（v38） | 部分符合 | S/T：E03–E05 人工履历和 E11 逐字段 evidence 存在；缺爬虫采集/检查/启用全链。 |
| 019 | 一校多招生，按学年/类型/年级分开且不共用截止日（v38、v58） | 冲突 | S：E06/E09 将每校映射为一个 flat AdmissionRecord、详情仅一套字段；无法呈现并存记录的独立截止日。E11 数组未接入。 |
| 020 | 未知不推断；无住宿资料不等于无宿舍（v38） | 冲突 | S：E09 用课室/住宿容量文字填住宿说明，没有“是否提供住宿”的显式三态；这不是可靠住宿字段映射。E11 bool known + unknown 本身无此冲突。 |
| 021 | 无截止日期不等于全年招生；暂缺仍未知（v38） | 部分符合 | S：E06 缺字段显示未知、E11 unknown 原因可表达；没有截止日启用规则和全链验证。未发现按当前时间补截止日代码。 |
| 022 | 不单列重复含义“学校类型”为首版主要字段（v38） | 超范围残留 | S：E06 的目录及 E09 的旧选校仍保留 school_type 列/筛选；现有版本管理 v1 快照 585/585 条的 school_type 与 finance_type 完全相同（如“資助”“私立”），有重复语义的实际资料证据，不仅凭字段名判定。 |
| 023 | 注册详情、容量、PDF 原文、评分留原始记录，不作首版主要字段（v38） | 超范围残留 | S：E09 容量 fallback 进入住宿说明，目录/旧选校仍展示和筛选 confidence；E11 留原始材料是基础，不能消除这些页面残留。 |
| 024 | 内部编号及系统更新履历由天行国际生成（v38） | 部分符合 | S/T：E01 UUID、E03/E04 人工版本/审计由服务生成；爬虫启用履历仍缺。外部 crawl_id 不等于内部履历已生成。 |
| 025 | 最低建档中文/英文至少一个非空，其余业务资料可未知（v38） | 部分符合 | S/T：E01 名称检查、可空字段、E07 只收名称，E12/E13 有资产；原来的地区/学制/学段/原因强制要求不再成立。 |
| 026 | 暂缺业务字段不免除系统自动记录版本/操作人（v38） | 部分符合 | S/T：E01 建档版本与操作者、审计/幂等存在；未运行。 |
| 027 | 官网用于辅助识别/抓取入口，与资料页分开（v40） | 部分符合 | S：E06 官网字段、E11 分开两个字段；没有主运行来源配置与抓取入口执行链。 |
| 028 | 资料页核对编号、名称和校址，与官网分开保存（v40） | 缺失 | S：E11 能表达，主目录/详情没有对应核对与存取链；普通 JSON 或官网字段不等于该规则。 |
| 029 | 招生页随学年变化，是资料来源而非固定身份（v40、v56） | 部分符合 | S：E09 单个 final_admission_url、E11 招生来源字段存在，E05 未将 URL 设作内部主键；没有多记录来源历史实现。 |
| 030 | 首次对应保存、网址变更留历史、有疑问人工核对；不凭名称/官网自动合并或重定向（v39、v40、v56） | 缺失 | S：E05 只有 source key 与 UUID，未找到对应维护/疑点处理路径。D1 及 D3（身份疑点、来源变化）的具体资格及效果另挂起，不据缺口提出默认流程。 |

### 爬虫启用、失败保护、恢复、并发与任务

| 序号 | 有效确认规则 / 出处 | 状态 | 依据层级与界限 |
| --- | --- | --- | --- |
| 031 | 对单校一次更新支持逐字段启用和全部接受（v41） | 缺失 | S：E06 待处理仅人工审批；E10 approved 状态不是逐字段启用。 |
| 032 | 只改选中字段；未选值保持且不视为接受（v41） | 缺失 | S：未找到爬虫字段选择/启用事务；普通人工单字段提交不能充当此路径。 |
| 033 | 两种启用均记批次/来源、实际字段、逐字段旧新值、版本、操作人时间（v41） | 缺失 | S：E10 按 school_key 覆写决定，未找到上述爬虫明细履历。 |
| 034 | 全部接受也保留明细、原始记录及旧版本（v41） | 缺失 | S：E05 提供不可变基础，没有全部接受命令和明细落点；不能单靠表约束证明。 |
| 035 | 失败/缺失/空值不清已有有效值；没有旧值保持未知（v42） | 部分符合 | S/T：E09 整包验证失败保留此前内存快照，E11 表达 failed/unknown；不是逐字段非空旧值保护，也未接主目录。 |
| 036 | 保留“未取得数据”、失败/缺失原因及本次抓取履历（v42） | 部分符合 | S：E11 有字段 reason 和 run failure；E10 的粗粒度状态不提供本校逐字段失败历史。 |
| 037 | 逐项及全部接受均不可用未取得数据覆盖旧值（v42） | 缺失 | S：未找到启用事务或相应保护分支。 |
| 038 | 保留旧值也保留其旧来源/采集时间；新检查时间不冒充新采集（v42、v48） | 缺失 | S：E11 分 collected_at/checked_at；主运行缺保留旧值的爬虫启用路径，不能证明时间关联。 |
| 039 | 明确有来源的“否”等有效值不当作缺失（v42） | 部分符合 | S：E11 boolean false 为 known，与 unknown 分开；E06 只把字符串作为可显示值，缺业务端三态读取/启用。显式清空另依赖 N6（恢复为空值、显式清空资料）。 |
| 040 | 爬虫与人工值冲突要提示曾人工修改、展示旧新来源、默认留人工值（v43） | 部分符合 | S/T：E05/E08 解析器保留 approved overlay 并返回 base_changed，E12 有测试；E06 尚未展示该爬虫冲突交互。 |
| 041 | 全部接受默认跳过人工冲突，不视为已接受（v43） | 缺失 | S：无全部接受命令，不能由解析器保留 overlay 推导“批量默认跳过”。 |
| 042 | 页面启用者可显式逐项采用冲突中的爬虫新值，不由批量推导（v43） | 缺失 | S：没有该选择/确认入口及服务授权链；旧 reconcile 不是普通用户的该操作。 |
| 043 | 显式采用仍保留启用人/时间/来源/旧新值/版本和原人工历史（v43） | 缺失 | S：人工审批回执存在，但没有爬虫冲突采用的履历事务。 |
| 044 | 页面启用者可从更新履历逐字段恢复已存在旧值（v45） | 缺失 | S：E08 是 Founder/Data Reviewer 旧整 revision 停用，E14 unavailable；不是此恢复授权/选择路径。 |
| 045 | 恢复只影响所选字段，不回退其他字段（v45） | 缺失 | S：没有逐字段恢复命令；不把旧 whole-overlay rollback 当作已符合。 |
| 046 | 恢复生成新版本，记操作人/时间/前后值/引用历史，中间版本保留（v45） | 部分符合 | S/T：E05 不可变版本和 E08 旧回滚回执是基础；缺本规则逐字段恢复链。 |
| 047 | 恢复不修改申请已固定引用版本（v45） | 部分符合 | S/T：E05/E15 固定版本约束存在；逐字段恢复流程未运行。 |
| 048 | 恢复只能选择履历已有值，不扩自由编辑/合并/停用权限（v45） | 缺失 | S：无对应恢复命令或字段历史选择检查；N4（恢复的旧值）及 N6 的后续保护及空值边界不作判定。 |
| 049 | 同字段当前值已变化时拒绝过期启用并提示“资料已变化”（v46） | 部分符合 | S/T：E03/E04 对人工提交/审批有有效值 hash 和 stale 拒绝，E06/E13 有界面/断言；爬虫启用分支缺失。 |
| 050 | 重新查看当前差异，不能复用旧确认（v46） | 部分符合 | S/T：E06 人工表单冻结请求并在 stale 后要求重看，E13 有资产；不是爬虫差异复核验收。 |
| 051 | 逐项/全部接受均检查并发，不限定技术方案（v46） | 缺失 | S：两种爬虫命令均未找到；不能以人工审批锁覆盖缺口。 |
| 052 | 同校进行中任务由可见用户共享同一进度（v47） | 缺失 | S：E10 粗粒度 crawler_runs 不构成按学校共享进行中任务/进度路径。 |
| 053 | 重复点击/多人发起不重启、不新增同校任务（v47） | 缺失 | S：人工请求幂等存在但无同校抓取复用机制、唯一进行中任务约束或路由。 |
| 054 | 终止后允许再发起；每次成功/失败任务历史不覆盖（v47） | 缺失 | S：未找到按学校的完整任务终止、再次发起及追加履历。停站运营规则依赖 N7（逐校计划）。 |
| 055 | 任务复用不代表自动启用（v47） | 缺失 | S：无复用/启用边界实现；兼容 v1 自动 active 冲突已计 007。 |
| 056 | 不同学年招生分别保存，新学年不覆盖旧学年（v48） | 冲突 | S：E06/E09 单校单记录形状不能保留并存年度；E11 仅交接数组。 |
| 057 | 来源学年无法识别标未知，不用抓取时间补当前学年（v48、v56、v57） | 部分符合 | S：E11 unknown/year 结构、E06 空白显示未知；未发现按抓取时间推学年，亦无运行端招生身份维护。 |
| 058 | 每次实际抓取时间关联任务/来源，成功失败无变化都留（v48） | 缺失 | S：E11 可表达真实时间和结果，但主运行无接入任务/结果历史；E09 summary 的 published/generated 不能替代本字段。 |
| 059 | 学年、信息发布日期、实际抓取时间、启用时间分别记录（v48） | 部分符合 | S：E11 拆开学年/发布日期/采集检查时间，E04 有人工批准时间；缺爬虫启用时间及主运行关联。 |
| 060 | 使用旧值保留原采集时间，本次只记检查/任务时间（v48） | 缺失 | S：未找到运行侧旧值时间继承分支；与 038 的要求一致，此项核对任务时间关联。 |

### 来源维护、人工补充、页面与初始目录

| 序号 | 有效确认规则 / 出处 | 状态 | 依据层级与界限 |
| --- | --- | --- | --- |
| 061 | 人工添加/修改抓取来源审批后生效（v49、v55 限定） | 部分符合 | S/T：E03/E04 对官网人工变更有审批，E06 有官网修改入口；资料页/招生源配置及添加链没有对应专项实现。提交资格未回答，依赖 D1。 |
| 062 | 来源变更留旧新值、提交/审批人、时间、决定；审批前不作有效来源（v49） | 部分符合 | S/T：E03/E04 对官网候选有明细回执，未接抓取来源配置，不能证明审批状态被实际抓取读取。 |
| 063 | 已有来源发起/启用按页面访问权，不重限为 Founder（v49） | 缺失 | S：抓取/启用路径未找到；E09 warning gate 另见 109。 |
| 064 | 逐项/全部接受/恢复不能绕过人工来源审批（v49） | 缺失 | S：这些运行命令均缺，未找到来源配置变更识别/审批边界。 |
| 065 | 来源维护不新增提交资格、禁止自审；发现证据链接不是批准来源（v49） | 部分符合 | S/T：E04 人工自审禁止、E11 evidence 与 school URL 分开；无完整来源候选/批准来源维护链。D1 资格仍挂起。 |
| 066 | 新招生先候选、保留内容来源，未知年级/学年不自动合并（v50） | 部分符合 | S：E11 admissions/candidate_id/unknown 可表达，未找到主运行招生候选接入/保存/展示。 |
| 067 | 用户确认招生候选后启用，沿用资格履历，缺失有效条件未确认（v50） | 缺失 | S：无运行候选启用路径；具体最低条件依赖 N1（招生候选最低有效条件），不判定未知候选应允许或拒绝。 |
| 068 | 内部页面用户能手动补未知资料，不开放外部客户（v51、v52） | 部分符合 | S/T：E02/E03 服务端资格、E06 基础字段补充、E13 L2 未知字段测试资产存在；招生等其余资料旁无补充入口。 |
| 069 | 手动补充记补充人/时间/原新值/人工来源，不伪装爬虫或改原始记录（v51） | 部分符合 | S/T：E03–E05 有提交旧新值/evidence、approved_overlay 来源与不可变原始快照；缺完整 V1 补充字段覆盖及运行证据。 |
| 070 | 内部可见用户提交补充、审批后生效，审批前不改当前值（v52） | 部分符合 | S/T：E03/E04 候选及 Founder/L1 试用业务审核链、E13 断言存在；建档后补齐交界 U1（建档完成后补齐）、U2（建档同时填写与之后补齐）、U3（后续补齐免审批）未回答。 |
| 071 | 补充及审批留决定、旧新值、版本，并禁止自审（v52） | 部分符合 | S/T：E03/E04/E06、E13 有人工提交/审批回执和显示；未运行。 |
| 072 | 未知补充不扩大已有值自由编辑/合并/停用；来源补充仍审批（v52） | 部分符合 | S/T：E03 限 L2 为未知、E06 根据 server can_edit_existing 显示，E04 审批；不据此答 U 项、退场资格或来源关联资格。 |
| 073 | 详情“基础资料/招生资料/待处理更新/更新履历”四块（v53） | 部分符合 | S/T：E06 四块明确存在，E13 有界面断言资产；待处理/历史目前只包含人工变更。 |
| 074 | 补充入口在资料旁，待审补充进入待处理（v53） | 部分符合 | S/T：E06 六个基础字段内联补充及人工 candidate 列表存在；招生资料旁没有该入口，未运行。 |
| 075 | 入口可见不授操作权，四块不改授权（v53） | 部分符合 | S/T：E02–E04 服务器重新校验，E06 使用服务端上下文、E13 有拒绝断言；不能由组件可见性宣称授权验收。 |
| 076 | 首次一次性 EDB 全量抓取/导入可作初始目录（v55） | 缺失 | S：E05 通用快照表与 E11 交接形状存在，未找到按 EDB 粒度完成导入/来源履历的运行入口。本票不执行导入。 |
| 077 | 人工/爬虫建档都免额外逐校可用审批，不自动合并疑似学校（v55） | 部分符合 | S/T：E01 人工创建无额外审批；E02 provisional 独立列表而详情需快照，尚无爬虫免审批入目录链。何谓 Cases 可选依赖 D7（Cases 可选资格的剩余问题），不把缺快照直接定为可选资格冲突。 |
| 078 | 允许无人周期抓取，但后续变化仍须启用（v55） | 缺失 | S：E10 配置 checkbox 不是调度执行器，未找到任务调度及独立启用闭环；具体计划运营依赖 N7。 |
| 079 | 周期频率逐校设，不采用全局统一频率（v55） | 冲突 | S：E10 全局 `key='default'` 频率配置及管理页统一 frequency，与逐校规则相反；没有证据该配置已驱动真实抓取。 |
| 080 | 最高频率每天一次，不能更频繁（v55） | 缺失 | S：E10 页面选项最短 daily，但服务只存字符串，未找到调度侧逐校最高频率校验；UI 选项不是系统限制证据。 |
| 081 | 仅首次 EDB 自动网址免人工来源审批，留 EDB 导入履历，不伪造人工回执（v55） | 缺失 | S：没有导入履历及区分该受限例外的来源配置链；E11 evidence 可表达不能证明豁免被正确使用。 |
| 082 | provisional 未验证标记不展示（v55） | 冲突 | S：E07 明确仍有“未验证学校”标题、成功提示、建档按钮及逐校徽标。无需运行即可定位相反文案；未验证实际浏览器呈现。 |
| 083 | 原始人工建档永久保留不可变，不伪装核实（v55） | 部分符合 | S/T：E01 追加建档事实与迁移不可变/RLS，E13 有资产；隐藏 UI 不应删记录，当前原始记录结构有基础。 |
| 084 | 隐藏未验证不取消未知、疑点提示、历史（v55） | 部分符合 | S：E06 未知及人工历史存在；E07 仍显示已取消标记，来源疑点处理缺，不能把这当完整符合。 |

### 学校粒度、招生身份、归一化及受限例外

| 序号 | 有效确认规则 / 出处 | 状态 | 依据层级与界限 |
| --- | --- | --- | --- |
| 085 | 一校对应 EDB scrn 前六位；不同编号不同学校（v56 D4） | 部分符合 | S：E11 六位 EDB 包边界有校验；E05 内部 source_school_key 唯一，无 EDB 粒度身份读取/接入执行链。不能将 JSON 字段当 EDB 唯一学校实现。 |
| 086 | 同编号校址/时段/学段为属性，建筑单元不拆校（v56 D4） | 部分符合 | S：E11 属性数组/复合样本可表达，运行目录未按 EDB 汇聚这些属性。 |
| 087 | 名称相似/同团体/同地区不合并不同编号（v56 D4） | 缺失 | S：主运行没有 EDB 对应/校核/疑点流程；未发现名称自动合并代码，但“没合并代码”不能证明接入校核已具备。D1/D3/T6 的纠正效果不回答。 |
| 088 | 校内小一/中一、插班、时段区别在招生层，不改学校身份（v56 D4） | 冲突 | S：E06/E09 单套招生结构无法表达该确认的多记录边界；E11 仅形状基础。未发现按时段拆学校的运行事实，不指控此额外行为。 |
| 089 | EDB 粒度不替换稳定内部 UUID/系统历史引用（v56 D4） | 部分符合 | S/T：E01/E05/E15 稳定 UUID/pin，E11 区分内部与 EDB；主运行尚无 EDB 绑定链。 |
| 090 | 人工无 EDB 不伪造、仍按名称最低建档；不补关联纠错答案（v56 D4） | 部分符合 | S/T：E01 无 EDB 必填/推造、E11 unknown/unbound 可表达；具体来源关联依赖 D1，不宣称它已解决。 |
| 091 | 官网/资料页仅辅助核对，疑点人工处理，不凭网址关联/合并（v56 D4） | 缺失 | S：E05 单 source key/UUID，无已确认的来源疑点提示链。普通网址变化的业务后果不代答 D3。 |
| 092 | N2 只同校三项全同更新，任一不同分记录（v56，v57/v58 比较基础，v59 受限例外） | 缺失 | S：E11 校验已知三元组，但校验不是跨批次对应/更新引擎；E06/E09 只读单套字段，运行侧未找到 N2。例外依赖 H3（语言/版本证据如何认定）、H4（同一招生如何判定）、H5（语言未知如何处理）、H6（两条记录的后续更新如何对应）。 |
| 093 | 普通来源 URL 不是第四条件；编码/大小写/等价变化不新建（v56、v59） | 部分符合 | S：E11 三元组 key 不含 URL、已知新 URL 样本存在；无跨抓取对应执行路径，不能以此证明真正更新行为。 |
| 094 | 不要求每条招生先人工确认内部编号；候选启用仍须确认（v56） | 部分符合 | S：E11 candidate_id 不要求审批回执，原文明确包非启用指令；主运行无招生候选确认链。 |
| 095 | 学年未知每次抓取新候选、不更新既有未知，不推年；认领清理不等于合并/删除授权（v56） | 部分符合 | S：E11 unknown 必须 new-candidate 且禁止未知候选报告 unchanged；运行侧每次新增、保留、认领/清理未找到，H1（人工手动合并两条招生候选）及 N1 不回答。 |
| 096 | 未知年级不当相同、不自动合并、不增加默认启用资格（v56） | 部分符合 | S：E11 unknown component 必须 new-candidate，E06 未知显示；运行身份对应未找到，N1 挂起。 |
| 097 | 对应更新不自动启用/改原始/改固定引用；继承失败和人工冲突保护（v56） | 缺失 | S：E05 有历史基础，无招生对应及更新启用链。不能由 generic overlay 推导 N2 全部保护。 |
| 098 | 招生学年 YYYY/YY，已确认写法统一；来源不明仍未知（v57） | 部分符合 | S：E11 已要求 YYYY/YY，格式本身一致；E06 主详情显示字面字段，无标准值及原值关联/归一化消费链。不是把 YYYY-YYYY 误判为当前契约格式。 |
| 099 | 本地词汇按页面归一 P1–P6/S1–S6，不依赖学校级学制判定（v57、v58） | 缺失 | S：E11 接受标准代码不等于执行中文/英文词汇归一，运行侧没有标准/原始双值与对应比较。H2（学制判定依据的缩小范围）不应重新成为此前置门禁。 |
| 100 | 国际 Y1–Y13、G1–G12、P 班保持原词汇，不与本地互映（v57、v58） | 冲突 | S：E11 grade regex 只允许 P1–6/S1–6/G1–13，拒绝 Y1–13 和 P 班，且 G13 超出已确认美制范围。这是 proposed 技术表达与现行 BR 差异，不是已批准业务枚举或生产映射证据；H2 展示分组仍挂起。 |
| 101 | 标准值只用于匹配/展示，原始字面值与逐字段证据永久保留（v57、v58） | 部分符合 | S/T：E11 raw_record/evidence、E05 不可变 JSON、E03 人工证据有基础；没有运行端附加标准值与原字面/证据关联，不能证明永久全链。 |
| 102 | N2 比较归一化学年/本地年级；招生类型不作同义词归一（v57、v58） | 缺失 | S：E11 比较包内已经标准化值；主运行无归一化 N2 对应链。没有证据证明运行侧 type 同义词合并，亦不把技术 token 当批准的类型词汇。 |
| 103 | 枚举 and/&/斜杠包含全部列出的标准年级（v58） | 缺失 | S：E11 可放代码集合，但运行侧未找到字面枚举→标准集合的消费/匹配/展示实现；不检查 school-tracker。 |
| 104 | 区间连续含首尾，原表达证据保留（v58） | 缺失 | S：E11 能放展开值，运行侧未找到区间语义及原始证据关联实现。 |
| 105 | 无法识别标准年级不猜、国际与本地不映射（v58） | 部分符合 | S：E11 unknown 形状有基础，运行侧未找到跨学制推断逻辑；格式拒绝国际有效词汇的冲突见 100，不据“无推断代码”宣称完整验收。 |
| 106 | 不同招生分开保留，不能因同校压成一条；同一招生更新仍按 N2（v58） | 冲突 | S：E06/E09 每校单条映射，与多记录展示相反；无法据其说明同一事项跨次更新已对应。 |
| 107 | 有明确同招生页面证据的不同语言/版本保留两条并标差异（v59） | 冲突 | S：E11 record 不含语言/版本限定身份，additionalProperties=false，已知三元组一律 MATCH_KEY_DUPLICATE；形状不能承载获确认的受限例外。**不认定具体页面**：H3/H4/H5/H6 尚挂起，不能据此直接补判定/更新规则。 |
| 108 | 例外不普遍加语言/URL 条件；外部未知保护及 N2 主结构仍保留（v59） | 部分符合 | S：E11 URL 未加入 key、unknown component 保护存在；缺受限例外与运行匹配链。未知语言适用方式依赖 H5，不默认任何语言。 |
| 109 | 旧 Founder-only/warning-only 爬虫启用门槛已被页面用户启用规则取代（v39 后的 superseded 范围） | 超范围残留 | S/T：E09 warning receipt 仍要求 reviewer recommendation + Founder accept，E12 测试仍保护该分支；与现行普通抓取启用资格不一致。该 v1 发布/读取机制不同于主数据库入口，不能混作生产启用记录。 |
| 110 | Data Reviewer 不作为独立学校人工审核资格；普通 Admin 文案不替代确认权限（原 BR 结合 BR-015 适用边界） | 超范围残留 | S/T：E08 领域类型/旧服务、迁移约束/回滚资格和旧治理测试仍允许 Data Reviewer；E10 兼容决定 API 仍允许 Admin/Data Reviewer，管理页称普通字段 Admin 审核。当前 v1 E04 已收紧，**不宣称它允许旧角色**；具体待决治理动作不由此定资格。 |

## 学校字段 V1：49 字段覆盖附表

这是字段逐一核对，不增加规则统计。`JSON` 可存仅说明容器能力；“交接字段存在”均属于 E11 的 proposed 表达，不证明正式读写或启用。字段未接业务链写明缺口，不将现有别名当成已经对齐。

| 分组 / 已确认字段 | 当前代码对应及界限 |
| --- | --- |
| 身份：系统内部编号 | E01/E05 的 school UUID、API school_id、固定引用；S/T。 |
| 身份：爬虫来源编号 | E05 source_school_key、E09 school_key；未找到来源确认/改绑链。 |
| 身份：教育局学校编号 | E11 edb_school_number；主运行没有六位粒度识别/对应实现。 |
| 身份：注册编号 | E11 registration_numbers；E06 主资料块未展示。 |
| 身份：校址标识 | E11 campuses 中标识；运行侧没有同编号多校址合并属性链。 |
| 身份：中文名称 | E01/E02/E06；最低非空要求及候选修改资产存在，S/T。 |
| 身份：英文名称 | E01/E02/E06；同上，S/T。 |
| 基础：官网 | E06 official_website、E09 website/official_website；人工审批基础有，抓取源配置链缺。 |
| 基础：学校资料页面地址 | E11 profile_urls；主详情/目录不消费。 |
| 基础：学段 | E09 school_level、建档 stage、E11 school_levels；别名/数组没有统一消费链。 |
| 基础：地区 | E06/E09 district，S；缺完整来源/启用关联。 |
| 基础：地址 | E06/E09 address，S；多校址属性未接。 |
| 基础：电话 | E06/E09 phone，S；人工变更 S/T。 |
| 基础：传真 | E11 faxes；E06/E09 主展示/映射缺。 |
| 基础：学生性别类别 | E11 gender_categories；E06/E09 主展示/映射缺。 |
| 基础：授课时间类别 | E11 sessions；运行消费及多时段聚合缺。 |
| 基础：资助类别 | E09 finance_type、E11 funding_categories；主运行缺语义完整消费。 |
| 基础：特殊教育标记 | E09 Boolean(raw.is_sen) 会将缺失转 false，字符串 false 转 true；这是**额外的静态未知值风险**，不能证明来源核实。E11 可区别 true/false/unknown，主语义未接。 |
| 学费住宿：学费说明 | E09 tuition_info/approved_course_and_tuition_info；只有文字映射。 |
| 学费住宿：学费适用学年 | E11 tuition_academic_year；运行映射/展示缺。 |
| 学费住宿：学费来源链接 | E11 tuition_source_urls；运行关联缺。 |
| 学费住宿：是否提供住宿（是/否/未知） | E11 boarding 的 known bool/unknown；E09 无该三态字段。旧 selector “有资料/无资料”不等于住宿事实的是/否。 |
| 学费住宿：住宿说明 | E09 dormitory_info fallback 容量文字、E11 boarding_description；fallback 不能证明实际住宿。 |
| 招生：适用学年 | E06 admission_school_year/school_year；E11 academic_year；旧 E09 normalizeSchool 不保留学年字段，无标准/原始双值消费链。 |
| 招生：招生类型 | E06/E09 admission_type，仅 transfer/s1_admission/unknown 映射；E11 admission_type；未实施同校身份对应，不能据限制类型提出新业务词汇。 |
| 招生：招生年级 | E06 admission_grade；E11 grade；旧 E09 normalizeSchool 不保留年级，没有主运行多招生/归一化链。 |
| 招生：信息发布日期 | E11 published_date；运行学校记录映射缺，summary 发布日不是逐招生发布日期。 |
| 招生：申请开始日期 | E06 application_start_date、E09 application_open、E11 application_open；字段别名未形成统一来源链。 |
| 招生：截止日期 | E06 application_deadline/submission_deadline、E09 submission_deadline、E11 application_deadline；运行只一套截止日。 |
| 招生：申请时段说明 | E09 application_dates/application_period、E11 application_period；主详情未逐条展示。 |
| 招生：申请方式 | E06/E11 application_method；E09 学校映射缺该字段。 |
| 招生：所需材料 | E09 required_materials、E11 required_materials；主详情多记录消费缺。 |
| 招生：申请表链接 | E09 application_form_links、E11 application_form_urls；主详情多记录消费缺。 |
| 招生：招生页面链接 | E09 单 final_admission_url、E11 admission_page_urls；主详情多来源记录消费缺。 |
| 招生：考试日期 | E09 exam_date、E11 exam_dates；主详情多记录消费缺。 |
| 招生：面试日期 | E09 interview_date、E11 interview_dates；主详情多记录消费缺。 |
| 招生：第二轮面试日期 | E09 normalizeReview 有 second_interview_date，normalizeSchool 无；E11 second_interview_dates；主学校消费缺。 |
| 招生：结果公布日期 | E09 result_notification_date、E11 result_dates；主详情多记录消费缺。 |
| 来源履历：来源网址 | E03 人工 evidence URL、E09 evidence_urls、E11 evidence.source_url；未形成爬虫逐字段有效值关联。 |
| 来源履历：证据摘要 | E03 quote、E11 quote；E09 仅粗粒度 notes/URLs，缺逐字段 evidence 主消费。 |
| 来源履历：采集时间 | E11 collected_at；主数据读取/保留旧值及来源关联缺，不用 summary 时间冒充。 |
| 来源履历：最近检查时间 | E11 checked_at；运行每校任务/字段关联缺。 |
| 来源履历：缺失项/复核原因 | E09 reviewQueue missing_fields/notes、E11 reason；主详情尚只人工变更列表。 |
| 来源履历：版本号 | E01/E03–E05 建档/修订/解析版本，S/T；缺爬虫逐字段启用版本链。 |
| 来源履历：变更前后值 | E03/E06/E13 人工提交时值/当前值/申请值，S/T；爬虫差异启用缺。 |
| 来源履历：发起人 | E01/E03 人工 created/requested actor，S/T；学校抓取发起人履历缺。 |
| 来源履历：启用人 | E04 人工审批人，不等同爬虫启用人；爬虫启用缺。 |
| 来源履历：启用时间 | E04 reviewed_at 是人工审批时间；爬虫启用时间缺。 |
| 来源履历：抓取结果 | E11 changed/unchanged/partial/failed；E10 粗粒度 run 状态；无主运行校级结果历史闭环。 |

附表发现的特教 Boolean 转换风险归入第 015 项的基础字段“不完整”，不新增已确认业务枚举或额外统计行。没有输入/运行证据时不声称某所真实学校已被标错。

## v39 之后确认、当前没有对应运行实现的主要部分

以下是“没有找到可执行业务路径”的清单；存在 proposed 契约表达、通用 JSON、空组合路由或人工路径相似代码时仍明确区分，不把它们误称为从零缺少所有资产。

归一化可以由产出方提供标准值，本报告不要求天行国际重复做页面抽取或规定归一化在哪一侧执行。相关缺口指**当前天行国际对标准值、原值/证据、N2 对应及统一展示尚无接通的消费链**；本票未审查 school-tracker，不能据此断言产出方没有归一化实现。

| 确认范围 | 完全缺少的运行部分 | 对应矩阵 |
| --- | --- | --- |
| v40 来源核对 | 官网/资料页分别使用的核对、首次对应保存、网址变更/疑点提示链 | 028、030、091；具体资格/后果依赖 D1/D3 |
| v41 字段启用 | 单校候选逐项/全部接受命令、未选保留、明细启用回执 | 031–034 |
| v42 失败保护 | 实际启用中的逐字段旧值保护、旧来源/采集时间继承、新检查履历 | 037、038、060；有技术表达但无启用链 |
| v43 人工冲突 | 全部接受跳过冲突、普通启用者显式逐项覆盖及履历 | 041–043 |
| v45 恢复 | 页面启用者从履历逐字段恢复，不影响其他字段及既有引用 | 044、045、048；旧整 revision 回滚不可替代 |
| v46 抓取并发 | 在两种爬虫启用命令中重新核对当前字段及确认 | 051；人工变更已有相似保护 |
| v47 任务复用 | 同校进行中任务共享、去重、终止后重抓、每次任务历史 | 052–055 |
| v48 时间 | 每次抓取实际时间关联校/任务/来源，成功失败无变化都留，旧值时间不改 | 058、060 |
| v49 维护边界 | 全部来源类型的配置/审批及启用、恢复不能绕过维护审批的执行链 | 063、064；官网人工审批有基础，未声称全部来源审核代码为零 |
| v50 候选 | 招生候选保存/展示/确认启用；最低资格问题仍停在 N1 | 067；E11 形状不是运行候选 |
| v55 初始/周期目录 | EDB 一次性目录接入、受限网址免审批履历、逐校调度及频率硬限制 | 076、078、080、081；N7 运营问题不补答案 |
| v56 N2 | 同校三项身份跨批次对应/更新、unknown 每次新候选的执行链 | 092、097；095/096 只有交接保护表达 |
| v57/v58 归一化 | 标准值与原值/证据关联消费、页面词汇年级归一、归一化 N2、枚举/区间消费 | 099、102–104；098/101 有静态表达基础 |

v38/v39 的单校更新入口本身也仍缺，但不把它错算成“v39 之后新确认”。v51/v52/v53 有人工补充、审批及四块组件，不能写成“完全无实现”；主要缺口是资料范围与运行验证。v59 受限例外的表达缺口见下节，具体执行标准仍依赖挂起项。

## 与现行规则相反的实现与已排除残留

| 类别 / 位置 | 静态差异 | 适用路径及限度 |
| --- | --- | --- |
| 冲突：自动 active（007） | E09 校验通过直接将候选置为 active，没有普通用户独立启用 | 兼容文件快照/旧 crawler GET；不是主 PostgreSQL 目录自动发布的证据 |
| 冲突：单校单招生（019、056、088、106） | E06/E09 单 flat record、一套年级/学年/截止日 | 四条核对项从字段、跨学年、D4 招生层、v58 展示不同确认范围检查**同一结构问题**；不是四个独立缺陷数量 |
| 冲突：容量当住宿（020） | E09 用 approved_classrooms_and_dormitory_capacity 填 dormitory_info，缺是/否/未知语义 | 文字有内容不等于是否住宿；旧 selector 文案是“有资料/无资料”，不指控它明确显示“无宿舍” |
| 冲突：统一频率（079） | E10 default 全局频率，与逐校设置相反 | 只有配置路径的证据，没有实际调度请求证据 |
| 冲突：未验证标记（082） | E07 仍在标题、按钮、通知和学校徽标呈现 | 当前主目录使用该组件；没有本轮浏览器验证 |
| 冲突：国际词汇（100） | E11 拒绝 Y1–13、P 班，接受 G13，未覆盖确认范围 | 仍为 proposed 交接 schema，不是批准接口；不能据该枚举推导业务标准 |
| 冲突：双语/版本表达（107） | E11 只有 candidate_id/identity_mode/fields，无限定身份，known key 一律判重复 | 仅判表达装不下，不据实例判例外成立；H3–H6 继续挂起 |
| 超范围残留：学校类型（022） | E06/E09 目录/筛选主要字段保留 school_type；受版本管理的 585 条快照均与 finance_type 同值 | v38 已不单列重复含义学校类型；这不否定已确认的学段/资助/性别等字段，也不把 BR-015 的国际/本地案件分类当此冗余字段 |
| 超范围残留：原始评分/容量业务化（023） | confidence 主要展示/筛选，容量 fallback 入住宿 | 原始记录仍可保留这些内容，本票不要求删原文 |
| 超范围残留：warning 专属门槛（109） | reviewer recommendation + Founder accept 仍是 v1 warning 进入 active 的前提，测试仍保护 | 旧门槛不构成当前普通抓取启用授权；发布动作仍独立，不能替它默认新发布规则 |
| 超范围残留：旧审核角色/文案（110） | 旧领域、迁移、测试、兼容决定 API 保留 Data Reviewer/Admin 路径或说明 | 当前 v1 人工审批已是 Founder/L1；不能把残留类型当作它的有效路由授权 |

额外静态风险不作确定冲突：E05 解析器仅选最高号 approved overlay，再从原始 base 应用该 revision 的字段；E03 每次可只提交单字段。连续审批不同字段时先前有效值是否被完整保留，现有已查看测试未提供该场景闭环证据。记录为**证据不足，需补充运行验证**，不据此补出未确认的候选合并规则，也不把“可能丢值”写成已发生的业务事实。

## 依赖挂起项而停止判定的范围

以下逐一沿用 v60 的 **28 项**，只是列出本次遇到或必须排除的判定边界。状态均仍为 `pending_review`，没有分配五种分析结论，也不计入任何完成度、交付或验收。具体登记、参考收口点及原文以 [v60 挂起项登记表](/Users/karo/Documents/txgj-doc/business-requirements/50-schools.zh-CN.md:502) 为准；本票不重分阶段、不调整范围。

| 挂起项（原标签/范围） | 本次不能据代码回答的判定；相关证据 |
| --- | --- |
| D1（人工来源/纯外部键关联） | 来源提交、首次对应确认、撤销/改绑资格及效果；E05 的 source key 不给答案。关联矩阵 003、030、061、065、090。 |
| D2（可用抓取入口组合） | 缺 EDB/外部号/网址时能否抓取及提示；不据 v2 unknown 模式或无按钮决定。关联 008、009、027–030。 |
| D3（身份疑点、来源变化） | 新任务、进行中任务、旧候选能否继续；没有实现不代表应一律停/继续。关联 030、087、091。 |
| D7（Cases 可选资格的剩余问题） | 免审批“可用”与 Cases 可选是否等价；E15 当前查询条件、E02 provisional 不在快照详情不能反推业务资格。关联 077。 |
| T1（适用情形） | 停办、录错、EDB 重复、抓取错误的分类与动作；snapshot retired 是技术快照状态，不是学校退场答案。 |
| T2（发起与审批） | 退场资格和审批；不据旧 disable role 或 Founder/L1 审核路径决定。关联 003、E08。 |
| T3（依据与生效） | 退场证据、生效时点、待处理展示/可选性；类型和查询过滤不充当确认。 |
| T4（目录与新使用） | 退场后目录/搜索/选校/新引用；不根据当前选项过滤作符合/冲突判定。 |
| T5（已有引用） | 退场对候选、申请、任务影响；已有 immutable pin 可证明历史基础，不能决定额外副作用。 |
| T6（停办与录错的差异） | 纠正、合并、改绑、历史归属；D4 已确认不意味这些流程也获批。关联 003、087。 |
| T7（后续抓取） | 退场后继续抓取及 EDB 再出现如何处理；周期允许性不提供答案。 |
| T8（恢复与留存） | 学校退场恢复/删除资格和展示；旧 overlay 回滚不等于学校恢复授权，不把技术不可变约束当退场完整方案。 |
| T9（履历与知会） | 退场额外记录及通知对象；现有审计不回答额外通知需求。 |
| U1（建档完成后补齐） | 补齐资料是否仍属 v52 审批；E01/E03 路径不同不等于业务界限已定。关联 070、072。 |
| U2（建档同时填写与之后补齐） | 两者是否同规则及界限；不能据数据库落表时间作答案。关联 070。 |
| U3（后续补齐免审批） | 条件性的豁免范围与取代范围；不据最低建档免审批推导。关联 070、072。 |
| N1（招生候选最低有效条件） | 未知字段候选是否可启用；不能把 proposed schema 必填字段当业务门禁。关联 067、095、096。 |
| N3（长期未处理/已过时候选） | 旧候选启用/重抓/提醒规则；不能套用 v1 warning 24 小时回执期限。 |
| N4（恢复的旧值） | 恢复后再遇爬虫冲突是否有人工保护；E05 sourceKind 不提供答案。关联 048。 |
| N5（未选/不想采用的变化） | 忽略/关闭/再次提醒等额外行为；未选不生效已确认，其他不据旧 queue 状态补出。 |
| N6（恢复为空值、显式清空资料） | generic proposed_value_json 能放 null 不是显式清空授权，也不能据其存在直接判业务允许/冲突。关联 039、048。 |
| N7（逐校计划） | 启停/维护资格、入计划、失败暂停/通知；E10 技术配置不回答这些业务问题，频率及上限已确认部分仍可核对。关联 054、078–080。 |
| H1（人工手动合并两条招生候选） | 不以 N2、认领清理或重复 key 校验授予人工合并。关联 095。 |
| H2（学制判定依据的缩小范围） | 国际英制/美制分组和混合学制呈现；不再阻塞页面词汇归一，不据学校类型推分组。关联 099、100。 |
| H3（语言/版本证据如何认定） | 不编判定词汇/证据门槛，不仅凭 URL 不同认定例外。关联 107。 |
| H4（同一招生如何判定） | 不判断任何真实中英文页面属于同一招生；只能指出形状缺限定身份。关联 092、107。 |
| H5（语言未知如何处理） | 不设默认语言/版本或未知保留/对应策略；未知学年/年级的既有规则不替代此题。关联 107、108。 |
| H6（两条记录的后续更新如何对应） | 不依据三元组、URL 或 candidate_id 选择更新哪条语言记录。关联 092、107、108。 |

22 项参考收口点来自 SCH-INT-04（学校待确认项阻塞分类清单），该文档状态 proposed，分类是依赖判断而非业务确认；本报告不把它用作判定标准。其余六项仍为“需负责人确认收口点”。到达任一收口点而决定未出，相关实现必须停下回到本对话，不按默认值、技术便利或旧设计继续。上述表格不是对这 28 项的新增业务回答。

## 本轮实际核对结果与未验证层

- 已核对业务 main 为 v60，Tianxingguoji 本地与远端 main 均为本报告记录的代码 SHA；txgj-doc main 已执行 `git pull --ff-only origin main`，输出 `Already up to date.`。
- 静态检索中，v2 路径引用位于契约/校验脚本；未在 `modules/schools` 或学校运行路由找到 v2 接入调用。目录 v1 API 与旧 `/api/crawler/*` 是不同路径，不能合并成一个“已接通”结论。
- 38 个学校模块文件、相关迁移、页面/路由和测试资产已作为上述结论的检查范围。测试名称、断言和迁移触发器是资产证据，没有本轮通过率。
- 文档核验只检查计数、引用存在、差异范围及历史原文保持；不编写或执行业务测试。本地运行、浏览器、Preview、生产四层全为**证据不足，需补充运行验证**。
- 不新增业务答案、不产生新基线、不触碰代码/迁移/测试/接口、其他八个模块分析、Tianxingguoji 或 school-tracker 文件及任何历史抓取产物。

本轮文档核对实际输出（不是业务测试输出）：

```text
核对项: 110 {'部分符合': 57, '冲突': 10, '缺失': 39, '超范围残留': 4}
V1 字段: 49
挂起项: 28
引用目标: 67 缺失: [] 越界行号: []
旧学校报告逐字节未变: True
business-requirements diff: ''
CHANGELOG.md diff: ''
其余八模块未变: True
```

本报告列出的缺口可以独立定位，但不能据此宣布完整学校集成闭环通过，或把挂起分支当作已完成部分。
