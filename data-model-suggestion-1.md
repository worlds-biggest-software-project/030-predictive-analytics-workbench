# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Predictive Analytics Workbench · Created: 2026-05-11

## Philosophy

This model follows a fully normalized relational approach where every domain concept — workspaces, datasets, features, experiments, models, predictions, drift events — occupies its own table with explicit foreign key relationships. The design mirrors how MLflow, SageMaker, and traditional ML platforms structure their metadata: separate tables for experiments, runs, parameters, metrics, tags, and artifacts, all linked by foreign keys.

The normalized approach prioritises data integrity and query flexibility. Every relationship is explicit and enforceable by the database. Complex cross-entity queries (e.g., "find all models trained on datasets where feature X had drift above threshold Y in the last 30 days") are straightforward JOINs rather than JSONB path traversals. This is the approach most PostgreSQL teams are familiar with.

The trade-off is table count: this model has the most tables of all proposals. Schema migrations require more coordination, and the rigid structure means adding jurisdiction-specific or domain-specific metadata requires ALTER TABLE statements rather than flexible JSONB fields. But for a platform where audit trails, reproducibility, and data lineage are core requirements, normalization provides the strongest foundation.

**Best for:** Teams prioritising data integrity, complex cross-entity reporting, and regulatory compliance with audit-ready SQL queries.

**Trade-offs:**
- (+) Full referential integrity enforced at the database level
- (+) Complex JOIN queries are straightforward and well-optimised by PostgreSQL
- (+) Standard schema that any backend developer can understand
- (+) Clean separation of concerns; each table has a single responsibility
- (-) Highest table count (~35 tables); more migrations to manage
- (-) Adding new metadata fields requires schema migrations
- (-) Many JOINs for common read paths (experiment → runs → metrics → model → predictions)
- (-) Less flexible for domain-specific or user-defined metadata

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MLflow Schema | Experiments, runs, params, metrics, tags tables mirror MLflow's proven relational schema |
| PMML 4.4.1 | Model export metadata stored in `model_versions.export_format` with PMML-specific fields |
| ONNX IR | Model artifact references track ONNX opset version and graph structure metadata |
| SHAP | Explainability results stored in dedicated `shap_explanations` table with per-feature values |
| ISO 8601 | All timestamps use `TIMESTAMPTZ` with ISO 8601 formatting |
| ISO 3166-1 | Jurisdiction/region codes for data residency compliance |
| Feast Feature Store | Feature definitions and feature sets mirror Feast's entity/feature-view concepts |

---

## Organisation & Multi-Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, pro, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(320) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),                         -- NULL if SSO-only
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local', -- local, google, github, saml
    avatar_url      TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member', -- owner, admin, member, viewer
    invited_by      UUID REFERENCES users(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_members_org ON organisation_members(organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members(user_id);
```

## Workspaces & Projects

```sql
CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    problem_type    VARCHAR(50) NOT NULL,  -- classification, regression, time_series, multi_class
    domain_context  TEXT,                  -- natural language description of the business problem
    target_column   VARCHAR(255),
    created_by      UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft', -- draft, active, archived
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, slug)
);

CREATE INDEX idx_projects_workspace ON projects(workspace_id);
CREATE INDEX idx_projects_status ON projects(status);
```

## Data Sources & Datasets

```sql
CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,  -- csv_upload, snowflake, bigquery, redshift, postgresql, s3, api
    connection_config JSONB NOT NULL DEFAULT '{}',
    -- Example connection_config for Snowflake:
    -- {"account": "xy12345", "warehouse": "COMPUTE_WH", "database": "ANALYTICS", "schema": "PUBLIC"}
    credentials_ref VARCHAR(255),          -- reference to secrets manager, never stored inline
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_synced_at  TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE datasets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    data_source_id  UUID REFERENCES data_sources(id),
    name            VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    row_count       BIGINT,
    column_count    INTEGER,
    file_size_bytes BIGINT,
    file_path       TEXT,                  -- S3/GCS path to stored data file
    file_format     VARCHAR(20),           -- csv, parquet, json
    schema_snapshot JSONB NOT NULL DEFAULT '[]',
    -- Example schema_snapshot:
    -- [{"name": "revenue", "dtype": "float64", "nullable": true, "unique_count": 1200},
    --  {"name": "date", "dtype": "datetime64", "nullable": false, "unique_count": 365}]
    data_quality    JSONB NOT NULL DEFAULT '{}',
    -- Example data_quality:
    -- {"null_rates": {"revenue": 0.02, "region": 0.0}, "outlier_count": 14, "class_balance": {"churn": 0.15, "retain": 0.85}}
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_datasets_project ON datasets(project_id);
CREATE INDEX idx_datasets_version ON datasets(project_id, version);

CREATE TABLE dataset_columns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    data_type       VARCHAR(50) NOT NULL,  -- float64, int64, string, datetime64, bool, category
    is_target       BOOLEAN NOT NULL DEFAULT false,
    is_feature      BOOLEAN NOT NULL DEFAULT true,
    null_count      BIGINT DEFAULT 0,
    null_rate       DOUBLE PRECISION DEFAULT 0.0,
    unique_count    BIGINT,
    mean            DOUBLE PRECISION,
    std_dev         DOUBLE PRECISION,
    min_value       TEXT,
    max_value       TEXT,
    sample_values   JSONB,  -- ["val1", "val2", "val3", ...]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dataset_columns_dataset ON dataset_columns(dataset_id);
```

## Feature Engineering

```sql
CREATE TABLE feature_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    generation_mode VARCHAR(50) NOT NULL, -- manual, auto_ml, llm_context_driven
    domain_prompt   TEXT,                 -- the natural language prompt that drove LLM feature generation
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE features (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    feature_set_id  UUID NOT NULL REFERENCES feature_sets(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    description     TEXT,
    feature_type    VARCHAR(50) NOT NULL,  -- raw, derived, lag, rolling_window, seasonal, cohort, interaction
    data_type       VARCHAR(50) NOT NULL,  -- float64, int64, string, bool, category
    source_columns  JSONB NOT NULL DEFAULT '[]',  -- ["revenue", "date"]
    transform_logic TEXT,                  -- SQL or Python expression defining the transformation
    -- Example: "LAG(revenue, 7) OVER (ORDER BY date)"
    parameters      JSONB NOT NULL DEFAULT '{}',
    -- Example for rolling_window: {"window_size": 30, "aggregation": "mean", "column": "revenue"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_features_set ON features(feature_set_id);
CREATE INDEX idx_features_type ON features(feature_type);
```

## Experiments & Runs (MLflow-aligned)

```sql
CREATE TABLE experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    dataset_id      UUID NOT NULL REFERENCES datasets(id),
    feature_set_id  UUID REFERENCES feature_sets(id),
    target_column   VARCHAR(255) NOT NULL,
    problem_type    VARCHAR(50) NOT NULL,  -- classification, regression, time_series
    split_strategy  VARCHAR(50) NOT NULL DEFAULT 'auto', -- auto, time_based, stratified, custom
    test_size       DOUBLE PRECISION NOT NULL DEFAULT 0.2,
    status          VARCHAR(50) NOT NULL DEFAULT 'created', -- created, running, completed, failed
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_experiments_project ON experiments(project_id);
CREATE INDEX idx_experiments_status ON experiments(status);

CREATE TABLE runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    run_number      INTEGER NOT NULL,
    algorithm       VARCHAR(100) NOT NULL,  -- lightgbm, xgboost, catboost, linear_regression, arima, deepar, chronos, ensemble
    status          VARCHAR(50) NOT NULL DEFAULT 'queued', -- queued, running, completed, failed, cancelled
    start_time      TIMESTAMPTZ,
    end_time        TIMESTAMPTZ,
    duration_seconds DOUBLE PRECISION,
    error_message   TEXT,
    artifact_path   TEXT,                   -- S3/GCS path to model artifact
    model_size_bytes BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runs_experiment ON runs(experiment_id);
CREATE INDEX idx_runs_status ON runs(status);

CREATE TABLE run_params (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
    key             VARCHAR(255) NOT NULL,
    value           TEXT NOT NULL,
    UNIQUE (run_id, key)
);

CREATE INDEX idx_run_params_run ON run_params(run_id);

CREATE TABLE run_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
    key             VARCHAR(255) NOT NULL,  -- accuracy, precision, recall, f1, auc_roc, rmse, mae, mape
    value           DOUBLE PRECISION NOT NULL,
    step            INTEGER DEFAULT 0,       -- for metrics logged at multiple steps (e.g., training epochs)
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (run_id, key, step)
);

CREATE INDEX idx_run_metrics_run ON run_metrics(run_id);
CREATE INDEX idx_run_metrics_key ON run_metrics(key);

CREATE TABLE run_tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
    key             VARCHAR(255) NOT NULL,
    value           TEXT NOT NULL,
    UNIQUE (run_id, key)
);
```

## Model Registry

```sql
CREATE TABLE registered_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, name)
);

CREATE TABLE model_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    registered_model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version         INTEGER NOT NULL,
    run_id          UUID NOT NULL REFERENCES runs(id),
    stage           VARCHAR(50) NOT NULL DEFAULT 'none', -- none, staging, production, archived
    artifact_path   TEXT NOT NULL,           -- S3/GCS path to model artifact
    export_format   VARCHAR(50),             -- autogluon, onnx, pmml, torchscript
    framework       VARCHAR(100),            -- autogluon, sklearn, pytorch, xgboost
    description     TEXT,
    promoted_by     UUID REFERENCES users(id),
    promoted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (registered_model_id, version)
);

CREATE INDEX idx_model_versions_model ON model_versions(registered_model_id);
CREATE INDEX idx_model_versions_stage ON model_versions(stage);
CREATE INDEX idx_model_versions_run ON model_versions(run_id);
```

## Model Leaderboard

```sql
CREATE TABLE leaderboard_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    run_id          UUID NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
    rank            INTEGER NOT NULL,
    primary_metric  VARCHAR(100) NOT NULL,   -- the metric used for ranking (e.g., rmse, accuracy)
    primary_score   DOUBLE PRECISION NOT NULL,
    is_best         BOOLEAN NOT NULL DEFAULT false,
    selection_reason TEXT,                    -- LLM-generated explanation of why this model was selected
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leaderboard_experiment ON leaderboard_entries(experiment_id);
CREATE UNIQUE INDEX idx_leaderboard_best ON leaderboard_entries(experiment_id) WHERE is_best = true;
```

## Explainability (SHAP)

```sql
CREATE TABLE shap_explanations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
    explanation_type VARCHAR(50) NOT NULL,   -- global, local
    base_value      DOUBLE PRECISION NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE shap_feature_values (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    explanation_id  UUID NOT NULL REFERENCES shap_explanations(id) ON DELETE CASCADE,
    feature_name    VARCHAR(255) NOT NULL,
    shap_value      DOUBLE PRECISION NOT NULL,
    feature_value   TEXT,                    -- the actual feature value for local explanations
    abs_importance  DOUBLE PRECISION NOT NULL, -- |shap_value| for sorting
    rank            INTEGER NOT NULL
);

CREATE INDEX idx_shap_values_explanation ON shap_feature_values(explanation_id);
CREATE INDEX idx_shap_values_rank ON shap_feature_values(explanation_id, rank);

CREATE TABLE prediction_narratives (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES runs(id),
    model_version_id UUID REFERENCES model_versions(id),
    narrative_type  VARCHAR(50) NOT NULL,    -- global_summary, prediction_explanation, drift_explanation
    narrative_text  TEXT NOT NULL,            -- LLM-generated plain-English explanation
    generated_by    VARCHAR(50) NOT NULL DEFAULT 'claude', -- claude, gpt4, local_llm
    prompt_tokens   INTEGER,
    completion_tokens INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_narratives_run ON prediction_narratives(run_id);
```

## Predictions & Batch Jobs

```sql
CREATE TABLE prediction_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_version_id UUID NOT NULL REFERENCES model_versions(id),
    project_id      UUID NOT NULL REFERENCES projects(id),
    job_type        VARCHAR(50) NOT NULL,    -- batch, realtime_endpoint
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- pending, running, completed, failed
    input_dataset_id UUID REFERENCES datasets(id),
    input_row_count BIGINT,
    output_path     TEXT,                    -- S3/GCS path to prediction output
    output_row_count BIGINT,
    writeback_target VARCHAR(255),           -- warehouse table name for write-back
    start_time      TIMESTAMPTZ,
    end_time        TIMESTAMPTZ,
    error_message   TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_prediction_jobs_model ON prediction_jobs(model_version_id);
CREATE INDEX idx_prediction_jobs_status ON prediction_jobs(status);

CREATE TABLE predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prediction_job_id UUID NOT NULL REFERENCES prediction_jobs(id) ON DELETE CASCADE,
    row_index       INTEGER NOT NULL,
    predicted_value TEXT NOT NULL,
    confidence      DOUBLE PRECISION,
    prediction_proba JSONB,                  -- {"churn": 0.82, "retain": 0.18}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_predictions_job ON predictions(prediction_job_id);
```

## Drift Detection & Monitoring

```sql
CREATE TABLE drift_monitors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_version_id UUID NOT NULL REFERENCES model_versions(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    check_interval_hours INTEGER NOT NULL DEFAULT 24,
    drift_threshold DOUBLE PRECISION NOT NULL DEFAULT 0.1,
    metric_threshold JSONB NOT NULL DEFAULT '{}',
    -- Example: {"rmse_max": 15.0, "mape_max": 0.12}
    retrain_on_drift BOOLEAN NOT NULL DEFAULT false,
    notify_on_drift BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE drift_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    drift_monitor_id UUID NOT NULL REFERENCES drift_monitors(id) ON DELETE CASCADE,
    check_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_drift_score DOUBLE PRECISION NOT NULL,
    is_drifted      BOOLEAN NOT NULL DEFAULT false,
    feature_drifts  JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"feature": "revenue", "psi": 0.23, "drifted": true}, {"feature": "region", "psi": 0.01, "drifted": false}]
    performance_metrics JSONB NOT NULL DEFAULT '{}',
    -- Example: {"rmse": 14.2, "mape": 0.11, "baseline_rmse": 10.5}
    action_taken    VARCHAR(50),             -- none, notification_sent, retrain_triggered
    narrative       TEXT,                    -- LLM-generated drift explanation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drift_checks_monitor ON drift_checks(drift_monitor_id);
CREATE INDEX idx_drift_checks_time ON drift_checks(check_time);
CREATE INDEX idx_drift_checks_drifted ON drift_checks(is_drifted) WHERE is_drifted = true;

CREATE TABLE retraining_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    drift_check_id  UUID REFERENCES drift_checks(id),
    model_version_id UUID NOT NULL REFERENCES model_versions(id),
    new_model_version_id UUID REFERENCES model_versions(id),
    trigger         VARCHAR(50) NOT NULL,    -- manual, drift_auto, scheduled
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- pending, running, completed, failed, rejected
    comparison_metrics JSONB,                -- {"old_rmse": 14.2, "new_rmse": 10.1, "improvement_pct": 28.9}
    auto_promoted   BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_retraining_jobs_model ON retraining_jobs(model_version_id);
```

## LLM Copilot Conversations

```sql
CREATE TABLE copilot_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    session_type    VARCHAR(50) NOT NULL,    -- onboarding, data_quality, feature_engineering, model_selection, interpretation
    status          VARCHAR(50) NOT NULL DEFAULT 'active', -- active, completed, abandoned
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE copilot_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES copilot_sessions(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL,    -- user, assistant, system
    content         TEXT NOT NULL,
    tool_calls      JSONB,                   -- LLM tool calls if any
    token_count     INTEGER,
    model_used      VARCHAR(100),            -- claude-opus-4-20250514, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_copilot_messages_session ON copilot_messages(session_id);
CREATE INDEX idx_copilot_messages_time ON copilot_messages(created_at);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,   -- model.created, model.promoted, prediction.run, dataset.uploaded, drift.detected
    resource_type   VARCHAR(50) NOT NULL,    -- project, dataset, experiment, model, prediction
    resource_id     UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);
CREATE INDEX idx_audit_log_time ON audit_log(created_at);
```

## Notifications

```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    notification_type VARCHAR(50) NOT NULL,  -- drift_alert, training_complete, prediction_ready, model_promoted
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,
    resource_type   VARCHAR(50),
    resource_id     UUID,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, is_read);
CREATE INDEX idx_notifications_time ON notifications(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | Multi-tenant with RBAC |
| Workspaces & Projects | 2 | Hierarchical organisation |
| Data Sources & Datasets | 3 | Including per-column statistics |
| Feature Engineering | 2 | Feature sets with transformation definitions |
| Experiments & Runs | 5 | MLflow-aligned with params, metrics, tags |
| Model Registry | 2 | Versioned model lifecycle |
| Leaderboard | 1 | Ranked model comparison |
| Explainability | 3 | SHAP values + LLM narratives |
| Predictions | 2 | Batch jobs + individual predictions |
| Drift & Monitoring | 3 | Monitor → check → retrain pipeline |
| LLM Copilot | 2 | Conversation history |
| Audit & Notifications | 2 | Compliance + user alerts |
| **Total** | **30** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — standard for distributed SaaS; no sequential ID leakage, safe for multi-region deployment.

2. **MLflow-compatible experiment structure** — experiments → runs → (params, metrics, tags) mirrors MLflow's proven schema, enabling future MLflow compatibility or migration.

3. **Separate `dataset_columns` table** — per-column statistics (null rates, distributions, data types) stored relationally rather than in JSONB, enabling efficient queries like "find all datasets with >5% null rate in the target column."

4. **Feature engineering as first-class entities** — `feature_sets` and `features` are top-level tables, not metadata buried in experiment configs, reflecting the platform's emphasis on LLM-driven context-aware feature generation.

5. **SHAP values normalised into rows** — each feature's SHAP contribution is a separate row in `shap_feature_values`, enabling queries like "which feature has the highest average |SHAP| across all models in this workspace."

6. **Drift monitoring as a pipeline** — `drift_monitors` → `drift_checks` → `retraining_jobs` models the full lifecycle from configuration to detection to remediation.

7. **Copilot conversations stored separately** — LLM interaction history is a first-class entity for audit, debugging, and potential fine-tuning, not an afterthought in a JSONB field.

8. **Audit log with resource polymorphism** — `resource_type` + `resource_id` pattern avoids a foreign key per entity type while maintaining queryability via composite index.

9. **Credentials never stored inline** — `data_sources.credentials_ref` points to an external secrets manager (AWS Secrets Manager, HashiCorp Vault), never storing connection passwords in the database.

10. **Partial indexes for common queries** — `WHERE is_best = true` on leaderboard, `WHERE is_drifted = true` on drift checks, reducing index size for frequently filtered queries.
