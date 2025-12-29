# TEMatrix Backend API Specification

**Created**: Nov 03, 2025  
**Last Updated**: Nov 18, 2025

---

**Design Rationale**: For now, frontend will handle the following to minimize load on backend:

- Filtering and sorting of patents and papers in Step 2 (Select patents and papers)
- Customizing blue/red ocean thresholds in Step 3 (View and customize matrix)
- Tapping on individual cells to view patents and papers in that cell in Step 3 (View and customize matrix)

> **Tip:** According to REST best pracitices, PUT is used to replace an entire resource or create it if it doesn't exist, while PATCH is for partially updating a resource. POST is used to submit data to create a new resource or add to an existing one. _Souce: [GeeksForGeeks](https://www.geeksforgeeks.org/java/what-is-the-difference-between-put-post-and-patch-in-restful-api/)_.

We use `PATCH` for updating selections, matrix specifications, and matrix finalization because these operations involve partial updates (e.g., saving progress with flags like `selectionConfirmed` or `specificationConfirmed`, or updating specific fields like view settings). `PATCH` allows flexible, partial changes without requiring the full resource replacement that `PUT` demands. For first-time creation of these sub-resources, `PATCH` handles it as an upsert (create if not exists, update otherwise), avoiding unnecessary `POST` endpoints and keeping the API simple and project-centric.

---

## Entity-Relationship Diagram (ERD)

<!-- ```
+----------------+     +-----------------+
|     User       |     |    Session      |
| (1)            |     | (1:1 with User) |
| - userId       |     | - sessionToken  |
| - username     |     | - expiresIn     |
+----------------+     +-----------------+
         | (1:N)
         v
+----------------+     +-----------------+
|   Project      |     |   Selections    |
| (1)            |     | (1:1 with Proj) |
| - projectId    |     | - patentIds[]   |
| - projectTitle |     | - paperIds[]    |
| - status       |     | - filters       |
| - createdAt    |     | - sorts         |
+----------------+     +-----------------+
         | (1:1)
         v
+----------------+     +-----------------+
| Matrix-Spec    |     |     Matrix      |
| (1:1 with Proj)|     | (1:1 with Proj) |
| - functionLabels|     | - cells[]       |
| - techLabels    |     | - createdAt     |
+----------------+     +-----------------+
         | (1:1)
         v
+----------------+     +-----------------+
|   View-Settings|     |     Progress    |
| (1:1 with Proj)|     | (1:1 with Proj) |
| - blueThreshold|     | - currentStep   |
| - redThreshold |     | - selections    |
|                |     | - specifications|
|                |     | - matrix        |
+----------------+     +-----------------+
         | (1:N)
         v
+----------------+     +-----------------+
|    Exports     |     |     Shares      |
| (N:1 with Proj)|     | (N:1 with Proj) |
+----------------+     +-----------------+

External Resources (Read-Only):
+----------------+     +-----------------+
|    Patents     |     |     Papers      |
| (Global)       |     | (Global)        |
| - id           |     | - id            |
| - title        |     | - title         |
| - inventor     |     | - author        |
| - ...          |     | - ...           |
+----------------+     +-----------------+
```

**Key Relationships:** -->

- **User (1) → (N) Project**: One user owns many projects.
- **Project (1) → (1) Selections**: Each project has one set of selections (patents/papers).
- **Project (1) → (1) Matrix-Spec**: Each project has one matrix specification.
- **Project (1) → (1) Matrix**: Each project has one matrix (composed of cells).
- **Matrix (1) → (M) Cells**: A matrix contains many cells (each cell links to patents/papers).
- **Project (1) → (N) View-Settings**: Each project has many sets of view settings (saved by different users).
- **Project (1) → (1) Progress**: Each project has one progress snapshot.
- **Project (1) → (N) Exports**: A project can have multiple exports.
- **Project (1) → (N) Shares**: A project can be shared with multiple users.
- **Patents/Papers**: Global, read-only collections; referenced by selections and cells but not owned by projects.

---

## Resources & Endpoints

Base URL: `/api/v1`

In line with REST best practices, the API is organized around key resources that map to user flows in the app.

### Resources:

- `users`: Currently only for authentication; a user has `username` and `password`.
- `sessions`: For managing user sessions (login/logout).
- `projects`: Each project represents a workflow – `Build query` -> `Select patents and papers` -> `Specify matrix dimensions` -> `Generate and finalize matrix`, etc.

### Endpoints:

**Authentication:**

- `POST /sessions`: User login, returns session token.
- `DELETE /sessions`: User logout, invalidates session token.

**Project Workflow:**

Step 0. Build query

- `GET /query-builder/keyword-suggestions?seedKeyword="<seedKeyword>"`: Get AI keyword suggestions based on seed keyword.
  > Q: Why `GET` with a query parameter `seedKeyword`?
  > A: Because:
  >
  > - `GET` is safe/idempotent and supports caching/bookmarking.
  > - Query params are discoverable and follow typical search endpoints.
- `POST /projects`: Executing a search is equivalent to creating a new project.

Step 1. Select patents and papers

- `PATCH /projects/{projectId}/selections`: Save selected patents and papers for the next step.

<!-- - `PATCH /projects/{projectId}`: Confirm selections (advance project state and optionally trigger matrix spec suggestions).
  - Request body includes ids of selected patents and papers.
  - Response includes AI suggestions for matrix dimensions. -->

Step 2. Specify matrix dimensions

- `PATCH /projects/{projectId}/matrix-specification`: Save/update matrix dimensions. Generate matrix based on selected patents/papers and specified dimensions.

Step 3. Save/finalize project and export matrix

- `PATCH /projects/{projectId}/matrix`: Finalize (irreversible). Matrix can be shared only after finalization. Shared matrix can be viewed by anyone using the project link.

- `GET /projects/{projectId}/export`: Export the matrix as an Excel file.

**User progress management:**

<!-- - `POST /users/{userId}/progress/{projectId}`: Save user progress at a specific step in the project workflow. -->

- `GET /users/me/progress/{projectId}`: Retrieve saved progress for a specific project.

**Matrix sharing and view settings management:**

- `GET /projects/{projectId}/share`: Share the matrix with others via a link. Anyone with the link can view the matrix.
- `GET /projects/{projectId}`: View (shared) matrix. Non-finalized projects cannot be shared/viewed.
- `PUT /projects/{projectId}/view-settings`: Anyone can have their own settings (blue/red ocean threshold). By default, use the creator's settings. Creator's action is treated the same as others, but they can change the default settings for new viewers.
  > Note: Here, we decided to use `PUT` instead of `PATCH` or `POST` to emphasize that the view settings are to be replaced as an entire resorce or created if they don't exist.

**Projects/searches management:**

<!-- - `GET /users/{userId}/projects`: Get list of user's projects (searches). -->

- `GET /users/me/projects`: Get list of current logged-in user's projects (searches).
- `PATCH /users/me/projects/{projectId}`: Rename a project.
- `DELETE /users/me/projects/{projectId}`: Delete a project.

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

## API Requests & Responses

### 1. Authentication

**`POST /sessions`: User login.**

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

**`DELETE /sessions`: User logout.**

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

### 2. Project Workflow

#### Step 0: Build Query

**`GET /query-builder/keyword-suggestions?seedKeyword="<seedKeyword>"`: Get AI keyword suggestions based on seed keyword.**

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

**`POST /projects`: Execute search and create new project.**

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

#### Step 1: Select Patents and Papers

**`PATCH /projects/{projectId}/selections` with `selectionConfirmed: false`: Save selected patents and papers.**

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

**`PATCH /projects/{projectId}/selections` with `selectionConfirmed: true`: Confirm selected patents and papers, and move to the next step.**

> This action is synchronous for our use case. For bigger datasets, we consider making this asynchronous with a job resource.

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

#### Step 2: Specify Matrix Dimensions

**`PATCH /projects/{projectId}/matrix-specification` with `specificationConfirmed: false`: Save/update matrix dimensions.**

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

**`PATCH /projects/{projectId}/matrix-specification` with `specificationConfirmed: true`: Confirm matrix dimensions and generate matrix.**

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

#### Step 3: Save/Finalize Project and Export Matrix

**`PATCH /projects/{projectId}/matrix` to finalize project.**

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

**`GET /projects/{projectId}/export`: Export the matrix as an Excel file.**

Request:

```json
{
  // No body needed, just projectId in URL
}
```

Response:

- Binary Excel file with `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

### 3. User Porgress Management

**`GET /users/me/progress/{projectId}`: Retrieve user progress for a specific project.**

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
        "currentStep": "selection" | "specification" | "matrix",
        "selectionBaseData": {
        // ---
        // same as in response from `POST /projects`
            "patents": [ ... ],
            "papers": [ ... ],
            "filterOptions": { ... },
            "sortOptions": { ... },
        // ---
        },
        "specificationBaseData": {
            "functionLabels": string[],
            "technologyLabels": string[]
        } | null, // null if "currentStep" is "selection"
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
        } | null, // null if "currentStep" is "selection" or "specification"
        "userProgress": {
            "selection": {
                // Similar to "Step 1. Select Patents and "Papers"
                    // `PATCH /projects/{projectId}/selections` with `selectionConfirmed: false` request body
                "selectedPatentIds": string[], // empty if user hasn't made any selection yet
                "selectedPaperIds": string[], // empty if user hasn't made any selection yet
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
        },
    }
}
```

### 4. Matrix Sharing & Viewing

**`GET /projects/{projectId}/share`: Share the matrix.**

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

**`GET /projects/{projectId}`: View (shared) matrix.**

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

**`PUT /projects/{projectId}/view-settings`: Save/update matrix view settings (blue/red ocean thresholds, etc.).**

Request:

> Note: Backend identify the user from session token in Auth header.

```json
{
    "blueOceanThreshold": number,
    "redOceanThreshold": number
}
```

### 5. Projects/Searches Management

On the home page, users can see their list of projects (searches), rename projects, and delete projects.

**`GET /users/me/projects`: Get list of user's projects.**

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

**`PATCH /users/me/projects/{projectId}`: Rename a project.**

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

**`DELETE /users/me/projects/{projectId}`: Delete a project.**

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

<!-- ---

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
``` -->
