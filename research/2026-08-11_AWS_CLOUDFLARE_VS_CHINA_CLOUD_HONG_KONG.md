# AWS/Cloudflare 與中國雲香港區比較研究

| 屬性 | 內容 |
| --- | --- |
| 日期 | 2026-08-11（Asia/Hong_Kong） |
| 問題 | Release 1 應沿用 AWS 香港 + Cloudflare，還是改用阿里雲／騰訊雲香港區 |
| 新情境 | 未來讓身處中國內地的申請人查詢申請狀態並上傳文件 |
| 研究邊界 | 官方／第一方來源；未取得供應商正式報價，未執行中國內地實測，不構成法律意見或採購／部署授權 |

## 1. 結論

**Release 1 保留已批准的 AWS 香港敏感核心 + Cloudflare 公開資料平面。不要只因未來可能有內地申請人 portal 就立即遷移到阿里雲或騰訊雲。**

原因不是 AWS 在所有維度都更好，而是：

1. 三份現行文件已把 RDS、Cognito、ECS、S3/KMS/SQS、審計、備份和部署證據連成一套香港區 contract。換雲會同時重開身份、授權、文件、事件、可觀測性、restore 和 cutover 決策，而非只換 compute。
2. AWS 香港有三個可用區；阿里雲的香港 ALB 文件列出 B/C/D 區；騰訊雲的 CVM API 列出香港 1/2/3 區，但目前標示 1 區已售罄。三家都有足以運行 ERP 的香港核心產品，但不應把區域地圖等同於當下可購容量。沒有官方證據證明未購買加速時，阿里雲／騰訊雲香港 origin 對所有內地網絡必然優於 AWS 香港。
3. 阿里雲／騰訊雲的真正潛在優勢是中國優化網絡產品，例如 Alibaba Global Accelerator／ESA、Tencent Global Accelerator／EdgeOne。這些是獨立、付費且具合規條件的 delivery capability，不是「選香港 Region」自然獲得的效果。
4. 內地 portal 是 Release 1 之外的新產品和法律邊界。現行 `EndClient portal` 只批准為後續 read-only interface；文件上傳、申請人身份、監護人授權、帳戶恢復和內地個資出境均尚未決策。

因此採兩階段策略：

- **現在：** 保留 AWS 香港作唯一敏感 system of record；Cloudflare 仍只承載經批准的公開學校 release，不進入 authenticated ERP 路徑。
- **Portal discovery：** 以相同 workload 對 AWS 香港、Alibaba 香港 + 加速、Tencent 香港 + 加速做 14 日 carrier-diverse PoC。只有測試、正式報價、資料流和法律 gate 都顯示全棧遷移具有實質優勢，才重開主雲決策。

## 2. 現行文件的硬約束

現行架構並非一般的「AWS + Cloudflare」：

- `app.<domain>` 直接到 AWS 香港 ALB/ECS；Cloudflare 只作 DNS-only，不得接收 cookie、authenticated request、PII、private response 或敏感 log。[TDR-001](../TECHNICAL_DECISION_PRODUCTION_PLATFORM.md#22-public-plane)
- `data.<domain>` 才以 Cloudflare R2 custom domain 分發匿名、經 allowlist/PII/licence/hash/read-back gate 的公開學校資料。[TD-002](../TECHNICAL_DESIGN_AWS_CLOUDFLARE_PRODUCTION_DEPLOYMENT.md#21-hostnames)
- 敏感 runtime、身份/session、RDS、文件、queue、audit、log、scan/OCR 暫存和 backup 必須留在香港；沒有香港 endpoint 的 workload service 不得靜默改用區外服務。[DEC-018/063](../PRD_IMPLEMENTATION_DECISIONS.md#8-文件域與香港資料駐留)
- Portal 不在 Release 1；現行後續 `EndClient portal` 是 read-only，且 Student/Guardian 不因存在 CRM 中自然取得帳號。[DEC-051/054/055](../PRD_IMPLEMENTATION_DECISIONS.md#17-多客戶多租戶與產品擴展決策)

如果改用阿里雲或騰訊雲，至少需以新決策修訂或取代 `DEC-019`、`DEC-020`、`DEC-021`、`DEC-024`、`DEC-037`、`DEC-038` 和 `DEC-063`，並重寫 TD-002 的 account、IaC、identity、migration、restore 和 rollback evidence。

## 3. 平台比較

| 判準 | AWS 香港 + Cloudflare | 阿里雲香港 | 騰訊雲香港 |
| --- | --- | --- | --- |
| 香港核心服務 | RDS PostgreSQL Multi-AZ、ECS/Fargate、S3、KMS、SQS、Cognito、CloudWatch/Trail 等足夠且現行設計已採用 | RDS PostgreSQL HA、ECS/ACK/Function Compute、OSS、RAM、監控、安全產品足夠 | TencentDB PostgreSQL、CVM/TKE/SCF、COS、CAM、CLS/WAF 等足夠 |
| 現行實施成本 | 最低；可沿用已批准 contracts、Terraform 方向和測試證據 | 高；需重寫 AWS-specific IaC、identity、event、KMS、audit、restore | 高；同樣需重寫完整控制平面和證據 |
| 內地普通公網 | 香港 proximity 不等於可靠；AWS 明示 China Regions 與 global/HK 分離 | 普通香港 endpoint 仍受跨境網絡波動影響 | 普通香港 endpoint 仍受跨境網絡波動影響 |
| 內地優化能力 | Cloudflare China Network + Global Acceleration 官方明示可改善 dynamic API，但需 Enterprise、China Network、ICP、當地合作方及合約 | GA cross-border、BGP Pro、ESA Access Optimization (Chinese Mainland Network)；OSS transfer acceleration 可改善 upload | GA/dedicated BGP、EdgeOne Mainland China Network Optimization；可代理 dynamic traffic/upload |
| 文件上傳 | 現行 direct presigned S3 HK 需另測內地路徑；若前置加速，不能讓 presigned URL 繞過加速 hostname | OSS 支援 multipart/resumable + transfer acceleration，但仍需實測及重做 document adapter | COS 支援 multipart/resumable、S3 V4 compatibility；EdgeOne 可代理 upload，但仍需實測、確認 body limit 及重做 adapter |
| 香港資料邊界 | Application payload 可放香港；IAM/global metadata、support/telemetry 仍要逐項盤點 | RAM control-plane/identity policy data官方文件指出位於新加坡；部分 HK Security Center log 已遷新加坡，是現行「全部敏感 log 香港」的明顯風險 | 官方 DPSA 以 selected region 為基線，但 subprocessors/support/產品差異仍需逐項確認 |
| DR | RDS local Multi-AZ/backup/PITR成熟；現行政策禁止自動區外 failover | PostgreSQL cross-region backup/geo-DR文件較清楚，但若跨出香港仍違反現行邊界 | local backup/PITR已有證據；PostgreSQL cross-region DR需供應商書面確認 |
| 帳戶／ICP | Cloudflare China Network需 ICP；AWS China另屬獨立營運體系 | China/International account、billing、support、ICP workflow分離。Alibaba CDN 的內地／Global 節點與 GA 內地 HTTP(S) acceleration region 需 ICP；ESA 的這個特定優化功能則以「Global (Excluding the Chinese Mainland)」及香港 POP 運作 | International/China account及mainland product資格分離。EdgeOne 的 Mainland/Global service region 需身份驗證及 ICP；經香港入口的 Mainland China Network Optimization 不應被描述為使用內地節點 |
| 成本可比性 | 現有 `$300–650/month` pilot envelope 不含 Cloudflare China Enterprise/acceleration | 香港 egress、ESA/GA、support需 matched quote | GA/EdgeOne/dedicated BGP另計；公開單價不可直接與 raw egress比較 |
| Vendor lock-in | Cognito/IAM/KMS/SQS/CloudWatch lock-in已存在，但 PostgreSQL、OCI container和module adapters可控制 | RAM/GA/CEN/OSS events/FC/monitoring/DR均形成新 lock-in | CAM/GA/CCN/EdgeOne/COS events/SCF/DR均形成新 lock-in |

## 4. 內地申請人 portal 的架構含義

這個 portal 同時有三條不同性質的流量：

1. 公開 shell/static assets：可由 CDN/edge cache 加速。
2. 登入、狀態和個人化 API：不可公開 cache，需要可靠的動態跨境 path、fresh authorization 和 private/no-store response。
3. 文件上傳：對 packet loss、timeout 和路徑切換最敏感，需要 multipart/resumable upload、checksum、idempotent completion、quarantine/scan 和明確 retry state。

上傳能力不能只看產品名稱。Tencent EdgeOne 官方文件顯示 request body 預設限制為 32 MB，符合資格的方案可調至 1–800 MB 或停用限制；因此 200 MB 測試前必須確認方案和配置。三條候選加速路徑亦都要驗證實際 upload hostname、presigned URL、redirect 和 multipart part 是否持續經過預期網絡，不能以首頁／下載加速結果推定上傳亦獲加速。

不建議先做「中國雲 portal + AWS 核心」雙 control-plane。它會引入跨雲身份/session、授權 freshness、文件權威、audit transaction、queue replay 和局部故障一致性問題。初始 PoC 應保持一個香港 system of record，只更換可比較的 ingress/acceleration path。

可測的候選：

```text
A. Mainland user -> ordinary Internet -> AWS Hong Kong core
B. Mainland user -> Cloudflare China/Global Acceleration -> AWS Hong Kong core
C. Mainland user -> Alibaba ESA/GA -> Alibaba Hong Kong reference core
D. Mainland user -> Tencent EdgeOne/GA -> Tencent Hong Kong reference core
```

候選 B 會讓 authenticated request 和申請人資料進入 Cloudflare/China Network 路徑，與現行「Cloudflare 只處理公開資料」的批准邊界不同。它只能在隔離的合成資料 PoC 中測試；任何真實申請人流量上線前，必須重新批准 processor/subprocessor、log、WAF/content scanning、保留和資料出境路徑。

若未取得 ICP／適格內地主體，另測只在香港 POP 進站、不使用內地節點的產品，但不得把它描述為內地可用性保證。

## 5. 合規 gate

- 香港 PDPO 不因選擇某家雲而解除資料使用者責任；必須以合約和技術措施控制 processor/subprocessor 的保留、安全、存取、刪除和 breach handling。[PCPD Cloud Computing Guidance](https://www.pcpd.org.hk/english/resources_centre/publications/files/IL_cloud_e.pdf)
- 若 portal 以向中國內地自然人提供服務為目的，PIPL 可能有域外適用；不滿 14 歲未成年人資料屬敏感個人信息。向香港傳送申請資料屬個人信息出境，需要依情境處理告知、單獨同意、個人信息保護影響評估、境內代表（如適用）及適用的標準合同／認證／安全評估機制。[PIPL](https://www.cac.gov.cn/2021-08/20/c_1631050028355286.htm)；[跨境流動規定](https://www.cac.gov.cn/2024-03/22/c_1712776611775634.htm)
- 現行 CAC 門檻需納入容量設計：非關鍵信息基礎設施運營者年內累計出境少於 10 萬人非敏感個人信息，可豁免申報安全評估、訂立標準合同或通過認證，但不豁免告知、單獨同意、影響評估和安全義務；10 萬至少於 100 萬人非敏感資料，或少於 1 萬人敏感資料，一般需標準合同或認證；達 100 萬人非敏感資料、1 萬人敏感資料、重要數據或 CIIO 情形則觸發安全評估，均受條文豁免所限。「為訂立、履行個人作為一方當事人的合同而確需出境」也可豁免上述申報／合同／認證機制，但不應未經中國法律意見就假定學校申請 portal 必然符合此豁免。
- 使用內地 server/CDN/edge node 一般需要 ICP；香港 origin 本身通常不需要 ICP，但這不等於可取得穩定內地體驗。Cloudflare China Network、Alibaba CDN 內地／Global 節點、Alibaba GA 內地 HTTP(S) acceleration region 和 Tencent EdgeOne Mainland/Global service region 均有 ICP／帳戶／內容條件。反之，Alibaba ESA Access Optimization 的這個文件化模式以香港 POP 和 Global (Excluding the Chinese Mainland) 運作；Tencent EdgeOne Mainland China Network Optimization 也以香港為入口。它們不是內地節點服務，亦不提供內地可用性保證。[Cloudflare China](https://developers.cloudflare.com/china-network/)；[Alibaba ICP](https://www.alibabacloud.com/help/en/icp-filing/basic-icp-service/user-guide/icp-filing-server-access-information-check)；[Alibaba GA](https://www.alibabacloud.com/help/en/ga/user-guide/create-and-manage-standard-ga-instances)；[Alibaba ESA](https://www.alibabacloud.com/help/en/edge-security-acceleration/esa/use-cases/accelerate-your-website-for-chinese-mainland-users)；[Tencent EdgeOne](https://www-sg.tencentcloud.com/document/product/1145/54208?has_map=1)

以上需要合資格的中國內地及香港法律／私隱顧問確認；本研究不是法律意見。

## 6. PoC 與決策門檻

以相同 build、API payload、文件尺寸和安全設定進行 14 日測試：

- 地點：廣州、深圳、上海、北京及一個低線城市；中國移動、聯通、電信，另納入實際學校／教育網絡（如可合法取得）。
- API：DNS、TCP、TLS、登入、authenticated status API 的 p50/p95/p99、5xx/timeout、session 成功率。
- Upload：10/50/200 MB 完成率、有效 throughput、重試次數、斷線續傳、checksum、一致性和 quarantine completion。
- 時段：工作日／週末、日間／晚間尖峰；記錄 route、provider product/config version 和原始 redacted evidence。
- 成本：同一 12 個月 workload 的 compute、DB HA、LB/WAF、object、NAT/endpoint、egress、logs、backup、support、China acceleration、稅和最低合約額。

只有下列條件同時成立才重開全棧遷移：

1. 內地 portal 已成為已批准、近期且有明確規模的產品需求，而非遠期可能性。
2. AWS/Cloudflare 路徑未達已批准 SLO，且調整後仍失敗。
3. Alibaba 或 Tencent 在相同測試下有顯著、持續的成功率／P95/P99優勢。
4. 供應商書面證明所有 PII、文件、authz、audit、logs、backup、support trace 的香港資料流符合批准邊界，或業務明確批准修訂該邊界。
5. matched quote 的三年 TCO 加上遷移、培訓、雙跑和退出成本仍優於保留 AWS。
6. 身份遷移、document/version、queue/replay、restore 和 rollback rehearsal 全部通過。

## 7. 官方來源

- [AWS regional services](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/)
- [AWS Regions and Availability Zones](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions.html)
- [AWS China Gateway](https://aws.amazon.com/china-gateway/)
- [AWS RDS PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html)
- [Cloudflare China Network](https://developers.cloudflare.com/china-network/)
- [Cloudflare Global Acceleration](https://developers.cloudflare.com/china-network/concepts/global-acceleration/)
- [Cloudflare China available products](https://developers.cloudflare.com/china-network/reference/available-products/)
- [Alibaba Cloud Global Accelerator](https://www.alibabacloud.com/help/en/ga/getting-started/get-started)
- [Alibaba GA FAQ / ICP condition](https://www.alibabacloud.com/help/en/ga/support/faq-about-global-accelerator)
- [Alibaba ESA China-network optimization](https://www.alibabacloud.com/help/en/edge-security-acceleration/esa/use-cases/accelerate-your-website-for-chinese-mainland-users)
- [Alibaba OSS transfer acceleration](https://www.alibabacloud.com/help/en/oss/user-guide/transfer-acceleration)
- [Alibaba RDS PostgreSQL HA](https://www.alibabacloud.com/help/en/rds/apsaradb-rds-for-postgresql/rds-high-availability-edition)
- [Alibaba ALB supported regions and zones](https://www.alibabacloud.com/help/en/ack/ack-managed-and-ack-dedicated/user-guide/regions-and-zones-supported-by-alb)
- [Alibaba Cloud and Alibaba Cloud China account comparison](https://www.alibabacloud.com/help/en/account/aliyun-vs-alibaba-cloud)
- [Alibaba RAM data location](https://www.alibabacloud.com/help/en/ram/user-guide/security/)
- [Alibaba Security Center Hong Kong log migration notice](https://www.alibabacloud.com/help/en/security-center/product-overview/notice-hong-kong-region-migration)
- [Tencent Global Accelerator pricing/features](https://proxy-hk.tencentcloud.com/document/product/608/70509)
- [Tencent EdgeOne pricing](https://www-sg.tencentcloud.com/document/product/1145/57404)
- [Tencent EdgeOne Mainland China Network Optimization](https://www-sg.tencentcloud.com/document/product/1145/54208?has_map=1)
- [Tencent EdgeOne upload contract](https://www-sg.tencentcloud.com/ind/document/product/1145/50547)
- [Tencent CVM availability zones](https://www.tencentcloud.com/zh/document/api/213/15753)
- [Tencent Cloud account identity and regional account separation](https://proxy-hk.tencentcloud.com/document/product/378/3629)
- [Tencent Cloud Data Processing and Security Agreement](https://intl.cloud.tencent.com/document/product/301/17347?lang=en)
- [Tencent COS Multi-AZ](https://www-sg.tencentcloud.com/document/product/436/35208)
- [Tencent COS release notes / S3 compatibility](https://www-sg.tencentcloud.com/document/product/436/34071)
- [TencentDB PostgreSQL automatic backup](https://www.tencentcloud.com/document/product/409/46146/)
- [TencentDB PostgreSQL point-in-time rollback](https://www.tencentcloud.com/document/product/409/46147?lang=en&pg=)
- [Hong Kong PCPD cloud guidance](https://www.pcpd.org.hk/english/resources_centre/publications/files/IL_cloud_e.pdf)
- [PRC Personal Information Protection Law](https://www.cac.gov.cn/2021-08/20/c_1631050028355286.htm)
- [CAC cross-border data rules](https://www.cac.gov.cn/2024-03/22/c_1712776611775634.htm)

上述外部官方來源的本次查閱日期均為 2026-08-11。來源中的產品可用性、價格和合規條件可能變更，採購前應重新查證。
