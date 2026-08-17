# Phase 3 空租戶交付與受控實地試點修訂方案

> **狀態：** 已於 2026-08-11 採納；同日 `R1X-00` 以 `DEC-064`–`DEC-066` 加入受限 Portal/Billing。2026-08-12 使用者另批准 `DEC-067` reconstruction 與 `DEC-068` AWS source/plan implementation contracts；`P3-02` 仍為 `needs_human`，`DP-01`–`DP-12` 仍為逐 ticket gate。這不授權 Terraform plan/apply 或任何 production action。  
> **日期：** 2026-08-11（Asia/Hong_Kong）  
> **適用範圍：** Tianxing K12 Release 1 Phase 3 與 Phase 4 交付、資料 onboarding、使用監控及驗收。  
> **授權邊界：** 本文件不授權 cloud resource 建立、migration execution、production data write、客戶邀請、部署、commit、push、release 或任何真實資料處理。

## 1. 結論

Phase 3 以「空租戶 web 軟件交付客戶，再透過實際使用監控驗證」的方式進行是可行的，但必須採取受控 rollout，不能把 live monitoring 當成唯一測試手段。

本方案採用以下模式：

- Phase 3 交付完整、可登入，但沒有 `Student`、`Guardian` 或 `ServiceCase` 資料的 production tenant。
- 上線前在隔離 staging 以版本化 synthetic cases 完成 deterministic verification。
- Phase 4 由客戶 Primary Advisor 人工重建既有案件歷史，Founder 核准後啟用。
- 首批最多處理 1 至 3 個案件；穩定後才擴至 5 至 10 個 active pilot cases。
- 工程團隊不需要事前取得學生 PII，但客戶仍須確認非識別化案件形狀、例外流程、pilot roles 與成功標準。
- 使用監控只補充驗證採用情況、現場流程與營運表現，不取代授權、狀態機、資料完整性、concurrency、replay、partial failure、rollback 或 restore 測試。

現有 implementation checkpoint 只證明本地 contracts、source、focused tests 與 implementation records 已覆蓋 `P0-01` 至 `P3-02`；`P3-02` deterministic evidence 仍為 `needs_human`。Production PostgreSQL/RDS adapters、AWS Hong Kong placement、browser/a11y、load、restore、production migration 與 release evidence 仍未完成，因此「頁面與後端邏輯已搭載」不等於目前已可直接交付。

## 2. 上游規格與決策修訂

本方案的採納已同步更新以下三份 source-of-truth 文件，不能只修改 phase implementation plan：

1. `PRD.md`
2. `docs/PRD_IMPLEMENTATION_DECISIONS.md`
3. `docs/PRD_PHASE_IMPLEMENTATION_PLAN.md`

### 2.1 決策 ledger

- 將 `DEC-001`、`DEC-031`、`DEC-057` 標記為 `amended`。
- 將 Release 1 rollout 順序改為：

  ```text
  HK staging synthetic verification
    -> empty production tenant
    -> 1-3 reconstructed existing cases
    -> 5-10 monitored active pilot cases
    -> human expand / hold / rollback decision
  ```

- 保留 `DEC-034` 的 deterministic verification gate，並明定 live monitoring 不可替代安全、授權、schema/migration、replay、concurrency、partial-failure 或 restore evidence。
- 保留 `DEC-036` 作為未來受控 batch migration 的規則；初始 pilot 不執行來源 snapshot backfill。
- 新增 `DEC-061`「歷史案件重建」決策：Primary Advisor 錄入，Founder 作為不同的 reviewer 批准；不允許 self-approval。
- 新增 `DEC-062`「產品使用監控」決策：只收 allowlisted events 與 opaque identifiers，禁止 session replay、輸入值、學生 PII、文件內容、URL query、自由文字與 keystroke capture。
- `DEC-067` 固定 reconstruction 的 versioned event、server-time/order、review-cycle、append-only correction、repository approval check、唯一 PII-free activation outbox 與 stable error contract。
- `DEC-068` 固定 P3-07 的 private two-AZ ECS/RDS/WAF source baseline、P3-07A 的 required external payload/plan-only gate，以及 P3-10 只認 saved binary `.tfplan` SHA-256 的 apply identity；目前不授權 plan/apply。

### 2.2 PRD

更新所有要求「30 個真實案例 dry-run」的條文，包括案件狀態機、驗收準則、實施順序、風險 register 與待決策事項，替換為：

- coverage-driven synthetic scenario gate；
- empty-production readiness gate；
- 1 至 3 案的受控 live onboarding；
- 5 至 10 案的 monitored pilot；
- 明確的人工 go/no-go。

不得刪除「真實案件驗證業務適配性」這項目標，只把驗證時點從交付前 real-data dry-run 移至交付後的受控 pilot。

### 2.3 Implementation plan

- 更新 Goal、Scope、Assumptions、Decision Traceability、phase contracts、requirements-to-evidence matrix、backlog、risk register 與 final self-check。
- 移除 Phase 3 對 30-case real source snapshot、real-data backfill preview 和 exact pilot-row cutover 的依賴。
- 把 production infrastructure、empty migration、customer access、monitoring baseline 與 first-case readiness 移入 Phase 3。
- 把既有案件歷史重建、逐案 reconciliation、分段擴展及 live product evidence 移入 Phase 4。
- 重新計算 decision inventory、ticket dependencies 與 backlog total。

## 3. Phase 3：Empty-Tenant Production Readiness

### 3.1 目標

交付一個完整、可登入、可操作空狀態，並已證明能安全接收第一筆真實資料的 production tenant。Phase 3 不宣稱真實業務模型已完成驗證。

### 3.2 In scope

- Production AWS Hong Kong infrastructure、private RDS、Cognito、S3/KMS/SQS/scanner、logging、metrics、traces 與 audit sink。
- Production schema migration 至空 database。
- 完整 Release 1 pages、Route Handler adapters、owning module implementations 與 production adapters。
- 隔離 staging 中的 synthetic golden scenarios、browser journeys、negative authorization、failure injection、rollback 與 restore rehearsal。
- Customer Founder/Admin/Advisor accounts、roles、empty states、first-case reconstruction entry point 與 support/runbook readiness。
- Privacy-safe product telemetry、technical monitoring、alert ownership 與 interview plan。
- Bounded ExternalPortalAccess、aggregate-only PlatformBilling、function-only portal lookup、separate platform audit 的 local/nonprod contract、negative authorization 和 failure evidence。

### 3.3 Out of scope

- Production synthetic Student 或 case records。
- 工程團隊接收或處理來源學生資料。
- Excel/CSV import UI、batch import execution 或 existing-source cutover。
- 超過 3 個首批案件、超過 10 個 pilot cases、第二 tenant activation、portal write/public registration/document download、tax invoice/payment/accounting、AI、external notifications 或 automated crawler sync。

### 3.4 Phase 3 tickets

本修訂文件不再複製 ticket 編號與 outcome，避免舊七關摘要和現行 backlog 產生衝突。唯一權威 ticket graph 是 `PRD_PHASE_IMPLEMENTATION_PLAN.md` v0.11 的 `P3-00`–`P3-19`、單一補充票 `P3-07A`，以及 `R1X-00`–`R1X-12`。當前 checkpoint 為本地 evidence 完成至 `P3-02`，但 P3-02 保持 `needs_human`；`P3-07` 只審 source，`P3-07A` 必須取得另行 exact-payload/tooling 批准才可執行 saved `terraform plan -out`，而 `P3-10` 只可在再一次 apply approval 後使用相同 binary `.tfplan` SHA-256。任何缺失 prerequisite 均 fail closed。

### 3.5 Phase 3 exit criteria

- Production `Student`、`Guardian`、`ServiceCase`、`Task`、`Document` 業務資料均為零。
- Synthetic golden scenarios 的 deterministic matrix 全部通過。
- Customer Founder、Primary Advisor 與 Operations owner 可登入，且角色與負向授權符合 policy。
- Core pages 的 empty/loading/error/denied states 通過 desktop、mobile、keyboard、focus 與 a11y verification。
- Monitoring、alert、PII scan、rollback 與 restore evidence 通過。
- Founder、Security、Privacy、Data 與 Operations owner 簽署 first-case go/no-go。

## 4. Phase 4：Monitored Existing-Case Pilot

### 4.1 目標

由客戶授權人員在 production 逐案重建既有案件，以真實使用證明業務適配性、委派能力、資料隔離、正確性、恢復能力與營運可支持性。

### 4.2 分段 rollout

1. Primary Advisor 重建首批最多 3 個既有案件。
2. Founder 完成逐案 review、history-gap disposition 與 activation approval。
3. 至少觀察 5 個營業日，完成逐案 reconciliation、每日技術 review 與至少一次結構化客戶 review。
4. 沒有 open security/data-integrity blocker，且所有 unexplained differences 為零後，Founder 才可批准擴展至 5 至 10 個案件。
5. Pilot 建議總期為四週；期末形成 `expand`、`hold` 或 `rollback` 決定。

### 4.3 停止與降級規則

- 發現跨案資料暴露、非香港副本、mandatory audit failure、無法恢復的資料錯誤或不可用 rollback 時，停止整個 pilot。
- 單一案件出現未被模型表達的業務情形時，只暫停該案件，保存 counterexample，其他案件可在 Security/Data owner 確認隔離後繼續。
- Telemetry unavailable 時，業務可繼續，但 Operations 必須進入明確 degraded state；mandatory audit unavailable 時，mutation 必須 fail closed。
- 不以刪除資料、改寫 audit、跳過 guard 或製造假的歷史轉態取得通過。

## 5. 歷史案件重建 interface

### 5.1 State model

```text
draft
  -> submitted
  -> changes_requested -> draft
  -> approved
  -> activated
```

- `draft`：Primary Advisor 可錄入 profile、case、SchoolTarget、Task、Document metadata 與歷史事件。
- `submitted`：內容凍結，等待 Founder review。
- `changes_requested`：Founder 指出具體 counterexample；Advisor 只修訂被退回的 draft version。
- `approved`：Founder 已批准內容及所有 history-gap dispositions，但案件尚未進入 live workflow。
- `activated`：單一 transaction 啟用案件、寫入 audit/outbox，並開始正常 workflow。

### 5.2 Module commands

Owning module 對 Route Handler adapters 提供小型、server-only interface，至少承擔以下 commands：

- `createReconstructionDraft`
- `appendHistoricalEvent`
- `recordHistoryGap`
- `submitReconstruction`
- `requestReconstructionChanges`
- `approveReconstruction`
- `activateReconstructedCase`

Route Handlers 只驗證輸入、取得 actor/request context、呼叫一個 owning interface，並映射 versioned success/error envelope；不得自行寫 tables 或拼裝狀態轉移。

### 5.3 Invariants

- Advisor 只能重建獲分配的 pilot case；Founder reviewer 必須是不同 actor，不允許 self-approval。
- `occurred_at` 表示歷史發生時間，`recorded_at` 表示本次錄入時間；歷史事件不得使用未來時間。
- 系統保存 recorder identity；原始 actor 只能作為有來源的 reported fact，不能冒充系統 User 或改寫 audit actor。
- 歷史事件必須依合法 transition 順序排列，最終狀態與客戶來源當前狀態一致。
- 每個必需節點必須具有來源證據，或具有已分類的 history gap。
- History gap 至少保存 type、reason、owner、resolution target、Founder decision 與 record version，並在 dashboard 持續可見，直至補齊或正式豁免。
- Reconstruction 未 activated 前，不得進入日常 dashboard counts、delegation、notification、document availability 或 operational SLA。
- Historical events 不觸發過期 notification、舊 task assignment 或其他即時副作用。
- Activation 以單一 transaction 寫入 authoritative facts、history、approved gaps、audit 與必要 outbox。
- Activated history 不可原地修改；任何修正建立帶原因、actor、expected record version 與 audit 的 corrective revision。
- 所有 write commands 使用 idempotency key 與 optimistic concurrency；stale version 回傳穩定 `409`，replay 不建立第二份 fact 或 effect。

## 6. Monitoring contract

### 6.1 Allowed product events

- Login/session outcome。
- Page or workspace viewed。
- Reconstruction draft created、submitted、changes requested、approved、activated。
- Historical event or history gap recorded。
- Case transition、Task delegation/acceptance/completion/approval outcome。
- Validation、authorization、conflict、retryable infrastructure 與 unexpected error class。
- Page/API latency、queue lag、stuck job、DLQ、scan、revoke 與 reconciliation status。

### 6.2 Allowed fields

- Versioned `event_name`。
- Opaque actor、organization、case、request、session 與 job identifiers。
- Actor role，不包含姓名或聯絡資料。
- Stable result/error code 與 retryability。
- Duration、timestamp、build/schema/policy/snapshot version。
- Allowlisted route template，不包含 query string、free text 或 record contents。

### 6.3 Forbidden fields and capture

- Student/Guardian name、date of birth、HKID、phone、email、address 或 school application contents。
- Document name、bytes、OCR text、internal notes、assessment answers 或 communication contents。
- Raw token、invite secret、cookie、presigned URL、query string 或 form value。
- Session replay、screen recording、keystroke capture、DOM text capture 或 unrestricted analytics properties。

### 6.4 Retention and review

- Product/technical telemetry baseline retention 為 30 天；audit retention 依既有 approved policy，兩者不可混用。
- 首批 1 至 3 案期間由 Operations 每個營業日 review errors、latency、denials、stuck jobs、PII scan 與 reconciliation。
- Product owner 每週進行一次結構化客戶訪談，記錄 location、expected、actual、impact、owner 與 disposition。
- Non-use 不自動等於功能無價值；必須以訪談確認是流程不需要、介面卡點、權限問題、資料不足或培訓問題。

## 7. Verification and acceptance

### 7.1 Pre-delivery deterministic verification

- Synthetic golden cases 覆蓋所有 approved case/target/task transitions、roles、scope/capability、expiry、exceptions 與 failure modes。
- Empty tenant 所有核心頁面通過 desktop/mobile、long/empty/loading/error/denied、keyboard/focus 與 representative a11y checks。
- Historical reconstruction 測試合法順序、非法跳階、future date、missing evidence、approved gap、self-approval denial、changes requested、concurrent update、replay、immutable activation 與 side-effect suppression。
- Negative authorization 覆蓋 UI、direct request、ID guessing、search、export、document、background job、cache 與 expired/disabled actors。
- Telemetry 測試 event schema、PII canary、retention、alert receipt、sink failure 與 mandatory-audit distinction。
- Migration、rollback、restore、RPO/RTO、HK residency、public exposure 與 secondary-copy inventory 通過。

### 7.2 Live pilot acceptance

- 每個 pilot case 與客戶來源逐案 reconciliation，unexplained difference 為零。
- 所有 history gaps 在 dashboard 可見，並有 owner、Founder decision 與處置狀態。
- 至少完成一個 Founder -> Primary Advisor -> Collaborator/Contractor -> approved Task 的真實委派閉環。
- Founder 無需透過通訊軟件追問，即可看見所有 pilot cases 的 current stage、next action、owner、deadline 與 exception。
- 所有 grants 按 expiry/revoke 即時失效，且沒有 cross-case exposure。
- PII telemetry scan、audit continuity、backup/restore reconciliation 與 operational alerts 通過。
- 使用指標與客戶訪談共同形成 signed `expand`、`hold` 或 `rollback` 決定。

## 8. 客戶必須提供的輸入

交付前不要求客戶把完整學生 case 或 PII 交給工程團隊，但仍需要以下輸入：

- Pilot User 名單、角色、邀請 channel 與 account lifecycle 批准。
- 5 至 10 個既有案件的 opaque references、目前階段，以及非識別化的 route、stage、exception 和 document-type 分布。
- 正式字段、transition、history evidence 與 history-gap disposition 的業務確認。
- 每案指定 Primary Advisor，以及不同 actor 的 Founder reviewer。
- Named Product、Operations、Security、Privacy 與 Data owners。
- 每日營運 review、每週訪談及 incident escalation 的可用時間。
- Product telemetry 告知、香港資料處理、retention、support access 與 rollback approval。

學生姓名、聯絡方式、文件及詳細歷史只由獲授權客戶使用者輸入 production。這些資料不得進入 Git、test fixtures、prompts、tickets、screenshots、一般 telemetry 或非香港支援路徑。

## 9. Assumptions and non-goals

- 首批 pilot 以既有案件為主，不是只接收新簽約案件。
- 客戶選擇人工重建完整歷史，不採 batch migration，也不只建立 current-state snapshot。
- Primary Advisor 負責錄入；Founder 作為不同 actor 批准。
- 缺少歷史證據不必然阻擋案件，但必須有明確 gap、reason、owner、Founder approval 與持續可見狀態。
- 監控採 event metrics 加結構化訪談，不採 session replay。
- Pilot 期間不引入長期 dual-write；客戶來源只用於逐案 reconstruction 與 reconciliation，權威切換以每案 activation receipt 為準。
- `R1X-00` 已把受限 Portal 與 aggregate-only PlatformBilling 加入 Release 1；本方案仍不擴大至 portal write/public registration/document download、external notification、AI、Excel/CSV import、tax invoice/payment/accounting、automated crawler sync 或 second tenant activation。
