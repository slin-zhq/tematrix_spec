# TEMatrix API 範例

**版本**: 1.1
**日期**: 2025 年 11 月 4 日

本文件提供 TEMatrix 後端 API 規範中所有 API 端點的完整 JSON 範例。

---

## 認證

### POST /auth/login

**請求:**

```json
{
  "username": "john.doe@example.com",
  "password": "securePassword123"
}
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T10:30:00Z",
  "apiVersion": "v1.0",
  "data": {
    "sessionToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJ1c2VyXzEyMyIsInVzZXJuYW1lIjoiam9obi5kb2VAZXhhbXBsZS5jb20iLCJpYXQiOjE2MzYwMjYwMDAsImV4cCI6MTYzNjAzMzIwMH0.signature",
    "userId": "user_123",
    "username": "john.doe@example.com",
    "expiresIn": 86400
  }
}
```

**回應 (401 未授權):**

```json
{
  "status": "error",
  "code": 401,
  "timestamp": "2025-11-04T10:30:00Z",
  "apiVersion": "v1.0",
  "error": {
    "type": "AUTHENTICATION_ERROR",
    "message": "Invalid username or password"
  }
}
```

### POST /auth/logout

**請求標頭:**

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**請求主體:**

```json
{}
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T11:00:00Z",
  "apiVersion": "v1.0",
  "data": {
    "message": "Logged out successfully"
  }
}
```

---

## 步驟 0: 查詢建構器

### GET /query-builder/keyword-suggestions

**請求:**

```
GET /query-builder/keyword-suggestions?seed_keyword=GaN
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T10:35:00Z",
  "apiVersion": "v1.0",
  "data": {
    "keyword": "GaN",
    "suggestions": [
      "Gallium Nitride",
      "GaN semiconductor",
      "Gallium mononitride",
      "Wide bandgap semiconductor",
      "Wurtzite gallium nitride",
      "GaN HEMT",
      "GaN transistor",
      "GaN power device",
      "Hexagonal gallium nitride",
      "Nitridogallium"
    ]
  }
}
```

---

## 步驟 1: 專利與論文選取

### POST /search/execute

**請求:**

```json
{
  "caseName": "GaN Technology Analysis 2025",
  "query": "(GaN OR \"gallium nitride\") AND semiconductor",
  "yearRange": {
    "min": 2020,
    "max": 2025
  }
}
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T10:40:00Z",
  "apiVersion": "v1.0",
  "data": {
    "caseId": "case_20251104_001",
    "caseName": "GaN Technology Analysis 2025",
    "query": "(GaN OR \"gallium nitride\") AND semiconductor",
    "patents": [
      {
        "id": "CN119852846A",
        "title": "一種氮化鎵晶體管及其製備方法",
        "inventor": "劉志宏、羅詩文",
        "patentHolder": "Xidian University",
        "filedYear": 2024,
        "priorityYear": 2024,
        "publicationYear": 2025,
        "grantYear": null,
        "cpc": ["H01L29/778", "H01L33/32"],
        "keywords": ["氮化鎵", "晶體管", "半導體"],
        "concepts": "氮化鎵晶體管, 寬禁帶半導體",
        "link": "https://patents.google.com/patent/CN119852846A/zh"
      },
      {
        "id": "US10991751B2",
        "title": "Gallium nitride transistor with improved thermal management",
        "inventor": "Jane Smith, Robert Johnson",
        "patentHolder": "TechCorp Inc.",
        "filedYear": 2022,
        "priorityYear": 2022,
        "publicationYear": 2024,
        "grantYear": 2024,
        "cpc": ["H01L29/778", "H01L23/36"],
        "keywords": ["GaN", "transistor", "thermal management"],
        "concepts": "Gallium nitride transistor, Thermal dissipation",
        "link": "https://patents.google.com/patent/US10991751B2/en"
      }
    ],
    "papers": [
      {
        "id": "PAPER-001",
        "title": "Performance investigation of silicon and gallium nitride transistors for power applications",
        "author": "Renan R. Duarte, Guilherme F. Ferreira",
        "journal": "IEEE Transactions on Industry Applications",
        "publicationYear": 2023,
        "citations": 15,
        "cpc": ["H02M3/335", "H05B45/10"],
        "concepts": "Silicon transistors, Gallium nitride transistors, Power electronics",
        "link": "https://ieeexplore.ieee.org/abstract/document/10012345"
      }
    ],
    "filterOptions": {
      "patentFilterOptions": {
        "cpc": {
          "type": "multiselect",
          "options": [
            {
              "value": "H01L29/778",
              "count": 45,
              "active": true
            },
            {
              "value": "H01L33/32",
              "count": 32,
              "active": true
            }
          ]
        },
        "patentHolder": {
          "type": "multiselect",
          "options": [
            {
              "value": "Xidian University",
              "count": 18,
              "active": true
            },
            {
              "value": "TechCorp Inc.",
              "count": 12,
              "active": true
            }
          ]
        },
        "publicationYear": {
          "type": "range",
          "min": 2020,
          "max": 2025,
          "selectedMin": 2020,
          "selectedMax": 2025
        }
      },
      "paperFilterOptions": {
        "journal": {
          "type": "multiselect",
          "options": [
            {
              "value": "IEEE Transactions on Industry Applications",
              "count": 8,
              "active": true
            },
            {
              "value": "Applied Physics Letters",
              "count": 5,
              "active": true
            }
          ]
        },
        "publicationYear": {
          "type": "range",
          "min": 2020,
          "max": 2025,
          "selectedMin": 2020,
          "selectedMax": 2025
        }
      }
    },
    "sortingOptions": ["Most Recent", "Most Cited", "Relevance"],
    "executedAt": "2025-11-04T10:40:00Z"
  }
}
```

---

## 步驟 2: 矩陣維度規格

### POST /matrix/initialize-specification

**請求:**

```json
{
  "caseId": "case_20251104_001",
  "selectedPatentIds": ["CN119852846A", "US10991751B2"],
  "selectedPaperIds": ["PAPER-001"]
}
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T10:45:00Z",
  "apiVersion": "v1.0",
  "data": {
    "caseId": "case_20251104_001",
    "caseName": "GaN Technology Analysis 2025",
    "query": "(GaN OR \"gallium nitride\") AND semiconductor",
    "efficacySuggestions": [
      "High Efficiency",
      "Low Power Consumption",
      "High Brightness",
      "Long Lifespan",
      "High Reliability"
    ],
    "technologySuggestions": [
      "Micro-LED",
      "GaN-on-Si",
      "VCSEL",
      "Driver IC",
      "Semiconductor Process"
    ],
    "generatedAtTimestamp": "2025-11-04T10:45:00Z"
  }
}
```

---

## 步驟 3: 矩陣產生

### POST /matrix/generate

**請求:**

```json
{
  "caseId": "case_20251104_001",
  "efficacyLabels": [
    "High Efficiency",
    "Low Power Consumption",
    "High Brightness",
    "Long Lifespan",
    "High Reliability"
  ],
  "technologyLabels": [
    "Micro-LED",
    "GaN-on-Si",
    "VCSEL",
    "Driver IC",
    "Semiconductor Process"
  ]
}
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T11:00:00Z",
  "apiVersion": "v1.0",
  "data": {
    "caseId": "case_20251104_001",
    "efficacyLabels": [
      "High Efficiency",
      "Low Power Consumption",
      "High Brightness",
      "Long Lifespan",
      "High Reliability"
    ],
    "technologyLabels": [
      "Micro-LED",
      "GaN-on-Si",
      "VCSEL",
      "Driver IC",
      "Semiconductor Process"
    ],
    "cells": [
      {
        "efficacyIndex": 0,
        "technologyIndex": 0,
        "patentCount": 2,
        "paperCount": 1,
        "patentIds": ["CN119852846A", "US10991751B2"],
        "paperIds": ["PAPER-001"]
      },
      {
        "efficacyIndex": 0,
        "technologyIndex": 1,
        "patentCount": 1,
        "paperCount": 0,
        "patentIds": ["US10991751B2"],
        "paperIds": []
      },
      {
        "efficacyIndex": 1,
        "technologyIndex": 0,
        "patentCount": 0,
        "paperCount": 1,
        "patentIds": [],
        "paperIds": ["PAPER-001"]
      }
    ],
    "generatedAt": "2025-11-04T11:00:00Z"
  }
}
```

### GET /matrix/{caseId}/cell/{efficacyIndex}/{technologyIndex}

**請求:**

```
GET /matrix/case_20251104_001/cell/0/0
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T11:05:00Z",
  "apiVersion": "v1.0",
  "data": {
    "cell": {
      "efficacyLabel": "High Efficiency",
      "technologyLabel": "Micro-LED",
      "patents": [
        {
          "id": "CN119852846A",
          "title": "一種氮化鎵晶體管及其製備方法",
          "inventor": "劉志宏、羅詩文",
          "patentHolder": "Xidian University",
          "filedYear": 2024,
          "priorityYear": 2024,
          "publicationYear": 2025,
          "grantYear": null,
          "cpc": ["H01L29/778", "H01L33/32"],
          "keywords": ["氮化鎵", "晶體管", "半導體"],
          "concepts": "氮化鎵晶體管, 寬禁帶半導體",
          "link": "https://patents.google.com/patent/CN119852846A/zh"
        },
        {
          "id": "US10991751B2",
          "title": "Gallium nitride transistor with improved thermal management",
          "inventor": "Jane Smith, Robert Johnson",
          "patentHolder": "TechCorp Inc.",
          "filedYear": 2022,
          "priorityYear": 2022,
          "publicationYear": 2024,
          "grantYear": 2024,
          "cpc": ["H01L29/778", "H01L23/36"],
          "keywords": ["GaN", "transistor", "thermal management"],
          "concepts": "Gallium nitride transistor, Thermal dissipation",
          "link": "https://patents.google.com/patent/US10991751B2/en"
        }
      ],
      "papers": [
        {
          "id": "PAPER-001",
          "title": "Performance investigation of silicon and gallium nitride transistors for power applications",
          "author": "Renan R. Duarte, Guilherme F. Ferreira",
          "journal": "IEEE Transactions on Industry Applications",
          "publicationYear": 2023,
          "citations": 15,
          "cpc": ["H02M3/335", "H05B45/10"],
          "concepts": "Silicon transistors, Gallium nitride transistors, Power electronics",
          "link": "https://ieeexplore.ieee.org/abstract/document/10012345"
        }
      ],
      "patentCount": 2,
      "paperCount": 1
    }
  }
}
```

### POST /matrix/view-settings

**請求:**

```json
{
  "caseId": "case_20251104_001",
  "settings": {
    "blue_ocean_threshold": 0.3,
    "red_ocean_threshold": 0.7
  }
}
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T11:10:00Z",
  "apiVersion": "v1.0",
  "data": {}
}
```

---

## 步驟 4: 矩陣匯出與檢視

### POST /matrix/{caseId}/export

**請求:**

```json
{
  "caseId": "case_20251104_001"
}
```

**回應:** Content-Type 為 `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` 的二進位 Excel 檔案

---

## 進度與工作階段管理

### POST /user/progress

**請求:**

```json
{
  "caseId": "case_20251104_001",
  "stepCode": "MATRIX_SPECIFICATION",
  "progress": {
    "matrixSpecification": {
      "selectedEfficacyLabels": [
        "High Efficiency",
        "Low Power Consumption",
        "High Brightness",
        "Long Lifespan",
        "High Reliability"
      ],
      "selectedTechnologyLabels": [
        "Micro-LED",
        "GaN-on-Si",
        "VCSEL",
        "Driver IC",
        "Semiconductor Process"
      ]
    }
  }
}
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T11:15:00Z",
  "apiVersion": "v1.0",
  "data": {
    "caseId": "case_20251104_001",
    "stepCode": "MATRIX_SPECIFICATION",
    "savedAt": "2025-11-04T11:15:00Z"
  }
}
```

### GET /user/progress/{caseId}

**請求:**

```
GET /user/progress/case_20251104_001
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T11:20:00Z",
  "apiVersion": "v1.0",
  "data": {
    "caseId": "case_20251104_001",
    "caseName": "GaN Technology Analysis 2025",
    "currentStepCode": "MATRIX_SPECIFICATION",
    "progress": {
      "patentsPapersSelection": {
        "selectedPatentIds": ["CN119852846A", "US10991751B2"],
        "selectedPaperIds": ["PAPER-001"],
        "currentTabIndex": 0,
        "filters": [],
        "sorting": [
          {
            "field": "Most Recent",
            "order": "DESC",
            "active": true
          }
        ]
      },
      "matrixSpecification": {
        "selectedEfficacyLabels": [
          "High Efficiency",
          "Low Power Consumption",
          "High Brightness",
          "Long Lifespan",
          "High Reliability"
        ],
        "selectedTechnologyLabels": [
          "Micro-LED",
          "GaN-on-Si",
          "VCSEL",
          "Driver IC",
          "Semiconductor Process"
        ]
      }
    },
    "lastModifiedAt": "2025-11-04T11:15:00Z"
  }
}
```

---

## 案例管理

### GET /user/cases/{userId}

**請求:**

```
GET /user/cases/user_123?page=1&pageSize=10
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T11:25:00Z",
  "apiVersion": "v1.0",
  "data": {
    "cases": [
      {
        "caseId": "case_20251104_001",
        "caseName": "GaN Technology Analysis 2025",
        "createdAt": "2025-11-04T10:30:00Z",
        "lastModifiedAt": "2025-11-04T11:15:00Z",
        "status": "in_progress"
      },
      {
        "caseId": "case_20251103_005",
        "caseName": "Silicon Carbide Research 2024",
        "createdAt": "2025-11-03T14:20:00Z",
        "lastModifiedAt": "2025-11-03T16:45:00Z",
        "status": "completed"
      }
    ]
  }
}
```

### POST /user/cases/{caseId}/rename

**請求:**

```json
{
  "newName": "Advanced GaN Technology Analysis 2025"
}
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T11:30:00Z",
  "apiVersion": "v1.0",
  "data": {
    "caseId": "case_20251104_001",
    "caseName": "Advanced GaN Technology Analysis 2025"
  }
}
```

### DELETE /user/cases/{caseId}

**請求:**

```
DELETE /user/cases/case_20251103_005
```

**回應 (200 OK):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "2025-11-04T11:35:00Z",
  "apiVersion": "v1.0",
  "data": {
    "message": "Case deleted successfully"
  }
}
```

---

## 錯誤範例

### 400 錯誤請求 - 驗證錯誤

```json
{
  "status": "error",
  "code": 400,
  "timestamp": "2025-11-04T11:40:00Z",
  "apiVersion": "v1.0",
  "error": {
    "type": "VALIDATION_ERROR",
    "message": "Invalid matrix specification",
    "details": [
      {
        "field": "efficacyLabels",
        "message": "At least one efficacy label is required"
      }
    ]
  }
}
```

### 404 找不到資源 - 資源不存在

```json
{
  "status": "error",
  "code": 404,
  "timestamp": "2025-11-04T11:45:00Z",
  "apiVersion": "v1.0",
  "error": {
    "type": "NOT_FOUND",
    "message": "Case not found: case_20251104_999"
  }
}
```

### 408 請求逾時 - AI 產生逾時

```json
{
  "status": "error",
  "code": 408,
  "timestamp": "2025-11-04T11:50:00Z",
  "apiVersion": "v1.0",
  "error": {
    "type": "TIMEOUT_ERROR",
    "message": "AI suggestion generation timed out. Please try again."
  }
}
```

### 429 請求過多 - 速率限制

```json
{
  "status": "error",
  "code": 429,
  "timestamp": "2025-11-04T11:55:00Z",
  "apiVersion": "v1.0",
  "error": {
    "type": "RATE_LIMIT_ERROR",
    "message": "Too many requests. Please try again later."
  }
}
```

---

## 附註

- 所有時間戳記均採用 ISO 8601 格式 (UTC)
- 所有已認證的請求都需要 `Authorization: Bearer <token>` 標頭
- 檔案上傳端點會以適當的 Content-Type 回傳二進位資料
- 分頁功能適用於相關端點
- 錯誤回應遵循與成功回應相同的結構
- 當欄位為 null/空值時，可選欄位會被省略
