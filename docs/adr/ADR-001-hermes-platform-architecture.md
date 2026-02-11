# Hermes Platform Architecture [HRM]

The Hermes Platform is a multi-tenant, multi-channel message delivery broker built in Go. It provides unified message delivery across Email, SMS, WhatsApp, Telegram, and Push channels with intelligent cascade delivery (automatic fallback across channels), tenant-tiered isolation, hot-loadable channel adapters, and an IP reputation engine. Hermes accepts inbound messages via a Protocol Gateway supporting REST, SMPP, and SMTP protocols, and delivers via carrier-direct infrastructure (SMPP connections to telecom carriers, own MTAs for email, WhatsApp BSP) with aggregator APIs as fallback. A Smart Routing Engine performs HLR/MNP lookups and least-cost routing for optimal per-message delivery paths. The platform uses Go's standard library `net/http` with a lightweight Chi router, using hexagonal architecture with ports and adapters (Go interfaces) for full infrastructure swappability.

## Status

**PROPOSED** - Pending architecture review and team sign-off

## Key Characteristics

- **Go with Lightweight Router**: Built on Go's `net/http` standard library with [Chi](https://github.com/go-chi/chi) for routing and middleware composition. Pragmatic choice -- Chi adds minimal overhead while providing composable middleware, subrouters, and context-based routing without framework lock-in.
- **Hexagonal Architecture**: Ports & Adapters pattern. Domain core (cascade engine, tenant registry, channel routing) is framework-agnostic. Infrastructure adapters are swappable. Go interfaces are implicitly satisfied, making hexagonal architecture natural -- no explicit `implements` declarations needed.
- **Multi-Tenant with Tiered Isolation**: Starter (shared schema), Business (dedicated project), Enterprise (self-hosted/BYOIP).
- **Cascade Delivery Engine (Core IP)**: State machine per message with configurable channel fallback chains, timeouts, and retry policies.
- **Hot-Loadable Channel Adapters**: Containerized adapters register via Ed25519 signatures, send heartbeats, and can be hot-swapped without platform restart.
- **9-Layer Message Validation Pipeline**: Idempotency, authentication, rate limiting, balance checks, schema validation, compliance, content scanning, and audit logging.
- **IP Reputation Engine**: Pool management, automated warming, real-time monitoring, tenant health scoring, ISP feedback loop processing.
- **NATS JetStream**: Tenant-keyed message streams with 10M+ msg/sec capability and built-in persistence.
- **Protocol Gateway**: Accepts REST, SMPP, and SMTP inbound. Normalizes all protocols to a unified internal message format before hitting the validation pipeline.
- **Smart Routing Engine**: HLR/MNP lookup, least-cost routing, carrier capacity management, quality scoring per route per carrier.
- **Carrier-Direct Delivery**: Direct SMPP connections to telecom carriers, own MTAs for email, WhatsApp BSP. Aggregator APIs (SendGrid, Twilio, etc.) as fallback only.
- **Scale**: Designed for 2B texts + 6B emails per year (~22M messages/day average, ~100M/day peak).

## Problem Statement

### Industry Gap

Businesses that need to reach users across multiple communication channels face a fragmented landscape:

```
+---------------------------------------------------------------------------+
|                        CURRENT STATE (FRAGMENTED)                         |
+---------------------------------------------------------------------------+
|                                                                           |
|  +------------------+  +------------------+  +------------------+         |
|  |  SendGrid API    |  |  Twilio API      |  |  FCM API         |        |
|  |  (Email only)    |  |  (SMS only)      |  |  (Push only)     |        |
|  +--------+---------+  +--------+---------+  +--------+---------+        |
|           |                      |                      |                 |
|           v                      v                      v                 |
|  +------------------+  +------------------+  +------------------+         |
|  |  App Integration |  |  App Integration |  |  App Integration |        |
|  |  Error handling  |  |  Error handling  |  |  Error handling  |        |
|  |  Retry logic     |  |  Retry logic     |  |  Retry logic     |        |
|  |  Rate limiting   |  |  Rate limiting   |  |  Rate limiting   |        |
|  +------------------+  +------------------+  +------------------+         |
|                                                                           |
|  Problems:                                                                |
|  - No cross-channel fallback (SMS fails, message is lost)                |
|  - Each channel requires separate integration, billing, monitoring       |
|  - No unified delivery confirmation across channels                      |
|  - IP reputation management is per-provider, reactive not proactive      |
|  - Compliance (GDPR, TCPA) must be implemented per integration           |
|                                                                           |
+---------------------------------------------------------------------------+
```

### Impact

| Issue | Severity | Business Impact |
|-------|----------|-----------------|
| No cascade delivery | CRITICAL | Messages lost when primary channel fails |
| Per-channel integration | HIGH | Weeks of engineering per new channel |
| No unified observability | HIGH | Blind spots across delivery channels |
| Fragmented compliance | HIGH | GDPR/TCPA violations risk per integration |
| No IP reputation management | MEDIUM | Deliverability degrades over time |
| No tenant isolation | MEDIUM | Bad actor impacts all customers |

### Hermes Solution

```
+---------------------------------------------------------------------------+
|                        TARGET STATE (HERMES)                              |
+---------------------------------------------------------------------------+
|                                                                           |
|  +------------------------------------------------------------------+    |
|  |               Protocol Gateway (Inbound Normalization)            |   |
|  |                                                                    |   |
|  |  +----------+   +----------+   +----------+                       |   |
|  |  | REST API |   |  SMPP    |   |  SMTP    |                       |   |
|  |  |  (Chi)   |   |  Server  |   |  Server  |                       |   |
|  |  | (modern  |   | (gosmpp) |   | (go-smtp |                       |   |
|  |  | clients) |   |(enterprise|   |  /Haraka)|                       |   |
|  |  +----+-----+   +----+-----+   +----+-----+                       |   |
|  |       |              |              |                              |   |
|  |       +-------+------+------+-------+                              |   |
|  |               |                                                    |   |
|  |               v                                                    |   |
|  |       +-------+--------+                                           |   |
|  |       | Unified Message |                                          |   |
|  |       |    Format       |                                          |   |
|  |       +-------+--------+                                           |   |
|  +------------------------------------------------------------------+    |
|           |                                                               |
|           v                                                               |
|  +------------------------------------------------------------------+    |
|  |              9-Layer Message Validation Pipeline                   |   |
|  |  Idempotency -> Auth -> Tenant -> Rate -> Balance -> Schema ->   |   |
|  |  Compliance -> Content Scan -> Audit -> Route to Cascade Engine  |   |
|  +------------------------------------------------------------------+    |
|           |                                                               |
|           v                                                               |
|  +------------------------------------------------------------------+    |
|  |                   Cascade Engine (Core IP)                        |   |
|  |  Email -> [wait 30s] -> SMS -> [wait 60s] -> Push -> [done]     |   |
|  |  State machine per message with tenant-configured rules          |   |
|  +------------------------------------------------------------------+    |
|           |                                                               |
|           v                                                               |
|  +------------------------------------------------------------------+    |
|  |                Smart Routing Engine                                |   |
|  |  HLR/MNP Lookup -> Least-Cost Routing -> Quality Scoring ->     |   |
|  |  Carrier Capacity Check -> Route Selection                        |   |
|  +------------------------------------------------------------------+    |
|           |              |              |              |                   |
|           v              v              v              v                   |
|  +------------+  +------------+  +------------+  +------------+           |
|  | Direct SMPP|  |  Own MTA   |  | WhatsApp   |  |    FCM     |          |
|  |  Carriers  |  | (Email)    |  |   BSP      |  |  Adapter   |          |
|  | (Telia,TDC)|  | (Haraka)   |  | (Meta API) |  | (container)|          |
|  +-----+------+  +-----+------+  +-----+------+  +-----+------+         |
|        |               |               |               |                  |
|   [FALLBACK]      [FALLBACK]      [FALLBACK]           |                  |
|  +-----+------+  +-----+------+                        |                  |
|  | Aggregator |  |  SendGrid  |                        |                  |
|  | APIs       |  |  / SES     |                        |                  |
|  | (Messente, |  | (burst     |                        |                  |
|  |  Infobip)  |  |  capacity) |                        |                  |
|  +------------+  +------------+                        |                  |
|                                                                           |
+---------------------------------------------------------------------------+
```

## Decision

### 0. Why Go over Rust

The original design considered Rust. After evaluating both languages against project constraints, Go was chosen as the implementation language.

**Rationale:**

| Factor | Go | Rust | Decision Driver |
|--------|-----|------|-----------------|
| **Time to PoC** | 1-2 months | 3-4 months | Go's simpler type system and faster iteration cycles cut PoC time in half. Critical for validating the cascade engine with real customers. |
| **Hiring (Denmark)** | Larger talent pool | Smaller talent pool | Go is widely adopted in Danish tech companies (Maersk, Trustpilot, Lunar). Finding Go developers with backend/infrastructure experience is significantly easier. |
| **Performance** | More than sufficient for 100M/day | Higher theoretical ceiling | At ~1,157 msg/sec sustained, Go's goroutine scheduler and garbage collector are well within performance requirements. The bottleneck is I/O to external providers, not CPU. |
| **Compile times** | 2-5 seconds (full rebuild) | 2-10 minutes (full rebuild) | Fast compile times directly impact developer productivity and CI pipeline speed. |
| **Ecosystem** | Mature messaging/infra libraries | Mature but smaller ecosystem | Go has first-class gRPC support, excellent NATS client, and battle-tested HTTP tooling. |
| **Concurrency model** | Goroutines + channels (native) | tokio async/await | Goroutines map naturally to concurrent cascade state machines. Each message cascade runs as a goroutine with channels for coordination. No colored function problem. |
| **Future optimization** | Can rewrite hot paths in Rust via CGo or microservices | N/A | Hexagonal architecture makes this possible -- extract a port's adapter as a Rust microservice if profiling shows a bottleneck. |

**Trade-off acknowledged:** Go lacks Rust's compile-time memory safety guarantees. Mitigated by:
- Go's race detector (`-race` flag) enabled in all CI test runs
- Goroutine leak detection in integration tests (via `goleak`)
- Strict linting with `golangci-lint` (including `errcheck`, `staticcheck`, `gosec`)
- Structured error handling patterns (no ignored errors)

### 1. Go Standard Library with Chi Router

Build the HTTP layer on Go's `net/http` standard library with [Chi](https://github.com/go-chi/chi) for routing and middleware. Not a full framework (no Gin, no Fiber, no Echo).

**Rationale:**
- Chi is a lightweight, idiomatic router built on `net/http` -- compatible with any standard middleware
- Full control over backpressure propagation from downstream channel adapters
- Custom connection pooling tuned per tenant tier via `http.Transport` configuration
- Request pipeline middleware via Chi's composable middleware chain (auth, rate limit, compression, tracing)
- Zero framework lock-in; Chi middleware are standard `net/http` handlers
- Message broker workload is I/O-bound with minimal routing complexity -- a full web framework provides no value

**Trade-off:** Slightly more glue code than a batteries-included framework, in exchange for zero framework lock-in and precise resource control. Chi minimizes this cost compared to raw `net/http` routing.

### 2. Hexagonal Architecture (Ports & Adapters)

The domain core is isolated behind port interfaces. Infrastructure concerns (database, queue, external APIs) are injected as adapters. Go interfaces are implicitly satisfied (structural typing), which makes hexagonal architecture natural -- any type that implements the methods of an interface automatically satisfies it, with no explicit `implements` declaration. Dynamic dispatch via interfaces is the default in Go.

```
+-------------------------------------------------------------------+
|                        HEXAGONAL LAYOUT                            |
+-------------------------------------------------------------------+
|                                                                    |
|              +----------------------------------+                  |
|              |         DOMAIN CORE              |                  |
|              |                                  |                  |
|              |  CascadeEngine                   |                  |
|              |  TenantRegistry                  |                  |
|              |  ChannelRouter                   |                  |
|              |  SmartRouter                     |                  |
|              |  ReputationScorer                |                  |
|              |  MessageValidator                |                  |
|              |                                  |                  |
|              +--+---+---+---+---+---+---+---+--+                  |
|                 |   |   |   |   |   |   |   |                     |
|  INBOUND PORTS  |   |   |   |   |   |   |   |  OUTBOUND PORTS    |
|  (driving)      |   |   |   |   |   |   |   |  (driven)          |
|                 v   v   v   v   v   v   v   v                     |
|  +-----------+                          +-----------+             |
|  | HTTP API  |                          | Supabase  |             |
|  | (net/http)|                          | (pgx)     |             |
|  +-----------+                          +-----------+             |
|  +-----------+                          +-----------+             |
|  | SMPP Srv  |                          | NATS      |             |
|  | (gosmpp)  |                          | JetStream |             |
|  +-----------+                          +-----------+             |
|  +-----------+                          +-----------+             |
|  | SMTP Srv  |                          | SMPP Cli  |             |
|  | (go-smtp) |                          | (carriers)|             |
|  +-----------+                          +-----------+             |
|  +-----------+                          +-----------+             |
|  | gRPC      |                          | MTA       |             |
|  | (grpc-go) |                          | (email)   |             |
|  +-----------+                          +-----------+             |
|  +-----------+                          +-----------+             |
|  | NATS Sub  |                          | HLR/MNP   |             |
|  | (consumer)|                          | Provider  |             |
|  +-----------+                          +-----------+             |
|                                         +-----------+             |
|                                         | Channel   |             |
|                                         | Adapters  |             |
|                                         +-----------+             |
|                                                                    |
+-------------------------------------------------------------------+
```

**Key Port Interfaces (Go):**

```go
// TenantRepository defines the port for tenant persistence.
type TenantRepository interface {
    FindByID(ctx context.Context, id TenantID) (*Tenant, error)
    FindByAPIKey(ctx context.Context, key APIKey) (*Tenant, error)
    Upsert(ctx context.Context, tenant *Tenant) (*Tenant, error)
}

// MessageQueue defines the port for async message publishing and consumption.
type MessageQueue interface {
    Publish(ctx context.Context, stream string, msg *CascadeMessage) error
    Subscribe(ctx context.Context, stream string, consumer string) (<-chan *Message, error)
}

// ChannelAdapter defines the port for channel delivery.
type ChannelAdapter interface {
    ChannelType() ChannelType
    Send(ctx context.Context, msg *ChannelMessage) (*DeliveryReceipt, error)
    CheckStatus(ctx context.Context, receipt *DeliveryReceipt) (*DeliveryStatus, error)
}

// SMPPServer defines the port for inbound SMPP connections from enterprise clients.
type SMPPServer interface {
    Start(ctx context.Context, addr string) error
    Stop(ctx context.Context) error
    OnSubmitSM(handler func(ctx context.Context, msg *SMPPMessage) error)
}

// SMPPClient defines the port for outbound SMPP connections to carriers.
type SMPPClient interface {
    Connect(ctx context.Context, carrier CarrierConfig) error
    SubmitSM(ctx context.Context, msg *SMPPMessage) (*SMPPResponse, error)
    Close(ctx context.Context) error
}

// SmartRouter defines the port for intelligent message routing.
type SmartRouter interface {
    Route(ctx context.Context, msg *RoutableMessage) (*RoutingDecision, error)
    UpdateCarrierHealth(ctx context.Context, carrier CarrierID, metrics *CarrierMetrics) error
}

// HLRProvider defines the port for number portability lookups.
type HLRProvider interface {
    Lookup(ctx context.Context, phoneNumber string) (*HLRResult, error)
}

// MTAService defines the port for email delivery via own MTAs.
type MTAService interface {
    Send(ctx context.Context, msg *EmailMessage) (*DeliveryReceipt, error)
    GetPoolHealth(ctx context.Context) (*IPPoolHealth, error)
}
```

Note: Any struct that has matching methods automatically satisfies these interfaces -- no explicit declaration required. This makes it trivial to swap adapters (e.g., replace a Supabase-backed `TenantRepository` with an in-memory implementation for testing).

### 3. Multi-Tenant with Tiered Isolation

Three tiers with escalating isolation guarantees:

| Aspect | Starter | Business | Enterprise |
|--------|---------|----------|------------|
| **Database** | Supabase Cloud shared project, schema-per-tenant | Dedicated Supabase project per tenant | On-premise / self-hosted Supabase |
| **Workers** | Shared cascade worker pool | Dedicated worker pool per tenant | Dedicated cluster |
| **IP Pool** | Shared IP pool (reputation-managed) | Dedicated IP range | BYOIP (Bring Your Own IP) |
| **Adapters** | Platform-managed adapters | Platform-managed + custom config | Bring Your Own Adapter containers |
| **SLA** | Best-effort, 99.5% | 99.9% uptime, priority support | 99.99% uptime, dedicated SRE |
| **Compliance** | Standard (GDPR) | Standard + HIPAA BAA available | Full compliance suite, on-prem audit |
| **Provisioning** | Instant (schema migration) | Minutes (project creation) | Hours (cluster deployment) |

**Schema-per-tenant isolation (Starter tier):**

```sql
-- Each tenant gets a dedicated schema within the shared Supabase project
CREATE SCHEMA IF NOT EXISTS tenant_{tenant_id};

-- Row-level security as defense-in-depth
ALTER TABLE tenant_{tenant_id}.messages ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON tenant_{tenant_id}.messages
    USING (tenant_id = current_setting('hermes.current_tenant')::uuid);
```

### 4. Channel Adapters as Hot-Loadable Containers

Each channel adapter (Email/SendGrid, SMS/Twilio, WhatsApp Business API, Telegram Bot API, Push/FCM) is a container that registers with the Channel Adapter Registry via Ed25519 signatures, sends heartbeats, and can be hot-swapped without platform restart. Enterprise tenants can bring their own adapter containers.

**Pattern borrowed from OmniRev Plugin Registry (ADR-PR-001).**

**Adapter lifecycle:**
1. Adapter container starts and generates an Ed25519 keypair (or loads from HSM)
2. Public key is pre-registered in the trusted keys table by platform admin
3. Container sends registration request signed with Ed25519 private key
4. Registry verifies signature, validates allowlist, generates bcrypt-hashed instance token
5. Container receives instance token and begins 30-second heartbeat cycle
6. Registry marks adapter inactive after 3 missed heartbeats (90s timeout)
7. Hot-swap: deploy new container version, old container receives `shouldShutdown: true` via heartbeat

**Adapter contract (gRPC via grpc-go):**

```protobuf
service ChannelAdapterService {
    rpc Send (SendRequest) returns (SendResponse);
    rpc CheckStatus (StatusRequest) returns (StatusResponse);
    rpc GetCapabilities (Empty) returns (CapabilitiesResponse);
}

message SendRequest {
    string message_id = 1;
    string tenant_id = 2;
    string channel_type = 3;       // EMAIL, SMS, WHATSAPP, TELEGRAM, PUSH
    string recipient = 4;          // Email address, phone number, device token
    string subject = 5;            // Email subject, empty for other channels
    string body = 6;               // Message body (text or template ID)
    string content_type = 7;       // text/plain, text/html, template
    map<string, string> metadata = 8;
    ChannelConfig channel_config = 9;
}

message SendResponse {
    bool accepted = 1;
    string provider_message_id = 2;
    DeliveryStatus initial_status = 3;
    string error_code = 4;
    string error_message = 5;
}
```

### 5. Cascade Engine (Core IP)

The Cascade Engine is the primary differentiator of Hermes. It implements a state machine per message that attempts delivery across multiple channels in a tenant-configured order, with configurable timeouts and retry policies at each step. Each cascade runs as a goroutine, with channels used for coordination between the cascade engine, timers, and delivery confirmation listeners.

**State machine:**

```
                    +-------------+
                    |   CREATED   |
                    +------+------+
                           |
                           v
                    +------+------+
              +---->| ATTEMPTING  |<----+
              |     |  Channel N  |     |
              |     +------+------+     |
              |            |            |
              |     +------+------+     |
              |     |   WAITING   |     |
              |     | Confirmation|     |
              |     +------+------+     |
              |       |          |      |
              |  CONFIRMED   TIMEOUT    |
              |       |          |      |
              |       v          v      |
              |  +----+----+ +--+---+  |
              |  |DELIVERED | |FAILED|  |
              |  +---------+ |Chan N|--+  (next channel in cascade)
              |              +------+
              |                 |
              |            NO MORE CHANNELS
              |                 |
              |                 v
              |           +-----+-----+
              +---------->| EXHAUSTED  |
                          +-----------+
```

**Cascade rule configuration (per tenant):**

```go
// CascadeRule defines a tenant's cascade delivery strategy.
type CascadeRule struct {
    RuleID              CascadeRuleID `json:"rule_id"`
    TenantID            TenantID      `json:"tenant_id"`
    Name                string        `json:"name"` // e.g., "Transactional - High Priority"
    Steps               []CascadeStep `json:"steps"`
    MaxCascadeDuration  time.Duration `json:"max_cascade_duration"` // default: 24h
    ConfirmationLevel   ConfirmationLevel `json:"confirmation_level"`
}

// CascadeStep defines a single attempt within a cascade.
type CascadeStep struct {
    Channel                  ChannelType     `json:"channel"`
    Timeout                  time.Duration   `json:"timeout"`
    MaxRetries               uint8           `json:"max_retries"`
    RetryBackoff             BackoffStrategy `json:"retry_backoff"`
    RequireChannelRegistered bool            `json:"require_channel_registered"`
    DeliveryWindow           *DeliveryWindow `json:"delivery_window,omitempty"`
}

// ChannelType represents a supported delivery channel.
type ChannelType int

const (
    ChannelEmail ChannelType = iota
    ChannelSMS
    ChannelWhatsApp
    ChannelTelegram
    ChannelPush
)

// ConfirmationLevel defines what constitutes "delivered".
type ConfirmationLevel int

const (
    // ConfirmationSent means the message was accepted by the provider.
    ConfirmationSent ConfirmationLevel = iota
    // ConfirmationDelivered means the message was delivered to the device/inbox.
    ConfirmationDelivered
    // ConfirmationRead means the message was read/opened by the recipient.
    ConfirmationRead
)

// BackoffStrategy defines retry backoff behavior.
type BackoffStrategy struct {
    Type      BackoffType   `json:"type"` // "fixed", "exponential", "linear"
    Base      time.Duration `json:"base"`
    Max       time.Duration `json:"max"`
    Increment time.Duration `json:"increment,omitempty"` // linear only
}

type BackoffType string

const (
    BackoffFixed       BackoffType = "fixed"
    BackoffExponential BackoffType = "exponential"
    BackoffLinear      BackoffType = "linear"
)
```

### 6. Tenant Registry

The Tenant Registry manages the lifecycle of tenants on the platform, from onboarding through provisioning to day-to-day operations. It handles API key management, tier upgrades, and tenant health monitoring.

**Pattern borrowed from OmniRev Service Registry (ADR-SR-001).**

**Tenant onboarding flow:**
1. Developer signs up via WorkOS SSO (magic link or Google/GitHub OAuth)
2. Tenant provisioner creates schema (Starter), project (Business), or cluster (Enterprise)
3. API keys generated (primary + secondary for rotation) with Ed25519 signing
4. Tenant configuration seeded with default cascade rules
5. Welcome message sent via Hermes itself (dogfooding)

**Tenant model:**

```go
// Tenant represents a platform tenant with all configuration.
type Tenant struct {
    ID           TenantID        `json:"id"`
    Name         string          `json:"name"`
    Tier         TenantTier      `json:"tier"`
    Status       TenantStatus    `json:"status"`
    APIKeys      []APIKeyRecord  `json:"api_keys"`
    CascadeRules []CascadeRule   `json:"cascade_rules"`
    RateLimits   RateLimitConfig `json:"rate_limits"`
    Billing      BillingConfig   `json:"billing"`
    Compliance   ComplianceConfig `json:"compliance"`
    CreatedAt    time.Time       `json:"created_at"`
    UpdatedAt    time.Time       `json:"updated_at"`
}

// TenantTier represents the isolation level for a tenant.
type TenantTier int

const (
    TierStarter TenantTier = iota
    TierBusiness
    TierEnterprise
)

// TenantStatus represents the current state of a tenant.
type TenantStatus int

const (
    TenantProvisioning TenantStatus = iota
    TenantActive
    TenantSuspended   // Non-payment or abuse
    TenantDeactivated // Voluntary
)

// APIKeyRecord stores a hashed API key with metadata.
type APIKeyRecord struct {
    KeyID             string       `json:"key_id"`
    KeyHash           string       `json:"key_hash"` // bcrypt hash, never store plaintext
    Label             string       `json:"label"`
    Permissions       []Permission `json:"permissions"`
    RateLimitOverride *RateLimitConfig `json:"rate_limit_override,omitempty"`
    ExpiresAt         *time.Time   `json:"expires_at,omitempty"`
    CreatedAt         time.Time    `json:"created_at"`
    LastUsedAt        *time.Time   `json:"last_used_at,omitempty"`
}
```

### 7. Message Validation Pipeline (9 Layers)

Every inbound message passes through a 9-layer validation pipeline before reaching the Cascade Engine. Failure at any layer stops processing and returns an appropriate error. This pipeline is inspired by the OmniRev Debug System Validation Pipeline (ADR-SR-003).

```
+-----------------------------------------------------------------------+
|                  9-LAYER MESSAGE VALIDATION PIPELINE                    |
+-----------------------------------------------------------------------+
|                                                                        |
|  Layer 0: IDEMPOTENCY CHECK                                           |
|  +------------------------------------------------------------------+ |
|  | Check message_id against idempotency store (Redis/NATS KV)       | |
|  | IF duplicate: RETURN cached response (no re-processing)          | |
|  | ELSE: Store message_id with TTL=24h, continue                    | |
|  +------------------------------------------------------------------+ |
|                              |                                         |
|  Layer 1: TENANT AUTHENTICATION                                       |
|  +------------------------------------------------------------------+ |
|  | Extract API key from Authorization header (Bearer or X-API-Key)  | |
|  | Validate key against bcrypt hash in tenant registry               | |
|  | Extract tenant_id, permissions, rate_limit_override               | |
|  | FAIL: 401 Unauthorized                                            | |
|  +------------------------------------------------------------------+ |
|                              |                                         |
|  Layer 2: TENANT ACTIVE CHECK                                         |
|  +------------------------------------------------------------------+ |
|  | Verify tenant.status == Active                                    | |
|  | Provisioning: 503 "Tenant still provisioning"                     | |
|  | Suspended: 403 "Account suspended" + reason                      | |
|  | Deactivated: 410 "Account deactivated"                            | |
|  +------------------------------------------------------------------+ |
|                              |                                         |
|  Layer 3: RATE LIMITING                                               |
|  +------------------------------------------------------------------+ |
|  | Sliding window rate limit per tenant (configurable per tier)      | |
|  | Starter: 100 msg/sec, Business: 1,000 msg/sec, Enterprise: custom| |
|  | Uses token bucket with NATS KV for distributed state              | |
|  | FAIL: 429 Too Many Requests + Retry-After header                  | |
|  +------------------------------------------------------------------+ |
|                              |                                         |
|  Layer 4: BALANCE / QUOTA CHECK (CRITICAL FOR BILLING)                |
|  +------------------------------------------------------------------+ |
|  | Check tenant has sufficient message credits or active billing     | |
|  | Prepaid: Decrement balance atomically (reserve, not deduct)       | |
|  | Postpaid: Check against credit limit                              | |
|  | FAIL: 402 Payment Required                                        | |
|  +------------------------------------------------------------------+ |
|                              |                                         |
|  Layer 5: SCHEMA VALIDATION                                          |
|  +------------------------------------------------------------------+ |
|  | Validate request body against JSON schema                         | |
|  | Required fields: to, channel_preferences or cascade_rule_id      | |
|  | Validate recipient format per channel (email regex, E.164 phone)  | |
|  | Validate body content (max size, encoding)                        | |
|  | FAIL: 422 Unprocessable Entity + field-level errors               | |
|  +------------------------------------------------------------------+ |
|                              |                                         |
|  Layer 6: COMPLIANCE CHECK                                            |
|  +------------------------------------------------------------------+ |
|  | GDPR: Check consent records for recipient + channel               | |
|  | TCPA: Verify opt-in for SMS in US jurisdictions                   | |
|  | Channel-specific: WhatsApp template approval, SMS 10DLC           | |
|  | Time restrictions: Quiet hours per jurisdiction                    | |
|  | FAIL: 451 Unavailable For Legal Reasons + compliance_code         | |
|  +------------------------------------------------------------------+ |
|                              |                                         |
|  Layer 7: PRE-FLIGHT CONTENT SCAN                                     |
|  +------------------------------------------------------------------+ |
|  | Anti-spam scoring (heuristic + ML model)                          | |
|  | Phishing URL detection                                            | |
|  | Prohibited content patterns (per tenant compliance config)        | |
|  | Reputation impact prediction                                      | |
|  | FAIL: 422 + content_violation details                             | |
|  +------------------------------------------------------------------+ |
|                              |                                         |
|  Layer 8: AUDIT LOG (PRE-EXECUTION)                                   |
|  +------------------------------------------------------------------+ |
|  | Record message attempt BEFORE execution:                          | |
|  |   message_id, tenant_id, recipient_hash, channels, timestamp     | |
|  | Best-effort: Pipeline continues even if audit write fails         | |
|  | Audit entries are append-only, hash-chained for tamper evidence   | |
|  +------------------------------------------------------------------+ |
|                              |                                         |
|  Layer 9: ROUTE TO CASCADE ENGINE                                     |
|  +------------------------------------------------------------------+ |
|  | Resolve cascade_rule_id or build rule from channel_preferences    | |
|  | Publish CascadeMessage to NATS JetStream (tenant-keyed stream)   | |
|  | RETURN 202 Accepted + message_id + tracking_url                   | |
|  +------------------------------------------------------------------+ |
|                                                                        |
+-----------------------------------------------------------------------+
```

### 8. IP Reputation Engine

The IP Reputation Engine is a first-class service that protects the platform's sending infrastructure and ensures high deliverability across all tenants.

**Responsibilities:**

| Component | Function |
|-----------|----------|
| **IP Pool Manager** | Assign IPs to tenant tiers. Starter: shared pool. Business: dedicated range. Enterprise: BYOIP. |
| **IP Warming Scheduler** | Automated ramp-up for new IPs (start at 50 msg/day, double daily to target volume). |
| **Reputation Monitor** | Real-time tracking: bounce rate, complaint rate, blacklist status per IP. |
| **Tenant Health Scorer** | Score tenants on sending behavior. Bad actors get throttled before they damage shared infrastructure. |
| **Content Analyzer** | Pre-flight content analysis for spam indicators, phishing URLs, reputation impact. |
| **ISP Feedback Processor** | Process feedback loops: Gmail Postmaster Tools, Microsoft SNDS, Yahoo CFL/FBL. |
| **Channel Protection** | Per-channel reputation management: Email (DKIM/SPF/DMARC), SMS (10DLC/toll-free verification), WhatsApp (quality rating monitoring). |

**Tenant health scoring algorithm:**

```go
// TenantHealthScore tracks a tenant's sending reputation.
type TenantHealthScore struct {
    TenantID            TenantID  `json:"tenant_id"`
    OverallScore        float64   `json:"overall_score"`         // 0.0 (toxic) to 100.0 (pristine)
    BounceRate          float64   `json:"bounce_rate"`           // < 2% good, > 5% critical
    ComplaintRate       float64   `json:"complaint_rate"`        // < 0.1% good, > 0.3% critical
    SpamTrapHits        uint32    `json:"spam_trap_hits"`        // 0 is target
    BlacklistAppearances uint32   `json:"blacklist_appearances"` // 0 is target
    ContentQuality      float64   `json:"content_quality"`       // ML-scored 0-100
    SendingPatternScore float64   `json:"sending_pattern_score"` // Consistency and volume patterns
    CalculatedAt        time.Time `json:"calculated_at"`
}

const (
    // ThrottleThreshold: tenants below this score get throttled to protect shared infra.
    ThrottleThreshold = 40.0
    // SuspendThreshold: tenants below this are suspended pending review.
    SuspendThreshold = 20.0
)
```

### 9. NATS JetStream for Message Queue

NATS JetStream provides the backbone for all asynchronous message processing. Tenant-keyed streams ensure isolation and enable independent scaling.

**Stream topology:**

```
NATS JetStream Streams:
+-----------------------------------------------------------------------+
|                                                                        |
|  hermes.messages.{tenant_id}         -- Inbound validated messages     |
|  hermes.cascade.{tenant_id}          -- Cascade state transitions      |
|  hermes.delivery.{channel_type}      -- Channel delivery queue         |
|  hermes.webhooks.{tenant_id}         -- Delivery status webhooks       |
|  hermes.dlq.{tenant_id}             -- Dead letter queue               |
|  hermes.audit.{tenant_id}           -- Audit event stream              |
|  hermes.reputation                   -- Reputation events (shared)     |
|  hermes.billing                      -- Usage metering events          |
|  hermes.smpp.{carrier_id}           -- SMPP message queue per carrier  |
|  hermes.routing.decisions            -- Routing decision audit trail   |
|  hermes.mta.{instance_id}           -- MTA delivery queue per instance |
|  hermes.hlr.lookups                  -- HLR lookup results cache       |
|                                                                        |
+-----------------------------------------------------------------------+
```

**Why NATS JetStream over Kafka/RabbitMQ:**

| Criteria | NATS JetStream | Kafka | RabbitMQ |
|----------|---------------|-------|----------|
| Operational complexity | Low (single binary) | High (ZooKeeper/KRaft) | Medium |
| Go client maturity | nats.go (excellent, first-party) | confluent-kafka-go (C wrapper) | amqp091-go (good) |
| Tenant-keyed streams | Native subject filtering | Topic partitioning | Exchange routing |
| Latency | Sub-millisecond | Low milliseconds | Low milliseconds |
| Throughput | 10M+ msg/sec | 10M+ msg/sec | 100K msg/sec |
| Built-in KV store | Yes (rate limit state) | No (need Redis) | No (need Redis) |
| Memory footprint | ~50MB | ~1GB+ | ~200MB |

### 10. Protocol Gateway Layer

Hermes accepts inbound messages via three protocols, normalizing all into a unified internal message format before reaching the validation pipeline. The Protocol Gateway is a new inbound port in the hexagonal architecture.

**Supported Inbound Protocols:**

| Protocol | Library | Target Clients | Purpose |
|----------|---------|----------------|---------|
| **REST API** | net/http + Chi | Modern applications, SaaS integrations | Existing JSON-based API for programmatic message submission |
| **SMPP Server** | [gosmpp](https://github.com/linxGnu/gosmpp) | Enterprise clients who speak SMPP natively (telcos, aggregators, legacy platforms) | Accept `submit_sm` PDUs, convert to unified internal format, return `submit_sm_resp` |
| **SMTP Server** | go-smtp or Haraka | Email ingestion, forwarding rules, legacy email systems | Accept inbound email via SMTP, parse MIME, extract metadata, normalize to internal format |

**Normalization Flow:**

```
Inbound Protocol          Normalized Output
+-----------+
| REST API  | --+
+-----------+   |
+-----------+   |     +--------------------+     +-------------------+
| SMPP Srv  | --+---> | Protocol Gateway   | --> | Unified Internal  |
| (gosmpp)  |   |     | Normalization Layer|     | Message Format    |
+-----------+   |     +--------------------+     +-------------------+
+-----------+   |                                         |
| SMTP Srv  | --+                                         v
| (go-smtp) |                                   Validation Pipeline
+-----------+
```

**Rationale:**
- SMPP is the de facto standard for enterprise SMS. Accepting SMPP inbound captures clients who cannot or will not migrate to REST APIs (banks, telcos, legacy notification platforms).
- SMTP ingestion enables email-based workflows (forwarding, reply tracking) and acts as an on-ramp for email migration from legacy MTAs.
- A unified internal format ensures the validation pipeline and cascade engine do not need protocol-specific logic. All downstream processing is protocol-agnostic.

**Trade-off:** Additional operational complexity (two more listening ports, protocol-specific error handling, SMPP session management) in exchange for broader enterprise market coverage and protocol flexibility.

### 11. Smart Routing Engine

The Smart Router sits between the Cascade Engine and the delivery infrastructure, making intelligent per-message routing decisions. Instead of blindly sending via a single provider, the Smart Router evaluates multiple delivery paths and selects the optimal route based on cost, quality, and capacity.

**Routing Decision Factors:**

| Factor | Description | Data Source |
|--------|-------------|-------------|
| **HLR/MNP Lookup** | Real-time number portability lookup to determine current carrier for phone numbers. Prevents misrouting when subscribers port between networks. | hlr-lookups.com or equivalent HLR/MNP provider |
| **Least-Cost Routing (LCR)** | Route via cheapest path that meets quality requirements. Priority: direct carrier connection > aggregator > fallback provider. | Internal rate tables per carrier per destination |
| **Quality Scoring** | Track delivery rates, latency, and cost per route per carrier. Routes with degrading quality are deprioritized automatically. | Historical delivery metrics (Prometheus) |
| **Carrier Capacity Management** | Track SMPP window sizes, throttle rates, and connection health per carrier. Avoid overloading carrier connections. | Real-time SMPP connection state |
| **Geographic Routing** | Route based on destination country/network. Example: Danish numbers via Telia DK direct, international via aggregator. | HLR lookup result + carrier routing tables |

**Routing Decision Flow:**

```
                    +-------------------+
                    |  Cascade Engine   |
                    | (channel decided) |
                    +--------+----------+
                             |
                             v
                    +--------+----------+
                    |   Smart Router    |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
              v              v              v
        +-----+----+  +-----+----+  +------+-----+
        | HLR/MNP  |  |   Rate   |  |  Quality   |
        |  Lookup   |  |  Tables  |  |  Scores    |
        +-----+----+  +-----+----+  +------+-----+
              |              |              |
              +--------------+--------------+
                             |
                             v
                    +--------+----------+
                    | Routing Decision  |
                    | (carrier + path)  |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
              v              v              v
        +-----+----+  +-----+----+  +------+-----+
        |  Direct   |  |Aggregator|  |  Fallback  |
        |  Carrier  |  |   API    |  |  Provider  |
        | (SMPP)    |  | (REST)   |  |  (REST)    |
        +----------+  +----------+  +------------+
```

**Rationale:**
- Direct carrier connections provide 50-80% margin on SMS vs. 10-20% through aggregators, but only if traffic is routed correctly.
- HLR/MNP lookup prevents misrouting (ported numbers going to wrong carrier), which causes delivery failures and wasted cost.
- Quality scoring creates a self-healing routing system -- degraded routes are automatically deprioritized without manual intervention.
- Carrier capacity management prevents SMPP connection overload, which causes message queuing and delivery delays.

### 12. Carrier-Direct Delivery Infrastructure

Instead of relying exclusively on provider APIs (SendGrid, Twilio), Hermes owns its delivery infrastructure with direct connections to carriers and ISPs. Provider APIs are retained as fallback routes for burst capacity and destinations without direct connections.

**SMS -- Direct SMPP Connections:**

| Component | Detail |
|-----------|--------|
| **Protocol** | SMPP 3.4 via [gosmpp](https://github.com/linxGnu/gosmpp) library |
| **Danish Carriers** | Direct SMPP connections to Telia DK, TDC/Nuuday, 3 (Three DK), Telenor DK |
| **Connection Management** | Connection pool manager with persistent TCP connections, bind management (transmitter/receiver/transceiver), window sizing |
| **Delivery Receipts** | DLR handling via SMPP `deliver_sm` PDUs |
| **Fallback Routes** | Aggregator APIs (Messente, Infobip, Twilio) when direct carrier connection is unavailable or at capacity |

**Email -- Own MTA Infrastructure:**

| Component | Detail |
|-----------|--------|
| **MTA** | Self-hosted MTAs (Haraka or custom Go-based MTA) |
| **IP Management** | IP pool management with automated warming schedules |
| **Authentication** | DKIM signing, SPF records, DMARC enforcement per tenant domain |
| **ISP Relations** | Feedback loop processing (Gmail Postmaster Tools, Microsoft SNDS, Yahoo FBL) |
| **Cost Model** | ~$50K/year own MTAs vs. ~$600K/year via SES at 6B emails/year |
| **Fallback** | SendGrid/SES as fallback for burst capacity and warming overflow |

**WhatsApp -- BSP (Business Solution Provider):**

| Component | Detail |
|-----------|--------|
| **Partnership** | Direct Meta partnership as WhatsApp Business Solution Provider |
| **Hosting** | Own hosting of WhatsApp Business API |
| **Features** | Template management, session window tracking, rich media support |

**Future Channels:**

| Channel | Status | Notes |
|---------|--------|-------|
| Apple Messages for Business | Planned | CSP (Customer Service Platform) program |
| RCS Business Messaging | Planned | Carrier-dependent rollout, Jibe/Google partnership |
| Viber Business Messages | Planned | Direct Rakuten Viber partnership |

**Rationale:**
- Carrier-direct SMPP connections provide 50-80% margin on SMS compared to 10-20% through aggregator APIs.
- Own MTA infrastructure saves ~$550K/year compared to SES at scale (6B emails/year).
- Full control over delivery infrastructure eliminates provider lock-in and enables fine-grained optimization (IP warming, carrier throttling, DLR handling).
- Direct carrier relationships are a competitive moat -- they require contracts, technical integration, and ongoing management that most competitors cannot replicate.

**Trade-off:** Significant operational complexity (SMPP binary protocol, TCP session management, MTA IP reputation, carrier contract negotiation) in exchange for dramatically better unit economics and full delivery control.

### 13. Observability

OpenTelemetry + Prometheus for comprehensive per-tenant, per-channel, per-adapter observability.

**Tracing:** Every message gets a trace ID that propagates through the entire cascade lifecycle (API Gateway -> Validation Pipeline -> Cascade Engine -> Channel Adapter -> Provider). Tenant ID is a trace attribute for filtering.

**Key dashboards:**
- Cascade success rate per tenant (how often does Channel 1 suffice vs. falling through?)
- Per-channel delivery latency (p50, p95, p99)
- Adapter health (heartbeat status, response times, error rates)
- Tenant health scores (reputation engine output)
- Rate limit utilization per tenant
- Billing metering accuracy (sent vs. billed)

---

## Bounded Context Canvas

```
+-----------------------------------------------------------------------------------------+
|                              HERMES PLATFORM [HRM]                                       |
+------------------------------------------------------------------------------------------+
|                                                                                          |
|  Description                                                                             |
|  Hermes is a multi-tenant, multi-channel message delivery broker. It receives messages  |
|  via a Protocol Gateway (REST, SMPP, SMTP), validates them through a 9-layer pipeline, |
|  routes them through a cascade engine that attempts delivery across channels in priority |
|  order with configurable fallbacks, and delivers via carrier-direct infrastructure       |
|  (SMPP to carriers, own MTAs, WhatsApp BSP) with aggregator APIs as fallback.           |
|  A Smart Routing Engine performs HLR/MNP lookups and least-cost routing per message.     |
|  The platform manages IP reputation, tenant isolation, billing, and compliance.          |
|                                                                                          |
+------------------------------------------------------------------------------------------+
|                               - - SUBDOMAINS - -                                         |
|  +-----------+ +-----------+ +-----------+ +-----------+ +-----------+ +-----------+   |
|  |  Tenant   | |  Message  | |  Smart    | |  Channel  | | Reputation| |  Billing  |   |
|  | Management| |  Routing  | |  Routing  | |  Delivery | | Management| | & Metering|   |
|  +-----------+ +-----------+ +-----------+ +-----------+ +-----------+ +-----------+   |
|                                                                                          |
+------------------------------------------------------------------------------------------+
|                           - - UBIQUITOUS LANGUAGE - -                                    |
|  +-----------+  +-----------+  +----------+  +----------+  +----------+  +----------+   |
|  |  Cascade  |  |  Channel  |  |  Adapter |  |  Tenant  |  |   IP     |  | Delivery |   |
|  |   Rule    |  |   Type    |  | Registry |  |   Tier   |  |   Pool   |  |  Receipt |   |
|  +-----------+  +-----------+  +----------+  +----------+  +----------+  +----------+   |
|  +-----------+  +-----------+  +----------+  +----------+  +----------+  +----------+   |
|  |  Health   |  |  Instance |  | Cascade  |  | Message  |  | Warming  |  |  Dead    |   |
|  |  Score    |  |   Token   |  |  Step    |  |Validation|  | Schedule |  |  Letter  |   |
|  +-----------+  +-----------+  +----------+  +----------+  +----------+  +----------+   |
|  +-----------+  +-----------+  +----------+  +----------+  +----------+  +----------+   |
|  | Protocol  |  |  Smart    |  | Carrier  |  | HLR/MNP  |  |  SMPP   |  |  MTA     |   |
|  | Gateway   |  |  Router   |  |  Direct  |  |  Lookup  |  |  Bind   |  |  Fleet   |   |
|  +-----------+  +-----------+  +----------+  +----------+  +----------+  +----------+   |
|                                                                                          |
+------------------------------------------------------------------------------------------+
| - - INPUT - - | - - BUSINESS PROCESSES/FEATURES - -   | - - - - OUTPUT - - - - - - - -  |
|               |                                        |                                 |
|  +---------+  |                                        |                                 |
|  |  REST   |  |                                        |                                 |
|  |  API    |--+                                        |                                 |
|  +---------+  |  +-------------+  +--------------+    |  +---------------------------+  |
|  +---------+  |  | Protocol    |  |   Cascade    |    |  |                           |  |
|  |  SMPP   |--+->| Gateway  -> |->|   Engine  -> |---+->|  Delivery via carrier-    |  |
|  |  Server |  |  | 9-Layer     |  | Smart Router |    |  |  direct or fallback       |  |
|  +---------+  |  | Validation  |  | (state machine)|   |  |                           |  |
|  +---------+  |  +-------------+  +--------------+    |  +---------------------------+  |
|  |  SMTP   |--+                                        |                                 |
|  |  Server |  |                                        |                                 |
|  +---------+  |                                        |                                 |
|               |                                        |                                 |
|  +---------+  |  +-------------+  +--------------+    |  +---------------------------+  |
|  |  gRPC   |  |  |  Verify     |  |   Store &    |    |  |                           |  |
|  | Adapter |--+->|  Ed25519    |->|   Issue      |----+->|  Instance Token +         |  |
|  |Register |  |  |  Signature  |  |   Token      |    |  |  Heartbeat Interval       |  |
|  +---------+  |  +-------------+  +--------------+    |  +---------------------------+  |
|               |                                        |                                 |
|  +---------+  |  +-------------+  +--------------+    |  +---------------------------+  |
|  | Webhook |  |  |  Process    |  |   Update     |    |  |                           |  |
|  | (ISP    |--+->|  Feedback   |->|   Reputation |----+->|  Health Score + Actions   |  |
|  | FBL)    |  |  |  Loop       |  |   Scores     |    |  |  (throttle/suspend)       |  |
|  +---------+  |  +-------------+  +--------------+    |  +---------------------------+  |
|               |                                        |                                 |
+------------------------------------------------------------------------------------------+
| - - Emphasis - - - - - - -  | - - - - - - - Assumptions - - - - - - - - - - - - - - -   |
|  +---------------------+    |   +--------------------------------------------------+    |
|  | Security            |    |   | Channel adapters run as containers with network   |    |
|  | Multi-Tenant Safety |    |   | isolation (Docker/K8s) and communicate via gRPC   |    |
|  | IP Reputation       |    |   +--------------------------------------------------+    |
|  | Cascade Reliability |    |   | Supabase provides PostgreSQL with RLS, Auth,      |    |
|  | Compliance (GDPR,   |    |   | and Realtime capabilities per tier                |    |
|  |  TCPA, CAN-SPAM)    |    |   +--------------------------------------------------+    |
|  +---------------------+    |   | NATS JetStream cluster is deployed alongside      |    |
|                             |   | Hermes services for low-latency messaging         |    |
|                             |   +--------------------------------------------------+    |
|                             |   | WorkOS handles SSO/SAML for tenant onboarding     |    |
|                             |   | and team management                               |    |
|                             |   +--------------------------------------------------+    |
+------------------------------------------------------------------------------------------+
| - - Architecture - - - - -  | - - - - - - - - Doubts - - - - - - - - - - - - - - - - -  |
|  +---------------------+    |   +--------------------------------------------------+    |
|  | Hexagonal           |    |   | Should cascade state be stored in NATS KV or     |    |
|  | Ports & Adapters    |    |   | PostgreSQL for durability vs. latency trade-off?  |    |
|  | Go (Chi router)     |    |   +--------------------------------------------------+    |
|  | net/http + Chi      |    |   | Is 30-second heartbeat interval optimal for       |    |
|  | NATS JetStream      |    |   | adapter health detection vs. network overhead?    |    |
|  | Supabase (tiered)   |    |   +--------------------------------------------------+    |
|  | gRPC (grpc-go)      |    |   | Should Enterprise tenants get their own NATS      |    |
|  | Ed25519 + bcrypt    |    |   | JetStream cluster or use subject-based isolation? |    |
|  +---------------------+    |   +--------------------------------------------------+    |
|                             |   | How to handle cascade state recovery after a      |    |
|                             |   | worker crash mid-cascade (at-least-once vs.       |    |
|                             |   | exactly-once delivery semantics)?                 |    |
|                             |   +--------------------------------------------------+    |
+------------------------------------------------------------------------------------------+
| - - - - - - - - - - - - - - - - - Risks - - - - - - - - - - - - - - - - - - - - - - -   |
|  +------------------+  +------------------+  +------------------+  +------------------+  |
|  | Cascade State    |  | IP Reputation    |  | Adapter Key      |  | Tenant Data      |  |
|  | Corruption       |  | Damage           |  | Compromise       |  | Leak             |  |
|  | Mitigated by     |  | Mitigated by     |  | Mitigated by     |  | Mitigated by     |  |
|  | NATS persistence |  | health scoring,  |  | key rotation,    |  | schema-per-      |  |
|  | + idempotency    |  | tenant throttle, |  | Ed25519 + bcrypt |  | tenant + RLS     |  |
|  | + recovery FSM   |  | IP pool isolate  |  | HSM recommended  |  | + tier isolation |  |
|  +------------------+  +------------------+  +------------------+  +------------------+  |
+-----------------------------------------------------------------------------------------+
```

---

## Functional Responsibilities

### Tenant Management Subdomain

- **Onboarding**: WorkOS SSO magic-link signup, tenant provisioning (schema/project/cluster per tier)
- **API Key Management**: Generate, rotate, revoke API keys with bcrypt hashing and permission scoping
- **Tier Management**: Upgrade/downgrade between Starter, Business, Enterprise with zero-downtime migration
- **Tenant Health Monitoring**: Real-time health scores based on sending behavior, compliance, and payment status
- **Configuration**: Cascade rules, rate limits, compliance settings, webhook URLs per tenant

### Message Routing Subdomain

- **Validation Pipeline**: 9-layer sequential validation (idempotency through audit logging)
- **Cascade Orchestration**: State machine per message executing tenant-configured cascade rules
- **Dead Letter Handling**: Messages that exhaust all cascade steps are routed to DLQ with diagnostic metadata
- **Webhook Delivery**: Real-time status callbacks to tenant webhook URLs on state transitions
- **Retry Management**: Per-step retry with configurable backoff (fixed, exponential, linear)

### Smart Routing Subdomain

- **HLR/MNP Lookup**: Real-time number portability lookups to determine current carrier for phone numbers
- **Least-Cost Routing**: Route via cheapest path meeting quality requirements (direct carrier > aggregator > fallback)
- **Quality Scoring**: Track delivery rates, latency, and cost per route per carrier with automatic deprioritization of degraded routes
- **Carrier Capacity Management**: Track SMPP window sizes, throttle rates, and connection health per carrier
- **Geographic Routing**: Route based on destination country/network (e.g., Danish numbers via Telia DK direct, international via aggregator)
- **Routing Audit Trail**: Record all routing decisions with rationale for post-hoc analysis and optimization

### Channel Delivery Subdomain

- **Adapter Registry**: Registration, heartbeat monitoring, and hot-swap of channel adapter containers
- **Adapter Routing**: Resolve active adapter for channel type, route gRPC calls, handle timeouts
- **Carrier-Direct Delivery**: SMPP client connections to telecom carriers (Telia DK, TDC, 3 DK, Telenor), connection pool management, DLR handling
- **Own MTA Delivery**: Self-hosted MTA fleet for email delivery, IP pool management, DKIM/SPF/DMARC enforcement
- **Delivery Confirmation**: Poll or receive webhooks from channel providers for delivery status; process SMPP DLRs from carriers
- **Channel-Specific Logic**: Email (MIME construction, attachment handling), SMS (segment counting, encoding, SMPP PDU construction), WhatsApp (template management via BSP), Push (platform routing iOS/Android)

### Reputation Management Subdomain

- **IP Pool Management**: Assign, warm, rotate IPs across tenant tiers
- **Reputation Monitoring**: Real-time bounce rate, complaint rate, blacklist status per IP and per tenant
- **ISP Feedback Processing**: Gmail Postmaster, Microsoft SNDS, Yahoo FBL integration
- **Tenant Health Scoring**: ML-assisted scoring of tenant sending behavior with automatic throttling
- **Channel-Specific Protection**: DKIM/SPF/DMARC for email, 10DLC for SMS, quality rating for WhatsApp

### Billing Subdomain

- **Usage Metering**: Count messages per channel per tenant with sub-second accuracy via NATS events
- **Balance Management**: Prepaid credit checking with atomic reservation, postpaid credit limit enforcement
- **Invoice Generation**: Monthly billing cycles with per-channel breakdowns
- **Overage Handling**: Configurable behavior when balance is exhausted (block, overdraft with limit, alert)

---

## Inbound Traffic

### Protocol Gateway (Inbound Normalization)

All inbound protocols are normalized to a unified internal message format by the Protocol Gateway before entering the validation pipeline. The gateway supports three protocols:

### REST API (Tenant-Facing)

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `POST /v1/messages/send` | POST | API Key | Send a message (enters cascade pipeline) |
| `POST /v1/messages/batch` | POST | API Key | Send up to 1,000 messages in one request |
| `GET /v1/messages/{id}` | GET | API Key | Get message status and cascade history |
| `GET /v1/messages/{id}/events` | GET | API Key | Get delivery event timeline |
| `POST /v1/cascade-rules` | POST | API Key | Create a cascade rule |
| `GET /v1/cascade-rules` | GET | API Key | List cascade rules |
| `PUT /v1/cascade-rules/{id}` | PUT | API Key | Update a cascade rule |
| `GET /v1/tenants/me` | GET | API Key | Get tenant profile and usage |
| `POST /v1/tenants/api-keys` | POST | API Key | Generate new API key |
| `DELETE /v1/tenants/api-keys/{id}` | DELETE | API Key | Revoke API key |

### SMPP Server (Enterprise Inbound)

| PDU | Direction | Auth | Purpose |
|-----|-----------|------|---------|
| `bind_transmitter` | Inbound | SMPP credentials (system_id + password) | Enterprise client establishes transmitter session |
| `bind_transceiver` | Inbound | SMPP credentials | Enterprise client establishes bidirectional session |
| `submit_sm` | Inbound | Bound session | Enterprise client submits SMS for delivery |
| `submit_sm_resp` | Outbound | Bound session | Acknowledge receipt of submitted message |
| `deliver_sm` | Outbound | Bound session | Deliver DLR (delivery receipt) back to enterprise client |
| `enquire_link` | Bidirectional | Bound session | Keep-alive heartbeat (30s interval) |

**Implementation:** gosmpp (github.com/linxGnu/gosmpp), SMPP 3.4 protocol. Enterprise clients connect using standard SMPP credentials mapped to tenant API keys. Inbound `submit_sm` PDUs are normalized to the unified internal message format and routed through the validation pipeline.

### SMTP Server (Email Inbound)

| Operation | Direction | Auth | Purpose |
|-----------|-----------|------|---------|
| `EHLO/HELO` | Inbound | None (TLS negotiated via STARTTLS) | SMTP session establishment |
| `MAIL FROM` | Inbound | SPF/DKIM verification | Sender identification and validation |
| `RCPT TO` | Inbound | Tenant domain mapping | Recipient routing to tenant |
| `DATA` | Inbound | Content scanning | Email body with MIME parsing |

**Implementation:** go-smtp or Haraka. Inbound emails are parsed (headers, body, attachments), mapped to a tenant via domain configuration, normalized to the unified internal message format, and routed through the validation pipeline. Supports STARTTLS, SPF verification, and DKIM validation on inbound.

### gRPC Internal (Adapter Communication)

| Service | Method | Auth | Purpose |
|---------|--------|------|---------|
| `AdapterRegistry` | `Register` | Ed25519 signature | Register channel adapter container |
| `AdapterRegistry` | `Heartbeat` | Instance token (bcrypt) | Send health signal |
| `AdapterRegistry` | `Deregister` | Instance token (bcrypt) | Graceful shutdown |
| `ChannelAdapterService` | `Send` | mTLS | Route message to adapter |
| `ChannelAdapterService` | `CheckStatus` | mTLS | Query delivery status |

### Webhooks (Inbound from Providers)

| Source | Purpose |
|--------|---------|
| SendGrid Event Webhook | Email delivery events (delivered, bounced, opened, clicked) |
| Twilio Status Callback | SMS delivery events (sent, delivered, failed, undelivered) |
| WhatsApp Business API Webhook | Message status updates, read receipts |
| Telegram Bot API Webhook | Message delivery confirmations |
| FCM Delivery Data | Push notification delivery and engagement |
| Gmail Postmaster Tools | Domain reputation, spam rate, authentication |
| Microsoft SNDS | IP reputation data |
| Yahoo FBL | Feedback loop complaints |

---

## Outbound Traffic

### External Service Communication

**Carrier-Direct Connections (Primary Delivery)**
- Direct SMPP connections to Danish carriers: Telia DK, TDC/Nuuday, 3 (Three DK), Telenor DK
- Own MTA infrastructure (Haraka or custom Go MTA) for direct email delivery
- WhatsApp Business API (self-hosted BSP)

**Aggregator/Provider APIs (Fallback)**
- SendGrid v3 API (Email fallback / burst capacity)
- Amazon SES (Email fallback / burst capacity)
- Twilio REST API (SMS fallback)
- Messente API (SMS aggregator fallback)
- Infobip API (SMS aggregator fallback)
- Telegram Bot API (direct, no carrier equivalent)
- Firebase Cloud Messaging v1 API (direct, no carrier equivalent)

**Routing & Lookup Services**
- HLR/MNP providers (hlr-lookups.com or equivalent) - Number portability lookups for SMS routing

**Infrastructure Services**
- Supabase (PostgreSQL via pgx) - Tenant data, cascade state, audit logs
- NATS JetStream - Message queuing, KV store for rate limits and idempotency
- WorkOS - Tenant authentication, SSO/SAML

**Monitoring**
- OpenTelemetry Collector - Traces and metrics export
- Prometheus - Metrics scraping endpoint

---

## Sequence Diagrams

### Tenant Onboarding

```
+-------------+     +-------------+     +-------------+     +-------------+
|  Developer  |     |   WorkOS    |     |   Tenant    |     |   Tenant    |
|             |     |   SSO       |     |  Registry   |     | Provisioner |
+------+------+     +------+------+     +------+------+     +------+------+
       |                   |                   |                   |
       | Sign up           |                   |                   |
       | (magic link)      |                   |                   |
       |------------------>|                   |                   |
       |                   |                   |                   |
       |   Auth callback   |                   |                   |
       |   (JWT + profile) |                   |                   |
       |<------------------|                   |                   |
       |                   |                   |                   |
       | Create tenant     |                   |                   |
       | (name, tier)      |                   |                   |
       |-------------------------------------->|                   |
       |                   |                   |                   |
       |                   |                   | provisionTenant   |
       |                   |                   | (tier-specific)   |
       |                   |                   |------------------>|
       |                   |                   |                   |
       |                   |                   |                   | Starter: CREATE SCHEMA
       |                   |                   |                   | Business: CREATE PROJECT
       |                   |                   |                   | Enterprise: DEPLOY CLUSTER
       |                   |                   |                   |
       |                   |                   |  provisioned      |
       |                   |                   |<------------------|
       |                   |                   |                   |
       |                   |                   | generateApiKey    |
       |                   |                   | (Ed25519 sign,    |
       |                   |                   |  bcrypt hash)     |
       |                   |                   |--------+          |
       |                   |                   |<-------+          |
       |                   |                   |                   |
       |                   |                   | seedDefaults      |
       |                   |                   | (cascade rules,   |
       |                   |                   |  rate limits)     |
       |                   |                   |--------+          |
       |                   |                   |<-------+          |
       |                   |                   |                   |
       |  tenant_id +      |                   |                   |
       |  api_key +         |                   |                   |
       |  dashboard_url    |                   |                   |
       |<--------------------------------------|                   |
       |                   |                   |                   |
```

### Message Send Flow (Happy Path)

```
+-------------+     +-------------+     +-------------+     +-------------+
|   Client    |     | API Gateway |     |  Validation |     |   NATS      |
|  (Tenant)   |     | (net/http)  |     |  Pipeline   |     |  JetStream  |
+------+------+     +------+------+     +------+------+     +------+------+
       |                   |                   |                   |
       | POST /v1/messages |                   |                   |
       | /send             |                   |                   |
       | {to, body,        |                   |                   |
       |  cascade_rule_id} |                   |                   |
       |------------------>|                   |                   |
       |                   |                   |                   |
       |                   | Layer 0: idempotency                  |
       |                   |------------------>|                   |
       |                   |                   |---+               |
       |                   |                   |<--+ check KV      |
       |                   |                   |                   |
       |                   | Layers 1-8:       |                   |
       |                   | auth, tenant,     |                   |
       |                   | rate, balance,    |                   |
       |                   | schema, compliance|                   |
       |                   | content, audit    |                   |
       |                   |                   |---+               |
       |                   |                   |<--+ all pass      |
       |                   |                   |                   |
       |                   | Layer 9: route    |                   |
       |                   |                   |                   |
       |                   |                   | publish to        |
       |                   |                   | hermes.messages   |
       |                   |                   | .{tenant_id}      |
       |                   |                   |------------------>|
       |                   |                   |      ack          |
       |                   |                   |<------------------|
       |                   |                   |                   |
       |  202 Accepted     |                   |                   |
       |  {message_id,     |                   |                   |
       |   tracking_url}   |                   |                   |
       |<------------------|                   |                   |
       |                   |                   |                   |
```

### Cascade Delivery Flow

```
+-------------+     +-------------+     +-------------+     +-------------+
|    NATS     |     |  Cascade    |     |  Channel    |     |  Adapter    |
|  JetStream  |     |  Engine     |     |  Router     |     | (SendGrid) |
+------+------+     +------+------+     +------+------+     +------+------+
       |                   |                   |                   |
       | CascadeMessage    |                   |                   |
       | (tenant_id,       |                   |                   |
       |  message_id,      |                   |                   |
       |  cascade_rule)    |                   |                   |
       |------------------>|                   |                   |
       |                   |                   |                   |
       |                   | State: CREATED    |                   |
       |                   | -> ATTEMPTING     |                   |
       |                   | (Step 1: Email)   |                   |
       |                   |---+               |                   |
       |                   |<--+               |                   |
       |                   |                   |                   |
       |                   | resolveAdapter    |                   |
       |                   | (Email)           |                   |
       |                   |------------------>|                   |
       |                   |                   | gRPC Send()       |
       |                   |                   |------------------>|
       |                   |                   |  accepted=true    |
       |                   |                   |  provider_msg_id  |
       |                   |                   |<------------------|
       |                   |  DeliveryReceipt  |                   |
       |                   |<------------------|                   |
       |                   |                   |                   |
       |                   | State: WAITING    |                   |
       |                   | (timeout: 30s)    |                   |
       |                   |---+               |                   |
       |                   |<--+ start timer   |                   |
       |                   |                   |                   |
       |                   |   ... 30s pass, no confirmation ...   |
       |                   |                   |                   |
       |                   | State: FAILED     |                   |
       |                   | (Step 1: Email)   |                   |
       |                   | -> ATTEMPTING     |                   |
       |                   | (Step 2: SMS)     |                   |
       |                   |---+               |                   |
       |                   |<--+               |                   |
       |                   |                   |                   |
       |                   |     [continues cascade to SMS...]     |
       |                   |                   |                   |
```

### Channel Adapter Registration

```
+-------------+     +-------------+     +-------------+     +-------------+
|  Adapter    |     |  Adapter    |     |  Adapter    |     |  Database   |
|  Container  |     |  Registry   |     | (gRPC svc)  |     | (Supabase)  |
+------+------+     +------+------+     +------+------+     +------+------+
       |                   |                   |                   |
       | gRPC Register     |                   |                   |
       | (signed payload,  |                   |                   |
       |  public_key_id,   |                   |                   |
       |  channel_type,    |                   |                   |
       |  capabilities)    |                   |                   |
       |------------------>|                   |                   |
       |                   |                   |                   |
       |                   | validateAllowlist |                   |
       |                   |---+               |                   |
       |                   |<--+               |                   |
       |                   |                   |                   |
       |                   | getTrustedKey     |                   |
       |                   |----------------------------------------->|
       |                   |                   |   public_key      |
       |                   |<-----------------------------------------|
       |                   |                   |                   |
       |                   | verifyEd25519     |                   |
       |                   | (payload, sig,    |                   |
       |                   |  public_key)      |                   |
       |                   |---+               |                   |
       |                   |<--+ verified      |                   |
       |                   |                   |                   |
       |                   | generateToken     |                   |
       |                   | (32 bytes random) |                   |
       |                   |---+               |                   |
       |                   |<--+               |                   |
       |                   |                   |                   |
       |                   | bcrypt.hash(token)|                   |
       |                   |---+               |                   |
       |                   |<--+               |                   |
       |                   |                   |                   |
       |                   | upsert adapter    |                   |
       |                   |----------------------------------------->|
       |                   |                   |   adapter row     |
       |                   |<-----------------------------------------|
       |                   |                   |                   |
       |  instance_token + |                   |                   |
       |  heartbeat_interval                   |                   |
       |<------------------|                   |                   |
       |                   |                   |                   |
       |                                                           |
       | [Every 30 seconds]                                        |
       |                   |                   |                   |
       | gRPC Heartbeat    |                   |                   |
       | (instance_token,  |                   |                   |
       |  health_status)   |                   |                   |
       |------------------>|                   |                   |
       |                   | bcrypt.compare    |                   |
       |                   |---+               |                   |
       |                   |<--+               |                   |
       |                   |                   |                   |
       |                   | updateHeartbeat   |                   |
       |                   |----------------------------------------->|
       |                   |                   |        OK         |
       |                   |<-----------------------------------------|
       |                   |                   |                   |
       | shouldShutdown    |                   |                   |
       |<------------------|                   |                   |
       |                   |                   |                   |
```

---

## Services (Main Cluster)

| Service | Responsibility | Interaction |
|---------|---------------|-------------|
| **Protocol Gateway** | Accepts REST (net/http + Chi), SMPP (gosmpp), and SMTP (go-smtp) inbound traffic. Normalizes all protocols to unified internal message format. | Inbound port; feeds Validation Pipeline |
| **API Gateway** | HTTP routing (net/http + Chi), TLS termination (crypto/tls), request validation, response compression | Inbound port within Protocol Gateway; calls Validation Pipeline |
| **Tenant Registry** | Tenant CRUD, API key management, tier configuration, health monitoring | Domain service; called by Validation Pipeline |
| **Channel Adapter Registry** | Adapter registration (Ed25519), heartbeat monitoring, adapter resolution, hot-swap coordination | Domain service; called by Channel Router |
| **Cascade Engine** | Per-message state machine (goroutine per cascade), cascade rule execution, timeout management, retry orchestration | Domain service; consumes from NATS, calls Channel Router |
| **Reputation Engine** | IP pool management, warming schedules, reputation monitoring, tenant health scoring, ISP FBL processing | Domain service; called by Validation Pipeline (Layer 7) and standalone cron |
| **Billing & Metering** | Usage counting, balance management, credit reservation, invoice generation | Domain service; called by Validation Pipeline (Layer 4) |
| **Tenant Provisioner** | Schema creation (Starter), Supabase project creation (Business), cluster deployment (Enterprise) | Infrastructure service; called by Tenant Registry |
| **Smart Router** | HLR/MNP lookups, least-cost routing, quality scoring, carrier capacity management, geographic routing | Domain service; called by Cascade Engine before delivery |
| **SMPP Connection Manager** | Persistent SMPP connections to carriers, connection pooling, bind management, window sizing, DLR processing | Infrastructure service; called by Smart Router for carrier-direct SMS delivery |
| **MTA Fleet** | Self-hosted email delivery, IP pool management, DKIM signing, bounce processing | Infrastructure service; called by Smart Router for direct email delivery |
| **Observability Collector** | OpenTelemetry trace/metric aggregation, Prometheus exposition endpoint | Infrastructure service; passive collection |

---

## Security Model

### OWASP Top 10 Alignment

| OWASP Category | Control | Implementation |
|----------------|---------|----------------|
| A01:2021 Broken Access Control | Tenant isolation + API key scoping | Schema-per-tenant with RLS, API key permissions, tier-based resource limits |
| A02:2021 Cryptographic Failures | Ed25519 + bcrypt + crypto/tls | Ed25519 for adapter signatures (crypto/ed25519), bcrypt for API keys and instance tokens (golang.org/x/crypto/bcrypt), TLS 1.3 via crypto/tls for all external traffic |
| A03:2021 Injection | Input validation (Layer 5) | JSON schema validation, recipient format validation, SQL parameterized queries via pgx |
| A04:2021 Insecure Design | Hexagonal architecture + defense-in-depth | 9-layer validation pipeline, cascade state machine with explicit states, no implicit trust |
| A05:2021 Security Misconfiguration | Secure defaults + environment-based config | All features disabled by default, allowlists for adapters, explicit tenant activation required |
| A06:2021 Vulnerable Components | Go + govulncheck + minimal dependencies | Memory-safe by default, `govulncheck` in CI pipeline, Go race detector enabled in CI |
| A07:2021 Auth Failures | Rate limiting + key rotation | Per-tenant sliding window (Layer 3), API key expiration, secondary key for rotation |
| A08:2021 Data Integrity | Signed adapter registration + hash-chained audit | Ed25519 payload signing, timestamp + nonce for replay prevention, append-only hash-chained audit log |
| A09:2021 Logging & Monitoring | Structured logging + OpenTelemetry | Layer 8 pre-execution audit, post-execution result logging, real-time alerting via Prometheus |
| A10:2021 SSRF | Adapter URL validation + network isolation | Adapter endpoint URLs validated against allowlist, containers in isolated network, no user-supplied URLs in outbound requests |

### Security Layers (Defense-in-Depth)

```
+-----------------------------------------------------------------------+
|                        SECURITY LAYERS                                 |
+-----------------------------------------------------------------------+
|                                                                        |
|  Layer 1: Network Isolation                                            |
|  +-- Channel adapter containers in isolated Docker/K8s network        |
|  +-- Internal gRPC endpoints not exposed to public internet           |
|  +-- NATS JetStream accessible only from service mesh                 |
|                                                                        |
|  Layer 2: TLS Everywhere                                               |
|  +-- crypto/tls for all external HTTP (TLS 1.3 only)                 |
|  +-- mTLS for adapter <-> registry gRPC communication                 |
|  +-- Encrypted NATS connections                                       |
|                                                                        |
|  Layer 3: Tenant Authentication                                        |
|  +-- API key validated via bcrypt hash comparison                     |
|  +-- Key scoping (read-only, send, admin)                             |
|  +-- Key expiration and rotation support                              |
|                                                                        |
|  Layer 4: Tenant Authorization                                         |
|  +-- Tier-based access control (feature gates per tier)               |
|  +-- Per-key permission scoping                                       |
|  +-- Rate limits enforced per tier and per key                        |
|                                                                        |
|  Layer 5: Input Validation                                             |
|  +-- JSON schema validation for all request bodies                    |
|  +-- Recipient format validation (email, E.164, device token)         |
|  +-- Content size limits (per channel, per tier)                      |
|                                                                        |
|  Layer 6: Adapter Authentication                                       |
|  +-- Ed25519 signature verification for registration                  |
|  +-- Bcrypt instance tokens for heartbeat/deregistration              |
|  +-- Adapter allowlist (only pre-approved adapters can register)      |
|  +-- Timestamp + nonce for replay attack prevention                   |
|                                                                        |
|  Layer 7: Compliance Enforcement                                       |
|  +-- GDPR consent verification per recipient per channel              |
|  +-- TCPA opt-in verification for US SMS                              |
|  +-- CAN-SPAM compliance for email (unsubscribe, physical address)    |
|  +-- Quiet hours enforcement per jurisdiction                         |
|                                                                        |
|  Layer 8: Content Protection                                           |
|  +-- Anti-spam scoring before sending                                 |
|  +-- Phishing URL detection                                          |
|  +-- Reputation impact prediction                                     |
|  +-- Tenant health scoring with automatic throttling                  |
|                                                                        |
|  Layer 9: Audit & Monitoring                                           |
|  +-- Hash-chained, append-only audit log                              |
|  +-- Pre-execution and post-execution logging                         |
|  +-- Real-time alerting on anomalous patterns                         |
|  +-- Per-tenant, per-channel, per-adapter metrics                     |
|                                                                        |
+-----------------------------------------------------------------------+
```

### Compliance Matrix

| Regulation | Requirement | Implementation |
|------------|-------------|----------------|
| **GDPR** (EU) | Consent management, right to erasure, data portability | Consent records per recipient per channel (Layer 6), tenant data export API, scheduled purge jobs |
| **TCPA** (US) | Prior express consent for SMS, time-of-day restrictions | Opt-in verification (Layer 6), quiet hours enforcement, revocation processing |
| **CAN-SPAM** (US) | Unsubscribe mechanism, sender identification | Automatic unsubscribe header injection, physical address requirement in email templates |
| **CASL** (Canada) | Express or implied consent, identification | Consent type tracking, sender identification enforcement |
| **PECR** (UK) | Consent for electronic marketing | Aligned with GDPR consent model, UK-specific quiet hours |
| **WhatsApp Business Policy** | Template pre-approval, 24h messaging window | Template registry with approval status, session window tracking |
| **10DLC** (US SMS) | Brand registration, campaign registration | Campaign ID tracking, throughput management per registration status |

---

## Non-Functional Requirements

| NFR | Requirement | Implementation |
|-----|-------------|----------------|
| **Throughput** | 2B texts + 6B emails/year (~22M msg/day sustained, ~100M/day peak) | Carrier-direct SMPP connections, own MTA fleet, NATS JetStream for buffering, horizontal scaling per tenant and per carrier |
| **Latency** | API response < 50ms (p99 for 202 Accepted) | Async processing after validation, NATS publish is sub-millisecond |
| **Latency** | Cascade step transition < 100ms | In-memory state machine, NATS consumer with direct get |
| **Availability** | 99.9% for Business tier, 99.99% for Enterprise | Multi-AZ deployment, NATS cluster replication, Supabase HA |
| **Durability** | Zero message loss after 202 Accepted | NATS JetStream persistence with file-based storage, at-least-once delivery |
| **Security** | Tenant data isolation | Schema-per-tenant (Starter), project-per-tenant (Business), cluster-per-tenant (Enterprise) |
| **Security** | Encryption at rest and in transit | crypto/tls TLS 1.3, Supabase encrypted storage, NATS TLS |
| **Compliance** | GDPR, TCPA, CAN-SPAM | 9-layer validation pipeline with compliance check at Layer 6 |
| **Observability** | Per-tenant, per-channel, per-adapter metrics | OpenTelemetry traces with tenant_id attribute, Prometheus counters and histograms |
| **Extensibility** | Add new channel in < 1 day | Hot-loadable adapter containers with standardized gRPC contract |
| **Operability** | Zero-downtime deployments | Rolling updates, adapter hot-swap, NATS consumer rebalancing |
| **SMPP Connections** | Maintain 50+ persistent SMPP connections | Connection pool manager with bind management, window sizing, automatic reconnection, health monitoring per carrier |
| **MTA Throughput** | 200K emails/hour per MTA instance | Horizontal MTA scaling, IP pool management, automated warming, bounce processing |

---

## Metrics

### API Gateway

| Metric | Type | Description |
|--------|------|-------------|
| `hermes_api_requests_total` | Counter | Total API requests by endpoint, method, status |
| `hermes_api_latency_seconds` | Histogram | Request latency by endpoint (p50, p95, p99) |
| `hermes_api_active_connections` | Gauge | Current active HTTP connections |

### Validation Pipeline

| Metric | Type | Description |
|--------|------|-------------|
| `hermes_validation_layer_duration_seconds` | Histogram | Time spent in each validation layer |
| `hermes_validation_rejections_total` | Counter | Rejections by layer number and reason |
| `hermes_validation_pass_total` | Counter | Messages that passed all 9 layers |
| `hermes_rate_limit_hits_total` | Counter | Rate limit violations per tenant |
| `hermes_balance_check_failures_total` | Counter | Balance/quota check failures (billing critical) |
| `hermes_compliance_blocks_total` | Counter | Compliance rejections by regulation type |

### Cascade Engine

| Metric | Type | Description |
|--------|------|-------------|
| `hermes_cascade_messages_total` | Counter | Total cascade messages by final state |
| `hermes_cascade_step_attempts_total` | Counter | Attempts per cascade step (channel) |
| `hermes_cascade_step_duration_seconds` | Histogram | Time per cascade step |
| `hermes_cascade_depth_total` | Histogram | How many steps before delivery (1 = first channel worked) |
| `hermes_cascade_success_rate` | Gauge | Percentage of messages delivered (any channel) per tenant |
| `hermes_cascade_active_messages` | Gauge | Messages currently in cascade processing |
| `hermes_cascade_goroutines_active` | Gauge | Active cascade goroutines (monitor for leaks) |

### Channel Adapters

| Metric | Type | Description |
|--------|------|-------------|
| `hermes_adapter_registration_total` | Counter | Adapter registration attempts and successes |
| `hermes_adapter_heartbeat_total` | Counter | Heartbeat attempts and token mismatches |
| `hermes_adapter_active` | Gauge | Currently active adapters by channel type |
| `hermes_adapter_send_duration_seconds` | Histogram | Send latency per adapter |
| `hermes_adapter_send_errors_total` | Counter | Send errors per adapter and error type |
| `hermes_adapter_signature_failures_total` | Counter | Ed25519 verification failures |

### Reputation Engine

| Metric | Type | Description |
|--------|------|-------------|
| `hermes_reputation_tenant_score` | Gauge | Current health score per tenant (0-100) |
| `hermes_reputation_bounce_rate` | Gauge | Current bounce rate per tenant per channel |
| `hermes_reputation_complaint_rate` | Gauge | Current complaint rate per tenant |
| `hermes_reputation_blacklist_count` | Gauge | Number of blacklist appearances per IP |
| `hermes_reputation_throttled_tenants` | Gauge | Tenants currently being throttled |
| `hermes_reputation_suspended_tenants` | Gauge | Tenants suspended for poor health |

### Billing

| Metric | Type | Description |
|--------|------|-------------|
| `hermes_billing_messages_metered_total` | Counter | Messages counted for billing per channel |
| `hermes_billing_balance_remaining` | Gauge | Remaining prepaid balance per tenant |
| `hermes_billing_reservation_failures_total` | Counter | Failed balance reservations |

### Carrier Infrastructure (SMPP / MTA / Routing)

| Metric | Type | Description |
|--------|------|-------------|
| `hermes_smpp_connections_active` | Gauge | Active SMPP binds per carrier |
| `hermes_smpp_submit_sm_total` | Counter | SMS submitted via SMPP per carrier |
| `hermes_smpp_deliver_sm_total` | Counter | Delivery receipts received per carrier |
| `hermes_smpp_window_utilization` | Gauge | SMPP window usage per connection (0.0-1.0) |
| `hermes_mta_emails_sent_total` | Counter | Emails sent via own MTAs per instance |
| `hermes_mta_bounce_rate` | Gauge | Bounce rate per MTA instance |
| `hermes_hlr_lookups_total` | Counter | HLR/MNP lookups performed |
| `hermes_routing_decisions_total` | Counter | Routing decisions by path type (direct/aggregator/fallback) |
| `hermes_carrier_cost_per_message` | Histogram | Cost per message by route |

### Critical Alerts

```yaml
# CRITICAL: Cascade engine backlog growing
- alert: CascadeBacklogGrowing
  expr: rate(hermes_cascade_active_messages[5m]) > 100
  for: 5m
  severity: critical
  annotations:
    summary: "Cascade engine backlog growing rapidly"
    description: "Active cascade messages increasing > 100/min for 5 minutes"

# CRITICAL: Tenant health score below suspension threshold
- alert: TenantHealthCritical
  expr: hermes_reputation_tenant_score < 20
  for: 0s
  severity: critical
  annotations:
    summary: "Tenant health score below suspension threshold"
    description: "Tenant {{ $labels.tenant_id }} score {{ $value }} < 20"

# WARNING: Adapter heartbeat missed
- alert: AdapterHeartbeatMissed
  expr: time() - hermes_adapter_last_heartbeat_timestamp > 90
  for: 0s
  severity: warning
  annotations:
    summary: "Channel adapter missed 3 heartbeats"
    description: "Adapter {{ $labels.adapter_id }} ({{ $labels.channel_type }}) has not sent a heartbeat in 90 seconds"

# CRITICAL: Balance check failures spiking (revenue impact)
- alert: BalanceCheckFailureSpike
  expr: rate(hermes_balance_check_failures_total[5m]) > 10
  for: 2m
  severity: critical
  annotations:
    summary: "Balance check failures spiking - potential billing system issue"

# WARNING: Goroutine leak detected
- alert: GoroutineLeakDetected
  expr: hermes_cascade_goroutines_active > 10000
  for: 5m
  severity: warning
  annotations:
    summary: "Excessive active cascade goroutines"
    description: "Active cascade goroutines {{ $value }} exceeds threshold. Possible goroutine leak."
```

---

## Contracts

### Core Domain Types (Go)

```go
package domain

import (
    "time"

    "github.com/google/uuid"
)

// --- Identity Types ---

type TenantID struct{ uuid.UUID }
type MessageID struct{ uuid.UUID }
type CascadeRuleID struct{ uuid.UUID }
type AdapterID struct{ uuid.UUID }
type APIKey string // Never stored plaintext; bcrypt hash in DB

// --- Message Types ---

// SendMessageRequest is the primary inbound API payload.
type SendMessageRequest struct {
    IdempotencyKey string            `json:"idempotency_key"`
    To             Recipient         `json:"to"`
    Body           MessageBody       `json:"body"`
    Routing        RoutingConfig     `json:"routing"`
    Metadata       map[string]string `json:"metadata,omitempty"`
    SendAt         *time.Time        `json:"send_at,omitempty"`
}

// Recipient identifies who receives the message.
// Discriminated by the Type field.
type Recipient struct {
    Type        RecipientType `json:"type"` // "email", "phone", "device_token", "user_id", "multi"
    Email       string        `json:"email,omitempty"`
    Phone       string        `json:"phone,omitempty"`       // E.164 format
    DeviceToken string        `json:"device_token,omitempty"` // FCM/APNs token
    UserID      string        `json:"user_id,omitempty"`     // Platform-resolved
}

type RecipientType string

const (
    RecipientEmail       RecipientType = "email"
    RecipientPhone       RecipientType = "phone"
    RecipientDeviceToken RecipientType = "device_token"
    RecipientUserID      RecipientType = "user_id"
    RecipientMulti       RecipientType = "multi"
)

// MessageBody contains the content to deliver.
type MessageBody struct {
    Text         string                 `json:"text"`
    HTML         string                 `json:"html,omitempty"`
    Subject      string                 `json:"subject,omitempty"`
    TemplateID   string                 `json:"template_id,omitempty"`
    TemplateVars map[string]interface{} `json:"template_vars,omitempty"`
}

// RoutingConfig determines how the message is routed.
// Discriminated by the Type field.
type RoutingConfig struct {
    Type          RoutingType         `json:"type"` // "cascade_rule", "channels", "direct"
    CascadeRuleID *CascadeRuleID     `json:"cascade_rule_id,omitempty"`
    Channels      []ChannelPreference `json:"channels,omitempty"`
    DirectChannel *ChannelType        `json:"direct_channel,omitempty"`
}

type RoutingType string

const (
    RoutingCascadeRule RoutingType = "cascade_rule"
    RoutingChannels    RoutingType = "channels"
    RoutingDirect      RoutingType = "direct"
)

// ChannelPreference defines a single channel attempt in an inline cascade.
type ChannelPreference struct {
    Channel    ChannelType    `json:"channel"`
    Timeout    *time.Duration `json:"timeout,omitempty"`
    MaxRetries *uint8         `json:"max_retries,omitempty"`
}

// --- Cascade State Machine ---

// CascadeState tracks the full lifecycle of a message through the cascade.
type CascadeState struct {
    MessageID   MessageID        `json:"message_id"`
    TenantID    TenantID         `json:"tenant_id"`
    Rule        CascadeRule      `json:"rule"`
    CurrentStep int              `json:"current_step"`
    Status      CascadeStatus    `json:"status"`
    Attempts    []CascadeAttempt `json:"attempts"`
    CreatedAt   time.Time        `json:"created_at"`
    UpdatedAt   time.Time        `json:"updated_at"`
}

// CascadeStatus represents the current state of a cascade.
// Discriminated by the State field.
type CascadeStatus struct {
    State    CascadeState_  `json:"state"`
    Step     int            `json:"step,omitempty"`
    Channel  ChannelType    `json:"channel,omitempty"`
    Deadline *time.Time     `json:"deadline,omitempty"`
}

type CascadeState_ string

const (
    StateCreated              CascadeState_ = "created"
    StateAttempting           CascadeState_ = "attempting"
    StateWaitingConfirmation  CascadeState_ = "waiting_confirmation"
    StateDelivered            CascadeState_ = "delivered"
    StateExhausted            CascadeState_ = "exhausted"
    StateCancelled            CascadeState_ = "cancelled"
    StateExpired              CascadeState_ = "expired"
)

// CascadeAttempt records a single delivery attempt within a cascade.
type CascadeAttempt struct {
    Step              int            `json:"step"`
    Channel           ChannelType    `json:"channel"`
    AdapterID         AdapterID      `json:"adapter_id"`
    Status            AttemptStatus  `json:"status"`
    ProviderMessageID string         `json:"provider_message_id,omitempty"`
    Error             string         `json:"error,omitempty"`
    StartedAt         time.Time      `json:"started_at"`
    CompletedAt       *time.Time     `json:"completed_at,omitempty"`
}

// AttemptStatus represents the outcome of a single delivery attempt.
type AttemptStatus struct {
    State   AttemptState `json:"state"`
    Code    string       `json:"code,omitempty"`    // failure code
    Message string       `json:"message,omitempty"` // failure message
}

type AttemptState string

const (
    AttemptPending   AttemptState = "pending"
    AttemptSent      AttemptState = "sent"
    AttemptDelivered AttemptState = "delivered"
    AttemptRead      AttemptState = "read"
    AttemptFailed    AttemptState = "failed"
    AttemptTimedOut  AttemptState = "timed_out"
)

// --- Adapter Registry Types ---

// AdapterRegistration represents a registered channel adapter instance.
type AdapterRegistration struct {
    AdapterID         AdapterID           `json:"adapter_id"`
    ChannelType       ChannelType         `json:"channel_type"`
    EndpointURL       string              `json:"endpoint_url"`
    Version           string              `json:"version"`
    Capabilities      []AdapterCapability `json:"capabilities"`
    InstanceTokenHash string              `json:"instance_token_hash"` // bcrypt hash
    PublicKeyID       string              `json:"public_key_id"`
    Status            AdapterStatus       `json:"status"`
    Health            AdapterHealth       `json:"health"`
    RegisteredAt      time.Time           `json:"registered_at"`
    LastHeartbeat     *time.Time          `json:"last_heartbeat,omitempty"`
}

type AdapterStatus string

const (
    AdapterActive   AdapterStatus = "active"
    AdapterInactive AdapterStatus = "inactive" // Missed heartbeats
    AdapterDisabled AdapterStatus = "disabled" // Admin disabled
    AdapterDraining AdapterStatus = "draining" // Shutting down gracefully
)

type AdapterCapability string

const (
    CapSend               AdapterCapability = "send"
    CapStatusCheck        AdapterCapability = "status_check"
    CapTemplateManagement AdapterCapability = "template_management"
    CapBulkSend           AdapterCapability = "bulk_send"
    CapScheduledSend      AdapterCapability = "scheduled_send"
    CapReadReceipts       AdapterCapability = "read_receipts"
)

// --- Reputation Types ---

// TenantHealthScore tracks a tenant's sending reputation.
type TenantHealthScore struct {
    TenantID             TenantID  `json:"tenant_id"`
    OverallScore         float64   `json:"overall_score"`          // 0.0 to 100.0
    BounceRate           float64   `json:"bounce_rate"`
    ComplaintRate        float64   `json:"complaint_rate"`
    SpamTrapHits         uint32    `json:"spam_trap_hits"`
    BlacklistAppearances uint32    `json:"blacklist_appearances"`
    ContentQualityScore  float64   `json:"content_quality_score"`  // ML-scored 0-100
    SendingPatternScore  float64   `json:"sending_pattern_score"`  // consistency 0-100
    CalculatedAt         time.Time `json:"calculated_at"`
}
```

### REST API Contract

```
POST /v1/messages/send
Content-Type: application/json
Authorization: Bearer {api_key}

Request:
{
    "idempotency_key": "txn-2024-abc-123",
    "to": {
        "type": "multi",
        "email": "user@example.com",
        "phone": "+14155551234"
    },
    "body": {
        "text": "Your order #1234 has shipped!",
        "html": "<h1>Your order #1234 has shipped!</h1>",
        "subject": "Order Shipped"
    },
    "routing": {
        "type": "cascade_rule",
        "cascade_rule_id": "550e8400-e29b-41d4-a716-446655440000"
    },
    "metadata": {
        "order_id": "1234",
        "user_id": "u-567"
    }
}

Response (202 Accepted):
{
    "message_id": "msg-7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "status": "accepted",
    "tracking_url": "https://api.hermes.dev/v1/messages/msg-7c9e6679-7425-40de-944b-e07fc1f90ae7/events",
    "created_at": "2026-02-10T12:00:00Z"
}

Response (429 Too Many Requests):
{
    "error": "rate_limit_exceeded",
    "message": "Rate limit exceeded for tenant tier Starter (100 msg/sec)",
    "retry_after": 1,
    "limit": 100,
    "remaining": 0,
    "reset_at": "2026-02-10T12:00:01Z"
}

Response (402 Payment Required):
{
    "error": "insufficient_balance",
    "message": "Prepaid balance exhausted. Add credits or upgrade to postpaid.",
    "balance": 0,
    "required": 1
}
```

---

## Migration Path

### Phase 1: MVP (Weeks 1-4)

**Goal:** Single-tenant, single-channel (Email via SendGrid) with basic cascade (Email -> SMS).

| Component | Scope |
|-----------|-------|
| API Gateway | net/http + Chi router, basic routing, API key auth |
| Validation Pipeline | Layers 1, 2, 5 only (auth, tenant check, schema validation) |
| Cascade Engine | 2-step cascade (Email -> SMS), in-memory state machine (goroutine per cascade) |
| Channel Adapters | SendGrid (email) and Twilio (SMS) as in-process adapters (not containerized) |
| Database | Single Supabase project, single schema (pgx) |
| Queue | NATS JetStream single stream |
| Observability | Basic Prometheus metrics, structured logging (slog) |

**Deliverable:** Working API that sends Email with SMS fallback.

### Phase 2: SMPP Inbound + Smart Routing + First Carrier (Weeks 5-10)

**Goal:** Add SMPP server for enterprise inbound, smart routing engine, first direct carrier SMPP connection (Telia DK).

| Component | Scope |
|-----------|-------|
| Protocol Gateway | SMPP server (gosmpp) accepting inbound from enterprise clients, normalization to unified format |
| Smart Router | HLR/MNP lookup integration, least-cost routing tables, quality scoring foundation |
| Direct Carrier | First SMPP client connection to Telia DK, connection pool manager, DLR handling |
| Tenant Registry | Full onboarding via WorkOS SSO, API key management |
| Validation Pipeline | All 9 layers operational |
| Cascade Engine | Full state machine with configurable rules |
| Multi-Tenancy | Schema-per-tenant (Starter tier) |
| Rate Limiting | Per-tenant sliding window via NATS KV |
| Billing | Basic prepaid balance tracking |

**Deliverable:** Multi-tenant platform with REST + SMPP inbound, smart routing, and first direct carrier connection alongside aggregator fallback.

### Phase 3: Own MTA + Additional Carriers + WhatsApp BSP (Weeks 11-18)

**Goal:** Own MTA for email delivery, additional Danish carrier connections, WhatsApp BSP, HLR/MNP integration.

| Component | Scope |
|-----------|-------|
| MTA Infrastructure | Self-hosted Haraka or Go MTA, IP pool management, automated warming |
| SMTP Server | Inbound SMTP for email ingestion, MIME parsing, tenant routing |
| Additional Carriers | SMPP connections to TDC/Nuuday, 3 (Three DK), Telenor DK |
| WhatsApp BSP | Direct Meta partnership, self-hosted WhatsApp Business API |
| HLR/MNP | Full number portability lookup integration for all SMS routing |
| Email Auth | DKIM signing, SPF, DMARC enforcement per tenant domain |
| ISP Relations | Gmail Postmaster, Microsoft SNDS, Yahoo FBL feedback loop processing |
| Reputation Engine | IP pool management, warming, monitoring, tenant health scoring |
| Business Tier | Dedicated Supabase project per tenant, dedicated worker pools |
| Compliance | GDPR consent management, TCPA opt-in verification |
| Billing | Postpaid billing, usage metering, invoice generation |
| Observability | Full OpenTelemetry tracing, per-tenant dashboards |

**Deliverable:** Production-ready platform with carrier-direct SMS, own email delivery, and WhatsApp BSP for Danish market.

### Phase 4: Full Carrier-Direct + International Expansion (Weeks 19-30)

**Goal:** Full carrier-direct delivery for Danish market, Apple Messages, international expansion via aggregator partnerships, scale to 100M msg/day.

| Component | Scope |
|-----------|-------|
| Full Danish Direct | All Danish carriers on direct SMPP, aggregator as fallback only |
| Apple Messages | CSP program integration for Apple Messages for Business |
| International | Aggregator partnerships (Messente, Infobip) for non-Danish destinations |
| Enterprise Tier | Self-hosted Supabase, dedicated cluster, BYOIP |
| Custom Adapters | Enterprise tenants bring their own adapter containers |
| Scale Testing | 100M msg/day load testing (~22M sustained), horizontal scaling validation |
| MTA Scale | Fleet of MTA instances, 200K emails/hour per instance, 6B emails/year capacity |
| HA/DR | Multi-AZ deployment, failover testing, backup/restore, SMPP connection failover |
| SOC 2 Prep | Audit log review, access control documentation, penetration testing |
| SDK | Go, Python, Node.js, Ruby client SDKs |
| Future Channels | RCS Business Messaging, Viber Business Messages evaluation |

**Deliverable:** Enterprise-ready platform at target scale (2B texts + 6B emails/year) with full carrier-direct delivery.

---

## Consequences

### Positive

- **Cascade delivery as competitive moat**: No competing platform offers configurable multi-channel cascade with per-tenant rules as a first-class primitive. This is the core IP.
- **Go for pragmatic velocity**: Fast compile times (2-5s), large hiring pool in Denmark, goroutines map naturally to per-message cascade state machines. Time-to-PoC cut in half vs. Rust.
- **Hexagonal architecture for adaptability**: Swapping Supabase for raw PostgreSQL, or NATS for Kafka, requires only adapter changes. Domain core is untouched. Go's implicit interface satisfaction makes this pattern natural.
- **Hot-loadable adapters for extensibility**: Adding a new channel (e.g., RCS, LINE) is deploying a container that implements the gRPC contract. No platform redeployment.
- **Tiered isolation for market breadth**: Starter tier for indie devs (instant, cheap), Business for SMBs (dedicated resources), Enterprise for regulated industries (on-premise, BYOIP).
- **IP reputation engine as infrastructure protection**: Proactive tenant health scoring prevents bad actors from damaging shared sending infrastructure before ISPs react.
- **NATS JetStream operational simplicity**: Single binary deployment, built-in KV store eliminates Redis dependency for rate limiting and idempotency.
- **Comprehensive compliance**: GDPR, TCPA, CAN-SPAM checks baked into the validation pipeline, not bolted on.
- **Future Rust optimization path**: Hexagonal architecture allows extracting hot-path adapters as Rust microservices if profiling reveals bottlenecks (e.g., content scanning, reputation scoring).
- **Carrier-direct margin advantage**: Direct SMPP connections provide 50-80% margin on SMS vs. 10-20% through aggregators. At 2B texts/year, this is the difference between a profitable business and a thin-margin reseller.
- **Own MTA cost savings**: Self-hosted MTAs save ~$550K/year compared to SES at scale (6B emails/year). Pays for an entire SRE team.
- **Full delivery control eliminates provider lock-in**: No single provider can hold Hermes hostage. Direct carrier relationships and own MTAs mean switching cost is near zero.
- **SMPP inbound captures enterprise market**: Enterprise clients who speak SMPP natively (banks, telcos, legacy notification platforms) can connect without changing their integration. No REST migration required.
- **Protocol Gateway enables market breadth**: REST for modern SaaS, SMPP for enterprise telco, SMTP for email migration. Three on-ramps to the same platform.

### Negative

- **Go lacks Rust's compile-time safety guarantees**: No borrow checker means data race prevention relies on discipline, linting (`go vet -race`), and CI enforcement rather than compiler guarantees. Mitigated by race detector in CI, `goleak` for goroutine leak detection, and `golangci-lint`.
- **Multi-tier complexity**: Three isolation models means three deployment topologies to maintain, test, and document.
- **Cascade engine state management**: Distributed state machine across NATS and PostgreSQL requires careful handling of crash recovery and exactly-once semantics.
- **Ed25519 + bcrypt for adapters adds operational burden**: Key rotation, trusted key table management, and HSM integration for Enterprise tenants are non-trivial.
- **9-layer validation pipeline adds latency**: Each layer adds microseconds to milliseconds. Must be carefully benchmarked to stay under 50ms p99 target.
- **Supabase dependency**: While the hexagonal architecture allows swapping, the tiered model (shared project, dedicated project, self-hosted) is Supabase-specific and would require significant adapter work to migrate.
- **GC pauses**: Go's garbage collector introduces occasional sub-millisecond pauses. For the Hermes workload (I/O-bound message routing, not latency-critical microsecond trading), this is acceptable. Monitor p99 tail latency.
- **SMPP protocol complexity**: Binary protocol with session management (bind/unbind), flow control (windowing), and encoding quirks (GSM 7-bit, UCS-2). Requires deep telco protocol knowledge to implement correctly.
- **MTA operational burden**: IP warming takes weeks per new IP, blacklist management is ongoing, ISP relations require human touchpoints. Deliverability is an operational discipline, not just a technical one.
- **Carrier relationship management**: Each direct carrier connection requires a contract, SLA negotiation, technical integration, and ongoing relationship management. This is a business development cost, not just an engineering cost.
- **Higher upfront infrastructure cost**: Own MTAs, SMPP connections, HLR lookups, and IP pools require infrastructure investment before reaching scale. Break-even on MTA is ~2B emails/year.

### Neutral

- **Go ecosystem maturity**: net/http, Chi, grpc-go, pgx, and nats.go are all production-grade. No experimental dependencies.
- **NATS JetStream is less widely adopted than Kafka**: Smaller community, but the Go client (nats.go) is first-party and actively maintained.
- **WorkOS for authentication**: Simplifies SSO/SAML integration but adds a third-party dependency for tenant onboarding.
- **OpenTelemetry overhead**: Tracing every message through the cascade adds ~5% CPU overhead, acceptable for the observability benefits.

---

## References

### Internal (OmniRev ADRs)

| ADR | Pattern Borrowed | Adaptation for Hermes |
|-----|-----------------|----------------------|
| [ADR-SR-001: Service Registry Architecture](../../OmniRev/omnitronix-docs/adr/ADR-SR-001-service-registry-architecture.md) | Tenant Registry design, heartbeat monitoring, allowlist validation, bcrypt token hashing | Adapted from service-level registration to tenant-level registry with tiered isolation and API key management |
| [ADR-PR-001: Plugin Registry Architecture](../../OmniRev/omnitronix-docs/adr/ADR-PR-001-plugin-registry-architecture.md) | Channel Adapter Registry design, Ed25519 signature verification, hot-loadable containers, adapter lifecycle | Adapted from operator wallet adapters to channel delivery adapters with gRPC contract instead of HTTP |
| [ADR-SR-003: 9-Layer Validation Pipeline](../../OmniRev/omnitronix-docs/adr/ADR-SR-003-debug-9layer-validation-pipeline.md) | 9-layer validation pattern, sequential fail-fast design, pre-execution audit logging, idempotency check | Adapted from debug command validation to message send validation with billing, compliance, and content scanning layers |

### External

| Resource | Relevance |
|----------|-----------|
| [OWASP Top 10 (2021)](https://owasp.org/Top10/) | Security model alignment for all 10 categories |
| [Ed25519 - RFC 8032](https://datatracker.ietf.org/doc/html/rfc8032) | Adapter registration signature scheme |
| [NATS JetStream Documentation](https://docs.nats.io/nats-concepts/jetstream) | Message queue architecture and stream topology |
| [Chi Router](https://github.com/go-chi/chi) | Lightweight, idiomatic HTTP router for Go |
| [Hexagonal Architecture (Alistair Cockburn)](https://alistair.cockburn.us/hexagonal-architecture/) | Ports & Adapters architectural pattern |
| [GDPR - Regulation (EU) 2016/679](https://gdpr-info.eu/) | Data protection compliance requirements |
| [TCPA - 47 U.S.C. 227](https://www.law.cornell.edu/uscode/text/47/227) | US telephone consumer protection (SMS compliance) |
| [CAN-SPAM Act](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business) | US email compliance requirements |
| [10DLC Registration](https://www.10dlc.org/) | US SMS brand and campaign registration |
| [SMPP 3.4 Specification](https://smpp.org/SMPP_v3_4_Issue1_2.pdf) | Short Message Peer-to-Peer protocol for carrier SMS connections |
| [gosmpp](https://github.com/linxGnu/gosmpp) | Go SMPP 3.4 library for carrier connections (server and client) |
| [Haraka](https://haraka.github.io/) | High-performance SMTP server/MTA for email delivery |
| [go-smtp](https://github.com/emersion/go-smtp) | SMTP server library for Go (inbound email) |

---

## Appendix A: Tech Stack Summary

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Language | Go 1.22+ | Systems programming, goroutine-based concurrency, fast compile times |
| HTTP Server | net/http (standard library) | HTTP/1.1 and HTTP/2 server |
| Router | Chi (go-chi/chi) | Lightweight, idiomatic URL routing and middleware composition |
| Async Concurrency | goroutines + channels | Native concurrent execution for cascade state machines and I/O |
| gRPC | grpc-go (google.golang.org/grpc) | Adapter communication (registration, send, status) |
| TLS | crypto/tls (standard library) | TLS 1.3 for all external connections |
| Serialization | encoding/json (standard library) | JSON request/response handling |
| Database | pgx (jackc/pgx) + PostgreSQL (Supabase) | High-performance PostgreSQL driver with connection pooling |
| Message Queue | nats.go (NATS JetStream) | Tenant-keyed message streams, KV store |
| Authentication | WorkOS SDK | Tenant SSO, magic link, OAuth |
| Crypto | crypto/ed25519 (stdlib), golang.org/x/crypto/bcrypt | Adapter signatures, token hashing |
| Observability | opentelemetry-go + prometheus/client_golang | Distributed tracing, metrics exposition |
| Testing | go test + benchmarks | Unit tests, integration tests, benchmarks, race detector |
| Linting | golangci-lint | Static analysis, security checks (gosec), error checking |
| SMPP | gosmpp (github.com/linxGnu/gosmpp) | SMPP 3.4 protocol for carrier connections (both server and client) |
| MTA | Haraka or custom Go MTA | Direct email delivery, IP pool management, DKIM signing |
| HLR/MNP | hlr-lookups.com or equivalent | Number portability lookups for SMS routing decisions |

## Appendix B: Service Interaction Map

```
+-----------------------------------------------------------------------+
|                     HERMES SERVICE INTERACTION MAP                      |
+-----------------------------------------------------------------------+
|                                                                        |
|                          [Internet]                                     |
|                              |                                         |
|              +---------------+---------------+                         |
|              |               |               |                         |
|              v               v               v                         |
|     +--------+------+ +-----+------+ +------+-----+                  |
|     |  REST API     | | SMPP Server| | SMTP Server|                  |
|     | (net/http+Chi)| | (gosmpp)   | | (go-smtp)  |                  |
|     +--------+------+ +-----+------+ +------+-----+                  |
|              |               |               |                         |
|              +-------+-------+-------+-------+                        |
|                      |                                                 |
|                      v                                                 |
|             +--------+--------+                                       |
|             | Protocol Gateway |                                      |
|             | (normalization)  |                                      |
|             +--------+--------+                                       |
|                      |                                                 |
|                    +---------+---------+                                |
|                    |                   |                                |
|                    v                   v                                |
|           +--------+------+   +-------+--------+                      |
|           |   Tenant      |   |   Validation   |                      |
|           |   Registry    |   |   Pipeline     |                      |
|           +--------+------+   +-------+--------+                      |
|                    |                   |                                |
|                    |          +--------+--------+                      |
|                    |          |                 |                       |
|                    |          v                 v                       |
|                    |  +-------+------+  +------+-------+               |
|                    |  |  Reputation  |  |   Billing    |               |
|                    |  |  Engine      |  |  & Metering  |               |
|                    |  +-------+------+  +------+-------+               |
|                    |          |                 |                       |
|                    |          +--------+--------+                      |
|                    |                   |                                |
|                    v                   v                                |
|           +--------+------+   +-------+--------+                      |
|           |   Tenant      |   | NATS JetStream |                      |
|           | Provisioner   |   |  (msg queue)   |                      |
|           +---------------+   +-------+--------+                      |
|                                       |                                |
|                                       v                                |
|                              +--------+--------+                      |
|                              |  Cascade Engine  |                     |
|                              | (goroutine/msg)  |                     |
|                              +--------+--------+                      |
|                                       |                                |
|                                       v                                |
|                              +--------+--------+                      |
|                              |  Smart Router    |                     |
|                              | (HLR/LCR/QoS)   |                     |
|                              +--------+--------+                      |
|                                       |                                |
|                    +--------+---------+---------+--------+             |
|                    |        |         |         |        |             |
|                    v        v         v         v        v             |
|              +-----+-+ +---+---+ +---+---+ +---+---+ +--+----+       |
|              |  Own  | | SMPP  | |WhatsApp| |Telegram| | Push |       |
|              |  MTA  | |Carrier| |  BSP   | |Bot API | |  FCM |       |
|              +---+---+ +---+---+ +---+---+ +---+---+ +---+---+       |
|                  |         |         |         |          |            |
|             [primary] [primary]  [primary]  [direct]  [direct]        |
|                  |         |         |                                 |
|             [fallback] [fallback]                                     |
|                  |         |                                          |
|              SendGrid  Aggregator                                     |
|              / SES     APIs (Messente,                                |
|                        Infobip, Twilio)                               |
|                                                                        |
+-----------------------------------------------------------------------+
```

## Appendix C: Go-Specific Engineering Practices

| Practice | Tool / Approach | Purpose |
|----------|----------------|---------|
| Race detection | `go test -race ./...` in CI | Catch data races at test time; mandatory for all CI runs |
| Goroutine leak detection | `go.uber.org/goleak` | Detect goroutine leaks in tests (critical for cascade engine) |
| Static analysis | `golangci-lint` with gosec, errcheck, staticcheck | Catch security issues, unchecked errors, and code smells |
| Vulnerability scanning | `govulncheck` | Scan dependencies for known vulnerabilities |
| Structured logging | `log/slog` (standard library) | JSON-structured logging with context propagation |
| Error handling | Explicit `error` returns, no panics in library code | All errors are returned and handled; panics only in `main` for truly unrecoverable situations |
| Context propagation | `context.Context` on all port interfaces | Cancellation, deadlines, and trace propagation through the entire request lifecycle |
| Connection pooling | `pgxpool` for PostgreSQL, `http.Transport` for outbound HTTP | Configurable per-tenant connection limits |
| Build | Multi-stage Docker build, single static binary | ~15MB binary, no runtime dependencies, fast container startup |
| Profiling | `net/http/pprof` (gated behind admin auth) | CPU and memory profiling in production when needed |
