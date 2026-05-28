# Data Model Suggestion 2: Event-Sourced / CQRS

> Project: Conversion Rate Optimization · Created: 2026-05-19

## Philosophy

This model treats every state change in the CRO platform as an immutable event appended to an event store. The event store is the single source of truth; all queryable state (experiment status, heatmap aggregations, funnel progress, variant results) is derived by replaying or projecting events into materialized read models. This follows the Command Query Responsibility Segregation (CQRS) pattern: writes go to the event store, reads come from purpose-built projections.

Event sourcing is a natural fit for CRO platforms because the domain is inherently event-driven. Visitor interactions are already events (clicks, scrolls, page views). Experiment lifecycle transitions are events (started, paused, variant_added, traffic_reallocated). The audit trail requirement is satisfied automatically -- every change is an event with a timestamp, actor, and payload. Thompson Sampling bandit allocation can replay the event stream to recalculate posteriors. Temporal queries ("what was the conversion rate at 3pm yesterday?") are answered by replaying events up to that timestamp.

The CloudEvents specification provides the envelope format for all events. The Segment Spec Track/Identify/Page taxonomy provides the event naming convention. The event store uses PostgreSQL with partitioned append-only tables, while read models use materialized views and summary tables that are rebuilt by event processors.

**Best for:** Teams that need full audit trails, temporal queries, real-time streaming analytics, and the ability to replay history for debugging or reprocessing with new algorithms.

**Trade-offs:**
- Pro: Complete audit trail by design; every state change is recorded permanently
- Pro: Temporal queries are natural; replay events to any point in time
- Pro: New read models can be built retroactively by replaying the event stream
- Pro: Bandit algorithms can be re-run with different parameters by replaying assignment events
- Pro: Event stream can feed real-time dashboards, anomaly detection, and AI pipelines
- Con: Eventual consistency; read models may lag behind the event store
- Con: More complex infrastructure; requires event processors, projection rebuilders
- Con: Storage grows monotonically; event store must be partitioned and archived
- Con: Simple queries require maintaining separate read models
- Con: Debugging requires understanding the event replay pipeline, not just reading a table

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 (CNCF) | Every event in the store uses the CloudEvents envelope (id, source, type, specversion, time, subject, data) |
| Segment Spec (Track/Identify/Page) | Visitor interaction events follow Segment naming conventions and property schemas |
| OpenFeature Specification | Flag evaluation events capture the OpenFeature evaluation context and resolution details |
| W3C Navigation Timing Level 2 | Page performance events include Navigation Timing attributes as event properties |
| AsyncAPI 3.0 | Event channels (experiment lifecycle, visitor interactions, AI insights) described using AsyncAPI |
| ISO/IEC 27001 | Immutable event store inherently satisfies audit log requirements |
| GDPR / CCPA | Consent events create an auditable consent history; deletion uses crypto-shredding |

---

## Event Store (Source of Truth)

```sql
-- The central event store. All state changes flow through here.
-- Partitioned by month for manageability and archival.
CREATE TABLE event_store (
    event_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    -- CloudEvents envelope fields
    ce_specversion  VARCHAR(10) NOT NULL DEFAULT '1.0',
    ce_type         VARCHAR(255) NOT NULL,       -- e.g., "cro.experiment.started", "cro.visitor.page_viewed"
    ce_source       VARCHAR(500) NOT NULL,       -- e.g., "/projects/{project_id}"
    ce_subject      VARCHAR(500),                -- entity the event is about, e.g., experiment ID
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- CRO-specific context
    project_id      UUID NOT NULL,
    aggregate_type  VARCHAR(100) NOT NULL,       -- experiment, visitor, session, heatmap, funnel, hypothesis
    aggregate_id    UUID NOT NULL,               -- ID of the entity this event belongs to
    sequence_number BIGINT NOT NULL,             -- per-aggregate sequence for ordering
    -- Event payload
    event_data      JSONB NOT NULL,              -- full event payload
    -- Metadata
    actor_id        UUID,                        -- user or system that caused the event
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'system', -- user, system, ai_agent
    correlation_id  UUID,                        -- links related events across aggregates
    causation_id    UUID,                        -- the event that caused this event
    schema_version  INTEGER NOT NULL DEFAULT 1,  -- payload schema version for evolution
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (event_id, ce_time)              -- composite PK for partitioning
) PARTITION BY RANGE (ce_time);

-- Create monthly partitions (example for current quarter)
CREATE TABLE event_store_2026_05 PARTITION OF event_store
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE event_store_2026_06 PARTITION OF event_store
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

-- Indexes on the partitioned table
CREATE INDEX idx_es_aggregate ON event_store(aggregate_type, aggregate_id, sequence_number);
CREATE INDEX idx_es_project ON event_store(project_id, ce_time);
CREATE INDEX idx_es_type ON event_store(ce_type, ce_time);
CREATE INDEX idx_es_correlation ON event_store(correlation_id) WHERE correlation_id IS NOT NULL;

-- Unique constraint to prevent duplicate events per aggregate
CREATE UNIQUE INDEX idx_es_aggregate_seq ON event_store(aggregate_id, sequence_number, ce_time);
```

### Event Type Taxonomy

```
-- Experiment lifecycle events:
cro.experiment.created          { name, type, hypothesis_id, traffic_pct }
cro.experiment.started          { started_at }
cro.experiment.paused           { reason }
cro.experiment.resumed          {}
cro.experiment.completed        { winner_variant_id, confidence }
cro.experiment.archived         {}
cro.experiment.variant_added    { variant_id, name, is_control, weight }
cro.experiment.variant_updated  { variant_id, changes }
cro.experiment.traffic_reallocated { allocations: [{variant_id, weight}] }
cro.experiment.goal_added       { goal_id, name, goal_type, is_primary }
cro.experiment.targeting_updated { rules: [...] }

-- Visitor identity events:
cro.visitor.identified          { anonymous_id, user_id, traits }
cro.visitor.attributed          { utm_source, utm_medium, utm_campaign }
cro.visitor.consent_granted     { consent_type, legal_basis, gpp_string }
cro.visitor.consent_withdrawn   { consent_type }

-- Visitor interaction events (Segment Spec Track equivalent):
cro.visitor.page_viewed         { url, title, referrer, viewport, nav_timing }
cro.visitor.element_clicked     { selector, x, y, element_text }
cro.visitor.element_hovered     { selector, duration_ms }
cro.visitor.scrolled            { depth_pct, direction }
cro.visitor.mouse_moved         { positions: [{x, y, t}] }  -- batched
cro.visitor.form_field_focused  { form_selector, field_selector }
cro.visitor.form_field_blurred  { field_selector, time_in_field_ms, hesitation_ms }
cro.visitor.form_submitted      { form_selector, success }
cro.visitor.form_abandoned      { form_selector, last_field }
cro.visitor.rage_clicked        { selector, x, y, click_count }
cro.visitor.js_error            { message, stack, url, line }

-- Session events:
cro.session.started             { visitor_id, entry_url, referrer, utm_* }
cro.session.ended               { duration_ms, page_count, is_bounced }

-- Assignment events:
cro.assignment.assigned         { experiment_id, variant_id, visitor_id, bucket }
cro.assignment.converted        { experiment_id, variant_id, goal_id, revenue }

-- Bandit events:
cro.bandit.posterior_updated    { experiment_id, variant_id, alpha, beta }
cro.bandit.allocation_updated   { experiment_id, allocations: [{variant_id, weight}] }

-- Hypothesis events:
cro.hypothesis.created          { title, source, lift_scores, pie_scores }
cro.hypothesis.scored           { priority_score, ai_confidence, predicted_uplift }
cro.hypothesis.approved         { approved_by }
cro.hypothesis.linked           { experiment_id }

-- AI insight events:
cro.insight.generated           { type, title, description, severity, revenue_impact }
cro.insight.acknowledged        { user_id }
cro.insight.dismissed           { user_id, reason }
cro.anomaly.detected            { experiment_id, type, detected_value, expected_value }
cro.anomaly.resolved            { resolution }
```

## Read Models (Projections)

These tables are built and maintained by event processors that consume the event store. They can be rebuilt from scratch by replaying events.

### Experiment Read Model

```sql
-- Materialized current state of experiments (projected from event store)
CREATE TABLE rm_experiment (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    hypothesis_id   UUID,
    type            VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    allocation_method VARCHAR(50) NOT NULL,
    traffic_percentage NUMERIC(5,2) NOT NULL,
    statistical_method VARCHAR(50) NOT NULL,
    confidence_level NUMERIC(5,4) NOT NULL,
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_by      UUID,
    last_event_id   UUID NOT NULL,               -- tracks projection position
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_exp_project ON rm_experiment(project_id);
CREATE INDEX idx_rm_exp_status ON rm_experiment(project_id, status);

CREATE TABLE rm_variant (
    id              UUID PRIMARY KEY,
    experiment_id   UUID NOT NULL REFERENCES rm_experiment(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    is_control      BOOLEAN NOT NULL,
    traffic_weight  NUMERIC(5,2) NOT NULL,
    sample_size     INTEGER NOT NULL DEFAULT 0,
    conversions     INTEGER NOT NULL DEFAULT 0,
    conversion_rate NUMERIC(10,6),
    revenue_total   NUMERIC(15,2) DEFAULT 0,
    confidence      NUMERIC(6,4),
    uplift          NUMERIC(10,4),
    -- Bandit posterior
    bandit_alpha    NUMERIC(15,4) DEFAULT 1.0,
    bandit_beta     NUMERIC(15,4) DEFAULT 1.0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_var_exp ON rm_variant(experiment_id);
```

### Visitor & Session Read Models

```sql
CREATE TABLE rm_visitor (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    anonymous_id    VARCHAR(255) NOT NULL,
    user_id         VARCHAR(255),
    device_type     VARCHAR(50),
    browser         VARCHAR(100),
    os              VARCHAR(100),
    country         CHAR(2),
    session_count   INTEGER NOT NULL DEFAULT 0,
    total_page_views INTEGER NOT NULL DEFAULT 0,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    consent_analytics BOOLEAN DEFAULT FALSE,
    consent_recording BOOLEAN DEFAULT FALSE,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_rm_vis_anon ON rm_visitor(project_id, anonymous_id);

CREATE TABLE rm_session (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    visitor_id      UUID NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    duration_ms     INTEGER,
    page_count      INTEGER NOT NULL DEFAULT 0,
    event_count     INTEGER NOT NULL DEFAULT 0,
    is_bounced      BOOLEAN,
    entry_url       VARCHAR(2000),
    exit_url        VARCHAR(2000),
    friction_score  NUMERIC(5,2) DEFAULT 0,
    rage_click_count INTEGER DEFAULT 0,
    dead_click_count INTEGER DEFAULT 0,
    js_error_count  INTEGER DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_sess_project ON rm_session(project_id, started_at);
CREATE INDEX idx_rm_sess_visitor ON rm_session(visitor_id);
```

### Heatmap Aggregation Read Model

```sql
-- Heatmap tiles are projections of cro.visitor.element_clicked / mouse_moved / scrolled events
CREATE TABLE rm_heatmap_tile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL,
    url_pattern     VARCHAR(2000) NOT NULL,
    tile_type       VARCHAR(50) NOT NULL,        -- click, scroll, movement
    device_type     VARCHAR(50) NOT NULL,
    period_date     DATE NOT NULL,               -- daily aggregation
    tile_x          INTEGER NOT NULL,
    tile_y          INTEGER NOT NULL,
    tile_width      INTEGER NOT NULL DEFAULT 20,
    tile_height     INTEGER NOT NULL DEFAULT 20,
    interaction_count INTEGER NOT NULL DEFAULT 0,
    unique_visitors INTEGER NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(project_id, url_pattern, tile_type, device_type, period_date, tile_x, tile_y)
);
CREATE INDEX idx_rm_ht_lookup ON rm_heatmap_tile(project_id, url_pattern, tile_type, period_date);
```

### Funnel Progress Read Model

```sql
CREATE TABLE rm_funnel (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    step_count      INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_funnel_step_stats (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    funnel_id       UUID NOT NULL REFERENCES rm_funnel(id),
    step_number     INTEGER NOT NULL,
    step_name       VARCHAR(255) NOT NULL,
    period_date     DATE NOT NULL,
    entered_count   INTEGER NOT NULL DEFAULT 0,
    completed_count INTEGER NOT NULL DEFAULT 0,
    drop_off_count  INTEGER NOT NULL DEFAULT 0,
    avg_time_ms     INTEGER,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(funnel_id, step_number, period_date)
);
CREATE INDEX idx_rm_fss_funnel ON rm_funnel_step_stats(funnel_id, period_date);
```

### Form Analytics Read Model

```sql
CREATE TABLE rm_form_field_stats (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL,
    form_selector   VARCHAR(500) NOT NULL,
    field_selector  VARCHAR(500) NOT NULL,
    field_name      VARCHAR(255),
    page_url        VARCHAR(2000) NOT NULL,
    period_date     DATE NOT NULL,
    focus_count     INTEGER NOT NULL DEFAULT 0,
    completion_count INTEGER NOT NULL DEFAULT 0,
    abandon_count   INTEGER NOT NULL DEFAULT 0,
    avg_time_ms     INTEGER,
    avg_hesitation_ms INTEGER,
    avg_corrections NUMERIC(5,2),
    drop_off_rate   NUMERIC(5,4),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(project_id, form_selector, field_selector, period_date)
);
CREATE INDEX idx_rm_ffs_lookup ON rm_form_field_stats(project_id, page_url, period_date);
```

### AI Insights Read Model

```sql
CREATE TABLE rm_insight (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    insight_type    VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    severity        VARCHAR(20),
    estimated_revenue_impact NUMERIC(15,2),
    related_experiment_id UUID,
    status          VARCHAR(50) NOT NULL DEFAULT 'new',
    confidence      NUMERIC(5,4),
    generated_at    TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_ins_project ON rm_insight(project_id, generated_at);
```

## Operational Infrastructure

```sql
-- Tracks projection (read model) rebuild state
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,    -- e.g., "rm_experiment", "rm_heatmap_tile"
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    lag_ms          INTEGER,                     -- current lag behind event store
    status          VARCHAR(50) NOT NULL DEFAULT 'running', -- running, rebuilding, paused, error
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed processing
CREATE TABLE event_dead_letter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    projection_name VARCHAR(100) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 5,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_edl_retry ON event_dead_letter(next_retry_at) WHERE retry_count < max_retries;

-- Snapshot store for aggregate state (avoids replaying full history)
CREATE TABLE aggregate_snapshot (
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    UUID NOT NULL,
    sequence_number BIGINT NOT NULL,             -- event sequence at snapshot time
    state_data      JSONB NOT NULL,              -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);
```

## Session Recording Storage

```sql
-- Session recordings stored as event streams (naturally event-sourced)
CREATE TABLE rm_session_recording (
    id              UUID PRIMARY KEY,
    session_id      UUID NOT NULL,
    project_id      UUID NOT NULL,
    visitor_id      UUID NOT NULL,
    duration_ms     INTEGER,
    event_count     INTEGER NOT NULL DEFAULT 0,
    has_rage_clicks BOOLEAN NOT NULL DEFAULT FALSE,
    has_js_errors   BOOLEAN NOT NULL DEFAULT FALSE,
    frustration_score NUMERIC(5,2),
    storage_url     VARCHAR(2000),               -- object storage for compressed recording data
    storage_size_bytes BIGINT,
    status          VARCHAR(50) NOT NULL DEFAULT 'recording',
    expires_at      TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_sr_project ON rm_session_recording(project_id, projected_at);
CREATE INDEX idx_rm_sr_frustration ON rm_session_recording(project_id, frustration_score DESC);
```

## Multi-Tenancy & Platform Configuration

```sql
CREATE TABLE rm_organization (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_project (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    domain          VARCHAR(255),
    snippet_key     VARCHAR(64) NOT NULL UNIQUE,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_app_user (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255),
    role            VARCHAR(50) NOT NULL,
    last_login_at   TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example: Temporal Query (Replay to a Point in Time)

```sql
-- "What was the conversion rate for Experiment X at 3pm yesterday?"
-- Replay assignment and conversion events up to the target timestamp.
WITH assignments AS (
    SELECT
        (event_data->>'variant_id')::UUID AS variant_id,
        (event_data->>'visitor_id')::UUID AS visitor_id
    FROM event_store
    WHERE aggregate_id = 'experiment-uuid-here'
      AND ce_type = 'cro.assignment.assigned'
      AND ce_time <= '2026-05-18 15:00:00+00'
),
conversions AS (
    SELECT
        (event_data->>'variant_id')::UUID AS variant_id,
        (event_data->>'visitor_id')::UUID AS visitor_id
    FROM event_store
    WHERE aggregate_id = 'experiment-uuid-here'
      AND ce_type = 'cro.assignment.converted'
      AND ce_time <= '2026-05-18 15:00:00+00'
)
SELECT
    a.variant_id,
    COUNT(DISTINCT a.visitor_id) AS sample_size,
    COUNT(DISTINCT c.visitor_id) AS conversions,
    COUNT(DISTINCT c.visitor_id)::NUMERIC / NULLIF(COUNT(DISTINCT a.visitor_id), 0) AS conversion_rate
FROM assignments a
LEFT JOIN conversions c ON a.variant_id = c.variant_id AND a.visitor_id = c.visitor_id
GROUP BY a.variant_id;
```

## Example: Rebuilding a Read Model

```sql
-- Rebuild rm_variant from event store (pseudocode in SQL)
-- In practice, this runs as an application-level event processor.

-- 1. Clear the projection
TRUNCATE rm_variant;

-- 2. Replay variant_added events
INSERT INTO rm_variant (id, experiment_id, name, slug, is_control, traffic_weight, projected_at)
SELECT
    (event_data->>'variant_id')::UUID,
    aggregate_id,
    event_data->>'name',
    event_data->>'slug',
    (event_data->>'is_control')::BOOLEAN,
    (event_data->>'weight')::NUMERIC,
    now()
FROM event_store
WHERE ce_type = 'cro.experiment.variant_added'
ORDER BY ce_time;

-- 3. Apply conversion counts
UPDATE rm_variant v SET
    conversions = sub.conv_count,
    sample_size = sub.sample_count,
    conversion_rate = sub.conv_count::NUMERIC / NULLIF(sub.sample_count, 0)
FROM (
    SELECT
        (event_data->>'variant_id')::UUID AS variant_id,
        COUNT(*) FILTER (WHERE ce_type = 'cro.assignment.converted') AS conv_count,
        COUNT(*) FILTER (WHERE ce_type = 'cro.assignment.assigned') AS sample_count
    FROM event_store
    WHERE ce_type IN ('cro.assignment.assigned', 'cro.assignment.converted')
    GROUP BY event_data->>'variant_id'
) sub
WHERE v.id = sub.variant_id;

-- 4. Update checkpoint
INSERT INTO projection_checkpoint (projection_name, last_event_id, last_event_time, events_processed)
SELECT 'rm_variant', event_id, ce_time, COUNT(*) OVER ()
FROM event_store ORDER BY ce_time DESC LIMIT 1
ON CONFLICT (projection_name) DO UPDATE SET
    last_event_id = EXCLUDED.last_event_id,
    last_event_time = EXCLUDED.last_event_time,
    events_processed = EXCLUDED.events_processed,
    updated_at = now();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 (partitioned) | Core source of truth; monthly partitions |
| Infrastructure | 3 | projection_checkpoint, event_dead_letter, aggregate_snapshot |
| Read Model: Experiments | 2 | rm_experiment, rm_variant |
| Read Model: Visitors & Sessions | 2 | rm_visitor, rm_session |
| Read Model: Heatmaps | 1 | rm_heatmap_tile (daily aggregation) |
| Read Model: Funnels | 2 | rm_funnel, rm_funnel_step_stats |
| Read Model: Forms | 1 | rm_form_field_stats |
| Read Model: AI | 1 | rm_insight |
| Read Model: Recordings | 1 | rm_session_recording |
| Read Model: Platform | 3 | rm_organization, rm_project, rm_app_user |
| **Total** | **~17 tables** | Plus monthly event_store partitions |

---

## Key Design Decisions

1. **Single event store as source of truth** — all domain state is derived from events. No direct writes to read model tables except through event processors. This guarantees consistency and full auditability.

2. **CloudEvents envelope** — every event uses the CloudEvents 1.0 context attributes (specversion, type, source, subject, time), making events portable to external systems (Kafka, webhooks, AI pipelines) without transformation.

3. **Per-aggregate sequencing** — `sequence_number` within each aggregate prevents out-of-order processing and enables optimistic concurrency control (append only if sequence = last + 1).

4. **Monthly partitioning** — the event store is partitioned by `ce_time` for efficient range queries and archival. Old partitions can be moved to cold storage while maintaining the ability to replay.

5. **Projection checkpoints** — each read model tracks its position in the event stream. If a projection crashes, it resumes from the last checkpoint rather than replaying the entire history.

6. **Aggregate snapshots** — for aggregates with long event histories (high-traffic experiments), snapshots avoid replaying thousands of events. The snapshot stores the materialized state at a known sequence number.

7. **Event schema versioning** — `schema_version` on each event enables payload evolution. Processors can handle multiple versions without breaking, and upcasters can transform old events to new schemas during replay.

8. **Crypto-shredding for GDPR** — visitor PII in events is encrypted with a per-visitor key. Deletion means destroying the key, rendering all events for that visitor unreadable while preserving aggregate statistics. This satisfies GDPR right-to-erasure without mutating the immutable event store.

9. **Correlation and causation IDs** — `correlation_id` links all events from a single user action across aggregates. `causation_id` tracks the causal chain (e.g., a conversion event was caused by an assignment event). Essential for debugging and AI analysis.

10. **Read models are disposable** — any read model can be dropped and rebuilt from the event store. This means new feature requirements (e.g., a new dashboard) can be satisfied by creating a new projection without modifying the core data model.
