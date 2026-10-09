# 学校模块：当前本地数据库只读盘点（2026-10-10）

本文件只补充[重建前业务问题清单第 01 项（本次重建的资料范围）](50-schools.rebuild-business-questions.2026-10-10.zh-CN.md:90)所缺的**当前本地数据库事实**，不是迁移盘点结论、设计文档或业务决定。没有回答第 01 项应处理哪些资料，也没有回答任何其他决定。既有现状报告和问题清单均未修改。

业务阅读边界：[BR-051（学校目录与数据变更）](/Users/karo/Documents/txgj-doc/business-requirements/50-schools.zh-CN.md)，BR-BASELINE-20261010-v60（基线）；没有修改需求或产生新基线。代码快照：Tianxingguoji `76a88eb2f853d8b4b6b5ea5d62a6aed93db915fc`；文档起点：txgj-doc main `e2cad5a629b1d6a4a5319252b15376ffe1f08ea4`。

## 环境、范围与只读保障

| 项目 | 本轮核实的事实 |
| --- | --- |
| 环境 | **本地开发、合成数据环境**。本机 `.env.local` 中 APP_ENV（业务环境）为 `development`，APP_RUNTIME_MODE（运行模式）为 `local-synthetic`，NODE_ENV（构建环境）为 `development`。未读取或连接远端数据库。 |
| 实例 | 本机 Colima 的 `colima-tianxing` Docker context（容器上下文），端点为本地 Unix socket；Compose project（容器项目）为 `tianxing-local`，容器为 `tianxing-local-postgres-1`，镜像为 `postgres:17.10-alpine3.24`。实际数据库 `tianxing`，服务器版本 **17.10**。 |
| 连接 | 直接在已运行的上述容器内，用现有 `tianxing_app`（应用数据库账号）连接本地 Unix socket；不通过宿主机端口推定数据库身份，不启动、重建或切换容器。该账号不是 superuser（超级用户）。 |
| 组织范围 | 使用 `.env.local` 中已有的 LOCAL_SYNTHETIC_ORGANIZATION_ID（本地组织标识）设置事务内组织上下文；对应的 `access_organizations`（组织表）可见行数为 **1**。以下业务表数字均为该配置组织的可见行数，不宣称已越过组织边界盘点其他不可见资料。 |
| 权限 | 保持 RLS（行级安全）开启；本库 9 张 `schools_*` 学校表的 `row_security_active` 均为 true。没有修改角色、权限或策略，也没有绕过 RLS。 |
| 事务 | 连接默认 `default_transaction_read_only=on`；每次查询使用 `REPEATABLE READ READ ONLY`（可重复读、只读事务），最后 `ROLLBACK`（回滚结束只读事务）。statement timeout 为 15 秒、lock timeout 为 3 秒。没有执行结构修改或资料写入语句。 |
| 主要统计时点 | **2026-10-09 19:04:49.72847 UTC**，即 **2026-10-10 03:04:49.72847 UTC+8**。七项主数量在同一只读事务内查询；元数据及字段来源的补充核对也仅用只读事务。 |
| 已安装基线标记 | `tianxing_baseline.installations`（数据库基线安装标记表）一行：`baseline_id=tianxing-one-role-v1`、`transform_version=one-role-transform-v3`、`source_migration_count=74`；安装时间 `2026-09-20 12:34:59.216369 UTC`。这是安装标记，不能推导为本轮执行了 74 个迁移。 |

首次连接尝试使用 `postgres`（镜像初始化账号），收到 `role "postgres" is not permitted to log in`；核对现有本地角色约定后改用已有应用账号，查询成功。没有为解决该错误执行任何角色修改或初始化操作。

**本库不能代表将来要迁移的那个库。**本轮只确认当前本地实例含 3 所合成学校；没有取得将来迁移目标的实例、环境及数据范围确认，没有连接共享、测试或生产库。结构安装标记存在、磁盘有 585 校、以前曾有演示结果，都不能证明本次本地库就是迁移目标或与其资料一致。因此下面数字只能作为本地事实，不能称为迁移前库存已盘点完成。

## 七项数量及统计条件

下表业务查询统一限定 `organization_id=current_setting('app.organization_id')::uuid`。除明确写出的状态或引用条件外，不额外排除历史行。

| 用户要求 | 实际数量 / 结论 | 来源表与统计条件 |
| --- | --- | --- |
| 1. 学校总数与来源 | **学校 3 所；真实爬虫发现 0 所；人工建档 0 所；合成种子 3 所。**现有 3 所不能归入真实爬虫学校。 | `public.schools_schools`（学校身份表）COUNT 为 3；`public.schools_provisional_records`（人工建档原始记录表）COUNT 为 0。全部 3 所均关联 `public.schools_snapshot_records`（学校快照记录表）及 `public.schools_snapshots`（快照批次表）中的 `source_release_id='env01-synthetic-schools-v1'`；12 项逐字段 provenance 的 `source_kind` 全部为 `synthetic_seed`，没有其他来源组。没有既非人工建档、又不属于该合成快照的学校。此分类只描述本轮事实，不新增业务来源枚举。 |
| 2. 快照与版本 | **快照批次总数 1，当前 active（生效）批次 1；校级原始快照记录/版本 3，属于 active 批次的基础资料版本 3；持久化解析版本 0；人工建档版本 0。** | `schools_snapshots` 全量 COUNT / `status='active'`；`schools_snapshot_records` 全量 COUNT，以及按 `snapshot_id`、`organization_id` 联结 active 批次后的 COUNT；`public.schools_resolved_revisions`（不可变解析版本表）全量 COUNT 为 0；人工建档表 COUNT 为 0。不同粒度不能相加成一个“总版本数”。当前基础资料有 3 份，不等于已有 3 份持久化解析版本，也不证明浏览器流程验收。 |
| 3. 已提交 ChangeRequest（学校资料修订请求） | **总数 0；待审批 0；已批准 0；已拒绝 0；另 disabled（已停用修订）0。** | `public.schools_overlay_revisions`（修订请求/版本表）COUNT，并分别按 `status='candidate'/'approved'/'rejected'/'disabled'` 过滤。当前学校修订提交代码把请求写入该表的 candidate 状态；不是将不存在的 ChangeRequest 同名表当 0。`public.schools_change_review_receipts`（修订审批回执表）也为 0。 |
| 4. 招生记录或招生候选 | **结构不存在。数量不填 0。** | 实际库的学校关系只有上述学校/快照/修订/解析及审核表，没有独立招生记录或招生候选关系；跨非系统 schema 的关系目录也没有 admission/enrol 对应关系。现有 3 行快照的 `fields_json` 均无 admissions 数组。`schools_snapshots.status='candidate'` 是快照批次状态；Cases 候选名单是案件选校资料，均不能充当招生候选数量。 |
| 5. Cases（案件模块）固定引用 | **被固定引用的不同学校解析版本 0；涉及 ServiceCase（服务案件）0、SchoolTarget（学校目标）0。**作为区分，本库 ServiceCase 总数 **2**，SchoolTarget 总数 **0**。 | 合并 `public.cases_school_targets`（学校目标表）及 `public.cases_candidate_school_list_items`（候选名单学校项表）的非空 `pinned_resolved_revision_id`，COUNT DISTINCT 解析版本、`service_case_id` 和对应目标。不限目标状态、不只看当前名单。两张表目前均为 0 行；`public.cases_candidate_school_list_versions`（候选名单版本表）也为 0。服务案件总数来自 `public.cases_service_cases`（服务案件表），不能误写成“数据库没有案件”。 |
| 6. 进行中的抓取任务 | **本库结构不存在，进行中数量查不到；不填 0。** | `to_regclass('public.crawler_runs')` 为空；非系统 schema 无 crawler/crawl/job/run 对应任务关系。现有兼容爬虫代码在另一连接配置下使用 crawler_runs，但不能用源码中的建表语句证明本地库有表。本轮没有调用会自动建表的兼容读取，也没有连接该其他数据库。 |
| 7. 585 校磁盘快照的入库对应 | **按原来源键对应：学校表匹配 0/585，快照记录表匹配 0/585，两个表均无对应的 585/585。**未发现这份快照的入库证据。**若问曾经改键/改绑后的跨键对应：查不到。** | 从磁盘 records.json 读取全部 585 个不同 `school_key`，以其字面值等值匹配两个表的 `source_school_key`，不按名称或网址猜身份。库内现有 3 所均为上述合成种子；其快照 fields_json 中 school_number、edb_school_number、edb_number 全部为空/缺失，没有可核对的 EDB（教育局学校编号）对应。不能把“585 个原键未匹配”扩大为对所有未记录的改键历史都已排除。 |

### 版本与来源口径说明

本地只有一个 active 批次，声明 `record_count=3`，实际关联记录也是 3。当前源码的基础资料读取条件是 active 快照加该校记录，见[当前资料读取](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-resolved-view-transaction.ts:147)；持久化解析版本另有表，本库为空。这里分别报告批次、原始校级版本和解析版本，不以学校行上的版本计数器冒充历史版本数量。

来源分类由本轮库内 release/provenance 标记核对，不因存在快照就说是爬虫取得。合成标记与[现有合成 fixture](/Users/karo/Documents/Tianxingguoji/scripts/db/neon-test-synthetic-fixture.ts:188)的来源表达一致；本票只读该文件，没有执行其 seed（初始化资料写入）。本库的 3 所与磁盘 585 校是两种资产，不能把两者相加或假定已经导入。

学校修订请求所用表和状态可在[提交实现](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/postgresql-change-repository.ts:102)核对。兼容爬虫任务表的源码位于[兼容存储入口](/Users/karo/Documents/Tianxingguoji/modules/schools/infrastructure/crawler/db.ts:67)，它的自动建表函数未执行；“本地无表”不代表其他环境也无表。

## 可复核的查询口径

以下只列本轮使用的只读 SQL 及 psql 变量用法，不是待执行迁移、脚本或接口。所有业务查询沿用上表组织条件。`inventory_org` 是既有本地组织标识，`disk_school_keys` 是从现有 records.json 读取的 585 个 school_key 的 JSON 数组；二者作为内存变量传入，均不是本票写入数据库的新数据。组织标识及连接秘密未写入报告。

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ READ ONLY;
-- 以既有本地配置作事务内组织上下文；本轮 psql 使用变量，不输出其值。
SELECT set_config('app.organization_id', :'inventory_org', true) AS inventory_org \gset

SELECT count(*) FROM schools_schools
WHERE organization_id=current_setting('app.organization_id')::uuid;
SELECT count(*) FROM schools_provisional_records
WHERE organization_id=current_setting('app.organization_id')::uuid;

SELECT p.value->>'source_kind' AS field_source_kind,
       count(*) AS field_entries, count(DISTINCT r.school_id) AS schools
FROM schools_snapshot_records r
CROSS JOIN LATERAL jsonb_each(r.provenance_json) p
WHERE r.organization_id=current_setting('app.organization_id')::uuid
GROUP BY p.value->>'source_kind';
SELECT source_release_id,status,record_count FROM schools_snapshots
WHERE organization_id=current_setting('app.organization_id')::uuid;

SELECT count(*) AS total,
       count(*) FILTER(WHERE status='active') AS active,
       count(*) FILTER(WHERE status='candidate') AS candidate,
       count(*) FILTER(WHERE status='retired') AS retired
FROM schools_snapshots
WHERE organization_id=current_setting('app.organization_id')::uuid;
SELECT count(*) FROM schools_snapshot_records
WHERE organization_id=current_setting('app.organization_id')::uuid;
SELECT count(*) FROM schools_snapshot_records r
JOIN schools_snapshots s ON s.id=r.snapshot_id AND s.organization_id=r.organization_id
WHERE r.organization_id=current_setting('app.organization_id')::uuid AND s.status='active';
SELECT count(*) FROM schools_resolved_revisions
WHERE organization_id=current_setting('app.organization_id')::uuid;

SELECT count(*) AS total,
       count(*) FILTER(WHERE status='candidate') AS pending_approval,
       count(*) FILTER(WHERE status='approved') AS approved,
       count(*) FILTER(WHERE status='rejected') AS rejected,
       count(*) FILTER(WHERE status='disabled') AS disabled
FROM schools_overlay_revisions
WHERE organization_id=current_setting('app.organization_id')::uuid;

WITH refs AS (
  SELECT service_case_id,id AS school_target_id,pinned_resolved_revision_id AS revision_id
  FROM cases_school_targets
  WHERE organization_id=current_setting('app.organization_id')::uuid
    AND pinned_resolved_revision_id IS NOT NULL
  UNION ALL
  SELECT service_case_id,school_target_id,pinned_resolved_revision_id
  FROM cases_candidate_school_list_items
  WHERE organization_id=current_setting('app.organization_id')::uuid
    AND pinned_resolved_revision_id IS NOT NULL
)
SELECT count(*) AS reference_rows,count(DISTINCT revision_id) AS school_versions,
       count(DISTINCT service_case_id) AS service_cases,
       count(DISTINCT school_target_id) AS school_targets
FROM refs;
SELECT count(*) FROM cases_service_cases
WHERE organization_id=current_setting('app.organization_id')::uuid;
SELECT count(*) FROM cases_school_targets
WHERE organization_id=current_setting('app.organization_id')::uuid;
SELECT count(*) FROM cases_candidate_school_list_versions
WHERE organization_id=current_setting('app.organization_id')::uuid;
SELECT count(*) FROM cases_candidate_school_list_items
WHERE organization_id=current_setting('app.organization_id')::uuid;

SELECT n.nspname,c.relname,c.relkind
FROM pg_catalog.pg_class c JOIN pg_catalog.pg_namespace n ON n.oid=c.relnamespace
WHERE n.nspname NOT IN ('pg_catalog','information_schema')
  AND c.relkind IN ('r','p','v','m','f')
  AND c.relname ~ '(admission|enrol|crawl|(^|_)jobs?($|_)|(^|_)runs?($|_))';
SELECT to_regclass('public.crawler_runs');
SELECT count(*) FROM schools_snapshot_records
WHERE organization_id=current_setting('app.organization_id')::uuid
  AND jsonb_typeof(fields_json->'admissions')='array';

WITH disk AS (
  SELECT value AS school_key FROM jsonb_array_elements_text(:'disk_school_keys'::jsonb)
)
SELECT count(*) AS disk_rows,
       count(*) FILTER(WHERE EXISTS(
         SELECT 1 FROM schools_schools s
         WHERE s.organization_id=current_setting('app.organization_id')::uuid
           AND s.source_school_key=d.school_key)) AS matched_school_rows,
       count(*) FILTER(WHERE EXISTS(
         SELECT 1 FROM schools_snapshot_records r
         WHERE r.organization_id=current_setting('app.organization_id')::uuid
           AND r.source_school_key=d.school_key)) AS matched_snapshot_rows,
       count(*) FILTER(WHERE NOT EXISTS(
         SELECT 1 FROM schools_schools s
         WHERE s.organization_id=current_setting('app.organization_id')::uuid
           AND s.source_school_key=d.school_key)
         AND NOT EXISTS(
         SELECT 1 FROM schools_snapshot_records r
         WHERE r.organization_id=current_setting('app.organization_id')::uuid
           AND r.source_school_key=d.school_key)) AS without_either_key_match
FROM disk d;
ROLLBACK;
```

不存在关系的结论还结合了实际 9 张学校表、Cases 固定引用列的目录查询和当前源码，未仅凭上面的名字搜索把“名称不含 admission”绝对等同于无业务结构。抓取任务/招生结构不存在时，不对不存在的关系运行 COUNT，也不把目录匹配关系数 0 写成业务记录数 0。

## 本轮实际查询结果汇总

下面保留主要统计事务的输出列名及值；所有数值均来自成功执行的只读查询，不是源码估算。

```text
observed_at_utc = 2026-10-09 19:04:49.72847
database_name = tianxing
server_version = 17.10
reader_role = tianxing_app
read_only = on
default_read_only = on
isolation = repeatable read
configured_organization_rows = 1
scoped_school_tables = 9
all_school_table_rls_active = true
schools_total = 3
manual_intake_school_count = 0
synthetic_school_count = 3
synthetic_field_entries = 12
schools_without_manual_or_env01_seed_source = 0
snapshots_total = 1
active_snapshots = 1
candidate_snapshots = 0
retired_snapshots = 0
source_release_id = env01-synthetic-schools-v1
record_count = 3
snapshot_record_versions = 3
active_base_school_versions = 3
persisted_resolved_school_versions = 0
change_requests_total = 0
pending_approval = 0
approved = 0
rejected = 0
disabled = 0
review_receipts = 0
service_cases_total = 2
school_targets_total = 0
candidate_list_versions = 0
candidate_list_items = 0
fixed_reference_rows = 0
fixed_school_versions = 0
involved_service_cases = 0
involved_school_targets = 0
admission_or_crawl_relations = 0  [关系目录结果，不是招生/任务数]
crawler_runs_exists = false
snapshot_rows_with_admissions_array = 0  [现有快照形状核对，不是招生数]
disk_rows = 585
matched_school_rows = 0
matched_snapshot_rows = 0
disk_rows_without_either_key_match = 585
snapshot_rows_with_edb_number = 0
baseline_id = tianxing-one-role-v1
transform_version = one-role-transform-v3
source_migration_count = 74
manifest_sha256 = 8f27be8df0bbc4ac242dcb0deceee99dd2b46152dd91d05b4b9213d4324afee0
installed_at = 2026-09-20 12:34:59.216369 UTC
transaction_end = ROLLBACK
psql_exit_code = 0
```

磁盘源文件：[records.json](/Users/karo/Documents/Tianxingguoji/data/crawler-source/latest/records.json)，585 行、585 个不同 school_key，SHA256 为 `1736b8a83e5066eb068834869e6b17bfb7902351a045f0241274929d2d6c5f0f`。没有导入、补字段、关联或改写这份文件。

## 未执行与结论边界

没有运行迁移、baseline 安装、seed、导入、抓取、清理、应用接口、自动建表入口、测试或部署；没有调整容器、环境配置、账号、角色、RLS 或任何业务数据。数据库请求只有连接检查、只读目录/聚合查询和事务内组织上下文设置。

没有盘点任何未来迁移目标、其他本地实例或远端库，没有识别跨键改绑的历史对应。**“结构不存在”“查不到”保持为这些事实，未用 0 冒充业务记录数量。**本票仅新增本报告；需求、既有两份报告及原抓取产物均保持原样。
