# Carrier-Direct Architect Review: Project Hermes

**Date:** 2026-02-11
**Overall Score:** 4/10
**Verdict:** As a diagram, 8/10. As a buildable spec for carrier-grade infrastructure, 4/10.

---

## Scores

| Dimension | Score |
|-----------|-------|
| Protocol Gateway Design | 6/10 |
| Smart Router Architecture | 5/10 |
| SMPP Infrastructure Design | 4/10 |
| MTA Architecture | 3/10 |
| Backpressure Handling | 2/10 |
| State Persistence | 2/10 |
| Multi-DC/HA | 1/10 |
| **Overall** | **4/10** |

---

## Critical Architecture Gaps

### 1. SMPP Connection Management (4/10)

**The math works** at average load (63 msg/sec, 4100 msg/sec theoretical max). But:
- Connection allocation doesn't reflect Danish market share (Telia ~35%, TDC ~30%)
- No bind lifecycle management specified
- No enquire_link management for outbound carrier connections
- No carrier maintenance window handling
- No active-active vs active-passive decision

**Need:** SMPP Connection Manager as separate process with full connection state machine (DISCONNECTED -> CONNECTING -> BINDING -> BOUND -> HEALTHY -> DEGRADED -> DRAINING -> UNBINDING).

### 2. Smart Router: HLR Caching Problem (5/10)

- HLR lookup: 50-200ms domestic, 200-800ms international
- hlr-lookups.com rate limit: 100/sec (peak need: 634/sec = 6x over limit)
- Cache TTL tradeoff: 7-day TTL means ~0.2% stale entries (ported numbers routed to wrong carrier)
- Need TWO HLR providers with automatic failover
- Pre-warm cache nightly with top-N most-messaged numbers

**Smart Router should be embedded library in Cascade Engine, not separate service.** Pure function, locally cached inputs, no network hop needed.

### 3. MTA Architecture (3/10)

- Need 50+ marketing IPs, not 11 (warming math: 40-60 IPs needed for full volume after 8 weeks)
- No MTA queue strategy (Gmail throttles for 2 hours = 68.5GB queue backlog)
- No bounce processing pipeline (120M bounces/year at 2% rate)
- MTA technology undecided (Haraka vs custom Go)
- ISP rate limiting per IP per destination domain not specified

**Use NATS JetStream for MTA queue.** Route MTA delivery via NATS streams. Haraka consumers pull from NATS. Backpressure propagates naturally.

### 4. Cascade + Route Fallback (5/10)

Two-dimensional cascade problem: channel fallback (Email -> SMS -> Push) AND route fallback within channel (direct carrier -> aggregator -> Twilio).

**Use Option A:** Smart Router internally handles route-level fallback. Cascade Engine only sees channel-level success/failure. SmartRouter.Route() returns a RoutingDecision with AttemptLog showing all routes tried.

### 5. NATS JetStream Storage (5/10)

- 8B messages/year with 5-8x internal events = ~2000 events/sec
- 7-day retention at 2KB avg = 2.4TB
- Current spec: 100Gi per node (3 nodes = 300Gi). **Need 500Gi NVMe per node, 5 nodes.**
- NATS KV won't scale to 153M active cascade state keys
- **Use PostgreSQL for cascade state** (UPSERT, partition by tenant_id + date), event-source transitions to NATS for replay

### 6. DLR Correlation (NOT ADDRESSED)

Map carrier_message_id -> internal_message_id. At 63 msg/sec with 72h TTL = 16.3M active keys.

**Use Redis:** SET dlr:{carrier_id}:{carrier_msg_id} internal_id EX 259200. 1.6GB memory. Redis handles trivially. DLR processing via NATS stream (async, handles DLR storms).

### 7. Backpressure (2/10 — MOST DANGEROUS GAP)

Full chain: Carrier ESME_RTHROTTLED -> SMPP Connection Manager -> carrier health KV -> Smart Router -> Cascade Engine -> NATS -> API Gateway admission control.

**Need 5-layer backpressure:** SMPP backoff per connection, carrier health in NATS KV, channel availability checks, per-tenant cost circuit breaker (max_cascade_cost), API Gateway 503 with Retry-After at last resort.

### 8. State Persistence (2/10)

Pod crash after submit_sm sent but before submit_sm_resp = message in quantum state.

**Need event sourcing:** Every state transition written to NATS BEFORE taking action. SMPP in-flight tracking stream. Recovery: check "SUBMITTED_AWAITING_RESP" messages, mark as UNKNOWN after 30s, send to manual review queue (~150 messages per crash event = manageable).

### 9. Multi-DC (1/10 — CRITICAL OMISSION)

SMPP connections are IP-whitelisted per carrier. DC failover = new IPs = carriers reject you. MTA IPs have reputation tied to specific addresses. Failover = cold IPs = ISPs throttle.

**Phase 1:** Single DC (AWS eu-north-1, Stockholm). DR = route all traffic to aggregators/providers.
**Phase 2:** Active-standby in eu-central-1. Pre-whitelist SMPP IPs. Warm MTA IPs by sending 10-20% of traffic.
**Phase 3:** Active-active for 99.99% Enterprise SLA.

---

## Priority Implementation Order

1. Event-sourced cascade state (Week 1-2)
2. DLR correlation store in Redis (Week 2)
3. SMPP Connection Manager with full lifecycle (Week 3-5)
4. Backpressure chain carrier-to-gateway (Week 5-7)
5. Smart Router with HLR caching (Week 7-9)
6. Protocol Gateway unified format with extensions (Week 9-10)
7. MTA infrastructure (Week 10-16, includes 8-week warming)
8. Multi-DC preparation (Week 16-20)

---

## Key Decisions

| Question | Recommendation |
|----------|---------------|
| Smart Router service boundary? | Embedded library in Cascade Engine process |
| SMPP Connection Manager boundary? | Separate Kubernetes Deployment (2+ replicas) |
| Cascade state storage? | PostgreSQL with UPSERT, event-source to NATS |
| DLR correlation storage? | Dedicated Redis instance, 72h TTL |
| MTA queue? | NATS JetStream streams per pool |
| NATS storage? | 500Gi NVMe per node, 5 nodes |
