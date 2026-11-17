# 本地登录核验与案件四个子页面 UI 重做（2026-11-17）

**status: partial。九个本地账号当前均可登录；四页改版、双账号评估保存和文件登记已完成。** 未复现负责人此前的登录失败，不能宣布其根因已消除。申请页没有既有编辑功能，面试登记缺少合成前置资料，两项写入未执行。自动检查有三项原有失败和一项因未提供独立测试库而跳过，均保留报告。

```text
status: partial
owner: frontend
scope: 本地合成环境；案件 assessment、applications、interviews、documents 及直接嵌入面板
changed: 共用页头及五签、标准卡片与控件、阅读后操作、技术资料折叠；没有认证修复或数据库重装
evidence: 九账号登录 HTTP 验证；Founder 和顾问浏览器操作；233 项测试通过、3 项原有失败、1 项跳过；类型检查与定向 lint 通过；十二张截图
not_run: 申请编辑、面试登记、文件上传／扫描／下载／删除／恢复、非本地环境、全量测试、构建、部署
risks_or_stop_conditions: 原登录故障原因未能确认；不能以空页面验证代替有申请／面试资料时的全流程验收
```

应用分支 `codex/ui-standard-case-subpages` 从 `origin/rebuild/schools` 的 `ec55d55` 新建；文档同名分支从 `origin/main` 的 `9cac1d8` 新建。开工已确认文档 main 含上一票报告，未自行合并。应用实现提交 `fd9268c`。应用合并目标仍为集成分支，本票未合并、未建 PR、未部署。

依据：BR-BASELINE-20261115-v68（学校字段归属与校址维护细节收口）为当前需求库基线；本次案件范围按 BR-032（Assessment）、BR-034（SchoolTarget 状态）、BR-035（申请准备与提交 Task）、BR-036（面试辅助 Task）、BR-050（文件）、BR-015（K12 试用等级与授权）、BR-070（审计与一致性）及现有契约保留业务行为。外观依据应用 `docs/implementation/UI_STANDARD.md`、共用 `standard.tsx / standard.css` 和本库 `design/ui-style-c/README.md`；样式稿不提供业务数据。

## 一、本地登录：检查事实与未确认原因

**非技术结论：检查时数据库和网站都已开着，九个账号的指定密码也都能使用。因此本次不需要重装、清空资料或改密码；此前为什么登不进去，现有证据不足以确定。** 昨天记录的一次“登入服務暫時不可用”也没有在这九个有效账号的本次登录中复现。

| 检查 | 实际结果 | 证据位置／命令 |
| --- | --- | --- |
| Docker 与数据库 | `colima-tianxing` 上的 `tianxing` profile 正在运行；数据库容器持续健康，回环地址 5432；不是数据库没开 | `docker context show`、`colima list`、`docker ps` |
| 网站进程 | 开工监听 3000 的 PID 为 87982，2026-11-16 启动，工作目录 `/Users/karo/Documents/Tianxingguoji`；其父进程为该目录的 Next 开发服务器 | `lsof -a -p 87982 -d cwd,1,2`、`ps`、`lsof -nP -iTCP:3000 -sTCP:LISTEN` |
| 运行提交边界 | 工作树开工为 `ec55d55`。旧进程跨越代码更新，会热更新；不能只凭进程启动时间证明内存中每个模块属于同一提交 | `git log -1` 与上述进程信息，未伪称旧进程固定运行 `ec55d55` |
| 健康与就绪 | HTTP 200，`ready`；数据库、身份、应用、文件假传输四项就绪 | `GET /api/v1/local/readiness` |
| 实际登录 | 九账号全部 HTTP 303，产生会话 Cookie；跟随已有目标页全部 HTTP 200。Cookie 内容没有打印、写入报告或提交 | `POST /api/v1/auth/login`，本地表单电邮＋指定密码；Founder 另有浏览器实际登录 |
| 本次恢复动作 | 无需启动数据库、重装、seed 或 provision。UI 改版验证前正常停止旧网站进程，再在本票工作树执行 `pnpm dev`，清楚建立本次运行来源；数据库未停止，合成数据保留 | 新进程 PID 13117，Next 16.2.7 `Ready`；随后再次核验全部九账号 |
| 历史失败 | 旧开发日志中有早期编辑期间的页面编译错误，无法与负责人今天的失败建立时间及请求关联；本次不据此认定根因，也没有修认证源码 | `.next/dev/logs/next-development.log` 只读筛查；此前登录报告 |

九个账号仅用于本地，密码均为负责人指定的 `123456`：

| 账号 | 登录响应／目标页 |
| --- | --- |
| `founder@env01.test.invalid` | 303 → `/today`，200 |
| `admin@env01.test.invalid` | 303 → `/today`，200 |
| `advisor@env01.test.invalid` | 303 → `/today`，200 |
| `advisor-secondary@env01.test.invalid` | 303 → `/today`，200 |
| `contractor@env01.test.invalid` | 303 → `/tasks`，200 |
| `l1@local.invalid` | 303 → `/today`，200 |
| `l2-international@local.invalid` | 303 → `/today`，200 |
| `l2-local@local.invalid` | 303 → `/today`，200 |
| `l3@local.invalid` | 303 → `/today`，200 |

账号清单按既有合成 fixture 核实。初查曾误测一个不在清单内的 `data-reviewer@env01.test.invalid`，该请求返回服务不可用；它不是上述九账号，不能用这次无效账号请求解释负责人的故障，未为它创建账号或修改认证。

另一个能解释“又回到登录页”的已知行为：本地测试身份同一账号再次登录，会撤销该账号此前的会话。末次九账号 HTTP 复核之后，浏览器原顾问会话因此回到登录页；重新登录成功。依据 `db/migrations/202608180120_028_expand_database_test_identity.sql` 的单活跃会话约束与 `database_test_relogin`（测试身份重新登录撤销）处理。这是本次受控复核造成的情况，**不据此推定负责人此前也遇到同一原因**。收尾最后一次登录留给浏览器中的 Founder，之后不再用 HTTP 登录覆盖它。

## 二、以后自己怎么启动

以下流程针对**这台已经装好数据库和九账号的 Mac**，不包含重装、清库、重新灌数据或改密码。

1. 打开 macOS「终端」，进入应用目录：
   ```sh
   cd /Users/karo/Documents/Tianxingguoji
   export PATH="/opt/homebrew/opt/node@22/bin:$PATH"
   node -v
   pnpm -v
   ```
   作用：使用本机已安装的 Node 22；应显示 `v22...` 和 `10.34.4`。不执行升级命令。
2. 检查本机容器运行器：
   ```sh
   colima list
   docker context show
   ```
   正常应为 `tianxing Running`，Docker context 为 `colima-tianxing`。只有 profile 停止时才执行：
   ```sh
   colima start --profile tianxing
   ```
   如果 context 不是上述本地 context，先停下核对，不向未知 context 执行后续命令。
3. 保留已有数据并启动／检查数据库：
   ```sh
   pnpm local:up
   pnpm local:ps
   ```
   正常应看到 `tianxing-local-postgres-1` 为 healthy。已经运行时该步骤不会重装数据库。不要运行 `down --volumes`。
4. 检查网站是否已启动：
   ```sh
   lsof -nP -iTCP:3000 -sTCP:LISTEN
   ```
   如果已有本项目的 Node 进程，直接打开网站，不重复启动；如果占用者不明，先核对。没有监听进程才运行：
   ```sh
   pnpm dev
   ```
   正常显示 `http://localhost:3000` 和 `Ready`。保留这个终端窗口运行；关闭它可能停止网站。
5. 另开一个终端窗口检查：
   ```sh
   curl --fail http://127.0.0.1:3000/api/v1/health
   curl --fail http://127.0.0.1:3000/api/v1/local/readiness
   ```
   正常应成功返回，readiness 显示 `ready`，四项依赖都 ready。失败时保留错误现象交查，不自行清库。
6. 浏览器打开 `http://localhost:3000/login`，使用上述账号和本地密码。浏览器地址用 `localhost`，不要切换成另一台机器或远程地址。同一账号在其他窗口重新登录后，原窗口可能需要再次登录。

## 三、四页交付结果与边界

| 页面／共用部分 | 已完成 | 保留的行为与限制 | 证据 |
| --- | --- | --- | --- |
| 共用页头 | 从案件详情抽出同一呈现组件；学生、案件号、年度、入学类型、顾问名称、阶段一致；有名称不附邮箱 | 详情和评估原服务端读取不变；其余三页通过现有 `getCase`（案件权威读取）获取页头，失败不隐藏或阻断原内容面板，不把页头当授权门禁 | `CaseHeader.tsx`、`CaseSubpage.tsx`；五个页面 |
| 分页签 | 复用五签，传入当前分区；仅当前签高亮 | 路径、权限、workspace 开关都不改 | `CasePageTabs.tsx`；五种当前签渲染测试 |
| 背景评估 | 与详情内嵌完全同一编辑组件；已填资料、未填数量、标题右侧编辑、集中保存、页底技术资料；冲突保留草稿按钮改次按钮，使保存是唯一主按钮 | 一次点击保存依旧按既有逐字段请求提交；不是新加批量事务；版本与冲突逻辑保留 | `AssessmentEditor.tsx`；两账号保存成功 |
| 选校与申请 | 白卡、标准表格／标签、空状态及重试提示 | 仍只显示既有接口提供的学校、状态、任务及可用操作文字；原组件没有申请编辑表单，本票不新增。当前合成案件没有目标，非空表格未做真实浏览器验收 | `F3CaseSection.tsx` |
| 面试 | 标准卡片、输入、标签和提示；有原有登记资格时在卡标题右侧展开登记；现有面试记录、任务恢复入口保留 | 学校资格、凭证要求、确认勾选、时间处理、幂等重试及任务恢复请求均不变。当前没有已提交申请和有效邀请凭证，登记入口按原条件不出现 | `InterviewInvitationsPanel.tsx` |
| 文件 | 阅读卡展示名称、分类、更新时间和真实状态；卡标题右侧进入登记／文件操作；登记为本页唯一主按钮，上传和恢复等用次／弱按钮；技术编号和锁版本收进系统资料 | 原权限、逐份登记、上传、扫描、下载、删除与恢复逻辑保留；业务文件的历史版本序号仍用于选择具体文件，不冒充内部锁版本删掉 | `CaseDocumentsPanel.tsx`、`CaseDocumentCard.tsx`、传输与生命周期面板 |
| 次要记录与系统资料 | 案件版本、评估编号／版本、逐字段版本、文件编号／锁版本置于页底默认收起区；文件版本管理弱化且初始不加载 | 折叠仅改变呈现，不修改值、请求或权限；操作区隐藏时仍挂载，避免切换阅读态销毁原上传控制器 | `CaseSystemDataProvider`（系统资料容器，既有组件）、新文件卡 |

仅使用现有共用组件，无新增依赖、无新组件库、无页面私设样式。没有改全站文件页的旧 `document-ui.tsx`；仅案件文件卡采用新呈现，避免把本票扩展为全站文件改版。申请组件初次加载改为微任务调度，避免 effect 内同步置状态的 lint 错误；请求和结果处理不变，结案分支不改外观或动作。

新增中英文界面键 16 项，均在 `caseUi`（案件界面文案）内：返回案件、页头加载／不可用、登记面试、面试填写说明、登记文件、文件操作、文件分类、更新时间、文件编号、资料版本、案件版本、学校／状态／任务／可用操作列。旧文案含义不改；页头标签直接复用原详情说法。没有新增业务状态或样式稿示意数字。

## 四、实际浏览器操作

案件沿用现有 `LOCAL-TRIAL-INTERNATIONAL`（本地国际学校合成案件），没有为申请或面试造前置资料。顾问账号为 `l2-international@local.invalid`。

| 页面 | Founder | 顾问 |
| --- | --- | --- |
| 背景评估 | 打开、阅读，改既有“目前年級”为合成验证文本，点击“儲存全部修改”，看到“已儲存。”；仍为 1/15 已填、14 项未填 | 读到 Founder 已保存值，再改为另一合成验证文本，保存成功 |
| 选校与申请 | 从分页签打开，显示“目前沒有可顯示的申請目標”；无编辑入口，写入未执行 | 同左 |
| 面试 | 从分页签打开，显示无可登记的已提交申请／只读说明；登记未执行 | 同左 |
| 文件 | 从分页签打开空态，展开“登記文件”，登记“合成 Founder 子頁驗證文件”，看到成功反馈与“等待上載” | 读到上述文件，再登记“合成顧問子頁驗證文件”，成功显示两份资料 |

最终重新登录 Founder 后仍能读到两份文件；系统资料展开显示案件版本 10、两份文件各自的内部编号及资料版本 1。没有上传实际文件、没有把“等待上载”改成“可使用”。最后折叠系统资料并保留 Founder 的文件页可点。

## 五、自动验证

运行环境：本机已安装 Node 22.23.2、pnpm 10.34.4；无安装或升级。以下均实际执行，不以静态计划替代：

| 命令 | 结果 |
| --- | --- |
| `node --conditions=react-server --test tests/unit/cases/*.test.ts tests/unit/documents/*.test.ts tests/unit/tasks/p3-fe-03.test.ts tests/unit/crm/f2-workspace.test.ts` | 199 项：195 通过、3 失败、1 跳过 |
| `node --test tests/unit/ui/standard.test.ts tests/unit/ui/case-subpages.test.ts` | 13 通过、0 失败、0 跳过 |
| `pnpm test:architecture` | 25 通过、0 失败、0 跳过 |
| `pnpm typecheck` | 退出码 0 |
| `pnpm exec eslint`，下列 17 个 TS/TSX 文件 | 退出码 0；最后两处呈现修改再单独重跑通过 |
| `git diff --check` | 通过 |

合计 **233 通过、3 失败、1 跳过**。定向 lint 为：本票五个案件 page；CaseHeader、CaseSubpage、CasePageTabs、F3CaseSection、InterviewInvitationsPanel、AssessmentEditor；CaseDocumentCard、CaseDocumentsPanel、DocumentLifecycleControls、DocumentTransferControls；新增 UI 测试及修改的 transfer-ui-contract 测试。

三项失败逐一：

| 测试名 | 原因 | 原基线确认 |
| --- | --- | --- |
| `P4 Case Documents route remains independent from Portal/Contractor` | 旧断言要求案件文件页使用 `DocumentWorkspace`，现行基线实际为 `CaseDocumentsPanel` | `ec55d55` 原始副本同样失败 |
| `DOC-02 permanent browser gate owns real dependencies without committing a malicious signature` | 旧断言硬编码生成文件数 36，与现有测试源不符 | 原始副本同样失败 |
| `Document routes use authoritative components and Case detail mounts the panel` | 旧断言要求全站文件页挂 `DocumentsDirectory`，现行基线挂的是 `DocumentWorkspace`；另仍假定详情内嵌文件面板 | 原始副本在第一项断言处同样失败 |

基线对照实际用 `git archive ec55d55` 导出独立临时副本，仅运行上述三个测试文件，共 14 项，11 通过、3 失败。未改三个旧断言来凑通过。

跳过项：`applies the additive document schema and exposes tenant, version, scan, and active-pointer constraints`，原因是未提供 `TEST_DATABASE_URL`（独立 PostgreSQL 测试库地址）。本票未连接替代或非本地数据库，不声明结构测试通过。

因呈现变化只调整一条旧断言：传输提示由原地 `role={isStatus ? "status" : "alert"}` 改用共用 `Notice`（提示条），测试改为同时验证 `warning={!isStatus}` 和共用组件的 `status/alert` 对应，未删检查。新增六项渲染测试覆盖五种分页签高亮、真实页头信息／邮箱回退／不出现技术版本。

另外将面试提交／恢复、文件登记、文件传输与生命周期处理的 **18 个函数**与 `ec55d55` 用 TypeScript 标准打印后的语法树逐一比较，全部相同；它只是源码差异证据，不替代未执行的上传或面试流程。`modules/`、`app/api/`、`db/`、认证页面、包清单与锁文件均无差异。

## 六、截图

六种场景，各桌面 1440×1000 与手机 390×844，共十二张，保存实际运行画面。工具原生只提供 JPEG，未把 JPEG 改扩展名冒充 PNG。实际浏览器计算主色 `rgb(15, 118, 110)`，即 `#0F766E`。检查到页面宽不超过视口；桌面／手机可能因滚动条分别占 1434／384 像素。评估及文件截图摄于验收不同步骤，内容是当时已存数据，不是统一伪造快照。

| 场景 | 桌面 | 手机 |
| --- | --- | --- |
| 背景评估阅读 | [截图](ui-standard-case-subpages.2026-11-17/01-assessment-desktop.jpg) | [截图](ui-standard-case-subpages.2026-11-17/01-assessment-mobile.jpg) |
| 选校与申请 | [截图](ui-standard-case-subpages.2026-11-17/02-applications-desktop.jpg) | [截图](ui-standard-case-subpages.2026-11-17/02-applications-mobile.jpg) |
| 面试 | [截图](ui-standard-case-subpages.2026-11-17/03-interviews-desktop.jpg) | [截图](ui-standard-case-subpages.2026-11-17/03-interviews-mobile.jpg) |
| 文件阅读 | [截图](ui-standard-case-subpages.2026-11-17/04-documents-desktop.jpg) | [截图](ui-standard-case-subpages.2026-11-17/04-documents-mobile.jpg) |
| 评估编辑 | [截图](ui-standard-case-subpages.2026-11-17/05-assessment-edit-desktop.jpg) | [截图](ui-standard-case-subpages.2026-11-17/05-assessment-edit-mobile.jpg) |
| 文件初始空态 | [截图](ui-standard-case-subpages.2026-11-17/06-documents-empty-desktop.jpg) | [截图](ui-standard-case-subpages.2026-11-17/06-documents-empty-mobile.jpg) |

## 七、文件清单与交接

应用新增四个文件：

- `components/cases/CaseHeader.tsx`
- `components/cases/CaseSubpage.tsx`
- `components/documents/CaseDocumentCard.tsx`
- `tests/unit/ui/case-subpages.test.ts`

应用修改十五个文件：

- `app/(erp)/cases/[caseId]/page.tsx`：仅抽出共用页头，原访问与内容保持。
- `app/(erp)/cases/[caseId]/assessment/page.tsx`
- `app/(erp)/cases/[caseId]/applications/page.tsx`
- `app/(erp)/cases/[caseId]/interviews/page.tsx`
- `app/(erp)/cases/[caseId]/documents/page.tsx`
- `components/cases/AssessmentEditor.tsx`
- `components/cases/CasePageTabs.tsx`
- `components/cases/F3CaseSection.tsx`
- `components/cases/InterviewInvitationsPanel.tsx`
- `components/documents/CaseDocumentsPanel.tsx`
- `components/documents/DocumentLifecycleControls.tsx`
- `components/documents/DocumentTransferControls.tsx`
- `i18n/zh-TW.json`
- `i18n/en.json`
- `tests/unit/documents/transfer-ui-contract.test.ts`

文档新增本报告及同名目录十二张截图；旧报告、样式稿、需求文件不改。school-tracker 无改动。应用原有未跟踪 `AGENTS.md` 与 `ui-style-c/` 保留，不提交。

没有触及新的业务二选一问题；停在申请不存在既有写入、面试前置不足及三项旧测试不符处，不扩展业务修复。文件传输与历史操作仅改外观，未实跑上传／扫描／下载／删除／恢复；无新增授权、无业务接口／状态转换变更。九账号登录和四页当前可用，不代表负责人此前偶发登录故障已查明，也不代表所有申请与面试数据状态已验收。

本票临时基线副本和日志已清理；未创建一次性容器。结束仅有 `tianxing-local-postgres-1` 及 `tianxing-local_tianxing_postgres_data`，网站保持运行。未读取或使用 `.env.migration.neon-*`，未访问真实网站、未使用真实资料、未改依赖、未合并或部署。
