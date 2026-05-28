# Data Model Suggestion 4: Dual-Store (PostgreSQL + ClickHouse)

> Project: Conversion Rate Optimization · Created: 2026-05-19

## Philosophy

This model separates the CRO platform into two purpose-built storage layers: PostgreSQL for operational state (experiments, variants, visitors, hypotheses, configuration) and ClickHouse for high-volume analytical workloads (clickstream events, heatmap interactions, session recording metadata, funnel analysis, experiment metrics). Each database handles what it does best.

ClickHouse's columnar storage and vectorized execution engine can aggregate billions of clickstream events in sub-second queries -- 3-10x faster than PostgreSQL for the aggregation patterns that dominate CRO analytics (heatmap tile generation, funnel drop-off calculation, conversion rate computation, scroll depth percentiles). Meanwhile, PostgreSQL provides ACID transactions, foreign keys, and JSONB flexibility for the operational side where data integrity and complex updates matter.

This mirrors how mature analytics platforms actually operate. AWS Clickstream Analytics on AWS uses a similar pattern with operational metadata in a relational store and event data in a columnar analytics engine. Platforms like Hotjar, FullStory, and Mouseflow all separate their event ingestion pipeline from their configuration and user management backend. The Segment Spec event taxonomy provides the schema for the ClickHouse event tables, while CloudEvents provides the envelope format for the event pipeline that bridges the two stores.

**Best for:** Teams expecting high traffic volumes (millions of events per day), who need sub-second analytics dashboards, real-time heatmap rendering, and fast funnel analysis while maintaining a clean operational data model.

**Trade-offs:**
- Pro: Sub-second analytical queries on billions of events; ClickHouse excels at aggregation
- Pro: Clean separation of concerns; operational logic does not compete with analytics for resources
- Pro: ClickHouse handles denormalized, append-heavy workloads without schema overhead
- Pro: PostgreSQL side stays lean and fast for experiment management CRUD
- Pro: Event ingestion can scale independently of operational database
- Con: Two database technologies to operate, monitor, and maintain
- Con: Data consistency between stores is eventually consistent (event pipeline lag)
- Con: Team needs expertise in both PostgreSQL and ClickHouse SQL dialects
- Con: Cross-store queries require application-level joins
- Con: Higher infrastructure cost than a single-database approach

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment Spec (Track/Identify/Page) | ClickHouse event tables follow Segment event taxonomy and property naming |
| CloudEvents 1.0 | Event pipeline between PostgreSQL and ClickHouse uses CloudEvents envelope |
| OpenFeature Specification | PostgreSQL experiment/variant tables model OpenFeature concepts |
| W3C Navigation Timing Level 2 | Performance metrics stored as ClickHouse columns in page_view events |
| W3C Largest Contentful Paint | Dedicated LCP column in ClickHouse page_view table |
| AsyncAPI 3.0 | Event channels (Kafka topics) connecting the stores described via AsyncAPI |
| GDPR / CCPA | Consent managed in PostgreSQL; ClickHouse events anonymized at ingestion |
| IAB GPP | GPP consent strings stored in PostgreSQL visitor table |

---

## PostgreSQL: Operational Store

The PostgreSQL side manages all CRUD operations, user-facing configuration, and entity relationships.

### Identity & Multi-Tenancy

```sql
-- PostgreSQL
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
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
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);
```

### Visitor Identity & Consent

```sql
-- PostgreSQL
CREATE TABLE visitor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    anonymous_id    VARCHAR(255) NOT NULL,
    user_id         VARCHAR(255),
    traits          JSONB NOT NULL DEFAULT '{}',
    consent         JSONB NOT NULL DEFAULT '{}',
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_visitor_anon ON visitor(project_id, anonymous_id);
CREATE INDEX idx_visitor_user ON visitor(project_id, user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_visitor_traits ON visitor USING GIN (traits);
```

### Experimentation

```sql
-- PostgreSQL
CREATE TABLE experiment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    hypothesis_id   UUID,
    type            VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config includes: allocation_method, traffic_percentage, statistical_method,
    -- confidence_level, min_sample_size, targeting rules, schedule
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
    slug            VARCHAR(100) NOT NULL,
    is_control      BOOLEAN NOT NULL DEFAULT FALSE,
    traffic_weight  NUMERIC(5,2) NOT NULL DEFAULT 0,
    changes         JSONB NOT NULL DEFAULT '{}',
    description     TEXT,
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_goal_experiment ON experiment_goal(experiment_id);

CREATE TABLE hypothesis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    source          VARCHAR(50) NOT NULL,
    scoring         JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(50) NOT NULL DEFAULT 'proposed',
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hypothesis_project ON hypothesis(project_id);

ALTER TABLE experiment ADD CONSTRAINT fk_experiment_hypothesis
    FOREIGN KEY (hypothesis_id) REFERENCES hypothesis(id);
```

### Funnels, Segments & Personalization (Configuration)

```sql
-- PostgreSQL
CREATE TABLE funnel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    steps           JSONB NOT NULL,              -- funnel step definitions
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_funnel_project ON funnel(project_id);

CREATE TABLE segment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    segment_type    VARCHAR(50) NOT NULL,
    rules           JSONB NOT NULL DEFAULT '[]',
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
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_personalization_project ON personalization_campaign(project_id);
```

### AI Insights & Audit (PostgreSQL)

```sql
-- PostgreSQL
CREATE TABLE ai_insight (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id),
    insight_type    VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    severity        VARCHAR(20),
    status          VARCHAR(50) NOT NULL DEFAULT 'new',
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_insight_project ON ai_insight(project_id, created_at);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## ClickHouse: Analytics Store

The ClickHouse side handles all high-volume event ingestion and analytical queries. Tables use the MergeTree family of engines, optimized for append-heavy, aggregation-dominant workloads.

### Clickstream Events

```sql
-- ClickHouse
CREATE TABLE events (
    -- Event identity
    event_id        UUID,
    project_id      UUID,
    session_id      UUID,
    visitor_id      UUID,
    anonymous_id    String,

    -- Event classification (Segment Spec naming)
    event_type      LowCardinality(String),      -- page_view, click, scroll, form_focus,
                                                  -- form_blur, form_submit, custom, conversion,
                                                  -- rage_click, dead_click, js_error
    event_name      Nullable(String),            -- Segment "Object Action": "Product Added"

    -- Location
    url             String,
    url_path        String,                      -- extracted path for grouping
    url_host        String,                      -- extracted host

    -- Element context
    element_selector Nullable(String),
    element_text    Nullable(String),

    -- Position
    x_position      Nullable(Int32),
    y_position      Nullable(Int32),
    scroll_depth_pct Nullable(Float32),

    -- Performance (W3C Navigation Timing / Core Web Vitals)
    ttfb_ms         Nullable(Int32),
    dom_ready_ms    Nullable(Int32),
    load_time_ms    Nullable(Int32),
    lcp_ms          Nullable(Int32),             -- Largest Contentful Paint
    fid_ms          Nullable(Int32),             -- First Input Delay
    cls             Nullable(Float32),           -- Cumulative Layout Shift

    -- Viewport
    viewport_width  Nullable(Int32),
    viewport_height Nullable(Int32),

    -- Form analytics
    form_selector   Nullable(String),
    field_selector  Nullable(String),
    field_name      Nullable(String),
    time_in_field_ms Nullable(Int32),
    hesitation_ms   Nullable(Int32),
    corrections     Nullable(Int32),

    -- Conversion / revenue
    revenue         Nullable(Decimal(15,2)),
    currency        Nullable(FixedString(3)),    -- ISO 4217

    -- Experiment context
    experiment_id   Nullable(UUID),
    variant_id      Nullable(UUID),
    goal_id         Nullable(UUID),

    -- Friction signals
    is_rage_click   UInt8 DEFAULT 0,
    is_dead_click   UInt8 DEFAULT 0,
    rage_click_count Nullable(Int32),

    -- Visitor context (denormalized for query performance)
    device_type     LowCardinality(String) DEFAULT '',
    browser         LowCardinality(String) DEFAULT '',
    os              LowCardinality(String) DEFAULT '',
    country         LowCardinality(FixedString(2)) DEFAULT '',

    -- Attribution (denormalized)
    utm_source      Nullable(String),
    utm_medium      Nullable(String),
    utm_campaign    Nullable(String),

    -- Extra properties (variable per event type)
    properties      String DEFAULT '{}',          -- JSON string for flexible properties

    -- Timestamps
    occurred_at     DateTime64(3, 'UTC'),
    ingested_at     DateTime64(3, 'UTC') DEFAULT now64(3)
)
ENGINE = MergeTree()
PARTITION BY (project_id, toYYYYMM(occurred_at))
ORDER BY (project_id, event_type, occurred_at, session_id)
TTL occurred_at + INTERVAL 13 MONTH
SETTINGS index_granularity = 8192;

-- Secondary indexes for common query patterns
ALTER TABLE events ADD INDEX idx_url url_path TYPE bloom_filter GRANULARITY 4;
ALTER TABLE events ADD INDEX idx_event_name event_name TYPE bloom_filter GRANULARITY 4;
ALTER TABLE events ADD INDEX idx_visitor visitor_id TYPE bloom_filter GRANULARITY 4;
ALTER TABLE events ADD INDEX idx_experiment experiment_id TYPE bloom_filter GRANULARITY 4;
ALTER TABLE events ADD INDEX idx_element element_selector TYPE bloom_filter GRANULARITY 4;
```

### Session Summaries (Materialized from Events)

```sql
-- ClickHouse
CREATE TABLE session_summaries (
    session_id      UUID,
    project_id      UUID,
    visitor_id      UUID,

    started_at      DateTime64(3, 'UTC'),
    ended_at        Nullable(DateTime64(3, 'UTC')),
    duration_ms     Nullable(Int32),

    page_count      Int32 DEFAULT 0,
    event_count     Int32 DEFAULT 0,
    is_bounced      UInt8 DEFAULT 0,

    entry_url       String DEFAULT '',
    exit_url        String DEFAULT '',

    -- Attribution
    referrer        String DEFAULT '',
    utm_source      String DEFAULT '',
    utm_medium      String DEFAULT '',
    utm_campaign    String DEFAULT '',

    -- Friction aggregates
    friction_score  Nullable(Float32),
    rage_click_count Int32 DEFAULT 0,
    dead_click_count Int32 DEFAULT 0,
    js_error_count  Int32 DEFAULT 0,

    -- Visitor context (denormalized)
    device_type     LowCardinality(String) DEFAULT '',
    browser         LowCardinality(String) DEFAULT '',
    country         LowCardinality(FixedString(2)) DEFAULT '',

    updated_at      DateTime64(3, 'UTC') DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(updated_at)
PARTITION BY (project_id, toYYYYMM(started_at))
ORDER BY (project_id, session_id)
TTL started_at + INTERVAL 13 MONTH;
```

### Experiment Metrics (Pre-Aggregated)

```sql
-- ClickHouse
-- Pre-aggregated experiment metrics, updated incrementally
CREATE TABLE experiment_metrics (
    project_id      UUID,
    experiment_id   UUID,
    variant_id      UUID,
    goal_id         UUID,
    period_date     Date,

    -- Counts
    assignments     Int64 DEFAULT 0,
    conversions     Int64 DEFAULT 0,
    unique_visitors Int64 DEFAULT 0,

    -- Revenue
    revenue_total   Decimal(15,2) DEFAULT 0,
    revenue_count   Int64 DEFAULT 0,

    updated_at      DateTime64(3, 'UTC') DEFAULT now64(3)
)
ENGINE = SummingMergeTree()
PARTITION BY (project_id, toYYYYMM(period_date))
ORDER BY (project_id, experiment_id, variant_id, goal_id, period_date);
```

### Heatmap Aggregations

```sql
-- ClickHouse
-- Pre-aggregated heatmap tile data
CREATE TABLE heatmap_tiles (
    project_id      UUID,
    url_path        String,
    tile_type       LowCardinality(String),      -- click, scroll, movement
    device_type     LowCardinality(String),
    period_date     Date,

    tile_x          Int32,
    tile_y          Int32,
    tile_width      Int32 DEFAULT 20,
    tile_height     Int32 DEFAULT 20,

    interaction_count Int64 DEFAULT 0,
    unique_visitors Int64 DEFAULT 0,

    updated_at      DateTime64(3, 'UTC') DEFAULT now64(3)
)
ENGINE = SummingMergeTree()
PARTITION BY (project_id, toYYYYMM(period_date))
ORDER BY (project_id, url_path, tile_type, device_type, period_date, tile_x, tile_y);
```

### Funnel Progress

```sql
-- ClickHouse
CREATE TABLE funnel_progress (
    project_id      UUID,
    funnel_id       UUID,
    visitor_id      UUID,
    session_id      UUID,

    max_step_reached Int32,
    completed       UInt8 DEFAULT 0,

    entered_at      DateTime64(3, 'UTC'),
    completed_at    Nullable(DateTime64(3, 'UTC')),
    duration_ms     Nullable(Int32),

    -- Visitor context for segmented funnel analysis
    device_type     LowCardinality(String) DEFAULT '',
    country         LowCardinality(FixedString(2)) DEFAULT '',
    utm_source      String DEFAULT '',

    updated_at      DateTime64(3, 'UTC') DEFAULT now64(3)
)
ENGINE = ReplacingMergeTree(updated_at)
PARTITION BY (project_id, toYYYYMM(entered_at))
ORDER BY (project_id, funnel_id, visitor_id, session_id);
```

### Form Field Analytics

```sql
-- ClickHouse
CREATE TABLE form_field_stats (
    project_id      UUID,
    page_url_path   String,
    form_selector   String,
    field_selector  String,
    field_name      String DEFAULT '',
    period_date     Date,

    focus_count     Int64 DEFAULT 0,
    completion_count Int64 DEFAULT 0,
    abandon_count   Int64 DEFAULT 0,
    total_time_ms   Int64 DEFAULT 0,
    total_hesitation_ms Int64 DEFAULT 0,
    total_corrections Int64 DEFAULT 0,

    updated_at      DateTime64(3, 'UTC') DEFAULT now64(3)
)
ENGINE = SummingMergeTree()
PARTITION BY (project_id, toYYYYMM(period_date))
ORDER BY (project_id, page_url_path, form_selector, field_selector, period_date);
```

### Session Recording Metadata

```sql
-- ClickHouse
-- Recording metadata for fast search; actual recording data in object storage
CREATE TABLE recording_metadata (
    recording_id    UUID,
    session_id      UUID,
    project_id      UUID,
    visitor_id      UUID,

    duration_ms     Nullable(Int32),
    page_count      Int32 DEFAULT 0,
    event_count     Int32 DEFAULT 0,

    has_rage_clicks UInt8 DEFAULT 0,
    has_dead_clicks UInt8 DEFAULT 0,
    has_js_errors   UInt8 DEFAULT 0,
    has_form_abandons UInt8 DEFAULT 0,
    frustration_score Nullable(Float32),

    -- Denormalized for filtering
    device_type     LowCardinality(String) DEFAULT '',
    browser         LowCardinality(String) DEFAULT '',
    country         LowCardinality(FixedString(2)) DEFAULT '',
    entry_url       String DEFAULT '',

    storage_url     String DEFAULT '',
    storage_size_bytes Int64 DEFAULT 0,
    status          LowCardinality(String) DEFAULT 'recording',

    created_at      DateTime64(3, 'UTC'),
    expires_at      Nullable(DateTime64(3, 'UTC'))
)
ENGINE = ReplacingMergeTree(created_at)
PARTITION BY (project_id, toYYYYMM(created_at))
ORDER BY (project_id, recording_id);

ALTER TABLE recording_metadata ADD INDEX idx_frustration frustration_score TYPE minmax GRANULARITY 4;
ALTER TABLE recording_metadata ADD INDEX idx_entry entry_url TYPE bloom_filter GRANULARITY 4;
```

---

## ClickHouse Materialized Views (Real-Time Aggregation)

```sql
-- ClickHouse
-- Automatically aggregate heatmap tiles from raw events
CREATE MATERIALIZED VIEW mv_heatmap_tiles TO heatmap_tiles AS
SELECT
    project_id,
    url_path,
    event_type AS tile_type,
    device_type,
    toDate(occurred_at) AS period_date,
    intDiv(x_position, 20) AS tile_x,
    intDiv(y_position, 20) AS tile_y,
    20 AS tile_width,
    20 AS tile_height,
    count() AS interaction_count,
    uniqExact(visitor_id) AS unique_visitors,
    now64(3) AS updated_at
FROM events
WHERE event_type IN ('click', 'scroll', 'rage_click')
  AND x_position IS NOT NULL
  AND y_position IS NOT NULL
GROUP BY project_id, url_path, event_type, device_type, period_date, tile_x, tile_y;

-- Automatically aggregate experiment metrics from conversion events
CREATE MATERIALIZED VIEW mv_experiment_metrics TO experiment_metrics AS
SELECT
    project_id,
    experiment_id,
    variant_id,
    goal_id,
    toDate(occurred_at) AS period_date,
    countIf(event_type = 'conversion') AS conversions,
    sumIf(revenue, event_type = 'conversion') AS revenue_total,
    countIf(revenue IS NOT NULL AND event_type = 'conversion') AS revenue_count,
    uniqExact(visitor_id) AS unique_visitors,
    now64(3) AS updated_at
FROM events
WHERE experiment_id IS NOT NULL
GROUP BY project_id, experiment_id, variant_id, goal_id, period_date;
```

---

## Data Pipeline Architecture

```
Browser SDK  ---->  Ingestion API  ---->  Kafka / Message Queue
                                              |
                                    +---------+---------+
                                    |                   |
                              ClickHouse            PostgreSQL
                              (events,              (visitor upsert,
                               aggregations)         assignment sync)
                                    |
                              Materialized Views
                              (heatmap tiles,
                               experiment metrics,
                               form stats)
```

Events flow from the browser SDK to an ingestion API, which publishes CloudEvents-formatted messages to a Kafka topic. ClickHouse consumers ingest the raw events. Materialized views automatically produce aggregations. A separate processor syncs visitor identity and experiment assignment state to PostgreSQL.

---

## Example Queries

### Heatmap Tile Data (ClickHouse -- sub-second on billions of events)

```sql
-- ClickHouse
SELECT
    tile_x, tile_y,
    sum(interaction_count) AS total_interactions,
    sum(unique_visitors) AS total_visitors
FROM heatmap_tiles
WHERE project_id = 'project-uuid'
  AND url_path = '/pricing'
  AND tile_type = 'click'
  AND device_type = 'desktop'
  AND period_date BETWEEN '2026-05-01' AND '2026-05-19'
GROUP BY tile_x, tile_y
ORDER BY total_interactions DESC;
```

### Experiment Conversion Rates (ClickHouse)

```sql
-- ClickHouse
SELECT
    variant_id,
    sum(unique_visitors) AS sample_size,
    sum(conversions) AS total_conversions,
    sum(conversions) / sum(unique_visitors) AS conversion_rate,
    sum(revenue_total) AS total_revenue
FROM experiment_metrics
WHERE project_id = 'project-uuid'
  AND experiment_id = 'experiment-uuid'
GROUP BY variant_id;
```

### Funnel Analysis with Segmentation (ClickHouse)

```sql
-- ClickHouse
-- Step-by-step drop-off for mobile visitors from US
WITH funnel_data AS (
    SELECT
        visitor_id,
        maxIf(1, event_type = 'page_view' AND url_path LIKE '/product/%') AS step1,
        maxIf(1, event_name = 'Product Added') AS step2,
        maxIf(1, event_type = 'page_view' AND url_path = '/checkout') AS step3,
        maxIf(1, event_name = 'Order Completed') AS step4
    FROM events
    WHERE project_id = 'project-uuid'
      AND occurred_at >= '2026-05-01'
      AND device_type = 'mobile'
      AND country = 'US'
    GROUP BY visitor_id
)
SELECT
    countIf(step1 = 1) AS step1_count,
    countIf(step1 = 1 AND step2 = 1) AS step2_count,
    countIf(step1 = 1 AND step2 = 1 AND step3 = 1) AS step3_count,
    countIf(step1 = 1 AND step2 = 1 AND step3 = 1 AND step4 = 1) AS step4_count
FROM funnel_data;
```

### Cross-Store Query (Application Layer)

```python
# Application code: combine PostgreSQL experiment config with ClickHouse metrics

# 1. Get experiment details from PostgreSQL
experiment = pg_query("""
    SELECT e.*, json_agg(v.*) as variants
    FROM experiment e
    JOIN variant v ON v.experiment_id = e.id
    WHERE e.id = %s
    GROUP BY e.id
""", experiment_id)

# 2. Get metrics from ClickHouse
metrics = ch_query("""
    SELECT variant_id, sum(unique_visitors) AS sample_size,
           sum(conversions) AS conversions,
           sum(revenue_total) AS revenue
    FROM experiment_metrics
    WHERE experiment_id = %s
    GROUP BY variant_id
""", experiment_id)

# 3. Merge in application layer
for variant in experiment['variants']:
    variant['metrics'] = metrics.get(variant['id'], {})
```

---

## Table Count Summary

| Category | Store | Tables | Notes |
|----------|-------|--------|-------|
| Identity & Multi-Tenancy | PostgreSQL | 3 | organization, project, app_user |
| Visitor & Consent | PostgreSQL | 1 | visitor |
| Experimentation | PostgreSQL | 4 | experiment, variant, experiment_goal, hypothesis |
| Funnels & Segments | PostgreSQL | 3 | funnel, segment, personalization_campaign |
| AI & Audit | PostgreSQL | 2 | ai_insight, audit_log (partitioned) |
| **PostgreSQL subtotal** | | **13** | |
| Raw Events | ClickHouse | 1 | events (high volume) |
| Session Summaries | ClickHouse | 1 | session_summaries |
| Experiment Metrics | ClickHouse | 1 | experiment_metrics (SummingMergeTree) |
| Heatmap Tiles | ClickHouse | 1 | heatmap_tiles (SummingMergeTree) |
| Funnel Progress | ClickHouse | 1 | funnel_progress |
| Form Analytics | ClickHouse | 1 | form_field_stats (SummingMergeTree) |
| Recording Metadata | ClickHouse | 1 | recording_metadata |
| Materialized Views | ClickHouse | 2 | mv_heatmap_tiles, mv_experiment_metrics |
| **ClickHouse subtotal** | | **9** | Plus 2 materialized views |
| **Combined total** | | **~22** | 13 PostgreSQL + 9 ClickHouse |

---

## Key Design Decisions

1. **Two databases, each playing to its strength** — PostgreSQL handles transactional CRUD with ACID guarantees for experiment management, user identity, and configuration. ClickHouse handles columnar analytics for high-volume event data where aggregation speed matters more than transactional guarantees.

2. **Denormalized ClickHouse tables** — visitor context (device_type, browser, country, UTM parameters) is denormalized into the events table and session summaries. ClickHouse performs best with denormalized data where JOINs are avoided. This trades storage efficiency for query speed.

3. **SummingMergeTree for incremental aggregation** — heatmap tiles, experiment metrics, and form field stats use ClickHouse's SummingMergeTree engine, which automatically sums numeric columns on merge for rows with the same ORDER BY key. New events produce new rows that are automatically merged, keeping aggregates up to date without full recomputation.

4. **ReplacingMergeTree for mutable summaries** — session summaries and funnel progress use ReplacingMergeTree, which keeps only the latest version of each row (by session_id or visitor+funnel). This handles the fact that session data is updated as the session progresses.

5. **Materialized views for real-time aggregation** — ClickHouse materialized views automatically transform raw events into heatmap tiles and experiment metrics at ingestion time, eliminating the need for batch processing jobs.

6. **TTL-based data lifecycle** — ClickHouse tables use TTL expressions (e.g., 13 months) for automatic data expiration, supporting GDPR data retention requirements without manual cleanup jobs.

7. **Kafka as the bridge** — the event pipeline uses Kafka (or a compatible message queue) as the buffer between the ingestion API and both data stores. This decouples ingestion throughput from storage write speed and enables replay if either store falls behind.

8. **Application-level cross-store joins** — queries that need both experiment configuration (PostgreSQL) and metrics (ClickHouse) are joined in the application layer. This is a deliberate trade-off: it adds application complexity but avoids the performance penalty of cross-database queries.

9. **Bloom filter indexes on ClickHouse** — secondary bloom filter indexes on high-cardinality columns (url_path, event_name, visitor_id, experiment_id) provide efficient point lookups without the overhead of B-tree indexes on columnar data.

10. **Session recordings in object storage** — actual recording data (compressed rrweb streams) lives in object storage (S3/GCS), with metadata searchable in ClickHouse. This keeps both databases lean while supporting terabytes of recording data.
