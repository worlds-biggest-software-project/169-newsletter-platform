# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Newsletter Platform · Created: 2026-05-20

## Philosophy

This model treats every meaningful action in the system as an immutable event written to an append-only event store. The event store is the single source of truth. Read-optimised materialised views (projections) are derived from the event stream and serve the API and UI. This is the Command Query Responsibility Segregation (CQRS) pattern combined with event sourcing.

Newsletter platforms have a natural affinity for event sourcing: subscriber engagement is inherently a stream of events (opened, clicked, scrolled, unsubscribed), email delivery is a pipeline of state transitions (queued, sent, delivered, bounced), and GDPR/CAN-SPAM compliance requires an immutable audit trail of consent. Platforms like Segment, Amplitude, and PostHog use event-based architectures for the same reasons — every user action is an event that can be replayed, aggregated, or fed into ML models.

The core advantage is temporal querying: "What segments did this subscriber belong to on March 15?" or "What was the open rate trend for this campaign hour by hour?" are first-class queries, not afterthoughts. The trade-off is increased complexity — every read path requires a projection, and the event store grows continuously.

**Best for:** Teams building AI-powered analytics (churn prediction, engagement scoring, send-time optimisation) that need rich historical event data as training input, and teams in regulated environments requiring immutable audit trails.

**Trade-offs:**
- (+) Complete audit trail — every state change is recorded with timestamp, actor, and payload
- (+) Temporal queries are trivial — replay events to reconstruct state at any point in time
- (+) ML/AI training data is a natural byproduct — engagement events feed directly into models
- (+) GDPR compliance is explicit — consent events form an immutable chain
- (+) Decoupled read/write paths scale independently
- (-) Higher storage requirements — events accumulate forever (snapshots mitigate)
- (-) Eventual consistency between event store and projections — reads may lag writes
- (-) More complex application code — every write is "append event + update projection"
- (-) Projection rebuild can be slow for large event stores without snapshots
- (-) Debugging requires understanding event replay, not just current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5321/5322 (SMTP) | Delivery events reference email headers (from, to, subject, message-id) |
| RFC 7208/6376/7489 (SPF/DKIM/DMARC) | `domain.verified` events record DNS authentication state changes |
| RFC 8058 (One-Click Unsubscribe) | `subscriber.unsubscribed` events capture unsubscribe method (one-click, link, manual) |
| GDPR Articles 6/7/17 | Consent events (`subscriber.consent_given`, `subscriber.consent_withdrawn`, `subscriber.erasure_requested`) form an immutable audit chain |
| CAN-SPAM Act | `campaign.sent` events include physical address compliance metadata |
| Stripe Webhooks | Stripe webhook payloads are stored as events (`payment.succeeded`, `subscription.updated`) |
| OCSF (Open Cybersecurity Schema Framework) | Event schema influenced by OCSF's structured event format for audit logging |

---

## Event Store

```sql
-- The event store is the single source of truth.
-- All state is derived from replaying events.

CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                  -- aggregate root ID (subscriber, campaign, etc.)
    stream_type     VARCHAR(100) NOT NULL,           -- 'subscriber', 'campaign', 'post', 'publication'
    event_type      VARCHAR(200) NOT NULL,           -- e.g., 'subscriber.created', 'campaign.sent'
    event_version   INTEGER NOT NULL DEFAULT 1,      -- schema version for this event type
    sequence_number BIGINT NOT NULL,                 -- per-stream ordering
    payload         JSONB NOT NULL,                  -- event-specific data
    metadata        JSONB DEFAULT '{}',              -- actor_id, ip_address, user_agent, request_id
    publication_id  UUID NOT NULL,                   -- tenant partition key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (stream_id, sequence_number)
);

-- Primary query: replay a stream
CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);

-- Query by type within a publication (for projections)
CREATE INDEX idx_events_pub_type ON events(publication_id, event_type, created_at);

-- Time-range queries for analytics
CREATE INDEX idx_events_pub_time ON events(publication_id, created_at);

-- Partition by month for performance (optional but recommended)
-- CREATE TABLE events PARTITION BY RANGE (created_at);
```

### Event Type Catalogue

```
-- Subscriber lifecycle
subscriber.created          { email, source, consent_source, consent_ip }
subscriber.confirmed        { confirmation_method }  -- double opt-in
subscriber.updated          { changed_fields: { name: { old, new } } }
subscriber.tagged           { tag_id, tag_name }
subscriber.untagged         { tag_id, tag_name }
subscriber.list_joined      { list_id, list_name }
subscriber.list_left        { list_id, list_name, reason }
subscriber.unsubscribed     { method, reason }       -- one_click, link, manual, complaint
subscriber.resubscribed     { }
subscriber.blocklisted      { reason }
subscriber.consent_given    { source, ip, form_id }
subscriber.consent_withdrawn { method }
subscriber.erasure_requested { }
subscriber.erasure_completed { fields_erased }

-- Campaign lifecycle
campaign.created            { name, subject, from_email }
campaign.scheduled          { send_at }
campaign.started            { recipient_count }
campaign.paused             { }
campaign.resumed            { }
campaign.completed          { stats: { sent, delivered, bounced } }
campaign.cancelled          { reason }

-- Delivery events (one per recipient)
delivery.queued             { subscriber_id, campaign_id }
delivery.sent               { subscriber_id, campaign_id, message_id }
delivery.delivered          { subscriber_id, campaign_id }
delivery.opened             { subscriber_id, campaign_id, ip, user_agent }
delivery.clicked            { subscriber_id, campaign_id, link_url, ip }
delivery.bounced            { subscriber_id, campaign_id, bounce_type, error }
delivery.complained         { subscriber_id, campaign_id }

-- Payment events (from Stripe webhooks)
payment.subscription_created  { subscriber_id, tier_id, stripe_subscription_id }
payment.invoice_paid          { subscriber_id, amount, currency, stripe_invoice_id }
payment.invoice_failed        { subscriber_id, amount, error }
payment.subscription_cancelled { subscriber_id, reason }
payment.refund_issued         { subscriber_id, amount }

-- Post lifecycle
post.created                { title, author_id }
post.updated                { changed_fields }
post.published              { visibility, published_at }
post.archived               { }

-- Automation events
automation.enrollment_started   { subscriber_id, automation_id }
automation.step_executed        { subscriber_id, automation_id, step_id, step_type }
automation.enrollment_completed { subscriber_id, automation_id }
automation.enrollment_exited    { subscriber_id, automation_id, reason }
```

## Snapshot Store

```sql
-- Snapshots accelerate projection rebuilds by caching aggregate state periodically.

CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    sequence_number BIGINT NOT NULL,                -- snapshot taken at this sequence
    state           JSONB NOT NULL,                  -- full aggregate state at this point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (stream_id, sequence_number)
);
```

## Materialised Read Models (Projections)

These tables are derived from the event store. They can be rebuilt at any time by replaying events.

### Subscriber Projection

```sql
CREATE TABLE v_subscribers (
    id              UUID PRIMARY KEY,
    publication_id  UUID NOT NULL,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255),
    status          VARCHAR(50) NOT NULL,
    tags            TEXT[] DEFAULT '{}',              -- denormalised tag names
    lists           TEXT[] DEFAULT '{}',              -- denormalised list names
    tier_name       VARCHAR(255),                    -- current paid tier (NULL = free)
    source          VARCHAR(100),
    country_code    CHAR(2),
    total_emails_received  INTEGER DEFAULT 0,
    total_opens     INTEGER DEFAULT 0,
    total_clicks    INTEGER DEFAULT 0,
    last_opened_at  TIMESTAMPTZ,
    last_clicked_at TIMESTAMPTZ,
    engagement_score FLOAT,                          -- computed from event history
    consent_given_at TIMESTAMPTZ,
    subscribed_at   TIMESTAMPTZ,
    unsubscribed_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (publication_id, email)
);

CREATE INDEX idx_v_subs_pub_status ON v_subscribers(publication_id, status);
CREATE INDEX idx_v_subs_engagement ON v_subscribers(publication_id, engagement_score DESC);
CREATE INDEX idx_v_subs_tags ON v_subscribers USING GIN(tags);
```

### Campaign Stats Projection

```sql
CREATE TABLE v_campaign_stats (
    campaign_id     UUID PRIMARY KEY,
    publication_id  UUID NOT NULL,
    name            VARCHAR(255),
    subject         VARCHAR(500),
    status          VARCHAR(50),
    total_recipients INTEGER DEFAULT 0,
    total_sent      INTEGER DEFAULT 0,
    total_delivered  INTEGER DEFAULT 0,
    total_opened    INTEGER DEFAULT 0,
    total_unique_opens INTEGER DEFAULT 0,
    total_clicked   INTEGER DEFAULT 0,
    total_unique_clicks INTEGER DEFAULT 0,
    total_bounced   INTEGER DEFAULT 0,
    total_complained INTEGER DEFAULT 0,
    total_unsubscribed INTEGER DEFAULT 0,
    open_rate       FLOAT,
    click_rate      FLOAT,
    bounce_rate     FLOAT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_camp_pub ON v_campaign_stats(publication_id);
```

### Hourly Engagement Projection (Time-Series)

```sql
CREATE TABLE v_engagement_hourly (
    publication_id  UUID NOT NULL,
    campaign_id     UUID,                            -- NULL for aggregate publication stats
    hour            TIMESTAMPTZ NOT NULL,            -- truncated to hour
    opens           INTEGER DEFAULT 0,
    unique_opens    INTEGER DEFAULT 0,
    clicks          INTEGER DEFAULT 0,
    unique_clicks   INTEGER DEFAULT 0,
    unsubscribes    INTEGER DEFAULT 0,
    bounces         INTEGER DEFAULT 0,
    PRIMARY KEY (publication_id, hour, campaign_id)
);

-- Efficient time-range queries for analytics dashboards
CREATE INDEX idx_v_eng_range ON v_engagement_hourly(publication_id, hour DESC);
```

### Subscriber Growth Projection

```sql
CREATE TABLE v_subscriber_growth_daily (
    publication_id  UUID NOT NULL,
    date            DATE NOT NULL,
    new_subscribers INTEGER DEFAULT 0,
    unsubscribes    INTEGER DEFAULT 0,
    net_growth      INTEGER DEFAULT 0,
    total_active    INTEGER DEFAULT 0,              -- running total
    total_paid      INTEGER DEFAULT 0,
    PRIMARY KEY (publication_id, date)
);
```

### Revenue Projection

```sql
CREATE TABLE v_revenue_monthly (
    publication_id  UUID NOT NULL,
    month           DATE NOT NULL,                  -- first of month
    mrr             INTEGER DEFAULT 0,              -- monthly recurring revenue in cents
    new_mrr         INTEGER DEFAULT 0,
    churned_mrr     INTEGER DEFAULT 0,
    total_payments  INTEGER DEFAULT 0,
    total_revenue   INTEGER DEFAULT 0,              -- cents
    paying_subscribers INTEGER DEFAULT 0,
    PRIMARY KEY (publication_id, month)
);
```

## Operational Tables (Not Event-Sourced)

Some tables hold configuration/reference data that doesn't benefit from event sourcing.

```sql
CREATE TABLE publications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    physical_address TEXT,
    default_from_email VARCHAR(255),
    timezone        VARCHAR(50) DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   TEXT,
    name            VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE publication_members (
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'editor',
    PRIMARY KEY (publication_id, user_id)
);

CREATE TABLE templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id),
    name            VARCHAR(255) NOT NULL,
    html_body       TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE sending_domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id),
    domain          VARCHAR(255) NOT NULL,
    spf_verified    BOOLEAN DEFAULT FALSE,
    dkim_verified   BOOLEAN DEFAULT FALSE,
    dmarc_verified  BOOLEAN DEFAULT FALSE,
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, domain)
);

CREATE TABLE subscription_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id),
    name            VARCHAR(255) NOT NULL,
    price_monthly   INTEGER,
    price_yearly    INTEGER,
    currency        CHAR(3) DEFAULT 'usd',
    stripe_product_id VARCHAR(255),
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Example Queries

### Replay subscriber state at a point in time

```sql
-- "What lists was subscriber X on as of March 15, 2026?"
SELECT
    e.event_type,
    e.payload->>'list_id' AS list_id,
    e.payload->>'list_name' AS list_name,
    e.created_at
FROM events e
WHERE e.stream_id = '{{subscriber_id}}'
  AND e.stream_type = 'subscriber'
  AND e.event_type IN ('subscriber.list_joined', 'subscriber.list_left')
  AND e.created_at <= '2026-03-15T23:59:59Z'
ORDER BY e.sequence_number;
```

### Calculate engagement score from events

```sql
-- Engagement score = weighted sum of recent activity
SELECT
    stream_id AS subscriber_id,
    SUM(CASE
        WHEN event_type = 'delivery.opened' AND created_at > NOW() - INTERVAL '30 days' THEN 1
        WHEN event_type = 'delivery.clicked' AND created_at > NOW() - INTERVAL '30 days' THEN 3
        WHEN event_type = 'delivery.opened' AND created_at > NOW() - INTERVAL '90 days' THEN 0.5
        WHEN event_type = 'delivery.clicked' AND created_at > NOW() - INTERVAL '90 days' THEN 1.5
        ELSE 0
    END) AS engagement_score
FROM events
WHERE publication_id = '{{pub_id}}'
  AND event_type IN ('delivery.opened', 'delivery.clicked')
GROUP BY stream_id;
```

### GDPR consent audit trail

```sql
-- Complete consent history for a subscriber (GDPR Article 7 compliance)
SELECT
    event_type,
    payload->>'source' AS consent_source,
    metadata->>'ip_address' AS ip,
    created_at
FROM events
WHERE stream_id = '{{subscriber_id}}'
  AND event_type LIKE 'subscriber.consent%'
ORDER BY sequence_number;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 2 | events, snapshots |
| Projections — Subscribers | 1 | v_subscribers |
| Projections — Campaigns | 1 | v_campaign_stats |
| Projections — Analytics | 3 | v_engagement_hourly, v_subscriber_growth_daily, v_revenue_monthly |
| Operational — Config | 6 | publications, users, publication_members, templates, sending_domains, subscription_tiers |
| **Total** | **13** | Plus the event type catalogue (application-level, not tables) |

---

## Key Design Decisions

1. **Single event store table** — all events across all aggregate types go into one `events` table partitioned by `created_at`. This simplifies infrastructure and enables cross-aggregate queries (e.g., correlating subscriber events with payment events).

2. **JSONB payload with versioned schema** — `event_version` allows evolving event payloads without breaking replay; old events are replayed with version-aware deserialisation.

3. **Stream-based ordering** — `sequence_number` per `stream_id` ensures causal ordering within an aggregate while allowing concurrent writes across aggregates.

4. **Projections are disposable** — every `v_*` table can be dropped and rebuilt from the event store. This means schema changes to read models are zero-downtime: create new projection, replay, swap.

5. **Operational tables for config** — publications, users, templates, and sending domains are not event-sourced because they are low-volume configuration data where current state is sufficient.

6. **Delivery events carry subscriber_id in payload** — rather than creating a separate stream per delivery, delivery events are written to the subscriber's stream, enabling "show me everything that happened to this subscriber" with a single stream replay.

7. **Hourly engagement projection** — pre-aggregated hourly stats avoid scanning millions of delivery events for dashboard rendering; the projection handler increments counters as events arrive.

8. **Stripe webhook events stored as-is** — `payment.*` events store the Stripe payload in the event's `payload` field, creating a Stripe audit trail independent of Stripe's own event retention.

9. **Metadata captures actor and context** — every event records who caused it (`actor_id`), from where (`ip_address`), and the request correlation ID, enabling full forensic analysis.

10. **Engagement score computed from events** — rather than a magic number in a column, engagement scores are derived from weighted event replay, making the scoring algorithm transparent and adjustable.
