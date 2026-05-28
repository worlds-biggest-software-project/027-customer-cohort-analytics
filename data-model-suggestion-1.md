# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Customer Cohort Analytics · Created: 2026-05-11

## Philosophy

This model follows classical relational database design: every concept in the domain gets its own table, relationships are expressed through foreign keys, and data integrity is enforced at the database level through constraints and triggers. The schema is fully normalized to at least 3NF, with denormalization applied only where query performance demands it (e.g., materialized views for retention curves).

This is the approach used by traditional enterprise analytics platforms and customer success tools like Gainsight and ChurnZero at their core. It provides the strongest data integrity guarantees, the most straightforward query patterns for ad-hoc analysis, and the best compatibility with standard BI tools, ORMs, and reporting frameworks.

The trade-off is rigidity: adding a new event property or a new dimension to cohort analysis requires a schema migration. For a platform that needs to ingest arbitrary event properties from diverse customer applications, this creates ongoing maintenance burden. However, for teams that value explicitness, auditability, and strong typing over flexibility, this is the most proven approach.

**Best for:** Teams that prioritize data integrity, SQL-first querying, regulatory compliance, and integration with standard BI tools over schema flexibility.

**Trade-offs:**
- (+) Strongest referential integrity and constraint enforcement
- (+) Standard SQL queries work naturally; no JSONB gymnastics
- (+) Excellent tooling support (ORMs, migration frameworks, BI connectors)
- (+) Easiest to reason about for new developers
- (-) Schema migrations required for every new event property or dimension
- (-) High table count increases complexity of joins for cross-entity queries
- (-) Less flexible for customers with widely varying event schemas
- (-) Slower iteration velocity for MVP development

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment Spec (`.track()`, `.identify()`, `.page()`, `.group()`) | Event types map directly to `event_type` enum; common fields (anonymous_id, user_id, context) have dedicated columns |
| ISO 8601 | All timestamps stored as `TIMESTAMPTZ` in ISO 8601 format |
| ISO 3166-1/2 | User and account geography stored as ISO country/subdivision codes |
| GDPR / CCPA | Dedicated `consent_records` and `data_deletion_requests` tables; `users.anonymized_at` supports right-to-erasure |
| RFM Segmentation | `rfm_scores` table stores precomputed Recency-Frequency-Monetary scores per user |
| SHAP (Shapley Additive Explanations) | `prediction_explanations` table stores per-user SHAP feature attributions |

---

## Multi-Tenancy & Identity

```sql
-- ============================================================
-- MULTI-TENANCY & ORGANIZATION
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, growth, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    data_region     VARCHAR(10) NOT NULL DEFAULT 'us',    -- ISO 3166-1 alpha-2
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants (slug);

CREATE TABLE tenant_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_account_id UUID NOT NULL,  -- references the platform user (not the tracked end-user)
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, member, viewer
    invited_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, user_account_id)
);

CREATE INDEX idx_tenant_members_tenant ON tenant_members (tenant_id);
```

## User & Account Identity (Tracked End-Users)

```sql
-- ============================================================
-- TRACKED END-USERS (the customers of our customers)
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),           -- customer-assigned user ID
    anonymous_id    VARCHAR(255),           -- Segment anonymous_id
    email           VARCHAR(255),
    display_name    VARCHAR(255),
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ,
    country_code    CHAR(2),               -- ISO 3166-1 alpha-2
    region_code     VARCHAR(6),            -- ISO 3166-2 subdivision
    timezone        VARCHAR(50),           -- IANA timezone
    language        VARCHAR(10),           -- BCP 47 language tag
    signup_source   VARCHAR(100),          -- acquisition channel
    plan_tier       VARCHAR(50),           -- user's subscription tier
    anonymized_at   TIMESTAMPTZ,           -- GDPR: when user data was anonymized
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_users_tenant_external ON users (tenant_id, external_id)
    WHERE external_id IS NOT NULL;
CREATE UNIQUE INDEX idx_users_tenant_anonymous ON users (tenant_id, anonymous_id)
    WHERE anonymous_id IS NOT NULL;
CREATE INDEX idx_users_tenant_first_seen ON users (tenant_id, first_seen_at);
CREATE INDEX idx_users_tenant_last_seen ON users (tenant_id, last_seen_at);

-- User properties as separate rows for full history tracking
CREATE TABLE user_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    property_name   VARCHAR(255) NOT NULL,
    property_value  TEXT,
    property_type   VARCHAR(20) NOT NULL DEFAULT 'string',  -- string, number, boolean, date
    set_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, user_id, property_name)
);

CREATE INDEX idx_user_props_lookup ON user_properties (tenant_id, property_name, property_value);

-- B2B: Account/Company grouping
CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,  -- Segment group_id
    name            VARCHAR(255),
    industry        VARCHAR(100),
    employee_count  INTEGER,
    plan_tier       VARCHAR(50),
    arr             NUMERIC(12, 2),         -- annual recurring revenue
    country_code    CHAR(2),               -- ISO 3166-1
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

CREATE INDEX idx_accounts_tenant ON accounts (tenant_id);

CREATE TABLE account_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(100),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, account_id, user_id)
);

CREATE INDEX idx_account_memberships_account ON account_memberships (tenant_id, account_id);
CREATE INDEX idx_account_memberships_user ON account_memberships (tenant_id, user_id);
```

## Event Tracking

```sql
-- ============================================================
-- EVENT TRACKING
-- ============================================================

CREATE TABLE event_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,      -- e.g., "Button Clicked", "Page Viewed"
    category        VARCHAR(100),               -- e.g., "engagement", "conversion", "lifecycle"
    description     TEXT,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_event_types_tenant ON event_types (tenant_id);

CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
    anonymous_id    VARCHAR(255),
    event_type_id   UUID NOT NULL REFERENCES event_types(id),
    event_name      VARCHAR(255) NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    session_id      VARCHAR(255),
    -- Segment context fields (denormalized for query performance)
    ip_address      INET,
    user_agent      TEXT,
    page_url        TEXT,
    page_title      VARCHAR(500),
    referrer        TEXT,
    utm_source      VARCHAR(255),
    utm_medium      VARCHAR(255),
    utm_campaign    VARCHAR(255),
    device_type     VARCHAR(50),        -- desktop, mobile, tablet
    os_name         VARCHAR(50),
    browser_name    VARCHAR(50),
    country_code    CHAR(2),            -- ISO 3166-1 from IP geolocation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions (example for 2026)
CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions created by automated job

CREATE INDEX idx_events_tenant_user_ts ON events (tenant_id, user_id, timestamp);
CREATE INDEX idx_events_tenant_type_ts ON events (tenant_id, event_type_id, timestamp);
CREATE INDEX idx_events_tenant_name_ts ON events (tenant_id, event_name, timestamp);
CREATE INDEX idx_events_session ON events (tenant_id, session_id) WHERE session_id IS NOT NULL;

-- Event properties as separate rows (fully normalized)
CREATE TABLE event_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    property_name   VARCHAR(255) NOT NULL,
    property_value  TEXT,
    property_type   VARCHAR(20) NOT NULL DEFAULT 'string',
    UNIQUE (event_id, property_name)
);

CREATE INDEX idx_event_props_event ON event_properties (event_id);
CREATE INDEX idx_event_props_lookup ON event_properties (tenant_id, property_name, property_value);
```

## Cohort Management

```sql
-- ============================================================
-- COHORT MANAGEMENT
-- ============================================================

CREATE TABLE cohorts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) NOT NULL,       -- behavioral, temporal, demographic, predictive, ai_discovered
    is_dynamic      BOOLEAN NOT NULL DEFAULT true,
    -- For behavioral cohorts: structured filter definition
    definition      JSONB NOT NULL DEFAULT '{}',
    -- Example definition:
    -- {
    --   "operator": "AND",
    --   "conditions": [
    --     {"type": "did_event", "event": "Purchase", "time_window": "30d", "count_operator": ">=", "count": 1},
    --     {"type": "user_property", "property": "plan_tier", "operator": "=", "value": "pro"}
    --   ]
    -- }
    created_by      UUID,                       -- tenant_member who created
    last_computed   TIMESTAMPTZ,
    member_count    INTEGER DEFAULT 0,
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cohorts_tenant ON cohorts (tenant_id);
CREATE INDEX idx_cohorts_tenant_type ON cohorts (tenant_id, type);

-- Cohort membership (materialized for query performance)
CREATE TABLE cohort_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    cohort_id       UUID NOT NULL REFERENCES cohorts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,                -- NULL = still active member
    UNIQUE (tenant_id, cohort_id, user_id, entered_at)
);

CREATE INDEX idx_cohort_members_cohort ON cohort_memberships (tenant_id, cohort_id)
    WHERE exited_at IS NULL;
CREATE INDEX idx_cohort_members_user ON cohort_memberships (tenant_id, user_id);

-- AI-discovered cohorts
CREATE TABLE discovered_cohorts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    cohort_id       UUID REFERENCES cohorts(id),  -- linked cohort once accepted
    discovery_run_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,       -- AI-generated name
    description     TEXT NOT NULL,                -- AI-generated explanation
    statistical_significance NUMERIC(5, 4),      -- p-value
    effect_size     NUMERIC(8, 4),               -- e.g., retention rate difference
    member_count    INTEGER NOT NULL,
    key_features    TEXT[] NOT NULL,              -- array of distinguishing features
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, accepted, dismissed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_discovered_cohorts_tenant ON discovered_cohorts (tenant_id, status);
```

## Retention & Funnel Analysis

```sql
-- ============================================================
-- RETENTION & FUNNEL ANALYSIS
-- ============================================================

-- Pre-computed daily activity summaries per user
CREATE TABLE daily_user_activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    activity_date   DATE NOT NULL,
    event_count     INTEGER NOT NULL DEFAULT 0,
    session_count   INTEGER NOT NULL DEFAULT 0,
    distinct_events INTEGER NOT NULL DEFAULT 0,  -- count of distinct event types
    first_event_at  TIMESTAMPTZ,
    last_event_at   TIMESTAMPTZ,
    UNIQUE (tenant_id, user_id, activity_date)
);

CREATE INDEX idx_daily_activity_tenant_date ON daily_user_activity (tenant_id, activity_date);

-- Funnel definitions
CREATE TABLE funnels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    time_window     INTERVAL NOT NULL DEFAULT '7 days',  -- max time to complete funnel
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE funnel_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    funnel_id       UUID NOT NULL REFERENCES funnels(id) ON DELETE CASCADE,
    step_order      INTEGER NOT NULL,
    event_name      VARCHAR(255) NOT NULL,
    filters         JSONB DEFAULT '{}',           -- additional property filters for this step
    UNIQUE (funnel_id, step_order)
);

CREATE INDEX idx_funnel_steps_funnel ON funnel_steps (funnel_id);

-- Retention report configurations
CREATE TABLE retention_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    start_event     VARCHAR(255) NOT NULL,       -- e.g., "Signed Up"
    return_event    VARCHAR(255) NOT NULL,        -- e.g., "Logged In"
    cohort_id       UUID REFERENCES cohorts(id),  -- optional cohort filter
    granularity     VARCHAR(20) NOT NULL DEFAULT 'day',  -- day, week, month
    lookback_periods INTEGER NOT NULL DEFAULT 12,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Materialized retention data (pre-computed for dashboard performance)
CREATE TABLE retention_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    report_id       UUID NOT NULL REFERENCES retention_reports(id) ON DELETE CASCADE,
    cohort_period   DATE NOT NULL,               -- the cohort's start period (e.g., signup week)
    period_offset   INTEGER NOT NULL,            -- 0, 1, 2, ... N periods after cohort_period
    cohort_size     INTEGER NOT NULL,            -- total users in this cohort period
    retained_count  INTEGER NOT NULL,            -- users who returned in this offset period
    retention_rate  NUMERIC(5, 4) NOT NULL,      -- retained_count / cohort_size
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, report_id, cohort_period, period_offset)
);

CREATE INDEX idx_retention_data_report ON retention_data (tenant_id, report_id, cohort_period);
```

## ML Churn Prediction & Explainability

```sql
-- ============================================================
-- ML CHURN PREDICTION & EXPLAINABILITY
-- ============================================================

-- ML model registry
CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    model_type      VARCHAR(50) NOT NULL,       -- churn_prediction, ltv_prediction, cohort_discovery
    model_version   INTEGER NOT NULL,
    algorithm       VARCHAR(50) NOT NULL,       -- xgboost, lightgbm, random_forest
    hyperparameters JSONB NOT NULL DEFAULT '{}',
    training_metrics JSONB NOT NULL DEFAULT '{}',
    -- Example training_metrics:
    -- {"auc_roc": 0.932, "precision": 0.84, "recall": 0.78, "f1": 0.81}
    feature_names   TEXT[] NOT NULL,
    training_rows   INTEGER,
    trained_at      TIMESTAMPTZ NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT false,
    artifact_path   TEXT,                        -- S3/GCS path to serialized model
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, model_type, model_version)
);

CREATE INDEX idx_ml_models_active ON ml_models (tenant_id, model_type)
    WHERE is_active = true;

-- Feature store: precomputed features per user for ML training/inference
CREATE TABLE ml_features (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    computed_at     TIMESTAMPTZ NOT NULL,
    -- Behavioral features (rolling windows)
    events_7d       INTEGER,                    -- event count, trailing 7 days
    events_30d      INTEGER,
    events_90d      INTEGER,
    sessions_7d     INTEGER,
    sessions_30d    INTEGER,
    days_active_30d INTEGER,                    -- distinct active days in 30d
    days_since_last_activity INTEGER,
    -- Engagement features
    avg_session_duration_30d NUMERIC(10, 2),
    distinct_features_used_30d INTEGER,
    -- Trend features
    event_trend_7d_vs_30d NUMERIC(6, 4),        -- ratio of 7d to 30d average
    session_trend_7d_vs_30d NUMERIC(6, 4),
    -- Lifecycle features
    days_since_signup INTEGER,
    -- Account features (B2B)
    account_id      UUID REFERENCES accounts(id),
    account_user_count INTEGER,
    account_active_user_pct NUMERIC(5, 4),
    UNIQUE (tenant_id, user_id, computed_at)
);

CREATE INDEX idx_ml_features_tenant_user ON ml_features (tenant_id, user_id, computed_at DESC);

-- Churn predictions
CREATE TABLE churn_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    model_id        UUID NOT NULL REFERENCES ml_models(id),
    prediction_date DATE NOT NULL,
    -- Prediction outputs
    churn_probability_30d NUMERIC(5, 4),        -- probability of churn in next 30 days
    churn_probability_60d NUMERIC(5, 4),
    churn_probability_90d NUMERIC(5, 4),
    risk_tier       VARCHAR(20) NOT NULL,       -- critical, high, medium, low
    -- Metadata
    feature_snapshot_id UUID REFERENCES ml_features(id),
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, user_id, model_id, prediction_date)
);

CREATE INDEX idx_predictions_tenant_date ON churn_predictions (tenant_id, prediction_date);
CREATE INDEX idx_predictions_risk ON churn_predictions (tenant_id, risk_tier, prediction_date);

-- SHAP explanations per prediction
CREATE TABLE prediction_explanations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prediction_id   UUID NOT NULL REFERENCES churn_predictions(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    feature_name    VARCHAR(255) NOT NULL,
    shap_value      NUMERIC(10, 6) NOT NULL,
    feature_value   TEXT,                       -- the actual feature value for this user
    direction       VARCHAR(10) NOT NULL,       -- increases_risk, decreases_risk
    rank            INTEGER NOT NULL,           -- 1 = most important
    plain_english   TEXT,                       -- "Login frequency dropped 60% vs cohort average"
    UNIQUE (prediction_id, feature_name)
);

CREATE INDEX idx_explanations_prediction ON prediction_explanations (prediction_id);
CREATE INDEX idx_explanations_top ON prediction_explanations (prediction_id, rank)
    WHERE rank <= 5;
```

## RFM Scoring

```sql
-- ============================================================
-- RFM SCORING
-- ============================================================

CREATE TABLE rfm_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    computed_date   DATE NOT NULL,
    recency_days    INTEGER NOT NULL,           -- days since last activity
    frequency_30d   INTEGER NOT NULL,           -- activity count in 30 days
    monetary_value  NUMERIC(12, 2),             -- revenue or value metric
    recency_score   INTEGER NOT NULL CHECK (recency_score BETWEEN 1 AND 5),
    frequency_score INTEGER NOT NULL CHECK (frequency_score BETWEEN 1 AND 5),
    monetary_score  INTEGER NOT NULL CHECK (monetary_score BETWEEN 1 AND 5),
    rfm_segment     VARCHAR(50) NOT NULL,       -- champion, loyal, at_risk, hibernating, etc.
    UNIQUE (tenant_id, user_id, computed_date)
);

CREATE INDEX idx_rfm_tenant_date ON rfm_scores (tenant_id, computed_date);
CREATE INDEX idx_rfm_segment ON rfm_scores (tenant_id, rfm_segment, computed_date);
```

## Dashboards & Reporting

```sql
-- ============================================================
-- DASHBOARDS & REPORTING
-- ============================================================

CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    layout          JSONB NOT NULL DEFAULT '[]',
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dashboard_widgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dashboard_id    UUID NOT NULL REFERENCES dashboards(id) ON DELETE CASCADE,
    widget_type     VARCHAR(50) NOT NULL,       -- retention_chart, funnel, cohort_table, churn_heatmap, rfm_grid
    title           VARCHAR(255),
    config          JSONB NOT NULL DEFAULT '{}',
    position_x      INTEGER NOT NULL DEFAULT 0,
    position_y      INTEGER NOT NULL DEFAULT 0,
    width           INTEGER NOT NULL DEFAULT 6,
    height          INTEGER NOT NULL DEFAULT 4,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_widgets_dashboard ON dashboard_widgets (dashboard_id);

-- Scheduled report delivery
CREATE TABLE scheduled_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    dashboard_id    UUID REFERENCES dashboards(id),
    name            VARCHAR(255) NOT NULL,
    schedule_cron   VARCHAR(100) NOT NULL,       -- cron expression
    delivery_channel VARCHAR(20) NOT NULL,       -- email, slack, webhook
    delivery_config JSONB NOT NULL DEFAULT '{}',
    -- Example: {"slack_channel": "#analytics", "webhook_url": "https://..."}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sent_at    TIMESTAMPTZ,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## NL Query & AI Features

```sql
-- ============================================================
-- NATURAL LANGUAGE QUERY & AI FEATURES
-- ============================================================

CREATE TABLE nl_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_account_id UUID NOT NULL,              -- platform user who asked
    query_text      TEXT NOT NULL,              -- "show me users who churned after trial"
    generated_sql   TEXT,                       -- SQL generated by LLM
    result_type     VARCHAR(50),               -- cohort, chart, table
    result_cohort_id UUID REFERENCES cohorts(id),
    execution_time_ms INTEGER,
    feedback        VARCHAR(20),               -- helpful, not_helpful, incorrect
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_nl_queries_tenant ON nl_queries (tenant_id, created_at DESC);

-- Proactive alert configurations
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    alert_type      VARCHAR(50) NOT NULL,       -- cohort_health_change, churn_spike, retention_drop
    cohort_id       UUID REFERENCES cohorts(id),
    metric          VARCHAR(100) NOT NULL,      -- retention_rate, churn_probability, active_users
    condition       VARCHAR(20) NOT NULL,       -- drops_below, rises_above, changes_by
    threshold       NUMERIC(10, 4) NOT NULL,
    lookback_window INTERVAL NOT NULL DEFAULT '7 days',
    delivery_channel VARCHAR(20) NOT NULL,      -- slack, email, webhook
    delivery_config JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    triggered_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    metric_value    NUMERIC(10, 4) NOT NULL,
    narrative       TEXT,                       -- LLM-generated explanation
    acknowledged    BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID,
    acknowledged_at TIMESTAMPTZ
);

CREATE INDEX idx_alert_history_tenant ON alert_history (tenant_id, triggered_at DESC);
```

## GDPR / Privacy Compliance

```sql
-- ============================================================
-- GDPR / PRIVACY COMPLIANCE
-- ============================================================

CREATE TABLE consent_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    consent_type    VARCHAR(50) NOT NULL,       -- analytics_tracking, data_processing, marketing
    granted         BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    ip_address      INET,
    user_agent      TEXT,
    source          VARCHAR(50) NOT NULL        -- sdk, api, manual
);

CREATE INDEX idx_consent_tenant_user ON consent_records (tenant_id, user_id);

CREATE TABLE data_deletion_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    requested_by    VARCHAR(255) NOT NULL,      -- email or identifier of requester
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, processing, completed, failed
    completed_at    TIMESTAMPTZ,
    tables_processed TEXT[],
    records_deleted INTEGER DEFAULT 0
);

CREATE INDEX idx_deletion_requests_status ON data_deletion_requests (tenant_id, status);
```

## Example Queries

### Day-N Retention Curve

```sql
-- Calculate Day-1, Day-7, Day-30 retention for users who signed up in April 2026
WITH signup_cohort AS (
    SELECT id AS user_id, tenant_id, first_seen_at::date AS signup_date
    FROM users
    WHERE tenant_id = :tenant_id
      AND first_seen_at >= '2026-04-01' AND first_seen_at < '2026-05-01'
),
activity AS (
    SELECT DISTINCT user_id, activity_date
    FROM daily_user_activity
    WHERE tenant_id = :tenant_id
)
SELECT
    sc.signup_date,
    COUNT(DISTINCT sc.user_id) AS cohort_size,
    COUNT(DISTINCT CASE WHEN a1.activity_date IS NOT NULL THEN sc.user_id END) AS day_1_retained,
    COUNT(DISTINCT CASE WHEN a7.activity_date IS NOT NULL THEN sc.user_id END) AS day_7_retained,
    COUNT(DISTINCT CASE WHEN a30.activity_date IS NOT NULL THEN sc.user_id END) AS day_30_retained
FROM signup_cohort sc
LEFT JOIN activity a1 ON sc.user_id = a1.user_id AND a1.activity_date = sc.signup_date + 1
LEFT JOIN activity a7 ON sc.user_id = a7.user_id AND a7.activity_date = sc.signup_date + 7
LEFT JOIN activity a30 ON sc.user_id = a30.user_id AND a30.activity_date = sc.signup_date + 30
GROUP BY sc.signup_date
ORDER BY sc.signup_date;
```

### Top Churn Risk Users with Explanations

```sql
-- Get top 20 at-risk users with their SHAP explanations
SELECT
    u.external_id,
    u.email,
    u.display_name,
    cp.churn_probability_30d,
    cp.risk_tier,
    ARRAY_AGG(pe.plain_english ORDER BY pe.rank) FILTER (WHERE pe.rank <= 3) AS top_reasons
FROM churn_predictions cp
JOIN users u ON cp.user_id = u.id
LEFT JOIN prediction_explanations pe ON cp.id = pe.prediction_id
WHERE cp.tenant_id = :tenant_id
  AND cp.prediction_date = CURRENT_DATE
  AND cp.risk_tier IN ('critical', 'high')
GROUP BY u.external_id, u.email, u.display_name, cp.churn_probability_30d, cp.risk_tier
ORDER BY cp.churn_probability_30d DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Identity | 2 | Tenant organization and membership |
| Tracked Users & Accounts | 4 | End-user identity, properties, B2B accounts, memberships |
| Event Tracking | 3 | Events (partitioned), event types, event properties |
| Cohort Management | 3 | Cohort definitions, memberships, AI-discovered cohorts |
| Retention & Funnels | 5 | Daily activity, funnels, funnel steps, retention reports, retention data |
| ML & Churn Prediction | 4 | Model registry, feature store, predictions, SHAP explanations |
| RFM Scoring | 1 | Precomputed RFM segments |
| Dashboards & Reporting | 3 | Dashboards, widgets, scheduled reports |
| NL Query & AI | 3 | NL queries, alert rules, alert history |
| GDPR Compliance | 2 | Consent records, deletion requests |
| **Total** | **30** | |

---

## Key Design Decisions

1. **Event properties as separate rows (EAV)** rather than JSONB: Enables indexing on any property name/value combination and ensures property types are enforced. The trade-off is more complex queries (joins) and higher write amplification per event.

2. **Time-partitioned events table**: Monthly partitioning on `timestamp` enables efficient partition pruning for time-range queries and straightforward data lifecycle management (drop old partitions for retention policies).

3. **Materialized retention data**: Day-N retention is precomputed into `retention_data` by a background job rather than calculated on-the-fly. This avoids expensive full-table scans on every dashboard load.

4. **Separate ML feature store table**: Features are snapshotted at prediction time so that SHAP explanations reference the exact feature values used during inference. This prevents data leakage and enables historical model auditing.

5. **Row-level multi-tenancy via `tenant_id`**: Every table includes `tenant_id` as part of its indexes. This is the simplest multi-tenant pattern and works well up to hundreds of tenants with row-level security policies.

6. **SHAP explanations stored per-prediction with plain English**: Each prediction stores its top feature attributions with human-readable text, enabling the frontend to display actionable explanations without re-running SHAP at query time.

7. **Dual user identity tables**: Platform users (tenant_members who log into the analytics dashboard) are separated from tracked end-users (the customers of our customers). This avoids confusion between the two identity domains.

8. **Cohort definition as JSONB within a strongly-typed wrapper**: While the rest of the schema is normalized, cohort filter definitions are stored as JSONB because their structure is recursive (AND/OR trees of conditions). The surrounding metadata (type, member_count, status) remains in typed columns.

9. **B2B account-level aggregation via explicit join tables**: `account_memberships` links users to accounts, enabling account-level health scoring by aggregating user-level features through the join.

10. **GDPR compliance as first-class tables**: Consent and deletion request tracking are not afterthoughts — they have dedicated tables with audit trails, supporting right-to-erasure workflows required by EU regulation.
