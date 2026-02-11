# Carrier-Direct UX/DX Review: Project Hermes

**Date:** 2026-02-11
**Overall Score:** 4/10
**Verdict:** The risk is not architecture. The risk is experience.

---

## Scores

| Dimension | Score |
|-----------|-------|
| Developer Onboarding Experience | 5/10 |
| API Design Clarity | 7/10 |
| Enterprise SMPP Experience | 3/10 |
| Ops Dashboard/Monitoring UX | 3/10 |
| Cost/Routing Transparency | 4/10 |
| **Overall DX** | **4/10** |

---

## Key Findings

### 1. SMPP Error Translation is the #1 DX Gap (4/10)

When a message fails via SMPP, the error is `0x00000058` (ESME_RTHROTTLED). This means nothing to 99% of developers.

**Need a Hermes Error Taxonomy:**
- Developer-facing: `DELIVERY_THROTTLED` + "Message delivery was throttled by the carrier. Retry after 5 seconds."
- Debug-level: `SMPP status: 0x00000058 (ESME_RTHROTTLED) from Telia DK`
- Every error code links to a documentation page (doc_url pattern)

Channel-agnostic error codes: `RECIPIENT_INVALID`, `RECIPIENT_OPTED_OUT`, `DELIVERY_THROTTLED`, `DELIVERY_REJECTED`, `CARRIER_UNAVAILABLE`, `CONTENT_REJECTED`, `SENDER_BLOCKED`.

### 2. Cost Unpredictability is a Wallet Bomb (5/10)

Same message, same number, two different days: 0.02 DKK (direct) or 0.15 DKK (Twilio fallback). That's a 7.5x cost variance invisible to the developer.

**Need:** Per-message cost in webhooks, real-time spend dashboard, cost alerts, route-preference API (`max_cost_dkk`, `cost_preference: "cheapest"/"fastest"`), spend cap with automatic cutoff.

### 3. Sandbox is Critical Path (2/10)

Before the pivot, sandbox was easy: mock SendGrid/Twilio. After the pivot, must simulate SMPP carriers, MTA delivery, HLR lookups, smart routing decisions, DLR flows, ISP feedback.

**Tier 1 sandbox (instant, simulated):** Test numbers that trigger specific behaviors (+4500000001 = direct delivery, +4500000003 = SMPP failure with cascade). No real carrier traffic. Works on signup.

**Tier 2 sandbox (staging, real):** Carrier test environments, dedicated MTA test IPs, real HLR lookups.

### 4. "3 Minutes to First Message" is Achievable (7/10)

But ONLY if first message uses provider API path (SendGrid), not carrier-direct. Sandbox must bypass Layer 6 compliance for test mode. SMPP/carrier complexity must be invisible during onboarding.

### 5. Enterprise SMPP Onboarding is a Black Hole (3/10)

Zero documentation, zero test environment, zero configuration guides, zero example code for enterprise clients connecting via SMPP. Need: SMPP sandbox with error injection, connection configuration guide, reference configs for Kannel/jSMPP/python-smpp.

### 6. Multi-Protocol Tracing is a Nightmare (5/10)

13 hops across 3 protocols with an async DLR gap. Standard OpenTelemetry headers don't work for SMPP.

**Fix:** Embed trace ID in SMPP message_id format (`HRM-{short_trace_id}-{seq}`). Map carrier_message_id -> trace_id in fast KV store. Developer-facing trace is a simplified timeline showing route, carrier, cost, and timing.

### 7. Persona-Based Onboarding Missing (4/10)

- Indie developer (email-only): should NEVER see SMPP, carriers, HLR
- SMB (email + SMS): gentle cascade introduction, cost tips after 1K messages
- Enterprise (SMPP + full control): SMPP config page immediately, dedicated onboarding support

Same platform, different entry points. API surface identical. Dashboard adapts to persona.

### 8. Ops Dashboards Don't Exist (3/10)

SMPP needs: connection status per carrier, window utilization, submit/DLR rates, error distribution, rebind frequency, enquire_link RTT.

MTA needs: per-IP reputation score, warming progress, blacklist status, ISP feedback, DKIM/SPF/DMARC status. This is a full-time discipline without purpose-built dashboards.

---

## Top 10 Recommendations

| # | Recommendation | Effort | Impact |
|---|---------------|--------|--------|
| 1 | Define Hermes Error Taxonomy | 3-5 days | Eliminates #1 developer frustration |
| 2 | Build Tier 1 Sandbox | 2-3 weeks | Unlocks 3-min onboarding |
| 3 | Formalize webhook payload schema (route, carrier, cost) | 2-3 days | Cost transparency |
| 4 | Build SMPP Operations Dashboard | 1-2 weeks | Ops can't run 50+ connections blind |
| 5 | Build MTA Operations Dashboard | 1-2 weeks | MTA without visibility = deliverability disaster |
| 6 | Implement cost preference API + spend caps | 1 week | Prevents cost explosion |
| 7 | Build SMPP Enterprise Onboarding Guide | 1 week | Enterprise clients can't onboard without docs |
| 8 | Implement persona-based onboarding flow | 1-2 weeks | Hide irrelevant complexity per persona |
| 9 | Define SMPP carrier failure automation | 1-2 weeks | The 3 AM carrier failure WILL happen |
| 10 | Implement trace ID propagation across SMPP | 1-2 weeks | Multi-protocol observability impossible without this |

---

## Closing

The carrier-direct architecture is technically sound and economically compelling. But it's designed from the inside out (how the system works) rather than the outside in (how the user experiences it). Every layer of carrier-direct complexity that leaks to the developer without translation is a developer who evaluates Resend or Twilio instead.

Build the abstraction layer before building the infrastructure.
