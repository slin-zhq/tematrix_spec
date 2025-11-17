# TEMatrix 後端 API 規格說明

**建立日期**：2025 年 11 月 03 日  
**最後更新**：2025 年 11 月 04 日

---

**設計理念**：「API 為唯一真相來源」——後端確認處理結果，前端可選擇快取於本地狀態，但非必要。

---

## 通用介面

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
  position?: number;
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

## 資料模型介面

### 專利介面

```typescript
interface Patent {
  id: string;
  title: string;
  inventor: string;
  patentHolder: string;
  filedYear: number;
  priorityYear?: number; // 資料可能缺少
  publicationYear?: number; // 資料可能缺少
  grantYear?: number; // 資料可能缺少
  cpc: string[];
  keywords: string | string[];
  concepts: string;
  link: string;
}
```

### 論文介面

```typescript
interface Paper {
  id: string;
  title: string;
  author: string;
  journal: string;
  publicationYear: number;
  citations: number;
  cpc: string[];
  concepts: string;
  link: string;
}
```

---

## 驗證

### 登入請求/回應

```typescript
interface LoginRequest {
  username: string;
  password: string;
}

interface LoginResponse {
  sessionToken: string;
  userId: string;
  username: string;
  expiresIn: number; // 秒
}

interface LogoutResponse {
  message: string;
}
```

**端點：**

- **`POST /auth/login`**：`LoginRequest` → `APIResponse<LoginResponse>`
- **`POST /auth/logout`**：標頭：`Authorization: Bearer <token>` → `APIResponse<LogoutResponse>`

---

## 步驟 0：查詢建構器

### 關鍵字建議請求/回應

```typescript
interface KeywordSuggestionsRequest {
  seed_keyword: string;
}

interface KeywordSuggestionsResponse {
  keyword: string;
  suggestions: string[];
}
```

**端點：**

- **`GET /query-builder-keyword-suggestions`**：`KeywordSuggestionsRequest` → `APIResponse<KeywordSuggestionsResponse>`

## 步驟 1：專利與論文選擇

```typescript
interface SearchExecutionRequest {
  projectName: string;
  query: string;
  yearRange?: YearRange; // 預設為近五年
}

interface YearRange {
  min: number;
  max: number;
}

interface SearchExecutionResponse {
  projectId: string;
  projectName: string;
  query: string;
  patents: Patent[];
  papers: Paper[];
  filterOptions: FilterOptions;
  sortOptions: SortOptions;
  executedAt: string;
}

interface FilterOptions {
  patentFilterOptions: Record<PatentFilterKey, FilterOption>;
  paperFilterOptions: Record<PaperFilterKey, FilterOption>;
}

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

interface SortOptions {
  patentSortOptions: PatentSortKey[]; //
  paperSortOptions: PaperSortKey[];
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

**端點：**

- **`POST /search`**：`SearchExecutionRequest` → `APIResponse<SearchExecutionResponse>`

## 步驟 2：矩陣維度規格

```typescript
interface MatrixSpecInitRequest {
  projectId: string;
  selectedPatentIds: string[];
  selectedPaperIds: string[];
}

interface MatrixSpecInitResponse {
  projectId: string;
  projectName: string;
  query: string;
  functionSuggestions: string[];
  technologySuggestions: string[];
}
```

**端點：**

- **`POST /matrix/initialize-specification`**：`MatrixSpecInitRequest` → `APIResponse<MatrixSpecInitResponse>`

## 步驟 3：矩陣生成

```typescript
interface MatrixGenerationRequest {
  projectId: string;
  functionLabels: string[];
  technologyLabels: string[];
}

interface MatrixGenerationResponse extends APIResponse {
  data: {
    projectId: string;
    functionLabels: string[];
    technologyLabels: string[];
    cells: MatrixCell[];
    createdAt: string;
  };
}

interface MatrixCell {
  functionIndex: number;
  technologyIndex: number;
  patentCount: number;
  paperCount: number;
  patentIds: string[];
  paperIds: string[];
}

interface MatrixCellDetailsRequest {
  projectId: string;
  functionIndex: number;
  technologyIndex: number;
}

interface MatrixCellDetailsResponse {
  cell: {
    functionLabel: string;
    technologyLabel: string;
    patents: Patent[]; // 完整專利資料
    papers: Paper[]; // 完整論文資料
    patentCount: number;
    paperCount: number;
  };
}
```

**端點：**

- **`POST /matrix/generate`**：`MatrixGenerationRequest` → `APIResponse<MatrixGenerationResponse>`
<!-- - **`GET /matrix/{projectId}/cell/{functionIndex}/{technologyIndex}`**：→ `APIResponse<MatrixCellDetailsResponse>` // We no longer need this, as Front-end will store the data. -->

### 矩陣檢視設定

儲存「鴻海臨界」與「藍海臨界」偏好值。

```typescript
interface MatrixViewSettingsSaveRequest {
  projectId: string;
  settings: {
    [key: "blue_ocean_threshold" | "red_ocean_threshold"]: number; // 值需為 double 型態
  };
}

interface MatrixViewSettingsSaveResponse {
  // 無特殊內容，可採用基礎回應介面
}
```

**端點：**

- **`POST /matrix/view-settings`**：`MatrixViewSettingsSaveRequest` → `APIResponse<MatrixViewSettingsSaveResponse>`

## 步驟 4：矩陣匯出與檢閱

### 矩陣匯出請求

```typescript
interface MatrixExportRequest {
  projectId: string;
}
```

**端點：**

- **`POST /matrix/{projectId}/export`**：`MatrixExportRequest` → 二進位 Excel 檔，`Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

### 矩陣分享與檢視

```typescript
interface MatrixShareRequest {
  projectId: string;
  userId: string; // 允許已驗證用戶存取
}

interface MatrixShareResponse {
  creatorId: string; // 檢查檢視者是否可更改檢視設定
  ...MatrixGenerationResponse;
}
```

---

## 進度與工作階段管理

```typescript
interface ProgressSaveRequest {
  projectId: string;
  stepCode: StepCode;
  progress: UserProgressData;
}

interface StepCode {
  //   QUERY_BUILDER: "query_builder";
  SELECTION: "selection";
  MATRIX_SPECIFICATION: "matrix_specification";
  // MATRIX_GENERATION: "matrix_generation"; // 最終狀態 - 唯讀
}

interface UserProgressData {
  patentsPapersSelection?: SelectionProgress;
  matrixSpecification?: MatrixSpecProgress;
}

interface SelectionProgress {
  selectedPatentIds: string[];
  selectedPaperIds: string[];
  selectedTab: "patent" | "paper";
  filters: FilterSettings;
  sorting: SortSettings;
}

interface FilterSettings {
    patentFilterSettings: Record<PatentFilterKey, FilterOptionSetting>;
    paperFilterSettings: Record<PaperFilterKey, FilterOptionSetting>;
}

interface FilterOptionSetting = {
    type: "multiselect" | "range";
    selectedValues?: string[]; // for multiselect
    selctedMin?: number; // for range
    selectedMax?: number; // for range
    // selectedCount?: number; // for multiselect。前端會計算
}

interface SortSettings {
  patentSortSettings: PatentSortSettings[];
  paperSortSettings: PaperSortSettings[];
}

interface PatentSortSettings {
  index?: number;
  field: PatentSortKey;
  order: "ASC" | "DESC" | "DISABLED";
}

interface PaperSortSettings {
  index?: number;
  field: PaperSortKey;
  order: "ASC" | "DESC" | "DISABLED";
}

interface MatrixSpecProgress {
  selectedFunctionLabels: string[];
  selectedTechnologyLabels: string[];
}

interface ProgressRetrievalResponse {
  projectId: string;
  projectName: string;
  currentStepCode: StepCode;
  progress: UserProgressData;
  lastModifiedAt: string;
}
```

**端點：**

- **`POST /user-progress`**：`ProgressSaveRequest` → `APIResponse<ProgressSaveResponse>`
- **`GET /user-progress/{projectId}`**：→ `APIResponse<ProgressRetrievalResponse>`

---

## 專案管理

### 取得專案列表

```typescript
interface ProjectsListResponse {
  projects: ProjectSummary[];
}

interface ProjectSummary {
  projectId: string;
  projectName: string;
  createdAt: string;
  lastModifiedAt: string;
  status: "in_progress" | "completed";
}

interface ProjectRenameRequest {
  newName: string;
}
```

**端點：**

- **`GET /projects/{userId}`**：查詢參數 → `APIResponse<ProjectsListResponse>`
- **`POST /projects/{projectId}/rename`**：`ProjectRenameRequest` → `APIResponse<{}`>
- **`DELETE /projects/{projectId}`**：→ `APIResponse<{}`>

### 檢閱個別專案 (歷史)

```typescript
interface ProjectViewResponse extends APIResponse {
  // Only completed projects are relevant for viewing
  data: {
    projectId: string;
    projectName: string;
    query: string;
    functionLabels: string[];
    technologyLabels: string[];
    cells: ProjectViewMatrixCell[];
    settings: {
      [key: "blue_ocean_threshold" | "red_ocean_threshold"]: number; // Value should be of `double` type.
    };
    createdAt: string;
  };
}

interface ProjectViewMatrixCell {
  functionIndex: number;
  technologyIndex: number;
  patentCount: number;
  paperCount: number;
  patents: Patent[];
  papers: Paper[];
}
```

> 註：任何持有該 專案連結 並已登入的使用者皆可檢視該專案詳細資訊。

**端點：**

- **`GET /projects/{projectId}`** - → `APIResponse<ProjectViewResponse>`

### 專案重新命名請求/回應

重新命名現有專案。

```typescript
interface ProjectRenameRequest {
  newName: string;
}

interface ProjectRenameResponse {
  ...APIResponse; // 無特殊內容，可採用基礎回應介面
}
```

**端點：**

- **`POST /user/projects/{projectId}/rename`** - `ProjectRenameRequest` → `APIResponse<ProjectRenameResponse>`

### 專案刪除回應

刪除專案及所有相關資料。

```typescript
interface ProjectDeletionResponse {
  ...APIResponse; // 無特殊內容，可採用基礎回應介面}
```

**端點：**

- **`DELETE /user/projects/{projectId}`** - → `APIResponse<ProjectDeletionResponse>`

---

## 驗證標頭

所有需驗證端點需：

```
Authorization: Bearer <sessionToken>
Content-Type: application/json
```

無需驗證端點：

- `POST /auth/login`

---

## 錯誤處理

| 代碼 | 意義                  | 常見使用情境                |
| ---- | --------------------- | --------------------------- |
| 200  | OK                    | 成功的 GET/PUT/PATCH/DELETE |
| 201  | Created               | 成功的 POST 建立資源        |
| 202  | Accepted              | 請求已接受，進行非同步處理  |
| 400  | Bad Request           | 參數錯誤、請求格式錯誤      |
| 401  | Unauthorized          | 缺少/無效的 session token   |
| 403  | Forbidden             | 權限不足                    |
| 404  | Not Found             | 資源不存在                  |
| 408  | Request Timeout       | 查詢/AI 產生逾時            |
| 429  | Too Many Requests     | 超過速率限制                |
| 500  | Internal Server Error | 未預期的後端錯誤            |

---

## 速率限制

**建議限制：**

| 端點                                    | 限制 | 時間窗 |
| --------------------------------------- | ---- | ------ |
| `POST /search/execute`                  | 30   | 1 小時 |
| `POST /matrix/initialize-specification` | 20   | 1 小時 |
| `POST /matrix/generate`                 | 10   | 1 小時 |
| `POST /matrix/export`                   | 50   | 1 小時 |
| 預設                                    | 1000 | 1 小時 |

**回應標頭：**

```
X-RateLimit-Limit: <limit>
X-RateLimit-Remaining: <remaining>
X-RateLimit-Reset: <timestamp>
```

---

## 逾時預期

| 端點                                    | 逾時     |
| --------------------------------------- | -------- |
| `POST /search/execute`                  | 30-60 秒 |
| `POST /matrix/initialize-specification` | 15-30 秒 |
| `POST /matrix/generate`                 | 30-60 秒 |
| `POST /matrix/export`                   | 30-60 秒 |
| 其他端點                                | 5-10 秒  |

---

## 未來增強（第二階段）

### 矩陣分享與編輯

```typescript
// 矩陣分享與檢視（未來增強 - 非 MVP）
// 規劃於第二階段（2026年2月以後）

interface MatrixShareRequest {
  projectId: string;
  sharedWithUserId: string;
  permissions: "view" | "edit"; // 預設為 "view"
}

interface MatrixShareResponse {
  shareId: string;
  projectId: string;
  creatorId: string;
  sharedWithUserId: string;
  permissions: string;
  sharedAt: string; // ISO 8601
}
```

### 長時間操作的非同步處理

MVP 可同步處理，但建議以下情境改為非同步：

- 大型查詢結果（1000+ 專利/論文）
- 複雜矩陣生成
- 匯出 Excel

```typescript
// 未來增強：
interface AsyncJobResponse {
  jobId: string;
  status: "queued" | "processing" | "completed" | "failed";
  progress?: number; // 0-100
  resultUrl?: string; // 當 status === "completed" 時提供
  createdAt: string;
  completedAt?: string;
}

// 輪詢結果：
GET /jobs/{jobId} → AsyncJobResponse
```

### 選用快取標頭

```typescript
// 穩定資料可快取：
GET /user/projects
  Cache-Control: private, max-age=300  // 5 分鐘

GET /matrix/{projectId}
  Cache-Control: private, max-age=3600  // 1 小時

// 易變資料：
POST /search/execute
  Cache-Control: no-cache
```
