# Temporal.io Deep Dive: Cascade Engine Evaluation

> **Context Note (2026-02-16):** This research is directly relevant to the current MVP. Temporal.io is the primary cascade engine for the aggregator-first architecture. The cascade workflow pattern documented here works identically whether downstream adapters are aggregator REST APIs (MVP) or direct SMPP connections (Phase 5+).

## Executive Summary

Temporal.io is a durable execution platform that can replace the custom event-sourced cascade state machine identified as the weakest part of the Hermes architecture (scored 2/10 on state persistence by the system architect review). The key discovery: **Twilio runs every message through Temporal workflows** — proving this pattern works for messaging at scale.

## What is Temporal?

Temporal is a durable execution platform. Code runs as "workflows" that survive process crashes, server restarts, and network failures. When a Temporal worker crashes mid-execution, the workflow automatically resumes from exactly where it left off.

Core concepts:
- **Workflow**: A function that orchestrates long-running business logic. Deterministic. Survives crashes.
- **Activity**: A function that performs side effects (API calls, DB writes, sending messages). Can fail, timeout, be retried.
- **Signal**: An external event delivered to a running workflow (e.g., DLR webhook confirming delivery).
- **Timer**: Durable timer that survives crashes (cascade timeouts).
- **Query**: Read workflow state without mutating it (check cascade status).

## Why Temporal for Hermes Cascade

The cascade engine is fundamentally a workflow:
1. Receive message
2. Try Channel 1 (e.g., Email via SendGrid)
3. Wait for delivery confirmation (DLR) with timeout
4. If timeout/failure → Try Channel 2 (e.g., SMS via SMPP)
5. Wait for DLR with timeout
6. If timeout/failure → Try Channel 3 (e.g., Push via FCM)
7. Report final status to tenant webhook

This maps directly to Temporal:
- **Workflow** = Cascade (one per message)
- **Activity** = Send to channel (publishes to NATS, sub-ms)
- **Signal** = DLR webhook confirmation
- **Timer** = Per-step timeout (30s, 60s, etc.)
- **Query** = "What's the current cascade status?"

## Scale Proof

| Company | Scale | Use Case |
|---------|-------|----------|
| Twilio | Billions of messages | Every message = Temporal workflow |
| Netflix | Millions of workflows/day | Streaming orchestration |
| Snap | High throughput | Content delivery |
| Instacart | 75M workflows/month | Order fulfillment |
| PlanetScale | Production sharding | Database operations with 100+ activity-heavy workflows per second |
| Stripe | Payment orchestration | Transaction workflows |
| Coinbase | Blockchain transactions | Crypto operations |
| Box | File processing | Document workflows |
| Datadog | Pipeline orchestration | Data processing |

Temporal reports 2,500+ customers and 7 million deployed clusters. Go SDK is the most mature with 1,442 known Go package importers (v1.31+).

Hermes at 22M messages/day = 22M workflows/day = ~255 workflows/sec. This is well within proven Temporal capacity. Twilio processes significantly more.

**Key insight:** For cascade with 30-60 second timeouts between steps, Temporal's per-step overhead (50-150ms) is irrelevant — it's <0.5% of the step duration. Idle workflows consume zero processing power — they're just database records waiting for a signal or timer.

## Go SDK Details

Temporal's Go SDK (v1.31+) is first-class — Go was the first SDK Temporal built.

```go
// Cascade workflow in Temporal
func CascadeWorkflow(ctx workflow.Context, msg CascadeMessage) (*CascadeResult, error) {
    for i, step := range msg.CascadeRule.Steps {
        // Activity: send message to channel (publishes to NATS)
        activityCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
            StartToCloseTimeout: 30 * time.Second,
            RetryPolicy: &temporal.RetryPolicy{
                MaximumAttempts: int32(step.MaxRetries),
                BackoffCoefficient: 2.0,
            },
        })

        var sendResult SendResult
        err := workflow.ExecuteActivity(activityCtx, SendMessageActivity, msg, step).Get(ctx, &sendResult)
        if err != nil {
            // Activity failed after retries, try next channel
            continue
        }

        // Wait for DLR signal with timeout
        dlrCh := workflow.GetSignalChannel(ctx, "dlr-confirmation")
        timerCtx, cancel := workflow.WithCancel(ctx)
        timer := workflow.NewTimer(timerCtx, step.Timeout)

        selector := workflow.NewSelector(ctx)
        var dlr DeliveryReport
        var delivered bool

        selector.AddReceive(dlrCh, func(c workflow.ReceiveChannel, more bool) {
            c.Receive(ctx, &dlr)
            delivered = true
            cancel() // Cancel timer
        })

        selector.AddFuture(timer, func(f workflow.Future) {
            // Timeout — try next channel
        })

        selector.Select(ctx)

        if delivered {
            return &CascadeResult{
                Status: "DELIVERED",
                Channel: step.Channel,
                Step: i,
                DLR: dlr,
            }, nil
        }
    }

    return &CascadeResult{Status: "EXHAUSTED"}, nil
}
```

**Signal pattern for DLR confirmations:**
1. Workflow sends message via Activity, which returns a correlation ID
2. Workflow enters a `workflow.Selector` that races a Timer (timeout) against a Signal channel
3. External DLR webhook hits API server, which calls `client.SignalWorkflow(workflowID, "delivery_confirmed", dlrPayload)`
4. The Signal wakes the workflow, which inspects the DLR and decides next step

**Cascade event budget:** 3 channels x 2 retries each = ~30-50 events per cascade. Well within Temporal's 50,000 event limit. No need for `ContinueAsNew`.

**Activity retry policy example:**
```go
retryPolicy := &temporal.RetryPolicy{
    InitialInterval:    time.Second,
    BackoffCoefficient: 2.0,
    MaximumInterval:    time.Minute,
    MaximumAttempts:    3,
}
```

## Hybrid Architecture: Temporal + NATS

Direct provider calls from Temporal Activities add latency and couple the cascade engine to delivery infrastructure. Instead:

```
Temporal Activity → publishes to NATS JetStream (sub-ms)
NATS Consumer → calls provider API (SendGrid, SMPP, etc.)
Provider → DLR webhook → Hermes API → client.SignalWorkflow()
```

Benefits:
- Activity completes in sub-ms (NATS publish), not 100-500ms (HTTP to provider)
- NATS provides high-throughput fan-out and buffering
- Delivery consumers scale independently from cascade workers
- If provider is slow, NATS buffers — Temporal workflow just waits for DLR signal

## Performance Characteristics

| Metric | Value | Notes |
|--------|-------|-------|
| Workflow start latency | 50-150ms | Includes persistence write |
| Local activity latency | 10-30ms | In-process, no persistence |
| Activity dispatch | 50-100ms | Worker poll + execute |
| Timer resolution | ~1 second | Acceptable for cascade timeouts |
| Signal delivery | <100ms | From API call to workflow receipt |
| Throughput | Thousands of workflows/sec per namespace | PlanetScale runs 100+ activity-heavy/sec |

At Hermes scale (255 workflows/sec average, ~1,157/sec peak):
- Well within single-namespace capacity
- Can shard by tenant_id if needed (Temporal supports multiple namespaces)
- PlanetScale's sharding blog details handling similar scale

## Persistence Options

| Backend | Fit | Notes |
|---------|-----|-------|
| PostgreSQL | Good for MVP/dev | Simplest. Use same PG as tenant data. **NOT recommended at 22M/day.** |
| MySQL | Good | Well-tested, good performance |
| Cassandra | Required at scale | For high-throughput. Vymo Engineering switched from PG to Cassandra under load. |
| SQLite (experimental) | Dev only | Single-node development |

**Important:** PostgreSQL cannot horizontally shard natively. Temporal's documentation states "PostgreSQL is not ideal for medium-to-large-scale systems." For the MVP prototype, PostgreSQL is fine. For production at 22M/day, plan to use Cassandra + Elasticsearch.

Recommendation: Start with PostgreSQL (prototype and early production). Plan Cassandra migration before scaling beyond ~1M workflows/day.

## Known Limitations at High Throughput

| Limitation | Impact | Mitigation |
|-----------|--------|------------|
| `numHistoryShards` is immutable | Cannot change after initial deployment. Controls parallelism. | Set correctly before production (2048-4096 recommended). Each shard at 10ms DB latency caps at ~100 updates/sec. |
| History replay memory | Workflows with >10,000 events cause significant degradation during replay | Cascade has ~30-50 events. Use `ContinueAsNew` if needed. |
| 10,000 signals per execution | Hard limit on signals per workflow | Not a concern for cascade (typically 1-6 signals) |
| 50,000 event history limit | Performance degrades above ~10,000 events | Not a concern for cascade |
| No built-in provider rate limiting | Must implement in Activity code | Use task queue rate limiting or implement in NATS consumer layer |
| Worker concurrency is static | Doesn't adapt to resource availability | GitHub issue #8356 tracks "Resource-Aware Worker Concurrency" |
| Database is always the bottleneck | Temporal acknowledges this in docs | Cassandra + proper shard count. PlanetScale reported 100K-200K QPS at peak. |

## Operational Requirements

**Self-hosted production cluster:**

| Component | Count | Resources |
|-----------|-------|-----------|
| Frontend service | 2-3 instances | 2 vCPU, 4GB RAM each |
| History service | 3-6 instances | 4 vCPU, 8GB RAM each (scales with shards) |
| Matching service | 2-3 instances | 2 vCPU, 4GB RAM each |
| Worker service | 2-3 instances | 2 vCPU, 4GB RAM each |
| Cassandra cluster (production) | 3-5 nodes | 8 vCPU, 32GB RAM each |
| Elasticsearch | 3 nodes | 4 vCPU, 16GB RAM each |
| Your worker fleet | 5-20 instances | Depends on activity workload |

**For development:**
- Temporal Server (Go binary, ~4 services: frontend, history, matching, worker)
- Database (PostgreSQL for dev)
- Optional: Elasticsearch/OpenSearch (for workflow search/visibility)

**Docker Compose (development):**
```yaml
services:
  temporal:
    image: temporalio/auto-setup:latest
    ports:
      - "7233:7233"  # gRPC frontend
    environment:
      - DB=postgresql
      - DB_PORT=5432
      - POSTGRES_USER=temporal
      - POSTGRES_PWD=temporal
      - POSTGRES_SEEDS=postgresql
    depends_on:
      - postgresql

  temporal-ui:
    image: temporalio/ui:latest
    ports:
      - "8080:8080"
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
```

**Production considerations:**
- Self-hosted cost: $5K-10K/mo infra + DevOps time (~$20K-40K/mo total)
- Temporal Cloud cost at Hermes scale: ~8-12 actions per cascade x 22M cascades/day = 176M-264M actions/day. At $25/million = **$132K-$198K/month** (disqualified on cost)
- Need monitoring: Temporal exposes Prometheus metrics natively
- Need alerting on: workflow failures, activity timeouts, task queue backlog
- **Critical:** `numHistoryShards` is immutable once set — must be configured correctly before production (can't change later without migration)
- At scale, real bottleneck is the persistence layer. PostgreSQL scales to medium volume. For 22M+/day, evaluate Cassandra + Elasticsearch.

## Comparison with Restate

| Dimension | Temporal | Restate |
|-----------|----------|---------|
| Maturity | 5+ years, battle-tested | ~2 years, newer |
| Go SDK | First-class, mature | Functional but newer |
| Latency | Higher (50-150ms/step) | Lower (3-10ms/step) |
| Operations | Heavier (cluster + DB) | Lighter (single Rust binary) |
| Scale proof | Twilio, Netflix, 2,500+ customers | 94K actions/sec, 8,571 full workflows/sec |
| Debugging | Temporal Web UI (excellent) | CLI + basic UI |
| Community | Large (12K+ GitHub stars) | Growing (5K+ stars) |
| Self-hosted | Well-documented | Simpler deployment |
| At Hermes scale | Proven | Needs validation |

### Recommendation: Prototype Both

Temporal is the safer choice (proven at Twilio's scale, mature Go SDK). But Restate's lower latency and simpler operations are appealing. Prototype the same 3-step cascade on both, compare:
1. Code simplicity and readability
2. Latency per step (measured with OpenTelemetry)
3. Recovery behavior after process kill
4. Operational complexity (what do I need to run?)
5. Debugging experience (can I inspect cascade state?)

Pick winner before Phase 2.

### Restate Detailed Benchmarks

| Metric | Value |
|--------|-------|
| Median latency per durable step (low load) | **3ms** |
| p99 latency per durable step (low load) | 54ms |
| Median latency (high load, 1200 concurrent) | **10ms** |
| p99 latency (high load, 1200 concurrent) | 98ms |
| Full workflow throughput | **8,571 workflows/sec** |
| Single binary deployment | Yes (Rust, built-in RocksDB) |
| External database required | No |

Restate's latency advantage is significant but matters less for Hermes because cascade step timeouts are 30-60 seconds — the difference between 3ms and 100ms per step is negligible against these timeouts.

## Restate Alternative: How It Maps

```go
// Cascade as Restate Virtual Object (keyed by message_id)
type CascadeHandler struct{}

func (h *CascadeHandler) StartCascade(ctx restate.ObjectContext, msg CascadeMessage) (*CascadeResult, error) {
    for i, step := range msg.CascadeRule.Steps {
        // Durable RPC call (survives crashes)
        sendResult, err := restate.Object[SendResult](ctx, "delivery", msg.ID, "Send").
            Request(SendRequest{Message: msg, Step: step})
        if err != nil {
            continue
        }

        // Awakeable: wait for external DLR confirmation
        awakeable := restate.Awakeable[DeliveryReport](ctx)
        // Store awakeable ID for webhook handler to resolve
        restate.Set(ctx, "dlr-awakeable-"+msg.ID, awakeable.Id())

        // Race: DLR confirmation vs timeout
        result, err := restate.Race(ctx,
            awakeable,
            restate.Sleep(ctx, step.Timeout),
        )

        if dlr, ok := result.(DeliveryReport); ok {
            return &CascadeResult{Status: "DELIVERED", Channel: step.Channel, DLR: dlr}, nil
        }
    }
    return &CascadeResult{Status: "EXHAUSTED"}, nil
}
```

## Other Durable Execution Alternatives Evaluated

| Platform | Verdict | Why |
|----------|---------|-----|
| **Inngest** | Poor fit | Event-driven model, wrong abstraction for complex cascade state machines. 450K actions/sec claimed. |
| **DBOS** | Disqualified | PostgreSQL-only, cannot scale to 22M concurrent workflows. Library approach limits orchestration. |
| **Hatchet** | Disqualified | Native Go but cannot handle 22M concurrent workflows. Benchmarks show similar to Temporal for small scale but significantly worse wait times at 10 workers (19.2s vs 2.1s Temporal). |
| **Custom on NATS** | High risk | Maximum performance but 3-6 months build + ongoing maintenance. |

### Temporal vs Custom NATS Event Sourcing

| Feature | Temporal (built-in) | Custom on NATS (you build) |
|---------|---------------------|---------------------------|
| Durable timers | `workflow.Sleep()` | Implement timer service with persistent storage |
| Signal/callback | `workflow.GetSignalChannel()` | Implement correlation engine matching DLRs to active cascades |
| Automatic retry with backoff | `RetryPolicy{}` | Implement retry scheduler with dead-letter queues |
| Workflow state machine | Implicit (code is the state machine) | Explicit state machine with event transitions |
| History/audit log | Automatic, queryable | Implement event store with replay capability |
| Concurrent workflow limit | Built-in task queue concurrency | Implement semaphore/rate limiter |
| Workflow visibility/search | Built-in search attributes | Implement separate indexing system |
| Crash recovery | Automatic replay from history | Implement replay engine reading from JetStream |
| Timeout management | `ScheduleToCloseTimeout`, `HeartbeatTimeout` | Implement timeout watchers |
| Debugging UI | Temporal Web UI | Build custom UI or use raw JetStream inspection |

**Estimated effort for NATS-based custom solution: 3-6 months of dedicated engineering.** This is exactly the custom infrastructure code that Temporal eliminates.

## Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Temporal adds 50-150ms per step | Medium | Use hybrid: Activity just publishes to NATS (sub-ms). Actual delivery via NATS consumer. |
| Temporal Cloud too expensive | High | Self-host from day 1. Cloud is $132K-540K/mo at our scale. |
| Temporal operational overhead | Medium | Start with PostgreSQL backend. Add Cassandra only when needed. |
| Go SDK breaking changes | Low | SDK is mature (v1.31+), stable API. |
| Single point of failure | Medium | Temporal cluster with multiple history shards. PostgreSQL with replicas. |

## Decision Points for Team

1. **Accept Temporal as primary cascade engine?** (with Restate prototype as comparison)
2. **Accept hybrid Temporal + NATS architecture?** (Activities publish to NATS, not direct provider calls)
3. **Start with PostgreSQL persistence?** (vs Cassandra from day 1)
4. **Self-host from day 1?** (Temporal Cloud disqualified on cost)

## Sources

- [Temporal Go SDK](https://github.com/temporalio/sdk-go)
- [Temporal at Twilio](https://medium.com/@milinangalia/the-rise-of-temporal)
- [PlanetScale Temporal Sharding](https://planetscale.com/blog/temporal-workflows-at-scale-sharding-in-production)
- [Restate Go SDK](https://github.com/restatedev/sdk-go)
- [Restate Benchmarks](https://www.restate.dev/blog/building-a-modern-durable-execution-engine-from-first-principles)
- [Temporal Architecture](https://docs.temporal.io/temporal)

---

*Generated: 2026-02-11 | For: Team Architecture Review*
