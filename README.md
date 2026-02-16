# Project Hermes

> **One API to rule them all.**

A multi-tenant, multi-channel message broker that unifies SMS, Email, Push, and WhatsApp behind a single API with intelligent cascade delivery, automatic fallback, and crash-proof durability.

**Status:** Pre-development (architecture + research complete, pending team approval)
**Language:** Go
**Architecture:** Hexagonal (Ports & Adapters)
**Team:** 2-3 developers + Claude Opus 4.6

---

## What Hermes Does

A customer sends one API call. Hermes handles the rest:

```
POST /v1/messages
{
  "to": "+4512345678",
  "body": "Your order has shipped",
  "cascade": ["sms", "email", "push"],
  "timeout_per_step": "30s"
}
```

Hermes will:
1. Try SMS via Twilio/aggregator
2. If no delivery confirmation within 30s, try Email via SES
3. If that fails, try Push via FCM
4. Report final status back via webhook

All of this survives server crashes, network failures, and restarts — guaranteed by Temporal.io durable execution (the same technology Twilio uses for every message they send).

---

## Why This Exists

No single provider solves multi-channel cascade delivery:

| Provider | SMS | Email | Push | WhatsApp | Cascade | Multi-tenant |
|----------|-----|-------|------|----------|---------|-------------|
| Twilio | Yes | Yes (SendGrid) | No | Yes | No | No |
| Vonage | Yes | No | No | Yes | No | No |
| Amazon SES | No | Yes | No | No | No | No |
| Firebase | No | No | Yes | No | No | No |
| **Hermes** | **Yes** | **Yes** | **Yes** | **Yes** | **Yes** | **Yes** |

Hermes is the orchestration layer. Providers are interchangeable adapters.

---

## Development Timeline

```
        Week 1        Week 2        Week 3        Week 4        Weeks 5-8
        ┌─────────────┬─────────────┬─────────────┬─────────────┬────────────────────┐
        │ FOUNDATION  │  CASCADE    │  PROVIDER   │  MULTI-     │   PRODUCTION       │
        │             │  ENGINE     │  ADAPTERS   │  TENANCY    │   READINESS        │
        │             │             │             │             │                    │
        │ Go workspace│ Temporal    │ WASM host   │ Tenant CRUD │ Vault secrets      │
        │ Domain types│ workflow    │ Twilio SMS  │ Validation  │ Kubernetes         │
        │ Port ifaces │ REST API    │ Amazon SES  │ Rate limits │ Grafana dashboards │
        │ Docker      │ NATS streams│ FCM Push    │ OTel traces │ Alerting           │
        │ CI pipeline │ DLR webhook │ Smart Router│ Load test   │ WhatsApp adapter   │
        │             │             │             │             │ Security audit     │
        └─────────────┴─────────────┴─────────────┴─────────────┴────────────────────┘
                                                        │                      │
                                                   MVP DEMO              PROD v1
                                                   (Week 4)              (Week 8)
```

| Milestone | Target | What It Means |
|-----------|--------|---------------|
| **MVP Demo** | End of Week 4 | Working API, cascade fallback, real SMS+Email+Push, multi-tenant, 1K msg/sec |
| **Production v1** | End of Week 8 | First customer live, monitoring, alerting, security hardened, K8s deployed |
| **Carrier-Direct** | Week 9+ | Business decision: own SMPP connections + MTA for higher margins |

---

## Cost Evaluation

### Development Costs (MVP: 4 Weeks)

| Resource | Monthly Cost | MVP Period (1 month) |
|----------|-------------|---------------------|
| Dev 1 (Lead) — full-time | Market rate | Market rate |
| Dev 2 — full-time | Market rate | Market rate |
| Dev 3 — full-time or part-time | Market rate | Market rate |
| Claude Opus 4.6 | ~$200/month (API usage) | ~$200 |
| **Infrastructure** | **$0** (Docker Compose) | **$0** |
| **Provider sandbox accounts** | **$0** (free tiers) | **$0** |

> Infrastructure and provider costs are zero during MVP development. Everything runs locally via Docker Compose. Provider sandboxes (Twilio, SES, FCM) are free.

### Infrastructure Costs (Production)

| Phase | Monthly Cost | What You Get |
|-------|-------------|-------------|
| MVP (Weeks 1-4) | ~$0 | Local Docker Compose |
| Staging (Weeks 5-6) | ~$500-1,000 | Small K8s cluster (3 nodes) |
| Production v1 (Week 8+) | ~$2,000-3,000 | HA K8s cluster (3-5 nodes), monitoring |
| Production at scale | ~$3,000-5,000 | Scaled for 22M msg/day |

Includes: Hermes services, PostgreSQL, NATS JetStream, Redis, Temporal Server, Prometheus, Grafana.

### Provider Costs (Per-Message)

| Channel | Provider | Cost | Free Tier |
|---------|----------|------|-----------|
| SMS (US) | Twilio | $0.0079/msg | — |
| SMS (EU/DK) | Twilio | $0.04-0.09/msg | — |
| SMS (EU/DK) | Messente | $0.02-0.06/msg | — |
| Email | Amazon SES | $0.10/1,000 emails | 62,000/month |
| Push | FCM | Free | Unlimited |
| WhatsApp | Meta Cloud API | $0.005-0.10/msg | 1,000 conversations/month |

### Total Cost of Ownership (First Year)

| Scenario | Monthly Volume | Infra/month | Provider/month | Total/month |
|----------|---------------|-------------|----------------|-------------|
| **Early stage** | 100K SMS + 500K email | ~$2,000 | ~$4,050 | ~$6,050 |
| **Growth** | 1M SMS + 5M email | ~$3,000 | ~$40,500 | ~$43,500 |
| **Scale** | 10M SMS + 50M email | ~$5,000 | ~$405,000 | ~$410,000 |

> Compare: Building carrier-direct from day one would cost $5K-10K/month infrastructure + 6-9 months development time before any revenue.

---

## Business Value Evaluation

### Market Opportunity

| Metric | Value | Source |
|--------|-------|--------|
| CPaaS market size (2025) | $19.87B | Industry reports |
| Projected (2030) | $80.40B | 30.4% CAGR |
| SMS market (business messaging) | ~$78B (2025) | GSMA |
| Email delivery market | ~$1.5B (2025) | Industry reports |

### Unit Economics

#### SMS (EU Market, Aggregator Path)

| Volume/month | Cost/SMS | Revenue/SMS | Gross Margin | Monthly Profit |
|-------------|----------|-------------|-------------|----------------|
| 100K | $0.04 | $0.06 | 33% | $2,000 |
| 1M | $0.04 | $0.06 | 33% | $20,000 |
| 10M | $0.035 (volume discount) | $0.055 | 36% | $200,000 |
| 100M | $0.03 (volume discount) | $0.05 | 40% | $2,000,000 |

#### SMS (EU Market, Carrier-Direct — Future Phase 5)

| Volume/month | Cost/SMS | Revenue/SMS | Gross Margin | Monthly Profit |
|-------------|----------|-------------|-------------|----------------|
| 10M | $0.01 | $0.05 | 80% | $400,000 |
| 100M | $0.008 | $0.045 | 82% | $3,700,000 |

> **Carrier-direct doubles margins** — but requires carrier contracts (3-6 months), SMPP engineering, and telecom registration. Pursue when volume justifies it.

#### Email

| Volume/month | SES Cost | Revenue | Gross Margin | Monthly Profit |
|-------------|----------|---------|-------------|----------------|
| 500K | $50 | $100 (at $0.20/1K) | 50% | $50 |
| 5M | $500 | $1,000 | 50% | $500 |
| 50M | $5,000 | $10,000 | 50% | $5,000 |
| 500M | $50,000 | $100,000 | 50% | $50,000 |

> Email margins are thinner at aggregator level. **KumoMTA (Phase 5+) changes this**: $50K/month SES cost drops to ~$5K/month infrastructure, boosting margin to 95%.

#### Cascade (The Differentiator)

Cascade delivery is the product moat. No competitor offers this:

| Scenario | Without Hermes | With Hermes | Value |
|----------|---------------|-------------|-------|
| SMS fails, customer unreachable | Message lost | Email auto-fallback | Message delivered |
| Provider outage | All messages queued/lost | Auto-route to backup provider | Zero downtime |
| Price spike on provider | Overpay or manual switch | Smart Router selects cheapest | Cost savings |
| Compliance violation | Per-provider config | Centralized compliance engine | Risk reduction |

**Cascade adds value that pure delivery providers don't offer.** This justifies premium pricing above aggregator cost.

### Revenue Projections

| Timeline | Monthly SMS | Monthly Email | SMS Revenue | Email Revenue | Cascade Premium | Total Monthly |
|----------|-----------|-------------|------------|--------------|----------------|--------------|
| Month 3 (first customers) | 500K | 2M | $30,000 | $400 | $5,000 | ~$35,000 |
| Month 6 | 2M | 10M | $120,000 | $2,000 | $20,000 | ~$142,000 |
| Month 12 | 10M | 50M | $550,000 | $10,000 | $80,000 | ~$640,000 |
| Month 24 | 50M | 200M | $2,500,000 | $40,000 | $300,000 | ~$2,840,000 |

> These projections assume aggregator pricing. Carrier-direct (Phase 5) would roughly double SMS margins at 10M+/month volumes.

### Competitive Positioning

| Competitor | Strengths | Hermes Advantage |
|-----------|-----------|-----------------|
| **Twilio** | Market leader, huge ecosystem | Hermes offers cascade across providers (including Twilio as one adapter) |
| **Vonage** | Strong EU presence | Hermes is provider-agnostic; Vonage is one option among many |
| **MessageBird** | Omnichannel, Flow Builder | Hermes cascade is code-first (API-native), not visual builder |
| **Infobip** | 1.2B transactions/day, 600+ operators | Hermes starts focused (Nordics), expands with data |
| **Sinch** | Carrier relationships | Hermes can use Sinch as an aggregator adapter |

**Key insight:** Hermes doesn't compete with aggregators — it sits on top of them. Every aggregator is a potential adapter, not a competitor.

### Break-Even Analysis

| Cost Category | Monthly | Cumulative |
|---------------|---------|-----------|
| Dev team (3 devs, 4 months to prod) | ~varies | — |
| Infrastructure (average) | $2,500 | — |
| Provider costs | Variable (pass-through) | — |

**Break-even on infrastructure:** ~125K SMS/month (at $0.02 margin/SMS) covers $2,500 infra.

**Break-even on full team cost:** Depends on team salaries. At $10K/month margin target, need ~500K SMS/month through aggregators.

### Strategic Value Beyond Revenue

| Value Driver | Description |
|-------------|-------------|
| **Data moat** | Every message creates delivery data — route optimization improves with scale |
| **Provider leverage** | Multi-provider routing gives negotiating power on pricing |
| **Platform lock-in** | Cascade rules, validation config, and webhook integrations create switching costs |
| **Email platform play** | Phase 5+: security scanning, encryption, compliance on top of owned email pipeline |
| **Carrier-direct option** | Preserved architecture allows 2x margin jump when volume justifies it |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  REST API  →  Validation (9 layers)  →  Cascade Engine (Temporal)   │
│                                              │                       │
│                                        Smart Router                  │
│                                              │                       │
│                                     NATS JetStream                   │
│                                              │                       │
│                    ┌─────────────┬────────────┼────────────┐         │
│                    │             │            │            │         │
│               Twilio SMS   Amazon SES    FCM Push    WhatsApp       │
│                (.wasm)     (native Go)    (.wasm)    (.wasm)        │
│                                                                      │
│  Infrastructure: PostgreSQL | NATS | Redis | Temporal | Grafana     │
└──────────────────────────────────────────────────────────────────────┘
```

**Key technologies:**
- **Temporal.io** — Durable execution for cascade (Twilio uses this)
- **WASM/Wazero** — Hot-swappable, sandboxed provider adapters
- **NATS JetStream** — High-throughput delivery transport
- **Go** — Performance, concurrency, simple deployment

See `docs/plans/implementation-plan.md` for full architecture details.

---

## Project Structure

```
hermes/
├── cmd/
│   ├── hermes-api/              # REST API server
│   └── hermes-cascade/          # Cascade engine workers (Temporal)
├── internal/
│   ├── domain/                  # Core types (Tenant, Message, CascadeRule)
│   ├── ports/                   # Port interfaces (hexagonal boundaries)
│   ├── cascade/                 # Temporal workflows + activities
│   ├── router/                  # Smart Router (provider selection)
│   ├── adapters/                # WASM host + webhook receiver
│   ├── security/                # Auth, tenant context
│   ├── validation/              # 9-layer validation pipeline
│   └── infra/                   # PostgreSQL, NATS, Redis clients
├── pkg/
│   ├── errors/                  # Error taxonomy
│   └── observability/           # OpenTelemetry setup
├── adapters/                    # WASM adapter source (.wasm binaries)
│   ├── twilio/                  # Twilio SMS
│   ├── ses/                     # Amazon SES (native Go — AWS SDK needs reflection)
│   ├── fcm/                     # Firebase Cloud Messaging
│   └── whatsapp/                # WhatsApp Cloud API
├── deploy/
│   └── docker/                  # Docker Compose (dev + production)
└── docs/
    ├── adr/                     # Architecture Decision Records
    ├── plans/                   # Implementation plan + timeline
    ├── research/                # Technology evaluations
    └── reviews/                 # Expert review findings
```

---

## Documentation

### Architecture Decisions
| Document | Description |
|----------|-------------|
| [ADR-001: Platform Architecture](docs/adr/ADR-001-hermes-platform-architecture.md) | Long-term vision — carrier-direct architecture, domain types, port interfaces |
| [ADR-002: Aggregator-First Strategy](docs/adr/ADR-002-aggregator-first-strategy.md) | MVP pivot — aggregators for SMS, SES for email, carrier-direct deferred |

### Plans
| Document | Description |
|----------|-------------|
| [Implementation Plan](docs/plans/implementation-plan.md) | Full architecture, phases, team allocation, cost analysis (v2.0) |
| [Timeline](docs/plans/timeline.md) | Gantt chart, milestones, decision gates, sprint cadence |

### Research
| Document | Description |
|----------|-------------|
| [Technology Investigation](docs/research/technology-investigation.md) | 16 technologies evaluated with verdicts |
| [Temporal Deep Dive](docs/research/temporal-deep-dive.md) | Cascade engine: Temporal vs Restate, scale proof, Go SDK |
| [Actor Model & WASM](docs/research/actor-model-wasm.md) | Ergo Framework, Wazero, KumoMTA (Phase 5+ reference) |
| [Encryption Strategy](docs/research/encryption-strategy.md) | Signal Protocol evaluation (future) |

### Expert Reviews
| Review | Score | Key Finding |
|--------|-------|-------------|
| [01-05](docs/reviews/) | 1.5-7.0/10 | Original architecture reviews |
| [06: Carrier-Direct Mentor](docs/reviews/06-carrier-direct-mentor.md) | 5/10 | "Stop writing ADRs. Start writing Go." |
| [07: Security](docs/reviews/07-carrier-direct-security.md) | 3.5/10 | 13 security findings, TLS on SMPP non-negotiable |
| [08: UX/DX](docs/reviews/08-carrier-direct-ux-dx.md) | 4/10 | Error taxonomy critical, 3-min onboarding |
| [09: Architect](docs/reviews/09-carrier-direct-architect.md) | 4/10 | Backpressure 2/10, state persistence 2/10 |
| [10: DevOps](docs/reviews/10-carrier-direct-devops.md) | 0.5/10 | 87-item checklist, $665K-$1M/year ops |

> Expert reviews drove the technology investigation, which led to the Temporal/Ergo/WASM breakthroughs, which informed the aggregator-first pivot.

---

## Decisions Pending Team Approval

| # | Decision | Recommendation |
|---|----------|----------------|
| 1 | Accept aggregator-first strategy? | **Yes** — 4-week MVP, margins viable, carrier-direct preserved for later |
| 2 | Temporal as cascade engine? | **Yes** — proven at Twilio scale, mature Go SDK |
| 3 | WASM for provider adapters? | **Yes** — sandboxed, hot-swappable, ~57ns overhead |
| 4 | Which SMS aggregator(s)? | **Twilio + Messente** — best docs + competitive EU pricing |
| 5 | SES region? | **eu-west-1 or eu-central-1** — GDPR compliance |
| 6 | When to evaluate carrier-direct? | **Quarterly review** based on volume and margins |

---

## Getting Started (After Approval)

```bash
# Phase 0: Foundation
go mod init github.com/hermes-platform/hermes
docker compose up -d  # PostgreSQL, NATS, Redis, Temporal
go build ./...

# Phase 1: Cascade Engine
go run ./cmd/hermes-api/
go run ./cmd/hermes-cascade/
curl -X POST http://localhost:8080/v1/messages \
  -H "Authorization: Bearer test-key" \
  -d '{"to": "+4512345678", "body": "Hello from Hermes", "cascade": ["sms", "email"]}'
```

---

*Project Hermes | 2026 | Status: PENDING TEAM APPROVAL*
