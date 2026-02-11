# DevOps Operational Readiness Review: Project Hermes

**Operational Readiness Score: 1.5/10**
**Date:** 2026-02-10

---

## Summary

Strong architecture, zero implementation. The gap between diagrams and production-ready at 100M/day with SLAs is significant. Zero deployable artifacts, zero infrastructure-as-code, zero monitoring, zero runbooks, zero CI/CD.

---

## Detailed Scores

| Dimension | Score |
|-----------|-------|
| CI/CD Pipeline | 0/10 |
| Kubernetes Architecture | 2/10 |
| NATS Operations | 1/10 |
| Adapter Hot-Reload | 2/10 |
| Supabase Operations | 1/10 |
| Disaster Recovery | 0/10 |
| Secrets Management | 0/10 |
| Cost Optimization | 0/10 |
| Monitoring & Alerting | 1/10 |
| Runbooks | 0/10 |

---

## Key Findings

### CI/CD for Go (vs Rust)
Go simplifies CI/CD dramatically:
- Compile: seconds not minutes
- Docker: multi-stage with scratch/distroless, tiny images (~10-20MB)
- No cargo-chef complexity needed
- go test -race in CI for race condition detection
- golangci-lint for comprehensive linting

### NATS at Scale
Current 100Gi PVC undersized. At 100M/day, 7-day retention, R3: need 500Gi/node. Use KEDA for cascade-engine HPA based on nats_consumer_num_pending. Stream mirroring for DR.

### Kubernetes
Namespace strategy correct. Missing: NetworkPolicies (default deny all), RBAC (per-tenant ServiceAccount), PSS (restricted profile), OPA Gatekeeper/Kyverno for enterprise adapter images.

### DR Targets

| Component | RPO | RTO |
|-----------|-----|-----|
| NATS JetStream | 0 (zero loss) | < 5 min |
| PostgreSQL (enterprise) | < 5 min | < 10 min |
| API Gateway | N/A (stateless) | < 2 min |
| Cascade Engine | < 1 min | < 5 min |

### Cost at 100M/day
Infrastructure: target < $0.00001/message (~$1,000/day). Use spot instances for adapters (50-70% savings).

### SLO/SLI Definitions

| SLI | Target SLO |
|-----|-----------|
| API Availability | 99.95% |
| Delivery Success Rate | 99.5% |
| Ingest Latency (p99) | < 100ms |
| E2E Delivery (p95) | < 5 min |
| Cascade Fallback Rate | < 5% |

---

## Prioritized Ops Backlog

| # | Item | Effort |
|---|------|--------|
| 1 | Go workspace + CI/CD pipeline | 2-3 days |
| 2 | Docker Compose dev environment | 2-3 days |
| 3 | K8s base manifests + Helm charts | 3-5 days |
| 4 | Secrets management (Vault + ESO) | 2-3 days |
| 5 | NATS JetStream deployment + streams | 2-3 days |
| 6 | Observability stack (OTel, Prom, Grafana) | 3-4 days |
| 7 | HPA + KEDA configuration | 1-2 days |
| 8 | Alerting rules + PagerDuty | 1-2 days |
| 9 | DR: NATS mirroring + PVC snapshots | 2-3 days |
| 10 | Runbooks (NATS recovery, provider outage, IP blacklist, tenant migration) | 3-5 days |

**Total: 20-33 days (1 engineer) — faster than Rust estimates due to Go's simpler build/deploy**

---

## Critical Alerts (Day One)

1. API Gateway error rate > 5% for 2 min
2. NATS cluster no leader for 30s
3. Consumer lag > 100K messages
4. Dead letter rate > 1% over 5 min
5. Adapter heartbeat missing 3 intervals
6. PostgreSQL connection pool > 90%
7. Certificate expiry within 7 days
8. Pod crash loop (restarts > 5 in 10 min)
9. PVC usage > 85% on NATS nodes
10. Tenant API key auth failure spike
