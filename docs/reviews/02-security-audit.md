# Security Architecture Audit: Project Hermes

**Overall Security Posture: 7.0/10 -- GOOD (with critical items before production)**
**Date:** 2026-02-10

---

## OWASP Top 10 Alignment

| Category | Severity | Key Issue |
|----------|----------|-----------|
| A01: Broken Access Control | HIGH | No explicit tenant-scoping middleware between tenants. Need mandatory tenant_id propagation + RLS. |
| A02: Cryptographic Failures | MEDIUM | No encryption at rest for message content in NATS/Supabase. Need TLS 1.3 everywhere + at-rest encryption. |
| A03: Injection | MEDIUM | Supabase PostgREST injection, ReDoS in content scanner, SMTP header injection via adapters. |
| A04: Insecure Design | LOW | Strong design patterns. Missing formal threat models (STRIDE). |
| A05: Security Misconfiguration | HIGH | K8s namespaces without NetworkPolicies, PSS, or RBAC. Enterprise tenant pods risk cluster escape. |
| A06: Vulnerable Components | MEDIUM | No dependency audit tooling specified. |
| A07: Auth Failures | HIGH | API key lifecycle gaps (no rotation, revocation, scoping). |
| A08: Data Integrity Failures | CRITICAL | Hot-loadable adapter containers verified by identity only, not image integrity. |
| A09: Logging/Monitoring | LOW | Good observability stack. Audit log gap if NATS publish fails at Layer 9. |
| A10: SSRF | MEDIUM | Content scanner URL checker could be SSRF vector to internal services. |

---

## Top 5 Priority Items

### 1. [CRITICAL] Shared-Schema Tenant Isolation
Schema-per-tenant in shared Supabase: single RLS misconfiguration exposes all starter tenants. Implement FORCE ROW LEVEL SECURITY, set search_path per connection, automated cross-tenant access tests in CI. Consider row-level (single-schema + RLS) instead of schema-per-tenant.

### 2. [CRITICAL] Adapter Container Supply Chain
Hot-loadable containers authenticated by Ed25519 identity but not image integrity. Compromised build pipeline = signed but malicious adapter. Implement Sigstore/Cosign image signing, read-only root filesystems, egress NetworkPolicies per adapter type, gVisor/Kata for enterprise custom adapters.

### 3. [HIGH] NATS JetStream Authentication
Unauthenticated NATS = any compromised service accesses any tenant's messages. Enable NKey auth, per-service accounts with subject-level authorization, TLS for all client connections.

### 4. [HIGH] API Key Lifecycle
No rotation, revocation, scoping, or multi-key support. Leaked key = unlimited blast radius, indefinite validity. Implement: rotation with grace periods, immediate revocation, scope-based permissions, key exposure detection.

### 5. [HIGH] Kubernetes Network Segmentation
Multiple namespaces but no NetworkPolicies, PSS, or RBAC. Enterprise tenant namespaces running custom containers without these = escape risk. Default deny all, whitelist per service, restricted PSS, block cloud metadata endpoint.

---

## Multi-Tenancy Risks

- **SEC-MT-01 (CRITICAL):** Shared schema cross-tenant data leakage via search_path, shared extensions, shared connection pools
- **SEC-MT-02 (HIGH):** NATS tenant message isolation - cross-tenant consumption, subject enumeration, noisy neighbor
- **SEC-MT-03 (HIGH):** Enterprise namespace escape - K8s API access, cross-namespace traffic, metadata endpoint

## Supply Chain Risks

- **SEC-SC-01 (CRITICAL):** Malicious adapter container image (exfiltration, lateral attack, crypto mining)
- **SEC-SC-02 (HIGH):** Adapter hot-reload attack window (valid token during compromise detection)
- **SEC-SC-03 (MEDIUM):** Dependency supply chain (typosquatting, dependency confusion)

## Compliance Risks

- **SEC-COMP-01 (HIGH):** GDPR consent chain - need DPA per tenant, data subject deletion API, data residency controls
- **SEC-COMP-02 (HIGH):** TCPA express written consent - $500-$1,500 per message statutory damages for non-compliance
- **SEC-COMP-03 (MEDIUM):** ePrivacy Directive - unsubscribe headers, suppression lists, do-not-call registries

## DDoS/Abuse Vectors

- **SEC-DDOS-01 (HIGH):** API Gateway volumetric attack pre-authentication. Need CDN/DDoS protection layer.
- **SEC-DDOS-02 (MEDIUM):** Webhook flood from providers. Separate webhook ingress from API ingress.
- **SEC-DDOS-03 (HIGH):** Tenant-as-attacker abusing shared IP pool. Progressive penalties + tenant health scoring.

## Go-Specific Security Notes

- Go's memory safety via garbage collection eliminates buffer overflow class
- No unsafe blocks to audit (unlike Rust)
- Standard library crypto is well-audited
- goroutine leaks can cause memory exhaustion — monitor goroutine counts
- Race detector (`go test -race`) should run in CI
