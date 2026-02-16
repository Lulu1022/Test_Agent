---
name: openspec-continue-change
description: 繼續進行 OpenSpec 變更，建立下一個產物。當使用者想推進變更、建立下一個產物或繼續流程時使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.0.2"
---

透過建立下一個產物來繼續進行變更。

**輸入**：可選指定變更名稱。若未提供，檢查是否可從對話脈絡推斷。若模糊或不明確，必須提示使用者選擇可用變更。

**步驟**

1. **若未提供變更名稱，提示選擇**

   執行 `openspec list --json` 取得可用變更（依最近修改排序）。使用 **AskUserQuestion 工具** 讓使用者選擇要繼續的變更。

   呈現最近修改的前 3–4 個變更作為選項，顯示：
   - 變更名稱
   - Schema（來自 `schema` 欄位，若無則為 "spec-driven"）
   - 狀態（例如「0/5 任務」、「完成」、「無任務」）
   - 最近修改時間（來自 `lastModified` 欄位）

   將最近修改的變更標為「（建議）」，因多半是使用者想繼續的。

   **重要**：不要猜測或自動選取變更。一律讓使用者選擇。

2. **檢查目前狀態**
   ```bash
   openspec status --change "<name>" --json
   ```
   解析 JSON 了解目前狀態。回應包含：
   - `schemaName`：使用的流程 schema（例如 "spec-driven"）
   - `artifacts`：產物陣列與狀態（"done"、"ready"、"blocked"）
   - `isComplete`：是否所有產物皆完成的布林值

3. **依狀態行動**：

   ---

   **若所有產物皆完成（`isComplete: true`）**：
   - 恭喜使用者
   - 顯示最終狀態（含使用的 schema）
   - 建議：「所有產物已建立！現在可以實作此變更或封存。」
   - 停止

   ---

   **若有產物可建立**（status 顯示 `status: "ready"` 的產物）：
   - 從 status 輸出中選取第一個 `status: "ready"` 的產物
   - 取得其指示：
     ```bash
     openspec instructions <artifact-id> --change "<name>" --json
     ```
   - 解析 JSON。關鍵欄位：
     - `context`：專案背景（給你的約束，不要放進輸出）
     - `rules`：產物專屬規則（給你的約束，不要放進輸出）
     - `template`：輸出檔的結構
     - `instruction`：schema 專屬指引
     - `outputPath`：產物寫入路徑
     - `dependencies`：需先讀取的已完成產物
   - **建立產物檔案**：
     - 讀取已完成依賴檔案以取得脈絡
     - 以 `template` 為結構填寫各節
     - 撰寫時遵守 `context` 與 `rules`，但不要把它們複製進檔案
     - 寫入指示中的 output path
   - 顯示建立了什麼以及現在解鎖了什麼
   - 建立一個產物後即停止

   ---

   **若無產物為 ready（皆 blocked）**：
   - 在有效 schema 下不應發生
   - 顯示狀態並建議檢查問題

4. **建立產物後顯示進度**
   ```bash
   openspec status --change "<name>"
   ```

**輸出**

每次呼叫後顯示：
- 建立了哪個產物
- 使用的 schema 流程
- 目前進度（N/M 完成）
- 現在解鎖的產物
- 提示：「要繼續嗎？跟我說繼續或告訴我下一步。」

**產物建立指引**

產物類型與用途依 schema 而定。使用指示輸出中的 `instruction` 欄位了解要建立的內容。

常見產物模式：

**spec-driven schema**（proposal → specs → design → tasks）：
- **proposal.md**：若不清楚可詢問使用者。填寫 Why、What Changes、Capabilities、Impact。
  - Capabilities 節很重要——列出的每個 capability 都需要一個 spec 檔。
- **specs/<capability>/spec.md**：依 proposal 的 Capabilities 節為每個 capability 建立一個 spec（用 capability 名稱，不是變更名稱）。
- **design.md**：記錄技術決策、架構與實作方式。
- **tasks.md**：將實作拆成勾選任務。

其他 schema 請依 CLI 輸出的 `instruction` 欄位操作。

**護欄**
- 每次呼叫只建立一個產物
- 建立新產物前一律先讀取依賴產物
- 不跳過產物或打亂順序建立
- 脈絡不清時先詢問使用者再建立
- 寫入後確認產物檔案存在再標記進度
- 依 schema 的產物順序，不要假設特定產物名稱
- **重要**：`context` 與 `rules` 是給你的約束，不是檔案內容
  - 不要把 `<context>`、`<rules>`、`<project_context>` 區塊複製進產物
  - 它們指引你撰寫，但不應出現在輸出中
