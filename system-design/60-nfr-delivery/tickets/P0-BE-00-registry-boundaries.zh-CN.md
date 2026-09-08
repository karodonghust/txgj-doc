# P0-BE-00 Registry、入口与架构门禁实现

状态：`merged`  
Owner：`backend`  
架构批准依据：项目负责人于 2026-08-25 确认通过 `P0-ARCH-01` 至 `P0-ARCH-03`  
业务依据：`BR-003`、`BR-012`、`BR-070`、`BR-071`

返回[开发票据索引](README.md)。

## 业务结果

产品源码只把批准的 Release 1 模块和入口视为 active；旧排除项不能通过 registry、navigation、正式 API 或 runtime 重新进入产品。

## 依赖

- `P0-ARCH-01`、`P0-ARCH-02`、`P0-ARCH-03` 全部 `approved`。
- 产品仓库旧工程指引与 confirmed `BR-003` 的冲突已完成同步。
- 当前 dirty worktree 范围已由 Backend 复核，不能覆盖用户改动。

## 范围

- `modules/shared/architecture/module-registry.ts` 及直接关联的架构 metadata。
- `tests/architecture/**` 和必要的 API envelope/root-entry contract tests。
- 隔离旧 Platform Billing/Future/DEC/legacy 正式入口所需的最小 server/navigation 变更。
- 更新代码版本绑定的 `docs/**`，不修改 canonical `BR-*`。

## 不在范围

- 不修改历史 migration，不执行数据库。
- 不实现 Shared 幂等 schema、Case/Task/Document/Portal 新业务。
- 不删除历史数据或 Git 历史。
- 不接真实云；`cloud-synthetic` 只能用于后续本地联调。

## 自测命令

```text
pnpm check:toolchain
pnpm test:architecture
node --test tests/contract/api-envelope.test.ts
pnpm typecheck
```

若仓库已存在更聚焦的 root-entry/registry test，优先补充并记录实际命令；不得用完整测试的旧输出代替本次证据。

## Expected Evidence

- changed files 精确清单和共享 diff 摘要。
- 四个命令各自 `passed/failed/not_run`。
- active registry snapshot 与排除项断言。
- Platform Billing/Future/legacy route 的历史保留和 Release 1 隔离证据。
- cloud/Preview/browser/database 均明确 `not_run`。

## 停止条件

- 需要重写 migration、删除历史数据或扩大到业务状态机。
- 测试要求与已批准 canonical 业务规则冲突。
- 发现重叠用户改动无法安全合并。
- 需要运行数据库、Docker、浏览器或外部云而尚未取得对应 gate。

## 回滚

- 仅回退本票据新增的 registry/入口/测试变更。
- 不使用 destructive Git 命令，不恢复或覆盖用户原有改动。
- 因本票据不执行 migration，无数据库回滚。

## 下一门禁

Backend 自测后交 Architect 做 readiness review；本次 readiness review 已通过，下一门禁是由现有独立测试工作流执行批准的 Local 只读验收。未经最终审查和用户批准，不 commit/push/PR。

## Architect readiness review（2026-08-26）

结论：`accepted_local_pending_git`。

### 范围核对

- Backend worktree 共显示 41 个 tracked changed files；其中 8 个是主工作区原先已有、且内容完全一致的用户文档改动，认定为 `inherited protected changes`，不计入本票据交付。
- 本票据实际拥有 33 个 tracked source/test/package 文件，另有 5 个新增文件，共 38 个 P0 文件。
- 未发现 migration、database、infra、Docker、`.github` 或真实云资源改动。
- 未执行 commit、push、PR，也未覆盖主工作区用户改动。

### Backend 自测证据

| 检查 | 结果 |
| --- | --- |
| `pnpm check:toolchain` | `passed` |
| `pnpm test:architecture` | `passed`（25/25） |
| `node --test tests/contract/api-envelope.test.ts` | `passed`（15/15） |
| `pnpm typecheck` | `passed` |
| 额外聚焦测试 | `passed`（13/13） |
| `git diff --check` | `passed` |
| browser / Preview / cloud / database / migration / Docker / full suite | `not_run` |

备注：独立 worktree 初始没有 `node_modules` 和 Next 生成的 `.next/types`；typecheck 验证时临时复用主检出同一 HEAD 的现有忽略目录，未安装依赖，验证后链接已移除。

### P0-owned changed files

```text
app/(erp)/cases/reconstructions/[reconstructionId]/page.tsx
app/(erp)/cases/reconstructions/new/page.tsx
app/(erp)/platform/billing/page.tsx
app/(erp)/students/duplicates/[candidateId]/page.tsx
app/(erp)/students/duplicates/page.tsx
app/api/v1/cases/reconstructions/route.ts
app/api/v1/crm/duplicate-candidates/[candidateId]/merges/route.ts
app/api/v1/crm/duplicate-candidates/[candidateId]/route.ts
app/api/v1/crm/duplicate-candidates/route.ts
app/api/v1/crm/duplicate-handler.ts
app/api/v1/crm/duplicate-merges/[mergeId]/corrections/route.ts
app/api/v1/crm/duplicate-records/search/route.ts
app/api/v1/platform/billing/overview/route.ts
components/crm/StudentsDirectory.tsx
components/layout/Sidebar.tsx
modules/access/domain/authorization.ts
modules/access/domain/contract.ts
modules/cases/public.ts
modules/cases/server.ts
modules/crm/infrastructure/runtime.ts
modules/crm/infrastructure/legacy-duplicate-runtime.ts
modules/crm/legacy-server.ts
modules/crm/server.ts
modules/external-portal/domain/contract.ts
modules/future/domain/feature-contracts.ts
modules/shared/architecture/module-registry.ts
modules/shared/presentation/release-one-entry-boundary.ts
modules/shared/public.ts
package.json
proxy.ts
tests/architecture/future-scope.test.ts
tests/architecture/module-boundaries.test.ts
tests/architecture/release-one-entry-boundaries.test.ts
tests/contract/api-envelope.test.ts
tests/integration/crm-student-create-dev-http.test.ts
tests/integration/portal-billing-module-ownership.test.ts
tests/unit/access/authorization-contract.test.ts
tests/unit/portal/contract-policy.test.ts
```

### 审查意见

- active registry、导航、root entry、正式 API 与 runtime 已形成同一套 Release 1 隔离边界。
- Platform Billing、Future、DEC、duplicate/merge、reconstruction 等旧实现保留在历史位置，但不能通过 active registry 或正式入口重新启用。
- 404/API envelope 及架构断言已补充，满足本票据的静态边界要求。
- Local QA 仍需验证实际本地运行时的入口、响应与回归；本次 readiness review 不把静态自测当作浏览器、数据库、云或生产证据。

### Local QA 预检（2026-08-26）

- `colima list`：`tianxing` profile 为 `Broken`；`colima status tianxing` 仍报告未运行。
- `limactl list`：没有可运行的实例；Docker API：`not_available`，无法访问项目要求的 PostgreSQL 17/LocalStack/ClamAV 本地依赖。
- `node_modules` 与 `.next/types` 虽存在，但不能替代完整 Local Dev 运行环境。
- 因此浏览器、Next Dev HTTP、数据库和容器验收均为 `not_run (unverified)`；不启动容器、不切换到 Vercel/Neon、不使用历史输出代替当前证据。
- 环境恢复后，沿用本票据冻结的范围执行一次独立 Local QA；失败即停止并回到 Architect 做归属判断。

### Local QA 首次运行结果（2026-08-26）

运行环境已恢复：Colima `tianxing`、PostgreSQL 17、LocalStack、ClamAV 和 Next Dev 均可启动。静态门禁复跑通过：架构 `25/25`、API envelope `15/15`、typecheck、`git diff --check`。

但真实 Next Dev HTTP 未通过，已停止并交回 Backend 修正：

- 排除页面 `/platform/billing`、`/cases/reconstructions/new`、`/cases/reconstructions/:id`、`/students/duplicates`、`/students/duplicates/:id` 的响应 body 含 404 内容，但 HTTP status 实际为 `200`；普通不存在路径实际为 `404`。
- 排除 API `/api/v1/crm/duplicate-records/search` 的 `GET` 返回 `405`，没有统一返回 `404 NOT_FOUND` envelope。
- 其余已探测排除 API 返回 `404`、`NOT_FOUND`、`no-store` 和 `X-Request-Id`。
- 以上两项属于 P0 入口隔离缺陷，未进入最终 QA；未修改主工作区和数据库。

### Local QA 修复后复验（2026-08-26）

结论：`passed`。

- Backend 新增最小 `proxy.ts` route boundary，在页面开始 streaming 前精确拦截 5 个冻结入口；排除页面真实 HTTP status 已由 `200` 修正为 `404`。
- `/api/v1/crm/duplicate-records/search` 的 `GET` 已统一为共享 `404 NOT_FOUND` envelope。
- Architect 独立复跑：`pnpm check:toolchain`、架构测试 `25/25`、API envelope `15/15`、typecheck、`git diff --check` 均通过。
- Architect 独立启动 Next 16.2.7 Dev，并逐一验证 5 个排除页面和 5 个排除 API：全部真实返回 `404`、`no-store` 和独立 `X-Request-Id`；JSON API 均返回 `NOT_FOUND`。
- 8 个主工作区既有文档与 Backend worktree 内容逐文件一致，继续认定为 `inherited protected changes`，不计入 P0 交付。
- 未修改 migration/database/infra/Docker/云配置；未执行数据库写入、浏览器、Preview、AWS、commit、push 或 PR。

### Architect 最终审查

结论：`accepted_local_pending_git`。

P0 的 registry、正式入口、导航、root exports、API fail-closed 和真实 Local Next Dev 行为现已一致。下一步由项目负责人决定是否进入 Git/PR；在明确批准前，源码继续留在 Backend worktree。

### Git 状态（2026-08-26）

- 已精确合入主工作区并创建本地提交：`2c55bcd`（`feat(architecture): enforce release 1 boundaries`）。
- 提交包含 38 个 P0 文件；8 个既有 `docs/**` 用户改动未暂存、未提交。
- 分支：`codex/case-flow-delta-01`。
- 初次沙箱内 `gh auth status` 误报 token 失效；根因是 Codex 沙箱不能读取 macOS Keychain，也不能连接 GitHub API。沙箱外只读验证确认账号 `karodonghust`、Keychain token 和 GitHub API 均正常，无需重新登录。
- 为避免复用已 squash 合并的旧分支造成 125 文件的错误 PR diff，已从最新 `origin/main` 建立 `codex/p0-release-one-boundaries`，只拣选 P0 提交；PR diff 精确为 38 个文件。
- PR：[GitHub PR #32](https://github.com/Kelvin-xing/Tianxingguoji/pull/32)。
- GitGuardian、Vercel deployment check 和 Vercel Preview Comments 均通过；该 Vercel check 只证明部署检查完成，不作为 Preview 业务运行验收。
- PR 已按 squash 方式合并到 `main`；远端提交：`fe961a41347acdbf192c49535815ff32456dca33`。
