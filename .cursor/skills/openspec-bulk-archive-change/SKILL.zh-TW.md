---
name: openspec-bulk-archive-change
description: 一次封存多個已完成的變更。當要封存多個並行變更時使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.0.2"
---

一次作業封存多個已完成的變更。

此 skill 可批次封存變更，並透過檢查程式庫判斷實際實作內容，智慧處理規格衝突。

**輸入**：無需輸入（會提示選擇）

**步驟**

1. **取得進行中變更**

   執行 `openspec list --json` 取得所有進行中變更。

   若無進行中變更，告知使用者並停止。

2. **提示選擇變更**

   使用 **AskUserQuestion 工具** 多選讓使用者選擇變更：
   - 顯示每個變更與其 schema
   - 包含「全部變更」選項
   - 允許選任意數量（1 個以上即可，典型為 2 個以上）

   **重要**：不要自動選取。一律讓使用者選擇。

3. **批次驗證——收集所有選取變更的狀態**

   對每個選取的變更收集：

   a. **產物狀態**：執行 `openspec status --change "<name>" --json`
      - 解析 `schemaName` 與 `artifacts` 列表
      - 註記哪些產物為 `done` 或其他狀態

   b. **任務完成**：讀取 `openspec/changes/<name>/tasks.md`
      - 統計 `- [ ]`（未完成）與 `- [x]`（已完成）
      - 若無任務檔，註記為「No tasks」

   c. **Delta specs**：檢查 `openspec/changes/<name>/specs/` 目錄
      - 列出存在哪些 capability spec
      - 對每個提取需求名稱（符合 `### Requirement: <name>` 的行）

4. **偵測規格衝突**

   建立 `capability -> [觸及它的變更]` 對照：

   ```
   auth -> [change-a, change-b]  <- 衝突（2+ 變更）
   api  -> [change-c]            <- 正常（僅 1 變更）
   ```

   當 2 個以上選取變更對同一 capability 有 delta spec 時即存在衝突。

5. **以 agent 方式解決衝突**

   **對每個衝突**，調查程式庫：

   a. **讀取各衝突變更的 delta specs**，了解各自聲稱的新增/修改

   b. **搜尋程式庫** 找實作證據：
      - 找實作各 delta spec 需求的程式碼
      - 檢查相關檔案、函式或測試

   c. **決定解決方式**：
      - 若僅一個變更實際有實作 → 同步該變更的 specs
      - 若兩者皆有實作 → 依時間順序套用（先舊後新，新覆寫舊）
      - 若兩者皆無實作 → 略過 spec 同步，警告使用者

   d. **記錄每個衝突的解決**：
      - 要套用哪個變更的 specs
      - 順序（若兩者都要）
      - 理由（程式庫中發現的內容）

6. **顯示彙總狀態表**

   顯示彙總所有變更的表格：

   ```
   | 變更               | 產物   | 任務 | Specs   | 衝突     | 狀態   |
   |---------------------|--------|------|---------|----------|--------|
   | schema-management  | Done   | 5/5  | 2 delta | 無       | Ready  |
   | project-config     | Done   | 3/3  | 1 delta | 無       | Ready  |
   | add-oauth          | Done   | 4/4  | 1 delta | auth (!) | Ready* |
   | add-verify-skill   | 剩 1   | 2/5  | 無      | 無       | Warn   |
   ```

   若有衝突，顯示解決方式：
   ```
   * 衝突解決：
     - auth spec：將依序套用 add-oauth 再 add-jwt（兩者皆已實作，依時間序）
   ```

   若有未完成變更，顯示警告：
   ```
   警告：
   - add-verify-skill：1 個未完成產物，3 個未完成任務
   ```

7. **確認批次操作**

   使用 **AskUserQuestion 工具** 單一確認：

   - 「封存 N 個變更？」依狀態提供選項
   - 選項可包含：
     - 「封存全部 N 個變更」
     - 「僅封存 N 個就緒變更（略過未完成）」
     - 「取消」

   若有未完成變更，明確說明它們會帶警告一併封存。

8. **對每個確認的變更執行封存**

   依決定的順序處理（遵守衝突解決）：

   a. **若有 delta specs 則同步規格**：
      - 採用 openspec-sync-specs 做法（agent 驅動智慧合併）
      - 有衝突時依解決順序套用
      - 記錄是否已同步

   b. **執行封存**：
      ```bash
      mkdir -p openspec/changes/archive
      mv openspec/changes/<name> openspec/changes/archive/YYYY-MM-DD-<name>
      ```

   c. **記錄每個變更的結果**：
      - 成功：已成功封存
      - 失敗：封存過程錯誤（記錄錯誤）
      - 略過：使用者選擇不封存（若適用）

9. **顯示摘要**

   顯示最終結果：

   ```
   ## 批次封存完成

   已封存 3 個變更：
   - schema-management-cli -> archive/2026-01-19-schema-management-cli/
   - project-config -> archive/2026-01-19-project-config/
   - add-oauth -> archive/2026-01-19-add-oauth/

   略過 1 個變更：
   - add-verify-skill（使用者選擇不封存未完成者）

   規格同步摘要：
   - 4 個 delta spec 已同步至主規格
   - 1 個衝突已解決（auth：依時間序套用兩者）
   ```

   若有失敗：
   ```
   失敗 1 個變更：
   - some-change：封存目錄已存在
   ```

**衝突解決範例**

範例 1：僅一個有實作
```
衝突：specs/auth/spec.md 被 [add-oauth, add-jwt] 觸及

檢查 add-oauth：
- Delta 新增「OAuth Provider Integration」需求
- 搜尋程式庫…找到 src/auth/oauth.ts 實作 OAuth 流程

檢查 add-jwt：
- Delta 新增「JWT Token Handling」需求
- 搜尋程式庫…未找到 JWT 實作

解決：僅 add-oauth 有實作。只同步 add-oauth 的 specs。
```

範例 2：兩者皆有實作
```
衝突：specs/api/spec.md 被 [add-rest-api, add-graphql] 觸及

檢查 add-rest-api（建立於 2026-01-10）：
- Delta 新增「REST Endpoints」需求
- 搜尋程式庫…找到 src/api/rest.ts

檢查 add-graphql（建立於 2026-01-15）：
- Delta 新增「GraphQL Schema」需求
- 搜尋程式庫…找到 src/api/graphql.ts

解決：兩者皆有實作。先套用 add-rest-api 的 specs，
再套用 add-graphql 的 specs（依時間序，新的優先）。
```

**成功時輸出**

```
## 批次封存完成

已封存 N 個變更：
- <change-1> -> archive/YYYY-MM-DD-<change-1>/
- <change-2> -> archive/YYYY-MM-DD-<change-2>/

規格同步摘要：
- N 個 delta spec 已同步至主規格
- 無衝突（或：M 個衝突已解決）
```

**部分成功時輸出**

```
## 批次封存完成（部分）

已封存 N 個變更：
- <change-1> -> archive/YYYY-MM-DD-<change-1>/

略過 M 個變更：
- <change-2>（使用者選擇不封存未完成者）

失敗 K 個變更：
- <change-3>：封存目錄已存在
```

**無變更時輸出**

```
## 無變更可封存

未找到進行中變更。使用 `/opsx:new` 建立新變更。
```

**護欄**
- 允許任意數量變更（1 個以上即可，典型為 2 個以上）
- 一律提示選擇，絕不自動選取
- 及早偵測規格衝突並透過檢查程式庫解決
- 兩變更皆有實作時，依時間順序套用 specs
- 僅在缺少實作時略過 spec 同步（並警告使用者）
- 確認前清楚顯示每個變更的狀態
- 整批使用單一確認
- 追蹤並回報所有結果（成功/略過/失敗）
- 移至封存時保留 .openspec.yaml
- 封存目錄目標使用目前日期：YYYY-MM-DD-<name>
- 若封存目標已存在，該變更失敗但繼續處理其他變更
