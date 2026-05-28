# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Customer Cohort Analytics · Created: 2026-05-11

## Philosophy

This model takes a pragmatic middle ground: core structural fields (identifiers, timestamps, foreign keys, status columns) live in strongly-typed relational columns with proper indexes, while variable or customer-specific data (event properties, user traits, ML feature vectors, SHAP explanations) lives in JSONB columns on the same rows. This avoids the table explosion of a fully normalized model and the complexity of event sourcing, while retaining the queryability advantages of relational design.

The hybrid approach mirrors how modern product analytics platforms actually store data. PostHog stores event properties in a flexible column format alongside fixed columns for timestamp, user ID, and event name. Mixpanel's internal architecture uses a similar pattern: fixed columns for the event envelope, flexible storage for arbitrary properties. PostgreSQL's JSONB type provides GIN indexing, containment queries (`@>`), and path-based access (`->>`), making it practical to query arbitrary event properties without schema migrations.

This is the fastest path to an MVP. A small team can model the entire domain in 15-20 tables, ingest events with arbitrary properties from day one, and defer decisions about which properties deserve their own columns until query patterns stabilize. The JSONB columns act as a schema-evolution escape valve: new properties appear instantly in the data without migration, and frequently-queried properties can be promoted to materialized columns later.

**Best for:** Early-stage teams building an MVP, products that must ingest events with widely varying property schemas across customers, and teams that want PostgreSQL-only deployment without a separate analytics database.

**Trade-offs:**
- (+) Fastest path to MVP: fewer tables, no EAV joins, no event replay infrastructure
- (+) Flexible: new event properties require zero schema changes
- (+) Single database (PostgreSQL): no ClickHouse, no Kafka, simpler ops
- (+) JSONB is queryable: GIN indexes, containment queries, path expressions
- (+) Progressive refinement: promote hot JSONB paths to materialized columns as needed
- (-) JSONB queries slower than dedicated columns for high-cardinality filtering
- (-) No compile-time type checking on JSONB fields; schema enforcement is application-level
- (-) Large JSONB blobs increase row size and can degrade sequential scan performance
- (-) Less rigorous audit trail than event sourcing
- (-) May hit PostgreSQL performance limits at very high event volumes (>100M events/day)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment Spec (`.track()`, `.identify()`, `.page()`, `.group()`) | Segment call types map to `event_type` enum; common Segment fields (anonymous_id, context) are fixed columns; custom properties go into `properties` JSONB |
| ISO 8601 | All timestamps as `TIMESTAMPTZ` |
| ISO 3166-1/2 | Country/region codes in user `traits` JSONB and as indexed columns |
| GDPR / CCPA | `users.anonymized_at` flag; `consent_records` table; `data_deletion_requests` table |
| JSON Schema (IETF draft) | Optional event property validation via `event_schemas.json_schema` column |
| SHAP | Prediction explanations stored as JSONB array in `churn_predictions.explanations` |
| RFM | Precomputed RFM scores in dedicated table with typed columns |

---

## Multi-Tenancy & Platform Identity

```sql
-- ============================================================
-- MULTI-TENANCY
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {"data_region": "eu", "retention_days": 365, "event_quota": 10000000,
    --  "features": {"ml_predictions": true, "nl_queries": true}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    auth_provider   VARCHAR(50),               -- google, github, email
    auth_subject    VARCHAR(255),              -- provider-specific subject ID
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

## Tracked Users & Accounts

```sql
-- ============================================================
-- TRACKED END-USERS
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    anonymous_ids   TEXT[] NOT NULL DEFAULT '{}',    -- all anonymous IDs merged into this user
    
    -- Common Segment identify traits as indexed columns
    email           VARCHAR(255),
    display_name    VARCHAR(255),
    plan_tier       VARCHAR(50),
    signup_source   VARCHAR(100),
    country_code    CHAR(2),                        -- ISO 3166-1
    
    -- All traits (including the above, plus custom ones) in JSONB
    traits          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {"email": "jane@acme.com", "name": "Jane Doe", "plan": "pro",
    --  "company": "Acme Inc", "role": "engineering_lead",
    --  "custom_onboarding_score": 85, "preferred_language": "en"}
    
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ,
    total_events    BIGINT NOT NULL DEFAULT 0,
    anonymized_at   TIMESTAMPTZ,                    -- GDPR
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_users_tenant_external ON users (tenant_id, external_id)
    WHERE external_id IS NOT NULL;
CREATE INDEX idx_users_tenant_first_seen ON users (tenant_id, first_seen_at);
CREATE INDEX idx_users_tenant_last_seen ON users (tenant_id, last_seen_at);
CREATE INDEX idx_users_tenant_plan ON users (tenant_id, plan_tier)
    WHERE plan_tier IS NOT NULL;
CREATE INDEX idx_users_tenant_country ON users (tenant_id, country_code)
    WHERE country_code IS NOT NULL;
-- GIN index on traits for JSONB containment queries
CREATE INDEX idx_users_traits ON users USING GIN (traits);

-- ============================================================
-- B2B ACCOUNTS
-- ============================================================

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,          -- Segment group_id
    name            VARCHAR(255),
    
    -- Common account traits as columns
    industry        VARCHAR(100),
    employee_count  INTEGER,
    plan_tier       VARCHAR(50),
    arr             NUMERIC(12, 2),
    country_code    CHAR(2),
    
    -- All traits in JSONB
    traits          JSONB NOT NULL DEFAULT '{}',
    
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

CREATE INDEX idx_accounts_traits ON accounts USING GIN (traits);

-- User-to-account membership
CREATE TABLE account_memberships (
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(100),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, account_id, user_id)
);
```

## Event Tracking

```sql
-- ============================================================
-- EVENTS — the core of the analytics platform
-- ============================================================

CREATE TABLE events (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    
    -- Segment spec fields (fixed columns, indexed)
    event_type      VARCHAR(20) NOT NULL,           -- track, identify, page, screen, group
    event_name      VARCHAR(255),                   -- for track events: "Purchase", "Signup"
    user_id         UUID,
    anonymous_id    VARCHAR(255),
    timestamp       TIMESTAMPTZ NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Commonly queried context fields (promoted from JSONB for performance)
    session_id      VARCHAR(255),
    page_url        TEXT,
    utm_source      VARCHAR(255),
    utm_medium      VARCHAR(255),
    utm_campaign    VARCHAR(255),
    device_type     VARCHAR(50),
    country_code    CHAR(2),
    
    -- All event-specific properties in JSONB
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for track "Purchase":
    -- {"product_id": "sku-123", "price": 29.99, "currency": "USD",
    --  "category": "shoes", "payment_method": "credit_card"}
    -- Example for page:
    -- {"title": "Pricing Page", "url": "/pricing", "referrer": "https://google.com"}
    
    -- Full Segment context in JSONB
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"ip": "1.2.3.4", "userAgent": "Mozilla/5.0...",
    --           "locale": "en-US", "library": {"name": "analytics.js", "version": "2.1.0"}}
    
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Monthly partitions
CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... automated partition management

-- Fixed-column indexes
CREATE INDEX idx_events_tenant_user_ts ON events (tenant_id, user_id, timestamp);
CREATE INDEX idx_events_tenant_name_ts ON events (tenant_id, event_name, timestamp)
    WHERE event_name IS NOT NULL;
CREATE INDEX idx_events_tenant_type_ts ON events (tenant_id, event_type, timestamp);
CREATE INDEX idx_events_session ON events (tenant_id, session_id)
    WHERE session_id IS NOT NULL;
CREATE INDEX idx_events_utm ON events (tenant_id, utm_source, timestamp)
    WHERE utm_source IS NOT NULL;

-- GIN index on properties for JSONB queries
CREATE INDEX idx_events_properties ON events USING GIN (properties);

-- ============================================================
-- EVENT SCHEMA REGISTRY (optional validation)
-- ============================================================

CREATE TABLE event_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    event_name      VARCHAR(255) NOT NULL,
    json_schema     JSONB,                          -- JSON Schema for properties validation
    schema_version  INTEGER NOT NULL DEFAULT 1,
    sample_count    BIGINT NOT NULL DEFAULT 0,
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, event_name, schema_version)
);
```

## Cohort Management

```sql
-- ============================================================
-- COHORTS
-- ============================================================

CREATE TABLE cohorts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) NOT NULL,           -- behavioral, temporal, demographic, predictive, ai_discovered
    is_dynamic      BOOLEAN NOT NULL DEFAULT true,
    
    -- Cohort filter definition — tree of AND/OR conditions
    definition      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "operator": "AND",
    --   "conditions": [
    --     {"type": "did_event", "event": "Purchase", "time_window": "30d",
    --      "property_filters": [{"key": "price", "op": ">", "value": 50}]},
    --     {"type": "user_trait", "key": "plan_tier", "op": "=", "value": "pro"},
    --     {"type": "not_did_event", "event": "Support Ticket", "time_window": "90d"}
    --   ]
    -- }
    
    member_count    INTEGER DEFAULT 0,
    last_computed   TIMESTAMPTZ,
    created_by      UUID,
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cohorts_tenant ON cohorts (tenant_id);

CREATE TABLE cohort_memberships (
    tenant_id       UUID NOT NULL,
    cohort_id       UUID NOT NULL REFERENCES cohorts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,
    PRIMARY KEY (tenant_id, cohort_id, user_id)
);

CREATE INDEX idx_cohort_members_active ON cohort_memberships (tenant_id, cohort_id)
    WHERE exited_at IS NULL;

-- AI-discovered cohorts
CREATE TABLE discovered_cohorts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    discovery_run_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    statistical_significance NUMERIC(5, 4),
    effect_size     NUMERIC(8, 4),
    member_count    INTEGER NOT NULL,
    distinguishing_features JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [{"feature": "used_feature_X_in_first_7d", "prevalence": 0.82, "cohort_avg": 0.23},
    --  {"feature": "invited_teammate", "prevalence": 0.71, "cohort_avg": 0.15}]
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    promoted_cohort_id UUID REFERENCES cohorts(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Retention & Funnels

```sql
-- ============================================================
-- DAILY ACTIVITY ROLLUP (materialized for performance)
-- ============================================================

CREATE TABLE daily_user_activity (
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    activity_date   DATE NOT NULL,
    event_count     INTEGER NOT NULL DEFAULT 0,
    session_count   INTEGER NOT NULL DEFAULT 0,
    distinct_events INTEGER NOT NULL DEFAULT 0,
    event_names     TEXT[],
    first_event_at  TIMESTAMPTZ,
    last_event_at   TIMESTAMPTZ,
    PRIMARY KEY (tenant_id, user_id, activity_date)
);

CREATE INDEX idx_daily_activity_date ON daily_user_activity (tenant_id, activity_date);

-- ============================================================
-- FUNNELS
-- ============================================================

CREATE TABLE funnels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    time_window     INTERVAL NOT NULL DEFAULT '7 days',
    steps           JSONB NOT NULL,
    -- Example:
    -- [{"order": 1, "event": "Viewed Pricing", "filters": {"properties.plan": "enterprise"}},
    --  {"order": 2, "event": "Started Trial"},
    --  {"order": 3, "event": "Activated"},
    --  {"order": 4, "event": "Subscribed"}]
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RETENTION REPORTS
-- ============================================================

CREATE TABLE retention_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    start_event     VARCHAR(255) NOT NULL,
    return_event    VARCHAR(255) NOT NULL,
    cohort_id       UUID REFERENCES cohorts(id),
    granularity     VARCHAR(20) NOT NULL DEFAULT 'day',
    lookback_periods INTEGER NOT NULL DEFAULT 12,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Precomputed retention data
CREATE TABLE retention_data (
    tenant_id       UUID NOT NULL,
    report_id       UUID NOT NULL REFERENCES retention_reports(id) ON DELETE CASCADE,
    cohort_period   DATE NOT NULL,
    period_offset   INTEGER NOT NULL,
    cohort_size     INTEGER NOT NULL,
    retained_count  INTEGER NOT NULL,
    retention_rate  NUMERIC(5, 4) NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, report_id, cohort_period, period_offset)
);
```

## ML Churn Prediction

```sql
-- ============================================================
-- ML MODELS & PREDICTIONS
-- ============================================================

CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    model_type      VARCHAR(50) NOT NULL,
    model_version   INTEGER NOT NULL,
    algorithm       VARCHAR(50) NOT NULL,
    
    -- Training config and results in JSONB (varies by algorithm)
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {"hyperparameters": {"n_estimators": 500, "max_depth": 6, "learning_rate": 0.05},
    --  "metrics": {"auc_roc": 0.932, "precision": 0.84, "recall": 0.78, "f1": 0.81},
    --  "feature_names": ["events_7d", "events_30d", "sessions_7d", ...],
    --  "training_rows": 45000, "training_window": "2025-01-01/2026-04-01"}
    
    trained_at      TIMESTAMPTZ NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT false,
    artifact_path   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, model_type, model_version)
);

CREATE INDEX idx_ml_models_active ON ml_models (tenant_id, model_type) WHERE is_active = true;

-- Feature snapshots for ML training and inference
CREATE TABLE ml_feature_snapshots (
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL,
    
    -- Fixed behavioral features as columns (for fast queries)
    events_7d       INTEGER,
    events_30d      INTEGER,
    events_90d      INTEGER,
    sessions_7d     INTEGER,
    sessions_30d    INTEGER,
    days_active_30d INTEGER,
    days_since_last_activity INTEGER,
    days_since_signup INTEGER,
    
    -- All features including custom/derived ones in JSONB
    features        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {"events_7d": 12, "events_30d": 45, "sessions_7d": 5,
    --  "avg_session_duration_30d": 342.5, "distinct_features_used_30d": 8,
    --  "event_trend_7d_vs_30d": 0.8, "support_tickets_30d": 0,
    --  "account_active_user_pct": 0.65, "nps_score": 8}
    
    PRIMARY KEY (tenant_id, user_id, computed_at)
);

-- Churn predictions with embedded SHAP explanations
CREATE TABLE churn_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    model_id        UUID NOT NULL REFERENCES ml_models(id),
    prediction_date DATE NOT NULL,
    
    -- Prediction scores
    churn_probability_30d NUMERIC(5, 4),
    churn_probability_60d NUMERIC(5, 4),
    churn_probability_90d NUMERIC(5, 4),
    risk_tier       VARCHAR(20) NOT NULL,
    
    -- SHAP explanations embedded in JSONB (no separate table needed)
    explanations    JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"rank": 1, "feature": "days_since_last_login", "shap_value": 0.31,
    --    "feature_value": 45, "direction": "increases_risk",
    --    "plain_english": "No login for 45 days (cohort avg: 3 days)"},
    --   {"rank": 2, "feature": "session_trend_7d_vs_30d", "shap_value": -0.22,
    --    "feature_value": -0.6, "direction": "increases_risk",
    --    "plain_english": "Session frequency dropped 60% over the last week"},
    --   {"rank": 3, "feature": "support_tickets_30d", "shap_value": 0.18,
    --    "feature_value": 5, "direction": "increases_risk",
    --    "plain_english": "Filed 5 support tickets in last 30 days (avg: 0.8)"}
    -- ]
    
    feature_snapshot_at TIMESTAMPTZ,            -- links to ml_feature_snapshots
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, user_id, prediction_date)
);

CREATE INDEX idx_predictions_tenant_date ON churn_predictions (tenant_id, prediction_date);
CREATE INDEX idx_predictions_risk ON churn_predictions (tenant_id, risk_tier, prediction_date);
```

## RFM Scoring

```sql
-- ============================================================
-- RFM SCORING
-- ============================================================

CREATE TABLE rfm_scores (
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    computed_date   DATE NOT NULL,
    recency_days    INTEGER NOT NULL,
    frequency_30d   INTEGER NOT NULL,
    monetary_value  NUMERIC(12, 2),
    recency_score   INTEGER NOT NULL CHECK (recency_score BETWEEN 1 AND 5),
    frequency_score INTEGER NOT NULL CHECK (frequency_score BETWEEN 1 AND 5),
    monetary_score  INTEGER NOT NULL CHECK (monetary_score BETWEEN 1 AND 5),
    rfm_segment     VARCHAR(50) NOT NULL,
    PRIMARY KEY (tenant_id, user_id, computed_date)
);

CREATE INDEX idx_rfm_segment ON rfm_scores (tenant_id, rfm_segment, computed_date);
```

## Dashboards, Alerts & NL Queries

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
    -- Example layout:
    -- [{"widget_id": "...", "x": 0, "y": 0, "w": 6, "h": 4},
    --  {"widget_id": "...", "x": 6, "y": 0, "w": 6, "h": 4}]
    is_default      BOOLEAN NOT NULL DEFAULT false,
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
    -- Example config for retention_chart:
    -- {"report_id": "...", "chart_type": "heatmap", "color_scale": "green_red"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scheduled_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    dashboard_id    UUID REFERENCES dashboards(id),
    name            VARCHAR(255) NOT NULL,
    schedule_cron   VARCHAR(100) NOT NULL,
    delivery        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"channel": "slack", "slack_channel": "#analytics"}
    -- Example: {"channel": "email", "recipients": ["team@acme.com"]}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sent_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ALERT RULES & HISTORY
-- ============================================================

CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    -- Example:
    -- {"alert_type": "retention_drop", "cohort_id": "...",
    --  "metric": "day_7_retention", "condition": "drops_below",
    --  "threshold": 0.30, "lookback": "7d",
    --  "delivery": {"channel": "slack", "slack_channel": "#alerts"}}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    triggered_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    metric_value    NUMERIC(10, 4) NOT NULL,
    narrative       TEXT,                           -- LLM-generated explanation
    context         JSONB NOT NULL DEFAULT '{}',    -- additional data about the alert
    acknowledged    BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID,
    acknowledged_at TIMESTAMPTZ
);

CREATE INDEX idx_alert_history_tenant ON alert_history (tenant_id, triggered_at DESC);

-- ============================================================
-- NATURAL LANGUAGE QUERIES
-- ============================================================

CREATE TABLE nl_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_account_id UUID NOT NULL,
    query_text      TEXT NOT NULL,
    generated_sql   TEXT,
    result_type     VARCHAR(50),
    result_data     JSONB,                         -- query results or cohort reference
    execution_time_ms INTEGER,
    feedback        VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_nl_queries_tenant ON nl_queries (tenant_id, created_at DESC);
```

## GDPR Compliance

```sql
-- ============================================================
-- GDPR / PRIVACY
-- ============================================================

CREATE TABLE consent_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    consent_type    VARCHAR(50) NOT NULL,
    granted         BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"ip": "1.2.3.4", "source": "sdk", "user_agent": "..."}
);

CREATE INDEX idx_consent_tenant_user ON consent_records (tenant_id, user_id);

CREATE TABLE data_deletion_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    requested_by    VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    completed_at    TIMESTAMPTZ,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"tables_processed": ["events", "users", "churn_predictions"],
    --           "records_deleted": 4521, "elapsed_ms": 12400}
);
```

## Example Queries

### JSONB Property Filtering in Cohort Queries

```sql
-- Find users who made a purchase over $100 in the last 30 days
-- AND have the "enterprise" plan trait
SELECT DISTINCT u.id, u.external_id, u.email
FROM users u
JOIN events e ON u.id = e.user_id AND u.tenant_id = e.tenant_id
WHERE u.tenant_id = :tenant_id
  AND e.event_name = 'Purchase'
  AND e.timestamp >= now() - interval '30 days'
  AND (e.properties->>'price')::numeric > 100
  AND u.traits @> '{"plan": "enterprise"}'::jsonb;
```

### JSONB Containment Query for Event Properties

```sql
-- Find events where properties contain a specific nested value
-- Uses GIN index for fast containment check
SELECT id, event_name, timestamp, properties
FROM events
WHERE tenant_id = :tenant_id
  AND event_name = 'Feature Used'
  AND properties @> '{"feature_name": "export", "format": "csv"}'::jsonb
  AND timestamp >= now() - interval '7 days'
ORDER BY timestamp DESC
LIMIT 100;
```

### Churn Predictions with Embedded SHAP Explanations

```sql
-- Top 20 at-risk users with their explanations (no join needed)
SELECT
    u.external_id,
    u.email,
    u.display_name,
    cp.churn_probability_30d,
    cp.risk_tier,
    cp.explanations
FROM churn_predictions cp
JOIN users u ON cp.user_id = u.id AND cp.tenant_id = u.tenant_id
WHERE cp.tenant_id = :tenant_id
  AND cp.prediction_date = CURRENT_DATE
  AND cp.risk_tier IN ('critical', 'high')
ORDER BY cp.churn_probability_30d DESC
LIMIT 20;

-- Extract just the plain English explanations as a text array
SELECT
    u.external_id,
    ARRAY(
        SELECT elem->>'plain_english'
        FROM jsonb_array_elements(cp.explanations) AS elem
        WHERE (elem->>'rank')::int <= 3
        ORDER BY (elem->>'rank')::int
    ) AS top_reasons
FROM churn_predictions cp
JOIN users u ON cp.user_id = u.id
WHERE cp.tenant_id = :tenant_id
  AND cp.prediction_date = CURRENT_DATE
  AND cp.risk_tier = 'critical';
```

### Promoting a Hot JSONB Path to a Materialized Column

```sql
-- When a specific event property is queried frequently, promote it
-- Step 1: Add the column
ALTER TABLE events ADD COLUMN prop_product_category VARCHAR(100);

-- Step 2: Backfill from JSONB
UPDATE events
SET prop_product_category = properties->>'product_category'
WHERE tenant_id = :tenant_id
  AND event_name = 'Purchase'
  AND properties ? 'product_category';

-- Step 3: Add index on the new column
CREATE INDEX idx_events_product_cat ON events (tenant_id, prop_product_category, timestamp)
    WHERE prop_product_category IS NOT NULL;

-- Step 4: Set up a trigger to auto-populate on insert
CREATE OR REPLACE FUNCTION extract_product_category()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.event_name = 'Purchase' AND NEW.properties ? 'product_category' THEN
        NEW.prop_product_category := NEW.properties->>'product_category';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_extract_product_category
    BEFORE INSERT ON events
    FOR EACH ROW EXECUTE FUNCTION extract_product_category();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy | 2 | Tenants, tenant members |
| Tracked Users & Accounts | 3 | Users (with JSONB traits), accounts (with JSONB traits), memberships |
| Event Tracking | 2 | Events (partitioned, with JSONB properties), event schemas |
| Cohort Management | 3 | Cohort definitions, memberships, AI-discovered cohorts |
| Retention & Funnels | 4 | Daily activity, funnels, retention reports, retention data |
| ML & Predictions | 3 | Model registry, feature snapshots (with JSONB), churn predictions (with JSONB SHAP) |
| RFM Scoring | 1 | RFM scores |
| Dashboards & Reporting | 3 | Dashboards, widgets, scheduled reports |
| Alerts & NL | 3 | Alert rules (JSONB config), alert history, NL queries |
| GDPR | 2 | Consent records, deletion requests |
| **Total** | **26** | Fewer tables than normalized model; JSONB absorbs what would be separate tables |

---

## Key Design Decisions

1. **JSONB for event properties, relational for event envelope**: The `events` table has fixed columns for commonly queried fields (event_name, user_id, timestamp, UTM parameters) and a `properties` JSONB column for everything else. This means no schema migration when a customer starts tracking a new event property.

2. **User traits as both columns and JSONB**: Frequently filtered traits (email, plan_tier, country_code) exist as indexed columns AND in the `traits` JSONB blob. The columns provide fast equality/range queries; the JSONB provides flexible containment queries for ad-hoc filtering. The application keeps them in sync.

3. **SHAP explanations embedded in prediction rows**: Instead of a separate `prediction_explanations` table (which would require joins), explanations are stored as a JSONB array directly on the `churn_predictions` row. This eliminates a join for the most common query pattern (show user + risk + reasons).

4. **GIN indexes on JSONB columns**: The `events.properties`, `users.traits`, and `accounts.traits` columns have GIN indexes enabling fast `@>` containment queries. This is essential for cohort filtering on arbitrary properties.

5. **Progressive column promotion**: When a JSONB property becomes frequently queried, it can be promoted to a dedicated column with a backfill + trigger. This allows the schema to evolve based on actual usage patterns rather than upfront guesses.

6. **Funnel steps as JSONB array**: Instead of a separate `funnel_steps` table, funnel step definitions are stored as a JSONB array on the `funnels` table. Funnels are always loaded as a unit, so there is no benefit to normalizing steps into their own table.

7. **Alert configuration as JSONB**: Alert rules store their full configuration (type, metric, threshold, delivery) in a single JSONB `config` column. This allows new alert types to be added without schema changes.

8. **Single database deployment**: The entire schema runs on PostgreSQL with no external dependencies. This is the simplest operational footprint and works well for tenants with up to ~50M events/month. Beyond that, consider Model 4 (dual-store with ClickHouse).

9. **Partitioned events table**: Monthly range partitioning on `timestamp` enables efficient time-range queries and partition-level data lifecycle management.

10. **ML feature snapshots with dual storage**: Commonly used ML features have dedicated columns for fast training data extraction, while the full feature vector lives in a `features` JSONB column. This supports both efficient batch training queries and flexible feature experimentation.
