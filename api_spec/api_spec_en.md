# TEMatrix Backend API Specification

**Created**: Nov 03, 2025  
**Last Updated**: Nov 04, 2025

---

**Design Rationale**: "API as single source of truth" - the backend confirms what was processed, and the frontend can cache it in local state if needed but doesn't have to.

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

- **`POST /auth/login`**: `LoginRequest` → `APIResponse<LoginResponse>`
- **`POST /auth/logout`**: Headers: `Authorization: Bearer <token>` → `APIResponse<LogoutResponse>`

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
  yearRange?: YearRange; // By default, it's past 5 years
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
  sortingOptions: string[];
  executedAt: string;
}

interface FilterOptions {
  patentFilterOptions: Record<string, FilterOption>;
  paperFilterOptions: Record<string, FilterOption>;
}

type FilterOption = MultiselectFilter | RangeFilter;

interface MultiselectFilter {
  type: "multiselect";
  options: FilterValue[];
}

interface RangeFilter {
  type: "range";
  min: number;
  max: number;
  selectedMin: number;
  selectedMax: number;
}

interface FilterValue {
  value: string;
  count: number;
  active: boolean;
}
```

**Endpoints:**

- **`POST /search/execute`**: `SearchExecutionRequest` → `APIResponse<SearchExecutionResponse>`

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

- **`POST /matrix/initialize-specification`**: `MatrixSpecInitRequest` → `APIResponse<MatrixSpecInitResponse>`

## Step 3: Matrix Generation

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

- **`POST /matrix/generate`**: `MatrixGenerationRequest` → `APIResponse<MatrixGenerationResponse>`
<!-- - **`GET /matrix/{projectId}/cell/{functionIndex}/{technologyIndex}`**: → `APIResponse<MatrixCellDetailsResponse>` // We no longer need this, as Front-end will store the data. -->

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

- **`POST /matrix/view-settings`**: `MatrixViewSettingsSaveRequest` → `APIResponse<MatrixViewSettingsSaveResponse>`

## Step 4: Matrix Export & Review

### Matrix Export Request

```typescript
interface MatrixExportRequest {
  projectId: string;
}
```

**Endpoint:**

- **`POST /matrix/{projectId}/export`**: `MatrixExportRequest` → Binary Excel file with `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

### Matrix Sharing & Viewing

```typescript
interface MatrixShareRequest {
  projectId: string;
  userId: string; // To allow access to authenticated users
}

interface MatrixShareResponse {
  creatorId: string; // To check if the viewer can change the view settings
  ...MatrixGenerationResponse;
}
```

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
  // MATRIX_GENERATION: "matrix_generation"; // Final state - read-only
}

interface UserProgressData {
  patentsPapersSelection?: SelectionProgress;
  matrixSpecification?: MatrixSpecProgress;
}

interface SelectionProgress {
  selectedPatentIds: string[];
  selectedPaperIds: string[];
  currentTabIndex: number;
  filters: FilterOptions[];
  sorting: SortSettings[];
}

interface SortSettings {
  index?: number;
  field: string;
  order: "ASC" | "DESC";
  active: boolean;
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

- **`POST /user/progress`**: `ProgressSaveRequest` → `APIResponse<ProgressSaveResponse>`
- **`GET /user/progress/{projectId}`**: → `APIResponse<ProgressRetrievalResponse>`

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
  status: "in_progress" | "completed";
}

interface ProjectRenameRequest {
  newName: string;
}
```

**Endpoints:**

- **`GET /user/projects/{userId}`**: Query params → `APIResponse<ProjectsListResponse>`
- **`POST /user/projects/{projectId}/rename`**: `ProjectRenameRequest` → `APIResponse<{}>`
- **`DELETE /user/projects/{projectId}`**: → `APIResponse<{}>`

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

- `POST /auth/login`

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

---

## Rate Limiting

**Suggested Limits:**

| Endpoint                                | Limit | Window |
| --------------------------------------- | ----- | ------ |
| `POST /search/execute`                  | 30    | 1 hour |
| `POST /matrix/initialize-specification` | 20    | 1 hour |
| `POST /matrix/generate`                 | 10    | 1 hour |
| `POST /matrix/export`                   | 50    | 1 hour |
| Default                                 | 1000  | 1 hour |

**Response Headers:**

```
X-RateLimit-Limit: <limit>
X-RateLimit-Remaining: <remaining>
X-RateLimit-Reset: <timestamp>
```

---

## Timeout Expectations

| Endpoint                                | Timeout |
| --------------------------------------- | ------- |
| `POST /search/execute`                  | 30-60s  |
| `POST /matrix/initialize-specification` | 15-30s  |
| `POST /matrix/generate`                 | 30-60s  |
| `POST /matrix/export`                   | 30-60s  |
| Other endpoints                         | 5-10s   |

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
