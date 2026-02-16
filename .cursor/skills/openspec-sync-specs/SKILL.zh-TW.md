---
name: openspec-sync-specs
description: 將變更的 delta specs 同步至主規格。當使用者想用 delta spec 的變更更新主規格、但不封存變更時使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.0.2"
---

將變更的 delta specs 同步至主規格。

此為 **agent 驅動** 操作——你會讀取 delta specs 並直接編輯主規格以套用變更。可進行智慧合併（例如只新增一個 scenario 而不複製整個 requirement）。

**輸入**：可選指定變更名稱。若未提供，檢查是否可從對話脈絡推斷。若模糊或不明確，必須提示使用者選擇可用變更。

**步驟**

1. **若未提供變更名稱，提示選擇**

   執行 `openspec list --json` 取得可用變更。使用 **AskUserQuestion 工具** 讓使用者選擇。

   顯示有 delta specs 的變更（在 `specs/` 目錄下）。

   **重要**：不要猜測或自動選取變更。一律讓使用者選擇。

2. **找出 delta specs**

   在 `openspec/changes/<name>/specs/*/spec.md` 尋找 delta spec 檔案。

   每個 delta spec 檔案包含如：
   - `## ADDED Requirements`：要新增的需求
   - `## MODIFIED Requirements`：對既有需求的修改
   - `## REMOVED Requirements`：要移除的需求
   - `## RENAMED Requirements`：要重新命名的需求（FROM:/TO: 格式）

   若未找到 delta specs，告知使用者並停止。

3. **對每個 delta spec，將變更套用到主規格**

   對每個在 `openspec/changes/<name>/specs/<capability>/spec.md` 有 delta spec 的 capability：

   a. **讀取 delta spec** 了解預期變更

   b. **讀取主規格** `openspec/specs/<capability>/spec.md`（可能尚不存在）

   c. **智慧套用變更**：

      **ADDED Requirements：**
      - 若主規格中無該需求 → 新增
      - 若已存在 → 更新以符合（視為隱含 MODIFIED）

      **MODIFIED Requirements：**
      - 在主規格中找到該需求
      - 套用變更——可能為：
        - 新增 scenario（不需複製既有）
        - 修改既有 scenario
        - 變更需求描述
      - 保留 delta 未提及的 scenarios/內容

      **REMOVED Requirements：**
      - 從主規格移除整個需求區塊

      **RENAMED Requirements：**
      - 找到 FROM 需求，重新命名為 TO

   d. **若 capability 尚不存在則建立新主規格**：
      - 建立 `openspec/specs/<capability>/spec.md`
      - 新增 Purpose 節（可簡短，標為 TBD）
      - 新增 Requirements 節並加入 ADDED 需求

4. **顯示摘要**

   套用所有變更後摘要：
   - 更新了哪些 capabilities
   - 做了哪些變更（需求新增/修改/移除/重新命名）

**Delta Spec 格式參考**

```markdown
## ADDED Requirements

### Requirement: New Feature
系統 SHALL 做某件新的事。

#### Scenario: Basic case
- **WHEN** 使用者做 X
- **THEN** 系統做 Y

## MODIFIED Requirements

### Requirement: Existing Feature
#### Scenario: New scenario to add
- **WHEN** 使用者做 A
- **THEN** 系統做 B

## REMOVED Requirements

### Requirement: Deprecated Feature

## RENAMED Requirements

- FROM: `### Requirement: Old Name`
- TO: `### Requirement: New Name`
```

**原則：智慧合併**

有別於程式化合併，你可做 **部分更新**：
- 要新增一個 scenario，只需在 MODIFIED 下包含該 scenario——不需複製既有 scenarios
- Delta 代表 *意圖*，不是整份替換
- 依判斷合理合併變更

**成功時輸出**

```
## 規格已同步：<change-name>

已更新主規格：

**<capability-1>**：
- 新增需求：「New Feature」
- 修改需求：「Existing Feature」（新增 1 個 scenario）

**<capability-2>**：
- 已建立新規格檔
- 新增需求：「Another Feature」

主規格已更新。變更仍為進行中——實作完成後再封存。
```

**護欄**
- 變更前先讀取 delta 與主規格
- 保留 delta 未提及的既有內容
- 有不清楚處先請求釐清
- 進行時顯示正在變更的內容
- 操作應為冪等——執行兩次結果相同
