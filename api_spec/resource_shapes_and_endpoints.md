# Resource Shapes & Endpoints

Last updated: 2025-11-17

This document describes canonical resource shapes and RESTful endpoints for the TEMatrix backend API, following REST best practices.

Base conventions

- Base path: `/api/v1`
- All endpoints require `Authorization: Bearer {token}` except public endpoints (e.g., `POST /api/v1/sessions`).
- Use plural nouns for resources: `/projects`, `/patents`, `/matrices`.
- Use HTTP verbs (GET/POST/PUT/PATCH/DELETE) and avoid verbs in the path.
- Support filtering, sorting and pagination for collection endpoints. Prefer cursor-based pagination for large datasets, query-based for smaller.
- Long-running operations should be handled asynchronously via `/jobs` resource.
- All POSTs that create resources should return 201 with Location header or 202 for async with job link.

Common response shape

```typescript
interface APIResponse<T = any> {
  status: "success" | "error";
  code: number; // HTTP status code
  timestamp: string; // ISO 8601
  apiVersion: string; // v1
  data?: T;
  error?: APIError;
}

interface APIError {
  type:
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
  message: string;
  details?: { field?: string }[];
}
```

Pagination, filtering, and sorting

- Pagination: `page` (1-based) and `limit` or `cursor` + `limit` when using cursor-based paging.
- Sorting: `sort=field,asc|desc` support multiple sorts: `sort=field1,asc&sort=field2,desc`.
- Filtering: Query params named after attributes: `cpc=H01L&from=2018&to=2024`.

Resources and shapes

## sessions

- Authentication tokens

POST /api/v1/sessions
Request:

```json
{ "username": "user@example.com", "password": "pass" }
```

Response 201:

```json
{
  "status": "success",
  "data": { "sessionToken": "...", "userId": "...", "expiresIn": 3600 }
}
```

DELETE /api/v1/sessions

- Invalidate current session token (Auth header required)

## users

`GET /api/v1/users/{userId}`
`PATCH /api/v1/users/{userId}`

Shape:

```typescript
interface User {
  id: string;
  email: string;
  username: string;
  createdAt: string;
}
```

## patents and papers (read-only)

`GET /api/v1/patents`
`GET /api/v1/patents/{patentId}`
`GET /api/v1/papers`
`GET /api/v1/papers/{paperId}`

Query parameters supported: `query`, `cpc`, `fromYear`, `toYear`, `journal`, `author`, `citations`, `page`, `limit`, `sort`.

Shapes

```typescript
interface Patent {
  id: string;
  title: string;
  inventor: string;
  patentHolder: string;
  filedYear: number;
  cpc: string[];
  keywords: string[];
  concepts?: string;
  link: string;
}
interface Paper {
  id: string;
  title: string;
  author: string;
  journal: string;
  publicationYear: number;
  citations: number;
  cpc: string[];
  concepts?: string;
  link: string;
}
```

## projects

A `project` represents saved progress (selection and matrix specs). Use `projects` resource to create, list, update, and delete projects.

`GET /api/v1/projects?ownerId={userId}&status=in_progress&page=1&limit=20`
`POST /api/v1/projects`
`GET /api/v1/projects/{projectId}`
`PUT /api/v1/projects/{projectId}`
`DELETE /api/v1/projects/{projectId}`

Project shape

```typescript
interface Project {
  projectId: string;
  projectName: string;
  ownerId: string;
  status: "in_progress" | "completed";
  query?: string;
  createdAt: string;
  lastModifiedAt: string;
}
```

POST request example

```json
{
  "projectName": "My Matrix",
  "query": "(AI OR ML) AND H01L",
  "selectedPatentIds": ["p123", "p456"],
  "selectedPaperIds": ["pp1"]
}
```

## searches (project-scoped; also support global searches)

- Execute a search and store results snapshot so projects are reproducible.

`POST /api/v1/projects/{projectId}/searches` - Run and save a search for a project (returns searchId)
`POST /api/v1/searches` - Run a global search and return results
`GET /api/v1/projects/{projectId}/searches/{searchId}`

Search request

```typescript
interface SearchExecutionRequest {
  caseName?: string;
  query: string;
  yearRange?: { min: number; max: number };
}
```

Search response (summary)

```typescript
interface SearchExecutionResponse {
  caseId: string;
  caseName: string;
  query: string;
  patents: string[];
  papers: string[];
  filterOptions: object;
  sortingOptions: string[];
  executedAt: string;
}
```

If search execution is long-running and the `mode=async` is requested or server decides it's heavy, return 202 and a job link: `Location: /api/v1/jobs/{jobId}`.

## selections

User selections of patents/papers within a project: store snapshots for reuse.

`POST /api/v1/projects/{projectId}/selections`
`GET /api/v1/projects/{projectId}/selections/{selectionId}`
`GET /api/v1/projects/{projectId}/selections`

Selection shape

```typescript
interface Selection {
  id: string;
  projectId: string;
  name?: string;
  notes?: string;
  selectedPatentIds: string[];
  selectedPaperIds: string[];
  createdAt: string;
}
```

## matrix-specifications (dimension specs)

`POST /api/v1/projects/{projectId}/matrix-specifications` - Stores the current dimension specification and returns suggestions.
`GET /api/v1/projects/{projectId}/matrix-specifications/{specId}`
`GET /api/v1/projects/{projectId}/matrix-specifications`
`PUT /api/v1/projects/{projectId}/matrix-specifications/{specId}`

Spec payload / response

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

## matrices

- Matrices are snapshots generated for a project.

`POST /api/v1/projects/{projectId}/matrices` - create a matrix (may be async)
`GET /api/v1/projects/{projectId}/matrices/{matrixId}`
`GET /api/v1/projects/{projectId}/matrices` - list matrices
`GET /api/v1/projects/{projectId}/matrices/{matrixId}?detail=summary|full` - fetch with optional detail
`DELETE /api/v1/projects/{projectId}/matrices/{matrixId}`

Create request

```typescript
interface MatrixGenerationRequest {
  projectId: string;
  functionLabels: string[];
  technologyLabels: string[];
}
```

Create response (sync)

- 201 with Location header: `/api/v1/projects/{projectId}/matrices/{matrixId}` and body `MatrixGenerationResponse`
  Create response (async)
- 202 Accepted with `Location: /api/v1/jobs/{jobId}`.

Matrix response shape

```typescript
interface MatrixCell {
  functionIndex: number;
  technologyIndex: number;
  patentCount: number;
  paperCount: number;
  patentIds: string[];
  paperIds: string[];
}
interface Matrix {
  matrixId: string;
  projectId: string;
  functionLabels: string[];
  technologyLabels: string[];
  cells: MatrixCell[];
  createdAt: string;
}
```

Cells
`GET /api/v1/projects/{projectId}/matrices/{matrixId}/cells?functionIndex={i}&technologyIndex={j}&page=1&limit=50` - lists cell(s) depending on filters.
`GET /api/v1/projects/{projectId}/matrices/{matrixId}/cells/{cellId}` - details of a single cell (includes patents and papers full details if detail=full).

Design note: prefer sending summary cells in the main matrix resource and allow clients to fetch full details of the cell on demand. Avoid overly deep nesting by exposing `/matrices/{matrixId}/cells` rather than repeating projectId if possible.

## matrix-view-settings

`PUT /api/v1/projects/{projectId}/matrix-view-settings` — store or update the blue/red ocean thresholds and other view preferences.
`GET /api/v1/projects/{projectId}/matrix-view-settings`

Shape

```typescript
interface MatrixViewSettings {
  projectId: string;
  blueOceanThreshold: number;
  redOceanThreshold: number;
  updatedBy: string;
  updatedAt: string;
}
```

## exports

`POST /api/v1/projects/{projectId}/matrices/{matrixId}/exports` - create an export (xlsx). Return 202 + job.
`GET /api/v1/projects/{projectId}/matrices/{matrixId}/exports/{exportId}` - status & download URL when finished.

Export shape

```typescript
interface ExportAttachment {
  exportId: string;
  projectId: string;
  matrixId: string;
  status: "queued" | "processing" | "completed" | "failed";
  location?: string;
  createdAt: string;
}
```

## shares

Project sharing management
`POST /api/v1/projects/{projectId}/shares`
`GET /api/v1/projects/{projectId}/shares`
`DELETE /api/v1/projects/{projectId}/shares/{shareId}`

Share payload

```typescript
interface ProjectShareRequest {
  projectId: string;
  sharedWithUserId: string;
  permission: "view" | "edit";
}
```

## progress

GET & PUT project progress snapshot
`GET /api/v1/projects/{projectId}/progress`
`PUT /api/v1/projects/{projectId}/progress` — store latest progress

Progress shape

```typescript
interface ProgressSaveRequest {
  projectId: string;
  stepCode: "selection" | "matrix_specification";
  progress: {
    selectedPatentIds?: string[];
    selectedPaperIds?: string[];
    selectedFunctionLabels?: string[];
    selectedTechnologyLabels?: string[];
  };
}
```

## jobs (async)

- POST /api/v1/jobs (if we want ad-hoc job injection)
- GET /api/v1/jobs/{jobId} — job status and result info.

Job shape

```typescript
interface AsyncJobResponse {
  jobId: string;
  status: "queued" | "processing" | "completed" | "failed";
  progress?: number;
  resultUrl?: string;
  type?: string;
  createdAt: string;
  completedAt?: string;
}
```

## policies & edge cases

1. Idempotency: where logical, use `idempotency-key` for POSTs that create but may be retried by clients (e.g., project creation, matrix generation) — accept a `Idempotency-Key` header.
2. Timeouts: keep synchronous requests capped at 30–60s; otherwise switch to async job flow.
3. Rate limiting: Provide `X-RateLimit-*` headers and 429 status.
4. Resource ownership: projects, matrices, exports, shares are owned resources; enforce owner-level and share-based permissions.
5. Large results: Use cursor-based pagination for large collections (patents, papers).
6. ETag / conditional GETs: support `ETag`, `If-None-Match` and `If-Modified-Since` for cacheable resources.

## Sample flows

1. Create a project + run a project-scoped search + save selection + initialize matrix specification + generate matrix + create export:

- POST /api/v1/projects (create)
- POST /api/v1/projects/{projectId}/searches (run search)
- POST /api/v1/projects/{projectId}/selections (save selection)
- POST /api/v1/projects/{projectId}/matrix-specifications (init spec)
- POST /api/v1/projects/{projectId}/matrices (generate) -> 202 + Location: /api/v1/jobs/{jobId}
- POST /api/v1/projects/{projectId}/matrices/{matrixId}/exports (export) -> 202 + Location: /api/v1/jobs/{exportJobId}

2. Fetch a matrix and cell details

- GET /api/v1/projects/{projectId}/matrices/{matrixId}?detail=summary
- GET /api/v1/projects/{projectId}/matrices/{matrixId}/cells/{cellId}?detail=full

## Clarifying questions

1. Should we standardize on "project" or "case" as the canonical resource name? (Prefer `project` in the UI.)
2. Should long-running tasks (matrix generation, export) be asynchronous by default? (Recommended: yes)
3. Are patents & papers strictly read-only via our API? (Recommended: yes, ingestion is an admin/backend process.)
4. Webhooks vs polling: Do you want to support webhooks for job completion or prefer polling? (Webhooks recommended for a push pattern.)

## Next steps

- Once you confirm the naming conventions (`project` vs `case`, async defaults, webhook support), I can update `api_spec_en.md` and `api_spec_zh_TW.md` to include the final endpoints and include example request/response payloads or generate an OpenAPI document.

# End of file

GitHub Copilot

Below is a concise, RESTful mapping for the project lifecycle you described (one project → one query → one selections → one matrix). I include recommended endpoints, HTTP methods, idempotency guidance, and short example payloads and responses.

Summary of actions and REST mappings

- Create project: POST /api/v1/projects
- Save/update selections: PUT /api/v1/projects/{projectId}/selections
- Confirm selections (advance project state and optionally trigger matrix spec suggestions): PATCH /api/v1/projects/{projectId}
- Save/update matrix dimensions (matrix specification): PUT /api/v1/projects/{projectId}/matrix-specification
- Generate or create matrix (1:1 with project): PUT /api/v1/projects/{projectId}/matrix (synchronous) or POST /api/v1/projects/{projectId}/jobs to generate async
- Finalize (irreversible): PATCH /api/v1/projects/{projectId} with final status

Reasoning & best-practices

- Use noun resources and HTTP verbs: no verbs in URL paths.
- Use PUT for idempotent full updates (selections, matrix-spec, matrix).
- Use PATCH for partial updates/status transitions (confirm, finalize).
- Use POST for creating jobs or non-idempotent resources (if generating a matrix is queued).
- Return appropriate codes: 201 Created (creation), 200/204 Success, 202 Accepted (async), 409 Conflict (e.g., trying to modify a finalized project), 423 Locked (optional).
- Provide HATEOAS links or a nextPhase field to avoid unnecessary round-trips.

Detailed endpoints and examples

1. Create a project

- POST /api/v1/projects
- Request:
  {
  "name": "New Project",
  "query": "((AI OR ML) AND battery) NOT obsolete",
  // optional meta
  }
- Response 201 Created
  Headers: Location: /api/v1/projects/{projectId}
  Body:
  {
  "status": "success",
  "data": {
  "projectId": "proj_123",
  "phase": "query_built",
  "links": {
  "selections": "/api/v1/projects/proj_123/selections"
  }
  }
  }

2. Save/update selections (idempotent)

- PUT /api/v1/projects/{projectId}/selections
- Request:
  {
  "patentIds": ["p1", "p2"],
  "paperIds": ["r1", "r2"]
  }
- Response (synchronous path) 200 OK:
  {
  "status": "success",
  "data": {
  "projectId": "proj_123",
  "selections": { "patentIds": [...], "paperIds": [...] },
  "matrixDimensionSuggestions": {
  "functions": ["f1", "f2"],
  "technologies": ["t1", "t2"]
  },
  "nextPhase": "matrix_specification",
  "links": {
  "matrixSpecification": "/api/v1/projects/proj_123/matrix-specification",
  "confirmSelections": "/api/v1/projects/proj_123"
  }
  }
  }
- If AI suggestions generation is long: return 202 Accepted + job link:
  Headers: Location: /api/v1/projects/proj_123/jobs/job_45
  Body:
  { "jobId":"job_45", "status":"pending", "links": {"jobStatus": "/api/v1/projects/proj_123/jobs/job_45"} }

3. Confirm selections (state transition; triggers suggestion generation or marks step complete)

- PATCH /api/v1/projects/{projectId}
- Request (example):
  { "status": "selections_confirmed" }
- Response 200 OK:
  {
  "status": "success",
  "data": {
  "projectId":"proj_123",
  "status":"selections_confirmed",
  "links": { "matrixSpecification": "/api/v1/projects/proj_123/matrix-specification" }
  }
  }
- If confirming triggers a heavy job (generation) → 202 Accepted + job URL.

4. Save/update matrix dimensions

- PUT /api/v1/projects/{projectId}/matrix-specification
- Request:
  {
  "functions": ["f1","f2"],
  "technologies": ["t1","t2"],
  "viewSettings": { "blueThreshold": 0.2, "redThreshold": 0.8 }
  }
- Response 200 OK:
  {
  "status": "success",
  "data": {
  "specId": "spec_1",
  "functions": [...],
  "technologies": [...],
  "nextPhase":"ready_to_generate",
  "links": { "generateMatrix": "/api/v1/projects/proj_123/matrix" }
  }
  }

5. Generate / create matrix (1:1)

- Option A — synchronous create/replace (fast generation):
  PUT /api/v1/projects/{projectId}/matrix
  Body: { "specId": "spec_1", "mode": "sync" }
  Response 201 Created:
  {
  "status":"success",
  "data": {
  "matrixId":"matrix_123",
  "projectId":"proj_123",
  "links": { "matrix": "/api/v1/projects/proj_123/matrix" }
  }
  }
- Option B — asynchronous generate (recommended if heavy):
  POST /api/v1/projects/{projectId}/jobs
  Body: { "type":"generate_matrix", "specId":"spec_1" }
  Response 202 Accepted
  Headers: Location: /api/v1/projects/{projectId}/jobs/{jobId}
  Body:
  { "jobId":"job_99", "status":"pending", "links":{"jobStatus":"/api/v1/projects/proj_123/jobs/job_99"} }

- GET matrix:
  GET /api/v1/projects/{projectId}/matrix
  Response 200 OK -> returns matrix data (cells, aggregated stats). Use caching/ETag.

6. Finalize (irreversible)

- PATCH /api/v1/projects/{projectId}
- Request:
  { "status":"finalized" }
- Behavior: backend enforces immutability for selections/specs; attempts to modify return 409/423.
- Response 200 OK:
  {
  "status":"success",
  "data": {
  "projectId":"proj_123",
  "status":"finalized",
  "links": { "export": "/api/v1/projects/proj_123/export" }
  }
  }

Rules & enforcement

- Use ETag/If-Match on PUT/PATCH for optimistic concurrency.
- Prevent modifications to selections/spec/specification/matrix if project.status === 'finalized' (return 409 Conflict).
- Use 202 + job resource for heavy or long-running generation/export.
- Return nextPhase & resource links with responses to let the UI transition without extra requests (reduce round trips).

Notes on your one-matrix-per-project rule

- Since one matrix per project, use PUT /projects/{projectId}/matrix to create/replace (idempotent) or disallow replace if you prefer single immutable final matrix (use 409).
- If generation is heavy, prefer asynchronous job creation and store matrix once job done, then GET /projects/{projectId}/matrix returns the result.

If you confirm these choices, I can generate a compact API contract (endpoints + request/response JSON schemas) to paste into your spec files.

---

建議：以同步回應預設（若建議生成輕量或可快速回傳），但支援非同步（若建議生成耗時）。Put /selections 應回傳更新後的 selections 並提供「下一步所需資料或連結」，避免前端多次 round-trip；若生成工作耗時則回 202 + job URL。以下為精簡建議與範例。

1. 同步（建議預生成且快速）

- 狀態碼：200 OK
- 回傳內容：更新後 selections、AI 建議的矩陣維度、下一步 phase 和導向連結
- 範例：

```json
{
  "status": "success",
  "data": {
    "projectId": "proj_123",
    "selections": {
      "patentIds": ["p1", "p2"],
      "paperIds": ["r1", "r2"]
    },
    "matrixDimensionSuggestions": {
      "functions": ["functionA", "functionB"],
      "technologies": ["techX", "techY"]
    },
    "nextPhase": "matrix_specification",
    "links": {
      "getMatrixSpec": "/api/v1/projects/proj_123/matrix-specification",
      "generateMatrix": "/api/v1/projects/proj_123/matrices"
    }
  }
}
```

2. 非同步（建議生成耗時或需排隊）

- 狀態碼：202 Accepted
- 回傳內容：job 資訊與 polling/Location
- Header：Location: /api/v1/projects/{projectId}/jobs/{jobId}
- 範例：

```json
{
  "status": "success",
  "data": {
    "jobId": "job_456",
    "jobType": "generate_matrix_suggestions",
    "status": "pending",
    "links": {
      "jobStatus": "/api/v1/projects/proj_123/jobs/job_456"
    }
  }
}
```

3. 若不想包含 suggestions（把工作拆成兩個請求）

- 回傳：200 OK + selections 以及一個 next link
- 前端呼叫 GET /api/v1/projects/{projectId}/matrix-specification 或 GET /api/v1/projects/{projectId}/ai-suggestions

要點摘要

- PUT /projects/{id}/selections：用於 idempotent 更新 selections，回 200（含 body）或 204（無回 body）。
- 若欲提供「一步到位」體驗（減少 round-trips），在回應包含 next-step data/links。
- 若 AI 處理耗時，採 202 + jobs 對外暴露 job resource。
- 建議同時回傳 links（HATEOAS）與 nextPhase，便於 client 自動導向。
- 加上 ETag/If-Match 或使用樂觀鎖來避免並發衝突。

快速問題（供決策）

1. AI 建議生成通常能在同步時段內完成還是會常常超時？
2. 你希望以單次 API 呼叫（PUT）就拿到 suggestions，還是分離成兩步（PUT selections + GET suggestions/job）？
