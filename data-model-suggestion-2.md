# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Predictive Analytics Workbench · Created: 2026-05-11

## Philosophy

This model treats every state change in the ML workflow as an immutable event appended to a central event store. The current state of any entity — a project, dataset, experiment, model, or prediction — is derived by replaying its event stream. Read-optimised materialised views (projections) serve the UI and API, while the event store serves as the authoritative source of truth.

This architecture is inspired by how financial trading systems, healthcare record systems, and regulatory-grade audit platforms operate. In the ML context, event sourcing provides exact reproducibility: you can reconstruct the state of any model, experiment, or dataset at any point in time. "What was the production model at 3pm on March 15th?" is a trivial query. "What sequence of changes led to this drift event?" is a replay of the event stream. This is particularly powerful for regulated industries (finance, healthcare, insurance) where audit trails for ML model decisions are a legal requirement.

The CQRS (Command Query Responsibility Segregation) pattern separates write operations (commands that produce events) from read operations (queries against materialised views). This allows the write side to be optimised for append-only durability while the read side can be denormalised for fast UI rendering — each projection is purpose-built for a specific read path.

**Best for:** Regulated industries requiring complete ML audit trails; platforms where temporal queries ("what was true on date X?") are essential; teams building AI-powered analytics on model change patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change to every entity is permanently recorded
- (+) Temporal queries are first-class: reconstruct any entity state at any point in time
- (+) Perfect reproducibility: replay the event stream to recreate any past experiment state
- (+) Write-side and read-side scale independently
- (+) Natural fit for ML workflow events (training started, metric logged, model promoted, drift detected)
- (-) Higher implementation complexity: event handlers, projections, eventual consistency
- (-) Eventual consistency between event store and read models requires careful handling
- (-) Read model rebuilds can be slow for entities with long event histories
- (-) Developers must think in terms of events rather than CRUD — steeper learning curve
- (-) Storage grows indefinitely (append-only); requires snapshot strategies for long-lived entities

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CQRS Pattern (Microsoft) | Separate command and query models; events are the bridge |
| Event Sourcing Pattern | Append-only event store as single source of truth |
| CloudEvents v1.0 (CNCF) | Event envelope schema for standardised event metadata |
| MLflow Concepts | Event types mirror MLflow's experiment/run/metric lifecycle but as immutable events |
| OCSF (Open Cybersecurity Schema Framework) | Audit event structure follows OCSF patterns for structured, queryable logs |
| PMML 4.4.1 / ONNX | Model export events capture format-specific metadata |
| ISO 8601 | All event timestamps use `TIMESTAMPTZ` |

---

## Event Store (Core)

The event store is the single source of truth. Every state change in the platform produces an event.

```sql
-- Core event store table — append-only, never updated or deleted
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,           -- the aggregate/entity this event belongs to
    stream_type     VARCHAR(50) NOT NULL,    -- project, dataset, experiment, run, model, prediction_job, drift_monitor
    event_type      VARCHAR(100) NOT NULL,   -- see Event Type Catalogue below
    version         INTEGER NOT NULL,        -- monotonically increasing per stream_id (optimistic concurrency)
    data            JSONB NOT NULL,          -- event payload (structured per event_type)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata follows CloudEvents v1.0 envelope:
    -- {"source": "api/v1", "subject": "user:abc123", "correlation_id": "req-xyz", "causation_id": "evt-prev"}
    organisation_id UUID NOT NULL,
    user_id         UUID,                    -- NULL for system-generated events
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, version)              -- optimistic concurrency control
);

-- Primary query patterns: replay a stream, query by type, time-range scans
CREATE INDEX idx_events_stream ON events(stream_id, version);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_org_time ON events(organisation_id, created_at);
CREATE INDEX idx_events_time ON events(created_at);

-- Partition by month for efficient time-range queries and archival
-- In production, use declarative partitioning:
-- CREATE TABLE events (...) PARTITION BY RANGE (created_at);
-- CREATE TABLE events_2026_01 PARTITION OF events FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

## Event Type Catalogue

```sql
-- Reference table documenting all valid event types and their expected data schemas
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    data_schema     JSONB NOT NULL,          -- JSON Schema for the event's data field
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payloads:
--
-- stream_type: project
--   project.created       {"name": "Churn Prediction", "problem_type": "classification", "domain_context": "SaaS subscriptions"}
--   project.updated       {"changes": {"name": {"old": "Churn v1", "new": "Churn Prediction"}}}
--   project.archived      {"reason": "Superseded by project XYZ"}
--
-- stream_type: dataset
--   dataset.uploaded      {"name": "customers_q1.csv", "row_count": 50000, "column_count": 24, "file_path": "s3://..."}
--   dataset.quality_checked {"null_rates": {...}, "outlier_count": 14, "warnings": ["class_imbalance"]}
--   dataset.columns_profiled {"columns": [{"name": "revenue", "dtype": "float64", "null_rate": 0.02}]}
--
-- stream_type: experiment
--   experiment.created    {"dataset_id": "...", "target_column": "churn", "problem_type": "classification"}
--   experiment.started    {"run_count": 8, "algorithms": ["lightgbm", "xgboost", "catboost"]}
--   experiment.completed  {"best_run_id": "...", "best_metric": {"accuracy": 0.94}}
--
-- stream_type: run
--   run.started           {"algorithm": "lightgbm", "hyperparameters": {"n_estimators": 500}}
--   run.metric_logged     {"key": "accuracy", "value": 0.94, "step": 0}
--   run.param_logged      {"key": "learning_rate", "value": "0.01"}
--   run.completed         {"duration_seconds": 142.5, "artifact_path": "s3://..."}
--   run.failed            {"error": "Out of memory", "duration_seconds": 45.2}
--
-- stream_type: model
--   model.registered      {"name": "churn_predictor", "version": 1, "run_id": "..."}
--   model.promoted        {"from_stage": "staging", "to_stage": "production", "promoted_by": "user:..."}
--   model.exported        {"format": "onnx", "artifact_path": "s3://..."}
--   model.archived        {"reason": "Superseded by v3"}
--
-- stream_type: prediction_job
--   prediction.requested  {"model_version_id": "...", "input_row_count": 10000}
--   prediction.completed  {"output_row_count": 10000, "output_path": "s3://..."}
--   prediction.failed     {"error": "Input schema mismatch"}
--
-- stream_type: drift_monitor
--   drift.check_completed {"overall_score": 0.23, "is_drifted": true, "feature_drifts": [...]}
--   drift.alert_sent      {"channel": "email", "recipients": ["analyst@co.com"]}
--   drift.retrain_triggered {"old_model_version": 2, "reason": "PSI exceeded threshold"}
--   drift.retrain_completed {"new_model_version": 3, "improvement_pct": 28.9, "auto_promoted": false}
--
-- stream_type: copilot
--   copilot.session_started {"session_type": "onboarding", "project_id": "..."}
--   copilot.message_sent    {"role": "user", "content": "What model should I use for churn?"}
--   copilot.message_received {"role": "assistant", "content": "For churn prediction...", "model": "claude-opus-4-20250514"}
--   copilot.session_ended   {"message_count": 12, "outcome": "model_trained"}
```

## Snapshots (Performance Optimisation)

For entities with long event histories, periodic snapshots avoid replaying hundreds of events.

```sql
CREATE TABLE snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    version         INTEGER NOT NULL,        -- the event version this snapshot represents
    state           JSONB NOT NULL,          -- the full entity state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, version)
);

CREATE INDEX idx_snapshots_stream ON snapshots(stream_id, version DESC);
```

## Read Models (Materialised Projections)

These tables are **derived** from the event store. They can be rebuilt from scratch by replaying events. They exist purely for query performance.

### Organisation & User Projection

```sql
CREATE TABLE v_organisations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    member_count    INTEGER NOT NULL DEFAULT 0,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE v_users (
    id              UUID PRIMARY KEY,
    email           VARCHAR(320) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE v_organisation_members (
    organisation_id UUID NOT NULL,
    user_id         UUID NOT NULL,
    role            VARCHAR(50) NOT NULL,
    joined_at       TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (organisation_id, user_id)
);
```

### Project Dashboard Projection

```sql
CREATE TABLE v_projects (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    organisation_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    problem_type    VARCHAR(50) NOT NULL,
    domain_context  TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    -- Denormalised counts for dashboard display
    dataset_count   INTEGER NOT NULL DEFAULT 0,
    experiment_count INTEGER NOT NULL DEFAULT 0,
    model_count     INTEGER NOT NULL DEFAULT 0,
    prediction_count INTEGER NOT NULL DEFAULT 0,
    -- Latest model info (denormalised for quick display)
    latest_model_name VARCHAR(255),
    latest_model_version INTEGER,
    latest_model_stage VARCHAR(50),
    latest_model_metric_key VARCHAR(100),
    latest_model_metric_value DOUBLE PRECISION,
    created_by      UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_projects_org ON v_projects(organisation_id);
CREATE INDEX idx_v_projects_workspace ON v_projects(workspace_id);
```

### Experiment Leaderboard Projection

```sql
CREATE TABLE v_experiment_leaderboard (
    experiment_id   UUID NOT NULL,
    run_id          UUID NOT NULL,
    algorithm       VARCHAR(100) NOT NULL,
    rank            INTEGER NOT NULL,
    -- Denormalised key metrics for sorting/filtering
    accuracy        DOUBLE PRECISION,
    precision_score DOUBLE PRECISION,
    recall          DOUBLE PRECISION,
    f1_score        DOUBLE PRECISION,
    auc_roc         DOUBLE PRECISION,
    rmse            DOUBLE PRECISION,
    mae             DOUBLE PRECISION,
    mape            DOUBLE PRECISION,
    duration_seconds DOUBLE PRECISION,
    is_best         BOOLEAN NOT NULL DEFAULT false,
    selection_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (experiment_id, run_id)
);

CREATE INDEX idx_v_leaderboard_best ON v_experiment_leaderboard(experiment_id) WHERE is_best = true;
```

### Model Registry Projection

```sql
CREATE TABLE v_model_registry (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    model_name      VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL,
    stage           VARCHAR(50) NOT NULL,
    algorithm       VARCHAR(100),
    primary_metric  VARCHAR(100),
    primary_score   DOUBLE PRECISION,
    export_format   VARCHAR(50),
    artifact_path   TEXT,
    promoted_by     UUID,
    promoted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_models_project ON v_model_registry(project_id);
CREATE INDEX idx_v_models_stage ON v_model_registry(stage);
```

### Drift Timeline Projection

```sql
CREATE TABLE v_drift_timeline (
    id              UUID PRIMARY KEY,
    model_version_id UUID NOT NULL,
    model_name      VARCHAR(255) NOT NULL,
    check_time      TIMESTAMPTZ NOT NULL,
    overall_drift_score DOUBLE PRECISION NOT NULL,
    is_drifted      BOOLEAN NOT NULL,
    top_drifted_features JSONB,   -- [{"feature": "revenue", "psi": 0.23}]
    action_taken    VARCHAR(50),
    narrative       TEXT,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_drift_model ON v_drift_timeline(model_version_id, check_time);
CREATE INDEX idx_v_drift_drifted ON v_drift_timeline(is_drifted) WHERE is_drifted = true;
```

### Prediction History Projection

```sql
CREATE TABLE v_prediction_history (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    model_name      VARCHAR(255) NOT NULL,
    model_version   INTEGER NOT NULL,
    job_type        VARCHAR(50) NOT NULL,
    input_row_count BIGINT,
    output_row_count BIGINT,
    status          VARCHAR(50) NOT NULL,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_predictions_project ON v_prediction_history(project_id, created_at);
```

## Projection Tracking

```sql
-- Tracks which event each projection has processed, enabling incremental rebuilds
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,  -- v_projects, v_experiment_leaderboard, etc.
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    last_rebuilt_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example: Rebuilding Entity State from Events

```sql
-- Reconstruct the state of a project as of a specific date
-- This is a temporal query — impossible with traditional CRUD
SELECT
    e.stream_id AS project_id,
    e.event_type,
    e.data,
    e.created_at
FROM events e
WHERE e.stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND e.stream_type = 'project'
  AND e.created_at <= '2026-03-15 15:00:00+00'
ORDER BY e.version ASC;

-- Find all models that were in production at a specific point in time
SELECT DISTINCT ON (e.stream_id)
    e.stream_id AS model_id,
    e.data->>'to_stage' AS stage,
    e.data->>'name' AS model_name,
    e.created_at AS promoted_at
FROM events e
WHERE e.event_type = 'model.promoted'
  AND e.data->>'to_stage' = 'production'
  AND e.created_at <= '2026-03-15 15:00:00+00'
ORDER BY e.stream_id, e.version DESC;

-- Audit trail: what happened to this experiment?
SELECT
    e.event_type,
    e.data,
    e.metadata->>'subject' AS actor,
    e.created_at
FROM events e
WHERE e.stream_id = '660e8400-e29b-41d4-a716-446655440001'
  AND e.stream_type = 'experiment'
ORDER BY e.version ASC;
```

## Example: Event Handler (Application Layer)

```sql
-- When a 'run.completed' event is written, the projection updater:
-- 1. Reads the event data
-- 2. Updates v_experiment_leaderboard with the new run's metrics
-- 3. Recalculates rank and is_best for all runs in the experiment
-- 4. Updates v_projects.experiment_count and latest_model info
-- 5. Updates projection_checkpoints

-- This is typically done by an async event processor, not in-database triggers,
-- to keep the write path fast and the read model update eventually consistent.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only events table (partitioned by month) |
| Event Registry | 1 | Documents all valid event types with JSON Schema |
| Snapshots | 1 | Performance optimisation for long streams |
| Projection: Org & Users | 3 | Materialised from org/user events |
| Projection: Projects | 1 | Dashboard-optimised with denormalised counts |
| Projection: Leaderboard | 1 | Experiment results ranked by metrics |
| Projection: Models | 1 | Registry with lifecycle stage |
| Projection: Drift | 1 | Timeline of drift checks |
| Projection: Predictions | 1 | Prediction job history |
| Projection Tracking | 1 | Checkpoint for incremental rebuilds |
| **Total** | **12** | Plus N partitions of events table |

---

## Key Design Decisions

1. **Single event store table** — all events for all entity types in one partitioned table. This simplifies event processing, enables cross-entity correlation queries, and makes the write path trivially simple (one INSERT).

2. **CloudEvents v1.0 metadata envelope** — the `metadata` JSONB field follows CNCF CloudEvents conventions (`source`, `subject`, `correlation_id`, `causation_id`), enabling future integration with event-driven architectures (Kafka, EventBridge).

3. **Optimistic concurrency via stream version** — the `UNIQUE (stream_id, version)` constraint prevents conflicting concurrent writes to the same entity. The application reads the current version, increments it, and writes — if another write landed first, the INSERT fails and the command is retried.

4. **Projections are disposable** — every `v_*` table can be dropped and rebuilt from the event store. This means projection schemas can evolve freely without data migrations — just change the projection builder and rebuild.

5. **Monthly partitioning** — events are partitioned by `created_at` month, enabling efficient time-range queries and zero-downtime archival of old partitions to cold storage.

6. **Snapshots for performance** — entities with hundreds of events (e.g., a long-running experiment with many metric logs) use periodic snapshots to avoid replaying the full stream on every read.

7. **Separation of write and read concerns** — the event store handles durability and ordering; projections handle query performance. Each projection is purpose-built for a specific UI view (dashboard, leaderboard, drift timeline).

8. **Event type registry** — the `event_type_registry` table with JSON Schema definitions acts as a contract between the write side (commands that produce events) and the read side (projections that consume events). Schema validation can be enforced at the application layer.

9. **Natural audit trail** — every action in the system is permanently recorded as an event. Compliance reports are simply filtered queries against the event store. No separate audit log table is needed because the event store IS the audit log.

10. **Temporal querying as a first-class feature** — the ability to reconstruct any entity's state at any historical point in time is a natural consequence of the event-sourced design. This is critical for ML reproducibility and regulatory compliance.
