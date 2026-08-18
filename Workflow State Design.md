# 從 Durable Execution 推導 Workflow State 設計

## TL;DR：怎麼想、有哪些狀態

**核心提問只有一個：**
> 如果 workflow engine / 這台機器現在 crash，我怎麼知道副作用「到底發生了沒」？

想不清楚這一題，狀態設計就會漏掉最壞的 crash window（已執行、未記錄）。

**思考順序（對每個跨系統呼叫都問一次）：**
1. 這個 action 有沒有**先寫 intent 再執行**？（順序反了 = 重複扣款的根源）
2. 它逾時 / 回 5xx 時，我能不能證明副作用**沒發生**？不能證明 → 一律歸 `UNKNOWN`，不准歸 `FAILED`
3. 這個 unknown **誰來、什麼時候**把它判定出來？（要有 durable timer，不能靠 in-memory 等）
4. 消解 unknown 靠什麼？優先序固定：**查詢 → 冪等重放 → 補償**，都不行才人工
5. 兩個 worker 會不會同時推進同一筆？（lease 不夠，要 fencing token）
6. 這狀態值 crash 後**能不能重算**？能重算就不用存

**三層（依賴方向單向向下，不可混放同一張表）：**

| 層 | 回答什麼問題 | 典型值 | 誰能讀 |
|---|---|---|---|
| **Business State** | 商業上現在是什麼？ | `CREATED / PAID / SHIPPED` | API、UI、報表 |
| **Integration State** | 每個跨系統邊界做到哪？ | `IN_FLIGHT → UNKNOWN → SUCCEEDED / FAILED → COMPENSATED` | workflow 內部 |
| **Recovery Status** | crash 後怎麼恢復？ | `retry_count / next_retry_at / deadline / lease_expire` | 只有 recovery 邏輯 |

**IN_FLIGHT ≠ IN_PROGRESS / RUNNING**：三者字面上都像「還在跑」，但 IN_FLIGHT 專指「已送出、結果未知」——它的唯一理由是 Two Generals 造成的不確定性，所以才需要 timeout 把它升格成 UNKNOWN。IN_PROGRESS / RUNNING 是泛用的「目前正在執行」，不隱含結果不可知，通常用在本地、確定性的執行，例如 `exec_state` 的 `RUNNING`（整個 workflow run 還在跑，不是某一次外部呼叫卡在不確定期）。粒度不同：`RUNNING` 是 workflow 整體，`IN_FLIGHT` 是單次跨系統呼叫。

**每個 integration boundary 的固定生命週期（背這張圖就夠）：**

```
(無紀錄=NOT_STARTED)
        │ 先寫 intent
        v
    IN_FLIGHT ──timeout──> UNKNOWN ──查詢/重放/補償──> SUCCEEDED / COMPENSATED
        │
   明確回應
        ├──> SUCCEEDED（投影回 business state）
        └──> FAILED（僅限對方明確拒絕）──> 對前序步驟發起補償 ──> COMPENSATED
```

**一句話判準集：**
- FAILED 只能來自「對方明確、穩定地拒絕」，其餘（timeout / 5xx / 無法判讀 / 自己 crash）一律 UNKNOWN
- 每增加一個 integration boundary，就是固定成本：四態 + 一個 durable timer + 一個 unknown resolver + 一條對帳路徑——不隨邊界重要性縮放
- 最有效的優化不是把邊界做得更可靠，是**減少邊界數量**（Outbox 就是把兩個邊界合成一個本地 transaction）

---

## 起點：Durable Execution 是什麼

一句話：**讓一段程式碼在 crash 之後，能從中斷的地方繼續，而不是從頭開始。**

要做到這件事，只有一個辦法——把「執行進度」從記憶體搬到儲存體。也就是說：

```text
Durable Execution 的核心 = 把 call stack 變成 durable data
```

沒有別的魔法。Temporal、Restate、Step Functions 都只是這件事的不同包裝。

而接下來整份筆記要做的，是從這一句定義出發，一步一步推導出所有設計決策。你會發現它們幾乎都不是選擇，而是**被逼出來的結論**。

---

### 1. 為什麼單機不需要這個東西

單一 DB 的世界裡，call stack 也是活在記憶體的，crash 一樣整段消失。但沒關係：

```text
BEGIN
  step 1
  step 2
  crash        ← call stack 沒了
ROLLBACK       ← 但世界也回到起點了
```

**進度資訊遺失是無害的，因為副作用也一起被回收了。** 兩者活在同一個 failure domain，同生共死，所以你不需要記錄任何進度。

一旦有了外部副作用，這個前提崩掉：

```text
BEGIN
  UPDATE orders
  call payment API      ← 這一行的效果，rollback 管不到
  crash
ROLLBACK                ← 只回收了 DB，錢已經扣了
```

問題的本質不是「不能原子化」，而是：

> **進度資訊（記憶體）與副作用（外部世界）不在同一個 failure domain。**

從這裡只有兩條出路：

- **把副作用拉進 transaction** —— 這是 2PC，代價是鎖住整個世界
- **把進度推出記憶體** —— 這是 Durable Execution，代價是要自己管狀態

後面談的都是第二條路。

---

### 2. 第一次嘗試：記錄「做到第幾步」

最直覺的做法：

```text
workflow_run
  run_id = abc
  step   = 3
```

跑完一步就更新一次。看起來合理，直到你問一個問題：**crash 發生在寫 `step=3` 之前那一瞬間呢？**

```text
step = 2 已寫入
    ↓
執行 step 3 的 payment 呼叫   ← 副作用發生
    ↓
crash                          ← 還來不及寫 step = 3
    ↓
重啟後看到 step = 2，於是重做 step 3   ← 重複扣款
```

「執行完才記錄」這個順序，讓 crash window 剛好落在**已執行但無紀錄**的區間。這是最壞的狀態，因為你連自己做過什麼都不知道。

> **推論 1：durable state 必須寫在 action 之前，不是之後。**

順序被鎖死成這樣：

```text
Command
   │
Persist INTENT      ← 先落地意圖
   │
Action              ← 才碰外部系統
   │
Persist RESULT
   │
Publish Event
```

於是最小狀態集合從一個數字變成三態：`INTENT / DONE / FAILED`。

---

### 3. Intent-first 還是不夠：「不知道」是一種永久狀態

現在重啟後看到 `INTENT`，你知道「我打算做，但不確定做了沒」。然後呢？

- 重做 → 可能重複扣款
- 跳過 → 可能根本沒扣

你需要更多資訊，但**這個資訊在原則上取不到**。呼叫 payment API 遇到 timeout 時，物理現實有三種：

- API 根本沒收到請求
- API 收到了但還在處理
- API 已經完成，只是 response 在回程遺失

這三者從本地觀測**完全無法區分**，而且不是工程沒做好——這是 Two Generals 的直接後果。無論你加多少 log、多少監控，這個區分永遠取不到。

既然消不掉，就只能承認它：

> **推論 2：`RESULT_UNKNOWN` 必須是一個正式狀態，不能被歸類到成功或失敗任何一邊。**

歸錯邊的後果是對稱的：

- 歸成 FAILED → 去補償一筆可能沒發生的交易，或重扣一筆已成功的款
- 歸成 NOT_STARTED → 盲目重試，重複副作用

這條推論還有一個實務上殺傷力很大的推論：

> **推論 3：`FAILED` 只有一種合法來源——對方明確、穩定地告訴你這件事沒成立。**

餘額不足、參數違規、業務規則拒絕，且帶可判讀錯誤碼，這些是 FAILED。下面這些**全部是 UNKNOWN**：

- Timeout / connection reset
- HTTP 5xx
- 回了非預期格式，無法判讀
- 你自己在送出後、寫入結果前 crash

判準是一句話：**這個回應能否證明副作用「沒有發生」？** 不能證明，就是 unknown。把 5xx 當 FAILED 然後觸發補償，是雙重副作用最常見的來源。

---

### 4. Unknown 要怎麼消掉

Unknown 不能停在那裡，流程要繼續就得把它變成 known。方法只有三條，按優先序：

1. **查詢**——對方提供以 request_id 查結果的 API。最乾淨，直接收斂成 succeeded 或 failed。
2. **冪等重放**——帶同一個 idempotency key 重送，對方回傳既有結果而不重複執行。等於用重試把 unknown 折疊成 success。
3. **補償**——承認「可能已經做過」，用業務上的反向操作抹平。最貴，而且補償本身也會產生新的 unknown。

三條都走不通，就只剩人工對帳。

注意第 2 條的前提：

> **推論 4：idempotency key 不是效能優化，是 unknown resolver 的前提條件。**

沒有它，你連「安全地重試」都做不到，只能一路退到補償或人工。而 key 必須是**穩定可重算**的（`hash(run_id, step_name)`），用 timestamp 或 random 等於沒有。

這也決定了對外部系統選型時，第一個該問的不是 SLA：

- 你支援 idempotency key 嗎？
- 有沒有以 request_id 查結果的 API？
- 有沒有業務上的反向操作？

這三個答案直接決定你的 recovery 複雜度。

---

### 5. Unknown 不會自己被發現

再往下一層問：`INTENT` 是什麼時候變成 `UNKNOWN` 的？

答案是——**沒有人告訴你**。外部系統不會打電話來說「我 timeout 了」。一筆卡在 IN_FLIGHT 的紀錄，如果沒有人主動去看它，就會永遠躺在那裡。

```text
IN_FLIGHT ──── 超過 deadline ────> RESULT_UNKNOWN
                     ↑
              需要有人在未來叫醒它
```

> **推論 5：unknown 是被 timer「判定」出來的，不是被寫入的。所以 durable timer 是必需品，不是優化。**

in-memory timer 在 crash 後消失，等於這筆流程靜靜卡死，而且**不會有任何告警**——它看起來只是還在跑。這是生產環境最難查的一類問題。

推廣成通則：**每個非終態都必須有 timeout owner。** 沒有 timeout 的中間態等於永久卡單。

---

### 6. 現在 recovery 退化成查表

把前五步累積的東西合起來，一個跨系統 action 的完整生命週期是：

```text
        (無紀錄)
      NOT_STARTED
           │
           │  先 commit intent
           v
       IN_FLIGHT ────── timeout ──────> RESULT_UNKNOWN
           │                                  │
     明確回應│                                 │ 查詢→重放→補償
           ├───────────────┐                  │
           v               v                  │
       SUCCEEDED     FAILED_DEFINITE <────────┘
           │               │
           │               v
           │         COMPENSATED
           v
      (推進下一步)
```

兩個容易漏掉的性質：

- **NOT_STARTED 不是一筆資料，是「紀錄不存在」。** 這是它唯一安全的表示法——一旦寫進 DB，它就已經不是 NOT_STARTED 了。
- **INTENT 與 IN_FLIGHT 在 recovery 眼中是同一態。** 「已記錄意圖但還沒送出」和「已送出但沒收到回應」，都無法確認副作用是否發生，處置方式完全相同。分成兩態只會製造假的精確感。

有了這些，recovery 就不再需要「判斷」，而是查表：

- **無紀錄** → 什麼都不做，副作用確定沒發生
- **IN_FLIGHT 未逾時** → 等，不要搶著重試
- **IN_FLIGHT 已逾時** → 升格成 UNKNOWN
- **UNKNOWN** → 依序嘗試查詢、冪等重放、補償
- **SUCCEEDED** → 投影回 business state，推進下一步
- **FAILED_DEFINITE** → 對所有已 SUCCEEDED 的前序步驟發起補償
- **COMPENSATED** → 終態

如果你的 recovery 邏輯裡出現「這種情況要看一下」的分支，通常代表狀態切得不夠細。

---

### 7. 誰來執行 recovery

Timer 到期後，會有一個 worker 醒來推進這筆流程。但在分散式環境裡，**可能同時醒來兩個**——網路分區、rebalance、重複投遞都會造成這件事。兩個 worker 同時對同一筆流程發 payment 請求，前面所有的努力就白費了。

第一個直覺是 lease：

```text
workflow_instance
  lease_owner  = worker-7
  lease_expire = 10:05
```

過期才可搶佔。但這**不夠**：worker-7 可能只是 GC pause 了 30 秒，醒來時它仍然認為自己持有 lease，於是兩個 worker 同時在跑。

> **推論 6：lease 只能減少衝突，不能消除。要真正安全，需要 fencing token。**

做法是讓一個單調遞增的版本號隨請求傳給下游，由**下游**拒絕舊 token。狀態轉移本身也用 optimistic lock（`WHERE state = ? AND version = ?`）保證單次生效。

---

### 8. 狀態變多了，該放在哪裡

到目前為止，我們被迫累積了這些東西：

- 業務狀態：`state = PAID`
- 執行狀態：`step`、`state`、`idem_key`、`result`
- 復原資訊：`retry_count`、`last_error`、`next_retry_at`、`deadline`、`lease`

如果全塞進 `orders`，腐化路徑是可預測的。先是：

```text
orders
  state
  payment_sent
  mq_sent
  retry_count
  last_error
```

接著一定會變成：

```text
orders
  state
  payment_sent
  payment_retry
  mq_sent
  mq_retry
  inventory_sent
  shipping_retry
```

Domain model 被執行細節汙染，state enum 無限膨脹，報表看不懂，換 orchestrator 要動 domain schema。

> **推論 7：三類狀態回答三個不同的問題，必須分層，而且依賴方向單向向下。**

```text
Recovery Status      retry_count / next_retry_at / deadline / last_error
        │
        │  驅動（誰該被喚醒、還能重試幾次）
        v
Integration State   IN_FLIGHT → UNKNOWN → SUCCEEDED / FAILED → COMPENSATED
        │
        │  投影（確定成功後才 commit 事實）
        v
Business State      CREATED → PAID → SHIPPED
```

- **Business State**——商業上目前是什麼狀態？通常很少，三五個就夠。給 API、UI、報表、稽核用。注意 `PAID` 只代表商業上已付款，不代表 MQ 已送出、consumer 已收到、所有 side effect 完成。這是真正的 `state`：互斥、有轉移規則、決定下一步合法動作。
- **Integration State**——每個跨系統 action 做到哪？這不是業務狀態，是**系統整合狀態**。
- **Recovery Status**——crash 後怎麼恢復？純粹為復原而存在，業務永遠不該讀它。

兩條必須守住的規則：

- **Business State 是投影，不是第二份真相。** 它應該是 integration state 的函數；獨立維護兩份會不一致，而且不一致時你分不出誰對。投影動作本身必須 idempotent。
- **但投影必須真的落地。** Workflow 是事實的產生者，不是儲存體。如果查「訂單是否已付款」要去 query workflow history，那 retention 過期後事實就消失了，engine 故障等於業務不可觀測。

判斷界線的測試：**如果 workflow engine 整組消失，這個資訊還必須查得到嗎？** 是 → Business State；否 → Integration 或 Recovery。

---

### 9. 一個關鍵觀察：狀態隨什麼成長

回頭看第 8 步的三層，會發現一件事：Business State 從頭到尾只有三個（`CREATED / PAID / SHIPPED`），膨脹的全是下面兩層。

```text
只有 DB            Order: CREATED → PAID          business state 就夠

加一個 MQ          DB 更新了，MQ 收到了嗎？         多一組 unknown

加 Payment         Payment 扣了嗎？                再多一組

加 Inventory       庫存鎖了嗎？                    再多一組
```

> **分散式系統增加的不是 Business State，而是 Integration State。每增加一個 integration boundary，就要付出一組完整的狀態代價。**

這句話是整份筆記真正的核心。它也解釋了 2PC、Saga、Durable Execution 為什麼看起來差很多、骨子裡在做同一件事——**它們都是在管理這組狀態，差別只在怎麼管**。

而每個邊界的固定成本是一樣的：四種狀態 + 一個 durable timer + 一個 unknown resolver + 一條對帳路徑。這個成本**不隨邊界的重要性縮放**——一個「只是發個通知」的 MQ，和一筆付款，付同樣的代價。

---

### 10. 所以：減少邊界，比優化邊界有效

既然每個邊界都是固定的高成本，最有效的優化不是把每個邊界做得更可靠，而是**讓邊界變少**。

這才是 Outbox 真正的價值。它常被說成「可靠投遞」，但更準確的說法是它**消除了一個邊界**：

```text
之前：兩個邊界，兩組 unknown

     DB 寫了嗎？        MQ 收到了嗎？


之後：一個本地 transaction + 一條單向重試

     DB 與 outbox 同一個 tx        （沒有 unknown）
     relay 投遞失敗就重送           （unknown 靠 at-least-once + 收端去重折疊掉）
```

收端以 message_id 去重（inbox），兩者成對使用才完整。

同樣的問題可以問流程的每一段：

- 這個邊界能不能收成本地 transaction？
- 兩次外部呼叫能不能合併成一次？
- 能不能改成單向通知，不等回應？

---

### 11. 失敗之後往哪走

前面談的都是「怎麼知道發生了什麼」。剩下最後一個問題：確定失敗之後怎麼辦。

單機的答案是 rollback，但 rollback 需要「回到過去」的能力，而外部系統不給你。所以只剩往前走：

> **compensation 不是 rollback，它是一筆新的業務事實。** 退款不等於撤銷付款——帳上會有兩筆紀錄，而這是正確的。

三個實務要點：

- **補償必須 idempotent 且不可失敗。** 補償失敗只能無限重試加告警，你不能為補償再寫補償。
- **待補償清單本身是 durable state。** 「哪些步驟已成功、需要反向處理」必須落地，不能靠 recovery 時去重掃。
- **forward recovery 優先。** 能往前推到完成就不要回頭，補償是最後手段。

有一類場景可以繞過補償：需要暫時占用資源時（搶票、庫存、座位），用 `RESERVED + expire_at` 取代長交易鎖。本質是**用一個帶過期時間的弱承諾，換掉 2PC 的資源鎖定**——占用方掛掉不會拖住別人，過期自動釋放，也就不需要補償。代價是「過期釋放」與「確認」之間的競態，確認時必須檢查 expire 並用 CAS。

---

### 12. 回頭看：我們剛剛重新發明了 Durable Execution

把前面十一步的產出攤開：

- 先寫 intent 再執行（第 2 步）
- 每個 action 的狀態 append 成紀錄（第 3 步）
- crash 後讀這些紀錄，決定續跑還是解消 unknown（第 4、6 步）
- timer 驅動喚醒（第 5 步）
- 單一 owner 推進（第 7 步）

這就是 Temporal 的 event history。**Durable Execution engine 幫你託管的，正是 Integration State 與 Recovery Status 這兩層**，讓你的程式碼只需要寫 happy path。

而這也解釋了它那條看似奇怪的限制——**workflow 程式碼必須 deterministic，side effect 必須包成 activity**：

```text
replay 時，engine 重跑你的 workflow 函式來重建 call stack
    │
    ├── 純決策邏輯：重算一次，無害
    │
    └── activity：不重跑，直接從 event history 讀既有結果
```

如果你在 workflow 裡直接呼叫外部 API，或用了 `time.Now()`、`rand`、map 迭代順序，replay 就會走到不同的分支，或者真的把副作用做第二次。這個限制不是框架龜毛，是**「用 replay 重建 call stack」這個設計的必然要求**。

`state = fold(events)` 這件事也順帶給了你回溯能力：可以回答「為什麼現在是 PAID」，而不只是「現在是 PAID」。

---

### 13. 放回光譜：各方案的定位

現在可以精確地說出差異了。不是「有沒有 prepare」，而是三軸：

- **Prepare 的語義**：鎖資源／記意圖／什麼都不做
- **Commit 的保證程度**：原子／最終一致／至少一次
- **失敗後的方向**：rollback 回到過去，還是 compensation 往前走出新事實

放在這三軸上：

- **2PC** 的 prepare 是資源鎖定並承諾可 commit，強一致，失敗走 rollback。它的價值是**把 unknown 集中到一個點**（coordinator 的 commit 決策），代價是阻塞性最高——coordinator 掛掉，參與者卡在 prepared。只適合同組織、短事務、參與者支援 XA。

- **Saga** 沒有 prepare，直接執行，最終一致，失敗走 compensation。不阻塞，適合長流程與跨組織。它把 unknown 留在每個邊界上，由各自的 resolver 處理。前提是每一步都能定義業務反向操作。

- **Outbox + Retry** 的 prepare 就是本地 commit 本身，at-least-once。它不處理 unknown，而是**消除一個產生 unknown 的邊界**。適合單向通知與事件擴散。

- **Durable Execution** 的 prepare 是記錄 intent 加 event log，執行語意上的 exactly-once，失敗要 rollback 還是 compensation 由程式碼決定。貢獻是**把 Integration 與 Recovery 兩層從業務程式碼裡抽走**。適合複雜編排、多分支、長時間等待。

- **Event Sourcing** 的事件本身即 prepare，單 aggregate 內強一致，失敗用 corrective event 修正。適合需要完整稽核與時間回溯。

一句話收攏：**Durable Execution 的立場是接受世界不是 transaction**，不再假裝能原子化，改為可靠地管理狀態演進。

---

### 14. 落地

Schema 骨架，三層分開：

```sql
-- Business State：領域事實，長期保存
CREATE TABLE orders (
  order_id    BIGINT PRIMARY KEY,
  state       VARCHAR(32) NOT NULL,   -- 僅業務語意: CREATED/PAID/SHIPPED/CANCELLED
  amount      DECIMAL(18,4) NOT NULL,
  updated_at  TIMESTAMP NOT NULL      -- 存 UTC
);

-- Integration State：每個邊界的執行狀態
CREATE TABLE integration_action (
  run_id      CHAR(36)     NOT NULL,
  step_name   VARCHAR(64)  NOT NULL,
  boundary    VARCHAR(32)  NOT NULL,  -- payment / inventory / mq ...
  state       VARCHAR(24)  NOT NULL,  -- IN_FLIGHT/UNKNOWN/SUCCEEDED/FAILED/COMPENSATED
  idem_key    VARCHAR(128) NOT NULL,
  result      JSON,                   -- 外部回傳的不可重算值
  PRIMARY KEY (run_id, step_name),
  UNIQUE KEY uk_idem (idem_key)
);

-- Recovery Status：純粹為復原而存在
CREATE TABLE action_recovery (
  run_id        CHAR(36)    NOT NULL,
  step_name     VARCHAR(64) NOT NULL,
  attempt       INT         NOT NULL DEFAULT 0,
  last_error    TEXT,
  next_retry_at TIMESTAMP,            -- durable timer
  deadline      TIMESTAMP   NOT NULL, -- 逾時後升格為 UNKNOWN
  PRIMARY KEY (run_id, step_name),
  KEY idx_wake (next_retry_at)
);

-- Workflow 實例：綁定業務、防重入、控制併發
CREATE TABLE workflow_instance (
  run_id        CHAR(36) PRIMARY KEY,
  workflow_type VARCHAR(64) NOT NULL,
  biz_key       VARCHAR(64) NOT NULL,
  current_step  VARCHAR(64) NOT NULL,
  exec_state    VARCHAR(16) NOT NULL,  -- RUNNING/COMPLETED/FAILED/COMPENSATING
  lease_owner   VARCHAR(64),
  lease_expire  TIMESTAMP,
  version       INT NOT NULL,          -- optimistic lock / fencing token
  UNIQUE KEY uk_biz (workflow_type, biz_key)
);

CREATE TABLE outbox (
  id           BIGINT AUTO_INCREMENT PRIMARY KEY,
  aggregate_id VARCHAR(64) NOT NULL,
  payload      JSON NOT NULL,
  published_at TIMESTAMP NULL,
  KEY idx_pending (published_at, id)
);

CREATE TABLE reservations (
  resource_id VARCHAR(64) PRIMARY KEY,
  holder      VARCHAR(64) NOT NULL,
  state       VARCHAR(16) NOT NULL,    -- RESERVED/CONFIRMED/RELEASED
  expire_at   TIMESTAMP   NOT NULL
);
```

Go 這一段是第 3 步那條推論的直接落地——把「什麼算 FAILED」寫死，不讓它變成每個 caller 各自判斷：

```go
type IntegrationState string

const (
    InFlight    IntegrationState = "IN_FLIGHT"
    Unknown     IntegrationState = "UNKNOWN"
    Succeeded   IntegrationState = "SUCCEEDED"
    Failed      IntegrationState = "FAILED"       // 僅限對方明確拒絕
    Compensated IntegrationState = "COMPENSATED"
)

var transitions = map[IntegrationState][]IntegrationState{
    InFlight: {Unknown, Succeeded, Failed},
    Unknown:  {Succeeded, Failed, Compensated}, // 解消後才離開
    Failed:   {Compensated},
}

func classify(err error, resp *Response) IntegrationState {
    switch {
    case err != nil:
        return Unknown                  // timeout / conn reset 一律 unknown
    case resp.Status >= 500:
        return Unknown                  // 5xx 不能證明副作用沒發生
    case resp.IsBusinessRejection():
        return Failed                   // 餘額不足、規則拒絕
    case resp.OK():
        return Succeeded
    default:
        return Unknown                  // 無法判讀 → unknown
    }
}
```

該存什麼的判準只有一句：**crash 後這個值能不能重新算出來？**

- 不能就存：決策、外部回傳的不可重算值、intent、去重憑據
- 能就不存：deterministic 衍生值 replay 時算就好
- 大 payload 存引用（GCS/S3 path）而非內容
- 外部快照存 reference 加版本

不是每個流程都要走完整套。決策順序：

1. 副作用可逆且可 idempotent replay → 純 retry，不需要 durable state
2. 只是單向通知，不需等回應 → Outbox + Inbox（消除邊界）
3. 需要對方確認，但流程線性、步驟少於五步 → 自己寫 write-ahead intent + idempotency key + 對帳
4. 需要暫時占用資源 → 加上 Reservation State + durable timer
5. 分支多，或有長時間等待（人工審核、外部回呼、天級 timeout）→ Durable Execution 或顯式 Saga orchestrator
6. 需要完整稽核與時間回溯 → 再疊加 Event Sourcing

---

### 15. 設計階段的產出物

把流程中所有 integration boundary 列出來，每一個都要能回答這六題。答不出來的欄位，就是未來的生產事故：

- 這個邊界的 idempotency key 是什麼？對方支援嗎？
- 有沒有「以 request_id 查結果」的 API？
- 有沒有業務上的反向操作？誰負責定義它的語意？
- timeout 設多久？逾時後由誰把它升格成 UNKNOWN？
- unknown 的 resolver 是查詢、重放、還是補償？
- 對帳跑多久一次？對帳的權威來源是哪一邊？

---

不論架構怎麼選，底線都是 **timeout 加對帳**。所有分散式流程最終都會走到「不知道對方到底做了沒」——這不是設計缺陷，是分散式的定義。對帳是唯一的收斂手段，不是可選項。

---

### 16. 換成 Temporal 之後，哪些狀態不用自己定義

第 12 步說過：Temporal 這類 durable execution engine 代管的正是 Integration State 與 Recovery Status 兩層。但「代管」不等於「消失」——只是換了地方存、換了人寫。把第 14 步的四張表拿出來逐一核對：

**Temporal 真的省下來的部分：**

| 文件裡的表 / 欄位 | Temporal 對應機制 | 結論 |
|---|---|---|
| `action_recovery`（retry_count / next_retry_at / deadline） | `RetryPolicy`（backoff、max attempts）+ Activity Timeout（StartToClose / ScheduleToClose）+ Timer 事件，全部 append 進 event history | 整張表可以刪掉 |
| `integration_action` 的執行進度（IN_FLIGHT / SUCCEEDED） | `ActivityTaskScheduled/Started/Completed` 事件天然是 append-only 紀錄 | 不用自己維護「跑到哪一步」 |
| lease / fencing（第 7 步的 `lease_owner` / `version`） | Task Queue 的 sticky execution + workflow 單一 owner 保證 | 不用自己刻 fencing token |
| durable timer（第 5 步） | `workflow.Sleep` / Activity timeout 本身即 durable timer | 不用另建 timer 表 |

**但這三件事，換到 Temporal 之後仍是你的責任，只是寫的位置換了：**

1. **UNKNOWN 的消解邏輯（第 4 步）不會自動發生。** Temporal 只保證 Activity 會被重跑到收斂，但它不知道外部 payment API 在 timeout 當下究竟做了沒。查詢 / 冪等重放 / 補償，這段邏輯要自己寫在 Activity 內部，engine 不會替你生成。

2. **FAILED 的判準（第 3、14 步的 `classify()`）要顯式搬進 Activity。** 對應 Temporal 的做法是：明確拋 `NonRetryableApplicationError`（= FAILED，觸發補償）vs 其他錯誤（= UNKNOWN，讓 RetryPolicy 自動重試）。這條分類規則沒有預設值——寫錯就是把 5xx 判成 FAILED 直接觸發補償，正是第 178 行講的雙重副作用最常見來源。

3. **Business State 的投影仍要落地成獨立儲存。** Event history 有 retention（過期即清除），不能拿它當業務事實的長期真相。`orders` 表還是要存在，還是要在 workflow 裡顯式呼叫一個 Activity 把 `PAID` 寫回自己的 DB——這是第 8 步「投影必須真的落地」規則的直接延伸，不會因為換了 engine 就失效。

一句話收攏（換成 Temporal 官方術語會更精確）：

> Temporal 自動代管的是 Activity 的 **execution bookkeeping**——也就是每次呼叫的 `Scheduled → Started → Completed / Failed / TimedOut` 這條 **state machine**，全部自動寫進 **Event History**，這對應到 Recovery Status 整層，加上 Integration State 裡「這次呼叫進行到哪一步」的部分。
>
> 但 Temporal **不會**替你做 domain-level 的判斷：呼叫的**回傳結果該解讀成什麼**（UNKNOWN 還是 FAILED）、以及**收斂它的手段**（query API / idempotent replay / compensation），這些邏輯要你自己寫在 Activity function 內部，用 `NonRetryableApplicationError` 顯式標記 FAILED，其餘一律讓 `RetryPolicy` 自動重試（= UNKNOWN 的行為）。

也就是：**execution bookkeeping（這次呼叫跑到哪一步了）Temporal 幫你記；但 result interpretation（這個結果算什麼）與 result resolution（怎麼把 unknown 收斂掉）永遠是你的程式碼要寫的部分。**

**落到 `integration_action` 這張表，該怎麼砍：**

| 欄位 | 换成 Temporal 之後 |
|---|---|
| `state`（IN_FLIGHT/UNKNOWN/…） | **可以刪**。Activity 的 `Scheduled/Started/Completed/Failed/TimedOut` 已經是 Event History 裡現成的 state machine，不用自己再維護一份同語意的欄位。 |
| `idem_key` | **可以不存**，只要它是 `hash(run_id, step_name)` 這種可重算值（第 4 步已強調要穩定可重算），每次呼叫時算出來即可，不需要落地。 |
| `result` | **不能刪，但要換位置**。這是外部系統回傳的不可重算值（交易編號、回應 payload），第 14 步的判準是「crash 後能不能重算」——不能重算就要存。Temporal 會把它寫進 `ActivityTaskCompleted` 事件的 payload，但那份紀錄有 retention、查詢要走 `GetWorkflowHistory`，不是一張能直接 `SELECT` 的表。 |

保留的理由是第 8 步那條規則的延伸——**投影必須真的落地**：如果查「這筆付款的外部交易編號是多少」要去翻 workflow history，retention 一過就查不到了，engine 出問題等於這筆事實不可觀測。所以實務上留一張瘦身版、拿掉 `state` 欄位的表，角色從「recovery 邏輯查表決定下一步」降級成「對帳 / 稽核用的唯讀投影」：

```sql
CREATE TABLE integration_action_result (
  run_id      CHAR(36)     NOT NULL,
  step_name   VARCHAR(64)  NOT NULL,
  boundary    VARCHAR(32)  NOT NULL,
  result      JSON,          -- 外部回傳的不可重算值，對帳/稽核用
  recorded_at TIMESTAMP    NOT NULL,
  PRIMARY KEY (run_id, step_name)
);
```

---

**`workflow_instance` 呢？——可以整張刪，理由是它的四組欄位在 Temporal 裡各自有對應的原生機制：**

| 欄位 | Temporal 對應 | 結論 |
|---|---|---|
| `current_step` | workflow 函式的程式計數器，crash 後靠 replay 重建（第 12 步的 deterministic replay） | 可重算 → 不用存 |
| `exec_state`（RUNNING/COMPLETED/…） | Temporal 原生的 Workflow Execution Status，`DescribeWorkflowExecution` / Visibility API 直接查得到 | 不用自己維護 |
| `lease_owner` / `lease_expire` / `version` | Task Queue 的 sticky execution + 內部 optimistic lock，本來就保證同一個 workflow run 同時只有一個 worker 在推進 | 不用自己刻 fencing token |
| `biz_key`（配合 `uk_biz` 防重入） | 把 `WorkflowID` 直接設成 `{workflow_type}-{biz_key}`，搭配 `WorkflowIDReusePolicy`（如 `RejectDuplicate`）讓 Temporal 在啟動時就擋掉重複的業務鍵 | 不需要一張表做唯一性檢查，Temporal 啟動 API 本身就是 idempotent 的 |

也就是說，只要**用業務鍵去建構 WorkflowID**，這張表存在的兩個理由（防重入、biz_key ↔ run_id 對照）就都不成立了——外部呼叫方要 signal/query 這個 workflow，直接用同一個規則算出 WorkflowID 即可，不需要查表。

**唯一的例外**：如果 retention 過期後，你仍需要長期查「這筆訂單當初對應哪個 workflow run」做稽核，那就不是靠 `workflow_instance` 撐著，而是把 `run_id` 當成一個欄位掛在 Business State（`orders`）表上——這仍然遵守第 8 步的分層原則：**biz_key ↔ run_id 的對照本身是業務事實的一部分，該落地在 Business State，不是重新造一張執行期用的表。**

一句話收攏：**`workflow_instance` 的四組欄位，Temporal 都有原生機制頂替，唯一該留下來的只有「run_id 對照」，而它的正確位置是 Business State 表的一個欄位，不是獨立表。**

落到第 14 步的 schema 上：`action_recovery` 可以整張刪；`integration_action` 可以簡化成只存 `idem_key` 與 `result`（`state` 轉移已經在 event history 裡，不必重複記錄）；`orders`（Business State）與 Activity 內的錯誤分類 / unknown resolver 邏輯,則不論用不用 Temporal 都要留著。