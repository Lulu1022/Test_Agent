---
name: openspec-new-change
description: 使用實驗性產物流程開始新的 OpenSpec 變更。當使用者想用結構化逐步方式建立新功能、修復或修改時使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.0.2"
---

使用實驗性產物驅動方式開始新變更。

**輸入**：使用者請求應包含變更名稱（kebab-case）或想建內容的描述。

**步驟**

1. **若無明確輸入，詢問想建什麼**

   使用 **AskUserQuestion 工具**（開放式，無預設選項）詢問：
   > 「你想做哪個變更？描述你想建或修的東西。」

   從描述推導 kebab-case 名稱（例如 "add user authentication" → `add-user-auth`）。

   **重要**：未了解使用者想建什麼前不要繼續。

2. **決定流程 schema**

   除非使用者明確要求不同流程，否則使用預設 schema（省略 `--schema`）。

   **僅在使用者提到以下情況時使用不同 schema：**
   - 特定 schema 名稱 → 使用 `--schema <name>`
   - 「show workflows」或「what workflows」→ 執行 `openspec schemas --json` 讓其選擇

   **否則**：省略 `--schema` 使用預設。

3. **建立變更目錄**
   ```bash
   openspec new change "<name>"
   ```
   僅在使用者要求特定流程時才加 `--schema <name>`。
   這會在 `openspec/changes/<name>/` 以所選 schema 建立 scaffold 變更。

4. **顯示產物狀態**
   ```bash
   openspec status --change "<name>"
   ```
   顯示需建立的產物以及已就緒者（依賴已滿足）。

5. **取得第一個產物的指示**
   第一個產物依 schema 而定（例如 spec-driven 為 `proposal`）。
   從 status 輸出找到第一個 status 為 "ready" 的產物。
   ```bash
   openspec instructions <first-artifact-id> --change "<name>"
   ```
   會輸出建立第一個產物的 template 與 context。

6. **停止並等待使用者指示**

**輸出**

完成步驟後摘要：
- 變更名稱與位置
- 使用的 schema/流程與其產物順序
- 目前狀態（0/N 產物完成）
- 第一個產物的 template
- 提示：「準備建立第一個產物了嗎？描述這個變更的內容我就會起草，或叫我繼續。」

**護欄**
- 尚不要建立任何產物——只顯示指示
- 不要超過顯示第一個產物 template 的範圍
- 若名稱無效（非 kebab-case），請其提供有效名稱
- 若該名稱變更已存在，建議改為繼續該變更
- 使用非預設流程時傳入 --schema
