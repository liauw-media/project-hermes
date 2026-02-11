# Project Hermes: Next-Generation Architecture Plan

> **Status:** PENDING TEAM APPROVAL
> **Generated:** 2026-02-11
> **Scope:** Multi-tenant, multi-channel message delivery broker in Go

---

## Table of Contents

1. [Context & Motivation](#context--motivation)
2. [The Three Breakthroughs](#the-three-breakthroughs)
3. [What This Eliminates](#what-this-eliminates)
4. [Architecture Stack](#architecture-stack)
5. [Project Structure](#project-structure)
6. [Implementation Phases](#implementation-phases)
7. [Key Technology Decisions](#key-technology-decisions)
8. [What We're NOT Building (and Why)](#what-were-not-building-and-why)
9. [Honest Timeline: 2-3 Devs + Claude](#honest-timeline-2-3-devs--claude)
10. [Verification Plan](#verification-plan)
11. [Expert Review Scores (Context)](#expert-review-scores-context)
12. [Sources](#sources)
13. [Decision Required](#decision-required)

---

## Context & Motivation

Before writing any Go code, three research agents investigated modern technologies to determine whether the architecture should copy traditional 2010-era telecom patterns or leverage newer technology. The investigation covered: Temporal.io, Restate, Ergo Framework, WASM/Wazero, KumoMTA, actor models, CRDTs, edge computing, eBPF, and ML-based routing.

**Key finding: Three technologies can eliminate 60-70% of the custom infrastructure code** that expert reviews (scoring 2/10 on state persistence, 2/10 on backpressure) identified as the hardest problems.

### Team Decisions Made During Planning

| Decision Area       | Outcome                                                             |
|---------------------|---------------------------------------------------------------------|
| Philosophy          | Maximum Innovation                                                  |
| Innovation Focus    | Carrier Connectivity                                                |
| Cascade Engine      | Prototype both Temporal and Restate                                 |
| Adapter Strategy    | Hybrid (WASM for REST adapters, Ergo actors for protocol adapters)  |

---

## The Three Breakthroughs

### 1. Temporal.io / Restate = Cascade Engine

Twilio literally runs every message through Temporal. The cascade engine (scored 2/10 on state persistence by architect review) is a solved problem via durable execution.

|                      | Temporal.io                                          | Restate                                              |
|----------------------|------------------------------------------------------|------------------------------------------------------|
| **Scale proof**      | Twilio, Netflix, Snap, Instacart (75M/month)         | 94K actions/sec benchmarked                          |
| **Per-step latency** | 50-150ms (regular), 10-30ms (local activity)         | 3ms (low load), 10ms (high load)                     |
| **Go SDK**           | Mature (v1.31+), first-class                         | New (late 2025), functional                          |
| **Persistence**      | Cassandra/PostgreSQL/MySQL                           | Built-in RocksDB (no external DB)                    |
| **Operations**       | Heavy (cluster + DB + ES)                            | Light (single Rust binary)                           |
| **Cascade fit**      | Workflow = cascade, Activity = send, Signal = DLR    | Virtual Object per message, awakeable = DLR          |
| **At 22M/day**       | Proven (Twilio does more)                            | Needs validation at this scale                       |
| **Self-hosted cost** | $5K-10K/mo infra + DevOps                            | Lower (single binary)                                |
| **Cloud cost**       | $132K-540K/mo (disqualified)                         | No cloud offering yet                                |

**Hybrid recommended:** Temporal for cascade orchestration + NATS JetStream for high-throughput delivery transport. Activities publish to NATS (sub-ms), NATS consumers handle actual provider delivery. DLR webhooks signal back to Temporal workflows.

---

### 2. Ergo Framework = Carrier Mesh

Ergo v3.2.0 (Feb 2026) brings genuine OTP patterns ported to Go with 21M msg/sec:

- **gen.Server** -- stateful actor (each SMPP connection)
- **gen.Supervisor** -- auto-restart with 4 strategies (OFO/AFO/RFO/SOFO)
- **gen.Stage** -- demand-driven backpressure (from Elixir's GenStage)
- **gen.Saga** -- distributed transactions with recovery
- **Meta Processes (TCP)** -- bridge blocking SMPP I/O with actor messaging
- Zero dependencies, 4.4K stars, MIT, actively maintained

#### Carrier Mesh Supervision Tree

```
SMPPSupervisor (gen.Supervisor, One-For-One)
├── CarrierSupervisor("telia-dk", gen.Supervisor)
│   ├── ConnectionActor (gen.Server + TCP Meta Process)
│   │   └── owns gosmpp session, TLS wrapper, window tracker
│   ├── ConnectionActor (gen.Server + TCP Meta Process)
│   └── DLRProcessor (gen.Server, correlates carrier→internal IDs)
├── CarrierSupervisor("tdc", gen.Supervisor)
│   └── ConnectionActor ...
├── HealthAggregator (gen.Server, publishes mesh health)
└── BackpressureManager (gen.Stage, demand-driven flow control)
```

**Why this beats connection pools:** supervisor auto-restarts, gen.Stage backpressure, actor-local state (no locks), mailbox depth as load signal, escalation on N restarts.

---

### 3. WASM/Wazero + KumoMTA

**Wazero:** production-ready (Arcjet: p50=10ms, p99=30ms), zero dependencies, ~57ns function call overhead, deny-by-default sandboxing.

**KumoMTA:** Rust-based open-source MTA, 7M+ messages/hour per node, Lua scripting config. ISP throttling, IP warming, bounce handling, DKIM built in. Apache 2 licensed.

#### WASM Adapter Interfaces

```go
// AdapterHost -- functions the host exposes to WASM plugins
type AdapterHost interface {
    SendHTTP(url string, headers map[string]string, body []byte) ([]byte, error)
    Log(level string, msg string)
    EmitMetric(name string, value float64)
    GetConfig(key string) string
    StoreGet(key string) []byte
    StoreSet(key string, value []byte)
}

// AdapterPlugin -- functions each WASM adapter must implement
type AdapterPlugin interface {
    OnMessage(msg []byte) ([]byte, error)
    OnDeliveryReport(report []byte) ([]byte, error)
    OnConfigure(config []byte) error
    OnHealthCheck() bool
}
```

**Hybrid adapter strategy:** WASM for stateless HTTP (SendGrid, Twilio REST, FCM, WhatsApp Cloud API), Ergo actors for stateful protocols (SMPP, SMTP/MTA).

---

## What This Eliminates

| Traditional Approach                                   | Next-Gen Replacement                     | Lines Saved               |
|--------------------------------------------------------|------------------------------------------|---------------------------|
| Custom event-sourced cascade state machine             | Temporal/Restate workflow                | ~3,000-5,000              |
| NATS KV for cascade state (won't scale to 153M keys)  | Temporal/Restate persistence             | Problem disappears        |
| Pod crash = message in quantum state                   | Durable execution guarantee              | Recovery code gone        |
| Docker + gRPC + Ed25519 + heartbeats for adapters      | WASM plugins via Wazero                  | ~2,000                    |
| Connection pool manager + health check polling         | Ergo supervision tree                    | ~1,500                    |
| Custom backpressure chain (5 layers)                   | gen.Stage + Temporal rate limiting        | 3 of 5 layers natural     |
| Manual DLR correlation store                           | Actor-local in DLRProcessor              | Simpler                   |
| Build own MTA from scratch                             | KumoMTA (Lua-configured)                 | Entire MTA codebase       |
| Custom IP warming scheduler                            | KumoMTA built-in                         | ~500                      |
| Custom bounce classification                           | KumoMTA built-in                         | ~1,000                    |

---

## Architecture Stack

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           INBOUND PROTOCOL GATEWAY                              │
│                                                                                 │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│   │   REST API   │  │ SMPP Server  │  │ SMTP Server  │  │  Webhooks    │       │
│   │  (net/http)  │  │   (gosmpp)   │  │  (go-smtp)   │  │  (DLR/MO)   │       │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│          └─────────────────┴─────────────────┴─────────────────┘               │
│                                    │                                            │
│                        Unified Message Normalization                            │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┴────────────────────────────────────────────┐
│                          VALIDATION PIPELINE (9 layers)                         │
│                                                                                 │
│   Auth → Tenant → Schema → Content → Compliance → DNC → Rate → Dedup → Enrich │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┴────────────────────────────────────────────┐
│                     CASCADE ENGINE (Temporal or Restate)                        │
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐       │
│   │  CascadeWorkflow                                                    │       │
│   │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐        │       │
│   │  │ Channel 1│──→│ Channel 2│──→│ Channel 3│──→│ Fallback │        │       │
│   │  │  (SMS)   │   │ (Email)  │   │  (Push)  │   │ (Notify) │        │       │
│   │  └──────────┘   └──────────┘   └──────────┘   └──────────┘        │       │
│   │                                                                     │       │
│   │  Durable state  |  Timer-based fallback  |  DLR signal handling    │       │
│   └─────────────────────────────────────────────────────────────────────┘       │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┴────────────────────────────────────────────┐
│                              SMART ROUTER                                       │
│                                                                                 │
│   Route Selection: Carrier capacity → HLR cache → Cost optimization            │
│   Embedded library (no network hop)                                             │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
                    ┌────────────────┴────────────────┐
                    │                                  │
┌───────────────────┴──────────────────┐  ┌───────────┴──────────────────────────┐
│        CARRIER MESH (Ergo)           │  │       WASM ADAPTER LAYER (Wazero)    │
│                                      │  │                                      │
│  ┌────────────────────────────────┐  │  │  ┌──────────┐  ┌──────────────────┐  │
│  │ SMPPSupervisor                 │  │  │  │ SendGrid │  │ Twilio REST API  │  │
│  │  ├── CarrierSupervisor(telia)  │  │  │  │  (.wasm) │  │     (.wasm)      │  │
│  │  │    ├── ConnectionActor x N  │  │  │  └──────────┘  └──────────────────┘  │
│  │  │    └── DLRProcessor         │  │  │  ┌──────────┐  ┌──────────────────┐  │
│  │  ├── CarrierSupervisor(tdc)    │  │  │  │   FCM    │  │ WhatsApp Cloud   │  │
│  │  ├── HealthAggregator          │  │  │  │  (.wasm) │  │     (.wasm)      │  │
│  │  └── BackpressureManager       │  │  │  └──────────┘  └──────────────────┘  │
│  └────────────────────────────────┘  │  │                                      │
│                                      │  │  Hot-swap | Sandboxed | Metered      │
│  Stateful SMPP/TCP connections       │  │  Stateless HTTP adapters             │
└──────────────────────────────────────┘  └──────────────────────────────────────┘
                    │                                  │
                    └────────────────┬────────────────┘
                                     │
┌────────────────────────────────────┴────────────────────────────────────────────┐
│                        EMAIL DELIVERY (KumoMTA)                                 │
│                                                                                 │
│   Lua-configured  |  ISP throttling  |  IP warming  |  DKIM/SPF/DMARC          │
│   Bounce handling |  7M+ msg/hr/node |  Apache 2.0                              │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┴────────────────────────────────────────────┐
│                            INFRASTRUCTURE                                       │
│                                                                                 │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│   │PostgreSQL│  │   NATS   │  │  Redis   │  │  Vault   │  │ KumoMTA  │        │
│   │ (tenant, │  │JetStream │  │ (HLR,   │  │(secrets, │  │  (email  │        │
│   │  config) │  │(delivery)│  │  cache)  │  │  creds)  │  │  relay)  │        │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┴────────────────────────────────────────────┐
│                           OBSERVABILITY                                         │
│                                                                                 │
│   OpenTelemetry Traces  |  Prometheus Metrics  |  Grafana Dashboards            │
│   ~45 metrics           |  ~30 alerts          |  Distributed tracing           │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
hermes/
├── cmd/
│   ├── hermes-api/              # REST API server
│   ├── hermes-cascade/          # Cascade engine workers
│   ├── hermes-smpp/             # SMPP Connection Manager (Ergo)
│   └── hermes-gateway/          # Protocol Gateway
├── internal/
│   ├── domain/                  # Core domain types
│   ├── ports/                   # Port interfaces
│   ├── cascade/                 # Cascade engine (Temporal/Restate)
│   ├── router/                  # Smart Router (embedded library)
│   ├── carrier/                 # Carrier Mesh (Ergo actors)
│   ├── gateway/                 # Protocol Gateway (REST, SMPP, SMTP)
│   ├── adapters/
│   │   ├── wasm/                # WASM adapter host (Wazero)
│   │   ├── sendgrid/            # SendGrid WASM adapter
│   │   ├── twilio/              # Twilio WASM adapter
│   │   └── smpp/                # SMPP adapter (Ergo + gosmpp)
│   ├── security/                # TenantSecurityContext, auth, TLS
│   ├── validation/              # 9-layer validation pipeline
│   └── infra/                   # Database, NATS, Redis, Vault
├── pkg/
│   ├── smpp/                    # Hardened gosmpp fork wrapper
│   ├── errors/                  # Error Taxonomy
│   └── observability/           # OpenTelemetry setup
├── adapters/                    # WASM adapter source -> .wasm
│   ├── sendgrid/
│   └── twilio/
├── deploy/
│   ├── docker/                  # Docker Compose
│   ├── k8s/                     # Kubernetes manifests
│   ├── terraform/               # IaC
│   └── kumomta/                 # KumoMTA Lua config
└── docs/
```

---

## Implementation Phases

### Phase 0: Foundation (Week 1)

Go workspace, CI, tooling, domain types, port interfaces, error taxonomy, Docker Compose (PostgreSQL, NATS, Redis, Temporal/Restate).

**Exit criteria:** `go build ./...` passes, Docker Compose up, domain types compile, CI green.

---

### Phase 1: Cascade Engine Prototypes (Weeks 2-4)

- **Temporal prototype:** CascadeWorkflow, SendMessageActivity, DLR signal handler, timer timeout
- **Restate prototype:** Virtual Object, durable RPC, awakeable
- **Shared:** NATS JetStream streams, webhook DLR endpoint, REST API
- **Comparison criteria:** code simplicity, latency (OTel), crash recovery, operational complexity, debugging

**Deliverable:** `POST /v1/messages` triggers cascade across 3 channels with simulated DLR and crash recovery.

---

### Phase 2: Carrier Mesh + SMPP (Weeks 5-8)

- Fork gosmpp, add TLS wrapping, PDU validation, fuzz testing
- Ergo actors: ConnectionActor, CarrierSupervisor, SMPPSupervisor, WindowTracker, DLRProcessor, HealthAggregator, BackpressureManager
- SMPP simulator testing
- Smart Router: capacity queries, HLR cache, least-cost routing

**Deliverable:** Route SMS through actor mesh to SMPP simulator. Supervisor auto-restart on failure. Backpressure under load.

---

### Phase 3: WASM Adapters + Protocol Gateway (Weeks 9-12)

- Wazero host: SendHTTP, Log, EmitMetric, GetConfig, StoreGet/Set
- go-plugin protobuf interface, hot-swap lifecycle
- Adapters: SendGrid, Twilio, WhatsApp Cloud API, FCM
- Protocol Gateway: SMPP server (inbound), SMTP server, unified message normalization

**Deliverable:** REST/SMPP/SMTP inbound. Multi-channel cascade. Hot-swap WASM adapters at runtime.

---

### Phase 4: KumoMTA + Email (Weeks 13-16)

- KumoMTA deployment and Lua config (ISP throttling, DKIM, SPF/DMARC, bounce)
- MTASupervisor: IPPoolManager, BounceProcessor, HealthMonitor
- IP warming begins (8-week process)

**Deliverable:** Email delivery via KumoMTA with DKIM/SPF. IP warming underway.

---

### Phase 5: Security + Production Hardening (Weeks 17-20)

- **Secrets:** HashiCorp Vault (carrier creds, DKIM keys, HLR tokens, API key encryption)
- **Security:** TLS everywhere, PDU fuzz hardening, multi-protocol auth unification
- **Abuse prevention:** MTA abuse rules, network segmentation
- **Observability:** OpenTelemetry traces, Prometheus metrics, Grafana dashboards, alerting (~45 metrics, ~30 alerts)
- **Multi-tenancy:** Sandbox environment, rate limiting, tenant isolation, cost tracking

**Deliverable:** Production-ready for first Danish PoC customer. Security audit passed.

---

## Key Technology Decisions

| Decision Area          | Choice                                              | Rationale                                                      |
|------------------------|-----------------------------------------------------|----------------------------------------------------------------|
| Cascade Engine         | Temporal (primary) + Restate (prototype)            | Temporal proven at Twilio scale; Restate evaluated for latency |
| Carrier Connectivity   | Ergo Framework v3 actors                            | OTP supervision, gen.Stage backpressure, 21M msg/sec           |
| REST Adapters          | WASM via Wazero                                     | Sandboxed, hot-swappable, ~57ns call overhead                  |
| Protocol Adapters      | Ergo actors (gen.Server + TCP Meta Process)         | Stateful connections need actor lifecycle management            |
| Email Delivery         | KumoMTA (self-hosted)                               | 7M+/hr/node, IP warming, bounce handling built in              |
| Message Transport      | NATS JetStream                                      | Sub-ms publish, persistent streams, proven at scale            |
| Cascade State          | Temporal/Restate persistence (not NATS KV)          | Eliminates 153M-key scaling problem                            |
| DLR Correlation        | Actor-local state in DLRProcessor                   | No external store needed, lives with SMPP connection           |
| Secrets                | HashiCorp Vault                                     | Dynamic secrets, rotation, audit trail                         |
| SMPP Library           | Hardened gosmpp fork                                | Add TLS, PDU validation, fuzz testing to existing library      |
| Auth                   | API keys (Phase 0) + OAuth2/mTLS (Phase 5)         | Progressive security; API keys for MVP speed                   |
| HLR                    | Redis cache + async lookup                          | Sub-ms cache hits, background refresh                          |
| Monitoring             | OpenTelemetry + Prometheus + Grafana                | Industry standard, ~45 metrics, ~30 alerts                     |
| ML Routing             | Deferred (rule-based first)                         | Need 6+ months of delivery data before ML adds value           |
| RCS Channel            | Deferred                                            | Market adoption insufficient, revisit 2027                     |
| Multi-DC               | Deferred (single-DC with HA)                        | Premature; add when customer demand requires it                |

---

## What We're NOT Building (and Why)

| Not Building                            | Why                                                                                   |
|-----------------------------------------|---------------------------------------------------------------------------------------|
| Custom MTA                              | KumoMTA does it better (7M+/hr, Lua config, IP warming, bounce handling)              |
| WhatsApp BSP                            | WhatsApp Cloud API is sufficient; BSP requires Meta partnership and compliance burden  |
| Custom event sourcing                   | Temporal/Restate provide durable execution out of the box                              |
| Docker containers for adapters          | WASM is lighter (~57ns overhead vs container cold start), sandboxed by default         |
| eBPF for SMPP inspection                | Overkill; actor-level metrics give sufficient visibility                               |
| Edge computing for routing              | Routing decisions need central state (carrier capacity, HLR); edge adds latency       |
| CRDTs for cascade state                 | Durable execution makes eventual consistency unnecessary for cascades                  |
| Multi-DC active-active                  | Premature optimization; single-DC HA is sufficient for 22M/day                        |
| Full ML routing                         | Need 6+ months of historical delivery data; rule-based routing is correct for launch   |

---

## Honest Timeline: 2-3 Devs + Claude

### 1-Month MVP Includes

- Working REST API (`POST /v1/messages`)
- Cascade engine on Temporal (durable, crash-recoverable)
- SMPP actor mesh with Ergo (against simulator)
- 2-3 WASM adapters (SendGrid, Twilio REST, FCM)
- Smart Router (basic rule-based)
- Docker Compose for local dev
- ~60% test coverage on core paths

### Cannot Ship in 1 Month

- KumoMTA integration (8-week IP warming alone)
- Full security hardening
- Production carrier connections (need contracts)
- Kubernetes/Terraform deployment
- gosmpp security audit
- Monitoring dashboards
- Sandbox environment
- Restate prototype

### Honest Assessment

| Milestone                        | Timeline     |
|----------------------------------|--------------|
| Working demoable MVP             | 1 month      |
| Production-ready for first customer | 3-4 months |
| Carrier-direct at scale          | 6-9 months   |

### Recommended Team Split

| Dev             | Focus                                      | Skills Required                        |
|-----------------|--------------------------------------------|----------------------------------------|
| Dev 1 (Lead)    | Architecture, cascade engine, Temporal      | Go, distributed systems, Temporal      |
| Dev 2           | SMPP + Ergo actors, carrier connectivity    | Go, networking, TCP protocols          |
| Dev 3           | API + WASM adapters, DevOps, CI/CD          | Go, Docker, WASM, infrastructure       |
| Claude          | Code generation, tests, adapters, docs      | Everything, 24/7                       |

---

## Verification Plan

### Phase 1 Verification: Cascade Engine Prototypes

| Test                              | Method                                                      | Pass Criteria                      |
|-----------------------------------|-------------------------------------------------------------|------------------------------------|
| Cascade executes 3 channels       | Send message, observe workflow history                      | All 3 channels attempted in order  |
| DLR signals complete cascade      | Simulate DLR webhook, verify workflow completes             | Workflow marked delivered           |
| Timeout triggers fallback         | Delay DLR past timeout, verify next channel fires           | Fallback within 1s of timeout      |
| Crash recovery                    | Kill worker mid-cascade, restart, verify resumption         | Message delivered despite crash     |
| Latency                           | OTel traces on 1000-message batch                           | p99 < 500ms per cascade step       |
| Temporal vs Restate comparison    | Run identical scenarios, compare metrics                    | Documented trade-off analysis      |

### Phase 2 Verification: SMPP Carrier Mesh

| Test                              | Method                                                      | Pass Criteria                      |
|-----------------------------------|-------------------------------------------------------------|------------------------------------|
| Connection lifecycle              | Bind, send, unbind to SMPP simulator                        | Clean session management           |
| Supervisor auto-restart           | Kill ConnectionActor, verify restart                        | New connection within 5s           |
| Backpressure                      | Flood actor mesh, observe gen.Stage throttling              | No message loss, throughput caps   |
| DLR correlation                   | Send message, receive DLR, verify ID mapping                | Internal ID resolved from SMSC ID  |
| Window tracking                   | Send N messages, verify in-flight window respected          | Never exceeds configured window    |
| Multi-carrier routing             | Route to telia-dk vs tdc based on prefix                    | Correct carrier selected           |

### Phase 3 Verification: WASM Adapters

| Test                              | Method                                                      | Pass Criteria                      |
|-----------------------------------|-------------------------------------------------------------|------------------------------------|
| WASM adapter loads                | Compile .wasm, load in Wazero host                          | OnConfigure returns success        |
| SendHTTP works                    | Adapter calls SendHTTP, verify HTTP request made            | Correct URL, headers, body         |
| Sandboxing                        | Attempt file system / network access from WASM              | Denied by default                  |
| Hot-swap                          | Replace .wasm at runtime, verify new version active         | Zero-downtime swap                 |
| Health check                      | Call OnHealthCheck, verify adapter responds                 | Returns true when healthy          |
| Error handling                    | Simulate HTTP 500 from provider                             | Error surfaced to cascade engine   |

### End-to-End Verification

| Test                              | Method                                                      | Pass Criteria                      |
|-----------------------------------|-------------------------------------------------------------|------------------------------------|
| Full message lifecycle            | REST API -> validation -> cascade -> SMPP delivery -> DLR   | Message delivered, DLR received    |
| Multi-channel fallback            | SMS fails, email succeeds                                   | Cascade falls through correctly    |
| Tenant isolation                  | Two tenants, verify no data leakage                         | Strict separation                  |
| Load test                         | 1000 msg/sec sustained for 5 minutes                        | No errors, p99 < 2s               |
| Chaos test                        | Random pod/actor kills during load                          | All messages eventually delivered  |

---

## Expert Review Scores (Context)

The next-gen architecture addresses findings from 5 expert reviews of the original ADR:

| Review                    | Score   | Key Finding Addressed                                         |
|---------------------------|---------|---------------------------------------------------------------|
| System Architect          | 6.5/10  | State persistence (2/10) -> Temporal durable execution        |
| Security Audit            | 7.0/10  | Adapter supply chain -> WASM sandboxing replaces containers   |
| Carrier-Direct Architect  | 4/10    | Backpressure (2/10) -> Ergo gen.Stage demand-driven flow      |
| Carrier-Direct Mentor     | 4/10    | "Timeline is fiction" -> Adjusted to honest 1mo MVP / 3-4mo production |
| DevOps                    | (from reviews) | Operational complexity -> KumoMTA replaces custom MTA    |

---

## Risk Register

### Technical Risks

| Risk | Severity | Probability | Mitigation |
|------|----------|-------------|------------|
| Ergo v3 is relatively new (Feb 2026) | Medium | Low | Zero deps, MIT license, can fork. Actor patterns well-understood. |
| Temporal operational overhead | Medium | Medium | Start with PostgreSQL. Cassandra migration planned for scale. |
| Wazero WASI gaps | Low | Low | Only need basic host functions. WASI preview 1 sufficient. |
| KumoMTA young open source | Medium | Low | Apache 2 license, PowerMTA team. Haraka as fallback. |
| TinyGo WASM compilation limits | Medium | Medium | Adapters are simple HTTP transforms — within TinyGo capability. |
| `numHistoryShards` immutable in Temporal | High | Medium | Set correctly before production (2048-4096). Cannot change later. |
| gosmpp single-maintainer, no security audit | High | Medium | Fork into hermes-platform org. Commission external audit. |

### Business Risks

| Risk | Severity | Probability | Mitigation |
|------|----------|-------------|------------|
| Carrier contracts take 3-6 months | High | High | Start negotiations in parallel with development. Hire telecom consultant. |
| Danish telecom registration required | **Critical** | Certain | Legal process must start immediately — criminal offense to operate without it. |
| IP warming takes 8 weeks minimum | Medium | Certain | Physical constraint, cannot be accelerated. Start in Phase 4. |
| No team SMPP experience | High | High | SMPP simulator first. Real carriers only after security audit. |
| Analysis paralysis (mentor: 85% probability) | High | High | This plan exists to prevent it. Ship MVP in 4 weeks. |

### Expert-Recommended Alternative Path

The expert reviews suggested a more conservative ordering than this plan:

| Phase | Expert Recommendation | This Plan |
|-------|----------------------|-----------|
| **First** | Cascade Engine + aggregator APIs (SendGrid/Twilio). Get PoC customers live. | Same (Phase 0-1) |
| **Parallel** | Start carrier contract negotiations. Hire telecom consultant. | Same (business track) |
| **Second** | SMPP inbound for enterprise clients. Smart routing against aggregators. | Phase 2 (SMPP actor mesh) |
| **Third** | First direct carrier connection after contract signed. | Phase 5 (production hardening) |
| **Only if email > 2B/year** | MTA investigation | Phase 4 (KumoMTA) |
| **Never until $5M+ ARR** | WhatsApp BSP | Not in plan (using Cloud API) |

The key difference: experts recommend shipping the cascade engine with aggregator APIs as the MVP product, and treating carrier-direct as a later optimization. This plan follows that philosophy — Phases 0-1 use simulated providers, with real carrier connections deferred to post-security-audit.

---

## Sources

- [Temporal Go SDK](https://github.com/temporalio/sdk-go)
- [Restate Go SDK](https://github.com/restatedev/sdk-go)
- [Ergo Framework v3](https://github.com/ergo-services/ergo)
- [Wazero](https://wazero.io/)
- [KumoMTA](https://kumomta.com)
- [knqyf263/go-plugin](https://github.com/knqyf263/go-plugin)
- [gosmpp](https://github.com/linxGnu/gosmpp)

---

## Decision Required

This plan requires team approval before Phase 0 begins. Key decisions for discussion:

1. **Cascade engine:** Accept dual-prototype approach (Temporal + Restate)?
2. **Adapter strategy:** Confirm hybrid WASM + Ergo actors?
3. **KumoMTA vs SES:** Commit to own MTA for email, or keep SES as primary?
4. **Team allocation:** Agree on dev focus areas?
5. **Timeline:** Accept 1-month MVP scope with items explicitly deferred?
6. **Legal:** Who initiates Danish telecom registration? (Criminal offense blocker)
7. **Carrier contracts:** Who starts Telia/TDC negotiations in parallel?

---

*Generated: 2026-02-11 | Status: PENDING TEAM APPROVAL*
