# 案件列表与案件详情 UI 标准实施记录（2026-11-16）

案件列表、案件详情及其直接嵌入面板已采用共用 UI。两个本地账号均能搜索、筛选、阅读案件、编辑并保存评估、建立合成任务。**本票为部分完成，不能报整票本地验收通过**：案件工作区现有路由返回 404，没有可供换样式的运行中分页签；暂停／恢复实际生效，但提交后显示服务不可用；来源保存与候选名单操作因现有数据／流程条件不能完成。没有绕过权限、修改接口或补造业务数据来掩盖这些限制。

```text
status: partial
owner: frontend
scope: 本地合成环境；案件列表、详情、直接嵌入面板、共用 UI、对应测试与文档
changed: 已完成外观、阅读／编辑呈现、样式稿归档与 10 张截图；未改权限、路由、认证、数据库或接口业务含义
evidence: 142 项自动测试通过；类型检查和指定文件 lint 通过；Founder 与顾问实际浏览器操作见下表
not_run: 候选名单建立／审核／家长确认、来源保存、完整分页签跳转；非本地环境、部署、构建、全量测试
risks_or_stop_conditions: 工作区导航缺少可用投影，阶段操作回执后的异常，以及家长入口拒绝／不可用，均需另票确认边界后处理
```

## 1. 起点、依据与样式稿归档

| 项目 | 本票事实 | 证据 |
| --- | --- | --- |
| 应用起点 | `origin/rebuild/schools` 的 `5ef5a40`；新分支 `codex/ui-standard-case-pages` | 应用 Git 历史 |
| 文档起点 | `origin/main` 的 `4bd6145` 已包含首批 UI 报告和截图；没有自行合并任何分支 | 文档 Git 历史、既有 `ui-standard-first-pages` 报告 |
| 开工第一步 | 将工作目录 `Tianxingguoji/ui-style-c/` 中 7 个文件逐字节复制到本仓库 `design/ui-style-c/`，首先提交为 `08e5a7e`；目录比较无差异 | `design/ui-style-c/README.md`、6 份 HTML；`diff -rq` |
| 实施依据 | 先读样式稿 README，再读案件列表／详情原稿；共用标准为应用仓库现行标准 | `design/ui-style-c/03-case-list.html`、`04-case-detail.html`；`docs/implementation/UI_STANDARD.md`、`components/ui/standard.tsx`、`standard.css` |
| 需求边界 | 案件阶段、评估、候选名单、任务和访问权限沿用原有行为；没有登记新的业务决定 | `business-requirements/30-cases.zh-CN.md` 与对应应用契约、服务端入口未改 |
| 回退 | 撤回本票应用提交即可恢复原外观；没有数据库变更脚本。浏览器产生的合成评估和任务是正常本地业务数据，不随外观回退删除 | 本票改动清单；数据库目录无差异 |

样式稿只是外观参考，HTML 中的演示人名、学校、电话、日期和数量没有移入应用。原稿引用的外部字体网址没有被访问。应用标准文档已改为指向上述归档位置。

## 2. 实际交付与保留的边界

| 范围 | 结果 | 技术证据 |
| --- | --- | --- |
| 案件列表 | 页头唯一主按钮为既有“建立案件”；白卡内搜索、原有状态与阶段筛选、表格、底部条数；学生名加粗，阶段标签化 | `app/(erp)/cases/page.tsx` |
| 列、排序、分页 | 仅使用既有返回字段，保持返回顺序和原筛选逻辑。接口没有分页参数／分页元数据，不添加上一页、下一页或虚假总页数；底部显示筛选条数／已加载条数 | `components/crm/f2-contract.ts` 的 `listCases`；列表源码 |
| 去掉假顾问 | 旧页面给所有行固定填 `primaryRole: 'advisor'`，然后显示 Advisor；本票删除这个伪造列。详情仍显示服务端返回的真实合成顾问名称 | 列表与详情差异 |
| 详情页头 | 返回案件、学生与案件标题、真实阶段标签；原有查看学生／家长入口／面试链接的条件和目标不变。没有合适的既有页头主要动作，因此不虚设按钮 | `app/(erp)/cases/[caseId]/page.tsx` |
| 主副栏 | 主栏为流程、任务、评估和候选名单；副栏为案件资料、评估设置、推荐来源；手机单栏 | 共用 `ui-columns`、`ui-stack` |
| 阶段 | 使用原来五个阶段与当前案件阶段顺序，分段显示已完成／进行中／未开始，不计算百分比；原负责人数据和回退逻辑保留 | `CaseStageTimeline.tsx` |
| 评估 | 默认阅读已保存且非未知的答案；未知项用剩余数量说明。卡标题旁编辑入口，编辑态保留全部字段、语义选择、校验、冲突处置及权限判断；一次点“儲存全部修改” | `AssessmentEditor.tsx` |
| 评估保存边界 | **原来就是按字段顺序逐次提交**，本票仍然如此，不伪装成原子整卡事务。全部成功才返回阅读；失败时保留编辑和既有冲突处理。此次每个账号实际保存一个字段，未做多字段部分失败的浏览器复现 | `saveAll`／`patchField` 原接口调用保留；单元测试覆盖原契约 |
| 推荐来源 | 阅读为默认；卡标题旁编辑入口；保存仍为一次来源关联命令，历史关联默认收起，弱化底色。空来源、只读与拒绝不变 | `CaseReferralSourcePanel.tsx` |
| 候选学校 | 建立新版本入口、版本资料、学校选择、审核及家长确认换共用控件；审核／家长记录弱化。版本提交、审核、家长确认仍是各自独立命令，不合并保存 | `CandidateListWorkspace.tsx` |
| 案件任务 | 原有展开／收起、创建表单和任务列表换共用组件；“新增任務”是标题侧弱入口。任务仍一次创建；没有新增详情内修改既有任务功能 | `CaseTasksPanel.tsx`；`task-ui.tsx` 仅新增可选标准呈现参数，其他调用默认保留原样 |
| 工作区分页签 | 仅把已有 `CaseWorkspace` 分页签呈现改成胶囊；键盘行为、名称、权限投影过滤、链接全部保留。**未完成在详情展示可运行的全套子页分页签**，原因见第 5 节 | `CaseWorkspace.tsx`、`workspace-model.ts`、既有 workspace 路由 |
| 共用补充 | 原生输入／下拉／多行输入包装保留字段名、引用、约束与禁用；补充选择块、胶囊路由签、弱化记录、细分隔线 | `components/ui/standard.tsx`、`standard.css` |

没有引入依赖。触及文件中的压缩控制流已展开，未全仓库格式化。`TaskListItem`（任务列表项）与 `TaskPageState`（任务页状态）的新呈现只在案件面板显式启用；任务独立页面仍使用默认旧分支。

### 文字变动

新增并中英文登记 `caseUi`（案件界面文案）四个键：`edit`（編輯資料／Edit details）、`read`（返回閱讀／Back to reading）、`missing`（尚有 N 項未填寫／N fields not filled）、`viewStudent`（查看學生／View student）。列表重复的学生名链接改成“查看學生”，链接目标不变；加载期间不先显示假“0 宗案件”，改用原说明中的“使用篩選查看需要的資料。”。其余旧文案含义不改；原来使用的 Founder、旧英文标签没有借本票修改权限语义。

## 3. 本地验证

环境为 Node 22、pnpm 10.34.4、现有本地应用 `http://localhost:3000` 和本地 PostgreSQL 17。没有重装数据库，没有读取 Neon 环境文件。自动测试中使用的一次性本地数据库容器由测试清理。

### 自动检查（最终结果）

| 命令 | 通过 | 失败 | 跳过 | 说明 |
| --- | ---: | ---: | ---: | --- |
| `node --conditions=react-server --test tests/unit/cases/*.test.ts tests/unit/case-workspace-model.test.ts tests/unit/tasks/ui-contract.test.ts tests/unit/tasks/p3-fe-03.test.ts tests/contract/case-intake-route.test.ts tests/contract/p2-be-04-candidate-list-routes.test.ts` | 109 | 0 | 0 | 案件、工作区、任务 UI 契约、建案与候选名单接口契约 |
| `node --test tests/unit/ui/standard.test.ts` | 6 | 0 | 0 | 共用组件、对比度；新增原生表单约束保留检查 |
| `pnpm test:architecture` | 25 | 0 | 0 | 架构边界 |
| `pnpm test:one-role-baseline-postgresql` | 2 | 0 | 0 | 真实一次性本地 PostgreSQL 测试，无远程数据库 |
| `pnpm typecheck` | — | 0 | — | 最终退出码 0 |
| 指定 14 个 TS／TSX 文件的 `pnpm exec eslint` | — | 0 | — | 最终退出码 0，无警告 |
| `git diff --check` | — | 0 | — | 无空白差异问题 |

合计 **142 项测试通过、0 失败、0 跳过**；类型检查与 lint 不伪记成测试条数。

lint 明确文件为：两个案件 page、AssessmentEditor、CandidateListWorkspace、CaseReferralSourcePanel、CaseStageControls、CaseStageTimeline、CaseWorkspace、CaseTasksPanel、task-ui、standard.tsx，以及三个修改的测试文件（路径见第 7 节）。未执行全量测试、`pnpm build` 或全仓库 lint。

开发中初次案件检查有两处源文本断言失败，已经按下列实际呈现调整；一度新增 Empty（空状态）组件使用了不支持的 children 参数导致类型失败，已改为组件现有 detail／action 参数，最终类型通过。没有放宽业务断言：

| 测试文件 | 调整及理由 |
| --- | --- |
| `tests/unit/cases/case-create-ui.test.ts` | 原生 select 的 `aria-label` 源码断言改成共用 Select 的 label 断言，组件仍产生关联 label；旧“主要顾问”列断言改为禁止伪造 primaryRole／Advisor，并确认首列学生名加粗 |
| `tests/unit/cases/case-workflow-ui.test.ts` | 同步提交锁的正则允许换行空白，仍检查同一拒绝条件与 return；没有删除防重复提交检查 |
| `tests/unit/ui/standard.test.ts` | 新增验证原生日期类型、字段名、required、disabled、标签、textarea 长度与 select 选中值被保留 |

### 双账号浏览器实操

使用既有本地合成账号：Founder 为 `founder@env01.test.invalid`；顾问为 `l2-international@local.invalid`。没有新增账号、调整角色或密码。验证案件为 `LOCAL-TRIAL-INTERNATIONAL`（本地国际学校合成案件），学生为 `ENV01 Synthetic Student One`（本地合成学生一）。

| 操作 | Founder 实际观察 | 顾问实际观察 |
| --- | --- | --- |
| 登录与列表 | 正常登录，列表有两宗合成案件；搜索 Student One + 进行中 + 背景资料得到一宗 | 正常登录，列表仅一宗可见案件；相同搜索与筛选得到一宗 |
| 进入详情／阅读 | 所有直接面板可读；五段阶段与资料来自本地数据 | 可读自己负责的案件；没有放宽列表可见范围 |
| 评估编辑保存 | 将目前年级保存为“合成測試年級”，回到阅读，已储存 1/15，未填 14 | 改为“合成顧問驗證年級”，保存后阅读实际显示新值 |
| 任务创建 | 建立“合成 UI 驗證任務”，分配现有可选 L3，页面显示成功并重新载入 | 建立“合成顧問 UI 驗證任務”，相同现有可选负责人，页面显示成功并重新载入 |
| 来源编辑 | 能进入编辑；显示“目前沒有可用的推薦來源。”，保存禁用 | 相同；没有为了演示建立来源数据 |
| 候选名单 | 空态；背景必填仅 1/10，建立新版本受原条件限制 | 相同；未伪造背景完成或候选学校 |
| 阶段操作 | 实际提交暂停和恢复；提交后服务不可用提示，刷新先见已暂停版本 3，再见进行中版本 4 | 同样实际暂停／恢复；刷新后进行中版本 6。状态已恢复，不能把提交反馈称为通过 |
| 分页签跳转 | 原工作区是 404，无运行中签可点击；逐个直达检查八个子页面 | 同样；未改开关、授权或工作区服务 |

本地保留两条明确标为合成验证的任务，以及一个合成评估字段的修改；没有删除原数据。页面最终保持 Founder 登录，案件已恢复进行中。

### 八个子页面逐页打开

下列为实际直达结果，**不是分页签跳转通过**。手机宽度下打开各页，标题、内容与操作未被顶部导航遮挡；没有重做子页文件。

| 子页 | Founder | 顾问 | 边界 |
| --- | --- | --- | --- |
| assessment（背景评估） | 可开，评估内容已载入 | 可开 | 复用同一评估组件，因共享组件而得到新外观；子页壳未改 |
| applications（逐校申请） | 无可显示申请目标 | 同左 | 既有空状态，无目标不造目标 |
| interviews（面试支援） | 无可登记面试的已提交申请／只读说明 | 同左 | 可点返回案件及任务；未登记面试 |
| documents（案件文件） | 可开，登记表单与“本案目前沒有文件” | 同左 | 未新增文件或上传 |
| close（人工结案） | 可开，原 Founder 结案按钮 | 也显示原 Founder 结案按钮 | 未点击结案，不能据按钮存在声称顾问可结案；原权限行为未改 |
| access（家长入口） | 加载后显示仅 Founder 或主要顾问可管理的拒绝说明 | 加载后显示服务暂时不可用 | 页面可见不等于入口业务通过；没有签发访问密钥 |
| workspace（案件工作区） | 404 | 404 | 原开关未开启；即使开启，源码当前投影也是空 tabs 和错误面板，不能在此票中补服务 |
| new（建立案件） | 可开，原业务分类及建案表单 | 可开 | 未创建新案件；内容与校验保留 |

## 4. 截图

共 **10 张**，五种状态各桌面 1440、手机 390。浏览器截图工具返回原生 JPEG 且没有 PNG 编码参数，因此按本票允许保留 JPEG，**没有把 JPEG 转成 PNG 后宣称原生 PNG**。部分长页图宽 1434／384 是滚动条占用后的内容宽；浏览器视口仍为 1440／390。页面测得文档宽均不超过对应视口，表格只在卡内滚动。实测阶段主色 `rgb(15, 118, 110)`，即 `#0F766E`。

| 状态 | 桌面 | 手机 |
| --- | --- | --- |
| 案件列表 | [截图](ui-standard-case-pages.2026-11-16/01-case-list-desktop.jpg) | [截图](ui-standard-case-pages.2026-11-16/01-case-list-mobile.jpg) |
| 详情阅读态 | [截图](ui-standard-case-pages.2026-11-16/02-detail-read-desktop.jpg) | [截图](ui-standard-case-pages.2026-11-16/02-detail-read-mobile.jpg) |
| 评估编辑态 | [截图](ui-standard-case-pages.2026-11-16/03-detail-edit-desktop.jpg) | [截图](ui-standard-case-pages.2026-11-16/03-detail-edit-mobile.jpg) |
| 列表搜索空状态 | [截图](ui-standard-case-pages.2026-11-16/04-empty-desktop.jpg) | [截图](ui-standard-case-pages.2026-11-16/04-empty-mobile.jpg) |
| 未重做的面试子页 | [截图](ui-standard-case-pages.2026-11-16/05-existing-interviews-desktop.jpg) | [截图](ui-standard-case-pages.2026-11-16/05-existing-interviews-mobile.jpg) |

## 5. 停下的部分与未能确认项

1. **详情全套分页签**：现有详情没有统一分区导航；唯一现成分页签属于独立工作区，取决于服务端可见签投影。不能把它复制成全员可见、静态拼接到八条子路由并声称权限不变。本票只换已有签的样式，未新增导航契约、授权或工作区投影。补齐运行中导航需另行明确现有各子页的导航契约及可见性来源。
2. **阶段提交反馈异常**：两个账号均观察到暂停／恢复服务不可用提示，但刷新后实际状态已改变。本票未改该命令、解码、权威读取或拒绝处理；根因未能确认，不能凭前端错误提示判断写入失败，也未为验收修接口。需要另票调查；此票只能确认状态最终恢复为进行中。
3. **来源与候选完整保存未执行**：没有有效推荐来源，背景收集未完成，没有候选版本。保留禁用与原条件，不把未执行记成通过。候选审核、家长确认、来源更新仍需有合适合成前置数据后单独验收。
4. **家长入口与工作区**：两账号实际结果如上。没有修改权限定义、路径判断、环境开关、服务或认证。顾问结案按钮存在是旧页面现象，未执行结案或推断后台授权。
5. **独立 QA 未执行**：本报告为实现者本地自测，非独立 QA、正式上线或远程环境验收。

## 6. 样式稿有、当前接口／页面没有的元素（7 项）

| 样式稿元素 | 本票处理与原因 |
| --- | --- |
| 列表“目标学校”列 | 现有列表返回没有该投影，不展示假学校 |
| 列表“下一步”列 | 现有列表没有下一步任务投影，不拼接示意任务 |
| 列表“负责顾问”姓名列 | 返回无姓名；删除旧硬编码 Advisor，不补接口 |
| 上一页／下一页与总页数 | 列表没有分页契约，仅保留真实条数 |
| 可运行的全套案件分页签 | 现有详情没有；工作区不可用，停止点见上 |
| 详情合并的目标学校申请／面试状态、学生现校与监护人电话卡 | 当前详情未提供同一投影，不抄样例；已有独立申请／面试／学生页面保留 |
| 统一案件事件时间轴、示例“已递交 3 所” | 没有在此详情提供统一事件数据／计数；仅呈现现有阶段及来源／候选记录，不造事件或百分比 |

## 7. 文件清单与交接

应用实施提交为 `4954780`；两仓库分支均为 `codex/ui-standard-case-pages`。文档原稿独立提交为 `08e5a7e`，本报告与截图另行提交。

应用仓库相对 `5ef5a40` **没有新增文件，修改 18 个**：

- `app/(erp)/cases/page.tsx`
- `app/(erp)/cases/[caseId]/page.tsx`
- `components/cases/AssessmentEditor.tsx`
- `components/cases/CandidateListWorkspace.tsx`
- `components/cases/CaseReferralSourcePanel.tsx`
- `components/cases/CaseStageControls.tsx`
- `components/cases/CaseStageTimeline.tsx`
- `components/cases/CaseWorkspace.tsx`
- `components/tasks/CaseTasksPanel.tsx`
- `components/tasks/task-ui.tsx`
- `components/ui/standard.tsx`
- `components/ui/standard.css`
- `docs/implementation/UI_STANDARD.md`
- `i18n/zh-TW.json`
- `i18n/en.json`
- `tests/unit/cases/case-create-ui.test.ts`
- `tests/unit/cases/case-workflow-ui.test.ts`
- `tests/unit/ui/standard.test.ts`

文档仓库新增 **18 个文件**：样式稿目录 7 个文件、本报告 1 个、同名目录截图 10 张。没有修改既有报告或业务需求。应用仓库原来的未跟踪 `AGENTS.md`、`ui-style-c/` 保留且未纳入应用提交。

自己查看：打开 `http://localhost:3000/cases`，用上述本地账号登录；搜索合成学生，进入案件，点“學生評估”标题旁“編輯資料”，改目前年级并“儲存全部修改”；任务卡点“新增任務”可展开表单；来源卡能进入编辑但当前无来源可选。暂停／恢复有上述反馈异常，不作为已通过的操作推荐。

没有合并、建 PR 或部署。未触碰非本地数据库、真实网站、真实资料、爬虫、LLM、构建与部署配置、依赖版本。一次性容器已清理，结束检查仅有 `tianxing-local-postgres-1` 和卷 `tianxing-local_tianxing_postgres_data`。临时转换脚本与测试日志清理后，本地应用保持运行。
