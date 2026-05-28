# Data Model Suggestion 4: Time-Series & Analytics-First with Feature Store

> Project: Predictive Analytics Workbench · Created: 2026-05-11

## Philosophy

This model is optimised for the workbench's core differentiator: time-series forecasting for business operations (demand planning, revenue forecasting, inventory optimisation). Rather than treating time-series as one of many problem types sharing a generic experiment schema, this architecture makes temporal data a first-class citizen with dedicated structures for time-series datasets, forecast results, feature stores, and drift monitoring — all leveraging PostgreSQL's table partitioning and TimescaleDB's hypertable capabilities for high-performance temporal queries.

The design draws from how purpose-built forecasting platforms (Amazon Forecast, Google Cloud AI Platform, Chronos) structure their data: datasets are explicitly temporal with defined granularity and time columns; features are stored in a dedicated feature store with point-in-time correctness guarantees; forecasts include prediction intervals (not just point estimates); and accuracy is tracked over time as actuals arrive. This is fundamentally different from classification/regression-focused platforms where predictions are single-shot outputs.

The feature store component is inspired by Feast's architecture: feature definitions registered in a central catalog, feature values materialised for both training (offline store) and serving (online store), and point-in-time joins that prevent future data leakage during training. This is critical for time-series problems where using future information during training silently inflates accuracy.

**Best for:** Platforms where time-series forecasting is the primary use case; teams needing a built-in feature store for temporal features; deployments requiring high-throughput time-series data ingestion and querying.

**Trade-offs:**
- (+) Purpose-built for time-series: temporal partitioning, forecast intervals, accuracy tracking over time
- (+) Built-in feature store with point-in-time correctness — prevents future data leakage
- (+) Efficient storage and querying of high-volume temporal data via hypertables/partitioning
- (+) Forecast accuracy backtesting is first-class: compare predictions against actuals as they arrive
- (+) Natural fit for the workbench's primary differentiator (no-code time-series for operations teams)
- (-) More complex schema for non-time-series problems (classification, regression)
- (-) Requires TimescaleDB extension or manual partitioning management
- (-) Feature store adds operational complexity (materialisation jobs, online/offline sync)
- (-) Less intuitive for developers unfamiliar with time-series database patterns
- (-) Over-engineered if the workbench evolves away from time-series focus

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Feast Feature Store | Feature catalog, feature views, point-in-time joins, online/offline stores |
| Chronos (Amazon) | Zero-shot forecast model references and context/prediction length parameters |
| TimescaleDB | Hypertable partitioning for time-series data, continuous aggregates for rollups |
| ISO 8601 | All temporal fields use `TIMESTAMPTZ`; duration fields use interval notation |
| ISO 4217 | Currency codes for revenue/financial forecasting (e.g., `USD`, `EUR`) |
| ISO 3166-1 | Region codes for multi-region forecasting segmentation |
| PMML TimeSeriesModel | Forecast output metadata aligns with PMML time-series model structure |
| MLflow | Experiment/run structure for non-time-series model types |

---

## Organisation & Users

```sql
-- Standard multi-tenant structure (same across all models)
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(320) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
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
```

## Projects

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    problem_type    VARCHAR(50) NOT NULL,    -- time_series, classification, regression
    domain_context  TEXT,                    -- NL description for LLM feature generation
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_projects_org ON projects(organisation_id);
CREATE INDEX idx_projects_type ON projects(problem_type);
```

## Time-Series Datasets

The core differentiator: datasets have explicit temporal metadata.

```sql
CREATE TABLE datasets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    source_type     VARCHAR(50) NOT NULL,
    source_config   JSONB NOT NULL DEFAULT '{}',
    -- Temporal metadata (first-class for time-series datasets)
    time_column     VARCHAR(255),            -- column name containing timestamps
    granularity     VARCHAR(50),             -- hourly, daily, weekly, monthly, quarterly, yearly
    time_zone       VARCHAR(50) DEFAULT 'UTC',
    start_time      TIMESTAMPTZ,             -- earliest timestamp in dataset
    end_time        TIMESTAMPTZ,             -- latest timestamp in dataset
    period_count    INTEGER,                 -- number of time periods
    -- Segmentation for grouped forecasting (e.g., forecast per product, per region)
    group_columns   TEXT[],                  -- {"product_id", "region"}
    group_count     INTEGER,                 -- number of unique groups
    -- General metadata
    row_count       BIGINT,
    column_count    INTEGER,
    file_path       TEXT,
    schema_info     JSONB NOT NULL DEFAULT '[]',
    quality_report  JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_datasets_project ON datasets(project_id);
CREATE INDEX idx_datasets_time_range ON datasets(start_time, end_time);
```

## Time-Series Data Store

Raw time-series data ingested into the platform, partitioned by time for efficient querying. This enables in-platform EDA and feature computation.

```sql
-- TimescaleDB hypertable for raw time-series data
-- If TimescaleDB is not available, use native PostgreSQL range partitioning
CREATE TABLE ts_data_points (
    id              BIGSERIAL,
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    timestamp       TIMESTAMPTZ NOT NULL,
    group_key       TEXT,                    -- composite group identifier (e.g., "product:SKU-123|region:US-WEST")
    target_value    DOUBLE PRECISION,        -- the value being forecasted
    dimensions      JSONB NOT NULL DEFAULT '{}',
    -- Example: {"product_id": "SKU-123", "region": "US-WEST", "channel": "online"}
    numeric_features JSONB NOT NULL DEFAULT '{}',
    -- Example: {"price": 29.99, "marketing_spend": 5000, "competitor_price": 32.99}
    categorical_features JSONB NOT NULL DEFAULT '{}',
    -- Example: {"season": "summer", "promotion_active": true, "holiday": "independence_day"}
    PRIMARY KEY (dataset_id, timestamp, id)
);

-- TimescaleDB: SELECT create_hypertable('ts_data_points', 'timestamp', partitioning_column => 'dataset_id');
-- Native PostgreSQL: PARTITION BY RANGE (timestamp)

CREATE INDEX idx_ts_data_dataset_time ON ts_data_points(dataset_id, timestamp);
CREATE INDEX idx_ts_data_group ON ts_data_points(dataset_id, group_key, timestamp);
```

## Feature Store

A dedicated feature store inspired by Feast's architecture, providing point-in-time correct feature serving.

```sql
-- Feature catalog: definitions of all features available in the platform
CREATE TABLE feature_catalog (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,   -- e.g., "revenue_rolling_30d_mean"
    display_name    VARCHAR(255),
    description     TEXT,
    feature_type    VARCHAR(50) NOT NULL,    -- lag, rolling_window, seasonal, calendar, external, raw
    data_type       VARCHAR(50) NOT NULL,    -- float64, int64, bool, category
    -- Transform definition
    source_columns  TEXT[] NOT NULL,          -- {"revenue", "date"}
    transform_sql   TEXT,                    -- SQL expression: "AVG(revenue) OVER (ORDER BY date ROWS 29 PRECEDING)"
    transform_params JSONB NOT NULL DEFAULT '{}',
    -- Example for rolling_window: {"window_size": 30, "aggregation": "mean"}
    -- Example for lag: {"lag_periods": 7}
    -- Example for seasonal: {"period": 12, "decomposition": "additive"}
    -- Example for calendar: {"features": ["day_of_week", "month", "is_holiday", "quarter"]}
    -- Lineage
    generation_mode VARCHAR(50) NOT NULL,    -- manual, auto_ml, llm_generated
    domain_rationale TEXT,                   -- LLM explanation: "Captures weekly revenue cycle..."
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, name)
);

CREATE INDEX idx_feature_catalog_project ON feature_catalog(project_id);
CREATE INDEX idx_feature_catalog_type ON feature_catalog(feature_type);

-- Feature view: a named collection of features materialised together (Feast concept)
CREATE TABLE feature_views (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    dataset_id      UUID NOT NULL REFERENCES datasets(id),
    entity_columns  TEXT[] NOT NULL,          -- {"customer_id"} or {"product_id", "region"}
    time_column     VARCHAR(255) NOT NULL,
    ttl_hours       INTEGER,                 -- how long before features go stale
    status          VARCHAR(50) NOT NULL DEFAULT 'created', -- created, materialising, ready, failed
    last_materialised_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, name)
);

CREATE TABLE feature_view_columns (
    feature_view_id UUID NOT NULL REFERENCES feature_views(id) ON DELETE CASCADE,
    feature_id      UUID NOT NULL REFERENCES feature_catalog(id),
    position        INTEGER NOT NULL,
    PRIMARY KEY (feature_view_id, feature_id)
);

-- Offline feature store: materialised feature values for training (partitioned by time)
CREATE TABLE feature_values_offline (
    id              BIGSERIAL,
    feature_view_id UUID NOT NULL REFERENCES feature_views(id) ON DELETE CASCADE,
    entity_key      TEXT NOT NULL,            -- composite entity key: "customer:C-1234"
    timestamp       TIMESTAMPTZ NOT NULL,
    feature_values  JSONB NOT NULL,           -- {"revenue_lag_7d": 45000, "rolling_30d_mean": 42300, "is_holiday": false}
    PRIMARY KEY (feature_view_id, entity_key, timestamp)
);

-- TimescaleDB: SELECT create_hypertable('feature_values_offline', 'timestamp');
-- Native: PARTITION BY RANGE (timestamp)

CREATE INDEX idx_fv_offline_entity_time ON feature_values_offline(feature_view_id, entity_key, timestamp DESC);

-- Online feature store: latest feature values for real-time serving
CREATE TABLE feature_values_online (
    feature_view_id UUID NOT NULL REFERENCES feature_views(id) ON DELETE CASCADE,
    entity_key      TEXT NOT NULL,
    feature_values  JSONB NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (feature_view_id, entity_key)
);
```

## Experiments & Runs

```sql
CREATE TABLE experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    dataset_id      UUID NOT NULL REFERENCES datasets(id),
    feature_view_id UUID REFERENCES feature_views(id),
    config          JSONB NOT NULL DEFAULT '{}',
    -- Time-series-specific config:
    -- {
    --   "target_column": "revenue",
    --   "forecast_horizon": 30,
    --   "context_length": 180,
    --   "evaluation_window": 30,         -- days of actuals withheld for backtesting
    --   "seasonality_mode": "auto",
    --   "algorithms": ["chronos", "deepar", "lightgbm_ts", "arima", "ets"],
    --   "primary_metric": "mape",
    --   "group_columns": ["product_id", "region"]
    -- }
    status          VARCHAR(50) NOT NULL DEFAULT 'created',
    summary         JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_experiments_project ON experiments(project_id);

CREATE TABLE runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    run_number      INTEGER NOT NULL,
    algorithm       VARCHAR(100) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',
    hyperparameters JSONB NOT NULL DEFAULT '{}',
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- Time-series-specific metrics:
    -- {"mape": 0.08, "rmse": 12400, "mae": 9800, "smape": 0.076,
    --  "coverage_80": 0.82, "coverage_95": 0.96,
    --  "per_group_mape": {"SKU-123": 0.05, "SKU-456": 0.12}}
    artifact_path   TEXT,
    start_time      TIMESTAMPTZ,
    end_time        TIMESTAMPTZ,
    duration_seconds DOUBLE PRECISION,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runs_experiment ON runs(experiment_id);
CREATE INDEX idx_runs_status ON runs(status);
```

## Models

```sql
CREATE TABLE models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL,
    run_id          UUID NOT NULL REFERENCES runs(id),
    stage           VARCHAR(50) NOT NULL DEFAULT 'none',
    artifact_path   TEXT NOT NULL,
    framework       VARCHAR(100),
    -- Forecasting-specific metadata
    forecast_config JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "forecast_horizon": 30,
    --   "context_length": 180,
    --   "granularity": "daily",
    --   "target_column": "revenue",
    --   "group_columns": ["product_id", "region"],
    --   "chronos_model": "chronos-t5-large",   -- if Chronos-based
    --   "supports_prediction_intervals": true,
    --   "interval_widths": [0.80, 0.95]
    -- }
    performance     JSONB NOT NULL DEFAULT '{}',
    export_config   JSONB NOT NULL DEFAULT '{}',
    promoted_by     UUID REFERENCES users(id),
    promoted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, name, version)
);

CREATE INDEX idx_models_project ON models(project_id);
CREATE INDEX idx_models_stage ON models(stage);
```

## Forecast Results

Purpose-built storage for time-series predictions with prediction intervals and backtesting.

```sql
-- Forecast jobs (batch prediction with time-series-specific output)
CREATE TABLE forecast_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    model_id        UUID NOT NULL REFERENCES models(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    forecast_start  TIMESTAMPTZ NOT NULL,    -- start of forecast period
    forecast_end    TIMESTAMPTZ NOT NULL,    -- end of forecast period
    granularity     VARCHAR(50) NOT NULL,    -- daily, weekly, monthly
    group_columns   TEXT[],                  -- forecast per group
    input_config    JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES users(id),
    start_time      TIMESTAMPTZ,
    end_time        TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_jobs_project ON forecast_jobs(project_id);
CREATE INDEX idx_forecast_jobs_model ON forecast_jobs(model_id);

-- Individual forecast points with prediction intervals
CREATE TABLE forecast_points (
    id              BIGSERIAL,
    forecast_job_id UUID NOT NULL REFERENCES forecast_jobs(id) ON DELETE CASCADE,
    timestamp       TIMESTAMPTZ NOT NULL,
    group_key       TEXT,                    -- "product:SKU-123|region:US-WEST"
    point_forecast  DOUBLE PRECISION NOT NULL,
    lower_80        DOUBLE PRECISION,        -- 80% prediction interval lower bound
    upper_80        DOUBLE PRECISION,        -- 80% prediction interval upper bound
    lower_95        DOUBLE PRECISION,        -- 95% prediction interval lower bound
    upper_95        DOUBLE PRECISION,        -- 95% prediction interval upper bound
    -- Actual value (populated when actuals arrive for backtesting)
    actual_value    DOUBLE PRECISION,
    absolute_error  DOUBLE PRECISION,        -- |actual - forecast|
    percentage_error DOUBLE PRECISION,       -- |actual - forecast| / |actual|
    actual_received_at TIMESTAMPTZ,
    PRIMARY KEY (forecast_job_id, timestamp, id)
);

-- TimescaleDB: SELECT create_hypertable('forecast_points', 'timestamp');
-- Native: PARTITION BY RANGE (timestamp)

CREATE INDEX idx_forecast_points_job_time ON forecast_points(forecast_job_id, timestamp);
CREATE INDEX idx_forecast_points_group ON forecast_points(forecast_job_id, group_key, timestamp);
CREATE INDEX idx_forecast_points_actuals ON forecast_points(forecast_job_id, timestamp)
    WHERE actual_value IS NOT NULL;
```

## Forecast Accuracy Tracking

Track how forecast accuracy evolves over time as actuals arrive.

```sql
CREATE TABLE forecast_accuracy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES models(id),
    forecast_job_id UUID NOT NULL REFERENCES forecast_jobs(id),
    evaluation_date TIMESTAMPTZ NOT NULL,    -- when this accuracy was calculated
    horizon_days    INTEGER NOT NULL,        -- how far ahead the forecast was (1-day, 7-day, 30-day)
    group_key       TEXT,                    -- NULL for overall, or specific group
    -- Accuracy metrics
    mape            DOUBLE PRECISION,
    rmse            DOUBLE PRECISION,
    mae             DOUBLE PRECISION,
    smape           DOUBLE PRECISION,
    coverage_80     DOUBLE PRECISION,        -- % of actuals within 80% interval
    coverage_95     DOUBLE PRECISION,        -- % of actuals within 95% interval
    point_count     INTEGER NOT NULL,        -- number of forecast points evaluated
    narrative       TEXT,                    -- LLM-generated accuracy summary
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_accuracy_model ON forecast_accuracy(model_id, evaluation_date);
CREATE INDEX idx_accuracy_horizon ON forecast_accuracy(model_id, horizon_days);
CREATE INDEX idx_accuracy_group ON forecast_accuracy(model_id, group_key) WHERE group_key IS NOT NULL;
```

## Drift Monitoring (Time-Series Specific)

```sql
CREATE TABLE drift_monitors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES models(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- Time-series-specific drift config:
    -- {
    --   "check_interval_hours": 24,
    --   "accuracy_thresholds": {"mape_max": 0.15, "coverage_80_min": 0.70},
    --   "distribution_tests": ["psi", "ks"],
    --   "seasonality_check": true,
    --   "structural_break_detection": true,
    --   "retrain_on_drift": true,
    --   "notification": {"channels": ["email"], "recipients": ["ops@co.com"]}
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
    drift_type      VARCHAR(50),             -- data_drift, concept_drift, seasonal_shift, structural_break
    results         JSONB NOT NULL DEFAULT '{}',
    -- Time-series-specific results:
    -- {
    --   "accuracy_degradation": {"current_mape": 0.18, "baseline_mape": 0.08, "degradation_pct": 125},
    --   "distribution_shift": {"target_psi": 0.31, "drifted_features": ["price", "marketing_spend"]},
    --   "seasonal_anomaly": {"expected_pattern": "monthly_cycle", "detected": "trend_break_march"},
    --   "structural_break": {"break_point": "2026-03-15", "test": "chow", "p_value": 0.002}
    -- }
    action_taken    VARCHAR(50),
    narrative       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drift_events_monitor ON drift_events(drift_monitor_id, check_time);
CREATE INDEX idx_drift_events_type ON drift_events(drift_type);
```

## Explainability

```sql
CREATE TABLE model_explanations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES models(id) ON DELETE CASCADE,
    explanation_type VARCHAR(50) NOT NULL,    -- global_shap, temporal_decomposition, feature_importance
    content         JSONB NOT NULL DEFAULT '{}',
    -- Example temporal_decomposition:
    -- {
    --   "trend": {"direction": "upward", "slope": 1240, "confidence": 0.92},
    --   "seasonality": {"period": 12, "amplitude": 8500, "pattern": "peaks in Q4"},
    --   "residual_std": 3200,
    --   "decomposition_method": "STL"
    -- }
    -- Example feature_importance:
    -- {
    --   "features": [
    --     {"name": "revenue_lag_7d", "importance": 0.34, "rank": 1},
    --     {"name": "day_of_week", "importance": 0.21, "rank": 2},
    --     {"name": "marketing_spend", "importance": 0.15, "rank": 3}
    --   ]
    -- }
    narrative       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_explanations_model ON model_explanations(model_id);
```

## Copilot & Audit

```sql
CREATE TABLE copilot_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    session_type    VARCHAR(50) NOT NULL,
    messages        JSONB NOT NULL DEFAULT '[]',
    message_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_time ON audit_log(organisation_id, created_at);
```

---

## Example Queries

```sql
-- Point-in-time feature retrieval for training (prevents future data leakage)
-- "Give me feature values as of each training timestamp"
SELECT
    fv.entity_key,
    fv.timestamp,
    fv.feature_values
FROM feature_values_offline fv
WHERE fv.feature_view_id = '550e8400-e29b-41d4-a716-446655440000'
  AND fv.timestamp <= '2026-01-01'  -- training cutoff date
ORDER BY fv.entity_key, fv.timestamp;

-- Forecast accuracy by horizon (1-day vs 7-day vs 30-day)
SELECT
    horizon_days,
    AVG(mape) AS avg_mape,
    AVG(rmse) AS avg_rmse,
    AVG(coverage_80) AS avg_coverage_80
FROM forecast_accuracy
WHERE model_id = '660e8400-e29b-41d4-a716-446655440001'
GROUP BY horizon_days
ORDER BY horizon_days;

-- Compare forecasts vs actuals for backtesting
SELECT
    fp.timestamp,
    fp.group_key,
    fp.point_forecast,
    fp.actual_value,
    fp.percentage_error,
    fp.lower_80,
    fp.upper_80,
    CASE WHEN fp.actual_value BETWEEN fp.lower_80 AND fp.upper_80
         THEN true ELSE false END AS within_80_interval
FROM forecast_points fp
WHERE fp.forecast_job_id = '770e8400-e29b-41d4-a716-446655440002'
  AND fp.actual_value IS NOT NULL
ORDER BY fp.timestamp;

-- Time-series continuous aggregate (TimescaleDB) for dashboard
-- CREATE MATERIALIZED VIEW daily_accuracy_summary
-- WITH (timescaledb.continuous) AS
-- SELECT
--     time_bucket('1 day', fa.evaluation_date) AS day,
--     fa.model_id,
--     AVG(fa.mape) AS avg_mape,
--     MIN(fa.mape) AS best_mape,
--     MAX(fa.mape) AS worst_mape
-- FROM forecast_accuracy fa
-- GROUP BY day, fa.model_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | Standard multi-tenant |
| Projects | 1 | |
| Datasets | 1 | With explicit temporal metadata |
| Time-Series Data | 1 | Hypertable/partitioned for raw data ingestion |
| Feature Store | 4 | Catalog + views + offline store + online store |
| Experiments & Runs | 2 | JSONB for params/metrics |
| Models | 1 | With forecast-specific config |
| Forecast Results | 2 | Jobs + point-level forecasts with intervals |
| Forecast Accuracy | 1 | Backtesting results by horizon |
| Drift Monitoring | 2 | Time-series-specific drift types |
| Explainability | 1 | Temporal decomposition + feature importance |
| Copilot & Audit | 2 | |
| **Total** | **21** | Plus hypertable partitions |

---

## Key Design Decisions

1. **Temporal metadata as first-class columns** — `datasets.time_column`, `granularity`, `start_time`, `end_time`, and `group_columns` are relational columns, not buried in JSONB. This enables efficient filtering (e.g., "find all daily datasets spanning 2025") and validates that time-series concerns are central to the schema.

2. **Dedicated `ts_data_points` hypertable** — raw time-series data is stored in a partitioned table optimised for time-range queries and group-level filtering. This enables in-platform EDA ("show me revenue by region for the last 6 months") without round-tripping to the source warehouse.

3. **Feature store with offline/online split** — `feature_values_offline` is a hypertable storing historical feature values for training with point-in-time correctness. `feature_values_online` stores only the latest feature values for real-time serving. This mirrors Feast's architecture and prevents the common time-series bug of future data leakage.

4. **Prediction intervals as columns** — `forecast_points` includes `lower_80`, `upper_80`, `lower_95`, `upper_95` as explicit columns rather than a JSONB array. This enables efficient queries like "find all forecast points where the actual fell outside the 80% interval" without JSONB extraction.

5. **Forecast accuracy tracking by horizon** — `forecast_accuracy` separately tracks 1-day, 7-day, and 30-day accuracy because forecast quality typically degrades with horizon length. This enables dashboard views showing "our 7-day forecast has 8% MAPE but 30-day is 15%."

6. **Actual value backfill** — `forecast_points.actual_value` starts NULL and is populated as actuals arrive. The `actual_received_at` timestamp records when backtesting data became available, enabling analysis of how quickly forecast accuracy can be evaluated.

7. **Time-series-specific drift types** — `drift_events.drift_type` distinguishes between `data_drift` (input distribution shift), `concept_drift` (target relationship change), `seasonal_shift` (seasonality pattern change), and `structural_break` (sudden regime change). Each requires different remediation.

8. **Temporal decomposition in explainability** — `model_explanations` supports `temporal_decomposition` type with trend, seasonality, and residual components — standard time-series analysis that doesn't apply to classification/regression models.

9. **Group-level forecasting** — `datasets.group_columns`, `ts_data_points.group_key`, `forecast_points.group_key`, and `forecast_accuracy.group_key` all support hierarchical forecasting (e.g., forecast revenue per product per region) as a first-class pattern.

10. **TimescaleDB compatibility** — the schema is designed to work with either TimescaleDB hypertables (preferred for time-series workloads) or native PostgreSQL range partitioning. Comments indicate where `create_hypertable()` and continuous aggregates would be applied.
