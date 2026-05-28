# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Customer Cohort Analytics · Created: 2026-05-11

## Philosophy

This model treats every piece of data as an immutable event appended to a single, ordered event store. There are no UPDATE or DELETE operations on the source-of-truth tables. Instead, the current state of any entity (a user's profile, a cohort's membership, a churn score) is derived by replaying or folding events from the store. Read-optimized projections (materialized views) serve the dashboard, API, and ML pipeline, while the event store provides a complete, tamper-proof audit trail.

This is the architecture behind systems like Apache Kafka-backed analytics pipelines, EventStoreDB-powered domains, and the internal event models of platforms like Mixpanel and Amplitude (which are fundamentally append-only event stores with query layers on top). For a customer cohort analytics platform, event sourcing is a natural fit: the raw material of the product IS events. Rather than ingesting events and then transforming them into mutable relational rows, this model keeps the events as the canonical representation and builds everything else as derived views.

The CQRS (Command Query Responsibility Segregation) pattern complements event sourcing by separating the write path (event ingestion) from the read path (analytics queries). Write-side tables are append-only and optimized for high-throughput ingestion. Read-side tables are denormalized, pre-aggregated, and optimized for the specific query patterns the dashboard needs. This separation allows each side to scale independently.

**Best for:** Teams that need a complete audit trail, temporal queries ("what was the cohort composition on March 15th?"), high-throughput event ingestion, and the ability to retroactively recompute analytics when business logic changes.

**Trade-offs:**
- (+) Complete, immutable audit trail of every state change
- (+) Temporal queries are trivial: replay events up to any point in time
- (+) Retroactive recomputation: change a cohort definition and replay history to see what would have happened
- (+) Natural fit for event-driven analytics: the domain IS events
- (+) Write path scales independently from read path
- (-) Increased storage: events are never deleted, read models duplicate data
- (-) Eventual consistency: read models may lag behind the event store
- (-) Higher complexity: developers must understand event replay, projection building, and idempotency
- (-) Debugging requires understanding the event sequence, not just current state
- (-) Schema evolution of events requires careful versioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment Spec (`.track()`, `.identify()`, `.page()`, `.group()`) | Maps directly to event types in the event store; each Segment call becomes one immutable event |
| CloudEvents v1.0 (CNCF) | Event envelope follows CloudEvents spec: `id`, `source`, `type`, `time`, `data` |
| ISO 8601 | All timestamps as `TIMESTAMPTZ`; event ordering uses microsecond precision |
| ISO 3166-1/2 | Geography fields in user profile projections |
| GDPR / CCPA | Crypto-shredding pattern: user PII encrypted with per-user key; key deletion renders events unreadable without data deletion |
| SHAP | Prediction explanation events stored in the event store alongside predictions |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- EVENT STORE — the single source of truth
-- ============================================================
-- All data enters the system as immutable events. Nothing is updated or deleted.

CREATE TABLE event_store (
    -- Event identity
    event_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    sequence_num    BIGSERIAL,                  -- global ordering within tenant
    
    -- CloudEvents-aligned envelope
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(255) NOT NULL,       -- hierarchical: tracking.track, tracking.identify, 
                                                 -- tracking.page, tracking.group,
                                                 -- system.cohort_computed, system.prediction_generated,
                                                 -- system.user_merged, system.feature_computed
    event_source    VARCHAR(255) NOT NULL,       -- sdk/javascript, sdk/ios, api/import, system/ml_pipeline
    event_time      TIMESTAMPTZ NOT NULL,        -- when the event occurred (client time)
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when the server received it
    
    -- Entity references
    user_id         UUID,                        -- tracked end-user (resolved after identity merge)
    anonymous_id    VARCHAR(255),                -- pre-identification anonymous ID
    account_id      UUID,                        -- B2B account (from group calls)
    
    -- Event payload (the actual data)
    event_name      VARCHAR(255),               -- for tracking.track events: "Purchase", "Page View"
    data            JSONB NOT NULL DEFAULT '{}', -- event-specific payload
    -- Example data for tracking.track:
    -- {"product_id": "sku-123", "price": 29.99, "currency": "USD", "category": "shoes"}
    -- Example data for tracking.identify:
    -- {"email": "user@example.com", "name": "Jane Doe", "plan": "pro", "$set": {"plan": "pro"}}
    -- Example data for system.prediction_generated:
    -- {"model_id": "...", "churn_30d": 0.82, "risk_tier": "critical", 
    --  "shap": [{"feature": "days_since_last_login", "value": 45, "shap_value": 0.31, 
    --            "plain_english": "No login for 45 days (cohort avg: 3 days)"}]}
    
    -- Context (Segment context object)
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"ip": "1.2.3.4", "userAgent": "...", "locale": "en-US", 
    --           "page": {"url": "...", "title": "..."}, "campaign": {"source": "google"}}
    
    -- Crypto-shredding support (GDPR)
    encryption_key_id VARCHAR(100),             -- references key in external KMS; NULL = plaintext
    
    PRIMARY KEY (event_id, event_time)          -- composite PK for partitioning
) PARTITION BY RANGE (event_time);

-- Monthly partitions
CREATE TABLE event_store_2026_01 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE event_store_2026_02 PARTITION OF event_store
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... automated partition creation

-- Write-path indexes (minimal — optimize for append throughput)
CREATE INDEX idx_es_tenant_seq ON event_store (tenant_id, sequence_num);
CREATE INDEX idx_es_tenant_user_time ON event_store (tenant_id, user_id, event_time);
CREATE INDEX idx_es_tenant_type_time ON event_store (tenant_id, event_type, event_time);
CREATE INDEX idx_es_tenant_name_time ON event_store (tenant_id, event_name, event_time)
    WHERE event_name IS NOT NULL;
```

## Event Type Registry

```sql
-- ============================================================
-- EVENT TYPE REGISTRY
-- ============================================================
-- Tracks known event types and their schemas for validation

CREATE TABLE event_type_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    event_name      VARCHAR(255),               -- NULL for system events
    description     TEXT,
    json_schema     JSONB,                       -- optional JSON Schema for data validation
    schema_version  INTEGER NOT NULL DEFAULT 1,
    sample_count    BIGINT NOT NULL DEFAULT 0,   -- number of events of this type received
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, event_type, event_name)
);
```

## Read-Side Projections (Query Models)

```sql
-- ============================================================
-- PROJECTION: Current User State
-- ============================================================
-- Materialized by folding all identify/track events for each user

CREATE TABLE proj_users (
    user_id         UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    external_id     VARCHAR(255),
    anonymous_ids   TEXT[],                     -- all anonymous IDs merged into this user
    email           VARCHAR(255),
    display_name    VARCHAR(255),
    current_properties JSONB NOT NULL DEFAULT '{}',  -- latest merged user properties
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    total_events    BIGINT NOT NULL DEFAULT 0,
    country_code    CHAR(2),
    timezone        VARCHAR(50),
    signup_source   VARCHAR(100),
    plan_tier       VARCHAR(50),
    -- Projection metadata
    last_event_seq  BIGINT NOT NULL,            -- sequence_num of last processed event
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, user_id)
);

CREATE INDEX idx_proj_users_external ON proj_users (tenant_id, external_id)
    WHERE external_id IS NOT NULL;
CREATE INDEX idx_proj_users_first_seen ON proj_users (tenant_id, first_seen_at);

-- ============================================================
-- PROJECTION: Current Account State (B2B)
-- ============================================================

CREATE TABLE proj_accounts (
    account_id      UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    external_id     VARCHAR(255) NOT NULL,
    name            VARCHAR(255),
    current_traits  JSONB NOT NULL DEFAULT '{}',
    member_user_ids UUID[],                     -- array of user_ids in this account
    member_count    INTEGER NOT NULL DEFAULT 0,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, account_id)
);

-- ============================================================
-- PROJECTION: Daily Activity Rollup
-- ============================================================
-- Rebuilt from event_store by aggregating events per user per day

CREATE TABLE proj_daily_activity (
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    activity_date   DATE NOT NULL,
    event_count     INTEGER NOT NULL DEFAULT 0,
    session_count   INTEGER NOT NULL DEFAULT 0,
    distinct_event_names INTEGER NOT NULL DEFAULT 0,
    first_event_at  TIMESTAMPTZ,
    last_event_at   TIMESTAMPTZ,
    event_names     TEXT[],                     -- distinct event names that day
    last_event_seq  BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, user_id, activity_date)
);

CREATE INDEX idx_proj_daily_tenant_date ON proj_daily_activity (tenant_id, activity_date);

-- ============================================================
-- PROJECTION: Retention Matrix
-- ============================================================
-- Precomputed retention percentages per cohort period

CREATE TABLE proj_retention (
    tenant_id       UUID NOT NULL,
    report_id       UUID NOT NULL,
    cohort_period   DATE NOT NULL,
    period_offset   INTEGER NOT NULL,           -- 0, 1, 2, ... N
    granularity     VARCHAR(10) NOT NULL,        -- day, week, month
    cohort_size     INTEGER NOT NULL,
    retained_count  INTEGER NOT NULL,
    retention_rate  NUMERIC(5, 4) NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, report_id, cohort_period, period_offset)
);

-- ============================================================
-- PROJECTION: Funnel Progress
-- ============================================================

CREATE TABLE proj_funnel_progress (
    tenant_id       UUID NOT NULL,
    funnel_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    entry_time      TIMESTAMPTZ NOT NULL,       -- when user entered step 1
    max_step        INTEGER NOT NULL,           -- furthest step reached
    completed       BOOLEAN NOT NULL DEFAULT false,
    step_times      TIMESTAMPTZ[],              -- timestamp of reaching each step
    last_event_seq  BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, funnel_id, user_id, entry_time)
);

CREATE INDEX idx_proj_funnel_tenant ON proj_funnel_progress (tenant_id, funnel_id);
```

## Cohort Management (Hybrid: Config + Projection)

```sql
-- ============================================================
-- COHORT DEFINITIONS (configuration, not event-sourced)
-- ============================================================

CREATE TABLE cohort_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) NOT NULL,       -- behavioral, temporal, demographic, predictive, ai_discovered
    is_dynamic      BOOLEAN NOT NULL DEFAULT true,
    definition      JSONB NOT NULL DEFAULT '{}',
    created_by      UUID,
    version         INTEGER NOT NULL DEFAULT 1,
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cohort_defs_tenant ON cohort_definitions (tenant_id);

-- ============================================================
-- PROJECTION: Cohort Membership (recomputed from event store)
-- ============================================================

CREATE TABLE proj_cohort_memberships (
    tenant_id       UUID NOT NULL,
    cohort_id       UUID NOT NULL REFERENCES cohort_definitions(id),
    user_id         UUID NOT NULL,
    is_member       BOOLEAN NOT NULL DEFAULT true,
    entered_at      TIMESTAMPTZ NOT NULL,
    exited_at       TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, cohort_id, user_id)
);

CREATE INDEX idx_proj_cohort_active ON proj_cohort_memberships (tenant_id, cohort_id)
    WHERE is_member = true;

-- ============================================================
-- PROJECTION: AI-Discovered Cohorts
-- ============================================================

CREATE TABLE proj_discovered_cohorts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    discovery_run_seq BIGINT NOT NULL,          -- event sequence that triggered discovery
    name            VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    statistical_significance NUMERIC(5, 4),
    effect_size     NUMERIC(8, 4),
    member_count    INTEGER NOT NULL,
    key_features    TEXT[],
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    promoted_cohort_id UUID REFERENCES cohort_definitions(id),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## ML Pipeline (Event-Driven)

```sql
-- ============================================================
-- PROJECTION: ML Feature Snapshots
-- ============================================================
-- Features computed from event replay; each snapshot tied to an event sequence position

CREATE TABLE proj_ml_features (
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL,
    as_of_seq       BIGINT NOT NULL,            -- event sequence position features were computed from
    -- Rolling-window behavioral features
    events_7d       INTEGER,
    events_30d      INTEGER,
    events_90d      INTEGER,
    sessions_7d     INTEGER,
    sessions_30d    INTEGER,
    days_active_30d INTEGER,
    days_since_last_activity INTEGER,
    avg_session_duration_30d NUMERIC(10, 2),
    distinct_features_used_30d INTEGER,
    event_trend_7d_vs_30d NUMERIC(6, 4),
    session_trend_7d_vs_30d NUMERIC(6, 4),
    days_since_signup INTEGER,
    -- B2B account features
    account_id      UUID,
    account_user_count INTEGER,
    account_active_user_pct NUMERIC(5, 4),
    -- All features as JSONB for flexibility
    all_features    JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (tenant_id, user_id, computed_at)
);

-- ============================================================
-- ML MODEL REGISTRY (configuration table)
-- ============================================================

CREATE TABLE ml_model_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    model_type      VARCHAR(50) NOT NULL,
    model_version   INTEGER NOT NULL,
    algorithm       VARCHAR(50) NOT NULL,
    hyperparameters JSONB NOT NULL DEFAULT '{}',
    training_metrics JSONB NOT NULL DEFAULT '{}',
    feature_names   TEXT[],
    trained_at      TIMESTAMPTZ NOT NULL,
    trained_from_seq BIGINT NOT NULL,           -- event sequence up to which model was trained
    is_active       BOOLEAN NOT NULL DEFAULT false,
    artifact_path   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, model_type, model_version)
);

-- ============================================================
-- PROJECTION: Latest Churn Scores (read-optimized)
-- ============================================================
-- The event_store contains system.prediction_generated events with full SHAP details.
-- This projection materializes the latest score per user for dashboard queries.

CREATE TABLE proj_churn_scores (
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    model_id        UUID NOT NULL REFERENCES ml_model_registry(id),
    prediction_date DATE NOT NULL,
    churn_probability_30d NUMERIC(5, 4),
    churn_probability_60d NUMERIC(5, 4),
    churn_probability_90d NUMERIC(5, 4),
    risk_tier       VARCHAR(20) NOT NULL,
    -- Top 5 SHAP explanations denormalized for fast read
    top_explanations JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"feature": "days_since_last_login", "shap": 0.31, "value": 45, 
    --    "text": "No login for 45 days (cohort avg: 3 days)"},
    --   {"feature": "session_trend", "shap": -0.22, "value": -0.6, 
    --    "text": "Session frequency dropped 60%"}
    -- ]
    source_event_id UUID NOT NULL,              -- the event_store event that produced this score
    source_event_seq BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, user_id, prediction_date)
);

CREATE INDEX idx_proj_churn_risk ON proj_churn_scores (tenant_id, risk_tier, prediction_date);

-- ============================================================
-- PROJECTION: RFM Scores
-- ============================================================

CREATE TABLE proj_rfm_scores (
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    computed_date   DATE NOT NULL,
    recency_days    INTEGER NOT NULL,
    frequency_30d   INTEGER NOT NULL,
    monetary_value  NUMERIC(12, 2),
    recency_score   INTEGER NOT NULL,
    frequency_score INTEGER NOT NULL,
    monetary_score  INTEGER NOT NULL,
    rfm_segment     VARCHAR(50) NOT NULL,
    source_event_seq BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, user_id, computed_date)
);
```

## Dashboards, Alerts & NL Queries (Configuration Tables)

```sql
-- ============================================================
-- CONFIGURATION TABLES (not event-sourced — mutable config)
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    data_region     VARCHAR(10) NOT NULL DEFAULT 'us',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_account_id UUID NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, user_account_id)
);

CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    layout          JSONB NOT NULL DEFAULT '[]',
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dashboard_widgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dashboard_id    UUID NOT NULL REFERENCES dashboards(id) ON DELETE CASCADE,
    widget_type     VARCHAR(50) NOT NULL,
    title           VARCHAR(255),
    config          JSONB NOT NULL DEFAULT '{}',
    position_x      INTEGER NOT NULL DEFAULT 0,
    position_y      INTEGER NOT NULL DEFAULT 0,
    width           INTEGER NOT NULL DEFAULT 6,
    height          INTEGER NOT NULL DEFAULT 4,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE retention_report_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    start_event     VARCHAR(255) NOT NULL,
    return_event    VARCHAR(255) NOT NULL,
    cohort_id       UUID REFERENCES cohort_definitions(id),
    granularity     VARCHAR(20) NOT NULL DEFAULT 'day',
    lookback_periods INTEGER NOT NULL DEFAULT 12,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE funnel_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    time_window     INTERVAL NOT NULL DEFAULT '7 days',
    steps           JSONB NOT NULL,
    -- Example: [{"order": 1, "event": "Page View", "filters": {}},
    --           {"order": 2, "event": "Add to Cart"},
    --           {"order": 3, "event": "Purchase"}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    alert_type      VARCHAR(50) NOT NULL,
    cohort_id       UUID REFERENCES cohort_definitions(id),
    metric          VARCHAR(100) NOT NULL,
    condition       VARCHAR(20) NOT NULL,
    threshold       NUMERIC(10, 4) NOT NULL,
    lookback_window INTERVAL NOT NULL DEFAULT '7 days',
    delivery_channel VARCHAR(20) NOT NULL,
    delivery_config JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- NL queries are recorded as events in event_store (type: system.nl_query)
-- but we keep a projection for fast retrieval
CREATE TABLE proj_nl_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_account_id UUID NOT NULL,
    query_text      TEXT NOT NULL,
    generated_sql   TEXT,
    result_type     VARCHAR(50),
    result_cohort_id UUID,
    execution_time_ms INTEGER,
    feedback        VARCHAR(20),
    source_event_id UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- GDPR: Encryption Key Registry (for crypto-shredding)
-- ============================================================

CREATE TABLE encryption_keys (
    key_id          VARCHAR(100) PRIMARY KEY,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID NOT NULL,              -- the tracked end-user this key belongs to
    encrypted_key   BYTEA NOT NULL,             -- the actual key, encrypted with master key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    destroyed_at    TIMESTAMPTZ,                -- when key was destroyed (crypto-shredded)
    UNIQUE (tenant_id, user_id)
);

CREATE TABLE data_deletion_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    key_destroyed_at TIMESTAMPTZ,               -- when the encryption key was destroyed
    projections_purged_at TIMESTAMPTZ,          -- when read models were purged
    completed_at    TIMESTAMPTZ
);
```

## Event Replay & Projection Tracking

```sql
-- ============================================================
-- PROJECTION CHECKPOINT TRACKING
-- ============================================================
-- Tracks which event each projection has processed up to, enabling incremental replay

CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) NOT NULL,
    tenant_id       UUID NOT NULL,
    last_processed_seq BIGINT NOT NULL DEFAULT 0,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'running',  -- running, paused, rebuilding
    PRIMARY KEY (projection_name, tenant_id)
);

-- Example entries:
-- ('proj_users', 'tenant-1', 1284567, '2026-05-11T10:30:00Z', 'running')
-- ('proj_daily_activity', 'tenant-1', 1284500, '2026-05-11T10:29:00Z', 'running')
-- ('proj_retention', 'tenant-1', 1280000, '2026-05-11T10:00:00Z', 'running')

-- ============================================================
-- SNAPSHOT STORE (optional: for long event streams)
-- ============================================================
-- Periodic snapshots of entity state to avoid replaying from event 0

CREATE TABLE entity_snapshots (
    tenant_id       UUID NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,       -- user, account
    entity_id       UUID NOT NULL,
    as_of_seq       BIGINT NOT NULL,
    state           JSONB NOT NULL,             -- full entity state at this point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, entity_type, entity_id, as_of_seq)
);
```

## Example Queries

### Temporal Query: What Was the Cohort Composition on a Past Date?

```sql
-- Replay cohort membership as of March 15, 2026
-- This is the killer feature of event sourcing: temporal queries
WITH events_as_of AS (
    SELECT *
    FROM event_store
    WHERE tenant_id = :tenant_id
      AND event_time <= '2026-03-15T23:59:59Z'
      AND event_type = 'tracking.track'
),
user_state_as_of AS (
    SELECT
        user_id,
        COUNT(*) AS total_events,
        MAX(event_time) AS last_active,
        MIN(event_time) AS first_seen,
        COUNT(DISTINCT DATE(event_time)) AS active_days
    FROM events_as_of
    WHERE user_id IS NOT NULL
    GROUP BY user_id
)
SELECT *
FROM user_state_as_of
WHERE active_days >= 5                         -- users active 5+ days
  AND last_active >= '2026-03-01'              -- active in the last 15 days
ORDER BY total_events DESC;
```

### Retroactive Retention Recalculation

```sql
-- Recalculate Day-7 retention for January signups using events replayed from the store
WITH jan_signups AS (
    SELECT DISTINCT ON (user_id)
        user_id,
        event_time::date AS signup_date
    FROM event_store
    WHERE tenant_id = :tenant_id
      AND event_type = 'tracking.identify'
      AND event_time >= '2026-01-01' AND event_time < '2026-02-01'
    ORDER BY user_id, event_time ASC
),
day7_activity AS (
    SELECT DISTINCT e.user_id
    FROM event_store e
    JOIN jan_signups s ON e.user_id = s.user_id
    WHERE e.tenant_id = :tenant_id
      AND e.event_type = 'tracking.track'
      AND e.event_time::date = s.signup_date + 7
)
SELECT
    COUNT(DISTINCT s.user_id) AS cohort_size,
    COUNT(DISTINCT d.user_id) AS day7_retained,
    ROUND(COUNT(DISTINCT d.user_id)::numeric / COUNT(DISTINCT s.user_id), 4) AS day7_retention
FROM jan_signups s
LEFT JOIN day7_activity d ON s.user_id = d.user_id;
```

### Read from Projection: Dashboard Churn Risk

```sql
-- Fast dashboard query against the projection (no event replay needed)
SELECT
    pu.external_id,
    pu.email,
    pu.display_name,
    cs.churn_probability_30d,
    cs.risk_tier,
    cs.top_explanations
FROM proj_churn_scores cs
JOIN proj_users pu ON cs.tenant_id = pu.tenant_id AND cs.user_id = pu.user_id
WHERE cs.tenant_id = :tenant_id
  AND cs.prediction_date = CURRENT_DATE
  AND cs.risk_tier IN ('critical', 'high')
ORDER BY cs.churn_probability_30d DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write Side) | 2 | Core event store (partitioned) + event type registry |
| Configuration | 10 | Tenants, members, dashboards, widgets, cohort defs, retention/funnel configs, alerts |
| Projections — Identity | 2 | User state, account state |
| Projections — Activity | 2 | Daily activity rollup, funnel progress |
| Projections — Analytics | 2 | Retention matrix, cohort memberships |
| Projections — ML | 3 | Feature snapshots, churn scores (with SHAP), RFM scores |
| Projections — AI/NL | 2 | Discovered cohorts, NL queries |
| Infrastructure | 2 | Projection checkpoints, entity snapshots |
| GDPR | 2 | Encryption keys, deletion requests |
| **Total** | **27** | Write: 2, Config: 10, Projections: 11, Infra: 2, GDPR: 2 |

---

## Key Design Decisions

1. **Single event store as source of truth**: All tracking data, ML predictions, identity merges, and system events flow through one append-only table. This guarantees a complete audit trail and enables temporal queries at any point in time.

2. **CloudEvents-aligned envelope**: Each event follows the CloudEvents specification (`id`, `source`, `type`, `time`, `data`), making the event store compatible with standard event streaming infrastructure (Kafka, Pulsar, EventBridge).

3. **Projections are disposable and rebuildable**: Every `proj_*` table can be dropped and rebuilt from the event store. The `projection_checkpoints` table tracks where each projection left off, enabling incremental catch-up after downtime or schema changes.

4. **SHAP explanations stored as events, projected for reads**: When the ML pipeline generates a churn prediction, it writes a `system.prediction_generated` event to the event store containing the full SHAP breakdown. The `proj_churn_scores` table denormalizes the top 5 explanations into JSONB for fast dashboard reads.

5. **Crypto-shredding for GDPR**: Instead of deleting events (which violates the append-only principle), PII fields are encrypted with a per-user key. Right-to-erasure is implemented by destroying the key, rendering the event data unreadable without physically deleting rows.

6. **Configuration tables are mutable**: Not everything is event-sourced. Dashboard layouts, alert rules, funnel configurations, and cohort definitions are stored in standard mutable tables. These are operational configuration, not analytical data.

7. **Partitioned by time**: The event store is range-partitioned by `event_time`, enabling efficient partition pruning for time-range queries and straightforward data lifecycle management.

8. **Sequence numbers for ordering**: Each event gets a `sequence_num` (bigserial) within its tenant, providing a total ordering that projections use for checkpointing. This is more reliable than timestamp ordering, which can have ties.

9. **Entity snapshots for performance**: For users with very long event histories (thousands of events), periodic snapshots capture the full entity state at a sequence number, allowing projections to resume from the snapshot rather than replaying from event zero.

10. **Eventual consistency is explicit**: Read projections may lag behind the event store. The `projected_at` and `last_event_seq` fields on every projection table make this lag visible, allowing the UI to show "data current as of X" when lag is significant.
