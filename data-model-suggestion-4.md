# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Newsletter Platform · Created: 2026-05-20

## Philosophy

This model adds a property graph layer on top of a relational core to handle the relationship-intensive features that differentiate modern newsletter platforms from simple email blasters: cross-newsletter recommendations, audience overlap analysis, referral networks, content similarity, and subscriber interest graphs. The relational tables handle operational CRUD (sending emails, processing payments, managing content), while the graph layer handles discovery, recommendation, and network analysis queries.

Newsletter platforms are increasingly competing on network effects. Substack's discovery network drove 32 million new subscriptions in 2025. beehiiv's Boosts marketplace connects publishers for cross-promotion based on audience overlap. Churn prediction, content recommendations, and smart segmentation all require understanding relationships between subscribers, content, and publications — relationships that are expensive to query with JOINs but natural in a graph.

This model uses PostgreSQL as the primary store with `graph_nodes` and `graph_edges` tables that form a lightweight property graph. For teams that outgrow this, the graph layer can be migrated to a dedicated graph database (Neo4j, Amazon Neptune) without changing the relational core. The graph is a secondary read model — the relational tables remain the source of truth.

**Best for:** Platforms that want to compete on discovery, cross-newsletter recommendations, referral networks, and AI-powered audience insights — features that require multi-hop relationship traversal.

**Trade-offs:**
- (+) Multi-hop relationship queries are natural — "find newsletters read by subscribers who also read my newsletter"
- (+) Audience overlap analysis is a simple graph query, not an expensive self-JOIN
- (+) Referral network visualisation is built into the data model
- (+) Content recommendation via shared-audience similarity is efficient
- (+) Graph layer is a secondary index — can be rebuilt from relational source of truth
- (-) Two data models to maintain — relational core plus graph sync
- (-) Graph queries require recursive CTEs or graph-specific query language
- (-) Higher write amplification — every relational change must also update the graph
- (-) More complex deployment if using an external graph database
- (-) Team must understand both relational and graph query patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5321/5322 (SMTP) | Campaign/delivery tables follow standard email conventions |
| RFC 7208/6376/7489 (SPF/DKIM/DMARC) | Sending domain verification in relational tables |
| RFC 8058 (One-Click Unsubscribe) | Campaign headers in relational campaigns table |
| GDPR Articles 6/7/17 | Consent tracking on subscriber records; graph edges carry consent metadata |
| Stripe API | Payment tables mirror Stripe objects |
| W3C RDF / Property Graph Model | Graph layer follows the Labeled Property Graph model (nodes with labels and properties, edges with types and properties) |
| ISO 3166-1 | Subscriber country codes for geographic graph analysis |

---

## Relational Core (Operational Tables)

The relational core is streamlined — it handles the operational workload (sending, billing, content management) and serves as the source of truth.

```sql
CREATE TABLE publications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    logo_url        TEXT,
    physical_address TEXT,
    default_from_name VARCHAR(255),
    default_from_email VARCHAR(255),
    timezone        VARCHAR(50) DEFAULT 'UTC',
    category        VARCHAR(100),                   -- tech, finance, culture, politics, etc.
    topics          TEXT[] DEFAULT '{}',             -- topic tags for graph matching
    subscriber_count INTEGER DEFAULT 0,
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

CREATE TYPE subscriber_status AS ENUM ('enabled', 'disabled', 'blocklisted');

CREATE TABLE subscribers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255),
    status          subscriber_status NOT NULL DEFAULT 'enabled',
    country_code    CHAR(2),
    source          VARCHAR(100),
    tier_id         UUID,
    consent_given_at TIMESTAMPTZ,
    consent_source  VARCHAR(255),
    tags            TEXT[] DEFAULT '{}',
    engagement_score FLOAT DEFAULT 0,
    subscribed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    unsubscribed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, email)
);

CREATE INDEX idx_subs_pub_status ON subscribers(publication_id, status);
CREATE INDEX idx_subs_email ON subscribers(email);
CREATE INDEX idx_subs_tags ON subscribers USING GIN(tags);
CREATE INDEX idx_subs_engagement ON subscribers(publication_id, engagement_score DESC);

CREATE TABLE lists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
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

CREATE TYPE post_status AS ENUM ('draft', 'scheduled', 'published', 'archived');

CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    author_id       UUID REFERENCES users(id),
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(500) NOT NULL,
    html_content    TEXT,
    plain_text      TEXT,
    excerpt         TEXT,
    status          post_status NOT NULL DEFAULT 'draft',
    visibility      VARCHAR(50) DEFAULT 'public',
    topics          TEXT[] DEFAULT '{}',             -- topic tags for content graph
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, slug)
);

CREATE INDEX idx_posts_pub_status ON posts(publication_id, status);
CREATE INDEX idx_posts_topics ON posts USING GIN(topics);

CREATE TYPE campaign_status AS ENUM ('draft', 'scheduled', 'sending', 'sent', 'paused', 'cancelled');

CREATE TABLE campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    post_id         UUID REFERENCES posts(id),
    name            VARCHAR(255) NOT NULL,
    subject         VARCHAR(500) NOT NULL,
    from_email      VARCHAR(255),
    status          campaign_status NOT NULL DEFAULT 'draft',
    total_sent      INTEGER DEFAULT 0,
    total_opened    INTEGER DEFAULT 0,
    total_clicked   INTEGER DEFAULT 0,
    send_at         TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',
    sent_at         TIMESTAMPTZ,
    opened_at       TIMESTAMPTZ,
    clicked_at      TIMESTAMPTZ,
    bounced_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_deliveries_campaign ON deliveries(campaign_id);
CREATE INDEX idx_deliveries_subscriber ON deliveries(subscriber_id);

CREATE TABLE subscription_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    price_monthly   INTEGER,
    price_yearly    INTEGER,
    currency        CHAR(3) DEFAULT 'usd',
    stripe_product_id VARCHAR(255),
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE subscriber_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
    tier_id         UUID NOT NULL REFERENCES subscription_tiers(id),
    stripe_subscription_id VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    current_period_end TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriber_subscriptions(id),
    amount          INTEGER NOT NULL,
    currency        CHAR(3) DEFAULT 'usd',
    status          VARCHAR(50) NOT NULL,
    stripe_invoice_id VARCHAR(255),
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE sending_domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id) ON DELETE CASCADE,
    domain          VARCHAR(255) NOT NULL,
    spf_verified    BOOLEAN DEFAULT FALSE,
    dkim_verified   BOOLEAN DEFAULT FALSE,
    dmarc_verified  BOOLEAN DEFAULT FALSE,
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (publication_id, domain)
);

CREATE TABLE templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id),
    name            VARCHAR(255) NOT NULL,
    html_body       TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id),
    filename        VARCHAR(500) NOT NULL,
    storage_url     TEXT NOT NULL,
    content_type    VARCHAR(100),
    file_size       BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publication_id  UUID NOT NULL REFERENCES publications(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    name            VARCHAR(255) NOT NULL,
    key_hash        TEXT NOT NULL,
    key_prefix      VARCHAR(10),
    scopes          TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Graph Layer

The graph layer is a lightweight property graph implemented in PostgreSQL. Nodes represent entities (publications, subscribers, posts, topics). Edges represent relationships (subscribes_to, read, clicked, referred, similar_to, covers_topic).

```sql
-- Graph nodes reference entities in the relational tables.
-- The node table enables cross-entity graph queries without knowing the source table.

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY,               -- same UUID as the relational entity
    node_type       VARCHAR(50) NOT NULL,            -- publication, subscriber, post, topic, email_address
    publication_id  UUID,                            -- tenant scope (NULL for cross-tenant nodes like email_address, topic)
    label           VARCHAR(500),                    -- human-readable label (publication name, email, post title)
    properties      JSONB DEFAULT '{}',              -- additional node properties
    -- Example properties by node_type:
    -- publication: { "category": "tech", "subscriber_count": 15000, "topics": ["ai", "devtools"] }
    -- subscriber:  { "engagement_score": 85.5, "tier": "premium", "country": "US" }
    -- post:        { "published_at": "2026-05-20", "topics": ["ai", "llm"], "word_count": 1200 }
    -- topic:       { "canonical_name": "Artificial Intelligence", "aliases": ["AI", "ML"] }
    -- email_address: { "domain": "gmail.com" }  -- cross-pub identity node
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_gnodes_type ON graph_nodes(node_type);
CREATE INDEX idx_gnodes_pub ON graph_nodes(publication_id);
CREATE INDEX idx_gnodes_props ON graph_nodes USING GIN(properties);

-- Graph edges represent directed relationships between nodes.

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
    -- Edge types:
    -- subscriber -> publication:  SUBSCRIBES_TO
    -- subscriber -> post:         READ, CLICKED, SHARED
    -- subscriber -> subscriber:   REFERRED
    -- subscriber -> topic:        INTERESTED_IN (inferred from reading patterns)
    -- publication -> topic:       COVERS
    -- publication -> publication: AUDIENCE_OVERLAP, SIMILAR_TO, BOOSTED_BY
    -- post -> topic:              TAGGED_WITH
    -- post -> post:               RELATED_TO (content similarity)
    -- email_address -> subscriber: IDENTITY (cross-pub identity linking)
    weight          FLOAT DEFAULT 1.0,              -- relationship strength / confidence
    properties      JSONB DEFAULT '{}',
    -- Example properties by edge_type:
    -- SUBSCRIBES_TO: { "since": "2026-01-15", "tier": "premium", "engagement": 85 }
    -- READ:          { "at": "2026-05-20T14:00:00Z", "read_time_seconds": 180, "scroll_depth": 0.92 }
    -- REFERRED:      { "at": "2026-03-01", "referral_program_id": "uuid" }
    -- AUDIENCE_OVERLAP: { "overlap_count": 3400, "overlap_pct": 0.23, "computed_at": "2026-05-20" }
    -- INTERESTED_IN: { "confidence": 0.87, "based_on_posts": 12, "computed_at": "2026-05-20" }
    -- SIMILAR_TO:    { "cosine_similarity": 0.82, "method": "tfidf", "computed_at": "2026-05-20" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_gedges_source ON graph_edges(source_id);
CREATE INDEX idx_gedges_target ON graph_edges(target_id);
CREATE INDEX idx_gedges_type ON graph_edges(edge_type);
CREATE INDEX idx_gedges_source_type ON graph_edges(source_id, edge_type);
CREATE INDEX idx_gedges_target_type ON graph_edges(target_id, edge_type);
CREATE INDEX idx_gedges_props ON graph_edges USING GIN(properties);

-- Materialised audience overlap table (computed periodically by background job)
CREATE TABLE audience_overlaps (
    publication_a   UUID NOT NULL REFERENCES publications(id),
    publication_b   UUID NOT NULL REFERENCES publications(id),
    overlap_count   INTEGER NOT NULL,               -- shared email addresses
    a_unique        INTEGER NOT NULL,               -- subscribers only in A
    b_unique        INTEGER NOT NULL,               -- subscribers only in B
    overlap_pct_a   FLOAT NOT NULL,                 -- overlap_count / A total
    overlap_pct_b   FLOAT NOT NULL,                 -- overlap_count / B total
    jaccard_index   FLOAT NOT NULL,                 -- overlap / union
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (publication_a, publication_b),
    CHECK (publication_a < publication_b)            -- avoid duplicate pairs
);

CREATE INDEX idx_overlap_a ON audience_overlaps(publication_a);
CREATE INDEX idx_overlap_b ON audience_overlaps(publication_b);
```

## Graph Queries

### Find newsletters with the highest audience overlap (Boost candidates)

```sql
-- "Which publications share the most subscribers with mine?"
SELECT
    p.name AS publication_name,
    ao.overlap_count,
    ao.overlap_pct_a AS my_overlap_pct,
    ao.jaccard_index
FROM audience_overlaps ao
JOIN publications p ON p.id = CASE
    WHEN ao.publication_a = '{{my_pub_id}}' THEN ao.publication_b
    ELSE ao.publication_a
END
WHERE ao.publication_a = '{{my_pub_id}}' OR ao.publication_b = '{{my_pub_id}}'
ORDER BY ao.jaccard_index DESC
LIMIT 10;
```

### Content recommendation via shared audience

```sql
-- "What posts from other publications are popular with MY subscribers?"
-- Two-hop traversal: my_pub <- SUBSCRIBES_TO - subscriber - READ -> post
WITH my_subscribers AS (
    SELECT source_id AS sub_node_id
    FROM graph_edges
    WHERE target_id = '{{my_pub_node_id}}'
      AND edge_type = 'SUBSCRIBES_TO'
)
SELECT
    gn.label AS post_title,
    gn.properties->>'published_at' AS published_at,
    COUNT(*) AS shared_readers,
    AVG((ge.properties->>'scroll_depth')::float) AS avg_scroll_depth
FROM graph_edges ge
JOIN my_subscribers ms ON ms.sub_node_id = ge.source_id
JOIN graph_nodes gn ON gn.id = ge.target_id
WHERE ge.edge_type = 'READ'
  AND gn.node_type = 'post'
  AND gn.publication_id != '{{my_pub_id}}'       -- exclude my own posts
GROUP BY gn.id, gn.label, gn.properties->>'published_at'
HAVING COUNT(*) >= 10                              -- minimum shared readers
ORDER BY shared_readers DESC
LIMIT 20;
```

### Subscriber interest inference

```sql
-- "What topics is this subscriber most interested in?"
-- Based on INTERESTED_IN edges (computed from reading patterns)
SELECT
    gn.label AS topic,
    ge.weight AS interest_score,
    (ge.properties->>'based_on_posts')::int AS posts_read,
    ge.properties->>'computed_at' AS last_computed
FROM graph_edges ge
JOIN graph_nodes gn ON gn.id = ge.target_id
WHERE ge.source_id = '{{subscriber_node_id}}'
  AND ge.edge_type = 'INTERESTED_IN'
ORDER BY ge.weight DESC
LIMIT 10;
```

### Referral network depth

```sql
-- "Show the referral chain: who referred whom?"
WITH RECURSIVE referral_chain AS (
    -- Start from a specific subscriber
    SELECT
        ge.source_id AS referrer_id,
        ge.target_id AS referred_id,
        gn.label AS referred_label,
        1 AS depth
    FROM graph_edges ge
    JOIN graph_nodes gn ON gn.id = ge.target_id
    WHERE ge.source_id = '{{subscriber_node_id}}'
      AND ge.edge_type = 'REFERRED'

    UNION ALL

    -- Follow the chain
    SELECT
        ge.source_id,
        ge.target_id,
        gn.label,
        rc.depth + 1
    FROM graph_edges ge
    JOIN graph_nodes gn ON gn.id = ge.target_id
    JOIN referral_chain rc ON rc.referred_id = ge.source_id
    WHERE ge.edge_type = 'REFERRED'
      AND rc.depth < 5                             -- limit depth
)
SELECT * FROM referral_chain ORDER BY depth;
```

### Cross-publication identity linking

```sql
-- "Find all publications this person subscribes to across the platform"
-- Uses email_address as a cross-pub identity node
SELECT
    gn_pub.label AS publication_name,
    ge_sub.properties->>'tier' AS tier,
    (ge_sub.properties->>'engagement')::float AS engagement
FROM graph_edges ge_id
JOIN graph_edges ge_sub ON ge_sub.source_id = ge_id.source_id
JOIN graph_nodes gn_pub ON gn_pub.id = ge_sub.target_id
WHERE ge_id.target_id = '{{email_address_node_id}}'
  AND ge_id.edge_type = 'IDENTITY'
  AND ge_sub.edge_type = 'SUBSCRIBES_TO'
  AND gn_pub.node_type = 'publication';
```

## Graph Sync Triggers

```sql
-- Sync subscriber creation to graph
CREATE OR REPLACE FUNCTION sync_subscriber_to_graph() RETURNS TRIGGER AS $$
BEGIN
    -- Create subscriber node
    INSERT INTO graph_nodes (id, node_type, publication_id, label, properties)
    VALUES (
        NEW.id,
        'subscriber',
        NEW.publication_id,
        NEW.email,
        jsonb_build_object(
            'engagement_score', COALESCE(NEW.engagement_score, 0),
            'tier', (SELECT name FROM subscription_tiers WHERE id = NEW.tier_id),
            'country', NEW.country_code,
            'tags', to_jsonb(NEW.tags)
        )
    )
    ON CONFLICT (id) DO UPDATE SET
        properties = EXCLUDED.properties,
        updated_at = NOW();

    -- Create SUBSCRIBES_TO edge
    INSERT INTO graph_edges (source_id, target_id, edge_type, properties)
    VALUES (
        NEW.id,
        NEW.publication_id,
        'SUBSCRIBES_TO',
        jsonb_build_object('since', NEW.subscribed_at, 'engagement', NEW.engagement_score)
    );

    -- Create or link email_address identity node
    INSERT INTO graph_nodes (id, node_type, label, properties)
    VALUES (
        gen_random_uuid(),
        'email_address',
        NEW.email,
        jsonb_build_object('domain', split_part(NEW.email, '@', 2))
    )
    ON CONFLICT DO NOTHING;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_subscriber_graph
    AFTER INSERT ON subscribers
    FOR EACH ROW EXECUTE FUNCTION sync_subscriber_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Relational — Identity | 3 | publications, users, publication_members |
| Relational — Subscribers | 3 | subscribers, lists, subscriber_lists |
| Relational — Content | 1 | posts |
| Relational — Campaigns | 2 | campaigns, deliveries |
| Relational — Monetisation | 3 | subscription_tiers, subscriber_subscriptions, payments |
| Relational — Supporting | 4 | sending_domains, templates, media, api_keys |
| Graph Layer | 3 | graph_nodes, graph_edges, audience_overlaps |
| **Total** | **19** | 16 relational + 3 graph |

---

## Key Design Decisions

1. **Graph as secondary index, not source of truth** — relational tables own the data; the graph layer is derived. If the graph becomes inconsistent, it can be rebuilt from the relational tables. This avoids the risk of graph corruption affecting operations.

2. **Email address as cross-publication identity node** — the `email_address` node type enables cross-publication queries ("which newsletters does this person subscribe to?") without exposing subscriber records across publication boundaries. Audience overlap is computed from email identity nodes.

3. **Typed edges with properties** — `SUBSCRIBES_TO`, `READ`, `REFERRED`, `INTERESTED_IN`, `AUDIENCE_OVERLAP` edges each carry type-specific properties. The edge type determines the property schema, documented in code but flexible in the database.

4. **Audience overlap as a materialised table** — rather than computing overlap on-the-fly (which requires cross-publication email joins), the `audience_overlaps` table is recomputed periodically by a background job. The Jaccard index enables fair comparison between publications of different sizes.

5. **Topic nodes as the recommendation bridge** — publications and posts are connected to topic nodes via `COVERS` and `TAGGED_WITH` edges. Subscribers are connected to topic nodes via `INTERESTED_IN` edges (inferred from reading patterns). This three-way graph enables content recommendations.

6. **Weight on edges for ranking** — `graph_edges.weight` allows ranking: a subscriber who clicked 5 links in a post has a higher READ weight than one who only opened it. INTERESTED_IN confidence scores are ML-derived.

7. **Recursive CTEs for graph traversal** — PostgreSQL's `WITH RECURSIVE` handles referral chains and multi-hop queries. For deeper traversals (6+ hops), a dedicated graph database would be more efficient.

8. **Graph sync via triggers** — database triggers keep the graph consistent with relational changes. For high-write workloads, this can be replaced with async event-driven sync (e.g., via a message queue).

9. **Streamlined relational core** — the relational layer has fewer tables than Model 1 because the graph layer absorbs the relationship-heavy queries (referrals, tag-based similarity, content recommendations) that would otherwise need junction tables.

10. **Privacy-preserving overlap** — audience overlap is expressed as counts and percentages, not individual subscriber identities. Publications can see "23% audience overlap with Newsletter X" without accessing each other's subscriber lists.
