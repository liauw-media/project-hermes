# ADR-002: Aggregator-First Strategy

> **Status:** PROPOSED
> **Date:** 2026-02-16
> **Supersedes:** Carrier-direct approach from ADR-001 (for MVP scope only)
> **Decision Makers:** Team (pending approval)

---

## Context

ADR-001 defined a carrier-direct architecture: own SMPP connections via Ergo actor mesh, own MTA via KumoMTA, HLR lookups, and full telecom infrastructure. Five expert reviews scored this approach 0.5/10 to 5/10, with the mentor calling the timeline "fiction" and the DevOps reviewer estimating $665K-$1M/year in operational costs.

Key constraints identified:
- Carrier contracts require 3-6 months to negotiate
- Danish telecom registration is legally required (criminal offense without it)
- IP warming for own MTA takes 8 weeks minimum
- Team has zero SMPP protocol experience
- 20+ weeks to MVP with carrier-direct approach

Colleague feedback (Feb 2026) crystallized a strategic pivot:
> "We will by 90% work directly with a telecom aggregator. This means we only need to handle inbound requests, maybe manipulate them to some degree and pass it on to another omnichannel API in the MVP. For email, simply wrapping Amazon SES and letting users set up DNS settings would suffice at stage one. Since according to our calculations margins are big enough. Building the reliable infrastructure to handle the throughput is the important part."

## Decision

**MVP (Phase 0-3, 4 weeks):** Build Hermes as a smart message broker using telecom aggregators (Twilio, Vonage, Messente) for SMS and Amazon SES for email. Focus engineering effort on the cascade engine, multi-tenancy, validation pipeline, and reliable throughput infrastructure.

**Phase 4 (Weeks 5-8):** Production hardening — security, Kubernetes, monitoring, additional adapters.

**Phase 5+ (business decision):** Evaluate carrier-direct connections and own MTA only when volume and margin analysis justify the investment. All research and architecture planning for carrier-direct is preserved.

## Rationale

### Why Aggregator-First

1. **Time to market:** 4 weeks vs. 20+ weeks
2. **Risk elimination:** No carrier contracts, no telecom registration, no IP warming, no SMPP expertise required
3. **Margins are viable:** 33% gross margin on SMS at aggregator pricing ($20K/month at 1M SMS/month, $2M/month at 100M SMS/month)
4. **Core product validation:** The cascade engine, multi-tenancy, and unified API are the product — not the carrier connections
5. **Reversible decision:** Hexagonal architecture allows swapping aggregator adapters for carrier-direct adapters without touching the core

### Why Not Carrier-Direct from Day One

1. Expert reviews scored carrier-direct feasibility 0.5-5/10
2. Carrier contracts alone take 3-6 months (blocks revenue)
3. Danish telecom registration is a criminal offense blocker
4. 8-week IP warming is a hard physical constraint for email
5. Team has no SMPP experience — high learning curve risk
6. Aggregators commoditize the last mile; Hermes differentiates on orchestration

### What Changes from ADR-001

| ADR-001 Component | ADR-002 Status |
|-------------------|----------------|
| Hexagonal architecture | **RETAINED** — same ports & adapters pattern |
| Domain types (Tenant, Message, etc.) | **RETAINED** — unchanged |
| 9-layer validation pipeline | **RETAINED** — unchanged |
| Cascade engine (Temporal/Restate) | **RETAINED** — core product |
| NATS JetStream transport | **RETAINED** — delivery stream |
| PostgreSQL + Redis | **RETAINED** — same infrastructure |
| OpenTelemetry + Prometheus | **RETAINED** — same observability |
| SMPP client/server (gosmpp) | **DEFERRED** — Phase 5+ business decision |
| Ergo Framework actor mesh | **DEFERRED** — Phase 5+ business decision |
| KumoMTA email delivery | **DEFERRED** — Phase 5+ business decision |
| HLR/MNP lookups | **DEFERRED** — aggregators handle routing |
| WASM adapters (Wazero) | **RETAINED** — for aggregator REST APIs |
| Smart Router | **SIMPLIFIED** — provider selection (not carrier routing) |

### What ADR-001 Gets Right (Unchanged)

- Go as the implementation language
- Hexagonal architecture with port interfaces
- Temporal.io / Restate for cascade durable execution
- WASM/Wazero for hot-swappable provider adapters
- NATS JetStream for delivery transport
- Multi-tenant tiered isolation model
- 9-layer validation pipeline design
- Error taxonomy
- OpenTelemetry observability

## Consequences

### Positive

- MVP achievable in 4 weeks with 2-3 devs + Claude
- All carrier-direct risks eliminated from MVP scope
- Revenue-generating product before carrier contracts are needed
- Engineering focus on core differentiator (cascade engine, multi-tenancy)
- Lower infrastructure costs ($500-930/month vs. $5K+/month)

### Negative

- Lower per-message margins vs. carrier-direct (33% vs. 83% on SMS)
- Dependency on aggregator pricing and rate limits
- SES sending limits require increase requests
- Perception risk: "just a wrapper" (mitigated by cascade engine value)

### Neutral

- All carrier-direct research and architecture preserved for Phase 5+
- ADR-001 remains valid as the long-term vision document
- Hexagonal architecture guarantees adapter-level swaps when carrier-direct is pursued

## Path to Carrier-Direct (Phase 5+)

Carrier-direct becomes a business decision triggered by:
- SMS aggregator margins below target threshold
- Monthly SMS volume > 5M (carrier contracts become economical)
- Customer requirements for carrier-specific features
- Team has 6+ months of platform operational experience

When triggered, the swap is an **adapter change, not a rewrite:**
```
Twilio WASM adapter  →  SMPP Ergo actor adapter
  (ProviderAdapter)       (ProviderAdapter)

Amazon SES adapter   →  KumoMTA adapter
  (ProviderAdapter)       (ProviderAdapter)
```

The cascade engine, validation pipeline, router, tenant management, and observability are completely unaffected.

## Email Platform Vision (Phase 5+)

Beyond delivery, Hermes can build value-added services on email:
- Security scanning (phishing, malware)
- Encryption (S/MIME, PGP)
- Compliance engine (GDPR, data residency)
- Hosted SMTP service
- Analytics and engagement tracking

This requires owning the email pipeline (KumoMTA), making it a natural next step when the platform is stable.

## Related Documents

| Document | Relationship |
|----------|-------------|
| `docs/adr/ADR-001-hermes-platform-architecture.md` | Long-term vision (carrier-direct). This ADR narrows MVP scope. |
| `docs/plans/implementation-plan.md` | Detailed implementation plan (v2.0, aggregator-first) |
| `docs/plans/timeline.md` | 4-week MVP + 4-week production timeline |
| `docs/research/technology-investigation.md` | Technology evaluation (16 technologies) |
| `docs/research/temporal-deep-dive.md` | Temporal cascade engine deep dive |
| `docs/research/actor-model-wasm.md` | Ergo + WASM evaluation (Phase 5+ reference) |

---

*Generated: 2026-02-16 | Status: PROPOSED — Pending Team Approval*
