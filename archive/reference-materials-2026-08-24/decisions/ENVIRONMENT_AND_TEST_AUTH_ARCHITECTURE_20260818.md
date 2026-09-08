# 三环境与 Vercel 合成测试身份架构

| 项目 | 决策 |
| --- | --- |
| 状态 | `accepted_for_source_implementation` |
| 日期 | 2026-08-18 |
| 决策编号 | `DEC-070` |
| 实现票据 | `ENV-01`、`P2-14` |
| 当前授权 | 允许架构、开发计划、本地源码、测试、PR 与既有 PR Preview 验收；不授权新建或修改 Vercel/PostgreSQL/AWS 云资源、写外部数据库、导入真实数据或生产发布 |

## 1. 环境矩阵

`NODE_ENV` 只表示 Node.js/Next.js 的执行优化语义，不能代替业务环境。业务环境必须由
`APP_ENV` 显式选择，身份和数据适配器再分别由 `APP_RUNTIME_MODE` 与 `AUTH_MODE` 选择。

| 环境 | 承载位置 | `NODE_ENV` | `APP_ENV` | `APP_RUNTIME_MODE` | `AUTH_MODE` | 数据 |
| --- | --- | --- | --- | --- | --- | --- |
| 开发 | 本机 | `development` | `development` | `local-synthetic` | `local-synthetic` | 本机确定性合成数据 |
| 测试 | Vercel | `production` | `test` | `test-database` | `database-test` | 独立测试 PostgreSQL，仅合成数据 |
| 生产 | AWS 香港 | `production` | `production` | `production-aws` | `cognito` | AWS 香港生产数据 |

`NODE_ENV=test` 只允许测试 runner 使用。不得在 Vercel Runtime 中设置 `NODE_ENV=test`。
Vercel 的 `VERCEL_ENV=preview|production` 表示发布通道，不表示本系统的业务生产环境；只要承载
测试系统，`APP_ENV` 都必须保持 `test`。

任何未列出的组合、缺失值或互相矛盾的组合都必须 fail closed。浏览器、URL、cookie、Git branch
或 Vercel system variable 都不能改变 `APP_ENV`、`APP_RUNTIME_MODE` 或 `AUTH_MODE`。

## 2. 测试数据库边界

- Vercel 测试数据库必须使用独立 PostgreSQL instance/project 和独立凭据，不能复用 AWS 生产
  RDS、生产快照、生产 Secret、生产网络路径或生产备份。
- 测试库只允许确定性合成 Organization、User、Student、Case 和学校数据；禁止真实姓名、邮箱、
  电话、文件、案件内容或从生产脱敏后难以证明不可重识别的数据。
- Web runtime 使用最小权限的 test identity role 与 application role；migration owner 凭据不能注入
  Vercel Web runtime。
- migration、角色 provision、synthetic seed 和测试账号 provision 是独立人工命令。应用启动、构建、
  request handler 和 health check 都不得自动迁移、seed 或创建账号。
- 测试数据库 URL 只能由 Vercel Secret 注入。源码、Git、PR、日志、错误响应和浏览器 bundle 不得
  包含连接串或密码。

## 3. Database Test 登录

`database-test` 是非生产身份适配器，不是 PostgreSQL 账号直登：

1. 登录页只接收测试账号 email 和 password，不显示任意 role selector。
2. email 必须指向预先 provision 的 synthetic identity；role、organization、membership 和 capability
   继续从 PostgreSQL 权威数据读取，不能由表单或 URL 提交。
3. 密码使用每账号随机 salt 的 versioned `scrypt` verifier；数据库不保存明文密码，Vercel 也不保存
   可作为默认账号密码的源码常量。
4. 失败返回统一 `authentication_failed`，不区分账号不存在、密码错误、账号停用或锁定；失败次数和
   临时锁定状态在数据库原子更新，并有固定安全上限。
5. 成功后建立 `session_kind=database_test` 的 opaque、HttpOnly、Secure、SameSite cookie session。
   每次敏感请求继续重新校验 User、Organization、Membership、RoleBinding、session version 和 expiry。
6. Vercel Deployment Protection 是外层测试访问门禁，但不能替代应用身份、数据库授权或 session revoke。

## 4. 运行时与部署约束

- `database-test` 只允许 `APP_ENV=test`、`NODE_ENV=production`、显式 `test-database` runtime，且必须
  检测到 Vercel runtime。AWS 生产组合出现 `database-test` 或 test database URL 时必须停止启动。
- `local-synthetic` 继续只接受 loopback 依赖，并在 `NODE_ENV=production` 时拒绝启动。
- `cognito` 生产组合必须是 `APP_ENV=production`、`APP_RUNTIME_MODE=production-aws`，不得回退到
  database-test、local role、mock、preview 或旧 Neon adapter。
- 所有环境继续使用同一追加式 migration 历史和相同 API contract；环境差异只能存在于 composition
  root、受控配置和外部资源，不建立测试专用业务分支。
- Vercel test 不满足香港生产驻留证据，因此即使功能验收通过，也不能成为生产、灾备、回滚目标或
  真实数据处理平面。

## 5. 前后端交付边界

后端会话先交付环境组合校验、`database-test` port/service/repository、追加 migration、独立 provision
命令、session 与 lockout 测试，以及数据库应用运行时。前端会话只在后端冻结表单和错误合同后交付
email/password 登录 UI、通用错误状态、loading/disabled/accessibility/mobile 测试；前端不得保存密码、
推导 role 或显示数据库连接信息。

两个会话使用独立 Git worktree 和分支。后端合同先合并；前端从包含该合同的最新 `main` 开始。
共享文件、API DTO、环境变量名或错误码发生变化时，必须先回到架构师裁决，不由两个会话自行兼容。

## 6. 验收与停止条件

- 表格中的三个合法环境组合通过，所有交叉组合和缺失配置都 fail closed。
- Vercel 构建保持 `NODE_ENV=production`，并接受其系统 build identity；AWS 显式 build identity 仍优先。
- 正确 synthetic 凭据可建立数据库 session；错误、不存在、停用、锁定账号均为 constant-shape denial。
- 登录后 API v1 读取和写入使用测试 application role，migration owner 不能出现在 Web runtime。
- 直接提交 role、重复登录、session revoke、过期、并发失败计数和数据库断连均有聚焦负向测试。
- Vercel 部署前必须另行批准准确的环境变量名、数据库供应商/区域、连接串 secret、Deployment
  Protection、migration/seed/provision 命令和清理策略。没有这些 exact payload 时，源码完成不等于环境可用。

回滚按适配器和部署版本完成：关闭 `database-test`、撤回 Vercel 测试部署并轮换测试 secret。追加
migration 不回删；若凭据结构需要纠正，使用新 migration 和重新 provision 的 synthetic accounts。
