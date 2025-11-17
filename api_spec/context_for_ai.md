# Requirements & background

Last updated: 2025.11.17

This is an AI-powered technology function matrix generation app, with the following key features:

1. Matrix generation
2. Project management and sharing
3. Folder management (will be added later)
4. User authentication

## 1. Matrix generation

These are the key steps:

1. Build query (using boolean logic)
2. Select patents and papers to form the matrix
3. Specify matrix dimensions
4. View and customize matrix
5. Save project, and export matrix

User can save their progress as a "project" at Step 2 and 3. Once the project is saved, it will appear as "ongoing" projects, and onece a project is finalized, it will become "completed" and appear under historical searches.

### Step 1. Building query

- User can input keywords, and get AI keyword suggestions from the backend API.
- User can build complex boolean queries using AND, OR, NOT operators.
- When satisfied, user can execute the search to get patents and papers matching the query.

## Step 2. Selecting patents and papers

User can view the search results, filter and sort them using various criteria.

## Step 3. Specifying matrix dimensions

- There will be AI suggestions for matrix dimensions based on the selected patents and papers.
- User can select from the suggested dimensions or customize their own dimensions.
- When ready, user can generate the matrix.

## Step 4. Viewing and customizing matrix

Here, user can view the generated matrix, and customize it by specifying:

- Blue ocean threshold
- Red ocean threshold

## Step 5. Saving project and exporting matrix

- User can save the project to finalize it: No backtracking to previous steps after saving.
- User can downoad the matrix as an Excel file.
- User can still modify the blue/red ocean thresholds after saving. Everytime user modifies the thresholds, the system will prompt to ask where user wishes to save the new view settings.

## API Resources & Relationships (Guided by REST Best Practices)

This section maps the user flows in the app to API resources and clarifies the relationships between them. All endpoints should be prefixed with a version, for example: /api/v1.

Key design principles applied from REST best practices

- Use plural resource names (e.g., /projects, /users, /patents).
- Use HTTP verbs for operations (GET/POST/PUT/PATCH/DELETE), and avoid verbs in URL paths.
- Prefer resource nesting for logical ownership but avoid deep nesting (>3 levels).
- Support filtering, sorting, and pagination on collection GETs.
- Use appropriate status codes and include helpful error payloads.
- Prefer resource-specific endpoints for long-running jobs (e.g., /jobs).

Top-level resources

- users: user profiles, permissions and user settings.
- sessions: authentication tokens (POST /sessions to login, DELETE /sessions to logout).
- projects: saved workflows / cases (one user owns many projects).
- patents and papers: searchable collections; read-only from our perspective. We won't allow users to directly access or modify these resources.
<!-- - searches: executed search sessions; these can be project-scoped. -->
- matrices: generated matrices tied to a project and optionally versioned.
  <!-- - jobs: long-running operations (matrix generation, export) and their statuses. -->
  <!-- - exports: attachments (such as an Excel file) created by jobs. -->
- exports: a project can be exported multiple times, but the exported Excel file is the same if the matrix and view settings are unchanged.
- shares: permissions to share projects (view only) with other users.

Relationships (cardinality)

- User (1) -> (N) Project: users own projects.
- Project (1) -> (1) Search/Matrix/Progress/Export: each project is one search/matrix/progress/export.
<!-- - Project (1) -> (N) Matrix: projects may have several generated matrices (each one is an immutable snapshot once generated). -->
- Matrix (1) -> (M) Cell: a matrix contains many cells; cells reference patents and papers.

## Endpoints

### Authentication

- Authentication tokens

`POST /api/v1/sessions`
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

`DELETE /api/v1/sessions`

### Step 1. Build query

`GET /api/v1/ai-suggestions/query-keywords`: Get AI keyword suggestions based on user input as seed keyword, for the query builder stage.

`POST /api/v1/projects`: A new search is equivalent to a new project, and get relevant papers and patents.

- Request body includes the boolean query built by user.
- Response includes the project ID and search results (patents and papers).

### Step 2. Select patents and papers

`PUT /api/v1/projects/{projectId}/selections`: Save selected patents and papers for the next step.

- Request body includes the selected patent and paper IDs.
- Response includes save confirmation, and pre-generated AI suggeestions for matrix dimensions.
- This is synchronous for our use case. For bigger datasets, consider making this asynchronous with a job resource.

- Confirm selections (advance project state and optionally trigger matrix spec suggestions): `PATCH /api/v1/projects/{projectId}`

### Step 3. Specify matrix dimensions

`PUT /api/v1/projects/{projectId}/matrix-specification`: Save/update matrix dimensions. Generate matrix based on selected patents/papers and specified dimensions.

### Step 4. Save project and export matrix

- Finalize (irreversible): `PATCH /api/v1/projects/{projectId}` with final status
- `GET /api/v1/projects/{projectId}/export`: Export the matrix as an Excel file.

### User progress management

`POST /api/v1/users/{userId}/progress/{projectId}`: Save user progress at a specific step in the project workflow.

`GET /api/v1/users/{userId}/progress/{projectId}`: Retrieve saved progress for a specific project.

### Matrix sharing and view settings management

`PUT /api/v1/projects/{projectId}/matrix-view-settings`: Save/update matrix view settings (blue/red ocean thresholds, etc.).

---

Guidelines for pagination, filtering, sorting

- Pagination params: page (1-based) & limit; alternatively use cursor-based pagination for large datasets (cursor & limit).
- Sorting param: sort=name,asc or sort=fieldName,desc. Support multiple sort values: sort=field1,asc&sort=field2,desc
- Filtering: Use query params for standard filterable attributes (cpc, publicationYear, inventor, journal, citations, etc.).
- Search queries: POST /search or POST /projects/{projectId}/searches with the query in request body. Include mode param (sync|async) for long queries.

Http semantics & idempotency

- POST creates; use 201 Created and return Location header for new resources.
- PUT is idempotent (replace resource) and may return 200/204.
- PATCH is allowed for partial updates - return 200/204.
- DELETE deletes resource and should return 204 No Content when successful.
- GET returns 200 with standard pagination headers and ETag. Use 304 Not Modified on conditional GETs.

Errors and response format

- Always return a consistent base response format, for example:
  {
  "status": "error", // or success
  "code": 400,
  "timestamp": "...",
  "apiVersion": "v1",
  "error": {
  "type": "VALIDATION_ERROR",
  "message": "Invalid parameter",
  "details": [{ "field": "query", "message": "required" }]
  }
  }
- Use proper HTTP status codes for common outcomes (200, 201, 202, 204, 400, 401, 403, 404, 409, 422, 500).

Security & headers

- Authorization: Bearer {token}
- Support rate-limiting headers: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset
- Support Cache-Control headers for cacheable responses (projects, matrices, patents, papers). For changeable lists (search results), use no-cache.

Versioning & discovery

- Keep a version prefix in all API endpoints: /api/v1
- Maintain an API discovery endpoint: GET /api/v1/ or return the OpenAPI/Swagger spec: GET /api/v1/openapi.json

Recommendations for large operations

- Prefer asynchronous job processing for heavy matrix generation and export; return 202 and a job URL for clients to poll.
- Provide a synchronous path for small/mid-sized runs with proper timeouts (e.g., 30-60 seconds) and fall back to asynchronous.

<!-- Notes & open questions

- Should the system support global searches (i.e., /searches) or always be project-scoped? Using project-scoped searches keeps traceability and makes saving workflows consistent.
- Should we keep a GET endpoint for matrix cell details separate (GET /projects/{projectId}/matrices/{matrixId}/cells/{cellId}) when the frontend already stores the dataset? It's useful for rehydration, auditing, and offline review.
- How should permissions be modeled for shared projects? A simple approach: shares resource with permission level view/edit; an alternative: support RBAC with groups. -->
