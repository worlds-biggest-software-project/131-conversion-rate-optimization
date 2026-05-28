# Conversion Rate Optimization — Phased Development Plan

> Project: 131-conversion-rate-optimization · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary Language | TypeScript (full-stack) | Unified language for browser SDK, ingestion API, dashboard, and event processors; mature ecosystem for real-time web analytics; Anthropic SDK and statistical libraries available; enables shared types between client tracking code and server processing |
| API Framework | Next.js 15 (App Router) | Combines dashboard UI (React Server Components), REST API routes, and server-side rendering; middleware for auth and rate limiting; built-in static optimization for documentation pages |
| Operational Database | PostgreSQL 16 | JSONB with GIN indexes supports the hybrid relational+JSONB data model (Data Model Suggestion 3); UUID generation, partitioning, and row-level security for multi-tenancy; mature GDPR compliance tooling |
| Analytics Database | ClickHouse | Columnar storage for high-volume clickstream events (millions per day); sub-second aggregation for heatmap tiles, funnel analysis, and experiment metrics; SummingMergeTree and materialized views for real-time aggregation (Data Model Suggestion 4 architecture) |
| ORM / Query Builder | Drizzle ORM | Type-safe PostgreSQL schema definition; generates migrations; native JSONB and UUID support; lightweight compared to Prisma with better raw SQL escape hatches |
| ClickHouse Client | @clickhouse/client | Official ClickHouse TypeScript client; supports streaming inserts for high-throughput event ingestion; parameterized queries |
| Message Queue | Apache Kafka (via KafkaJS) | Bridges browser SDK events to both ClickHouse and PostgreSQL; replay capability for rebuilding materialized views; CloudEvents envelope format; handles burst traffic from high-volume sites |
| Task Queue | BullMQ (Redis-backed) | Async AI hypothesis generation, statistical analysis, anomaly detection, and session recording processing; priority queues and rate limiting for LLM API calls; Bull Board for monitoring |
| Frontend UI | React 19 + Tailwind CSS 4 + shadcn/ui | Dashboard with heatmap overlays, session replay player, funnel visualizations, and experiment configuration; Tailwind for design tokens; shadcn/ui for accessible form components and data tables |
| Session Recording | rrweb | Industry-standard DOM snapshot and incremental recording library; produces compressed event streams; existing player component (rrweb-player) for replay UI; MIT licensed |
| Heatmap Rendering | Canvas API + custom renderer | Click/scroll/movement heatmaps rendered as canvas overlays on page screenshots; custom tile-based renderer reads pre-aggregated tile data from ClickHouse |
| Statistical Engine | Custom (jStat + stdlib) | Frequentist confidence intervals (z-test, chi-squared) and Bayesian analysis (Beta-Binomial for Thompson Sampling); jStat provides distribution functions; custom implementation avoids vendor lock-in for bandit allocation |
| LLM Provider | Anthropic Claude (claude-sonnet-4-6) | AI hypothesis generation from heatmap/session data; natural language querying; qualitative feedback synthesis; anomaly explanation; structured output via tool use |
| Browser SDK | Custom TypeScript SDK | Lightweight (<15KB gzipped) tracking snippet for clicks, scrolls, mouse movement, page views, form interactions, and rage-click detection; consent-gated data collection; first-party cookie identity |
| Server-Side SDK | Custom TypeScript SDK | Feature flag evaluation, experiment assignment, and server-side event tracking; OpenFeature-compatible provider interface |
| Schema Validation | Zod | Runtime validation of API inputs, SDK payloads, configuration, and LLM outputs; generates TypeScript types from schemas; used at all ingestion boundaries |
| Authentication | NextAuth.js v5 | OAuth 2.0 (Google, GitHub) and email/password; JWT sessions; supports multi-tenant RBAC; OpenID Connect compliant |
| Testing | Vitest + Playwright | Vitest for unit/integration (fast, ESM-native); Playwright for E2E testing of dashboard, WYSIWYG editor, and session replay |
| Linting & Formatting | Biome | Single tool for lint + format; TypeScript-aware; 10-30x faster than ESLint + Prettier |
| Package Manager | pnpm | Strict dependency resolution; workspace support for monorepo; efficient disk usage |
| Containerisation | Docker + Docker Compose | Local development orchestrates app + PostgreSQL + ClickHouse + Redis + Kafka; production deployment via Docker images |
| Object Storage | S3-compatible (MinIO for self-hosted) | Session recording compressed chunks; heatmap screenshots; large export files |
| Event Format | CloudEvents 1.0 | Standard envelope for all events flowing through Kafka; portable to external systems |
| Event Taxonomy | Segment Spec (Track/Identify/Page) | Naming convention for custom events; familiar to marketing/analytics teams |
| Feature Flag Interface | OpenFeature SDK | Vendor-neutral experiment assignment API; enables third-party provider integration |

### Project Structure

```
conversion-rate-optimization/
├── package.json
├── pnpm-workspace.yaml
├── biome.json
├── docker-compose.yml
├── Dockerfile
├── turbo.json
├── packages/
│   ├── web/                              # Next.js application (Dashboard + API)
│   │   ├── next.config.ts
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── (auth)/               # Login, signup, OAuth callback
│   │   │   │   ├── (dashboard)/          # Project dashboard, experiment list
│   │   │   │   │   ├── experiments/      # Experiment CRUD, results, WYSIWYG editor
│   │   │   │   │   ├── heatmaps/        # Heatmap viewer
│   │   │   │   │   ├── recordings/      # Session recording list + player
│   │   │   │   │   ├── funnels/         # Funnel builder + visualization
│   │   │   │   │   ├── forms/           # Form analytics dashboard
│   │   │   │   │   ├── hypotheses/      # Hypothesis backlog + AI suggestions
│   │   │   │   │   ├── segments/        # Segment builder
│   │   │   │   │   ├── insights/        # AI insights feed
│   │   │   │   │   └── settings/        # Project + org settings
│   │   │   │   └── api/
│   │   │   │       ├── v1/              # Public REST API
│   │   │   │       │   ├── experiments/
│   │   │   │       │   ├── events/      # Event ingestion endpoint
│   │   │   │       │   ├── heatmaps/
│   │   │   │       │   ├── recordings/
│   │   │   │       │   ├── funnels/
│   │   │   │       │   ├── segments/
│   │   │   │       │   ├── hypotheses/
│   │   │   │       │   └── insights/
│   │   │   │       └── internal/        # Dashboard-specific endpoints
│   │   │   ├── components/
│   │   │   │   ├── heatmap/             # Heatmap overlay renderer
│   │   │   │   ├── recording-player/    # rrweb replay components
│   │   │   │   ├── experiment/          # WYSIWYG editor, variant builder
│   │   │   │   ├── funnel/             # Funnel chart components
│   │   │   │   ├── stats/              # Statistical result displays
│   │   │   │   └── ui/                 # shadcn/ui primitives
│   │   │   ├── lib/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── db.ts
│   │   │   │   ├── clickhouse.ts
│   │   │   │   ├── queue.ts
│   │   │   │   └── kafka.ts
│   │   │   └── styles/
│   │   └── drizzle/
│   │       ├── schema.ts
│   │       └── migrations/
│   ├── core/                             # Shared business logic
│   │   ├── src/
│   │   │   ├── experiments/              # Experiment lifecycle, assignment, bucketing
│   │   │   ├── statistics/               # Frequentist + Bayesian analysis
│   │   │   ├── bandit/                   # Thompson Sampling allocation
│   │   │   ├── heatmap/                  # Tile aggregation logic
│   │   │   ├── funnel/                   # Funnel computation engine
│   │   │   ├── forms/                    # Form analytics aggregation
│   │   │   ├── friction/                 # Friction signal detection
│   │   │   ├── segmentation/             # Segment rule evaluation
│   │   │   ├── personalization/          # Variant delivery engine
│   │   │   ├── ai/                       # Hypothesis generation, NLQ, anomaly detection
│   │   │   ├── consent/                  # Consent gate logic
│   │   │   └── events/                   # CloudEvents builder, Segment Spec validation
│   │   └── package.json
│   ├── db/                               # PostgreSQL schema + queries
│   │   ├── src/
│   │   │   ├── schema.ts                 # Drizzle schema definition
│   │   │   ├── queries/                  # Typed query functions per entity
│   │   │   └── migrate.ts
│   │   └── package.json
│   ├── sdk-browser/                      # Browser tracking SDK
│   │   ├── src/
│   │   │   ├── tracker.ts                # Core tracking engine
│   │   │   ├── heatmap-collector.ts      # Click/scroll/movement data collection
│   │   │   ├── session-recorder.ts       # rrweb integration
│   │   │   ├── form-tracker.ts           # Form field interaction tracking
│   │   │   ├── friction-detector.ts      # Rage click, dead click detection
│   │   │   ├── consent.ts                # Consent gate
│   │   │   ├── identity.ts               # First-party cookie management
│   │   │   └── transport.ts              # Beacon/fetch event sender
│   │   ├── rollup.config.js
│   │   └── package.json
│   ├── sdk-server/                       # Server-side SDK
│   │   ├── src/
│   │   │   ├── client.ts                 # Server client for experiment assignment
│   │   │   ├── openfeature-provider.ts   # OpenFeature provider implementation
│   │   │   └── events.ts                 # Server-side event tracking
│   │   └── package.json
│   └── workers/                          # Background event processors
│       ├── src/
│       │   ├── event-ingestion.ts        # Kafka → ClickHouse consumer
│       │   ├── session-aggregator.ts     # Session summary builder
│       │   ├── heatmap-aggregator.ts     # Heatmap tile generator
│       │   ├── funnel-processor.ts       # Funnel progress tracker
│       │   ├── form-aggregator.ts        # Form field stats aggregation
│       │   ├── stats-calculator.ts       # Experiment statistical analysis
│       │   ├── bandit-updater.ts         # Thompson Sampling posterior updates
│       │   ├── anomaly-detector.ts       # Experiment anomaly monitoring
│       │   ├── recording-processor.ts    # Session recording compression + storage
│       │   ├── ai-hypothesis.ts          # AI hypothesis generation worker
│       │   └── ai-insight.ts             # AI insight generation worker
│       └── package.json
├── fixtures/                             # Test fixtures
│   ├── events/                           # Sample clickstream event batches
│   ├── recordings/                       # Sample rrweb recording data
│   └── experiments/                      # Sample experiment configurations
└── docs/
    └── architecture.md
```

---

## Phase 1: Foundation & Project Infrastructure

### Purpose
Establish the monorepo scaffolding, PostgreSQL database schema, authentication system, and multi-tenant project management CRUD. After this phase, a user can sign up, create an organization and project, and obtain a snippet key for SDK integration. This is the skeleton that every subsequent phase builds upon.

### Tasks

#### 1.1 — Monorepo Scaffolding & Tooling

**What**: Initialize the pnpm monorepo with all packages, configure TypeScript, Biome, Vitest, Docker Compose (PostgreSQL, ClickHouse, Redis, Kafka), and CI pipeline.

**Design**:

Root `package.json`:
```json
{
  "private": true,
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "biome check .",
    "db:migrate": "pnpm --filter @cro/db migrate",
    "db:push": "pnpm --filter @cro/db push"
  }
}
```

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: cro
      POSTGRES_USER: cro
      POSTGRES_PASSWORD: cro_dev
    ports: ["5432:5432"]
    volumes: ["pg_data:/var/lib/postgresql/data"]

  clickhouse:
    image: clickhouse/clickhouse-server:24.6
    ports: ["8123:8123", "9000:9000"]
    volumes: ["ch_data:/var/lib/clickhouse"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  kafka:
    image: bitnami/kafka:3.7
    environment:
      KAFKA_CFG_NODE_ID: 0
      KAFKA_CFG_PROCESS_ROLES: controller,broker
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@kafka:9093
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
    ports: ["9092:9092"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["minio_data:/data"]

volumes:
  pg_data:
  ch_data:
  minio_data:
```

Environment configuration (`packages/web/.env.example`):
```
DATABASE_URL=postgresql://cro:cro_dev@localhost:5432/cro
CLICKHOUSE_URL=http://localhost:8123
CLICKHOUSE_DATABASE=cro
REDIS_URL=redis://localhost:6379
KAFKA_BROKERS=localhost:9092
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=cro-recordings
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
NEXTAUTH_SECRET=dev-secret-change-in-production
NEXTAUTH_URL=http://localhost:3000
ANTHROPIC_API_KEY=
```

**Testing**:
- `Unit: biome check returns no lint errors on scaffolded code`
- `Unit: TypeScript compilation succeeds across all packages with strict mode`
- `Integration: docker-compose up starts all services without errors`
- `Integration: PostgreSQL accepts connections on port 5432`
- `Integration: ClickHouse accepts HTTP queries on port 8123`
- `Integration: Redis responds to PING on port 6379`
- `Integration: Kafka broker is reachable on port 9092`

#### 1.2 — PostgreSQL Schema & Migrations

**What**: Implement the operational database schema using Drizzle ORM, covering organizations, projects, users, visitors, and consent.

**Design**:

Drizzle schema (`packages/db/src/schema.ts`):
```typescript
import { pgTable, uuid, varchar, text, timestamp, jsonb, uniqueIndex, index, boolean } from 'drizzle-orm/pg-core';

export const organizations = pgTable('organization', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  planTier: varchar('plan_tier', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const projects = pgTable('project', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  name: varchar('name', { length: 255 }).notNull(),
  domain: varchar('domain', { length: 255 }),
  snippetKey: varchar('snippet_key', { length: 64 }).notNull().unique(),
  timezone: varchar('timezone', { length: 50 }).notNull().default('UTC'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_project_org').on(table.organizationId),
]);

export const appUsers = pgTable('app_user', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  email: varchar('email', { length: 255 }).notNull(),
  name: varchar('name', { length: 255 }),
  role: varchar('role', { length: 50 }).notNull().default('viewer'),
  passwordHash: varchar('password_hash', { length: 255 }),
  preferences: jsonb('preferences').notNull().default({}),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_app_user_org_email').on(table.organizationId, table.email),
]);

export const visitors = pgTable('visitor', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id),
  anonymousId: varchar('anonymous_id', { length: 255 }).notNull(),
  userId: varchar('user_id', { length: 255 }),
  traits: jsonb('traits').notNull().default({}),
  consent: jsonb('consent').notNull().default({}),
  firstSeenAt: timestamp('first_seen_at', { withTimezone: true }).notNull().defaultNow(),
  lastSeenAt: timestamp('last_seen_at', { withTimezone: true }).notNull().defaultNow(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_visitor_anon').on(table.projectId, table.anonymousId),
  index('idx_visitor_user').on(table.projectId, table.userId),
]);
```

Query functions (`packages/db/src/queries/organizations.ts`):
```typescript
import { eq } from 'drizzle-orm';
import type { DrizzleClient } from '../client';
import { organizations } from '../schema';

export interface CreateOrganizationInput {
  name: string;
  slug: string;
  planTier?: 'free' | 'pro' | 'enterprise';
}

export async function createOrganization(db: DrizzleClient, input: CreateOrganizationInput) {
  const [org] = await db.insert(organizations).values(input).returning();
  return org;
}

export async function getOrganizationBySlug(db: DrizzleClient, slug: string) {
  return db.query.organizations.findFirst({ where: eq(organizations.slug, slug) });
}

export async function getOrganizationById(db: DrizzleClient, id: string) {
  return db.query.organizations.findFirst({ where: eq(organizations.id, id) });
}
```

**Testing**:
- `Unit: schema compiles and generates valid SQL DDL`
- `Integration: drizzle-kit push creates all tables in PostgreSQL`
- `Integration: createOrganization inserts row and returns UUID`
- `Integration: duplicate slug → unique constraint violation error`
- `Integration: createProject with valid org ID succeeds; invalid org ID → foreign key error`
- `Unit: createProject generates unique snippet_key`
- `Integration: visitor upsert by anonymous_id merges traits correctly`

#### 1.3 — Authentication & Multi-Tenant Authorization

**What**: Implement NextAuth.js v5 with email/password and OAuth (Google, GitHub), JWT sessions with organization and role claims, and RBAC middleware.

**Design**:

Auth configuration (`packages/web/src/lib/auth.ts`):
```typescript
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import Google from 'next-auth/providers/google';
import GitHub from 'next-auth/providers/github';
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Google({ clientId: process.env.GOOGLE_CLIENT_ID!, clientSecret: process.env.GOOGLE_CLIENT_SECRET! }),
    GitHub({ clientId: process.env.GITHUB_CLIENT_ID!, clientSecret: process.env.GITHUB_CLIENT_SECRET! }),
    Credentials({
      credentials: { email: { type: 'email' }, password: { type: 'password' } },
      async authorize(credentials) {
        const { email, password } = loginSchema.parse(credentials);
        // Verify against app_user table
        // Return user with organizationId and role
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.organizationId = user.organizationId;
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      session.user.organizationId = token.organizationId as string;
      session.user.role = token.role as string;
      return session;
    },
  },
});
```

RBAC middleware types:
```typescript
export type Role = 'admin' | 'editor' | 'analyst' | 'viewer';

export const ROLE_HIERARCHY: Record<Role, number> = {
  admin: 40,
  editor: 30,
  analyst: 20,
  viewer: 10,
};

export function requireRole(minimumRole: Role) {
  return async function middleware(req: NextRequest) {
    const session = await auth();
    if (!session?.user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    const userLevel = ROLE_HIERARCHY[session.user.role as Role] ?? 0;
    const requiredLevel = ROLE_HIERARCHY[minimumRole];
    if (userLevel < requiredLevel) return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
  };
}
```

**Testing**:
- `Unit: loginSchema rejects empty email → ZodError`
- `Unit: loginSchema rejects password < 8 chars → ZodError`
- `Unit: ROLE_HIERARCHY admin > editor > analyst > viewer`
- `Unit: requireRole('editor') allows admin → passes`
- `Unit: requireRole('editor') blocks viewer → 403`
- `Integration (mocked): OAuth callback creates app_user if not exists`
- `Integration (mocked): OAuth callback for existing user updates last_login_at`
- `E2E: login with valid credentials → redirects to dashboard`
- `E2E: login with invalid credentials → shows error message`

#### 1.4 — Organization & Project Management API

**What**: REST API endpoints for CRUD operations on organizations and projects, including snippet key generation.

**Design**:

API routes:
```
POST   /api/v1/organizations          → Create organization
GET    /api/v1/organizations/:id      → Get organization
PATCH  /api/v1/organizations/:id      → Update organization
POST   /api/v1/projects               → Create project (generates snippet_key)
GET    /api/v1/projects               → List projects for current org
GET    /api/v1/projects/:id           → Get project with settings
PATCH  /api/v1/projects/:id           → Update project
DELETE /api/v1/projects/:id           → Soft-delete project
GET    /api/v1/projects/:id/snippet   → Get JS snippet embed code
```

Request/response schemas:
```typescript
// Zod schemas for validation
export const CreateProjectSchema = z.object({
  name: z.string().min(1).max(255),
  domain: z.string().url().optional(),
  timezone: z.string().default('UTC'),
  settings: z.object({
    recordingSampleRate: z.number().min(0).max(1).default(1.0),
    heatmapTileSize: z.number().int().min(5).max(50).default(20),
    consentRequired: z.boolean().default(true),
    excludedIps: z.array(z.string()).default([]),
    customDimensions: z.array(z.string()).max(20).default([]),
  }).default({}),
});

export const ProjectResponse = z.object({
  id: z.string().uuid(),
  organizationId: z.string().uuid(),
  name: z.string(),
  domain: z.string().nullable(),
  snippetKey: z.string(),
  timezone: z.string(),
  settings: z.record(z.unknown()),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});
```

Snippet key generation:
```typescript
import { randomBytes } from 'crypto';

export function generateSnippetKey(): string {
  return `cro_${randomBytes(24).toString('base64url')}`;
}
```

**Testing**:
- `Unit: generateSnippetKey produces 36-char string starting with "cro_"`
- `Unit: CreateProjectSchema validates valid input → success`
- `Unit: CreateProjectSchema rejects recordingSampleRate > 1 → ZodError`
- `Integration: POST /api/v1/projects → 201 with snippet_key in response`
- `Integration: GET /api/v1/projects → returns only projects in caller's organization`
- `Integration: POST /api/v1/projects without auth → 401`
- `Integration: viewer role POST /api/v1/projects → 403`
- `Integration: DELETE /api/v1/projects/:id sets deleted flag, does not remove row`

#### 1.5 — ClickHouse Schema Initialization

**What**: Create the ClickHouse analytics tables for events, session summaries, experiment metrics, heatmap tiles, funnel progress, form field stats, and recording metadata.

**Design**:

Migration script (`packages/db/src/clickhouse-schema.sql`):
```sql
CREATE DATABASE IF NOT EXISTS cro;

CREATE TABLE IF NOT EXISTS cro.events (
    event_id        UUID,
    project_id      UUID,
    session_id      UUID,
    visitor_id      UUID,
    anonymous_id    String,
    event_type      LowCardinality(String),
    event_name      Nullable(String),
    url             String,
    url_path        String,
    url_host        String,
    element_selector Nullable(String),
    element_text    Nullable(String),
    x_position      Nullable(Int32),
    y_position      Nullable(Int32),
    scroll_depth_pct Nullable(Float32),
    ttfb_ms         Nullable(Int32),
    dom_ready_ms    Nullable(Int32),
    load_time_ms    Nullable(Int32),
    lcp_ms          Nullable(Int32),
    fid_ms          Nullable(Int32),
    cls             Nullable(Float32),
    viewport_width  Nullable(Int32),
    viewport_height Nullable(Int32),
    form_selector   Nullable(String),
    field_selector  Nullable(String),
    field_name      Nullable(String),
    time_in_field_ms Nullable(Int32),
    hesitation_ms   Nullable(Int32),
    corrections     Nullable(Int32),
    revenue         Nullable(Decimal(15,2)),
    currency        Nullable(FixedString(3)),
    experiment_id   Nullable(UUID),
    variant_id      Nullable(UUID),
    goal_id         Nullable(UUID),
    is_rage_click   UInt8 DEFAULT 0,
    is_dead_click   UInt8 DEFAULT 0,
    device_type     LowCardinality(String) DEFAULT '',
    browser         LowCardinality(String) DEFAULT '',
    os              LowCardinality(String) DEFAULT '',
    country         LowCardinality(FixedString(2)) DEFAULT '',
    utm_source      Nullable(String),
    utm_medium      Nullable(String),
    utm_campaign    Nullable(String),
    properties      String DEFAULT '{}',
    occurred_at     DateTime64(3, 'UTC'),
    ingested_at     DateTime64(3, 'UTC') DEFAULT now64(3)
)
ENGINE = MergeTree()
PARTITION BY (project_id, toYYYYMM(occurred_at))
ORDER BY (project_id, event_type, occurred_at, session_id)
TTL occurred_at + INTERVAL 13 MONTH
SETTINGS index_granularity = 8192;

-- Session summaries, experiment_metrics, heatmap_tiles, funnel_progress,
-- form_field_stats, recording_metadata tables as defined in
-- data-model-suggestion-4.md ClickHouse section
```

ClickHouse client wrapper (`packages/web/src/lib/clickhouse.ts`):
```typescript
import { createClient, type ClickHouseClient } from '@clickhouse/client';

let client: ClickHouseClient | null = null;

export function getClickHouseClient(): ClickHouseClient {
  if (!client) {
    client = createClient({
      url: process.env.CLICKHOUSE_URL ?? 'http://localhost:8123',
      database: process.env.CLICKHOUSE_DATABASE ?? 'cro',
      clickhouse_settings: {
        async_insert: 1,
        wait_for_async_insert: 0,
      },
    });
  }
  return client;
}
```

**Testing**:
- `Integration: ClickHouse schema migration creates all tables without errors`
- `Integration: INSERT into cro.events succeeds with sample event data`
- `Integration: SELECT from cro.events with project_id filter returns inserted rows`
- `Integration: SummingMergeTree tables (heatmap_tiles, experiment_metrics) sum correctly on OPTIMIZE`
- `Unit: getClickHouseClient returns singleton instance`

---

## Phase 2: Browser SDK & Event Ingestion Pipeline

### Purpose
Build the browser tracking SDK and the end-to-end event ingestion pipeline from browser to ClickHouse. After this phase, a website can embed the SDK snippet, and click/scroll/pageview events flow through Kafka into ClickHouse for querying. This is the data foundation for all analytics features.

### Tasks

#### 2.1 — Browser SDK Core (Tracking & Identity)

**What**: Lightweight TypeScript SDK (<15KB gzipped) that tracks page views, clicks, scrolls, and manages first-party cookie identity with consent gating.

**Design**:

SDK initialization API:
```typescript
// Public API exposed as window.CRO
export interface CROConfig {
  snippetKey: string;
  apiEndpoint?: string;                  // defaults to same-origin /api/v1/events
  consentRequired?: boolean;             // default true
  sampleRate?: number;                   // 0.0-1.0, default 1.0
  cookieDomain?: string;                 // default: current domain
  cookieMaxAgeDays?: number;             // default: 365
  excludeSelectors?: string[];           // CSS selectors to exclude from tracking
  respectDNT?: boolean;                  // default true (W3C Tracking Preference Expression)
}

export interface CROTracker {
  init(config: CROConfig): void;
  identify(userId: string, traits?: Record<string, unknown>): void;
  track(eventName: string, properties?: Record<string, unknown>): void;
  setConsent(consent: ConsentMap): void;
  getVisitorId(): string;
  reset(): void;
}

export interface ConsentMap {
  analytics?: boolean;
  sessionRecording?: boolean;
  personalization?: boolean;
}
```

Identity management:
```typescript
export class IdentityManager {
  private anonymousId: string;
  private userId: string | null = null;
  private cookieName = '_cro_id';

  constructor(private config: CROConfig) {
    this.anonymousId = this.readCookie() ?? this.generateId();
    this.writeCookie(this.anonymousId);
  }

  private generateId(): string {
    // UUID v4 using crypto.randomUUID() with fallback
    return crypto.randomUUID?.() ?? self.crypto.getRandomValues(new Uint8Array(16))
      .reduce((s, b) => s + b.toString(16).padStart(2, '0'), '');
  }

  identify(userId: string): void {
    this.userId = userId;
  }

  getIdentity(): { anonymousId: string; userId: string | null } {
    return { anonymousId: this.anonymousId, userId: this.userId };
  }
}
```

Event batch transport:
```typescript
export class EventTransport {
  private buffer: CROEvent[] = [];
  private flushInterval: number;
  private maxBatchSize = 20;
  private flushIntervalMs = 2000;

  constructor(private endpoint: string, private snippetKey: string) {
    this.flushInterval = window.setInterval(() => this.flush(), this.flushIntervalMs);
    window.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') this.flush();
    });
    window.addEventListener('pagehide', () => this.flush());
  }

  enqueue(event: CROEvent): void {
    this.buffer.push(event);
    if (this.buffer.length >= this.maxBatchSize) this.flush();
  }

  private flush(): void {
    if (this.buffer.length === 0) return;
    const batch = this.buffer.splice(0, this.maxBatchSize);
    const payload = JSON.stringify({ snippetKey: this.snippetKey, events: batch });
    // Use sendBeacon for reliability on page unload, fetch otherwise
    if (document.visibilityState === 'hidden' && navigator.sendBeacon) {
      navigator.sendBeacon(this.endpoint, payload);
    } else {
      fetch(this.endpoint, { method: 'POST', body: payload, keepalive: true });
    }
  }
}
```

Click/scroll/pageview auto-tracking:
```typescript
export interface CROEvent {
  eventId: string;
  eventType: string;            // page_view, click, scroll, custom
  eventName?: string;           // Segment Spec: "Product Added"
  url: string;
  urlPath: string;
  timestamp: string;            // ISO 8601
  sessionId: string;
  anonymousId: string;
  userId?: string;
  properties: Record<string, unknown>;
}
```

**Testing**:
- `Unit: IdentityManager generates valid UUID on first visit`
- `Unit: IdentityManager reads existing cookie and reuses anonymousId`
- `Unit: IdentityManager.identify sets userId`
- `Unit: EventTransport batches events and flushes at maxBatchSize`
- `Unit: EventTransport flushes on visibilitychange → hidden`
- `Unit: consentRequired=true + no consent → events not enqueued`
- `Unit: consentRequired=true + consent granted → events enqueued`
- `Unit: respectDNT=true + navigator.doNotTrack=1 → tracking disabled`
- `Unit: click on excluded selector → no click event emitted`
- `Unit: page_view event includes url, urlPath, title, viewport dimensions`
- `Fixture: sample click events match CROEvent schema`

#### 2.2 — Event Ingestion API & Kafka Producer

**What**: REST endpoint that receives batched events from the browser SDK, validates them, wraps in CloudEvents envelope, and publishes to Kafka.

**Design**:

Ingestion endpoint (`POST /api/v1/events`):
```typescript
import { z } from 'zod';

const EventPayloadSchema = z.object({
  snippetKey: z.string(),
  events: z.array(z.object({
    eventId: z.string().uuid(),
    eventType: z.enum(['page_view', 'click', 'scroll', 'custom', 'identify',
                        'rage_click', 'dead_click', 'form_focus', 'form_blur',
                        'form_submit', 'js_error', 'conversion']),
    eventName: z.string().optional(),
    url: z.string(),
    urlPath: z.string(),
    timestamp: z.string().datetime(),
    sessionId: z.string(),
    anonymousId: z.string(),
    userId: z.string().optional(),
    properties: z.record(z.unknown()).default({}),
  })).min(1).max(100),
});

export async function POST(req: Request) {
  const body = await req.json();
  const { snippetKey, events } = EventPayloadSchema.parse(body);

  // Validate snippet key → get project_id
  const project = await getProjectBySnippetKey(snippetKey);
  if (!project) return Response.json({ error: 'Invalid snippet key' }, { status: 401 });

  // Wrap each event in CloudEvents envelope and publish to Kafka
  const cloudEvents = events.map(event => ({
    specversion: '1.0',
    type: `cro.visitor.${event.eventType}`,
    source: `/projects/${project.id}`,
    id: event.eventId,
    time: event.timestamp,
    subject: event.sessionId,
    data: {
      projectId: project.id,
      ...event,
    },
  }));

  await publishToKafka('cro.events.raw', cloudEvents);
  return Response.json({ accepted: events.length }, { status: 202 });
}
```

Kafka producer wrapper:
```typescript
import { Kafka, type Producer, CompressionTypes } from 'kafkajs';

export class EventProducer {
  private producer: Producer;

  constructor(brokers: string[]) {
    const kafka = new Kafka({ brokers, clientId: 'cro-ingestion' });
    this.producer = kafka.producer({
      maxInFlightRequests: 5,
      idempotent: true,
    });
  }

  async publish(topic: string, events: CloudEvent[]): Promise<void> {
    await this.producer.send({
      topic,
      compression: CompressionTypes.LZ4,
      messages: events.map(e => ({
        key: e.data.projectId,
        value: JSON.stringify(e),
        headers: { 'ce-type': e.type, 'ce-source': e.source },
      })),
    });
  }
}
```

**Testing**:
- `Unit: EventPayloadSchema validates valid batch → success`
- `Unit: EventPayloadSchema rejects batch > 100 events → ZodError`
- `Unit: EventPayloadSchema rejects unknown event_type → ZodError`
- `Integration (mocked Kafka): valid POST → 202 with accepted count`
- `Integration (mocked Kafka): invalid snippet_key → 401`
- `Integration (mocked Kafka): events wrapped in CloudEvents envelope with correct type/source`
- `Integration: events published to Kafka with LZ4 compression and project_id as key`
- `Load: 1000 concurrent event batches → all return 202 within 500ms`

#### 2.3 — Kafka → ClickHouse Event Consumer

**What**: Background worker that consumes raw events from Kafka and inserts them into the ClickHouse `events` table with denormalized visitor context.

**Design**:

Consumer worker (`packages/workers/src/event-ingestion.ts`):
```typescript
import { Kafka, type EachBatchPayload } from 'kafkajs';
import { getClickHouseClient } from '@cro/web/lib/clickhouse';

export class EventIngestionWorker {
  private kafka: Kafka;
  private batchSize = 1000;
  private flushIntervalMs = 1000;

  async start(): Promise<void> {
    const consumer = this.kafka.consumer({ groupId: 'cro-event-ingestion' });
    await consumer.subscribe({ topic: 'cro.events.raw', fromBeginning: false });

    await consumer.run({
      eachBatch: async ({ batch, resolveOffset, heartbeat }: EachBatchPayload) => {
        const rows = batch.messages.map(msg => {
          const event = JSON.parse(msg.value!.toString());
          return this.transformToClickHouseRow(event.data);
        });

        await this.insertBatch(rows);

        for (const msg of batch.messages) {
          resolveOffset(msg.offset);
        }
        await heartbeat();
      },
    });
  }

  private transformToClickHouseRow(event: RawEvent): ClickHouseEventRow {
    return {
      event_id: event.eventId,
      project_id: event.projectId,
      session_id: event.sessionId,
      visitor_id: event.visitorId ?? '',
      anonymous_id: event.anonymousId,
      event_type: event.eventType,
      event_name: event.eventName ?? null,
      url: event.url,
      url_path: event.urlPath,
      url_host: new URL(event.url).hostname,
      occurred_at: event.timestamp,
      // ... map remaining fields from properties
    };
  }

  private async insertBatch(rows: ClickHouseEventRow[]): Promise<void> {
    const client = getClickHouseClient();
    await client.insert({
      table: 'cro.events',
      values: rows,
      format: 'JSONEachRow',
    });
  }
}
```

**Testing**:
- `Unit: transformToClickHouseRow maps all CloudEvent fields correctly`
- `Unit: transformToClickHouseRow extracts url_host from URL`
- `Unit: transformToClickHouseRow handles missing optional fields (null instead of undefined)`
- `Integration: publish 100 events to Kafka → consumer inserts 100 rows into ClickHouse events table`
- `Integration: consumer handles malformed messages by logging and skipping (dead letter)`
- `Integration: consumer commits offsets after successful insert`
- `Integration: consumer restarts from last committed offset after crash`

#### 2.4 — Visitor Upsert & Session Management

**What**: Worker process that syncs visitor identity from incoming events to PostgreSQL and manages session boundaries (30-minute inactivity timeout).

**Design**:

Session boundary detection:
```typescript
export const SESSION_TIMEOUT_MS = 30 * 60 * 1000; // 30 minutes

export interface SessionState {
  sessionId: string;
  projectId: string;
  visitorId: string;
  startedAt: Date;
  lastActivityAt: Date;
  pageCount: number;
  eventCount: number;
  entryUrl: string;
  lastUrl: string;
}

export function isNewSession(lastActivityAt: Date, now: Date): boolean {
  return now.getTime() - lastActivityAt.getTime() > SESSION_TIMEOUT_MS;
}
```

Visitor upsert logic:
```typescript
export async function upsertVisitor(
  db: DrizzleClient,
  projectId: string,
  anonymousId: string,
  traits: Record<string, unknown>,
  consent: Record<string, unknown>,
): Promise<string> {
  const result = await db
    .insert(visitors)
    .values({
      projectId,
      anonymousId,
      traits,
      consent,
    })
    .onConflictDoUpdate({
      target: [visitors.projectId, visitors.anonymousId],
      set: {
        traits: sql`visitor.traits || ${JSON.stringify(traits)}::jsonb`,
        consent: sql`${JSON.stringify(consent)}::jsonb`,
        lastSeenAt: sql`now()`,
        updatedAt: sql`now()`,
      },
    })
    .returning({ id: visitors.id });
  return result[0].id;
}
```

**Testing**:
- `Unit: isNewSession returns true when gap > 30 minutes`
- `Unit: isNewSession returns false when gap < 30 minutes`
- `Integration: upsertVisitor creates new visitor on first call`
- `Integration: upsertVisitor merges traits on second call with same anonymousId`
- `Integration: upsertVisitor updates lastSeenAt on repeat visit`
- `Integration: identify event links userId to existing anonymousId`

---

## Phase 3: Behavioural Analytics — Heatmaps & Session Recording

### Purpose
Deliver the first user-visible analytics features: click/scroll/movement heatmaps and session recording with playback. After this phase, users can see visual representations of how visitors interact with their pages and replay individual sessions. These capabilities feed the AI hypothesis engine in later phases.

### Tasks

#### 3.1 — Heatmap Data Collection & Aggregation

**What**: Extend the browser SDK to collect click positions, scroll depth, and mouse movement, then aggregate into pre-computed heatmap tiles in ClickHouse.

**Design**:

Browser SDK heatmap collector:
```typescript
export class HeatmapCollector {
  private moveBuffer: { x: number; y: number; t: number }[] = [];
  private moveSampleIntervalMs = 100;
  private lastMoveTime = 0;

  constructor(private tracker: CROTracker) {
    document.addEventListener('click', this.handleClick.bind(this));
    document.addEventListener('mousemove', this.handleMouseMove.bind(this));
    window.addEventListener('scroll', this.handleScroll.bind(this), { passive: true });
  }

  private handleClick(e: MouseEvent): void {
    const target = e.target as HTMLElement;
    this.tracker.track('click', {
      x: e.pageX,
      y: e.pageY,
      selector: this.getCSSSelector(target),
      elementText: target.textContent?.slice(0, 100) ?? '',
      viewportWidth: window.innerWidth,
      viewportHeight: window.innerHeight,
    });
  }

  private handleScroll(): void {
    const scrollDepth = (window.scrollY + window.innerHeight) / document.documentElement.scrollHeight * 100;
    this.tracker.track('scroll', {
      scrollDepthPct: Math.min(100, Math.round(scrollDepth * 100) / 100),
    });
  }

  private getCSSSelector(el: HTMLElement): string {
    // Generate a unique CSS selector for the element
    // Uses id > data attributes > nth-child path
  }
}
```

ClickHouse materialized view for heatmap tiles:
```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS cro.mv_heatmap_tiles TO cro.heatmap_tiles AS
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
FROM cro.events
WHERE event_type IN ('click', 'scroll', 'rage_click')
  AND x_position IS NOT NULL AND y_position IS NOT NULL
GROUP BY project_id, url_path, event_type, device_type, period_date, tile_x, tile_y;
```

Heatmap API endpoint:
```typescript
// GET /api/v1/heatmaps?projectId=...&url=/pricing&type=click&device=desktop&from=2026-05-01&to=2026-05-25
export interface HeatmapTileResponse {
  tiles: Array<{
    x: number;
    y: number;
    width: number;
    height: number;
    count: number;
    uniqueVisitors: number;
    intensity: number;  // 0.0-1.0 normalized
  }>;
  metadata: {
    url: string;
    type: string;
    device: string;
    totalInteractions: number;
    period: { from: string; to: string };
  };
}
```

**Testing**:
- `Unit: HeatmapCollector emits click event with correct x, y, selector`
- `Unit: HeatmapCollector throttles mousemove events to 100ms intervals`
- `Unit: scroll depth calculation handles pages shorter than viewport (100%)`
- `Unit: getCSSSelector returns unique selector for nested elements`
- `Integration: click events flow through Kafka → ClickHouse → materialized view populates heatmap_tiles`
- `Integration: GET /api/v1/heatmaps returns normalized tiles with intensity 0-1`
- `Integration: heatmap tiles aggregate correctly across multiple days`
- `Integration: device filter (mobile/desktop/tablet) returns separate tile sets`

#### 3.2 — Heatmap Visualization UI

**What**: React component that renders heatmap tiles as a canvas overlay on a page screenshot, with controls for type selection, date range, and device filter.

**Design**:

Heatmap renderer component:
```typescript
export interface HeatmapOverlayProps {
  projectId: string;
  url: string;
  screenshotUrl?: string;
}

export function HeatmapOverlay({ projectId, url, screenshotUrl }: HeatmapOverlayProps) {
  const [type, setType] = useState<'click' | 'scroll' | 'movement'>('click');
  const [device, setDevice] = useState<'all' | 'desktop' | 'mobile' | 'tablet'>('all');
  const [dateRange, setDateRange] = useState<{ from: Date; to: Date }>({...});
  const { data: tiles } = useSWR(
    `/api/v1/heatmaps?projectId=${projectId}&url=${url}&type=${type}&device=${device}`,
    fetcher,
  );
  // Canvas rendering with gradient colormap (blue → green → yellow → red)
}
```

Canvas rendering logic:
```typescript
export function renderHeatmap(
  ctx: CanvasRenderingContext2D,
  tiles: HeatmapTile[],
  width: number,
  height: number,
): void {
  // 1. Create offscreen canvas for the raw heatmap
  // 2. Draw each tile as a colored rectangle based on intensity
  // 3. Apply Gaussian blur for smooth gradients
  // 4. Map intensity values to color gradient (blue → green → yellow → red)
  // 5. Set opacity based on intensity (0.1 min, 0.8 max)
  // 6. Composite onto main canvas
}
```

**Testing**:
- `Unit: renderHeatmap draws correct number of tiles on canvas`
- `Unit: intensity normalization maps max count to 1.0 and min to 0.0`
- `Unit: color gradient maps 0.0 → blue, 0.5 → green, 1.0 → red`
- `E2E: heatmap page loads screenshot and overlays click data`
- `E2E: switching from click to scroll type updates the overlay`
- `E2E: changing date range re-fetches and re-renders tiles`

#### 3.3 — Session Recording (Browser-Side Capture)

**What**: Integrate rrweb into the browser SDK for DOM snapshot and incremental mutation recording, with consent-gated activation and compression.

**Design**:

Session recorder integration:
```typescript
import { record, type eventWithTime } from 'rrweb';

export class SessionRecorder {
  private events: eventWithTime[] = [];
  private chunkSizeBytes = 256 * 1024; // 256KB chunks
  private stopFn: (() => void) | null = null;

  constructor(
    private transport: EventTransport,
    private sessionId: string,
    private config: { maskAllInputs: boolean; maskTextSelector: string },
  ) {}

  start(): void {
    this.stopFn = record({
      emit: (event) => {
        this.events.push(event);
        if (this.estimateSize() >= this.chunkSizeBytes) {
          this.flushChunk();
        }
      },
      maskAllInputs: this.config.maskAllInputs,
      maskTextSelector: this.config.maskTextSelector ?? '.cro-mask',
      sampling: { mousemove: true, mouseInteraction: true, scroll: true, input: 'last' },
      recordCrossOriginIframes: false,
    });
  }

  stop(): void {
    this.stopFn?.();
    this.flushChunk();
  }

  private flushChunk(): void {
    if (this.events.length === 0) return;
    const chunk = this.events.splice(0);
    const compressed = this.compress(JSON.stringify(chunk));
    this.transport.enqueue({
      eventType: 'recording_chunk',
      sessionId: this.sessionId,
      properties: {
        sequenceNumber: this.chunkCounter++,
        chunkType: this.chunkCounter === 1 ? 'full_snapshot' : 'incremental',
        eventCount: chunk.length,
        data: compressed, // base64-encoded compressed data
      },
    });
  }
}
```

**Testing**:
- `Unit: SessionRecorder starts rrweb recording and receives events`
- `Unit: SessionRecorder flushes chunk when size exceeds 256KB`
- `Unit: SessionRecorder stops recording and flushes remaining events`
- `Unit: maskAllInputs=true → input values are masked in emitted events`
- `Unit: SessionRecorder respects consent — does not start if recording consent not granted`
- `Integration: recording chunks flow through ingestion pipeline to S3 storage`

#### 3.4 — Session Recording Storage & Playback API

**What**: Worker that processes recording chunks from Kafka, compresses and stores them in S3, and indexes metadata in ClickHouse. REST API for listing and streaming recordings.

**Design**:

Recording processor worker:
```typescript
export class RecordingProcessor {
  async processChunk(event: CloudEvent): Promise<void> {
    const { sessionId, projectId, sequenceNumber, data } = event.data;

    // 1. Store compressed chunk in S3
    const key = `recordings/${projectId}/${sessionId}/${sequenceNumber}.rrweb.gz`;
    await s3.putObject({ Bucket: S3_BUCKET, Key: key, Body: data, ContentEncoding: 'gzip' });

    // 2. Update recording metadata in ClickHouse
    await clickhouse.insert({
      table: 'cro.recording_metadata',
      values: [{ recording_id: sessionId, session_id: sessionId, project_id: projectId, ... }],
    });
  }
}
```

Recording API:
```
GET  /api/v1/recordings?projectId=...&hasRageClicks=true&minFrustration=50
GET  /api/v1/recordings/:id                → Recording metadata
GET  /api/v1/recordings/:id/chunks         → Signed S3 URLs for all chunks
```

**Testing**:
- `Integration: recording chunks stored in S3 with correct key structure`
- `Integration: recording metadata queryable by frustration score in ClickHouse`
- `Integration: GET /api/v1/recordings returns recordings sorted by frustration score`
- `Integration: GET /api/v1/recordings/:id/chunks returns signed S3 URLs`
- `E2E: session replay player loads chunks from S3 and plays back DOM changes`

#### 3.5 — Session Replay Player UI

**What**: React component wrapping rrweb-player for session playback with timeline scrubbing, speed controls, and friction event markers.

**Design**:

```typescript
export interface SessionReplayProps {
  recordingId: string;
  projectId: string;
}

export function SessionReplayPlayer({ recordingId, projectId }: SessionReplayProps) {
  const { data: metadata } = useSWR(`/api/v1/recordings/${recordingId}`);
  const { data: chunkUrls } = useSWR(`/api/v1/recordings/${recordingId}/chunks`);
  // 1. Fetch and decompress all chunks
  // 2. Concatenate rrweb events in sequence
  // 3. Render rrweb-player with events
  // 4. Overlay friction markers on timeline (rage clicks, dead clicks, JS errors)
}
```

Player controls:
```typescript
export interface PlayerControls {
  play(): void;
  pause(): void;
  seek(offsetMs: number): void;
  setSpeed(multiplier: 1 | 2 | 4 | 8): void;
  skipToFriction(frictionIndex: number): void;
}
```

**Testing**:
- `E2E: replay player renders recorded DOM snapshot correctly`
- `E2E: play/pause controls work`
- `E2E: speed selector changes playback speed (1x, 2x, 4x, 8x)`
- `E2E: friction markers appear on timeline at correct positions`
- `E2E: clicking a friction marker jumps playback to that moment`

---

## Phase 4: Experimentation Engine

### Purpose
Build the core A/B testing and multivariate testing engine, including experiment CRUD, variant assignment (deterministic bucketing per OpenFeature), goal tracking, and statistical significance calculation. After this phase, users can create experiments, assign visitors to variants, track conversions, and see real-time statistical results.

### Tasks

#### 4.1 — Experiment & Variant Management

**What**: PostgreSQL schema for experiments, variants, goals, and targeting rules, plus CRUD API endpoints.

**Design**:

Additional Drizzle schema:
```typescript
export const experiments = pgTable('experiment', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  description: text('description'),
  hypothesisId: uuid('hypothesis_id'),
  type: varchar('type', { length: 50 }).notNull(), // ab, multivariate, multi_page, server_side
  status: varchar('status', { length: 50 }).notNull().default('draft'),
  config: jsonb('config').notNull().default({}),
  startedAt: timestamp('started_at', { withTimezone: true }),
  endedAt: timestamp('ended_at', { withTimezone: true }),
  createdBy: uuid('created_by').references(() => appUsers.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_experiment_slug').on(table.projectId, table.slug),
  index('idx_experiment_status').on(table.projectId, table.status),
]);

export const variants = pgTable('variant', {
  id: uuid('id').primaryKey().defaultRandom(),
  experimentId: uuid('experiment_id').notNull().references(() => experiments.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  isControl: boolean('is_control').notNull().default(false),
  trafficWeight: numeric('traffic_weight', { precision: 5, scale: 2 }).notNull().default('0'),
  changes: jsonb('changes').notNull().default({}),
  description: text('description'),
  stats: jsonb('stats').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_variant_slug').on(table.experimentId, table.slug),
]);

export const experimentGoals = pgTable('experiment_goal', {
  id: uuid('id').primaryKey().defaultRandom(),
  experimentId: uuid('experiment_id').notNull().references(() => experiments.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  isPrimary: boolean('is_primary').notNull().default(false),
  config: jsonb('config').notNull(), // { goalType, eventName, selector, urlPattern, revenueProperty }
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Experiment lifecycle state machine:
```
draft → running → paused → running → completed → archived
  │                                       ↑
  └──────────────────────────────────────┘ (direct complete from draft for cancelled)
```

API endpoints:
```
POST   /api/v1/experiments              → Create experiment with variants and goals
GET    /api/v1/experiments              → List experiments (filterable by status)
GET    /api/v1/experiments/:id          → Get experiment with variants, goals, stats
PATCH  /api/v1/experiments/:id          → Update experiment config
POST   /api/v1/experiments/:id/start    → Transition to running
POST   /api/v1/experiments/:id/pause    → Transition to paused
POST   /api/v1/experiments/:id/complete → Transition to completed (declare winner)
DELETE /api/v1/experiments/:id          → Archive experiment
POST   /api/v1/experiments/:id/variants → Add variant
PATCH  /api/v1/experiments/:id/variants/:vid → Update variant changes/weight
POST   /api/v1/experiments/:id/goals    → Add goal
```

**Testing**:
- `Unit: experiment status transitions follow state machine (draft→running valid, completed→draft invalid)`
- `Unit: variant traffic weights must sum to 100 when experiment starts`
- `Unit: exactly one variant must be marked as control`
- `Integration: POST /api/v1/experiments creates experiment + variants + goals atomically`
- `Integration: POST /api/v1/experiments/:id/start rejects if traffic weights don't sum to 100`
- `Integration: PATCH /api/v1/experiments/:id rejects changes to running experiment's type`
- `Integration: DELETE archives (status=archived) rather than deleting rows`

#### 4.2 — Deterministic Visitor Assignment (Bucketing)

**What**: Implement deterministic hash-based bucketing per the OpenFeature specification, ensuring the same visitor always sees the same variant across devices and SDKs.

**Design**:

Bucketing algorithm:
```typescript
import { createHash } from 'crypto';

export const BUCKET_COUNT = 10000; // 0-9999

/**
 * Deterministic bucket assignment.
 * Same (experimentId, visitorId) always produces the same bucket.
 * Compatible with OpenFeature targeting_key evaluation context.
 */
export function computeBucket(experimentId: string, visitorId: string): number {
  const hash = createHash('md5')
    .update(`${experimentId}:${visitorId}`)
    .digest();
  // Use first 4 bytes as unsigned int, mod bucket count
  const value = hash.readUInt32BE(0);
  return value % BUCKET_COUNT;
}

/**
 * Assign a visitor to a variant based on their bucket and variant traffic weights.
 */
export function assignVariant(
  bucket: number,
  variants: Array<{ id: string; slug: string; trafficWeight: number }>,
): { variantId: string; variantSlug: string } {
  // Convert percentages to bucket ranges
  // e.g., control=50%, variant_a=50% → control gets buckets 0-4999, variant_a gets 5000-9999
  let cumulative = 0;
  for (const variant of variants) {
    const bucketRange = Math.round(variant.trafficWeight * BUCKET_COUNT / 100);
    if (bucket < cumulative + bucketRange) {
      return { variantId: variant.id, variantSlug: variant.slug };
    }
    cumulative += bucketRange;
  }
  // Fallback to last variant (handles rounding)
  const last = variants[variants.length - 1];
  return { variantId: last.id, variantSlug: last.slug };
}
```

Assignment storage (PostgreSQL):
```typescript
export const experimentAssignments = pgTable('experiment_assignment', {
  id: uuid('id').primaryKey().defaultRandom(),
  experimentId: uuid('experiment_id').notNull().references(() => experiments.id),
  variantId: uuid('variant_id').notNull().references(() => variants.id),
  visitorId: uuid('visitor_id').notNull().references(() => visitors.id),
  bucketValue: integer('bucket_value').notNull(),
  context: jsonb('context'),    // OpenFeature evaluation context snapshot
  assignedAt: timestamp('assigned_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_assignment_unique').on(table.experimentId, table.visitorId),
]);
```

OpenFeature provider:
```typescript
import { Provider, ResolutionDetails, EvaluationContext } from '@openfeature/server-sdk';

export class CROProvider implements Provider {
  metadata = { name: 'cro-provider' };

  async resolveStringEvaluation(
    flagKey: string,               // experiment slug
    defaultValue: string,          // control variant slug
    context: EvaluationContext,
  ): Promise<ResolutionDetails<string>> {
    const visitorId = context.targetingKey;
    if (!visitorId) return { value: defaultValue, reason: 'DEFAULT' };

    const experiment = await this.getExperiment(flagKey);
    if (!experiment || experiment.status !== 'running') {
      return { value: defaultValue, reason: 'DISABLED' };
    }

    // Check for existing assignment
    const existing = await this.getAssignment(experiment.id, visitorId);
    if (existing) return { value: existing.variantSlug, reason: 'CACHED' };

    // New assignment
    const bucket = computeBucket(experiment.id, visitorId);
    const assignment = assignVariant(bucket, experiment.variants);
    await this.storeAssignment(experiment.id, assignment.variantId, visitorId, bucket, context);

    return { value: assignment.variantSlug, reason: 'TARGETING_MATCH' };
  }
}
```

**Testing**:
- `Unit: computeBucket is deterministic — same inputs always produce same output`
- `Unit: computeBucket distributes uniformly across 10000 buckets (chi-squared test on 100K samples)`
- `Unit: assignVariant with 50/50 split assigns first variant to bucket 0, second to bucket 5000`
- `Unit: assignVariant handles three-way split (33/33/34) correctly`
- `Unit: assignVariant handles rounding — all buckets are covered`
- `Integration: existing assignment returns same variant on re-evaluation`
- `Integration: new visitor gets assigned and stored in experiment_assignment table`
- `Integration: OpenFeature provider returns defaultValue when experiment is paused`
- `Integration: OpenFeature provider returns defaultValue when experiment not found`

#### 4.3 — Goal Tracking & Conversion Events

**What**: Track conversion events against experiment goals, matching goal conditions (click on selector, page view URL, custom event name) and recording conversions in ClickHouse.

**Design**:

Goal matching engine:
```typescript
export interface GoalConfig {
  goalType: 'click' | 'pageview' | 'custom_event' | 'revenue' | 'form_submit';
  selector?: string;
  urlPattern?: string;
  eventName?: string;
  revenueProperty?: string;
}

export function matchGoal(event: CROEvent, goal: GoalConfig): boolean {
  switch (goal.goalType) {
    case 'click':
      return event.eventType === 'click' && matchSelector(event.properties.selector, goal.selector!);
    case 'pageview':
      return event.eventType === 'page_view' && matchUrlPattern(event.urlPath, goal.urlPattern!);
    case 'custom_event':
      return event.eventType === 'custom' && event.eventName === goal.eventName;
    case 'revenue':
      return event.eventType === 'custom' && event.eventName === goal.eventName
        && typeof event.properties[goal.revenueProperty!] === 'number';
    case 'form_submit':
      return event.eventType === 'form_submit' && matchSelector(event.properties.formSelector, goal.selector!);
    default:
      return false;
  }
}

export function matchUrlPattern(path: string, pattern: string): boolean {
  // Convert /product/* to regex /^\/product\/.*$/
  const regex = new RegExp('^' + pattern.replace(/\*/g, '.*') + '$');
  return regex.test(path);
}
```

Conversion tracking worker:
```typescript
export class ConversionTracker {
  async processEvent(event: CloudEvent): Promise<void> {
    // 1. Get visitor's active experiment assignments
    const assignments = await this.getActiveAssignments(event.data.visitorId);

    // 2. For each assignment, check if this event matches any goal
    for (const assignment of assignments) {
      const goals = await this.getGoals(assignment.experimentId);
      for (const goal of goals) {
        if (matchGoal(event.data, goal.config)) {
          // 3. Record conversion event in ClickHouse
          await this.recordConversion({
            projectId: event.data.projectId,
            experimentId: assignment.experimentId,
            variantId: assignment.variantId,
            goalId: goal.id,
            visitorId: event.data.visitorId,
            revenue: event.data.properties?.revenue ?? null,
            occurredAt: event.data.timestamp,
          });
        }
      }
    }
  }
}
```

**Testing**:
- `Unit: matchGoal click type matches exact selector`
- `Unit: matchGoal pageview type matches wildcard URL pattern /product/*`
- `Unit: matchGoal custom_event type matches event name exactly`
- `Unit: matchGoal revenue type extracts revenue from specified property`
- `Unit: matchUrlPattern handles multiple wildcards /a/*/b/*`
- `Integration: conversion event recorded in ClickHouse experiment_metrics table`
- `Integration: conversion increments materialized view counters correctly`
- `Integration: same visitor converting twice for same goal → only one conversion counted (dedup)`

#### 4.4 — Statistical Analysis Engine

**What**: Implement both frequentist (z-test for proportions) and Bayesian (Beta-Binomial) statistical analysis, calculating confidence intervals, p-values, and uplift for each variant vs. control.

**Design**:

```typescript
import jStat from 'jstat';

export interface StatisticalResult {
  variantId: string;
  sampleSize: number;
  conversions: number;
  conversionRate: number;
  revenueTotal: number;
  confidence: number;          // 0.0-1.0
  pValue: number;              // frequentist p-value
  uplift: number;              // relative improvement vs control
  upliftCI: [number, number];  // 95% confidence interval for uplift
  isWinner: boolean;
  bayesianProbability: number; // probability of being best variant
}

/**
 * Frequentist z-test for two proportions.
 */
export function frequentistTest(
  controlConversions: number, controlSize: number,
  variantConversions: number, variantSize: number,
): { zScore: number; pValue: number; confidence: number } {
  const p1 = controlConversions / controlSize;
  const p2 = variantConversions / variantSize;
  const pPooled = (controlConversions + variantConversions) / (controlSize + variantSize);
  const se = Math.sqrt(pPooled * (1 - pPooled) * (1 / controlSize + 1 / variantSize));
  const z = (p2 - p1) / se;
  const pValue = 2 * (1 - jStat.normal.cdf(Math.abs(z), 0, 1));
  return { zScore: z, pValue, confidence: 1 - pValue };
}

/**
 * Bayesian probability that variant B beats variant A using Beta distribution.
 * Uses Monte Carlo simulation with 50,000 samples.
 */
export function bayesianProbabilityOfBeatingControl(
  controlAlpha: number, controlBeta: number,
  variantAlpha: number, variantBeta: number,
  samples: number = 50000,
): number {
  let wins = 0;
  for (let i = 0; i < samples; i++) {
    const controlSample = jStat.beta.sample(controlAlpha, controlBeta);
    const variantSample = jStat.beta.sample(variantAlpha, variantBeta);
    if (variantSample > controlSample) wins++;
  }
  return wins / samples;
}
```

Stats calculator worker:
```typescript
export class StatsCalculator {
  private recalcIntervalMs = 60_000; // Recalculate every minute

  async calculateForExperiment(experimentId: string): Promise<StatisticalResult[]> {
    // 1. Fetch variant metrics from ClickHouse
    const metrics = await this.fetchMetrics(experimentId);
    const control = metrics.find(m => m.isControl)!;
    const results: StatisticalResult[] = [];

    for (const variant of metrics) {
      const freq = frequentistTest(
        control.conversions, control.sampleSize,
        variant.conversions, variant.sampleSize,
      );
      const bayesProb = bayesianProbabilityOfBeatingControl(
        control.conversions + 1, control.sampleSize - control.conversions + 1,
        variant.conversions + 1, variant.sampleSize - variant.conversions + 1,
      );
      results.push({
        variantId: variant.variantId,
        sampleSize: variant.sampleSize,
        conversions: variant.conversions,
        conversionRate: variant.conversions / variant.sampleSize,
        revenueTotal: variant.revenueTotal,
        confidence: freq.confidence,
        pValue: freq.pValue,
        uplift: (variant.conversions / variant.sampleSize - control.conversions / control.sampleSize)
                / (control.conversions / control.sampleSize),
        upliftCI: this.computeUpliftCI(control, variant),
        isWinner: freq.confidence >= 0.95,
        bayesianProbability: bayesProb,
      });
    }

    // 2. Write results back to PostgreSQL variant.stats JSONB
    await this.updateVariantStats(results);
    return results;
  }
}
```

**Testing**:
- `Unit: frequentistTest with equal proportions → pValue ≈ 1.0, confidence ≈ 0.0`
- `Unit: frequentistTest with 5% vs 7% at n=10000 → confidence > 0.95`
- `Unit: frequentistTest with 5% vs 5.1% at n=100 → confidence < 0.95`
- `Unit: bayesianProbabilityOfBeatingControl with strong winner → probability > 0.95`
- `Unit: bayesianProbabilityOfBeatingControl with equal performance → probability ≈ 0.5`
- `Unit: uplift calculation (7% vs 5%) → uplift = 0.40 (40%)`
- `Fixture: known statistical datasets produce expected p-values within 0.01 tolerance`
- `Integration: StatsCalculator reads from ClickHouse and updates variant.stats in PostgreSQL`

#### 4.5 — Experiment Results Dashboard UI

**What**: Dashboard page showing experiment status, variant performance table with conversion rates, confidence intervals, statistical significance indicators, and cumulative conversion charts.

**Design**:

```typescript
export interface ExperimentResultsProps {
  experimentId: string;
}

export function ExperimentResults({ experimentId }: ExperimentResultsProps) {
  // Renders:
  // 1. Experiment header (name, status, duration, sample size)
  // 2. Variant comparison table with columns:
  //    - Variant name, sample size, conversions, rate, uplift vs control,
  //      confidence, Bayesian probability, status badge (winner/loser/inconclusive)
  // 3. Cumulative conversion rate chart (line chart over time)
  // 4. Sample ratio mismatch indicator
  // 5. Action buttons (pause, complete, extend)
}
```

**Testing**:
- `E2E: experiment results page loads and shows variant table`
- `E2E: winner variant highlighted with green badge when confidence >= 95%`
- `E2E: inconclusive variants shown with gray badge`
- `E2E: cumulative chart renders with correct data points`
- `E2E: "Declare Winner" button enabled only when confidence >= 95%`

---

## Phase 5: Conversion Funnels & Form Analytics

### Purpose
Add multi-step conversion funnel tracking and field-level form analytics. After this phase, users can define funnels, see where visitors drop off, and identify problematic form fields causing abandonment. These are key inputs for the AI hypothesis engine.

### Tasks

#### 5.1 — Funnel Definition & Progress Tracking

**What**: API for defining multi-step funnels and a ClickHouse-based engine that computes step-by-step drop-off rates from the event stream.

**Design**:

Funnel Drizzle schema:
```typescript
export const funnels = pgTable('funnel', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  steps: jsonb('steps').notNull(),
  // steps: [
  //   { step: 1, name: "Landing", type: "page_view", urlPattern: "/landing*" },
  //   { step: 2, name: "Product", type: "page_view", urlPattern: "/product/*" },
  //   { step: 3, name: "Add to Cart", type: "custom_event", eventName: "Product Added" },
  //   { step: 4, name: "Purchase", type: "custom_event", eventName: "Order Completed" }
  // ]
  stats: jsonb('stats').notNull().default({}),
  createdBy: uuid('created_by').references(() => appUsers.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Funnel computation query (ClickHouse):
```sql
WITH funnel_data AS (
    SELECT
        visitor_id,
        maxIf(1, event_type = 'page_view' AND match(url_path, {step1_pattern:String})) AS step1,
        maxIf(1, event_type = {step2_type:String} AND (event_name = {step2_event:String} OR match(url_path, {step2_pattern:String}))) AS step2,
        maxIf(1, event_type = {step3_type:String} AND (event_name = {step3_event:String} OR match(url_path, {step3_pattern:String}))) AS step3,
        maxIf(1, event_type = {step4_type:String} AND (event_name = {step4_event:String} OR match(url_path, {step4_pattern:String}))) AS step4
    FROM cro.events
    WHERE project_id = {project_id:UUID}
      AND occurred_at >= {from_date:DateTime64(3)}
      AND occurred_at <= {to_date:DateTime64(3)}
    GROUP BY visitor_id
)
SELECT
    countIf(step1 = 1) AS step1_count,
    countIf(step1 = 1 AND step2 = 1) AS step2_count,
    countIf(step1 = 1 AND step2 = 1 AND step3 = 1) AS step3_count,
    countIf(step1 = 1 AND step2 = 1 AND step3 = 1 AND step4 = 1) AS step4_count
FROM funnel_data;
```

API endpoints:
```
POST   /api/v1/funnels                → Create funnel with steps
GET    /api/v1/funnels                → List funnels for project
GET    /api/v1/funnels/:id            → Get funnel with computed stats
GET    /api/v1/funnels/:id/breakdown  → Segmented funnel breakdown (by device, country, UTM)
PATCH  /api/v1/funnels/:id            → Update funnel steps
DELETE /api/v1/funnels/:id            → Delete funnel
```

**Testing**:
- `Unit: funnel query builder generates correct ClickHouse SQL for N steps`
- `Unit: funnel supports mixed step types (page_view + custom_event)`
- `Integration: 4-step funnel with known test data → correct drop-off counts`
- `Integration: funnel breakdown by device_type returns separate counts per device`
- `Integration: empty funnel (no matching events) → all steps at 0`
- `E2E: funnel visualization renders bar chart with drop-off percentages between steps`

#### 5.2 — Form Analytics

**What**: Extend the browser SDK with form field tracking (focus, blur, hesitation, corrections), aggregate in ClickHouse, and display per-field analytics.

**Design**:

Browser SDK form tracker:
```typescript
export class FormTracker {
  private trackedForms = new Map<HTMLFormElement, FormState>();

  observe(): void {
    // Use MutationObserver to detect dynamically added forms
    const observer = new MutationObserver((mutations) => {
      for (const mutation of mutations) {
        for (const node of mutation.addedNodes) {
          if (node instanceof HTMLFormElement) this.trackForm(node);
          if (node instanceof HTMLElement) {
            node.querySelectorAll('form').forEach(f => this.trackForm(f));
          }
        }
      }
    });
    observer.observe(document.body, { childList: true, subtree: true });
    document.querySelectorAll('form').forEach(f => this.trackForm(f));
  }

  private trackForm(form: HTMLFormElement): void {
    const fields = form.querySelectorAll('input, select, textarea');
    fields.forEach(field => {
      field.addEventListener('focus', () => this.onFieldFocus(form, field as HTMLInputElement));
      field.addEventListener('blur', () => this.onFieldBlur(form, field as HTMLInputElement));
    });
    form.addEventListener('submit', () => this.onFormSubmit(form));
  }

  private onFieldFocus(form: HTMLFormElement, field: HTMLInputElement): void {
    // Record focus timestamp and field identity
  }

  private onFieldBlur(form: HTMLFormElement, field: HTMLInputElement): void {
    // Calculate: time_in_field_ms, hesitation_ms (time before first keystroke), corrections count
    // Emit form_blur event with these metrics
  }
}
```

ClickHouse form field stats table (already created in Phase 1):
```sql
-- Aggregated by SummingMergeTree
-- Query: average hesitation, drop-off rate, completion rate per field
SELECT
    field_selector,
    field_name,
    sum(focus_count) AS total_focus,
    sum(completion_count) AS total_complete,
    sum(abandon_count) AS total_abandon,
    sum(total_time_ms) / sum(focus_count) AS avg_time_ms,
    sum(total_hesitation_ms) / sum(focus_count) AS avg_hesitation_ms,
    sum(total_corrections) / sum(focus_count) AS avg_corrections,
    1.0 - (sum(completion_count) / sum(focus_count)) AS drop_off_rate
FROM cro.form_field_stats
WHERE project_id = {project_id:UUID}
  AND page_url_path = {url:String}
  AND form_selector = {form:String}
GROUP BY field_selector, field_name
ORDER BY drop_off_rate DESC;
```

**Testing**:
- `Unit: FormTracker detects dynamically added forms via MutationObserver`
- `Unit: hesitation_ms correctly measures time between focus and first input event`
- `Unit: corrections count tracks backspace/delete key events`
- `Integration: form field events aggregate correctly in ClickHouse form_field_stats`
- `Integration: GET /api/v1/forms/:projectId returns forms with per-field drop-off rates`
- `E2E: form analytics dashboard shows field-level bar chart with drop-off percentages`

#### 5.3 — Funnel & Form Analytics Dashboard UI

**What**: Dashboard pages for funnel visualization (horizontal bar chart with drop-off arrows) and form analytics (field-by-field table with heatmap coloring for problem fields).

**Design**:

Funnel visualization component:
```typescript
export interface FunnelChartProps {
  funnel: {
    name: string;
    steps: Array<{
      name: string;
      entered: number;
      completed: number;
      dropOffRate: number;
    }>;
  };
}

// Renders horizontal bars decreasing in width, with drop-off % labels between steps
// Color: green (>80% completion) → yellow (50-80%) → red (<50%)
```

Form analytics component:
```typescript
export interface FormAnalyticsProps {
  fields: Array<{
    fieldName: string;
    fieldSelector: string;
    focusCount: number;
    completionRate: number;
    avgTimeMs: number;
    avgHesitationMs: number;
    avgCorrections: number;
    dropOffRate: number;
  }>;
}

// Renders table with cells color-coded by severity
// High hesitation (>3s) → yellow; High drop-off (>20%) → red; High corrections (>3) → orange
```

**Testing**:
- `E2E: funnel chart renders correct number of bars matching step count`
- `E2E: funnel chart shows correct drop-off percentages between steps`
- `E2E: form analytics table sorts by drop-off rate descending`
- `E2E: problem fields highlighted in red when drop-off > 20%`
- `E2E: clicking a form field row opens session recordings filtered to that form`

---

## Phase 6: Friction Detection & Visitor Segmentation

### Purpose
Implement automated friction signal detection (rage clicks, dead clicks, u-turns, speed browsing) in the browser SDK and build the segment rule engine for visitor targeting. After this phase, the platform can automatically flag frustrating experiences and users can define audience segments for experiments and analysis.

### Tasks

#### 6.1 — Friction Signal Detection (Browser SDK)

**What**: Extend the SDK to detect rage clicks, dead clicks, thrashed cursor, u-turns, and speed browsing, emitting typed friction events.

**Design**:

```typescript
export class FrictionDetector {
  private clickBuffer: { x: number; y: number; t: number }[] = [];
  private rageClickThreshold = 3;       // 3+ rapid clicks in same area
  private rageClickWindowMs = 1000;     // within 1 second
  private rageClickRadiusPx = 30;       // within 30px

  detectRageClick(click: { x: number; y: number; t: number }): boolean {
    this.clickBuffer.push(click);
    // Remove clicks outside time window
    const cutoff = click.t - this.rageClickWindowMs;
    this.clickBuffer = this.clickBuffer.filter(c => c.t >= cutoff);
    // Count clicks within radius
    const nearby = this.clickBuffer.filter(c =>
      Math.hypot(c.x - click.x, c.y - click.y) <= this.rageClickRadiusPx
    );
    return nearby.length >= this.rageClickThreshold;
  }

  detectDeadClick(click: { selector: string; target: HTMLElement }): boolean {
    // A click on a non-interactive element (no href, no onclick, not a button/input/select)
    const tag = click.target.tagName.toLowerCase();
    const interactive = ['a', 'button', 'input', 'select', 'textarea', 'label'];
    const hasClickHandler = click.target.onclick !== null;
    const hasRole = click.target.getAttribute('role') === 'button';
    return !interactive.includes(tag) && !hasClickHandler && !hasRole
      && !click.target.closest('a, button');
  }
}
```

Friction event types:
```typescript
export type FrictionType =
  | 'rage_click'      // 3+ rapid clicks in same area
  | 'dead_click'      // click on non-interactive element
  | 'thrashed_cursor' // rapid mouse movement with no click
  | 'u_turn'          // navigated forward then immediately back
  | 'speed_browse'    // < 3 seconds on a page before navigating away
  | 'form_abandon'    // started filling form, left without submitting
  | 'js_error';       // JavaScript error caught by window.onerror
```

**Testing**:
- `Unit: detectRageClick returns true for 3 clicks within 30px in 1 second`
- `Unit: detectRageClick returns false for 2 clicks (below threshold)`
- `Unit: detectRageClick returns false for 3 clicks spread across 50px (outside radius)`
- `Unit: detectDeadClick returns true for click on <div> with no handlers`
- `Unit: detectDeadClick returns false for click on <button>`
- `Unit: detectDeadClick returns false for click on element inside <a>`
- `Unit: u_turn detected when pageB duration < 5 seconds and user returns to pageA`
- `Unit: speed_browse detected when page duration < 3 seconds`
- `Integration: friction events flow through pipeline and increment friction counters in session summaries`

#### 6.2 — Visitor Segmentation Engine

**What**: Build the segment rule engine that evaluates visitor traits, behavioral data, and consent status against configurable AND/OR rule groups.

**Design**:

```typescript
export interface SegmentRule {
  group: number;        // Rules in same group are AND'd; groups are OR'd
  attribute: string;    // e.g., "traits.country", "session.pageCount", "consent.analytics"
  operator: SegmentOperator;
  value: string | number | boolean | string[];
}

export type SegmentOperator =
  | 'equals' | 'not_equals'
  | 'contains' | 'not_contains'
  | 'gt' | 'gte' | 'lt' | 'lte'
  | 'in' | 'not_in'
  | 'matches_regex'
  | 'exists' | 'not_exists';

export function evaluateSegment(
  rules: SegmentRule[],
  context: Record<string, unknown>,
): boolean {
  // Group rules by group number
  const groups = new Map<number, SegmentRule[]>();
  for (const rule of rules) {
    const group = groups.get(rule.group) ?? [];
    group.push(rule);
    groups.set(rule.group, group);
  }

  // OR across groups, AND within each group
  for (const [, groupRules] of groups) {
    const allMatch = groupRules.every(rule => evaluateRule(rule, context));
    if (allMatch) return true; // One group passing is enough (OR)
  }
  return false;
}

function evaluateRule(rule: SegmentRule, context: Record<string, unknown>): boolean {
  const value = getNestedValue(context, rule.attribute);
  switch (rule.operator) {
    case 'equals': return value === rule.value;
    case 'gt': return typeof value === 'number' && value > (rule.value as number);
    case 'in': return (rule.value as string[]).includes(value as string);
    case 'matches_regex': return new RegExp(rule.value as string).test(value as string);
    case 'exists': return value !== undefined && value !== null;
    // ... other operators
  }
}
```

Segment Drizzle schema and API:
```typescript
export const segments = pgTable('segment', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  segmentType: varchar('segment_type', { length: 50 }).notNull(),
  rules: jsonb('rules').notNull().default([]),
  visitorCount: integer('visitor_count').notNull().default(0),
  isDynamic: boolean('is_dynamic').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

API:
```
POST   /api/v1/segments              → Create segment with rules
GET    /api/v1/segments              → List segments with visitor counts
GET    /api/v1/segments/:id          → Get segment with rules
GET    /api/v1/segments/:id/visitors → List visitors in segment (paginated)
PATCH  /api/v1/segments/:id          → Update rules
DELETE /api/v1/segments/:id          → Delete segment
POST   /api/v1/segments/evaluate     → Evaluate a visitor against all active segments
```

**Testing**:
- `Unit: evaluateSegment with single rule group — all rules AND'd`
- `Unit: evaluateSegment with two groups — OR logic between groups`
- `Unit: evaluateRule 'equals' string comparison`
- `Unit: evaluateRule 'gt' numeric comparison`
- `Unit: evaluateRule 'in' array membership`
- `Unit: evaluateRule 'matches_regex' pattern matching`
- `Unit: evaluateRule 'exists' on undefined → false`
- `Unit: getNestedValue extracts 'traits.custom.industry' from nested object`
- `Integration: POST /api/v1/segments creates segment and triggers initial visitor count`
- `Integration: GET /api/v1/segments/:id/visitors returns visitors matching rules via ClickHouse query`

---

## Phase 7: AI-Powered Hypothesis Generation & Insights

### Purpose
Implement the AI-native differentiator: automated hypothesis generation from heatmap/session/funnel data, natural language querying of analytics data, and proactive insight surfacing. After this phase, the platform automatically suggests experiments based on observed visitor behaviour, replacing manual PIE/ICE scoring.

### Tasks

#### 7.1 — Hypothesis Management System

**What**: PostgreSQL schema for hypotheses with LIFT model and PIE/ICE scoring, lifecycle management, and experiment linking.

**Design**:

```typescript
export const hypotheses = pgTable('hypothesis', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id),
  title: varchar('title', { length: 500 }).notNull(),
  description: text('description'),
  source: varchar('source', { length: 50 }).notNull(), // manual, ai_generated, heatmap_insight, session_insight
  scoring: jsonb('scoring').notNull().default({}),
  // scoring: {
  //   lift: { valueProposition: 7, relevance: 8, clarity: 6, anxiety: 4, distraction: 3, urgency: 5 },
  //   pie: { potential: 8, importance: 7, ease: 6 },
  //   priorityScore: 7.0,
  //   aiConfidence: 0.82,
  //   predictedUplift: 0.045,
  //   modelVersion: "hypothesis-scorer-v1"
  // }
  status: varchar('status', { length: 50 }).notNull().default('proposed'),
  createdBy: uuid('created_by').references(() => appUsers.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Hypothesis lifecycle: `proposed → approved → testing → validated | rejected`

API endpoints:
```
POST   /api/v1/hypotheses                    → Create hypothesis (manual or AI)
GET    /api/v1/hypotheses                    → List hypotheses (sortable by priority_score)
GET    /api/v1/hypotheses/:id                → Get hypothesis with linked experiment
PATCH  /api/v1/hypotheses/:id                → Update scoring/status
POST   /api/v1/hypotheses/:id/approve        → Approve for testing
POST   /api/v1/hypotheses/:id/link/:expId    → Link to experiment
POST   /api/v1/hypotheses/generate           → Trigger AI hypothesis generation
```

**Testing**:
- `Unit: hypothesis status transitions follow lifecycle (proposed→approved valid, rejected→testing invalid)`
- `Integration: POST /api/v1/hypotheses creates hypothesis with default scoring`
- `Integration: GET /api/v1/hypotheses sorts by priorityScore descending`
- `Integration: linking hypothesis to experiment updates hypothesis status to 'testing'`

#### 7.2 — AI Hypothesis Generation Engine

**What**: Background worker that analyses heatmap patterns, session friction data, funnel drop-offs, and form analytics to generate prioritized experiment hypotheses using Claude.

**Design**:

Hypothesis generation prompt:
```typescript
export function buildHypothesisPrompt(context: HypothesisContext): string {
  return `You are a Conversion Rate Optimization expert analysing website behavioural data.

Given the following data, generate 3-5 specific, testable hypotheses for improving conversion rates.
Each hypothesis should follow the format: "If we [change], then [metric] will [improve/decrease] because [reason]."

DATA:
- URL: ${context.url}
- Page heatmap summary: ${JSON.stringify(context.heatmapSummary)}
  (Click concentrations, dead zones, rage-click hotspots)
- Funnel analysis: ${JSON.stringify(context.funnelDropOffs)}
  (Step-by-step drop-off rates)
- Form analytics: ${JSON.stringify(context.formFieldStats)}
  (Per-field hesitation times, drop-off rates, correction counts)
- Session friction summary: ${JSON.stringify(context.frictionSummary)}
  (Top friction signals: rage clicks, dead clicks, u-turns)
- Performance metrics: ${JSON.stringify(context.performanceMetrics)}
  (LCP, FID, CLS averages)

For each hypothesis, provide:
1. title: One-sentence hypothesis statement
2. description: Detailed rationale explaining the insight
3. liftFactors: Score each LIFT model factor (1-10): value_proposition, relevance, clarity, anxiety, distraction, urgency
4. pieScores: Score PIE framework (1-10): potential, importance, ease
5. predictedUplift: Estimated conversion rate improvement (decimal, e.g., 0.05 = 5%)
6. confidence: Your confidence in this prediction (0.0-1.0)
7. suggestedChanges: What to change in the variant (CSS, copy, layout)`;
}
```

Hypothesis generation worker:
```typescript
export class HypothesisGenerator {
  async generateForUrl(projectId: string, url: string): Promise<void> {
    // 1. Gather context from ClickHouse
    const context = await this.gatherContext(projectId, url);

    // 2. Call Claude with structured output
    const response = await anthropic.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 4096,
      tools: [{ name: 'generate_hypotheses', input_schema: HypothesisOutputSchema }],
      messages: [{ role: 'user', content: buildHypothesisPrompt(context) }],
    });

    // 3. Parse and store hypotheses
    const hypotheses = parseToolUseResponse(response);
    for (const h of hypotheses) {
      await db.insert(hypothesesTable).values({
        projectId,
        title: h.title,
        description: h.description,
        source: 'ai_generated',
        scoring: {
          lift: h.liftFactors,
          pie: h.pieScores,
          priorityScore: this.computePriorityScore(h.pieScores),
          aiConfidence: h.confidence,
          predictedUplift: h.predictedUplift,
          modelVersion: 'hypothesis-gen-v1',
        },
        status: 'proposed',
      });
    }

    // 4. Generate AI insight about the analysis
    await db.insert(aiInsightsTable).values({
      projectId,
      insightType: 'hypothesis_suggestion',
      title: `${hypotheses.length} new experiment ideas for ${url}`,
      description: `AI analysis identified ${hypotheses.length} potential improvements...`,
      details: { url, hypothesesGenerated: hypotheses.length },
    });
  }
}
```

**Testing**:
- `Unit (mocked LLM): buildHypothesisPrompt includes all context sections`
- `Unit (mocked LLM): response parsing extracts hypotheses with correct schema`
- `Unit: computePriorityScore averages PIE scores correctly`
- `Integration (mocked LLM): generateForUrl creates hypotheses in PostgreSQL`
- `Integration (mocked LLM): generateForUrl creates AI insight record`
- `Integration: generated hypotheses have source='ai_generated' and status='proposed'`
- `Integration: hypotheses sorted by priorityScore appear in correct order`

#### 7.3 — Natural Language Analytics Query

**What**: API endpoint that accepts natural language questions about analytics data, translates them to ClickHouse queries using Claude, and returns formatted answers.

**Design**:

```typescript
export async function handleNaturalLanguageQuery(
  projectId: string,
  question: string,
): Promise<NLQResponse> {
  // 1. Build context about available data
  const schemaContext = buildClickHouseSchemaContext();

  // 2. Ask Claude to generate a ClickHouse SQL query
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 2048,
    system: `You are a ClickHouse SQL expert. Given a natural language question about website analytics data, generate a ClickHouse SQL query. The database schema is:\n${schemaContext}\n\nThe project_id is '${projectId}'. Always filter by project_id. Return ONLY the SQL query, no explanation.`,
    messages: [{ role: 'user', content: question }],
  });

  // 3. Validate and execute the query (with safety checks)
  const sql = extractSQL(response);
  if (!isReadOnlyQuery(sql)) throw new Error('Only SELECT queries are allowed');
  if (!sql.includes(projectId)) throw new Error('Query must filter by project_id');

  const result = await clickhouse.query({ query: sql, format: 'JSON' });

  // 4. Ask Claude to summarize the result in natural language
  const summary = await generateNLSummary(question, result);

  return { question, sql, data: result, summary };
}
```

API endpoint:
```
POST /api/v1/query
Body: { "question": "What pages had the most rage clicks last week?" }
Response: {
  "question": "...",
  "sql": "SELECT url_path, count() ... FROM cro.events WHERE ...",
  "data": [...],
  "summary": "The checkout page (/checkout) had 47 rage clicks last week, followed by..."
}
```

**Testing**:
- `Unit (mocked LLM): question about rage clicks → generates query filtering event_type='rage_click'`
- `Unit: isReadOnlyQuery rejects INSERT, UPDATE, DELETE, DROP, ALTER`
- `Unit: isReadOnlyQuery accepts SELECT with subqueries`
- `Unit: query without project_id filter → rejected`
- `Integration (mocked LLM): full NLQ flow returns data and summary`
- `Integration: malicious SQL injection attempt → rejected by safety checks`

#### 7.4 — Proactive AI Insights Feed

**What**: Background worker that periodically analyses project data and surfaces actionable insights (friction patterns, optimization opportunities, anomalies) into an insights feed.

**Design**:

```typescript
export const aiInsights = pgTable('ai_insight', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id),
  insightType: varchar('insight_type', { length: 50 }).notNull(),
  title: varchar('title', { length: 500 }).notNull(),
  description: text('description').notNull(),
  severity: varchar('severity', { length: 20 }),
  status: varchar('status', { length: 50 }).notNull().default('new'),
  details: jsonb('details').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export type InsightType =
  | 'friction_pattern'        // Repeated friction on same element/page
  | 'hypothesis_suggestion'   // AI-generated experiment idea
  | 'anomaly'                 // Statistical anomaly in experiment
  | 'opportunity'             // Revenue impact opportunity
  | 'performance_regression'  // Core Web Vitals degradation
  | 'funnel_bottleneck';      // Significant funnel drop-off detected
```

Insight generation patterns:
```typescript
export class InsightGenerator {
  async generateInsights(projectId: string): Promise<void> {
    await Promise.all([
      this.detectFrictionPatterns(projectId),
      this.detectFunnelBottlenecks(projectId),
      this.detectPerformanceRegressions(projectId),
      this.detectRevenueOpportunities(projectId),
    ]);
  }

  private async detectFrictionPatterns(projectId: string): Promise<void> {
    // Query ClickHouse for pages with friction_score > 70
    // If a page consistently appears, generate insight with estimated revenue impact
  }

  private async detectFunnelBottlenecks(projectId: string): Promise<void> {
    // For each active funnel, find steps with > 40% drop-off
    // Generate insight with specific drop-off data
  }
}
```

**Testing**:
- `Integration: friction pattern detector creates insight when page has >70 friction score for 3+ days`
- `Integration: funnel bottleneck detector creates insight for step with >40% drop-off`
- `Integration: duplicate insights not created if existing 'new' insight exists for same pattern`
- `E2E: insights feed page shows insights sorted by severity (critical → warning → info)`
- `E2E: dismissing an insight updates status to 'dismissed'`
- `E2E: "Create experiment" action from insight pre-fills hypothesis from insight data`

---

## Phase 8: Dynamic Traffic Allocation (Thompson Sampling)

### Purpose
Implement multi-armed bandit traffic allocation using Thompson Sampling, allowing experiments to dynamically shift traffic toward winning variants in real time. This shortens test durations by an estimated 30-50% compared to fixed splits.

### Tasks

#### 8.1 — Thompson Sampling Engine

**What**: Implement the Thompson Sampling algorithm using Beta distribution posterior updates, with configurable exploration parameters and allocation boundaries.

**Design**:

```typescript
import jStat from 'jstat';

export interface BanditState {
  variantId: string;
  alpha: number;    // successes + prior (Beta distribution parameter)
  beta: number;     // failures + prior
  allocatedWeight: number;
}

export interface BanditConfig {
  explorationWeight: number;      // 0.0-1.0, default 0.1 (10% exploration)
  minAllocation: number;          // minimum traffic % per variant, default 5
  updateIntervalMs: number;       // posterior update frequency, default 300000 (5 min)
  burnInSamples: number;          // samples before bandit kicks in, default 100 per variant
}

/**
 * Draw from each variant's Beta posterior and allocate traffic
 * proportional to the probability of being the best.
 */
export function computeBanditAllocations(
  states: BanditState[],
  config: BanditConfig,
  simulationRuns: number = 10000,
): Map<string, number> {
  const winCounts = new Map<string, number>();
  states.forEach(s => winCounts.set(s.variantId, 0));

  for (let i = 0; i < simulationRuns; i++) {
    let bestVariant = '';
    let bestSample = -1;
    for (const state of states) {
      const sample = jStat.beta.sample(state.alpha, state.beta);
      if (sample > bestSample) {
        bestSample = sample;
        bestVariant = state.variantId;
      }
    }
    winCounts.set(bestVariant, (winCounts.get(bestVariant) ?? 0) + 1);
  }

  // Convert win counts to allocations with minimum allocation floor
  const allocations = new Map<string, number>();
  const totalWins = simulationRuns;
  let allocated = 0;

  for (const [variantId, wins] of winCounts) {
    const rawWeight = (wins / totalWins) * 100;
    const clampedWeight = Math.max(config.minAllocation, rawWeight);
    allocations.set(variantId, clampedWeight);
    allocated += clampedWeight;
  }

  // Normalize to sum to 100
  const scale = 100 / allocated;
  for (const [variantId, weight] of allocations) {
    allocations.set(variantId, Math.round(weight * scale * 100) / 100);
  }

  return allocations;
}

/**
 * Update Beta posterior with new conversion data.
 */
export function updatePosterior(
  currentAlpha: number,
  currentBeta: number,
  newConversions: number,
  newNonConversions: number,
): { alpha: number; beta: number } {
  return {
    alpha: currentAlpha + newConversions,
    beta: currentBeta + newNonConversions,
  };
}
```

**Testing**:
- `Unit: computeBanditAllocations with equal alpha/beta → roughly equal allocations`
- `Unit: computeBanditAllocations with dominant variant → >50% allocation to winner`
- `Unit: computeBanditAllocations respects minAllocation floor`
- `Unit: allocations sum to 100`
- `Unit: updatePosterior correctly increments alpha and beta`
- `Unit: burn-in period maintains equal allocation until threshold met`
- `Fixture: known conversion data produces expected allocation ratios (within 5% tolerance)`

#### 8.2 — Bandit Allocation Worker

**What**: Background worker that periodically updates Thompson Sampling posteriors from ClickHouse metrics and reallocates traffic for bandit-mode experiments.

**Design**:

```typescript
export class BanditUpdater {
  async updateExperiment(experimentId: string): Promise<void> {
    const experiment = await getExperiment(experimentId);
    if (experiment.config.allocationMethod !== 'bandit_thompson') return;

    // 1. Fetch current metrics from ClickHouse
    const metrics = await fetchExperimentMetrics(experimentId);

    // 2. Update posteriors
    const updatedStates = metrics.map(m => {
      const newConversions = m.conversions - (m.previousConversions ?? 0);
      const newTrials = m.sampleSize - (m.previousSampleSize ?? 0);
      const posterior = updatePosterior(
        m.currentAlpha, m.currentBeta,
        newConversions, newTrials - newConversions,
      );
      return { variantId: m.variantId, ...posterior, allocatedWeight: 0 };
    });

    // 3. Compute new allocations
    const allocations = computeBanditAllocations(updatedStates, experiment.config.banditConfig);

    // 4. Update variant traffic weights in PostgreSQL
    for (const [variantId, weight] of allocations) {
      await updateVariantWeight(variantId, weight);
    }

    // 5. Update bandit state in variant.stats JSONB
    for (const state of updatedStates) {
      const weight = allocations.get(state.variantId) ?? 0;
      await updateVariantStats(state.variantId, {
        bandit: { alpha: state.alpha, beta: state.beta, allocatedWeight: weight },
      });
    }
  }
}
```

**Testing**:
- `Integration: bandit updater reads metrics from ClickHouse and updates PostgreSQL variant weights`
- `Integration: bandit updater skips non-bandit experiments`
- `Integration: bandit allocations shift toward winning variant over time`
- `Integration: traffic allocation changes reflected in next visitor assignment`
- `Unit: worker handles division by zero (zero conversions) gracefully`

---

## Phase 9: Personalization Engine

### Purpose
Build real-time variant delivery per visitor segment or predicted intent, allowing the platform to serve personalized experiences without requiring manual experiment setup. After this phase, users can create personalization campaigns that deliver targeted content to specific audience segments.

### Tasks

#### 9.1 — Personalization Campaign Management

**What**: CRUD API for personalization campaigns that link segments to content changes, with priority-based conflict resolution.

**Design**:

```typescript
export const personalizationCampaigns = pgTable('personalization_campaign', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id),
  name: varchar('name', { length: 255 }).notNull(),
  segmentId: uuid('segment_id').references(() => segments.id),
  priority: integer('priority').notNull().default(0),
  status: varchar('status', { length: 50 }).notNull().default('draft'),
  changes: jsonb('changes').notNull().default({}),
  // changes: { css: "...", js: "...", htmlPatches: [...], serverConfig: {...} }
  startedAt: timestamp('started_at', { withTimezone: true }),
  endedAt: timestamp('ended_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Personalization resolution:
```typescript
export async function resolvePersonalization(
  projectId: string,
  visitorContext: Record<string, unknown>,
): Promise<PersonalizationResult> {
  // 1. Get all active campaigns for project, sorted by priority DESC
  const campaigns = await getActiveCampaigns(projectId);

  // 2. Evaluate visitor against each campaign's segment
  for (const campaign of campaigns) {
    if (!campaign.segmentId) continue;
    const segment = await getSegment(campaign.segmentId);
    if (evaluateSegment(segment.rules, visitorContext)) {
      return {
        campaignId: campaign.id,
        changes: campaign.changes,
        segmentId: campaign.segmentId,
      };
    }
  }

  return { campaignId: null, changes: {}, segmentId: null };
}
```

API:
```
POST   /api/v1/personalization                   → Create campaign
GET    /api/v1/personalization                   → List campaigns
PATCH  /api/v1/personalization/:id               → Update campaign
POST   /api/v1/personalization/:id/activate      → Start campaign
POST   /api/v1/personalization/:id/deactivate    → Stop campaign
POST   /api/v1/personalization/resolve           → Resolve for a visitor context
```

**Testing**:
- `Unit: resolvePersonalization returns highest-priority matching campaign`
- `Unit: resolvePersonalization returns empty changes when no segment matches`
- `Unit: priority conflict resolution — higher priority campaign wins`
- `Integration: activated campaign applies changes to matching visitors`
- `Integration: deactivated campaign stops being resolved`
- `E2E: campaign builder UI creates campaign with segment and HTML changes`

#### 9.2 — Browser SDK Personalization Client

**What**: Extend the browser SDK to fetch personalization decisions on page load and apply CSS/JS/HTML changes before the page becomes visible.

**Design**:

```typescript
export class PersonalizationClient {
  async apply(): Promise<void> {
    // 1. Call personalization resolve endpoint with visitor context
    const result = await fetch('/api/v1/personalization/resolve', {
      method: 'POST',
      body: JSON.stringify({
        snippetKey: this.config.snippetKey,
        anonymousId: this.identity.anonymousId,
        url: window.location.pathname,
        traits: this.identity.traits,
      }),
    }).then(r => r.json());

    if (!result.campaignId) return;

    // 2. Apply changes
    if (result.changes.css) {
      const style = document.createElement('style');
      style.textContent = result.changes.css;
      document.head.appendChild(style);
    }
    if (result.changes.htmlPatches) {
      for (const patch of result.changes.htmlPatches) {
        const el = document.querySelector(patch.selector);
        if (el) {
          if (patch.action === 'replace') el.outerHTML = patch.content;
          if (patch.action === 'append') el.insertAdjacentHTML('beforeend', patch.content);
        }
      }
    }
    if (result.changes.js) {
      new Function(result.changes.js)();
    }
  }
}
```

**Testing**:
- `E2E: personalization changes applied before first contentful paint (no flicker)`
- `E2E: CSS changes render correctly on matching segment visitor`
- `E2E: HTML patches replace/append content at correct selectors`
- `E2E: non-matching visitor sees original page (no changes applied)`
- `Unit: personalization client handles network errors gracefully (no changes applied)`

---

## Phase 10: Experiment Anomaly Detection & Monitoring

### Purpose
Implement real-time automated monitoring of live experiments for statistical anomalies: sample ratio mismatches, novelty effects, early stopping violations, and traffic drops. After this phase, the platform alerts users to problems that current tools leave to manual analyst vigilance.

### Tasks

#### 10.1 — Anomaly Detection Engine

**What**: Background worker that monitors all running experiments for common statistical anomalies and generates alerts.

**Design**:

```typescript
export type AnomalyType =
  | 'sample_ratio_mismatch'    // Actual traffic split deviates from expected
  | 'novelty_effect'           // Conversion rate decays over time (initial spike)
  | 'early_winner'             // High confidence too early (likely false positive)
  | 'traffic_drop'             // Sudden decrease in experiment traffic
  | 'conversion_spike'         // Unusual conversion spike (bot traffic?)
  | 'statistical_regression';  // Winner reverts to losing after initial win

export interface AnomalyCheck {
  type: AnomalyType;
  check(experiment: ExperimentWithMetrics): AnomalyResult | null;
}

export class SampleRatioMismatchCheck implements AnomalyCheck {
  type = 'sample_ratio_mismatch' as const;

  check(experiment: ExperimentWithMetrics): AnomalyResult | null {
    // Chi-squared goodness-of-fit test
    // Expected: traffic weights. Observed: actual sample sizes.
    const observed = experiment.variants.map(v => v.sampleSize);
    const totalSamples = observed.reduce((a, b) => a + b, 0);
    const expected = experiment.variants.map(v => totalSamples * (v.trafficWeight / 100));

    let chiSquared = 0;
    for (let i = 0; i < observed.length; i++) {
      chiSquared += Math.pow(observed[i] - expected[i], 2) / expected[i];
    }

    const df = observed.length - 1;
    const pValue = 1 - jStat.chisquare.cdf(chiSquared, df);

    if (pValue < 0.01) { // Significant mismatch at 1% level
      return {
        type: this.type,
        severity: 'critical',
        description: `Sample ratio mismatch detected (p=${pValue.toFixed(4)}). Expected traffic split does not match actual.`,
        detectedValue: chiSquared,
        expectedValue: 0,
      };
    }
    return null;
  }
}

export class NoveltyEffectCheck implements AnomalyCheck {
  type = 'novelty_effect' as const;

  check(experiment: ExperimentWithMetrics): AnomalyResult | null {
    // Compare conversion rate in first 20% of experiment duration
    // vs last 20%. If variant uplift decays by >50%, flag novelty effect.
  }
}
```

**Testing**:
- `Unit: SampleRatioMismatchCheck detects 60/40 split on expected 50/50 with n=10000 → anomaly`
- `Unit: SampleRatioMismatchCheck passes 51/49 split on expected 50/50 with n=1000 → no anomaly`
- `Unit: NoveltyEffectCheck detects >50% uplift decay between first and last quintile`
- `Unit: early_winner check flags significance reached before min_sample_size`
- `Integration: anomaly detection worker creates ai_insight records for detected anomalies`
- `Integration: anomaly severity levels map correctly (SRM → critical, novelty → warning)`
- `E2E: experiment results page shows anomaly warning banner when SRM detected`

#### 10.2 — Real-Time Experiment Monitoring Dashboard

**What**: Live dashboard widget showing experiment health indicators: traffic flow, conversion trends, anomaly alerts, and estimated time to significance.

**Design**:

```typescript
export interface ExperimentHealthProps {
  experimentId: string;
}

export function ExperimentHealth({ experimentId }: ExperimentHealthProps) {
  // Auto-refreshing dashboard showing:
  // 1. Traffic flow sparkline (events per minute over last 24 hours)
  // 2. Conversion rate trend per variant (rolling 1-hour windows)
  // 3. Anomaly alert badges with descriptions
  // 4. Estimated time to reach 95% confidence (based on current rates)
  // 5. Sample ratio indicator (actual vs expected traffic split)
}

export function estimateTimeToSignificance(
  currentConversionRate: number,
  minimumDetectableEffect: number,
  currentSampleSize: number,
  dailyTraffic: number,
  targetConfidence: number = 0.95,
): { daysRemaining: number; samplesNeeded: number } {
  // Use power analysis formula
  const z_alpha = jStat.normal.inv(1 - (1 - targetConfidence) / 2, 0, 1);
  const z_beta = jStat.normal.inv(0.8, 0, 1); // 80% power
  const p1 = currentConversionRate;
  const p2 = p1 * (1 + minimumDetectableEffect);
  const samplesPerVariant = Math.ceil(
    Math.pow(z_alpha * Math.sqrt(2 * p1 * (1 - p1)) + z_beta * Math.sqrt(p1 * (1 - p1) + p2 * (1 - p2)), 2)
    / Math.pow(p2 - p1, 2)
  );
  const samplesNeeded = samplesPerVariant * 2;
  const daysRemaining = Math.max(0, Math.ceil((samplesNeeded - currentSampleSize) / dailyTraffic));
  return { daysRemaining, samplesNeeded };
}
```

**Testing**:
- `Unit: estimateTimeToSignificance with 5% rate, 10% MDE, 1000/day → ~15 days`
- `Unit: estimateTimeToSignificance with sufficient current sample → 0 days remaining`
- `E2E: experiment health dashboard shows live traffic sparkline`
- `E2E: anomaly warning appears when SRM check fails`
- `E2E: time-to-significance countdown updates as traffic flows in`

---

## Phase 11: WYSIWYG Experiment Editor & Server-Side SDK

### Purpose
Build the visual editor for creating A/B test variations without code (critical for non-technical users) and the server-side SDK for backend experimentation. After this phase, marketers can design test variations visually, and engineers can run server-side experiments via the OpenFeature-compatible SDK.

### Tasks

#### 11.1 — WYSIWYG Visual Editor

**What**: Browser-based visual editor that loads the target page in an iframe and allows point-and-click editing of elements (text, colors, images, visibility, layout) to create experiment variants.

**Design**:

```typescript
export interface VisualChange {
  selector: string;
  action: 'replace_text' | 'replace_html' | 'change_style' | 'hide' | 'show' | 'move' | 'insert';
  value: string;    // new text, HTML, CSS property, or insertion content
  property?: string; // CSS property name for change_style
}

export interface VisualEditorProps {
  experimentId: string;
  variantId: string;
  targetUrl: string;
}

export function VisualEditor({ experimentId, variantId, targetUrl }: VisualEditorProps) {
  // 1. Load target URL in sandboxed iframe via proxy
  // 2. Inject editor overlay with element selector
  // 3. On element click → show editing toolbar (edit text, change color, hide, etc.)
  // 4. Record changes as VisualChange[] array
  // 5. Save changes to variant.changes JSONB on submit
}
```

Changes are serialized to the variant's `changes.htmlPatches` and `changes.css`:
```typescript
export function serializeVisualChanges(changes: VisualChange[]): VariantChanges {
  const htmlPatches = changes
    .filter(c => c.action !== 'change_style')
    .map(c => ({ selector: c.selector, action: c.action, content: c.value }));

  const cssRules = changes
    .filter(c => c.action === 'change_style')
    .map(c => `${c.selector} { ${c.property}: ${c.value}; }`)
    .join('\n');

  return { htmlPatches, css: cssRules };
}
```

**Testing**:
- `E2E: visual editor loads target page in iframe`
- `E2E: clicking an element highlights it and shows editing toolbar`
- `E2E: editing text → change recorded as replace_text VisualChange`
- `E2E: changing background color → change recorded as change_style VisualChange`
- `E2E: saving changes stores serialized htmlPatches and CSS in variant.changes`
- `E2E: preview mode renders variant with applied changes`
- `Unit: serializeVisualChanges converts VisualChange[] to valid CSS + HTML patches`

#### 11.2 — Server-Side SDK

**What**: TypeScript SDK for server-side experiment assignment and event tracking, with OpenFeature provider interface.

**Design**:

```typescript
// packages/sdk-server/src/client.ts
export interface CROServerConfig {
  apiKey: string;
  apiEndpoint: string;
  pollIntervalMs?: number;  // default 60000 (1 minute)
}

export class CROServerClient {
  private experimentCache: Map<string, Experiment> = new Map();

  constructor(private config: CROServerConfig) {
    this.startPolling();
  }

  /**
   * Get the variant assignment for a visitor in an experiment.
   * Deterministic: same visitorId always gets same variant.
   */
  async getVariant(
    experimentSlug: string,
    visitorId: string,
    context?: Record<string, unknown>,
  ): Promise<{ variant: string; isControl: boolean }> {
    const experiment = this.experimentCache.get(experimentSlug);
    if (!experiment || experiment.status !== 'running') {
      return { variant: 'control', isControl: true };
    }

    const bucket = computeBucket(experiment.id, visitorId);
    const assignment = assignVariant(bucket, experiment.variants);
    return {
      variant: assignment.variantSlug,
      isControl: experiment.variants.find(v => v.slug === assignment.variantSlug)?.isControl ?? true,
    };
  }

  /**
   * Track a server-side event (conversion, custom event).
   */
  async track(
    visitorId: string,
    eventName: string,
    properties?: Record<string, unknown>,
  ): Promise<void> {
    await fetch(`${this.config.apiEndpoint}/api/v1/events`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${this.config.apiKey}` },
      body: JSON.stringify({
        events: [{
          eventType: 'custom',
          eventName,
          anonymousId: visitorId,
          properties,
          timestamp: new Date().toISOString(),
        }],
      }),
    });
  }
}
```

OpenFeature provider:
```typescript
// packages/sdk-server/src/openfeature-provider.ts
import { Provider, ResolutionDetails, EvaluationContext } from '@openfeature/server-sdk';

export class CROOpenFeatureProvider implements Provider {
  metadata = { name: 'cro-openfeature-provider' };

  constructor(private client: CROServerClient) {}

  async resolveStringEvaluation(
    flagKey: string,
    defaultValue: string,
    context: EvaluationContext,
  ): Promise<ResolutionDetails<string>> {
    const visitorId = context.targetingKey;
    if (!visitorId) return { value: defaultValue, reason: 'DEFAULT' };

    const result = await this.client.getVariant(flagKey, visitorId);
    return { value: result.variant, reason: 'TARGETING_MATCH' };
  }
}
```

**Testing**:
- `Unit: CROServerClient.getVariant returns 'control' for non-running experiment`
- `Unit: CROServerClient.getVariant returns deterministic variant for same visitorId`
- `Integration: server SDK polls and caches experiment config`
- `Integration: CROServerClient.track sends event to ingestion API`
- `Integration: OpenFeature provider resolves string flag to variant slug`
- `Integration: OpenFeature provider returns defaultValue when experiment not found`

---

## Phase 12: Privacy, Compliance & Audit Trail

### Purpose
Harden the platform for production privacy requirements: GDPR right-to-erasure, CCPA opt-out, consent-gated data collection, audit logging, and data retention policies. After this phase, the platform meets the privacy-first standard identified as a key market differentiator.

### Tasks

#### 12.1 — Consent Management & Data Gating

**What**: Enforce consent at every data collection point — browser SDK, ingestion API, and worker processors. Implement per-purpose consent with IAB GPP string support.

**Design**:

Consent enforcement at ingestion:
```typescript
export async function enforceConsent(
  projectId: string,
  visitorId: string,
  eventType: string,
): Promise<boolean> {
  const visitor = await getVisitor(projectId, visitorId);
  const consent = visitor?.consent ?? {};

  // Map event types to required consent purposes
  const consentRequirements: Record<string, string> = {
    page_view: 'analytics',
    click: 'analytics',
    scroll: 'analytics',
    recording_chunk: 'sessionRecording',
    personalization_resolve: 'personalization',
  };

  const requiredConsent = consentRequirements[eventType] ?? 'analytics';
  return consent[requiredConsent]?.status === 'granted';
}
```

**Testing**:
- `Unit: enforceConsent blocks recording_chunk when sessionRecording not granted`
- `Unit: enforceConsent allows page_view when analytics granted`
- `Unit: enforceConsent blocks all events when no consent exists and project requires consent`
- `Integration: events without consent are dropped at ingestion, not stored in ClickHouse`

#### 12.2 — GDPR Right-to-Erasure

**What**: Implement visitor data deletion that removes PII from PostgreSQL and anonymizes ClickHouse records while preserving aggregate statistics.

**Design**:

```typescript
export async function deleteVisitorData(
  projectId: string,
  visitorId: string,
): Promise<DeletionResult> {
  // 1. Delete visitor record from PostgreSQL
  await db.delete(visitors).where(eq(visitors.id, visitorId));

  // 2. Delete experiment assignments
  await db.delete(experimentAssignments).where(eq(experimentAssignments.visitorId, visitorId));

  // 3. Anonymize ClickHouse events (replace visitor_id with hash, null out PII)
  await clickhouse.command({
    query: `ALTER TABLE cro.events UPDATE
      visitor_id = toUUID('00000000-0000-0000-0000-000000000000'),
      anonymous_id = 'deleted',
      properties = '{}'
    WHERE project_id = {project_id:UUID} AND visitor_id = {visitor_id:UUID}`,
    query_params: { project_id: projectId, visitor_id: visitorId },
  });

  // 4. Delete session recordings from S3
  await deleteRecordings(projectId, visitorId);

  return { deletedRecords: true, anonymizedEvents: true, deletedRecordings: true };
}
```

API:
```
DELETE /api/v1/visitors/:id → Initiate GDPR deletion
GET    /api/v1/visitors/:id/deletion-status → Check deletion progress
```

**Testing**:
- `Integration: deleteVisitorData removes visitor from PostgreSQL`
- `Integration: deleteVisitorData anonymizes all ClickHouse events for that visitor`
- `Integration: deleteVisitorData deletes S3 recording files`
- `Integration: aggregate statistics (experiment results, heatmap totals) remain accurate after deletion`
- `Integration: deleted visitor's anonymousId cannot be re-associated`

#### 12.3 — Audit Log

**What**: Record all admin actions (experiment create/update/delete, campaign changes, user role changes) in a partitioned audit log table.

**Design**:

```typescript
export const auditLogs = pgTable('audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  userId: uuid('user_id').references(() => appUsers.id),
  action: varchar('action', { length: 100 }).notNull(),
  entityType: varchar('entity_type', { length: 100 }).notNull(),
  entityId: uuid('entity_id').notNull(),
  changes: jsonb('changes'),         // { before: {...}, after: {...} }
  metadata: jsonb('metadata').notNull().default({}), // { ipAddress, userAgent, requestId }
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export async function logAuditEvent(
  organizationId: string,
  userId: string,
  action: string,
  entityType: string,
  entityId: string,
  changes?: { before: unknown; after: unknown },
  metadata?: Record<string, unknown>,
): Promise<void> {
  await db.insert(auditLogs).values({
    organizationId, userId, action, entityType, entityId, changes, metadata,
  });
}
```

**Testing**:
- `Integration: experiment.created action logged with experiment details`
- `Integration: experiment.updated action includes before/after diff`
- `Integration: audit log queryable by entityType and entityId`
- `Integration: audit log includes IP address and user agent from request`
- `E2E: audit log page shows chronological activity feed per organization`

#### 12.4 — Data Retention Policies

**What**: Configurable per-project data retention with automated cleanup of expired ClickHouse partitions, S3 recordings, and PostgreSQL session data.

**Design**:

```typescript
export interface RetentionPolicy {
  eventRetentionDays: number;          // default 395 (13 months)
  recordingRetentionDays: number;      // default 90
  sessionRetentionDays: number;        // default 395
  heatmapRetentionDays: number;        // default 395
}

export class RetentionEnforcer {
  async enforce(projectId: string): Promise<void> {
    const policy = await getRetentionPolicy(projectId);

    // 1. ClickHouse TTL handles event cleanup automatically
    // 2. Delete expired recordings from S3
    await this.deleteExpiredRecordings(projectId, policy.recordingRetentionDays);
    // 3. Clean up ClickHouse recording metadata
    await this.cleanRecordingMetadata(projectId, policy.recordingRetentionDays);
  }
}
```

**Testing**:
- `Integration: retention enforcer deletes S3 recordings older than policy days`
- `Integration: ClickHouse TTL drops events older than 13 months`
- `Integration: retention policy configurable per project`
- `Unit: default retention policy applied when no custom policy set`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Infrastructure         ─── required by everything
    │
Phase 2: Browser SDK & Event Ingestion       ─── requires Phase 1
    │
    ├── Phase 3: Heatmaps & Session Recording ─── requires Phase 2
    │       │
    │       └── Phase 7: AI Hypothesis & Insights ─── requires Phases 3, 5, 6
    │               │
    │               └── Phase 10: Anomaly Detection ─── requires Phases 4, 7
    │
    ├── Phase 4: Experimentation Engine       ─── requires Phase 2
    │       │
    │       ├── Phase 8: Thompson Sampling    ─── requires Phase 4
    │       │
    │       └── Phase 11: WYSIWYG & Server SDK ─── requires Phase 4
    │
    ├── Phase 5: Funnels & Form Analytics     ─── requires Phase 2
    │
    └── Phase 6: Friction & Segmentation      ─── requires Phase 2
            │
            └── Phase 9: Personalization      ─── requires Phase 6

Phase 12: Privacy & Compliance               ─── requires Phases 1-4 (can start after Phase 4)

Parallelism opportunities:
  - Phases 3, 4, 5, 6 can be developed concurrently after Phase 2
  - Phases 8, 10, 11 can be developed concurrently after Phase 4
  - Phase 9 can run in parallel with Phases 7 and 8 after Phase 6
  - Phase 12 can start as early as Phase 4 completion
```

---

## Definition of Done (per phase)

1. All tasks implemented and code compiles without errors.
2. All unit tests pass with >90% branch coverage for new code.
3. All integration tests pass against Docker Compose services.
4. Biome lint and format checks pass with zero warnings.
5. TypeScript strict mode compilation succeeds across all packages.
6. Docker build succeeds for the application image.
7. Database migrations run cleanly on a fresh PostgreSQL + ClickHouse instance.
8. API endpoints documented with Zod schemas (auto-generated OpenAPI where applicable).
9. New environment variables documented in `.env.example`.
10. Feature works end-to-end from browser SDK through to dashboard display.
11. No regression in existing tests (full test suite passes).
12. GDPR consent gating enforced on all new data collection paths (Phase 2+).
