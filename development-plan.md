# Newsletter Platform — Phased Development Plan

> Project: 169-newsletter-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language — Backend | TypeScript (Node.js 22 LTS) | Newsletter platforms are I/O-heavy (SMTP, Stripe webhooks, HTTP APIs) rather than CPU-heavy; TypeScript provides type safety across API, data models, and frontend shared types; the Ghost and Buttondown ecosystems are JS/TS-native |
| Language — Frontend | TypeScript (React 19) | Shared types with backend; React's component model fits the visual email editor, automation workflow builder, and analytics dashboards |
| API Framework | Fastify 5 | Faster than Express for high-volume webhook ingestion; built-in schema validation via JSON Schema (aligns with OpenAPI 3.1 generation); plugin architecture for modular feature loading |
| Frontend Framework | Next.js 15 (App Router) | Server-side rendering for public newsletter pages and landing pages; API routes co-located with the frontend; static generation for published posts; React Server Components for analytics dashboards |
| Database | PostgreSQL 16 | Required by the data model's JSONB usage, GIN indexes, array types, row-level security, and ENUM types; the normalized relational model (Data Model Suggestion 1) with selective JSONB columns (from Suggestion 3) provides the right balance of integrity and flexibility |
| Cache / Queue | Redis 7 (via BullMQ) | Email sending requires a robust job queue with retries, rate limiting, and dead-letter handling; BullMQ is the standard for Node.js; Redis also serves as a cache layer for subscriber counts and campaign stats |
| Email Sending | Resend (primary) / SMTP fallback | Resend provides a modern REST API with DKIM auto-configuration, webhook delivery events, and React Email compatibility; SMTP fallback (Nodemailer) supports self-hosted deployments using any provider |
| Payments | Stripe SDK (@stripe/stripe-node) | Industry standard for newsletter subscriptions; Stripe Checkout handles PCI compliance; Stripe Webhooks drive subscription lifecycle events; Stripe Connect enables multi-publication payouts |
| ORM / Query Builder | Drizzle ORM | Type-safe SQL with full PostgreSQL feature support (JSONB, arrays, enums); generates migrations; lighter than Prisma; direct SQL escape hatch for complex queries |
| Testing | Vitest + Playwright | Vitest for unit/integration tests (compatible with Jest API, faster execution); Playwright for E2E tests of the email editor, automation builder, and subscriber management UI |
| Code Quality | Biome (lint + format) + tsc --noEmit | Biome replaces ESLint + Prettier with a single tool; tsc for type checking |
| Containerisation | Docker + docker-compose | Required for self-hosted deployment; compose orchestrates PostgreSQL, Redis, the API server, the worker, and the Next.js frontend |
| Package Manager | pnpm 9 | Workspace support for monorepo; faster and more disk-efficient than npm |
| Monorepo Structure | pnpm workspaces + Turborepo | Shared types between API and frontend; parallel builds; dependency graph awareness |
| Key Libraries | @react-email/components (email templates), tiptap (rich text editor), zustand (UI state), recharts (analytics charts), zod (runtime validation), jose (JWT), argon2 (password hashing) |

### Project Structure

```
newsletter-platform/
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.web
├── .env.example
├── packages/
│   └── shared/                        # Shared types, constants, validation schemas
│       ├── src/
│       │   ├── types/
│       │   │   ├── publication.ts
│       │   │   ├── subscriber.ts
│       │   │   ├── campaign.ts
│       │   │   ├── post.ts
│       │   │   ├── automation.ts
│       │   │   ├── subscription-tier.ts
│       │   │   └── api.ts             # Request/response types
│       │   ├── validation/
│       │   │   ├── subscriber.ts
│       │   │   ├── campaign.ts
│       │   │   └── publication.ts
│       │   └── constants/
│       │       ├── email.ts           # CAN-SPAM, GDPR constants
│       │       └── stripe.ts
│       ├── package.json
│       └── tsconfig.json
├── apps/
│   ├── api/                           # Fastify API server
│   │   ├── src/
│   │   │   ├── server.ts              # Fastify app bootstrap
│   │   │   ├── config.ts              # Environment-based configuration
│   │   │   ├── db/
│   │   │   │   ├── schema.ts          # Drizzle schema definitions
│   │   │   │   ├── migrations/
│   │   │   │   └── seed.ts
│   │   │   ├── routes/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── publications.ts
│   │   │   │   ├── subscribers.ts
│   │   │   │   ├── posts.ts
│   │   │   │   ├── campaigns.ts
│   │   │   │   ├── automations.ts
│   │   │   │   ├── tiers.ts
│   │   │   │   ├── analytics.ts
│   │   │   │   ├── webhooks/
│   │   │   │   │   ├── stripe.ts
│   │   │   │   │   └── email-events.ts
│   │   │   │   └── api-keys.ts
│   │   │   ├── services/
│   │   │   │   ├── email-sender.ts
│   │   │   │   ├── subscriber-service.ts
│   │   │   │   ├── campaign-service.ts
│   │   │   │   ├── automation-engine.ts
│   │   │   │   ├── stripe-service.ts
│   │   │   │   ├── analytics-service.ts
│   │   │   │   ├── domain-verifier.ts
│   │   │   │   └── ai/
│   │   │   │       ├── writing-copilot.ts
│   │   │   │       ├── subject-line-optimizer.ts
│   │   │   │       └── churn-predictor.ts
│   │   │   ├── workers/
│   │   │   │   ├── campaign-sender.ts
│   │   │   │   ├── automation-processor.ts
│   │   │   │   ├── bounce-handler.ts
│   │   │   │   ├── analytics-aggregator.ts
│   │   │   │   └── engagement-scorer.ts
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── publication-scope.ts
│   │   │   │   └── rate-limit.ts
│   │   │   └── lib/
│   │   │       ├── email-renderer.ts
│   │   │       ├── link-tracker.ts
│   │   │       ├── unsubscribe.ts
│   │   │       └── rss-feed.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── web/                           # Next.js frontend
│       ├── src/
│       │   ├── app/
│       │   │   ├── (auth)/
│       │   │   │   ├── login/
│       │   │   │   └── register/
│       │   │   ├── (dashboard)/
│       │   │   │   ├── layout.tsx
│       │   │   │   ├── publications/
│       │   │   │   ├── subscribers/
│       │   │   │   ├── posts/
│       │   │   │   ├── campaigns/
│       │   │   │   ├── automations/
│       │   │   │   ├── analytics/
│       │   │   │   ├── settings/
│       │   │   │   └── monetisation/
│       │   │   ├── (public)/           # Public newsletter pages
│       │   │   │   ├── [slug]/
│       │   │   │   └── subscribe/
│       │   │   └── api/                # Next.js API routes (BFF)
│       │   ├── components/
│       │   │   ├── editor/             # Tiptap email editor
│       │   │   ├── automation/         # Visual workflow builder
│       │   │   ├── analytics/          # Charts and dashboards
│       │   │   ├── subscribers/        # Subscriber management UI
│       │   │   └── ui/                 # Shared UI primitives
│       │   ├── hooks/
│       │   ├── lib/
│       │   │   └── api-client.ts
│       │   └── styles/
│       ├── public/
│       ├── package.json
│       └── tsconfig.json
├── emails/                            # React Email templates
│   ├── newsletter-default.tsx
│   ├── welcome.tsx
│   ├── double-optin.tsx
│   └── payment-receipt.tsx
└── tests/
    ├── fixtures/
    │   ├── subscribers.json
    │   ├── campaigns.json
    │   └── email-events.json
    └── e2e/
        ├── subscriber-flow.spec.ts
        ├── campaign-send.spec.ts
        └── paid-subscription.spec.ts
```

---

## Phase 1: Project Foundation & Core Schema

### Purpose

Establish the monorepo structure, database schema, configuration system, and development toolchain. After this phase, the project builds, lints, type-checks, has a running PostgreSQL database with all core tables, and has a Fastify server that starts and responds to health checks. Every subsequent phase builds on this foundation.

### Tasks

#### 1.1 — Monorepo Scaffold & Toolchain

**What**: Create the pnpm workspace with `packages/shared`, `apps/api`, and `apps/web` packages, with Turborepo, Biome, and TypeScript configured.

**Design**:

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
  - "apps/*"
  - "emails"
```

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "typecheck": {},
    "test": { "dependsOn": ["^build"] },
    "db:migrate": { "cache": false }
  }
}
```

```json
// biome.json (root)
{
  "$schema": "https://biomejs.dev/schemas/2.0.0/schema.json",
  "organizeImports": { "enabled": true },
  "linter": { "enabled": true, "rules": { "recommended": true } },
  "formatter": { "enabled": true, "indentStyle": "space", "indentWidth": 2 }
}
```

Root `tsconfig.json` base:
```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist"
  }
}
```

**Testing**:
- `Unit: pnpm install completes without errors`
- `Unit: pnpm turbo build succeeds across all packages`
- `Unit: pnpm turbo lint reports zero errors on scaffold code`
- `Unit: pnpm turbo typecheck passes with no type errors`

#### 1.2 — Shared Types & Validation Schemas

**What**: Define the core TypeScript types and Zod validation schemas shared between API and frontend.

**Design**:

```typescript
// packages/shared/src/types/publication.ts
export interface Publication {
  id: string;                          // UUID
  name: string;
  slug: string;
  description: string | null;
  logoUrl: string | null;
  websiteUrl: string | null;
  physicalAddress: string | null;      // CAN-SPAM requirement
  defaultFromName: string | null;
  defaultFromEmail: string | null;
  timezone: string;                    // IANA timezone (e.g., "America/New_York")
  language: string;                    // ISO 639-1
  createdAt: Date;
  updatedAt: Date;
}

// packages/shared/src/types/subscriber.ts
export type SubscriberStatus = 'enabled' | 'disabled' | 'blocklisted';
export type SubscriptionListStatus = 'unconfirmed' | 'confirmed' | 'unsubscribed';

export interface Subscriber {
  id: string;
  publicationId: string;
  email: string;
  name: string | null;
  status: SubscriberStatus;
  countryCode: string | null;         // ISO 3166-1 alpha-2
  source: string | null;              // signup_form, import, api, referral
  referralSource: string | null;
  consentGivenAt: Date | null;        // GDPR Article 7
  consentSource: string | null;
  consentIp: string | null;
  doubleOptinConfirmed: boolean;
  gdprErasureRequestedAt: Date | null;
  subscribedAt: Date;
  unsubscribedAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

// packages/shared/src/types/campaign.ts
export type CampaignStatus = 'draft' | 'scheduled' | 'sending' | 'sent' | 'paused' | 'cancelled';

export interface Campaign {
  id: string;
  publicationId: string;
  postId: string | null;
  templateId: string | null;
  name: string;
  subject: string;
  previewText: string | null;
  fromName: string | null;
  fromEmail: string | null;
  replyTo: string | null;
  htmlBody: string | null;
  plainTextBody: string | null;
  status: CampaignStatus;
  listUnsubscribeUrl: string | null;   // RFC 8058
  listUnsubscribePost: string | null;
  sendAt: Date | null;
  startedAt: Date | null;
  completedAt: Date | null;
  totalRecipients: number;
  totalSent: number;
  totalDelivered: number;
  totalOpened: number;
  totalClicked: number;
  totalBounced: number;
  totalComplained: number;
  totalUnsubscribed: number;
  createdAt: Date;
  updatedAt: Date;
}

// packages/shared/src/types/post.ts
export type PostStatus = 'draft' | 'scheduled' | 'published' | 'archived';
export type PostVisibility = 'public' | 'members' | 'paid';

export interface Post {
  id: string;
  publicationId: string;
  authorId: string | null;
  title: string;
  slug: string;
  htmlContent: string | null;
  markdownContent: string | null;
  plainText: string | null;
  excerpt: string | null;
  featuredImage: string | null;
  status: PostStatus;
  visibility: PostVisibility;
  minPaidTierId: string | null;
  rssGuid: string | null;             // RSS 2.0 feed guid
  publishedAt: Date | null;
  scheduledAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

// packages/shared/src/types/subscription-tier.ts
export interface SubscriptionTier {
  id: string;
  publicationId: string;
  name: string;                        // e.g., "Free", "Premium", "Founding"
  description: string | null;
  priceMonthly: number | null;         // cents (Stripe convention)
  priceYearly: number | null;          // cents
  currency: string;                    // ISO 4217
  stripeProductId: string | null;
  stripePriceMonthlyId: string | null;
  stripePriceYearlyId: string | null;
  isActive: boolean;
  sortOrder: number;
  createdAt: Date;
  updatedAt: Date;
}
```

Zod validation for API inputs:

```typescript
// packages/shared/src/validation/subscriber.ts
import { z } from 'zod';

export const createSubscriberSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().max(255).optional(),
  source: z.string().max(100).optional(),
  referralSource: z.string().max(255).optional(),
  listIds: z.array(z.string().uuid()).optional(),
  tags: z.array(z.string().max(255)).optional(),
  consentSource: z.string().max(255).optional(),
});

export const updateSubscriberSchema = z.object({
  name: z.string().max(255).optional(),
  status: z.enum(['enabled', 'disabled', 'blocklisted']).optional(),
  tags: z.array(z.string().max(255)).optional(),
});

// packages/shared/src/validation/campaign.ts
import { z } from 'zod';

export const createCampaignSchema = z.object({
  name: z.string().max(255),
  subject: z.string().max(500),
  previewText: z.string().max(255).optional(),
  fromName: z.string().max(255).optional(),
  fromEmail: z.string().email().max(255).optional(),
  replyTo: z.string().email().max(255).optional(),
  postId: z.string().uuid().optional(),
  templateId: z.string().uuid().optional(),
  htmlBody: z.string().optional(),
  listIds: z.array(z.string().uuid()).min(1),
});
```

**Testing**:
- `Unit: createSubscriberSchema — valid email and name passes validation`
- `Unit: createSubscriberSchema — invalid email format rejects with path ["email"]`
- `Unit: createSubscriberSchema — email exceeding 255 chars rejects`
- `Unit: createCampaignSchema — missing subject rejects with required error`
- `Unit: createCampaignSchema — empty listIds array rejects with min length error`
- `Unit: all types are importable from @newsletter/shared`

#### 1.3 — Database Schema & Migrations

**What**: Define the Drizzle ORM schema based on Data Model Suggestion 1 (normalized relational) and generate the initial migration.

**Design**:

The schema follows Data Model Suggestion 1's 31-table normalized design with selective JSONB enhancements from Suggestion 3 (automation definitions stored as JSONB to support branching workflows, delivery events as JSONB timeline per row). The full DDL is defined in the data model suggestion files; the Drizzle schema maps those SQL definitions to TypeScript.

```typescript
// apps/api/src/db/schema.ts
import {
  pgTable, uuid, varchar, text, boolean, integer, bigint,
  timestamp, pgEnum, inet, uniqueIndex, index, jsonb
} from 'drizzle-orm/pg-core';

export const subscriberStatusEnum = pgEnum('subscriber_status', ['enabled', 'disabled', 'blocklisted']);
export const subscriptionListStatusEnum = pgEnum('subscription_status', ['unconfirmed', 'confirmed', 'unsubscribed']);
export const postStatusEnum = pgEnum('post_status', ['draft', 'scheduled', 'published', 'archived']);
export const postVisibilityEnum = pgEnum('post_visibility', ['public', 'members', 'paid']);
export const campaignStatusEnum = pgEnum('campaign_status', ['draft', 'scheduled', 'sending', 'sent', 'paused', 'cancelled']);

export const publications = pgTable('publications', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 255 }).notNull().unique(),
  description: text('description'),
  logoUrl: text('logo_url'),
  websiteUrl: text('website_url'),
  physicalAddress: text('physical_address'),
  defaultFromName: varchar('default_from_name', { length: 255 }),
  defaultFromEmail: varchar('default_from_email', { length: 255 }),
  timezone: varchar('timezone', { length: 50 }).default('UTC'),
  language: varchar('language', { length: 10 }).default('en'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  passwordHash: text('password_hash'),
  name: varchar('name', { length: 255 }),
  avatarUrl: text('avatar_url'),
  emailVerified: boolean('email_verified').default(false),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const subscribers = pgTable('subscribers', {
  id: uuid('id').primaryKey().defaultRandom(),
  publicationId: uuid('publication_id').notNull().references(() => publications.id, { onDelete: 'cascade' }),
  email: varchar('email', { length: 255 }).notNull(),
  name: varchar('name', { length: 255 }),
  status: subscriberStatusEnum('status').notNull().default('enabled'),
  countryCode: varchar('country_code', { length: 2 }),
  source: varchar('source', { length: 100 }),
  referralSource: varchar('referral_source', { length: 255 }),
  consentGivenAt: timestamp('consent_given_at', { withTimezone: true }),
  consentSource: varchar('consent_source', { length: 255 }),
  consentIp: inet('consent_ip'),
  doubleOptinConfirmed: boolean('double_optin_confirmed').default(false),
  gdprErasureRequestedAt: timestamp('gdpr_erasure_requested_at', { withTimezone: true }),
  subscribedAt: timestamp('subscribed_at', { withTimezone: true }).notNull().defaultNow(),
  unsubscribedAt: timestamp('unsubscribed_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_subscribers_pub_email').on(table.publicationId, table.email),
  index('idx_subscribers_pub_status').on(table.publicationId, table.status),
  index('idx_subscribers_email').on(table.email),
]);

// ... remaining tables follow the same pattern from Data Model Suggestion 1
// (campaigns, campaign_sends, posts, lists, subscriber_lists, tags,
//  subscriber_tags, sending_domains, templates, subscription_tiers,
//  subscriber_subscriptions, payments, automations, automation_steps,
//  automation_enrollments, referral_programs, referral_milestones,
//  referrals, landing_pages, signup_forms, media, api_keys, bounces,
//  links, link_clicks, publication_members, post_tags, campaign_lists)
```

Configuration:

```typescript
// apps/api/src/config.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3001),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().default('redis://localhost:6379'),
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRY: z.string().default('7d'),
  RESEND_API_KEY: z.string().optional(),
  SMTP_HOST: z.string().optional(),
  SMTP_PORT: z.coerce.number().optional(),
  SMTP_USER: z.string().optional(),
  SMTP_PASS: z.string().optional(),
  STRIPE_SECRET_KEY: z.string().optional(),
  STRIPE_WEBHOOK_SECRET: z.string().optional(),
  CORS_ORIGIN: z.string().default('http://localhost:3000'),
  LOG_LEVEL: z.enum(['trace', 'debug', 'info', 'warn', 'error']).default('info'),
});

export type Config = z.infer<typeof envSchema>;
export const config = envSchema.parse(process.env);
```

**Testing**:
- `Unit: envSchema — valid complete env object passes validation`
- `Unit: envSchema — missing DATABASE_URL rejects with required error`
- `Unit: envSchema — invalid NODE_ENV value rejects with enum error`
- `Integration: drizzle-kit generate produces migration SQL matching expected table count (31 tables)`
- `Integration: drizzle-kit migrate applies to fresh PostgreSQL without errors`
- `Integration: rollback and re-apply migration is idempotent`

#### 1.4 — Fastify Server Bootstrap & Health Check

**What**: Create the Fastify application with plugin registration, CORS, request logging, error handling, and a `/health` endpoint.

**Design**:

```typescript
// apps/api/src/server.ts
import Fastify from 'fastify';
import cors from '@fastify/cors';
import { config } from './config.js';
import { db } from './db/client.js';

export async function buildApp() {
  const app = Fastify({
    logger: {
      level: config.LOG_LEVEL,
      transport: config.NODE_ENV === 'development'
        ? { target: 'pino-pretty' }
        : undefined,
    },
  });

  await app.register(cors, { origin: config.CORS_ORIGIN });

  // Health check
  app.get('/health', async () => ({
    status: 'ok',
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version ?? 'unknown',
  }));

  // Readiness check (includes database connectivity)
  app.get('/ready', async () => {
    await db.execute('SELECT 1');
    return { status: 'ready' };
  });

  return app;
}
```

**Testing**:
- `Unit: GET /health returns 200 with { status: "ok" } and ISO timestamp`
- `Integration: GET /ready returns 200 when database is reachable`
- `Integration: GET /ready returns 503 when database connection fails`
- `Unit: CORS headers present for configured origin`
- `Unit: unknown routes return 404 with JSON error body`

#### 1.5 — Docker Compose Development Environment

**What**: Create Docker Compose configuration for local development with PostgreSQL, Redis, API, and web services.

**Design**:

```yaml
# docker-compose.yml
version: '3.9'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: newsletter
      POSTGRES_USER: newsletter
      POSTGRES_PASSWORD: newsletter_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U newsletter"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    ports:
      - "3001:3001"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  web:
    build:
      context: .
      dockerfile: Dockerfile.web
    ports:
      - "3000:3000"
    env_file: .env
    depends_on:
      - api

volumes:
  pgdata:
```

**Testing**:
- `Integration: docker-compose up --build starts all services without errors`
- `Integration: API health check accessible at localhost:3001/health`
- `Integration: PostgreSQL accepts connections on localhost:5432`
- `Integration: Redis accepts connections on localhost:6379`

---

## Phase 2: Authentication & Publication Management

### Purpose

Implement user authentication (registration, login, JWT sessions, password reset) and multi-tenant publication management (create, update, list publications; manage publication members and roles). After this phase, users can create accounts, create publications, and invite team members. This is the tenant boundary that scopes all subsequent features.

### Tasks

#### 2.1 — User Registration & Login

**What**: Implement user registration with email/password, login with JWT token issuance, and password hashing with argon2.

**Design**:

```typescript
// apps/api/src/routes/auth.ts
// POST /api/auth/register
interface RegisterRequest {
  email: string;
  password: string;         // min 8 chars
  name?: string;
}
interface RegisterResponse {
  user: { id: string; email: string; name: string | null };
  token: string;            // JWT
}

// POST /api/auth/login
interface LoginRequest {
  email: string;
  password: string;
}
interface LoginResponse {
  user: { id: string; email: string; name: string | null };
  token: string;
}

// POST /api/auth/forgot-password
interface ForgotPasswordRequest {
  email: string;
}
// Always returns 200 (prevents email enumeration)

// POST /api/auth/reset-password
interface ResetPasswordRequest {
  token: string;
  password: string;
}
```

JWT payload structure:

```typescript
interface JwtPayload {
  sub: string;              // user ID
  email: string;
  iat: number;
  exp: number;
}
```

Auth middleware:

```typescript
// apps/api/src/middleware/auth.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { jwtVerify } from 'jose';

export async function requireAuth(request: FastifyRequest, reply: FastifyReply): Promise<void> {
  const authHeader = request.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return reply.status(401).send({ error: 'Missing or invalid authorization header' });
  }
  const token = authHeader.slice(7);
  try {
    const { payload } = await jwtVerify(token, secret);
    request.user = { id: payload.sub as string, email: payload.email as string };
  } catch {
    return reply.status(401).send({ error: 'Invalid or expired token' });
  }
}
```

Password handling: argon2id with default parameters (time cost 3, memory cost 65536 KiB, parallelism 4).

**Testing**:
- `Unit: register with valid email + password (8+ chars) creates user, returns JWT`
- `Unit: register with duplicate email returns 409 Conflict`
- `Unit: register with password under 8 chars returns 400 with validation error`
- `Unit: login with correct credentials returns JWT with correct sub claim`
- `Unit: login with wrong password returns 401`
- `Unit: login with non-existent email returns 401 (no email enumeration)`
- `Unit: requireAuth middleware — valid JWT sets request.user`
- `Unit: requireAuth middleware — expired JWT returns 401`
- `Unit: requireAuth middleware — missing Authorization header returns 401`
- `Unit: password hash is argon2id format, not plaintext`

#### 2.2 — Publication CRUD & Multi-Tenancy

**What**: Implement publication creation, update, listing, and deletion with owner-based access control.

**Design**:

```typescript
// apps/api/src/routes/publications.ts
// POST /api/publications
interface CreatePublicationRequest {
  name: string;
  slug?: string;             // auto-generated from name if omitted
  description?: string;
  physicalAddress?: string;  // CAN-SPAM
  timezone?: string;         // default "UTC"
}

// GET /api/publications — returns publications the user is a member of
// GET /api/publications/:id
// PATCH /api/publications/:id
// DELETE /api/publications/:id — owner only
```

Publication-scoping middleware:

```typescript
// apps/api/src/middleware/publication-scope.ts
// Extracts publicationId from route params or query,
// verifies the authenticated user is a member of that publication,
// and attaches the publication and member role to the request.
export async function requirePublicationAccess(
  request: FastifyRequest,
  reply: FastifyReply
): Promise<void> {
  const publicationId = (request.params as any).publicationId;
  const membership = await db.query.publicationMembers.findFirst({
    where: and(
      eq(publicationMembers.publicationId, publicationId),
      eq(publicationMembers.userId, request.user.id)
    ),
  });
  if (!membership) {
    return reply.status(403).send({ error: 'Not a member of this publication' });
  }
  request.publication = { id: publicationId, role: membership.role };
}
```

When a publication is created, the creating user is automatically added as `owner` in `publication_members`.

**Testing**:
- `Unit: POST /api/publications with valid data creates publication and adds user as owner`
- `Unit: POST /api/publications with duplicate slug returns 409`
- `Unit: GET /api/publications returns only publications user is a member of`
- `Unit: PATCH /api/publications/:id by owner succeeds`
- `Unit: PATCH /api/publications/:id by non-member returns 403`
- `Unit: DELETE /api/publications/:id by editor returns 403 (owner-only)`
- `Unit: slug auto-generation from name (e.g., "My Newsletter" -> "my-newsletter")`
- `Unit: slug auto-generation handles special characters and unicode`

#### 2.3 — Publication Members & Roles

**What**: Implement team management: invite members, change roles (owner, admin, editor, author), and remove members.

**Design**:

```typescript
// POST /api/publications/:publicationId/members
interface InviteMemberRequest {
  email: string;
  role: 'admin' | 'editor' | 'author';
}

// PATCH /api/publications/:publicationId/members/:userId
interface UpdateMemberRoleRequest {
  role: 'admin' | 'editor' | 'author';
}

// DELETE /api/publications/:publicationId/members/:userId
```

Role hierarchy: `owner > admin > editor > author`. Only `owner` and `admin` can manage members. An `owner` cannot be removed or demoted.

**Testing**:
- `Unit: owner invites member by email — membership created with specified role`
- `Unit: inviting an existing member returns 409`
- `Unit: admin can invite members, editor cannot (403)`
- `Unit: owner role change on another member succeeds`
- `Unit: owner cannot be removed (400)`
- `Unit: member removal deletes the publication_members row`

---

## Phase 3: Subscriber Management & Email Infrastructure

### Purpose

Implement the subscriber lifecycle (create, import, segment, tag, unsubscribe, GDPR erasure), sending domain verification (SPF/DKIM/DMARC), and the email sending infrastructure (Resend + SMTP fallback). After this phase, publications can manage subscriber lists, verify their sending domain, and send transactional emails (welcome, double opt-in confirmation).

### Tasks

#### 3.1 — Subscriber CRUD & Import

**What**: Implement subscriber creation (single + bulk import via CSV), retrieval with pagination and filtering, update, and deletion.

**Design**:

```typescript
// POST /api/publications/:publicationId/subscribers
// Body: CreateSubscriberRequest (from shared validation schema)
// Returns: Subscriber with 201

// POST /api/publications/:publicationId/subscribers/import
// Body: multipart/form-data with CSV file
// CSV columns: email (required), name, tags (comma-separated)
// Returns: { imported: number, skipped: number, errors: ImportError[] }
interface ImportError {
  row: number;
  email: string;
  reason: string;           // "invalid_email", "duplicate", "blocklisted"
}

// GET /api/publications/:publicationId/subscribers
// Query params: status, tag, listId, search (email/name), page, limit, sortBy, sortDir
// Returns: { data: Subscriber[], total: number, page: number, limit: number }

// GET /api/publications/:publicationId/subscribers/:id
// PATCH /api/publications/:publicationId/subscribers/:id
// DELETE /api/publications/:publicationId/subscribers/:id
```

CSV import uses a streaming parser (csv-parse) to handle large files without loading them entirely into memory. Each row is validated, and duplicates (by publication_id + email) are skipped. Consent fields are auto-populated for imported subscribers: `consentSource = "csv_import"`, `consentGivenAt = now()`.

**Testing**:
- `Unit: create subscriber with valid email succeeds`
- `Unit: create subscriber with duplicate email in same publication returns 409`
- `Unit: same email in different publication succeeds (per-publication scoping)`
- `Unit: CSV import with 3 valid rows returns { imported: 3, skipped: 0 }`
- `Unit: CSV import with 1 invalid email returns { imported: 2, skipped: 1, errors: [...] }`
- `Unit: CSV import with 10,000-row file completes within 10 seconds (streaming)`
- `Unit: GET subscribers with tag filter returns only matching subscribers`
- `Unit: GET subscribers with search "john" matches both email and name fields`
- `Unit: delete subscriber removes row and cascades to subscriber_lists and subscriber_tags`
- `Fixture: tests/fixtures/subscribers-import-valid.csv (100 rows)`
- `Fixture: tests/fixtures/subscribers-import-mixed.csv (valid + invalid rows)`

#### 3.2 — Lists, Tags & Segmentation

**What**: Implement mailing lists, tags (add/remove from subscribers), and rule-based segments.

**Design**:

```typescript
// Lists
// POST /api/publications/:publicationId/lists
interface CreateListRequest { name: string; description?: string; type?: 'public' | 'private'; }
// GET /api/publications/:publicationId/lists
// PATCH /api/publications/:publicationId/lists/:id
// DELETE /api/publications/:publicationId/lists/:id

// Add/remove subscribers to/from lists
// POST /api/publications/:publicationId/lists/:listId/subscribers
interface AddToListRequest { subscriberIds: string[]; }
// DELETE /api/publications/:publicationId/lists/:listId/subscribers/:subscriberId

// Tags
// POST /api/publications/:publicationId/tags
interface CreateTagRequest { name: string; }
// GET /api/publications/:publicationId/tags
// DELETE /api/publications/:publicationId/tags/:id

// Tag subscribers
// POST /api/publications/:publicationId/subscribers/:id/tags
interface TagSubscriberRequest { tagIds: string[]; }
// DELETE /api/publications/:publicationId/subscribers/:id/tags/:tagId

// Segments (dynamic queries — not stored subscriber groups)
// POST /api/publications/:publicationId/segments/preview
interface SegmentQuery {
  conditions: SegmentCondition[];
  operator: 'and' | 'or';
}
interface SegmentCondition {
  field: 'status' | 'tag' | 'list' | 'source' | 'subscribedAfter' | 'subscribedBefore' | 'country';
  operator: 'eq' | 'neq' | 'in' | 'notIn' | 'gt' | 'lt';
  value: string | string[];
}
// Returns: { count: number, sample: Subscriber[] }
```

The segment query engine translates `SegmentQuery` into SQL WHERE clauses with JOINs to `subscriber_tags` and `subscriber_lists` as needed. The `preview` endpoint returns the count and a sample of 10 matching subscribers.

**Testing**:
- `Unit: create list and add 3 subscribers — list.subscriber_count = 3`
- `Unit: remove subscriber from list decrements subscriber_count`
- `Unit: tag subscriber creates subscriber_tags row`
- `Unit: untag subscriber removes subscriber_tags row`
- `Unit: segment preview with status=enabled returns only enabled subscribers`
- `Unit: segment preview with tag=premium AND status=enabled returns intersection`
- `Unit: segment preview with OR operator returns union`
- `Unit: segment preview with subscribedAfter condition filters by date`
- `Unit: delete tag cascades to subscriber_tags`
- `Unit: delete list cascades to subscriber_lists`

#### 3.3 — Double Opt-In & Unsubscribe (GDPR/CAN-SPAM)

**What**: Implement double opt-in confirmation flow, one-click unsubscribe (RFC 8058), web-based unsubscribe, and GDPR data erasure.

**Design**:

Double opt-in flow:
1. Subscriber is created with `doubleOptinConfirmed = false`
2. Confirmation email sent with a signed token (JWT, 24h expiry)
3. `GET /confirm/:token` validates the token and sets `doubleOptinConfirmed = true`

Unsubscribe mechanisms:
1. **One-click (RFC 8058)**: `POST /unsubscribe` with `List-Unsubscribe-Post` header. The campaign email includes `List-Unsubscribe: <url>` and `List-Unsubscribe-Post: List-Unsubscribe=One-Click` headers.
2. **Web-based**: `GET /unsubscribe/:token` shows a confirmation page; `POST /unsubscribe/:token` processes the unsubscribe.
3. **GDPR erasure**: `POST /api/publications/:publicationId/subscribers/:id/erasure` — sets `gdprErasureRequestedAt`, anonymises email (`hash@erased.invalid`), removes name, removes from all lists and tags, deletes engagement data from `campaign_sends` and `link_clicks`.

```typescript
// Unsubscribe token payload
interface UnsubscribeTokenPayload {
  subscriberId: string;
  publicationId: string;
  campaignId?: string;      // for tracking which campaign triggered unsubscribe
}
```

**Testing**:
- `Unit: subscriber creation sends double opt-in email (mocked sender)`
- `Unit: GET /confirm/:validToken sets doubleOptinConfirmed = true`
- `Unit: GET /confirm/:expiredToken returns 400`
- `Unit: POST /unsubscribe (one-click) sets status to disabled, sets unsubscribedAt`
- `Unit: GET /unsubscribe/:token renders confirmation page (200)`
- `Unit: POST /unsubscribe/:token processes unsubscribe`
- `Unit: GDPR erasure anonymises email, clears name, removes tags and lists`
- `Unit: GDPR erasure sets gdprErasureRequestedAt timestamp`
- `Unit: GDPR erasure deletes campaign_sends and link_clicks for subscriber`
- `Integration (mocked): one-click unsubscribe within 10 business days (CAN-SPAM)`

#### 3.4 — Sending Domain Verification

**What**: Implement sending domain registration with SPF/DKIM/DMARC DNS record verification.

**Design**:

```typescript
// POST /api/publications/:publicationId/domains
interface AddDomainRequest { domain: string; }
// Returns required DNS records the user must add:
interface DomainSetupResponse {
  domain: string;
  records: DnsRecord[];
}
interface DnsRecord {
  type: 'TXT' | 'CNAME';
  host: string;              // e.g., "mail._domainkey.example.com"
  value: string;
  verified: boolean;
}

// POST /api/publications/:publicationId/domains/:id/verify
// Checks DNS records and updates verification status
// Returns: { spfVerified: boolean, dkimVerified: boolean, dmarcVerified: boolean }

// GET /api/publications/:publicationId/domains
// DELETE /api/publications/:publicationId/domains/:id
```

DNS verification uses Node.js `dns.resolveTxt()` to check for expected SPF include, DKIM selector, and DMARC policy records. Verification runs on-demand and via a periodic background job (every 6 hours).

**Testing**:
- `Unit: add domain returns required DNS records`
- `Integration (mocked DNS): verify domain with correct SPF record sets spfVerified = true`
- `Integration (mocked DNS): verify domain with missing DKIM record sets dkimVerified = false`
- `Unit: domain uniqueness per publication enforced`
- `Unit: delete domain removes sending_domains row`

#### 3.5 — Email Sending Service

**What**: Implement the email sending abstraction layer supporting Resend (REST API) and SMTP (Nodemailer) backends.

**Design**:

```typescript
// apps/api/src/services/email-sender.ts
interface EmailMessage {
  from: { name: string; email: string };
  to: string;
  subject: string;
  html: string;
  text: string;
  replyTo?: string;
  headers?: Record<string, string>;  // List-Unsubscribe, etc.
  tags?: { name: string; value: string }[];
}

interface SendResult {
  messageId: string;
  provider: 'resend' | 'smtp';
}

interface EmailSender {
  send(message: EmailMessage): Promise<SendResult>;
  sendBatch(messages: EmailMessage[], options?: { rateLimit: number }): Promise<SendResult[]>;
}

// Factory function selects provider based on config
export function createEmailSender(config: Config): EmailSender {
  if (config.RESEND_API_KEY) {
    return new ResendEmailSender(config.RESEND_API_KEY);
  }
  return new SmtpEmailSender({
    host: config.SMTP_HOST!,
    port: config.SMTP_PORT!,
    auth: { user: config.SMTP_USER!, pass: config.SMTP_PASS! },
  });
}
```

Rate limiting: BullMQ job queue with configurable concurrency (default: 10 concurrent sends) and rate limiting (default: 100 sends/second).

**Testing**:
- `Unit: ResendEmailSender sends email via Resend API (mocked HTTP)`
- `Unit: SmtpEmailSender sends email via Nodemailer (mocked SMTP)`
- `Unit: sendBatch respects rate limit (10 messages at 2/second takes ~5 seconds)`
- `Unit: send failure retries up to 3 times with exponential backoff`
- `Unit: List-Unsubscribe headers included when headers param provided`
- `Unit: factory selects Resend when RESEND_API_KEY is set, SMTP otherwise`

---

## Phase 4: Content Publishing & Campaign Sending

### Purpose

Implement the post editor, campaign assembly, and bulk email sending pipeline. After this phase, users can write posts, create campaigns targeting subscriber lists, and send newsletters to thousands of subscribers with link tracking and delivery status monitoring.

### Tasks

#### 4.1 — Post CRUD & Editor API

**What**: Implement post creation, editing, publishing, and scheduling with Markdown and HTML content support.

**Design**:

```typescript
// POST /api/publications/:publicationId/posts
interface CreatePostRequest {
  title: string;
  slug?: string;              // auto-generated from title
  markdownContent?: string;
  htmlContent?: string;
  excerpt?: string;
  featuredImage?: string;
  visibility?: PostVisibility;
  status?: 'draft' | 'scheduled';
  scheduledAt?: string;       // ISO 8601
  tagIds?: string[];
  minPaidTierId?: string;
}

// PATCH /api/publications/:publicationId/posts/:id
// GET /api/publications/:publicationId/posts — paginated, filterable by status
// GET /api/publications/:publicationId/posts/:id
// DELETE /api/publications/:publicationId/posts/:id

// POST /api/publications/:publicationId/posts/:id/publish
// Transitions from draft/scheduled to published, sets publishedAt
```

Markdown to HTML conversion uses `marked` with sanitisation via `DOMPurify` (server-side via jsdom). Plain text extraction for search/AI uses `html-to-text`.

Scheduled posts: a BullMQ repeatable job checks every minute for posts with `status = 'scheduled'` and `scheduledAt <= now()`, transitions them to `published`.

**Testing**:
- `Unit: create post with markdown converts to HTML`
- `Unit: create post auto-generates slug from title`
- `Unit: publish post sets status = published and publishedAt = now()`
- `Unit: scheduled post with past scheduledAt is immediately published by worker`
- `Unit: slug uniqueness per publication enforced`
- `Unit: visibility = paid with minPaidTierId sets correct tier gate`
- `Unit: delete post cascades to post_tags`
- `Fixture: tests/fixtures/sample-markdown-post.md`

#### 4.2 — Campaign Assembly & Sending Pipeline

**What**: Implement campaign creation, recipient resolution (from lists), email rendering with template and tracking pixel injection, and bulk sending via the job queue.

**Design**:

Campaign sending pipeline:

1. **Assemble**: Resolve recipients from `campaign_lists` JOIN `subscriber_lists` JOIN `subscribers` (where status = 'enabled' and doubleOptinConfirmed = true). Deduplicate by email.
2. **Render**: For each recipient, render the HTML body with merge tags (`{{subscriber.name}}`, `{{subscriber.email}}`), inject tracking pixel (`<img src="/track/open/:campaignId/:subscriberId" ...>`), rewrite links for click tracking (replace each `href` with `/track/click/:linkId/:subscriberId`).
3. **Enqueue**: Create `campaign_sends` rows (status = 'queued') and enqueue BullMQ jobs, one per recipient.
4. **Send**: Worker picks up jobs, sends via `EmailSender`, updates `campaign_sends.status` and timestamps.
5. **Complete**: When all jobs finish, update `campaigns.status = 'sent'` and `campaigns.completedAt`.

```typescript
// apps/api/src/services/campaign-service.ts
export class CampaignService {
  async assembleCampaign(campaignId: string): Promise<{
    recipients: { subscriberId: string; email: string; name: string | null }[];
    renderedCount: number;
  }>;

  async startSending(campaignId: string): Promise<void>;

  async pauseSending(campaignId: string): Promise<void>;

  async getCampaignProgress(campaignId: string): Promise<{
    total: number;
    sent: number;
    delivered: number;
    failed: number;
    remaining: number;
  }>;
}

// apps/api/src/lib/email-renderer.ts
export function renderCampaignEmail(
  htmlBody: string,
  subscriber: { id: string; email: string; name: string | null },
  campaign: { id: string; publicationId: string },
  baseUrl: string,
): { html: string; plainText: string; trackingLinks: { originalUrl: string; trackingUrl: string }[] };
```

Email headers per RFC compliance:

```typescript
const emailHeaders = {
  'List-Unsubscribe': `<${baseUrl}/unsubscribe/${token}>`,
  'List-Unsubscribe-Post': 'List-Unsubscribe=One-Click',  // RFC 8058
  'X-Campaign-Id': campaignId,
  'Precedence': 'bulk',
};
```

**Testing**:
- `Unit: assembleCampaign resolves recipients from 2 lists, deduplicates by email`
- `Unit: assembleCampaign excludes disabled, blocklisted, and unconfirmed subscribers`
- `Unit: renderCampaignEmail replaces {{subscriber.name}} with actual name`
- `Unit: renderCampaignEmail injects tracking pixel before </body>`
- `Unit: renderCampaignEmail rewrites 3 links to tracking URLs`
- `Unit: renderCampaignEmail generates plain text fallback from HTML`
- `Integration (mocked sender): startSending enqueues N jobs, sends N emails`
- `Integration (mocked sender): pauseSending stops processing remaining queue`
- `Unit: campaign status transitions: draft -> sending -> sent`
- `Unit: campaign with 0 recipients returns error (cannot send empty campaign)`
- `Unit: List-Unsubscribe and List-Unsubscribe-Post headers present on every email`

#### 4.3 — Link Tracking & Open Tracking

**What**: Implement click tracking via redirect endpoint and open tracking via tracking pixel endpoint.

**Design**:

```typescript
// GET /track/click/:linkId/:subscriberId
// 1. Look up link by linkId
// 2. Insert link_clicks row
// 3. Increment links.click_count
// 4. Update campaign_sends.clicked_at (first click only)
// 5. 302 redirect to link.original_url

// GET /track/open/:campaignId/:subscriberId
// 1. Insert/update campaign_sends.opened_at (first open only)
// 2. Increment campaigns.total_opened
// 3. Return 1x1 transparent PNG
```

The tracking pixel is a 1x1 transparent GIF served with `Cache-Control: no-store` to prevent caching. Note: Apple MPP will trigger false opens; the `user_agent` is recorded on `link_clicks` to enable MPP filtering in analytics.

**Testing**:
- `Unit: GET /track/click/:linkId/:subscriberId redirects 302 to original URL`
- `Unit: click tracking increments link.click_count`
- `Unit: click tracking creates link_clicks row with IP and user_agent`
- `Unit: first click on a campaign sets campaign_sends.clicked_at`
- `Unit: second click does not overwrite campaign_sends.clicked_at`
- `Unit: GET /track/open returns 1x1 transparent GIF with correct Content-Type`
- `Unit: first open sets campaign_sends.opened_at`
- `Unit: tracking pixel has Cache-Control: no-store header`

#### 4.4 — Email Templates & React Email

**What**: Implement email template management and rendering using React Email components.

**Design**:

```typescript
// POST /api/publications/:publicationId/templates
interface CreateTemplateRequest {
  name: string;
  htmlBody: string;
  plainTextBody?: string;
  isDefault?: boolean;
}

// GET /api/publications/:publicationId/templates
// PATCH /api/publications/:publicationId/templates/:id
// DELETE /api/publications/:publicationId/templates/:id
// POST /api/publications/:publicationId/templates/:id/preview
// Body: { subscriberEmail?: string }
// Returns: { html: string, plainText: string }
```

Built-in React Email templates:

```typescript
// emails/newsletter-default.tsx
import { Html, Head, Body, Container, Text, Link, Img, Hr } from '@react-email/components';

interface NewsletterProps {
  publicationName: string;
  content: string;          // HTML content from the post
  unsubscribeUrl: string;
  physicalAddress: string;  // CAN-SPAM
  trackingPixelUrl: string;
}

export const NewsletterDefault = (props: NewsletterProps) => (
  <Html>
    <Head />
    <Body style={{ fontFamily: 'Georgia, serif', maxWidth: 600 }}>
      <Container>
        {/* Content injected here */}
        <Hr />
        <Text style={{ fontSize: 12, color: '#666' }}>
          {props.physicalAddress}
        </Text>
        <Link href={props.unsubscribeUrl} style={{ fontSize: 12 }}>
          Unsubscribe
        </Link>
        <Img src={props.trackingPixelUrl} width="1" height="1" />
      </Container>
    </Body>
  </Html>
);
```

**Testing**:
- `Unit: template CRUD operations`
- `Unit: template preview renders merge tags`
- `Unit: setting isDefault = true on a template clears isDefault on the previous default`
- `Unit: newsletter-default template includes unsubscribe link and physical address`
- `Unit: newsletter-default template includes tracking pixel`

---

## Phase 5: Paid Subscriptions & Monetisation

### Purpose

Implement Stripe integration for paid newsletter subscriptions: subscription tier management, Stripe Checkout for payments, webhook processing for subscription lifecycle events, and subscriber tier tracking. After this phase, publications can offer paid subscriptions with zero platform revenue cut.

### Tasks

#### 5.1 — Subscription Tier Management

**What**: Implement CRUD for subscription tiers with Stripe product/price synchronisation.

**Design**:

```typescript
// POST /api/publications/:publicationId/tiers
interface CreateTierRequest {
  name: string;              // e.g., "Premium"
  description?: string;
  priceMonthly: number;      // cents
  priceYearly?: number;      // cents
  currency?: string;         // default "usd" (ISO 4217)
}

// When a tier is created:
// 1. Create Stripe Product (stripe.products.create)
// 2. Create Stripe Price for monthly (stripe.prices.create)
// 3. Create Stripe Price for yearly if priceYearly provided
// 4. Store stripe_product_id, stripe_price_monthly_id, stripe_price_yearly_id

// GET /api/publications/:publicationId/tiers
// PATCH /api/publications/:publicationId/tiers/:id
// DELETE /api/publications/:publicationId/tiers/:id (soft-deactivate: is_active = false)
```

```typescript
// apps/api/src/services/stripe-service.ts
export class StripeService {
  async createTierProducts(tier: CreateTierRequest, publicationId: string): Promise<{
    stripeProductId: string;
    stripePriceMonthlyId: string;
    stripePriceYearlyId: string | null;
  }>;

  async createCheckoutSession(
    subscriberId: string,
    tierId: string,
    interval: 'month' | 'year',
    successUrl: string,
    cancelUrl: string,
  ): Promise<{ checkoutUrl: string; sessionId: string }>;

  async createCustomerPortalSession(
    stripeCustomerId: string,
    returnUrl: string,
  ): Promise<{ portalUrl: string }>;
}
```

**Testing**:
- `Unit: create tier creates Stripe product and price (mocked Stripe)`
- `Unit: create tier with yearly price creates two Stripe prices`
- `Unit: deactivate tier sets is_active = false, does not delete Stripe product`
- `Unit: list tiers returns only active tiers by default`
- `Integration (mocked Stripe): checkout session creation returns valid URL`

#### 5.2 — Stripe Webhook Processing

**What**: Implement Stripe webhook endpoint for subscription lifecycle events.

**Design**:

```typescript
// POST /api/webhooks/stripe
// Stripe signature verification using stripe.webhooks.constructEvent()

// Handled events:
// checkout.session.completed   -> create subscriber_subscriptions row
// invoice.paid                 -> create payments row, extend period
// invoice.payment_failed       -> update status to 'past_due'
// customer.subscription.updated -> sync status changes
// customer.subscription.deleted -> set status to 'cancelled'
```

Webhook handler:

```typescript
// apps/api/src/routes/webhooks/stripe.ts
export async function handleStripeWebhook(
  event: Stripe.Event,
  db: Database,
): Promise<void> {
  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object as Stripe.Checkout.Session;
      // 1. Find subscriber by session.client_reference_id (subscriberId)
      // 2. Create subscriber_subscriptions row with stripe_customer_id, stripe_subscription_id
      // 3. Update subscriber.tier_id (if using the denormalised reference from Model 3)
      break;
    }
    case 'invoice.paid': {
      const invoice = event.data.object as Stripe.Invoice;
      // 1. Find subscriber_subscriptions by stripe_subscription_id
      // 2. Create payments row (amount, currency, stripe_invoice_id)
      // 3. Update current_period_start and current_period_end
      break;
    }
    case 'customer.subscription.deleted': {
      const subscription = event.data.object as Stripe.Subscription;
      // 1. Find subscriber_subscriptions by stripe_subscription_id
      // 2. Set status = 'cancelled', cancelled_at = now()
      // 3. Clear subscriber.tier_id
      break;
    }
  }
}
```

**Testing**:
- `Unit: valid Stripe signature is accepted, invalid signature returns 400`
- `Unit: checkout.session.completed creates subscriber_subscriptions row`
- `Unit: invoice.paid creates payments row with correct amount`
- `Unit: invoice.payment_failed sets subscription status to past_due`
- `Unit: customer.subscription.deleted sets status to cancelled`
- `Unit: duplicate event processing is idempotent (same invoice.paid twice)`
- `Unit: unknown event type is logged and ignored (200 returned)`

#### 5.3 — Content Gating by Tier

**What**: Implement content visibility enforcement based on subscriber's paid tier.

**Design**:

Posts have `visibility` (public, members, paid) and `minPaidTierId`. When a subscriber requests a post:

```typescript
// apps/api/src/lib/content-gate.ts
export function canAccessPost(
  post: Post,
  subscriber: { tierId: string | null } | null,
  tiers: SubscriptionTier[],       // sorted by sort_order
): { allowed: boolean; gatedAt?: number } {
  if (post.visibility === 'public') return { allowed: true };
  if (!subscriber) return { allowed: false, gatedAt: 0 };
  if (post.visibility === 'members') return { allowed: true };
  // visibility === 'paid'
  if (!subscriber.tierId) return { allowed: false, gatedAt: 0 };
  if (!post.minPaidTierId) return { allowed: true };    // any paid tier
  const subTierOrder = tiers.find(t => t.id === subscriber.tierId)?.sortOrder ?? 0;
  const minTierOrder = tiers.find(t => t.id === post.minPaidTierId)?.sortOrder ?? 0;
  return { allowed: subTierOrder >= minTierOrder };
}
```

Public newsletter pages (`/[slug]/[postSlug]`) render the full post for authorised subscribers and a truncated preview with a subscribe CTA for others.

**Testing**:
- `Unit: public post accessible by anonymous visitor`
- `Unit: members-only post accessible by any subscriber`
- `Unit: members-only post blocked for anonymous visitor`
- `Unit: paid post accessible by subscriber with matching or higher tier`
- `Unit: paid post blocked for free subscriber`
- `Unit: paid post blocked for subscriber with lower tier`

---

## Phase 6: Analytics & Engagement Reporting

### Purpose

Implement the analytics dashboard: campaign performance metrics, subscriber growth tracking, engagement scoring, and revenue reporting. After this phase, publications have a comprehensive analytics view of their newsletter performance, with metrics that go beyond basic open rates to include click rates, growth trends, and revenue.

### Tasks

#### 6.1 — Campaign Analytics

**What**: Implement campaign performance reporting with open rate, click rate, bounce rate, and per-link click statistics.

**Design**:

```typescript
// GET /api/publications/:publicationId/campaigns/:id/analytics
interface CampaignAnalytics {
  campaignId: string;
  name: string;
  subject: string;
  sentAt: string;
  stats: {
    totalRecipients: number;
    totalSent: number;
    totalDelivered: number;
    totalOpened: number;
    totalUniqueOpens: number;
    totalClicked: number;
    totalUniqueClicks: number;
    totalBounced: number;
    totalComplained: number;
    totalUnsubscribed: number;
    openRate: number;         // unique_opens / delivered
    clickRate: number;        // unique_clicks / delivered
    bounceRate: number;       // bounced / sent
    clickToOpenRate: number;  // unique_clicks / unique_opens
  };
  links: {
    url: string;
    clickCount: number;
    uniqueClicks: number;
  }[];
  hourlyEngagement: {
    hour: string;             // ISO timestamp truncated to hour
    opens: number;
    clicks: number;
  }[];
}
```

Campaign stats (denormalised counters on the `campaigns` table) are updated by the analytics aggregator worker, which runs every 5 minutes. Unique open/click counts are calculated from `campaign_sends` rows where `opened_at IS NOT NULL` / `clicked_at IS NOT NULL`.

**Testing**:
- `Unit: campaign analytics returns correct open rate (opens / delivered)`
- `Unit: campaign analytics with 0 delivered returns 0 rates (no division by zero)`
- `Unit: per-link click stats sorted by click count descending`
- `Unit: hourly engagement aggregation groups clicks by hour`
- `Integration: analytics aggregator worker updates campaign stats from campaign_sends`

#### 6.2 — Subscriber Growth & Engagement Reporting

**What**: Implement subscriber growth dashboard (daily new/unsubscribe/net growth) and engagement scoring.

**Design**:

```typescript
// GET /api/publications/:publicationId/analytics/growth
// Query: period=30d|90d|12m
interface GrowthAnalytics {
  period: string;
  data: {
    date: string;             // YYYY-MM-DD
    newSubscribers: number;
    unsubscribes: number;
    netGrowth: number;
    totalActive: number;
  }[];
  summary: {
    totalActive: number;
    totalPaid: number;
    growthRate: number;       // percentage over period
  };
}

// GET /api/publications/:publicationId/analytics/engagement
interface EngagementAnalytics {
  segments: {
    label: string;            // "Highly engaged", "Engaged", "Passive", "At risk", "Inactive"
    count: number;
    percentage: number;
    criteria: string;
  }[];
}
```

Engagement scoring algorithm:

```typescript
// apps/api/src/workers/engagement-scorer.ts
// Runs daily. Scores each subscriber 0-100 based on:
// - Opens in last 30 days: weight 1 per open (max 10 points)
// - Clicks in last 30 days: weight 3 per click (max 30 points)
// - Opens in last 31-90 days: weight 0.5 (max 5 points)
// - Clicks in last 31-90 days: weight 1.5 (max 15 points)
// - Paid subscriber bonus: +20 points
// - Recency: +20 points if any activity in last 7 days
// Total capped at 100

export function calculateEngagementScore(activity: {
  opens30d: number;
  clicks30d: number;
  opens90d: number;
  clicks90d: number;
  isPaid: boolean;
  lastActivityDaysAgo: number | null;
}): number;
```

Engagement segments: 80-100 = Highly engaged, 60-79 = Engaged, 40-59 = Passive, 20-39 = At risk, 0-19 = Inactive.

**Testing**:
- `Unit: growth analytics with 5 new and 2 unsubscribes on a day shows netGrowth = 3`
- `Unit: growth analytics 30-day period returns 30 data points`
- `Unit: engagement score — 5 opens + 3 clicks in 30d = 5 + 9 = 14 points + bonuses`
- `Unit: engagement score caps at 100`
- `Unit: engagement score for subscriber with zero activity = 0`
- `Unit: engagement segments correctly classify 5 score ranges`
- `Unit: paid subscriber gets +20 engagement bonus`

#### 6.3 — Revenue Analytics

**What**: Implement revenue reporting: MRR, subscriber churn rate, revenue by tier.

**Design**:

```typescript
// GET /api/publications/:publicationId/analytics/revenue
interface RevenueAnalytics {
  currentMrr: number;                   // cents
  mrrHistory: {
    month: string;                       // YYYY-MM
    mrr: number;
    newMrr: number;
    churnedMrr: number;
    payingSubscribers: number;
  }[];
  byTier: {
    tierName: string;
    subscribers: number;
    mrr: number;
  }[];
  churnRate: number;                     // monthly percentage
  averageRevenuePerSubscriber: number;   // cents
}
```

MRR calculation: sum of all active subscriber_subscriptions' monthly equivalent price (yearly / 12 for annual subscribers).

**Testing**:
- `Unit: MRR calculation with 10 monthly at $10 + 5 yearly at $100 = $10*10 + ($100/12)*5`
- `Unit: churn rate = cancelled in month / active at start of month`
- `Unit: byTier breakdown sums correctly`
- `Unit: revenue with 0 paid subscribers returns all zeros`

---

## Phase 7: Automation Engine

### Purpose

Implement the visual automation workflow engine: trigger-based automations (subscriber added, tag added, date-based), step execution (send email, wait, add/remove tag, condition), and enrollment tracking. After this phase, publications can create automated email sequences (welcome series, onboarding drips, re-engagement campaigns).

### Tasks

#### 7.1 — Automation CRUD & Definition Model

**What**: Implement automation creation, editing, and the JSONB-based workflow definition model that supports linear sequences and conditional branching.

**Design**:

The automation definition uses a JSONB structure (influenced by Data Model Suggestion 3) that can represent both linear sequences and branching workflows:

```typescript
// packages/shared/src/types/automation.ts
export type AutomationStatus = 'active' | 'paused' | 'draft';

export type AutomationTrigger =
  | { type: 'subscriber_added'; config: { listId?: string } }
  | { type: 'tag_added'; config: { tagId: string } }
  | { type: 'subscription_started'; config: { tierId?: string } }
  | { type: 'date_based'; config: { field: 'subscribedAt'; offsetDays: number } };

export type AutomationStep =
  | { id: string; type: 'send_email'; campaignId: string }
  | { id: string; type: 'wait'; duration: string }  // e.g., "1h", "3d", "1w"
  | { id: string; type: 'add_tag'; tagId: string }
  | { id: string; type: 'remove_tag'; tagId: string }
  | {
      id: string;
      type: 'condition';
      field: string;          // e.g., "engagement_score", "opened_last_email"
      operator: 'gt' | 'lt' | 'eq' | 'exists';
      value?: string | number;
      thenStepId: string;
      elseStepId: string;
    };

export interface AutomationDefinition {
  trigger: AutomationTrigger;
  steps: AutomationStep[];
}

// POST /api/publications/:publicationId/automations
interface CreateAutomationRequest {
  name: string;
  description?: string;
  definition: AutomationDefinition;
}

// PATCH /api/publications/:publicationId/automations/:id
// GET /api/publications/:publicationId/automations
// DELETE /api/publications/:publicationId/automations/:id

// POST /api/publications/:publicationId/automations/:id/activate
// POST /api/publications/:publicationId/automations/:id/pause
```

**Testing**:
- `Unit: create automation with valid definition succeeds`
- `Unit: create automation with invalid step type rejects`
- `Unit: activate automation changes status from draft to active`
- `Unit: pause automation changes status from active to paused`
- `Unit: condition step requires thenStepId and elseStepId`
- `Unit: wait step parses duration strings ("1h", "3d", "1w")`

#### 7.2 — Automation Trigger & Enrollment

**What**: Implement the trigger listener that enrolls subscribers into active automations when trigger conditions are met.

**Design**:

```typescript
// apps/api/src/services/automation-engine.ts
export class AutomationEngine {
  // Called by subscriber service after subscriber.created, tag.added, etc.
  async evaluateTriggers(
    event: 'subscriber_added' | 'tag_added' | 'subscription_started',
    context: { subscriberId: string; publicationId: string; listId?: string; tagId?: string; tierId?: string },
  ): Promise<void>;

  // Enrolls a subscriber in an automation
  async enrollSubscriber(automationId: string, subscriberId: string): Promise<void>;
}
```

Enrollment creates an `automation_enrollments` row with `currentStepId` set to the first step and `nextActionAt` calculated from the step type (immediate for send_email/add_tag, future for wait steps).

**Testing**:
- `Unit: subscriber_added event triggers automation with matching list filter`
- `Unit: subscriber_added event does not trigger automation for different list`
- `Unit: subscriber already enrolled is not enrolled again (UNIQUE constraint)`
- `Unit: paused automation does not accept new enrollments`
- `Unit: enrollment sets nextActionAt correctly for wait step (e.g., now + 3 days)`
- `Unit: enrollment sets nextActionAt to now for immediate step types`

#### 7.3 — Step Execution Worker

**What**: Implement the background worker that processes automation steps at their scheduled times.

**Design**:

```typescript
// apps/api/src/workers/automation-processor.ts
// BullMQ repeatable job, runs every minute.
// 1. Query automation_enrollments WHERE status = 'active' AND next_action_at <= NOW()
// 2. For each enrollment, execute the current step:
//    - send_email: enqueue campaign send for the subscriber
//    - wait: (already waited — advance to next step)
//    - add_tag: insert subscriber_tags row
//    - remove_tag: delete subscriber_tags row
//    - condition: evaluate condition, pick thenStepId or elseStepId
// 3. Advance to the next step (or mark completed if no more steps)
// 4. Set next_action_at for the new current step

export class AutomationProcessor {
  async processdue(): Promise<{ processed: number; errors: number }>;

  private async executeStep(
    enrollment: AutomationEnrollment,
    step: AutomationStep,
  ): Promise<{ nextStepId: string | null; nextActionAt: Date | null }>;
}
```

**Testing**:
- `Unit: send_email step enqueues campaign send job for the subscriber`
- `Unit: wait step with "3d" advances after 3 days`
- `Unit: add_tag step creates subscriber_tags row`
- `Unit: condition step with engagement_score > 50 follows thenStepId when score is 75`
- `Unit: condition step follows elseStepId when condition is false`
- `Unit: last step in sequence marks enrollment as completed`
- `Unit: processor handles 100 due enrollments in single batch`
- `Unit: failed step execution does not block other enrollments`

---

## Phase 8: Growth Infrastructure

### Purpose

Implement growth tooling: landing pages, signup forms, referral programs, and RSS feed generation. After this phase, publications have the subscriber acquisition infrastructure needed to grow their audience beyond direct email signups.

### Tasks

#### 8.1 — Landing Pages

**What**: Implement landing page CRUD and public rendering for subscriber acquisition.

**Design**:

```typescript
// POST /api/publications/:publicationId/landing-pages
interface CreateLandingPageRequest {
  title: string;
  slug: string;
  htmlContent: string;
  listId?: string;           // subscribers go to this list
}

// GET /api/publications/:publicationId/landing-pages
// PATCH /api/publications/:publicationId/landing-pages/:id
// POST /api/publications/:publicationId/landing-pages/:id/publish
// DELETE /api/publications/:publicationId/landing-pages/:id

// Public route: GET /:publicationSlug/p/:pageSlug
// Renders the landing page with an embedded signup form
// POST /:publicationSlug/p/:pageSlug/subscribe
// Creates subscriber, increments landing_pages.conversion_count
```

Landing page view counting: increment `view_count` on each GET request (debounced per IP via Redis, 1 count per IP per hour).

**Testing**:
- `Unit: create landing page with valid data succeeds`
- `Unit: publish landing page sets is_published = true`
- `Unit: public GET renders HTML content with signup form`
- `Unit: subscribe via landing page creates subscriber in correct list`
- `Unit: subscribe via landing page increments conversion_count`
- `Unit: view counting deduplicates by IP within 1 hour`
- `Unit: slug uniqueness per publication enforced`

#### 8.2 — Signup Forms & Embeds

**What**: Implement configurable signup forms (inline, popup, slide-in) with embeddable HTML/JS snippets.

**Design**:

```typescript
// POST /api/publications/:publicationId/forms
interface CreateFormRequest {
  name: string;
  formType: 'inline' | 'popup' | 'slide_in';
  listId?: string;
  htmlSnippet?: string;      // custom HTML
}

// GET /api/publications/:publicationId/forms
// GET /api/publications/:publicationId/forms/:id/embed
// Returns: { html: string, scriptTag: string }
// The embed code loads a script that renders the form on the publisher's site

// POST /api/forms/:id/submit — public endpoint
// Body: { email: string, name?: string }
// CORS: accepts requests from any origin (embed on external sites)
```

**Testing**:
- `Unit: create form with valid data succeeds`
- `Unit: embed endpoint returns valid HTML with form action URL`
- `Unit: form submission creates subscriber and increments submission_count`
- `Unit: form submission from external origin succeeds (CORS: *)`
- `Unit: form submission with invalid email returns 400`
- `Unit: form submission with existing subscriber returns 200 (idempotent)`

#### 8.3 — Referral Program

**What**: Implement a referral program where subscribers earn rewards for referring new subscribers.

**Design**:

```typescript
// POST /api/publications/:publicationId/referral-programs
interface CreateReferralProgramRequest {
  name: string;
  milestones: {
    referralsRequired: number;
    rewardDescription: string;
    rewardType: 'digital_product' | 'tier_upgrade' | 'custom';
  }[];
}

// Each subscriber gets a unique referral link:
// https://{publication}.{domain}/subscribe?ref={subscriberId}
// When a new subscriber signs up via the referral link:
// 1. Create subscriber with referralSource = referrer's subscriber ID
// 2. Create referrals row (referrer_id, referred_id)
// 3. Check if referrer has hit any milestones
// 4. Trigger milestone reward automation

// GET /api/publications/:publicationId/referral-programs/:id/leaderboard
interface ReferralLeaderboard {
  entries: {
    subscriberEmail: string;
    referralCount: number;
    currentMilestone: string | null;
    nextMilestone: { referralsRequired: number; reward: string } | null;
  }[];
}
```

**Testing**:
- `Unit: create referral program with 3 milestones succeeds`
- `Unit: subscriber signup via referral link creates referrals row`
- `Unit: referral count reaching milestone triggers reward`
- `Unit: leaderboard returns top referrers sorted by count`
- `Unit: self-referral (same email) is rejected`
- `Unit: duplicate referral (same referred subscriber) is rejected`

#### 8.4 — RSS & Atom Feed Generation

**What**: Implement RSS 2.0 and Atom (RFC 4287) feed generation for published posts.

**Design**:

```typescript
// GET /:publicationSlug/rss — RSS 2.0 feed
// GET /:publicationSlug/atom — Atom feed

// apps/api/src/lib/rss-feed.ts
export function generateRssFeed(
  publication: Publication,
  posts: Post[],             // published, public visibility, sorted by publishedAt desc
  baseUrl: string,
): string;                   // XML string

export function generateAtomFeed(
  publication: Publication,
  posts: Post[],
  baseUrl: string,
): string;
```

Feed items include: title, link, pubDate, description (excerpt), content:encoded (full HTML for public posts), guid (rssGuid from posts table), author. Feed is limited to the 20 most recent published posts.

**Testing**:
- `Unit: RSS feed contains correct XML declaration and channel metadata`
- `Unit: RSS feed includes 20 most recent public posts`
- `Unit: RSS feed excludes posts with visibility = paid`
- `Unit: Atom feed validates against RFC 4287 structure`
- `Unit: feed item guid matches post.rssGuid`
- `Unit: empty publication returns valid feed with 0 items`
- `Fixture: tests/fixtures/expected-rss-output.xml`

---

## Phase 9: Public Newsletter Website

### Purpose

Implement the public-facing newsletter website using Next.js: publication homepage, post archive, individual post pages with content gating, and subscriber login for paid content access. After this phase, each publication has a fully functional web presence for reader discovery and content consumption.

### Tasks

#### 9.1 — Publication Homepage & Post Archive

**What**: Implement the public homepage showing publication info, recent posts, and a subscribe form.

**Design**:

```typescript
// Next.js App Router pages
// app/(public)/[publicationSlug]/page.tsx — Homepage
// app/(public)/[publicationSlug]/archive/page.tsx — Paginated post archive
// app/(public)/[publicationSlug]/[postSlug]/page.tsx — Individual post

// Homepage data fetching (Server Component):
async function getPublicationData(slug: string) {
  const publication = await api.get(`/publications/by-slug/${slug}`);
  const recentPosts = await api.get(`/publications/${publication.id}/posts?status=published&limit=10`);
  const tiers = await api.get(`/publications/${publication.id}/tiers`);
  return { publication, recentPosts, tiers };
}
```

The homepage renders: publication name/description/logo, recent post cards (title, excerpt, date, visibility badge), subscribe CTA with email input, and paid tier cards with pricing.

Static generation: pages are statically generated at build time with ISR (revalidate every 60 seconds) for fast load times.

**Testing**:
- `E2E: visit /{slug} shows publication name and recent posts`
- `E2E: visit /{slug}/archive shows paginated post list`
- `E2E: subscribe form on homepage creates subscriber`
- `E2E: paid tier cards show correct pricing`

#### 9.2 — Post Pages with Content Gating

**What**: Implement individual post pages that render full content for authorised readers and truncated previews with subscribe CTAs for others.

**Design**:

Post page rendering logic:
1. Fetch post and publication tiers
2. If post is `public`: render full content
3. If post is `members` or `paid`: check for subscriber session cookie
4. If subscriber has access: render full content
5. If subscriber lacks access: render excerpt + paywall CTA

Subscriber authentication for public site: magic link login (email a one-time login link) that sets an HTTP-only cookie with a JWT.

```typescript
// POST /api/auth/magic-link
interface MagicLinkRequest {
  email: string;
  publicationId: string;
  returnUrl: string;
}

// GET /api/auth/magic-link/verify?token=xxx
// Sets HTTP-only cookie and redirects to returnUrl
```

**Testing**:
- `E2E: public post renders full content for anonymous visitor`
- `E2E: paid post renders excerpt with subscribe CTA for anonymous visitor`
- `E2E: paid post renders full content for logged-in paid subscriber`
- `E2E: magic link login sets cookie and redirects`
- `Unit: expired magic link returns error page`

---

## Phase 10: Dashboard UI

### Purpose

Implement the creator dashboard using Next.js: the authenticated admin interface where publication owners and team members manage subscribers, write posts, create campaigns, configure automations, view analytics, and manage settings. This phase builds the frontend for all API functionality from phases 2-8.

### Tasks

#### 10.1 — Dashboard Layout & Navigation

**What**: Implement the authenticated dashboard shell with sidebar navigation, publication switcher, and responsive layout.

**Design**:

Dashboard navigation structure:
- Posts (list, create/edit)
- Campaigns (list, create, analytics)
- Subscribers (list, detail, import)
- Automations (list, create/edit)
- Analytics (overview, campaigns, growth, revenue)
- Growth (landing pages, forms, referrals)
- Monetisation (tiers, subscriber management)
- Settings (publication, team, domains, API keys)

```typescript
// apps/web/src/components/ui/sidebar.tsx
interface NavItem {
  label: string;
  href: string;
  icon: React.ComponentType;
  badge?: number;             // e.g., draft count
}
```

Auth: the dashboard uses a JWT stored in an HTTP-only cookie, set during login. API calls include the cookie automatically.

**Testing**:
- `E2E: login redirects to dashboard`
- `E2E: sidebar navigation renders all sections`
- `E2E: publication switcher shows user's publications`
- `E2E: unauthenticated access to /dashboard redirects to /login`

#### 10.2 — Post Editor (Tiptap)

**What**: Implement the rich text post editor using Tiptap with Markdown support, image uploads, and live preview.

**Design**:

Tiptap editor with extensions:
- StarterKit (bold, italic, headings, lists, code blocks)
- Image (drag-and-drop upload to media API)
- Link (with URL validation)
- Placeholder
- Markdown input/output (via tiptap-markdown)
- Paywall marker (custom node: inserts a paywall break point)

```typescript
// apps/web/src/components/editor/post-editor.tsx
interface PostEditorProps {
  initialContent?: string;    // HTML or Markdown
  onSave: (content: { html: string; markdown: string; plainText: string }) => void;
  onPublish: () => void;
  publicationId: string;
}
```

Auto-save: editor content is debounced (2 seconds) and saved via PATCH to the posts API.

**Testing**:
- `E2E: type text in editor and save — post content persists`
- `E2E: insert image via drag-and-drop — image appears in editor`
- `E2E: toggle Markdown mode — content round-trips correctly`
- `E2E: insert paywall marker — marker visible in editor`
- `E2E: auto-save triggers after 2 seconds of inactivity`

#### 10.3 — Subscriber Management UI

**What**: Implement the subscriber list view with search, filters, bulk actions, and detail view.

**Design**:

Subscriber list features:
- Table with columns: email, name, status, tags, tier, subscribedAt
- Search by email or name
- Filter by status, tag, list, tier
- Bulk actions: add tag, remove tag, add to list, delete
- CSV export
- Detail view: subscriber profile with engagement history (campaigns received, opened, clicked)

**Testing**:
- `E2E: subscriber list loads and displays subscribers`
- `E2E: search by email filters results`
- `E2E: bulk tag add applies tag to selected subscribers`
- `E2E: CSV export downloads file with correct columns`
- `E2E: subscriber detail shows engagement history`

#### 10.4 — Campaign Builder & Analytics UI

**What**: Implement the campaign creation wizard (select lists, compose email, preview, schedule/send) and campaign analytics dashboard.

**Design**:

Campaign creation flow:
1. Name and select target lists
2. Compose email (use post content or write directly)
3. Preview (render in email client preview)
4. Schedule or send immediately
5. Monitor sending progress

Analytics dashboard: open rate, click rate, bounce rate charts; per-link click table; hourly engagement timeline.

**Testing**:
- `E2E: create campaign, select lists, compose email, send — campaign status becomes "sent"`
- `E2E: schedule campaign for future — campaign status is "scheduled"`
- `E2E: campaign analytics page shows open rate chart`
- `E2E: campaign analytics shows per-link click counts`

#### 10.5 — Automation Builder UI

**What**: Implement the visual automation workflow builder where users can drag-and-drop steps to create automation sequences.

**Design**:

The automation builder uses a visual flow editor:
- Trigger node at the top (select trigger type and config)
- Step nodes below (send_email, wait, condition, add_tag, remove_tag)
- Drag-and-drop reordering
- Condition nodes show branching paths (then/else)
- Each step node has a configuration panel

State management: zustand store for the automation definition being edited.

```typescript
// apps/web/src/components/automation/automation-builder.tsx
interface AutomationBuilderProps {
  publicationId: string;
  automationId?: string;     // undefined for new, set for edit
  initialDefinition?: AutomationDefinition;
  onSave: (definition: AutomationDefinition) => void;
}
```

**Testing**:
- `E2E: add send_email step — step appears in workflow`
- `E2E: add wait step with "3 days" — step shows "Wait 3 days"`
- `E2E: add condition step — branching paths render`
- `E2E: save automation — definition persisted via API`
- `E2E: edit existing automation — steps pre-populated`

#### 10.6 — Analytics Dashboard UI

**What**: Implement the publication-level analytics dashboard with growth, engagement, and revenue views.

**Design**:

Three analytics tabs:
1. **Growth**: line chart of subscriber growth over time, new vs. unsubscribed; funnel showing acquisition sources
2. **Engagement**: pie chart of engagement segments (highly engaged, engaged, passive, at risk, inactive); list of top-engaged subscribers
3. **Revenue**: MRR trend chart; revenue by tier breakdown; churn rate

Charts use Recharts (responsive, server-side renderable).

**Testing**:
- `E2E: growth tab shows line chart with data points`
- `E2E: engagement tab shows pie chart with 5 segments`
- `E2E: revenue tab shows MRR number and trend chart`
- `E2E: date range selector updates chart data`

---

## Phase 11: API Keys & Developer API

### Purpose

Implement the public REST API with API key authentication, OpenAPI 3.1 documentation, and rate limiting. After this phase, developers can programmatically manage subscribers, posts, and campaigns, enabling integrations with Zapier, Make, and custom tools.

### Tasks

#### 11.1 — API Key Management

**What**: Implement API key creation, listing, revocation, and authentication middleware.

**Design**:

```typescript
// POST /api/publications/:publicationId/api-keys
interface CreateApiKeyRequest {
  name: string;
  scopes: string[];          // e.g., ["subscribers:read", "subscribers:write", "posts:read"]
  expiresAt?: string;        // ISO 8601, optional
}
interface CreateApiKeyResponse {
  id: string;
  key: string;               // Only returned once, at creation
  keyPrefix: string;
  name: string;
  scopes: string[];
  expiresAt: string | null;
}

// GET /api/publications/:publicationId/api-keys
// DELETE /api/publications/:publicationId/api-keys/:id (revoke)
```

API key format: `nl_pub_{random32chars}`. The key is hashed with SHA-256 before storage. Only the `key_prefix` (first 8 chars) and `key_hash` are stored. Authentication: `Authorization: Bearer nl_pub_...`.

Scope enforcement middleware:

```typescript
// apps/api/src/middleware/api-key-auth.ts
export function requireApiKey(requiredScope: string) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const key = extractBearerToken(request);
    if (!key?.startsWith('nl_pub_')) return reply.status(401).send({ error: 'Invalid API key' });
    const hash = sha256(key);
    const apiKey = await db.query.apiKeys.findFirst({ where: eq(apiKeys.keyHash, hash) });
    if (!apiKey) return reply.status(401).send({ error: 'Invalid API key' });
    if (apiKey.expiresAt && apiKey.expiresAt < new Date()) return reply.status(401).send({ error: 'Expired API key' });
    if (!apiKey.scopes.includes(requiredScope)) return reply.status(403).send({ error: 'Insufficient scope' });
    request.publication = { id: apiKey.publicationId };
  };
}
```

**Testing**:
- `Unit: create API key returns key only once (not in subsequent GETs)`
- `Unit: API key authentication with valid key succeeds`
- `Unit: API key authentication with revoked key returns 401`
- `Unit: API key authentication with expired key returns 401`
- `Unit: scope enforcement — subscribers:read key cannot POST subscribers (403)`
- `Unit: API key prefix stored for identification`

#### 11.2 — Public API Endpoints & OpenAPI Spec

**What**: Expose the subscriber, post, campaign, and analytics endpoints under the public API with OpenAPI 3.1 auto-generation.

**Design**:

Public API routes (prefixed `/v1/`):
```
GET    /v1/subscribers          (scope: subscribers:read)
POST   /v1/subscribers          (scope: subscribers:write)
GET    /v1/subscribers/:id      (scope: subscribers:read)
PATCH  /v1/subscribers/:id      (scope: subscribers:write)
DELETE /v1/subscribers/:id      (scope: subscribers:write)

GET    /v1/posts                (scope: posts:read)
POST   /v1/posts                (scope: posts:write)
GET    /v1/posts/:id            (scope: posts:read)
PATCH  /v1/posts/:id            (scope: posts:write)

GET    /v1/campaigns            (scope: campaigns:read)
POST   /v1/campaigns            (scope: campaigns:write)
POST   /v1/campaigns/:id/send   (scope: campaigns:write)
GET    /v1/campaigns/:id/stats  (scope: analytics:read)
```

OpenAPI generation: Fastify's `@fastify/swagger` + `@fastify/swagger-ui` auto-generates the OpenAPI 3.1 spec from route schemas. Available at `/v1/docs`.

Rate limiting: 100 requests/minute per API key (configurable), implemented with `@fastify/rate-limit` backed by Redis.

**Testing**:
- `Unit: all /v1/ endpoints require API key authentication`
- `Unit: GET /v1/docs returns valid OpenAPI 3.1 JSON`
- `Unit: rate limit returns 429 after 100 requests in 1 minute`
- `Integration: /v1/subscribers CRUD via API key matches dashboard behaviour`

---

## Phase 12: AI-Native Features

### Purpose

Implement the AI-powered differentiating features: writing co-pilot for drafting posts and subject lines, subject line optimisation, send-time optimisation, and churn prediction. After this phase, the platform has the AI-native advantages described in the project README that set it apart from Substack, beehiiv, and Ghost.

### Tasks

#### 12.1 — Writing Co-Pilot

**What**: Implement an AI writing assistant that generates draft posts, subject lines, and hooks based on the creator's writing style.

**Design**:

```typescript
// POST /api/publications/:publicationId/ai/draft
interface DraftRequest {
  prompt: string;            // "Write about AI trends in 2026"
  tone?: 'casual' | 'professional' | 'conversational';
  length?: 'short' | 'medium' | 'long';
  referencePostIds?: string[];  // Posts to use as style reference
}
interface DraftResponse {
  content: string;           // Markdown
  suggestedTitle: string;
  suggestedSubjectLines: string[];
}

// POST /api/publications/:publicationId/ai/subject-lines
interface SubjectLineRequest {
  postTitle: string;
  postExcerpt: string;
  count?: number;            // default 5
}
interface SubjectLineResponse {
  suggestions: {
    text: string;
    estimatedOpenRate: 'low' | 'medium' | 'high';
    reasoning: string;
  }[];
}
```

The writing co-pilot uses the publication's existing posts as context (most recent 10 posts' plain text) to match the creator's voice and style. The LLM call uses a system prompt that establishes tone calibration from the archive.

```typescript
// apps/api/src/services/ai/writing-copilot.ts
const systemPrompt = `You are a writing assistant for the newsletter "${publicationName}".
Your task is to write in the same voice, tone, and style as the creator.
Below are recent posts from this newsletter to calibrate your writing style.

${referencePosts.map(p => `--- ${p.title} ---\n${p.plainText}`).join('\n\n')}

Write content that matches this voice. Output in Markdown format.`;
```

**Testing**:
- `Unit: draft request with valid prompt returns markdown content`
- `Unit: draft request includes reference posts in LLM context (mocked LLM)`
- `Unit: subject line request returns 5 suggestions by default`
- `Unit: draft with tone=professional adjusts system prompt`
- `Integration (mocked LLM): end-to-end draft generation completes within 30 seconds`

#### 12.2 — Subject Line Optimiser

**What**: Implement A/B testing for subject lines with automated winner selection based on open rates.

**Design**:

```typescript
// POST /api/publications/:publicationId/campaigns/:id/ab-test
interface AbTestRequest {
  subjectLines: string[];     // 2-5 variants
  testPercentage: number;     // 10-50% of recipients for initial test
  winnerMetric: 'open_rate';
  testDurationHours: number;  // 1-24
}

// AB test flow:
// 1. Split testPercentage of recipients into N equal groups
// 2. Send each group a different subject line
// 3. After testDurationHours, pick the winner (highest open rate)
// 4. Send the winning subject line to the remaining recipients
```

**Testing**:
- `Unit: AB test with 2 subject lines splits test group into 2 equal groups`
- `Unit: AB test with 20% test on 1000 recipients sends to 100 per variant`
- `Unit: winner selection picks subject line with highest open rate after duration`
- `Unit: remaining 80% receives the winning subject line`
- `Unit: AB test with 1 subject line is rejected (minimum 2)`

#### 12.3 — Churn Prediction

**What**: Implement a churn prediction model that identifies at-risk paid subscribers and triggers retention campaigns.

**Design**:

```typescript
// apps/api/src/services/ai/churn-predictor.ts
export interface ChurnPrediction {
  subscriberId: string;
  churnProbability: number;    // 0-1
  riskLevel: 'low' | 'medium' | 'high';
  factors: string[];           // e.g., ["No opens in 30 days", "Payment failed once"]
  suggestedAction: string;     // e.g., "Send re-engagement email", "Offer discount"
}

// GET /api/publications/:publicationId/ai/churn-predictions
// Query: riskLevel=high, limit=50
// Returns: ChurnPrediction[]

// Churn signal features:
// - Days since last open
// - Days since last click
// - Open rate trend (declining, stable, increasing)
// - Payment failures in last 90 days
// - Engagement score trend
// - Time since subscription started
```

The churn prediction runs as a daily background job. It calculates a churn probability score for each paid subscriber based on weighted features. Subscribers with `churnProbability > 0.7` are tagged with `at_risk_churn`, which can trigger a retention automation.

**Testing**:
- `Unit: subscriber with 0 opens in 60 days gets high churn probability`
- `Unit: subscriber with daily clicks in last 30 days gets low churn probability`
- `Unit: payment failure increases churn probability`
- `Unit: churn predictions correctly tag at-risk subscribers`
- `Unit: churn prediction worker processes 10,000 subscribers within 60 seconds`
- `Unit: churn factors list includes human-readable reasons`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Core Schema         ─── required by everything
    │
Phase 2: Auth & Publication Management    ─── requires Phase 1
    │
Phase 3: Subscriber & Email Infrastructure ─── requires Phase 2
    │
    ├── Phase 4: Content & Campaign Sending ─── requires Phase 3
    │       │
    │       └── Phase 5: Paid Subscriptions ─── requires Phase 4
    │               │
    │               └── Phase 6: Analytics  ─── requires Phase 4 + 5
    │
    ├── Phase 7: Automation Engine          ─── requires Phase 3 + 4
    │
    └── Phase 8: Growth Infrastructure      ─── requires Phase 3
            │
            └── Phase 9: Public Website     ─── requires Phase 4 + 5 + 8
                    │
                    └── Phase 10: Dashboard UI ─── requires Phases 2-8
                            │
                            └── Phase 11: API Keys & Developer API ─── requires Phase 10
                                    │
                                    └── Phase 12: AI-Native Features ─── requires Phase 6 + 7

Parallelism opportunities:
  - Phases 7 and 8 can be developed concurrently after Phase 3 + 4
  - Phase 5 and Phase 8 can be developed concurrently after Phase 4
  - Phase 9 and Phase 10 can be started concurrently once their dependencies are met
```

---

## Definition of Done (per phase)

1. All tasks for the phase are implemented.
2. All unit tests pass (`pnpm turbo test`).
3. All integration tests pass (with mocked external dependencies).
4. Biome linting passes with zero errors (`pnpm turbo lint`).
5. TypeScript type checking passes (`pnpm turbo typecheck`).
6. Docker build succeeds for all modified services (`docker-compose build`).
7. Database migrations are created and apply cleanly to a fresh database.
8. API endpoints return correct HTTP status codes and JSON error bodies.
9. New API endpoints have JSON Schema validation on request bodies.
10. New configuration options are documented in `.env.example`.
11. E2E tests pass for user-facing features (Playwright).
12. No regression in existing tests from prior phases.
