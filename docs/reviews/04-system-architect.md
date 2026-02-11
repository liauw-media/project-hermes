# System Architecture Review: Project Hermes

**Architecture Rating: 6.5/10**
**Date:** 2026-02-10

---

## Scores

| Dimension | Score |
|-----------|-------|
| Design clarity | 9/10 |
| Technology choices | 5/10 (improved to 7/10 with Go + Redpanda consideration) |
| Failure handling | 4/10 |
| Multi-tenancy isolation | 7/10 |
| Scalability headroom | 6/10 |
| Operational readiness | 6/10 |
| Security | 8/10 |

---

## Language Decision: Go

Go was chosen over Rust for pragmatic reasons:
- Time to PoC: 1-2 months (vs 3-4 for Rust)
- Hiring in Denmark: much larger talent pool
- Performance: more than sufficient for 100M/day (bottleneck is provider APIs, not CPU)
- Compile times: 2-5 seconds vs 2-10 minutes
- Ecosystem: mature messaging libraries (nats.go, twilio-go, sendgrid-go)
- Goroutines map naturally to concurrent state machines
- Can rewrite hot paths in Rust later if needed

Note: Elixir/OTP was also considered (purpose-built for concurrent state machines, supervisor trees, hot code reload). Worth evaluating for Cascade Engine if team has Elixir experience.

---

## Top 5 Changes (Priority Order)

### 1. Replace Supabase with Self-Managed PostgreSQL + Read Replicas
Five-nines requires control of the data layer. Use AWS RDS / Cloud SQL with Multi-AZ, read replicas, RLS for tenant isolation. Supabase adds an uncontrollable dependency.

### 2. Implement Backpressure and Cost Circuit Breakers
Without these, a provider outage cascades into cost explosion. If SendGrid is down 2hrs, 5M emails cascade to SMS = $37K surprise bill. Need:
- Per-provider circuit breakers
- Per-channel retry queues with exponential backoff
- Tenant-configurable cascade policies ("do NOT fallback to SMS on email outage")
- Real-time spend tracking with automatic cutoff

### 3. Event-Source the Cascade Engine State
In-memory state machines without persistence = pod crash loses ~130K in-flight messages. Every state transition publishes to durable stream before commit. Use Kafka/Redpanda log compaction for latest-state-per-message-id. Redis sorted sets for timeout management.

### 4. Evaluate Redpanda as Primary Message Bus
NATS JetStream lacks key-based consumer routing, log compaction, and hard multi-tenant isolation. Redpanda (Kafka-compatible, no JVM) solves all three. Keep NATS for lightweight request-reply (heartbeats, health checks).

### 5. Use Standard Go HTTP Patterns
Go's standard library net/http is excellent. Use Chi or Echo for routing (minimal overhead). Don't build a custom framework.

---

## Critical Architecture Concerns

### State Machine Persistence
- Peak: ~652K concurrent state machines
- Memory: ~229 MB (trivially in-memory)
- Problem: pod crash loses all in-flight state
- Solution: event-sourced state + checkpoint/replay + consistent hashing by message_id

### Backpressure (Missing)
SendGrid down 2 hours at 60M emails/day = 5M queued. Without circuit breaker, all cascade to SMS. Need:
- Provider health circuit breaker (closed/half-open/open)
- Per-channel retry queue (not cascade)
- Tenant-level admission control
- Cost explosion protection

### Consistency During Deploys
Rolling deploy kills pod with in-flight state machines. Need graceful drain protocol: stop accepting new messages, checkpoint state, publish handoff events, new pod picks up.

### NATS vs Redpanda

| Feature | NATS JetStream | Redpanda |
|---------|---------------|----------|
| Key-based routing | No | Yes (partition key) |
| Log compaction | No | Yes |
| Exactly-once | Consumer-side dedup | Native |
| Multi-tenant isolation | Soft (subjects) | Hard (topic quotas) |
| Operations | Simpler | More complex but proven |

---

## Cost Model at 100M/day

Infrastructure: ~$12,500-$15,500/month
At 1B/day: ~$40-50K/month
Provider costs (pass-through): SMS $187K/mo, WhatsApp $25-400K/mo, Email $6K/mo
