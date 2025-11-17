# TEMatrix Backend API Specification

**Created**: Nov 03, 2025  
**Last Updated**: Nov 17, 2025

---

**Design Rationale**: "API as single source of truth" - the backend confirms what was processed, and the frontend can cache it in local state if needed but doesn't have to.

---

## Resources & Endpoints

Base URL: `/api/v1`

In line with REST best practices, the API is organized around key resources that map to user flows in the app.

### Resources:

- `users`: Currently only for authentication, as a user has `username` and `password`.
- `sessions`: For managing user sessions (login/logout).
- `projects`: Each project represents a workflow – `Build query` -> `Select patents and papers` -> `Specify matrix dimensions` -> `Generate and finalize matrix`, etc.

### Endpoints:

**Authentication:**

- `POST /sessions`: User login, returns session token.
- `DELETE /sessions`: User logout, invalidates session token.

**Project Workflow:**

Step 0. Build query

- `GET /query-builder/keyword-suggestions`: Get AI keyword suggestions based on seed keyword.
  - Request param: `seed_keyword`
  - Response: List of suggested keywords.
- `POST /projects`: Executing a search is equivalent to creating a new project.
  - Request body includes the boolean query built by user, and user ID from session.
  - Response includes the project ID and search results (patents and papers).

Step 1. Select patents and papers

- `PATCH /projects/{projectId}/selections`: Save selected patents and papers for the next step.

  - Request body includes the selected patent and paper IDs.
  - Response includes save confirmation, and pre-generated AI suggestions for matrix dimensions. This action is synchronous for our use case. For bigger datasets, we consider making this asynchronous with a job resource.

<!-- - `PATCH /projects/{projectId}`: Confirm selections (advance project state and optionally trigger matrix spec suggestions).
  - Request body includes ids of selected patents and papers.
  - Response includes AI suggestions for matrix dimensions. -->

Step 2. Specify matrix dimensions

- `PATCH /projects/{projectId}/matrix-specification`: Save/update matrix dimensions. Generate matrix based on selected patents/papers and specified dimensions.
  - Request body includes function and technology labels.
  - Response includes generated matrix data.

Step 3. Save/finalize project and export matrix

- `PATCH /projects/{projectId}/status`: Finalize (irreversible). Matrix can be shared only after finalization. Shared matrix can be viewed by anyone using the project link.
  - Request body includes final status.
  - Response includes confirmation of finalization.
- `GET /projects/{projectId}/export`: Export the matrix as an Excel file.

**User progress management:**

- `POST /users/{userId}/progress/{projectId}`: Save user progress at a specific step in the project workflow.
- `GET /users/{userId}/progress/{projectId}`: Retrieve saved progress for a specific project.

**Matrix sharing and view settings management:**

- `GET /projects/{projectId}/share`: Share the matrix with others via a link. Anyone with the link can view the matrix.
- `GET /projects/{projectId}`: View (shared) matrix. Non-finalized projects cannot be shared/viewed.
- `POST /projects/{projectId}/view-settings`: Anyone can have their own settings (blue/red ocean threshold). By default, use the creator's settings. Creator's action is treated the same as others, but they can change the default settings for new viewers.

**Projects/searches management:**

- `GET /users/{userId}/projects`: Get list of user's projects (searches).
- `GET /users/me/projects`: Get list of current logged-in user's projects (searches).
- `PATCH /users/{userId}/projects/{projectId}`: Rename a project.
- `DELETE /users/{userId}/projects/{projectId}`: Delete a project.

**Notes:**

> To clear confusion, when user continues an open project, we load the saved progress.
> When user moves forward to next step, we use `PATCH` end points to update the project state and data.
> When user reverts to previous step and make changes, if they choose to move forward again, we overwrite the project state and data with new inputs, using `PATCH` end points.

// [TODO]: To resolve `PATCH` ambiguity, right now, there's collision with saving progress and moving to the next step. We can consider adding a query param like `?advance=true` to indicate moving forward to next step.

---

## Common Interfaces

### Base Response Interface

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

### Error Interface

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

## Data Model Interfaces

### Patent Interface

```typescript
interface Patent {
  id: string;
  title: string;
  inventor: string;
  patentHolder: string;
  filedYear: number;
  priorityYear?: number; // Data may not be available
  publicationYear?: number; // Data may not be available
  grantYear?: number; // Data may not be available
  cpc: string[];
  keywords: string | string[];
  concepts: string;
  link: string;
}
```

### Paper Interface

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

## Authentication

### Login Request/Response

```typescript
interface LoginRequest {
  username: string;
  password: string;
}

interface LoginResponse {
  sessionToken: string;
  userId: string;
  username: string;
  expiresIn: number; // seconds
}

interface LogoutResponse {
  message: string;
}
```

**Endpoints:**

- **`POST /sessions`**: `LoginRequest` → `APIResponse<LoginResponse>`
- **`DELETE /sessions`**: Headers: `Authorization: Bearer <token>` → `APIResponse<LogoutResponse>`

---

## Step 0: Query Builder

### Keyword Suggestions Request/Response

```typescript
interface KeywordSuggestionsRequest {
  seed_keyword: string;
}

interface KeywordSuggestionsResponse {
  keyword: string;
  suggestions: string[];
}
```

**Endpoint:**

- **`GET /query-builder/keyword-suggestions`**: `KeywordSuggestionsRequest` → `APIResponse<KeywordSuggestionsResponse>`

## Step 1: Patent & Paper Selection

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

**Endpoints:**

- **`PATCH /projects/{projectId}/selections`**: `SearchExecutionRequest` → `APIResponse<SearchExecutionResponse>`

## Step 2: Matrix Dimension Specification

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

**Endpoint:**

- **`PATCH /projects/{projectId}/matrix-specification`**: `MatrixSpecInitRequest` → `APIResponse<MatrixSpecInitResponse>`

## Step 3: Save/Finalize Project and Export Matrix

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
    patents: Patent[]; // Full patent details
    papers: Paper[]; // Full paper details
    patentCount: number;
    paperCount: number;
  };
}
```

**Endpoints:**

- **`PATCH /projects/{projectId}/status`**: `MatrixGenerationRequest` → `APIResponse<MatrixGenerationResponse>`

### Matrix Export Request

```typescript
interface MatrixExportRequest {
  projectId: string;
}
```

**Endpoint:**

- **`GET /projects/{projectId}/export`**: `MatrixExportRequest` → Binary Excel file with `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

## Matrix Sharing & Viewing

// [TODO]: `GET /projects/{projectId}/share` and `GET /projects/{projectId}` already defined above

### Matrix View Settings

To save the "鴻海臨界" and "藍海臨界" preferences.

```typescript
interface MatrixViewSettingsSaveRequest {
  projectId: string;
  settings: {
    [key: "blue_ocean_threshold" | "red_ocean_threshold"]: number; // Value should be of `double` type.
  };
}

interface MatrixViewSettingsSaveResponse {
  // Nothing special, could adopt the base response interface
}
```

**Endpoint:**

- **`POST /projects/{projectId}/view-settings`**: `MatrixViewSettingsSaveRequest` → `APIResponse<MatrixViewSettingsSaveResponse>`

---

## Progress & Session Management

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

**Endpoints:**

- **`POST /users/{userId}/progress/{projectId}`**: `ProgressSaveRequest` → `APIResponse<ProgressSaveResponse>`
- **`GET /users/{userId}/progress/{projectId}`**: → `APIResponse<ProgressRetrievalResponse>`

---

## Project Management

### Getting Project List

```typescript
interface ProjectsListResponse {
  projects: ProjectSummary[];
}

interface ProjectSummary {
  projectId: string;
  projectName: string;
  createdAt: string;
  lastModifiedAt: string;
  status: "OPEN" | "COMPLETED";
}

interface ProjectRenameRequest {
  newName: string;
}
```

**Endpoints:**

- **`GET /users/{userId}/projects` or `GET /users/me/projects`**: Query params → `APIResponse<ProjectsListResponse>`
- **`PATCH /users/{userId}/projects/{projectId}`**: `ProjectRenameRequest` → `APIResponse<{}>`
- **`DELETE /users/{userId}/projects/{projectId}`**: → `APIResponse<{}>`

### Reviewing Individual Projects (History)

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

> Note: Any "logged in" user with the project link can view the project details.

**Endpoint:**

- **`GET /projects/{projectId}`** - → `APIResponse<ProjectViewResponse>`

### Project Rename Request/Response

Renames an existing project.

```typescript
interface ProjectRenameRequest {
  newName: string;
}

interface ProjectRenameResponse {
  ...APIResponse; // Nothing special, could adopt the base response interface
}
```

**Endpoint:**

- **`POST /user/projects/{projectId}/rename`** - `ProjectRenameRequest` → `APIResponse<ProjectRenameResponse>`

### Project Deletion Response

Deletes a project and all associated data.

```typescript
interface ProjectDeletionResponse {
  ...APIResponse; // Nothing special, could adopt the base response interface}
```

**Endpoint:**

- **`DELETE /user/projects/{projectId}`** - → `APIResponse<ProjectDeletionResponse>`

---

## Authentication Headers

All authenticated endpoints require:

```
Authorization: Bearer <sessionToken>
Content-Type: application/json
```

Unauthenticated endpoints:

- `POST /sessions`

---

## Error Handling

| Code | Meaning               | Common Use Projects                   |
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

<!-- ---

## Rate Limiting

**Suggested Limits:**

| Endpoint                                           | Limit | Window |
| -------------------------------------------------- | ----- | ------ |
| `POST /projects`                                   | 30    | 1 hour |
| `PATCH /projects/{projectId}/selections`           | 20    | 1 hour |
| `PATCH /projects/{projectId}/matrix-specification` | 10    | 1 hour |
| `GET /projects/{projectId}/export`                 | 50    | 1 hour |
| Default                                            | 1000  | 1 hour |

**Response Headers:**

```
X-RateLimit-Limit: <limit>
X-RateLimit-Remaining: <remaining>
X-RateLimit-Reset: <timestamp>
```

---

## Timeout Expectations

| Endpoint                                           | Timeout |
| -------------------------------------------------- | ------- |
| `POST /projects`                                   | 30-60s  |
| `PATCH /projects/{projectId}/selections`           | 15-30s  |
| `PATCH /projects/{projectId}/matrix-specification` | 30-60s  |
| `GET /projects/{projectId}/export`                 | 30-60s  |
| Other endpoints                                    | 5-10s   | -->

---

## Future Enhancements (Phase 2)

### Matrix Sharing & Editing

```typescript
// Matrix Sharing & Viewing (Future Enhancement - Not MVP)
// Planned for Phase 2 (Feb 2026+)

interface MatrixShareRequest {
  projectId: string;
  sharedWithUserId: string;
  permissions: "view" | "edit"; // "view" by default
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

### Async processing for long operations

For MVP, synchronous is fine, but consider async for:

- Large search results (1000+ patents/papers)
- Complex matrix generation
- Export to Excel

```typescript
// Future enhancement:
interface AsyncJobResponse {
  jobId: string;
  status: "queued" | "processing" | "completed" | "failed";
  progress?: number; // 0-100
  resultUrl?: string; // Available when status === "completed"
  createdAt: string;
  completedAt?: string;
}

// Poll for results:
GET /jobs/{jobId} → AsyncJobResponse
```

### Optional Caching Headers

```typescript
// For stable data that can be cached:
GET /user/projects
  Cache-Control: private, max-age=300  // 5 minutes

GET /matrix/{projectId}
  Cache-Control: private, max-age=3600  // 1 hour

// For volatile data:
POST /search/execute
  Cache-Control: no-cache
```
