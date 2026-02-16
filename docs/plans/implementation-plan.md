# Project Hermes: Aggregator-First Implementation Plan

> **Status:** PENDING TEAM APPROVAL
> **Date:** 2026-02-16
> **Revision:** 2.0 (Strategic Pivot from carrier-direct to aggregator-first)
> **Scope:** Multi-tenant, multi-channel message delivery broker in Go

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Strategic Decision: Aggregator-First](#strategic-decision-aggregator-first)
3. [Architecture Stack](#architecture-stack)
4. [Project Structure](#project-structure)
5. [Implementation Phases](#implementation-phases)
6. [Key Technology Decisions](#key-technology-decisions)
7. [What We're NOT Building in MVP (and Why)](#what-were-not-building-in-mvp-and-why)
8. [What We Preserve for Phase 2+](#what-we-preserve-for-phase-2)
9. [Risk Register](#risk-register)
10. [Team Allocation](#team-allocation)
11. [Cost Analysis](#cost-analysis)
12. [Decision Points for Team](#decision-points-for-team)
13. [Sources](#sources)

---

## Executive Summary

Hermes is a **smart message broker**, not a telecom operator. The core product is **"One API to rule them all"** -- a unified API that accepts messages across channels (SMS, email, push, WhatsApp) and delivers them through intelligent cascade logic with automatic fallback, retry, and observability.

For MVP, Hermes does **not** own the last mile. Third-party providers handle actual delivery:

- **SMS**: Telecom aggregators (Twilio, Vonage, Messente) handle carrier contracts, SMPP, routing, and compliance
- **Email**: Amazon SES handles deliverability, IP reputation, DKIM signing, and bounce processing
- **Push**: Firebase Cloud Messaging (FCM)
- **WhatsApp**: Meta Cloud API

What Hermes **does** own is the hard part that no single provider solves:

- **Cascade engine** (durable execution via Temporal/Restate): if SMS fails, try email, then push -- automatically, reliably, surviving crashes
- **Multi-tenant isolation**: API keys, rate limits, tenant-scoped routing
- **9-layer validation pipeline**: auth, rate limiting, schema validation, content compliance, deduplication
- **Smart routing**: provider selection based on tenant config, cost, and provider health
- **Unified observability**: OpenTelemetry traces across the entire message lifecycle, regardless of downstream provider
- **Provider abstraction via WASM adapters**: hot-swappable, sandboxed plugins for each provider

**Scale target:** 22M messages/day (2B texts + 6B emails/year).

This plan delivers a **working, demoable, multi-tenant MVP in 4 weeks** with 3 developers + Claude. Carrier-direct connections and own MTA infrastructure are deferred to Phase 5+ as business decisions driven by margin optimization.

---

## Strategic Decision: Aggregator-First

### Why This Approach

The previous implementation plan (v1.0, 2026-02-11) targeted carrier-direct from day one: SMPP connections via Ergo actor mesh, KumoMTA for email, gosmpp fork, HLR lookups. Expert reviews scored this approach 4/10 on feasibility and called the timeline "fiction."

Colleague feedback (Feb 2026) crystallized the pivot:

| Factor | Carrier-Direct MVP | Aggregator-First MVP |
|--------|-------------------|---------------------|
| **Time to market** | 20+ weeks | 4 weeks |
| **Carrier contracts** | 3-6 months to negotiate | Not needed |
| **SMPP expertise required** | Yes (team has none) | No |
| **IP warming for email** | 8 weeks minimum | SES handles it |
| **Danish telecom registration** | Criminal offense blocker | Aggregators are registered |
| **Infrastructure complexity** | SMPP mesh + MTA + HLR | REST API calls to providers |
| **First PoC customers** | 6-9 months | 4-8 weeks |
| **Margins still viable?** | Higher (eventually) | Yes (see Cost Analysis) |

The core product value is in the **cascade engine, multi-tenancy, and unified API** -- not in owning SMPP connections. Aggregators commoditize the last mile. Hermes commoditizes the orchestration layer on top.

### What Aggregators Provide

SMS aggregators (Twilio, Vonage, Messente, etc.) handle:

- Carrier contracts and relationships
- SMPP protocol connections to carriers
- Number routing and porting databases (HLR/MNP)
- Delivery receipt (DLR) processing and normalization
- Compliance (TCPA, GDPR opt-out lists)
- Number provisioning and sender ID management
- Throughput scaling (no per-carrier window management)

### What Amazon SES Provides

- Deliverability optimization (shared and dedicated IPs)
- IP reputation management and warming
- DKIM signing (user provides DNS records)
- SPF and DMARC alignment
- Bounce and complaint handling
- ISP feedback loops
- Sending statistics and deliverability dashboards
- Up to 50,000 emails/second per account (with limit increase)

Users set up their own DNS (SPF/DKIM/DMARC records). SES provides the verification flow and signing keys. This is standard practice -- every SES-based platform works this way.

### Path to Carrier-Direct (Phase 5+)

Carrier-direct connections become a **business decision**, not a technical prerequisite:

```
Trigger: Margins on aggregator SMS < X% (e.g., Twilio takes $0.0075/SMS, carrier-direct costs $0.001-0.003)
         AND monthly SMS volume > Y million (where carrier contracts make economic sense)
         AND team has built operational expertise with the platform

Action:  Implement Ergo actor mesh for SMPP (all research is done, architecture supports it)
         Swap aggregator WASM adapter -> SMPP actor adapter at the port interface level
         Keep aggregator as fallback path

Same logic applies to email: SES costs $0.10/1000 -> KumoMTA is effectively free at scale
```

The hexagonal architecture means this swap is a **port adapter change**, not a rewrite. The cascade engine, validation pipeline, router, and tenant management are provider-agnostic by design.

---

## Architecture Stack

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              INBOUND API GATEWAY                                │
│                                                                                 │
│   ┌──────────────────┐                    ┌──────────────────┐                  │
│   │     REST API      │                    │    Webhooks       │                  │
│   │  POST /v1/messages│                    │  (DLR / Status)   │                  │
│   │  GET  /v1/status  │                    │  Provider callbacks│                  │
│   │  (Chi router)     │                    │  (per-provider)    │                  │
│   └────────┬─────────┘                    └────────┬──────────┘                  │
│            │                                        │                            │
│            └──────────────┬─────────────────────────┘                            │
│                           │                                                      │
│               Unified Message Normalization                                      │
└───────────────────────────┬──────────────────────────────────────────────────────┘
                            │
┌───────────────────────────┴──────────────────────────────────────────────────────┐
│                      VALIDATION PIPELINE (9 layers)                              │
│                                                                                  │
│   Auth → Tenant → Schema → Content → Compliance → DNC → Rate → Dedup → Enrich  │
└───────────────────────────┬──────────────────────────────────────────────────────┘
                            │
┌───────────────────────────┴──────────────────────────────────────────────────────┐
│                 CASCADE ENGINE (Temporal.io / Restate)                            │
│                                                                                  │
│   ┌───────────────────────────────────────────────────────────────────────────┐  │
│   │  CascadeWorkflow (one per message)                                        │  │
│   │                                                                           │  │
│   │  ┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐    │  │
│   │  │ Channel 1 │────→│ Channel 2 │────→│ Channel 3 │────→│ Exhaust / │    │  │
│   │  │   (SMS)   │     │  (Email)  │     │  (Push)   │     │  Notify   │    │  │
│   │  └─────┬─────┘     └─────┬─────┘     └─────┬─────┘     └───────────┘    │  │
│   │        │                 │                 │                              │  │
│   │   Wait for DLR      Wait for DLR      Wait for DLR                      │  │
│   │   (Signal/Timer)    (Signal/Timer)    (Signal/Timer)                     │  │
│   │                                                                           │  │
│   │  Durable state  |  Timer-based fallback  |  Crash recovery automatic     │  │
│   └───────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────┬──────────────────────────────────────────────────────┘
                            │
┌───────────────────────────┴──────────────────────────────────────────────────────┐
│                            SMART ROUTER                                          │
│                                                                                  │
│   Tenant config → Provider health → Cost optimization → Provider selection       │
│   (embedded library, no network hop)                                             │
└───────────────────────────┬──────────────────────────────────────────────────────┘
                            │
                      NATS JetStream
                    (delivery stream)
                            │
┌───────────────────────────┴──────────────────────────────────────────────────────┐
│                   PROVIDER ADAPTERS (WASM / Wazero)                              │
│                                                                                  │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│   │ Twilio SMS   │  │ Amazon SES   │  │   FCM Push   │  │  WhatsApp    │       │
│   │   (.wasm)    │  │   (.wasm)    │  │   (.wasm)    │  │ Cloud API    │       │
│   │              │  │              │  │              │  │   (.wasm)    │       │
│   │ REST API     │  │ AWS SDK      │  │ REST API     │  │ REST API     │       │
│   └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                                                  │
│   ┌──────────────┐  ┌──────────────┐                                            │
│   │ Vonage SMS   │  │ Messente SMS │    Hot-swap  |  Sandboxed  |  Metered      │
│   │   (.wasm)    │  │   (.wasm)    │                                            │
│   └──────────────┘  └──────────────┘                                            │
└───────────────────────────┬──────────────────────────────────────────────────────┘
                            │
                    DLR / Status Webhooks
                    (callbacks from providers)
                            │
                    Temporal Signal → Update cascade state
                            │
                    Tenant webhook notification
                            │
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            INFRASTRUCTURE                                        │
│                                                                                  │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│   │  PostgreSQL  │  │     NATS     │  │    Redis     │  │   Temporal   │       │
│   │  (tenants,   │  │  JetStream   │  │  (cache,     │  │   Server     │       │
│   │  config,     │  │  (delivery   │  │   rate       │  │  (cascade    │       │
│   │  routing)    │  │   transport) │  │   limits)    │  │   state)     │       │
│   └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘       │
└──────────────────────────────────────────────────────────────────────────────────┘
                            │
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           OBSERVABILITY                                          │
│                                                                                  │
│   OpenTelemetry Traces  |  Prometheus Metrics  |  Grafana Dashboards             │
│   End-to-end message tracing across cascade steps and provider calls             │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Key Architecture Principles

1. **Hexagonal (Ports & Adapters):** Domain core (cascade, routing, validation) knows nothing about providers. Provider adapters implement port interfaces. Swapping Twilio for SMPP-direct is an adapter change.
2. **Durable Execution:** Temporal/Restate guarantees every message completes its cascade -- even through crashes, restarts, and network partitions.
3. **WASM Sandboxing:** Provider adapters run in Wazero with deny-by-default permissions. A buggy adapter cannot crash the host or access other tenants' data.
4. **NATS JetStream as Transport:** Decouples cascade engine from delivery. Activities publish to NATS (sub-ms), NATS consumers drive adapter execution. Provides backpressure and persistence.

---

## Project Structure

```
hermes/
├── cmd/
│   ├── hermes-api/                  # REST API server (Chi router)
│   └── hermes-cascade/              # Cascade engine workers (Temporal)
│
├── internal/
│   ├── domain/                      # Core domain types
│   │   ├── message.go               #   Message, Channel, Priority
│   │   ├── tenant.go                #   Tenant, TenantConfig, APIKey
│   │   ├── cascade.go               #   CascadeRule, CascadeStep, CascadeStatus
│   │   ├── provider.go              #   Provider, ProviderConfig, ProviderHealth
│   │   └── delivery.go              #   DeliveryReport, DeliveryStatus
│   │
│   ├── ports/                       # Port interfaces (hexagonal boundaries)
│   │   ├── message_port.go          #   MessageSubmitter, MessageQuerier
│   │   ├── cascade_port.go          #   CascadeExecutor, CascadeQuerier
│   │   ├── provider_port.go         #   ProviderAdapter, ProviderHealthChecker
│   │   ├── router_port.go           #   Router, RouteSelector
│   │   ├── tenant_port.go           #   TenantRepository, TenantAuthenticator
│   │   └── notification_port.go     #   WebhookNotifier
│   │
│   ├── cascade/                     # Cascade engine implementation
│   │   ├── workflow.go              #   CascadeWorkflow (Temporal)
│   │   ├── activities.go            #   SendMessage, NotifyTenant activities
│   │   └── signals.go               #   DLR signal handler
│   │
│   ├── router/                      # Smart Router
│   │   ├── router.go                #   Route selection logic
│   │   ├── cost.go                  #   Cost-based provider ranking
│   │   └── health.go                #   Provider health tracking
│   │
│   ├── adapters/
│   │   ├── wasm/                    # WASM adapter host (Wazero)
│   │   │   ├── host.go              #     Host functions (SendHTTP, Log, etc.)
│   │   │   ├── loader.go            #     WASM module loader + hot-swap
│   │   │   └── sandbox.go           #     Permission enforcement
│   │   └── webhook/                 # DLR/status webhook receiver
│   │       ├── handler.go           #     Per-provider webhook parsing
│   │       └── dispatcher.go        #     Route DLR to Temporal Signal
│   │
│   ├── security/                    # Auth and tenant context
│   │   ├── apikey.go                #   API key validation
│   │   ├── tenant_context.go        #   Request-scoped tenant context
│   │   └── middleware.go            #   Auth middleware
│   │
│   ├── validation/                  # 9-layer validation pipeline
│   │   ├── pipeline.go              #   Pipeline orchestrator
│   │   ├── auth.go                  #   Authentication validator
│   │   ├── tenant.go                #   Tenant status validator
│   │   ├── schema.go                #   Message schema validator
│   │   ├── content.go               #   Content policy validator
│   │   ├── compliance.go            #   Regulatory compliance
│   │   ├── dnc.go                   #   Do-Not-Contact list check
│   │   ├── ratelimit.go             #   Rate limit enforcer
│   │   ├── dedup.go                 #   Deduplication (idempotency)
│   │   └── enrich.go                #   Message enrichment
│   │
│   └── infra/                       # Infrastructure adapters
│       ├── postgres/                #   PostgreSQL repositories
│       ├── nats/                    #   NATS JetStream client
│       ├── redis/                   #   Redis cache client
│       └── temporal/                #   Temporal client setup
│
├── pkg/
│   ├── errors/                      # Error taxonomy (domain errors)
│   │   └── errors.go
│   └── observability/               # OpenTelemetry setup
│       ├── tracing.go
│       └── metrics.go
│
├── adapters/                        # WASM adapter source code
│   ├── twilio/                      #   Twilio SMS adapter
│   │   ├── main.go                  #     TinyGo source
│   │   └── twilio.wasm              #     Compiled WASM module
│   ├── vonage/                      #   Vonage SMS adapter
│   ├── messente/                    #   Messente SMS adapter
│   ├── ses/                         #   Amazon SES email adapter
│   ├── fcm/                         #   Firebase Cloud Messaging adapter
│   └── whatsapp/                    #   WhatsApp Cloud API adapter
│
├── deploy/
│   ├── docker/                      # Docker Compose (dev + demo)
│   │   ├── docker-compose.yml       #   PG, NATS, Redis, Temporal, Grafana
│   │   └── docker-compose.prod.yml  #   Production overrides
│   └── terraform/                   # Hetzner Cloud + RKE2 (production)
│       ├── main.tf                  #   terraform-hcloud-rke2 module
│       ├── variables.tf             #   Cluster config (node types, counts)
│       └── outputs.tf               #   Kubeconfig, IPs
│
├── docs/
│   ├── adr/                         # Architecture Decision Records
│   ├── plans/                       # This file
│   ├── research/                    # Technology research (Temporal, Ergo, WASM, etc.)
│   └── reviews/                     # Expert review findings
│
├── scripts/
│   ├── seed-tenant.sh               # Create test tenant + API key
│   └── load-test.sh                 # k6 or vegeta load test
│
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

### What Changed from v1.0

| Removed | Reason |
|---------|--------|
| `cmd/hermes-smpp/` | No SMPP connections in MVP |
| `cmd/hermes-gateway/` | No SMPP/SMTP inbound in MVP; REST-only |
| `internal/carrier/` | No Ergo actor mesh in MVP |
| `internal/gateway/` | No multi-protocol gateway in MVP |
| `internal/adapters/smpp/` | No SMPP adapter in MVP |
| `pkg/smpp/` | No gosmpp fork in MVP |
| `deploy/k8s/` | Kubernetes deferred to Phase 4 |
| `deploy/terraform/` | Terraform deferred to Phase 4 |
| `deploy/kumomta/` | No KumoMTA in MVP |

---

## Implementation Phases

### Phase 0: Foundation (Week 1)

**Goal:** Buildable Go workspace with infrastructure running and domain model defined.

| Task | Description | Owner |
|------|-------------|-------|
| Go workspace init | `go mod init`, directory structure, Makefile | Dev 1 |
| Docker Compose | PostgreSQL 16, NATS 2.10+, Redis 7, Temporal Server | Dev 3 |
| CI pipeline | GitHub Actions: lint, test, build | Dev 3 |
| Domain types | `Message`, `Tenant`, `CascadeRule`, `Channel`, `Provider`, `DeliveryReport` | Dev 1 |
| Port interfaces | All port interfaces in `internal/ports/` | Dev 1 |
| Error taxonomy | Structured error types with codes | Dev 2 |
| PostgreSQL schema | Tenants, provider configs, message log tables | Dev 2 |
| NATS streams | JetStream stream definitions for delivery | Dev 2 |
| Basic README | Setup instructions, architecture overview | Claude |

**Exit Criteria:**

- `go build ./...` passes with no errors
- `docker compose up` brings up PG, NATS, Redis, Temporal
- CI pipeline green on main branch
- Domain types and port interfaces compile and have godoc comments

---

### Phase 1: Cascade Engine + API (Week 2)

**Goal:** End-to-end message flow from API to cascade with simulated providers.

| Task | Description | Owner |
|------|-------------|-------|
| REST API | `POST /v1/messages`, `GET /v1/messages/{id}`, Chi router, JSON validation | Dev 2 |
| Validation pipeline | Auth + Rate + Schema layers (3 of 9 layers) | Dev 2 |
| CascadeWorkflow | Temporal workflow: 3-step cascade with timer-based fallback | Dev 1 |
| SendMessage activity | Publishes to NATS JetStream, NATS consumer invokes mock provider | Dev 1 |
| DLR webhook endpoint | Receives provider callbacks, dispatches Temporal Signal | Dev 1 |
| Mock providers | HTTP server simulating Twilio/SES/FCM responses + DLR callbacks | Dev 3 |
| NATS delivery consumer | Consumes from JetStream, calls provider adapter, publishes result | Dev 3 |
| Integration tests | Full cascade test: submit -> cascade 3 channels -> DLR -> complete | Claude |

**Exit Criteria:**

- `POST /v1/messages` triggers a 3-channel cascade
- Each cascade step waits for DLR, times out, and falls through to next channel
- DLR webhook signals Temporal workflow to mark delivery successful
- Message survives worker crash mid-cascade (kill worker, restart, cascade resumes)
- `GET /v1/messages/{id}` returns current cascade status

---

### Phase 2: Provider Adapters + Real Integration (Week 3)

**Goal:** Real messages sent through real providers via WASM adapters.

| Task | Description | Owner |
|------|-------------|-------|
| WASM host (Wazero) | `SendHTTP`, `Log`, `EmitMetric`, `GetConfig` host functions | Dev 1 |
| WASM loader | Module loading, hot-swap lifecycle, version tracking | Dev 1 |
| Twilio SMS adapter | TinyGo WASM: format Twilio API request, parse response/DLR | Dev 3 |
| Amazon SES adapter | Native Go adapter (AWS SDK v2 uses reflection, incompatible with TinyGo/WASM). Implements same `ProviderAdapter` port interface. Migrate to WASM if TinyGo support improves. | Dev 3 |
| FCM push adapter | TinyGo WASM: format FCM v1 API request, parse response | Claude |
| Smart Router (basic) | Tenant config -> channel -> provider selection | Dev 2 |
| Provider health tracking | Track success/failure rates per provider, circuit breaker | Dev 2 |
| Sandbox testing | Twilio test credentials, SES sandbox, FCM test project | Dev 3 |
| Adapter unit tests | Test each WASM adapter with recorded HTTP responses | Claude |

**Exit Criteria:**

- Real SMS sent via Twilio (test credentials / sandbox)
- Real email sent via Amazon SES (sandbox mode)
- Real push notification via FCM (test project)
- WASM adapters load, execute, and hot-swap without host restart
- Sandbox enforcement verified (WASM cannot access filesystem or arbitrary network)
- Smart Router selects provider based on tenant config

---

### Phase 3: Multi-Tenancy + Hardening (Week 4)

**Goal:** Multi-tenant demo with production-grade validation and observability.

| Task | Description | Owner |
|------|-------------|-------|
| Tenant CRUD | Create/update/delete tenants, manage API keys | Dev 2 |
| Tenant isolation | Request-scoped tenant context, data isolation in queries | Dev 2 |
| Rate limiting | Per-tenant, per-channel rate limits via Redis | Dev 2 |
| Full validation pipeline | Remaining 6 layers (tenant, content, compliance, DNC, dedup, enrich) | Dev 2 + Claude |
| OpenTelemetry tracing | End-to-end traces: API -> validation -> cascade -> provider -> DLR | Dev 1 |
| Prometheus metrics | Message counts, latencies, error rates, provider health, cascade outcomes | Dev 1 |
| Grafana dashboards | Message flow dashboard, provider health, tenant usage | Dev 3 |
| Load testing | k6 script: 1000 msg/sec sustained, measure latency distribution | Dev 3 |
| E2E integration tests | Multi-tenant scenarios, cascade fallback, provider failure | Claude |
| Docker Compose full stack | All services + Grafana + Prometheus in one `docker compose up` | Dev 3 |

**Exit Criteria:**

- Two tenants sending messages independently with no data leakage
- Rate limiting enforced per tenant
- 9-layer validation pipeline operational
- 1000 msg/sec load test passes with p99 < 2s end-to-end
- Grafana dashboard shows live message flow
- Full Docker Compose demo: `docker compose up` -> create tenant -> send messages -> view in Grafana

---

### Phase 4: Production Readiness (Weeks 5-8) -- Post-MVP

**Goal:** Harden MVP for production deployment and first customers.

| Task | Timeline | Description |
|------|----------|-------------|
| RKE2 cluster on Hetzner | Week 5 | Provision RKE2 cluster via `terraform-hcloud-rke2` (wenzel-felix) at Falkenstein (`fsn1`). HA control plane (3x CX33), dedicated workers (2x CPX31 + 1x CCX23). |
| Secrets management | Week 5 | HashiCorp Vault for API keys, provider credentials |
| TLS everywhere | Week 5 | mTLS between services, TLS for external APIs |
| Monitoring + alerting | Week 6 | Prometheus alerting rules (~30 alerts), PagerDuty/Slack |
| Sandbox environment | Week 6 | Isolated tenant sandbox for testing integrations |
| WhatsApp Cloud API adapter | Week 6 | WASM adapter for WhatsApp Business Cloud API |
| Additional SMS aggregators | Week 7 | Vonage and/or Messente WASM adapters |
| Restate prototype | Week 7-8 | Build CascadeWorkflow in Restate, compare with Temporal |
| Security audit | Week 8 | External review of auth, tenant isolation, WASM sandbox |
| Documentation | Week 8 | API docs (OpenAPI), onboarding guide, runbook |

**Exit Criteria:**

- RKE2 cluster running on Hetzner Cloud (Falkenstein `fsn1`) with HA control plane
- Secrets in Vault, no hardcoded credentials
- 4+ provider adapters operational (Twilio, SES, FCM, WhatsApp)
- Restate vs Temporal comparison documented with recommendation
- Security audit passed with no critical findings

---

### Phase 5: Carrier-Direct (Business Decision -- Future)

> This phase is triggered by business metrics, not a fixed timeline.

**Trigger criteria** (any of these may justify the investment):

- SMS aggregator margins below target threshold
- Monthly SMS volume exceeds N million (carrier contracts become economical)
- Customer requires carrier-direct for compliance/latency reasons
- Team has 6+ months of platform operational experience

**Scope (when triggered):**

| Task | Description |
|------|-------------|
| Ergo Framework actor mesh | SMPPSupervisor, CarrierSupervisor, ConnectionActor (all research done) |
| gosmpp fork | TLS wrapping, PDU validation, fuzz testing |
| SMPP adapter | Ergo actors replacing WASM adapter for SMPP-connected carriers |
| HLR/MNP lookups | Redis-cached, async refresh |
| KumoMTA deployment | Lua config, ISP throttling, IP warming (8-week process) |
| Carrier contracts | Telia, TDC, etc. (3-6 month negotiation) |

**Architecture impact:** Minimal. The hexagonal architecture means carrier-direct adapters implement the same `ProviderAdapter` port interface. The cascade engine, router, validation pipeline, and tenant management are completely unaffected.

---

## Key Technology Decisions

| Decision Area | MVP Choice | Future Option | Rationale |
|---------------|-----------|---------------|-----------|
| **Language** | Go | -- | Performance, concurrency, mature ecosystem |
| **Cascade Engine** | Temporal.io | Restate (prototype in Phase 4) | Temporal proven at Twilio scale (billions of messages). Go SDK mature (v1.31+). |
| **Provider Adapters** | WASM via Wazero | Ergo actors for SMPP (Phase 5) | Sandboxed, hot-swappable, ~57ns call overhead. TinyGo compilation. |
| **Message Transport** | NATS JetStream | -- | Sub-ms publish, persistent streams, proven at scale, built-in backpressure |
| **State Persistence** | Temporal (cascade) + PostgreSQL (config) | -- | Eliminates custom event sourcing. Temporal handles durable state. |
| **SMS Delivery** | Twilio / Vonage / Messente (aggregator API) | Carrier-direct SMPP (Phase 5) | Aggregators handle carrier contracts, routing, compliance. |
| **Email Delivery** | Amazon SES | KumoMTA (Phase 5) | SES handles deliverability, IP reputation, DKIM, bounces. No 8-week IP warming. |
| **Push Notifications** | Firebase Cloud Messaging (FCM) | -- | De facto standard, free tier, reliable |
| **WhatsApp** | Meta Cloud API | -- | No BSP partnership needed. Cloud API is sufficient. |
| **API Framework** | Chi router (net/http) | -- | Minimal overhead, composable middleware, no framework lock-in |
| **Auth** | API keys | OAuth2 / mTLS (Phase 4) | API keys for MVP speed. Progressive security. |
| **Rate Limiting** | Redis (sliding window) | -- | Sub-ms checks, per-tenant counters |
| **Monitoring** | OpenTelemetry + Prometheus + Grafana | -- | Industry standard. End-to-end trace correlation. |
| **Secrets** | Environment variables | HashiCorp Vault (Phase 4) | Env vars for MVP. Vault for production rotation and audit. |
| **Deployment** | Docker Compose | RKE2 on Hetzner Cloud (Phase 4) | Docker Compose for dev. RKE2 on Hetzner for production (~€121/mo vs ~$1,350/mo on AWS). |
| **Infrastructure** | Local | Hetzner Cloud, Falkenstein `fsn1` (Phase 4) | 87% cheaper than AWS. 20TB traffic included. GDPR-compliant (Germany). Terraform: `wenzel-felix/rke2/hcloud`. |

---

## What We're NOT Building in MVP (and Why)

| Not Building | Why | When It Makes Sense |
|-------------|-----|-------------------|
| SMPP carrier connections | Aggregators handle carrier protocols. Team has no SMPP experience. | Phase 5: when margins justify carrier contracts |
| Own MTA (KumoMTA) | SES handles deliverability, reputation, DKIM, bouncing. IP warming takes 8 weeks. | Phase 5: when email volume makes SES costs significant |
| HLR/MNP lookups | Aggregators handle number routing and porting databases. | Phase 5: with carrier-direct, need own routing |
| Ergo actor supervision trees | Only needed for stateful protocol connections (SMPP). Aggregator APIs are stateless HTTP. | Phase 5: actor mesh for SMPP connection management |
| SMPP/SMTP inbound gateway | MVP is REST-only inbound. Enterprise SMPP inbound is a niche requirement. | Phase 4+: when enterprise customers require it |
| Multi-DC active-active | Single DC with HA is sufficient for 22M/day. Premature optimization. | When customer SLAs or geography require it |
| ML-based routing | Need 6+ months of historical delivery data before ML adds value over rules. | After accumulating delivery data across providers |
| WhatsApp BSP | Cloud API is sufficient. BSP requires Meta partnership and compliance burden. | Only if > $5M ARR and Cloud API limits hit |
| Custom event sourcing | Temporal/Restate provide durable execution out of the box. | Never -- this is the whole point of Temporal |
| IP reputation engine | SES manages IP reputation. Not our problem in MVP. | Phase 5: with own MTA |

---

## What We Preserve for Phase 2+

All research completed to date remains valid and directly applicable:

| Research Area | Document | Applicability |
|--------------|----------|---------------|
| Temporal.io cascade patterns | `docs/research/temporal-deep-dive.md` | **Phase 1 (now)** -- cascade engine implementation |
| Ergo Framework actor mesh | `docs/research/actor-model-wasm.md` | **Phase 5** -- SMPP carrier mesh when carrier-direct |
| WASM/Wazero adapter strategy | `docs/research/actor-model-wasm.md` | **Phase 2 (now)** -- provider adapters |
| KumoMTA evaluation | `docs/research/technology-investigation.md` | **Phase 5** -- own MTA when email volume justifies |
| Encryption strategy (Signal Protocol) | `docs/research/encryption-strategy.md` | **Future** -- end-to-end encryption offering |
| 9-layer validation pipeline | `docs/adr/ADR-001-hermes-platform-architecture.md` | **Phase 1-3 (now)** -- validation pipeline |
| Expert review findings | `docs/reviews/` | **Ongoing** -- architecture quality gates |

### Phase 5+ Product Vision: Email as a Platform

Beyond owning the delivery lifecycle (KumoMTA, SMPP-direct), Hermes can build **value-added services on top of email** that aggregators and SES don't offer:

| Capability | Description | Competitive Advantage |
|-----------|-------------|----------------------|
| **Email Security Scanning** | Phishing detection, malware link scanning, content threat analysis on outbound | Customers trust Hermes not just for delivery but for protecting their brand |
| **Encryption Services** | S/MIME, PGP, or custom encryption for sensitive email (healthcare, finance) | Enterprise differentiator — no aggregator offers this |
| **Compliance Engine** | GDPR data residency enforcement, retention policies, audit trails per jurisdiction | Regulatory selling point for EU customers |
| **Analytics & Engagement** | Open tracking, click tracking, engagement scoring, deliverability insights | Deeper analytics than SES provides out of the box |
| **Hosted SMTP Service** | Customers point their SMTP at Hermes instead of configuring SES directly | Simpler DX, stickier product, full control over the email pipeline |
| **Abuse Prevention** | Outbound content scanning, reputation monitoring, automatic throttling | Protects shared IP reputation across tenants |

This transforms Hermes from a **delivery broker** into an **email intelligence platform** — a much larger TAM and stronger moat than pure message routing.

> **Trigger:** Pursue this when MVP is stable and customer feedback validates demand for email features beyond basic delivery.

### Architecture Guarantees for Future Expansion

The hexagonal architecture ensures these swaps are **adapter changes, not rewrites:**

```
MVP (Phase 1-3)                          Future (Phase 5+)
─────────────────                        ─────────────────
Twilio WASM adapter  ──────────────→     SMPP Ergo actor adapter
  (implements ProviderAdapter)             (implements ProviderAdapter)

Amazon SES WASM adapter  ─────────→     KumoMTA adapter
  (implements ProviderAdapter)             (implements ProviderAdapter)

Redis HLR stub  ──────────────────→     Real HLR/MNP lookup + Redis cache
  (implements RouteSelector)               (implements RouteSelector)
```

The cascade engine, validation pipeline, tenant management, API gateway, and observability layer are **completely unchanged** when swapping providers.

---

## Risk Register

### Technical Risks

| # | Risk | Severity | Probability | Mitigation |
|---|------|----------|-------------|------------|
| T1 | Temporal operational complexity (cluster + PostgreSQL + Elasticsearch) | Medium | Medium | Start with PostgreSQL backend (simplest). Evaluate Temporal Cloud if self-hosted is too heavy. Restate as alternative (single binary). |
| T2 | TinyGo WASM compilation limits (no full reflect, limited stdlib) | Medium | Medium | Adapters are simple HTTP transforms -- within TinyGo capability. **Exception: Amazon SES adapter must be native Go** (AWS SDK v2 uses reflection). Other adapters (Twilio, FCM, WhatsApp) use simple REST and compile fine with TinyGo. |
| T3 | NATS JetStream tuning for high throughput | Low | Low | Well-documented. Start with defaults, tune stream/consumer config under load test. |
| T4 | Wazero WASI gaps (host function limitations) | Low | Low | Only need `SendHTTP`, `Log`, `GetConfig`, `EmitMetric`. WASI preview 1 sufficient. |
| T5 | `numHistoryShards` immutable in Temporal | High | Medium | Set correctly before production data (2048-4096 recommended). Cannot change later without migration. |
| T6 | Provider webhook reliability (DLR delivery not guaranteed) | Medium | Medium | Implement DLR timeout with cascade fallback. Periodic polling for stuck messages. Reconciliation job. |
| T7 | WASM adapter cold start latency | Low | Low | Wazero benchmarked at p50=10ms, p99=30ms (Arcjet production). Pre-warm adapters on startup. |

### Business Risks

| # | Risk | Severity | Probability | Mitigation |
|---|------|----------|-------------|------------|
| B1 | Aggregator pricing changes (Twilio raises rates) | Medium | Medium | Support multiple aggregators from launch. WASM adapter per provider. Switch routing in minutes. |
| B2 | SES sending limit too low for launch | Medium | Low | Default 200/sec, need to request increase. Apply early (takes 24-48 hours). Dedicated IPs available. |
| B3 | Aggregator rate limits throttle throughput | Medium | Medium | Multi-provider routing distributes load. Per-provider rate limit tracking in Smart Router. |
| B4 | Customer perception: "just a wrapper" | Medium | Medium | Communicate value: cascade engine, multi-tenancy, unified observability, compliance. These are hard problems. |
| B5 | Analysis paralysis (85% probability per mentor review) | High | High | This plan exists to prevent it. Ship MVP in 4 weeks. Iterate based on real customer feedback. |
| B6 | Provider lock-in concern | Low | Low | WASM adapters abstract providers. Hexagonal architecture ensures swappability. Demonstrate with multi-provider routing. |
| B7 | Danish telecom registration for aggregator model | Medium | Unknown | Aggregator-only model may still require registration if Hermes transforms/manipulates messages. **Legal must evaluate immediately in Week 1.** Criminal offense risk if operating without required registration. |

### Mitigated Risks (vs. v1.0 Plan)

These risks from the carrier-direct plan are **eliminated** by the aggregator-first approach:

| Risk (from v1.0) | Status | Why |
|-------------------|--------|-----|
| Carrier contracts take 3-6 months | **Eliminated** | Aggregators have contracts |
| Danish telecom registration (criminal offense) | **Eliminated** | Aggregators are registered operators |
| IP warming takes 8 weeks | **Eliminated** | SES handles reputation |
| No team SMPP experience | **Eliminated** | No SMPP in MVP |
| gosmpp single-maintainer / no security audit | **Eliminated** | No gosmpp in MVP |
| Ergo v3 is new (Feb 2026) | **Deferred** | No Ergo in MVP |
| KumoMTA young open source | **Eliminated** | Using SES instead |

---

## Team Allocation

### 3 Developers + Claude (4-Week MVP Sprint)

| Role | Focus Areas | Key Skills |
|------|-------------|------------|
| **Dev 1 (Lead)** | Architecture, cascade engine (Temporal), WASM host (Wazero), OpenTelemetry | Go, distributed systems, Temporal SDK, system design |
| **Dev 2** | REST API, validation pipeline, multi-tenancy, Smart Router, rate limiting | Go, API design, PostgreSQL, Redis, security |
| **Dev 3** | WASM adapters (Twilio/SES/FCM), provider integration, Docker Compose, CI/CD, load testing | Go, TinyGo, Docker, infrastructure, third-party API integration |
| **Claude** | Code generation, unit tests, integration tests, adapter scaffolding, documentation | Everything, 24/7 availability |

### Week-by-Week Allocation

| Week | Dev 1 | Dev 2 | Dev 3 | Claude |
|------|-------|-------|-------|--------|
| **1** | Go workspace, domain types, port interfaces | Error taxonomy, PostgreSQL schema, NATS streams | Docker Compose, CI pipeline | README, test scaffolding, godoc |
| **2** | CascadeWorkflow (Temporal), DLR signals | REST API, validation (3 layers) | Mock providers, NATS consumer | Integration tests, mock helpers |
| **3** | WASM host (Wazero), module loader | Smart Router, provider health | Twilio + SES + FCM adapters | FCM adapter, adapter unit tests |
| **4** | OpenTelemetry tracing, Prometheus metrics | Tenant CRUD, rate limiting, full validation | Grafana dashboards, load testing, full Docker Compose | E2E tests, remaining validation layers, docs |

---

## Cost Analysis

### Aggregator Costs (SMS)

| Provider | Cost per SMS (US) | Cost per SMS (EU/DK) | DLR Included | Notes |
|----------|-------------------|----------------------|--------------|-------|
| **Twilio** | $0.0079 | $0.04-0.09 (varies by country) | Yes | Most mature API, best docs |
| **Vonage (Nexmo)** | $0.0068 | $0.03-0.08 | Yes | Good EU coverage |
| **Messente** | $0.005-0.01 | $0.02-0.06 | Yes | Strong Nordic/EU focus, competitive pricing |
| **Carrier-direct** (future) | $0.001-0.003 | $0.005-0.02 | Yes | Requires contracts, SMPP, compliance |

**Margin example at early stage (1M SMS/month, EU):**

| Path | Cost per SMS | Total Cost | Hermes Price | Gross Margin |
|------|-------------|------------|--------------|--------------|
| Messente (aggregator) | ~$0.04 | $40,000 | $0.06 | $20,000/mo (33%) |
| Carrier-direct (future) | ~$0.01 | $10,000 | $0.06 | $50,000/mo (83%) |

**Margin example at scale (100M SMS/month, EU):**

| Path | Cost per SMS | Total Cost | Hermes Price | Gross Margin |
|------|-------------|------------|--------------|--------------|
| Messente (aggregator) | ~$0.04 | $4,000,000 | $0.06 | $2,000,000/mo (33%) |
| Carrier-direct (future) | ~$0.01 | $1,000,000 | $0.06 | $5,000,000/mo (83%) |

Even at 1M SMS/month with aggregator pricing, margins are $20K/month — viable from day one. Carrier-direct is a **margin optimization**, not a business enabler. At scale, the margin delta ($3M/month) justifies the Phase 5 investment.

### Amazon SES Costs (Email)

| Component | Cost | Notes |
|-----------|------|-------|
| Sending | $0.10 per 1,000 emails | First 62,000/month free (from EC2) |
| Dedicated IPs | $24.95/month per IP | Optional, for high-volume senders |
| Receiving | $0.10 per 1,000 emails | If accepting inbound |
| Attachments | $0.12 per GB | Outbound data |

**At 6B emails/year (500M/month):**

| Component | Monthly Cost |
|-----------|-------------|
| Sending (500M x $0.10/1000) | $50,000 |
| Dedicated IPs (10 IPs) | $250 |
| **Total SES** | **~$50,250/month** |
| **Hermes price (at $0.20/1000)** | **$100,000/month** |
| **Gross margin** | **~$50,000/month (50%)** |

KumoMTA would reduce the $50K/month to infrastructure costs (~$5K/month for servers), but that optimization can wait.

### Infrastructure Costs: Hetzner Cloud + RKE2

| Phase | Configuration | EUR/month | USD equivalent |
|-------|--------------|-----------|----------------|
| **MVP (Weeks 1-4)** | Local Docker Compose | **€0** | $0 |
| **Staging (Week 5)** | 1x CX43 + 1x CX33 + storage + LB | **~€29-45** | ~$32-49 |
| **Production v1 (Week 6+)** | 3x CX33 (CP) + 2x CPX31 + 1x CCX23 + 1x CX33 + storage + LB | **~€121** | ~$131 |
| **Production at scale** | 11 nodes, all CCX dedicated vCPU | **~€399** | ~$432 |
| **Dedicated alternative** | 3x AX42 bare metal (24 cores, 192GB) | **~€224** | ~$242 |

**Production v1 breakdown (€121/month):**

| Role | Server | Specs | Count | EUR/mo |
|------|--------|-------|-------|--------|
| Control Plane (HA) | CX33 | 4 vCPU, 8GB | 3 | €16.47 |
| Workers (stateless) | CPX31 | 4 dedicated vCPU, 8GB | 2 | €32.98 |
| Workers (stateful: PG, Redis, NATS) | CCX23 | 4 dedicated vCPU, 16GB | 1 | €24.49 |
| Monitoring | CX33 | 4 vCPU, 8GB | 1 | €5.49 |
| Block Storage | Volumes | 500GB total | 3 | €22.00 |
| Load Balancer | LB21 | Medium | 1 | €16.40 |
| Floating IP | IPv4 | Stable ingress | 1 | €3.57 |

**Hetzner vs AWS comparison (production v1 equivalent):**

| Provider | Monthly Cost | Annual Cost | Savings |
|----------|-------------|-------------|---------|
| **Hetzner** | ~€158 (~$170) | ~€1,900 | **87% cheaper** |
| AWS | ~$1,350 | ~$16,200 | Baseline |
| GCP | ~$1,280 | ~$15,360 | 5% cheaper than AWS |

> **Why Hetzner:** 20TB traffic included (AWS charges ~$0.09/GB egress). Germany = GDPR. Dedicated servers available at €49/month (8 cores, 64GB DDR5 ECC, NVMe). See `docs/research/hetzner-infrastructure.md` for full analysis.

Compare to carrier-direct v1.0 plan: infrastructure alone would cost $5K+/month on AWS. Aggregator-first on Hetzner runs production for ~€121/month.

---

## Decision Points for Team

These decisions require team consensus before Phase 0 begins:

### Must Decide Now

| # | Decision | Options | Recommendation |
|---|----------|---------|----------------|
| 1 | **Accept aggregator-first strategy?** | (a) Aggregator-first MVP, carrier-direct later (b) Carrier-direct from day one | **(a)** -- 4-week MVP vs. 20+ weeks, eliminated risks, margins still viable |
| 2 | **Accept Temporal as cascade engine?** | (a) Temporal only (b) Temporal + Restate prototype (c) Restate only | **(a)** for MVP, **(b)** in Phase 4 -- Temporal is proven at Twilio scale, Restate prototype can inform future |
| 3 | **Accept WASM for provider adapters?** | (a) WASM/Wazero (b) Native Go plugins (c) gRPC microservices | **(a)** -- sandboxed, hot-swappable, ~57ns overhead, no container orchestration |
| 4 | **Which SMS aggregator(s) to start with?** | (a) Twilio only (b) Twilio + Messente (c) Messente only | **(b)** -- Twilio for best docs/sandbox, Messente for competitive EU pricing |
| 5 | **SES region selection?** | (a) eu-west-1 (Ireland) (b) eu-central-1 (Frankfurt) (c) us-east-1 | **(a)** or **(b)** -- EU data residency for GDPR compliance |

### Decide During MVP

| # | Decision | Decide By | Context |
|---|----------|-----------|---------|
| 6 | Restate vs. Temporal long-term | End of Phase 4 (Week 8) | Build Restate prototype, compare operationally |
| 7 | When to evaluate carrier-direct | Quarterly review | Based on volume, margins, team expertise |
| 8 | Hetzner Cloud vs Dedicated servers | Phase 4 (Week 5) | Start Cloud (flexibility), evaluate AX42 dedicated (€49/mo, 8c/64GB) at stable load |
| 9 | Dedicated SES IPs vs. shared | Phase 3 (Week 4) | Depends on customer email volume and deliverability requirements |

---

## Sources

### Core Technologies (MVP)

- [Temporal.io Go SDK](https://github.com/temporalio/sdk-go) -- Durable execution, cascade engine
- [Temporal Documentation](https://docs.temporal.io/) -- Workflow, Activity, Signal, Timer patterns
- [Restate Go SDK](https://github.com/restatedev/sdk-go) -- Alternative cascade engine (Phase 4 prototype)
- [Wazero](https://wazero.io/) -- Zero-dependency WebAssembly runtime for Go
- [TinyGo](https://tinygo.org/) -- Go compiler targeting WASM
- [NATS JetStream](https://docs.nats.io/nats-concepts/jetstream) -- Persistent message streaming
- [Chi Router](https://github.com/go-chi/chi) -- Lightweight HTTP router for Go

### Provider APIs (MVP)

- [Twilio SMS API](https://www.twilio.com/docs/sms/api) -- SMS aggregator
- [Vonage SMS API](https://developer.vonage.com/en/messaging/sms/overview) -- SMS aggregator
- [Messente API](https://messente.com/documentation) -- SMS aggregator (EU focus)
- [Amazon SES v2 API](https://docs.aws.amazon.com/ses/latest/APIReference-V2/) -- Email delivery
- [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging) -- Push notifications
- [WhatsApp Cloud API](https://developers.facebook.com/docs/whatsapp/cloud-api/) -- WhatsApp messaging

### Infrastructure

- [Hetzner Cloud](https://www.hetzner.com/cloud/) -- EU cloud hosting (87% cheaper than AWS)
- [terraform-hcloud-rke2](https://github.com/wenzel-felix/terraform-hcloud-rke2) -- RKE2 on Hetzner Terraform module
- [RKE2](https://docs.rke2.io/) -- Rancher Kubernetes Engine 2 (CIS-hardened)
- [Hetzner Infrastructure Research](docs/research/hetzner-infrastructure.md) -- Full pricing, layouts, gotchas

### Observability

- [OpenTelemetry Go](https://opentelemetry.io/docs/languages/go/) -- Distributed tracing and metrics
- [Prometheus](https://prometheus.io/) -- Metrics collection and alerting
- [Grafana](https://grafana.com/) -- Dashboards and visualization

### Deferred Technologies (Phase 5+ Research)

- [Ergo Framework v3](https://github.com/ergo-services/ergo) -- OTP-style actor mesh for SMPP
- [KumoMTA](https://kumomta.com) -- High-performance MTA for email
- [gosmpp](https://github.com/linxGnu/gosmpp) -- SMPP client library for Go
- [knqyf263/go-plugin](https://github.com/knqyf263/go-plugin) -- WASM plugin framework

---

*Generated: 2026-02-16 | Revision: 2.0 (Aggregator-First Pivot) | Status: PENDING TEAM APPROVAL*
