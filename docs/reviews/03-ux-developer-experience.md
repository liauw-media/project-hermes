# UX/Developer Experience Review: Project Hermes

**DX Readiness Score: 3/10**
**Date:** 2026-02-10

---

## Time-to-First-Message Benchmarks

| Platform | Time to First Message |
|----------|----------------------|
| Resend | ~2 min |
| Twilio | ~5 min |
| SendGrid | ~5 min |
| Vonage | ~8 min |
| Postmark | ~10 min |
| **Hermes (target)** | **< 3 min** |

---

## 5 Make-or-Break Recommendations

### 1. Ship a Sandbox Before Anything Else
Instant provisioning, no DB setup. Test mode that never hits real providers. Developers who can't send a test message in 5 minutes evaluate competitors instead.

### 2. Cascade Trace View Is the Killer Feature
No competitor shows: "Email sent 10:00 → bounced 10:02 → WhatsApp sent 10:03 → delivered 10:05". This timeline view in API responses AND dashboard generates word-of-mouth.

### 3. Launch with TypeScript and Python SDKs
These cover 80%+ of messaging API consumers. Go SDK is a natural differentiator given the platform is built in Go. Auto-generate from OpenAPI spec.

### 4. Design Errors as a First-Class Product
Every error needs: machine-readable code, human-readable message with correct example, param that caused it, doc_url link, request_id for support. Batch validation errors.

### 5. Build a Visual Cascade Builder
No competitor has drag-and-drop multi-channel fallback configuration. Combined with cascade trace, this is the competitive moat.

---

## Cascade Configuration: Triple-Mode

1. **Visual Builder (Dashboard):** Drag-and-drop channel cards, set timeouts, live preview, cost estimate
2. **JSON/YAML Config:** Import/export, config-as-code, API-specifiable
3. **Inline per-message (API):** Pass cascade rules directly in message payload

---

## Dashboard Requirements

- **Home:** Messages today, delivery rate by channel, cascade success rate, active alerts, spend
- **Message Explorer:** Searchable table, per-message cascade trace timeline (CRITICAL debugging tool)
- **Cascade Configs:** Visual builder, saved configs, test/simulate
- **Analytics:** Delivery rate over time, cascade depth, cost per delivered message, channel latency
- **Billing:** Balance, cost breakdown, invoices, usage alerts
- **Developer Tools:** API request log, interactive explorer, webhook debugger, code snippets with API key pre-filled

---

## Competitive Advantages (If Executed)

- Multi-channel cascade with state machine trace — unique
- 9-layer validation with compliance automation — enterprise-ready
- Idempotency at Layer 0 — ahead of most competitors
- Hot-loadable channel adapters — unique for enterprise
- Visual cascade builder — no competitor has this

## Critical Gaps

- No sandbox/test mode
- No OAuth login (add GitHub, Google alongside magic link)
- Enterprise manual provisioning (let them start on shared, migrate async)
- No MCP server / llms.txt (30%+ of API traffic from AI agents in 2026)
- No interactive API explorer
- No dashboard spec or UI code

---

## Path to 9/10

| Phase | Weeks | Score |
|-------|-------|-------|
| Foundation: OpenAPI spec, sandbox, TS+Python SDKs, quickstart docs, API explorer | 1-8 | 5/10 |
| Dashboard: cascade trace, visual builder, webhook debugger, error docs | 9-16 | 7/10 |
| Enterprise: migration guides, MCP server, cost analytics, CLI, status page | 17-24 | 9/10 |
