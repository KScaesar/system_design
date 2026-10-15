# Ingestion Pipeline Control Plane

## 重點摘要

- Ingestion Pipeline 分為兩個部分：
  - Data Plane：負責資料傳輸與儲存。
  - Control Plane：負責管理 ingestion metadata 與 processing state。
- 若需要 Replay 後產生完全相同的 Object，必須保存原始 batch boundary。
- Flush decision 不應只存在於 Consumer runtime memory。
- Metadata Store 是 Ingestion Pipeline Control Plane 的 Source of Truth。
- Replay 不重新執行 flush decision，而是使用已保存的 batch metadata。
- Consumer 的起始讀取位置由 Metadata Store 決定，MQ 自身的 offset commit 只是 best-effort 加速，不是 Source of Truth。

---

# 架構概念

Ingestion Pipeline 包含兩個 Plane。

## Data Plane

負責實際資料流動。

```
Message Queue -> Consumer -> Buffer -> Object Storage
```

保存：Event data、Object file、Partition data。

## Control Plane

負責管理 ingestion 狀態。

```
Consumer -> Metadata Store -> Batch Metadata
```

保存：Batch 資訊、Source position、Object mapping、Batch state。

---

# 為什麼需要 Control Plane

Consumer 通常使用多個 flush 條件：Buffer size、Row count、Time interval。

例如原始 ingestion：`event-1, event-2, event-3` 等待超過 timeout 後 Flush，產生 `object-001`。

但 Replay 時來源多了新事件（`event-1 ~ event-5`），Consumer 可能因為 Buffer size 或 Row count 提前觸發 flush，導致：

- Event data 相同。
- Object file 不同。
- Batch boundary 不同。

因此 Replay 不具備 deterministic behavior。

---

# Batch Boundary

定義一次 Object 產生的資料範圍，需要保存：Start position、End position、Flush reason。

例如：

```
Batch ID:        batch-001
Offset Start:    100
Offset End:      102
Flush Reason:    TIMEOUT
```

Replay 流程：讀取 Batch Metadata → 取得原始 boundary → 重新產生 Object。

Replay 不重新計算：Flush time、Buffer size、Row count。

---

# Metadata Store

Control Plane 的核心元件，保存 ingestion lifecycle，例如 `batch_catalog`。

Schema：

```sql
CREATE TABLE batch_catalog (
    batch_id        VARCHAR PRIMARY KEY,
    source_name     VARCHAR,
    partition_id    VARCHAR,
    offset_start    BIGINT,
    offset_end      BIGINT,
    row_count       BIGINT,
    byte_size       BIGINT,
    flush_reason    VARCHAR,
    run_type        VARCHAR,   -- REALTIME / BACKFILL / REPROCESS，預設 REALTIME
    source_batch_id VARCHAR,   -- 若此 batch 是重新產生某個既有 Object，指向原始 batch_id；REALTIME 為 NULL
    writer_schema_id VARCHAR,  -- 寫入當下使用的 Avro Writer Schema（Schema Registry 版本號或 fingerprint）
    object_path     VARCHAR,
    checksum        VARCHAR,
    state           VARCHAR,
    created_at      TIMESTAMP,
    completed_at    TIMESTAMP
);
```

---

# Batch Lifecycle

- **CREATED**：已建立 Batch Metadata，但 Object 尚未完成。
- **WRITING**：正在寫入 Object Storage。
- **COMMITTED**：Object 已成功寫入，Metadata 已更新。
- **FAILED**：寫入失敗，需要 Recovery。

---

# Ingestion Flow

```
Message Queue -> Consumer Buffer -> Create Batch Metadata
  -> Write Object Storage -> Update Batch State -> Commit Message Offset
```

Commit Message Offset 必須在 Object write 成功、Metadata update 成功之後執行。

`Commit Message Offset` 是 best-effort 步驟：它只是為了讓 MQ 下次能從接近的位置開始讀取，藉此縮短重複讀取的範圍。它是否成功，不影響 Batch 是否視為完成——這由 Batch State 決定。

---

# Consumer Position 管理

Consumer 啟動或重啟時，不採用 MQ 自身記錄的 committed offset 作為起始位置，而是向 Metadata Store 查詢：

```
SELECT MAX(offset_end)
FROM batch_catalog
WHERE partition_id = :partition_id
  AND state = 'COMMITTED'
```

並從 `offset_end + 1` 開始讀取。

原因：

- MQ 的 offset commit 與 Batch 的 COMMITTED 狀態是兩個獨立的寫入操作，之間沒有 transaction，一定存在其中一個成功、另一個失敗的 window。
- 若以 MQ committed offset 為準，這個 window 會造成資料遺失（offset 已 commit，但對應 Batch 從未完成）或重複攤入未知範圍。
- 以 Metadata Store 為準，起始位置永遠對齊「已確定完成」的 Batch，MQ offset commit 失敗與否不再影響正確性，只影響下次要重讀多少範圍。

因此 MQ offset commit 失敗不需要被視為一種 failure case ——它的唯一後果是下次啟動會重讀一段已經處理過、但尚未 ack 給 MQ 的範圍，而這段範圍會被下面的重複偵測機制擋下。

---

# Failure Recovery

## Object Write 前失敗

狀態：`Batch State: CREATED`

Recovery：找出未完成 Batch → 重新寫入 Object → 更新 Batch State。

## Object Write 後失敗

狀態：`Object exists`、`Batch State: WRITING`

Recovery：檢查 Object Metadata → 驗證 checksum → 更新 Batch State。

## 重複讀取相同 Offset 範圍

成因：Consumer 依 [Consumer Position 管理](#consumer-position-管理) 從 Metadata Store 取得起始位置，但該範圍先前已存在一個 `COMMITTED` Batch（因為 MQ offset commit 未完成才被重讀）。

Recovery：Create Batch Metadata 前，先以 `partition_id + offset_start` 查詢 `batch_catalog` 是否已有 `COMMITTED` 且涵蓋相同範圍的 Batch：

- 若存在，直接略過（不重新 flush、不重新寫 Object），視同已完成。
- 若不存在，走正常 Ingestion Flow。

---

# Replay

```
Metadata Store -> 取得 Batch Boundary -> 讀取原始資料範圍 -> 產生相同 Object
```

Replay 不依賴：Current time、Current buffer size、Current flush guard。

---

# Backfill

Backfill 不直接覆蓋原始 Object，而是在 `batch_catalog` 新增一筆 batch，用 `source_batch_id` 指回原始 batch：

- `run_type` = `BACKFILL` 或 `REPROCESS`（`REALTIME` 為一般 ingestion）。
- `source_batch_id` = 被重新處理的原始 batch_id；`REALTIME` 為 `NULL`。
- 原始 batch 的 row 不變動、不刪除——`batch_id` 是各自獨立產生的新值，兩筆 row 之間只透過 `source_batch_id` 建立單向關聯，不需要回頭修改原始 row。

Original Object、New Object 因此分別對應兩筆 row 各自的 `object_path`；Processing Time 對應新 batch 的 `created_at`。

判斷某個 batch 是否已被更新的 backfill 取代（現行版本查詢），用查詢 derive，不額外存欄位：

```sql
SELECT b.*
FROM batch_catalog b
WHERE b.state = 'COMMITTED'
  AND NOT EXISTS (
    SELECT 1 FROM batch_catalog s WHERE s.source_batch_id = b.batch_id
  )
```

（`source_batch_id` 需建 index；`NOT EXISTS` 讓 query planner 直接識別為 anti-join，比 `NOT IN` 對大表更友善，也不受 NULL 語意影響。）

---

# Flush Reason

需要保存 flush 原因：`SIZE`、`ROW_COUNT`、`TIMEOUT`、`SHUTDOWN`、`MANUAL`、`BACKFILL`

不同 Flush Reason 代表不同 ingestion 行為，例如：

```
Consumer Shutdown -> Flush remaining buffer -> Create Batch
```

除了描述當下行為，`flush_reason` 也是事後優化的依據：統計一段時間內各 Reason 的分布，可以判斷目前的 flush 條件是否設得合理。例如 `SIZE`/`ROW_COUNT` 佔比過高代表 buffer 門檻太小、Object 被切得太碎；`TIMEOUT` 佔比過高則代表流量不足以撐滿 buffer，門檻可以調低以降低延遲。這類調整可以直接對 `batch_catalog` 做 `GROUP BY flush_reason` 分析，不需要額外的監控管道。

Replay 時需要保留這個資訊。

---

# Schema Resolution（Avro）

Object 檔案格式若為 Avro，Schema Evolution 是無可避免的問題：上游 Producer 會隨時間新增欄位、調整型別，但 Object Storage 裡已寫入的舊 Object 不會被回頭重寫。

## Writer Schema vs Reader Schema

Avro 的二進位 payload 不帶欄位名稱與型別資訊，Reader 必須拿到寫入當下的 **Writer Schema** 才能正確切分 byte；下游實際讀取用的則是 **Reader Schema**，可能比 Writer Schema 新（新增欄位）或舊。

讀取時 Avro 依 Writer Schema 與 Reader Schema 逐欄位比對，套用 Resolution Rule：

- Reader 有、Writer 沒有的欄位 → 使用 Reader Schema 的 default value（若無 default，讀取失敗）。
- Writer 有、Reader 沒有的欄位 → 忽略。
- 欄位改名 → 透過 alias 比對。
- 型別 promotion（例如 int → long → float → double）→ 自動轉換，其餘型別不相容視為 breaking change。

這個機制讓 Consumer 不需要為每個歷史 Object 各自處理格式差異——Resolution 在讀取當下自動完成。問題是：**Reader 要從哪裡拿到 Writer Schema？** 這在 Object Storage（檔案）與 Message Queue（單筆訊息）兩種情境下答案不同。

## Object Storage：Writer Schema 內嵌於檔案

Object 若寫成 Avro OCF（Object Container File），Writer Schema 直接內嵌在檔案 header，Reader 打開檔案就能還原，不需要外部查詢。Ingestion Pipeline 這一段的 Object（`object_path` 指向的檔案）屬於這個情境，Schema 傳遞不是問題。

## Message Queue：單筆訊息不含完整 Schema

Consumer 從 MQ 讀到的是一筆一筆的訊息，不是 OCF，若每筆訊息都內嵌完整 Avro Schema（JSON，數 KB）在高吞吐下會浪費大量頻寬，因此業界作法是傳一個「指標」而非整份 schema：

- **Confluent Wire Format**：payload 前綴 `magic byte (1) + schema id (4 bytes)`，Reader 用這個 4-byte id 向 Schema Registry 查完整 schema（通常有 client-side cache，非每筆都打 API）。
- **Avro Single Object Encoding**（Avro 官方規範，不綁定特定 Registry）：payload 前綴 `0xC3 0x01`（marker）+ 8-byte schema fingerprint（schema 內容的 hash，而非中心化配發的 id）。Reader 拿 fingerprint 對應到本地或自建的 schema 表，不強制要有 Registry 服務。

## 沒有 Schema Registry 或拿不到 Schema ID 時

若第三方系統不配合、維運上不想維護一套 Confluent 等級的 Registry，或流量太大不想為了 schema 查詢多打一次網路請求，可用的替代方案：

- **Message Header / Attribute 帶版本或路徑**：payload 本身不動，另外在 MQ 的 header/attribute 帶 `schema_version` 或 `schema_url`（例如指向 GCS/S3 上的 `.avsc`），Reader 先讀 header 再取對應 schema，不依賴 Schema ID 這個中間層。
- **自訂 1-byte protocol version 前綴**：雙方約定 payload 第一個 byte 代表版本，Reader 依版本對應到程式內建（hardcoded）或啟動時載入的 schema，overhead 壓到最低。
- **雲端原生託管服務**：GCP Pub/Sub 可直接把 Schema 綁在 Topic 上，由平台驗證與傳遞；AWS Glue Schema Registry 是 serverless、按呼叫計費，不需要自架叢集。
- **輕量開源 Registry**：Apicurio Registry 可用 in-memory 或既有 PostgreSQL 作後端，比 Confluent Schema Registry（通常綁定 Kafka/ZooKeeper）更輕。
- **自建（GCS/S3 + 版本控制）**：`.avsc` 檔案用 Git 管理版本，CI/CD 同步發布到 GCS/S3，Producer/Consumer 啟動時抓取對應版本，省去獨立服務。

這些方案本質上都是同一件事：把「Reader 如何找到 Writer Schema」這條鏈路從「查詢一個中心化 Registry」換成「查詢一個更輕量的位置（header、固定路徑、本地快取）」，Resolution Rule 本身不變。

## Control Plane 需要保存什麼

不論用哪種方案取得 Writer Schema，`batch_catalog` 都需要額外保存 `writer_schema_id`（Registry 版本號、schema fingerprint，或自訂 protocol version），理由：

- Writer Schema 內嵌於 Object 或需額外查詢才能取得，都只能在打開 Object／訊息之後才知道，無法用來做 catalog 層級查詢（例如「找出所有用 schema v3 寫入的 batch」）。
- 比對 `writer_schema_id` 遠比比對整份 schema JSON 輕量。

跟 `flush_reason` 一樣，這是「事後可查詢的 metadata」，不需要打開 Object 就能知道某個 batch 用了哪個 schema。

## 對 Replay 的影響

Replay 產生的 Object 必須與原始 Object bit-for-bit 相同，因此 Replay 使用的是**當時的 Writer Schema**，不是 Replay 當下 Consumer 手上最新的 Reader Schema：

- 若 Replay 誤用目前最新 schema 序列化，即使欄位語意相同，也可能改變欄位順序、default value 或型別，導致 checksum 不一致。
- 因此 `writer_schema_id` 與 `flush_reason`、`offset_start`/`offset_end` 一樣，必須被 Batch Metadata 保存並在 Replay 時原樣沿用，而不是重新解析當下的 schema。

## Schema Compatibility 檢查

允許上游變更 schema 前，應強制檢查 Compatibility Mode（通常是 `BACKWARD`：新 Reader Schema 要能讀舊 Writer Schema），確保：

- 舊 Object（舊 Writer Schema）仍可被新版 Consumer 的 Reader Schema 正確 Resolve。
- 新增欄位必須提供 default value，否則舊 Object 讀取時該欄位無法決定值。

若 Compatibility 檢查未通過就允許 schema 變更，Resolution 會在讀取舊 Object 時失敗——這不是 Ingestion Pipeline 該處理的問題，而是 Schema 治理層（不論是否有 Registry）該擋下的問題。

---

# Data Lineage

Control Plane 建立資料關係：

```
Message -> Batch -> Object -> Dataset
```

可以回答：Object 來源是哪一批資料、Object 包含哪些 source position、哪一次 ingestion run 產生。

---

# 與 Durable Execution 的關係

參考：[Durable execution with Apache Airflow（Astronomer）](https://www.astronomer.io/blog/durable-execution-with-apache-airflow/)

本節先整理文章的內容，再說明它與本文設計的對應。

## Durable Execution 是什麼

執行程式的 process 中途掛掉後，可以由另一個 process 接手，從「最後一次記錄的結果」繼續，而不是從頭開始。文章指出這不只是「crash proof execution」（Temporal 的說法）。

**核心限制：Durable Execution 是 at-least-once，不是 exactly-once。** 提交 job、付款這類 side effect 仍需要冪等，做法是 idempotency key、穩定的 job ID 或 status check。

## 邊界放在哪裡（Four places to put the boundary）

> Durable execution is a promise about recovery, not a single implementation.

各家系統的差別，主要在「進度在哪一個單位變成持久的」：整個 function、function 內一個命名的 step、state machine 的一個 state，或一次 task attempt。文章整理成四種：

**1. 重播的 orchestration code**（Temporal、Azure Durable Functions）

- 邊界：整個 orchestration function。
- 恢復：重新執行 orchestration code，並從歷史紀錄餵回先前的結果。
- 實作方式：每做完一件事，就把結果追加到一份 history。掛掉後從頭重跑 code，遇到 history 裡已有的結果就直接取用，不真的執行，直到追上最後一筆才開始執行新的工作。
- 代價：orchestration code 必須是 deterministic，否則重跑時會走到和 history 對不上的路徑。

**2. function 內有 checkpoint 的 step**（Restate、Inngest、DBOS、Cloudflare Workflows）

- 邊界：function 內一個命名的 step。
- 恢復：重新呼叫 function，已完成的 step 直接回傳存好的結果。
- 實作方式：把每個有 side effect 的動作包成 step，並給它一個身分。執行前先查有沒有這個身分的結果，有就直接回傳，沒有才執行並存下結果。
- 代價：side effect 要放進 step，step 的身分必須保持有效。

**3. 宣告式 state machine**（AWS Step Functions）

- 邊界：state machine 的一個 state。
- 恢復：服務根據儲存的 state 往下推進，不重播你的程式碼。
- 實作方式：用服務規定的語言宣告有哪些 state 與轉移條件，每個 state 的工作交給外部執行，由服務負責記錄目前在哪個 state。
- 代價：控制流程受限於服務的 state machine 語言。

**4. 明確的 task checkpoint**（Airflow）

- 邊界：一次 task attempt。
- 恢復：retry 讀取已存的 state，或重新接上已存的外部工作。
- 實作方式：Airflow 3.3 有兩種寫法。
  - 外部 job：operator 內建，`durable=True`（預設），operator 在等待前先記錄 job 的身分。
  - 自己的程式碼：在 task 內呼叫 `task_state_store.set()` 記錄進度，retry 時用 `task_state_store.get()` 讀回來。
- 代價：程式只能從「有記錄」的 state 繼續。

上面第 1 到 3 種的「實作方式」，是我依文章對恢復方式的描述，整理出的概念示意，不是各產品的實際 API，文章沒有逐一說明它們的內部做法。第 4 種的寫法來自文章。

文章補充兩點：

- 這四種描述的是開發者看到的介面，不是四種獨立的實作。Checkpoint step 的系統同樣會重播 function，只是要你標出哪些較小的單位，其結果必須保留。
- 四種都有同一個限制：「None of them can make an external side effect and its own state update a single atomic write」，除非外部系統加入同一個 transaction。所以不論選哪一種，idempotency key、穩定的 job ID 和 status check 都仍然需要。

## Airflow 3.3 的做法

3.3 之前，Airflow 只在 task 之間是 durable：清除失敗的 run，會從失敗的 task 繼續。但執行 task 的 worker 若在 task 執行到一半時掛掉，retry 會把整個 task 從頭重跑。3.3 把 durability 推進到 task 內部。

文章把 task 分成**兩種形態**：

> what follows comes in two shapes rather than one. Operators that already know how to reconnect to the specific system they talk to, and a plain API for when the long-running thing is your own code.

### 形態 1：把工作交給外部系統的 task

文章的前提是：Airflow 的 task 通常不做實際的工作，而是提交 Spark job、啟動 pod、對 warehouse 執行 query。

**要解決的情境**：Airflow 的 worker（執行這個 task 的 process）把一個需要兩小時的 job 提交到 Spark 叢集後，正在等待 job 完成時，worker 自己掛掉。Spark 叢集是獨立的系統，不受影響，job 仍在叢集上繼續執行，但已經沒有任何 process 在等它。

兩個版本的前半段相同：worker 向 Spark 叢集提交 job，叢集回傳一個 job id（例如 `job-A`）。差別在於這個 `job-A` 有沒有被存到 worker 以外的地方。

| 步驟 | 3.3 之前 | 3.3 之後 |
|---|---|---|
| 1. 提交 job | worker 1 提交，拿到 `job-A` | worker 1 提交，拿到 `job-A` |
| 2. 開始等待之前 | `job-A` 只存在 worker 1 的記憶體 | operator 把 `job-A` 寫進 task state store（存在 Airflow 的 metadata database） |
| 3. worker 1 掛掉 | `job-A` 跟著消失，Spark 上的 `job-A` 仍在執行 | 記憶體裡的 `job-A` 消失，但 task state store 裡仍有 |
| 4. Airflow 用 worker 2 重跑（retry） | worker 2 沒有任何紀錄，只能重新提交，產生 `job-B` | worker 2 先讀 task state store，讀到 `job-A`，不重新提交 |
| 5. 結果 | `job-A` 和 `job-B` 同時寫同一張表，要付兩份費用 | worker 2 向 Spark 叢集查詢 `job-A` 的狀態，再依下表處理 |

3.3 之前若要避免重複，只能自己把工作拆成 submit task 和 wait task，讓 job id 存在 XCom 或 Variable。3.3 之後，operator 內建這個做法，不需要自己拆。

worker 2 查詢 `job-A` 的狀態後，依結果分三種處理：

| job 狀態 | retry 的行為 |
|---|---|
| 還在跑 | 重新接上（reattach），繼續 polling |
| 已成功 | 跳過提交和 polling，直接回傳結果 |
| 已失敗 | 重新提交 |

各 operator 記錄的「身分」，就是之後能向外部系統找回這個 job 的識別資訊，依外部系統而不同。以兩個為例：

| Operator | 記錄的內容 | retry 時怎麼接回 |
|---|---|---|
| `BigQueryInsertJobOperator` | BigQuery 的 job id | 用 job id 向 BigQuery 查詢該 job 的狀態 |
| `KubernetesPodOperator` | pod name 和 namespace | 找到同一個 pod 接回，不再啟動新的 pod |

其他 operator（Spark、Databricks、Snowflake、Redshift、Glue、Livy）也是同樣的做法，各自記錄該系統的 job id、run id 或 statement id。

- **機制**：operator 內建，由 `durable` 控制，3.3 預設為 `True`（「For external jobs you opt out」）。
- **限制**：只涵蓋同步路徑。`deferrable=True` 時由 Triggerer 追蹤 job，`durable` 不起作用。

### 形態 2：自己在 process 內做工作的 task

沒有 external job 可以 reattach，要靠記錄進度來恢復。文章再分兩個情況：

- **自己的程式碼**：用 `task_state_store` 的 `get` 和 `set` 記錄 job id、batch offset、checkpoint 等，retry 時從最後記錄的位置繼續。例如記錄 batch offset，讓 retry 從 record 8,001 開始。
- **AI agent**：`@task.agent` 或 `AgentOperator` 設定 `durable=True`，每個 model response 和 tool result 完成時就快取。retry 時重播已完成的步驟，只對未完成的步驟真的呼叫。stored fingerprint 會在 request 改變時丟棄過期的步驟。這是 opt-in（「For agents you opt in」），且需要設定 retries。

### State 的儲存（AIP-103）

| 種類 | 範圍 | 用途 |
|---|---|---|
| Task state store | 單一 task instance（dag_id、run_id、task_id、map_index），retry 之間存活，下一個 run 就沒有了 | 當次 attempt 的 in-flight 進度 |
| Asset state store | 單一 asset，跨 run 存活，其他 DAG 可讀取 | watermark、offset |

- XComs 在每次 retry 時會被清除，task state 則會保留。
- 後端可替換，預設為 metadata database，較大的值可放 object storage。
- 值必須能轉成 JSON。
- Asset state 是「makes the next run cheaper rather than the current attempt survivable」，不屬於 crash recovery。

### 文章的設計原則

- **存指標，不存內容**：存 checkpoint 的 URI 和 epoch，不存 model weights。
- **外部寫入完成後才記錄**：先記錄再寫入，會把寫到一半的檔案當成已完成的 checkpoint。
- **優先用現成 operator**：submit 加 sensor 的拆法，submit task 綠燈只代表 API call 成功，不代表 job 完成。
- **Durability 取決於目標系統**：Spark driver、pod、query 的 reattach 方式各不相同。
- **State 用於 in-flight 的工作，不當作歷史紀錄**。

## 對照本文的設計

### 本文把邊界放在哪裡

回到「進度在哪一個單位變成持久的」這個問題：本文的答案是 **Batch**。一個 Batch 對應一個 Object，`batch_catalog` 的一筆 row 就是一個持久化的單位，記錄它的範圍（`offset_start`、`offset_end`）、產生條件（`flush_reason`）、使用的 schema（`writer_schema_id`）和狀態。

以下是依文章的四種分類所做的判斷，不是文章本身的結論：

- **第 4 種（明確的 task checkpoint）：最接近。** Consumer 的恢復完全依賴自己記錄的 state，沒有任何框架替它保存進度。實作方式是 Consumer 在 `batch_catalog` 寫入 Batch Metadata，重啟時查詢 `COMMITTED` 的最大 `offset_end`，從下一筆開始讀（[Consumer Position 管理](#consumer-position-管理)）。
- **第 3 種（宣告式 state machine）：只有形狀相似。** `CREATED` → `WRITING` → `COMMITTED` / `FAILED` 是 state machine，但沒有服務替我們推進，每個狀態的 Recovery 邏輯（[Failure Recovery](#failure-recovery)）要自己寫。
- **第 1、2 種（重播與 checkpoint step）：Replay 借用了它們的做法。** 把先前的結果（Batch Boundary）存下來，重播時沿用，不重新計算。實作方式是 Replay 讀取已保存的 `offset_start`、`offset_end`、`flush_reason` 和 `writer_schema_id`，不重新決定 flush，直接重新產生 Object。

第 4 種的代價是「程式只能從有記錄的 state 繼續」，這直接解釋了 Rule 3：**沒有記錄下來的決定，就無法恢復。** Flush 的時間點、當下的 buffer size 都沒有被記錄，所以 Replay 時就不能依賴它們，只能用已保存的 Batch Boundary。

文章指出的「外部 side effect 與自己的 state 更新無法成為單一 atomic write」，在本文的對應是：Object Storage 的寫入與 Metadata Store 的更新，是兩個不同的系統。要消除這個縫隙，只有兩條路：讓外部系統加入同一個 transaction，或用冪等補償。本文採用後者，也就是 `WRITING` 狀態搭配 checksum 驗證。

### 本文屬於哪一種形態

- **Consumer 是形態 2**：自己在 process 內處理資料，沒有 external job，所以用 Metadata Store 記錄進度。這和 `task_state_store` 加 batch offset 是同一類做法，差別只在把 state 放在自己的 `batch_catalog`，不放在 Airflow。
- **Object write 是形態 1 的縮影**：外部有 Object Storage，所以 `WRITING` 狀態搭配 checksum，先檢查外部是否已有結果，再決定接上或重做，對應 operator 的 reattach 三種分支。

### 設計對應

| 文章的原則或機制 | 本文的對應 | 章節 |
|---|---|---|
| 狀態外部化，process 換掉仍能接手 | Metadata Store 是 Source of Truth | Design Rules（Rule 4） |
| 從最後一次記錄的結果繼續 | 查詢 `MAX(offset_end)`（`state = 'COMMITTED'`），從 `offset_end + 1` 開始讀取 | [Consumer Position 管理](#consumer-position-管理) |
| 外部寫入完成後才記錄 | Object write 和 Metadata update 成功後，才 Commit Message Offset | [Ingestion Flow](#ingestion-flow) |
| 存指標，不存內容 | `batch_catalog` 只存 `object_path`、`checksum`、offset 範圍 | [Metadata Store](#metadata-store) |
| 重試時 reattach，依外部狀態分支 | 依 Batch State 決定 Recovery，並驗證 Object 的 checksum | [Failure Recovery](#failure-recovery) |
| at-least-once 搭配冪等 | 以 `partition_id + offset_start` 去重 | [Failure Recovery](#failure-recovery) |
| Asset state 存 watermark 和 offset | `batch_catalog` 中 `COMMITTED` 的 `offset_end` | [Consumer Position 管理](#consumer-position-管理) |
| 外部系統的進度不當作真相 | MQ offset commit 只是 best-effort | Design Rules（Rule 5） |

### Replay 與 Durable Execution 的重播是同一個概念

文章中 AI agent 的重播，是把不確定的 model response 快取下來，retry 時沿用，不重新呼叫。本文的 Replay 是同一個模式：flush 的時機是不確定的決定，所以保存 Batch Boundary（offset 範圍與 `flush_reason`），Replay 時沿用，不重新決定（Rule 3）。

剩下的差異只有三點：

- **觸發原因**：文章的重播用於失敗後的 retry。本文的 Replay 也用於已成功之後的 Backfill 和 Reprocess，由人主動觸發。
- **驗證精度**：本文要求產出的 Object 用 `checksum` 驗證 bit-for-bit 相同，所以連 `writer_schema_id` 也要保存。文章沒有討論到這個精度。
- **保存期限**：本文的 `batch_catalog` 是長期的歷史紀錄，同時是 lineage、backfill 與 schema 的紀錄。文章建議 task state 不當作歷史紀錄。

### 文章指出的縫隙，本文同樣存在

文章承認：「A crash can always leave the durable execution system unsure whether the external call landed.」也就是 worker 在「已向外部系統提交 job」和「把 job id 記錄下來」這兩個動作之間掛掉，retry 的新 worker 找不到 id，仍可能重複提交。本文在 `CREATED` 到 `WRITING` 之間，Consumer 失敗的情況相同，靠 [Failure Recovery](#failure-recovery) 補上：檢查 Object 是否存在，並以 `checksum` 驗證。

### 搭配 Airflow 時的注意事項

- **分工不混用**：Airflow 的 task state store 管單次 attempt 的 in-flight 進度，`batch_catalog` 管跨 run 的永久 Source of Truth。Airflow 的 state store 不取代 Metadata Store。
- **不能假設所有 operator 都有 durable**：以 `DbtCloudRunJobOperator` 為例，文章沒有提到它，provider changelog 也沒有 durable 或 reattach 的紀錄。[PR #72423](https://github.com/apache/airflow/pull/72423) 曾提出：Airflow worker 掛掉後，retry 重新接上 dbt Cloud 上仍在執行的 run。但該 PR 已 CLOSED，沒有 merge。所以同步模式下，Airflow worker 掛掉後，retry 的新 worker 仍可能向 dbt Cloud 再提交一個 run，需要自己處理冪等。Local dbt 的情況則查不到資料。

---

# Design Rules

## Rule 1

Data Plane 保存資料，Control Plane 保存 metadata。

## Rule 2

Object Storage 不作為 Metadata Database。

Object Storage 適合：Large object storage、Immutable data storage。

不適合：Batch state management、Workflow state query。

## Rule 3

Replay 使用已保存的 Batch Metadata，不重新決定 Batch Boundary。

## Rule 4

Metadata Store 是 Ingestion Pipeline 的 Source of Truth。

Object Storage 保存資料結果，Metadata Store 保存資料產生過程。

## Rule 5

MQ 自身的 offset commit 不是 Source of Truth，只是 best-effort 優化。

Consumer 的起始讀取位置一律由 Metadata Store 的 `COMMITTED` Batch 決定，重複讀到的範圍以 `partition_id + offset_start` 去重。

## Rule 6

Batch Metadata 保存 `writer_schema_id`，Replay 使用當時的 Writer Schema，不使用 Replay 當下最新的 Reader Schema。

Schema Resolution（Reader Schema 讀 Writer Schema）交由 Avro 機制本身處理，Ingestion Pipeline 只負責保存「用了哪個 schema」，不負責重新解析。
