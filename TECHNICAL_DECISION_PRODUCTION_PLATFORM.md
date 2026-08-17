# TDR-001：生產平台、身份、文件與 AI Agent 運行邊界

| 屬性 | 內容 |
| --- | --- |
| 文件類型 | Technical Decision Record（TDR） |
| 狀態 | AWS 香港敏感平面、Cloudflare 公開 release 與 `DEC-068` production source/plan baseline 已 `accepted`；Terraform plan/apply 和 exact execution payload 仍待另行批准 |
| 日期 | 2026-08-12（Asia/Hong_Kong） |
| 適用範圍 | 天星國際教育平台 Release 1 生產架構與後續 AI agent 基線 |
| 主要系統 | `erp-frontend/` Next.js 16.2.7 / React 19.2.4 |
| 決策依據 | `PRD.md`、`DEC-018`–`DEC-024`、`DEC-051`–`DEC-063`、`DEC-068`、現有 Terraform/runtime contracts |
| 研究證據 | [`docs/research/2026-08-11_PRODUCTION_HOSTING_AUTH_OPTIONS.md`](research/2026-08-11_PRODUCTION_HOSTING_AUTH_OPTIONS.md) |
| 後續設計 | [`TECHNICAL_DESIGN_AWS_CLOUDFLARE_PRODUCTION_DEPLOYMENT.md`](TECHNICAL_DESIGN_AWS_CLOUDFLARE_PRODUCTION_DEPLOYMENT.md)；`DEC-063` 取代原本未決 public-plane 選項並禁止 cutover 後 Vercel deployment |
| 覆核人 | Founder、Product、Security、Privacy、Data、Operations、Budget owner |

## 1. 問題與目標

Vercel Hobby 不允許商業使用，而本系統的 Next.js repository 同時包含 UI、BFF、Route Handlers 和敏感資料處理。技術選型必須回答的不只是「前端放在哪裡」，還包括：

1. authenticated runtime、資料庫、文件、queue、audit 和備份能否維持批准的香港資料邊界；
2. 多客戶情境下，身份、租戶、案件、文件和 background job 是否維持一致隔離；
3. 認證 provider 與應用程式授權是否保持清晰所有權；
4. 未來 AI agent 是否能在不放寬現有授權、資料主權和人工批准 gate 的前提下加入；
5. 遷移、營運複雜度、成本和退出路徑是否可驗證。

### 1.1 Stakeholder outcome

- Founder／Product：以最小遷移風險完成 Release 1 上線，並保留未來擴展空間。
- Advisor／Operations：登入、文件與案件工作流穩定，不因平台拆分而增加人工補救。
- Customer／Data subject：學生、家長、案件和文件資料不因 frontend hosting 便利而離開批准邊界。
- Security／Privacy：身份、授權、audit、telemetry 和 support access 有單一可驗證 enforcement owner。
- Engineering：沿用現有 AWS/IaC/module contracts，不同時維護多套 runtime、session 和 cloud control plane。

### 1.2 不在本決策內

- 不批准任何 AWS、Azure、Cloudflare、Vercel 或身份供應商採購。
- 不批准 Terraform apply、production migration、資料寫入、帳號邀請、DNS 切換或 deployment。
- 不批准把任何真實案件、文件或 PII 傳送到 AI model/provider。
- 不解決 `DEC-060` 的第二訂閱組織、retention、support grant、termination/export 和 region outage 語義。

## 2. 決策

### 2.1 Authenticated production plane

**建議把完整 authenticated Next.js 16 application/BFF 部署到 AWS Hong Kong `ap-east-1`，使用 ALB + ECS Fargate，並把 runtime tasks 放在 private subnets。**

```text
Browser
  -> DNS / TLS / WAF
  -> ALB, ap-east-1
  -> ECS Fargate Next.js web/BFF tasks, private subnets, >= 2 AZs
       -> Cognito User Pool, ap-east-1
       -> private RDS PostgreSQL, ap-east-1
       -> private S3 + KMS, ap-east-1
       -> SQS/DLQ + scanner/worker tasks, ap-east-1
       -> Hong Kong audit/log/alert stores
```

現有 staging Terraform 已採 `ap-east-1` provider、private runtime、RDS 和香港 KMS/ACM validation；因此這是把已接受的架構推進至 production IaC，而不是另建一套平台。

### 2.2 Public plane

`DEC-063` 已批准把經審核的全港學校招生公開 release 分發於 Cloudflare R2。只有經 public field allowlist、PII/licence review、schema/count/SHA-256/read-back gate 的 immutable records 和 sanitized manifest 可發布；review/fallback queue、source capture、tickets/config/review decisions、run/log/LLM trace 和 tenant data 仍留在 AWS control plane。

`data.<domain>` 使用 R2 custom domain/proxy；`app.<domain>` 保持 Cloudflare DNS-only 並直達 AWS。Cloudflare 不得接收 cookie、authenticated request、PII、ERP private response、preview data、Server Action 或敏感 log。Vercel 只作 migration inventory，AWS cutover 後不再部署或作 rollback target。完整發布、account/DNS、Vercel migration 和 AWS-only rollback contract 以 `TECHNICAL_DESIGN_AWS_CLOUDFLARE_PRODUCTION_DEPLOYMENT.md` 為準。

### 2.3 Identity and authorization

**保留 Cognito `ap-east-1` 作 managed identity provider，不以 Auth.js 取代。**

- Cognito 負責 user authentication、TOTP enrollment/challenge、credential recovery 和 provider-level revocation。
- opaque BFF cookie/session 維持現有 application session boundary。
- RDS 是 `organization_id`、membership、role、case scope、capability、expiry 和 `session_version` 的唯一業務授權來源。
- email 是可變 contact attribute，不是 identity key；內部 identity 使用 `(provider, provider_subject)`。
- Cognito group、JWT claim、Auth.js callback 或 browser-provided tenant ID 不得成為業務授權真相。
- provider token 被撤銷後仍可能通過離線 JWT signature/expiry 驗證，因此 request-time RDS `session_version`／membership check 不得移除。
- 不建立需要 `us-east-1` ACM certificate 和 global CloudFront 的 Cognito custom login domain；使用經驗證的 `ap-east-1` regional/provider domain。若 data-flow probe 不符合香港邊界，authentication launch blocked，不 fallback 區外 AWS service。
- 未來 Microsoft-centric 客戶可經批准的 OIDC/SAML federation 接入同一 identity adapter；不得用 Entra role 直接取代案件授權。

Auth.js 是 application auth/session library，不是 managed IdP。現在加入它會重疊 opaque BFF session ownership，且若採 Credentials provider，密碼、MFA、reset、anti-abuse 和 audit 責任會轉回應用團隊，沒有消除 Cognito 所解決的問題。

### 2.4 Documents and asynchronous processing

- 文件 bytes 存 private S3 `ap-east-1`，使用 KMS encryption、versioning 和明確 retention。
- 文件 metadata、版本、tenant/case linkage、scan state 和 authorization 存 RDS。
- browser 只取得短效、單一 object/version、單一 action 的 upload/download intent，不取得 bucket credential。
- 新 upload 先進入 quarantine；SQS/DLQ 驅動 scanner/OCR worker，通過後才可轉為 available。
- scanner、OCR、thumbnail、index 和 notification 是可重試副作用；權威 document state、audit 和 outbox 必須維持 transaction/reconciliation contract。
- object key、queue message、cache、search/index 和 export 全部帶 opaque organization scope；不得把原始文件名、學生姓名或自由文字放進 key、log 或 telemetry。

### 2.5 Multi-customer boundary

選擇 AWS 不會自動令系統成為安全的 multi-tenant SaaS。第二個客戶組織啟用前仍必須完成 `DEC-060`，並提供以下證據：

- shared-schema tables 的 mandatory `organization_id`、foreign-key locality 和 RLS；
- unprivileged application DB role，不能繞過 RLS；
- UI、direct API、ID guessing、search、export、S3、background job 和 cache 的跨租戶負向測試；
- support grant、retention、legal hold、termination、export、backup expiry 和 purge 的批准語義；
- customer-specific identity federation 仍映射至 server-side `OrganizationMembership`。

在這些 gate 完成前，production 只允許已批准的第一個 empty tenant/pilot organization。

### 2.6 Future native AI agents

初始 agent orchestration 和 tool workers 建議以 ECS Fargate/container worker 形式運行於 `ap-east-1`，而不是因某個 managed agent product 改變 transactional system 的所在地。

```text
User command
  -> BFF authentication + fresh RDS authorization
  -> AgentRun(scope, policy/version, budget, expiry, idempotency)
  -> HK queue
  -> isolated HK agent worker
       -> approved model adapter
       -> versioned tool gateway
            -> authorized read tools
            -> separately approved mutation tools
  -> append-only redacted audit/receipts
  -> human approval for high-impact actions
```

必須維持以下不變量：

- agent runtime residency 與 model inference residency 分開批准；香港 worker 呼叫海外 model 仍屬資料出境。
- AgentRun 綁定 organization/case scope，不繼承操作者的所有 ambient access。
- prompt、model、tool schema、policy、knowledge snapshot、evaluator 和 cost budget 都需版本化。
- agent mutation 仍走普通 application command：authorization、idempotency、audit transaction、outbox、bounded retry 和 reconciliation。
- 高影響操作需 exact-payload human approval；agent 無權自行擴張工具或資料範圍。
- model、embedding、OCR 和 agent framework 均置於 provider adapter 後，避免把業務模型綁死於單一供應商。

## 3. 未採用方案與理由

| 方案 | 本次結論 | 主要原因 | 重新評估條件 |
| --- | --- | --- | --- |
| Vercel Hobby | 不採用於商業 production | 官方限個人／非商業；也不能提供完整敏感資料邊界 | 不重新評估 Hobby；public plane 使用 Pro 或其他商業方案 |
| Vercel Pro 全站 | 不採用；AWS cutover 後退場 | 增加第二個 server/log/support boundary，不能取代 RDS/S3/SQS；`DEC-063` 明確禁止後續 Vercel deployment/standby/rollback | 無排定重評；只有新的使用者決策才可重開 |
| Cloudflare Workers 核心 ERP | 不採用 | `workerd`/OpenNext runtime 差異；HK localization 為 Enterprise；subrequest/Queues/Cron 與 R2 jurisdiction 存在邊界缺口 | localization 覆蓋所有 execution/state/log/support，R2 有 HK jurisdiction，完整 Next.js production matrix 通過 |
| Azure East Asia | 不遷移 | 功能可行，但需重寫 IaC、identity、storage events、logging、restore 和 evidence；現有資料邊界證據不足以證明所有服務 HK-only | 主要客戶要求 Microsoft 生態，取得 service-specific HK evidence，且 costed migration/rollback 優於 AWS |
| AWS Amplify Hosting | 不採用 | 現行文件未覆蓋本 repository 的 Next.js 16 要求，部分 SSR feature/region behavior 不符合邊界 | Amplify 正式支援所需 Next.js 版本、features 和 HK-only processing |
| Auth.js 取代 Cognito | 不採用 | 產品類別不同；重複 session ownership 或把 credential/MFA burden 移回 application | 它能消除而非重複現有邊界，production maturity 和 migration/exit evidence 通過 |
| Managed agent runtime 決定主雲 | 不採用 | 現有 AgentCore/Foundry hosted agent region availability 不能證明 HK processing | managed runtime、model、state、logs、support 全部符合批准的資料分類與香港邊界 |

## 4. 非功能要求與 enforcement owner

| 要求 | 最低基線 | Enforcement owner | Release evidence |
| --- | --- | --- | --- |
| Data residency | PII、文件、authorization、audit、logs、backup 和 support trace 留在批准的香港邊界 | Infrastructure + Privacy | Terraform plan、region probes、service configuration inventory、DPA review |
| Availability | 至少兩個 AZ 的 web tasks；DB RPO 5 分鐘、RTO 4 小時作 measured target | Infrastructure + Operations | failover/restore drill、health and alert receipts |
| Authorization | 每個 request 重新驗證 membership、scope、expiry、session version；fail closed | Access/Identity modules | negative authz、revocation、cross-tenant tests |
| Documents | private encrypted object、quarantine-first、versioned metadata、no public bucket | Documents module + Infrastructure | bucket policy/KMS probe、scan-state and replay tests |
| Audit | mutation-bound append-only audit；audit persistence failure 時 mutation fail closed | AuditOperations | transaction/failure-injection evidence |
| Telemetry | `DEC-062` allowlist、30 日、無 PII；sink failure fail open into degraded state | AuditOperations + Privacy | schema/PII rejection、retention、outage evidence |
| Performance | 一般 API P95 < 500 ms、主要頁面 P95 < 2 s，未測量前不是 SLA | Application + Operations | pinned-build load report |
| Deployment | build once/promote by immutable digest；multi-instance keys/cache/build ID 一致 | Release + Infrastructure | Git SHA、image digest、SBOM、config checksum、canary/rollback receipt |
| Cost | exact calculator export、budget alert 和 stop owner；不得以估算代替批准 | Budget owner | dated calculator/export and alert probe |

## 5. Production deployment 所需輸入及取得位置

所有 secret 應直接進入 AWS Secrets Manager、SSM 或受批准 CI secret store，不得貼進聊天、Git、Terraform plan artifact 或 release evidence。

| 輸入 | 取得／批准位置 | 可保存的 release evidence |
| --- | --- | --- |
| Production AWS account ID、deploy role、break-glass owners | AWS Organizations／IAM Identity Center，由 Cloud/Security owner 提供 | opaque account ref、role ARN、owner、approval ID；不保存 credential |
| `ap-east-1` AZ、VPC CIDR、private/public subnet plan | AWS account inventory + reviewed Terraform variables | redacted network diagram、saved binary `.tfplan` SHA-256、policy check |
| Terraform state bucket/key/locking/KMS | Infrastructure owner 建立並批准的 backend | backend opaque ref、KMS ARN、versioning/lock probes |
| Domain ownership和 DNS change window | Domain registrar／Route 53 owner | zone ID、change ticket、TTL/rollback plan |
| ACM certificate ARN | ACM `ap-east-1`，經 domain validation | ARN、status、expiry monitor；不保存 DNS secret |
| Monthly budget、alerts、cost owner | AWS Billing/Budgets + Founder/Finance | dated calculator、thresholds、recipients/owner receipt |
| Git SHA、container digest、SBOM、scan result | approved CI + ECR | immutable digest、scanner checksum、build provenance |
| Cognito pool/client/domain、callback origins、TOTP policy | Cognito `ap-east-1` + Identity/Security review | resource IDs、allowlisted origins、policy checksum；不保存 client secret |
| RDS engine/minor、DB name、backup/retention、migration payload | production Terraform + Data owner | saved binary `.tfplan` SHA-256、migration checksum、backup/restore receipt；不保存 password/URL |
| S3 buckets、KMS keys、retention、CORS、scanner/OCR path | production Terraform + Documents/Privacy owner | policy hashes、region/encryption/public-access probes |
| WAF/rate limit、log redaction、alerts/runbooks | WAF/CloudWatch/CloudTrail config + Operations/Security | rule/config checksum、synthetic alert receipt |
| First 1–3 pilot references、users、reviewers、stop conditions | `P3-19` human go/no-go | opaque refs and signed approval only；Git 不保存 PII |

## 6. 成本基線

2026-08-11 的研究估算，在兩個常駐 ARM Fargate tasks（每個 1 vCPU/2 GiB）、一個 ALB、RDS PostgreSQL Multi-AZ `db.t4g.small + 20 GiB gp3` 和 100-case S3/KMS 假設下：

| Component | 月度規劃值（USD） |
| --- | ---: |
| ECS Fargate | 約 79.18 |
| ALB + 平均 1 LCU | 約 26.64 |
| RDS PostgreSQL Multi-AZ + 20 GiB | 約 87.03 |
| S3/KMS 文件情境 | 約 1.71–6.57 |
| Cognito direct/social MAU | pilot 規模在當期免費額內時為 0 |
| **Core subtotal** | **約 194.56–199.42** |

這不是 production budget。正式預算必須另外計入 WAF、CloudWatch/CloudTrail、Route 53、Secrets Manager/KMS keys、ECR、scanner/OCR、SQS、額外 backup、network egress、support 和稅。`DEC-068` 同時要求每 AZ 一個 NAT Gateway 與 ECR/S3/Logs/SQS/KMS/Secrets Manager/STS endpoints；成本 evidence 必須同時包含兩者，不得以二選一估算取代批准 baseline。

## 7. 部署順序與批准 gate

本決策如獲批准，只固定 architecture direction；仍按 Phase 3 ticket dependency 執行：

1. `P3-01`–`P3-06`：完成 synthetic、reconstruction 和 telemetry contracts/evidence。
2. `P3-07`：依 `DEC-068` 建立 private two-AZ ECS、one-NAT-per-AZ、required endpoints、fixed ECS/RDS/WAF baseline 的 production IaC source、static policy checks 和 evidence contract；不得執行 Terraform plan/apply。
3. `P3-07A`：只有在 P3-07 source review 通過且取得另行 exact production payload/tooling approval 後，才可執行 saved `terraform plan -out`，保存 binary `.tfplan` SHA-256 和分離的 redacted summary/review evidence；不得 apply。
4. `P3-08`–`P3-09`：完成 production repositories/effect adapters 和 focused evidence；不得接觸 production resources。
5. `P3-10`：只可在另行 exact-payload apply 批准後 apply P3-07A 保存且 SHA-256 完全相同的 binary `.tfplan`；redacted JSON、provider lock 與 source hashes 不能取代該 identity。
6. `P3-11`：只可執行已批准且 checksum 完全相同的 migration payload。
7. `P3-12`：只可部署已批准的 Git SHA/container digest。
8. `P3-13`–`P3-18`：依序完成 identity invite、browser、telemetry、安全、rollback 和 restore gates。
9. `P3-19`：由人工簽署首批 1–3 個 opaque case references；no-go 保持 empty tenant。
10. Phase 4：逐案、順序執行 reconstruction/activation/observation；不得把 pilot 當成開發環境。

任何 plan、migration、image、domain、account、region、data flow 或 cohort payload 變更，都使原批准失效，必須重新產生 evidence 和批准。

## 8. Failure modes、停止條件與回滾

- region、account、public/private、encryption 或 data-flow probe 與批准 payload 不一致：停止，不 apply／不 deploy。
- runtime 無法取得 RDS、Cognito、S3 或 audit dependency：依 contract fail closed；不得靜默切回 mock、local JSON、Neon 或區外服務。
- telemetry sink 不可用：進入 `DEC-062` degraded state，業務可 fail open；mandatory audit 不可用時 mutation fail closed。
- migration checksum/schema drift 不一致：停止 migration，保留 evidence；不得以臨時 SQL 修補 production。
- deploy health/canary 不通過：停止流量切換，回到已知 container digest；DB rollback 只按已批准 forward-fix/restore path。
- suspected cross-tenant、PII、public bucket 或 cross-region exposure：停止 pilot，撤銷 session/access，保存 redacted evidence，啟動 incident runbook。
- 同一 deterministic failure 修正後仍重複兩次且沒有新 evidence：terminal state 為 `needs_human`，不得盲目重試。

## 9. Consequences

### 9.1 正面

- 敏感 runtime、身份、資料庫、文件、queue 和 audit 收斂於一個香港 cloud control plane。
- 最大程度沿用現有 Terraform、repository adapters、Cognito 和 AWS document contracts。
- 不需要同時驗證 Node 與 edge runtime 差異，也不增加第二個 session abstraction。
- AI agent 可在不改動 transactional system ownership 的情況下逐步加入。
- public marketing plane 仍保留日後獨立優化或更換供應商的自由。

### 9.2 代價與殘餘風險

- 團隊需承擔 ECS/ALB、multi-instance Next.js、WAF、observability、capacity 和 patch operations。
- AWS `ap-east-1` 價格通常高於部分主要區域；仍需 exact calculator 和 budget alerts。
- Cognito optional features、email/SMS、support/log paths 需逐項證明不造成未批准 cross-region processing。
- 單一香港 region 策略須接受批准的停機／manual fallback；不能自行增加區外 replica。
- shared-schema multi-tenancy 的安全仍取決於 application policy、RLS 和負向測試，而非 cloud vendor。
- AI model inference 的 residency、retention 和 training policy 尚未決定。

## 10. 覆核觸發條件

出現以下任一情況時重新評估本 TDR：

- 客戶合約要求 Entra、Microsoft 365/Graph 或指定 cloud/compliance package；
- AWS 香港服務、價格、region behavior 或 Cognito MFA/residency 發生重大變更；
- Cloudflare 能以可接受合約覆蓋 Workers execution、subrequests、Queues/Cron、logs/metadata、Durable Objects、R2、support 和 backup 的香港邊界；
- managed agent runtime/model 在香港提供完整 state、trace、support 和 data-processing evidence；
- 實測 workload 超出 Fargate/RDS 架構的 latency、throughput、availability 或成本門檻；
- `DEC-060` 批准的 multi-customer/retention/termination 語義要求不同 isolation topology。

## 11. 官方來源

- [Vercel Hobby plan](https://vercel.com/docs/plans/hobby)；[Vercel pricing](https://vercel.com/pricing)
- [Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting)
- [AWS Regions](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions.html)；[Fargate regions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate-Regions.html)
- [Cognito TOTP](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-mfa-totp.html)；[Cognito token revocation](https://docs.aws.amazon.com/cognito/latest/developerguide/token-revocation.html)；[Cognito pricing](https://aws.amazon.com/cognito/pricing/)
- [Auth.js Credentials provider](https://authjs.dev/getting-started/providers/credentials)；[Auth.js session strategies](https://authjs.dev/concepts/session-strategies)；[Auth.js security](https://authjs.dev/security)
- [Cloudflare Next.js](https://developers.cloudflare.com/workers/framework-guides/web-apps/nextjs/)；[Data Localization](https://developers.cloudflare.com/data-localization/)；[R2 data location](https://developers.cloudflare.com/r2/reference/data-location/)
- [Microsoft Entra External ID pricing](https://learn.microsoft.com/en-us/entra/external-id/external-identities-pricing)；[Azure Foundry hosted-agent regions](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents)
- [Amazon Bedrock AgentCore regions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-regions.html)

## 12. 批准欄

| Decision | Owner | Status | Date / evidence |
| --- | --- | --- | --- |
| AWS `ap-east-1` 為 authenticated production plane | User / Founder | `accepted` | `DEC-021`；exact ECS/ALB plan/deploy pending |
| Cognito `ap-east-1` + opaque BFF session + RDS authorization | User / Founder / Security | `accepted` | `DEC-020,063`；regional/provider domain probe pending |
| S3/KMS/SQS/scanner 文件邊界 | Data / Privacy / Security | `accepted` | `DEC-024`；exact provision/deploy pending |
| Cloudflare R2 只承載已批准公開學校招生 release；AWS 保留全部 tenant/personal data；Vercel 退場 | User / Product / Privacy | `accepted` | `DEC-063`，2026-08-11；upload/DNS/deploy 仍待 exact-payload 批准 |
| Production AWS source/plan baseline 與 binary plan identity | User / Founder / Security / Operations | `accepted` | `DEC-068`，2026-08-12；Terraform plan/apply/provision 仍未授權 |
| 初始 AI agent orchestration 使用 HK self-hosted workers | User / Product / Privacy / Security | `pending` |  |
| 月度成本上限和告警 owner | Founder / Finance / Operations | `pending` |  |

`DEC-068` 已把 source/plan baseline 升格為實施權威；任何 Terraform plan/apply、production action 或 payload 變更仍需獨立 exact-payload 批准。
