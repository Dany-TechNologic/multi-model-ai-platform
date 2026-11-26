# Low-Level Design (LLD)
## Multi-Model AI Inference and Serving Platform

## 1. Introduction

### 1.1 Purpose

This document defines the internal design of the Multi-Model AI Inference and Serving Platform.

It describes:
- Python project structure.
- Module responsibilities.
- Class and interface design.
- Database entities.
- API contracts.
- Model training and inference workflows.
- Error handling.
- Testing strategy.

### 1.2 Initial Implementation Scope

The initial implementation will focus on:

1. Client and tenant management.
2. Dataset upload and validation.
3. Canonical schema mapping.
4. Universal Recommendation Engine.
5. Model training and evaluation.
6. Model registration and versioning.
7. Recommendation inference APIs.

Other ML models will be integrated in later phases.

## 2. Python Project Structure

The project will use a modular structure.

```text
multi-model-ai-platform/
│
├── app/
│   ├── main.py
│   │
│   ├── api/
│   │   ├── routes/
│   │   │   ├── auth.py
│   │   │   ├── datasets.py
│   │   │   ├── models.py
│   │   │   ├── training.py
│   │   │   └── predictions.py
│   │   │
│   │   └── dependencies.py
│   │
│   ├── core/
│   │   ├── config.py
│   │   ├── security.py
│   │   ├── exceptions.py
│   │   └── logging.py
│   │
│   ├── clients/
│   │   ├── models.py
│   │   ├── schemas.py
│   │   └── service.py
│   │
│   ├── datasets/
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── service.py
│   │   ├── validator.py
│   │   └── storage.py
│   │
│   ├── training/
│   │   ├── schemas.py
│   │   ├── service.py
│   │   ├── pipeline.py
│   │   └── jobs.py
│   │
│   ├── model_registry/
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── service.py
│   │   └── repository.py
│   │
│   ├── inference/
│   │   ├── schemas.py
│   │   ├── service.py
│   │   ├── router.py
│   │   └── model_loader.py
│   │
│   ├── ml_models/
│   │   ├── base.py
│   │   │
│   │   └── recommendation/
│   │       ├── popularity.py
│   │       ├── content_based.py
│   │       ├── collaborative.py
│   │       ├── matrix_factorization.py
│   │       ├── hybrid.py
│   │       └── evaluation.py
│   │
│   ├── database/
│   │   ├── connection.py
│   │   ├── base.py
│   │   └── session.py
│   │
│   └── utils/
│       ├── file_validation.py
│       └── helpers.py
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── api/
│
├── docs/
│
├── scripts/
│
├── storage/
│   ├── datasets/
│   └── models/
│
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

This is the target structure. We will create directories as their corresponding modules are implemented rather than building everything upfront.

## 3. Module Responsibilities

### 3.1 API Layer

**Location:** `app/api/`

Responsible for:
- Receiving HTTP requests.
- Validating request schemas.
- Calling application services.
- Returning standardized responses.
- Handling API-level exceptions.

The API layer should not contain machine learning algorithm implementations or complex business logic.

### 3.2 Core Module

**Location:** `app/core/`

Responsible for:
- Application configuration.
- Authentication utilities.
- Security policies.
- Logging configuration.
- Common exceptions.

### 3.3 Dataset Module

**Location:** `app/datasets/`

Responsible for:
- Uploading datasets.
- Inspecting CSV files.
- Validating data quality.
- Mapping columns.
- Managing dataset metadata.
- Storing dataset files.

### 3.4 Training Module

**Location:** `app/training/`

Responsible for:
- Creating training jobs.
- Preparing training data.
- Selecting algorithms.
- Executing training workflows.
- Recording training results.

### 3.5 ML Models Module

**Location:** `app/ml_models/`

Responsible for implementing reusable machine learning algorithms.

Each model implementation should expose a consistent interface wherever the algorithm supports the same operations.

### 3.6 Model Registry Module

**Location:** `app/model_registry/`

Responsible for:
- Registering trained models.
- Tracking model versions.
- Maintaining model metadata.
- Managing model lifecycle states.
- Resolving active model versions.

### 3.7 Inference Module

**Location:** `app/inference/`

Responsible for:
- Receiving prediction requests from the API layer.
- Resolving model identifiers and versions.
- Loading registered model artifacts.
- Executing predictions.
- Formatting responses.

## 4. Model Interface Design

To support multiple ML models, we will define a common model interface.

### 4.1 Base Model Interface

**File:** `app/ml_models/base.py`

```python
from abc import ABC, abstractmethod


class BaseModel(ABC):

    @abstractmethod
    def train(self, data):
        pass

    @abstractmethod
    def predict(self, data):
        pass

    @abstractmethod
    def evaluate(self, data):
        pass
```

Each model implementation will follow this contract where appropriate.

Some algorithms, particularly recommenders, may need additional methods such as `recommend(user_id, top_k)`.

### 4.2 Recommendation Model Implementations

| Class | Responsibility |
|---|---|
| PopularityRecommender | Recommend popular items |
| ContentBasedRecommender | Recommend similar items |
| CollaborativeRecommender | Learn from user-item interactions |
| MatrixFactorizationRecommender | Learn latent user and item representations |
| HybridRecommender | Combine multiple recommendation strategies |

Each algorithm will be implemented and tested independently.

## 5. Database Design

The initial relational database will use PostgreSQL.

### 5.1 Client Table

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| name | VARCHAR | Client name |
| email | VARCHAR | Client email |
| password_hash | VARCHAR | Hashed password |
| created_at | TIMESTAMP | Creation time |

### 5.2 Dataset Table

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| tenant_id | UUID | Client reference |
| name | VARCHAR | Dataset name |
| file_path | TEXT | Storage location |
| file_format | VARCHAR | Dataset format |
| row_count | INTEGER | Number of records |
| column_count | INTEGER | Number of columns |
| created_at | TIMESTAMP | Upload time |

### 5.3 Schema Configuration Table

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| tenant_id | UUID | Client reference |
| dataset_id | UUID | Dataset reference |
| model_type | VARCHAR | Selected model type |
| column_mapping | JSONB | Source-to-canonical mapping |
| created_at | TIMESTAMP | Creation time |

### 5.4 Training Job Table

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| tenant_id | UUID | Client reference |
| dataset_id | UUID | Dataset reference |
| model_type | VARCHAR | Model being trained |
| status | VARCHAR | Job status |
| parameters | JSONB | Training configuration |
| started_at | TIMESTAMP | Start time |
| completed_at | TIMESTAMP | Completion time |

### 5.5 Model Table

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| tenant_id | UUID | Client reference |
| name | VARCHAR | Model name |
| model_type | VARCHAR | Model category |
| description | TEXT | Model description |
| created_at | TIMESTAMP | Creation time |

### 5.6 Model Version Table

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| model_id | UUID | Model reference |
| version | INTEGER | Version number |
| artifact_path | TEXT | Model artifact location |
| metrics | JSONB | Evaluation metrics |
| status | VARCHAR | Version status |
| created_at | TIMESTAMP | Registration time |

### 5.7 Inference Log Table

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| tenant_id | UUID | Client reference |
| model_version_id | UUID | Model version |
| status | VARCHAR | Prediction status |
| latency_ms | FLOAT | Execution latency |
| created_at | TIMESTAMP | Request time |

All tenant-owned records must be protected through authorization and tenant-aware database access.

Foreign keys, unique constraints, indexes, deletion policies, and detailed migration definitions will be finalized before database implementation.

## 6. API Contract Design

The platform will expose versioned REST APIs.

### 6.1 Dataset Upload

**Endpoint:**

`POST /api/v1/datasets`

**Request:** Multipart file upload.

**Response example:**

```json
{
  "dataset_id": "uuid",
  "name": "books.csv",
  "status": "uploaded",
  "row_count": 10000,
  "column_count": 5
}
```

### 6.2 Dataset Validation

**Endpoint:**

`POST /api/v1/datasets/{dataset_id}/validate`

**Response example:**

```json
{
  "dataset_id": "uuid",
  "valid": true,
  "issues": [],
  "total_rows": 10000
}
```

### 6.3 Schema Mapping

**Endpoint:**

`POST /api/v1/datasets/{dataset_id}/schema`

**Request example:**

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

### 6.4 Start Training

**Endpoint:**

`POST /api/v1/training/jobs`

**Request example:**

```json
{
  "dataset_id": "uuid",
  "model_type": "recommendation",
  "algorithm": "collaborative_filtering",
  "parameters": {
    "n_factors": 50,
    "epochs": 20
  }
}
```

**Response example:**

```json
{
  "job_id": "uuid",
  "status": "pending",
  "message": "Training job accepted"
}
```

### 6.5 Get Training Status

**Endpoint:**

`GET /api/v1/training/jobs/{job_id}`

**Response example:**

```json
{
  "job_id": "uuid",
  "status": "completed",
  "model_id": "uuid",
  "version": 1
}
```

### 6.6 Generate Recommendations

**Endpoint:**

`POST /api/v1/predictions/recommendations`

**Request example:**

```json
{
  "model_id": "uuid",
  "user_id": "user_101",
  "top_k": 5
}
```

**Response example:**

```json
{
  "model_id": "uuid",
  "model_version": 1,
  "user_id": "user_101",
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
```

These examples illustrate proposed contracts. Exact validation rules and response schemas will be defined during implementation.

## 7. Recommendation Data Contracts

The initial recommendation engine will distinguish two types of data.

### 7.1 User-Item Interactions

```text
user_id
item_id
rating
```

Represents user interactions with items.

### 7.2 Item Metadata

```text
item_id
item_name
category
description
```

Represents item attributes.

These datasets will be connected using `item_id`.

Additional attributes may be supported by individual recommendation algorithms.

## 8. Model Training Workflow

The training workflow will follow these steps:

1. Receive a training request.
2. Verify tenant ownership of the dataset.
3. Retrieve the selected dataset.
4. Load and validate the configured schema.
5. Preprocess the dataset.
6. Split data into training and evaluation sets.
7. Initialize the selected recommendation algorithm.
8. Train the model.
9. Evaluate the trained model.
10. Save the model artifact.
11. Register the model and version.
12. Return the training result.

Training should execute as a background job once long-running workloads are introduced.

## 9. Inference Workflow

The inference workflow will follow these steps:

1. Receive a prediction request.
2. Authenticate the client.
3. Validate tenant access.
4. Resolve the requested model.
5. Resolve the selected model version.
6. Load the model artifact if it is not already available.
7. Validate prediction input.
8. Execute the model's prediction method.
9. Format the output.
10. Record inference metadata.
11. Return the prediction response.

## 10. Model Artifact Management

Trained models will be serialized and stored separately from relational metadata.

Each registered model version must retain:
- Model identifier.
- Version identifier.
- Algorithm type.
- Artifact location.
- Training dataset reference.
- Training parameters.
- Evaluation metrics.
- Creation timestamp.

The artifact must be loaded only after the caller's authorization and model ownership have been verified.

Serialized model artifacts must be treated as trusted internal files. Untrusted uploaded pickle files must never be loaded.

## 11. Error Handling Design

The platform will use consistent error responses.

Example:

```json
{
  "error": {
    "code": "DATASET_NOT_FOUND",
    "message": "The requested dataset does not exist."
  }
}
```

Initial error categories:

| Error Code | Description |
|---|---|
| INVALID_FILE_FORMAT | Unsupported dataset format |
| DATASET_NOT_FOUND | Dataset does not exist |
| DATA_VALIDATION_FAILED | Dataset failed validation |
| INVALID_SCHEMA | Incorrect column mapping |
| TRAINING_FAILED | Model training failed |
| MODEL_NOT_FOUND | Requested model does not exist |
| MODEL_VERSION_NOT_FOUND | Requested version does not exist |
| INVALID_PREDICTION_INPUT | Invalid prediction request |
| UNAUTHORIZED_ACCESS | Caller lacks required access |

Error responses must not expose passwords, secrets, internal file paths, or stack traces.

## 12. Testing Strategy

### 12.1 Unit Testing

Test individual functions and classes.

Examples:
- Dataset validation.
- Schema mapping.
- Recommendation algorithms.
- Model evaluation.
- Model version resolution.

### 12.2 Integration Testing

Test interactions between modules.

Examples:
- Dataset upload and database storage.
- Training and model registration.
- Model loading and inference.

### 12.3 API Testing

Test HTTP endpoints.

Examples:
- Valid and invalid uploads.
- Authentication failures.
- Training requests.
- Prediction responses.
- Tenant isolation.

### 12.4 ML Testing

Validate:
- Data leakage prevention.
- Recommendation ranking correctness.
- Metric calculations.
- Cold-start behavior.
- Reproducibility.

## 13. Design Principles

1. Separation of concerns.
2. Reusable model interfaces.
3. Explicit data contracts.
4. Tenant-aware resource access.
5. Independent training and inference workflows.
6. Consistent error handling.
7. Automated testing.
8. Reproducible model training.
9. Secure model artifact management.
10. Incremental architectural evolution.

## 14. Future Extensions

The LLD will be extended when the following capabilities are introduced:

- Fraud detection implementation.
- Price prediction implementation.
- Customer churn implementation.
- Advanced background job processing.
- Kafka-based event processing.
- Distributed inference services.
- Spring Boot integration.
- Advanced model monitoring.

---

**Project:** Multi-Model AI Inference and Serving Platform

**Document:** Low-Level Design

**Status:** Initial Draft