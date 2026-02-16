# Technology Investigation: Next-Generation Architecture

> **Context Note (2026-02-16):** This research was conducted during the carrier-direct planning phase (v1.0). Following the strategic pivot to aggregator-first MVP, the relevant technologies are:
> - **NOW (MVP):** Temporal.io/Restate (cascade engine), WASM/Wazero (provider adapters), NATS JetStream (transport)
> - **PHASE 5+ (business decision):** Ergo Framework (SMPP actor mesh), KumoMTA (own MTA), gosmpp (SMPP library)
>
> See `docs/plans/implementation-plan.md` (v2.0) for the current architecture.

## Executive Summary

Three research agents investigated modern technologies to determine whether Project Hermes should follow traditional 2010-era telecom patterns or leverage newer technology to solve infrastructure problems fundamentally better. The investigation covered 10+ technologies across durable execution, actor models, WebAssembly, email infrastructure, edge computing, and data synchronization.

**Bottom line: Three technologies (Temporal.io, Ergo Framework, WASM/Wazero + KumoMTA) can eliminate 60-70% of the custom infrastructure code** that five expert reviews identified as the hardest unsolved problems.

## Investigation Scope

### Technologies Investigated

| Technology | Category | Verdict |
|------------|----------|---------|
| **Temporal.io** | Durable execution | **ADOPTED** — Cascade engine |
| **Restate** | Durable execution | **PROTOTYPE** — Compare with Temporal |
| **Ergo Framework v3** | Actor model (Go) | **ADOPTED** — Carrier mesh |
| **Wazero** | WASM runtime (Go) | **ADOPTED** — REST adapters |
| **KumoMTA** | Email MTA (Rust) | **DEFERRED** — Phase 5 business decision (using SES for MVP) |
| **knqyf263/go-plugin** | WASM plugin interface | **ADOPTED** — Adapter contract |
| **NATS JetStream** | Message transport | **RETAINED** — High-throughput delivery transport |
| eBPF | Kernel-level packet processing | **REJECTED** — SMPP too complex for eBPF |
| CRDTs | Conflict-free replicated data types | **REJECTED** — Cascade is sequential, not commutative |
| Edge computing | Cloudflare Workers / Deno Deploy | **REJECTED** — SMPP needs persistent TCP from fixed IPs |
| ML-based routing | Multi-armed bandit / RL | **DEFERRED** — Need 3-6 months of delivery data first |
| Proto.Actor | Actor model (Go) | **REJECTED** — No gen.Stage equivalent |
| Hollywood | Actor model (Go) | **REJECTED** — Too minimal |
| Wasmtime | WASM runtime | **REJECTED** — CGo dependency |
| Wasmer | WASM runtime | **REJECTED** — CGo dependency |
| Haraka | Email MTA (Node.js) | **REJECTED** — Lower throughput than KumoMTA |

### What Motivated the Investigation

Five expert reviews of the original ADR (Architecture Decision Record) identified critical gaps:

| Review | Score | Critical Gap |
|--------|-------|-------------|
| System Architect | 6.5/10 | State persistence scored 2/10 — pod crash loses ~130K in-flight messages |
| Security Audit | 7.0/10 | Adapter container supply chain — Ed25519 identity but not image integrity |
| Carrier-Direct Architect | 4/10 | Backpressure scored 2/10 — no mechanism to propagate SMPP throttling |
| Carrier-Direct Mentor | 4/10 | "Vision correct, timeline fiction" — SMPP/MTA complexity underestimated |
| DevOps | — | MTA operations "hell" — building own MTA from scratch is catastrophic |

The question became: **can modern technology solve these 2/10-scored problems without writing thousands of lines of custom infrastructure code?**

## Research Findings

### Finding 1: Durable Execution Eliminates Custom Event Sourcing

**The Problem:**
The cascade engine — Hermes's core IP — maintains a state machine per message. The architect review scored state persistence 2/10 because:
- Pod crash loses all in-flight state machines (~130K concurrent at peak)
- NATS KV proposed for cascade state won't scale to 153M active keys
- Custom event sourcing on NATS requires ~3,000-5,000 lines of recovery code
- "Message in quantum state" — crash after submit_sm sent but before submit_sm_resp

**The Discovery:**
Twilio runs every message through Temporal.io workflows. Netflix, Snap, and Instacart (75M/month) also use Temporal. This exact problem — orchestrating multi-step delivery with crash recovery — is a solved problem.

**What durable execution provides:**
- Crash recovery: workflow resumes exactly where it left off after worker restart
- Timeout handling: durable timers survive crashes (no custom timer management)
- Retry policies: per-activity retry with backoff (no custom retry loops)
- Event history: every state transition persisted automatically (no custom event sourcing)
- Visibility: search workflows by status, tenant, message ID (no custom query system)

**Impact:** ~3,000-5,000 lines of custom state machine, event sourcing, recovery, and timeout code eliminated. The NATS KV scaling problem disappears entirely.

See: `docs/research/temporal-deep-dive.md` for detailed comparison.

### Finding 2: Actor Supervision Trees Solve Connection Management

**The Problem:**
SMPP carrier connections are stateful, long-lived TCP connections that:
- Die unexpectedly (carrier maintenance, network issues)
- Need per-connection window tracking and enquire_link management
- Require backpressure when carrier throttles (ESME_RTHROTTLED)
- Must propagate health status to the routing layer

The architect review scored backpressure 2/10. The original design used connection pools with goroutines and mutexes — requiring ~1,500 lines of manual reconnection, health checking, and backpressure code.

**The Discovery:**
Ergo Framework v3 faithfully ports Erlang/OTP patterns to Go — not just naming conventions, but protocol-level compatibility. Key finding: gen.Stage implements demand-driven backpressure (from Elixir's GenStage), which directly solves the 2/10 backpressure score.

**What Ergo provides:**
- gen.Supervisor: Automatic restart on connection failure (One-For-One strategy)
- gen.Stage: Demand-driven flow control — consumer requests N items, producer sends N items
- gen.Server: Each SMPP connection is an isolated actor with local state (no locks)
- TCP Meta Process: Bridges blocking SMPP I/O with actor messaging
- Escalation: After N failed restarts, escalate to parent supervisor (carrier-level failure)

**Impact:** ~1,500 lines of connection pool, health check, reconnection, and backpressure code eliminated. Backpressure becomes a natural property of the actor pipeline rather than a bolt-on mechanism.

See: `docs/research/actor-model-wasm.md` for detailed framework comparison.

### Finding 3: WASM Replaces Container-Based Adapter Isolation

**The Problem:**
The original ADR specified Docker containers for each channel adapter with:
- Ed25519 keypair generation and signature verification
- Heartbeat lifecycle (30-second cycle, 3 missed = inactive)
- gRPC adapter contract
- Container registry, image signing, hot-swap via deployment

This is ~2,000 lines of registration, lifecycle, and security code. For stateless REST adapters (SendGrid, Twilio, FCM), containers are overkill.

**The Discovery:**
Wazero (pure Go WASM runtime, zero CGo) is production-ready. Arcjet runs it with p50=10ms, p99=30ms. Function call overhead is ~57ns. Deny-by-default sandboxing eliminates the need for container isolation.

**What WASM/Wazero provides:**
- Hot-swap: Compile new .wasm, swap pointer atomically, zero downtime
- Sandboxing: No network, no filesystem unless host explicitly grants access
- Per-tenant isolation: Separate WASM instance per tenant with memory limits
- Minimal overhead: ~57ns per function call vs container startup + gRPC
- Tenant custom adapters: Tenants upload .wasm files, run safely sandboxed

**Impact:** ~2,000 lines of container lifecycle, Ed25519 registration, gRPC, and health check code eliminated. Docker dependency removed for adapter management.

### Finding 4: KumoMTA Eliminates Custom MTA Development

**The Problem:**
The original ADR left MTA technology undecided. Building a custom MTA requires:
- ISP-specific throttling rules (Gmail, Microsoft, Yahoo all different)
- IP warming scheduler (8-week process, physical time constraint)
- Bounce classification engine (~120M bounces/year at 2% rate)
- DKIM signing per tenant domain
- SPF/DMARC enforcement
- ISP feedback loop processing
- Queue management per destination domain

The mentor review estimated own-MTA operations at ~$480K/year and scored it 2/10.

**The Discovery:**
KumoMTA is a Rust-based open-source MTA doing 7M+ messages/hour per node, created by the team behind PowerMTA (used by SendGrid, Mailchimp). It handles ISP throttling, IP warming, bounce classification, DKIM, and queue management via Lua scripting — not just config files, but actual programmable logic.

**What KumoMTA provides:**
- 7M+ msg/hr/node throughput
- ISP-specific throttling (Gmail, Microsoft, Yahoo presets)
- Built-in IP warming schedules
- Bounce classification engine
- DKIM signing with key rotation
- SPF/DMARC enforcement
- Prometheus metrics endpoint
- JSON structured logging and webhook delivery reports

**Impact:** Entire custom MTA codebase eliminated (~500 lines IP warming + ~1,000 lines bounce classification + queue management + DKIM + SPF). Operational complexity dramatically reduced.

## Technologies Rejected (and Why)

### eBPF for SMPP Protocol Handling
**What it is:** Kernel-level packet processing, can inspect/modify network traffic at wire speed.
**Why rejected:** SMPP protocol has complex PDU structures with variable-length fields, optional TLVs, and multi-part message concatenation. This is far too complex for eBPF programs which are limited in instruction count and cannot make complex branching decisions. eBPF is better suited for observability (counting PDUs, measuring latency) rather than protocol handling.
**Verdict:** Use for observability only, not protocol processing.

### CRDTs (Conflict-Free Replicated Data Types)
**What it is:** Data structures that can be replicated across nodes and merged without coordination.
**Why rejected:** The cascade engine is inherently sequential — Email THEN SMS THEN Push. CRDTs solve the problem of concurrent, commutative updates (e.g., counters, sets). Cascade state transitions are ordered and non-commutative (you can't deliver SMS before checking if Email succeeded). Wrong abstraction entirely.
**Verdict:** No fit for cascade. Potentially useful for distributed counters (rate limiting) in future.

### Edge Computing (Cloudflare Workers / Deno Deploy)
**What it is:** Run code at CDN edge locations, close to users.
**Why rejected:** SMPP connections require persistent TCP from fixed, carrier-whitelisted IP addresses. Edge computing runs at dynamic IPs across hundreds of PoPs. Carrier contracts specify exact source IPs. MTA reputation is tied to specific IP addresses. Edge is incompatible with the fundamental requirements of carrier-direct infrastructure.
**Verdict:** Potentially useful for webhook ingestion (DLR callbacks from providers) but not for core delivery.

### ML-Based Routing (Multi-Armed Bandit)
**What it is:** Use machine learning to optimize routing decisions in real-time.
**Why rejected for now:** ML routing requires training data — delivery rates, latency, cost per route per carrier per destination. We have zero data until we start delivering messages. Starting with ML would be premature optimization.
**Plan:** Start with rule-based routing (HLR lookup → least-cost → quality score). After 3-6 months of delivery data, add multi-armed bandit for route optimization. The Smart Router interface is designed to swap the algorithm without changing the integration.

### Proto.Actor for Go
**What it is:** Actor framework with Protobuf-native messaging, 5.4K stars, Apache 2.0.
**Benchmarks:** 70M msg/sec local, 5.4M remote (ping-pong). Virtual actors (Orleans-style grains). Cross-language: Go, C#, Java/Kotlin via gRPC.
**Why rejected:** No gen.Stage equivalent for demand-driven backpressure (the #1 requirement). Heavy dependency tree (gRPC, Consul, Protobuf). Users report losing precise control as projects grow complex. Supervision is basic compared to Ergo's full OTP tree model.

### Hollywood Actor Framework
**What it is:** Lightweight actor framework for Go, 2.2K stars, MIT.
**Benchmarks:** 10M messages in under 1 second. dRPC transport (faster than gRPC). WASM compilation support (unique). Active Discord community (2K+ members).
**Why rejected:** No supervision trees (basic restart only). No gen.Stage, no Saga, no TCP Meta Process. Benchmark shows 2GB vs 100MB memory compared to raw channels for simple cases. Too minimal for SMPP connection management.

### GoAkt Actor Framework
**What it is:** Virtual actor framework for Go, 320 stars, v3.13.0, MIT.
**Features:** True virtual actor (Grain) support, multiple clustering backends (Consul, etcd, K8s, NATS), cluster singletons, OpenTelemetry observability.
**Why rejected:** Smaller community and less proven than Ergo. No gen.Stage equivalent. Virtual actor model is a better fit for distributed grain-based systems than for persistent TCP connection management.

### Wasmtime / Wasmer WASM Runtimes
**What they are:** Alternative WASM runtimes (more mature, wider language support).
**Wasmtime:** Very mature (Bytecode Alliance), used by Fastly, Redpanda, Envoy. Generally faster for CPU-bound work due to Cranelift JIT. But requires CGo.
**Wasmer:** Multiple compiler backends (Singlepass, Cranelift). Reduced maintenance activity, less active Go ecosystem support. Also requires CGo.
**Why both rejected:** CGo dependency breaks Go's simple build/cross-compilation story. Wazero's pure-Go implementation with zero dependencies is a significant advantage for maintainability and CI/CD.

### io_uring (Gain Framework)
**What it is:** Linux kernel async I/O interface, ~30% faster than gnet for networking.
**Assessment:** Useful optimization for SMPP connection handling at scale. Not critical for MVP.
**Verdict:** P2 optimization. Evaluate after SMPP actor mesh is working.

## Summary: What Changes vs Original ADR

| ADR Component | Original Design | Next-Gen Design | Lines Saved |
|---------------|----------------|-----------------|-------------|
| Cascade Engine | Custom state machine + NATS KV | Temporal/Restate workflow | ~3,000-5,000 |
| State Persistence | NATS KV (won't scale) | Temporal/Restate built-in | Problem gone |
| Crash Recovery | Custom event sourcing | Durable execution guarantee | ~1,000 |
| Channel Adapters | Docker + gRPC + Ed25519 | WASM plugins (Wazero) | ~2,000 |
| Connection Management | Pool manager + health checks | Ergo supervision tree | ~1,500 |
| Backpressure | Custom 5-layer chain | gen.Stage + Temporal rate limiting | 3/5 layers natural |
| DLR Correlation | Custom store | Actor-local in DLRProcessor | Simpler |
| Email MTA | Build from scratch (Haraka/custom) | KumoMTA (Lua-configured) | Entire codebase |
| IP Warming | Custom scheduler | KumoMTA built-in | ~500 |
| Bounce Classification | Custom engine | KumoMTA built-in | ~1,000 |

**Total estimated reduction: 60-70% less custom infrastructure code.**

## What Stays the Same from Original ADR

Not everything changes. These ADR decisions remain solid:
- **Go language** — correct for all the same reasons
- **Hexagonal architecture** — port interfaces still apply (implementations change)
- **Chi router** — lightweight, idiomatic
- **NATS JetStream** — retained for high-throughput delivery transport (not cascade state)
- **PostgreSQL** — retained for tenant data (also used by Temporal)
- **Redis** — retained for DLR correlation, rate limiting, idempotency
- **9-layer validation pipeline** — unchanged, solid design
- **Multi-tenant tiered isolation** — unchanged
- **OpenTelemetry + Prometheus + Grafana** — unchanged
- **WorkOS SSO** — unchanged

## Expert Consensus: What Drove the Investigation

The next-gen investigation was triggered after 10 expert reviews (5 original + 5 carrier-direct) produced declining scores — carrier-direct additions made the architecture *harder*, not easier:

| Expert | Original ADR | Carrier-Direct | Delta |
|--------|-------------|----------------|-------|
| Mentor | 4/10 | 5/10 | +1 (vision better, execution same) |
| Security | 7/10 | 3.5/10 | **-3.5** (new attack surfaces unmitigated) |
| UX/DX | 3/10 | 4/10 | +1 (API improving, ops UX terrible) |
| Architect | 6.5/10 | 4/10 | **-2.5** (hard problems now harder) |
| DevOps | 1.5/10 | 0.5/10 | **-1** (zero implementation, more to build) |

### 5 Things Every Expert Agreed On

1. **The Cascade Engine is the real product.** Ship it first with aggregator APIs. Carrier-direct is a margin optimization, not the MVP.
2. **The timeline is fiction.** 30 weeks for what took competitors 3-5 years. Carrier contracts alone take 3-6 months.
3. **Backpressure is the #1 unsolved technical problem.** No chain exists from carrier SMPP throttle through to API Gateway.
4. **State persistence is the #2 unsolved problem.** Pod crash after submit_sm = message in quantum state.
5. **The cost model is misleading.** Email MTA saves ~$120K/year (not $550K) after ops costs. SMS direct margins are the real money ($20-40M/year at scale).

### Non-Negotiables Before Production

| Blocker | Type | Timeline |
|---------|------|----------|
| Danish telecom registration | **Legal** (criminal offense without it) | Immediate |
| TLS on all SMPP connections | **Engineering** (GDPR violation) | 2-3 weeks |
| Secrets management (Vault/HSM) | **Engineering** | 1-2 weeks |
| Event-sourced cascade state | **Engineering** | 2 weeks |
| DLR correlation store (Redis) | **Engineering** | 1 week |
| gosmpp security audit | **External** | 2-3 weeks |
| HLR GDPR Data Processing Agreement | **Legal** | 1 week |

### How the Three Breakthroughs Address Expert Findings

| Expert Finding (2/10) | Next-Gen Solution | New Score (estimated) |
|----------------------|-------------------|----------------------|
| State persistence (Architect) | Temporal durable execution | 8/10 — solved by design |
| Backpressure (Architect) | Ergo gen.Stage demand-driven flow | 7/10 — natural property |
| Crash recovery (Architect) | Temporal automatic workflow replay | 8/10 — solved by design |
| Adapter supply chain (Security) | WASM deny-by-default sandboxing | 7/10 — better than containers |
| MTA operations (DevOps/Mentor) | KumoMTA replaces custom MTA | 7/10 — proven Rust binary |
| Connection management (Architect) | Ergo supervisor auto-restart | 7/10 — OTP pattern |

## User Decisions Made During Planning

These decisions were made interactively during the planning session:

1. **Philosophy: Maximum Innovation** — "Use the newest, most modern approaches everywhere. Accept some risk for potentially massive simplification. We're building the NEXT generation, not copying the current one."
2. **Innovation Focus: Carrier Connectivity** — "Reinvent how we talk to carriers. Modern protocol handling, intelligent connection management, self-healing routing. Make SMPP feel like a REST API internally."
3. **Cascade Engine: Prototype Both** — "Build the same cascade flow on both Temporal and Restate, compare latency/ops/DX. Takes extra time but de-risks the choice. The hexagonal architecture means we can swap later."
4. **Adapter Strategy: Hybrid** — "WASM for stateless REST adapters (SendGrid, Twilio API, FCM). Ergo actors for stateful protocol adapters (SMPP, SMTP). Best of both worlds."

The assistant characterized this as: "Maximum Innovation + Carrier Connectivity focus. That's the boldest combination. You're not building 'Twilio but cheaper' — you're building what Twilio would build if they started today with zero legacy."

## Industry Context

Research into the competitive landscape provided important context for architectural decisions:

### CPaaS Market
- Market size: $19.87B (2025) → projected $80.40B (2030) at 30.4% CAGR
- Infobip: 1.2 billion transactions/day, event-driven architecture on Confluent/Kafka
- Sinch: 600+ direct operator connections, 44 redundant POPs, 99.999% uptime
- Telnyx: Built own private global IP network rather than aggregating — most architecturally distinctive

### SMPP Protocol Status (2025-2026)
- SMPP 3.4 (1999) remains dominant. SMPP 5.0 exists but only 8% adoption.
- Carriers offering REST APIs alongside SMPP: Telia (Nordic) offers both REST and SMPP
- GSMA Open Gateway: covers 79% of global mobile market (73 operator groups, 285 networks) with standardized carrier REST APIs — potential future channel

### RCS Inflection Point
- Apple adopted RCS in iOS 18 (September 2025) — game-changer for rich messaging
- Not included in MVP scope but Smart Router interface designed to accommodate future RCS channel
- Deferred to Phase 2+ after carrier contracts established

### SMPP Scale Math (Hermes)
- 2B texts/year = 5.5M texts/day = ~64 texts/sec average, ~300/sec peak
- SMPP bottleneck: typically 100-500 TPS per bind
- Need multiple binds per carrier: e.g., Telia DK: 10 binds x 100 TPS = 1,000 TPS; TDC: 5 binds x 200 TPS = 1,000 TPS

## Technology Priority Matrix

| Technology | Verdict | Priority | Impact |
|---|---|---|---|
| Temporal.io for cascade | Game-changer | P0 | Eliminates ~5,000 lines of state management |
| Restate (evaluate) | Game-changer | P0 | Lower latency alternative to Temporal |
| Ergo Framework for SMPP | Game-changer | P0 | Eliminates ~1,500 lines, solves backpressure |
| KumoMTA for email | Game-changer | P0 | Eliminates entire MTA codebase |
| Wazero/WASM for adapters | Strong improvement | P1 | Eliminates ~2,000 lines, better security |
| RCS channel | Game-changer for reach | P1 | After carrier contracts |
| io_uring (Gain) for SMPP | Optimization (~30%) | P2 | After actor mesh is working |
| ML routing | Incremental improvement | P2 | After 3-6 months of delivery data |
| CRDTs | Niche utility | P3 | Useful for distributed counters only |
| Edge computing | Wrong tool | Skip | Incompatible with carrier-direct |
| eBPF for protocol handling | Impractical | Skip | SMPP too complex for eBPF programs |

## Related Documents

| Document | Contents |
|----------|----------|
| `docs/research/temporal-deep-dive.md` | Temporal.io scale proof, Go SDK, Restate comparison, hybrid architecture |
| `docs/research/actor-model-wasm.md` | Ergo Framework analysis, WASM runtime comparison, KumoMTA evaluation |
| `docs/plans/implementation-plan.md` | Full architecture, phases, timeline, team allocation |
| `docs/adr/ADR-001-hermes-platform-architecture.md` | Original architecture (domain types, port interfaces, validation pipeline) |

## Sources

- [Temporal Go SDK](https://github.com/temporalio/sdk-go) — Cascade engine option 1
- [Restate Go SDK](https://github.com/restatedev/sdk-go) — Cascade engine option 2
- [Ergo Framework v3](https://github.com/ergo-services/ergo) — Actor model for carrier mesh
- [Wazero](https://wazero.io/) — WASM runtime for Go adapters
- [KumoMTA](https://kumomta.com) — Rust MTA, 7M+ msg/hr/node
- [knqyf263/go-plugin](https://github.com/knqyf263/go-plugin) — WASM plugin interface via protobuf
- [Arcjet WASM Production](https://blog.arcjet.com/lessons-from-running-webassembly-in-production-with-go-wazero/)
- [Temporal at Twilio, Netflix, Snap](https://medium.com/@milinangalia/the-rise-of-temporal)
- [PlanetScale Temporal Sharding](https://planetscale.com/blog/temporal-workflows-at-scale-sharding-in-production)
- [Restate Benchmarks](https://www.restate.dev/blog/building-a-modern-durable-execution-engine-from-first-principles)
- [gosmpp](https://github.com/linxGnu/gosmpp) — SMPP 3.4 library (to be forked)

---

*Generated: 2026-02-11 | For: Team Architecture Review*
