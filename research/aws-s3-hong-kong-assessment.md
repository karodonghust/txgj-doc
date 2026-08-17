# AWS S3 香港區域：學生文件儲存的穩定性、駐留與成本評估

> 研究日期：2026-07-31（Asia/Hong_Kong）  
> 適用範圍：天星 Release 1 的香港 K12 私密學生文件；不代表整套 CRM 已完成香港資料駐留或法律合規認證。  
> 來源原則：只採用 AWS 官方合約、產品文件、SLA、定價頁及 AWS Price List API。價格均為美元、按需公開牌價，不含稅、AWS Support、掃毒運算、CloudTrail data events、CloudWatch、資料庫及應用程式運算成本。

## 一、結論先行

**建議 Release 1 使用 AWS S3 Standard，區域固定為 Asia Pacific (Hong Kong) `ap-east-1`。** 對這個小型教育 CRM，S3 的物件儲存與請求費很低；成本風險主要不是容量，而是大量對外下載、日誌/掃毒等配套服務，以及錯誤保留大量舊版本。

這個結論有四個邊界：

1. **穩定性適合作為生產文件庫，但不是零中斷承諾。** S3 Standard 設計耐久性為 99.999999999%（11 個 9），預設跨至少 3 個 Availability Zones；設計可用性為 99.99%，合約 SLA 為 99.9%。SLA 未達標的救濟是帳單 credit，不是資料或業務損失賠償。[S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)、[Amazon S3 SLA](https://aws.amazon.com/s3/sla/)
2. **S3 可以把物件固定在香港區域，但 S3 選區不等於全系統駐留。** AWS 文件明示：特定區域 bucket 中的物件不會離開該區域，除非客戶明確轉移；AWS Customer Agreement 也承諾不把 `Your Content` 移出客戶選擇的區域，法律或政府強制命令除外。[General purpose buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingBucket.html#general-purpose-buckets-overview)、[AWS Customer Agreement §1.4](https://aws.amazon.com/agreement/)
3. **香港強制駐留意味著不能用跨區容災換取區域級可用性。** 不啟用 Cross-Region Replication、Multi-Region Access Point 或區外備份。若整個香港區域不可用，文件功能進入只顯示 metadata／暫停上下載的降級狀態，等香港區域恢復。若業務要求區域級災難仍可讀取，只能另設一個仍位於香港的獨立備份/供應商，不能把副本放到新加坡後仍聲稱「香港駐留」。
4. **正式採購前仍需法律與架構 gate。** 必須明確「駐留」是否只涵蓋文件內容，還包括物件 key、文件 metadata、預覽快取、掃毒樣本、CloudTrail/應用日誌、備份及客服工單。AWS 合約中的 `Your Content` 不等同於所有 account/service metadata；不能把產品文件當成香港 PDPO 的法律意見。

## 二、可靠性判斷

### 2.1 S3 Standard 的可用性與耐久性

| 指標 | S3 Standard | 對本專案的含義 |
| --- | ---: | --- |
| 設計耐久性 | 99.999999999% | 適合保存不可重建的學生文件，但仍需版本、刪除保護和可恢復演練 |
| 儲存範圍 | 至少 3 AZ | 香港區域在推出時即為 3-AZ；單一資料中心故障不應要求應用切換 bucket |
| 設計可用性 | 99.99% | 是設計目標，不是每月保證 |
| 可用性 SLA | 99.9% | 月可用性低於 99.9% 才開始有 10% service credit；低於 99.0% 為 25%，低於 95.0% 為 100% |
| 首 byte latency | milliseconds | 適合正常上傳、預覽和下載 |

來源：[S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)、[S3 SLA](https://aws.amazon.com/s3/sla/)、[AWS Hong Kong Region launch](https://aws.amazon.com/blogs/aws/now-open-aws-asia-pacific-hong-kong-region/)。

不能從 11 個 9 推導「服務永不離線」；耐久性回答的是物件永久丟失風險，可用性回答的是當下能否讀寫。應用仍要處理 timeout、429/5xx、使用者網路中斷、簽名 URL 過期和掃描工作卡住。

### 2.2 生產級失敗處理

建議的文件狀態機：

```text
pending_upload -> quarantined -> scanning -> available
                              -> rejected
                              -> scan_failed -> scanning（有界重試）

available -> superseded
          -> retention_hold
          -> pending_delete -> deleted
```

關鍵不變量：

- `available` 之前，應用不得簽發下載 URL；bucket 私有不是這條業務規則的替代品。
- 每個文件版本使用新的不可猜 UUID object key，不覆寫舊 key；Neon 的 `active_document_version_id` 是正式版本指標。
- 啟用 S3 Versioning 作防誤刪的第二層保護。S3 Versioning 會保留完整物件版本，覆寫不是 diff，因此必須配合 noncurrent-version lifecycle 和保留期，否則費用會持續累積。[S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- 掃描事件必須冪等。S3 Event Notifications 是 at-least-once、可能重複且不保證順序；以 `(bucket, key, version_id, scan_policy_version)` 作唯一工作鍵，重複事件回傳既有結果，並以定時 reconciliation 找出長時間停在 `quarantined/scanning` 的物件。[Event notification types and destinations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-how-to-event-types-and-destinations.html#event-ordering-and-duplicate-events)
- 使用 SQS + DLQ 解耦上傳與掃描；SQS、Lambda/掃描運算及 DLQ 全部在 `ap-east-1`。AWS 要求直接 S3 event destination 的 SNS、SQS、Lambda 與 bucket 位於同一區域。[Event notification destinations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-how-to-event-types-and-destinations.html#supported-notification-destinations)
- 下載失敗不改動文件狀態；client 對安全的讀取可用 SDK 指數退避重試。上傳完成通知丟失時，由 reconciliation 依 S3 object HEAD 和 DB 狀態修復，而不是要求使用者重傳。

## 三、香港資料駐留設計

### 3.1 必須固定在 `ap-east-1` 的資料與運算

```text
瀏覽器
  | 1. 向應用申請短期 upload intent（只傳 metadata）
  | 2. 直接 TLS 上傳到 s3.ap-east-1.amazonaws.com
  v
S3 私有文件 bucket（ap-east-1）
  -> SQS 掃描佇列（ap-east-1）
  -> 掃描運算（ap-east-1）
  -> 掃描結果/DLQ（ap-east-1）
  -> CloudTrail、S3 access logs 與 audit bucket（ap-east-1）
```

應用不可代理文件 bytes 經過位於香港以外的 Vercel Function。應由服務端完成授權後簽發短期 presigned PUT/GET，browser 直接連到香港 S3 regional endpoint。預覽若需要轉圖/OCR，也必須在香港區域運行，輸入、暫存、輸出及日誌均留在香港。

**整套 CRM 的駐留仍有未解風險：** 若 Neon 裡的文件 metadata、學生姓名/案件關係或應用日誌位於香港以外，S3 合規不會修補這個缺口。object key 只放隨機 ID，不放姓名、學校、電話或證件資訊；KMS encryption context 也不可放敏感資料，因 AWS 明示 encryption context 本身不加密。[Using SSE-KMS](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html#encryption-context)

### 3.2 區域護欄

最低控制集：

- AWS account 明確啟用 opt-in Region `ap-east-1`；以 Organizations SCP/IAM condition 拒絕為文件工作負載在其他區域建立資源。
- bucket Region 固定 `ap-east-1`，啟用 account-level 及 bucket-level Block Public Access，停用 ACL，所有存取經 IAM role + bucket policy。
- bucket policy 拒絕非 TLS (`aws:SecureTransport=false`)、錯誤 KMS key、未授權 principal，以及不符合預期 object-key prefix 的請求。
- 不啟用 CRR、Multi-Region Access Point、Transfer Acceleration、CloudFront 文件快取或任何區外備份。AWS 的 CRR 會把物件複製到不同 Region；SRR 才是同區複製。[S3 Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- KMS 使用 `ap-east-1` **single-Region customer managed key**；不要使用 multi-Region replica。AWS 要求 SSE-KMS key 與 S3 bucket 位於同一區域。[Using SSE-KMS](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html)
- CloudTrail object-level data events 和 S3 server access logs 均送往香港 audit bucket；停用 CloudWatch Logs 的跨區聚合。AWS 的 S3 server-log bucket 必須與來源 bucket 位於同一 Region 及 account。[S3 server access logging](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html)
- AWS Config/部署檢查持續驗證 Region、public access、encryption、versioning、replication、logging 和 lifecycle；任何 drift 阻斷 release 並告警。

### 3.3 加密、審計與回滾

S3 自 2023-01-05 起已對所有新物件預設使用免費 SSE-S3；本專案仍建議 **SSE-KMS + customer managed key + S3 Bucket Key**，因為它能把 key policy、停用、輪換和使用審計納入自有控制。S3 Bucket Key 可把 S3 到 KMS 的 request traffic/cost 降低最多 99%。[Using SSE-KMS](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html#sse-kms-bucket-keys)

安全邊界：

- presigned URL 只授權單一 opaque key、單一 HTTP method、短時效；上傳另外限制 content length、允許 MIME 和 checksum。metadata 由服務端驗證，不能信任 browser 的 MIME。
- S3 key 和 KMS key policy 分開授權：應用簽名 role 可簽發受限操作，掃描 role 只讀 quarantine/寫掃描狀態，維運 break-glass role 需 MFA、原因和告警。
- CloudTrail 記錄 bucket/object API；應用審計另記 actor、case、document version、授權 scope、request ID 和業務結果。AWS 建議 CloudTrail 用於 bucket-level 和 object-level actions。[Logging options for S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/logging-with-S3.html)
- S3 回滾只能恢復 bytes；正式業務回滾是 DB transaction 把 `active_document_version_id` 指回已掃描通過的舊版本，寫入 audit event。不能把未掃描或已撤銷版本設為 active。
- KMS key deletion 必須設為獨立的創始人 approval gate；停用或刪除 key 會令所有文件不可讀，不能被一般文件刪除流程觸發。
- Object Lock 適合不可變 audit log 或具有明確法定保留期的版本；不應在尚未定義 PDPO 刪除/保留規則前直接鎖住所有學生文件。[S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)

## 四、`ap-east-1` 公開牌價

下表取自 AWS Price List API 的 `current/ap-east-1` 官方資料：S3 版本發布於 2026-07-28，AWS Data Transfer 版本發布於 2026-07-20；KMS current 版本發布於 2025-08-28，查核日仍為 current。價格可能隨時調整，上線前應用 AWS Pricing Calculator 和實際帳單重新核對。

| 項目 | `ap-east-1` 價格 | 計價注意 |
| --- | ---: | --- |
| S3 Standard，首 50 TB/月 | $0.025 / GB-month | 50–500 TB 為 $0.024；500 TB 以上 $0.023 |
| PUT/COPY/POST/LIST | $0.005 / 1,000 requests | 完成 multipart、複製、掃描後改 key 都會增加請求 |
| GET 及其他 Tier-2 requests | $0.004 / 10,000 requests | 大量預覽通常仍是 egress 比 request 貴 |
| Internet Data Transfer OUT，首 10 TB | $0.12 / GB | AWS 每月提供 100 GB 免費 DTO，但在所有適用 AWS services/Regions 合併計算；不能假定完全由 S3 獨享 |
| Internet DTO，下一個 40 TB | $0.085 / GB | 其後 50–150 TB 為 $0.082；150 TB 以上 $0.080 |
| Customer managed KMS key | $1 / key-month | 自動/on-demand rotation 的第 1、2 個舊 key version 各再加 $1/月，之後封頂 |
| KMS symmetric requests | $0.03 / 10,000 requests | 每月首 20,000 requests 在所有 KMS Regions 合併免費；特定非對稱操作除外 |
| S3 Standard-IA | $0.0138 / GB-month | 另收 $0.01/GB retrieval、$0.01/1,000 transition；128 KB 最低計費物件和 30 日最低儲存期 |

官方定價資料：[Amazon S3 `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/ap-east-1/index.json)、[AWS Data Transfer `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDataTransfer/current/ap-east-1/index.json)、[AWS KMS `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/awskms/current/ap-east-1/index.json)、[AWS data transfer pricing and 100 GB free allowance](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer)、[AWS KMS pricing](https://aws.amazon.com/kms/pricing/)、[S3 pricing caveats](https://aws.amazon.com/s3/pricing/)。

## 五、三個月費情境

### 5.1 共通假設

- 只估文件 storage、PUT、GET、Internet DTO 及一把 customer managed KMS key。
- 一個「文件版本」是一個完整物件；不以壓縮率抵扣。
- KMS 保守按每次 PUT/GET 各一個 request 計算，**未計入 S3 Bucket Key 最多 99% 的降低**，因此 KMS request cost 偏高。
- 「有免費額度」假設帳戶的 100 GB DTO 和 20,000 KMS requests 未被其他服務消耗；「無免費額度」則假設已全部被其他工作負載消耗。
- AWS 以實際 bytes、請求類型、每日平均 GB-month 和下載 bytes 計費；以下使用十進位 MB/GB 作規劃估算，會與帳單有小幅差異。

### 5.2 情境 A：試營運

假設：30 個 retained cases；每案 25 個文件版本；平均 4 MB；每月 1,000 PUT、5,000 GET、5 GB 對外下載。

```text
storage = 30 * 25 * 4 MB / 1,000 = 3 GB
storage cost = 3 * $0.025 = $0.075
PUT = 1,000 / 1,000 * $0.005 = $0.005
GET = 5,000 / 10,000 * $0.004 = $0.002
KMS key = $1.000
KMS requests = 6,000 * $0.000003 = $0.018（有免費額度時為 $0）
DTO = 5 * $0.12 = $0.600（有 100 GB 免費額度時為 $0）
```

**估計總額：$1.08/月（免費額度可用）至 $1.70/月（免費額度已用完）。**

### 5.3 情境 B：100 個活躍案件

假設：100 個 retained cases；每案 50 個文件版本；平均 5 MB；每月 5,000 PUT、30,000 GET、40 GB 對外下載。

```text
storage = 100 * 50 * 5 MB / 1,000 = 25 GB
storage cost = 25 * $0.025 = $0.625
PUT = 5,000 / 1,000 * $0.005 = $0.025
GET = 30,000 / 10,000 * $0.004 = $0.012
KMS key = $1.000
KMS requests with free tier = (35,000 - 20,000) * $0.000003 = $0.045
KMS requests without free tier = 35,000 * $0.000003 = $0.105
DTO = $0（免費額度可用）或 40 * $0.12 = $4.800
```

**估計總額：$1.71/月（免費額度可用）至 $6.57/月（免費額度已用完）。**

### 5.4 情境 C：成長期

假設：1,000 個 retained cases（含已結案）；每案 80 個文件版本；平均 6 MB；每月 40,000 PUT、300,000 GET、300 GB 對外下載。

```text
storage = 1,000 * 80 * 6 MB / 1,000 = 480 GB
storage cost = 480 * $0.025 = $12.000
PUT = 40,000 / 1,000 * $0.005 = $0.200
GET = 300,000 / 10,000 * $0.004 = $0.120
KMS key = $1.000
KMS requests with free tier = (340,000 - 20,000) * $0.000003 = $0.960
KMS requests without free tier = 340,000 * $0.000003 = $1.020
DTO with free tier = (300 - 100) * $0.12 = $24.000
DTO without free tier = 300 * $0.12 = $36.000
```

**估計總額：$38.28/月（免費額度可用）至 $50.34/月（免費額度已用完）。** 其中 63%–72% 是 Internet egress；即使再壓低 storage 價格，總成本也不會同比例下降。

### 5.5 估算未涵蓋的成本與不確定性

- CloudTrail S3 data events、CloudWatch Logs/metrics/alarms、AWS Config 和 audit-log storage。
- Lambda/container 掃毒、病毒特徵更新、SQS/DLQ、失敗重試和 reconciliation。
- quarantine 中的暫存物件、縮圖/OCR 輸出、multipart 未完成部分、server access logs，以及 S3 Versioning 的 noncurrent versions。
- Vercel 到 AWS 的控制流、Neon、身份供應商、Email、Support plan、稅及工程維運成本。
- 使用者重複預覽、下載大檔、Range requests 中斷，以及批量匯出會顯著增加 DTO。AWS 說明連線提早終止時，計費的 Data Transfer OUT 可能高於應用實際收到的 bytes。[S3 pricing](https://aws.amazon.com/s3/pricing/)

建議建立 AWS Budget：試營運 `$25/月`、100 案件 `$50/月` 告警，並對 DTO、current/noncurrent storage、KMS requests 和掃描失敗率分別監控；告警不是硬停服務，避免成本閾值意外阻斷文件存取。

## 六、是否值得使用 Standard-IA

Release 1 **先全部使用 S3 Standard**。原因不是 IA 不可靠，而是初期絕對節省太小：

- 25 GB 全部由 Standard 轉為 Standard-IA，理論 storage 差額只有 `25 * ($0.025 - $0.0138) = $0.28/月`，尚未扣 retrieval 和 transition。
- 成長情境若 400 GB 已結案、低存取文件轉為 Standard-IA，storage 節省 `400 * $0.0112 = $4.48/月`；若當月取回 20 GB，再扣 `$0.20` retrieval，淨節省約 `$4.28`。
- Standard-IA 有 30 日最低儲存期、128 KB 最低計費物件和 retrieval fee；頻繁替換、刪除或預覽的活躍文件不適合過早轉入。[S3 pricing](https://aws.amazon.com/s3/pricing/)

在累積 90 日真實 access metrics 後，可只把「案件已結案 + 文件版本超過 90 日 + 非 retention hold + 最近 60 日未讀」的物件轉 Standard-IA。Glacier 等 archive class 應等保留、取回時效、legal hold 和刪除流程定案後另行設計，不能只為幾美元節省引入數小時 restore 狀態。

## 七、Release 1 驗收與停止條件

上線前必須以外部證據證明：

1. 新增、覆寫、刪除、掃描、預覽、下載、日誌及 KMS key 均沒有建立香港以外的資源或副本。
2. account/bucket public access 全部阻斷；匿名、跨帳號、錯誤 case scope、過期 URL 和錯誤 KMS key 均被拒絕。
3. 重複及亂序 S3 events 不會重複發布文件或把較舊掃描結果覆蓋較新結果；DLQ 和 reconciliation 能恢復卡住工作。
4. 回滾演練可把 active pointer 切回已驗證舊版本，並保留前後 audit；誤刪可由 S3 version 恢復。
5. 香港區域無法存取時，系統進入明確降級狀態，不把文件靜默寫到其他 Region；恢復後可重試未完成工作且不重複建立 document version。
6. 以 30 個真實案例 dry-run 核對 object 數量、平均大小、PUT/GET、DTO、掃描時間、失敗率和帳單；實測超過估算 2 倍時停止擴大試營運並分類原因。
7. 法律/隱私 owner 簽署駐留資料分類、AWS DPA/合約審查、保留與刪除規則，以及「香港區域故障時不跨區 failover」的業務接受。

## 八、最終選型

**採用：AWS S3 Standard in `ap-east-1`，搭配單區 customer managed KMS key、S3 Bucket Key、S3 Versioning、香港區域內的 SQS/掃描運算及香港 audit bucket。**

**暫不採用：** S3 One Zone-IA、Cross-Region Replication、Multi-Region Access Point、CloudFront 文件快取、Transfer Acceleration，以及自動 Glacier lifecycle。One Zone class在單一 AZ 損毀時可能丟失資料，不適合不可重建的學生文件；跨區功能與香港強制駐留衝突。[S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)

這個選型在目前規模下的成本效益是合理的：典型 100 案件的 S3+KMS 核心費用約 `$1.71–$6.57/月`，遠低於自行維護文件伺服器的運維成本。真正需要治理的是資料流和授權，而不是為每 GB 再節省一美分。

## 九、官方來源與版本

- [AWS Customer Agreement](https://aws.amazon.com/agreement/)：區域選擇、內容移動和客戶備份責任；查核於 2026-07-31。
- [General purpose buckets overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingBucket.html#general-purpose-buckets-overview)：特定 Region 中物件不會離區，除非明確轉移；查核於 2026-07-31。
- [S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)：耐久性、可用性、AZ、latency 及 storage-class caveats；查核於 2026-07-31。
- [Amazon S3 SLA](https://aws.amazon.com/s3/sla/)：頁面標示最後更新 2023-11-28；查核於 2026-07-31。
- [S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)、[S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)、[S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-how-to-event-types-and-destinations.html)、[S3 logging](https://docs.aws.amazon.com/AmazonS3/latest/userguide/logging-with-S3.html)：查核於 2026-07-31。
- [SSE-KMS for S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html)：同區 key、Bucket Key、權限和 encryption-context 注意事項；查核於 2026-07-31。
- [Amazon S3 `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/ap-east-1/index.json)：publication date `2026-07-28T13:10:00Z`。
- [AWS Data Transfer `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDataTransfer/current/ap-east-1/index.json)：publication date `2026-07-20T18:46:45Z`。
- [AWS KMS `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/awskms/current/ap-east-1/index.json)：publication date `2025-08-28T15:39:13Z`，於 2026-07-31 仍為 current。
