# Project Hermes: Development Timeline

> **Status:** PENDING TEAM APPROVAL
> **Date:** 2026-02-16
> **Supersedes:** Original 20-week timeline from implementation-plan.md
> **Strategic Pivot:** MVP via telecom aggregators (Twilio, Vonage) + Amazon SES, not direct carrier connections

---

## Timeline Overview

```
Week:  1       2       3       4       5       6       7       8       9-12      13-16     17+
       ├───────┼───────┼───────┼───────┼───────┼───────┼───────┼───────┼─────────┼─────────┼──────
Ph 0   ████████                                                                              Foundation
Ph 1           ████████                                                                      Cascade + API
Ph 2                   ████████                                                              Provider Adapters
Ph 3                           ████████                                                      Multi-Tenancy
       ─────────────────────────────────
                 M V P   D E M O  (Week 4)
       ─────────────────────────────────
Ph 4                                   ████████████████████████████████                      Production Ready
                                                                       ─────────────────
                                                                        P R O D   v 1
                                                                       ─────────────────
Ph 5                                                                             ████████   Carrier-Direct*

       ── PARALLEL BUSINESS TRACKS ──────────────────────────────────────────────────────
BIZ    ████████████████████████████████████████████████████████████████████████████████████   Contracts + Legal
CUST           ████████████████████████████████████████████████████████████████████████████   Customer Acquisition

 * Carrier-Direct = business decision only if margins justify direct contracts
```

### What Changed from the Original Plan

The original implementation plan (2026-02-11) assumed direct carrier connections (SMPP/Ergo) and self-hosted KumoMTA from the start. The strategic pivot:

| Original Plan (20 weeks to MVP)       | New Plan (4 weeks to MVP)                 |
|---------------------------------------|-------------------------------------------|
| Direct SMPP to Telia/TDC              | Twilio/Vonage aggregator APIs             |
| Ergo actor mesh for carrier mesh      | Deferred to Phase 5 (if ever)             |
| KumoMTA self-hosted email             | Amazon SES                                |
| gosmpp fork + security audit          | Not needed for MVP                        |
| IP warming (8-week blocker)           | SES handles deliverability                |
| Carrier contracts required            | Aggregator signup = same day              |

**Result:** MVP timeline compressed from ~20 weeks to 4 weeks. Carrier-direct becomes an optimization, not a prerequisite.

---

## Phase Breakdown

### Phase 0: Foundation (Week 1)

| Attribute          | Detail |
|--------------------|--------|
| **Duration**       | 1 week |
| **Dependencies**   | None (this is the starting point) |

**Team Focus:**
- Dev 1 (Lead): Go workspace setup, domain types, port interfaces
- Dev 2: Error taxonomy, PostgreSQL schema, NATS stream definitions
- Dev 3: Docker Compose environment (PostgreSQL, NATS, Redis, Temporal), CI/CD pipeline

**Claude Focus:**
- Generate domain types: `Tenant`, `Message`, `CascadeRule`, `Channel`, `Provider`, `DeliveryReport`
- Generate port interfaces: `MessagePort`, `CascadePort`, `AdapterPort`, `TenantPort`, `RouterPort`
- Generate error taxonomy with typed error codes per channel
- Generate CI pipeline (lint, test, build, Docker)
- Generate Docker Compose with health checks

**Deliverables:**
- [ ] Go workspace with `cmd/`, `internal/`, `pkg/` structure
- [ ] Domain types compile (`internal/domain/`)
- [ ] Port interfaces defined (`internal/ports/`)
- [ ] Error taxonomy (`pkg/errors/`)
- [ ] Docker Compose: PostgreSQL + NATS JetStream + Redis + Temporal Server
- [ ] CI pipeline: `go vet`, `go test`, `go build`, Docker build
- [ ] Makefile with standard targets

**Exit Criteria:**
- `go build ./...` passes with zero errors
- `docker compose up` brings all services healthy
- CI pipeline runs green on main branch
- Domain types have godoc comments and basic validation

**Risk:**
| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Temporal Docker setup issues | Low | 1-day delay | Temporal has official Docker Compose; fallback to Temporal Cloud free tier |
| Go module dependency conflicts | Low | Hours | Pin all versions from day 1 |

---

### Phase 1: Cascade Engine + API (Week 2)

| Attribute          | Detail |
|--------------------|--------|
| **Duration**       | 1 week |
| **Dependencies**   | Phase 0 complete (Go workspace, Docker Compose, domain types) |

**Team Focus:**
- Dev 1 (Lead): Temporal CascadeWorkflow implementation, DLR signal handling
- Dev 2: NATS JetStream stream setup, delivery consumers, DLR webhook endpoint
- Dev 3: REST API (`POST /v1/messages`, `GET /v1/messages/{id}`), request validation

**Claude Focus:**
- Generate Temporal workflow: `CascadeWorkflow` with channel-ordered activities
- Generate Temporal activities: `SendMessageActivity` (publishes to NATS)
- Generate NATS JetStream stream configuration and consumer workers
- Generate mock providers (return configurable success/failure/timeout)
- Generate REST API handlers with OpenAPI spec
- Generate integration tests: cascade through 3 channels, crash recovery

**Deliverables:**
- [ ] `POST /v1/messages` accepts message, returns message ID
- [ ] `GET /v1/messages/{id}` returns message status
- [ ] CascadeWorkflow: Email -> SMS -> Push (configurable order)
- [ ] Timer-based fallback: if no DLR within timeout, try next channel
- [ ] DLR webhook: `POST /v1/webhooks/dlr` signals Temporal workflow
- [ ] NATS JetStream: `hermes.delivery.sms`, `hermes.delivery.email`, `hermes.delivery.push`
- [ ] Mock providers: configurable latency, success rate, DLR delay
- [ ] Crash recovery: kill worker mid-cascade, restart, message completes

**Exit Criteria:**
- Message cascades Email -> SMS -> Push when channels fail (measured via mock providers)
- Crash recovery: kill `hermes-cascade` worker, restart, workflow resumes from last step
- DLR webhook triggers workflow completion/fallback
- p99 cascade step latency < 500ms (measured via OpenTelemetry)
- Integration test suite passes

**Risk:**
| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Temporal learning curve | Medium | 1-2 day delay | Claude generates scaffolding; team learns by reviewing generated code |
| NATS JetStream configuration | Low | Hours | Well-documented; Claude generates config from official docs |

---

### Phase 2: Provider Adapters (Week 3)

| Attribute          | Detail |
|--------------------|--------|
| **Duration**       | 1 week |
| **Dependencies**   | Phase 1 complete (Cascade engine, NATS streams, mock providers) |

**Team Focus:**
- Dev 1 (Lead): WASM adapter host (Wazero runtime), adapter lifecycle management
- Dev 2: Twilio SMS adapter + Amazon SES email adapter (real sandbox accounts)
- Dev 3: FCM push adapter, Smart Router (tenant config -> provider selection)

**Claude Focus:**
- Generate Wazero host runtime with host functions: `SendHTTP`, `Log`, `EmitMetric`, `GetConfig`
- Generate adapter plugin interface: `OnMessage`, `OnDeliveryReport`, `OnConfigure`, `OnHealthCheck`
- Generate Twilio SMS adapter (.wasm) targeting sandbox API
- Generate Amazon SES email adapter (.wasm) targeting sandbox
- Generate FCM push adapter (.wasm)
- Generate Smart Router: read tenant config, select provider, handle fallback
- Generate adapter integration tests with real sandbox accounts

**Deliverables:**
- [ ] Wazero adapter host with host function bridge
- [ ] Twilio SMS adapter (.wasm) -- real SMS sent via sandbox
- [ ] Amazon SES email adapter (.wasm) -- real email sent via sandbox
- [ ] FCM push adapter (.wasm) -- real push notification sent
- [ ] Smart Router: tenant config -> provider selection logic
- [ ] Adapter health checks and error reporting
- [ ] End-to-end: `POST /v1/messages` -> cascade -> real SMS + email delivery

**Exit Criteria:**
- Real SMS received on a test phone via Twilio sandbox
- Real email received in inbox via Amazon SES sandbox
- Push notification received on test device via FCM
- Smart Router correctly selects provider based on tenant configuration
- WASM adapters load, configure, and health-check successfully
- Adapter errors propagate to cascade engine (triggering fallback)

**Risk:**
| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| WASM/TinyGo compilation issues | Medium | 1-2 day delay | Fallback: native Go adapters initially, migrate to WASM later |
| Provider sandbox rate limits | Medium | Testing bottleneck | Use mock providers for load testing; sandbox for functional tests only |
| SES sandbox: manual recipient verification | Low | Minor friction | Pre-verify test recipients; request production access early |

---

### Phase 3: Multi-Tenancy + Hardening (Week 4)

| Attribute          | Detail |
|--------------------|--------|
| **Duration**       | 1 week |
| **Dependencies**   | Phase 2 complete (working adapters, Smart Router) |

**Team Focus:**
- Dev 1 (Lead): Validation pipeline (9 layers), rate limiting per tenant
- Dev 2: Tenant management (CRUD, API key generation/rotation), tenant isolation
- Dev 3: Observability (OpenTelemetry traces, Prometheus metrics), E2E + load tests

**Claude Focus:**
- Generate tenant management: create/read/update/delete, API key hashing, key rotation
- Generate validation pipeline: Auth -> Tenant -> Schema -> Content -> Compliance -> DNC -> Rate -> Dedup -> Enrich
- Generate rate limiter: per-tenant sliding window (Redis-backed)
- Generate OpenTelemetry instrumentation for all critical paths
- Generate Prometheus metrics: message counts, latencies, error rates, per-tenant usage
- Generate E2E test suite: multi-tenant scenarios
- Generate load test: target 1,000 msg/sec sustained

**Deliverables:**
- [ ] Tenant CRUD API: `POST/GET/PUT/DELETE /v1/tenants`
- [ ] API key management: generate, rotate, revoke
- [ ] Validation pipeline: 9 layers, configurable per tenant
- [ ] Rate limiting: per-tenant, per-channel, sliding window
- [ ] OpenTelemetry: distributed traces across API -> Cascade -> Adapter
- [ ] Prometheus metrics: ~15 key metrics exposed at `/metrics`
- [ ] E2E tests: 2+ tenants, cascade fallback, DLR delivery, tenant isolation
- [ ] Load test: 1,000 msg/sec sustained for 5 minutes, zero message loss

**Exit Criteria:**
- Two tenants can send messages independently with no data leakage
- Rate limiting enforced: tenant exceeding limit receives 429
- Validation pipeline rejects malformed/unauthorized requests
- Load test: 1,000 msg/sec for 5 min, p99 < 2s, zero message loss
- OpenTelemetry traces visible in Jaeger/Tempo
- Prometheus metrics scrapable

**Risk:**
| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Load test reveals bottleneck | Medium | 1-2 day tuning | Profile early; NATS JetStream and Temporal both proven at higher throughput |
| Tenant isolation gaps | Low | Security issue | Claude generates isolation tests; review all DB queries for tenant scoping |

---

### MVP Milestone (End of Week 4)

```
 ┌────────────────────────────────────────────────────────────────────┐
 │                      M V P   D E M O                              │
 │                      End of Week 4                                │
 ├────────────────────────────────────────────────────────────────────┤
 │                                                                    │
 │  WHAT IT INCLUDES:                                                 │
 │  - Single API: POST /v1/messages (SMS + Email + Push)             │
 │  - Cascade fallback: configurable channel ordering                 │
 │  - Real delivery via Twilio (SMS), SES (Email), FCM (Push)        │
 │  - Delivery reports via webhook                                    │
 │  - Multi-tenant with API keys                                      │
 │  - Rate limiting per tenant                                        │
 │  - 1,000 msg/sec throughput                                        │
 │  - Crash recovery (Temporal durable execution)                     │
 │                                                                    │
 │  WHAT IT DEMONSTRATES:                                             │
 │  - "One API to rule them all" value proposition                    │
 │  - Cascade intelligence: automatic fallback across channels        │
 │  - Reliability: message delivery survives infrastructure crashes   │
 │  - Multi-tenancy: tenant isolation, per-tenant rate limiting       │
 │  - Performance: 1K msg/sec with sub-2s p99 latency                │
 │                                                                    │
 │  WHAT IT DOES NOT INCLUDE:                                         │
 │  - Production security hardening (Vault, TLS everywhere)          │
 │  - Kubernetes deployment                                           │
 │  - Monitoring dashboards (Grafana)                                 │
 │  - Alerting rules                                                  │
 │  - Customer self-service onboarding                                │
 │  - Direct carrier connections (SMPP)                               │
 │  - Custom email infrastructure (KumoMTA)                           │
 │  - WhatsApp channel                                                │
 │                                                                    │
 │  WHO CAN SEE IT:                                                   │
 │  - PoC customers (controlled demo)                                 │
 │  - Investors (capability demonstration)                            │
 │  - Internal team (load test results, architecture walkthrough)     │
 │                                                                    │
 └────────────────────────────────────────────────────────────────────┘
```

---

### Phase 4: Production Readiness (Weeks 5-8)

| Attribute          | Detail |
|--------------------|--------|
| **Duration**       | 4 weeks |
| **Dependencies**   | MVP milestone complete (Week 4) |

**Team Focus:**
- Dev 1 (Lead): Security hardening (Vault, TLS, auth), Restate evaluation (if Temporal chosen for MVP)
- Dev 2: Kubernetes manifests, Helm charts, staging environment, deployment automation
- Dev 3: Grafana dashboards, alerting rules, customer sandbox, additional adapters

**Claude Focus:**
- Generate Vault integration: secret storage, rotation, dynamic credentials
- Generate Kubernetes manifests: Deployments, Services, ConfigMaps, HPA
- Generate Helm chart for single-command deployment
- Generate Grafana dashboards (~15 panels): throughput, latency, errors, per-tenant usage
- Generate alerting rules (~15 alerts): delivery failure rate, cascade timeout rate, provider health
- Generate WhatsApp Cloud API adapter (.wasm)
- Generate additional aggregator adapters: Vonage (.wasm), Messente (.wasm)
- Generate customer sandbox environment with test mode
- Generate Restate prototype (if team wants to evaluate against Temporal)

**Deliverables:**

*Weeks 5-6 (Security + Infrastructure):*
- [ ] HashiCorp Vault: provider API keys, tenant secrets, auto-rotation
- [ ] TLS everywhere: API, internal services, database connections
- [ ] Auth hardening: API key scoping, request signing, IP allowlisting
- [ ] Kubernetes manifests: all services deployable to K8s
- [ ] Helm chart: `helm install hermes ./deploy/helm`
- [ ] Staging environment: deployed and accessible

*Weeks 7-8 (Observability + Onboarding):*
- [ ] Grafana dashboards: ~15 panels covering system and business metrics
- [ ] Alerting rules: ~15 alerts (PagerDuty/Slack integration)
- [ ] Customer sandbox: test mode, simulated delivery, no real messages sent
- [ ] WhatsApp Cloud API adapter (.wasm)
- [ ] Vonage SMS adapter (.wasm)
- [ ] Messente SMS adapter (.wasm)
- [ ] Customer onboarding flow: signup -> API key -> sandbox -> production
- [ ] Restate evaluation report (if prototyped)

**Exit Criteria:**
- Production Kubernetes cluster running all services
- Vault managing all secrets (zero hardcoded credentials)
- TLS on every connection (internal and external)
- Grafana dashboards showing real-time system health
- Alerting fires correctly (tested with synthetic failures)
- First customer can onboard via sandbox, then promote to production
- WhatsApp messages deliverable via Cloud API

**Risk:**
| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Kubernetes complexity | Medium | 1-week delay | Start with simple manifests; Helm chart for repeatability |
| Vault operational overhead | Low | Days | Use managed Vault or simple Docker deployment for staging |
| SES sending limit increase | Medium | Email throughput capped | Request production access in Week 3; start with low volume |
| Temporal at scale | Medium | Performance unknowns | Load test in staging; tune worker count, history shard count |

---

### Phase 5: Carrier-Direct (Weeks 9+, Business Decision)

| Attribute          | Detail |
|--------------------|--------|
| **Duration**       | 8+ weeks (if pursued) |
| **Dependencies**   | Production v1 running, carrier contracts signed, business case validated |

> **This phase is a business decision, not a technical inevitability.**
> Only pursue if: direct carrier margins justify the contract negotiations,
> compliance overhead, and engineering investment. All Phase 5 work leverages
> research already completed (see `docs/research/`).

**Triggers for Pursuing Phase 5:**
- Aggregator margins too thin at current volume (Twilio takes ~40-60% margin)
- Customer demand for carrier-specific features (sender ID, premium routes)
- Volume exceeds 5M+ SMS/month (direct contracts become cost-effective)
- Specific carrier SLAs required by enterprise customers

**Team Focus:**
- Dev 1 (Lead): Ergo Framework actor mesh for SMPP connections
- Dev 2: gosmpp fork, TLS hardening, security audit, HLR lookups
- Dev 3: KumoMTA deployment (if email volume justifies vs SES), Smart Router upgrade

**Claude Focus:**
- Generate Ergo supervision tree: `SMPPSupervisor`, `CarrierSupervisor`, `ConnectionActor`, `DLRProcessor`
- Generate gosmpp fork with TLS wrapper, PDU validation, fuzz tests
- Generate HLR lookup integration (Redis-cached)
- Generate Smart Router upgrade: carrier capacity awareness, least-cost routing
- Generate KumoMTA Lua configuration (if email path chosen)
- Generate IP warming schedule (if KumoMTA path chosen)

**Deliverables (if pursued):**
- [ ] Ergo actor mesh: SMPP connections to 1+ Danish carrier
- [ ] gosmpp fork: TLS, PDU validation, fuzz testing, security audit
- [ ] HLR lookups: number portability, carrier identification
- [ ] Smart Router: least-cost routing, carrier capacity, failover to aggregator
- [ ] KumoMTA: deployed, Lua-configured, IP warming started (if email path chosen)

**Exit Criteria:**
- SMS delivered via direct SMPP connection to at least one carrier
- Supervisor auto-restart: connection failure recovers within 5 seconds
- Backpressure: gen.Stage throttling under load, zero message loss
- Cost per SMS measurably lower than aggregator path
- Graceful fallback: if carrier connection fails, route through aggregator

---

## Milestones Table

| # | Milestone | Target | Criteria | Owner |
|---|-----------|--------|----------|-------|
| M1 | Foundation complete | End Week 1 | `go build` passes, CI green, Docker Compose healthy | All devs |
| M2 | Cascade working | End Week 2 | 3-channel cascade, crash recovery, DLR handling | Dev 1 |
| M3 | Provider integration | End Week 3 | Real SMS + email sent via Twilio + SES sandbox | Dev 2 |
| M4 | **MVP Demo** | **End Week 4** | **Multi-tenant, 1K msg/sec, cascade fallback** | **All devs** |
| M5 | Security hardened | End Week 6 | Vault, TLS, auth hardening complete | Dev 1 |
| M6 | Staging deployed | End Week 6 | K8s cluster running, Helm chart working | Dev 2 |
| M7 | Observability live | End Week 7 | Grafana dashboards, alerting rules active | Dev 3 |
| M8 | **Production v1** | **End Week 8** | **First customer live, monitoring, alerting** | **All devs** |
| M9 | Carrier-direct eval | Week 9+ | Business decision: pursue or defer | Team + Business |

---

## Parallel Work Streams

```
Week:  1       2       3       4       5       6       7       8       9+
       ├───────┼───────┼───────┼───────┼───────┼───────┼───────┼───────┼──────

DEV    ████████████████████████████████████████████████████████████████████████
       Ph0     Ph1     Ph2     Ph3     Ph4 (Security+Infra) Ph4 (Obs+Onboard)

LEGAL  ████████████████████████████████
       Danish telecom registration evaluation
       (Determine if aggregator-only model requires registration)

BIZ-1  ████████████████████████████████████████████████████████████████████████
       Aggregator contracts: Twilio, Vonage, Messente
       (Sandbox = instant; production accounts by Week 3)

BIZ-2          ████████████████████████████████████████████████████████████████
               Customer acquisition: PoC discussions, pilot agreements
               (Demo-ready at Week 4)

BIZ-3                                                          ████████████████
                                                               Carrier contract
                                                               exploration
                                                               (only if Phase 5)

SES                    ████████████████
                       SES production access request + domain verification
                       (Request early; approval takes days to weeks)

IP-WARM                                                        ████████████████
                                                               IP warming
                                                               (8 weeks, only if
                                                               KumoMTA chosen)
```

### Key Parallel Actions by Week

| Week | Dev Track | Business Track |
|------|-----------|----------------|
| 1 | Foundation setup | Twilio/Vonage account setup; legal: telecom registration evaluation |
| 2 | Cascade engine | SES production access request; customer outreach begins |
| 3 | Provider adapters (sandbox) | SES domain verification; aggregator production accounts |
| 4 | Multi-tenancy + load test | MVP demo to first PoC prospects |
| 5-6 | Security + K8s | Customer pilot agreements; pricing model finalization |
| 7-8 | Observability + onboarding | First customer onboarding; feedback collection |
| 9+ | Carrier-direct (if decided) | Carrier contract negotiations (if decided) |

---

## Resource Requirements

### Infrastructure Costs by Phase

| Phase | Environment | Monthly Cost | Notes |
|-------|-------------|-------------|-------|
| 0-3 (Weeks 1-4) | Local dev (Docker Compose) | ~$0 | All services run locally |
| 4 (Weeks 5-8) | Staging K8s cluster | ~$500-1,000/mo | Small cluster: 3 nodes, managed K8s |
| 4 (Weeks 5-8) | Production K8s cluster | ~$2,000-3,000/mo | Production-grade: 3-5 nodes, HA |
| 5+ (Weeks 9+) | Production at scale | ~$3,000-5,000/mo | Depends on message volume and carrier infra |

### Provider Costs (Pay-as-you-go)

| Provider | Cost | Phase | Notes |
|----------|------|-------|-------|
| Twilio SMS | ~$0.0079/msg (US), varies by country | Week 3+ | Sandbox is free; production per-message |
| Amazon SES | ~$0.10 per 1,000 emails | Week 3+ | Free tier: 62,000/month from EC2 |
| FCM Push | Free | Week 3+ | No per-message cost |
| WhatsApp Cloud API | $0.005-0.10/msg (varies by country) | Week 7+ | 1,000 free conversations/month |
| Temporal | Self-hosted (included in K8s cost) | Week 1+ | No licensing cost |
| NATS | Self-hosted (included in K8s cost) | Week 1+ | No licensing cost |

### Team Cost Assumptions

| Resource | Allocation | Notes |
|----------|-----------|-------|
| Dev 1 (Lead) | Full-time | Architecture, cascade engine, security |
| Dev 2 | Full-time | Infrastructure, adapters, carrier work |
| Dev 3 | Full-time or part-time | API, DevOps, observability |
| Claude Opus 4.6 | Continuous | Code generation, tests, adapters, reviews |

---

## Risk Timeline

| # | Risk | When | Impact | Prob. | Mitigation |
|---|------|------|--------|-------|------------|
| R1 | Temporal learning curve | Week 2 | 1-2 day delay | Medium | Claude generates scaffolding; team learns by reviewing. Official Go SDK is mature. |
| R2 | Provider sandbox rate limits | Week 3 | Testing bottleneck | Medium | Use mock providers for load testing; sandbox only for functional verification. |
| R3 | WASM/TinyGo compilation issues | Week 3 | Adapter delays (1-2 days) | Medium | Fallback: ship native Go adapters initially, migrate to WASM post-MVP. |
| R4 | SES sending limit increase delay | Weeks 4-5 | Email throughput capped | Medium | Request production access in Week 2; start with low volume; SES scales gradually. |
| R5 | Load test reveals bottleneck | Week 4 | 1-2 day tuning | Medium | Profile early; both NATS and Temporal proven at much higher throughput. |
| R6 | Temporal at production scale | Weeks 5-8 | Performance unknowns | Medium | Start with PostgreSQL persistence; plan Cassandra migration if needed. Set `numHistoryShards` correctly (2048+). |
| R7 | Kubernetes deployment complexity | Weeks 5-6 | 1-week delay | Medium | Start with simple manifests; Claude generates Helm chart; use managed K8s. |
| R8 | Customer onboarding friction | Weeks 7-8 | Slow adoption | Low | Build sandbox environment; provide SDK examples; Claude generates client libraries. |
| R9 | Danish telecom registration required | Week 1 (legal) | Blocker if required | Low (for aggregator model) | Evaluate immediately; aggregator-only model may not require registration. |
| R10 | Analysis paralysis | Any week | Schedule slip | High | This timeline exists to prevent it. Ship MVP in 4 weeks. Decide, don't deliberate. |

---

## Decision Gates

Decisions that must be made at specific points. Delaying these decisions delays everything downstream.

```
Week 1  ─── GATE 1 ────────────────────────────────────────────────────────────
         Decision: Temporal for MVP cascade engine, Restate prototype deferred to Phase 4?
         Owner: Dev 1 (Lead)
         Input needed: Temporal Docker setup experience from Phase 0
         Impact of delay: Week 2 work cannot start without this
         Note: Team previously chose "prototype both" — this gate confirms sequencing (Temporal first, Restate later)

Week 2  ─── GATE 2 ────────────────────────────────────────────────────────────
         Decision: WASM adapters or native Go adapters for MVP?
         Owner: Dev 1 (Lead)
         Input needed: TinyGo compilation test results
         Impact of delay: Week 3 adapter work blocked
         Fallback: Native Go adapters (can migrate to WASM later)

Week 4  ─── GATE 3 ────────────────────────────────────────────────────────────
         Decision: MVP scope freeze — what ships in demo?
         Owner: Team
         Input needed: Load test results, provider integration status
         Impact of delay: Phase 4 priorities unclear

Week 6  ─── GATE 4 ────────────────────────────────────────────────────────────
         Decision: Production infrastructure choices
         Owner: Team
         Decisions: Cloud provider, region, managed vs self-hosted K8s
         Impact of delay: Staging/production deployment blocked

Week 8  ─── GATE 5 ────────────────────────────────────────────────────────────
         Decision: Carrier-direct — yes / no / later?
         Owner: Team + Business
         Input needed: Customer feedback, margin analysis, volume projections
         Impact of delay: Phase 5 planning cannot begin

Week 8  ─── GATE 6 ────────────────────────────────────────────────────────────
         Decision: KumoMTA vs keep SES for email?
         Owner: Team + Business
         Input needed: Email volume projections, cost comparison
         Threshold: KumoMTA only justified if > 2B emails/year
         Impact of delay: None (SES continues to work)
```

---

## What "Done" Looks Like at Each Stage

### MVP (End of Week 4)

**A customer can:**
- Create an account and receive an API key
- Send an SMS via `POST /v1/messages` with `channel: "sms"`
- Send an email via `POST /v1/messages` with `channel: "email"`
- Send a push notification via `POST /v1/messages` with `channel: "push"`
- Configure a cascade: "try email, then SMS, then push"
- Receive delivery reports via webhook

**The system handles:**
- Cascade fallback when a channel fails or times out
- Crash recovery (Temporal durable execution)
- Delivery reports from providers
- Rate limiting per tenant
- Request validation (auth, schema, content)

**What is missing (and that is acceptable):**
- Production-grade security (no Vault, limited TLS)
- No monitoring dashboards
- No alerting
- No Kubernetes deployment (Docker Compose only)
- No customer self-service onboarding
- No custom domains for email
- No WhatsApp channel
- No direct carrier connections

---

### Production v1 (End of Week 8)

**A customer can:**
- Self-service onboard: signup -> sandbox -> production
- Send at scale across SMS, email, push, and WhatsApp
- Receive delivery reports via webhook
- View usage via API
- Rotate API keys without downtime

**The system handles:**
- Everything from MVP, plus:
- Multi-tenancy with strict isolation
- Full observability (traces, metrics, dashboards)
- Alerting on failures and anomalies
- Secret management via Vault
- TLS on all connections
- Kubernetes deployment with auto-scaling
- Multiple aggregator providers (Twilio, Vonage, Messente)

**What is missing (and that is a conscious choice):**
- Direct carrier connections (SMPP) -- business decision pending
- Custom email infrastructure (KumoMTA) -- volume does not yet justify
- ML-based routing -- need 6+ months of delivery data first
- RCS channel -- market adoption insufficient
- Multi-datacenter deployment -- single-DC with HA is sufficient

---

### Carrier-Direct (Week 9+, if pursued)

**Additional capabilities:**
- Direct SMPP connections to Danish carriers (Telia, TDC)
- HLR lookups for number portability and carrier identification
- Least-cost routing: direct when cheaper, aggregator as fallback
- Self-hosted email via KumoMTA (if volume justifies)
- Lower per-message costs on high-volume routes

**Business justification required:**
- Direct carrier margin improvement > cost of contracts + engineering
- Customer demand for carrier-specific features
- Volume threshold: 5M+ SMS/month makes direct contracts cost-effective

---

## Sprint Cadence

| Week | Monday | Wednesday | Friday |
|------|--------|-----------|--------|
| Every week | Sprint planning (30 min) | Mid-week sync (15 min) | Demo + retrospective (45 min) |
| Week 4 | MVP scope freeze | Final integration testing | **MVP Demo** |
| Week 8 | Production readiness review | Customer onboarding dry run | **Production v1 Go/No-Go** |

---

## Appendix: Claude's Role by Phase

Claude Opus 4.6 operates as a force multiplier, not a replacement for engineering judgment. The team reviews, tests, and owns all generated code.

| Phase | Claude Generates | Team Reviews & Owns |
|-------|-----------------|---------------------|
| 0 | Domain types, interfaces, CI pipeline, Docker Compose | Architecture decisions, module structure |
| 1 | Temporal workflows, NATS config, REST handlers, integration tests | Cascade logic, failure handling, API design |
| 2 | WASM host, provider adapters, Smart Router, sandbox tests | Provider API integration, adapter correctness |
| 3 | Tenant management, validation pipeline, rate limiter, load tests | Security model, tenant isolation, performance tuning |
| 4 | K8s manifests, Helm chart, Grafana dashboards, alerting rules | Infrastructure choices, security hardening, customer onboarding |
| 5 | Ergo actors, gosmpp fork, HLR integration, KumoMTA config | Carrier contracts, SMPP protocol correctness, security audit |

**Estimated Claude impact:** 2-3x development velocity. A 2-developer team with Claude can deliver what typically requires 4-6 developers, primarily through:
- Boilerplate generation (adapters, tests, CI/CD, K8s manifests)
- Rapid prototyping (try both Temporal and Restate in the time it takes to try one)
- Test generation (integration tests, load tests, chaos tests)
- Documentation (API specs, runbooks, architecture docs)

---

*Generated: 2026-02-16 | Status: PENDING TEAM APPROVAL*
*This timeline supersedes the original 20-week plan from implementation-plan.md*
*Next action: Team review and approval before Phase 0 begins*
