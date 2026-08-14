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

保存：Batch 資訊、Source position、Object mapping、Processing status。

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
    status          VARCHAR,
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
  -> Write Object Storage -> Update Batch Status -> Commit Message Offset
```

Commit Message Offset 必須在 Object write 成功、Metadata update 成功之後執行。

`Commit Message Offset` 是 best-effort 步驟：它只是為了讓 MQ 下次能從接近的位置開始讀取，藉此縮短重複讀取的範圍。它是否成功，不影響 Batch 是否視為完成——這由 Batch Status 決定。

---

# Consumer Position 管理

Consumer 啟動或重啟時，不採用 MQ 自身記錄的 committed offset 作為起始位置，而是向 Metadata Store 查詢：

```
SELECT MAX(offset_end)
FROM batch_catalog
WHERE partition_id = :partition_id
  AND status = 'COMMITTED'
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

狀態：`Batch Status: CREATED`

Recovery：找出未完成 Batch → 重新寫入 Object → 更新 Batch Status。

## Object Write 後失敗

狀態：`Object exists`、`Batch Status: WRITING`

Recovery：檢查 Object Metadata → 驗證 checksum → 更新 Batch Status。

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

Replay 不依賴：Current time、Current buffer size、Current flush condition。

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
WHERE b.status = 'COMMITTED'
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
