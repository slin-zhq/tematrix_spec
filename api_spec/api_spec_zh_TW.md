# TEMatrix 後端 API 規格

**建立日期**：Nov 03, 2025  
**最後更新**：Nov 18, 2025

---

**設計原則**：為了減少後端負載，目前前端將處理下列工作：

- 在步驟 2（選擇專利和論文）中進行專利與論文的篩選與排序
- 在步驟 3（查看與自訂矩陣）中自訂藍/紅海洋閾值
- 在步驟 3（查看與自訂矩陣）中點擊個別儲存格以查看該儲存格內的專利與論文

> **提示：** 根據 REST 最佳實務，PUT 用於替換整個資源或在資源不存在時建立新資源，PATCH 用於部分更新，POST 用於提交資料以建立或新增資源。_來源：[GeeksForGeeks](https://www.geeksforgeeks.org/java/what-is-the-difference-between-put-post-and-patch-in-restful-api/)_。

我們對於選擇、矩陣規格與矩陣最終化採用 `PATCH`，因為這些操作通常是部分更新（例如使用 `selectionConfirmed` 或 `specificationConfirmed` 旗標儲存進度，或更新視圖設定等欄位）。`PATCH` 允許彈性的部分變更，而不需 `PUT` 的整個資源替換。對於這些子資源的首次建立，`PATCH` 可被視為 upsert（若不存在則建立），避免不必要的 `POST` 端點，並保持 API 以專案為中心且簡潔。

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

依照 REST 最佳實務，API 以應用程式中的主要資源組織，對應使用者操作流程。

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
  > 注意：此處選擇 `PUT` 而非 `PATCH` 或 `POST`，表明視圖設定是替換整個資源或在不存在時建立。

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
  timestamp: string; // ISO 8601 timestamp
  apiVersion: string; // e.g., "v1.0"
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
  //   position?: number;
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
    "expiresInHours": number // Frontend can use this to set token expiry time, and auto-logout accordingly
  }
}
```

**`DELETE /sessions`：使用者登出。**

Request:

```json
{
  // No body needed, just Auth header with session token
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
  // None
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

> Note: The user ID is obtained from the session token in Auth header.

```json
{
  "projectTitle": string,
  "query": string,
  "yearRange": {
    "start": number,
    "end": number
  } // optional, default to last 5 years
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
    "createdAt": string // ISO 8601 timestamp
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
  /// --- For saving user progress
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
  "selectionConfirmed": false // whether user confirms selections to advance project state
                // If true, prepare AI suggestions for matrix dimensions
                // If false, just save selections

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
  "selectionConfirmed": true // Every time the backend receives true, it re-generates AI suggestions for matrix dimensions.
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
```

#### 步驟 2：指定矩陣維度

**`PATCH /projects/{projectId}/matrix-specification` 搭配 `specificationConfirmed: false`：儲存/更新矩陣維度。**

Request:

```json
{
  /// --- For saving user progress
  "functionLabels": string[], // The order of labels matters
  "technologyLabels": string[],
  /// ---
  "specificationConfirmed": false // whether user confirms matrix spec to generate matrix
                  // If true, generate matrix based on selected patents/papers and specified dimensions
                  // If false, just save dimensions
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
    // Note: frontend already has functionLabels and technologyLabels from request; however, backend needs to respect the orders sent by frontend.
    // Otherwise, backend needs to return the labels in order again here.
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
    // "createdAt": string // ISO 8601 timestamp
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
  // No body needed, just projectId in URL
}
```

Response:

- Binary Excel file with `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

### 3. 使用者進度管理

**`GET /users/me/progress/{projectId}`：擷取使用者在特定專案的進度。**

Request:

```json
{
  // No body needed, just userId and projectId in URL
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    "projectTitle": string,
    "query": string,
    // ---
    // same as in response from `POST /projects`
    "patents": [ ... ],
    "papers": [ ... ],
    "filterOptions": { ... },
    "sortOptions": { ... },
    // ---
    "currentProgress": {
      "currentStep": "selection" | "specification" | "matrix",
      "selection": {
        // Similar to "Step 1. Select Patents and "Papers"
          // `PATCH /projects/{projectId}/selections` with `selectionConfirmed: false` requet body
        "selectedPatentIds": string[],
        "selectedPaperIds": string[],
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

      },
      "specification": {
        "functionLabels": string[],
        "technologyLabels": string[]
      } | null, // null if currentStepCode is "selection"
      "matrix": { // Note: unless user has finalized (and saved) the project, the view settings wouldn't be saved yet, so no need for backend to return them here.
        "cells": [
          {
            "functionIndex": number,
            "technologyIndex": number,
            "patentIds": string[],
            "paperIds": string[]
          },
          {...}, ...
        ] | null // null if currentStepCode is "selection" or "specification"
      }
    },
  }
}
```

### 4. 矩陣分享與檢視

**`GET /projects/{projectId}/share`：分享矩陣。**

Request:

```json
{
  // No body needed, just projectId in URL
}
```

Response:

```json
{
  ...APIResponse,
  "data": {
    "shareableLink": string // URL to view the shared matrix
  }
}
```

**`GET /projects/{projectId}`：查看（分享的）矩陣。**

Request:

> Note: Backend identify the user from session token in Auth header. If the user has her view preferences (i.e., blue/red ocean thresholds) saved, return those settings; otherwise, return the creator's preferences by default.

```json
{
  // No body needed, just projectId in URL
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
    "patents": [ ... ], // Only selected patents
    "papers": [ ... ], // Only selected papers
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
    "createdAt": string // ISO 8601 timestamp
  }
}
```

**`PUT /projects/{projectId}/view-settings`：儲存/更新矩陣視圖設定（藍色/紅色海洋閾值、等）。**

Request:

> Note: Backend identify the user from session token in Auth header.

```json
{
  "blueOceanThreshold": number,
  "redOceanThreshold": number
}
```

### 5. 專案／搜尋管理

**`GET /users/me/projects`：取得使用者的專案清單。**

Request:

> Note: Backend identify the user from session token in Auth header.

```json
{
  // No body needed, just Auth header with session token
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
        "createdAt": string, // ISO 8601 timestamp; Frontend can use this to sort projects by creation date
        "status": "OPEN" | "COMPLETED"
      },
      {...}, ...
    ]
  }
}
```

**`PATCH /users/me/projects/{projectId}`：重新命名專案。**

Request:

> Note: Backend identify the user from session token in Auth header.

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

**`DELETE /users/{userId}/projects/{projectId}`：刪除專案。**

Request:

> Note: Backend identify the user from session token in Auth header.

```json
{
  // No body needed, just userId and projectId in URL
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
