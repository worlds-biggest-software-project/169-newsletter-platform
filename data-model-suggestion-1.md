# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Newsletter Platform · Created: 2026-05-20

## Philosophy

This model follows classical relational database design: every domain concept gets its own table, relationships are expressed through foreign keys and junction tables, and data integrity is enforced at the database level through constraints, unique indexes, and referential integrity rules. Every field has a defined type and constraint — there are no JSONB catch-all columns.

This is the approach used by mature email platforms like Listmonk (PostgreSQL-backed, ~10 core tables) and mirrors the entity structures exposed by the Ghost Admin API and beehiiv REST API. It prioritises query predictability, strong typing, and straightforward indexing over schema flexibility.

The normalized approach works best when the domain is well-understood (newsletter publishing is a mature domain), when complex cross-entity queries are common (e.g., "find all paid subscribers in segment X who opened campaign Y but didn't click link Z"), and when regulatory compliance (GDPR erasure, CAN-SPAM audit) demands clear data lineage.

**Best for:** Teams that want maximum query flexibility, strong data integrity, and a well-understood schema that maps directly to the API surface.

**Trade-offs:**
- (+) Strong referential integrity — orphan records are impossible
- (+) Every query is a standard SQL SELECT/JOIN — no JSONB operators to learn
- (+) Schema is self-documenting — column names and constraints serve as documentation
- (+) Indexes are straightforward and predictable
- (-) Schema changes require migrations — adding a new subscriber attribute means ALTER TABLE
- (-) Higher table count increases JOIN complexity for cross-cutting queries
- (-) Custom fields per publication require either a dedicated EAV table or schema changes
- (-) Multi-tenant queries need tenant_id filtering on every query

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5321/5322 (SMTP/Email Format) | `campaigns.from_email`, `campaigns.reply_to`, `campaigns.subject` fields follow email header conventions |
| RFC 7208/6376/7489 (SPF/DKIM/DMARC) | `sending_domains` table stores DNS authentication records per domain |
| RFC 8058 (One-Click Unsubscribe) | `campaigns.list_unsubscribe_url` and `campaigns.list_unsubscribe_post` headers |
| GDPR Articles 6/7/17 | `subscribers.consent_given_at`, `subscribers.consent_source`, `subscribers.gdpr_erasure_requested_at` |
| CAN-SPAM Act | `publications.physical_address` (required in every commercial email) |
| ISO 3166-1 | `subscribers.country_code` uses ISO 3166-1 alpha-2 |
| Stripe API | `subscriptions` and `payments` tables mirror Stripe objects (customer, subscription, price, invoice) |
| RSS 2.0 / Atom (RFC 4287) | `posts` table includes `rss_guid` for feed syndication |
| OAuth 2.0 (RFC 6749) / JWT (RFC 7519) | `api_keys` and `oauth_tokens` tables for API authentication |

---

## Core Identity & Multi-Tenancy

```sql
-- Each publication is a tenant. All subscriber/content data is scoped to a publication.

CREATE TABLE publications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    logo_url        TEXT,
    website_url     TEXT,
    physical_address TEXT,                  -- CAN-SPAM requirement
    default_from_name VARCHAR(255),
    default_from_email VARCHAR(255),
    timezone        VARCHAR(50) DEFAULT 'UTC',
    language        VARCHAR(10) DEFAULT 'en',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   TEXT,
    name            VARCHAR(255),
    avatar_url      TEXT,
    email_verified  BOOLEAN DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE publication_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'editor',  -- owner, admin, editor, author
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, user_id)
);

CREATE INDEX idx_pub_members_pub ON publication_members(publication_id);
CREATE INDEX idx_pub_members_user ON publication_members(user_id);
```

## Email Infrastructure

```sql
CREATE TABLE sending_domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    domain          VARCHAR(255) NOT NULL,
    spf_verified    BOOLEAN DEFAULT FALSE,
    dkim_verified   BOOLEAN DEFAULT FALSE,
    dmarc_verified  BOOLEAN DEFAULT FALSE,
    dkim_selector   VARCHAR(255),
    dkim_public_key TEXT,
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, domain)
);

CREATE TABLE templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    html_body       TEXT NOT NULL,
    plain_text_body TEXT,
    is_default      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_templates_pub ON templates(publication_id);
```

## Subscriber Management

```sql
CREATE TYPE subscriber_status AS ENUM ('enabled', 'disabled', 'blocklisted');
CREATE TYPE subscription_status AS ENUM ('unconfirmed', 'confirmed', 'unsubscribed');

CREATE TABLE subscribers (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id          UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    email                   VARCHAR(255) NOT NULL,
    name                    VARCHAR(255),
    status                  subscriber_status NOT NULL DEFAULT 'enabled',
    country_code            CHAR(2),                -- ISO 3166-1 alpha-2
    source                  VARCHAR(100),            -- signup_form, import, api, referral
    referral_source         VARCHAR(255),            -- UTM source or referring subscriber
    consent_given_at        TIMESTAMPTZ,             -- GDPR Article 7
    consent_source          VARCHAR(255),            -- which form/page collected consent
    consent_ip              INET,
    double_optin_confirmed  BOOLEAN DEFAULT FALSE,
    gdpr_erasure_requested_at TIMESTAMPTZ,
    subscribed_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    unsubscribed_at         TIMESTAMPTZ,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, email)
);

CREATE INDEX idx_subscribers_pub_status ON subscribers(publication_id, status);
CREATE INDEX idx_subscribers_email ON subscribers(email);
CREATE INDEX idx_subscribers_created ON subscribers(publication_id, created_at);

CREATE TABLE lists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) DEFAULT 'public',  -- public, private
    subscriber_count INTEGER DEFAULT 0,             -- denormalised counter
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_lists_pub ON lists(publication_id);

CREATE TABLE subscriber_lists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    list_id         UUID NOT NULL REFERENCES lists(id) ON DELETE CASCADE,
    status          subscription_status NOT NULL DEFAULT 'unconfirmed',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (subscriber_id, list_id)
);

CREATE INDEX idx_sub_lists_list ON subscriber_lists(list_id);
CREATE INDEX idx_sub_lists_sub ON subscriber_lists(subscriber_id);

-- Tag-based segmentation
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, name)
);

CREATE TABLE subscriber_tags (
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (subscriber_id, tag_id)
);

CREATE INDEX idx_sub_tags_tag ON subscriber_tags(tag_id);
```

## Content & Publishing

```sql
CREATE TYPE post_status AS ENUM ('draft', 'scheduled', 'published', 'archived');
CREATE TYPE post_visibility AS ENUM ('public', 'members', 'paid');

CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    author_id       UUID REFERENCES users(id),
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(500) NOT NULL,
    html_content    TEXT,
    markdown_content TEXT,
    plain_text      TEXT,                   -- extracted for search/AI
    excerpt         TEXT,
    featured_image  TEXT,
    status          post_status NOT NULL DEFAULT 'draft',
    visibility      post_visibility NOT NULL DEFAULT 'public',
    min_paid_tier_id UUID,                  -- gate behind a specific tier
    rss_guid        VARCHAR(255),           -- RSS 2.0 feed guid
    published_at    TIMESTAMPTZ,
    scheduled_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, slug)
);

CREATE INDEX idx_posts_pub_status ON posts(publication_id, status);
CREATE INDEX idx_posts_published ON posts(publication_id, published_at DESC);

CREATE TABLE post_tags (
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);
```

## Campaign & Email Delivery

```sql
CREATE TYPE campaign_status AS ENUM ('draft', 'scheduled', 'sending', 'sent', 'paused', 'cancelled');

CREATE TABLE campaigns (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id      UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    post_id             UUID REFERENCES posts(id),          -- optional link to a post
    template_id         UUID REFERENCES templates(id),
    name                VARCHAR(255) NOT NULL,
    subject             VARCHAR(500) NOT NULL,
    preview_text        VARCHAR(255),
    from_name           VARCHAR(255),
    from_email          VARCHAR(255),
    reply_to            VARCHAR(255),
    html_body           TEXT,
    plain_text_body     TEXT,
    status              campaign_status NOT NULL DEFAULT 'draft',
    list_unsubscribe_url TEXT,                               -- RFC 8058
    list_unsubscribe_post TEXT,
    send_at             TIMESTAMPTZ,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    total_recipients    INTEGER DEFAULT 0,
    total_sent          INTEGER DEFAULT 0,
    total_delivered     INTEGER DEFAULT 0,
    total_opened        INTEGER DEFAULT 0,
    total_clicked       INTEGER DEFAULT 0,
    total_bounced       INTEGER DEFAULT 0,
    total_complained    INTEGER DEFAULT 0,
    total_unsubscribed  INTEGER DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_campaigns_pub ON campaigns(publication_id);
CREATE INDEX idx_campaigns_status ON campaigns(publication_id, status);

-- Junction: which lists receive a campaign
CREATE TABLE campaign_lists (
    campaign_id     UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    list_id         UUID NOT NULL REFERENCES lists(id) ON DELETE CASCADE,
    PRIMARY KEY (campaign_id, list_id)
);

-- Individual send records
CREATE TABLE campaign_sends (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',  -- queued, sent, delivered, bounced, failed
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    opened_at       TIMESTAMPTZ,
    clicked_at      TIMESTAMPTZ,
    bounced_at      TIMESTAMPTZ,
    bounce_type     VARCHAR(50),                            -- hard, soft
    complained_at   TIMESTAMPTZ,
    unsubscribed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sends_campaign ON campaign_sends(campaign_id);
CREATE INDEX idx_sends_subscriber ON campaign_sends(subscriber_id);
CREATE INDEX idx_sends_status ON campaign_sends(campaign_id, status);
```

## Link Tracking

```sql
CREATE TABLE links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    original_url    TEXT NOT NULL,
    tracking_url    TEXT NOT NULL,
    click_count     INTEGER DEFAULT 0,    -- denormalised counter
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_links_campaign ON links(campaign_id);

CREATE TABLE link_clicks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    link_id         UUID NOT NULL REFERENCES links(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    campaign_id     UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    ip_address      INET,
    user_agent      TEXT,
    clicked_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clicks_link ON link_clicks(link_id);
CREATE INDEX idx_clicks_subscriber ON link_clicks(subscriber_id);
CREATE INDEX idx_clicks_campaign_date ON link_clicks(campaign_id, clicked_at);
```

## Monetisation (Stripe-Aligned)

```sql
CREATE TABLE subscription_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,          -- e.g., "Free", "Premium", "Founding"
    description     TEXT,
    price_monthly   INTEGER,                        -- cents (Stripe convention)
    price_yearly    INTEGER,                        -- cents
    currency        CHAR(3) DEFAULT 'usd',          -- ISO 4217
    stripe_product_id VARCHAR(255),
    stripe_price_monthly_id VARCHAR(255),
    stripe_price_yearly_id VARCHAR(255),
    is_active       BOOLEAN DEFAULT TRUE,
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tiers_pub ON subscription_tiers(publication_id);

CREATE TABLE subscriber_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    tier_id         UUID NOT NULL REFERENCES subscription_tiers(id),
    stripe_customer_id VARCHAR(255),
    stripe_subscription_id VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',  -- active, past_due, cancelled, trialing
    current_period_start TIMESTAMPTZ,
    current_period_end   TIMESTAMPTZ,
    cancel_at_period_end BOOLEAN DEFAULT FALSE,
    cancelled_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sub_subs_subscriber ON subscriber_subscriptions(subscriber_id);
CREATE INDEX idx_sub_subs_stripe ON subscriber_subscriptions(stripe_subscription_id);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriber_subscriptions(id),
    stripe_invoice_id VARCHAR(255),
    stripe_charge_id VARCHAR(255),
    amount          INTEGER NOT NULL,               -- cents
    currency        CHAR(3) DEFAULT 'usd',
    status          VARCHAR(50) NOT NULL,            -- succeeded, failed, refunded
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_sub ON payments(subscription_id);
```

## Automation

```sql
CREATE TYPE automation_status AS ENUM ('active', 'paused', 'draft');
CREATE TYPE automation_trigger AS ENUM ('subscriber_added', 'tag_added', 'list_joined', 'subscription_started', 'date_based', 'api');

CREATE TABLE automations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    trigger_type    automation_trigger NOT NULL,
    trigger_config  TEXT,                           -- e.g., list_id or tag_id for the trigger
    status          automation_status NOT NULL DEFAULT 'draft',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_automations_pub ON automations(publication_id);

CREATE TABLE automation_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    automation_id   UUID NOT NULL REFERENCES automations(id) ON DELETE CASCADE,
    step_order      INTEGER NOT NULL,
    step_type       VARCHAR(50) NOT NULL,           -- send_email, wait, condition, add_tag, remove_tag
    campaign_id     UUID REFERENCES campaigns(id),  -- for send_email steps
    wait_duration   INTERVAL,                       -- for wait steps
    condition_field VARCHAR(255),                    -- for condition steps
    condition_op    VARCHAR(50),
    condition_value TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_auto_steps ON automation_steps(automation_id, step_order);

CREATE TABLE automation_enrollments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    automation_id   UUID NOT NULL REFERENCES automations(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    current_step    INTEGER DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',  -- active, completed, exited
    next_action_at  TIMESTAMPTZ,
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    UNIQUE (automation_id, subscriber_id)
);

CREATE INDEX idx_enrollments_next ON automation_enrollments(next_action_at) WHERE status = 'active';
```

## Growth & Referrals

```sql
CREATE TABLE referral_programs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE referral_milestones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id      UUID NOT NULL REFERENCES referral_programs(id) ON DELETE CASCADE,
    referrals_required INTEGER NOT NULL,
    reward_description TEXT NOT NULL,
    reward_type     VARCHAR(50),            -- digital_product, tier_upgrade, custom
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE referrals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id      UUID NOT NULL REFERENCES referral_programs(id) ON DELETE CASCADE,
    referrer_id     UUID NOT NULL REFERENCES subscribers(id),
    referred_id     UUID NOT NULL REFERENCES subscribers(id),
    status          VARCHAR(50) DEFAULT 'pending',  -- pending, confirmed, rewarded
    confirmed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (program_id, referred_id)
);

CREATE INDEX idx_referrals_referrer ON referrals(referrer_id);
```

## Landing Pages & Forms

```sql
CREATE TABLE landing_pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    html_content    TEXT,
    is_published    BOOLEAN DEFAULT FALSE,
    list_id         UUID REFERENCES lists(id),       -- subscribers go to this list
    view_count      INTEGER DEFAULT 0,
    conversion_count INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, slug)
);

CREATE TABLE signup_forms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    form_type       VARCHAR(50) NOT NULL,           -- inline, popup, slide_in, full_page
    list_id         UUID REFERENCES lists(id),
    html_snippet    TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    submission_count INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Media & Assets

```sql
CREATE TABLE media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100),
    file_size       BIGINT,
    storage_url     TEXT NOT NULL,
    alt_text        TEXT,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_media_pub ON media(publication_id);
```

## API Access

```sql
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    name            VARCHAR(255) NOT NULL,
    key_hash        TEXT NOT NULL,                   -- bcrypt hash of the API key
    key_prefix      VARCHAR(10),                     -- first few chars for identification
    scopes          TEXT[],                           -- e.g., {'subscribers:read', 'posts:write'}
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
```

## Bounces

```sql
CREATE TABLE bounces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    campaign_id     UUID REFERENCES campaigns(id),
    type            VARCHAR(50) NOT NULL,           -- hard, soft, complaint
    source          VARCHAR(100),                   -- ses, sendgrid, postmark
    error_code      VARCHAR(50),
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bounces_subscriber ON bounces(subscriber_id);
CREATE INDEX idx_bounces_campaign ON bounces(campaign_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | publications, users, publication_members |
| Email Infrastructure | 2 | sending_domains, templates |
| Subscriber Management | 5 | subscribers, lists, subscriber_lists, tags, subscriber_tags |
| Content & Publishing | 2 | posts, post_tags |
| Campaign & Delivery | 3 | campaigns, campaign_lists, campaign_sends |
| Link Tracking | 2 | links, link_clicks |
| Monetisation | 3 | subscription_tiers, subscriber_subscriptions, payments |
| Automation | 3 | automations, automation_steps, automation_enrollments |
| Growth & Referrals | 3 | referral_programs, referral_milestones, referrals |
| Landing Pages & Forms | 2 | landing_pages, signup_forms |
| Media & Assets | 1 | media |
| API Access | 1 | api_keys |
| Bounces | 1 | bounces |
| **Total** | **31** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe for API exposure, and avoids sequential enumeration attacks.

2. **Publication as tenant boundary** — every subscriber-facing table has a `publication_id` foreign key; Row-Level Security policies can enforce tenant isolation at the database level.

3. **Subscriber scoped per publication** — a person's email can exist in multiple publications as separate subscriber records (`UNIQUE (publication_id, email)`), matching how beehiiv and Ghost model subscribers.

4. **Denormalised counters on campaigns and lists** — `total_opened`, `total_clicked`, `subscriber_count` avoid expensive COUNT queries on high-volume tables; updated via triggers or application-level increments.

5. **Stripe object IDs stored as strings** — `stripe_customer_id`, `stripe_subscription_id`, `stripe_price_id` allow the platform to sync with Stripe without duplicating Stripe's schema; Stripe remains the source of truth for billing.

6. **GDPR consent fields on subscriber** — `consent_given_at`, `consent_source`, `consent_ip`, and `gdpr_erasure_requested_at` enable compliance auditing and right-to-erasure workflows.

7. **Campaign sends as a separate table** — one row per subscriber per campaign enables per-recipient delivery tracking without bloating the campaigns table; this mirrors Listmonk's architecture.

8. **Automation as a step-based sequence** — `automation_steps` with `step_order` supports linear sequences (send, wait, condition, tag); more complex branching would require a DAG model in a future iteration.

9. **Tag-based segmentation** — tags are first-class entities with a junction table, enabling both manual and automated segmentation; more flexible than Listmonk's subscriber-list-only model.

10. **Referral tracking with milestones** — supports beehiiv-style referral programs where subscribers earn rewards at defined referral thresholds.
