# 学校重建只读评估与拆票建议

日期：2026-11-14。结论：已有建档、人工修订、版本固定引用和审计基础可供后续设计复用，但尚不能证明学校重建闭环符合现行需求。建议分为 **12 张实施／验收票**，逐张审阅放行；三个已知缺口和一个新发现的问题仍需先取得业务答案。本报告不构成实施授权，不表示学校重建已获开工放行。

唯一业务依据：BR-BASELINE-20261114-v64（学校重建开工前必答项收口），重点为 BR-051（学校目录与数据变更）的 v61（重建范围、旧资料效力与责任归属）、v62（客户业务确认与学校模块挂起项收口）、v63（学校模块四个边角挂起项收口）、v64 及未被取代的历史条文；审计依据 BR-070（审计与一致性）。设计文档只作参考。以下状态是**静态差距分类**，不等于测试或本地验收结论；没有足够证据时明确写“未能确认”。

## 取证边界

- 文档仓库起点 `8c3431b`；应用仓库 `76a88eb2f853d8b4b6b5ea5d62a6aed93db915fc`；爬虫仓库 `25475e6eefb664165e37123a120715ce9b231a5d`。这是本机源码，不证明部署版本。
- 只读文件、Git 状态、源码检索及 JSON 结构／数量／哈希检查；没有启动服务、运行产品测试、连接数据库或访问真实网站。数据库结构仅由迁移源码判断，**实际是否应用、实际库存、RLS 是否有效均未能确认**。
- 应用有未跟踪 `AGENTS.md`；爬虫已有六份跟踪文件修改及一份未跟踪 `.env.example`，全部保留。不读取秘密配置内容；爬虫评估基于工作树，已有修改涉及配置及 LLM 客户端，运行效果未验证。
- v61 的“3 所合成学校”是已确认清除对象的历史事实，不把旧盘点当本次在线库存。本票未执行清除。

## 证据索引与主要读取文件

以下位置均为本机只读证据；行号用于定位当前源码。后表使用证据简称，不把文件存在当运行成功。

| 证据 | 主要文件与所见内容 |
| --- | --- |
| 需求 | [需求索引](/Users/karo/Documents/txgj-doc/business-requirements/README.md)、[学校全文](/Users/karo/Documents/txgj-doc/business-requirements/50-schools.zh-CN.md)、[审计规则](/Users/karo/Documents/txgj-doc/business-requirements/70-notifications-audit.zh-CN.md)；重点逐节核对 v61–v64，历史匹配、启用、恢复、空值与冲突保护规则 |
| 执行边界 | [本地计划](/Users/karo/Documents/Tianxingguoji/PROJECT_PLAN.local.md)、[工程路由](/Users/karo/Documents/Tianxingguoji/docs/README.md)、[历史库盘点](/Users/karo/Documents/txgj-doc/current-state-analysis/50-schools.local-db-inventory.2026-10-10.zh-CN.md)、[问题重分类](/Users/karo/Documents/txgj-doc/current-state-analysis/50-schools.rebuild-questions-reclass.2026-10-21.zh-CN.md)；计划与旧盘点不作当前运行证明 |
| 目录／建档 | [正式目录路由](/Users/karo/Documents/Tianxingguoji/app/api/v1/schools/route.ts:13)、[建档路由](/Users/karo/Documents/Tianxingguoji/app/api/v1/schools/provisionals/route.ts:25)、[建档服务](/Users/karo/Documents/Tianxingguoji/modules/schools/application/service.ts:346)、[建档仓储](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-provisional-repository.ts:15)、[目录仓储](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-directory-repository.ts:35) |
| 页面 | [学校列表](</Users/karo/Documents/Tianxingguoji/app/(erp)/schools/page.tsx:25>)、[学校详情](/Users/karo/Documents/Tianxingguoji/components/schools/SchoolDetail.tsx:52)、[建档面板](/Users/karo/Documents/Tianxingguoji/components/schools/ProvisionalSchoolsPanel.tsx)、[人工修改表单](/Users/karo/Documents/Tianxingguoji/components/schools/SchoolChangeForm.tsx)、[审核表单](/Users/karo/Documents/Tianxingguoji/components/schools/SchoolReviewForm.tsx) |
| 修订／审计 | [修订领域](/Users/karo/Documents/Tianxingguoji/modules/schools/domain/contract.ts)、[审批仓储](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-review-repository.ts:19)、[恢复相关服务](/Users/karo/Documents/Tianxingguoji/modules/schools/application/resolved-view.ts:145)、[变更历史读取](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/change-history-client.ts:11)、[变更提交](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-change-repository.ts) |
| 结构／引用 | [学校迁移](/Users/karo/Documents/Tianxingguoji/db/migrations/202608022030_004_expand_school_overlay.sql:1)、[建档扩展](/Users/karo/Documents/Tianxingguoji/db/migrations/202609190130_070_expand_provisional_school_intake.sql)、[审批回执](/Users/karo/Documents/Tianxingguoji/db/migrations/202609190150_072_record_school_change_reviews.sql)、[前值记录](/Users/karo/Documents/Tianxingguoji/db/migrations/202609190160_073_capture_school_change_previous_value.sql)、[案件引用仓储](/Users/karo/Documents/Tianxingguoji/modules/cases/infrastructure/postgresql-school-target-repository.ts:90)、[门户学校读取](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-portal-read-adapter.ts) |
| 权限 | [学校能力映射](/Users/karo/Documents/Tianxingguoji/modules/access/domain/contract.ts:156)、[试用判定](/Users/karo/Documents/Tianxingguoji/modules/access/domain/trial-policy.ts:58)、[服务端目录检查](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-directory-repository.ts:45) |
| 旧读取／计划 | [快照读取](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/crawler/server.ts:5)、[快照校验](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/crawler/snapshot.ts)、[旧配置数据库适配](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/crawler/db.ts:27)、[旧配置路由](/Users/karo/Documents/Tianxingguoji/app/api/crawler/config/route.ts:7)、[旧爬虫后台](</Users/karo/Documents/Tianxingguoji/app/(erp)/admin/crawler/page.tsx:88>)、[公共类型](/Users/karo/Documents/Tianxingguoji/types/index.ts) |
| 测试源码 | [快照门槛测试](/Users/karo/Documents/Tianxingguoji/tests/unit/crawler/snapshot-manifest.test.ts:107)、[学校审核测试](/Users/karo/Documents/Tianxingguoji/tests/integration/school-governance-workflow.test.ts:41)、[领域测试](/Users/karo/Documents/Tianxingguoji/tests/unit/schools/resolver.test.ts:323)、[建档断言](/Users/karo/Documents/Tianxingguoji/tests/integration/trial-provisional-school-assertions.ts)、[变更断言](/Users/karo/Documents/Tianxingguoji/tests/integration/trial-school-change-assertions.ts)、[审核断言](/Users/karo/Documents/Tianxingguoji/tests/integration/trial-school-review-assertions.ts)；均只读，未执行 |
| 爬虫身份／抽取 | [注册表生成](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/tools/build_school_registry.py:118)、[教育局列表 spider](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/school_crawler/spiders/edb_school_list_spider.py)、[结果选择器](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/tools/select_final_admission_records.py:438)、[招生身份](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/tools/admission_record_identity.py:43)、[词汇归一化](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/tools/admission_normalization.py)、[多招生测试](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/tools/test_multi_admission_records.py:30) |
| 爬虫交接／运行 | [发布工具](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/tools/publish_run_outputs.py:19)、[计划运行工具](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/tools/run_scheduled_admission_crawl.py)、[差异工具](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/tools/diff_final_records.py:25)、[旧值保留工具](/Users/karo/Documents/school-tracker/hk-school-platform/apps/crawler/tools/carry_forward_previous_records.py:82)、[爬虫工程索引](/Users/karo/Documents/school-tracker/hk-school-platform/docs/AGENTS.md)、[教育局抓取计划](/Users/karo/Documents/school-tracker/docs/EDB_CRAWL_PLAN.md) |

## 一、现行实现差距

“部分符合”只表示找到对应的部分源码；“不存在”仅指本次列明范围未找到该链路，不声称全仓绝对不存在。**本次没有任何完整业务流程可评为“已符合”**，实际本地运行均未能确认。

| 业务要求 | 静态判断与业务结论 | 证据及限制 |
| --- | --- | --- |
| 教育局名单导入；D4（学校对象粒度）按编号前六位 | 部分符合：爬虫有教育局取名单与注册表工具，但应用仍以内部学校及来源 key 承载，未能确认从名单到前六位学校身份的一致导入约束 | 爬虫身份／抽取；结构／引用。注册表以名称／官网生成 key；学校迁移第 17 行唯一索引按来源 key，不等同于教育局前六位唯一学校。没有运行导入 |
| 手工建档与无教育局编号学校，v64 R5（D2 剩余：无教育局编号的学校） | 部分符合：中英文至少一个非空、内部编号、无教育局编号建档已有源码；官网增改后无编号手动更新／每日入计划、无网址禁用及提示链路不存在于已查正式学校路由 | 目录／建档：建档请求只允许名称、地区、学制、学段、原因；建档记录与 current resolved 读取分离，不能据建档成功推断立即出现在选校中 |
| 招生生成、分条、未知及无最低门槛，v64 R3（B 类 04：招生候选认定）、v62 C5（候选最低有效条件） | 部分符合：爬虫已有多条及字段证据，应用详情仍一套招生字段；爬虫选择器在无合格招生 URL 时仍返回 build_record(rows)，不等于“不生成资料”；对“识别出但分不清生成一条”和缺年级／学年可启用，未能确认端到端覆盖 | 爬虫多招生测试包含中一与插班分条；结果选择器 1420 行按校收集多条；详情第 52 行仍单套字段。旧 normalizeSchool 将招生类型限为两类／unknown |
| 忽略，v62 P13（不采用的变化） | 不存在：已查学校正式命令及页面未找到资料变化的忽略和不再提醒闭环 | 页面、修订／审计。旧 review decision／工单处理不是新业务忽略，不能替代；爬虫不应自行定义顾问忽略状态 |
| 未知学年重抓，v64 R4（B 类 05：学年未知的重复抓取） | 不符合：爬虫仍按未知维度的新观察追加，尚无可证明按已确认比较口径避免相同内容重复的链路 | 招生身份 reconcile_records 第 234 行及多招生测试第 186 行。差异工具有 unchanged_count，不等于顾问忽略保留及新候选判定；比较口径仍待答 |
| 中英两条、英文默认及未知语言对应，v62 P3（同一招生的认定）、P4（语言未知）、P5（后续更新对应）、v63 Q4（未知语言的后续对应） | 不符合：匹配键没有语言限定，已知三项相同会合并；未知语言后续归属与英文默认展示未能确认 | 招生身份 exact_n2_key 第 43 行返回学校、学年、类型、年级；reconcile 合并同键；选择器及详情未找到语言记录展示对应。不能用不同 URL 冒充语言 |
| 逐字段启用／恢复与独立履历，v64 R6（第一类内部 15：启用／恢复的作用范围） | 部分符合：字段修订、前值、审批、不可变有效版本及审计事务已有基础；现有禁用 overlay 修订不等于逐条招生逐字段恢复任意旧值 | 修订／审计、结构／引用。没有独立招生对象定位的完整应用链路；不能把整 revision 禁用当作本次要求的字段恢复；并发与事务只见源码，未验证 |
| 人工资料与来源增改免审批，v62 P7（U1–U3 范围）／P8（抓取来源维护审批） | 不符合：提交仍产生 submitted／candidate，等待审批；官网修改也走人工修订入口 | 页面、修订／审计；审批仓储显式禁止自审，详情显示“申请修改”和待审批。不能靠移除按钮解决服务端生效方式 |
| 默认每日计划、单校手动发起、连续 7 次失败暂停及暂停显示，v62 P16（逐校计划维护）、v63 Q1（连续失败阈值） | 不存在：未找到应用的逐校抓取任务／计划／失败计数和恢复闭环；旧全局 enabled/frequency 不能替代逐校设置 | 旧读取／计划；检索 modules/schools、app/api/v1/schools、workers。爬虫有批次运行脚本，不证明消费应用计划，未见应用消费者证据 |
| 已停办标记及标记动作停更、可保留标记恢复；教育局再出现提示，v62 P11（学校停用／退场）／P12（退场后抓取）、v63 Q2（停办停更与恢复更新的关系）／Q3（教育局名单再出现） | 不存在：已查正式学校 API、详情与 worker 未找到此信息标记、更新联动或名单比较提示链路 | 页面、旧读取／计划；不得把 overlay disabled 或学校 registration_status 当已停办标记；不得借此限制选校 |
| Founder／L1／L2／L3 学校同等，v62 P10（等级交界） | 不符合：L3 能力表无 schools.read；L2 与 Founder/L1 的已有值修改权限不同 | 权限：能力映射第 167 行、目录检查第 48/56 行；建档第 54 行检查能力。正式路由直接使用 PostgreSQL 仓储，不能因旧 runtime 文件抛错就说全部接口不存在 |
| 「暂无招生数据」「未找到招生页面」 | 不符合：已查详情固定招生字段、通用空态，没有两类业务结果显示闭环；如何并列／取舍仍待业务决定 | 页面第 52–54 行；旧快照读取仍存。新结构空值不得从 585 校旧原件回填；两种文案关系未答，不设计默认优先级 |
| 原件归档、无并行读取与无旧值回落，v61 | 部分符合：正式目录从 PostgreSQL resolved 读取且不回落文件；旧 crawler API 仍加载文件快照并保留上次 active 内存结果 | 目录／建档、旧读取／计划。应用归档 records.json 实测 585 条，SHA256 `1736b8a83e5066eb068834869e6b17bfb7902351a045f0241274929d2d6c5f0f` 与 v61 一致；这不证明其产品读取已退出 |

## 二、五样旧机制退出的边界

v64 R2（第一类内部 12、19、20、21、22：旧机制随重建一起退出）要求退出业务使用与展示、保留历史，不授权修改历史迁移或物理删结构。下表“数据库”仅指源码结构；未查实际数据库。

| 旧机制 | 代码／页面及权限 | 数据库结构／测试证据 | 学校以外连带影响（只报告） |
| --- | --- | --- | --- |
| 旧爬虫页面及读取口径 | 旧爬虫后台、app/api/crawler 下 schools/summary/review-queue、crawler/server.ts 的快照读取；config 仍限 founder/admin；正式学校列表也复用旧 AdmissionRecord 展示模型，不能只去掉旧 URL 就算完成 | crawler/db.ts 在请求链内建 crawler_config、工单、review decision 等旧表；快照测试。正式 resolved 链路另有学校表 | 公共 types/index.ts 及导航／会话接入是共享边界；Cases 固定 resolved 引用和门户学校名称读取仍依赖 Schools，不可等同退出 |
| v1 warning→active 门槛 | snapshot.ts 的 validateReceipt 要 reviewer 推荐及 Founder 接受，warn 无回执拒绝加载；爬虫发布工具同样验证。它是文件激活门槛，不等于新资料启用 | 此回执在 manifest JSON，未找到专用 SQL 表；snapshot-manifest.test.ts 第 107/122 行直接断言旧门槛 | 生产／发布校验若复用 manifest 会受影响；不能由本报告决定发布健康校验、账号权限或其他模块安全门槛一起退出 |
| 学校 Data Reviewer 审核路径 | domain/contract.ts 允许 data_reviewer；governance 测试仍认可。当前正式审核仓储又按 Founder/L1 控制，旧新链路并存 | 学校迁移 approved_role 与 RLS 留有该角色；后续 trial／review 迁移另有规则；school-governance-workflow、resolver 测试仍有该角色 | identity/session-service.ts 第 252 行、access 角色类型与能力兼容、cases/transition-policy.ts 第 21 行、identity/tasks 迁移也包含角色。**仅报告，不提出删除或改造这些共享角色的方案**；BR-015（K12 试用等级与授权）表述不动 |
| 重复 school_type 业务展示 | schools/page.tsx 第 51/123/138 行搜索、取值与展示；crawler normalizeSchool 保留；公共 AdmissionRecord 有字段 | 快照及 fields_json 承载，学校 SQL 未见独立 school_type 列；旧原件必须保留。已读测试未找到专门覆盖当前类型展示退出的验收，未能确认覆盖完整性 | 公共类型、国际／本地业务分类命名易混淆；不能把学校类型展示退出扩展为案件分类或权限分类退出，不给跨模块处理方案 |
| confidence 业务展示及计算 | 列表第 31/66/140 行过滤和徽章；crawler summary 第 121 行计数；i18n/zh-TW.json 第 61/84 行文案；爬虫 selector 第 1084/1256 行计算并决定 review_status | JSON 及 fields_json 承载，无专用评分 SQL 列证据；旧 warn 测试不能证明评分退出；需未来新增针对性验证，本票未写测试 | 共享类型及原始归档评分、爬虫 review_status 与发布统计被牵连；旧“approved/auto_selected”不得当新启用事实，不能据此改其他业务审批 |

## 三、爬虫与应用衔接及参考设计冲突

爬虫不是从零开始：现有工具会输出多条招生、学年／年级归一化、证据与采集观察；应用仍把目录字段转换为旧 AdmissionRecord，学校详情是一套字段。仅复制新 JSON 不能形成逐条候选、忽略、生效与恢复闭环。

| 差距 | 爬虫侧责任边界 | 应用侧责任边界 |
| --- | --- | --- |
| 学校身份 | 保留教育局原编号／来源／校址证据，不能把名称 key 当学校身份答案 | 按前六位确定学校粒度、内部身份；人工学校后续对应遇到未决项即停止，不猜绑 |
| 多条、未知及语言 | 输出原始条目、来源与未知事实；当前精确键合并可能吞语言版本，须在后续契约票评估 | 保存多招生和语言限定身份，候选与有效值分开；英文默认、未知语言路由、无额外资格选学校 |
| 重复抓取 | 当前 observation token／diff 的比较不能替业务尚未确定的“完全相同”；抓取时间不等于学年 | 保存顾问忽略及人工保护；不得拿旧爬虫 review_status 当新状态，比较决定未答时停止此分支 |
| 结果与生效 | 报告成功／失败／无变化和实际采集时间，保留原始来源，不自动代表业务启用 | 逐字段差异、全部接受、并发校验、人工冲突保护、旧值／独立履历与恢复，审计原子性 |
| 更新计划 | 执行已批准的单校任务；批次定时脚本本身不是应用逐校计划 | 逐校计划、同校共用进行中任务、七次失败暂停、停办标记动作和人工恢复、页面状态 |
| 交接包 | publish_run_outputs.py 仍发布 crawler-handoff/v1 的四文件与 warn 回执；另有 v2 草案及离线校验器 | 不能绕过结构／哈希验证，但旧激活回执必须与新业务启用分开；契约需重新审阅后方能接入 |

既有应用归档只证明旧 585 校原件存在；本机 `apps/crawler/output/published/latest/records.json` 未找到，不能报告“最新发布包已就绪”。已见 2026-06 分片和十校历史产物及多招生测试源码，不证明当前真实站点效果。**国际学校网站识别效果未经验证**；本票没有新抓取，也不从测试名、历史数量或计划推断覆盖率。

参考设计逐项适用性：

| 设计文档（首次编号带原文标题） | 与当前基线的冲突／限制 |
| --- | --- |
| SCH-INT-01（学校数据对接方案 V1） | 基于旧审批安排；人工补充／来源审批与 v62 免审冲突；自动周期更新“不纳入”不能覆盖后续允许决定。状态 draft_for_review，不是授权 |
| SCH-INT-02（人工建档学校的来源关联与抓取资格设计） | 提议 Founder 审批关联及以适配器支持作为入口条件，不能替代现行免审和无编号官网规则；首次对应职责仍未答，不采用其建议答案 |
| SCH-INT-03（无爬虫快照学校的有效版本补充设计） | 手工基线与统一版本可供技术参考，但不能用“没有 resolved 就不可选”的当前限制定义业务资格；按 v62 已确认选学校、无额外资格核对；设计仍 proposed |
| SCH-INT-04（学校待确认项阻塞分类清单） | 按旧基线登记的待答项已被后续基线大量收口；来源审批、退场、最低候选条件等旧状态不可直接照抄。剩余问题按 v64，不以参考阶段放行 |
| SCH-INT-05（D4 / N2 真实抓取数据决策材料） | 历史证据不是当前网站验证；其 D4/N2 未决定表述已过时。不能用不同日期、过滤范围、分片数量作当前全量证明 |
| SCH-INT-06（Stage A 第一块：单校结果契约草案与校验记录） | v2 草案按 v56 要求未知学年每次新候选，缺后续双语限定、同内容不新建等规则。11 合法／41 非法样本及当时离线通过只证明当时草案，需对照 v64 重审；不得据旧测试通过宣布集成完成 |

其中 N2（同校招生记录身份与跨批次对应）及 D4 已由历史 confirmed 条文确定的部分不重开；普通 URL 等价变化不是新身份，不能把 URL 增加为第四匹配条件。

## 四、建议票与独立关卡

下列数字仅为本报告建议顺序，不新设正式票号。每票需独立批准，后端先定义契约、唯一写入负责人、停止条件与验证命令，前端不得自创接口。QA 只读验收。技术草案不得越过 pending_review。不可逆“是”不表示本报告允许执行。

| 顺序／目标（一句业务话） | 基线依据、仓库与写入负责人／大致范围 | 前置与验证方式 | 风险、停止条件及回退；不可逆操作 |
| --- | --- | --- | --- |
| 1 让两端对同一学校和招生资料说同一种话 | 现行 BR-051，v64 R3–R6、v62 双语及 v63 对应；两仓契约／合成样本；architect 负责契约，后端负责应用、爬虫负责人负责另一仓，文件写入范围互斥 | 先确认三个已知缺口的答案或把相关分支明确挂起；离线校验结构、证据、版本与拒绝样本，无真实网络 | 任何新业务枚举、身份／内容比较暗含答案即停；回退草案版本，不改历史原件；不可逆：否 |
| 2 让新学校结构能容纳独立招生与履历 | v61 空起点、v62 两条、v64 R6；应用新增结构／仓储、追加迁移设计；backend | 依赖 1 的已答契约；离线审阅 SQL、引用及回滚计划；获单独执行授权后才在可丢弃 PostgreSQL 17 用独立连接验 RLS、事务、隔离；真实网站不需要 | 触及其他模块引用／共享角色只提依赖请求，不擅改；回退新结构发布版本，实际删结构另批；不可逆：本票不含删除，执行迁移另行批准 |
| 3 单独清除已确认的三所本地合成学校 | v61 清除决定、v64 R1 前提；应用本地开发库；platform-operations 执行，backend 提供经审阅工具 | 依赖 2 已审阅且取得**当次清除批准**；执行前另做只读库存比对／备份、明确对象与引用。无真实网站 | 数量、seed 标识或引用偏离历史对象立即停；不扩大到其他学校；回退只能依经验证备份恢复；**不可逆：是，独立票，不与其他写入混合** |
| 4 让教育局名单和顾问建档进入可用目录 | v55 首次导入、v56 D4、v62 P1/P10、v64 R5；应用身份／建档／来源、爬虫名单适配；backend 主责，爬虫范围单独写入 | 依赖 1/2；离线合成六位编号、多校址、无编号官网／无网址、最低名称与权限校验；真实名单只到票 12 才取 | 绕不开 03 或 D1 首次对应就停，不自动认领／合并；可撤回新代码，保留履历；不可逆：否，不含旧数据迁移 |
| 5 让爬虫交回分条、未知与语言来源资料 | v64 R3/R4、v62 P3–P5、v63 Q4；school-tracker 选择／归一化／输出；爬虫负责人 | 依赖 1、4 身份契约；离线页面 fixture、双语／未知、非招生／无页面样本；真实效果票 12 | H3、07、08、16 等若无法避开则停该分支；不得把评分当可靠性依据；回退产出版本，保留原始证据；不可逆：否 |
| 6 让未知学年重抓和忽略不反复制造待办 | v64 R4、v62 P13；两仓交接／应用候选与忽略；backend 负责事实存储，爬虫负责人负责观察输出 | 依赖 1/5，**完全相同比较口径须先回答**；离线相同／变化／已忽略／重复投递与并发测试；真实网络非必需 | 不得用内容 hash 自行定义业务等价；旧值不覆盖；回退处理器，保留候选历史；不可逆：否 |
| 7 让顾问直接增改、逐字段启用和恢复 | v62 P7/P8/P14、v64 R6、BR-070；应用命令／仓储／历史 API；backend | 依赖 2/4/6；离线静态与领域测试，批准后真实本地 PG HTTP 验并发、重复请求、空值／人工冲突、失败回滚；无网站 | 独立招生／学校字段不得串改，来源纠错触及 D3 时停；回退代码与新版本，不能删除履历；不可逆：否 |
| 8 让逐校更新、停办与失败暂停可控 | v62 P11/P12/P16、v63 Q1–Q3、v64 R5；应用任务／计划／worker及爬虫执行入口；backend 主责 | 依赖 4/5，须回答本报告新增失败计数问题；离线时钟／合成执行器验同校共用任务、每日限制、七次失败、不通知、恢复不取消标记；授权后本地链路；真实调用票 12 | 延后的教育局提示关闭条件不得补默认，名单比较可提示但不实现未决关闭分支；回退调度器并保留任务履历；不可逆：否 |
| 9 随重建退出五样旧机制的学校业务使用 | v64 R2；应用学校入口／读取／权限／展示／定向测试及爬虫旧业务评分路径；backend 为主要写入负责人，frontend／爬虫范围分开交接 | 依赖 4–8 的替代链路可集成，发布顺序保证无旧值回落；离线引用扫描与源码测试、授权后本地回归；无网站 | 影响账号角色、Cases、门户等共享规则先停并报边界，本报告不提出其处理方案；保留旧原件与历史迁移；回退应用版本需重新确认旧路径业务效力；不可逆：否，**不含删表或删角色** |
| 10 让学校页面展示可用资料、差异和履历 | v61 空态、v62 四块详情／英文默认／等级同等、v63 暂停和名单提示、v64 R3/R5/R6；应用页面／文案；frontend | 依赖后端 4/6/7/8/9 契约与自测；**两空态显示关系须在本票前回答**；离线组件及类型检查，授权后真实本地浏览器验正常／空／拒绝／失败／响应式及键盘访问；不访问真实学校站点 | 不把按钮隐藏当权限，不实现未决提示关闭；回退页面版本不改业务事实；不可逆：否 |
| 11 独立验证完整本地流程及共享引用保护 | 现行 BR-051、BR-070、v61 第 8 条核对责任；应用与爬虫离线合成环境；qa 只读 | 依赖 4–10 前后端自测后确认可集成；逐票冻结真实本地 PG／HTTP／浏览器命令，核实脚本实际覆盖文件；Founder 逐条核对权限，结构由实施方报告、Founder 抽查 | 跑不起来报 blocked，不以模拟、云环境或构建顶替；QA 不改源码／数据库／Git；失败回对应 owner；不可逆：否 |
| 12 经清单批准后验证真实来源试运行 | 教育局名单、单校更新与真实效果；两仓；platform-operations 负责获准执行，qa 只读取证 | **先通过 1–11 的离线／本地关卡，再交抓取清单待批准，批准后才运行**。清单列教育局页／学校页／PDF、校数、频率、资源上限、原件存放、停止阈值、导入是否另批；新范围再次审批 | 未批准不访问；国际识别仍未经验证直到取得实际证据；未知业务不试做；中止任务保留证据，不把抓取成功当启用；不可逆：真实网络外部副作用须单独授权，任何数据导入另行批准 |

建议依赖主线：1 → 2 → 3（独立清除关卡）；1/2 → 4 → 5 → 6 → 7；4/5 → 8；4–8 → 9 → 10 → 11 → 12。3 未获批准不执行清除，不能用新结构存在宣称重建完成。清除的恢复方案尚未演练，不承诺可逆。

已知三个缺口的最晚答复点：两种空态并存／取舍在票 10 开始前；完全相同比较口径在票 6（或票 1 提前冻结该分支契约）前；有官网但无法识别的专门提示在票 8/10 设计该专门分支前，未答时只限现有 R3 与失败暂停处理，不另造状态。晚答不会允许票 1 先用默认值冻结接口。

## 五、新发现的业务问题与停止点

本次新发现 **1 项**，只是待负责人判断的调查发现，不自动写入正式 BR：一次单校更新“部分字段成功、部分页面／招生失败”在连续七次失败计数中算失败还是成功，何种结果打断连续性？已有 v63 Q1 给出次数，但未定义混合结果计数；契约样本已有 partial，故执行器选择会悄悄改变暂停时机。**不选择、不推荐默认值**；票 8 的失败计数分支及票 1 对应结果含义冻结前停止，回项目负责人确认。

以下 9 项不是新发现，不重复回答：03（学校身份对应）、D1（人工来源/纯外部键关联）首次对应、07（旧 token 业务含义）、08（资料归属）、16（来源配置与证据）、24（派生值与原文）、D3（身份疑点、来源变化）、H3（语言/版本证据如何认定）、“教育局名单中仍有此校”提示能否忽略／关闭及何时消失。按 v64 试运行后再定；任何票绕不开就停止对应分支，不以人工处理能力为理由自动认领、映射、合并或清理。明确缺口不等于整项可以先照旧设计实现。

## 自检与交接

- 读过的主要文件见证据索引；六份参考设计均核对开头依据、状态及所涉规则，未将 proposed 当授权。
- 未能确认：当前数据库库存与结构应用、RLS／事务实际效果、完整本地链路、爬虫最新发布包、实时教育局名单范围、国际站点识别效果、所有角色真实浏览器可用性、逐字段恢复完整性、停办／计划／忽略运行效果。没有把其中任何一项写为已符合。
- 建议 12 张票，顺序和依赖见上表；新发现业务缺口 1 项；已知三个缺口及延后九项保持待答。
- 本次只新增本报告。三仓库其余内容零改动：应用 Git 跟踪差异为空且未跟踪 AGENTS.md 保留；爬虫原有状态与跟踪差异校验和前后相同；需求模块、旧分析和所有抓取原件未修改。此检查证明本次可观察的 Git 差异未变，不证明数据库或网站当前状态。

```text
status: pass
owner: architect
scope: 本份只读评估报告
changed: 仅新增本报告，无实现改动
evidence: 源码/迁移/文档静态核对、JSON 数量及哈希核对、三仓库 Git 差异检查、报告链接检查
not_run: 产品测试、数据库连接、清除、浏览器、真实网站、抓取、部署、业务验收
risks_or_stop_conditions: 每张实施票另行批准；新旧未决问题不得默认填答；不构成学校重建开工放行
```
