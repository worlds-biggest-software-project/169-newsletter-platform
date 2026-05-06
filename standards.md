# Standards & API Reference

> Project: Newsletter Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management; governs access controls, encryption, and audit logging for newsletter platforms handling subscriber personal data, payment information (paid subscriptions), and creator content. URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 27018:2019 — Protection of PII in Public Clouds** — Governs processing of subscriber personal data (email addresses, payment details, reading behaviour analytics) in cloud-hosted newsletter platforms; requires data minimisation and retention transparency. URL: https://www.iso.org/standard/76559.html

### W3C & IETF Standards

- **RFC 5321 — SMTP: Simple Mail Transfer Protocol** — Foundational email transmission standard; newsletter platforms deliver content through SMTP-based email infrastructure (Postmark, SendGrid, Mailgun, Amazon SES, or self-hosted). URL: https://datatracker.ietf.org/doc/html/rfc5321

- **RFC 5322 — Internet Message Format** — Defines email message header and body syntax; governs newsletter email structure including subject line, From/Reply-To headers, and MIME multipart body (HTML + plain text alternative). URL: https://datatracker.ietf.org/doc/html/rfc5322

- **RFC 2045-2049 — MIME: Multipurpose Internet Mail Extensions** — Defines multipart email format with HTML content, plain text fallback, inline images, and file attachments used in newsletter delivery. URL: https://datatracker.ietf.org/doc/html/rfc2045

- **RFC 7208 — SPF: Sender Policy Framework** — Required DNS authentication for all newsletter sending domains; enforced by Gmail (February 2024), Outlook (May 2025), and Yahoo for bulk senders. URL: https://datatracker.ietf.org/doc/html/rfc7208

- **RFC 6376 — DKIM: DomainKeys Identified Mail** — Required cryptographic email signing; mandated by Gmail, Outlook, and Yahoo for bulk newsletter senders. URL: https://datatracker.ietf.org/doc/html/rfc6376

- **RFC 7489 — DMARC** — Domain-level email authentication policy; required for bulk senders by Gmail and Outlook; DMARCbis progressing as IETF Proposed Standard in 2025. URL: https://datatracker.ietf.org/doc/html/rfc7489

- **RFC 8058 — One-Click List-Unsubscribe** — Defines the List-Unsubscribe-Post header for one-click unsubscribe; required by Gmail and Yahoo for bulk senders; critical for newsletter platforms. URL: https://datatracker.ietf.org/doc/html/rfc8058

- **RFC 4287 — Atom Syndication Format** — IETF standard for publishing newsletter content as web feeds; used by Ghost and newsletter platforms for RSS/Atom feed endpoints that allow readers to subscribe via feed readers. URL: https://datatracker.ietf.org/doc/html/rfc4287

- **RSS 2.0** — The most widely supported web feed format; newsletter platforms (Ghost, Substack, Beehiiv) expose RSS feeds of published issues for feed reader subscribers and content aggregators. URL: https://www.rssboard.org/rss-specification

- **RFC 6749 — OAuth 2.0** — Authorization framework used by newsletter platforms for third-party integrations (Zapier, Make), CRM connections (HubSpot, Salesforce), and payment processor integrations. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — Used for newsletter platform API authentication tokens and subscriber magic-link authentication flows. URL: https://datatracker.ietf.org/doc/html/rfc7519

### Data Model & API Specifications

- **OpenAPI 3.1** — Used by Beehiiv, Kit (ConvertKit), Ghost, and email infrastructure providers to describe their REST APIs; enables SDK code generation and Zapier/Make integration creation. URL: https://spec.openapis.org/oas/latest.html

- **Ghost Content API** — REST/JSON API for read-only access to published Ghost newsletter posts, pages, authors, and tags; used for headless Ghost deployments and static site generation with Gatsby/Next.js; JSON-based with API key authentication. URL: https://docs.ghost.org/content-api

- **Ghost Admin API** — REST/JSON API for managing Ghost content programmatically (creating posts, managing members, sending newsletters); JWT-based authentication; used for custom integrations and migration scripts. URL: https://ghost.org/docs/admin-api/

- **Stripe API** — The universal payment processing API used by Ghost, Beehiiv, Substack, and Kit for paid newsletter subscriptions; Stripe Checkout, Billing, and webhooks handle subscription lifecycle management. URL: https://stripe.com/docs/api

- **Webhook Events** — JSON over HTTPS webhook delivery for newsletter platform events (new subscriber, unsubscribe, payment received, newsletter sent); used by Beehiiv, Ghost, Kit, and others for Zapier/Make automation triggers. URL: https://zapier.com/

### Security & Authentication Standards

- **GDPR Article 6 — Lawful Basis for Processing** — Newsletter subscriber email processing typically relies on "consent" as the lawful basis; newsletters must collect explicit opt-in consent and maintain consent records with timestamp, IP, and form identifier. URL: https://gdpr-info.eu/art-6-gdpr/

- **GDPR Article 7 — Conditions for Consent** — Consent must be freely given, specific, informed, and unambiguous; double opt-in is the recommended implementation; consent withdrawal must be as easy as giving it (one-click unsubscribe). URL: https://gdpr-info.eu/art-7-gdpr/

- **GDPR Article 17 — Right to Erasure** — Subscribers must be able to request deletion of all personal data from the newsletter platform; platforms must purge email address, engagement history, and any stored preferences on request. URL: https://gdpr-info.eu/art-17-gdpr/

- **CAN-SPAM Act (US)** — US federal commercial email law; requires honest From/subject lines, physical mailing address, and working unsubscribe mechanism honoured within 10 business days; FTC maximum penalty $53,088 per violation (effective January 2025). URL: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business

- **CASL — Canada's Anti-Spam Legislation** — Stricter than CAN-SPAM; requires express or implied consent before sending; applies to newsletters with Canadian subscribers. URL: https://crtc.gc.ca/eng/internet/anti.htm

- **CCPA/CPRA — California Consumer Privacy Act** — Governs personal data processing for California newsletter subscribers; rights to access, delete, and opt out of data sale. URL: https://oag.ca.gov/privacy/ccpa

- **SOC 2 Type II** — Required enterprise compliance certification for SaaS newsletter platforms; Beehiiv, Kit, and Ghost Pro maintain compliance certifications. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **PCI DSS v4.0** — Payment Card Industry standard governing credit card data for platforms processing paid newsletter subscription payments (if not fully delegating to Stripe/Paddle). URL: https://www.pcisecuritystandards.org/

- **OWASP API Security Top 10 (2023)** — Governs REST API security for newsletter management APIs; API1 (Broken Object Level Authorization) is critical for ensuring creators can only access their own subscriber lists and analytics. URL: https://owasp.org/API-Security/

### MCP Server Specifications

Newsletter platforms are beginning to integrate with the MCP ecosystem:

- **Ghost MCP Pattern** — Ghost's Admin and Content APIs are well-documented for programmatic content management; community MCP servers enable AI agents to create posts, manage members, and trigger newsletter sends via Ghost's REST API.

- **Beehiiv + MCP Automation** — Beehiiv's REST API and Webhook support enable AI-agent-driven newsletter workflows via MCP servers; natural-language prompted newsletter creation, subscriber segmentation queries, and analytics retrieval are emerging use cases.

- **AI Newsletter Generation Pattern (2025-2026)** — Emerging architecture: AI agents query content sources (RSS feeds, web searches, knowledge bases) via MCP, generate draft newsletters using LLMs, and publish via newsletter platform API; editorial review then triggers send; median time to first dollar for AI-assisted newsletters dropped to 66 days in 2025.

---

## Similar Products — Developer Documentation & APIs

### Substack

- **Description:** Largest creator newsletter platform; no transaction fees model (10% + Stripe fees on paid subscriptions); built-in discovery/recommendations network; simple writing interface; no public REST API (closed platform).
- **API Documentation:** No public REST API; Substack Import API for migrations only
- **SDKs/Libraries:** No official SDK; substack-api (community scraper, unofficial); Zapier integration (limited)
- **Developer Guide:** Not available (closed platform)
- **Standards:** No developer API; RSS 2.0 feed for public newsletters; email delivery via proprietary infrastructure
- **Authentication:** Substack account; no public API auth

### Ghost (Open Source)

- **Description:** Open-source (MIT) publishing platform with native newsletter functionality; self-hostable or Ghost Pro (managed); REST Content and Admin APIs; member subscription management; Stripe integration for paid newsletters; flat fee pricing (no transaction fees).
- **API Documentation:** https://docs.ghost.org/content-api and https://ghost.org/docs/admin-api/
- **SDKs/Libraries:** @tryghost/content-api (JavaScript); @tryghost/admin-api (JavaScript); ghost-python (community); api-demos (TryGhost/api-demos)
- **Developer Guide:** https://ghost.org/docs/
- **Standards:** REST/JSON, OpenAPI, JWT (Admin API), API key (Content API), RSS 2.0 + Atom, OAuth 2.0 (OAuth App integrations), Stripe for payments
- **Authentication:** Admin API: JWT (30-second expiry, HMAC-SHA256 signed); Content API: API key (Bearer token)

### Beehiiv

- **Description:** Modern creator newsletter platform founded by Morning Brew alumni; REST API v2 (2025) for subscriber management, publications, and automations; AI writing tools; paid newsletter subscriptions (lower fees than Substack); growing rapidly ($19M paid subscriptions in 2025, up 138% YoY).
- **API Documentation:** https://developers.beehiiv.com/
- **SDKs/Libraries:** REST API (JSON); Zapier/Make integrations (1,000+ apps); Webhook events
- **Developer Guide:** https://www.beehiiv.com/features/api-and-integrations
- **Standards:** REST/JSON (OpenAPI 3.0), OAuth 2.0, API token auth, Webhooks, RSS 2.0 + Atom, Stripe for payments
- **Authentication:** API token (Authorization Bearer); OAuth 2.0 for partner integrations

### Kit (formerly ConvertKit)

- **Description:** Creator-focused email marketing and newsletter platform with advanced automation (sequences, tags, custom fields); REST API v4 (2025, closed beta); strong in digital product sales alongside newsletters.
- **API Documentation:** https://developers.kit.com/api-reference/overview (v4 closed beta) and https://developers.convertkit.com/ (v3 public)
- **SDKs/Libraries:** REST API (JSON); convertkit-node; convertkit-python (community); Zapier/Make integrations
- **Developer Guide:** https://developers.kit.com/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, Personal API keys, Webhooks, RSS to Email, Stripe for products
- **Authentication:** API secret key; OAuth 2.0 for app integrations

### Mailchimp (Intuit)

- **Description:** Leading email marketing platform with newsletter capabilities; comprehensive REST API for campaigns, audiences, templates, automations, and analytics; 300+ integrations; strong for e-commerce newsletters with Shopify/WooCommerce sync.
- **API Documentation:** https://mailchimp.com/developer/marketing/api/
- **SDKs/Libraries:** mailchimp-marketing (Node.js, Python, PHP, Ruby); @mailchimp/mailchimp_marketing (npm)
- **Developer Guide:** https://mailchimp.com/developer/
- **Standards:** REST/JSON, OpenAPI 3.0, OAuth 2.0, API keys, Webhooks, RSS 2.0 (RSS campaign trigger)
- **Authentication:** API key (Basic Auth: user:apikey); OAuth 2.0 for partner apps

### Resend

- **Description:** Modern email API for developers (REST API); designed for transactional email and newsletter sending from code; React Email compatible; simple pricing ($20/month for 100K emails); used as the email infrastructure layer in developer-built newsletter platforms.
- **API Documentation:** https://resend.com/docs/api-reference/introduction
- **SDKs/Libraries:** resend (Node.js); resend-python; resend-go; resend-ruby; resend-php; resend-java; React Email integration
- **Developer Guide:** https://resend.com/docs
- **Standards:** REST/JSON, OpenAPI 3.1, API key auth, Webhooks (delivery events), DKIM/SPF auto-configuration
- **Authentication:** API key (Authorization: Bearer)

### Listmonk (Open Source)

- **Description:** Open-source (AGPL-3.0) self-hosted newsletter and mailing list manager; full REST API for campaigns, lists, subscribers, and templates; supports multiple SMTP providers; PostgreSQL backend; designed for self-hosted high-volume newsletter sending.
- **API Documentation:** https://listmonk.app/docs/api-reference/
- **SDKs/Libraries:** REST API (JSON); listmonk-node (community); Docker deployment
- **Developer Guide:** https://listmonk.app/docs/
- **Standards:** REST/JSON, OpenAPI, API token auth (Basic Auth), Webhooks, RSS 2.0, SMTP, AGPL-3.0 licence
- **Authentication:** Admin username/password; API token via Basic Auth

### Buttondown

- **Description:** Minimalist developer-friendly newsletter platform; REST API for subscribers, emails, and imports; Markdown-native editor; used by technical writers and developer advocates; reasonable pricing starting at $9/month.
- **API Documentation:** https://api.buttondown.email/v1/schema
- **SDKs/Libraries:** REST API (JSON); buttondown-python (community); Zapier integration
- **Developer Guide:** https://docs.buttondown.email/api-reference/introduction
- **Standards:** REST/JSON, OpenAPI, API key auth, Webhooks, RSS 2.0, Stripe for paid subscriptions
- **Authentication:** API key (Authorization: Token)

---

## Notes

- **Email deliverability as the dominant technical constraint (2025-2026)**: Gmail (February 2024), Outlook (May 2025), and Yahoo now enforce SPF, DKIM, DMARC, and one-click unsubscribe for bulk senders; newsletter platforms must auto-configure these DNS records for custom sending domains or face inbox delivery failures.

- **Creator monetization growth (2025)**: Paid newsletter subscriptions generated $19M on Beehiiv in 2025 (138% YoY growth); median time to first dollar dropped to 66 days; average paid price $11/month with 5-10% conversion from free; the creator economy makes API-integrated subscription management a critical platform feature.

- **Substack's closed platform risk**: Substack has no public API, making it difficult to automate workflows or migrate subscribers; the platform's 10% fee model makes it expensive at scale; Ghost (MIT, open-source) and Beehiiv (flat fee) are the primary alternatives for developers requiring API access.

- **Ghost MIT licence**: Ghost's open-source (MIT) licence makes it the most permissive newsletter platform foundation for building custom newsletter applications; the Admin API supports full programmatic content management and subscriber management.

- **Double opt-in as GDPR best practice**: GDPR-compliant newsletter platforms should implement double opt-in by default for EU subscribers; explicit consent with timestamp and source IP must be recorded; single opt-in lists may face deliverability penalties from major inbox providers.

- **Open-source landscape**: Ghost (MIT), Listmonk (AGPL-3.0), and Mautic (GPL v3) are the leading open-source newsletter platforms; Listmonk is the best option for self-hosted high-volume newsletter sending infrastructure; Ghost is the best option for content-driven creator newsletters requiring full ownership.
