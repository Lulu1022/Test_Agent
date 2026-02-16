---
name: openspec-apply-change
description: 實作 OpenSpec 變更中的任務。當使用者要開始實作、繼續實作或依任務執行時使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.0.2"
---

根據 OpenSpec 變更實作任務。

**輸入**：可選指定變更名稱。若未提供，檢查是否可從對話脈絡推斷。若模糊或不明確，必須提示使用者選擇可用變更。

**步驟**

1. **選擇變更**

   若已提供名稱則使用。否則：
   - 若使用者提到某變更，從對話脈絡推斷
   - 若僅有一個進行中變更，可自動選取
   - 若不明確，執行 `openspec list --json` 取得可用變更，並用 **AskUserQuestion 工具** 讓使用者選擇

   一律宣告：「使用變更：<name>」以及如何覆寫（例如 `/opsx:apply <other>`）。

2. **檢查狀態以了解 schema**
   ```bash
   openspec status --change "<name>" --json
   ```
   解析 JSON 以了解：
   - `schemaName`：使用的流程（例如 "spec-driven"）
   - 哪個產物包含任務（通常 spec-driven 為 "tasks"，其他 schema 請看 status）

3. **取得 apply 指示**

   ```bash
   openspec instructions apply --change "<name>" --json
   ```

   回傳內容包括：
   - 脈絡檔案路徑（依 schema 而異，可能是 proposal/specs/design/tasks 或 spec/tests/implementation/docs）
   - 進度（總數、已完成、剩餘）
   - 任務列表與狀態
   - 依目前狀態的動態指示

   **狀態處理：**
   - 若 `state: "blocked"`（缺少產物）：顯示訊息，建議使用 openspec-continue-change
   - 若 `state: "all_done"`：恭喜並建議封存
   - 否則：繼續實作

4. **讀取脈絡檔案**

   從 apply 指示輸出中的 `contextFiles` 讀取所列檔案。
   檔案依 schema 而異：
   - **spec-driven**：proposal、specs、design、tasks
   - 其他 schema：依 CLI 輸出的 contextFiles

5. **顯示目前進度**

   顯示：
   - 使用的 schema
   - 進度：「N/M 任務已完成」
   - 剩餘任務概覽
   - CLI 的動態指示

6. **實作任務（迴圈直到完成或受阻）**

   對每個待辦任務：
   - 顯示正在處理的任務
   - 進行所需程式碼變更
   - 變更保持最小且聚焦
   - 在 tasks 檔案中將任務標為完成：`- [ ]` → `- [x]`
   - 繼續下一項任務

   **以下情況暫停：**
   - 任務不清楚 → 請求釐清
   - 實作發現設計問題 → 建議更新產物
   - 發生錯誤或阻礙 → 回報並等待指示
   - 使用者中斷

7. **完成或暫停時顯示狀態**

   顯示：
   - 本階段完成的任務
   - 整體進度：「N/M 任務已完成」
   - 若全部完成：建議封存
   - 若暫停：說明原因並等待指示

**實作過程輸出**

```
## 實作中：<change-name>（schema：<schema-name>）

進行任務 3/7：<任務描述>
[...實作中...]
✓ 任務完成

進行任務 4/7：<任務描述>
[...實作中...]
✓ 任務完成
```

**完成時輸出**

```
## 實作完成

**變更：** <change-name>
**Schema：** <schema-name>
**進度：** 7/7 任務完成 ✓

### 本階段完成
- [x] 任務 1
- [x] 任務 2
...

所有任務已完成！可以封存此變更。
```

**暫停時輸出（遇到問題）**

```
## 實作暫停

**變更：** <change-name>
**Schema：** <schema-name>
**進度：** 4/7 任務完成

### 遇到的問題
<問題描述>

**選項：**
1. <選項 1>
2. <選項 2>
3. 其他做法

你想怎麼做？
```

**護欄**
- 持續處理任務直到完成或受阻
- 開始前一律讀取脈絡檔案（來自 apply 指示輸出）
- 任務不明確時先暫停詢問再實作
- 實作發現問題時暫停並建議更新產物
- 程式碼變更保持最小且限於各任務範圍
- 每完成一項任務立即更新任務勾選框
- 遇錯誤、阻礙或需求不清時暫停，不要猜測
- 使用 CLI 輸出的 contextFiles，不要假設特定檔名

**流程整合**

此 skill 支援「對變更執行動作」模式：

- **可隨時呼叫**：產物未全完成時（若有任務）、部分實作後、與其他動作交錯進行皆可
- **允許更新產物**：若實作發現設計問題，建議更新產物——不鎖階段，流暢作業
