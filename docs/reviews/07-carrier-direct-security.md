# Carrier-Direct Security Audit: Project Hermes

**Date:** 2026-02-11
**Overall Score:** 3.5/10
**Previous Score (API-wrapper):** 7.0/10
**Verdict:** Carrier-direct pivot blew the threat model wide open.

---

## Scores

| Domain | Score |
|--------|-------|
| SMPP Security Posture | 3/10 |
| MTA Security Posture | 5/10 |
| Credential Management | 2/10 |
| Protocol Gateway Security | 4/10 |
| Regulatory Compliance Readiness | 3/10 |
| **Overall** | **3.5/10** |

---

## Three Non-Negotiables Before Any Carrier Connection Goes Live

1. **TLS on all SMPP connections** (enterprise-facing AND carrier-facing)
2. **Secrets management** (Vault/HSM for carrier credentials, DKIM keys, HLR tokens)
3. **Danish telecom registration** with Rigspolitiet's Tele Data section

---

## Critical Findings

### F1: SMPP Connections Transmit Everything in Plaintext (CRITICAL, CVSS 9.1)

SMPP 3.4 has ZERO native encryption. bind_transceiver transmits system_id and password as plaintext octets. Every submit_sm (including OTP codes) traverses the wire unencrypted.

**Attack scenarios:** Credential theft via network sniffing, message interception (OTPs, financial alerts), TCP session hijacking, GDPR Article 32 violation.

**Fix:** Mandate SMPPS (SMPP over TLS) on port 2776 for enterprise. Use stunnel or VPN for carrier-side. Minimum TLS 1.2.

### F2: No Secrets Management for Carrier Credentials (CRITICAL, CVSS 9.0)

4+ carrier credentials, DKIM private keys, HLR API keys, WhatsApp tokens, aggregator keys — zero specified storage mechanism. DevOps previously scored secrets at 0/10.

**Fix:** Deploy HashiCorp Vault with KMS-backed auto-unseal. DKIM keys in HSM. K8s External Secrets Operator. Emergency rotation runbook (< 15 min).

### F3: SMPP PDU Injection/Parsing Vulnerabilities (HIGH, CVSS 8.1)

No validation rules for SMPP-specific attacks: oversized command_length (4GB allocation DoS), malformed TLVs (OOB reads), data_coding exploits (XSS via garbled encoding), UDH concatenation attacks (unbounded memory), source_addr injection (tenant impersonation).

**Fix:** Hardened PDU validator: command_length < 64KB, source_addr validated against tenant's registered sender IDs, data_coding must match encoding, fuzz-test gosmpp parser.

### F4: gosmpp Supply Chain Risk (HIGH, CVSS 7.5)

Single maintainer (linxGnu), ~171 stars, zero security audits, no CVEs registered (nobody has looked). Library parses untrusted binary data from network sockets — highest-risk code category.

**Fix:** Commission security audit of gosmpp PDU parsing. Fork into Hermes org. Evaluate alternatives (fiorix/go-smpp). Run govulncheck in CI.

### F5: HLR Lookups as GDPR-Regulated PII Processing (HIGH, CVSS 7.0)

Phone numbers sent to third-party HLR provider without DPA (Data Processing Agreement). HLR responses contain IMSI, carrier, roaming status = additional personal data. Cross-border transfer if provider is outside EU.

**Fix:** Execute DPA under GDPR Article 28. Verify EU processing. Document lawful basis (legitimate interest). Consider DPIA.

### F6: Multi-Protocol Auth Inconsistency (HIGH, CVSS 7.2)

REST uses API keys, SMPP uses system_id/password, SMTP uses SASL. No unified permission model. Tenant rate-limited on REST could bypass via SMPP. SMPP submit_sm_resp sent BEFORE validation pipeline completes.

**Fix:** Canonical TenantSecurityContext resolved after auth, regardless of protocol. Unified rate limiting across all protocols.

### F7: MTA Open Relay / Tenant Abuse (HIGH, CVSS 7.8)

One compromised Starter-tier tenant sends 100K phishing emails through shared MTA pool. Gmail/Microsoft blacklist those IPs. ALL Starter-tier email delivery ceases.

**Fix:** URL reputation checking (Safe Browsing, PhishTank). New tenants start at 100 emails/hour. Auto-quarantine at > 0.1% complaint rate. Probationary IP sub-pools for new tenants.

### F8: DLR Spoofing (MEDIUM, CVSS 6.5)

Without TLS, attacker injects fake deliver_sm PDUs claiming messages delivered/not delivered. Corrupts delivery stats, cascade state, billing.

**Fix:** TLS on carrier connections. HMAC-SHA256 correlation binding. DLR rate limiting.

### F9: Carrier Connection Hijacking (MEDIUM, CVSS 6.0)

TCP RST injection, BGP hijacking of carrier IPs, DNS poisoning of SMPP endpoints, bind credential replay.

**Fix:** Hardcode carrier SMPP endpoint IPs. TLS with cert pinning. Connection health heuristics.

### F10: DKIM Key Management Gaps (MEDIUM, CVSS 6.5)

Per-tenant DKIM keys (2048-bit RSA) with no specified generation, storage, rotation, or revocation. Leaked DKIM key = perfectly authenticated phishing from tenant's domain.

**Fix:** Generate in HSM/Vault Transit. 6-month rotation. Ed25519 DKIM (RFC 8463) for new deployments.

### F11: Danish Telecom Regulatory Non-Compliance (HIGH, regulatory)

Operating SMPP connections to carriers without registering with Rigspolitiet is a criminal offense under the Danish Tele Act. No lawful intercept capability. No data retention for targeted orders.

**Fix:** Engage Danish telecom regulatory attorney immediately. Register before carrier connections go live. Design ETSI LI interface.

### F12: WhatsApp BSP Token Exposure (MEDIUM, CVSS 6.0)

BSP system tokens provide broad Meta API access. Token compromise = impersonation of any tenant's WhatsApp business.

**Fix:** Vault storage. Per-tenant WhatsApp Business Account isolation. Quality rating monitoring.

### F13: Smart Router Data Sensitivity (LOW, CVSS 4.0)

Carrier pricing, capacity, quality scores = competitive intelligence.

**Fix:** Encrypt at rest. Admin-only access. Audit log reads.

---

## Architecture Recommendations

1. **Carrier Security Gateway** — TLS termination, PDU validation, rate limiting, credential management between gosmpp and carrier/enterprise endpoints
2. **Protocol-Agnostic TenantSecurityContext** — immutable identity object flowing through entire pipeline regardless of ingress protocol
3. **Separate network segments** — enterprise-facing DMZ, carrier-facing DMZ, internal zone
4. **Formal STRIDE threat model** for all new carrier-direct attack surfaces

---

## Remediation Priority

| Priority | Findings | Effort |
|----------|----------|--------|
| P0 (Block production) | F1: SMPP TLS, F2: Secrets, F11: Telecom registration | 2-4 weeks |
| P1 (Before first tenant) | F3: PDU validation, F4: gosmpp audit, F6: Auth, F7: MTA abuse | 2-3 weeks |
| P2 (Before scale) | F5: HLR GDPR, F8: DLR integrity, F10: DKIM | 2 weeks |
| P3 (Ongoing) | F9: Connection resilience, F12: WhatsApp, F13: Routing data | 1 week |
