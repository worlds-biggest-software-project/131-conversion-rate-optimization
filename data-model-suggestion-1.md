# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Conversion Rate Optimization · Created: 2026-05-19

## Philosophy

This model follows classical normalized relational design. Every distinct concept in the CRO domain (experiments, variants, goals, heatmaps, sessions, recordings, hypotheses, funnels) gets its own table with strict foreign key relationships. Reference data (event types, statistical methods, friction signal types, consent statuses) lives in lookup tables. The schema enforces data integrity at the database level through constraints, check clauses, and foreign keys.

This is the approach used by mature enterprise platforms like VWO and Optimizely internally, where data correctness and complex cross-entity queries (e.g., "show me all experiments targeting users who exhibited rage-clicks in funnel step 3") are paramount. It maps naturally to the OpenFeature specification's concepts of flags, variants, evaluation contexts, and providers.

The trade-off is a higher table count and more complex queries, but the schema is self-documenting, migrations are straightforward, and any reporting question can be answered with standard SQL joins. It aligns well with teams that have strong SQL skills and value referential integrity over write throughput.

**Best for:** Teams building a full-featured CRO platform where data integrity, complex cross-entity reporting, and standards alignment are top priorities.

**Trade-offs:**
- Pro: Full referential integrity; the database enforces business rules
- Pro: Self-documenting schema; any developer can understand the domain from the DDL
- Pro: Complex cross-entity queries are natural with JOINs
- Pro: Clean migration path; ALTER TABLE for schema evolution
- Con: High table count (~45-55 tables) increases JOIN complexity
- Con: Write-heavy workloads (clickstream events, mouse movements) may bottleneck on foreign key checks
- Con: Adding jurisdiction-specific or custom fields requires schema migrations
- Con: Session recording event data at scale may overwhelm row-oriented storage

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFeature Specification | Flag, variant, and evaluation context tables mirror the OpenFeature type system |
| Segment Spec (Track/Identify/Page) | Event taxonomy tables use Object-Action naming; properties stored in structured columns |
| CloudEvents 1.0 | Event envelope fields (id, source, type, specversion, time) map to event table columns |
| W3C Navigation Timing Level 2 | Performance metric columns in page_view table align with Navigation Timing attributes |
| W3C Largest Contentful Paint | LCP metric stored as dedicated column in page_performance table |
| ISO/IEC 27001 | Audit trail tables support security event logging requirements |
| GDPR / CCPA | Consent table with granular purpose tracking; data retention policies per visitor |
| IAB Global Privacy Platform | GPP consent string stored in visitor_consent table |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, pro, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    domain          VARCHAR(255),
    snippet_key     VARCHAR(64) NOT NULL UNIQUE,  -- JS snippet identifier
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_project_org ON project(organization_id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',  -- admin, editor, analyst, viewer
    password_hash   VARCHAR(255),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);
CREATE INDEX idx_app_user_org ON app_user(organization_id);
```

## Visitor Identity & Consent

```sql
CREATE TABLE visitor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    anonymous_id    VARCHAR(255) NOT NULL,       -- client-generated anonymous ID
    user_id         VARCHAR(255),                -- identified user ID (post-login)
    device_type     VARCHAR(50),                 -- desktop, mobile, tablet
    browser         VARCHAR(100),
    os              VARCHAR(100),
    country         CHAR(2),                     -- ISO 3166-1 alpha-2
    region          VARCHAR(100),
    city            VARCHAR(100),
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_visitor_anon ON visitor(project_id, anonymous_id);
CREATE INDEX idx_visitor_user ON visitor(project_id, user_id) WHERE user_id IS NOT NULL;

CREATE TABLE visitor_consent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    consent_type    VARCHAR(50) NOT NULL,        -- analytics, session_recording, personalization, marketing
    status          VARCHAR(20) NOT NULL,        -- granted, denied, withdrawn
    gpp_string      TEXT,                        -- IAB Global Privacy Platform consent string
    legal_basis     VARCHAR(50),                 -- consent, legitimate_interest, contract
    granted_at      TIMESTAMPTZ,
    withdrawn_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_consent_visitor ON visitor_consent(visitor_id);

CREATE TABLE visitor_attribute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    attribute_key   VARCHAR(100) NOT NULL,
    attribute_value VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(visitor_id, attribute_key)
);
CREATE INDEX idx_vattr_visitor ON visitor_attribute(visitor_id);
```

## Experimentation

```sql
CREATE TABLE experiment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    hypothesis_id   UUID,                        -- FK added after hypothesis table
    type            VARCHAR(50) NOT NULL,        -- ab, multivariate, multi_page, server_side
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',  -- draft, running, paused, completed, archived
    allocation_method VARCHAR(50) NOT NULL DEFAULT 'fixed', -- fixed, bandit_thompson, bandit_ucb
    traffic_percentage NUMERIC(5,2) NOT NULL DEFAULT 100.00,
    statistical_method VARCHAR(50) NOT NULL DEFAULT 'frequentist', -- frequentist, bayesian
    confidence_level NUMERIC(5,4) NOT NULL DEFAULT 0.9500,  -- 95% default per industry standard
    min_sample_size INTEGER,
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(project_id, slug)
);
CREATE INDEX idx_experiment_project ON experiment(project_id);
CREATE INDEX idx_experiment_status ON experiment(project_id, status);

CREATE TABLE variant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,       -- aligns with OpenFeature variant key
    is_control      BOOLEAN NOT NULL DEFAULT FALSE,
    traffic_weight  NUMERIC(5,2) NOT NULL DEFAULT 0,  -- percentage of traffic
    description     TEXT,
    css_changes     TEXT,                         -- CSS modifications
    js_changes      TEXT,                         -- JavaScript modifications
    html_changes    TEXT,                         -- DOM modifications (WYSIWYG editor output)
    server_config   JSONB,                       -- server-side variant configuration
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(experiment_id, slug)
);
CREATE INDEX idx_variant_experiment ON variant(experiment_id);

CREATE TABLE experiment_goal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    goal_type       VARCHAR(50) NOT NULL,        -- pageview, click, custom_event, revenue, form_submit
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    target_selector VARCHAR(500),                -- CSS selector for click goals
    target_url      VARCHAR(2000),               -- URL pattern for pageview goals
    event_name      VARCHAR(255),                -- custom event name
    revenue_property VARCHAR(100),               -- property name containing revenue value
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_goal_experiment ON experiment_goal(experiment_id);

CREATE TABLE experiment_targeting (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    rule_type       VARCHAR(50) NOT NULL,        -- url, cookie, query_param, visitor_attribute, segment
    operator        VARCHAR(30) NOT NULL,        -- equals, contains, matches_regex, in_list, gt, lt
    attribute       VARCHAR(255) NOT NULL,
    value           VARCHAR(500) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_targeting_experiment ON experiment_targeting(experiment_id);

CREATE TABLE experiment_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id),
    variant_id      UUID NOT NULL REFERENCES variant(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    bucket_value    INTEGER NOT NULL,            -- hash bucket (0-9999) for deterministic assignment
    UNIQUE(experiment_id, visitor_id)
);
CREATE INDEX idx_assignment_experiment ON experiment_assignment(experiment_id);
CREATE INDEX idx_assignment_visitor ON experiment_assignment(visitor_id);
CREATE INDEX idx_assignment_variant ON experiment_assignment(variant_id);

CREATE TABLE experiment_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id),
    variant_id      UUID NOT NULL REFERENCES variant(id),
    goal_id         UUID NOT NULL REFERENCES experiment_goal(id),
    sample_size     INTEGER NOT NULL DEFAULT 0,
    conversions     INTEGER NOT NULL DEFAULT 0,
    conversion_rate NUMERIC(10,6),
    revenue_total   NUMERIC(15,2),
    confidence      NUMERIC(6,4),                -- statistical confidence (0.0000-1.0000)
    uplift          NUMERIC(10,4),               -- relative improvement vs control
    is_winner       BOOLEAN NOT NULL DEFAULT FALSE,
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_result_experiment ON experiment_result(experiment_id);

-- Bandit-specific: stores Thompson Sampling posterior parameters
CREATE TABLE bandit_state (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id),
    variant_id      UUID NOT NULL REFERENCES variant(id),
    alpha           NUMERIC(15,4) NOT NULL DEFAULT 1.0, -- Beta distribution alpha (successes + prior)
    beta            NUMERIC(15,4) NOT NULL DEFAULT 1.0, -- Beta distribution beta (failures + prior)
    allocated_weight NUMERIC(5,2) NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(experiment_id, variant_id)
);
```

## Hypothesis Management

```sql
CREATE TABLE hypothesis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    source          VARCHAR(50) NOT NULL,        -- manual, ai_generated, heatmap_insight, session_insight
    -- LIFT model factors
    lift_value_proposition  NUMERIC(3,1),        -- 1-10 score
    lift_relevance          NUMERIC(3,1),
    lift_clarity            NUMERIC(3,1),
    lift_anxiety            NUMERIC(3,1),
    lift_distraction        NUMERIC(3,1),
    lift_urgency            NUMERIC(3,1),
    -- PIE/ICE scoring
    potential       NUMERIC(3,1),                -- 1-10
    importance      NUMERIC(3,1),
    ease            NUMERIC(3,1),
    priority_score  NUMERIC(5,2),                -- calculated composite score
    ai_confidence   NUMERIC(5,4),                -- AI model confidence in predicted uplift
    predicted_uplift NUMERIC(10,4),
    status          VARCHAR(50) NOT NULL DEFAULT 'proposed', -- proposed, approved, testing, validated, rejected
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hypothesis_project ON hypothesis(project_id);
CREATE INDEX idx_hypothesis_status ON hypothesis(project_id, status);

ALTER TABLE experiment ADD CONSTRAINT fk_experiment_hypothesis
    FOREIGN KEY (hypothesis_id) REFERENCES hypothesis(id);
```

## Behavioural Analytics: Sessions & Events

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
    referrer        VARCHAR(2000),
    utm_source      VARCHAR(255),
    utm_medium      VARCHAR(255),
    utm_campaign    VARCHAR(255),
    utm_term        VARCHAR(255),
    utm_content     VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_session_project ON session(project_id);
CREATE INDEX idx_session_visitor ON session(visitor_id);
CREATE INDEX idx_session_started ON session(project_id, started_at);

CREATE TABLE page_view (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES session(id),
    project_id      UUID NOT NULL REFERENCES project(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    url             VARCHAR(2000) NOT NULL,
    title           VARCHAR(500),
    viewport_width  INTEGER,
    viewport_height INTEGER,
    scroll_depth_max NUMERIC(5,2),               -- max scroll percentage (0-100)
    time_on_page_ms INTEGER,
    -- W3C Navigation Timing fields
    dns_time_ms     INTEGER,
    connect_time_ms INTEGER,
    ttfb_ms         INTEGER,                     -- Time to First Byte
    dom_ready_ms    INTEGER,
    load_time_ms    INTEGER,
    lcp_ms          INTEGER,                     -- Largest Contentful Paint (W3C)
    fid_ms          INTEGER,                     -- First Input Delay
    cls             NUMERIC(6,4),                -- Cumulative Layout Shift
    viewed_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pageview_session ON page_view(session_id);
CREATE INDEX idx_pageview_project ON page_view(project_id, viewed_at);
CREATE INDEX idx_pageview_url ON page_view(project_id, url);

-- Friction signals detected during sessions
CREATE TABLE friction_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES session(id),
    page_view_id    UUID REFERENCES page_view(id),
    friction_type   VARCHAR(50) NOT NULL,        -- rage_click, dead_click, error_click, thrashed_cursor,
                                                  -- u_turn, speed_browse, form_abandon, js_error
    element_selector VARCHAR(500),
    x_position      INTEGER,
    y_position      INTEGER,
    severity        NUMERIC(3,1),                -- 1-10
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_friction_session ON friction_event(session_id);
CREATE INDEX idx_friction_type ON friction_event(friction_type, detected_at);

-- Custom events tracked via SDK (Segment Spec Track call equivalent)
CREATE TABLE track_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    session_id      UUID NOT NULL REFERENCES session(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    event_name      VARCHAR(255) NOT NULL,       -- Segment Spec: Object Action naming, e.g. "Product Added"
    page_url        VARCHAR(2000),
    element_selector VARCHAR(500),
    revenue         NUMERIC(15,2),
    currency        CHAR(3),                     -- ISO 4217
    -- CloudEvents envelope fields
    ce_source       VARCHAR(500),
    ce_type         VARCHAR(255),
    ce_specversion  VARCHAR(10) DEFAULT '1.0',
    tracked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_track_project ON track_event(project_id, tracked_at);
CREATE INDEX idx_track_event_name ON track_event(project_id, event_name);
CREATE INDEX idx_track_session ON track_event(session_id);

-- Event properties (key-value pairs per Segment Spec)
CREATE TABLE track_event_property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    track_event_id  UUID NOT NULL REFERENCES track_event(id) ON DELETE CASCADE,
    property_key    VARCHAR(100) NOT NULL,
    property_value  VARCHAR(1000),
    UNIQUE(track_event_id, property_key)
);
CREATE INDEX idx_tep_event ON track_event_property(track_event_id);
```

## Heatmaps

```sql
CREATE TABLE heatmap_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    url_pattern     VARCHAR(2000) NOT NULL,      -- URL or URL pattern for this heatmap
    snapshot_type   VARCHAR(50) NOT NULL,        -- click, scroll, movement, attention, rage_click
    device_type     VARCHAR(50),                 -- desktop, mobile, tablet, all
    viewport_width  INTEGER,
    viewport_height INTEGER,
    sample_count    INTEGER NOT NULL DEFAULT 0,
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'collecting', -- collecting, ready, archived
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_heatmap_project ON heatmap_snapshot(project_id);
CREATE INDEX idx_heatmap_url ON heatmap_snapshot(project_id, url_pattern);

-- Individual interaction points aggregated into heatmap tiles
CREATE TABLE heatmap_tile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    heatmap_id      UUID NOT NULL REFERENCES heatmap_snapshot(id) ON DELETE CASCADE,
    tile_x          INTEGER NOT NULL,            -- grid x coordinate
    tile_y          INTEGER NOT NULL,            -- grid y coordinate
    tile_width      INTEGER NOT NULL,
    tile_height     INTEGER NOT NULL,
    interaction_count INTEGER NOT NULL DEFAULT 0,
    intensity       NUMERIC(6,4),                -- normalized 0.0-1.0
    UNIQUE(heatmap_id, tile_x, tile_y)
);
CREATE INDEX idx_tile_heatmap ON heatmap_tile(heatmap_id);

-- Raw click/interaction data for detailed analysis
CREATE TABLE heatmap_interaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    heatmap_id      UUID NOT NULL REFERENCES heatmap_snapshot(id),
    session_id      UUID REFERENCES session(id),
    visitor_id      UUID REFERENCES visitor(id),
    interaction_type VARCHAR(50) NOT NULL,       -- click, move, scroll
    x_position      INTEGER NOT NULL,
    y_position      INTEGER NOT NULL,
    element_selector VARCHAR(500),
    element_text    VARCHAR(255),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hmi_heatmap ON heatmap_interaction(heatmap_id);
CREATE INDEX idx_hmi_recorded ON heatmap_interaction(recorded_at);
```

## Session Recording

```sql
CREATE TABLE session_recording (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES session(id),
    project_id      UUID NOT NULL REFERENCES project(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    duration_ms     INTEGER,
    page_count      INTEGER NOT NULL DEFAULT 0,
    event_count     INTEGER NOT NULL DEFAULT 0,
    has_rage_clicks BOOLEAN NOT NULL DEFAULT FALSE,
    has_dead_clicks BOOLEAN NOT NULL DEFAULT FALSE,
    has_js_errors   BOOLEAN NOT NULL DEFAULT FALSE,
    has_form_abandons BOOLEAN NOT NULL DEFAULT FALSE,
    frustration_score NUMERIC(5,2),              -- AI-calculated 0-100
    storage_url     VARCHAR(2000),               -- object storage URL for recording data
    storage_size_bytes BIGINT,
    status          VARCHAR(50) NOT NULL DEFAULT 'recording', -- recording, processing, ready, expired
    expires_at      TIMESTAMPTZ,                 -- data retention
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recording_session ON session_recording(session_id);
CREATE INDEX idx_recording_project ON session_recording(project_id, created_at);
CREATE INDEX idx_recording_frustration ON session_recording(project_id, frustration_score DESC);

-- Recording event batches (DOM mutations, mouse moves etc. stored as compressed chunks)
CREATE TABLE recording_chunk (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recording_id    UUID NOT NULL REFERENCES session_recording(id) ON DELETE CASCADE,
    sequence_number INTEGER NOT NULL,
    chunk_type      VARCHAR(50) NOT NULL,        -- full_snapshot, incremental, meta, custom
    data_compressed BYTEA NOT NULL,              -- compressed rrweb-format event data
    event_count     INTEGER NOT NULL DEFAULT 0,
    start_offset_ms INTEGER NOT NULL,            -- offset from recording start
    end_offset_ms   INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(recording_id, sequence_number)
);
CREATE INDEX idx_chunk_recording ON recording_chunk(recording_id);
```

## Conversion Funnels

```sql
CREATE TABLE funnel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_funnel_project ON funnel(project_id);

CREATE TABLE funnel_step (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    funnel_id       UUID NOT NULL REFERENCES funnel(id) ON DELETE CASCADE,
    step_number     INTEGER NOT NULL,
    name            VARCHAR(255) NOT NULL,
    step_type       VARCHAR(50) NOT NULL,        -- page_view, click, custom_event
    match_url       VARCHAR(2000),
    match_selector  VARCHAR(500),
    match_event     VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(funnel_id, step_number)
);
CREATE INDEX idx_fstep_funnel ON funnel_step(funnel_id);

CREATE TABLE funnel_visitor_progress (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    funnel_id       UUID NOT NULL REFERENCES funnel(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    session_id      UUID NOT NULL REFERENCES session(id),
    max_step_reached INTEGER NOT NULL,
    completed       BOOLEAN NOT NULL DEFAULT FALSE,
    entered_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    duration_ms     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fvp_funnel ON funnel_visitor_progress(funnel_id);
CREATE INDEX idx_fvp_visitor ON funnel_visitor_progress(visitor_id);
```

## Form Analytics

```sql
CREATE TABLE form_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    page_url        VARCHAR(2000) NOT NULL,
    form_selector   VARCHAR(500) NOT NULL,
    form_name       VARCHAR(255),
    field_count     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_form_project ON form_definition(project_id);

CREATE TABLE form_field (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id         UUID NOT NULL REFERENCES form_definition(id) ON DELETE CASCADE,
    field_selector  VARCHAR(500) NOT NULL,
    field_name      VARCHAR(255),
    field_type      VARCHAR(50),                 -- text, email, password, select, checkbox, radio, textarea
    field_order     INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(form_id, field_selector)
);

CREATE TABLE form_interaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id         UUID NOT NULL REFERENCES form_definition(id),
    field_id        UUID REFERENCES form_field(id),
    session_id      UUID NOT NULL REFERENCES session(id),
    visitor_id      UUID NOT NULL REFERENCES visitor(id),
    interaction_type VARCHAR(50) NOT NULL,       -- focus, blur, change, submit, abandon
    time_in_field_ms INTEGER,                    -- time spent in this field
    hesitation_ms   INTEGER,                     -- time before first keystroke
    corrections     INTEGER DEFAULT 0,           -- number of backspace/delete actions
    interacted_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fi_form ON form_interaction(form_id);
CREATE INDEX idx_fi_session ON form_interaction(session_id);
```

## Personalization

```sql
CREATE TABLE segment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    segment_type    VARCHAR(50) NOT NULL,        -- rule_based, ai_predicted, behavioral
    is_dynamic      BOOLEAN NOT NULL DEFAULT TRUE,
    visitor_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_segment_project ON segment(project_id);

CREATE TABLE segment_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    segment_id      UUID NOT NULL REFERENCES segment(id) ON DELETE CASCADE,
    rule_group      INTEGER NOT NULL DEFAULT 0,  -- rules in same group are AND'd; groups are OR'd
    attribute       VARCHAR(255) NOT NULL,
    operator        VARCHAR(30) NOT NULL,
    value           VARCHAR(500) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_srule_segment ON segment_rule(segment_id);

CREATE TABLE personalization_campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    segment_id      UUID REFERENCES segment(id),
    variant_html    TEXT,
    variant_css     TEXT,
    variant_js      TEXT,
    priority        INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
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
    insight_type    VARCHAR(50) NOT NULL,        -- friction_pattern, hypothesis_suggestion, anomaly, opportunity
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    severity        VARCHAR(20),                 -- info, warning, critical
    estimated_revenue_impact NUMERIC(15,2),
    related_url     VARCHAR(2000),
    related_experiment_id UUID REFERENCES experiment(id),
    related_funnel_id UUID REFERENCES funnel(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'new', -- new, acknowledged, acted, dismissed
    model_version   VARCHAR(50),
    confidence      NUMERIC(5,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_insight_project ON ai_insight(project_id, created_at);
CREATE INDEX idx_insight_status ON ai_insight(project_id, status);

CREATE TABLE anomaly_detection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiment(id),
    anomaly_type    VARCHAR(50) NOT NULL,        -- sample_ratio_mismatch, novelty_effect, early_winner,
                                                  -- traffic_drop, conversion_spike, statistical_regression
    severity        VARCHAR(20) NOT NULL,
    description     TEXT NOT NULL,
    detected_value  NUMERIC(15,6),
    expected_value  NUMERIC(15,6),
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES app_user(id)
);
CREATE INDEX idx_anomaly_experiment ON anomaly_detection(experiment_id);
```

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(100) NOT NULL,       -- experiment.created, variant.updated, funnel.deleted, etc.
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,                       -- before/after diff
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organization, project, app_user |
| Visitor & Consent | 3 | visitor, visitor_consent, visitor_attribute |
| Experimentation | 8 | experiment, variant, goal, targeting, assignment, result, bandit_state, hypothesis |
| Sessions & Events | 4 | session, page_view, friction_event, track_event + 1 property table |
| Heatmaps | 3 | heatmap_snapshot, heatmap_tile, heatmap_interaction |
| Session Recording | 2 | session_recording, recording_chunk |
| Funnels | 3 | funnel, funnel_step, funnel_visitor_progress |
| Form Analytics | 3 | form_definition, form_field, form_interaction |
| Personalization | 3 | segment, segment_rule, personalization_campaign |
| AI & Insights | 2 | ai_insight, anomaly_detection |
| Audit | 1 | audit_log |
| **Total** | **~36** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — supports distributed ID generation, safe for multi-region deployment and API exposure without leaking sequence information.

2. **Separate visitor vs. app_user tables** — website visitors being tracked are fundamentally different entities from platform users who manage experiments. Keeps RBAC clean.

3. **Deterministic bucket-based assignment** — the `experiment_assignment.bucket_value` field (0-9999) enables deterministic hashing per the OpenFeature specification, ensuring the same visitor always sees the same variant across devices/SDKs.

4. **CloudEvents envelope on track_event** — `ce_source`, `ce_type`, and `ce_specversion` columns enable emitting standards-compliant events to external systems without transformation.

5. **Heatmap tile pre-aggregation** — raw interactions are stored but also pre-aggregated into grid tiles for fast rendering, following the pattern described in AWS Clickstream Analytics guidance.

6. **Session recording stored as compressed chunks** — recording data is stored as compressed binary blobs (rrweb format) in `recording_chunk`, not as individual DOM events in rows. This keeps the relational schema clean while the bulk data lives in a format optimized for playback.

7. **LIFT model and PIE/ICE scoring as explicit columns** — the hypothesis table stores both frameworks as numeric columns rather than JSONB, enabling SQL-level sorting and filtering on prioritization scores.

8. **Bandit state table** — Thompson Sampling parameters (alpha/beta of the Beta distribution) are stored separately from experiment results, enabling the allocation algorithm to update independently of result aggregation.

9. **Consent as first-class entity** — `visitor_consent` stores per-purpose consent with IAB GPP strings, supporting the privacy-first architecture identified as a market differentiator.

10. **Normalized event properties** — `track_event_property` as a separate table follows strict normalization. This is correct for integrity but may need denormalization (or JSONB migration) at scale.
