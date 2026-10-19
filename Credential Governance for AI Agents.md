## 如何管理 Agent 使用的 credential

### 什麼是 credential
- 定義：用來向某個系統證明身分或授權的資料。
- 常見形式：API token、access key（key ID 加 secret）、密碼、SSH key、OAuth token、bearer token。
- 在 agent 場景裡，agent 以「你」或「某個服務帳號」的身分操作系統，所以它拿到的 credential 決定它能做什麼。

### 問題情境

**案例 1：長期 key 散落在各人電腦**
- 新成員加入，管理員把一組雲端 access key 傳給他。他放進本機 `.env` 或 MCP 設定檔，agent 因此能查詢雲端資源。
- 問題：key 長期有效，存在每個人的電腦。成員離開後 key 仍可使用，也無法得知誰用過。
- 希望解決：不讓長期 key 留在每個人的電腦上。Agent 取得的 credential 有期限、可收回，成員離開後不再有效，並且能知道是誰用的。

**案例 2：正式環境與測試環境的存取範圍**
- 團隊有 production 與 staging 兩個環境。若兩組 credential 放在共用位置，任何人的 agent 都能動到 production。
- 問題：缺少「誰能用哪組 credential」的控管，也難以追溯是誰的 agent 做了什麼。
- 希望解決：每組 credential 只有被允許的人能用，例如 production 只給小群組、staging 開放全隊，並且能追溯是誰的 agent 做了什麼。

**案例 3：agent 被誘導洩漏 credential**
- Agent 讀到惡意內容（prompt injection），被誘導把手上的 token 寫進 log、訊息或工具呼叫中。
- 問題：若是長期 token，外洩後攻擊者可長期使用；若是短效且範圍受限，損害範圍小得多。
- 希望解決：縮小 agent 手上 credential 的有效時間與權限範圍，即使 agent 被誘導洩漏，損害也限縮在短時間與有限範圍內。

**為什麼 agent 讓問題更複雜**
- agent 可串接多個工具呼叫、在 session 中改變上下文，並傳入會影響下游系統的參數。
- 這是推論：人類使用者通常一次只做一件事，agent 會自動重複與串接操作，所以一份過寬的 credential 被濫用的速度與範圍更大。

### 風險對照（由左到右）

**常見做法：agent 直接持有長期 key**

```
[1 Admin]  -->  [2 Members]  -->  [3 Local files]  -->  [4 Agent]
creates a       receive a         .env or MCP           holds the
long-lived key  copy of key       config files          key directly
```

- 1：管理員產生一組長期有效的 key。
- 2：每位成員各拿到一份副本。
- 3：副本存在本機的 `.env` 或 MCP 設定檔。
- 4：agent 直接持有 key，一旦外洩就長期可用。

**解法方向：agent 不持有原始 credential**

```
[1 Agent]  -->  [2 Gateway]  -->  [3 Secrets]  -->  [4 Target]
sends a         checks identity   keeps the raw     receives only a
request         and policy        secret here       short-lived
                                                    credential
```

- 1：agent 只送出請求。
- 2：gateway 或 broker 驗證身分並檢查政策。
- 3：原始 credential 留在 secrets store，不交給 agent。
- 4：目標系統收到的是短效或有範圍的 credential。

### 現有解法方案（通用做法）

**1. 短效憑證加聯合認證（federation）**
- 做法：不存長期金鑰，與目標系統建立信任關係，每次依需求發放有效期很短的 credential。
- 收益：外洩後的有效窗口小，不需要長期 key 的輪替。
- 成本與風險：需在目標系統設定信任與角色；若目標系統（例如舊的 MCP server）只支援長期 bearer token，就沒有每個任務發放或撤銷的執行點。
- 適用：目標系統支援聯合認證或臨時憑證。

**2. 憑證代理（broker 或 proxy）**
- 做法：agent 發出請求，代理在網路邊界才附上真實 credential，agent 自己不持有。
- 收益：即使 agent 被 prompt injection 誘導，也沒有 credential 可洩漏。
- 成本與風險：多一層網路元件，要設定允許存取的服務；有些此類實作仍標示為預覽階段，不建議用於正式環境。
- 適用：agent 呼叫的服務以 HTTP API 為主。

**3. 身分感知的 gateway**
- 做法：agent 以 OAuth、JWT 或 mTLS 向 gateway 認證，由政策決定是否允許該動作，再交換短效的上游 credential。
- 收益：每次呼叫都可授權、可記錄，並能追溯「哪個 agent 用了什麼」。
- 成本與風險：需要建置並維運 gateway，也要撰寫與維護政策。
- 適用：agent 與工具數量多，或需要集中稽核。

**4. Secrets store 搭配參考（reference）與動態 secrets**
- 做法：原始 secret 留在 secrets store，gateway 只在執行時解析參考；記錄取得與輪替事件，但不記錄 secret 的值。長時間的 session 需要續期時，應該產生新的短效 credential，而不是延長舊的。
- 收益：集中輪替與稽核。
- 成本與風險：要維運 secrets store，並處理可用性。
- 適用：已有 secrets 管理基礎設施的組織。

**5. 存取控制與最小權限**
- 做法：角色權限（RBAC）、對每組 credential 設可使用成員的白名單、在每個 session 中只授予被選定的 credential。
- 收益：縮小誰能用、能用多廣。
- 成本與風險：需要持續維護白名單與角色。
- 適用：所有團隊，且通常是第一步。

**6. 人工核准與即時（JIT）存取**
- 做法：會改變系統的動作先提案，由人核准後才執行；高權限帳號只在任務期間存在，任務結束即移除。
- 收益：把「長期持有高權限」變成「按任務授予」。
- 成本與風險：核准會增加流程時間，需要平衡自動化程度。
- 適用：影響正式環境的操作。

**7. 盤點與清理**
- 做法：發現端點上的明文 `.env`、未加密的 SSH key 等 secrets，並對應到使用者與裝置。
- 收益：先看見現況，才知道要替換什麼。
- 成本與風險：只能發現與提醒，不能取代前面的發放機制。
- 適用：導入前的第一步。

### 實際落地

**建議的落地順序（這是我的判斷）**
- 先做 7（盤點）與 5（最小權限），成本最低。
- 再對支援的系統導入 1（短效憑證）。
- 當 agent 數量或環境風險提高時，再加入 2 或 3（代理或 gateway），並以 4 集中管理原始 secrets。
- 對會改變正式環境的操作，用 6 補上人工核准。

**目前可用的產品（依解法編號，非完整市場調查）**
- 解法 1（短效憑證與聯合認證）
  - 各雲端的原生機制：AWS 以 IAM role 搭配 OIDC web-identity federation，GCP 以 service account impersonation。
  - HashiCorp Vault：以 dynamic secrets 發放短效 credential。
  - Nuphos：以信任關係連接雲端帳號，每個 session 發放約 1 小時到期的臨時憑證；官方文件標示為 early access。
- 解法 2（憑證代理）
  - Infisical Agent Vault：在請求離開網路時才附上真實 credential；官方文件標示為 preview，不建議用於 production。
- 解法 3（身分感知 gateway）
  - Aembit MCP Identity Gateway：agent 不會收到 MCP server 的 credential，gateway 逐次取得並快取 60 秒，不寫入磁碟。2026 年 2 月的更新紀錄標題顯示進入 beta，目前狀態我沒有查到。
  - Gravitee、MintMCP：皆自述提供 agent 或 MCP 的 gateway。我只有廠商自述，沒有確認各自的發佈狀態。
  - Runlayer：agent control plane；1Password 的公告說它整合 1Password，把 credential 注入其管理的 agent session。
- 解法 4（secrets store 搭配參考與動態 secrets）
  - HashiCorp Vault、Infisical、AWS Secrets Manager：一篇比較文章列為適用於 agent 的 secrets 管理工具。
  - 1Password Unified Access：2026 年 3 月發表，Unified Access Pro 已正式提供；公告當時說「今年稍後」才會加入 runtime 發放 scoped credentials。
- 解法 5（存取控制與最小權限）
  - Claude Code managed settings：透過 claude.ai admin console 或 `managed-settings.json` 派送，`allowedMcpServers` 可設定 MCP server 白名單。
  - Nuphos：Administrator、Editor、Viewer 角色，對每個連接器設成員白名單，session 內只授予選定的 credential。
- 解法 6（人工核准與 JIT）
  - Nuphos：變更以 plan 提出，核准後才執行，並留有稽核紀錄。
  - 1Password Privileged Access：2026 年 7 月 28 日發表，依需求開通存取並自動撤銷。公告沒有標示它是 GA 或 preview；其中的 Credential Broker 針對 GitHub Actions 為 public preview，對 agent 場景的可用狀態公告沒有寫。
- 解法 7（盤點與清理）
  - 1Password Unified Access 的 Discover：公告寫為「available now」，可發現端點上的 AI 工具與 agent 活動，以及未加密的 SSH key、明文 `.env`，並對應到使用者與裝置。
  - Varlock（開源）：agent 只看到 `.env.schema`（變數名稱與型別），secret 值由後端在執行時解析；`varlock scan` 可掃描 AI 產生的程式碼是否洩漏 secrets。
