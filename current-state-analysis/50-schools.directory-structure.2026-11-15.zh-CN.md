# 学校重建：目录结构实施记录

**status: blocked（目录结构及既有基线安装方式验证通过；必查的旧选校页失败）。**上一票的安装权限阻塞已消除：本次按仓库既有做法，把新迁移纳入完整基线，并在安装事务内临时开放、随后收回引用权限。常用本地库已删卷重装为 75 源基线、88 张表，四张新目录表为空。Founder 登录、学校列表、学校详情已实际通过，三所合成学校仍在；旧选校页返回 500，不能报告整票 Local Dev accepted，也不扩大本票修复旧链路。

本报告于 2026-11-16 更新，覆盖原票及安装方式续票。业务依据为 BR-BASELINE-20261115-v68（学校字段归属与校址维护细节收口）、BR-051（学校目录与数据变更）未被取代条文、BR-070（审计与一致性）、BR-015（K12 试用等级与授权）；目录约定第四节与分期方案二仅作接缝参考。本票不接入新页面、不调整业务权限、不处理招生及更新后半段。

两仓库均继续 `codex/schools-rebuild-directory-structure`。应用基点 main 为 `76a88eb`，原票提交 `bc2c423`，本次续票提交 `4729da6`；文档基点 main 为 `0894e15`，原报告提交 `9e033fd`。开工先获取远程并执行 `git merge origin/rebuild/schools`，集成分支最新为 `76a88eb`，返回“Already up to date”；已包含其最新内容，无合并提交。最终验收后的合并目标为 **rebuild/schools 集成分支**，本次不合并任何目标、不建 PR、不部署。owner 为 backend，负责结构、脚本与自测；独立 QA 未执行。

## 一、八项结果与证据边界

下表“已达到”限本票结构与必要数据访问范围。新表已在常用本地库安装，结构业务写入约束使用一次性本地 PostgreSQL 17 合成库验证；常用库不写新目录业务数据。既有页面验收结果另见第六节。

| 本票结果 | 结论 | 业务结果及限制 | 技术证据（Tianxingguoji） |
| --- | --- | --- | --- |
| 1. 学校身份与最低建档要求 | 已达到（结构范围） | 内部编号稳定；六位教育局编号可未知，同一组织同一编号不能建第二所；中英文名不能皆空；名单／人工建档来源可区分。未建立首次导入入口 | 新迁移第 14 行学校表、身份不可修改触发器；新增 PostgreSQL 测试的名称／编号用例 |
| 2. 一个或多个校址／时段 | 已达到（结构范围） | 提交时至少一个未删除校址；添加校址不新增学校；来源不可静默改写。名单校址不可删、最后一个不可删，包含并发删除验证；人工校址删除保留实体与履历 | 新迁移第 33 行校址表、延迟约束及父学校锁；测试的添加、删除、并发删除用例 |
| 3. 23 字段归属与未知 | 已达到（结构范围） | 12 项整校、11 项按校址／时段，错放字段被拒。业务值可未知；内部编号必有，名称最低条件仍生效，不能借“未知”绕过它 | 新迁移字段白名单／类型约束；`modules/schools/domain/directory-structure.ts`；字段归属及未知值测试 |
| 4. 共用值与按校址分开 | 已达到（结构范围） | 每一项校址字段明确记载共用／分开；共用修改作用于各有效校址同一项，改分开后可独立维护。没有把共用方式写为业务默认值 | 新迁移第 46 行维护方式表及延迟一致性约束；内部仓储 `setSharedField`、`splitField`；共用改分开测试 |
| 5. 逐字段履历与恢复 | 已达到（结构范围） | 同事务记录前后值、来源、实际操作人、时间、版本；审计失败则修改回滚；过期版本拒绝。恢复只改对应字段，校址分开时不影响另一校址；共用字段恢复仍只改这一项。删除校址不真删履历，禁止改写／删除历史 | 新迁移第 54 行历史表及写入／历史保护触发器；内部仓储 `restoreField`；审计失败、跨组织上下文拒绝、恢复与不可改写测试 |
| 6. 已有申请固定引用不变 | 已达到（结构范围） | 不改任何旧学校、旧版本或案件引用表；隔离库真实建立一条非空的合成案件学校固定引用，修改、恢复、撤回后摘要均相同。学校列表、详情已实测；旧选校页失败，见第六节。未把新学校接入选校 | 新增 PostgreSQL 测试的旧引用准备及前后摘要；既有学校测试 main／本分支均 66 项通过 |
| 7. 已停办为信息 | 已达到（结构范围） | 独立保存可未知的停办信息并留历史；不与校址删除、目录可选资格或更新计划绑定。本票没有更新计划结构 | 新学校表 `is_closed`（停办信息）；停办修改与逐字段恢复测试 |
| 8. 官网及名称比较的结构支持 | 已达到（结构范围） | 原始文字、网址及前后值原样存储，不使用名称或网址作为学校唯一键，不强制按网址原文决定共用方式。后续能按已确认口径比较，无须推翻身份结构；本票没有比较器、归一化键或提示 | 字段为文本／未知、内部编号主键与教育局编号唯一约束；共用改分开用例保留不同写法的原始官网 |

第 8 项沿用 v68 X3（官网）、X4（名称比较的剩余差异）及 v67 W1（名称「完全相同」的比较口径）。不建名称／网址唯一身份键、不破坏原文；本票不实现比较器或提示。结构中的共用方式不是业务相同判定：后续可以按已确认口径比较原文，再选择共用维护，不必推翻学校身份。

## 二、新增结构及 23 字段对应

| 保存的业务事实 | 新增技术位置 | 保护方式 |
| --- | --- | --- |
| 学校稳定身份、整校资料、建档来源、停办信息和版本 | `schools_directory_schools`（新学校表） | 内部身份／来源不可改；教育局六位编号唯一；名称最低条件；组织外键 |
| 各校址／时段的来源、资料、删除留存和版本 | `schools_directory_campuses`（新校址表） | 学校／组织复合关联；名单来源不可删；至少一个有效校址；逻辑删除保留历史 |
| 每项校址资料共用还是分开维护 | `schools_directory_field_modes`（字段维护方式表） | 11 项完整、显式提供；共用值一致；方式变更留历史 |
| 每次字段修改和恢复证据 | `schools_directory_history`（逐字段历史表） | 同事务记录前后值、来源、实际操作人、时间、版本；历史只追加 |

四表均强制行级安全（RLS），没有新增应用策略或业务访问入口。单角色基线下应用身份同时为表所有者，故查询新表会被强制 RLS 过滤为空，**不是上一票独立管理员建表时的表级权限错误**；测试同时确认管理员可见合成记录、应用身份不可见且没有策略。未授权应用读写新目录。

字段集合的缺项／显式空值表示未知；布尔项可未知，不把缺资料推断为否。内部编号必有，名称最低条件不能被未知值绕过。来源、停办、维护方式和版本为配套信息，不冒充第 24 个业务字段。

| 序号 | v68 字段原文 | 归属 | 技术落位 |
| --- | --- | --- | --- |
| 1 | 系统内部编号 | 整校 | 学校表 `id` |
| 2 | 爬虫来源编号 | 校址／时段 | 校址字段 `source_school_key` |
| 3 | 教育局学校编号 | 整校 | 学校表 `edb_number` |
| 4 | 注册编号 | 校址／时段 | 校址字段 `registration_number` |
| 5 | 校址标识 | 校址／时段 | 校址字段 `location_marker` |
| 6 | 中文名称 | 整校 | 学校字段 `school_name_zh` |
| 7 | 英文名称 | 整校 | 学校字段 `school_name_en` |
| 8 | 官网 | 校址／时段 | 校址字段 `official_website` |
| 9 | 学校资料页面地址 | 校址／时段 | 校址字段 `profile_url` |
| 10 | 学段 | 整校 | 学校字段 `school_level` |
| 11 | 地区 | 校址／时段 | 校址字段 `district` |
| 12 | 地址 | 校址／时段 | 校址字段 `address` |
| 13 | 电话 | 校址／时段 | 校址字段 `phone` |
| 14 | 传真 | 校址／时段 | 校址字段 `fax` |
| 15 | 学生性别类别 | 整校 | 学校字段 `student_gender` |
| 16 | 授课时间类别 | 校址／时段 | 校址字段 `school_session` |
| 17 | 资助类别 | 校址／时段 | 校址字段 `finance_type` |
| 18 | 特殊教育标记 | 整校 | 学校字段 `is_sen` |
| 19 | 学费说明 | 整校 | 学校字段 `tuition_info` |
| 20 | 学费适用学年 | 整校 | 学校字段 `tuition_year` |
| 21 | 学费来源链接 | 整校 | 学校字段 `tuition_source_url` |
| 22 | 是否提供住宿（是／否／未知） | 整校 | 学校字段 `has_boarding` |
| 23 | 住宿说明 | 整校 | 学校字段 `boarding_info` |

汇总：整校 12 项（两个身份列、十个学校字段），校址／时段 11 项。共用改分开只改变维护方式及所选字段的值，不重建学校。逻辑删除校址仍保留其历史主键；不提供整校回到某时间点的操作，符合 v64 R6（第一类内部 15：启用／恢复的作用范围）。

## 三、对现有功能与后半段的影响

现有页面、路由、学校服务组合入口、旧学校结构和案件引用均未改。新仓储只供内部显式事务使用，不从学校公共入口导出。原学校列表、详情和选校仍走各自旧路径；本票既不切换它们，也不以旧流程定义新业务规则。角色平权、五样旧机制退出、页面及首次切换另票处理。

后半段可以引用稳定学校编号、原始校址网址／来源事实及独立逐字段历史；本票没有预建招生、候选、抓取输出、更新计划、失败计数、网址任务路由或提示状态。不必推翻学校／校址归属，但不保证后续无需追加结构。招生来源网址与多官网更新方式未在这里选默认值。

未发现本票结构必须二选一的新业务缺口。继续保留 03（学校身份对应）、D1（人工来源/纯外部键关联）的剩余学校对应、D3（身份疑点、来源变化）的更新影响、多官网更新处理、疑似重复提示后续及停办学校名单再现提示生命周期；本票不放行这些分支。

## 四、两处偏离如何改回、重装与撤回

**权限改回既有安装窗口。**核对 066（任务文档关联）、070（学校人工建档）、072（学校变更审批回执）、075（邀请操作记录）的 SQL；076（学校目录结构）照其 `current_user='tianxing_app'` 条件，在创建外键前授予组织表／用户表 REFERENCES（建立外键引用）权限，建表后收回。基线执行器将全部 SQL 放在同一事务内，失败整笔回滚。安装前两项均为 false，安装后仍均为 false；新增 PostgreSQL 测试还独立验证同一身份安装前后权限一致，不改共享表结构、不新增永久权限。

**安装方式改回完整生成基线。**生成器严格校验逻辑恢复为 main 原文，只把源数量 74 改为 75、更新完整源清单校验码；做法与提交 `bc1b657` 加入 075 时一致。重新生成清单与新 SQL，原有生成 SQL 未改。删除上一票 `scripts/db/school-directory-local.ts`，不再保留专用增量安装、专用回退路径。076 为纯向上安装 SQL，测试专用的删除新结构语句仅留在一次性测试中，不混入生成基线。运行手册只增加 75 源、76 生成文件、88 表、四新表为空的说明。

按用户本票明确授权执行删卷重装，目标是本地 `tianxing-local` Compose 项目、回环 `tianxing` 库及 `tianxing_app` 身份。没有读用三份 `.env.migration.neon-*`，也没有修改环境文件。以下步骤顺序执行，均实际完成：

| 顺序 | 命令／动作 | 结果 |
| --- | --- | --- |
| 0 | 对重建前旧库只读核对；新代码下运行 `pnpm db:baseline:local:dry-run` | 原为 74 源、84 表，旧清单校验码不匹配；空跑在数据库前置检查拒绝，事务未开始。非空旧库本来也不允许重复安装 |
| 1 | 停止已有本地 Next.js；`pnpm local:down --volumes` | 通过；删除旧本地合成数据库卷及会话，这是用户本票授权的重建，不是执行独立三校清除票 |
| 2 | `pnpm local:up` | 通过，PostgreSQL 17.10 健康 |
| 3 | `pnpm db:plan:local` | 通过，列出 75 源、76 生成文件 |
| 4 | `pnpm db:baseline:local:dry-run` | 通过；整笔安装回滚，独立后检 public 对象 0、无 marker 残留 |
| 5 | `ONE_ROLE_BASELINE_APPLY_CONFIRM=tianxing-one-role-v1 pnpm db:baseline:local` | 通过；75 源、76 文件、657 个 public 对象；错误 owner、未强制 RLS、不安全 definer 均为 0 |
| 6 | `pnpm db:seed:local-release1` | 通过；目标证据为 local-loopback；88 表、3 所合成学校、3 条合成学校快照记录 |
| 7 | `pnpm db:seed:local-trial-demo` | 通过；5 principal、2 case、2 task |
| 8 | Founder 账号密码 provision | 通过，`founder@env01.test.invalid`，按用户指定本地密码通过标准输入设置。最初未提供标准输入的一次调用在 password_input 阶段拒绝、未开始事务；随后正确提供输入创建成功 |
| 9 | 安装后独立数据库核对 | 75 源、88 表；四表均为空且强制 RLS；组织／用户表 REFERENCES 两项均 false，与重装前一致 |
| 10 | `pnpm dev`，本地 health／readiness | 启动通过；两接口均 200，readiness 为 ready，四项依赖均 ready |

空表核对由表所有者在事务内逐表暂时取消 FORCE RLS 后数记录，每次立即 ROLLBACK，避免把默认隐藏后的零行误报为空表；复核强制 RLS 全部恢复。没有留下策略或权限改动。

撤回沿用运行手册的本地合成库受控重建，不提供已安装库增量降级。一次性 PostgreSQL 测试实际删除四张新表及六个函数、核对非空旧案件固定引用摘要不变、重新建空结构，并验证应用所有者能在安装窗口后收回权限。该删除 SQL 只在可丢弃合成测试库运行；常用库未直接删除新表。若需回到旧基线，只能另按当时放行和旧代码执行受控本地重建；本票没有实际演练退回旧基线，不将向下 SQL 测试等同于该演练。

## 五、集成目标与旧基线数据库的影响

**本票及后续学校票验收后合入 rebuild/schools，不合入 main。**本票只更新并推送工作分支，未合并；集成分支也未改。将来集成分支何时进入 main，须独立处理旧库基线兼容与部署关卡，本报告只陈述证据、不处理线上库。

| 问题 | 核对结论 | 技术证据 |
| --- | --- | --- |
| 旧库会被哪些脚本拒绝 | 基线执行器只接收空 public，无旧 marker；对旧已安装库重跑会拒绝，并非升级器。种子、身份 provision 则明确要求当前完整清单校验码及源数量，旧标记会被拒绝；仅追加一张表或手改 marker 都不是本票安排 | `scripts/db/run-one-role-baseline.ts:694`、`:714`；`seed-neon-test-release1.ts:356`；`seed-local-release1.ts:205`；`seed-local-identity.ts:123`；`provision-database-test-identity.ts:510` |
| 应用启动是否因旧基线校验码拒绝 | 在当前 `app/`、`lib/`、`modules/` 未发现读取基线安装 marker 并比对当前校验码的启动检查；readiness 检查连接、身份、业务种子和文档传输。不能把脚本拒绝等同于应用必然启动拒绝。未连接旧线上库实测；它是否满足运行需要仍未能确认 | 对上述目录搜索基线校验调用／marker；`lib/runtime/local-synthetic-readiness.ts`、应用启动与本地运行配置 |
| 075 加入时线上测试库如何处理 | **未能确认。**提交 `bc1b657` 记录数量由 73 加到 74、更新清单并生成；实施记录只证明本地临时 PostgreSQL 安装，且写“未部署”。既有远程测试库手册为早期记录，不能证明加入 075 时重建、升级或保留旧库的实际处理 | `git show bc1b657`；`docs/implementation/ACCESS-TRIAL-01_PROGRESS.md:674`、`:678`；`docs/runbooks/ENV-01_NEON_TEST_DATABASE.md` |
| 构建是否执行本票结构 | 未改构建或部署配置。现有生产 prebuild 仍读取 TEST_DATABASE_URL（测试库连接配置），执行指定的 052 与 038、053–057；不遍历基线清单。静态核对未发现它执行 076 或重新安装完整基线的路径，未运行 build 或远程验证 | `package.json:13`、`vercel.json:3`；两个 `apply-production-uat-*` 脚本的显式文件列表 |
| 集成分支／main 是否自动部署 | **未能确认平台当前联动设置**，不因换目标就断言集成分支无自动副作用。仓库配置不能证明实际 Git 部署分支；本次未访问平台 | 仓库部署配置，仅静态核对 |

结论：更新基线会影响要求精确 marker 的运维脚本；未发现同类应用启动校验；075 的线上处理无可确认记录。旧库既不会因为 Git 更新自动升级，也不能据此保证仍可运行。将来 main 放行必须另案确认，本票不改启动检查、构建脚本、部署配置或非本地数据库。

## 六、测试与实际页面验证

以下命令使用 Node.js 22.23.2、pnpm 10.34.4，全部测试目标为本地或纯静态；没有使用 Neon 配置。数字按测试运行器总计：新增结构 14 包括父测试 1、子测试 13。

| 命令／检查 | 结果 | 范围 |
| --- | --- | --- |
| `node --conditions=react-server --test tests/integration/school-directory-structure-postgresql.test.ts` | 14 通过，0 失败，0 跳过 | 原结构约束、审计、恢复、并发删除、旧引用不变、测试专用撤回；新验证同一应用所有者安装并收回权限 |
| 下方既有学校测试命令 | 66 通过，0 失败，0 跳过 | 本次重跑；上一票 main 对比也是 66／0／0，本次未另起 main 服务器 |
| `pnpm test:migration` | 11 通过，0 失败，0 跳过 | 结构漂移 |
| `pnpm test:architecture` | 25 通过，0 失败，0 跳过 | 模块边界 |
| `pnpm db:baseline:check` | 通过，75 源、76 生成文件 | 严格完整清单与生成结果核对 |
| `node --conditions=react-server --test tests/migration/one-role-baseline.test.ts` | 28 通过，0 失败，0 跳过 | 既有基线测试，未改断言 |
| `pnpm test:one-role-baseline-postgresql` | 2 通过，0 失败，0 跳过 | 仓库既有可丢弃 PostgreSQL 17 gate；dry-run／安装／独立后检及既有权限业务断言 |
| `pnpm typecheck` | 通过，退出 0 | 不等同运行验收 |
| `pnpm exec eslint modules/schools/domain/directory-structure.ts modules/schools/infrastructure/directory-structure-store.ts scripts/db/generate-one-role-baseline.ts tests/integration/school-directory-structure-postgresql.test.ts` | 通过，退出 0 | 本票 TypeScript 针对性检查；本续票未重跑全量 lint，上一票的既有 42 错误／50 警告不冒称本次新证据 |
| `verifyOrderedMigrationManifest()` 与 `git diff --check` | 通过，清单 75 项；差异空白检查通过 | 只读校验 |

```bash
node --conditions=react-server --test tests/unit/schools/*.test.ts tests/integration/school-target-workflow.test.ts tests/integration/school-change-workflow.test.ts tests/integration/school-governance-workflow.test.ts tests/integration/postgresql-resolved-school-transaction.test.ts tests/integration/postgresql-school-target-repository.test.ts tests/contract/school-target-route.test.ts
```

初次新结构测试有 1 子测试失败（含父测试共计 2 失败）：旧断言期待独立管理员建表时的表级 ACL 拒绝；改回基线安装后应用身份为所有者，正确保护是强制 RLS 隐藏全部记录。已改测应用查询为零、管理员查询非零、强制 RLS 为真、无策略，不放宽实际数据保护；最终 14 项通过。原有业务测试未修改。

实际浏览器使用本地 `http://localhost:3000`，Founder 正常表单登录，不用模拟客户端替代：

| 必查位置 | 实际结果 | 可观察证据 |
| --- | --- | --- |
| Founder 登录 | 通过 | 登录 POST 303，转到 `/today`，当前身份 API 200 |
| 学校列表 `/schools` | 通过 | 正式学校 API 200；显示 3／3 所，三条 `env01-synthetic-school-001` 至 `003`、对应英文合成校名 |
| 学校详情 | 通过 | 从列表点击第一所进入详情；显示英文名称、地区、官网、未知字段与原有履历区；resolved 与变更记录 API 均 200 |
| 选校 `/selector` | **失败** | 页面已实际打开，显示“學校資料暫時無法載入，請稍後重試。”；旧 `/api/crawler/schools` 500 |

选校失败证据链：页面调用旧 crawler 客户端；旧服务读取本地 `data/crawler-source/latest/publish_manifest.json`；该已跟踪清单没有 `schema_version`／`state`，解析器要求 `crawler-handoff/v1`／`candidate`，在 `snapshot.ts:140` 抛 `SCHEMA_MISMATCH`。相关页面、客户端、旧服务、解析器、快照均与 main 相同，本票新增四表不在调用链。未在 main 另开服务器复测，不把源码相同说成 main 运行验收；不修旧入口、不改抓取产物或环境配置。旧读取链路的处理需负责人决定，当前不能声称三处全通过。

未执行：独立 QA、全量测试、全量 lint、本地回退旧基线演练、build、任何非本地验证、部署。浏览器没有执行建档／修改／清除／导入业务操作；只核对现有三处。

## 七、相对 main 的最终文件清单与交接

以下为整个目录结构分支相对应用 main `76a88eb` 的最终清单，**不是只列续票差异**：

| 类别 | 文件（Tianxingguoji 仓库根路径） | 原因 |
| --- | --- | --- |
| 新增 | `db/migrations/202611150010_076_expand_school_directory_structure.sql` | 目录四表、约束、审计及安装期权限窗口 |
| 新增 | `db/baselines/one-role/generated/075_202611150010_076_expand_school_directory_structure.sql` | 从新增迁移自动生成的基线 SQL |
| 新增 | `modules/schools/domain/directory-structure.ts` | 学校／校址字段定义 |
| 新增 | `modules/schools/infrastructure/directory-structure-store.ts` | 内部字段访问与恢复 |
| 新增 | `tests/integration/school-directory-structure-postgresql.test.ts` | 结构约束、安装窗口与撤回的合成测试 |
| 修改 | `db/migrations/manifest.json` | 追加 076 校验和，历史项不变 |
| 修改 | `db/baselines/one-role/manifest.json` | 75 源完整清单及生成文件校验和 |
| 修改 | `scripts/db/generate-one-role-baseline.ts` | 仅源数量及完整清单校验码两项，严格逻辑与 main 一致 |
| 修改 | `docs/runbooks/local-synthetic.md` | 仅增加本次源数、文件数、表数及空目录说明 |

共新增 5、修改 4，无相对 main 的文件删除。续票删除的专用脚本本来就不在 main 中，最终清单没有该脚本。txgj-doc 相对其 main 只新增本文；相对原报告提交则为更新同一文件。既有需求与其他报告不变，school-tracker 不变。未跟踪应用 AGENTS.md 不改、不暂存、不提交。

临时数据库容器测试结束即删除，所有本票及上一票遗留的专用临时文件／归档副本在摘要落入本文后清理。结束时容器仅保留正常本地 PostgreSQL；`docker volume ls` 仅 `tianxing-local_tianxing_postgres_data`。开发服务器继续提供重建后的本地环境，未部署。

```text
status: blocked
owner: backend
scope: 学校目录结构、既有基线安装方式、本地手册和测试；文档只更新本报告
changed: 权限安装窗口沿用旧迁移；恢复严格完整清单；重新生成基线；删除专用安装脚本；本地删卷重装完成
evidence: 75 源/76 文件/88 表；四新表空；REFERENCES 前后 false；结构14、学校66、漂移11、架构25、基线28、完整PG2均通过；类型与针对性lint通过；列表/详情通过，选校旧入口500
not_run: 独立QA、main服务器对比、本地回退旧基线、build、全量测试、全量lint、非本地验证与部署
risks_or_stop_conditions: 旧选校快照格式不被旧入口接受；075线上测试库处理未能确认；平台自动部署未能确认；等待负责人核对，不合并、不推进下一票
```

未读取或使用三份 Neon 迁移配置，未触碰任何非本地数据库或环境；未执行三校定向清除票、教育局导入或 585 校承接，未访问真实网站、运行爬虫或调用 LLM。按本票授权删卷重建确实删除了旧本地合成库，随后正常种子重建三校，不能将这件事表述为“完全没有删除本地数据”。未改构建、部署、环境或密钥文件，未建 PR、未合并、未部署。
