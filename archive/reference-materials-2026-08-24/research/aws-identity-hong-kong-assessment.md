# 香港資料駐留下的身份服務評估

> 查核日期：2026-07-31（Asia/Hong_Kong）  
> 範圍：Release 1 內部單一組織、邀請制 Advisor/Founder/Admin 帳號；所有 Student、Guardian、ServiceCase、身份、會話、審計、日誌和備份資料必須駐留香港。  
> 研究規則：只把第一方官方文件、官方價格表和官方 API 參考作為證據。服務商的「全球 edge/PoP」或「可選亞太地區」不等同於香港資料駐留。

## 結論

### 建議採用

**AWS Amazon Cognito User Pools `ap-east-1` + 香港 RDS 業務授權/會話層**，但只採用不產生區外訊息副作用的功能子集：

1. 建立 Cognito user pool 和 app client 於 `ap-east-1`，使用 regional endpoint `cognito-idp.ap-east-1.amazonaws.com`。
2. 由香港 API/BFF 呼叫 Cognito；Cognito 只證明 actor 身份。Primary Advisor、CaseCollaborator、scope、capability、案件狀態和所有業務授權仍由香港 RDS 強制。
3. 不使用 Cognito Hosted UI 作為前端 token 儲存機制；BFF 將 Cognito refresh/access token 加密保存於香港資料庫，瀏覽器只收到沒有 PII 的短期 opaque session cookie。
4. 使用 TOTP authenticator app（或未來完成區域和供應商審核後的 passkey）作強制 MFA；Release 1 不啟用 SMS MFA、Email MFA、Cognito 自動邀請、Cognito 自助密碼重設或其他需要 AWS 郵件/短信的流程。
5. 邀請使用 `AdminCreateUser` 的 `MessageAction=SUPPRESS`，在香港 RDS 建立一次性 invite record/hash；先由 Founder/Primary Advisor 透過已批准的香港通信流程傳遞啟用資訊。任何交易郵件提供商都要另做香港駐留審核。
6. 應用自身在香港寫入登入、登出、拒絕、MFA 和管理操作審計。CloudTrail/CloudWatch 僅作 AWS 控制面和可用的安全事件補充，不能取代應用審計。

這個限制不是只由 endpoint 推論：[Cognito regional data considerations](https://docs.aws.amazon.com/cognito/latest/developerguide/security-cognito-regional-data-considerations.html) 明確說 user profile data 可只存於建立 user pool 的 Region，但選用的 optional feature 可能把 user data 送到另一 Region。Release 1 必須將 Pinpoint analytics 關閉，並把所有會發送 email/SMS 的 feature 當作未批准的跨區資料流。

這是**技術選型建議，不是建立 AWS 資源、採購或上線授權**。在建立 user pool 前，需取得 AWS 合約/DPA、服務條款和 Cognito 郵件依賴的法務確認，並完成一個香港區域的 data-flow/restore/delete 演練。

### 為何不是直接採用自建 Keycloak

Keycloak 是可行的香港自建 fallback，因為其 user、realm、session、event 和資料庫都可以部署在 `ap-east-1`；但需要自行承擔身份服務的高風險運維：版本升級、漏洞修補、HA、session/key rotation、郵件、備份恢復、admin audit、容量和退出遷移。Release 1 只有 30 個 dry-run、5–10 個 pilot，Cognito 的 regional endpoint 和成熟的 MFA/token lifecycle 能以較少自建程式滿足需求，故先採 Cognito，保留 Keycloak 作供應商退出或 Cognito 區域政策改變時的受控替代方案。

## 第一方證據

| 主張 | 官方來源 | 查核結果／不確定性 |
| --- | --- | --- |
| Cognito User Pools 在香港有真正的 regional API endpoint | [AWS General Reference：Cognito endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/cognito.html)；頁面列出 `Asia Pacific (Hong Kong) / ap-east-1 / cognito-idp.ap-east-1.amazonaws.com` | 已核實（2026-07-31）。這證明服務端點存在，不單是 edge/PoP；仍需在目標帳號用 `CreateUserPool` 做一次非生產 smoke test。 |
| Cognito User Pools 是 regional service；API 額度表以「Each supported Region」列出 | 同一 [endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/cognito.html) | 已核實。API regionality 不等同於所有附加依賴（例如 email）都在同區。 |
| AWS 的一般承諾是客戶選擇內容儲存 Region；不會在未同意下移動/複製至所選區域外，但服務所需或法律要求除外 | [AWS Data Privacy FAQ](https://aws.amazon.com/compliance/data-privacy-faq/)（Where is customer content stored? / Storage） | 已核實。這是 AWS 一般承諾，不是 Cognito 專屬香港法律意見；需把服務啟動的跨區處理例外逐項審核。 |
| AWS DPA 把 AWS 定義為 processor，提供服務控制、資料主體請求協助、subprocessor 通知/異議、刪除/更正控制 | [AWS Data Processing Addendum (PDF)](https://d1.awsstatic.com/legal/aws-gdpr/AWS_GDPR_DPA.pdf)，§1、§6、§7 | 已核實（文件查核 2026-07-31）。DPA 是一般資料保護條款，不保證 Cognito 所有內部備份的即時刪除；香港 PDPO 適用性和合同版本需由法務確認。 |
| Cognito 的 user profile data 預設可只存於 user pool Region，但 optional features 可能跨區；Pinpoint analytics event data 會路由到 `us-east-1` | [Cognito regional data considerations](https://docs.aws.amazon.com/cognito/latest/developerguide/security-cognito-regional-data-considerations.html) | 已核實。這是服務專屬證據；故禁止 Pinpoint analytics，並要求逐項審核所有 optional feature。 |
| Cognito at-rest data 可由 AWS-owned key 或同區 customer-managed symmetric KMS key 加密 | [Cognito data protection](https://docs.aws.amazon.com/cognito/latest/developerguide/data-protection.html) | 已核實。建議 `ap-east-1` single-Region CMK；KMS key rotation、key deletion 和 recovery window 要納入 IaC/restore runbook。 |
| Hong Kong user pool 的 SES alternate regions 是 Singapore 和 Tokyo | [Cognito email configuration and SES Regions](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-email.html)；表格列 `Asia Pacific (Hong Kong)` 為 `Alternate Region`，可用 `Asia Pacific (Singapore), Asia Pacific (Tokyo)` | 已核實。這是本方案最大的駐留限制：啟用 Cognito/SES 郵件可能使訊息內容、收件地址、事件或服務 metadata 接觸區外。Release 1 必須 suppress 這些流程，除非另有書面批准。 |
| Admin invitation 可抑制自動訊息 | [AdminCreateUser API](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminCreateUser.html)，`MessageAction=SUPPRESS` | 已核實。密碼/啟用資料由應用的香港流程產生和保存，禁止把原始秘密寫入 audit/log。 |
| Cognito 支援 authenticator app MFA；Email/SMS MFA 需要訊息通道 | [User pool MFA](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-mfa.html)；[Cognito message settings](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-email.html) | 已核實。TOTP 不依賴 AWS email/SES；SMS/Email 需另行做跨區和電信供應商審核。 |
| Cognito 有 token revocation API，但自行驗證 JWT 的 API 不會自動知道每一個已撤銷 token | [Token revocation](https://docs.aws.amazon.com/cognito/latest/developerguide/token-revocation.html)；[AdminUserGlobalSignOut API](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminUserGlobalSignOut.html) | 已核實 API 能力；實施不變量由 BFF 強制：短 access-token TTL、伺服端 token introspection/refresh 控制、`session_version` 和 deny/revoke record。不可只在瀏覽器刪 cookie。 |
| User pool admin API calls 可送入 CloudTrail；應用登入審計仍需自行保存 | [Cognito CloudTrail logging](https://docs.aws.amazon.com/cognito/latest/developerguide/logging-using-cloudtrail.html) | 已核實 CloudTrail 範圍是 AWS API activity。CloudTrail trail、CloudWatch log group、S3 archive 必須明確建立在 `ap-east-1`；Cognito Plus 的 threat/auth event export 不是 Release 1 的唯一審計來源。 |
| User profiles 可透過 `ListUsers`/`AdminGetUser` 讀取，密碼不能匯出 | [ListUsers API](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_ListUsers.html)；[AdminGetUser API](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminGetUser.html)；[User import](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-using-import-tool.html) | 已核實 API 只提供使用者屬性；沒有受客戶控制的 password-hash export/restore。退出時需強制所有使用者重設密碼或以 migration trigger 分批遷移，不能宣稱「完整可攜出」。 |
| 可刪除 user/user pool，但服務備份保留和傳播時間不是 user API 的可配置契約 | [AdminDeleteUser](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminDeleteUser.html)；[DeleteUserPool](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_DeleteUserPool.html) | 已核實刪除 API；Cognito 服務內部 backup/retention 的逐項保證未在上述 API 文件中找到，故需在 DPA/Support 與法律保留政策中確認，並以應用 tombstone/刪除驗證補強。 |
| AWS 的 Cognito user-profile export reference architecture 使用 Step Functions + DynamoDB Global Table，明確不 export passwords；其跨區 backup 不符合本項目香港-only 要求 | [Cognito user profiles export reference architecture](https://docs.aws.amazon.com/solutions/latest/cognito-user-profiles-export-reference-architecture/overview.html) | 已核實。Release 1 不部署該方案；如需出口，只做香港 ListUsers snapshot（不含密碼/MFA）並安排強制重設。 |
| IAM Identity Center 也有 `sso.ap-east-1.amazonaws.com` endpoint | [AWS General Reference：IAM Identity Center endpoints](https://docs.aws.amazon.com/general/latest/gr/sso.html) | 已核實 endpoint 存在，但它是 AWS workforce/account access SSO，不是適合本產品的 customer/application user pool；不能用它替代 Cognito 的 user lifecycle 和 app user attributes。 |

## 功能與控制評估

| 要求 | Cognito `ap-east-1` | 香港自建 Keycloak | Release 1 取捨 |
| --- | --- | --- | --- |
| MFA | TOTP/authenticator app；SMS/Email 依賴訊息服務 | TOTP、WebAuthn、可自行接入短信/郵件 | 強制 TOTP；先不啟用 SMS/Email。 |
| Invite | `AdminCreateUser`、可 `SUPPRESS` | Admin REST/CLI + 自建 invite service | invite token 只在香港 RDS 存 hash；應用負責過期、一次性和 audit。 |
| Session revoke | `AdminUserGlobalSignOut`、`RevokeToken`；BFF 還需 `session_version` | Realm/user session revocation；仍需應用 session store | disable user、provider revoke、DB session revoke 必須同一個可重試 workflow。 |
| 審計 | CloudTrail API logs；Plus/advanced security 可增加 auth events | Admin/user events、可導出 log；由團隊運維 | 應用 RDS audit 是業務真相；CloudTrail/CloudWatch 是補充。 |
| 匯出／刪除 | 屬性可讀出；密碼 hash 不可匯出；user 可刪除 | realm/user export 和 PostgreSQL backup 較完整 | Cognito 退出 runbook 必須包含強制重設密碼；Keycloak 只作 fallback。 |
| DPA／子處理者 | AWS DPA、subprocessor 機制；需審核 Cognito/SES service-specific 例外 | AWS 為底層 hosting processor，Keycloak 專案本身不是托管商 | 對所有接觸資料的 AWS service 建立 subprocessor/region 清單；不把開源軟件等同 DPA。 |
| 故障與升級 | AWS managed control plane；區域故障時不可自動 failover 到區外 | 自行處理兩個 task、ALB、DB HA、升級和演練 | Cognito 故障時 fail closed；BFF 不繞過業務授權。 |

## 建議資料流

```text
Browser
  -> 香港 HTTPS ingress / API-BFF（ap-east-1）
  -> Cognito user pool endpoint（ap-east-1；只作 authentication）
  -> BFF 驗證 token、查詢/更新香港 RDS User + session + session_version
  -> 香港 RDS policy engine（role/case/scope/capability）
  -> business API / audit transaction（同一香港資料庫交易邊界）

Cognito admin API/CloudTrail -> 香港 CloudTrail trail / log archive
Application auth + authorization events -> 香港 RDS audit + 香港 CloudWatch Logs
```

### 不變量與故障處理

- 所有 provider identity 以 `(provider='cognito-ap-east-1', provider_subject)` 唯一；email 不作外鍵。
- `User.status=disabled`、`session_version` 遞增和 RDS session revoke 必須在同一個業務命令完成；Cognito revoke 是可重試的外部 effect，使用 idempotency key 和 reconciliation job。
- BFF 不接受只由前端提供的 role/scope；每次案件讀寫由香港 RDS policy query 決定。所有跨案件、identity_contact、internal_notes 和 export 都以負向測試證明。
- Cognito/CloudTrail/CloudWatch 任一依賴不可用時，禁止發出新 session 或升級 scope；現有 opaque session 依 `expires_at` 和 `session_version` 受限，服務進入 `degraded-read` 或 `fail-closed` 明確狀態。
- Cognito 郵件/SMS 不可被 retry loop 意外啟用。若未來需要，先建立 provider/region evidence、message redaction、DPA 和香港通信供應商批准，再開 feature flag。

## 自建 Keycloak 的香港部署方案（fallback）

### 架構

- 在 `ap-east-1` 的兩個可用區部署至少兩個 Keycloak Quarkus container task，前置香港 ALB；不使用 public admin endpoint。
- 使用現有 RDS Multi-AZ PostgreSQL 但最好獨立 database/role；若共用 instance，至少以 DB role、network policy、backup/restore ownership 隔離。大於 pilot 後應量測並考慮獨立 instance。
- Keycloak realm、user profile、session、offline session、event 和 admin event 全在香港 PostgreSQL；Keycloak container、ALB、Secrets Manager、CloudWatch、ECR image cache 和 backup 都鎖定 `ap-east-1`。
- 應用仍以 BFF opaque session 為外部契約；Keycloak OIDC token 不直接成為前端業務授權。

官方設計和運維入口：[Keycloak High Availability introduction](https://www.keycloak.org/high-availability/introduction)、[Keycloak container guide](https://www.keycloak.org/server/containers)、[Keycloak server administration](https://www.keycloak.org/docs/latest/server_admin/)、[Keycloak license/source](https://github.com/keycloak/keycloak)。這些文件證明可自行運行和配置，不代表 Red Hat 或 AWS 代為提供香港托管 SLA。

### 官價量級（僅自建成本估算）

以 AWS 公開價格表查核（2026-07-31，USD，730 小時/月）作可重現估算：

- ECS Fargate ARM：每 task 1 vCPU + 2 GiB，香港價 `0.04449/vCPU-hour + 0.00487/GB-hour`。兩個 task 約 `US$79.18/月`。
- Application Load Balancer：`US$0.0277/ALB-hour`，約 `US$20.22/月`；低流量另按 LCU `US$0.0088/LCU-hour`，假設平均 1 LCU 約 `US$6.42/月`。
- 以上約 `US$105.82/月`，不包括 Fargate ephemeral storage、ECR、CloudWatch、Secrets Manager、VPC endpoints/NAT、流量、Support、稅、工程維運和 RDS。若另用一個 RDS Multi-AZ `db.t4g.small + 20 GiB gp3`，沿用 [PostgreSQL 研究](aws-postgresql-hong-kong-assessment.md) 的約 `US$87.03/月`，合計約 `US$192.85/月`。
- 來源：[AWS public price list Amazon ECS `ap-east-1`](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonECS/current/ap-east-1/index.json)、[AWS public price list AWSELB `ap-east-1`](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSELB/current/ap-east-1/index.json)。價格是時間點估算，不含稅和承諾折扣；正式採購前需重新下載並保留原始 JSON checksum。

### 自建風險

1. Keycloak admin/realm 設定和 DB schema 版本必須一起升級；未做 restore drill 不能升級生產。
2. 兩個 task 不等於可用性保證：ALB health、資料庫 failover、cache/session replication、key rotation 和 email/MFA provider 的部分失敗都要有測試。
3. Realm export 能協助配置退出，但密碼、active session、client secret、外部 IdP secret 和事件保留仍需單獨處理；不能把 realm JSON 當作完整 backup。
4. 開源專案不提供本產品的 DPA、香港駐留承諾或運維 SLA；這些責任由天星和底層 AWS 合約承擔。

## Managed provider 審查邊界

已核實的真正香港區域 managed identity 選項是 **AWS Cognito User Pools**（regional endpoint 證據如上）。AWS IAM Identity Center 也有香港 endpoint，但其產品定位是 AWS workforce/account access，非本產品的 application user lifecycle，故不選。

對 Auth0、Clerk、WorkOS、Okta 等常見 SaaS，本輪沒有找到能同時以第一方文件證明「user profile、session、audit、backup 全部可選香港 at-rest region」的資料；因此不把其「亞太可用」、「edge 全球化」或「region selectable」說法作為符合條件的證據。這不是對所有供應商的永久否定；若未來候選商能提供明確香港 region、跨區處理清單、DPA、刪除/匯出承諾和子處理者清單，才可重新進入採購評估。

## 需要批准的下一步

1. **DEC-020（建議）**：批准 Cognito `ap-east-1` 作 Release 1 identity provider，但採上述「無 AWS 郵件/SMS、TOTP、BFF opaque session、RDS 業務授權」限制。
2. 明確批准或拒絕「Founder/Primary Advisor 以已批准香港通信流程傳送 invite activation secret」；在批准 HK transactional email provider 前，不實作自動郵件。
3. 批准一個非生產 smoke test（建立/邀請/登入/TOTP/revoke/delete），只使用 synthetic data，測試完成後刪除 user pool；這不是生產資源授權。
4. 另行決定 Cognito 的 user profile retention、應用 audit retention、backup retention、RPO/RTO 和 provider-exit deadline。這些未決定前不得聲稱完整刪除或完整可攜出。
