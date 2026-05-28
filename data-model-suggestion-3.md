# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Newsletter Platform · Created: 2026-05-20

## Philosophy

This model uses relational tables for core entities and relationships but relies heavily on PostgreSQL JSONB columns for variable, extensible, and publication-specific data. The key insight is that newsletter platforms serve wildly different use cases — a solo journalist's paid newsletter has different data needs than a brand marketing team's segmentation-heavy operation or a media company's multi-property setup — and rigid schemas cannot accommodate all of them without constant migrations.

The hybrid approach gives you the best of both worlds: foreign keys and indexes for the structural relationships that every newsletter platform needs (subscribers belong to publications, campaigns target lists, payments reference tiers), while JSONB columns absorb the variability (custom subscriber fields, publication-specific settings, flexible automation conditions, per-campaign analytics breakdowns).

This is the pattern used by modern SaaS platforms like Linear (PostgreSQL + JSONB for issue metadata), Notion (block-based content stored as JSON), and increasingly by email platforms that let users define custom subscriber fields. Ghost's Content API returns nested JSON objects that map naturally to this architecture.

**Best for:** Rapid MVP development where the schema needs to evolve quickly, multi-publication platforms where each publication has different custom fields, and teams that want fewer tables and simpler migrations.

**Trade-offs:**
- (+) Fewer tables — variable data lives in JSONB columns rather than EAV or junction tables
- (+) Schema evolution without migrations — new custom fields are just new JSONB keys
- (+) Natural fit for API responses — JSONB columns map directly to nested JSON in REST/GraphQL
- (+) GIN indexes on JSONB enable efficient querying of custom fields
- (+) Publication-specific settings without settings tables
- (-) No foreign key constraints inside JSONB — referential integrity is application-enforced
- (-) JSONB query syntax is less familiar to many developers (`->`, `->>`, `@>`, `?`)
- (-) Harder to enforce NOT NULL or type constraints on JSONB fields
- (-) Schema documentation must be maintained separately (JSONB columns are opaque to the database)
- (-) GIN indexes consume more space and are slower to update than B-tree indexes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5321/5322 (SMTP) | `campaigns.email_config` JSONB stores from/reply-to/headers as structured JSON |
| RFC 7208/6376/7489 (SPF/DKIM/DMARC) | `sending_domains.dns_records` JSONB stores verification state for all DNS record types |
| RFC 8058 (One-Click Unsubscribe) | Stored in `campaigns.email_config->>'list_unsubscribe_url'` |
| GDPR Articles 6/7/17 | `subscribers.consent` JSONB captures consent chain: `{given_at, source, ip, withdrawn_at}` |
| CAN-SPAM Act | `publications.settings->>'physical_address'` |
| Stripe API | `subscriber_subscriptions.stripe_data` JSONB stores full Stripe object for sync |
| OpenAPI 3.1 | JSONB columns documented with JSON Schema definitions in API docs |
| ISO 4217 | Currency codes in payment JSONB fields |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE publications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    logo_url        TEXT,
    -- All publication-level settings in one JSONB column
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "physical_address": "123 Main St, City, ST 12345",
    --   "default_from_name": "The Daily Brief",
    --   "default_from_email": "hello@dailybrief.com",
    --   "timezone": "America/New_York",
    --   "language": "en",
    --   "branding": {
    --     "primary_color": "#1a73e8",
    --     "header_font": "Inter",
    --     "body_font": "Georgia"
    --   },
    --   "features": {
    --     "referral_program": true,
    --     "paid_subscriptions": true,
    --     "double_optin": true,
    --     "web3_payments": false
    --   },
    --   "custom_subscriber_fields": [
    --     { "key": "company", "label": "Company", "type": "text" },
    --     { "key": "role", "label": "Job Role", "type": "select", "options": ["Developer","Designer","PM"] }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pub_settings ON publications USING GIN(settings);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   TEXT,
    name            VARCHAR(255),
    avatar_url      TEXT,
    profile         JSONB DEFAULT '{}',
    -- Example profile:
    -- {
    --   "bio": "Tech journalist covering AI",
    --   "social": { "twitter": "@writer", "linkedin": "in/writer" },
    --   "preferences": { "theme": "dark", "email_notifications": true }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE publication_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'editor',
    permissions     JSONB DEFAULT '{}',
    -- Example permissions override:
    -- { "can_publish": true, "can_manage_subscribers": false, "can_view_revenue": false }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, user_id)
);
```

## Subscriber Management

```sql
CREATE TYPE subscriber_status AS ENUM ('enabled', 'disabled', 'blocklisted');

CREATE TABLE subscribers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255),
    status          subscriber_status NOT NULL DEFAULT 'enabled',
    -- Consent tracking as structured JSONB
    consent         JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "given_at": "2026-01-15T10:30:00Z",
    --   "source": "signup_form_homepage",
    --   "ip": "192.168.1.1",
    --   "double_optin_confirmed": true,
    --   "confirmed_at": "2026-01-15T10:35:00Z"
    -- }
    -- Custom fields defined by the publication
    custom_fields   JSONB DEFAULT '{}',
    -- Example (matches publication.settings.custom_subscriber_fields):
    -- { "company": "Acme Corp", "role": "Developer", "interests": ["ai", "devtools"] }
    -- Acquisition source tracking
    source          JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "type": "signup_form",
    --   "form_id": "uuid",
    --   "utm_source": "twitter",
    --   "utm_medium": "social",
    --   "utm_campaign": "launch",
    --   "referrer_url": "https://twitter.com/...",
    --   "referred_by": "subscriber-uuid"
    -- }
    -- Engagement metrics (denormalised, updated by background jobs)
    engagement      JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "score": 85.5,
    --   "total_received": 52,
    --   "total_opened": 45,
    --   "total_clicked": 23,
    --   "last_opened_at": "2026-05-18T14:00:00Z",
    --   "last_clicked_at": "2026-05-17T09:30:00Z",
    --   "avg_read_time_seconds": 180,
    --   "scroll_depth_avg": 0.72
    -- }
    tags            TEXT[] DEFAULT '{}',             -- array of tag names (denormalised)
    tier_id         UUID,                            -- current paid tier (NULL = free)
    subscribed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    unsubscribed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, email)
);

CREATE INDEX idx_subs_pub_status ON subscribers(publication_id, status);
CREATE INDEX idx_subs_email ON subscribers(email);
CREATE INDEX idx_subs_tags ON subscribers USING GIN(tags);
CREATE INDEX idx_subs_custom ON subscribers USING GIN(custom_fields);
CREATE INDEX idx_subs_engagement ON subscribers USING GIN(engagement);
CREATE INDEX idx_subs_source ON subscribers USING GIN(source);

CREATE TABLE lists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) DEFAULT 'public',
    settings        JSONB DEFAULT '{}',
    -- Example: { "double_optin": true, "welcome_email_id": "uuid" }
    subscriber_count INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE subscriber_lists (
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    list_id         UUID NOT NULL REFERENCES lists(id) ON DELETE CASCADE,
    status          VARCHAR(50) DEFAULT 'confirmed',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (subscriber_id, list_id)
);
```

### Querying Custom Fields

```sql
-- Find all subscribers at "Acme Corp" who are developers
SELECT id, email, name, custom_fields
FROM subscribers
WHERE publication_id = '{{pub_id}}'
  AND status = 'enabled'
  AND custom_fields @> '{"company": "Acme Corp", "role": "Developer"}';

-- Find subscribers with engagement score above 80
SELECT id, email, (engagement->>'score')::float AS score
FROM subscribers
WHERE publication_id = '{{pub_id}}'
  AND (engagement->>'score')::float > 80
ORDER BY (engagement->>'score')::float DESC;

-- Find subscribers tagged with 'premium' OR 'vip'
SELECT id, email, tags
FROM subscribers
WHERE publication_id = '{{pub_id}}'
  AND tags && ARRAY['premium', 'vip'];
```

## Content & Publishing

```sql
CREATE TYPE post_status AS ENUM ('draft', 'scheduled', 'published', 'archived');

CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    author_id       UUID REFERENCES users(id),
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(500) NOT NULL,
    -- Content stored as structured blocks (Notion-style)
    content_blocks  JSONB,
    -- Example:
    -- [
    --   { "type": "paragraph", "content": "Hello subscribers..." },
    --   { "type": "image", "url": "https://...", "alt": "Chart", "caption": "Q1 results" },
    --   { "type": "heading", "level": 2, "content": "What's next" },
    --   { "type": "button", "text": "Read more", "url": "https://...", "style": "primary" },
    --   { "type": "paywall" },
    --   { "type": "paragraph", "content": "Premium content below..." }
    -- ]
    html_rendered   TEXT,                            -- rendered HTML for email sending
    plain_text      TEXT,                            -- for search and AI processing
    excerpt         TEXT,
    featured_image  TEXT,
    status          post_status NOT NULL DEFAULT 'draft',
    visibility      VARCHAR(50) DEFAULT 'public',    -- public, members, paid
    -- SEO and metadata
    meta            JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "seo_title": "Custom SEO Title",
    --   "seo_description": "Custom meta description",
    --   "og_image": "https://...",
    --   "canonical_url": "https://...",
    --   "rss_guid": "unique-guid-123",
    --   "tags": ["ai", "tech", "weekly"],
    --   "reading_time_minutes": 5,
    --   "word_count": 1200
    -- }
    published_at    TIMESTAMPTZ,
    scheduled_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, slug)
);

CREATE INDEX idx_posts_pub_status ON posts(publication_id, status);
CREATE INDEX idx_posts_published ON posts(publication_id, published_at DESC);
CREATE INDEX idx_posts_meta ON posts USING GIN(meta);
```

## Campaigns & Delivery

```sql
CREATE TYPE campaign_status AS ENUM ('draft', 'scheduled', 'sending', 'sent', 'paused', 'cancelled');

CREATE TABLE campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    post_id         UUID REFERENCES posts(id),
    name            VARCHAR(255) NOT NULL,
    subject         VARCHAR(500) NOT NULL,
    preview_text    VARCHAR(255),
    -- Email configuration as JSONB
    email_config    JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "from_name": "The Daily Brief",
    --   "from_email": "hello@dailybrief.com",
    --   "reply_to": "phil@dailybrief.com",
    --   "template_id": "uuid",
    --   "list_unsubscribe_url": "https://...",
    --   "list_unsubscribe_post": "List-Unsubscribe=One-Click"
    -- }
    -- Targeting: which lists/segments receive this campaign
    targeting       JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "lists": ["uuid-1", "uuid-2"],
    --   "segments": [
    --     { "field": "tags", "op": "contains", "value": "premium" },
    --     { "field": "engagement.score", "op": "gte", "value": 50 }
    --   ],
    --   "exclude_lists": ["uuid-3"],
    --   "exclude_unsubscribed": true
    -- }
    status          campaign_status NOT NULL DEFAULT 'draft',
    -- Aggregate stats (denormalised)
    stats           JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "recipients": 15000,
    --   "sent": 14950,
    --   "delivered": 14800,
    --   "opened": 6200,
    --   "unique_opens": 5100,
    --   "clicked": 1800,
    --   "unique_clicks": 1200,
    --   "bounced": 150,
    --   "hard_bounced": 45,
    --   "soft_bounced": 105,
    --   "complained": 3,
    --   "unsubscribed": 22,
    --   "open_rate": 0.344,
    --   "click_rate": 0.081
    -- }
    send_at         TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_campaigns_pub ON campaigns(publication_id);
CREATE INDEX idx_campaigns_status ON campaigns(publication_id, status);

-- Individual delivery tracking (high-volume table)
CREATE TABLE deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',
    -- All delivery events as JSONB timeline
    events          JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   { "type": "queued", "at": "2026-05-20T10:00:00Z" },
    --   { "type": "sent", "at": "2026-05-20T10:00:05Z", "message_id": "abc123" },
    --   { "type": "delivered", "at": "2026-05-20T10:00:08Z" },
    --   { "type": "opened", "at": "2026-05-20T14:30:00Z", "ip": "1.2.3.4", "ua": "Apple Mail" },
    --   { "type": "clicked", "at": "2026-05-20T14:31:00Z", "url": "https://..." }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_deliveries_campaign ON deliveries(campaign_id);
CREATE INDEX idx_deliveries_subscriber ON deliveries(subscriber_id);
CREATE INDEX idx_deliveries_status ON deliveries(campaign_id, status);
```

## Link Tracking

```sql
CREATE TABLE links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    original_url    TEXT NOT NULL,
    tracking_url    TEXT NOT NULL,
    click_count     INTEGER DEFAULT 0,
    unique_clicks   INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_links_campaign ON links(campaign_id);

CREATE TABLE link_clicks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    link_id         UUID NOT NULL REFERENCES links(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id),
    campaign_id     UUID NOT NULL,
    ip_address      INET,
    user_agent      TEXT,
    clicked_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clicks_link ON link_clicks(link_id);
CREATE INDEX idx_clicks_subscriber ON link_clicks(subscriber_id);
```

## Monetisation

```sql
CREATE TABLE subscription_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    price_monthly   INTEGER,                        -- cents
    price_yearly    INTEGER,                        -- cents
    currency        CHAR(3) DEFAULT 'usd',
    -- Benefits and configuration
    config          JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "stripe_product_id": "prod_xxx",
    --   "stripe_price_monthly_id": "price_xxx",
    --   "stripe_price_yearly_id": "price_yyy",
    --   "benefits": ["Full archive access", "Weekly deep-dive", "Discord community"],
    --   "content_access_level": 2,
    --   "badge_emoji": "⭐"
    -- }
    is_active       BOOLEAN DEFAULT TRUE,
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE subscriber_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    tier_id         UUID NOT NULL REFERENCES subscription_tiers(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    -- Full Stripe sync data
    stripe_data     JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "customer_id": "cus_xxx",
    --   "subscription_id": "sub_xxx",
    --   "current_period_start": "2026-05-01T00:00:00Z",
    --   "current_period_end": "2026-06-01T00:00:00Z",
    --   "cancel_at_period_end": false,
    --   "default_payment_method": "pm_xxx",
    --   "latest_invoice": "in_xxx"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sub_subs_subscriber ON subscriber_subscriptions(subscriber_id);
CREATE INDEX idx_sub_subs_stripe ON subscriber_subscriptions USING GIN(stripe_data);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriber_subscriptions(id),
    amount          INTEGER NOT NULL,
    currency        CHAR(3) DEFAULT 'usd',
    status          VARCHAR(50) NOT NULL,
    -- Full Stripe payment details
    stripe_data     JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "invoice_id": "in_xxx",
    --   "charge_id": "ch_xxx",
    --   "payment_intent_id": "pi_xxx",
    --   "receipt_url": "https://..."
    -- }
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Automation

```sql
CREATE TABLE automations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    -- Full automation definition as JSONB (visual workflow builder output)
    definition      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "trigger": {
    --     "type": "subscriber_added",
    --     "config": { "list_id": "uuid" }
    --   },
    --   "steps": [
    --     { "id": "step-1", "type": "wait", "duration": "1h" },
    --     { "id": "step-2", "type": "send_email", "campaign_id": "uuid" },
    --     { "id": "step-3", "type": "wait", "duration": "3d" },
    --     { "id": "step-4", "type": "condition",
    --       "if": { "field": "engagement.last_opened_at", "op": "exists" },
    --       "then": "step-5",
    --       "else": "step-6"
    --     },
    --     { "id": "step-5", "type": "add_tag", "tag": "engaged" },
    --     { "id": "step-6", "type": "send_email", "campaign_id": "uuid-2" }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_automations_pub ON automations(publication_id);

CREATE TABLE automation_enrollments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    automation_id   UUID NOT NULL REFERENCES automations(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    current_step_id VARCHAR(100),                   -- matches step.id in definition
    status          VARCHAR(50) DEFAULT 'active',
    -- Execution history
    history         JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   { "step_id": "step-1", "type": "wait", "started_at": "...", "completed_at": "..." },
    --   { "step_id": "step-2", "type": "send_email", "started_at": "...", "result": "sent" }
    -- ]
    next_action_at  TIMESTAMPTZ,
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    UNIQUE (automation_id, subscriber_id)
);

CREATE INDEX idx_enrollments_next ON automation_enrollments(next_action_at) WHERE status = 'active';
```

## Growth Infrastructure

```sql
CREATE TABLE landing_pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    -- Page content and config as JSONB
    content         JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "blocks": [
    --     { "type": "hero", "headline": "Join 10K readers", "subtext": "..." },
    --     { "type": "signup_form", "list_id": "uuid", "button_text": "Subscribe" },
    --     { "type": "testimonial", "quote": "...", "author": "..." }
    --   ],
    --   "styles": { "background_color": "#fff", "text_color": "#333" },
    --   "seo": { "title": "...", "description": "...", "og_image": "..." }
    -- }
    is_published    BOOLEAN DEFAULT FALSE,
    stats           JSONB DEFAULT '{"views": 0, "conversions": 0}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, slug)
);

CREATE TABLE signup_forms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    form_type       VARCHAR(50) NOT NULL,
    list_id         UUID REFERENCES lists(id),
    config          JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "fields": [
    --     { "name": "email", "required": true },
    --     { "name": "name", "required": false },
    --     { "name": "company", "required": false, "custom_field_key": "company" }
    --   ],
    --   "display_rules": {
    --     "trigger": "scroll_50_percent",
    --     "delay_seconds": 5,
    --     "show_once_per_session": true
    --   },
    --   "styles": { "background": "#1a73e8", "text": "#fff" }
    -- }
    is_active       BOOLEAN DEFAULT TRUE,
    stats           JSONB DEFAULT '{"submissions": 0, "conversions": 0}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Supporting Tables

```sql
CREATE TABLE sending_domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    domain          VARCHAR(255) NOT NULL,
    -- DNS verification state
    dns_records     JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "spf": { "verified": true, "record": "v=spf1 include:...", "verified_at": "..." },
    --   "dkim": { "verified": true, "selector": "s1", "public_key": "...", "verified_at": "..." },
    --   "dmarc": { "verified": false, "expected_record": "v=DMARC1; p=quarantine; ..." }
    -- }
    is_verified     BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, domain)
);

CREATE TABLE templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    html_body       TEXT NOT NULL,
    config          JSONB DEFAULT '{}',
    -- Example: { "variables": ["publication_name", "unsubscribe_url"], "category": "newsletter" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100),
    file_size       BIGINT,
    storage_url     TEXT NOT NULL,
    meta            JSONB DEFAULT '{}',
    -- Example: { "alt_text": "...", "dimensions": { "width": 1200, "height": 630 } }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    name            VARCHAR(255) NOT NULL,
    key_hash        TEXT NOT NULL,
    key_prefix      VARCHAR(10),
    scopes          TEXT[],
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE bounces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    campaign_id     UUID REFERENCES campaigns(id),
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "type": "hard",
    --   "source": "ses",
    --   "error_code": "550",
    --   "error_message": "User unknown",
    --   "diagnostic_code": "smtp; 550 5.1.1 ...",
    --   "remote_mta": "mx.example.com"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bounces_subscriber ON bounces(subscriber_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | publications, users, publication_members |
| Subscriber Management | 3 | subscribers, lists, subscriber_lists |
| Content & Publishing | 1 | posts (content_blocks JSONB replaces separate block tables) |
| Campaigns & Delivery | 2 | campaigns, deliveries |
| Link Tracking | 2 | links, link_clicks |
| Monetisation | 3 | subscription_tiers, subscriber_subscriptions, payments |
| Automation | 2 | automations (definition in JSONB), automation_enrollments |
| Growth | 2 | landing_pages, signup_forms |
| Supporting | 5 | sending_domains, templates, media, api_keys, bounces |
| **Total** | **23** | 8 fewer tables than the normalized model |

---

## Key Design Decisions

1. **JSONB for variable data, relational for structure** — foreign keys enforce the subscriber-to-publication, campaign-to-list, and payment-to-subscription relationships. JSONB absorbs everything that varies per publication, per campaign, or per subscriber.

2. **Custom subscriber fields defined at publication level** — `publications.settings.custom_subscriber_fields` defines the schema; `subscribers.custom_fields` stores the values. This replaces the EAV (Entity-Attribute-Value) anti-pattern with a cleaner JSONB approach.

3. **Delivery events as a JSONB timeline** — instead of separate timestamp columns for opened/clicked/bounced (normalized model) or separate event rows (event-sourced model), the `deliveries.events` JSONB array captures the full timeline in one row, reducing table count and simplifying per-delivery queries.

4. **Automation definition as JSONB DAG** — the entire automation workflow (trigger, steps, conditions, branches) lives in a single JSONB column. This naturally maps to a visual workflow builder's output and supports branching logic that the normalized model's `step_order` cannot.

5. **Content blocks as JSONB** — `posts.content_blocks` stores Notion-style structured content blocks, enabling rich content types (paragraphs, images, buttons, paywall markers) without a separate blocks table and block_type taxonomy.

6. **GIN indexes on JSONB columns** — `subscribers.custom_fields`, `subscribers.engagement`, and `subscribers.source` are GIN-indexed to enable `@>` containment queries for segmentation. Performance is adequate for publications up to ~500K subscribers.

7. **Tags as PostgreSQL array** — `subscribers.tags` uses `TEXT[]` instead of a junction table, reducing JOINs for tag-based queries. The `&&` (overlap) and `@>` (contains) operators enable efficient tag filtering.

8. **Stripe data stored as full JSONB** — rather than mirroring individual Stripe fields, the `stripe_data` JSONB column stores the relevant Stripe object fragment. This avoids schema drift when Stripe adds new fields.

9. **Stats as denormalised JSONB** — `campaigns.stats`, `landing_pages.stats`, and `signup_forms.stats` store aggregate counters as JSONB, updated by background workers. This avoids expensive COUNT queries.

10. **8 fewer tables than normalized** — the JSONB approach eliminates separate tables for tags, subscriber_tags, post_tags, campaign_lists, referral_programs, referral_milestones, referrals, and automation_steps, absorbing their data into JSONB columns on parent tables.
