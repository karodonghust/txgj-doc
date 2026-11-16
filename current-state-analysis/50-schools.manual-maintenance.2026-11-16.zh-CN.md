# 学校手工建档与基本资料维护：本地实施记录

status: pass（本票本地自测通过，Local Dev accepted）。新目录可在本地由四个试用等级访问，已完成手工维护的后端动作及页面；不代表正式切换、独立 QA 签署、集成合并或上线。此前共享文案文件的范围停止点已由项目负责人本轮澄清解除，本报告直接替换旧结论。

依据：BR-BASELINE-20261115-v68（学校字段归属与校址维护细节收口）、BR-051（学校目录与数据变更）、BR-070（审计与一致性）、BR-015（K12 试用等级与授权），及未被取代的历史条文。目录约定的九动作、十六场景按本票范围核对；目录结构实施记录与分期方案二作为工程参考。

应用基点 `92803c8`，本票实现提交 `da84da8`；文档基点 `e593d7f`，两仓库均为 `codex/schools-rebuild-manual-maintenance`。验收后的合并目标是应用 `rebuild/schools`，不是 main；本次不合并、不建 PR、不部署。

## 一、八项结果与证据

| 结果 | 结论与实际观察 | 证据位置（应用仓库相对路径） |
| --- | --- | --- |
| 1 名单外手工建档 | 已达到。中文或英文至少一项非空；无教育局编号可建，默认一个人工校址，未填写资料为未知。浏览器分别由 Founder 与 L3 创建合成学校；两名皆空的写入被测试拒绝 | `modules/schools/domain/manual-directory.ts`；`modules/schools/infrastructure/manual-directory-repository.ts`；新动作 PostgreSQL 测试 |
| 2 查找、打开与 23 项资料 | 已达到。名称搜索；12 项整校资料（含两个身份字段）、11 项校址资料。相同值共同显示，不同值按校址编号并排；官网比较仅忽略已确认三种差异 | `app/(erp)/school-directory/page.tsx`；`modules/schools/domain/manual-directory.ts`；浏览器电话拆分结果 |
| 3 免审批即时修改与等级同等 | 已达到。四等级真实 PostgreSQL 链路均完成建档、修改、拆分、恢复、标记与删除；浏览器 Founder、L3 修改后立即可见，L1、L2 可打开同一目录。不扩大旧学校能力 | 新动作测试；独立能力与权限清单见下节 |
| 4 添加与删除校址 | 已达到。只软删人工校址；名单校址、最后一个不可删。浏览器看到最后一个的禁用按钮及原因；名单校址禁删由结构测试验证，本地页面未造名单导入样本 | 076（学校目录结构）既有约束原文未改；077（学校目录维护访问）新权限迁移；仓储删除资格检查；页面原因文案 |
| 5 共用改为分别维护 | 已达到。Founder 与 L3 均把共用电话拆开，修改其中一个后另一个保持原值；恢复后可再次分别维护 | 仓储 `split`／`field` 动作；页面两列校址输入；结构与新动作测试 |
| 6 履历与逐字段恢复 | 已达到。显示操作人、时间、前后值和版本；恢复追加版本，并在来源记录中引用被选履历；其他字段不变。删除校址后原字段履历仍可读，不允许恢复进已删除校址 | 仓储来源中的 `restoredHistoryId`（被恢复履历编号）；原四表审计触发器；两组 PostgreSQL 测试；浏览器恢复电话与已删除校址履历 |
| 7 停办信息 | 已达到。标记／取消均即时保存和留历史；不从目录隐藏、不限制查找。Founder、L3 浏览器均实际切换 | `closed` 动作及页面；无更新计划或招生代码 |
| 8 并发不覆盖 | 已达到。同一学校的所有动作先锁定并检查学校版本，校址独立修改也推进学校版本。两名操作者并发仅一人成功；浏览器两编辑页后提交得到明确冲突提示，草稿保留，重新载入显示先提交值 | 新动作 PostgreSQL 并发测试；API 返回 409；浏览器分别检查 Founder、L3 冲突提示 |

内部编号是稳定身份，只读；手工学校的教育局编号显示未知，未开放编号回填或身份改绑。这是既有约定禁止人工改写内部身份及本票身份对应排除项，不把“修改任一项资料”扩成身份治理。其余 21 个业务资料字段支持未知值与人工维护，名称仍受最低条件约束。

## 二、契约、权限与旧入口保护

新地址 `/school-directory`（学校目录页面），仅地址直达；不增加共享侧栏项。新 API `/api/v1/school-directory`（学校目录读写接口）：读取列表或单校；提交建档、逐字段保存、拆分、添加／删除校址、停办、逐字段恢复。服务端提供动作定义，页面按同一契约调用。身份与组织来自服务端会话，客户端不能指定组织或操作人；每次事务重新检查有效试用成员、组织、成员关系与用户，并锁住授权事实。

| 权限／共享文件 | 只新增什么及为什么绕不开 | 旧行为未变的证据 |
| --- | --- | --- |
| `modules/access/domain/contract.ts` | 新增 `school_directory.manage`（新学校目录维护能力）；需要被现有身份响应与客户端合法解码 | 既有角色能力数组未改；只有总能力枚举新增一项 |
| `modules/access/domain/authorization.ts` | 在有效试用等级返回值中加新能力；Founder 使用总能力集合，其他三个等级在原列表后追加 | 单元测试逐项比较 L1／L2／L3 去掉新项后与原角色列表完全相同；停用成员仍空列表；旧 advisor／admin／contractor／审核角色无新能力 |
| `components/layout/workspace-navigation.ts` | 只匹配新地址及其子路径到新能力；否则新入口无法与旧学校能力区分 | 旧导航数组不改；单元测试旧路径匹配、相近非目录路径、旧 schools 子路径及 selector 行为不变；侧栏无新项 |
| `i18n/zh-TW.json`、`i18n/en.json` | 只新增 `schoolDirectory`（学校目录文案）命名空间 | 解析 JSON 与基点逐键比较：移除新增命名空间后完全相等，无旧键改删 |
| 077（学校目录维护访问）迁移 | 在四张新目录表新增有效四等级且组织相符的行级策略；学校／校址／共用模式读写，履历读和追加；履历序列使用及校验函数执行 | 不给任何旧表新增授权；不授予物理删除；旧触发器继续拒绝历史更新／删除。匿名新接口 401，缺少组织／错误组织／无试用资格／停用账号被拒绝 |
| `app/` 两个新路由文件 | 新页面与新正式 API，复用身份检查入口，不复用旧 `schools.read`（旧学校读取能力）授权新目录 | 旧列表、详情、选校路由零修改；旧列表与详情浏览器通过；架构边界 25 项通过 |

四表权限仍由强制行级安全保护，不创建第二个组织；错误组织测试使用不存在的组织标识验证拒绝，未建立多机构业务样本。新动作使用既有事务幂等记录：同一个调用键重试不重复建档或追加修改；页面在失败后保留相同请求的调用键。履历写入与业务修改同事务，失败一并回滚。

原 076（学校目录结构）迁移未改。077 不创建新表、没有跨表外键，因此不需要打开关联权限窗口；本地重装后组织表、用户表的 REFERENCES（建外键关联权限）仍为 false。旧学校的五样机制在旧页面中仍存在，仅限集成分支本地检查点；正式切换及退出另票，不把本票称为正式交付。

## 三、九动作与十六场景对照

| 目录约定动作 | 本票覆盖情况 |
| --- | --- |
| 1 教育局名单进入目录 | 不覆盖，禁止首次抓取、名单导入及依赖名单的提示 |
| 2 名单外手工建档 | 覆盖；合成学校可建、可查，不附加审批或未验证状态 |
| 3 补齐／修改基本资料与未知 | 覆盖；身份字段只读边界见第一节 |
| 4 三类来源网址维护 | 部分覆盖：官网、学校资料页；招生资讯页属后半段，不新增其结构 |
| 5 查看逐字段履历 | 覆盖；仅学校目录，不开放平台全量审计 |
| 6 恢复某字段旧值 | 覆盖；追加新版本、保留历史和已有引用 |
| 7 四等级同等 | 覆盖；不扩展案件、账号管理或旧学校能力 |
| 8 选入名单及固定引用 | 新目录选校不覆盖；旧固定引用不变由结构回归测试保护 |
| 9 已停办信息 | 目录标记／取消覆盖；招生、选校接入、自动更新联动不覆盖 |

| 约定场景 | 本票覆盖／验证边界 |
| --- | --- |
| 1 名称最低要求 | 覆盖；解析及数据库拒绝两名为空，浏览器仅中文建档；只英文为结构测试样本 |
| 2 无外部编号可用 | 目录建档／读取覆盖；显示未知；不回落旧目录 |
| 3 教育局同编号粒度 | 既有结构测试覆盖同编号唯一；不执行导入 |
| 4 不同编号不合并 | 不新增合并行为；本票未另做两所同名同官网不同编号的完整场景，不能声称该样本已验收 |
| 5 补空白 | 覆盖；电话由未知改合成值，四等级动作测试与浏览器 |
| 6 修改已有值 | 覆盖；电话／学费修改即生效，前值保留 |
| 7 三类网址分别维护 | 只覆盖官网和资料页的独立字段；招生网址及任务影响不覆盖。页面未逐一提交所有网址字段，字段允许范围由契约和结构约束验证 |
| 8 人工改未知 | 动作接受 null，结构测试覆盖未知；浏览器未单独执行电话清空，不能写浏览器已验收此样本 |
| 9 恢复旧值 | 覆盖电话旧值恢复、其他学校字段不变及所选履历引用；未把当前样本冒称为完整甲→乙→丙且另改地址的指定序列 |
| 10 保留来源事实 | 原结构测试保留来源事实和原始值；本票全为人工合成资料，不引入教育局原件 |
| 11 等级同等 | 四等级真实数据库全部维护动作；浏览器 Founder、L3 完整主要流程，L1、L2 打开与读取 |
| 12 学校可选 | 不覆盖；明确不接入新目录选校 |
| 13 已有引用不改 | 结构测试用实际合成固定引用核对摘要不变，非空断言；旧页面零改动 |
| 14 停办是信息 | 标记、取消、目录显示覆盖；新目录选校及更新联动不覆盖 |
| 15 并发不覆盖 | 覆盖；两操作者数据库并发及浏览器两编辑页冲突 |
| 16 审计与重复请求 | 覆盖；重复建档幂等、审计失败回滚；无远程验收含义 |

## 四、实际验证记录

以下均在本机执行，未连接非本地数据库。使用 Node 22.23.2 与仓库 pnpm；命令前设置本机 Node 22 的 PATH。

| 步骤／命令 | 结果 |
| --- | --- |
| 停止本仓库旧 Next.js 进程；`pnpm local:down --volumes`；`pnpm local:up` | 成功；只重建指定本地合成库和会话。不是清除三所学校业务票；后续标准 seed 重建原三所样本 |
| `pnpm db:plan:local` | pass，76 源、77 生成文件 |
| `pnpm db:baseline:local:dry-run` | pass，空跑回滚，目标 clean |
| `ONE_ROLE_BASELINE_APPLY_CONFIRM=tianxing-one-role-v1 pnpm db:baseline:local` | pass，76 源安装；全部归属／RLS／安全定义函数检查通过 |
| `pnpm db:seed:local-release1` | pass，88 张表、原三所合成学校；没有向新目录搬运资料 |
| `pnpm db:seed:local-trial-demo` | pass，5 principals、2 cases、2 tasks |
| 九账号 `pnpm db:provision:local-identity --email=...`，本地密码从 stdin 输入 | 九个均 created，密码统一为用户指定的 123456 |
| `node --conditions=react-server --test tests/integration/school-directory-structure-postgresql.test.ts` | 14 通过、0 失败；包括撤回／重装、已有引用保护及名单／最后校址保护 |
| `node --conditions=react-server --test tests/unit/schools/manual-directory.test.ts tests/integration/school-directory-maintenance-postgresql.test.ts` | 11 通过、0 失败（3 个单元测试、7 个数据库子测试及父测试） |
| 既有学校测试命令，见下方 | 66 通过、0 失败；执行时尚未加入本票新单元文件，不把新增用例混算为原 66 |
| `pnpm test:migration` | 11 通过、0 失败 |
| `pnpm test:architecture` | 25 通过、0 失败 |
| `pnpm db:baseline:check` | verified，76 源／77 生成文件 |
| `node --conditions=react-server --test tests/migration/one-role-baseline.test.ts` | 28 通过、0 失败 |
| `pnpm test:one-role-baseline-postgresql` | 2 通过、0 失败 |
| `pnpm typecheck` | 通过，退出码 0 |
| `pnpm exec eslint` 本票全部 TS／TSX 文件、基线生成器及本票修改的结构测试 | 通过，0 错误、0 警告；未运行全库 lint 或 build |
| `git diff --check`；旧 i18n 键与旧路由、076、package、compose 对照 | 通过；旧文案键与旧页面不改，构建／部署配置不改 |
| 匿名 `GET /api/v1/school-directory`；本地 readiness | 401；readiness 四项 ready |
| 本地应用账号只读 SQL 核验 | 组织／用户 REFERENCES 均 false；88 张表、旧学校记录 3、新目录合成学校 2。容器 postgres 登录被既有安全规则拒绝后，改用配置校验为 loopback 的应用连接读取；未改账号或安全设置 |

既有学校测试原命令：

```sh
node --conditions=react-server --test tests/unit/schools/*.test.ts tests/integration/school-target-workflow.test.ts tests/integration/school-change-workflow.test.ts tests/integration/school-governance-workflow.test.ts tests/integration/postgresql-resolved-school-transaction.test.ts tests/integration/postgresql-school-target-repository.test.ts tests/contract/school-target-route.test.ts
```

首次检查曾发现并已修正：Node 原生类型擦除不接受参数属性，改为普通成员赋值；新测试合成身份需建立完整员工资料与等级绑定；架构测试要求双引号形式的 server-only 标记；页面 effect 写法触发 lint；新测试数组的类型推断过窄。结构测试原“零策略”断言因本票明确开放四张表失效，改为精确断言四个新策略，同时保留无身份零可见与强制 RLS 断言；未放宽拒绝条件来换通过。最终没有未处理的本票测试失败。

浏览器实际观察：Founder、L3 分别创建“本地合成目錄驗收一／本地合成目錄驗收L3”，修改学费或学段，添加校址、电话共同值改分开、恢复电话、停办标记及取消、删除一个人工校址并看到最后校址禁止原因；已删除校址履历仍显示。两编辑页的冲突提示保留后提交草稿，重新载入显示先提交值。L1、L2 本地账号能打开同一新目录与资料。旧学校列表实际显示 3/3，旧详情显示合成学校资料。旧选校页既有故障不修、本轮不重新验收。

未执行：独立 QA、真实名单校址页面样本、全 23 项逐项浏览器提交、单独手机尺寸与完整键盘流程、远程环境、部署、构建、真实网站、名单导入、招生与更新。本票数据库约束／动作测试不能替代这些未执行项。

## 五、页面文字与自己点一遍

新增文字均在专用命名空间，不新增业务状态枚举。名称最低要求、未知、校址保护、停办、履历来自已确认规则；冲突提示直接实现本票第八项要求。加载、保存、重试与登录失效是操作反馈，不作为新业务确认。

| 文案键 | 繁体原文 |
| --- | --- |
| `title` | 學校目錄 |
| `search` | 搜尋學校名稱 |
| `create` | 新增學校 |
| `nameZh` | 中文名稱 |
| `nameEn` | 英文名稱 |
| `minimum` | 中文名稱或英文名稱至少填寫一項。 |
| `unknown` | 未知 |
| `yes` | 是 |
| `no` | 否 |
| `save` | 儲存 |
| `saved` | 已儲存 |
| `loading` | 載入中 |
| `empty` | 尚無學校 |
| `error` | 暫時無法完成，請重試。 |
| `denied` | 登入狀態或存取權限已變更，請重新登入。 |
| `invalid` | 請檢查資料；中文或英文名稱至少保留一項。 |
| `conflict` | 此學校已由其他操作更新，尚未覆蓋你的輸入。請重新載入最新資料後再修改。 |
| `reload` | 重新載入 |
| `back` | 返回目錄 |
| `addCampus` | 添加校址 |
| `deleteCampus` | 刪除校址 |
| `edbCampus` | 教育局名單中的校址不可刪除。 |
| `lastCampus` | 最後一個校址不可刪除。 |
| `campus` | 校址 {{number}} |
| `shared` | 共同顯示 |
| `split` | 按校址分開填寫 |
| `history` | 修改履歷 |
| `restore` | 恢復此舊值 |
| `actor` | 操作人 |
| `time` | 時間 |
| `before` | 修改前 |
| `after` | 修改後 |
| `version` | 版本 |
| `closed` | 已停辦 |
| `markClosed` | 標記已停辦 |
| `unmarkClosed` | 取消已停辦標記 |
| `origin` | 資料來源 |
| `manual` | 顧問建檔 |
| `edb` | 教育局名單 |
| `id` | 系統內部編號 |
| `edbNumber` | 教育局學校編號 |
| `deletedCampus` | 已刪除校址 |
| `fields.school_name_zh` | 中文名稱 |
| `fields.school_name_en` | 英文名稱 |
| `fields.school_level` | 學段 |
| `fields.student_gender` | 學生性別類別 |
| `fields.is_sen` | 特殊教育標記 |
| `fields.tuition_info` | 學費說明 |
| `fields.tuition_year` | 學費適用學年 |
| `fields.tuition_source_url` | 學費來源連結 |
| `fields.has_boarding` | 是否提供住宿 |
| `fields.boarding_info` | 住宿說明 |
| `fields.source_school_key` | 爬蟲來源編號 |
| `fields.registration_number` | 註冊編號 |
| `fields.location_marker` | 校址標識 |
| `fields.official_website` | 官網 |
| `fields.profile_url` | 學校資料頁面地址 |
| `fields.district` | 地區 |
| `fields.address` | 地址 |
| `fields.phone` | 電話 |
| `fields.fax` | 傳真 |
| `fields.school_session` | 授課時間類別 |
| `fields.finance_type` | 資助類別 |
| `fields.is_closed` | 已停辦 |

自己复查（只用本机）：

1. 打开 `http://localhost:3000/login`，使用 `founder@env01.test.invalid` 或 `l3@local.invalid`，密码 `123456`。其他试用账号为 `l1@local.invalid`、`l2-local@local.invalid`、`l2-international@local.invalid`，密码相同。账号只用于本地合成环境。
2. 地址栏打开 `http://localhost:3000/school-directory`；侧栏仍是旧入口，不要把侧栏“學校資料”当作新页面。
3. 在“新增學校”只填一个名称并提交；打开后教育局编号为“未知”，一个校址，其余未填资料未知。
4. 修改“學費說明”并点该字段“儲存”。点“添加校址”；在“電話 · 共同顯示”填合成电话并保存，再点电话区的“按校址分開填寫”。只改其中一列，另一列应不变。
5. 在“修改履歷”找到电话变化，点对应“恢復此舊值”；只恢复该项。点“標記已停辦”及“取消已停辦標記”；学校仍可搜索打开。
6. 删除一个人工校址；剩下一个时按钮禁用且写明原因。履历中的“已刪除校址”资料仍在。
7. 用两个编辑页先后打开同一学校；第一个保存后，第二个基于旧页面保存，应看到“此學校已由其他操作更新…”；输入保留，点“重新載入”后才看到最新值。自动测试另以两个不同有效操作者证明同一冲突边界。
8. 回旧 `/schools` 核对三所合成学校及详情。新目录样本不进入旧选校；不要把新旧并存当作正式切换。

## 六、停止分支、范围与回退

没有新增业务默认值。03（学校身份对应）、D1（人工来源/纯外部键关联）的剩余身份对应、D3（身份疑点、来源变化）的更新任务影响、多官网更新、教育局名单再现提示的关闭／消失，以及招生候选均不实施。官网和资料页允许保存人工值，不触发更新或改绑。招生来源网址不在本票 23 字段内，留在后半段。未回答问题只挡对应分支，不再以共享 i18n 或新路由范围阻塞已授权动作。

安装沿用完整基线，源数量由 75 加为 76；生成器只改计数与严格匹配的清单摘要，不放宽校验、无专用安装器。旧基线库会被当前基线检查拒绝沿用；本地按手册重装。既有运行手册中的 75/76 数字尚未在本票改动（不扩大至其他既有文档），本报告记录本次实际 76/77，步骤完全相同。

回退以停止新页面使用、回退本票代码并按旧集成基线重建本地合成库为单位；这会丢弃本票本地新建样本，不能用于保留业务数据的远程回退。076 结构撤回／重装在可丢弃 PostgreSQL 的 14 项测试中实际执行；本票没有执行持久本地库回退，因为结束时须保留可点的新页面。无破坏性远程操作。

合入 `rebuild/schools` 不由本票执行，也不等于部署放行。构建与部署配置未改；未来集成分支合入 main 的自动部署及旧基线库处理必须另行核对、另行放行，不以本次本地通过批准。

相对应用集成基点的最终文件清单：

新增 9 份：

- `app/(erp)/school-directory/page.tsx`
- `app/api/v1/school-directory/route.ts`
- `db/baselines/one-role/generated/076_202611160010_077_open_school_directory_maintenance.sql`
- `db/migrations/202611160010_077_open_school_directory_maintenance.sql`
- `modules/schools/domain/manual-directory.ts`
- `modules/schools/infrastructure/manual-directory-client.ts`
- `modules/schools/infrastructure/manual-directory-repository.ts`
- `tests/integration/school-directory-maintenance-postgresql.test.ts`
- `tests/unit/schools/manual-directory.test.ts`

修改 12 份：

- `components/layout/workspace-navigation.ts`
- `db/baselines/one-role/manifest.json`
- `db/migrations/manifest.json`
- `i18n/en.json`
- `i18n/zh-TW.json`
- `modules/access/domain/authorization.ts`
- `modules/access/domain/contract.ts`
- `modules/schools/client.ts`
- `modules/schools/public.ts`
- `modules/schools/server.ts`
- `scripts/db/generate-one-role-baseline.ts`
- `tests/integration/school-directory-structure-postgresql.test.ts`

文档仓库只更新本报告。未修改业务需求、其他分析报告、school-tracker、旧学校三处页面、076、构建脚本、部署配置或环境文件；AGENTS.md 未暂存。未连接非本地环境、未读取 Neon 配置、未使用真实学校资料、未执行三校清除业务票、未导入名单或旧学校、未访问真实网站、未调用 LLM。临时 PostgreSQL 测试容器使用 tmpfs 并在 finally 清理；结束只保留本地 PostgreSQL 数据卷与运行中的本地应用。

```text
status: pass
owner: backend
scope: 学校目录手工维护后端、页面及必要的独立权限增量；仅本地合成环境
changed: 追加 077 并重新生成基线；新增独立页面与动作；学校事务、测试、专用文案及本报告
evidence: 本报告列出的实际本地命令、四等级数据库链路与浏览器观察
not_run: 独立 QA、远程、部署、全量构建、名单导入、招生与更新、手机专项验收
risks_or_stop_conditions: 待项目负责人核对；其他业务分支另行放行；不得合入 main 或继续下一票
```
