# Database Design
## Multi-Model AI Inference and Serving Platform

## 1. Introduction

This document defines the relational database design for the Multi-Model AI Inference and Serving Platform.

PostgreSQL will be used as the primary relational database for storing platform metadata, client information, dataset configurations, training jobs, model versions, and inference records.

Large dataset files and trained model artifacts will be stored separately using file or object storage.

## 2. Database Design Objectives

The database design must support:

- Multiple independent clients.
- Tenant-level data isolation.
- Dataset and schema management.
- Training job tracking.
- Model registration and versioning.
- Model evaluation results.
- Prediction metadata.
- Future integration of additional ML models.

## 3. Database Entities

The initial database will contain the following entities:

| Entity | Purpose |
|---|---|
| tenants | Stores client/tenant information |
| users | Stores platform user accounts |
| datasets | Stores uploaded dataset metadata |
| dataset_versions | Tracks dataset versions |
| schema_configurations | Stores dataset column mappings |
| training_jobs | Tracks model training executions |
| models | Stores logical model information |
| model_versions | Stores individual model versions |
| inference_logs | Records prediction execution metadata |

## 4. Entity Relationship Diagram

```text
                    TENANTS
                       |
             +---------+---------+
             |                   |
             v                   v
           USERS              DATASETS
                                  |
                                  |
                           DATASET_VERSIONS
                                  |
                                  v
                         SCHEMA_CONFIGURATIONS
                                  |
                                  v
                             TRAINING_JOBS
                                  |
                                  v
                                MODELS
                                  |
                                  v
                            MODEL_VERSIONS
                                  |
                                  v
                            INFERENCE_LOGS
```

Additional relationships:

- A tenant can have multiple users.
- A tenant can own multiple datasets.
- A dataset can have multiple versions.
- A dataset can have multiple schema configurations.
- A tenant can initiate multiple training jobs.
- A model can have multiple versions.
- A model version can be referenced by multiple inference records.

## 5. Table Definitions

### 5.1 tenants

Stores information about registered clients.

| Column | Data Type | Constraints |
|---|---|---|
| id | UUID | Primary Key |
| name | VARCHAR(150) | NOT NULL |
| slug | VARCHAR(100) | UNIQUE, NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 5.2 users

Stores user accounts associated with tenants.

| Column | Data Type | Constraints |
|---|---|---|
| id | UUID | Primary Key |
| tenant_id | UUID | Foreign Key |
| email | VARCHAR(255) | NOT NULL |
| password_hash | TEXT | NOT NULL |
| role | VARCHAR(30) | NOT NULL |
| is_active | BOOLEAN | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Unique constraint:

`UNIQUE (tenant_id, email)`

Roles may initially include:
- TENANT_ADMIN
- TENANT_MEMBER

Platform-level administration will be designed separately if required.

### 5.3 datasets

Stores metadata about uploaded datasets.

| Column | Data Type | Constraints |
|---|---|---|
| id | UUID | Primary Key |
| tenant_id | UUID | Foreign Key |
| name | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| file_format | VARCHAR(20) | NOT NULL |
| current_version_id | UUID | NULL |
| status | VARCHAR(30) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Possible statuses:
- UPLOADED
- VALIDATING
- VALID
- INVALID
- ARCHIVED

### 5.4 dataset_versions

Tracks individual uploaded versions of datasets.

| Column | Data Type | Constraints |
|---|---|---|
| id | UUID | Primary Key |
| dataset_id | UUID | Foreign Key |
| version_number | INTEGER | NOT NULL |
| storage_path | TEXT | NOT NULL |
| file_size_bytes | BIGINT | NOT NULL |
| checksum | VARCHAR(128) | NULL |
| row_count | BIGINT | NULL |
| column_count | INTEGER | NULL |
| uploaded_at | TIMESTAMPTZ | NOT NULL |

Unique constraint:

`UNIQUE (dataset_id, version_number)`

The checksum can be used to identify identical uploaded files.

### 5.5 schema_configurations

Stores mappings between client dataset columns and canonical model fields.

| Column | Data Type | Constraints |
|---|---|---|
| id | UUID | Primary Key |
| tenant_id | UUID | Foreign Key |
| dataset_id | UUID | Foreign Key |
| dataset_version_id | UUID | Foreign Key |
| model_type | VARCHAR(50) | NOT NULL |
| mapping | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Example mapping:

```json
{
  "user_id": "customer_id",
  "item_id": "book_code",
  "rating": "user_rating"
}
```

### 5.6 training_jobs

Tracks training requests and their execution status.

| Column | Data Type | Constraints |
|---|---|---|
| id | UUID | Primary Key |
| tenant_id | UUID | Foreign Key |
| dataset_version_id | UUID | Foreign Key |
| schema_configuration_id | UUID | Foreign Key |
| model_id | UUID | NULL, Foreign Key |
| model_type | VARCHAR(50) | NOT NULL |
| algorithm | VARCHAR(100) | NOT NULL |
| parameters | JSONB | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| error_message | TEXT | NULL |
| started_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Possible statuses:
- PENDING
- RUNNING
- COMPLETED
- FAILED
- CANCELLED

### 5.7 models

Stores logical model information.

| Column | Data Type | Constraints |
|---|---|---|
| id | UUID | Primary Key |
| tenant_id | UUID | Foreign Key |
| name | VARCHAR(150) | NOT NULL |
| model_type | VARCHAR(50) | NOT NULL |
| description | TEXT | NULL |
| status | VARCHAR(30) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

A logical model can have multiple registered versions.

### 5.8 model_versions

Stores individual trained model versions.

| Column | Data Type | Constraints |
|---|---|---|
| id | UUID | Primary Key |
| model_id | UUID | Foreign Key |
| training_job_id | UUID | Foreign Key |
| version_number | INTEGER | NOT NULL |
| artifact_path | TEXT | NOT NULL |
| metrics | JSONB | NOT NULL |
| parameters | JSONB | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| is_active | BOOLEAN | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Unique constraint:

`UNIQUE (model_id, version_number)`

Possible statuses:
- REGISTERED
- READY
- INACTIVE
- FAILED
- ARCHIVED

Only an eligible version should be activated for inference.

### 5.9 inference_logs

Stores metadata about prediction requests.

| Column | Data Type | Constraints |
|---|---|---|
| id | UUID | Primary Key |
| tenant_id | UUID | Foreign Key |
| model_version_id | UUID | Foreign Key |
| request_id | UUID | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| latency_ms | FLOAT | NULL |
| error_code | VARCHAR(100) | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Sensitive prediction inputs and outputs should not be logged by default. Detailed prediction audit storage can be introduced with explicit retention and privacy policies.

## 6. Database Relationships

| Parent Entity | Child Entity | Relationship |
|---|---|---|
| tenants | users | One-to-Many |
| tenants | datasets | One-to-Many |
| tenants | training_jobs | One-to-Many |
| tenants | models | One-to-Many |
| datasets | dataset_versions | One-to-Many |
| datasets | schema_configurations | One-to-Many |
| dataset_versions | training_jobs | One-to-Many |
| models | model_versions | One-to-Many |
| training_jobs | model_versions | One-to-One or One-to-Many, depending on training strategy |
| model_versions | inference_logs | One-to-Many |

## 7. Indexing Strategy

Indexes will be created for frequently queried columns.

| Table | Index | Purpose |
|---|---|---|
| users | tenant_id, email | Tenant-specific login lookup |
| datasets | tenant_id | Retrieve tenant datasets |
| dataset_versions | dataset_id, version_number | Dataset version lookup |
| training_jobs | tenant_id, status | Retrieve training jobs |
| models | tenant_id, model_type | Model discovery |
| model_versions | model_id, version_number | Model version lookup |
| inference_logs | tenant_id, created_at | Tenant-specific log retrieval |

Unique constraints will also create supporting indexes where appropriate.

## 8. Data Integrity and Constraints

The database must enforce:

1. Primary key constraints.
2. Foreign key constraints.
3. Unique constraints for tenant-specific user emails.
4. Unique dataset version numbers within a dataset.
5. Unique model version numbers within a model.
6. Non-null constraints for required fields.
7. Appropriate status values using database constraints or validated application enums.

Tenant ownership must also be checked at the application/service layer. Foreign keys alone do not guarantee tenant isolation.

For stronger protection, PostgreSQL Row-Level Security may be evaluated in a later phase.

## 9. Data Storage Strategy

PostgreSQL:
- Tenant metadata.
- User information.
- Dataset metadata.
- Training job metadata.
- Model metadata.
- Model version information.
- Inference metadata.

File/Object Storage:
- Uploaded CSV datasets.
- Processed datasets where necessary.
- Serialized model artifacts.
- Training output artifacts.

Redis:
- Temporary job state where needed.
- Caching.
- Rate-limiting state.

## 10. Database Migration Strategy

Database schema changes will be managed using Alembic.

Every database schema change should be version-controlled through migrations.

The application must not rely on manually modifying production database tables.

## 11. Future Enhancements

Potential future database improvements include:

- PostgreSQL Row-Level Security.
- Partitioning inference logs.
- Audit event storage.
- Dataset lineage tracking.
- Model deployment history.
- Usage and billing records.
- Database read replicas.

These will be introduced when required.

---

**Project:** Multi-Model AI Inference and Serving Platform

**Document:** Database Design

**Status:** Initial Draft