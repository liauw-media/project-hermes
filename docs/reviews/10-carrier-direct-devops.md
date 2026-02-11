# Carrier-Direct DevOps/SRE Review: Project Hermes

**Date:** 2026-02-11
**Overall Score:** 0.5/10
**Verdict:** The 0.5 is for documentation demonstrating genuine domain knowledge. But documentation is not operations.

---

## Scores

| Dimension | Score |
|-----------|-------|
| SMPP Operational Readiness | 0/10 |
| MTA Operational Readiness | 0/10 |
| Monitoring/Alerting Readiness | 0/10 |
| DR/HA Readiness | 0/10 |
| IaC/Automation Readiness | 0/10 |
| **Overall** | **0.5/10** |

---

## Critical Gaps

### SMPP Operations (0/10)
- No runbooks for carrier connection failures
- No carrier escalation contacts
- No maintenance window procedures
- No connection flapping detection
- No SMPP error code classification (which codes page on-call vs log-and-continue)
- No failover threshold defined ("Connection degraded" = what exactly?)

### MTA Operations (0/10)
- IP warming is an 8-week HARD BLOCKER that cannot be parallelized
- No warming automation implemented
- No blacklist monitoring (50+ blacklists, 1000+ checks every 30 min at 20 IPs)
- No ISP-specific throttling handling (Gmail, Yahoo, Microsoft all different)
- No bounce classification engine
- No suppression list management
- No DKIM key rotation procedure
- No ISP feedback loop registration procedures

### Monitoring (0/10)
- Zero carrier-specific metrics implemented
- Zero carrier-specific alerts defined
- Need ~20 SMPP metrics, ~15 MTA metrics, ~5 HLR metrics, ~5 routing metrics
- Need ~15 alert rules across SMPP/MTA/HLR

### Infrastructure (0/10)
- Zero Terraform files
- Zero Helm charts
- Zero Kubernetes manifests
- Zero Dockerfiles
- Zero CI/CD pipelines
- No static IP provisioning for SMPP
- No DNS automation for MTA (rDNS, SPF, DKIM, DMARC)
- No TLS certificate management

### DR/HA (0/10)
- No SMPP failover plan (in-flight messages lost on pod crash)
- MTA IP reputation not portable (failover to new IPs = cold reputation = ISPs throttle)
- No HLR cache persistence (loss = cost spike of $3K-$11K/hour during rebuild)
- No NATS stream mirroring across DCs

---

## Operational Cost Reality

| Item | Annual Cost |
|------|------------|
| 2 SREs (Denmark, senior) | $400K-$560K |
| 1 Deliverability specialist | $120K-$180K |
| 0.5 FTE carrier relations | $60K-$90K |
| MTA infrastructure | $50K-$80K |
| HLR lookup costs (after caching) | $20K-$50K |
| Blacklist monitoring service | $5K-$15K |
| SMPP carrier connection fees | $10K-$30K |
| **Total** | **$665K-$1.005M/year** |

**The $550K/year email savings is offset by ~$665K-$1M/year in operational costs.** Net savings marginal to negative at initial scale. Only clearly positive at >10B emails/year.

---

## Production Readiness Checklist (87 Items, 6 Phases, ~22 Weeks)

### Phase 0: Foundation (Weeks 1-4) — 5 items
- Go workspace, CI pipeline, secrets management, observability stack, base K8s manifests, Docker Compose for local dev

### Phase 1: SMPP Infrastructure (Weeks 3-8) — 20 items
- SMPP client wrapper, Connection Manager, PDU logging, error classification, DLR processing, metrics, dashboards, alerts, SMPP server (inbound), carrier sandbox accounts, production accounts, static IPs, TLS certs, integration tests, load tests, 5 runbooks

### Phase 2: HLR/Smart Routing (Weeks 5-8) — 12 items
- HLR provider contracts (2), lookup client with cache, failover logic, metrics, Smart Router service, routing table config, audit trail, dry-run mode, dashboard, 2 runbooks

### Phase 3: MTA Infrastructure (Weeks 5-14) — 29 items
- MTA technology selection, IP provisioning (20+), rDNS configuration, base MTA config, DKIM key generation/storage/rotation, DNS record automation, IP warming scheduler, warming monitoring, ISP registrations (Gmail/Microsoft/Yahoo), ARF processing, bounce classification, suppression lists, blacklist monitoring, automated delisting, queue monitoring, metrics, dashboards, alerts, **BEGIN 8-WEEK IP WARMING**, 6 runbooks

### Phase 4: Integration Testing (Weeks 10-14) — 14 items
- E2E tests (REST->SMPP->carrier, REST->MTA, SMPP inbound->outbound), failover tests (SMPP, MTA, HLR), load tests (315 SMS/sec, 950 emails/sec), security tests, chaos engineering

### Phase 5: Production Readiness (Weeks 12-16) — 15 items
- Carrier production cutover, IP warming completion verification, on-call rotation (3+ people), runbook peer review, dashboard review, alert testing, DR drills (SMPP + MTA), carrier contact verification, ISP registration verification, log searchability, secrets rotation test, cert expiry monitoring, cost monitoring, billing reconciliation

### Phase 6: Gradual Traffic Migration (Weeks 14-22) — 6 items
- 1% -> 5% -> 25% -> 50% -> 100% traffic migration with monitoring at each stage, post-migration cost review

---

## Storage Requirements

| Log Source | Volume/Day | 90-Day Retention |
|-----------|-----------|------------------|
| SMPP PDU logs | 2-5 GB | 200-490 GB |
| MTA SMTP logs | 16-33 GB | 1.4-3 TB |
| Carrier DLR logs | 1 GB | ~100 GB |
| Routing decision logs | 6-7 GB | ~590 GB |
| HLR lookup logs | < 1 GB | Negligible |
| ISP feedback reports | < 1 GB | Negligible |
| **Total** | **~26-47 GB/day** | **2.3-4.2 TB** |

---

## Strongest Recommendation

Do not build carrier-direct first. Launch with aggregator APIs (Twilio, SendGrid, Messente). Get customers, traffic, revenue. Then build carrier-direct as cost optimization. The hexagonal architecture makes this swap straightforward: implement SMPPClient and MTAService ports with aggregator adapters first, swap in carrier-direct adapters when ready.

The cost savings only make financial sense at high volume and only after subtracting operational costs. At low volume (first 1-2 years), operational overhead exceeds savings.
