# Data Model Suggestion 4: Analytics-Optimized Dual-Store (PostgreSQL + ClickHouse)

> Project: Customer Cohort Analytics · Created: 2026-05-11

## Philosophy

This model separates operational concerns from analytical concerns into two purpose-built databases: **PostgreSQL** handles transactional/operational data (user accounts, cohort definitions, ML model registry, dashboard configuration, GDPR compliance) while **ClickHouse** handles high-volume event storage and analytical queries (event ingestion, retention curves, funnel analysis, cohort membership computation). An event streaming layer (Kafka or equivalent) bridges the two, ensuring events flow from ingestion to both stores.

This is the architecture that PostHog actually uses in production: ClickHouse for event storage and analytics queries, PostgreSQL for user/person metadata and application state, with data replicated between them as needed. It is also the pattern behind Plausible Analytics (ClickHouse), Countly (MongoDB for events), and enterprise deployments of Amplitude and Mixpanel (custom columnar stores).

The rationale is performance at scale. PostgreSQL excels at transactional consistency, complex joins, and mutable data, but struggles with analytical queries over hundreds of millions of event rows. ClickHouse excels at columnar scans, time-range aggregations, and high-throughput appends, but has limited support for updates, deletes, and complex transactions. By using each database for what it does best, the platform can ingest millions of events per day while serving sub-second dashboard queries.

**Best for:** Teams expecting high event volumes (>50M events/month), those who need sub-second analytics dashboards at scale, and production deployments where query performance on large datasets is a hard requirement.

**Trade-offs:**
- (+) Sub-second analytical queries over billions of events (ClickHouse columnar scans)
- (+) High-throughput event ingestion (ClickHouse append-only, no index maintenance on write)
- (+) Each database used for its strengths: PostgreSQL for ACID transactions, ClickHouse for OLAP
- (+) Proven architecture: PostHog, Plausible, and enterprise analytics platforms use this pattern
- (+) Horizontal scalability: ClickHouse cluster can be scaled independently of PostgreSQL
- (-) Operational complexity: two databases to deploy, monitor, backup, and upgrade
- (-) Data synchronization: user metadata must be replicated from PostgreSQL to ClickHouse
- (-) Higher infrastructure cost: ClickHouse requires dedicated compute/storage
- (-) Development complexity: queries must target the correct database
- (-) Not suitable for single-binary / Docker Compose MVP deployments

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment Spec | Event types and common fields define the ClickHouse event table schema |
| CloudEvents v1.0 | Event envelope in ClickHouse follows CloudEvents naming conventions |
| ISO 8601 | All timestamps as `DateTime64(3)` in ClickHouse, `TIMESTAMPTZ` in PostgreSQL |
| ISO 3166-1/2 | `LowCardinality(String)` country codes in ClickHouse events |
| GDPR / CCPA | PostgreSQL handles consent/deletion; ClickHouse events deleted via `ALTER TABLE DELETE` |
| SHAP | Prediction explanations stored in PostgreSQL; precomputed SHAP summaries cached in ClickHouse for dashboard queries |

---

## PostgreSQL: Operational Data

### Multi-Tenancy & Identity

```sql
-- ============================================================
-- POSTGRESQL: OPERATIONAL TABLES
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

### Tracked Users (Source of Truth in PostgreSQL)

```sql
-- Users are the source of truth in PostgreSQL (like PostHog's architecture)
-- User data is replicated to ClickHouse for join-free event queries

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    anonymous_ids   TEXT[] NOT NULL DEFAULT '{}',
    email           VARCHAR(255),
    display_name    VARCHAR(255),
    traits          JSONB NOT NULL DEFAULT '{}',
    plan_tier       VARCHAR(50),
    signup_source   VARCHAR(100),
    country_code    CHAR(2),
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ,
    total_events    BIGINT NOT NULL DEFAULT 0,
    anonymized_at   TIMESTAMPTZ,
    -- Replication tracking
    ch_synced_at    TIMESTAMPTZ,                -- last time user was synced to ClickHouse
    ch_version      INTEGER NOT NULL DEFAULT 1, -- version counter for ClickHouse ReplacingMergeTree
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_users_tenant_external ON users (tenant_id, external_id)
    WHERE external_id IS NOT NULL;
CREATE INDEX idx_users_tenant_first_seen ON users (tenant_id, first_seen_at);
CREATE INDEX idx_users_needs_sync ON users (tenant_id)
    WHERE ch_synced_at IS NULL OR ch_synced_at < updated_at;

-- Identity merge log (for resolving anonymous -> identified users)
CREATE TABLE identity_merges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    old_user_id     UUID NOT NULL,
    new_user_id     UUID NOT NULL REFERENCES users(id),
    merged_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    merge_reason    VARCHAR(50) NOT NULL DEFAULT 'identify'  -- identify, manual, dedup
);

CREATE INDEX idx_merges_old ON identity_merges (tenant_id, old_user_id);

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    name            VARCHAR(255),
    traits          JSONB NOT NULL DEFAULT '{}',
    industry        VARCHAR(100),
    plan_tier       VARCHAR(50),
    arr             NUMERIC(12, 2),
    country_code    CHAR(2),
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

CREATE TABLE account_memberships (
    tenant_id       UUID NOT NULL,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(100),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, account_id, user_id)
);
```

### Cohort Definitions & ML Models (PostgreSQL)

```sql
-- Cohort definitions live in PostgreSQL (mutable, relational)
-- Cohort membership is computed in ClickHouse from events

CREATE TABLE cohorts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) NOT NULL,
    is_dynamic      BOOLEAN NOT NULL DEFAULT true,
    definition      JSONB NOT NULL DEFAULT '{}',
    member_count    INTEGER DEFAULT 0,
    last_computed   TIMESTAMPTZ,
    created_by      UUID,
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE discovered_cohorts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    discovery_run_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    statistical_significance NUMERIC(5, 4),
    effect_size     NUMERIC(8, 4),
    member_count    INTEGER NOT NULL,
    key_features    JSONB NOT NULL DEFAULT '[]',
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    promoted_cohort_id UUID REFERENCES cohorts(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    model_type      VARCHAR(50) NOT NULL,
    model_version   INTEGER NOT NULL,
    algorithm       VARCHAR(50) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    trained_at      TIMESTAMPTZ NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT false,
    artifact_path   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, model_type, model_version)
);

-- Churn predictions and SHAP explanations in PostgreSQL
-- (mutable: predictions update daily, need ACID guarantees)
CREATE TABLE churn_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    model_id        UUID NOT NULL REFERENCES ml_models(id),
    prediction_date DATE NOT NULL,
    churn_probability_30d NUMERIC(5, 4),
    churn_probability_60d NUMERIC(5, 4),
    churn_probability_90d NUMERIC(5, 4),
    risk_tier       VARCHAR(20) NOT NULL,
    explanations    JSONB NOT NULL DEFAULT '[]',
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, user_id, prediction_date)
);

CREATE INDEX idx_predictions_risk ON churn_predictions (tenant_id, risk_tier, prediction_date);
```

### Dashboards, Alerts, NL Queries (PostgreSQL)

```sql
CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE funnels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    time_window     INTERVAL NOT NULL DEFAULT '7 days',
    steps           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE retention_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    start_event     VARCHAR(255) NOT NULL,
    return_event    VARCHAR(255) NOT NULL,
    cohort_id       UUID REFERENCES cohorts(id),
    granularity     VARCHAR(20) NOT NULL DEFAULT 'day',
    lookback_periods INTEGER NOT NULL DEFAULT 12,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scheduled_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    dashboard_id    UUID REFERENCES dashboards(id),
    name            VARCHAR(255) NOT NULL,
    schedule_cron   VARCHAR(100) NOT NULL,
    delivery        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sent_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    triggered_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    metric_value    NUMERIC(10, 4),
    narrative       TEXT,
    context         JSONB NOT NULL DEFAULT '{}',
    acknowledged    BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE nl_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_account_id UUID NOT NULL,
    query_text      TEXT NOT NULL,
    generated_sql   TEXT,
    result_type     VARCHAR(50),
    result_data     JSONB,
    execution_time_ms INTEGER,
    feedback        VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GDPR
CREATE TABLE consent_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    consent_type    VARCHAR(50) NOT NULL,
    granted         BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}'
);

CREATE TABLE data_deletion_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    pg_completed_at TIMESTAMPTZ,                -- when PostgreSQL records were deleted
    ch_completed_at TIMESTAMPTZ,                -- when ClickHouse records were deleted
    completed_at    TIMESTAMPTZ
);
```

---

## ClickHouse: Analytical Data

### Event Table (Primary Analytics Table)

```sql
-- ============================================================
-- CLICKHOUSE: EVENT STORAGE
-- ============================================================
-- This is the primary event table. ClickHouse stores events in columnar format
-- with extremely fast aggregation performance.

CREATE TABLE events (
    -- Event identity
    event_id        UUID,
    tenant_id       UUID,
    
    -- Segment spec fields
    event_type      LowCardinality(String),     -- track, identify, page, screen, group
    event_name      LowCardinality(String),     -- "Purchase", "Page View", etc.
    user_id         UUID,
    anonymous_id    String,
    
    -- Timestamps
    timestamp       DateTime64(3),              -- millisecond precision, event time
    received_at     DateTime64(3),              -- server receive time
    
    -- Commonly queried dimensions (LowCardinality for compression)
    session_id      String,
    page_url        String,
    page_title      String,
    referrer        String,
    utm_source      LowCardinality(String),
    utm_medium      LowCardinality(String),
    utm_campaign    LowCardinality(String),
    device_type     LowCardinality(String),     -- desktop, mobile, tablet
    os_name         LowCardinality(String),
    browser_name    LowCardinality(String),
    country_code    LowCardinality(FixedString(2)),  -- ISO 3166-1
    
    -- User properties snapshot at event time (denormalized from PostgreSQL)
    user_email      String,
    user_plan_tier  LowCardinality(String),
    user_signup_source LowCardinality(String),
    
    -- Flexible event properties
    properties      String                      -- JSON string (not native JSON for perf)
)
ENGINE = MergeTree()
PARTITION BY (tenant_id, toYYYYMM(timestamp))
ORDER BY (tenant_id, event_name, user_id, timestamp)
TTL timestamp + INTERVAL 2 YEAR
SETTINGS index_granularity = 8192;

-- Skip index for fast filtering on specific event properties
ALTER TABLE events ADD INDEX idx_properties properties TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 4;
```

### User Lookup (Replicated from PostgreSQL)

```sql
-- ============================================================
-- CLICKHOUSE: USER LOOKUP (replicated from PostgreSQL)
-- ============================================================
-- ReplacingMergeTree collapses duplicate rows by version, keeping the latest

CREATE TABLE users (
    tenant_id       UUID,
    user_id         UUID,
    external_id     String,
    email           String,
    display_name    String,
    plan_tier       LowCardinality(String),
    signup_source   LowCardinality(String),
    country_code    LowCardinality(FixedString(2)),
    first_seen_at   DateTime64(3),
    last_seen_at    DateTime64(3),
    total_events    UInt64,
    version         UInt32                      -- incremented on each PostgreSQL update
)
ENGINE = ReplacingMergeTree(version)
ORDER BY (tenant_id, user_id);

-- Identity override table (for user merges)
CREATE TABLE person_overrides (
    tenant_id       UUID,
    old_user_id     UUID,
    new_user_id     UUID,
    merged_at       DateTime64(3),
    version         UInt32
)
ENGINE = ReplacingMergeTree(version)
ORDER BY (tenant_id, old_user_id);
```

### Materialized Views for Pre-Aggregation

```sql
-- ============================================================
-- CLICKHOUSE: MATERIALIZED VIEWS
-- ============================================================

-- Daily activity rollup (populated automatically on insert)
CREATE TABLE daily_activity_mv (
    tenant_id       UUID,
    user_id         UUID,
    activity_date   Date,
    event_count     UInt64,
    session_count   AggregateFunction(uniqExact, String),
    distinct_event_names AggregateFunction(uniqExact, String),
    min_timestamp   SimpleAggregateFunction(min, DateTime64(3)),
    max_timestamp   SimpleAggregateFunction(max, DateTime64(3))
)
ENGINE = AggregatingMergeTree()
PARTITION BY (tenant_id, toYYYYMM(activity_date))
ORDER BY (tenant_id, user_id, activity_date);

CREATE MATERIALIZED VIEW daily_activity_mv_writer TO daily_activity_mv AS
SELECT
    tenant_id,
    user_id,
    toDate(timestamp) AS activity_date,
    count() AS event_count,
    uniqExactState(session_id) AS session_count,
    uniqExactState(event_name) AS distinct_event_names,
    min(timestamp) AS min_timestamp,
    max(timestamp) AS max_timestamp
FROM events
WHERE user_id != toUUID('00000000-0000-0000-0000-000000000000')
GROUP BY tenant_id, user_id, activity_date;

-- Event name counts per tenant (for schema discovery UI)
CREATE TABLE event_counts_mv (
    tenant_id       UUID,
    event_name      LowCardinality(String),
    event_date      Date,
    event_count     UInt64,
    unique_users    AggregateFunction(uniqExact, UUID)
)
ENGINE = AggregatingMergeTree()
PARTITION BY (tenant_id, toYYYYMM(event_date))
ORDER BY (tenant_id, event_name, event_date);

CREATE MATERIALIZED VIEW event_counts_mv_writer TO event_counts_mv AS
SELECT
    tenant_id,
    event_name,
    toDate(timestamp) AS event_date,
    count() AS event_count,
    uniqExactState(user_id) AS unique_users
FROM events
WHERE event_name != ''
GROUP BY tenant_id, event_name, event_date;

-- Cohort membership cache (populated by batch computation)
CREATE TABLE cohort_memberships (
    tenant_id       UUID,
    cohort_id       UUID,
    user_id         UUID,
    entered_at      DateTime64(3),
    is_member       UInt8,                      -- 1 = active, 0 = exited
    computed_at     DateTime64(3),
    version         UInt32
)
ENGINE = ReplacingMergeTree(version)
ORDER BY (tenant_id, cohort_id, user_id);
```

### ML Feature Snapshots in ClickHouse

```sql
-- ============================================================
-- CLICKHOUSE: ML FEATURE SNAPSHOTS
-- ============================================================
-- Feature vectors computed by the ML pipeline, stored in ClickHouse
-- for fast batch training data export

CREATE TABLE ml_feature_snapshots (
    tenant_id       UUID,
    user_id         UUID,
    computed_at     DateTime64(3),
    
    -- Behavioral features
    events_7d       UInt32,
    events_30d      UInt32,
    events_90d      UInt32,
    sessions_7d     UInt32,
    sessions_30d    UInt32,
    days_active_30d UInt16,
    days_since_last_activity UInt16,
    avg_session_duration_30d Float64,
    distinct_features_used_30d UInt16,
    event_trend_7d_vs_30d Float64,
    session_trend_7d_vs_30d Float64,
    days_since_signup UInt16,
    
    -- B2B account features
    account_id      UUID,
    account_user_count UInt16,
    account_active_user_pct Float64,
    
    -- Churn label (for training data)
    churned_30d     Nullable(UInt8),            -- 1 = churned, 0 = retained, NULL = unknown
    churned_60d     Nullable(UInt8),
    churned_90d     Nullable(UInt8)
)
ENGINE = MergeTree()
PARTITION BY (tenant_id, toYYYYMM(computed_at))
ORDER BY (tenant_id, user_id, computed_at);

-- Churn score cache (replicated from PostgreSQL for dashboard queries)
CREATE TABLE churn_scores_cache (
    tenant_id       UUID,
    user_id         UUID,
    prediction_date Date,
    churn_probability_30d Float32,
    churn_probability_60d Float32,
    churn_probability_90d Float32,
    risk_tier       LowCardinality(String),
    top_explanation_1 String,
    top_explanation_2 String,
    top_explanation_3 String,
    version         UInt32
)
ENGINE = ReplacingMergeTree(version)
ORDER BY (tenant_id, user_id, prediction_date);

-- RFM scores
CREATE TABLE rfm_scores (
    tenant_id       UUID,
    user_id         UUID,
    computed_date   Date,
    recency_days    UInt16,
    frequency_30d   UInt16,
    monetary_value  Float64,
    recency_score   UInt8,
    frequency_score UInt8,
    monetary_score  UInt8,
    rfm_segment     LowCardinality(String)
)
ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, user_id, computed_date);
```

---

## Example Queries

### ClickHouse: Day-N Retention Curve (Sub-Second on Billions of Events)

```sql
-- Calculate Day-1, Day-7, Day-30 retention for April 2026 signups
-- This query runs against the daily_activity_mv materialized view
WITH signup_cohort AS (
    SELECT
        tenant_id,
        user_id,
        min(activity_date) AS signup_date
    FROM daily_activity_mv
    WHERE tenant_id = {tenant_id:UUID}
    GROUP BY tenant_id, user_id
    HAVING signup_date >= '2026-04-01' AND signup_date < '2026-05-01'
)
SELECT
    signup_date,
    count() AS cohort_size,
    countIf(d1.user_id != toUUID('00000000-0000-0000-0000-000000000000')) AS day_1,
    countIf(d7.user_id != toUUID('00000000-0000-0000-0000-000000000000')) AS day_7,
    countIf(d30.user_id != toUUID('00000000-0000-0000-0000-000000000000')) AS day_30
FROM signup_cohort sc
LEFT JOIN daily_activity_mv d1
    ON sc.tenant_id = d1.tenant_id AND sc.user_id = d1.user_id
    AND d1.activity_date = sc.signup_date + 1
LEFT JOIN daily_activity_mv d7
    ON sc.tenant_id = d7.tenant_id AND sc.user_id = d7.user_id
    AND d7.activity_date = sc.signup_date + 7
LEFT JOIN daily_activity_mv d30
    ON sc.tenant_id = d30.tenant_id AND sc.user_id = d30.user_id
    AND d30.activity_date = sc.signup_date + 30
GROUP BY signup_date
ORDER BY signup_date;
```

### ClickHouse: Funnel Analysis with windowFunnel

```sql
-- Funnel: Viewed Pricing -> Started Trial -> Activated -> Subscribed
-- windowFunnel is a native ClickHouse function optimized for this exact use case
SELECT
    level,
    count() AS users_at_step,
    round(count() * 100.0 / max(total), 2) AS conversion_pct
FROM (
    SELECT
        user_id,
        windowFunnel(604800)(                  -- 7-day window (seconds)
            timestamp,
            event_name = 'Viewed Pricing',
            event_name = 'Started Trial',
            event_name = 'Activated',
            event_name = 'Subscribed'
        ) AS level
    FROM events
    WHERE tenant_id = {tenant_id:UUID}
      AND timestamp >= '2026-04-01' AND timestamp < '2026-05-01'
      AND event_name IN ('Viewed Pricing', 'Started Trial', 'Activated', 'Subscribed')
    GROUP BY user_id
)
CROSS JOIN (
    SELECT count(DISTINCT user_id) AS total
    FROM events
    WHERE tenant_id = {tenant_id:UUID}
      AND timestamp >= '2026-04-01' AND timestamp < '2026-05-01'
      AND event_name = 'Viewed Pricing'
) AS t
GROUP BY level
HAVING level > 0
ORDER BY level;
```

### ClickHouse: Event Property Analysis

```sql
-- Top product categories by revenue from Purchase events
SELECT
    JSONExtractString(properties, 'category') AS category,
    count() AS purchase_count,
    sum(JSONExtractFloat(properties, 'price')) AS total_revenue,
    uniqExact(user_id) AS unique_buyers
FROM events
WHERE tenant_id = {tenant_id:UUID}
  AND event_name = 'Purchase'
  AND timestamp >= '2026-04-01' AND timestamp < '2026-05-01'
GROUP BY category
ORDER BY total_revenue DESC
LIMIT 20;
```

### PostgreSQL: Churn Predictions with SHAP (Operational Query)

```sql
-- Top at-risk users with explanations (from PostgreSQL)
SELECT
    u.external_id,
    u.email,
    cp.churn_probability_30d,
    cp.risk_tier,
    cp.explanations
FROM churn_predictions cp
JOIN users u ON cp.user_id = u.id
WHERE cp.tenant_id = :tenant_id
  AND cp.prediction_date = CURRENT_DATE
  AND cp.risk_tier IN ('critical', 'high')
ORDER BY cp.churn_probability_30d DESC
LIMIT 20;
```

---

## Data Flow Architecture

```
SDK Events                                         
    |                                              
    v                                              
[Ingestion API] ──> [Kafka / Event Stream]         
                         |              |          
                         v              v          
                   [ClickHouse]    [PostgreSQL]    
                   (events,        (users,         
                    daily_activity, accounts,       
                    cohort_cache,   cohorts,        
                    ml_features,    ml_models,      
                    rfm_scores)     predictions,    
                                    dashboards,     
                                    GDPR)          
                         |              |          
                         v              v          
                   [Query Router] ─────────        
                         |                         
                         v                         
                   [API / Dashboard]               
```

**Query Router Logic:**
- Retention curves, funnel analysis, event counts, cohort computation --> ClickHouse
- User profiles, churn predictions with SHAP, dashboard config, alert management --> PostgreSQL
- Combined queries (e.g., "at-risk users who dropped off funnel step 3") --> PostgreSQL for user data + ClickHouse for event data, joined in the application layer

---

## Table Count Summary

| Category | Database | Tables | Notes |
|----------|----------|--------|-------|
| Multi-Tenancy | PostgreSQL | 2 | Tenants, members |
| Users & Accounts | PostgreSQL | 4 | Users, identity merges, accounts, memberships |
| Users (replica) | ClickHouse | 2 | User lookup, person overrides |
| Events | ClickHouse | 1 | Events (partitioned by tenant + month) |
| Materialized Views | ClickHouse | 3 | Daily activity, event counts, cohort memberships |
| Cohort Definitions | PostgreSQL | 2 | Cohort definitions, discovered cohorts |
| ML & Predictions | PostgreSQL | 2 | Model registry, churn predictions |
| ML & Features | ClickHouse | 3 | Feature snapshots, churn score cache, RFM scores |
| Dashboards & Config | PostgreSQL | 6 | Dashboards, widgets, funnels, retention reports, scheduled reports, alert rules |
| Alerts & NL | PostgreSQL | 2 | Alert history, NL queries |
| GDPR | PostgreSQL | 2 | Consent records, deletion requests |
| **Total** | **Both** | **29** | PostgreSQL: 20, ClickHouse: 9 |

---

## Key Design Decisions

1. **PostgreSQL as source of truth for users, ClickHouse for events**: User data is mutable (traits change, identities merge) and needs ACID transactions. Events are append-only and need columnar scan performance. Each database handles what it does best.

2. **User data replicated to ClickHouse via ReplacingMergeTree**: User properties are snapshotted onto events at ingestion time (like PostHog) and also maintained in a ReplacingMergeTree lookup table. The `version` column ensures only the latest user state is returned after merge.

3. **Identity merge via person_overrides table**: When anonymous users are identified (Segment `identify` call), a mapping is written to `person_overrides` in ClickHouse. Queries join through this table to resolve merged identities without rewriting historical events.

4. **Materialized views for pre-aggregation**: ClickHouse materialized views (`daily_activity_mv`, `event_counts_mv`) aggregate data at insert time, so dashboard queries hit pre-aggregated tables rather than scanning raw events. This is the key to sub-second dashboard performance.

5. **windowFunnel for native funnel analysis**: ClickHouse's built-in `windowFunnel` function is purpose-built for funnel queries, avoiding the complex self-joins required in PostgreSQL.

6. **LowCardinality strings for dimension columns**: ClickHouse's `LowCardinality(String)` type provides dictionary encoding for columns with limited distinct values (event_type, country_code, plan_tier), dramatically reducing storage and improving scan speed.

7. **Churn predictions in PostgreSQL, cached in ClickHouse**: The source of truth for predictions (with full SHAP explanations) lives in PostgreSQL where they can be updated atomically. A simplified cache in ClickHouse serves dashboard aggregation queries (e.g., "distribution of risk tiers across cohorts").

8. **TTL on ClickHouse events**: The `TTL timestamp + INTERVAL 2 YEAR` clause automatically drops events older than 2 years, managing storage growth without manual intervention.

9. **GDPR deletion spans both databases**: The `data_deletion_requests` table tracks completion status for both PostgreSQL (`pg_completed_at`) and ClickHouse (`ch_completed_at`). ClickHouse deletion uses `ALTER TABLE DELETE` (heavyweight mutation) and is tracked separately.

10. **Kafka as the bridge**: Events flow through Kafka (or equivalent) to both databases, providing replay capability if either database needs to be rebuilt, backfill capability for new materialized views, and decoupling between ingestion and storage.
