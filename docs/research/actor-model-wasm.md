# Actor Model Frameworks & WASM Runtime Comparison

## Executive Summary

Two technology categories were evaluated to solve Hermes's hardest infrastructure problems: (1) Actor frameworks for SMPP carrier connection management (replacing custom connection pools, scored 2/10 on backpressure by architect review), and (2) WASM runtimes for hot-loadable REST channel adapters (replacing Docker containers with Ed25519 signatures and heartbeats).

**Key findings:**
- Ergo Framework v3 provides genuine Erlang/OTP patterns in Go with 21M msg/sec throughput
- Wazero provides production-proven WASM execution with ~57ns function call overhead
- KumoMTA eliminates the need to build a custom MTA entirely
- The hybrid adapter strategy (WASM for REST, actors for protocols) leverages each technology's strengths

---

## Part 1: Actor Model Frameworks for Go

### Why Actors for Carrier Connectivity

The SMPP carrier connection management problem has these characteristics:
- **Stateful**: Each SMPP connection has bind state, window state, enquire_link timers
- **Long-lived**: TCP connections persist for hours/days
- **Failure-prone**: Carriers drop connections, throttle, go into maintenance
- **Concurrent**: Multiple connections per carrier, multiple carriers
- **Backpressure**: When SMPP window fills, must stop accepting messages

Traditional approach (connection pool + goroutines + mutexes) was scored 2/10 on backpressure by the architect review. The Erlang/OTP supervision tree pattern solves this naturally.

### Frameworks Evaluated

#### 1. Ergo Framework v3 (SELECTED)

**[github.com/ergo-services/ergo](https://github.com/ergo-services/ergo)**

- Version: v3.2.0 (Feb 2026) — complete rewrite of network stack in v3.x
- Stars: 4.4K | License: MIT | Dependencies: Zero | Contributors: 14
- Maintainer: Active, regular releases
- Benchmarks: **21M msg/sec locally** (64 cores), **5M msg/sec over network**, **2.9M msg/sec to 1M subscribers across 10 nodes** (distributed pub/sub)
- Erlang interop: implements actual Erlang DIST protocol and ETF data format — an Ergo node can join an Erlang cluster

**Core primitives:**

| Primitive | OTP Equivalent | Hermes Use |
|-----------|---------------|------------|
| gen.Server | GenServer | ConnectionActor (SMPP session state) |
| gen.Supervisor | Supervisor | CarrierSupervisor, SMPPSupervisor |
| gen.Stage | GenStage (Elixir) | BackpressureManager (demand-driven flow) |
| gen.Saga | - (extended) | Cascade lifecycle (distributed transactions) |
| gen.Web | - | Admin/monitoring endpoints |
| gen.Pool | - | Worker pool with round-robin distribution |
| gen.Raft | - | Consensus (leader election if needed) |
| gen.Application | Application | Application lifecycle (Permanent/Temporary/Transient) |
| Meta Process (TCP) | - | Bridge blocking SMPP I/O with actor messages |

**Supervision strategies:**
- **OneForOne (OFO)**: Restart only the failed child → for individual SMPP connections
- **AllForOne (AFO)**: Restart all children when one fails → for tightly coupled groups
- **RestForOne (RFO)**: Restart failed child and all started after it → for dependency chains
- **SimpleOneForOne (SOFO)**: Dynamic child pool → for on-demand connection scaling

**gen.Stage for backpressure:**
gen.Stage implements demand-driven data flow (from Elixir's GenStage/Flow):
- Producer → ProducerConsumer → Consumer pipeline
- Consumer requests N items → Producer sends exactly N items
- When SMPP window fills, Consumer stops requesting → backpressure propagates naturally
- No polling, no busy-waiting, no manual queue management

**gen.Stage backpressure in SMPP context:**
Each carrier connection actor is a **consumer** (finite SMPP sending window). The cascade engine actors are **producers** (generate messages). gen.Stage automatically manages demand: when a carrier's window is full, the consumer stops requesting messages from upstream. Backpressure propagates through the actor hierarchy naturally — no explicit flow control code needed. This directly addresses the 2/10 backpressure score.

**Priority mailboxes:**
Ergo provides 4 priority queues per actor: urgent, system, main, log. For SMPP:
- **Urgent**: enquire_link responses (must respond within carrier's timeout window)
- **System**: bind/unbind operations (connection lifecycle)
- **Main**: submit_sm messages (normal message delivery)
- **Log**: telemetry/metrics (best-effort)

**Meta Processes for TCP:**
SMPP uses blocking TCP I/O. Ergo's Meta Process API bridges this with **two goroutines** — one runs blocking I/O (the SMPP connection), the other handles actor messages:
- TCP socket wrapped as actor-addressable process
- Incoming bytes → actor message
- Outgoing actor message → TCP bytes
- Socket errors → actor error handling (supervisor restart)
- Supported meta types: TCP, UDP, Port, Web, WebSocket, SSE
- Note: Erlang doesn't need this because all I/O is non-blocking in BEAM. Meta processes are Ergo's novel adaptation of OTP to Go's blocking I/O reality.

**Carrier Mesh supervision tree:**

```
SMPPSupervisor (gen.Supervisor, One-For-One)
├── CarrierSupervisor("telia-dk", gen.Supervisor, One-For-One)
│   ├── ConnectionActor (gen.Server + TCP Meta Process)
│   │   └── State: gosmpp session, TLS wrapper, window tracker, enquire_link timer
│   ├── ConnectionActor (gen.Server + TCP Meta Process)
│   │   └── State: gosmpp session, TLS wrapper, window tracker, enquire_link timer
│   └── DLRProcessor (gen.Server)
│       └── State: carrier_msg_id → internal_id map (backed by Redis, 72h TTL)
├── CarrierSupervisor("tdc", gen.Supervisor, One-For-One)
│   └── ConnectionActor ...
├── CarrierSupervisor("three-dk", gen.Supervisor, One-For-One)
│   └── ConnectionActor ...
├── HealthAggregator (gen.Server)
│   └── Collects health from all CarrierSupervisors, publishes to Smart Router
└── BackpressureManager (gen.Stage)
    └── Demand-driven flow: NATS consumer → BackpressureManager → ConnectionActors
```

**Why Ergo beats connection pools:**

| Aspect | Connection Pool | Ergo Actors |
|--------|----------------|-------------|
| Connection failure | Manual reconnect logic, error handling | Supervisor auto-restarts child actor |
| Backpressure | Manual queue + polling + mutex | gen.Stage demand-driven (zero polling) |
| State isolation | Shared mutable state + locks | Actor-local state (no locks, no mutex) |
| Health monitoring | Separate health check goroutines | Mailbox depth = natural load signal |
| Carrier down | Manual escalation logic | Supervisor escalation after N restarts |
| Observability | Custom metrics per connection | Actor mailbox depth, message rates built-in |
| Graceful shutdown | Manual drain logic | Supervisor coordinates child shutdown |

**Genuine OTP fidelity (not just naming conventions):**
1. Protocol-level compatibility with Erlang DIST protocol and ETF data format
2. All four supervision strategies with all three restart types (Permanent, Transient, Temporary) and restart intensity tracking
3. gen.Stage implements the actual demand-driven backpressure protocol from Elixir's GenStage
4. gen.Saga goes beyond OTP — Erlang doesn't have a built-in saga behavior
5. Meta processes are a genuinely novel adaptation of OTP to Go's blocking I/O reality

**What you get (~80% of BEAM benefit in Go):**
- Full supervision tree semantics (identical to OTP)
- All gen.* behaviors ported faithfully
- Network transparency (same code for local and remote actors)
- Higher raw throughput than Erlang (Ergo claims 5x faster network messaging)
- Go's superior tooling, type safety, and deployment model

**What you lose (~20%):**
- No hot code loading (WASM plugins partially address this for business logic)
- No per-process GC (Go has single global GC — occasional pause spikes)
- No preemptive scheduling (Go's goroutine scheduler is cooperative, though with async preemption since Go 1.14)
- No soft real-time guarantees (Go's GC pauses may occasionally violate tight latency SLAs)
- Less mature process introspection tools compared to Erlang's Observer/etop

#### 2. Proto.Actor (EVALUATED, NOT SELECTED)

**[github.com/asynkron/protoactor-go](https://github.com/asynkron/protoactor-go)**

- Stars: 5.4K | License: Apache 2.0 | Status: Still in beta
- Benchmarks: **70M msg/sec local**, 5.4M remote (ping-pong)
- Virtual Actors (Orleans-style grains) support
- Cross-language: Go, C#, Java/Kotlin via gRPC
- Built-in DataDog and Grafana dashboards, Consul-based clustering
- Heavy dependency tree (gRPC, Consul, Protobuf) — upgrades are painful
- Users report losing precise control as projects grow complex
- **Rejected**: No gen.Stage for backpressure, heavy dependencies, basic supervision compared to Ergo's full OTP model

#### 3. Hollywood (EVALUATED, NOT SELECTED)

**[github.com/anthdm/hollywood](https://github.com/anthdm/hollywood)**

- Stars: 2.2K | License: MIT
- Benchmarks: **10M messages in under 1 second**
- dRPC transport (faster than gRPC)
- WASM compilation support (unique among these frameworks)
- Active Discord community (2K+ members)
- Production use: Sensora IoT, Market Monkey Terminal
- Benchmark caveat: 2GB vs 100MB memory compared to raw channels for simple cases
- **Rejected**: No supervision trees (basic restart only), no gen.Stage, no Saga, no TCP Meta Process. Too minimal for SMPP connection management.

#### 4. GoAkt (EVALUATED, NOT SELECTED)

**[github.com/tochemey/goakt](https://github.com/tochemey/goakt)**

- Stars: 320 | License: MIT | Version: v3.13.0 (Jan 2026)
- True virtual actor (Grain) support with guardian-based lifecycle
- Multiple clustering backends: Consul, etcd, Kubernetes, NATS, mDNS, static
- Cluster singletons, passivation (idle actor reclamation), priority mailboxes
- OpenTelemetry observability built-in
- Production users: Baki Money, Event Processor
- **Rejected**: Smaller community, less proven than Ergo. Virtual actor model better suited to distributed grain systems than persistent TCP connection management.

#### 5. Raw Goroutines + Channels (BASELINE)

- No framework overhead, full control
- **Rejected**: Would require writing ~1,500 lines of supervision, backpressure, and health monitoring code that Ergo provides out of the box

### Framework Comparison Matrix

| Feature | Ergo v3 | Proto.Actor | Hollywood | GoAkt | Raw Go |
|---------|---------|-------------|-----------|-------|--------|
| Local msg/sec | 21M | 70M | 10M | N/A | Fastest |
| Network msg/sec | 5M | 5.4M | N/A | N/A | N/A |
| Supervision trees | Full OTP (4 strategies) | Basic | None | Guardian | Manual |
| Backpressure | gen.Stage (demand-driven) | Manual | Manual | Manual | Manual |
| TCP bridge | Meta Process | Manual | Manual | Manual | Manual |
| Distributed transactions | gen.Saga | Manual | Manual | Manual | Manual |
| Dependencies | Zero | Heavy (gRPC, Consul) | Light | Medium | Zero |
| Erlang interop | Yes (DIST protocol) | No | No | No | No |
| Maturity | Stable v3 | Beta | Stable | Stable v3 | N/A |
| License | MIT | Apache 2.0 | MIT | MIT | N/A |

### Actor Framework Decision

**Ergo Framework v3** selected because:
1. Genuine OTP patterns (not just naming conventions — protocol-level Erlang compatibility)
2. gen.Stage solves the backpressure problem directly (scored 2/10 in architect review)
3. gen.Supervisor solves the connection restart problem directly
4. TCP Meta Process bridges blocking SMPP I/O naturally
5. Zero dependencies (no transitive dependency risk)
6. 21M msg/sec throughput (orders of magnitude above our needs)
7. MIT license, actively maintained

---

## Part 2: WASM Runtimes for Adapter Plugins

### Why WASM for REST Adapters

The original ADR specified Docker containers with Ed25519 signatures, gRPC, and heartbeats for channel adapters. This is ~2,000 lines of registration, health checking, and lifecycle management code. For stateless REST API adapters (SendGrid, Twilio REST, FCM, WhatsApp Cloud API), WASM provides:

- **Hot-swap**: Compile new .wasm, swap pointer, zero downtime (no container restart)
- **Sandboxing**: Deny-by-default (no network, no filesystem unless host grants)
- **Performance**: ~57ns function call overhead (vs container startup + gRPC overhead)
- **Tenant safety**: Per-tenant WASM instances with memory limits
- **No Docker dependency**: Adapters run in-process

### Runtimes Evaluated

#### 1. Wazero (SELECTED)

**[wazero.io](https://wazero.io/) / [github.com/tetratelabs/wazero](https://github.com/tetratelabs/wazero)**

- Zero CGo dependencies (pure Go)
- WASI preview 1 support
- Deny-by-default sandboxing
- Stars: ~5K | License: Apache 2.0

**Production evidence:**
- **Arcjet**: p50=10ms, p99=30ms for security rule evaluation in production
- Lessons from Arcjet's blog: pre-compile WASM modules, reuse compiled modules across requests, set memory limits per instance

**Performance benchmarks:**
| Metric | Value |
|--------|-------|
| Function call overhead | ~57 nanoseconds |
| Module compilation | ~10ms (one-time, cache the compiled module) |
| Instance creation | ~1ms |
| p50 execution (Arcjet) | 10ms |
| p99 execution (Arcjet) | 30ms |
| Memory per instance | Configurable (recommend 16MB per adapter) |

**Security model:**
- No network access by default → host must provide SendHTTP function
- No filesystem access by default → host provides StoreGet/StoreSet
- No environment variables → host provides GetConfig
- Memory limits per instance → tenant can't exhaust host memory
- CPU limits via fuel metering → infinite loop protection

**WASM vs Docker Containers comparison:**

| Dimension | WASM (Wazero) | Docker Containers |
|---|---|---|
| Cold start | 1-10ms | 1-5 seconds |
| Memory baseline | 1-10 MB per instance | 50-200 MB per instance |
| Image size | 0.5-10 MB (.wasm) | 50-500 MB |
| Instances per host | 15-20x more | Limited by memory |
| Security isolation | Memory sandboxing, no default syscalls | Full OS-level isolation |
| Network access | Host-mediated only (capability-based) | Full network stack |
| Hot-swap speed | Microseconds to milliseconds | Seconds to tens of seconds |
| Tenant custom adapters | Upload .wasm, run sandboxed | Upload container, heavier trust model |

**Production architecture patterns (from Envoy, Redpanda, Fastly):**
- **Envoy Proxy**: Proxy-Wasm spec defines two-directional ABI. Host exports functions to plugin, plugin exports callbacks to host. Each worker thread gets its own WASM VM replica.
- **Redpanda**: Shard-per-core model, each CPU core runs its own WASM VM instance. Memory reserved per-function with configurable limits.
- **Fastly Compute**: Instance instantiation in microseconds, dynamic component loading from registry.

**SMPP in WASM — NOT possible:**
WASI wasip1 does not support opening sockets. SMPP requires long-lived TCP connections with bidirectional communication. Correct architecture: SMPP connection management stays in Go host (Ergo actors); WASM plugins handle message transformation, routing logic, and channel-specific business logic only.

#### 2. Wasmtime (EVALUATED, NOT SELECTED)

**[github.com/bytecodealliance/wasmtime-go](https://github.com/bytecodealliance/wasmtime-go)**

- Very mature (Bytecode Alliance), used by Fastly, Redpanda, Envoy
- CGo wrapper around Rust-based runtime
- Generally faster for CPU-bound work due to Cranelift JIT
- Better WASI preview 2 support
- **Rejected**: CGo dependency significantly complicates cross-compilation and build toolchain

#### 3. Wasmer (EVALUATED, NOT SELECTED)

**[github.com/wasmerio/wasmer-go](https://github.com/wasmerio/wasmer-go)**

- Multiple compiler backends (Singlepass, Cranelift)
- CGo wrapper around C API
- Reduced maintenance activity, less active Go ecosystem support
- **Rejected**: Same CGo issue as Wasmtime, smaller Go ecosystem, less community activity

### WASM Adapter Architecture

**Host interface (Go side):**

```go
type AdapterHost interface {
    // Network: host-mediated HTTP calls (adapter has no direct network)
    SendHTTP(url string, headers map[string]string, body []byte) ([]byte, error)

    // Observability
    Log(level string, msg string)
    EmitMetric(name string, value float64)

    // Configuration
    GetConfig(key string) string

    // State (for adapters that need session tokens, etc.)
    StoreGet(key string) []byte
    StoreSet(key string, value []byte)
}
```

**Plugin interface (WASM side):**

```go
type AdapterPlugin interface {
    // Transform unified message to provider format, make API call via host
    OnMessage(msg []byte) ([]byte, error)

    // Parse provider-specific DLR into unified format
    OnDeliveryReport(report []byte) ([]byte, error)

    // Initialize with provider credentials and settings
    OnConfigure(config []byte) error

    // Liveness check
    OnHealthCheck() bool
}
```

**Protocol Buffers interface via go-plugin:**

[knqyf263/go-plugin](https://github.com/knqyf263/go-plugin) provides protobuf-based interface between host and WASM guest. Define the adapter contract in .proto, generate Go host + WASM guest stubs.

**Hot-swap lifecycle:**
1. Compile adapter: `tinygo build -o sendgrid.wasm -target wasi ./adapters/sendgrid/`
2. Load compiled module (one-time ~10ms)
3. Create instance per request or per-tenant (~1ms)
4. Hot-swap: compile new .wasm → atomically swap module pointer → new requests use new module
5. Old instances drain naturally (no in-flight request interruption)

**Adapters suitable for WASM:**
| Adapter | Protocol | Why WASM |
|---------|----------|----------|
| SendGrid | REST | Stateless HTTP transform |
| Twilio SMS | REST | Stateless HTTP transform |
| WhatsApp Cloud API | REST | Stateless HTTP transform |
| FCM (Firebase) | REST | Stateless HTTP transform |
| Messente | REST | Stateless HTTP transform |
| Infobip | REST | Stateless HTTP transform |

**Adapters NOT suitable for WASM (use Ergo actors instead):**
| Adapter | Protocol | Why Actors |
|---------|----------|-----------|
| SMPP carriers | SMPP 3.4 (TCP) | Long-lived TCP, stateful sessions, window management |
| KumoMTA | SMTP injection | Stateful MTA management, IP pool coordination |
| Future: RCS | Jibe RBM API | May need WebSocket for real-time |

### WASM Runtime Decision

**Wazero** selected because:
1. Zero CGo dependencies (pure Go, simple cross-compilation)
2. Production-proven at Arcjet with real latency numbers
3. Deny-by-default sandboxing (no need for container isolation)
4. ~57ns function call overhead (negligible)
5. Apache 2.0 license

---

## Part 3: KumoMTA for Email Delivery

### Why Not Build Our Own MTA

The original ADR left MTA technology undecided (Haraka vs custom Go). The carrier-direct mentor review scored MTA operations 2/10 and noted the "real cost" of own MTAs is ~$480K/year when including IP warming, deliverability engineering, and incident response.

**KumoMTA changes the equation** by providing a production-grade MTA that handles the hardest parts:

### KumoMTA Capabilities

**[kumomta.com](https://kumomta.com/) / [github.com/KumoCorp/kumomta](https://github.com/KumoCorp/kumomta)**

- Language: Rust | License: Apache 2.0
- Performance: 7M+ messages/hour per node
- Configuration: Lua scripting (not config files — actual programmable logic)
- Maintainer: Previously maintained Momentum/PowerMTA (industry standard commercial MTAs)

| Feature | Detail |
|---------|--------|
| ISP throttling | Per-ISP, per-IP rate limiting (Gmail, Microsoft, Yahoo presets) |
| IP warming | Built-in warming schedules with automatic volume ramp |
| Bounce handling | Classification engine (hard/soft/block/transient) |
| DKIM signing | Per-domain DKIM with key rotation |
| SPF/DMARC | Enforcement and reporting |
| Feedback loops | ISP FBL processing (Gmail Postmaster, Microsoft SNDS, Yahoo) |
| Queue management | Per-destination queues with smart retry |
| TLS | STARTTLS, DANE, MTA-STS support |
| Logging | JSON structured logs, webhook delivery reports |
| Metrics | Prometheus endpoint built-in |

### KumoMTA vs Alternatives

| MTA | Language | Performance | Config | License | Notes |
|-----|----------|-------------|--------|---------|-------|
| **KumoMTA** | Rust | 7M+/hr/node | Lua | Apache 2 | By PowerMTA creators |
| Haraka | Node.js | ~500K/hr | JS plugins | MIT | Simpler, lower throughput |
| Postfix | C | ~1M/hr | Config files | IBM PL | Industry standard but rigid |
| Custom Go | Go | Variable | Go code | N/A | Months of development |

**KumoMTA selected** because: highest throughput, Lua scripting for ISP-specific rules, built-in IP warming (8-week process), built-in bounce classification, created by the team behind PowerMTA (used by SendGrid, Mailchimp, etc.).

### Integration with Hermes

```
Cascade Engine → Smart Router → MTASupervisor (Ergo) → KumoMTA (SMTP injection)
                                                              ↓
                                                        ISP delivery
                                                              ↓
                                                   Bounce/DLR webhook
                                                              ↓
                                              BounceProcessor (Ergo actor)
                                                              ↓
                                              Signal back to Temporal workflow
```

**Ergo MTASupervisor actors:**
- IPPoolManager: warming schedules, IP rotation, reputation tracking per IP
- BounceProcessor: KumoMTA webhooks → cascade signals (hard bounce = stop, soft bounce = retry)
- HealthMonitor: blacklist checking, ISP feedback loop aggregation

---

## Combined Architecture: How It Fits Together

```
                    ┌──────────────────┐
                    │   Cascade Engine  │  (Temporal/Restate)
                    │   (durable exec)  │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │   Smart Router    │  (embedded library)
                    │  (HLR, LCR, QoS) │
                    └───┬────┬────┬────┘
                        │    │    │
           ┌────────────┘    │    └────────────┐
           ▼                 ▼                  ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │ Carrier Mesh │  │ MTA Cluster  │  │ WASM Adapters│
    │ (Ergo actors)│  │  (KumoMTA)   │  │  (Wazero)    │
    │              │  │              │  │              │
    │ SMPP carriers│  │ Email ISPs   │  │ REST APIs    │
    │ (TCP, stateful) │ (SMTP, stateful) │ (HTTP, stateless)│
    └──────────────┘  └──────────────┘  └──────────────┘
```

Each technology handles what it's best at:
- **Ergo actors**: Stateful, long-lived connections (SMPP, MTA management)
- **WASM/Wazero**: Stateless, hot-swappable transforms (REST API adapters)
- **KumoMTA**: Email delivery (ISP throttling, warming, bounces, DKIM)

---

## Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Ergo v3 is relatively new | Medium | Zero dependencies, MIT license, can fork if abandoned. Actor patterns are well-understood. |
| Wazero WASI support gaps | Low | We only need basic host functions. WASI preview 1 is sufficient. |
| KumoMTA is young open source | Medium | Apache 2 license. Created by PowerMTA team (decades of MTA experience). Can fall back to Haraka. |
| TinyGo WASM compilation limits | Medium | TinyGo doesn't support full Go stdlib. Adapters are simple HTTP transforms — within TinyGo's capability. |
| gen.Stage learning curve | Low | Well-documented pattern from Elixir's GenStage. Team will need to learn but concept is straightforward. |

## Decision Points for Team

1. **Accept Ergo Framework v3 for carrier mesh?** (vs raw goroutines + channels)
2. **Accept Wazero for REST adapters?** (vs Docker containers from original ADR)
3. **Accept KumoMTA for email delivery?** (vs Haraka, vs keep using SES)
4. **Accept hybrid adapter strategy?** (WASM for REST, Ergo for protocols)
5. **Accept go-plugin for WASM interface?** (vs custom host functions)

## Sources

- [Ergo Framework v3](https://github.com/ergo-services/ergo) — Actor model
- [Wazero](https://wazero.io/) — WASM runtime
- [Arcjet WASM in Production](https://blog.arcjet.com/lessons-from-running-webassembly-in-production-with-go-wazero/)
- [KumoMTA](https://kumomta.com) — Email MTA
- [knqyf263/go-plugin](https://github.com/knqyf263/go-plugin) — WASM plugin interface
- [gosmpp](https://github.com/linxGnu/gosmpp) — SMPP library
- [TinyGo WASM](https://tinygo.org/docs/guides/webassembly/) — Go to WASM compilation

---

*Generated: 2026-02-11 | For: Team Architecture Review*
