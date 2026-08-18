# 天星國際教育平台 PRD 實施決策台帳

| 屬性 | 內容 |
| --- | --- |
| 文件版本 | v0.14 |
| 文件狀態 | 第 18 節 phase topology 已批准，可依 dependency order 本地實作；外部副作用與上線操作仍需逐項批准 |
| 任務 ID | `prd-implementation-design-2026-07-31` |
| 最後更新 | 2026-08-12（Asia/Hong_Kong） |
| 當前目標 | 把 `PRD.md` 轉成可實施、可驗證、可遷移和可回滾的生產級方案 |
| 當前階段 | `P3-02` deterministic matrix 已完成本地 evidence，但因已識別 contract gaps 保持 `needs_human`；使用者於 2026-08-12 批准 `DEC-067` reconstruction implementation contract 與 `DEC-068` AWS source/plan contract。`DP-01`–`DP-12` 與 `DEC-060` 仍為 fail-closed gates。這不等於 Phase 3 release pass，亦不授權 Terraform plan/apply、production/cloud/data action |

## 1. 文件用途與更新規則

本文件保存 PRD 實施訪談中已確認的決策、研究證據、被後續條件修訂的決策，以及下一步仍需業務批准的事項。它是設計過程的持久 checkpoint，避免聊天上下文成為唯一的專案記憶。

狀態定義：

| 狀態 | 含義 |
| --- | --- |
| `accepted` | 使用者已明確批准，可作後續設計前提 |
| `accepted_with_constraint` | 原則已批准，但供應商或實現受後續條件約束，尚不可直接實作 |
| `amended` | 邏輯決策仍成立，但其中的技術載體已被新條件取代 |
| `superseded` | 已被後續決策明確取代，不得再作現行前提 |
| `recommended` | 已完成研究並有建議，仍待使用者批准 |
| `open` | 尚未決策；若會影響資料、授權或生產狀態，不得自行假設 |
| `resolved` | 用於 TODO：其決策問題已解決，實施交付物仍按對應決策驗收 |
| `partially_resolved` | 用於 TODO：部分邊界已批准，列出的剩餘問題仍不得自行假設 |

更新規則：

1. 新決策使用穩定 ID，不重排既有 ID。
2. 不刪除被取代決策；標示取代原因和新決策 ID。
3. 外部事實記錄來源、查核日期和不確定性。
4. 建議與批准分開；研究結論不等於採購或部署授權。
5. 在 P0 阻斷決策完成前，不開始生產 schema、認證或雲端資源實作。

## 2. 權威來源與現況基線

### 2.1 來源優先級

1. 根目錄 `agents.md`／`AGENTS.md`、子專案規則和可驗證的現行程式。
2. `PRD.md` 中已批准的業務規則與不變量。
3. 本文件中的 `accepted` 決策。
4. 已批准的功能 requirements、design、ADR 和 tasks。
5. 歷史 PRD、展示資料和研究摘要。

主要輸入：

- `PRD.md` v2.0，2026-07-31。
- `MDN_Web_開發課程評讀與程式設計洞見.md`，尤其是 requirements → design → tasks、harness/loop/graph、資料耐久、權限、可及性和七道階段門。
- `agents.md`／`AGENTS.md` 的資料主權、權限、驗證和人工批准規則。
- `erp-frontend/AGENTS.md`、`package.json` 和現行 Git source。
- `automated_tracker_for_school_listing/hk-school-platform/docs/AGENTS.md` 與 crawler production workflow。

### 2.2 已核實的現況

| 邊界 | 現況事實 | 實施含義 |
| --- | --- | --- |
| 前端 | Next.js 16.2.7、React 19.2.4、Vercel | 實作前需讀相應 Next.js 16 本地文件 |
| 可變資料 | 目前使用 Neon PostgreSQL | Neon 不提供香港 region；在全量香港駐留要求下不能作生產資料庫 |
| DB schema | `modules/schools/infrastructure/crawler/db.ts` 在 runtime 使用 `CREATE TABLE IF NOT EXISTS` | 不具備版本化 migration、rollback、dry-run 或 schema drift 控制，不能沿用為核心 CRM migration 策略 |
| CRM | 學生資料來自 `modules/crm/infrastructure/mock-students.ts` | 現有 16 欄是展示資料，不是正式持久化契約 |
| 認證 | `package.json` 未見正式 auth 套件 | 帳號生命週期、MFA、session revoke 和服務端授權仍待交付 |
| 測試／migration | 未發現正式 frontend test、spec 或 migration 基線 | Phase 1 前需建立最小可重複驗證和 migration harness |
| 學校資料 | 爬蟲發布四檔 immutable snapshot，前端 Git snapshot 讀取 | 基礎快照與可變人工修正必須維持不同所有權邊界 |
| 爬蟲可變狀態 | tickets、review decisions、config、runs 現存 Neon 邊界 | 香港駐留後需遷移到香港資料庫，不得遺留雙寫或區外副本 |

現行 crawler-to-frontend 流程：

```text
Crawler published/latest
  -> erp-frontend/data/crawler-source/latest
  -> Next.js /api/crawler/**
  -> frontend pages
```

## 3. 已批准的 Release 1 邊界

### DEC-001：首個正式版本是內部 K12 營運核心

- 狀態：`amended`
- 決策：Release 1 先服務天星內部單一公司；先以版本化 synthetic golden scenarios 完成 deterministic gate，再交付 business-data-empty production tenant，由客戶逐案重建 1–3 個既有案件並由不同 Founder reviewer 核准；至少觀察五個營業日後，才可由人工 gate 擴至總計 5–10 個受監控 active pilot cases。`[DEC-057, DEC-061, DEC-062]`
- 包含：身份與帳號生命週期、RBAC/案件授權、審計、Student/Guardian/ServiceCase/SchoolTarget/Task、受控狀態、看板、文件、站內通知、受限單一案件只讀 Portal，以及 aggregate-only PlatformBilling／合同版本／月度收費通知草稿。`[DEC-064-066]`
- 不包含：Portal 寫入／公開註冊／文件下載、正式稅務發票、付款、退款、會計總帳、收入確認、外部 Email／SMS／WhatsApp 自動通知、自動快照同步、Excel 匯入和 AI 對客報告。`DP-01`–`DP-12` 未批准部分一律 fail closed。
- 驗收最小證據：跨帳號負向授權、併發更新、migration/restore/rollback 演練，以及至少一個真實委派閉環。

### DEC-002：合作渠道暫不登入

- 狀態：`accepted`
- 決策：銀行、保險等合作渠道在 Release 1 只建模為 `ReferralSource`，不是 `User`，不取得案件讀取權。
- 不變量：未來擴展前保留 `organization_id`；首版只允許天星一個 organization。

### DEC-003：只啟用 K12，但保留其他申請類型架構

- 狀態：`accepted`
- 決策：顧問建案時使用 `application_type`；Release 1 只啟用 K12 正式流程。大學、碩士等類型保留 registry/schema 能力和前台佔位選項，清楚標示「正在開發中」，不可建立詳細流程或正式案件。
- K12 業務範圍：香港境內現有客戶需要的本地、國際、小學、中學等 K12 路徑都可正式建案；不以現有 crawler 覆蓋範圍限制 CRM。
- 被取代的假設：Release 1 只做香港中學 S1/S2–S5 的提議為 `superseded`。

## 4. 核心領域與身份決策

### DEC-004：Student 與 ServiceCase 分離

- 狀態：`accepted`
- `Student` 表示自然人；`ServiceCase` 表示一次具體服務委託。
- 同一 Student 可有多個 ServiceCase。
- S1 入學與 S2–S5 插班不得合併為同一案件。
- 同一 `student_id + intake_year + admission_type` 最多一個未結束案件；由資料庫 partial unique constraint 強制。
- 已取消或結案後重新簽約，建立新案件，不覆寫或重開舊案件。
- 每個案件使用不可變 UUID 和人類可讀 `case_number`。

### DEC-005：Guardian 是獨立實體

- 狀態：`accepted`
- `Guardian` 與 `Student` 為多對多，透過 `StudentGuardianRelationship` 保存關係、法定監護狀態、主要／緊急／帳單聯絡用途、通知同意和有效期。
- 同一 Guardian 可關聯多名兄弟姊妹。
- 每名 Student 至少一名有效主要聯絡人；同一時間最多一名主要聯絡人，交接保留歷史。
- Guardian 身份使用 UUID；email、電話不設全局唯一約束，可共用，只提示疑似重複。
- 不得根據姓名、email 或電話自動合併。人工合併需創始人批准並保留舊 ID 對應和審計。

### DEC-006：Release 1 不保存法定證件號碼和影像

- 狀態：`accepted`
- 不保存 HKID、內地身份證、護照號碼或證件影像。
- 只記錄證件類型、是否已提供、到期日和材料狀態。
- 若後續必須保存，需建立獨立加密文件域、遮罩、下載授權、存取審計、保留和刪除流程，不可直接加到 Student/Profile。

## 5. 帳號、案件所有權與授權

### DEC-007：邀請制、MFA、身份與業務授權分離

- 狀態：`accepted_with_constraint`
- 已批准原則：不公開註冊、不由本系統保存密碼、所有內部角色強制 MFA；身份服務只證明 actor 身份，香港資料庫負責業務授權。
- 內部 `User UUID` 是穩定身份；另存 `provider + provider_subject`，email 不作業務外鍵。
- 停用帳號需同時停用內部 User、撤銷 provider session 並遞增 `session_version`。
- 後續約束：`DEC-020` 已選定受限使用 Cognito `ap-east-1`；所有身份和會話資料仍須駐留香港，且正式建置前必須完成 DPA、刪除、匯出、審計和郵件依賴 gate。

### DEC-008：Primary Advisor + Case Collaborator

- 狀態：`accepted`
- 每個活躍案件必須且只能有一名 `primary_advisor`；創始人可暫代。
- 案件轉交是原子操作，記錄轉出人、轉入人、原因、時間和版本。
- `CaseCollaborator` 是正式 Advisor 對特定案件的限權成員；Founder/Admin 不需加入，Part-time/Contractor 不可成為 Collaborator。
- 兼職只經 `TaskAssignment` 取得單一任務所需的脫敏資料。
- 被取代的假設：「只有單一主責、沒有 Collaborator」為 `superseded`。

### DEC-009：Collaborator 使用預定義 scope

- 狀態：`accepted`
- 不允許任意逐欄 ACL；scope 版本化、新資料類別預設不可見。
- scopes：`case_summary`、`education_profile`、`school_targets`、`task_workspace`、`communications`、`identity_contact`、`internal_notes`。
- `identity_contact` 和 `internal_notes` 預設不授權。

### DEC-010：資料 scope 與 capability 分離

- 狀態：`accepted`
- 每個 scope 可授予 `view`、`comment` 或 `edit`。
- 即使有 `edit`，也不可下放：更換 Primary Advisor、邀請 Collaborator、案件推進/回退/取消/結案、最終選校和對客內容批准、敏感匯出、發布、刪除正式歷史／審計／證據。
- 寫入需保存變更前後值並使用版本號；衝突回應 `409`，不可 last-write-wins。

### DEC-011：授權 owner、期限和敏感批准（已由 DEC-029 修訂期限）

- 狀態：`amended`
- 普通 scope 由 Primary Advisor 授予；原「預設 30 天」已被 `DEC-029` 的 7 天預設期限取代。
- `identity_contact`、`internal_notes` 由 Primary Advisor 申請、Founder 批准，每次最長 7 天且必須填理由。
- 到期、帳號停用、移除 Collaborator 或案件結案時立即失效；續期是新的決定，不可靜默延長。
- 敏感存取記錄 actor、case、scope、時間和 request ID；批量或異常存取需告警。

## 6. Assessment schema 與完整度

### DEC-012：評估表採可組合、版本化 schema

- 狀態：`accepted`
- 邏輯模型：

```text
Student
  -> ServiceCase(application_type)
      -> Assessment(schema_manifest, answers, status)

K12 base schema
  + education stage module
  + school system module
  + admission route module
```

- 初始分類方向：教育階段（幼稚園／小學／中學）、學校體系（香港本地／香港國際）、入學路徑（起始年級／插班）、目標年級和入學學年。
- 建案時解析並固定 `assessment_schema_manifest` 及各 module version；schema 升級不可靜默改寫舊答案。
- 現有 demo 16 欄不可直接作正式契約；可重用的概念需映射至新 schema，不為 mock data 建 migration。

### DEC-013：必填要求按案件階段收緊

- 狀態：`accepted`
- Draft case 只要求最小身份、申請分類、目標入學年份和 Primary Advisor。
- 「背景收集完成」和「選校確認」各有明確 blocking fields；單純百分比不能替代。
- AI 未來除加權完整度至少 80%，還需所有安全關鍵 blocking fields 完成。
- 欄位狀態需區分 `unknown`、`not_applicable`、`declined_to_provide`，不可用空字串或「暫無」混淆語義。
- UI 和服務端必須使用同一 schema 規則；服務端為強制 owner。

## 7. 學校資料、人工修正與回滾

### DEC-014：允許 provisional school

- 狀態：`accepted`
- 香港學校缺少於現有資料庫時，Advisor 可建立 `provisional School`，填最小身份、地區、體系、階段和原因。
- 狀態：`provisional -> under_review -> verified -> retired`。
- provisional school 可關聯案件和資料搜集任務，但需清楚標示未驗證；不可進 AI 受控候選或對客呈現為已核實事實。
- 官方 URL 不可推測；需被發現、驗證和保存 evidence。

### DEC-015：所有 School 欄位都可提出變更建議

- 狀態：`accepted`
- `SchoolChangeRequest` 保存欄位、現值、建議值、原因、官方 URL、引文／附件、requester 和基礎 `school_version`。
- 一般欄位由 Data Reviewer 批准；學校身份、合併、拆分、停用和官方網站主身份變更需 Founder 批准。
- 提交者不可審核自己；批准和拒絕都需理由。

### DEC-016：immutable snapshot + approved overlay

- 狀態：`amended`
- 邏輯決策仍有效：crawler 四檔 snapshot 不手改；批准的人工 field override 在可變資料庫中立即生效，Resolved School View 由 `base_snapshot_id + overlay_revision + field provenance` 組成。
- 新 crawler snapshot 和 override 相同時可關閉 override；衝突時保持已批准人工值並建立重新審核項。
- 報告和已批准選校方案固定引用當時 resolved version，不隨新 snapshot 靜默改變。
- 回滾停用錯誤 revision，恢復上一個有效 revision／base value，不覆寫歷史。
- 修訂：原先指定 Neon 保存 overlay；香港駐留決策後，必須改為 `DEC-019` 已批准的香港 RDS for PostgreSQL。邏輯 contract 不變。

## 8. 文件域與香港資料駐留

### DEC-017：Release 1 包含平台內文件上傳和版本管理

- 狀態：`accepted`
- DB 只存 metadata、授權、關聯、版本和狀態；file bytes 存私有 object storage。
- 無公開 URL；上下載都先做 server-side authn/authz，再簽發短期、單一 object/method 的 URL。
- 文件狀態至少涵蓋 quarantine、scan、available、rejected、superseded、pending delete 和 deleted。
- 替換建立新版本，不覆寫舊版本；回滾只可指回已掃描且未撤銷的版本。
- 預覽、下載、匯出、刪除和恢復都記錄審計。

### DEC-018：所有敏感資料必須駐留香港

- 狀態：`accepted`
- 香港駐留涵蓋：Student、Guardian、ServiceCase、Assessment、SchoolTarget、Task、文件 bytes、文件 metadata、案件授權、identity/session data、審計、應用日誌、掃描輸入/結果、預覽/OCR 暫存和備份。
- 不得以香港 S3 掩蓋香港以外的 DB、auth、runtime、log 或 backup。
- 不允許以新加坡作自動 failover；香港區域故障時需 fail closed／進入明確降級狀態。
- 直接影響：現行 Neon 不符合；所有可能接觸敏感資料的第三方服務需重新評估。

### DEC-019：Release 1 使用香港 RDS for PostgreSQL Multi-AZ

- 狀態：`accepted`
- 決策：Release 1 主資料庫採 AWS 香港 `ap-east-1` 的 `RDS for PostgreSQL Multi-AZ db.t4g.small + 20 GiB gp3`。
- RDS 必須保持 private；敏感 API runtime 應位於香港同一 VPC，或使用經驗證且不造成區外處理／副本的香港 private connectivity。
- 初始核心按需官價估算約 `$87.03/月`，不含 Proxy、備份超額、監控、流量、Support 和稅。
- Aurora 不在 Release 1 初始範圍；只有在量測顯示可用性、failover、讀擴展或連線負載超出 RDS 方案時才重新評估。
- 本決策不等於建立 AWS 資源的授權。backup retention、RPO/RTO、連線 runtime、migration/cutover 和 restore drill 已由後續 `DEC-021`、`DEC-022`、`DEC-031`、`DEC-036`、`DEC-037`、`DEC-038` 補齊決策。

### DEC-020：身份服務使用 Cognito `ap-east-1`（受限功能集）

- 狀態：`accepted`
- 研究：`docs/research/aws-identity-hong-kong-assessment.md`，查核日期 2026-07-31。
- 決策：使用 Amazon Cognito User Pools `ap-east-1` 作身份證明；香港 RDS 的 User、session、`session_version`、角色、案件 scope/capability 和業務授權才是平台真相。
- Release 1 只啟用 TOTP authenticator app；不啟用 SMS MFA、Email MFA、Cognito 自助密碼重設、自動邀請郵件或其他需要 AWS 郵件／短信的流程。
- 邀請使用 `AdminCreateUser` + `MessageAction=SUPPRESS`；一次性 invite secret 只保存 hash，透過尚未選定但須通過香港駐留/DPA 審核的通信流程傳送。
- BFF 在香港驗證 Cognito token，向瀏覽器只發無 PII 的短期 opaque session cookie；每次業務讀寫仍重新查香港 RDS policy。
- Cognito token revoke 是外部可重試副作用；停用帳號必須同時更新 RDS User、遞增 `session_version`、撤銷 provider session，並由 reconciliation 補償部分失敗。
- Cognito 不能匯出 password hash；provider exit 必須包含強制密碼重設或受控 migration，不能宣稱完整身份可攜出。
- 本決策不等於建立 user pool 或採購授權；建立前需完成 AWS DPA/服務條款、郵件依賴、刪除/備份與香港 data-flow smoke test 審核。

## 9. AWS S3 香港區域研究

### 9.1 研究狀態與建議

- 狀態：`accepted`，但尚未構成採購／建置資源授權。
- 研究日期：2026-07-31。
- 詳細報告：`docs/research/aws-s3-hong-kong-assessment.md`。
- 已批准：`S3 Standard ap-east-1 + single-Region customer-managed KMS key + S3 Bucket Key + Versioning + 香港 SQS/掃描/audit bucket`。
- Release 1 不採用：One Zone-IA、Cross-Region Replication、Multi-Region Access Point、CloudFront 私密文件快取、Transfer Acceleration 和自動 Glacier lifecycle；後續啟用需新決策。

### 9.2 穩定性

| 指標 | 官方產品目標／合約 | 解讀 |
| --- | ---: | --- |
| S3 Standard 設計耐久性 | 99.999999999% | 回答永久遺失風險，不代表永不離線 |
| 儲存拓撲 | 至少 3 AZ | 可承受單 AZ 損失；香港區域推出時為 3 AZ |
| 設計可用性 | 99.99% | 產品設計目標，不是每月合約保證 |
| 月度 SLA | 99.9% | 未達標救濟為 service credit，不是資料／業務損失賠償 |

AWS 說明特定 region 的 S3 object 不會離開該 region，除非客戶明確轉移；法律或政府強制命令仍屬例外。此結論不自動涵蓋 account/service metadata，也不構成 PDPO 法律意見。

### 9.3 `ap-east-1` 官方按需牌價

| 項目 | 價格（USD） | 注意 |
| --- | ---: | --- |
| S3 Standard 首 50 TB | $0.025/GB-month | 版本保留按完整 object 計費 |
| PUT/COPY/POST/LIST | $0.005/1,000 requests | multipart 和 scan flow 會增加 request |
| GET 及其他 Tier-2 | $0.004/10,000 requests | 通常遠低於 egress |
| Internet DTO 首 10 TB | $0.12/GB | 每月首 100GB 免費額度由帳戶跨適用服務／region 共用 |
| KMS customer-managed key | $1/key-month | key version/rotation 可能增加費用 |
| KMS symmetric requests | $0.03/10,000 | 每月首 20,000 requests 為跨 region 共用免費額度 |
| GuardDuty S3 scan data | 首 1GB 免費，其後 $0.144/GB | 香港 region 現行 Price List API |
| GuardDuty S3 objects | 首 1,000 免費，其後 $0.345/1,000 | 另有 S3 API 費用 |

### 9.4 成本情境

以下只包含 S3 storage/requests/DTO 和一把 KMS key；不含 GuardDuty、SQS、Lambda、CloudTrail、CloudWatch、Config、OCR、database、application runtime、Support 和稅。

| 情境 | 假設 | S3 + KMS 月估算 |
| --- | --- | ---: |
| Pilot | 30 cases、3GB、1,000 PUT、5,000 GET、5GB DTO | $1.08–$1.70 |
| 100 cases | 25GB、5,000 PUT、30,000 GET、40GB DTO | $1.71–$6.57 |
| 1,000 retained cases | 480GB、40,000 PUT、300,000 GET、300GB DTO | $38.28–$50.34 |

100-case 情境若每月掃描 25GB、5,000 個新 objects，GuardDuty 在免費額度後約再增加 `$4.84/月`。在目前規模，S3 容量不是主要成本；Internet egress、掃描/OCR、錯誤累積的 noncurrent versions 和工程維運才是主要風險。

### 9.5 建議文件資料流

```text
Browser
  -> HK application runtime：authz + create upload intent
  -> direct TLS presigned upload to private S3 ap-east-1
  -> S3 event (at-least-once)
  -> SQS ap-east-1 + DLQ
  -> scanner ap-east-1
  -> HK PostgreSQL document state + audit
  -> short-lived presigned GET after fresh authz
```

事件冪等鍵建議為 `(bucket, key, version_id, scan_policy_version)`；定時 reconciliation 修復遺漏事件和長時間卡住的 quarantine/scanning objects。

### 9.6 官方來源

- [Amazon S3 SLA](https://aws.amazon.com/s3/sla/)
- [S3 data durability](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DataDurability.html)
- [AWS Hong Kong data privacy](https://aws.amazon.com/compliance/hong-kong-data-privacy/)
- [AWS Hong Kong Region `ap-east-1`](https://aws.amazon.com/blogs/aws/now-open-aws-asia-pacific-hong-kong-region/)
- [S3 `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/ap-east-1/index.json)
- [AWS Data Transfer `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDataTransfer/current/ap-east-1/index.json)
- [AWS KMS `ap-east-1` Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/awskms/current/ap-east-1/index.json)
- [Amazon GuardDuty pricing](https://aws.amazon.com/guardduty/pricing/)

## 10. 香港駐留後的已批准敏感系統邊界

本節中的敏感 runtime、身份、DB、文件和 log 落點已由 `DEC-018` 至 `DEC-024` 批准；public non-sensitive frontend hosting 和未列出的第三方服務仍待逐項批准：

```text
Public browser assets
  -> CDN（不得含 PII 或個人化 cache）

Authenticated browser
  -> application runtime in Hong Kong
      -> auth/session service in Hong Kong
      -> PostgreSQL in Hong Kong
      -> S3/KMS/SQS/scanner/audit logs in Hong Kong
      -> external email/identity/monitoring only after residency + DPA gate

Crawler immutable public-school snapshot
  -> base school view
  + approved field overlays in Hong Kong PostgreSQL
  -> resolved school view
  -> SchoolTarget pinned revision
```

必須分開驗證三件事：

1. storage location：bytes 和 database rows 在哪裡。
2. processing location：function、scanner、preview、OCR 在哪裡執行。
3. secondary copies：log、trace、support ticket、backup、analytics 和 provider telemetry 在哪裡。

## 11. 下一步待決策

### P0：會阻斷所有實作的決策

#### DEC-TODO-001：香港主資料庫

- 狀態：`resolved`（決策層），由 `DEC-019`、`DEC-021`、`DEC-022`、`DEC-031`、`DEC-036`、`DEC-037`、`DEC-038` 解決；具體 IaC、migration tool 和 cutover runbook 仍是實施交付物。
- 問題：現行 Neon 不提供香港 region。
- 研究：已完成，見 `docs/research/aws-postgresql-hong-kong-assessment.md`。
- 已批准：`RDS for PostgreSQL Multi-AZ db.t4g.small + 20 GiB gp3`，核心按需官價約 `$87.03/月`；見 `DEC-019`。
- 對比：RDS Multi-AZ SLA 99.95%，典型 failover 60–120 秒；Aurora 跨至少兩 AZ、具 writer+reader 時 SLA 99.99%，同等常駐官價約 `$163–185/月` 起。
- 架構條件：RDS 保持 private；敏感 API runtime 位於香港同 VPC，或採已驗證的香港 private connectivity。RDS Proxy 不能 public。
- 遷移條件：建立版本化 migration，停止 runtime schema auto-create；先香港 staging，再受控搬遷、核對和切換，禁止無 reconciliation 的長期雙寫。
- 已批准：7 日 backup retention、RPO 5 分鐘／RTO 4 小時、AWS 香港敏感 runtime、expand/contract 和單向 backfill + reconciliation。

#### DEC-TODO-002：香港身份服務

- 問題：已接受 managed identity 原則，但 provider 必須讓 email、device、session 和 audit data 駐留香港。
- 研究：已完成，見 `docs/research/aws-identity-hong-kong-assessment.md`。
- 狀態：`resolved`，由 `DEC-020` 解決選型；正式建置仍受 DPA/data-flow/delete/export/smoke-test gate 約束。
- 已批准：Cognito `ap-east-1`，停用會觸發新加坡／東京 SES alternate region 的郵件流程；Keycloak `ap-east-1` 作退出／政策變更 fallback。
- 禁止：自行實作 password hashing、MFA、recovery 和 session protocol。
- 必須產出：region、subprocessor、retention/export/delete、MFA、invite、revoke、break-glass 和 provider exit plan。

#### DEC-TODO-003：香港 application runtime

- 狀態：`resolved`，由 `DEC-021` 解決敏感 runtime；public non-sensitive frontend hosting 仍 open，但不影響敏感 runtime 的已批准邊界。
- 事實：Vercel Functions 支援香港 `hkg1`，但新專案預設 `iad1`（美國）。
- 已批准：敏感 API/BFF/worker 直接部署 AWS `ap-east-1`；Vercel 僅可作經驗證的非敏感 frontend hosting 或完全停用。
- 若保留 Vercel，仍需驗證 build、preview、log、error payload、support access、analytics 和 failover 不接觸 PII。

#### DEC-TODO-004：香港觀測與通知邊界

- 狀態：`partially_resolved`，由 `DEC-023`、`DEC-032`、`DEC-033`、`DEC-039` 鎖定資料、通知、保留和副作用邊界；精確 metric/alert catalogue 仍 open。
- 已批准：log/metric/trace 欄位白名單、30 天一般 log、1 年 audit、PII redaction 和香港落點；Release 1 只做站內通知。
- 已批准：通知預設只含「有待辦事項」，不含 Student/Case 細節；未批准任何外部 Email provider。
- 已批准：所有外部副作用需 outbox、idempotency key、有界重試和 delivery receipt。

#### DEC-TODO-005：RPO、RTO 和區域故障政策

- 狀態：`partially_resolved`。
- 已批准：DB RPO 5 分鐘／RTO 4 小時、RDS backup 7 天、每月 staging restore drill、每季 DB + 文件完整演練。
- 仍需業務批准：文件 RPO/RTO，以及香港 region 長時間故障的可接受停機。
- 限制：不能以區外 replica 達成 RTO 後仍聲稱全量香港駐留。

#### DEC-TODO-006：正式批准 S3 選型與文件保留

- 狀態：`partially_resolved`。
- 已批准：`DEC-024` S3 選型、`DEC-030` 30 天 soft-delete/legal hold/restore 邊界、`DEC-045` 刪除流程和 `DEC-039` audit 保留。
- 仍需決定：文件分類與結案後總保留年限。

### P1：領域與工作流決策

1. `partially_resolved`：`DEC-026`/`DEC-043` 已鎖定 K12 module/欄位 contract；正式欄位、資料類型、枚舉和可見性清單仍 open。
2. `partially_resolved`：`DEC-027` 已鎖定案件/逐校狀態分離；中退、暫停、多校 join、取消、重簽和結案 guard 仍 open。
3. `partially_resolved`：`DEC-027` 已鎖定共用狀態機 + route-specific template/evidence；精確 DDL、材料和合法轉移仍 open。
4. `partially_resolved`：`DEC-028` 已鎖定 Task 核心狀態；精確 actor matrix 和 transition guards 是 Phase 1 contract 交付物。
5. `partially_resolved`：`DEC-029` 已鎖定期限/revoke；Primary Advisor 缺席、break-glass 和批量授權仍 open。
6. `partially_resolved`：`DEC-042` 已鎖定候選、人工 merge、alias 和 corrective undo；欄位 reducer 需在實施規格中定義。

### P2：交付與營運決策

1. `partially_resolved`：`DEC-031`/`DEC-036` 已鎖定順序和 data flow；migration/feature-flag 工具選型仍 open。
2. `partially_resolved`：`DEC-032`/`DEC-044` 已鎖定 error、concurrency、transaction/outbox/audit 原則；具體 schema 待實施規格。
3. `partially_resolved`：`DEC-034` 已鎖定驗證門檻；CI 工具與 project commands 待專案規格和明確執行授權。
4. `partially_resolved`：desktop/mobile smoke 已批准；鍵盤、焦點、讀屏、慢網路、空／錯誤／權限不足 matrix 仍需列成 acceptance cases。
5. `partially_resolved`：`DEC-035` 已鎖定 named owner/營業時間值守；精確 SLO、alerts、成本預算和 stopping conditions 仍 open。

### P3：擴展到第二個訂閱組織前的決策

1. `partially_resolved`：`DEC-051` 至 `DEC-059` 已鎖定正式詞彙、資料分級、租戶隔離、三個產品平面、角色頁面、模組 interface、驗收門檻和知識庫治理原則。
2. `open`：`DEC-060` 列出的訂閱狀態語義、平台支援存取時限、案件／文件保留期、文件 RPO/RTO、跨租戶知識授權和租戶退出流程仍需逐項批准。
3. 第二個 `CustomerOrganization` 建立前，必須完成 tenant-aware schema、RLS、跨租戶負向測試、S3／cache／search／job 隔離、support access threat model、遷移與回滾演練；Release 1 單公司 pilot 不自動構成多租戶上線批准。

## 12. 當前批准與禁止操作

目前允許：讀取倉庫、查閱官方文件、更新本決策台帳和研究文件，以及依第 18 節 dependency order 修改和驗證本地 source、test、contract、migration plan 與 IaC plan 檔案。

目前不授權：

- 建立 AWS/Vercel/身份服務資源或產生費用。
- 寫入或遷移 Neon／生產資料。
- 執行 frontend lint/build、full crawler、LLM batch、snapshot sync、commit、push 或 deploy。
- 接受 crawler warning、刪除 output/cache 或建立區外副本。

## 13. 最近批准記錄

使用者於 2026-08-01 批准第 18 節原始 phase topology；使用者於 2026-08-11 批准 `DEC-061`、`DEC-062` 並以修訂方案取代其中的 Phase 3/4 rollout：`Phase 0 contracts/harness -> Phase 1 vertical slice -> Phase 2 K12 breadth -> Phase 3 synthetic gate + empty production tenant -> Phase 4 1–3 reconstructed cases + five-business-day observation + human-gated 5–10 monitored pilot`。

使用者同日另行批准把受限 Portal 與 PlatformBilling 納入 Release 1（`DEC-064`），並批准 Portal redeem 的 function-only database capability（`DEC-065`）及獨立 Platform Control audit aggregate（`DEC-066`）。這些批准只授權 authority synchronization 與依 dependency order 的本地 source/test/contract/migration/IaC plan；不批准 `DP-01`–`DP-12` 未決語義，也不取代任何 cloud、資料、migration execution、commit、push、deploy、notice delivery 或 production gate。

使用者於 2026-08-12 批准 Historical Case Reconstruction 的完整 implementation contract（`DEC-067`），以及 production AWS source/reviewed-plan baseline（`DEC-068`）。批准只允許依既有 dependency order 建立本地 reconstruction source/test/additive migration artifact 與 Terraform source/static/redacted plan-manifest contract；不授權執行 Terraform plan/apply、建立資源、執行 migration、寫入資料、commit、push、deploy，亦不改變 `P3-02` 的 `needs_human` 狀態、`DP-01`–`DP-12` 或 `DEC-060`。

## 14. 香港 PostgreSQL 研究摘要

### 14.1 RDS 與 Aurora

| 項目 | RDS PostgreSQL Multi-AZ | Aurora PostgreSQL Multi-AZ |
| --- | --- | --- |
| 建議最低 topology | primary + synchronous standby | writer + 不同 AZ reader |
| 月 SLA | 99.95% | 99.99% |
| 典型核心月費 | `t4g.small + 20 GiB gp3` 約 $87.03 | Serverless v2 約 $163.24；provisioned 約 $185.14 |
| Failover | 典型 60–120 秒 | 有 reader 時通常較快 |
| 讀擴展 | standby 不讀 | 支援多 readers |
| PostgreSQL 可攜性 | 較高 | 較多 Aurora 專屬能力和鎖定 |
| 當前 workload 適配 | 足夠且成本可預測 | 能力超出 100-case 需求 |

### 14.2 官價與預算邊界

- RDS `db.t4g.small` Multi-AZ：`$0.111/hour`，730 小時約 `$81.03/月`。
- Multi-AZ gp3：`$0.30/GiB-month`，20 GiB 約 `$6/月`。
- RDS Proxy：`$0.022/vCPU-hour`；2 vCPU 約 `$32.12/月`，且只能 private VPC 存取。
- Aurora Serverless v2：`$0.22/ACU-hour`；Multi-AZ writer+reader 各 0.5 ACU 常駐即約 `$160.60/月` compute。
- Aurora provisioned `db.t4g.medium`：`$0.125/hour/instance`；writer+reader 約 `$182.50/月` compute。
- 價格來源：AWS `AmazonRDS current/ap-east-1` Price List，publication date `2026-07-29T23:42:48Z`。

### 14.3 回復與 rollback 邊界

- RDS transaction logs 約每 5 分鐘上傳；PITR restore 建立新 instance，不是原地回滾。
- Automated backup 可設 0–35 天，但 production 不可設 0；`DEC-037` 已批准 7 天 baseline。
- Active DB storage 100% 以內的 backup storage 無額外費用；超出部分香港官價 `$0.095/GiB-month`。
- DB failover、PITR、schema rollback 和外部副作用 compensation 是四個不同流程，不能互相替代。
- 所有 DB、standby、snapshot、backup、KMS、logs 和 restore drill 必須留在 `ap-east-1`。

完整研究與官方來源：`docs/research/aws-postgresql-hong-kong-assessment.md`。

## 15. 已批准的香港基礎設施與恢復決策

### DEC-021：敏感 API runtime 直接部署於 AWS 香港區域

- 狀態：`accepted`
- 所有可接觸 PII、身份、session、案件、文件 metadata、audit 或權限的 API/BFF/worker 必須部署於 AWS `ap-east-1`。
- Vercel 只可承載經驗證不含 PII、個人化 cache、敏感 log 或 server-side data fetch 的非敏感前端資產；也可完全停用。具體 public frontend hosting 仍可在不改變此邊界下決定。
- 不允許把敏感 Vercel Function、preview data、error payload、analytics 或 support trace 當作香港 runtime 的延伸。

### DEC-022：資料庫恢復目標起點

- 狀態：`accepted`
- Release 1 資料庫規劃起點為 `RPO 5 分鐘`、`RTO 4 小時`。
- 這是待演練驗證的目標，不是未經測試的 SLA 承諾；PITR 建立新 instance，切換前需 reconciliation 和人工批准。
- 文件 RPO/RTO、整個香港 region 長時間故障的可接受停機仍是 open，不得以區外 replica 靜默解決。

### DEC-023：香港觀測與最小披露通知

- 狀態：`accepted`
- log、metric、trace 採欄位白名單和 PII redaction，落點與 archive 全部位於香港；禁止記錄 raw token、invite secret、文件內容或未遮罩個人資料。
- 通知內容只表達「有待辦事項」，不含 Student、Guardian、Case、SchoolTarget 或文件細節。
- 所有外部副作用使用 transactional outbox、idempotency key、有界重試、delivery receipt 和 reconciliation；不得以 blind retry 造成重複通知。

### DEC-024：正式採用香港 S3 私有文件域

- 狀態：`accepted`
- 文件 bytes 採 `S3 Standard ap-east-1`、single-Region customer-managed KMS key、S3 Bucket Key、Versioning、香港 SQS/DLQ、香港 scanner 和 audit archive。
- 不啟用 CRR、Multi-Region Access Point、CloudFront 私密文件快取、Transfer Acceleration、One Zone-IA 或區外 failover。
- 本決策是技術選型，不是建立 AWS 資源或產生費用的授權。

### DEC-025：non-K12 前台佔位規則

- 狀態：`accepted`
- 大學、碩士等 non-K12 類型在前台保留可見佔位選項，明確標示「正在開發中」。
- 佔位選項不得進入詳細流程、建立正式案件或繞過 Release 1 只啟用 K12 的服務端限制。

## 16. 已批准的領域、工作流與授權決策

### DEC-026：四層 K12 assessment schema 是唯一規則來源

- 狀態：`accepted`
- 正式 assessment schema 由 `K12 base + education stage + school system + admission route` 四層版本化 module 組合。
- UI 顯示、服務端 validation、blocking rules、完整度和 migration 都從同一 manifest 解析，不另寫平行規則。

### DEC-027：案件主狀態與學校目標狀態分離

- 狀態：`accepted`
- `ServiceCase` 使用八階段案件狀態機作營運主狀態；`SchoolTarget` 逐校獨立追蹤候選、準備、已提交、面試、候補、錄取、拒絕和撤回。
- 案件級狀態只作摘要，不得覆蓋或推導不存在的逐校事實。
- 不同 admission route 共用核心 SchoolTarget 狀態機，但使用獨立欄位模板、材料、DDL 和 evidence requirements。

### DEC-028：Task 使用獨立受控狀態機

- 狀態：`accepted`
- Task 至少涵蓋 `accepted`、`rejected`、`reassigned`、`completed`、`approved`、`overdue`、`cancelled`。
- 只有 assignee 或具明確權限的 actor 可操作；完成不等於驗收，重新分派、拒絕、取消和驗收都保留理由與 audit。

### DEC-029：CaseCollaborator 授權預設 7 天

- 狀態：`accepted`
- 所有 CaseCollaborator grant 預設 7 天到期，取代 `DEC-011` 的普通 scope 30 天預設；不得超過案件結束日。
- 敏感 `identity_contact`／`internal_notes` 仍需 Founder 批准、填理由且最長 7 天。
- revoke、帳號停用、移除 collaborator 或案件結案必須立即生效並寫 audit；續期是新的授權決定。

### DEC-030：文件 soft-delete、legal hold 與版本恢復

- 狀態：`accepted`
- 文件與附件預設有 30 天 `soft-delete` 恢復窗口；legal hold 期間不得 purge，且 legal hold 本身不設自動到期上限。
- restore 只能指向已掃描、未撤銷且未被 policy 禁止的版本；恢復建立審計事件，不覆寫版本歷史。
- 結案後總保留年限和各文件分類的 retention schedule 仍待法律／業務決策。

### DEC-042：duplicate merge 與 undo

- 狀態：`accepted`
- Student、Guardian、School 疑似重複只產生候選，不自動合併。
- 正式 merge 需相應批准，保留來源 ID、alias mapping、欄位 provenance 和 merge audit；原記錄不得 hard delete。
- undo 以新的 corrective revision 恢復映射和 resolved view，不改寫既有 merge 歷史。

### DEC-043：assessment 欄位級資料契約

- 狀態：`accepted`
- 每個 answer 保存 schema/module version、最後更新者、更新時間、來源和可見性；重要派生值還需保存計算／規則版本。
- `unknown`、`not_applicable`、`declined_to_provide` 是明確狀態，不得以 null、空字串或自由文字代替。

### DEC-044：併發更新禁止 last-write-wins

- 狀態：`accepted`
- 寫入使用 record version／ETag 等 optimistic concurrency token；stale write 回應一致的 `409 conflict` contract。
- UI 顯示目前版本與使用者修改的差異，由 actor 重新確認；禁止靜默覆蓋。

### DEC-045：個人與案件刪除流程

- 狀態：`accepted`
- Student、Guardian、ServiceCase 刪除先進 `pending_delete`，經 Founder 批准和 retention/legal-hold/referential-integrity 檢查後才 purge。
- purge 後 audit 只保留必要事件 metadata 和非 PII tombstone identifier，不保留原始 PII；任何例外保留須有法律依據和 owner。

### DEC-046：敏感資料匯出控制

- 狀態：`accepted`
- 所有敏感 export 需 Founder 批准；CaseCollaborator 不具 export capability。
- 產物使用一次性短效下載、watermark、香港儲存和完整 audit；到期後依 retention job 清除，禁止公開 URL 或永久分享連結。

## 17. 已批准的資料流、交付與營運決策

### DEC-031：rollout 與 schema migration 順序

- 狀態：`amended`
- 使用 versioned `expand/contract` migration、feature flags 和受控 canary；停止以 runtime `CREATE TABLE IF NOT EXISTS` 管理核心 schema。
- 順序為香港 staging synthetic verification -> business-data-empty 香港 production tenant -> 1–3 個逐案 reconstruction/review/activation/reconciliation -> 至少五個營業日 observation -> 人工 gate 擴至總計 5–10 個 monitored active pilot cases -> `expand`／`hold`／`rollback` 決定；每一關都需 acceptance evidence 和人工 go/no-go。`[DEC-061, DEC-062]`
- 不做沒有 reconciliation、owner 和退出條件的長期雙寫。

### DEC-032：API、transaction、outbox 與 audit contract

- 狀態：`accepted`
- API 使用版本化 success/error schema、穩定 error code、request ID 和明確 retryability；caller 不解析自由文字判斷錯誤。
- 業務寫入使用 transaction + optimistic concurrency；外部副作用寫入同交易 outbox，由 worker 以 idempotency key 執行。
- 所有狀態轉移、授權、批准、匯出、刪除、恢復和高風險讀取寫入 append-only audit。

### DEC-033：Release 1 只提供站內通知

- 狀態：`amended`
- Release 1 不發外部 Email、WhatsApp 或 SMS；身份邀請的香港通信流程需另行受控處理，不能等同一般通知功能。
- 外部通知只有在供應商、香港處理位置、DPA、payload redaction、retention 和 delivery contract 通過批准後才可用 feature flag 開啟。
- Portal raw key 和 charge notice 只可由已批准人工渠道在平台外傳遞；系統可保存不含通訊內容的 opaque delivery receipt。這不構成平台外部通知功能，且 `DP-12` 未批准前不得建立 delivery workflow。`[DEC-064]`

### DEC-034：Release 1 驗證門檻

- 狀態：`accepted`
- 最低 gate 包含 targeted unit/integration/authorization tests、schema/migration checks，以及代表性 desktop/mobile browser smoke tests。
- auth/data/migration 邊界還需負向授權、併發、partial failure、replay/idempotency 和 rollback/restore evidence。
- 不得以刪除測試、放寬 assertion、忽略 failure 或只跑容易子集取得通過；frontend lint/build 仍須依專案規則取得明確授權。

### DEC-035：Release 1 營運值守模式

- 狀態：`accepted`
- Release 1 設 named operational owner 和營業時間內處理機制，不建立 24/7 on-call。
- 這不降低 fail-closed、告警、事件紀錄、runbook 和次一營業時段處置要求；重大資料外洩／安全事件仍依 incident policy 即時升級。

### DEC-036：初始資料遷移採單向 backfill + reconciliation

- 狀態：`accepted`
- 初始資料從權威來源單向 backfill 至香港系統；每批保存 source snapshot/version、mapping version、counts、hashes、rejects 和 reconciliation report。
- 不做長期雙寫。cutover 前需 freeze 或明確 delta capture；失敗批次不可靜默部分成功。
- 回退恢復舊 read path 或停用 feature flag；已發生的外部副作用使用 compensation，不把 checkpoint 當 transaction。

### DEC-037：RDS automated backup 保留 7 天

- 狀態：`accepted`
- Production RDS automated backup retention 設 7 天作 baseline；snapshot、transaction log、KMS 和 restore target 全部留在 `ap-east-1`。
- 7 天不取代 audit、migration rollback、legal hold 或獨立 restore drill；量測與法規要求可觸發後續修訂。

### DEC-038：恢復演練頻率

- 狀態：`accepted`
- 每月在香港 staging 執行一次 restore drill；每季執行一次涵蓋 DB + 文件 metadata/bytes 的完整恢復演練。
- 報告需記錄 source backup、目標環境、實際 RPO/RTO、counts/hashes、失敗、補救 owner 和是否達標；未達標不得自評通過。

### DEC-039：audit 與 application log 保留期

- 狀態：`accepted`
- append-only audit log 保留 1 年；一般 application log 保留 30 天，兩者皆位於香港並執行 PII redaction。
- security/legal hold、法規或事件調查需要更長保留時，必須記錄 scope、法律／政策依據、owner 和到期 review；不得默認無限保存 PII。

### DEC-040：首個里程碑採 end-to-end vertical slice

- 狀態：`accepted`
- 第一個實施里程碑先完成 invite/login/TOTP、Student、ServiceCase、SchoolTarget、服務端授權和 audit 的端到端最小閉環，再擴展其餘 breadth。
- vertical slice 必須使用正式 migration、error contract、負向授權和 rollback 路徑，不得以 mock-only demo 代替。

### DEC-041：首條端到端驗收場景

- 狀態：`accepted`
- Founder 邀請 Advisor -> Advisor 建立 K12 正式案件 -> 指定受限 CaseCollaborator -> 建立 provisional School／SchoolChangeRequest -> 上傳並掃描文件 -> 推進 Task 與案件狀態 -> 驗證 audit、revoke 和 rollback。
- 驗收同時包含未授權 actor、stale write、掃描失敗、outbox retry、權限到期和 rollback 後 resolved view 的反例。

### DEC-047：AI 輸出治理

- 狀態：`accepted`
- AI 評估／報告建立 immutable version，綁定 input data snapshot、prompt/instruction version、model/version、knowledge snapshot、sources 和 evaluator evidence。
- AI 不得覆寫 source-of-truth，不得成為自身唯一 release judge，也不得未經人工批准直接對客發布。
- 本決策是未來功能的治理 contract，不代表把 AI 對客報告加入 Release 1。

### DEC-048：外部 AI provider 的資料邊界

- 狀態：`accepted`
- 未以第一方文件和 DPA 證明香港處理位置、零訓練、retention/delete、subprocessor 和 support access 前，外部 AI provider 不得接觸 PII。
- Release 1 只能傳送經驗證不可逆脫敏的輸入，否則停用該 AI 功能；log、trace、prompt cache 和 human review queue 也在此限制內。

### DEC-049：Excel／CSV 匯入 contract

- 狀態：`accepted`
- 匯入流程必須先做欄位映射、預覽、schema/business-rule 驗證和人工批准，才可寫入正式資料。
- 每個 import batch 有 idempotency key、source hash、mapping/schema version、逐列錯誤報告和 reconciliation；正式寫入採 atomic batch，或以新的 corrective batch 回復，不刪 audit。
- 本決策定義未來匯入功能的 contract，不代表把 Excel 匯入加入 Release 1。

### DEC-050：crawler-to-frontend release gate

- 狀態：`accepted`
- snapshot 只接受 `records.json`、`review_queue.json`、`run_summary.json`、`publish_manifest.json` 四檔完整集合。
- 候選同步前驗證 manifest、schema、counts、hashes、file set 和 warnings；有 warning 時不得自動接受、同步或發布。
- crawler publish、snapshot copy、frontend commit/push 和 Vercel deploy 是四個獨立人工批准的 release actions。

## 18. 已批准的下一階段實施計畫結構

已批准的完整實施計畫拆成：

```text
Phase 0  contracts / harness / threat model
  -> Phase 1  end-to-end vertical slice
  -> Phase 2  K12 workflow and data breadth
  -> Phase 3  synthetic verification + empty production tenant
  -> Phase 4  1-3 reconstructed cases + five-business-day observation
              -> human-gated 5-10 monitored active pilot
```

每個 phase 均需列出 owner、input/output contract、migration、acceptance evidence、security/privacy gate、rollback、budget、stopping condition 和人工批准點。使用者於 2026-08-01 批准原始 phase topology，並於 2026-08-11 依 `DEC-061`／`DEC-062` 批准上述修訂 topology 按 dependency order 開始本地實作；這不代表批准任何 ticket-level 外部副作用、真實資料操作或 release action。

## 19. 已批准的多租戶、平台控制與知識治理決策

本節描述 Release 1 與其後擴展到其他訂閱公司的架構前提。`DEC-064` 已把受限單一案件 Portal 與 aggregate-only PlatformBilling 加入 Release 1；公開獲客、第二租戶 activation、完整 subscription lifecycle 和 AI 知識工作流仍不因此加入。`DEC-001`／`DEC-002` 的「首版只服務天星一個 organization」仍然有效。

### DEC-051：拆分訂閱客戶、組織使用者、終端客戶與平台操作者

- 狀態：`accepted`
- `CustomerOrganization` 是購買或試用平台的教育服務公司，也是租戶資料的業務歸屬與授權隔離單位；以不可變 UUID 識別。其在適用法律下的資料責任角色仍需法律／合約確認，不由產品術語代替。
- `OrganizationUser` 是透過 `OrganizationMembership` 加入公司的使用者，包含 `OrganizationOwner`、`OrganizationAdmin`、`PrimaryAdvisor`、`CaseCollaborator` 和 `Contractor` 等角色／委派形態。
- `EndClient` 是接受升學服務的 Student／Guardian，不等同訂閱公司，也不因出現在 CRM 而取得內部帳號。
- `PlatformOperator` 是平台設計、運維、安全或支援人員；其平台身份與任何 `OrganizationMembership` 分離，不因是平台管理員而自動取得租戶內容權。
- `User` 是跨組織穩定 actor identity；同一 User 可持有多個 OrganizationMembership，但每次業務請求只能在一個明確 active organization context 中執行。email 不作 membership、案件或資料所有權外鍵。
- 後續新增的多租戶 schema、interface、UI 和實施文件不得單獨使用含義不明的「客戶」；必須寫明 `CustomerOrganization`、`OrganizationUser` 或 `EndClient`。既有歷史決策可保留原文，但新契約必須消除歧義。

### DEC-052：資料分為權威事實、受治理產物和可重建派生資料

- 狀態：`accepted`
- A 類「權威業務事實」包括 Student、Guardian、ServiceCase、Assessment 原始答案、SchoolTarget／CaseOutcome、Task、文件版本、membership／grant、批准決定、來源證據和 append-only audit；由其 owning module 的服務端規則和資料庫不變量強制。
- B 類「受治理業務產物」包括已批准選校方案、對客報告、KnowledgeArticleVersion、KnowledgeSnapshot 和 pinned resolved school revision。它們雖由 A 類資料產生，但批准後本身成為不可靜默改寫的正式版本，必須綁定 input/source snapshot、規則版本、owner 和批准證據。
- C 類「可重建派生資料」包括 dashboard projection、完整度、搜尋／向量索引、cache、提醒和未批准 AI draft。它們不得成為唯一真相，必須保存 derivation/version 或可從 A／B 類資料重新建立。
- audit 是控制證據，不是權威資料的備份；cache、checkpoint、standby、PITR、legal hold 和正式版本也不得互相替代。
- 權威寫入採 `authn -> organization/case authz -> schema/state validation -> transaction(data + audit + outbox)`；副作用 worker 消費 outbox，以 idempotency key、有界重試、receipt 和 reconciliation 更新自身狀態。
- source 被更正、撤銷或依法 purge 時，所有 B／C 類引用必須進入 impact check；禁止留下可重新識別的派生副本，亦禁止無證據地改寫已批准歷史。

### DEC-053：多租戶採共享香港 PostgreSQL、強制 tenant key 與 RLS 防禦

- 狀態：`accepted`
- Release 1 繼續只允許天星一個 CustomerOrganization；共享 schema 多租戶能力是第二個訂閱組織前的 migration，不得在 mock-only schema 中假裝完成。
- `Organization`、`Subscription`、`Entitlement`、`OrganizationMembership` 和 tenant-scoped role/grant 是正式基礎實體。所有租戶擁有的資料列都必須有不可為空的 `organization_id`；不得只靠祖先 join、route parameter 或前端狀態推導租戶。
- tenant-owned 關係使用包含 `organization_id` 的 composite foreign key／unique constraint，阻止把一個租戶的 Student、Case、Task、Document、Outcome、Knowledge 或 export 關聯到另一租戶。
- 香港 RDS 採 shared database/shared schema 起步，應用服務端授權為第一道控制，PostgreSQL `FORCE ROW LEVEL SECURITY` 為第二道控制；一般 application DB role 不得具 `BYPASSRLS` 或 table owner 權限。
- 連線池只能在 transaction 中以可信 server context 設定 tenant，交易結束後不得殘留；migration、restore 和受控維運使用分離的高權限身份、明確 runbook 和 audit。
- Cognito 只證明 User identity；active organization、membership status、role、case scope、capability、expiry 和 `session_version` 一律由香港 RDS 決定。瀏覽器傳入的 `organization_id` 不構成授權證據。
- S3 object metadata/key、presigned intent、cache key、search/vector namespace、outbox/job、notification、rate/cost budget、log/audit 和 export 都必須攜帶並驗證 organization context；僅在資料庫加一欄不算完成隔離。
- 平台學校 base snapshot 可作全局受控資料；租戶的 shortlist、內部評語、案件關聯、私有 overlay 和知識仍是 tenant-owned。租戶提出的事實修正只有經平台資料治理流程另行批准，才可成為新的全局 revision。

### DEC-054：公開獲客、平台控制和租戶業務是三個產品平面

- 狀態：`amended`
- `Public Plane` 承載無 PII／無個人化 cache 的公司官網、產品資訊和獲客入口。公開官網有助於獲客但不是第二個租戶的技術前置條件；Lead／Consent 是獨立域，不得由表單直接建立正式 membership、Student 或 ServiceCase。
- `Platform Control Plane` 承載 tenant onboarding、subscription/entitlement、quota、feature flag、release、job/outbox/DLQ、告警、事件、備份／restore drill、安全審計、支援授權和全局學校資料治理。建立第二個租戶前必須有最小可用控制平面或等價受控 runbook。
- `Tenant Data Plane` 承載各 CustomerOrganization 的 CRM、案件、文件、知識、報告與內部營運；不得讓 control-plane list/query 預設返回 Student、Guardian、文件內容、內部評語或其他 tenant PII。
- PlatformOperator 不持有常駐 tenant-content super-admin 權。支援或事故調查需建立 target organization、scope、capability、reason、approver、expiry 和 request/incident ID 完整的 time-bound support grant；啟用、每次使用、撤銷和到期都寫 audit。
- 租戶品牌門戶／EndClient portal 是 Tenant Data Plane 的受限 read-only interface，不等同公開官網或 Platform Control Plane；`DEC-064` 只把單一案件、allowlisted、無文件下載的最小 surface 加入 Release 1。

### DEC-055：角色頁面是授權結果，不是授權來源

- 狀態：`accepted`

| Actor | 必要頁面／工作面 | 明確禁止的預設範圍 |
| --- | --- | --- |
| PlatformOperator | 平台概覽、租戶與訂閱、版本／feature flag、job/outbox/DLQ、告警／事件、備份與演練、安全審計、支援授權、全局資料治理、成本配額 | tenant Student／Guardian、案件內容、內部評語和文件 bytes |
| OrganizationOwner | 公司看板、全公司案件、owner-only 審批、團隊／邀請、角色／授權、案件分派、知識治理、公司 audit、敏感 export、delete/legal hold、設定、用量／訂閱 | 其他 tenant 資料、平台 secrets、高權限維運和跨租戶統計明細 |
| OrganizationAdmin | 公司營運看板、全公司案件、一般審批、團隊／顧問邀請、案件分派、獲委派的知識治理與設定 | Founder／Owner 保留的敏感 export 批准、purge、legal-hold release、owner/訂閱終止和 support-access 批准 |
| PrimaryAdvisor | 我的看板、被分配 Student／Case、案件工作台、Assessment、選校／逐校進度、Task、Document、Communication、Report、通知、已批准知識搜尋 | 未分配案件、公司級安全設定、全量敏感 export |
| CaseCollaborator | 指定案件中明確 grant 的 scope 及 `view/comment/edit` capability | 未授權 scope；`identity_contact`／`internal_notes` 預設不可見；不可 export 或執行不可下放操作 |
| Contractor | 我的 Task、任務所需脫敏背景、交付與退回 | 完整案件、真實聯絡資料、家庭詳情、內部評語和原始案例知識 |
| EndClient（後續） | 單一學生的已批准進度、DDL、材料狀態與報告 | 內部評語、主觀未批准內容、其他家庭和內部流程 |

- 案件工作台使用 `overview / student-guardian / assessment / school-targets / tasks / documents / communications / reports / timeline-audit` 穩定信息架構；每個 tab 的資料仍由同一服務端 organization + case + scope + capability policy 判斷。
- 前端隱藏 navigation、button 或 tab 只改善使用體驗，不構成 authz。未授權 API 應按資料存在性披露規則回應 `403` 或 `404`，並不得以 counts、search suggestions、error details 或 timing 洩露其他租戶資料。

### DEC-056：先採模組化單體，以 owning module interface 保持 locality

- 狀態：`accepted`
- Release 1 和初始多租戶擴展採 modular monolith；不得為組織圖或假想規模預先拆微服務。只有量測證明獨立擴展、故障隔離、部署節奏或資料治理需要時，才為特定 module 建立遠端 adapter 和遷移決策。

| Module | 擁有的規則與資料 |
| --- | --- |
| Identity | provider subject、invite/login/TOTP/session revoke adapter；不擁有業務角色 |
| TenantAccess | Organization、Subscription、Entitlement、Membership、RoleBinding、grant、policy evaluation、support access |
| CRM | Student、Guardian、relationship、identity/contact profile、duplicate candidate/merge |
| CaseWorkflow | ServiceCase、Assessment manifest/answers、SchoolTarget、CaseOutcome、受控轉態與批准 |
| Task | Task、assignment、accept/reject/reassign/complete/approve/cancel 和 DDL |
| SchoolIntelligence | crawler snapshot、evidence、SchoolChangeRequest、overlay 和 resolved revision |
| Document | upload intent、metadata、version、scan、download/export、retention/legal hold/delete/restore |
| Knowledge | LessonCandidate、KnowledgeArticleVersion、KnowledgeSnapshot、citation、visibility 和 review |
| ReportingAI | report version、input snapshot、de-identification、provider adapter、evaluation 和 human approval |
| Notification | in-app delivery、template、outbox consumption、receipt、retry 和 deduplication |
| AuditOperations | append-only audit query、allowed telemetry、alert/incident/runbook 和 restore evidence |

- 每個 module 只有一個對 callers 和 tests 公開的 interface；interface 必須包含 input/output、organization/actor context、不變量、error code、concurrency token、retryability、performance 和 audit effect，不得只定義函式型別。
- module 可有自己的 table/schema，但其他 module 不得直接寫入；跨 module 的同步不變量走 owning interface／同交易 command，通知、索引、報告等可延後副作用走 versioned outbox event。
- read model 可組合多 module 的已授權資料，但不得取得寫入所有權；cache、dashboard、search 和 AI index 可被刪除並由權威資料重建。
- 每次 module 變更只透過其 interface、contract tests、negative authorization tests、migration 和 event compatibility 驗證；caller 不解析自由文字或依賴 owning module 內部 table shape。

### DEC-057：成功需同時通過業務、隔離、正確性與恢復證據

- 狀態：`amended`
- Release 1 成效 gate 改為：coverage-driven synthetic scenario gate、business-data-empty production readiness、1–3 個逐案重建／審核／啟用／reconciliation、至少五個營業日 observation，以及人工 gate 擴至總計 5–10 個 active pilot；負責人仍需無需通訊追問即可掌握下一步，並完成至少一個真實委派閉環和明確 `expand`／`hold`／`rollback` 決定。`[DEC-061, DEC-062]`
- 多租戶 gate 另外要求：跨租戶 UI／direct API／ID guessing／search／export／S3／background job／cache 負向測試全部通過；任何未授權跨租戶讀寫均為 release blocker，不以低發生率或前端不可見降級。
- 正確性 gate 要求 stale write 回 `409`、replay/idempotency 不重複副作用、partial failure 可 reconciliation、approved version 不被失敗生成覆寫、來源與 derivation 可追溯。
- 營運 gate 沿用主要頁面 P95 < 2 秒、一般 API P95 < 500 ms、至少 100 個活躍案件、月可用性 99.5%、DB RPO 5 分鐘／RTO 4 小時等現行目標；未經實測不得升格為 SLA。
- 恢復 gate 要求 restore 後以 counts/hashes、tenant distribution、document version/metadata linkage、audit continuity 和抽樣業務查詢驗證；僅有 AWS job success 不是 passed。
- failure mode 至少覆蓋：租戶 context 缺失／殘留、授權過期未生效、併發覆寫、部分 migration、outbox poison message、S3 event 遺漏／重播、外部 provider timeout、索引過期、錯誤知識發布、PII 進入 log/AI、香港 region 長時間故障和 support grant 濫用。
- 每個告警和演練需記錄 impact organization、detector、owner、runbook、停止／降級狀態和關閉證據；精確 SLI/alert catalogue 仍按 P2 待決策項交付。

### DEC-058：完整案例保留與可重用知識分離

- 狀態：`accepted`
- 原始 Student／Guardian／ServiceCase／SchoolTarget／Document 是受案件授權和 retention 管理的 CRM 記錄，不因結案或被選為案例而自動向全公司或平台開放。
- 每個完成、拒絕、候補、撤回、未提交或中止的 SchoolTarget 都必須建立 `CaseOutcome`，保存受控 outcome code、日期、evidence、source、actor 和 record version；一個 ServiceCase 可同時含錄取、拒絕和撤回，不得用單一「成功／失敗」覆蓋逐校事實。
- `ServiceGoalOutcome` 可另記是否達成已批准的整體服務目標，但不能改寫 SchoolTarget 結果。案件結案 guard 必須確認所有 active targets 已有結果或具理由的 terminal state。
- `LessonCandidate` 是 Advisor 根據案件提出的經驗候選；「某因素導致錄取／拒絕」是需來源、反例、信心和 reviewer 的分析 claim，不是從 outcome 自動推導的事實。
- `KnowledgeArticleVersion` 是經去識別化、審核、版本化後的可重用組織知識；必須保存 source case/target reference（受限）、適用 route/stage/system、owner、reviewer、visibility、citation、confidence、valid-from 和 supersession reason。
- 「保留所有成功和失敗案例」表示在合法 retention 期內完整保存權威 outcome/evidence，並讓每個結案 outcome 可進入 lesson review；不表示永久保存全部 PII，亦不表示所有原始案件自動進入 AI corpus。

### DEC-059：知識採 tenant-private、受審核版本與香港可重建索引

- 狀態：`accepted`
- 知識發布狀態為 `draft -> submitted -> reviewed -> approved/published -> superseded/archived`；修改建立 immutable version，不原地覆寫。作者可修訂或提交，但不得單獨批准自己提出的正式案例知識。
- OrganizationOwner／明確委派的 KnowledgeReviewer 可批准、限制可見性、supersede 或 archive；Advisor 可搜尋已批准 tenant knowledge 並提交 LessonCandidate；CaseCollaborator／Contractor 只可取得其任務或 grant 明確需要的知識，不取得 source case 權。
- tenant knowledge 預設只屬其 CustomerOrganization。PlatformOperator 不預設可讀；租戶內容不得因匿名化程序或平台條款而自動提升為 global knowledge。
- global platform knowledge 與 tenant knowledge 使用分離的 namespace、owner、review 和 snapshot。任何 tenant-to-global promotion 必須取得明確 opt-in、去識別化／重識別風險審核、使用範圍、撤回／刪除和利益／授權條款批准。
- Knowledge metadata、version、citation、permission、review 和 snapshot manifest 存香港 RDS；附件存香港 S3。全文／vector index 是 C 類派生資料，必須位於批准的香港處理邊界、按 organization namespace 隔離並可由 approved versions 重建。
- 提供給 AI／報告的 KnowledgeSnapshot 必須 immutable，綁定 article versions、建立時間、policy/schema version 和 evaluator evidence；新知識不得靜默改寫舊報告。
- source case 更正、撤銷、purge 或 visibility 收緊時觸發 knowledge impact review；在 review 完成前相關 article/snapshot 不得新用於對客生成。依法刪除後不得在 citation、index、prompt cache、trace 或附件保留可重新識別內容。

### DEC-060：第二個訂閱組織前仍需批准的不可逆語義

- 狀態：`open`
- Subscription lifecycle 及 entitlement 行為：`trial`、`active`、`past_due`、`suspended`、`terminated` 的正式狀態、轉移、寬限期，以及每個狀態是可寫、只讀、可 export 或完全拒絕。
- PlatformOperator support grant：可批准人、緊急事故例外、最長有效期、可見 scope、雙人批准門檻、租戶通知和事後 review。
- Student／Guardian／ServiceCase、Outcome、文件分類和知識來源在結案／解約後的 retention schedule；legal hold、PDPO data-subject request、租戶 export 和最終 purge 的 owner 與證據。
- 文件 RPO/RTO、香港 region 長時間故障可接受停機，以及不得使用區外 replica 時的明確 fail-closed／read-only／manual fallback。
- tenant-to-global knowledge 的 opt-in 文本、撤回後對既有 approved report 的處理、知識貢獻權益和重識別風險接受人。
- 租戶終止／轉移流程：完整 export schema、短效下載、identity disable、support access revoke、備份到期、衍生索引清理、tombstone 和刪除證明。
- 上述問題會改變資料保留、授權、收入或生產狀態，不得由 schema 預設值、UI 文案或工程方便性代替業務／法律批准。

### DEC-061：Historical Case Reconstruction 採逐案錄入、獨立審核與原子啟用

- 狀態：`accepted`
- 只有被分配到該 opaque pilot reference 的 Primary Advisor 可建立和修訂 reconstruction draft；Founder reviewer 必須是不同 actor，禁止 self-approval。
- 狀態為 `draft -> submitted -> changes_requested -> draft` 或 `submitted -> approved -> activated`。提交後版本凍結；最多兩次 changes-requested cycle，之後進入 `needs_human`。
- 歷史事件分開保存 `occurred_at`、`recorded_at`、recorder identity 與有來源的 reported actor；不得使用未來時間、冒充原始 actor 或製造不合法歷史順序。
- 每個必要節點須有 source evidence 或具 type、reason、owner、resolution target、Founder decision 和 record version 的可見 HistoryGap。
- activated 前不得進入 operational dashboard、delegation、notification、document availability 或 SLA；歷史事件不得發送過期 notification、舊 task 或其他即時副作用。
- activation 必須在 CaseWorkflow owning module 的單一 transaction 寫入 authoritative facts、history、approved gaps、append-only audit 和必要 outbox。activated history 不可原地修改；修正建立帶 reason、actor、expected version 和 audit 的 corrective revision。
- 所有 write command 使用 idempotency key 與 optimistic concurrency；stale version 回穩定 `409`，replay 不建立第二份 fact 或 effect。每案 activation 後須與客戶來源逐案 reconciliation，unexplained difference 必須為零。

### DEC-062：Privacy-safe Product Telemetry 採封閉 allowlist 並與 mandatory audit 分離

- 狀態：`accepted`
- Telemetry 只接受版本化 allowlisted event names、opaque actor／organization／case／request／session／job identifiers、actor role、stable result/error code、retryability、duration、timestamp、build/schema/policy version 和不含 query 的 route template。
- 禁止 Student／Guardian PII、文件名稱或內容、OCR、notes、assessment answers、communication content、raw token／secret／cookie／presigned URL、query string、form/free-text value、session replay、screen recording、keystroke 或 DOM text capture。
- Product/technical telemetry baseline retention 為 30 天，與保留 1 年的 append-only audit 分離；telemetry 不得成為授權來源、business truth 或 deterministic release gate 的替代品。
- Telemetry sink outage 進入明確 degraded Operations state，業務可 fail open 並告警，不重播 business mutation；mandatory audit 仍與 mutation transaction 綁定，audit 無法持久化時 mutation 必須 fail closed。
- Privacy-safe telemetry policy 由 AuditOperations owning module 的 schema／allowlist 在 producer 邊界強制；不符合 event 必須拒絕。Production 啟用仍需 Privacy、Operations、retention、product notice 和 exact cloud configuration 批准。

### DEC-063：公開學校招生情報採 Cloudflare R2，全部租戶和個人資料維持 AWS 香港邊界

- 狀態：`accepted`（2026-08-11 使用者批准架構方向；不等於註冊、採購、provision、upload、migration、DNS、deployment 或 deletion 授權）
- Cloudflare 只承載已通過 crawler audit／人工 publish gate、經 public field allowlist 和 PII／licence review 的全港學校招生公開 release。`review_queue`、fallback queue、source capture、run/log/LLM trace、crawler tickets/config/review decisions 和任何 tenant/customer/personal data 不得公開或上傳 Cloudflare。
- 公開 release 使用 R2 custom domain 和不可變 `releases/<release_id>/...` keys；`latest.json` 只在 upload、read-back、schema/count/SHA-256 reconciliation 全部通過後最後更新。R2 不具等同 S3 Versioning/Object Lock/KMS/event/inventory 的完整 contract，由 immutable keys、external approval receipt、prior pointer 和 reconciliation 補足；不得把 S3-compatible 當語義等同。
- Cloudflare 可作 authoritative DNS，但 `app.<domain>` 必須 DNS-only；只有 `data.<domain>` 走 R2 custom-domain/proxy。Cloudflare 不得接收 authenticated ERP request、cookie、authorization header、private response、Server Action、tenant data 或敏感 log。
- 所有可選 region 的 AWS workload、data、identity、database、document、queue、key、log 和 backup resources 一律使用香港 `ap-east-1`。無香港 endpoint 的 AWS workload service 不採用；IAM、Organizations、Billing 等無 region 選項的 global control plane 只可處理 account/permission/billing metadata，不承載敏感 business payload。
- Cognito 使用 `ap-east-1` User Pool 和經驗證的 regional/provider domain；不建立需要 `us-east-1` ACM certificate／global CloudFront 的 custom login domain。若 regional managed-login 不能通過香港 data-flow probe，authentication launch 必須 blocked，不得 fallback 到區外 AWS service。
- AWS `ap-east-1` 承載完整 authenticated Next.js UI/BFF、Cognito、RDS business authorization/CRM、private S3/KMS documents、SQS/scanner、audit/log/backup。Crawler review/control state 和 publication receipt 亦留在 AWS control plane。
- 現有 Git-committed four-file snapshot 是 migration 期間的 bounded local adapter；不得整包公開。正式 consumer 經 `PublicSchoolCatalogue` interface 讀已批准 R2 release；R2 失敗只令 SchoolIntelligence 進入 degraded state，不得影響 identity、CRM、case 或 document mutation。
- Vercel 只作 migration source inventory。AWS cutover 後不得再部署 Vercel，也不得把 Vercel 作 runtime standby 或 rollback target；rollback 只可在 AWS 內回到 prior compatible ECS task definition/image digest，或走 approved forward repair/restore。
- 完整註冊、配置、migration state machine 和 approval gates 見 `TECHNICAL_DESIGN_AWS_CLOUDFLARE_PRODUCTION_DEPLOYMENT.md`。

### DEC-064：Release 1 加入受限 External Portal 與 PlatformBilling

- 狀態：`accepted`（2026-08-11 使用者批准 scope；不等於批准 `DP-01`–`DP-12` 或任何 production action）
- `ExternalPortalAccess` 是與 internal User／OrganizationMembership／Cognito workforce identity 分離的 owning module，只提供單一 organization、viewer、ServiceCase 和 capability-set version 綁定的 read-only workspace。Raw key 只顯示一次、不進 URL/log/telemetry，RDS 只存 keyed hash/fingerprint；文件 bytes/download、export、comment、edit、internal notes 和未 allowlist 欄位預設拒絕。
- `PlatformBilling` 是 Platform Control Plane owning module，只讀不含 PII 的 tenant aggregate projection，管理 immutable contract version、monthly metric snapshot 和 charge-notice draft。Release 1 不產生 tax invoice、不收付款、不進 accounting ledger、不自動傳送 notice。
- `DP-01`–`DP-05`、`DP-10` 未批准前不得完成 Portal schema/policy activation；`DP-06`–`DP-08`、`DP-10` 未批准前不得生成 billing amount/notice workflow；`DP-09`–`DP-12` 和 `DEC-060` 繼續阻擋 subscription enforcement、retention/purge、第二 tenant、delivery 和 production activation。
- 第一個 portal grant、第一份 charge notice、第二 tenant 和任何 migration/cloud/deploy action各自需要 exact-payload human go/no-go。Scope approval 不建立資料、不開 feature flag，也不代表 production-ready。

### DEC-065：Portal redeem 採 function-only database capability

- 狀態：`accepted`
- Application 在香港 runtime 以 Secrets Manager pepper 對 raw portal secret 計算 keyed hash；raw secret 不進 database、URL、log、telemetry、evidence 或 browser storage。
- 獨立 `portal_auth` PostgreSQL role 不具 Portal table `SELECT`、table owner 或 `BYPASSRLS` 權限，只可 `EXECUTE` 一個 hardened `SECURITY DEFINER` equality-lookup function。Function 固定 `search_path`、禁止 dynamic SQL，只返回啟動 tenant-scoped request-time authorization 所需的最小 opaque `organization_id`／`grant_id` lookup result，不返回 viewer、case 或其他 PII。
- Lookup 後必須在新的 tenant-scoped repository transaction 重新檢查 organization、viewer relationship、case、grant、session、expiry/revoke 和 capability version；不能把 lookup receipt 或 cookie claim當成授權真相。Public invalid/expired/revoked response 保持 constant-shape generic `401`。

### DEC-066：Platform Control audit 與 tenant audit 分離

- 狀態：`accepted`
- PlatformOperator 不屬任何 CustomerOrganization membership；不得為了記 audit 偽造 tenant membership、放寬 tenant `audit_events.organization_id`／`actor_user_id` nullability，或授予 tenant-content super-admin。
- PlatformBilling/control-plane mutation 使用獨立 append-only `platform_audit_events` aggregate，actor FK 指向 PlatformOperator identity；`target_organization_id` 只表示受影響 resource scope，不表示 actor 屬於該 tenant。Payload 只允許 billing/control metadata，不保存 Student、Guardian、case content、document 或 notes。
- Billing draft/approve/void/supersede 與 platform audit 在同一 PlatformBilling repository transaction fail closed；tenant application role 不可讀 platform audit，platform role 不可藉 audit path 讀 tenant detail tables。Retention、hash-chain/immutability、failure evidence 和 correction policy不得弱於 tenant audit。

### DEC-067：Historical Case Reconstruction implementation contract

- 狀態：`accepted`（2026-08-12 使用者批准；不等於批准 migration execution、production data 或 case activation）
- Reconstruction 只可使用既有 Case、SchoolTarget、Task 與 Document metadata 的 versioned event types；evidence 只保存 opaque reference 和 type，不得保存文件內容、PII 或自由文字證據。
- `recorded_at` 由 server UTC clock 產生，且必須滿足 `occurred_at <= recorded_at`；相同時間的事件以不可變 `sequence_no` 排序，不得依 database implicit order 或 client timestamp 打破平手。
- Founder 的 `changes_requested` 每次增加 review cycle，並把該 frozen version 退回修訂；只有 assigned Primary Advisor 可建立下一 draft version。第三次 `changes_requested` 不建立下一 draft，直接進入 `needs_human`。
- 每一輪 Founder reviewer 可以不同，但 reviewer 必須與其所審 version 的 recorder 不同。Activated facts 不可覆寫或原地修改，只可用帶 reason、actor、expected version 和 audit 的 append-only corrective revision 更正。
- Activation 唯一允許的 outbox effect 是不含 PII 的 `case_reconstruction.activated.v1`；不得重播歷史 Task、Notification 或其他過期副作用。Authoritative facts、history、approved gaps、audit 與該 outbox event 必須由 CaseWorkflow owning repository 在同一 transaction 寫入。
- Opaque pilot reference 在 organization 內唯一；production approval 必須透過 repository port 驗證，P3-03 additive migration 不得偽造或取代 P3-19 authority。Reconstruction-specific failure 使用 `RECONSTRUCTION_*` stable codes；stale version 統一回 `VERSION_CONFLICT`。

### DEC-068：Production AWS source 與 reviewed-plan implementation contract

- 狀態：`accepted`（2026-08-12 使用者批准 source/plan baseline；明確不授權 Terraform plan/apply、provision、migration 或 deployment）
- ECS web tasks 位於兩個 AZ 的 private subnets，每個 AZ 一個 NAT Gateway；建立 ECR、S3、CloudWatch Logs、SQS、KMS、Secrets Manager 和 STS VPC endpoints。每個 web task 初始為 `1 vCPU / 2 GiB`，desired/minimum count 為 `2`，autoscaling maximum 為 `4`。
- RDS 使用 PostgreSQL 17、`db.t4g.small`、20 GiB gp3、Multi-AZ 和 7-day backup retention；任何規格、storage、availability 或 retention 變更都使 reviewed plan evidence 失效。
- WAF 啟用 AWS managed core 與 known-bad-input rules；rate limit 保持可配置且無 production default，部署前必須以 exact payload 另行批准。
- P3-10 唯一可 apply identity 是保存的 binary `.tfplan` SHA-256。Redacted plan JSON、provider lockfile 和 source-tree hashes 是補充 evidence，不能代替 binary plan identity；任何 identity mismatch 一律停止。
- AWS account、Terraform backend、CIDR、ACM certificate、container image digest、notification recipients 等值全部是必填 external exact payload；Git 不得提供 production default、placeholder fallback 或可被誤用的隱式值。

### DEC-069：統一權限與配置治理

- 狀態：`accepted_for_planning`（2026-08-18 使用者批准架構設計並列入開發計畫；不等於批准 production config、deployment 或真實資料）
- Role、capability、resource、scope、action 與穩定 denial code 是程式契約；role-capability 關係採 organization-scoped immutable versioned policy，只有明確 allow，缺失即 deny。首版由追加 migration 固定現有已批准矩陣，Release 1 不提供任意線上編輯器。
- Access module 是 workspace capability 的唯一服務端決策入口；Route Handler、Service、page guard 和 navigation registry 共用 capability ID。UI visibility 只改善體驗，不能替代 owning repository 在同一 transaction 重新檢查 organization、membership、RoleBinding、resource assignment、scope、expiry 和 policy version。
- 配置分為安全不變量、版本化業務策略、部署運行配置與 organization 展示配置四層；不得以一個通用 key-value 表混合承載。詳見 `decisions/ACCESS_AND_CONFIGURATION_ARCHITECTURE_20260818.md`，並依 `P2-13`、`P2-14`、`P2-15` 實施。
