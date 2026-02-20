# .NET 10 Web API + EF Core 三層式架構開發說明

本專案為 **Web API** 專案，僅提供 REST API 端點。

## 一、架構概覽

三層式架構將職責分為：

| 層級 | 專案/資料夾 | 職責 | 依賴 |
|------|-------------|------|------|
| **表現層 (Presentation)** | **Controller** | 接收 HTTP 請求、回傳回應、驗證輸入 | 僅依賴 Business |
| **業務邏輯層 (Business)** | **Service** | 業務規則、流程、交易邊界 | 僅依賴 Data（介面） |
| **資料存取層 (Data)** | **Repository** | 資料庫存取、DbContext、實體對應 | 無依賴其他層 |

三層對應的**資料夾命名**：`Controller`、`Service`、`Repository`。

依賴方向：**API → Business → Data**（單向，不反向依賴）。

---

## 二、環境需求

- **.NET 10 SDK**  
  - 下載：https://dotnet.microsoft.com/download  
  - 確認：`dotnet --version` 顯示 10.x.x
- **IDE**：Visual Studio 2022 或 Cursor / VS Code（含 C# 擴充）
- **資料庫**：MySQL（本說明使用 MySQL）

---

## 三、方案與專案結構

建議的目錄結構（`src` 下含三層資料夾及 Entity、Dto）：

```
Test_Agent/
├── src/
│   ├── Controller/                 # 表現層：Web API 控制器
│   ├── Service/                    # 業務邏輯層（應用服務、介面）
│   ├── Repository/                 # 資料存取層（EF Core、DbContext、Repository 實作）
│   ├── Entity/                     # 與資料表對應的實體類別
│   └── Dto/                        # 請求/回應 DTO（不暴露實體到 API）
├── tests/                          # 測試專案集中放此，與 src 分離
│   └── TestAgent.Tests/            # 單元測試專案（{主專案名}.Tests 慣例）
├── Test_Agent.sln
└── INSTRUCTIONS.md（本檔）
```

**依賴關係：** Controller → Service → Repository（單向，不反向依賴）。

---

## 四、各層實作要點

### 4.1 領域層 (Domain)（選用）

- **實體**：與資料表對應的類別，僅含屬性與領域行為，不依賴 EF。
- **放置**：`src/Entity/`，例如 `Product.cs`、`Order.cs`。

範例實體：

```csharp
// Domain/Entities/Product.cs
namespace TestAgent.Domain.Entities;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

### 4.2 資料存取層 (Infrastructure)

- **資料夾**：`Repository/`。
- **DbContext**：繼承 `DbContext`，在 `OnModelCreating` 設定對應與關聯。
- **Repository 實作**：實作 Application 層定義的介面（如 `IProductRepository`），內部使用 `DbContext`，放在 `Repository/`。
- **套件**：僅在 Infrastructure 專案引用 EF Core 與資料庫提供者。

要點：

- 註冊 `DbContext` 與所有 Repository 為 DI 服務（在 API 的 `Program.cs` 或擴充方法中）。
- 連線字串放在 **API 專案** 的 `appsettings.json`，透過選項或 `IConfiguration` 傳入 Infrastructure。

### 4.3 業務邏輯層 (Application)

- **資料夾**：`Service/`。
- **介面**：定義 Repository 或外部服務介面（如 `IProductRepository`、`IProductService`），由 Infrastructure 實作 Repository。
- **服務**：應用服務（如 `ProductService`）放在 `Service/`，呼叫 Repository、實作業務規則、回傳 DTO。
- **DTO**：請求/回應模型放在 `src/Dto/`，不暴露 Entity 到 API。

要點：

- 不在 Application 引用 EF Core 或具體資料庫型別，只依賴介面與 Domain（若有）。

### 4.4 表現層 (API)

- **資料夾**：`Controller/`。
- **Controllers**：控制器放在 `Controller/`，只做參數綁定、呼叫 Application 服務、回傳 HTTP 狀態與 DTO。
- **Program.cs**：註冊服務（Application 的 Service、Infrastructure 的 DbContext 與 Repository）、中介軟體、Swagger。
- **設定**：`appsettings.json` 的連線字串、CORS、Logging 等。

要點：

- Controller 不直接使用 `DbContext` 或 Repository 實作類別，只透過 Application 的介面/服務。

---

## 五、API 設計規範

本專案 Web API 依下列規範實作，以維持一致、可預期的介面。

### 5.1 路由與命名

- **基底路徑**：`/api`，必要時可加版本，例如 `/api/v1`。
- **資源名稱**：使用**複數名詞**，例如 `/api/products`、`/api/orders`。
- **單一資源**：以 ID 區分，例如 `GET /api/products/{id}`。
- **命名風格**：URL 建議 **kebab-case**（如 `/api/product-reviews`），或全專案統一採用一種（如 PascalCase）即可。

### 5.2 HTTP 方法語意

| 方法 | 用途 | 範例 |
|------|------|------|
| GET | 查詢（單筆或列表，不變更狀態） | `GET /api/products`、`GET /api/products/1` |
| POST | 新增資源 | `POST /api/products` |
| PUT | 全量更新（替換整筆資源） | `PUT /api/products/1` |
| PATCH | 部分更新 | `PATCH /api/products/1` |
| DELETE | 刪除 | `DELETE /api/products/1` |

### 5.3 HTTP 狀態碼

- **200 OK**：GET/PUT/PATCH 成功，回應本體為資源或 DTO。
- **201 Created**：POST 成功建立，`Location` header 建議帶新資源 URI。
- **204 No Content**：DELETE 成功或更新後無需回傳內容時。
- **400 Bad Request**：請求參數或 body 驗證失敗（含 ModelState）。
- **401 Unauthorized**：未登入或未提供有效憑證。
- **403 Forbidden**：已驗證但無權限。
- **404 Not Found**：資源不存在。
- **500 Internal Server Error**：伺服器未預期錯誤；正式環境回應中**不暴露內部例外細節**。

### 5.4 請求與回應格式

- **Content-Type**：請求/回應皆使用 **application/json**；若有檔案上傳再依需求使用 multipart。
- **請求 body**：以 JSON 傳送，與 DTO 對應；必填欄位以 Data Annotations 或 FluentValidation 驗證。
- **回應 body**：直接回傳 DTO 或統一包裝格式（例如 `{ "data": ..., "error": null }`）；**全專案擇一，保持一致**。
- **分頁**：列表 API 建議支援 `page`、`pageSize`（或 `limit`/`offset`），回應中帶總筆數或是否有下一頁。

### 5.5 錯誤回應格式

錯誤時（4xx/5xx）建議統一結構，方便前端處理，例如：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "請求參數驗證失敗",
  "details": [
    { "field": "Name", "message": "名稱為必填" }
  ]
}
```

- `code`：業務或技術錯誤代碼（字串）。
- `message`：給人看的簡短說明。
- `details`：選填，驗證錯誤清單或補充資訊；**生產環境勿回傳堆疊或內部訊息**。

### 5.6 Controller 實作要點

- 僅做**參數綁定、呼叫 Service、回傳 IActionResult/ActionResult&lt;T&gt;**；不寫業務邏輯。
- 善用 `[FromRoute]`、`[FromQuery]`、`[FromBody]` 明確綁定來源。
- 驗證失敗時回傳 `BadRequest(ModelState)` 或自訂錯誤 DTO。
- 資源不存在時回傳 `NotFound()`；無權限時回傳 `Forbid()` 或 `StatusCode(403)`。
- 可透過 **Swagger/OpenAPI** 註解（`[ProducesResponseType]`）標註各端點的回應型別與狀態碼。

### 5.7 例外處理規範

- **分層職責**
  - **Controller**：原則上不 try-catch；未處理的例外由**全域例外處理中介軟體**統一捕捉並轉成 HTTP 回應。
  - **Service**：可拋出業務例外（例如「資源不存在」「違反業務規則」），由上層或中介軟體轉成對應狀態碼。
  - **Repository**：不吞掉例外；可讓 DbContext 與資料庫錯誤往上拋，由中介軟體或 Service 層決定如何對外呈現。

- **例外與 HTTP 狀態對應**
  - 資源不存在（例如查無此 ID）→ **404 Not Found**。
  - 參數/驗證錯誤、業務規則違反（例如重複建立）→ **400 Bad Request**。
  - 未授權、權限不足 → **401 Unauthorized** / **403 Forbidden**。
  - 衝突（例如樂觀鎖失敗）→ **409 Conflict**。
  - 其餘未預期例外 → **500 Internal Server Error**；正式環境**僅回傳通用錯誤訊息，不回傳堆疊或內部細節**。

- **錯誤回應格式**
  - 與 5.5 一致：統一使用 `code`、`message`、`details`（選填）結構；同一種錯誤使用相同 `code`，方便前端與監控辨識。

- **紀錄（Logging）**
  - 在捕捉例外處（例如全域處理器）記錄 **Warning** 或 **Error**，並寫入例外類型、訊息與必要上下文（例如 RequestId、UserId）；**敏感資料（密碼、個資）不得寫入 log**。
  - 開發環境可記錄完整堆疊；正式環境依資安政策決定是否記錄堆疊或僅記錄摘要。

- **實作建議**
  - 使用 **Exception Handler Middleware**（或 `IExceptionHandler` / `IExceptionFilter`）集中處理，避免在每個 Controller 重複 try-catch。
  - 可自訂業務例外類別（例如 `NotFoundException`、`ValidationException`），在中介軟體內依例外型別對應 HTTP 狀態碼與錯誤 DTO。

#### 基本規範（集中處理與自訂例外）

| 項目 | 規範 |
|------|------|
| **集中處理方式** | 在 **API 專案**內實作 **Exception Handler Middleware** 或 **IExceptionHandler**（.NET 8+ 建議），於 `Program.cs` 註冊；順序為先 `UseExceptionHandler`，再 `UseRouting` / `MapControllers`（Web API 控制器），確保所有未處理例外皆經此管道。 |
| **自訂例外放置** | 業務例外類別放在 **Application** 層（例如 `Service/Exceptions/`），供 Service 拋出、API 層參考；若僅 API 使用可放在 API 專案。 |
| **自訂例外命名** | 以 `*Exception` 結尾，語意清楚；建議至少提供：`NotFoundException`（404）、`ValidationException`（400）、`ConflictException`（409）、`ForbiddenException`（403）。建構式可接受 `message`、選填 `code` 或 `details`。 |
| **例外 → HTTP 對應** | 在 Exception Handler 內依 `exception.GetType()` 對應：`NotFoundException` → 404、`ValidationException` → 400、`ConflictException` → 409、`ForbiddenException` → 403、`UnauthorizedException` → 401；其餘（含 `Exception`）→ 500。 |
| **錯誤 DTO** | 統一使用一組類別（例如 `ErrorResponse`）：含 `code`（string）、`message`（string）、`details`（object 或陣列，選填）。由 Exception Handler 從例外訊息或自訂屬性填入，並序列化為 JSON 回傳；**500 在正式環境固定為通用訊息，不帶堆疊**。 |
| **Logging** | 在 Exception Handler 內捕捉後先記錄（`ILogger`），再寫入回應；記錄內容包含例外類型、訊息、RequestId；是否記錄堆疊依環境區分。 |

---

## 六、Middleware 與 Filter 開發要求

本節規範自訂 **Middleware** 與 **Filter** 的撰寫方式、放置位置與註冊順序，以維持管線行為可預期且易維護。

### 6.1 Middleware 開發要求

- **職責單一**：每個中介軟體只處理一類橫切關注（例如例外、紀錄、RequestId、CORS）；不在一支中介軟體內混雜多種職責。
- **放置位置**：自訂中介軟體建議放在 API 專案內，例如 `Middleware/` 資料夾（如 `ExceptionHandlerMiddleware.cs`、`RequestIdMiddleware.cs`）；若與基礎建設強相關，可另建 `Infrastructure` 專案並在 API 引用。
- **註冊順序**（`Program.cs` 內 `Use*` 的順序即為執行順序）：
  1. **最先**：`UseExceptionHandler`（或自訂例外處理中介軟體），以捕捉後續管線所有未處理例外。
  2. 其後依需求：`UseHttpsRedirection`、`UseCors`、`UseAuthentication`、`UseAuthorization`。
  3. **路由與端點**：`UseRouting`、`UseEndpoints` 或 `MapControllers`（Web API 控制器）。
  4. 需在「進入 API Controller 前」執行的邏輯（例如記錄請求、注入 RequestId）應放在 `UseRouting` 之前。
- **依賴注入**：透過建構式注入 `ILogger`、`RequestDelegate` 等；需的服務從 DI 解析，不在中介軟體內 `new` 或直接取靜態實例。
- **短路與呼叫下一棒**：若要直接回應不進入後續管線，則**不要**呼叫 `await next(context)`；否則應呼叫一次並 `await`，以維持管線延續。
- **非同步與例外**：使用 `async`/`await`，避免 `.Result` 或 `.Wait()`；不要吞掉例外，讓未處理例外往上拋給例外處理中介軟體。

**建議順序範例：**

```text
UseExceptionHandler → UseHttpsRedirection → UseCors → UseAuthentication → UseAuthorization → UseRouting → MapControllers
```

### 6.2 Filter 開發要求

- **Filter 與 Middleware 分工**：
  - **Middleware**：與 HTTP 管線、請求/回應串流、未進入 API Controller 前的處理（例外、CORS、RequestId、記錄請求層級）相關。
  - **Filter**：與 **Web API Controller 執行週期** 綁定（Action 執行前後、授權、模型綁定後驗證、結果執行前後、Controller 內例外）。
- **Filter 種類與用途**：

| 種類 | 介面 / 抽象類別 | 典型用途 |
|------|------------------|----------|
| **Authorization** | `IAuthorizationFilter` / `AsyncAuthorizationFilter` | 授權檢查（角色、原則）；未通過可設 `context.Result` 短路。 |
| **Resource** | `IResourceFilter` / `IAsyncResourceFilter` | 快取、模型綁定前後資源取得/釋放。 |
| **Action** | `IActionFilter` / `IAsyncActionFilter` | Action 執行前後（紀錄、計時、修改參數或結果）。 |
| **Exception** | `IExceptionFilter` / `IAsyncExceptionFilter` | 僅捕捉 **該 Controller/Action** 內拋出的例外；若需全站統一例外格式，建議以 **Exception Handler Middleware** 為主。 |
| **Result** | `IResultFilter` / `IAsyncResultFilter` | 結果執行前後（修改回應標頭、包裝結果）。 |

- **放置位置**：Filter 類別建議放在 API 專案內，例如 `Filters/` 資料夾（如 `ValidationActionFilter.cs`、`LoggingActionFilter.cs`），命名清楚表達職責。
- **註冊方式**：
  - **全域**：在 `Program.cs` 或 `AddControllers` 時以 `options.Filters.Add(typeof(MyFilter))` 或 `AddService` 註冊，適用所有 Controller/Action。
  - **Controller/Action 層級**：以屬性標註 `[TypeFilter(typeof(MyFilter))]`、`[ServiceFilter(typeof(MyFilter))]` 或實作 `IFilterFactory` 的屬性，僅套用於該 Controller 或 Action。
- **職責單一**：一個 Filter 只做一類事（例如只做紀錄、只做驗證、只做授權）；不在同一個 Filter 內混雜多種邏輯。
- **依賴注入**：需要服務時，Filter 以 `ServiceFilter` 或 `TypeFilter` 搭配 DI 註冊，或實作 `IFilterFactory` 從 `context` 取得服務，避免在 Filter 內直接 `new` 依賴。
- **Exception Filter 與 Middleware**：若已使用 **Exception Handler Middleware** 做全站例外處理，Exception Filter 僅用於「僅某 Controller/Action 需不同錯誤格式或紀錄」的場合，避免重複處理或與 Middleware 行為不一致。

### 6.3 與本文件其他章節的對應

- **例外處理**：全站未處理例外以 **Middleware**（5.7、基本規範表）處理；Controller 內業務例外由 Service 拋出，經 Middleware 轉成 HTTP 與錯誤 DTO。
- **授權**：可搭配 `IAuthorizationFilter` 或 ASP.NET Core 的 `AuthorizationHandler` / 原則；若僅需角色檢查，可用 `[Authorize(Roles = "...")]`。
- **紀錄**：請求層級（URL、Method、RequestId）建議在 **Middleware** 紀錄；Action 層級（哪個 Action、參數、耗時）可用 **Action Filter**。

---

## 七、開發流程建議

1. **Domain**：先定義實體與列舉。
2. **Infrastructure**：在 `Repository/` 建立 `DbContext`、實體對應、Repository 實作，並執行第一次遷移。
3. **Application**：在 `Service/` 定義 Repository/服務介面與 DTO，實作應用服務。
4. **API**：在 `Controller/` 建立 Controller、註冊 DI、設定連線與 Swagger。
5. 以 **Swagger** 或 **MCP 工具**（見第八節）測試 API，再視需要補單元測試（測試專案參考對應層）。

---

## 八、API 測試用 MCP 工具建議

在 Cursor / VS Code 中可透過 MCP（Model Context Protocol）讓 AI 助理代為呼叫本專案或外部 API，以下為建議選項。

| 工具 | 說明 | 適用情境 |
|------|------|----------|
| **Playwright MCP** | 透過 Playwright 發送 HTTP 請求、執行 API 測試腳本；可搭配 `@playwright/test` 的 APIRequestContext 或既有 Playwright 專案。在 Cursor 中設定 Playwright MCP 後，可由 AI 代為執行請求或測試。 | 本專案 API 的 CRUD 驗證、回傳狀態碼與 JSON 結構檢查；可撰寫並執行 Playwright API 測試。 |
