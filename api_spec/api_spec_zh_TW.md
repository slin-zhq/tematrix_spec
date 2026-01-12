# TEMatrix 後端 API 規格

**建立日期**：Nov 03, 2025  
**最後更新**：Nov 18, 2025

---

**設計原則**：為了減少後端負載，目前前端將處理下列工作：

- 在步驟 2（選擇專利和論文）中進行專利與論文的篩選與排序
- 在步驟 3（查看與自訂矩陣）中自訂藍/紅海洋閾值
- 在步驟 3（查看與自訂矩陣）中點擊個別儲存格以查看該儲存格內的專利與論文

> 實作說明：根據 REST 最佳實務，PUT 用於替換整個資源或在資源不存在時建立；PATCH 用於部分更新；POST 用於建立或新增資源。_來源：[GeeksForGeeks](https://www.geeksforgeeks.org/java/what-is-the-difference-between-put-post-and-patch-in-restful-api/)_。

實作說明：在選擇階段、矩陣規格設定與矩陣最終化等操作，本系統預設使用 `PATCH` 做彈性、部分更新（例如以 `selectionConfirmed` 或 `specificationConfirmed` 儲存進度，或更新局部欄位如 view settings）。`PATCH` 支援 upsert（不存在則建立），避免為每個子資源提供額外的 `POST`，維持 API 的專案導向與簡潔性。

---

## 實體關係圖（ERD）

**關鍵關係：**

- **User (1) → (N) Project**：一位使用者可擁有多個專案。
- **Project (1) → (1) Selections**：每個專案對應一組選擇（專利／論文）。
- **Project (1) → (1) Matrix-Spec**：每個專案對應一個矩陣規格。
- **Project (1) → (1) Matrix**：每個專案對應一個矩陣（由多個儲存格組成）。
- **Matrix (1) → (M) Cells**：矩陣包含多個儲存格（每個儲存格連結至專利／論文）。
- **Project (1) → (N) View-Settings**：每個專案可儲存多組視圖設定（由不同使用者保存）。
- **Project (1) → (1) Progress**：每個專案有一個進度快照。
- **Project (1) → (N) Exports**：每個專案可擁有多個匯出次數記錄。
- **Project (1) → (N) Shares**：每個專案可與多名使用者分享。
- **Patents/Papers**：為全域唯讀集合，僅由選擇與儲存格參照，而非專案擁有。

---

## 資源與端點

Base URL: `/api/v1`

設計說明：API 按主要資源建模以對應使用者操作流程（例如：查詢建立 → 選擇 → 指定矩陣 → 產生/匯出矩陣）。

### 資源：

- `users`：目前僅用於認證；使用者擁有 `username` 與 `password`。
- `sessions`：用於管理使用者工作階段（登入／登出）。
- `projects`：每個專案代表一個工作流程（查詢建立 → 選擇專利／論文 → 指定矩陣尺寸 → 產生並最終化矩陣等）。

### 端點：

**認證：**

- `POST /sessions`：使用者登入，回傳 session token。
- `DELETE /sessions`：使用者登出，使 session token 失效。

**專案工作流程：**

步驟 0. 建置查詢

- `GET /query-builder/keyword-suggestions?seedKeyword="<seedKeyword>"`：根據種子關鍵字取得 AI 關鍵字建議。
  > Q：為何用 `GET` 並以查詢參數 `seedKeyword`？
  > A：因為：
  >
  > - `GET` 是安全且冪等，支援快取與書籤。
  > - 查詢參數有助於端點可發現性，並符合典型搜尋 API 的用法。
- `POST /projects`：執行搜尋並建立新的專案。

步驟 1. 選擇專利與論文

- `PATCH /projects/{projectId}/selections`：儲存已選擇的專利與論文，供下一步使用。

步驟 2. 指定矩陣維度

- `PATCH /projects/{projectId}/matrix-specification`：儲存／更新矩陣維度，並根據選擇的專利／論文與指定維度產生矩陣。

步驟 3. 儲存／最終化專案並匯出矩陣

- `PATCH /projects/{projectId}/matrix`：最終化專案（不可逆）；矩陣僅在最終化後可分享。分享的矩陣可以透過專案連結由任何人查看。

- `GET /projects/{projectId}/export`：將矩陣匯出為 Excel 檔案。

**使用者進度管理：**

- `GET /users/me/progress/{projectId}`：讀取特定專案的已儲存進度。

**矩陣分享與視圖設定管理：**

- `GET /projects/{projectId}/share`：產生專案分享連結。任何擁有該連結的人可查看矩陣（視專案是否已最終化）。
- `GET /projects/{projectId}`：查看（分享的）矩陣。未最終化的專案不可分享／檢視。
- `PUT /projects/{projectId}/view-settings`：儲存／更新使用者的矩陣視圖設定（如藍海／紅海閾值）。預設使用建立者的設定。
  > 設計說明：此處採用 `PUT` 而非 `PATCH` 或 `POST`，以明確視圖設定為整體替換（或在不存在時建立）。如需部分更新請使用 `PATCH`。

**專案清單與管理：**

- `GET /users/me/projects`：取回登入使用者的專案清單。
- `PATCH /users/me/projects/{projectId}`：重新命名專案。
- `DELETE /users/me/projects/{projectId}`：刪除專案。

---

## 共用介面

### 基礎回應介面

```typescript
interface APIResponse<T = any> {
  status: "success" | "error";
  code: number;
  timestamp: string; // ISO 8601 時間戳
  apiVersion: string; // 例如 "v1.0"
  data?: T;
  error?: APIError;
}
```

### 錯誤介面

```typescript
interface APIError {
  type: ErrorType;
  message: string;
  details?: ValidationErrorDetail[];
}

interface ValidationErrorDetail {
  field?: string;
  message: string;
  //   position?: number; // 位置（可選）
}

type ErrorType =
  | "VALIDATION_ERROR"
  | "AUTHENTICATION_ERROR"
  | "AUTHORIZATION_ERROR"
  | "NOT_FOUND"
  | "QUERY_EXECUTION_ERROR"
  | "DATABASE_ERROR"
  | "AI_SERVICE_ERROR"
  | "EXPORT_ERROR"
  | "TIMEOUT_ERROR"
  | "RATE_LIMIT_ERROR"
  | "INTERNAL_SERVER_ERROR";
```

---

## API 請求與回應（範例）

### 1. 認證

**`POST /sessions`：使用者登入。**

Request:

```json
{
  "username": string,
  "password": string
}
```

Response:

```json
{
  ...APIResponse,
  data: {
    "sessionToken": string,
    "userId": string,
    "username": string,
    "expiresInHours": number // 前端可使用此值設定 token 過期時間並自動登出
  }
}
```

**`DELETE /sessions`：使用者登出。**

Request:

```json
{
  // 無需 body，只需在 Authorization 標頭中提供 session token
}
```

Response:

```json
{
  ...APIResponse
}
```

### 2. 專案工作流程

#### 步驟 0：建置查詢

**`GET /query-builder/keyword-suggestions?seedKeyword="<seedKeyword>"`：取得 AI 關鍵字建議。**

Request:

```json
{
  // 無
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    "seedKeyword": string,
    "suggestions": string[]
  }
}
```

**`POST /projects`：執行搜尋並建立新專案。**

Request:

> 實作說明：後端由 Authorization 標頭內的 session token 解析並取得使用者 ID。

```json
{
  "projectTitle": string,
  "query": string,
  "yearRange": {
    "start": number,
    "end": number
  } // 可選，預設為最近 5 年
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    "projectId": string,
    "projectTitle": string,
    "query": string,
    "patents": [
      {
        "id": string,
        "title": string,
        "inventor": string,
        "patentHolder": string,
        "filedYear": number,
        "priorityYear": number | null,
        "publicationYear": number | null,
        "grantYear": number | null,
        "cpc": string[],
        "keywords": string[],
        "concepts": string[],
        "link": string
      },
      {...}, ...
    ],
    "papers": [
      {
        "id": string,
        "title": string,
        "author": string[],
        "journal": string,
        "publicationYear": number,
        "citations": number,
        "cpc": string[],
        "concepts": string[],
        "link": string
      },
      {...}, ...
    ],
    "filterOptions": {
      "patent": {
        PatentFilterKey: FilterOption,
        ...
      },
      "paper": {
        PaperFilterKey: FilterOption,
        ...
      }
    },
    "sortOptions": {
      "patent": PatentSortKey[],
      "paper": PaperSortKey[]
    },
    "createdAt": string // ISO 8601 時間戳
  }
}
```

```typescript
type PatentFilterKey =
  | "cpc"
  | "filedYear"
  | "priorityYear"
  | "publicationYear"
  | "grantYear"
  | "patentHolder"
  | "keywords"
  | "concepts";

type PaperFilterKey =
  | "journal"
  | "publicationYear"
  | "cpc"
  | "citations"
  | "author"
  | "concepts";

type FilterOption = MultiselectFilter | RangeFilter;

interface MultiselectFilter {
  type: "multiselect";
  options: string[];
}

interface RangeFilter {
  type: "range";
  min: number;
  max: number;
}

type PatentSortKey =
  | "id"
  | "title"
  | "inventor"
  | "patentHolder"
  | "filedYear"
  | "priorityYear"
  | "publicationYear"
  | "grantYear";

type PaperSortKey =
  | "title"
  | "author"
  | "journal"
  | "publicationYear"
  | "citations";
```

#### 步驟 1：選擇專利與論文

**`PATCH /projects/{projectId}/selections` 搭配 `selectionConfirmed: false`：儲存選定的專利與論文。**

Request:

```json
{
  "selectedPatentIds": string[],
  "selectedPaperIds": string[],
  /// --- 用於儲存使用者進度
  "selectedTab": "patent" | "paper",
  "filterSettings": {
    "patentFilterSettings": {
      PatentFilterKey: {
        "type": "multiselect",
        "selectedValues": string[]
      },
      PatentFilterKey: {
        "type": "range",
        "selectedMin": number,
        "selectedMax": number
      }
      ...
    },
    "paperFilterSettings": {
      PaperFilterKey: {
        "type": "multiselect",
        "selectedValues": string[]
      },
      PaperFilterKey: {
        "type": "range",
        "selectedMin": number,
        "selectedMax": number
      }
      ...
    }
  },
  "sortSettings": {
    "patentSortSettings": {
      PatentSortKey: {
        "index": number | null,
        "order": "ASC" | "DESC" | null,
      }
      ...
    },
    "paperSortSettings": {
      PaperSortKey: {
        "index": number | null,
        "order": "ASC" | "DESC" | null,
      }
      ...
    }
  },
  /// ---
  "selectionConfirmed": false // 是否確認選擇以推進專案狀態
                // 若為 true，準備矩陣維度的 AI 建議
                // 若為 false，僅儲存選擇

}
```

Response:

```json
{
  ...APIResponse,
  // "data": {
  //     "projectId": string,
  // }
}
```

**`PATCH /projects/{projectId}/selections` 搭配 `selectionConfirmed: true`：確認選定的專利與論文，並移至下一步。**

Request:

```json
{
  // ... same as above...,
  "selectionConfirmed": true // 每次後端收到 true 都會重新生成矩陣規格的 AI 建議。
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    // "projectId": string,
    "matrixSpecificationAISuggestions": {
      "functionLabels": string[],
      "technologyLabels": string[]
    }
  }
}

> 實作說明：當前流程設計為同步回傳 AI 建議，適用於中小規模資料集。若使用情境有大量資料或耗時運算，建議改採非同步作業（回傳 job id，後續以 job id 查詢進度/結果）。
```

#### 步驟 2：指定矩陣維度

**`PATCH /projects/{projectId}/matrix-specification` 搭配 `specificationConfirmed: false`：儲存/更新矩陣維度。**

Request:

```json
{
  /// --- 用於儲存使用者進度
  "functionLabels": string[], // 標籤順序很重要
  "technologyLabels": string[],
  /// ---
  "specificationConfirmed": false // 是否確認矩陣規格以產生矩陣
                  // 若為 true，根據所選專利/論文與指定維度產生矩陣
                  // 若為 false，僅儲存維度
}
```

Response:

```json
{
  ...APIResponse,
  // "data": {
  //     "projectId": string,
  // }
}
```

**`PATCH /projects/{projectId}/matrix-specification` 搭配 `specificationConfirmed: true`：確認矩陣維度並產生矩陣。**

Request:

```json
{
  "functionLabels": string[],
  "technologyLabels": string[],
  "specificationConfirmed": true
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    // "projectId": string,
    /// ---
    // 說明：前端請求已包含 `functionLabels` 與 `technologyLabels`，後端應依該順序處理。
    // 若後端調整順序，請在回應中維持並回傳一致順序以避免前後端混淆。
    // "functionLabels": string[],
    // "technologyLabels": string[],
    /// ---
    "cells": [
      {
        "functionIndex": number,
        "technologyIndex": number,
        "patentIds": string[],
        "paperIds": string[]
      },
      {...}, ...
    ],
    // "createdAt": string // ISO 8601 時間戳
  }
}
```

#### 步驟 3：儲存/最終化專案並匯出矩陣

**`PATCH /projects/{projectId}/matrix` 以最終化專案。**

Request:

```json
{
  // "projectId": string,
  "viewSettings": {
    "blueOceanThreshold": number,
    "redOceanThreshold": number
  },
  "finalized": true
}
```

Response:

```json
{
  ...APIResponse,
  // "data": {
  //     "projectId": string,
  // }
}
```

**`GET /projects/{projectId}/export`：將矩陣匯出為 Excel 檔案。**

Request:

```json
{
  // 無需 body，projectId 在 URL
}
```

Response:

- Binary Excel file with `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

### 3. 使用者進度管理

**`GET /users/me/progress/{projectId}`：擷取使用者在特定專案的進度。**

Request:

```json
{
  // 無需 body，URL 中包含 userId 與 projectId
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    "projectTitle": string,
    "query": string,
    "currentStep": "selection" | "specification" | "matrix",
    // ---
    // 與 `POST /projects` 的回應相同
    "selectionBaseData": {
      "patents": [ ... ],
      "papers": [ ... ],
      "filterOptions": { ... },
      "sortOptions": { ... }
    },
    "specificationBaseData": {
      "functionLabels": string[],
      "technologyLabels": string[]
    } | null, // 若 currentStep 為 "selection" 則為 null
    "matrixBaseData": {
      "cells": [
        {
          "functionIndex": number,
          "technologyIndex": number,
          "patentIds": string[],
          "paperIds": string[]
        },
        {...}, ...
      ]
    } | null, // 若 currentStep 為 "selection" 或 "specification" 則為 null
    "userProgress": {
      "selection": {
        // 與「步驟 1：選擇專利與論文」相似
        "selectedPatentIds": string[], // 若使用者尚未選擇則為空的陣列
        "selectedPaperIds": string[], // 若使用者尚未選擇則為空的陣列
        "selectedTab": "patent" | "paper",
        "filterSettings": {
          "patentFilterSettings": {
            PatentFilterKey: {
              "type": "multiselect",
              "selectedValues": string[]
            },
            PatentFilterKey: {
              "type": "range",
              "selectedMin": number,
              "selectedMax": number
            }
            ...
          },
          "paperFilterSettings": {
            PaperFilterKey: {
              "type": "multiselect",
              "selectedValues": string[]
            },
            PaperFilterKey: {
              "type": "range",
              "selectedMin": number,
              "selectedMax": number
            }
            ...
          }
        },
        "sortSettings": {
          "patentSortSettings": {
            PatentSortKey: {
              "index": number | null,
              "order": "ASC" | "DESC" | null
            }
            ...
          },
          "paperSortSettings": {
            PaperSortKey: {
              "index": number | null,
              "order": "ASC" | "DESC" | null
            }
            ...
          }
        }
      },
      "specification": {
        "functionLabels": string[],
        "technologyLabels": string[]
      } | null
    }
  }
}
```

### 4. 矩陣分享與檢視

**`GET /projects/{projectId}/share`：分享矩陣。**

Request:

```json
{
  // 無需 body，projectId 在 URL
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    "shareableLink": string // 用於檢視共享矩陣的 URL
  }
}
```

**`GET /projects/{projectId}`：查看（分享的）矩陣。**

Request:

> 實作說明：後端由 Authorization 標頭內的 session token 解析使用者身分。若該使用者已儲存檢視偏好（例如藍海/紅海閾值），系統優先回傳該偏好；否則以建立者設定為預設值。

```json
{
  // 無需 body，projectId 在 URL
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    // "projectId": string,
    "projectTitle": string,
    "query": string,
    "patents": [ ... ], // 僅包含已選定的專利
    "papers": [ ... ], // 僅包含已選定的論文
    "functionLabels": string[],
    "technologyLabels": string[],
    "cells": [
      {
        "functionIndex": number,
        "technologyIndex": number,
        "patentIds": string[],
        "paperIds": string[]
      },
      {...}, ...
    ],
    "viewSettings": {
      "blueOceanThreshold": number,
      "redOceanThreshold": number
    },
    "createdAt": string // ISO 8601 時間戳
  }
}
```

**`PUT /projects/{projectId}/view-settings`：儲存/更新矩陣視圖設定（藍色/紅色海洋閾值、等）。**

Request:

> 實作說明：後端由 Authorization 標頭內的 session token 解析使用者身分，供授權與偏好處理使用。

```json
{
  "blueOceanThreshold": number,
  "redOceanThreshold": number
}
```

### 5. 專案／搜尋管理

**`GET /users/me/projects`：取得使用者的專案清單。**

Request:

> 實作說明：後端由 Authorization 標頭內的 session token 解析使用者身分，供授權與偏好處理使用。

```json
{
  // 無需 body，只需在 Authorization 標頭中提供 session token
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    "projects": [
      {
        "projectId": string,
        "projectTitle": string,
        "createdAt": string, // ISO 8601 時間戳；前端可使用此值以建立時間排序專案
        "status": "OPEN" | "COMPLETED"
      },
      {...}, ...
    ]
  }
}
```

**`PATCH /users/me/projects/{projectId}`：重新命名專案。**

Request:

> 實作說明：後端由 Authorization 標頭內的 session token 解析使用者身分，供授權與偏好處理使用。

```json
{
  "newTitle": string
}
```

Response:

```json
{
  ...APIResponse,
}
```

**`DELETE /users/me/projects/{projectId}`：刪除專案。**

Request:

> 實作說明：後端由 Authorization 標頭內的 session token 解析使用者身分，供授權與偏好處理使用。

```json
{
  // 無需 body，URL 中包含 userId 與 projectId
}
```

Response:

```json
{
  ...APIResponse
}
```

---

## 驗證標頭

所有需要驗證的端點需：

```
Authorization: Bearer <sessionToken>
Content-Type: application/json
```

未驗證端點：

- `POST /sessions`

---

## 錯誤處理

| Code | Meaning               | Common Use Cases                      |
| ---- | --------------------- | ------------------------------------- |
| 200  | OK                    | Successful GET/PUT/PATCH/DELETE       |
| 201  | Created               | Successful POST for resource creation |
| 202  | Accepted              | Request accepted for async processing |
| 400  | Bad Request           | Invalid parameters, malformed request |
| 401  | Unauthorized          | Missing/invalid session token         |
| 403  | Forbidden             | Insufficient permissions              |
| 404  | Not Found             | Resource not found                    |
| 408  | Request Timeout       | Query/AI generation timeout           |
| 429  | Too Many Requests     | Rate limit exceeded                   |
| 500  | Internal Server Error | Unexpected backend error              |
