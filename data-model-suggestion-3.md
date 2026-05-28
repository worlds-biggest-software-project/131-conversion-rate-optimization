# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Conversion Rate Optimization · Created: 2026-05-19

## Philosophy

This model uses a pragmatic middle ground: core structural relationships (experiments, variants, visitors, sessions) are modeled as conventional relational tables with typed columns and foreign keys, but flexible, variable, or rapidly-evolving data is stored in JSONB columns within those tables. This avoids the schema explosion of full normalization while retaining the query power and integrity guarantees of PostgreSQL for the parts of the domain that are stable.

The hybrid approach is particularly well-suited to CRO platforms because the domain has both stable structures (an experiment always has variants, a session always has a visitor) and highly variable structures (event properties differ per event type, targeting rules vary by experiment, visitor attributes are customer-defined, heatmap interaction details depend on the interaction type). Rather than creating a separate table for every property bag or a separate column for every possible attribute, JSONB columns absorb the variability.

PostgreSQL's JSONB support is mature: GIN indexes enable fast containment queries, jsonb_path_query supports SQL/JSON path expressions, and partial indexes on JSONB fields provide targeted performance. This is the approach used internally by platforms like GrowthBook, which explicitly supports "a single events table with JSON fields or something in between."

**Best for:** Teams building an MVP or iterating rapidly, where schema flexibility is more important than strict normalization, and where the team has PostgreSQL expertise.

**Trade-offs:**
- Pro: Fewer tables (~20-25) than normalized model; simpler to understand and maintain
- Pro: Flexible event properties, targeting rules, and visitor attributes without schema migrations
- Pro: Fast iteration; add new fields to JSONB without ALTER TABLE
- Pro: Single database technology (PostgreSQL); no additional infrastructure
- Pro: GIN indexes on JSONB provide excellent query performance for containment queries
- Con: JSONB fields are opaque to the database schema; documentation must compensate
- Con: No referential integrity within JSONB structures
- Con: Complex JSONB queries can be slower than joins on indexed relational columns
- Con: JSONB fields can become catch-all dumping grounds without discipline
- Con: Reporting queries on JSONB fields require jsonb extraction operators, which are less intuitive

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFeature Specification | Experiment and variant tables model OpenFeature flag/variant concepts; evaluation context stored as JSONB |
| Segment Spec (Track/Identify/Page) | Event properties stored in JSONB following Segment naming conventions |
| CloudEvents 1.0 | Event metadata JSONB column follows CloudEvents envelope structure |
| W3C Navigation Timing Level 2 | Performance metrics stored as JSONB object within page_view, keyed to Navigation Timing attributes |
| W3C Largest Contentful Paint | LCP included in performance_metrics JSONB |
| GDPR / CCPA | Consent stored as structured JSONB with per-purpose granularity |
| IAB GPP | GPP consent string stored within consent JSONB |
| JSON Schema 2020-12 | Application-level validation of JSONB payloads against JSON Schema definitions |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "data_retention_days": 90,
    --   "max_experiments": 50,
    --   "features": ["heatmaps", "recordings", "ai_insights"],
    --   "branding": {"logo_url": "...", "primary_color": "#..."}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    domain          VARCHAR(255),
    snippet_key     VARCHAR(64) NOT NULL UNIQUE,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "recording_sample_rate": 0.5,
    --   "heatmap_tile_size": 20,
    --   "consent_required": true,
    --   "excluded_ips": ["10.0.0.0/8"],
    --   "custom_dimensions": ["plan_type", "industry", "company_size"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_project_org ON project(organization_id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    password_hash   VARCHAR(255),
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "default_project_id": "...",
    --   "notification_channels": ["email", "slack"],
    --   "dashboard_layout": "compact"
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);
```

## Visitor Identity & Consent

```sql
CREATE TABLE visitor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    anonymous_id    VARCHAR(255) NOT NULL,
    user_id         VARCHAR(255),
    traits          JSONB NOT NULL DEFAULT '{}',
    -- traits example (Segment Spec Identify traits):
    -- {
    --   "device_type": "desktop",
    --   "browser": "Chrome 126",
    --   "os": "macOS 15.4",
    --   "country": "US",
    --   "region": "California",
    --   "city": "San Francisco",
    --   "plan": "pro",
    --   "company": "Acme Corp",
    --   "custom": {"industry": "fintech", "team_size": 25}
    -- }
    consent         JSONB NOT NULL DEFAULT '{}',
    -- consent example:
    -- {
    --   "analytics": {"status": "granted", "granted_at": "...", "legal_basis": "consent"},
    --   "session_recording": {"status": "granted", "granted_at": "..."},
    --   "personalization": {"status": "denied"},
    --   "gpp_string": "DBACNYA~CPXxRfAPXxRfAAfKABENB-CgAAAAAAAAAAYgAAAAAAAA"
    -- }
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_visitor_anon ON visitor(project_id, anonymous_id);
CREATE INDEX idx_visitor_user ON visitor(project_id, user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_visitor_traits ON visitor USING GIN (traits);
CREATE INDEX idx_visitor_consent ON visitor USING GIN (consent);
```

## Experimentation

```sql
CREATE TABLE experiment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    hypothesis_id   UUID,                        -- FK to hypothesis table
    type            VARCHAR(50) NOT NULL,        -- ab, multivariate, multi_page, server_side
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "allocation_method": "fixed",           // or "bandit_thompson", "bandit_ucb"
    --   "traffic_percentage": 100.0,
    --   "statistical_method": "frequentist",    // or "bayesian"
    --   "confidence_level": 0.95,
    --   "min_sample_size": 1000,
    --   "min_runtime_hours": 168,
    --   "targeting": [
    --     {"rule_type": "url", "operator": "matches", "value": "/pricing*"},
    --     {"rule_type": "visitor_attribute", "attribute": "plan", "operator": "equals", "value": "free"}
    --   ],
    --   "schedule": {"start": "2026-06-01T00:00:00Z", "end": null}
    -- }
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(project_id, slug)
);
CREATE INDEX idx_experiment_project ON experiment(project_id);
CREATE INDEX idx_experiment_status ON experiment(project_id, status);
CREATE INDEX idx_experiment_config ON experiment USING GIN (config);

CREATE TABLE variant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,       -- OpenFeature variant key
    is_control      BOOLEAN NOT NULL DEFAULT FALSE,
    traffic_weight  NUMERIC(5,2) NOT NULL DEFAULT 0,
    changes         JSONB NOT NULL DEFAULT '{}',
    -- changes example:
    -- {
    --   "css": ".cta-button { background: #ff6600; }",
    --   "js": "document.querySelector('.headline').textContent = 'New Headline';",
    --   "html_patches": [
    --     {"selector": ".hero-title", "action": "replace", "content": "<h1>Better Title</h1>"}
    --   ],
    --   "server_config": {"feature_x": true, "pricing_display": "annual"}
    -- }
    description     TEXT,
    -- Live statistics (updated by background processor)
    stats           JSONB NOT NULL DEFAULT '{}',
    -- stats example:
    -- {
    --   "sample_size": 5432,
    --   "conversions": 287,
    --   "conversion_rate": 0.0528,
    --   "revenue_total": 14350.00,
    --   "confidence": 0.9723,
    --   "uplift": 0.0312,
    --   "is_winner": false,
    --   "bandit": {"alpha": 288.0, "beta": 5145.0, "allocated_weight": 35.2}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(experiment_id, slug)
);
CREATE INDEX idx_variant_experiment ON variant(experiment_id);

CREATE TABLE experiment_goal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    config          JSONB NOT NULL,
    -- config example:
    -- {
    --   "goal_type": "custom_event",
    --   "event_name": "Purchase Completed",
    --   "revenue_property": "total",
    --   "filters": [{"property": "currency", "operator": "equals", "value": "USD"}]
    -- }
    -- or:
    -- {
    --   "goal_type": "click",
    --   "selector": ".add-to-cart",
    --   "url_pattern": "/product/*"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_goal_experiment ON experiment_goal(experiment_id);

CREATE TABLE experiment_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id),
    variant_id      UUID NOT NULL REFERENCES variant(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    bucket_value    INTEGER NOT NULL,
    context         JSONB,                       -- OpenFeature evaluation context snapshot
    -- context example:
    -- {
    --   "targeting_key": "visitor-uuid",
    --   "device_type": "mobile",
    --   "country": "DE",
    --   "url": "/pricing"
    -- }
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(experiment_id, visitor_id)
);
CREATE INDEX idx_assignment_experiment ON experiment_assignment(experiment_id);
CREATE INDEX idx_assignment_visitor ON experiment_assignment(visitor_id);
```

## Hypothesis Management

```sql
CREATE TABLE hypothesis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    source          VARCHAR(50) NOT NULL,        -- manual, ai_generated, heatmap_insight, session_insight
    scoring         JSONB NOT NULL DEFAULT '{}',
    -- scoring example:
    -- {
    --   "lift": {
    --     "value_proposition": 7, "relevance": 8, "clarity": 6,
    --     "anxiety": 4, "distraction": 3, "urgency": 5
    --   },
    --   "pie": {"potential": 8, "importance": 7, "ease": 6},
    --   "priority_score": 7.0,
    --   "ai_confidence": 0.82,
    --   "predicted_uplift": 0.045,
    --   "model_version": "hypothesis-scorer-v3"
    -- }
    status          VARCHAR(50) NOT NULL DEFAULT 'proposed',
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hypothesis_project ON hypothesis(project_id);
CREATE INDEX idx_hypothesis_status ON hypothesis(project_id, status);

ALTER TABLE experiment ADD CONSTRAINT fk_experiment_hypothesis
    FOREIGN KEY (hypothesis_id) REFERENCES hypothesis(id);
```

## Sessions & Events

```sql
CREATE TABLE session (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ,
    duration_ms     INTEGER,
    page_count      INTEGER NOT NULL DEFAULT 0,
    event_count     INTEGER NOT NULL DEFAULT 0,
    is_bounced      BOOLEAN,
    entry_url       VARCHAR(2000),
    exit_url        VARCHAR(2000),
    attribution     JSONB NOT NULL DEFAULT '{}',
    -- attribution example:
    -- {
    --   "referrer": "https://google.com",
    --   "utm_source": "google", "utm_medium": "cpc",
    --   "utm_campaign": "summer-sale", "utm_term": "cro tool",
    --   "landing_page": "/pricing",
    --   "gclid": "abc123"
    -- }
    friction        JSONB NOT NULL DEFAULT '{}',
    -- friction example (aggregated signals):
    -- {
    --   "score": 72.5,
    --   "rage_clicks": 3,
    --   "dead_clicks": 7,
    --   "u_turns": 1,
    --   "js_errors": 2,
    --   "speed_browsing": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_session_project ON session(project_id, started_at);
CREATE INDEX idx_session_visitor ON session(visitor_id);
CREATE INDEX idx_session_friction ON session USING GIN (friction);

-- Unified event table for all visitor interactions
-- This is the high-volume table; consider partitioning by project_id + month
CREATE TABLE event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    session_id      UUID NOT NULL REFERENCES session(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    event_type      VARCHAR(100) NOT NULL,       -- page_view, click, scroll, form_field_focus,
                                                  -- form_submit, custom, rage_click, js_error, conversion
    event_name      VARCHAR(255),                -- Segment Spec: "Product Added", "Checkout Started"
    url             VARCHAR(2000),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties vary by event_type:
    --
    -- page_view:
    -- {
    --   "title": "Pricing - Acme",
    --   "viewport": {"width": 1920, "height": 1080},
    --   "scroll_depth_max": 85.5,
    --   "time_on_page_ms": 45000,
    --   "performance": {
    --     "ttfb_ms": 120, "dom_ready_ms": 450, "load_time_ms": 890,
    --     "lcp_ms": 650, "fid_ms": 12, "cls": 0.05
    --   }
    -- }
    --
    -- click:
    -- {
    --   "selector": ".cta-button",
    --   "element_text": "Start Free Trial",
    --   "x": 450, "y": 320,
    --   "is_rage": false, "is_dead": false
    -- }
    --
    -- form_field_focus / form_field_blur:
    -- {
    --   "form_selector": "#signup-form",
    --   "field_selector": "#email",
    --   "field_name": "email",
    --   "time_in_field_ms": 3200,
    --   "hesitation_ms": 800,
    --   "corrections": 2
    -- }
    --
    -- conversion:
    -- {
    --   "experiment_id": "...",
    --   "variant_id": "...",
    --   "goal_id": "...",
    --   "revenue": 99.00,
    --   "currency": "USD"
    -- }
    --
    -- js_error:
    -- {
    --   "message": "TypeError: Cannot read property 'x' of null",
    --   "stack": "...",
    --   "line": 42,
    --   "column": 15,
    --   "filename": "app.js"
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

-- Monthly partitions
CREATE TABLE event_2026_05 PARTITION OF event
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE event_2026_06 PARTITION OF event
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_event_session ON event(session_id);
CREATE INDEX idx_event_project_type ON event(project_id, event_type, occurred_at);
CREATE INDEX idx_event_name ON event(project_id, event_name, occurred_at) WHERE event_name IS NOT NULL;
CREATE INDEX idx_event_properties ON event USING GIN (properties);
```

## Heatmaps & Session Recordings

```sql
CREATE TABLE heatmap (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    url_pattern     VARCHAR(2000) NOT NULL,
    heatmap_type    VARCHAR(50) NOT NULL,        -- click, scroll, movement, attention, rage_click
    device_type     VARCHAR(50) NOT NULL DEFAULT 'all',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "viewport_width": 1920, "viewport_height": 1080,
    --   "tile_size": 20,
    --   "sample_count": 15432,
    --   "period_start": "2026-05-01", "period_end": "2026-05-19"
    -- }
    tile_data       JSONB NOT NULL DEFAULT '[]',
    -- tile_data: pre-aggregated tile array for fast rendering
    -- [
    --   {"x": 0, "y": 0, "count": 45, "intensity": 0.12},
    --   {"x": 1, "y": 0, "count": 230, "intensity": 0.67},
    --   ...
    -- ]
    status          VARCHAR(50) NOT NULL DEFAULT 'collecting',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_heatmap_project ON heatmap(project_id);
CREATE INDEX idx_heatmap_url ON heatmap(project_id, url_pattern, heatmap_type);

CREATE TABLE session_recording (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES session(id),
    project_id      UUID NOT NULL REFERENCES project(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    duration_ms     INTEGER,
    page_count      INTEGER NOT NULL DEFAULT 0,
    event_count     INTEGER NOT NULL DEFAULT 0,
    flags           JSONB NOT NULL DEFAULT '{}',
    -- flags example:
    -- {
    --   "has_rage_clicks": true, "rage_click_count": 3,
    --   "has_dead_clicks": true, "dead_click_count": 7,
    --   "has_js_errors": false,
    --   "has_form_abandons": true,
    --   "frustration_score": 72.5,
    --   "notable_moments": [
    --     {"offset_ms": 15000, "type": "rage_click", "selector": ".submit-btn"},
    --     {"offset_ms": 42000, "type": "form_abandon", "form": "#checkout"}
    --   ]
    -- }
    storage_url     VARCHAR(2000),
    storage_size_bytes BIGINT,
    status          VARCHAR(50) NOT NULL DEFAULT 'recording',
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recording_project ON session_recording(project_id, created_at);
CREATE INDEX idx_recording_session ON session_recording(session_id);
CREATE INDEX idx_recording_flags ON session_recording USING GIN (flags);

-- Query example: find recordings with rage clicks on a specific element
-- SELECT * FROM session_recording
-- WHERE project_id = '...'
--   AND flags @> '{"has_rage_clicks": true}'
-- ORDER BY (flags->>'frustration_score')::NUMERIC DESC
-- LIMIT 20;
```

## Conversion Funnels

```sql
CREATE TABLE funnel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    steps           JSONB NOT NULL,
    -- steps example:
    -- [
    --   {"step": 1, "name": "Landing Page", "type": "page_view", "url": "/landing*"},
    --   {"step": 2, "name": "Product View", "type": "page_view", "url": "/product/*"},
    --   {"step": 3, "name": "Add to Cart", "type": "custom_event", "event_name": "Product Added"},
    --   {"step": 4, "name": "Checkout", "type": "page_view", "url": "/checkout"},
    --   {"step": 5, "name": "Purchase", "type": "custom_event", "event_name": "Order Completed"}
    -- ]
    stats           JSONB NOT NULL DEFAULT '{}',
    -- stats: pre-computed funnel statistics (updated by background job)
    -- {
    --   "period": "2026-05-01/2026-05-19",
    --   "total_entered": 12500,
    --   "total_completed": 875,
    --   "overall_rate": 0.07,
    --   "steps": [
    --     {"step": 1, "entered": 12500, "completed": 8200, "drop_off_rate": 0.344},
    --     {"step": 2, "entered": 8200, "completed": 4100, "drop_off_rate": 0.50},
    --     ...
    --   ]
    -- }
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_funnel_project ON funnel(project_id);
```

## Segments & Personalization

```sql
CREATE TABLE segment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    segment_type    VARCHAR(50) NOT NULL,        -- rule_based, ai_predicted, behavioral
    rules           JSONB NOT NULL DEFAULT '[]',
    -- rules example:
    -- [
    --   {"group": 0, "attribute": "traits.country", "operator": "in", "value": ["US", "CA", "GB"]},
    --   {"group": 0, "attribute": "traits.plan", "operator": "equals", "value": "enterprise"},
    --   {"group": 1, "attribute": "session.page_count", "operator": "gte", "value": 5}
    -- ]
    -- Groups are OR'd; rules within a group are AND'd
    visitor_count   INTEGER NOT NULL DEFAULT 0,
    is_dynamic      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_segment_project ON segment(project_id);

CREATE TABLE personalization_campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    segment_id      UUID REFERENCES segment(id),
    priority        INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    changes         JSONB NOT NULL DEFAULT '{}',
    -- changes example (same structure as variant.changes):
    -- {
    --   "css": "...", "js": "...",
    --   "html_patches": [...],
    --   "server_config": {...}
    -- }
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_personalization_project ON personalization_campaign(project_id);
```

## AI & Insights

```sql
CREATE TABLE ai_insight (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    insight_type    VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    severity        VARCHAR(20),
    status          VARCHAR(50) NOT NULL DEFAULT 'new',
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "estimated_revenue_impact": 12500.00,
    --   "related_experiment_id": "...",
    --   "related_funnel_id": "...",
    --   "related_url": "/checkout",
    --   "confidence": 0.87,
    --   "model_version": "insight-gen-v4",
    --   "evidence": [
    --     {"type": "heatmap", "finding": "87% of clicks miss the CTA on mobile"},
    --     {"type": "funnel", "finding": "42% drop-off at checkout step 2"}
    --   ],
    --   "suggested_hypothesis": {
    --     "title": "Enlarging mobile CTA will increase checkout conversion",
    --     "predicted_uplift": 0.035
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_insight_project ON ai_insight(project_id, created_at);
CREATE INDEX idx_insight_type ON ai_insight(project_id, insight_type, status);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,                       -- {"before": {...}, "after": {...}}
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {"ip_address": "1.2.3.4", "user_agent": "...", "request_id": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example Queries

### Find visitors matching a segment rule (JSONB containment)

```sql
SELECT v.id, v.anonymous_id, v.traits
FROM visitor v
WHERE v.project_id = 'project-uuid'
  AND v.traits @> '{"country": "US"}'
  AND v.traits @> '{"custom": {"industry": "fintech"}}'
  AND v.consent @> '{"analytics": {"status": "granted"}}';
```

### Funnel drop-off analysis from the unified event table

```sql
WITH step1 AS (
    SELECT DISTINCT visitor_id
    FROM event
    WHERE project_id = 'project-uuid'
      AND event_type = 'page_view'
      AND url LIKE '/product/%'
      AND occurred_at >= '2026-05-01'
),
step2 AS (
    SELECT DISTINCT e.visitor_id
    FROM event e
    JOIN step1 s1 ON e.visitor_id = s1.visitor_id
    WHERE e.project_id = 'project-uuid'
      AND e.event_name = 'Product Added'
      AND e.occurred_at >= '2026-05-01'
)
SELECT
    (SELECT COUNT(*) FROM step1) AS step1_count,
    (SELECT COUNT(*) FROM step2) AS step2_count,
    ROUND((SELECT COUNT(*) FROM step2)::NUMERIC / NULLIF((SELECT COUNT(*) FROM step1), 0), 4) AS conversion_rate;
```

### Find high-frustration recordings with specific friction types

```sql
SELECT sr.id, sr.session_id, sr.duration_ms,
       sr.flags->>'frustration_score' AS frustration,
       sr.flags->'notable_moments' AS moments
FROM session_recording sr
WHERE sr.project_id = 'project-uuid'
  AND sr.flags @> '{"has_rage_clicks": true}'
  AND (sr.flags->>'frustration_score')::NUMERIC > 70
ORDER BY (sr.flags->>'frustration_score')::NUMERIC DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organization, project, app_user |
| Visitor & Consent | 1 | visitor (consent in JSONB) |
| Experimentation | 4 | experiment, variant, experiment_goal, experiment_assignment |
| Hypothesis | 1 | hypothesis (scoring in JSONB) |
| Sessions & Events | 2 | session, event (partitioned) |
| Heatmaps | 1 | heatmap (tile_data in JSONB) |
| Session Recording | 1 | session_recording (flags in JSONB) |
| Funnels | 1 | funnel (steps and stats in JSONB) |
| Segments & Personalization | 2 | segment, personalization_campaign |
| AI & Insights | 1 | ai_insight |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **~18** | Plus monthly partitions for event and audit_log |

---

## Key Design Decisions

1. **JSONB for variable structures, relational for stable ones** — experiment/variant/session/visitor are stable domain concepts that deserve typed columns. Event properties, targeting rules, scoring details, and tile data vary widely and live in JSONB. This reduces the table count from ~36 (normalized) to ~18.

2. **Unified event table** — instead of separate tables for page views, clicks, form interactions, and custom events, a single `event` table with `event_type` and a JSONB `properties` column handles all interaction types. This simplifies ingestion and funnel queries while using GIN indexes for property filtering.

3. **GIN indexes on JSONB** — PostgreSQL GIN indexes on `visitor.traits`, `session.friction`, `event.properties`, and `session_recording.flags` enable fast containment queries (`@>`) without extracting JSONB into separate columns or tables.

4. **Heatmap tile data as JSONB array** — pre-aggregated tile data stored as a JSONB array in the heatmap table. The frontend fetches one row and renders directly from the JSON, avoiding thousands of tile rows. Regenerated by background processors.

5. **Funnel steps as JSONB array** — funnel step definitions and pre-computed statistics live in JSONB columns on the funnel table, eliminating funnel_step and funnel_visitor_progress tables. The trade-off is that individual visitor funnel progress requires querying the event table rather than a dedicated table.

6. **Variant stats as JSONB** — live experiment statistics (sample size, conversions, confidence, bandit parameters) are stored as a JSONB column on the variant table, updated by background processors. This avoids a separate results table and keeps the variant's current state in one place.

7. **Event table partitioned by time** — the highest-volume table is partitioned monthly, enabling efficient time-range queries and archival of old partitions.

8. **JSON Schema validation at application layer** — JSONB columns are validated against JSON Schema definitions in the application code (not the database), per JSON Schema 2020-12. This provides schema documentation and validation without database-level constraints on JSONB structure.

9. **Consent as nested JSONB** — per-purpose consent (analytics, recording, personalization) with timestamps and GPP strings stored as a nested JSONB object on the visitor table. This is simpler than a separate consent table and supports arbitrary consent types without schema changes.

10. **Friction signals aggregated on session** — instead of a separate friction_event table, friction signals are aggregated into a JSONB column on the session table. Individual friction events can be found by querying the event table with `event_type IN ('rage_click', 'dead_click', 'js_error')`.
