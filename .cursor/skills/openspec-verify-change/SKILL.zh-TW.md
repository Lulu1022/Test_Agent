---
name: openspec-verify-change
description: 驗證實作是否符合變更產物。當使用者想在封存前確認實作完整、正確且一致時使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.0.2"
---

驗證實作是否符合變更產物（specs、tasks、design）。

**輸入**：可選指定變更名稱。若未提供，檢查是否可從對話脈絡推斷。若模糊或不明確，必須提示使用者選擇可用變更。

**步驟**

1. **若未提供變更名稱，提示選擇**

   執行 `openspec list --json` 取得可用變更。使用 **AskUserQuestion 工具** 讓使用者選擇。

   顯示有實作任務的變更（存在 tasks 產物）。
   若有 schema 資訊一併顯示。
   未完成任務的變更標為「（進行中）」。

   **重要**：不要猜測或自動選取變更。一律讓使用者選擇。

2. **檢查狀態以了解 schema**
   ```bash
   openspec status --change "<name>" --json
   ```
   解析 JSON 了解：
   - `schemaName`：使用的流程（例如 "spec-driven"）
   - 此變更存在哪些產物

3. **取得變更目錄並載入產物**

   ```bash
   openspec instructions apply --change "<name>" --json
   ```

   會回傳變更目錄與脈絡檔案。從 `contextFiles` 讀取所有可用產物。

4. **初始化驗證報告結構**

   建立三維度報告結構：
   - **完整性**：追蹤任務與 spec 涵蓋
   - **正確性**：追蹤需求實作與 scenario 涵蓋
   - **一致性**：追蹤設計遵循與模式一致

   每個維度可有 CRITICAL、WARNING 或 SUGGESTION 問題。

5. **驗證完整性**

   **任務完成**：
   - 若 contextFiles 中有 tasks.md，讀取它
   - 解析勾選框：`- [ ]`（未完成）與 `- [x]`（已完成）
   - 統計已完成與總任務數
   - 若有未完成任務：
     - 每個未完成任務加一筆 CRITICAL 問題
     - 建議：「完成任務：<描述>」或「若已實作請標為完成」

   **Spec 涵蓋**：
   - 若 `openspec/changes/<name>/specs/` 有 delta specs：
     - 擷取所有需求（標記為 "### Requirement:"）
     - 對每個需求：
       - 在程式庫搜尋與需求相關的關鍵字
       - 評估是否可能有實作
     - 若需求看似未實作：
       - 加 CRITICAL 問題：「未找到需求：<需求名稱>」
       - 建議：「實作需求 X：<描述>」

6. **驗證正確性**

   **需求實作對應**：
   - 對 delta specs 的每個需求：
     - 在程式庫搜尋實作證據
     - 若有找到，註記檔案路徑與行範圍
     - 評估實作是否符合需求意圖
     - 若發現分歧：
       - 加 WARNING：「實作可能偏離規格：<細節>」
       - 建議：「對照需求 X 檢查 <file>:<lines>」

   **Scenario 涵蓋**：
   - 對 delta specs 的每個 scenario（標記為 "#### Scenario:"）：
     - 檢查程式是否處理條件
     - 檢查是否有涵蓋該 scenario 的測試
     - 若 scenario 看似未涵蓋：
       - 加 WARNING：「Scenario 未涵蓋：<scenario 名稱>」
       - 建議：「為 scenario 新增測試或實作：<描述>」

7. **驗證一致性**

   **設計遵循**：
   - 若 contextFiles 中有 design.md：
     - 擷取關鍵決策（尋找 "Decision:"、"Approach:"、"Architecture:" 等節）
     - 驗證實作是否遵循這些決策
     - 若發現矛盾：
       - 加 WARNING：「未遵循設計決策：<決策>」
       - 建議：「更新實作或修正 design.md 以符合現況」
   - 若無 design.md：略過設計遵循檢查，註記「無 design.md 可驗證」

   **程式模式一致**：
   - 檢查新程式與專案模式是否一致
   - 檢查檔名、目錄結構、程式風格
   - 若發現明顯偏差：
     - 加 SUGGESTION：「程式模式偏差：<細節>」
     - 建議：「考慮遵循專案模式：<範例>」

8. **產生驗證報告**

   **摘要計分**：
   ```
   ## 驗證報告：<change-name>

   ### 摘要
   | 維度     | 狀態           |
   |----------|----------------|
   | 完整性   | X/Y 任務, N 需求|
   | 正確性   | M/N 需求已涵蓋 |
   | 一致性   | 已遵循/問題    |
   ```

   **依優先順序的問題**：

   1. **CRITICAL**（封存前必須修）：
      - 未完成任務
      - 缺少需求實作
      - 每項附具體可執行建議

   2. **WARNING**（建議修）：
      - 規格/設計分歧
      - 缺少 scenario 涵蓋
      - 每項附具體建議

   3. **SUGGESTION**（可選修）：
      - 模式不一致
      - 小改進
      - 每項附具體建議

   **最終評估**：
   - 若有 CRITICAL：「發現 X 個嚴重問題。封存前請先修復。」
   - 若僅有 WARNING：「無嚴重問題。有 Y 個警告可考慮。可封存（並依註記改進）。」
   - 若全部通過：「所有檢查通過。可封存。」

**驗證啟發**

- **完整性**：聚焦客觀檢查項目（勾選框、需求列表）
- **正確性**：用關鍵字搜尋、檔案路徑分析、合理推論——不要求絕對確定
- **一致性**：找明顯不一致，不吹毛求疵
- **誤報**：不確定時，優先標 SUGGESTION 而非 WARNING，WARNING 而非 CRITICAL
- **可執行**：每個問題都須有具體建議，並在適用處附檔案/行參照

**優雅降級**

- 若僅有 tasks.md：只驗證任務完成，略過 spec/design 檢查
- 若有 tasks + specs：驗證完整性與正確性，略過 design
- 若產物完整：驗證三個維度
- 一律註記略過了哪些檢查及原因

**輸出格式**

使用清楚 markdown：
- 摘要計分用表格
- 問題分組列表（CRITICAL/WARNING/SUGGESTION）
- 程式參照格式：`file.ts:123`
- 具體可執行建議
- 避免「考慮檢視」等模糊建議
