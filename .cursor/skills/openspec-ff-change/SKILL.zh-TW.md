---
name: openspec-ff-change
description: 快轉完成 OpenSpec 產物建立。當使用者想快速建立實作所需的所有產物、不必逐項建立時使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.0.2"
---

快轉產物建立——一次產生開始實作所需的一切。

**輸入**：使用者請求應包含變更名稱（kebab-case）或想建內容的描述。

**步驟**

1. **若無明確輸入，詢問想建什麼**

   使用 **AskUserQuestion 工具**（開放式，無預設選項）詢問：
   > 「你想做哪個變更？描述你想建或修的東西。」

   從描述推導 kebab-case 名稱（例如 "add user authentication" → `add-user-auth`）。

   **重要**：未了解使用者想建什麼前不要繼續。

2. **建立變更目錄**
   ```bash
   openspec new change "<name>"
   ```
   這會在 `openspec/changes/<name>/` 建立 scaffold 變更。

3. **取得產物建立順序**
   ```bash
   openspec status --change "<name>" --json
   ```
   解析 JSON 取得：
   - `applyRequires`：實作前需要的產物 ID 陣列（例如 `["tasks"]`）
   - `artifacts`：所有產物列表與狀態、依賴

4. **依序建立產物直到可 apply**

   使用 **TodoWrite 工具** 追蹤產物進度。

   依依賴順序迴圈（先處理無待辦依賴的產物）：

   a. **對每個 `ready` 的產物（依賴已滿足）**：
      - 取得指示：
        ```bash
        openspec instructions <artifact-id> --change "<name>" --json
        ```
      - 指示 JSON 包含：
        - `context`：專案背景（給你的約束，不要放進輸出）
        - `rules`：產物專屬規則（給你的約束，不要放進輸出）
        - `template`：輸出檔結構
        - `instruction`：此產物類型的 schema 指引
        - `outputPath`：產物寫入路徑
        - `dependencies`：需先讀取的已完成產物
      - 讀取已完成依賴檔案以取得脈絡
      - 以 `template` 為結構建立產物檔案
      - 遵守 `context` 與 `rules`，但不要複製進檔案
      - 簡短顯示進度：「✓ 已建立 <artifact-id>」

   b. **直到所有 `applyRequires` 產物完成**
      - 每建立一個產物後重新執行 `openspec status --change "<name>" --json`
      - 檢查 `applyRequires` 中每個產物 ID 在 artifacts 陣列是否皆為 `status: "done"`
      - 全部 done 時停止

   c. **若產物需要使用者輸入**（脈絡不清）：
      - 使用 **AskUserQuestion 工具** 釐清
      - 再繼續建立

5. **顯示最終狀態**
   ```bash
   openspec status --change "<name>"
   ```

**輸出**

完成所有產物後摘要：
- 變更名稱與位置
- 已建立產物列表與簡短說明
- 現況：「所有產物已建立！可開始實作。」
- 提示：「執行 `/opsx:apply` 或叫我實作以開始處理任務。」

**產物建立指引**

- 依 `openspec instructions` 的 `instruction` 欄位處理各產物類型
- schema 定義各產物應含內容——依其撰寫
- 建立新產物前先讀取依賴產物取得脈絡
- 以 `template` 為輸出檔結構填寫各節
- **重要**：`context` 與 `rules` 是給你的約束，不是檔案內容
  - 不要把 `<context>`、`<rules>`、`<project_context>` 區塊複製進產物
  - 它們指引撰寫，但不應出現在輸出中

**護欄**
- 建立 schema 的 `apply.requires` 定義之實作所需全部產物
- 建立新產物前一律先讀取依賴產物
- 脈絡嚴重不清時詢問使用者——但傾向做合理決策以保持節奏
- 若該名稱變更已存在，建議改為繼續該變更
- 每寫完一個產物確認檔案存在再進行下一個
