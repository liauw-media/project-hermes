# Encryption Strategy Research: Signal Protocol & Content Protection

## Executive Summary

Evaluate where Signal Protocol (libsignal) and other encryption approaches fit within Hermes's architecture. Signal Protocol is the gold standard for end-to-end encrypted messaging — used by Signal, WhatsApp, Google Messages (RCS), Facebook Messenger, and Skype. However, Hermes operates as a message **broker**, not an endpoint, which constrains where true E2E encryption is applicable.

**Key finding:** Signal Protocol is directly relevant for two future Hermes channels (RCS, in-app messaging SDK) but is the wrong tool for content protection within the broker pipeline. Standard AES-256-GCM at rest + TLS 1.3 in transit covers the broker's needs.

---

## Signal Protocol Overview

**Repository:** [github.com/signalapp/libsignal](https://github.com/signalapp/libsignal)
**License:** AGPLv3 (important: copyleft implications for commercial use)
**Implementations:** Rust (core), Java, Swift, TypeScript
**No official Go implementation** — would need to use the Rust core via CGo/FFI, or a community Go port

### Core Cryptographic Primitives

| Component | Algorithm | Purpose |
|-----------|-----------|---------|
| Key Agreement | X3DH (Extended Triple Diffie-Hellman) | Initial key exchange between two parties |
| Message Encryption | Double Ratchet | Per-message key derivation with forward secrecy |
| Symmetric Cipher | AES-256-CBC or AES-256-GCM | Actual message content encryption |
| MAC | HMAC-SHA256 | Message authentication |
| Key Derivation | HKDF | Derive encryption/MAC keys from shared secret |
| Identity Keys | Curve25519 | Long-term identity verification |
| Ephemeral Keys | Curve25519 | Per-session key exchange |

### What Signal Protocol Provides

- **Forward secrecy:** Compromising current keys doesn't expose past messages
- **Future secrecy (self-healing):** Compromised keys are replaced in subsequent messages
- **Deniability:** Messages can't be cryptographically proven to come from a specific sender
- **Asynchronous setup:** X3DH allows initiating encrypted sessions even when recipient is offline
- **Per-message keys:** Every single message uses a unique encryption key

### Who Uses Signal Protocol

| Product | Usage | Scale |
|---------|-------|-------|
| Signal | Full app encryption | ~40M users |
| WhatsApp | Default E2E for all messages | 2B+ users |
| Google Messages | RCS E2E encryption | Hundreds of millions |
| Facebook Messenger | Default E2E (since late 2023) | 1B+ users |
| Skype | "Private Conversations" | Hundreds of millions |

---

## The Broker Problem: Why E2E Doesn't Fit Everywhere

Hermes is a message **broker** — it sits between the sender (tenant application) and the recipient (end user). The message flows:

```
Tenant App -> Hermes API -> Validation Pipeline -> Cascade Engine -> Channel Adapter -> Provider/Carrier -> End User
```

True E2E encryption (Tenant App <-> End User) means Hermes **cannot read message content**. This directly conflicts with Hermes's core functions:

| Hermes Function | Requires Reading Content | Conflict with E2E |
|----------------|------------------------|--------------------|
| Content scanning (Layer 7) | Yes — anti-spam, phishing URL detection | Cannot scan encrypted content |
| Compliance checking (Layer 6) | Yes — GDPR consent, TCPA opt-in, quiet hours | Cannot verify compliance of encrypted content |
| Template rendering | Yes — merge variables into templates | Cannot render encrypted templates |
| Channel format transformation | Yes — Email HTML vs SMS 160-char vs Push JSON | Cannot transform encrypted payloads |
| Message body size validation | Yes — SMS segment counting (GSM-7 vs UCS-2) | Cannot count segments of encrypted content |
| Audit logging (Layer 8) | Content hash for tamper evidence | Can hash encrypted blob, but loses audit value |
| Billing | Per-segment SMS billing | Cannot count segments without reading content |

**Conclusion:** For the broker pipeline, Hermes needs to read message content. E2E encryption between tenant and end user is architecturally incompatible with Hermes's value proposition (validation, compliance, transformation, routing).

---

## Encryption Layers in Hermes

Instead of a single E2E approach, Hermes needs **layered encryption** at different boundaries:

### Layer 1: Transit Encryption (TLS 1.3)

| Connection | Protocol | Priority |
|-----------|----------|----------|
| Tenant -> Hermes REST API | TLS 1.3 (mandatory) | Phase 0 |
| Tenant -> Hermes SMPP Server | SMPPS (TLS wrapper on port 2776) | Phase 2 |
| Tenant -> Hermes SMTP Server | STARTTLS (mandatory) | Phase 3 |
| Hermes -> NATS JetStream | TLS (internal) | Phase 0 |
| Hermes -> PostgreSQL | TLS (internal) | Phase 0 |
| Hermes -> Redis | TLS (internal) | Phase 0 |
| Hermes -> Temporal Server | TLS (internal) | Phase 1 |
| Hermes -> Carrier SMPP | SMPPS (TLS wrapper) — **CRITICAL: SMPP 3.4 has no native encryption** | Phase 2 |
| Hermes -> KumoMTA | TLS (SMTP injection) | Phase 4 |
| Hermes -> Provider REST APIs | TLS 1.3 (SendGrid, Twilio, FCM, WhatsApp) | Phase 1 |

**SMPP TLS is non-negotiable.** SMPP 3.4 sends bind credentials and message content in plaintext. Without TLS wrapping, this is a GDPR violation for personal data in transit. The security audit scored SMPP security posture 3/10 partly for this reason.

### Layer 2: Encryption at Rest (AES-256-GCM)

| Data Store | What's Encrypted | Key Management |
|-----------|-----------------|----------------|
| PostgreSQL (tenant data) | Message content, recipient PII, API keys | Vault-managed keys, per-tenant encryption keys |
| Temporal (cascade state) | Message payloads in workflow history | Temporal's built-in payload codec with custom encryption |
| NATS JetStream (delivery streams) | Message bodies in transit streams | Encrypted before publish, decrypted by consumer |
| Redis (DLR correlation, rate limits) | Correlation IDs only (no content) | Not needed — no PII stored |
| KumoMTA (email queue) | Email bodies in queue | KumoMTA handles internally |
| Audit logs | Content hashes (not raw content) | Hash-chained, tamper-evident |

**Temporal Payload Codec:** Temporal supports custom payload codecs that encrypt/decrypt workflow data transparently. This means cascade state (which includes message content) is encrypted at rest in Temporal's persistence store without changing workflow code.

```go
// Temporal custom encryption codec
type EncryptionCodec struct {
    key []byte // From Vault, per-tenant
}

func (c *EncryptionCodec) Encode(payloads []*commonpb.Payload) ([]*commonpb.Payload, error) {
    // AES-256-GCM encrypt each payload
}

func (c *EncryptionCodec) Decode(payloads []*commonpb.Payload) ([]*commonpb.Payload, error) {
    // AES-256-GCM decrypt each payload
}
```

### Layer 3: Credential Encryption (HashiCorp Vault)

| Secret | Storage | Rotation |
|--------|---------|----------|
| Carrier SMPP credentials (system_id/password) | Vault KV v2 | Per carrier contract terms |
| DKIM private keys (per tenant domain) | Vault Transit or PKI engine | Annual or on-demand |
| HLR API tokens | Vault KV v2 | Quarterly |
| Tenant API keys (encryption key for bcrypt hashes) | Vault Transit | On-demand |
| TLS certificates (SMPP, SMTP, internal) | Vault PKI engine | Auto-renewal (cert-manager) |
| WASM adapter configs (provider API keys) | Vault KV v2, injected at runtime | Per provider terms |

### Layer 4: Signal Protocol — Future Channels

This is where Signal Protocol becomes directly relevant:

#### 4a. RCS E2E Encryption (Phase 2+)

Google Messages uses Signal Protocol for RCS E2E encryption. When Hermes adds RCS as a delivery channel:
- RCS Business Messaging (RBM) via Jibe API currently does **not** support E2E (business messages are server-encrypted only)
- However, Google is working on E2E for business RCS — when available, Hermes will need Signal Protocol integration
- This means: Hermes would negotiate a Signal Protocol session with the recipient's device via the RCS infrastructure
- The message content would be encrypted before leaving Hermes, and only decryptable by the recipient's device
- **This IS true E2E** — but it happens AFTER Hermes's validation pipeline processes the plaintext content

Flow:
```
Tenant -> Hermes (plaintext, validated, compliant) -> Signal Protocol encrypt -> RCS delivery -> Recipient device decrypts
```

Hermes still reads the content for validation. Signal Protocol encrypts at the delivery boundary, not the ingestion boundary.

#### 4b. In-App Messaging SDK (Future Product)

If Hermes offers a proprietary messaging channel (in-app chat SDK for tenants):
- Signal Protocol provides the gold standard E2E encryption
- Tenant's app integrates Hermes SDK -> establishes Signal Protocol session with recipient's app
- Messages are E2E encrypted — Hermes facilitates key exchange and message relay but cannot read content
- This channel would bypass content scanning (tenant accepts responsibility for content)
- **libsignal Rust core** can be compiled to WASM — potentially runs inside Wazero for sandboxed key operations

#### 4c. Tenant API Forward Secrecy (Evaluate at Scale)

Beyond TLS, a Signal-inspired ratcheting protocol for Hermes API sessions:
- Each API session establishes a Double Ratchet
- Per-request encryption keys with forward secrecy
- Compromised API key doesn't expose historical message content
- **Overkill for MVP** — TLS 1.3 already provides forward secrecy at the transport layer
- Evaluate only if tenants demand application-layer forward secrecy (regulated industries: healthcare, finance)

---

## Signal Protocol in Go: Implementation Options

| Option | Approach | Trade-offs |
|--------|----------|------------|
| **libsignal via CGo/FFI** | Call Rust libsignal from Go via C FFI | Best correctness (official implementation). CGo adds build complexity. |
| **libsignal compiled to WASM** | Run Rust libsignal in Wazero | Sandboxed, no CGo. Performance overhead. Experimental. |
| **Community Go ports** | e.g., signal-protocol-go | Not officially audited. Risk of subtle crypto bugs. Not recommended for production. |
| **Go crypto primitives** | Implement X3DH + Double Ratchet using Go's crypto/elliptic, crypto/aes, golang.org/x/crypto | Full control, no dependencies. High risk of implementation bugs. Not recommended unless team has cryptography expertise. |
| **Noise Protocol Framework** | Use Go's flynn/noise library | Noise is the foundation under Signal Protocol. Simpler API. Well-audited Go implementation. Good for tenant API sessions. |

**Recommendation:** For RCS E2E, use **libsignal via CGo** (official, audited). For tenant API forward secrecy experiments, use **Noise Protocol Framework** in Go (simpler, well-audited Go library).

---

## AGPLv3 License Implications

**Critical:** libsignal is licensed under **AGPLv3**, which is copyleft. This means:
- Any software that links to libsignal and is made available over a network (SaaS) must make its source code available
- This could require open-sourcing parts of Hermes that directly integrate libsignal
- **Mitigation options:**
  1. Isolate Signal Protocol in a separate microservice with a clean API boundary (most common approach)
  2. Use the Noise Protocol Framework (BSD-licensed) for non-RCS use cases
  3. Negotiate a commercial license with Signal Foundation (unlikely to be available)
  4. Implement the protocol from the published specification using Go's stdlib crypto (risky)

For the RCS channel, isolating the Signal Protocol integration in a dedicated service with a well-defined gRPC interface is the recommended approach. This service handles key exchange and message encryption only — the rest of Hermes doesn't link to AGPLv3 code.

---

## Encryption Priority Matrix

| Encryption Layer | Technology | Phase | Priority | Blocks Production? |
|-----------------|-----------|-------|----------|-------------------|
| TLS 1.3 on all external APIs | Go stdlib crypto/tls | Phase 0 | **P0** | Yes |
| SMPPS (TLS on carrier SMPP) | TLS wrapper on gosmpp | Phase 2 | **P0** | Yes (GDPR) |
| TLS on all internal services | NATS, PG, Redis, Temporal TLS | Phase 0-1 | **P0** | Yes |
| AES-256-GCM at rest (PG, NATS) | Go stdlib crypto/aes | Phase 5 | **P1** | No (but expected) |
| Temporal payload encryption | Custom codec with Vault keys | Phase 5 | **P1** | No |
| Vault for all secrets | HashiCorp Vault | Phase 5 | **P0** | Yes (security audit) |
| DKIM key management | Vault PKI/Transit | Phase 4 | **P1** | For email delivery |
| Signal Protocol for RCS E2E | libsignal (isolated service) | Phase 2+ | **P2** | No (future channel) |
| Signal Protocol for in-app SDK | libsignal or Noise Framework | Future | **P3** | No (future product) |
| Tenant API forward secrecy | Noise Protocol Framework | Evaluate | **P3** | No |

---

## Decision Points for Team

1. **Accept layered encryption approach?** (TLS transit + AES-256-GCM at rest + Vault secrets + Signal for future channels)
2. **Temporal payload encryption:** Enable custom encryption codec from Phase 1, or defer to Phase 5?
3. **AGPLv3 risk:** Accept isolated microservice approach for Signal Protocol, or avoid libsignal entirely?
4. **RCS E2E timeline:** When do we need Signal Protocol? Tied to RCS channel priority.
5. **Noise Protocol for API sessions:** Worth evaluating for regulated-industry tenants?

## Sources

- [Signal Protocol / libsignal](https://github.com/signalapp/libsignal) — AGPLv3
- [Signal Protocol Specification](https://signal.org/docs/) — X3DH, Double Ratchet
- [Noise Protocol Framework](https://noiseprotocol.org/) — Foundation under Signal Protocol
- [flynn/noise (Go)](https://github.com/flynn/noise) — Go Noise implementation
- [Temporal Payload Codec](https://docs.temporal.io/dataconversion#payload-codec) — Custom encryption for workflow data
- [Google RCS Business Messaging](https://developers.google.com/business-communications/rcs-business-messaging) — RCS API
- [SMPP 3.4 Security](https://smpp.org/) — No native encryption in protocol spec

---

*Generated: 2026-02-11 | For: Team Architecture Review*
