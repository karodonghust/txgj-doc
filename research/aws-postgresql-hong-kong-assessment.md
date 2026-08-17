# AWS 香港 PostgreSQL：RDS Multi-AZ 與 Aurora 評估

> 研究日期：2026-07-31（Asia/Hong_Kong）  
> 適用範圍：天星 Release 1、約 100 個活躍 K12 案件、低併發但含敏感 PII、案件授權、審計與文件 metadata。  
> 來源原則：只採 AWS／Vercel 官方文件、SLA 和 AWS Price List API。未建立雲端資源，價格為 USD 按需牌價，不含稅、Support、application runtime、NAT、log、監控和資料遷移工時。

## 一、結論

**建議 Release 1 使用 Amazon RDS for PostgreSQL Multi-AZ DB instance，香港 `ap-east-1`，初始規格 `db.t4g.small + 20 GiB gp3`。**

推薦理由：

1. 目前目標只有約 100 個活躍案件，交易量和資料量低，不需要 Aurora 的 15 個 reader、快速橫向讀擴展或細粒度 autoscaling。
2. RDS PostgreSQL 比 Aurora PostgreSQL 更接近標準 PostgreSQL，從 Neon 遷移及日後離開 AWS 的相容性較高。
3. `db.t4g.small` Multi-AZ 加 20 GiB gp3 的核心按需價約 `$87.03/月`；同等符合 Aurora 99.99% Multi-AZ SLA 的最低常駐方案約 `$163–185/月` 起。
4. RDS Multi-AZ 提供同步 standby 和 99.95% 月 SLA，對 PRD 目前 99.5% 月可用性目標有餘量。
5. Aurora 的更高 SLA 和較快 failover 是實質優勢，但現階段每月增加約 `$76–98`，尚無已量測的產品或容量證據支持。

這項建議有一個不可忽略的架構條件：**RDS 應保持 private，敏感 API runtime 應位於同一香港 VPC，或使用已驗證的香港 private connectivity。** RDS Proxy 不可公開存取。若堅持由普通 Vercel Functions 直接連 private RDS，會被迫採 Vercel Enterprise Secure Compute/VPC peering，或把 RDS 暴露為 public endpoint；後者不建議作生產基線。

## 二、比較基準

### 2.1 工作負載假設

| 維度 | 規劃假設 |
| --- | --- |
| 活躍案件 | 100 |
| retained cases | 初期少於 1,000 |
| 內部帳號 | 小型團隊，代表性併發低於 20 |
| 一般 API 目標 | P95 < 500 ms |
| DB 資料 | Student、Guardian、Case、Assessment、Task、School overlay、authz、audit、document metadata |
| 初始 allocated storage | 20 GiB gp3 |
| 可靠性 | 單 AZ 故障自動恢復；不得跨出香港 failover |
| 駐留 | DB、backup、logs、KMS、connection processing 均在 `ap-east-1` |

若實測持續 CPU、memory、connections、storage 或 IOPS 超過既定門檻，應垂直升級 instance 或重新評估 Aurora；不能把 100 案件數直接當成效能證據。

## 三、可靠性與故障恢復

| 項目 | RDS PostgreSQL Multi-AZ DB instance | Aurora PostgreSQL Multi-AZ |
| --- | --- | --- |
| 月 SLA | 99.95% | 99.99% |
| 同區資料冗餘 | primary + 不同 AZ synchronous standby | 6 個 storage nodes，跨 AZ 同步；compute 需至少兩 AZ 的 instances 才符合 Multi-AZ SLA |
| standby/reader | standby 不提供讀流量 | reader 可讀，writer 故障時升級 |
| 典型 failover | 60–120 秒；大 transaction/recovery 可能更久 | 有 reader 時通常較快；無 reader 需重建 writer，時間更長 |
| endpoint | failover 後 DNS 指向新 primary | cluster endpoint 指向 writer |
| PITR | transaction logs 約每 5 分鐘上傳；restore 產生新 DB instance | continuous backup/PITR；restore 產生新 cluster |
| backup retention | DB instance 0–35 天；生產不可設 0 | 1–35 天 |
| region failure | 不跨區自動 failover | 本案禁止 Aurora Global Database／跨區 replica |

SLA 的唯一標準救濟是 future service credit，不是資料或業務損失賠償。RDS Multi-AZ 仍需處理 DNS refresh、連線重建、in-flight transaction rollback、短暫 5xx 和 retry budget。

### 3.1 建議失敗契約

- transaction 只在明確可重試且有 idempotency key 時重試；未知 commit outcome 不可盲目重放外部副作用。
- DB failover 時 API 回傳 `503 DATABASE_TEMPORARILY_UNAVAILABLE` 和 request ID，不把錯誤細節送到 client。
- 連線恢復後由 outbox worker 重做尚未送出的通知；不得重做已留下 delivery receipt 的副作用。
- PITR 不是 deployment rollback。PITR 會建立新 instance，需校驗資料、切換 secret/endpoint、重啟 pool 和做 smoke tests。
- schema rollback 使用 expand/contract 和 forward repair；不可假設 restore 整個 DB 能只回退單一 migration。

## 四、香港按需官價

AWS Price List API `AmazonRDS current/ap-east-1` publication date：`2026-07-29T23:42:48Z`。

### 4.1 RDS PostgreSQL

| 項目 | `ap-east-1` 官價 | 730 小時月估算 |
| --- | ---: | ---: |
| `db.t4g.micro` Multi-AZ | $0.056/hour | $40.88 |
| `db.t4g.small` Multi-AZ | $0.111/hour | $81.03 |
| `db.t4g.medium` Multi-AZ | $0.222/hour | $162.06 |
| Multi-AZ gp3 | $0.30/GiB-month | 20 GiB = $6.00 |
| 額外 backup storage | $0.095/GiB-month | active DB storage 100% 以內免額外費用 |
| T4g CPU credit | $0.075/vCPU-hour | 持續 burst 才產生 |
| RDS Proxy | $0.022/vCPU-hour | `t4g.small` 2 vCPU 約 $32.12/月 |

`db.t4g.small` 是 2 vCPU、2 GiB RAM。20–399 GiB gp3 baseline 為 3,000 IOPS／125 MiB/s；初期無需購買額外 IOPS。

建議核心月預算：

```text
compute = $0.111 * 730 = $81.03
storage = 20 GiB * $0.30 = $6.00
core = $87.03/month

optional RDS Proxy = 2 vCPU * $0.022 * 730 = $32.12/month
core with proxy = $119.15/month
```

未計入：CloudWatch logs/alarms、Performance Insights 長期保留、Secrets Manager、KMS、VPC NAT、application runtime、額外 snapshots 和 Support。

### 4.2 Aurora PostgreSQL

| 項目 | `ap-east-1` 官價 | 說明 |
| --- | ---: | --- |
| Serverless v2 Standard | $0.22/ACU-hour | 每個 writer/reader 分別計費 |
| Serverless v2 IO-Optimized | $0.29/ACU-hour | 本案無高 I/O 證據，不採用 |
| Provisioned `db.t4g.medium` | $0.125/hour/instance | Multi-AZ 至少 writer + reader |
| Standard storage | $0.12/GiB-month | 按實際 consumed storage |
| Standard I/O | $0.24/million requests | IO-Optimized 才取消 I/O charge |
| 額外 backup storage | $0.023/GiB-month | free allocation 以上 |

同等 Multi-AZ 對比：

```text
Serverless v2, writer 0.5 ACU + reader 0.5 ACU always active
= 1 ACU * $0.22 * 730
= $160.60 compute
+ 20 GiB storage $2.40
+ 1 million I/O $0.24
= about $163.24/month

Provisioned, writer + reader db.t4g.medium
= 2 * $0.125 * 730
= $182.50 compute
+ 20 GiB storage $2.40
+ 1 million I/O $0.24
= about $185.14/month
```

Aurora Serverless v2 可把 minimum 設為 0 ACU 以 auto-pause；pause 時 compute charge 為 0。但是：

- 典型 resume 約 15 秒，深度 sleep 可達 30 秒或更久，與一般 API P95 < 500 ms 不相容。
- writer 和 failover-priority 0/1 reader 會一起 pause/resume。
- health check、連線、maintenance 或其他活動會阻止或喚醒 pause。
- 若保持 0.5 ACU 常駐，兩個 AZ 的 writer+reader 最低就是合計 1 ACU。

因此不能用「大部分時間一定為 0 ACU」作 production 預算前提。Auto-pause 適合 preview/dev 或可接受 cold start 的工作流，不適合作為內部 CRM 核心 SLA 的唯一手段。

## 五、連線與 runtime 耦合

### 5.1 RDS private connectivity

RDS Proxy 必須和 DB 位於同一 VPC，且不能 public access。這對當前 Vercel 架構有三條路：

| 路徑 | 優點 | 代價／風險 | 建議 |
| --- | --- | --- | --- |
| 敏感 API runtime 遷到 AWS `ap-east-1` 同 VPC | private network、香港邊界清楚、可用普通 pool/RDS Proxy | 增加 AWS runtime/deploy 邊界 | 推薦進一步設計 |
| Vercel Enterprise Secure Compute + same-region VPC peering | 保留 Vercel Functions，private connectivity | Enterprise 採購、價格需詢價；需證明 log/runtime/support 駐留 | 只在有合約報價後比較 |
| Public RDS endpoint + IP allowlist/TLS | 表面最少改動 | database 暴露 public routing、來源 IP/egress 管理、駐留證據弱 | 不建議作 production 基線 |

Vercel Secure Compute 是 Enterprise 付費功能；官方文件說 VPC peering 流量不收 Private Data Transfer fee，但 Enterprise 基礎價格需詢價。其跨 region passive failover 必須停用，否則違反香港駐留。

### 5.2 Aurora Data API

Aurora PostgreSQL Data API 在香港支援指定 engine versions，可透過 regional HTTPS + IAM 存取而不維護 persistent connection。它能降低 VPC connection management 複雜度，但不能自動解決：

- Vercel function/log/support 是否留港。
- Data API 交易、payload、timeout 和 SQL client 相容性。
- Aurora Multi-AZ 的較高常駐成本。
- 未來離開 Aurora/Data API 的 vendor lock-in。

因此 Data API 是 Aurora 的真實優勢，但不足以在當前規模推翻 RDS 建議。

## 六、migration、備份和回滾方向

### 6.1 從 Neon 遷移

1. 盤點 Neon region、schema、row counts、extensions、functions、roles、secrets 和現有資料分類。
2. 建立版本化 migration baseline；停止以 runtime `CREATE TABLE IF NOT EXISTS` 擁有 schema。
3. 在香港 staging RDS 執行 schema migration，再用去識別 fixture 做契約和負向授權測試。
4. 若 production Neon 有資料，使用 `pg_dump/pg_restore` 或受控 logical migration；切換前進入短暫寫入凍結，核對 counts/checksums/referential integrity。
5. 切換 application secret/endpoint，做 smoke test，再開放寫入。
6. Neon 保持 read-only quarantine 到 rollback window 結束；確認不需回退後，才另行批准刪除。

不得在未設計 dual-write reconciliation 前使用長期雙寫。短暫寫入凍結比不可靠雙寫更符合目前小型內部系統的風險比例。

### 6.2 建議 backup 基線（待業務批准）

- Automated backup retention：7 天起步，PITR enabled。
- 每日 automated backup；每次重大 migration 前建立 manual snapshot。
- 所有 backup/snapshot/KMS key 留在 `ap-east-1`，禁止 cross-region copy。
- 每月至少一次 restore drill 到隔離香港環境；驗證不是「snapshot 存在」，而是 schema、row counts、核心旅程和權限可用。
- RPO 規劃起點 5 分鐘，RTO 規劃起點 4 小時；需以實測 restore 時間由業務 owner 正式批准。

## 七、何時重新評估 Aurora

任一條件被量測證明後重新評估，而不是按案件數主觀升級：

- `db.t4g.medium` 仍持續 CPU/memory/connection 飽和。
- 讀流量需要多個 reader 或無停機橫向擴展。
- RDS 60–120 秒 failover 已造成不可接受的案件操作失敗，且業務願意為 99.99% SLA 付費。
- 工作負載高度尖峰、長時間完全閒置，Aurora Serverless 實測總成本低於 RDS。
- Aurora Data API 能明顯降低 Vercel Enterprise/private networking 的總成本，且供應商鎖定獲批准。

## 八、批准後的最小驗收

1. RDS、standby、backup、snapshot、KMS、logs 和 secrets 都在 `ap-east-1`；不存在 cross-region copy。
2. DB private；匿名網路、錯誤 security group、錯誤 role 和非 TLS 連線被拒絕。
3. 強制 failover 演練記錄實際 downtime、DNS/pool 恢復和 transaction outcome；API 有可恢復錯誤。
4. PITR restore drill 建立新 instance，通過 row counts、constraints、authz negative tests 和核心旅程。
5. Migration expand/contract、舊版 app 相容 window、forward repair 和 feature flag 經 staging 演練。
6. 以代表性 100-case fixture 測量 P95 latency、connections、CPU、memory、storage、IOPS 和 monthly cost。
7. 超過 `$150/月` DB+connection core budget 或連續兩次 failover/restore 未達 RTO 時停止擴大試營運，先分類原因。

## 九、官方來源

- [Amazon RDS SLA](https://aws.amazon.com/rds/sla/)：Multi-AZ 99.95%、Single-DB 99.5%；查核於 2026-07-31。
- [Amazon Aurora SLA](https://aws.amazon.com/rds/aurora/sla/)：跨至少兩 AZ cluster 99.99%、Single-AZ 99.9%；查核於 2026-07-31。
- [RDS Multi-AZ failover](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html)：典型 60–120 秒、DNS 行為；查核於 2026-07-31。
- [RDS PITR](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html)：約每 5 分鐘上傳 transaction logs、restore 產生新 instance；查核於 2026-07-31。
- [RDS backup retention](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.BackupRetention.html)：0–35 天；查核於 2026-07-31。
- [RDS PostgreSQL pricing](https://aws.amazon.com/rds/postgresql/pricing/)：backup free allocation；查核於 2026-07-31。
- [RDS storage](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html)：gp3 size/baseline；查核於 2026-07-31。
- [Aurora high availability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)：六個 storage nodes、reader/failover；查核於 2026-07-31。
- [Aurora Serverless auto-pause](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html)：0 ACU、resume latency 和 limitations；查核於 2026-07-31。
- [Aurora Data API region support](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.Aurora_Fea_Regions_DB-eng.Feature.Data_API.html)：香港 engine version support；查核於 2026-07-31。
- [RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)：same-VPC、不可 public、pool/failover；查核於 2026-07-31。
- [Vercel Secure Compute](https://vercel.com/docs/networking/secure-compute)：Enterprise、香港同區 network/VPC peering、跨區 failover；頁面最後更新 2026-06-30。
- [Amazon RDS `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRDS/current/ap-east-1/index.json)：publication date `2026-07-29T23:42:48Z`。

