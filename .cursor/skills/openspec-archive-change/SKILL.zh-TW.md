---
name: openspec-archive-change
description: 在實驗性流程中封存已完成的變更。當使用者想在實作完成後定案並封存變更時使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.0.2"
---

在實驗性流程中封存已完成的變更。

**輸入**：可選指定變更名稱。若未提供，檢查是否可從對話脈絡推斷。若模糊或不明確，必須提示使用者選擇可用變更。

**步驟**

1. **若未提供變更名稱，提示選擇**

   執行 `openspec list --json` 取得可用變更。使用 **AskUserQuestion 工具** 讓使用者選擇。

   只顯示進行中的變更（尚未封存者）。
   若有 schema 資訊，一併顯示各變更使用的 schema。

   **重要**：不要猜測或自動選取變更。一律讓使用者選擇。

2. **檢查產物完成狀態**

   執行 `openspec status --change "<name>" --json` 檢查產物完成狀態。

   解析 JSON 以了解：
   - `schemaName`：使用的流程
   - `artifacts`：產物列表與狀態（`done` 或其他）

   **若有產物未為 `done`：**
   - 顯示警告並列出未完成產物
   - 使用 **AskUserQuestion 工具** 確認使用者仍要繼續
   - 使用者確認後再繼續

3. **檢查任務完成狀態**

   讀取任務檔案（通常為 `tasks.md`）檢查未完成任務。

   統計 `- [ ]`（未完成）與 `- [x]`（已完成）數量。

   **若發現未完成任務：**
   - 顯示警告與未完成任務數量
   - 使用 **AskUserQuestion 工具** 確認使用者仍要繼續
   - 使用者確認後再繼續

   **若無任務檔案：** 不顯示任務相關警告，直接繼續。

4. **評估 delta spec 同步狀態**

   檢查 `openspec/changes/<name>/specs/` 是否有 delta specs。若無，則不提示同步。

   **若有 delta specs：**
   - 將每個 delta spec 與對應主規格 `openspec/specs/<capability>/spec.md` 比較
   - 判斷會套用的變更（新增、修改、移除、重新命名）
   - 在提示前顯示合併摘要

   **提示選項：**
   - 若有變更需套用：「立即同步（建議）」、「不同步直接封存」
   - 若已同步：「立即封存」、「仍要同步」、「取消」

   若使用者選擇同步，執行 /opsx:sync 邏輯（使用 openspec-sync-specs skill）。不論選擇為何，之後皆可繼續封存。

5. **執行封存**

   若封存目錄不存在則建立：
   ```bash
   mkdir -p openspec/changes/archive
   ```

   以目前日期產生目標名稱：`YYYY-MM-DD-<change-name>`

   **檢查目標是否已存在：**
   - 若存在：回報錯誤，建議重新命名既有封存或使用不同日期
   - 若不存在：將變更目錄移至封存

   ```bash
   mv openspec/changes/<name> openspec/changes/archive/YYYY-MM-DD-<name>
   ```

6. **顯示摘要**

   顯示封存完成摘要，包含：
   - 變更名稱
   - 使用的 schema
   - 封存位置
   - 規格是否已同步（若適用）
   - 若有警告（未完成產物/任務）請註明

**成功時輸出**

```
## 封存完成

**變更：** <change-name>
**Schema：** <schema-name>
**封存至：** openspec/changes/archive/YYYY-MM-DD-<name>/
**規格：** ✓ 已同步至主規格（或「無 delta specs」或「已略過同步」）

所有產物已完成。所有任務已完成。
```

**護欄**
- 未提供變更時一律提示選擇
- 使用產物圖（openspec status --json）檢查完成狀態
- 有警告時不阻擋封存，僅告知並確認
- 移至封存時保留 .openspec.yaml（隨目錄一併移動）
- 清楚摘要發生的事
- 若要求同步，採用 openspec-sync-specs 做法（由 agent 驅動）
- 若有 delta specs，一律先執行同步評估並在提示前顯示合併摘要
