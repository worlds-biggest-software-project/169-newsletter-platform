# Newsletter Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open alternative to Substack and beehiiv that gives independent writers and media teams full ownership of their audience, their revenue, and their growth stack.

Newsletter Platform is a candidate project for an end-to-end publishing system covering subscriber management, paid subscriptions, automation, analytics, and growth tooling. It targets independent journalists, creators, brand and marketing teams, and multi-property media companies who want a Substack alternative without revenue cuts or platform lock-in.

---

## Why Newsletter Platform?

- Substack takes a 10% cut of paid subscription revenue, which becomes expensive for any publication earning more than ~$390/month from subscribers.
- Existing zero-cut platforms (beehiiv, Kit, Ghost) are either proprietary SaaS or — in Ghost's case — open source but lacking modern growth tooling and built-in discovery.
- Apple Mail Privacy Protection has eroded open-rate reliability, yet most platforms still default to opens as their primary engagement metric instead of scroll depth, click heatmaps, or referral tracking.
- Creator monetisation is fragmenting across paid subscriptions, ad networks, referral marketplaces (beehiiv Boosts), digital products (Kit), and podcasting — but no open-source platform unifies these revenue streams.
- AI writing assistance, churn prediction, and ML-based segmentation are becoming differentiating features on growth platforms, leaving an open-source AI-native gap.

---

## Key Features

### Publishing & Email Delivery

- Visual or Markdown-based email editor with responsive templates
- Reliable SMTP delivery with DKIM/SPF/DMARC authentication
- RSS-to-email automation for syndicated publishing
- Mobile-responsive rendering across email clients
- Unsubscribe management compliant with CAN-SPAM, GDPR, and CASL

### Subscriber & Audience Management

- Subscriber import, tagging, and segmentation
- Conditional logic and rule-based segments
- Visual workflow automation builder for sequences and triggers
- Multiple paid subscription tiers
- Team accounts and multi-property organisation support

### Monetisation

- Stripe-native paid subscriptions with zero platform revenue cut
- Digital product sales alongside subscriptions
- Referral programme for creator-to-creator collaboration
- Multi-tier pricing with gated content per tier

### Growth Infrastructure

- Landing pages, signup forms, and pop-ups for lead capture
- Growth analytics tracking acquisition sources
- Engagement metrics beyond opens (clicks, scroll depth, referrals)
- Subscriber acquisition reporting

### Analytics & Insights

- Open rates, click rates, and subscriber metrics
- Subscriber behaviour and engagement reporting
- Growth source attribution

---

## AI-Native Advantage

The platform layers AI capabilities that incumbents lack: a writing co-pilot trained on the creator's own archive and voice to draft posts, subject lines, and hooks; dynamic paywall positioning that picks the optimal gating point per reader based on engagement history; ML-based segmentation that builds intent-based audiences from reading behaviour and referral source; and a churn prediction model that flags at-risk paid subscribers before cancellation and triggers personalised retention. A cross-newsletter recommendation engine surfaces collaboration and Boost-style opportunities based on audience overlap.

---

## Tech Stack & Deployment

The project targets both self-hosted and managed cloud deployment, following the precedent set by Ghost. Core integrations include Stripe (and Stripe Connect) for paid subscriptions and creator payouts, standard email authentication (DKIM, SPF, DMARC) for inbox placement, and RSS/Atom for cross-platform syndication. A REST API and Markdown-native authoring path are intended for developer-friendly integration, in line with Buttondown and Ghost.

---

## Market Context

The broader email marketing market exceeds $10 billion globally in 2026, with the independent newsletter tooling segment estimated in the hundreds of millions annually based on beehiiv's reported $30M annualised revenue in mid-2025. Substack has raised over $82M and beehiiv approximately $50M (including a $33M Series B in April 2024 led by NEA, Lightspeed, and Sapphire). Primary buyers are independent journalists and essayists, creator-economy operators bundling newsletters with courses or communities, brand and marketing teams pursuing owned-audience strategies, and media companies running multiple newsletter properties.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
