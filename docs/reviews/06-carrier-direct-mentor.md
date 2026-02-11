# Carrier-Direct Architecture Pivot: Ruthless Mentor Review

**Date:** 2026-02-11
**Overall Score:** 4/10
**Verdict:** Vision is correct. Timeline is fiction.

---

## Scores

| Dimension | Score | Evidence |
|-----------|-------|----------|
| **Vision** | 8/10 | Cascade Engine is genuine insight. Multi-channel fallback + carrier-direct economics is a real business. Protocol gateway for market breadth is smart. |
| **Technical Feasibility** | 5/10 | Software architecture is solid (hexagonal, NATS, Go). SMPP/MTA/BSP infrastructure dramatically underestimated. gosmpp gets 60% of the way; other 40% is carrier-specific quirks. |
| **Execution Risk** | 9/10 | Zero code, zero contracts, zero team SMPP experience. Five simultaneous infrastructure bets. 30-week timeline for what took competitors 3-5 years. |
| **Timing Alignment with PoC** | 4/10 | Phase 1 correctly scoped for PoC. But ADR treats Phase 1 as stepping stone rather than the actual product. |
| **Overall** | 5/10 | Carrier-direct VISION is correct. Carrier-direct TIMELINE is fiction. |

---

## 1. Reality Check on Carrier-Direct (3/10)

ADR says "Phase 2: Weeks 5-10" for first direct carrier SMPP connection to Telia DK. This is delusional.

**Carrier Contract Requirements (Telia DK / TDC / 3 DK / Telenor):**
- Registered Danish business entity (CVR number) with track record required
- Carrier sales cycle: 2-6 months from first contact to signed contract
- Security deposit: 50,000-200,000 DKK per carrier
- Technical qualification: must demonstrate working SMPP implementation on test/staging SMSC first
- Minimum monthly commitment: often 100K+ messages/month
- IP whitelisting: static source IPs, pre-approved
- Short code or alphanumeric sender ID registration with Danish telecom authority

**Who on the team has ever negotiated a carrier contract?** This is a business development problem, not an engineering problem.

---

## 2. SMPP Complexity (5/10)

ADR shows Wikipedia-level SMPP awareness. Real failure modes:

1. **Enquire_link timeouts** vary per carrier (15s, 30s, 60s). Miss the window = carrier drops your bind. Must handle PER CARRIER, not global setting.
2. **Split binds vs transceiver** — carriers have different requirements. Telia DK historically requires transceiver; TDC allows both. You discover this AFTER signing.
3. **Window fills** — when carrier SMSC is slow, your window fills and you MUST stop sending. Need per-connection window tracking and backpressure propagation.
4. **DLR reliability** — delivery receipts are "best effort" per SMPP spec. Some carriers drop them under load, delay by hours, or send DLRs for failed messages. Your 60s SMS timeout will trigger false fallbacks.
5. **Encoding** — single emoji drops SMS from 160 to 70 chars (GSM-7 to UCS-2). Billing must count SEGMENTS, not messages.
6. **No TLS** — SMPP 3.4 has zero native encryption. Carrier connections send content in PLAINTEXT. GDPR compliance issue.
7. **Mid-message disconnects** — carrier drops TCP while you have in-flight PDUs. Messages in UNKNOWN state. Reconciliation logic is yours to write.

---

## 3. MTA Operations Hell (2/10)

ADR claims "$50K/yr own MTAs vs ~$600K/yr SES." This is technically accurate and operationally catastrophic.

**The real cost at 6B emails/year:**

| Item | Cost |
|------|------|
| Infrastructure (servers, IPs) | $50K |
| IP warming period (SES overlap, 6 months) | $200K |
| Deliverability team (1 engineer + tools) | $150K |
| Blacklist monitoring services | $30K |
| Incident response | $50K |
| **Total** | **~$480K/year** |

**Break-even vs SES is at ~3B emails/year, not zero.** At 1B emails/year, you're LOSING money vs SES. Real savings at 6B: ~$120K/year, not $550K.

**The tenant spam problem kills you.** One bad Starter-tier tenant sends a spam batch, your entire shared IP pool reputation drops. Gmail applies sender reputation at IP AND domain level. Pre-flight content analysis doesn't help — ISPs care about recipient engagement, not your content scanner.

---

## 4. Smart Routing vs Reality (6/10)

**HLR Economics:**
- Cost: $0.003-$0.01 per lookup
- At 2B texts/year without caching: $6M-$20M/year
- Must cache with 7-14 day TTL (Danish porting rate ~2-3%/year)
- At 7-day TTL: ~2B lookups/year = $6M-$20M
- Sometimes it's cheaper to just route via aggregator than to do HLR + direct

Quality scoring per route is the right idea. Geographic routing is the obvious split. Fallback from direct to aggregator is correct.

**Missing:** Real-time carrier health detection, SMPP window-based backpressure feeding routing decisions, cost optimization that accounts for HLR lookup cost.

---

## 5. WhatsApp BSP (2/10)

Becoming a BSP takes 8-15 months minimum. Meta requires established business communications provider, technical audit (3-6 months), on-premises API hosting, minimum scale requirements, ongoing re-certification.

**Use WhatsApp Cloud API instead.** Takes days, not months. Phase 1 obvious approach that the ADR skips entirely.

---

## 6. Timing Mismatch (3/10)

**ADR timeline vs reality:**

| Phase | ADR | Reality |
|-------|-----|---------|
| Phase 1: MVP | Weeks 1-4 | Weeks 1-8 (achievable) |
| Phase 2: First carrier | Weeks 5-10 | Months 4-9 (needs signed contract first) |
| Phase 3: Own MTA + more carriers | Weeks 11-18 | Months 10-18 (8 weeks IP warming alone) |
| Phase 4: Full carrier-direct | Weeks 19-30 | Months 18-30 |

**The correct strategy:** Ship Phase 1 in 4-6 weeks. This IS the product for PoC customers. Nobody cares whether SMS goes via Twilio or Telia DK — they care about cascade fallback, unified API, delivery tracking. Start carrier contract negotiations DAY ONE in parallel.

---

## 7. Cost Model Reality (4/10)

**Email MTA economics (corrected):**
- Break-even vs SES: ~3B emails/year
- At 1B/year: LOSING $80K vs SES
- At 6B/year: saving ~$120K (not $550K)

**SMS direct economics (much stronger):**
- Twilio retail: $0.05/SMS, 10-20% margin
- Aggregator: $0.02-0.03/SMS, 30-40% margin
- Direct carrier: $0.005-0.01/SMS, 60-80% margin
- At 2B texts/year: $20M-$40M difference between aggregator and direct

**The email carrier-direct story is weak. The SMS carrier-direct story is strong. Focus accordingly.**

---

## 8. Competitive Moat vs Suicide Mission (5/10)

**What IS a moat:** Cascade Engine (nobody does this well), direct carrier relationships (6-12 months to replicate), Protocol Gateway (captures three market segments).

**What is NOT a moat:** Own MTAs (anyone with $500K can do it), smart routing (every CPaaS has this), HLR lookups (commodity).

**You are simultaneously trying to be:** SaaS platform company + telecom company + email infrastructure company + WhatsApp BSP. Each is a full company's worth of complexity. Bird took 10 years to do this.

---

## What You Should Actually Do

1. **Weeks 1-6:** Ship Phase 1. REST API, SendGrid, Twilio, basic cascade. Get into Danish customers' hands.
2. **Weeks 1-6 (parallel, biz dev):** Start carrier contract conversations. Hire a telecom consultant with existing carrier relationships ($2K-5K/month, saves 3-6 months).
3. **Weeks 7-14:** Add SMPP inbound for enterprise. Add smart routing against aggregator APIs.
4. **Months 4-9:** First direct carrier connection (Telia DK) once contract signed. Prove economics on real traffic.
5. **Months 9-18:** If SMS direct economics proven, add more carriers. Start MTA investigation only if email > 2B/year.
6. **Never (until $5M+ ARR):** WhatsApp BSP. Use Cloud API.

**The Cascade Engine is your moat. Carrier-direct is your margin play. Do not let the margin play kill the moat.**

---

## Final Verdict

Stop writing ADRs. Start writing Go.
