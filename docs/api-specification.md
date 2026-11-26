# API Specification
## Multi-Model AI Inference and Serving Platform

## 1. Introduction

This document defines the REST API contracts for the Multi-Model AI Inference and Serving Platform.

The API layer will be implemented using FastAPI.

All endpoints will follow consistent request validation, response structures, authentication, authorization, and error-handling conventions.

## 2. API Standards

### 2.1 Base URL

Local development:

`http://localhost:8000`

API base path:

`/api/v1`

### 2.2 Data Format

- Request format: JSON unless file upload is required.
- Response format: JSON.
- File uploads: Multipart/form-data.
- Character encoding: UTF-8.

### 2.3 Authentication

Protected endpoints will use Bearer Token authentication.

Example:

```http
Authorization: Bearer <access_token>
```

The authenticated identity determines the tenant context. Clients must not be allowed to select another tenant by supplying a different tenant ID in the request body.

### 2.4 Standard Response Structure

Successful responses should follow a consistent structure where appropriate.

```json
{
  "success": true,
  "data": {},
  "message": "Request completed successfully"
}
```

Errors:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data"
  }
}
```

### 2.5 HTTP Status Codes

| Status | Meaning |
|---|---|
| 200 | Successful request |
| 201 | Resource created |
| 202 | Request accepted for background processing |
| 400 | Invalid request |
| 401 | Authentication required |
| 403 | Access forbidden |
| 404 | Resource not found |
| 409 | Resource conflict |
| 413 | File too large |
| 415 | Unsupported file type |
| 422 | Request validation failed |
| 429 | Rate limit exceeded |
| 500 | Internal server error |
| 503 | Service temporarily unavailable |

## 3. Authentication APIs

### API-001: Register Tenant

**Endpoint:** `POST /api/v1/auth/register`

**Authentication:** Public

**Request:**

```json
{
  "tenant_name": "BookStore Inc",
  "admin_name": "Admin User",
  "email": "admin@example.com",
  "password": "StrongPassword123"
}
```

**Response:** `201 Created`

```json
{
  "success": true,
  "data": {
    "tenant_id": "uuid",
    "user_id": "uuid",
    "tenant_name": "BookStore Inc"
  },
  "message": "Registration successful"
}
```

### API-002: Login

**Endpoint:** `POST /api/v1/auth/login`

**Authentication:** Public

**Request:**

```json
{
  "email": "admin@example.com",
  "password": "StrongPassword123"
}
```

**Response:** `200 OK`

```json
{
  "success": true,
  "data": {
    "access_token": "<token>",
    "token_type": "bearer",
    "expires_in": 3600
  },
  "message": "Login successful"
}
```

The token expiry and refresh-token strategy will be finalized during implementation.

## 4. Dataset APIs

### API-003: Upload Dataset

**Endpoint:** `POST /api/v1/datasets`

**Authentication:** Required

**Content-Type:** `multipart/form-data`

**Request:**
- `file`: CSV file
- `name`: Dataset name
- `description`: Optional description

**Response:** `201 Created`

```json
{
  "success": true,
  "data": {
    "dataset_id": "uuid",
    "name": "books.csv",
    "status": "UPLOADED",
    "file_format": "csv"
  },
  "message": "Dataset uploaded successfully"
}
```

### API-004: List Datasets

**Endpoint:** `GET /api/v1/datasets`

**Authentication:** Required

**Query Parameters:**
- `page`: Page number.
- `page_size`: Number of records.
- `status`: Optional dataset status.

**Response:** `200 OK`

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "dataset_id": "uuid",
        "name": "books.csv",
        "status": "VALID"
      }
    ],
    "page": 1,
    "page_size": 10,
    "total": 1
  }
}
```

### API-005: Get Dataset Details

**Endpoint:** `GET /api/v1/datasets/{dataset_id}`

**Authentication:** Required

**Response:** `200 OK`

```json
{
  "success": true,
  "data": {
    "dataset_id": "uuid",
    "name": "books.csv",
    "row_count": 10000,
    "column_count": 5,
    "status": "VALID"
  }
}
```

### API-006: Inspect Dataset

**Endpoint:** `GET /api/v1/datasets/{dataset_id}/preview`

**Authentication:** Required

**Query Parameters:**
- `limit`: Maximum number of preview records.

**Response:** `200 OK`

```json
{
  "success": true,
  "data": {
    "columns": [
      {
        "name": "user_id",
        "data_type": "integer"
      },
      {
        "name": "book_id",
        "data_type": "integer"
      },
      {
        "name": "rating",
        "data_type": "float"
      }
    ],
    "records": [
      {
        "user_id": 101,
        "book_id": 501,
        "rating": 5
      }
    ]
  }
}
```

### API-007: Validate Dataset

**Endpoint:** `POST /api/v1/datasets/{dataset_id}/validate`

**Authentication:** Required

**Request:**

```json
{
  "model_type": "recommendation"
}
```

**Response:** `200 OK`

```json
{
  "success": true,
  "data": {
    "dataset_id": "uuid",
    "valid": true,
    "total_rows": 10000,
    "issues": []
  }
}
```

## 5. Schema Configuration APIs

### API-008: Configure Schema Mapping

**Endpoint:** `POST /api/v1/datasets/{dataset_id}/schema`

**Authentication:** Required

**Request:**

```json
{
  "model_type": "recommendation",
  "mapping": {
    "user_id": "customer_id",
    "item_id": "book_code",
    "rating": "user_rating"
  }
}
```

**Response:** `201 Created`

```json
{
  "success": true,
  "data": {
    "schema_configuration_id": "uuid",
    "model_type": "recommendation",
    "status": "VALID"
  }
}
```

### API-009: Get Schema Configuration

**Endpoint:** `GET /api/v1/datasets/{dataset_id}/schema`

**Authentication:** Required

**Response:** `200 OK`

Returns the saved canonical field mappings for the requested dataset.

## 6. Model Discovery APIs

### API-010: List Supported Models

**Endpoint:** `GET /api/v1/models`

**Authentication:** Required

**Response:** `200 OK`

```json
{
  "success": true,
  "data": [
    {
      "model_type": "recommendation",
      "name": "Universal Recommendation Engine",
      "algorithms": [
        "popularity",
        "content_based",
        "collaborative_filtering",
        "matrix_factorization",
        "hybrid"
      ]
    }
  ]
}
```

The supported model catalogue may contain models available to all tenants. Tenant-specific trained model records will be exposed through separate authorized endpoints.

## 7. Training APIs

### API-011: Start Training

**Endpoint:** `POST /api/v1/training/jobs`

**Authentication:** Required

**Request:**

```json
{
  "dataset_version_id": "uuid",
  "schema_configuration_id": "uuid",
  "model_type": "recommendation",
  "algorithm": "collaborative_filtering",
  "parameters": {
    "n_factors": 50,
    "epochs": 20
  }
}
```

**Response:** `202 Accepted`

```json
{
  "success": true,
  "data": {
    "job_id": "uuid",
    "status": "PENDING"
  },
  "message": "Training job accepted"
}
```

Training should be handled asynchronously once background job processing is implemented.

### API-012: Get Training Status

**Endpoint:** `GET /api/v1/training/jobs/{job_id}`

**Authentication:** Required

**Response:** `200 OK`

```json
{
  "success": true,
  "data": {
    "job_id": "uuid",
    "status": "COMPLETED",
    "model_id": "uuid",
    "model_version_id": "uuid"
  }
}
```

### API-013: List Training Jobs

**Endpoint:** `GET /api/v1/training/jobs`

**Authentication:** Required

**Query Parameters:**
- `status`
- `page`
- `page_size`

Returns training jobs belonging to the authenticated tenant.

## 8. Model Registry APIs

### API-014: List Registered Models

**Endpoint:** `GET /api/v1/model-registry/models`

**Authentication:** Required

Returns registered models belonging to the authenticated tenant.

### API-015: Get Model Details

**Endpoint:** `GET /api/v1/model-registry/models/{model_id}`

**Authentication:** Required

Returns model metadata and available versions.

### API-016: List Model Versions

**Endpoint:** `GET /api/v1/model-registry/models/{model_id}/versions`

**Authentication:** Required

Returns the registered versions of the specified model.

### API-017: Activate Model Version

**Endpoint:** `POST /api/v1/model-registry/models/{model_id}/versions/{version_id}/activate`

**Authentication:** Required

Activates an eligible model version after verifying tenant ownership and model readiness.

### API-018: Deactivate Model Version

**Endpoint:** `POST /api/v1/model-registry/models/{model_id}/versions/{version_id}/deactivate`

**Authentication:** Required

Deactivates the specified model version.

## 9. Inference APIs

### API-019: Generate Recommendations

**Endpoint:** `POST /api/v1/predictions/recommendations`

**Authentication:** Required

**Request:**

```json
{
  "model_id": "uuid",
  "user_id": "user_101",
  "top_k": 5
}
```

**Response:** `200 OK`

```json
{
  "success": true,
  "data": {
    "model_id": "uuid",
    "model_version": 1,
    "recommendations": [
      {
        "item_id": "book_501",
        "score": 0.95
      },
      {
        "item_id": "book_502",
        "score": 0.91
      }
    ]
  }
}
```

### API-020: Get Prediction History

**Endpoint:** `GET /api/v1/predictions/history`

**Authentication:** Required

**Query Parameters:**
- `page`
- `page_size`
- `model_id`
- `status`

Returns authorized inference metadata. Sensitive request and response payloads are not exposed by default.

## 10. Health APIs

### API-021: Application Health

**Endpoint:** `GET /health`

**Authentication:** Public or restricted according to deployment configuration.

**Response:** `200 OK`

```json
{
  "status": "healthy"
}
```

### API-022: Readiness Check

**Endpoint:** `GET /ready`

Checks whether required application dependencies are ready to serve requests.

## 11. Standard Validation Rules

The API must enforce:

- Valid UUID formats for resource identifiers.
- Supported dataset formats.
- File size limits.
- Required request fields.
- Supported model types.
- Valid algorithm names.
- Valid algorithm parameters.
- Positive values for `top_k`.
- Maximum allowed values for pagination and recommendation counts.
- Tenant ownership for every tenant-specific resource.

## 12. API Security Requirements

1. Protected endpoints require authentication.
2. Authorization must be checked before accessing tenant resources.
3. Tenant identity must come from trusted authentication context.
4. Client-supplied tenant IDs must not override authenticated identity.
5. File uploads must be validated before storage.
6. Error responses must not reveal internal stack traces or secrets.
7. Rate limits should be introduced for public and resource-intensive endpoints.

## 13. API Versioning Strategy

The API will use URL-based versioning.

Example:

`/api/v1/datasets`

Breaking API changes will be introduced through a new API version when necessary.

## 14. Documentation and Testing

FastAPI will generate OpenAPI documentation.

The API will be tested using:
- Pytest.
- FastAPI TestClient or HTTPX.
- Swagger UI.
- Integration tests.

Every implemented endpoint should have:
- Request validation tests.
- Successful response tests.
- Error response tests.
- Authorization tests.
- Tenant isolation tests where applicable.

---

**Project:** Multi-Model AI Inference and Serving Platform

**Document:** API Specification

**Status:** Initial Draft