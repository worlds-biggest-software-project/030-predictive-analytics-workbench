# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Predictive Analytics Workbench · Created: 2026-05-11

## Philosophy

This model uses a pragmatic hybrid approach: core structural fields (identifiers, foreign keys, status, timestamps) are relational columns with standard indexes, while variable, domain-specific, and rapidly evolving fields are stored as JSONB. The result is fewer tables than a fully normalised model, faster development velocity, and built-in flexibility for features that differ across problem types (classification vs. regression vs. time-series) and domains (finance vs. retail vs. healthcare).

PostgreSQL's JSONB support is mature and performant: GIN indexes enable efficient containment queries (`@>`), path expressions (`->`, `->>`), and full-text search within JSONB columns. The hybrid pattern is used extensively in modern SaaS platforms (Stripe, Shopify, Linear) where some fields are universal (created_at, status, owner_id) but others vary by customer, use case, or configuration.

For a predictive analytics workbench, this is particularly useful because: (a) experiment hyperparameters vary by algorithm (a random forest has `n_estimators` and `max_depth`; a neural network has `learning_rate` and `batch_size`), (b) data quality profiles vary by column type (numeric columns have mean/std; categorical columns have cardinality), (c) feature engineering transformations vary by feature type (lag features need window size; rolling features need aggregation function), and (d) drift detection metrics vary by statistical test (PSI, KS, JS divergence). Rather than creating separate tables or columns for every variation, JSONB captures the variability while relational columns anchor the structure.

**Best for:** Rapid MVP development; platforms that span multiple ML problem types and domains; teams that want schema flexibility without sacrificing query performance on core fields.

**Trade-offs:**
- (+) Fewer tables (~18 vs. ~30 in normalised model); simpler migrations
- (+) Adding new metadata fields requires no schema migration — just add a key to the JSONB
- (+) Algorithm-specific hyperparameters, domain-specific features, and per-column statistics all stored naturally in JSONB
- (+) GIN indexes on JSONB enable efficient queries on variable fields
- (+) Faster development velocity; schema evolution is low-friction
- (-) JSONB fields lack database-enforced constraints (nullability, type, foreign keys)
- (-) Complex JSONB queries can be slower than normalised column queries without careful indexing
- (-) No referential integrity within JSONB — application must enforce consistency
- (-) JSONB fields can accumulate undocumented keys over time ("schema drift in the schema-less fields")
- (-) Backup and replication overhead for large JSONB columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MLflow Concepts | Experiment/run structure aligns with MLflow but params and metrics are JSONB maps rather than separate tables |
| PMML 4.4.1 | Model export config stored in `models.export_config` JSONB field |
| ONNX IR | ONNX metadata stored in `models.export_config` with opset version |
| SHAP | Explainability stored as JSONB arrays in `model_explanations.shap_values` |
| Feast Feature Store | Feature definitions stored as JSONB with type-specific transform parameters |
| PostgreSQL JSONB | GIN indexes, containment operators, path queries used throughout |
| ISO 8601 | All timestamps use `TIMESTAMPTZ` |
| ISO 3166-1 | Region codes in `organisations.settings` JSONB |

---

## Organisation & Users

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "region": "us-east-1",
    --   "data_residency": "US",              -- ISO 3166-1
    --   "default_compute": "cpu",
    --   "max_concurrent_training": 5,
    --   "sso_config": {"provider": "okta", "domain": "corp.okta.com"},
    --   "notification_channels": [{"type": "email"}, {"type": "slack", "webhook": "https://..."}]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(320) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {"avatar_url": "https://...", "timezone": "America/New_York", "preferences": {"theme": "dark"}}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_members_org ON organisation_members(organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members(user_id);
```

## Projects

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    problem_type    VARCHAR(50) NOT NULL,    -- classification, regression, time_series, multi_class
    domain_context  TEXT,                    -- NL description: "monthly SaaS subscription data, predict churn 90 days out"
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "target_column": "churn",
    --   "time_column": "date",                  -- for time-series problems
    --   "forecast_horizon": 30,                 -- days ahead to predict
    --   "seasonality": "monthly",
    --   "evaluation_metric": "f1_score",
    --   "train_test_split": 0.8,
    --   "cross_validation_folds": 5,
    --   "max_training_time_minutes": 60
    -- }
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_projects_org ON projects(organisation_id);
CREATE INDEX idx_projects_status ON projects(status);
CREATE INDEX idx_projects_problem_type ON projects(problem_type);
```

## Datasets

```sql
CREATE TABLE datasets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    source_type     VARCHAR(50) NOT NULL,    -- csv_upload, snowflake, bigquery, redshift, postgresql, s3
    source_config   JSONB NOT NULL DEFAULT '{}',
    -- Example source_config for Snowflake:
    -- {"account": "xy12345", "warehouse": "COMPUTE_WH", "database": "ANALYTICS",
    --  "schema": "PUBLIC", "table": "customers", "query": "SELECT * FROM customers WHERE year = 2025"}
    -- Credentials reference (never stored inline):
    -- {"credentials_ref": "vault://snowflake/prod-readonly"}
    row_count       BIGINT,
    file_path       TEXT,
    file_format     VARCHAR(20),
    schema_info     JSONB NOT NULL DEFAULT '[]',
    -- Example schema_info (per-column profiling):
    -- [
    --   {"name": "revenue", "dtype": "float64", "nullable": true, "null_rate": 0.02,
    --    "stats": {"mean": 45230.5, "std": 12400.3, "min": 0, "max": 250000, "p25": 28000, "p75": 58000}},
    --   {"name": "region", "dtype": "category", "nullable": false, "null_rate": 0.0,
    --    "stats": {"cardinality": 12, "top_values": [{"value": "US-WEST", "count": 5200}]}},
    --   {"name": "signup_date", "dtype": "datetime64", "nullable": false, "null_rate": 0.0,
    --    "stats": {"min": "2023-01-01", "max": "2025-12-31"}}
    -- ]
    quality_report  JSONB NOT NULL DEFAULT '{}',
    -- Example quality_report:
    -- {
    --   "overall_score": 0.87,
    --   "issues": [
    --     {"type": "class_imbalance", "severity": "warning", "detail": "Target 'churn' is 85/15 split"},
    --     {"type": "high_null_rate", "severity": "info", "column": "referral_source", "null_rate": 0.34},
    --     {"type": "potential_leakage", "severity": "critical", "column": "cancellation_date",
    --      "detail": "Column appears to directly encode the target variable"}
    --   ],
    --   "recommendations": ["Consider oversampling or SMOTE for class imbalance",
    --                        "Remove 'cancellation_date' — likely target leakage"]
    -- }
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_datasets_project ON datasets(project_id);
CREATE INDEX idx_datasets_version ON datasets(project_id, version);
-- GIN index for querying schema_info (e.g., find datasets with specific column names)
CREATE INDEX idx_datasets_schema ON datasets USING GIN (schema_info);
```

## Feature Engineering

```sql
CREATE TABLE feature_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    generation_mode VARCHAR(50) NOT NULL,    -- manual, auto_ml, llm_context_driven
    domain_prompt   TEXT,                    -- NL prompt that drove LLM feature generation
    features        JSONB NOT NULL DEFAULT '[]',
    -- Example features array:
    -- [
    --   {
    --     "name": "revenue_lag_7d",
    --     "display_name": "Revenue 7-Day Lag",
    --     "type": "lag",
    --     "data_type": "float64",
    --     "source_columns": ["revenue", "date"],
    --     "transform": "LAG(revenue, 7) OVER (ORDER BY date)",
    --     "params": {"lag_periods": 7, "column": "revenue", "order_by": "date"},
    --     "rationale": "Captures weekly revenue patterns for trend detection",
    --     "is_active": true
    --   },
    --   {
    --     "name": "revenue_rolling_30d_mean",
    --     "display_name": "30-Day Rolling Revenue Mean",
    --     "type": "rolling_window",
    --     "data_type": "float64",
    --     "source_columns": ["revenue", "date"],
    --     "transform": "AVG(revenue) OVER (ORDER BY date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW)",
    --     "params": {"window_size": 30, "aggregation": "mean", "column": "revenue"},
    --     "rationale": "Smooths short-term volatility to reveal underlying trend",
    --     "is_active": true
    --   },
    --   {
    --     "name": "cohort_month",
    --     "display_name": "Signup Cohort Month",
    --     "type": "cohort",
    --     "data_type": "category",
    --     "source_columns": ["signup_date"],
    --     "transform": "DATE_TRUNC('month', signup_date)",
    --     "params": {"granularity": "month", "column": "signup_date"},
    --     "rationale": "Groups customers by acquisition period for retention analysis",
    --     "is_active": true
    --   }
    -- ]
    feature_count   INTEGER NOT NULL DEFAULT 0,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_feature_sets_project ON feature_sets(project_id);
-- GIN index for searching features by type or name
CREATE INDEX idx_feature_sets_features ON feature_sets USING GIN (features);
```

## Experiments & Runs

```sql
CREATE TABLE experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    dataset_id      UUID NOT NULL REFERENCES datasets(id),
    feature_set_id  UUID REFERENCES feature_sets(id),
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "target_column": "churn",
    --   "problem_type": "classification",
    --   "split_strategy": "stratified",
    --   "test_size": 0.2,
    --   "algorithms": ["lightgbm", "xgboost", "catboost", "random_forest", "logistic_regression"],
    --   "time_limit_seconds": 3600,
    --   "evaluation_metrics": ["accuracy", "f1", "auc_roc"],
    --   "primary_metric": "f1"
    -- }
    status          VARCHAR(50) NOT NULL DEFAULT 'created',
    summary         JSONB NOT NULL DEFAULT '{}',
    -- Populated on completion:
    -- {
    --   "total_runs": 8,
    --   "best_run_id": "abc-123",
    --   "best_algorithm": "lightgbm",
    --   "best_score": {"f1": 0.94, "accuracy": 0.92, "auc_roc": 0.97},
    --   "total_training_time_seconds": 2847,
    --   "selection_reason": "LightGBM achieved the highest F1 score (0.94) while training 3x faster than XGBoost..."
    -- }
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
    algorithm       VARCHAR(100) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',
    -- Hyperparameters vary by algorithm — perfect JSONB use case
    hyperparameters JSONB NOT NULL DEFAULT '{}',
    -- Example for LightGBM:
    -- {"n_estimators": 500, "learning_rate": 0.01, "max_depth": 8, "num_leaves": 63,
    --  "subsample": 0.8, "colsample_bytree": 0.8, "reg_alpha": 0.1, "reg_lambda": 0.1}
    -- Example for ARIMA (time-series):
    -- {"order": [1, 1, 1], "seasonal_order": [1, 1, 1, 12], "trend": "c"}
    -- Example for Chronos (zero-shot):
    -- {"model_name": "chronos-t5-large", "prediction_length": 30, "context_length": 512}
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {"accuracy": 0.92, "precision": 0.89, "recall": 0.91, "f1": 0.90, "auc_roc": 0.96,
    --  "training_loss_history": [0.45, 0.32, 0.28, 0.25, 0.24]}
    artifact_path   TEXT,
    model_size_bytes BIGINT,
    start_time      TIMESTAMPTZ,
    end_time        TIMESTAMPTZ,
    duration_seconds DOUBLE PRECISION,
    error_message   TEXT,
    tags            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"autogluon_preset": "best_quality", "gpu": false, "fold": 3}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runs_experiment ON runs(experiment_id);
CREATE INDEX idx_runs_status ON runs(status);
CREATE INDEX idx_runs_algorithm ON runs(algorithm);
-- GIN index for querying specific hyperparameters or metrics
CREATE INDEX idx_runs_metrics ON runs USING GIN (metrics);
CREATE INDEX idx_runs_hyperparams ON runs USING GIN (hyperparameters);
```

## Model Registry

```sql
CREATE TABLE models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL,
    run_id          UUID NOT NULL REFERENCES runs(id),
    stage           VARCHAR(50) NOT NULL DEFAULT 'none',  -- none, staging, production, archived
    description     TEXT,
    artifact_path   TEXT NOT NULL,
    framework       VARCHAR(100),
    export_config   JSONB NOT NULL DEFAULT '{}',
    -- Example for ONNX export:
    -- {"format": "onnx", "opset_version": 17, "onnx_path": "s3://models/churn_v3.onnx"}
    -- Example for PMML export:
    -- {"format": "pmml", "pmml_version": "4.4.1", "pmml_path": "s3://models/churn_v3.pmml"}
    performance     JSONB NOT NULL DEFAULT '{}',
    -- Denormalised from run metrics for quick model comparison:
    -- {"f1": 0.94, "accuracy": 0.92, "auc_roc": 0.97, "training_time_seconds": 142}
    promoted_by     UUID REFERENCES users(id),
    promoted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, name, version)
);

CREATE INDEX idx_models_project ON models(project_id);
CREATE INDEX idx_models_stage ON models(stage);
CREATE INDEX idx_models_run ON models(run_id);
```

## Explainability

```sql
CREATE TABLE model_explanations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES models(id) ON DELETE CASCADE,
    run_id          UUID NOT NULL REFERENCES runs(id),
    explanation_type VARCHAR(50) NOT NULL,    -- global_shap, local_shap, surrogate
    shap_values     JSONB NOT NULL DEFAULT '{}',
    -- Example global SHAP:
    -- {
    --   "base_value": 0.15,
    --   "features": [
    --     {"name": "revenue_lag_7d", "mean_abs_shap": 0.23, "rank": 1},
    --     {"name": "cohort_month", "mean_abs_shap": 0.18, "rank": 2},
    --     {"name": "total_sessions", "mean_abs_shap": 0.12, "rank": 3}
    --   ]
    -- }
    -- Example local SHAP (for a single prediction):
    -- {
    --   "base_value": 0.15,
    --   "prediction": 0.82,
    --   "features": [
    --     {"name": "revenue_lag_7d", "value": -12400, "shap_value": 0.35},
    --     {"name": "cohort_month", "value": "2024-01", "shap_value": 0.22},
    --     {"name": "total_sessions", "value": 3, "shap_value": 0.10}
    --   ]
    -- }
    narrative       TEXT,                    -- LLM-generated plain-English explanation
    generated_by    VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_explanations_model ON model_explanations(model_id);
CREATE INDEX idx_explanations_type ON model_explanations(explanation_type);
```

## Predictions

```sql
CREATE TABLE prediction_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    model_id        UUID NOT NULL REFERENCES models(id),
    job_type        VARCHAR(50) NOT NULL,    -- batch, realtime
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    input_config    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"source": "dataset", "dataset_id": "...", "row_count": 10000}
    -- Example: {"source": "warehouse", "table": "analytics.new_customers", "query": "SELECT ..."}
    output_config   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"format": "csv", "path": "s3://predictions/job-123.csv"}
    -- Example: {"writeback": true, "target_table": "analytics.churn_scores", "warehouse": "snowflake"}
    result_summary  JSONB NOT NULL DEFAULT '{}',
    -- Example: {"row_count": 10000, "prediction_distribution": {"churn": 1500, "retain": 8500},
    --           "avg_confidence": 0.87, "low_confidence_count": 230}
    start_time      TIMESTAMPTZ,
    end_time        TIMESTAMPTZ,
    error_message   TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_prediction_jobs_project ON prediction_jobs(project_id);
CREATE INDEX idx_prediction_jobs_model ON prediction_jobs(model_id);
CREATE INDEX idx_prediction_jobs_status ON prediction_jobs(status);
```

## Drift Monitoring

```sql
CREATE TABLE drift_monitors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES models(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "check_interval_hours": 24,
    --   "drift_threshold": 0.1,
    --   "statistical_test": "psi",          -- psi, ks, js_divergence
    --   "performance_thresholds": {"rmse_max": 15.0, "mape_max": 0.12},
    --   "retrain_on_drift": true,
    --   "notification": {"channels": ["email", "slack"], "recipients": ["analyst@co.com"]}
    -- }
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE drift_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    drift_monitor_id UUID NOT NULL REFERENCES drift_monitors(id) ON DELETE CASCADE,
    check_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_drifted      BOOLEAN NOT NULL DEFAULT false,
    results         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "overall_drift_score": 0.23,
    --   "feature_drifts": [
    --     {"feature": "revenue", "psi": 0.23, "drifted": true, "baseline_mean": 45000, "current_mean": 38000},
    --     {"feature": "region", "psi": 0.01, "drifted": false}
    --   ],
    --   "performance": {"rmse": 14.2, "mape": 0.11, "baseline_rmse": 10.5, "degradation_pct": 35.2},
    --   "action_taken": "retrain_triggered",
    --   "retrain_result": {"new_model_version": 3, "new_rmse": 10.1, "improvement_pct": 28.9}
    -- }
    narrative       TEXT,                    -- LLM-generated: "Revenue distribution shifted 16% lower..."
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drift_events_monitor ON drift_events(drift_monitor_id);
CREATE INDEX idx_drift_events_time ON drift_events(check_time);
CREATE INDEX idx_drift_events_drifted ON drift_events(is_drifted) WHERE is_drifted = true;
```

## Copilot Conversations

```sql
CREATE TABLE copilot_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    session_type    VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    messages        JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"role": "user", "content": "I have SaaS subscription data...", "timestamp": "2026-05-11T10:00:00Z"},
    --   {"role": "assistant", "content": "Great! Let me analyze...", "timestamp": "2026-05-11T10:00:05Z",
    --    "model": "claude-opus-4-20250514", "tokens": {"prompt": 1200, "completion": 450}},
    --   {"role": "user", "content": "What features should I use?", "timestamp": "2026-05-11T10:01:00Z"},
    --   {"role": "assistant", "content": "Based on your domain...", "timestamp": "2026-05-11T10:01:08Z",
    --    "tool_calls": [{"name": "generate_features", "args": {"domain": "saas_churn"}}]}
    -- ]
    message_count   INTEGER NOT NULL DEFAULT 0,
    total_tokens    INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_copilot_project ON copilot_sessions(project_id);
CREATE INDEX idx_copilot_user ON copilot_sessions(user_id);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_time ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Example Queries

```sql
-- Find all experiments using LightGBM with learning_rate < 0.05
SELECT e.name, r.algorithm, r.hyperparameters, r.metrics
FROM runs r
JOIN experiments e ON r.experiment_id = e.id
WHERE r.algorithm = 'lightgbm'
  AND (r.hyperparameters->>'learning_rate')::float < 0.05;

-- Find datasets with potential target leakage
SELECT d.name, d.quality_report->'issues' AS issues
FROM datasets d
WHERE d.quality_report @> '{"issues": [{"type": "potential_leakage"}]}';

-- Find all features of type "rolling_window" across all projects
SELECT fs.name AS feature_set, f->>'name' AS feature_name, f->'params' AS params
FROM feature_sets fs,
     jsonb_array_elements(fs.features) AS f
WHERE f->>'type' = 'rolling_window';

-- Get drift history with narrative for a model
SELECT de.check_time,
       de.results->>'overall_drift_score' AS drift_score,
       de.is_drifted,
       de.narrative
FROM drift_events de
JOIN drift_monitors dm ON de.drift_monitor_id = dm.id
WHERE dm.model_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY de.check_time DESC
LIMIT 30;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | Same relational core as Model 1 |
| Projects | 1 | Config in JSONB |
| Datasets | 1 | Column profiling + quality in JSONB (vs. 3 tables in Model 1) |
| Feature Engineering | 1 | Features array in JSONB (vs. 2 tables in Model 1) |
| Experiments & Runs | 2 | Params/metrics/tags consolidated into JSONB (vs. 5 tables in Model 1) |
| Model Registry | 1 | Performance + export config in JSONB |
| Explainability | 1 | SHAP values + narrative in single table |
| Predictions | 1 | Input/output config + results in JSONB |
| Drift Monitoring | 2 | Monitor config + event results in JSONB |
| Copilot | 1 | Full conversation in JSONB array |
| Audit | 1 | Same as Model 1 |
| **Total** | **15** | ~50% fewer tables than normalised Model 1 |

---

## Key Design Decisions

1. **JSONB for algorithm-specific fields** — hyperparameters, metrics, feature transforms, drift results, and quality reports all vary by type. JSONB captures this variability without polymorphic table hierarchies or sparse columns.

2. **GIN indexes on JSONB columns** — `runs.metrics`, `runs.hyperparameters`, `datasets.schema_info`, and `feature_sets.features` all have GIN indexes enabling containment queries (`@>`) and path queries (`->>`) at index-backed speed.

3. **Conversations as JSONB arrays** — copilot message history stored as a JSONB array in `copilot_sessions.messages` rather than a separate messages table. This optimises for the dominant read pattern (load full conversation) at the cost of per-message queries.

4. **Denormalised experiment summary** — `experiments.summary` JSONB contains the best run info, avoiding a JOIN to runs for the most common dashboard query.

5. **Quality report as structured JSONB** — `datasets.quality_report` uses a consistent schema (`issues[]` with `type`, `severity`, `detail`, `recommendations[]`) but the issues vary by dataset, making JSONB more appropriate than a separate issues table.

6. **Features embedded in feature_sets** — rather than a separate `features` table, the feature definitions live as a JSONB array in `feature_sets`. This reflects the pattern that features are always loaded as a set, not individually queried.

7. **Drift event results in JSONB** — drift check results include per-feature PSI scores, performance comparisons, and retrain outcomes — all of which vary by check. JSONB captures this naturally while the relational `is_drifted` and `check_time` columns enable efficient filtering and time-range queries.

8. **Relational anchors for query performance** — despite heavy JSONB use, all frequently filtered/sorted fields (`status`, `problem_type`, `algorithm`, `stage`, `is_drifted`, `check_time`) remain relational columns with standard B-tree indexes.

9. **Schema documentation in code** — each JSONB column includes a detailed comment block showing the expected structure. In production, these schemas should also be enforced via application-layer validation (e.g., Zod, JSON Schema, or Pydantic) since the database cannot enforce JSONB structure.

10. **Progressive migration path** — if a JSONB field proves to need relational querying (e.g., individual feature lookups become common), it can be extracted to a separate table without rebuilding the entire schema. The hybrid model is designed to evolve.
