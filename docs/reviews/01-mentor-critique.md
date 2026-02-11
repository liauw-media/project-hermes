# Mentor Critique: Project Hermes

**Confidence Score: 92/100**
**Overall Score: 4/10**
**Date:** 2026-02-10

---

## VERDICT

Architecture design scores 8/10. Implementation readiness scores 1/10. Ship probability: 3/10.

The architecture is genuinely thoughtful. The Cascade Engine is the right abstraction. The tier model is sensible. The validation pipeline ordering is correct (cheap checks first). The adapter protocol with Ed25519 + heartbeat is solid. The IP reputation engine shows domain expertise.

But none of that matters without code.

---

## WHAT WILL KILL YOU

### 1. Analysis Paralysis (Probability: 85%)
Spending energy on diagrams instead of code. The project has documentation but zero source files.

### 2. Multi-Tenant Complexity (Probability: 70%)
Three different infrastructure topologies simultaneously. Each one is a startup's worth of work. Data migration between tiers (Starter to Business) alone is months of work.

### 3. Supabase at 100M Messages/Day (Probability: 60%)
100M messages x 5 DB operations = 5K-50K ops/sec. Schema-per-tenant breaks past ~50-100 schemas.

### 4. "100M from Day One" Delusion
Zero customers at design time. Build for 100K/day. Scale decisions come from real traffic patterns.

### 5. Hot-Loadable Container Security
Ed25519 handshake doesn't protect against what runs inside tenant-provided containers. Need gVisor/Firecracker.

---

## WHAT YOU'RE NOT THINKING ABOUT

- **Webhook reliability**: Duplicate, out-of-order, missing webhooks. Diagrams show happy paths only.
- **State machine persistence**: 100M state machines. Where do they live? Memory = lost on restart. DB = 100M more writes.
- **Exactly-once billing**: Two concurrent requests both pass balance check before either decrements. Needs saga patterns.
- **Multi-region**: Not mentioned once. Single-region is non-starter for enterprise.
- **GDPR data residency**: EU clients need data in EU. Changes entire data layer.
- **Observability at scale**: Prometheus/Loki struggle at 100M msg/day log volume.

---

## THE LANGUAGE QUESTION

**Verdict:** Go was chosen over Rust for pragmatic reasons — faster time-to-PoC (1-2 months vs 3-4), better hiring pool in Denmark, and more than sufficient performance for 100M/day.

---

## THE NO-FRAMEWORK QUESTION

**Use a framework.** Whether Rust (Axum) or Go (standard library net/http + Chi/Echo), don't reinvent HTTP handling. Build the Cascade Engine, not a web framework.

---

## WHAT'S ACTUALLY SMART

1. Cascade Engine as configurable state machine (correct abstraction)
2. NATS JetStream over Kafka for initial scale (simpler operations)
3. Validation pipeline ordering (cheap checks first)
4. Adapter protocol design (Ed25519 + bcrypt + heartbeat)
5. IP Reputation Engine scope (real domain expertise)
6. Tier-based isolation model (correct cost/isolation tradeoff)

---

## ACTION ITEMS

### This Week
1. `go mod init` — Create the project workspace
2. Build ONE endpoint: `POST /v1/messages` → validate → publish to NATS → return 202
3. Build the Cascade Engine for one channel (email/SendGrid)
4. Get an email from API to inbox in an integration test

### Never (until message flow works end-to-end)
- Tenant provisioner, billing system, adapter registry, reputation engine, or Kubernetes deployment

---

## SCORES

| Category | Score |
|---|---|
| Architecture Design | 8/10 |
| Implementation Readiness | 1/10 |
| Technology Choices | 7/10 (improved with Go decision) |
| Scope Realism | 2/10 |
| Ship Probability | 3/10 |

**Decision:** Go chosen over Rust. Ship the PoC. Iterate.
