# Project Hermes -- Architecture Diagrams

A comprehensive visual reference for the Hermes multi-tenant, multi-channel message broker platform built in Go.

---

## 1. System Overview (C4 Context)

High-level view of external actors, the Hermes platform boundary, and the third-party systems it integrates with. Enterprise tenants interact with a single API surface; Hermes handles authentication, routing, delivery, and observability behind the scenes.

```mermaid
C4Context
    title Hermes Platform -- System Context

    Person(tenant, "Enterprise Tenant", "Sends messages via REST API, SMPP, or SMTP")
    Person(admin, "Hermes Operator", "Manages platform, monitors health")

    System_Boundary(hermes, "Hermes Platform") {
        System(pgw, "Protocol Gateway", "REST, SMPP, SMTP inbound normalization")
        System(api, "API Gateway", "Go (net/http + Chi)")
        System(core, "Core Services", "Cascade, Smart Router, Reputation, Billing, Registry")
        System(carrier_infra, "Carrier-Direct Infrastructure", "SMPP Connection Manager, MTA Manager")
        System(adapters, "Channel Adapters", "Hot-loadable containers")
    }

    System_Ext(workos, "WorkOS", "Auth & Magic Link SSO")
    System_Ext(supabase, "Supabase", "Tenant data, queues, analytics")
    System_Ext(nats, "NATS JetStream", "Message bus & durable streams")

    System_Ext(carriers, "Telecom Carriers", "SMPP direct: Telia DK, TDC, 3 DK, Telenor")
    System_Ext(own_mta, "Own MTAs", "Transactional & marketing email delivery")
    System_Ext(aggregators, "Aggregators", "Messente, Infobip -- contractual fallback")
    System_Ext(provider_apis, "Provider APIs", "SendGrid, Twilio, FCM -- burst/fallback")
    System_Ext(whatsapp_bsp, "WhatsApp BSP", "Direct Meta partnership")
    System_Ext(hlr_provider, "HLR/MNP Provider", "Number lookups & carrier identification")
    System_Ext(telegram, "Telegram Bot API", "Telegram delivery")

    Rel(tenant, pgw, "SMPP bind / SMTP session", "SMPP 3.4 / SMTP")
    Rel(tenant, api, "REST / SDK calls", "HTTPS + API Key")
    Rel(admin, core, "Admin dashboard", "HTTPS + WorkOS SSO")
    Rel(pgw, api, "Normalized messages")
    Rel(api, core, "Validated requests")
    Rel(core, carrier_infra, "Route decisions")
    Rel(core, adapters, "Routed messages (fallback)")
    Rel(core, nats, "Pub / Sub / Stream")
    Rel(core, supabase, "Tenant data R/W")
    Rel(api, workos, "Auth verification")
    Rel(core, hlr_provider, "HLR/MNP lookups")

    Rel(carrier_infra, carriers, "SMPP 3.4 submit_sm/deliver_sm")
    Rel(carrier_infra, own_mta, "SMTP relay")
    Rel(adapters, aggregators, "REST API")
    Rel(adapters, provider_apis, "REST API")
    Rel(adapters, whatsapp_bsp, "Cloud API")
    Rel(adapters, telegram, "Bot API")
```

---

## 2. Main Cluster Architecture (Component Diagram)

Internal services that make up the Hermes main cluster. All inter-service communication flows through NATS JetStream for decoupling and durability. The Protocol Gateway and API Gateway are the externally exposed surfaces, accepting SMPP, SMTP, and REST traffic. The Smart Router and carrier-direct infrastructure enable lowest-cost delivery paths.

```mermaid
flowchart TB
    subgraph external["External"]
        client_rest(["Tenant Client (REST)"])
        client_smpp(["Enterprise Client (SMPP)"])
        client_smtp(["Enterprise Client (SMTP)"])
        carriers(["Telecom Carriers<br/><i>Telia DK, TDC, 3 DK, Telenor</i>"])
        aggregators(["Aggregators<br/><i>Messente, Infobip</i>"])
        providers(["Provider APIs<br/><i>SendGrid, Twilio, FCM</i>"])
        webhooks(["Provider Webhooks / DLRs"])
    end

    subgraph main_cluster["Main Cluster"]
        direction TB

        subgraph gateway_layer["Ingress"]
            pgw["Protocol Gateway<br/><i>SMPP Server (gosmpp)<br/>SMTP Server (go-smtp)</i><br/>Protocol normalization"]
            gw["API Gateway<br/><i>Go: net/http + Chi router</i><br/>TLS termination, routing,<br/>9-layer validation pipeline"]
        end

        subgraph core_services["Core Services"]
            tr["Tenant Registry<br/><i>Onboarding, WorkOS magic-link,<br/>heartbeat, tier management</i>"]
            car["Channel Adapter Registry<br/><i>Hot-loadable containers,<br/>Ed25519 auth, allowlist</i>"]
            ce["Cascade Engine<br/><i>State machine, multi-channel<br/>fallback orchestration</i><br/><b>Core IP</b>"]
            sr["Smart Router<br/><i>HLR/MNP lookup, least-cost<br/>routing, carrier selection,<br/>direct vs aggregator decision</i>"]
            re["Reputation Engine<br/><i>IP pools, warming,<br/>blacklist monitor,<br/>tenant health scoring</i>"]
        end

        subgraph carrier_services["Carrier-Direct Infrastructure"]
            smpp_mgr["SMPP Connection Manager<br/><i>Connection pools, bind mgmt,<br/>window flow control, DLR handling</i>"]
            mta_mgr["MTA Manager<br/><i>MTA pool orchestration,<br/>IP warming, DKIM/SPF/DMARC,<br/>ISP feedback loops</i>"]
        end

        subgraph support_services["Support Services"]
            bm["Billing & Metering<br/><i>Usage tracking, quota<br/>enforcement, invoicing</i>"]
            tp["Tenant Provisioner<br/><i>Supabase schema creation,<br/>namespace setup, key gen</i>"]
            obs["Observability Stack<br/><i>OpenTelemetry traces,<br/>Prometheus metrics,<br/>structured logging</i>"]
        end
    end

    subgraph messaging["Message Bus"]
        nats[("NATS JetStream<br/>Durable streams,<br/>consumer groups")]
    end

    subgraph data["Data Layer"]
        supa[("Supabase<br/>Tenant data,<br/>message logs")]
    end

    subgraph auth_layer["Auth"]
        workos["WorkOS<br/>Magic Link SSO"]
    end

    client_rest -->|"HTTPS + API Key"| gw
    client_smpp -->|"SMPP 3.4 bind"| pgw
    client_smtp -->|"SMTP session"| pgw
    pgw -->|"Normalized messages"| gw
    webhooks -->|"Delivery callbacks / DLRs"| gw

    gw <-->|"Auth check"| workos
    gw -->|"Validated request"| nats

    nats <--> ce
    nats <--> tr
    nats <--> car
    nats <--> sr
    nats <--> re
    nats <--> bm
    nats <--> tp
    nats <--> obs
    nats <--> smpp_mgr
    nats <--> mta_mgr

    ce -->|"Route decision"| sr
    sr -->|"DIRECT: carrier"| smpp_mgr
    sr -->|"DIRECT: own MTA"| mta_mgr
    sr -->|"AGGREGATOR / FALLBACK"| car
    car -->|"Dispatch"| aggregators
    car -->|"Dispatch"| providers
    smpp_mgr -->|"SMPP submit_sm"| carriers
    carriers -->|"deliver_sm (DLR)"| smpp_mgr
    smpp_mgr -->|"Forward DLR"| pgw

    ce <-->|"Score query"| re
    ce <-->|"Balance check"| bm
    tr <-->|"Provision"| tp

    tr <--> supa
    ce <--> supa
    bm <--> supa
    tp <--> supa
    re <--> supa
    sr <--> supa

    style ce fill:#2d6a4f,stroke:#1b4332,color:#fff
    style sr fill:#e76f51,stroke:#9b2226,color:#fff
    style pgw fill:#264653,stroke:#2a9d8f,color:#fff
    style gw fill:#1d3557,stroke:#0d1b2a,color:#fff
    style smpp_mgr fill:#6a040f,stroke:#e5383b,color:#fff
    style mta_mgr fill:#6a040f,stroke:#e5383b,color:#fff
    style nats fill:#6a040f,stroke:#370617,color:#fff
    style supa fill:#3a0ca3,stroke:#240046,color:#fff
```

---

## 3. Tenant Tier Model

Hermes supports three tenant tiers, each with progressively more isolation. The tier determines the Supabase topology, worker allocation, IP pool strategy, and whether custom adapters are allowed.

```mermaid
flowchart LR
    subgraph starter_tier["Starter Tier"]
        direction TB
        s_desc["Shared infrastructure<br/>Low cost, fast onboarding"]
        s_supa[("Shared Supabase Project<br/><i>schema-per-tenant isolation</i>")]
        s_workers["Shared Worker Pool"]
        s_ip["Shared IP Pool"]
        s_limits["Rate limits: standard<br/>Channels: email, SMS<br/>Support: community"]
    end

    subgraph business_tier["Business Tier"]
        direction TB
        b_desc["Dedicated data layer<br/>Higher throughput"]
        b_supa[("Dedicated Supabase Project<br/><i>full database isolation</i>")]
        b_workers["Dedicated Worker Pool"]
        b_ip["Dedicated IP Addresses"]
        b_limits["Rate limits: elevated<br/>Channels: all standard<br/>Support: priority"]
    end

    subgraph enterprise_tier["Enterprise Tier"]
        direction TB
        e_desc["Full isolation<br/>Custom everything"]
        e_supa[("On-Premise Supabase<br/><i>self-hosted or VPC</i>")]
        e_workers["Dedicated Cluster"]
        e_ip["BYOIP<br/>Bring Your Own IPs"]
        e_adapters["Custom Adapters<br/><i>tenant-provided channels</i>"]
        e_limits["Rate limits: custom<br/>Channels: all + custom<br/>Support: dedicated SLA"]
    end

    starter_tier ---|"Upgrade"| business_tier
    business_tier ---|"Upgrade"| enterprise_tier

    style starter_tier fill:#f0f4f8,stroke:#9db5c9,color:#1a1a2e
    style business_tier fill:#e8f5e9,stroke:#66bb6a,color:#1a1a2e
    style enterprise_tier fill:#fff3e0,stroke:#ffa726,color:#1a1a2e
```

---

## 4. Message Flow (Sequence Diagram)

End-to-end lifecycle of a single message: from tenant API call through the 9-layer validation pipeline, into the Cascade Engine for multi-channel orchestration, out through an adapter to a provider, and back via webhook for confirmation or fallback.

```mermaid
sequenceDiagram
    autonumber
    participant T as Tenant Client
    participant GW as API Gateway
    participant VP as Validation Pipeline
    participant NATS as NATS JetStream
    participant CE as Cascade Engine
    participant RE as Reputation Engine
    participant CAR as Channel Adapter Registry
    participant CA as Channel Adapter
    participant P as Provider (e.g. SendGrid)

    T->>+GW: POST /v1/messages (API Key + payload)
    GW->>+VP: Run 9-layer validation

    Note over VP: L0: Idempotency check
    Note over VP: L1: Auth (API key verify)
    Note over VP: L2: Tenant active?
    Note over VP: L3: Rate limit check
    Note over VP: L4: Balance / quota (CRITICAL)
    Note over VP: L5: Schema validation
    Note over VP: L6: Compliance (GDPR/TCPA)
    Note over VP: L7: Pre-flight content scan
    Note over VP: L8: Audit log write
    Note over VP: L9: Route to Cascade

    VP-->>-GW: Validation passed
    GW-->>T: 202 Accepted (message_id)
    GW->>NATS: Publish validated message

    NATS->>+CE: Deliver message (consumer group)
    CE->>CE: Create Cascade state machine
    CE->>RE: Query reputation for IP selection
    RE-->>CE: Recommended IP + throttle hints
    CE->>CAR: Resolve best adapter for channel 1
    CAR-->>CE: Adapter endpoint + instance token
    CE->>+CA: Dispatch to adapter (channel 1)
    CA->>+P: Send via provider API
    P-->>-CA: Accepted (provider_id)
    CA-->>-CE: Dispatch acknowledged

    Note over CE: State: Channel1_Waiting

    P--)GW: Webhook: delivery confirmation
    GW->>NATS: Publish delivery event
    NATS->>CE: Delivery event

    alt Delivered successfully
        CE->>CE: State -> Channel1_Delivered (terminal)
        CE->>NATS: Publish success event
    else Delivery failed or timeout
        CE->>CE: State -> Channel1_Failed
        CE->>CAR: Resolve adapter for channel 2
        CE->>CA: Dispatch to channel 2 (fallback)
        Note over CE: Repeat until delivered or AllChannelsFailed
    end

    CE->>NATS: Publish final outcome event
    NATS->>T: Webhook callback (if configured)
```

---

## 5. Cascade Engine State Machine

The Cascade Engine is the core intellectual property of Hermes. It manages a per-message state machine that orchestrates delivery across multiple channels with automatic fallback. Each channel attempt has a configurable timeout before escalation.

```mermaid
stateDiagram-v2
    [*] --> Created: Message accepted

    Created --> Channel1_Sending: Dispatch to primary channel

    Channel1_Sending --> Channel1_Waiting: Provider accepted
    Channel1_Sending --> Channel1_Failed: Dispatch error

    Channel1_Waiting --> Channel1_Delivered: Delivery confirmed
    Channel1_Waiting --> Channel1_Failed: Timeout expired
    Channel1_Waiting --> Channel1_Failed: Bounce / rejection webhook

    Channel1_Delivered --> [*]: SUCCESS

    Channel1_Failed --> Channel2_Sending: Fallback to channel 2

    Channel2_Sending --> Channel2_Waiting: Provider accepted
    Channel2_Sending --> Channel2_Failed: Dispatch error

    Channel2_Waiting --> Channel2_Delivered: Delivery confirmed
    Channel2_Waiting --> Channel2_Failed: Timeout expired
    Channel2_Waiting --> Channel2_Failed: Bounce / rejection webhook

    Channel2_Delivered --> [*]: SUCCESS

    Channel2_Failed --> Channel3_Sending: Fallback to channel 3

    Channel3_Sending --> Channel3_Waiting: Provider accepted
    Channel3_Sending --> Channel3_Failed: Dispatch error

    Channel3_Waiting --> Channel3_Delivered: Delivery confirmed
    Channel3_Waiting --> Channel3_Failed: Timeout expired
    Channel3_Waiting --> Channel3_Failed: Bounce / rejection webhook

    Channel3_Delivered --> [*]: SUCCESS

    Channel3_Failed --> AllChannelsFailed: No more channels

    AllChannelsFailed --> [*]: FAILURE -- notify tenant

    note right of Channel1_Waiting
        Timeout is configurable per channel.
        Default: email 300s, SMS 60s,
        push 30s, WhatsApp 120s
    end note

    note right of AllChannelsFailed
        Dead-letter queue entry created.
        Tenant notified via webhook
        and dashboard alert.
    end note
```

---

## 6. Tenant Onboarding Flow

The complete provisioning sequence when a new tenant signs up. Authentication is handled by WorkOS magic links (passwordless). Once verified, the Tenant Provisioner creates the appropriate Supabase resources based on the selected tier.

```mermaid
sequenceDiagram
    autonumber
    participant T as Tenant (Browser)
    participant W as Hermes Website
    participant WO as WorkOS
    participant TR as Tenant Registry
    participant TP as Tenant Provisioner
    participant SB as Supabase
    participant KG as Key Generator

    T->>W: Sign up (email + org name + tier)
    W->>WO: Create magic link session
    WO-->>T: Send magic link email

    T->>WO: Click magic link
    WO->>WO: Verify email ownership
    WO-->>W: Auth callback (id_token + org_id)
    W->>TR: Create tenant record (org_id, tier, email)

    TR->>TR: Validate org uniqueness
    TR->>TP: Provision tenant infrastructure

    alt Starter Tier
        TP->>SB: Create schema in shared project
        SB-->>TP: Schema ready (connection string)
    else Business Tier
        TP->>SB: Create dedicated project
        SB-->>TP: Project ready (host, keys)
    else Enterprise Tier
        TP->>TP: Queue for manual VPC setup
        TP-->>TR: Provisioning pending (manual step)
    end

    TP->>KG: Generate API key pair
    KG-->>TP: api_key (public) + api_secret (hashed)
    TP->>SB: Store tenant config + hashed secret

    TP-->>TR: Provisioning complete
    TR-->>W: Tenant ready (tenant_id, api_key)

    W-->>T: Redirect to dashboard
    T->>W: View dashboard (API key, quick start guide)

    Note over T,W: Tenant can now send first message via API
```

---

## 7. Channel Adapter Registration

Channel adapters are hot-loadable containers that implement the Hermes adapter protocol. They register via Ed25519 signature verification and maintain liveness through heartbeats. The registry can signal graceful shutdown via the heartbeat response.

```mermaid
sequenceDiagram
    autonumber
    participant AC as Adapter Container
    participant CAR as Channel Adapter Registry
    participant AL as Allowlist Store
    participant TS as Token Store

    Note over AC: Container starts with Ed25519 keypair

    AC->>CAR: POST /adapters/register<br/>{public_key, channel_type, capabilities,<br/>signature(payload, private_key)}

    CAR->>CAR: Verify Ed25519 signature
    alt Signature invalid
        CAR-->>AC: 401 Unauthorized
    end

    CAR->>AL: Check public_key against allowlist
    alt Not in allowlist
        CAR-->>AC: 403 Forbidden (not allowlisted)
    end

    CAR->>CAR: Generate instance token (256-bit random)
    CAR->>CAR: bcrypt hash instance token
    CAR->>TS: Store {adapter_id, public_key,<br/>bcrypt_hash, channel_type,<br/>capabilities, registered_at}

    CAR-->>AC: 200 OK<br/>{adapter_id, instance_token,<br/>heartbeat_interval: 30s}

    Note over AC,CAR: Heartbeat loop begins

    loop Every heartbeat_interval
        AC->>CAR: POST /adapters/{id}/heartbeat<br/>Authorization: Bearer {instance_token}<br/>{status, messages_processed, error_rate}

        CAR->>CAR: Verify token (bcrypt compare)
        CAR->>CAR: Update adapter health metrics

        alt Normal operation
            CAR-->>AC: 200 OK {shouldShutdown: false}
        else Graceful shutdown requested
            CAR-->>AC: 200 OK {shouldShutdown: true,<br/>drainTimeoutMs: 30000}
            AC->>AC: Drain in-flight messages
            AC->>CAR: POST /adapters/{id}/deregister
            CAR->>TS: Mark adapter inactive
            CAR-->>AC: 200 OK (deregistered)
        end
    end

    Note over CAR: If 3 heartbeats missed:<br/>mark adapter unhealthy,<br/>stop routing traffic
```

---

## 8. IP Reputation Engine

The Reputation Engine manages IP health across the platform. It combines warming schedules, blacklist monitoring, ISP feedback loops, and per-tenant health scores to make real-time routing decisions about which IP to use for each message.

```mermaid
flowchart TB
    subgraph inputs["Inputs"]
        isp_fbl["ISP Feedback Loops<br/><i>Complaint reports from<br/>Gmail, Yahoo, Outlook</i>"]
        bl_feeds["Blacklist Feeds<br/><i>Spamhaus, Barracuda,<br/>SURBL, URIBL</i>"]
        webhooks["Provider Webhooks<br/><i>Bounces, deferrals,<br/>spam reports</i>"]
        sending_logs["Sending Logs<br/><i>Volume, cadence,<br/>recipient domains</i>"]
    end

    subgraph reputation_engine["Reputation Engine"]
        direction TB

        subgraph processors["Event Processors"]
            fbl_proc["ISP Feedback Loop<br/>Processor<br/><i>Parse ARF reports,<br/>update complaint rates</i>"]
            bl_mon["Blacklist Monitor<br/><i>Periodic checks against<br/>50+ blacklists per IP</i>"]
        end

        subgraph scoring["Scoring Layer"]
            ip_scorer["IP Health Scorer<br/><i>Per-IP reputation<br/>0-100 score</i>"]
            tenant_scorer["Tenant Health Scorer<br/><i>Per-tenant sending<br/>reputation</i>"]
            preflight["Pre-flight Scanner<br/><i>Content heuristics,<br/>URL checks, headers</i>"]
        end

        subgraph management["Pool Management"]
            pool_mgr["IP Pool Manager<br/><i>Assignment, rotation,<br/>isolation by tier</i>"]
            warming["Warming Engine<br/><i>Graduated volume ramps<br/>per ISP per IP</i>"]
        end
    end

    subgraph decisions["Routing Decisions"]
        select_ip{"Select IP<br/>for message"}
        throttle{"Apply<br/>throttle?"}
        quarantine{"Quarantine<br/>tenant?"}
    end

    subgraph outcomes["Outcomes"]
        route_ok["Route via selected IP<br/>at calculated rate"]
        route_warm["Route via warming IP<br/>with volume cap"]
        route_deny["Reject: IP unavailable<br/>or tenant quarantined"]
    end

    isp_fbl --> fbl_proc
    bl_feeds --> bl_mon
    webhooks --> ip_scorer
    webhooks --> tenant_scorer
    sending_logs --> warming

    fbl_proc --> ip_scorer
    fbl_proc --> tenant_scorer
    bl_mon --> ip_scorer
    bl_mon --> pool_mgr

    ip_scorer --> select_ip
    tenant_scorer --> select_ip
    tenant_scorer --> quarantine
    preflight --> select_ip
    pool_mgr --> select_ip
    warming --> throttle

    select_ip -->|"Healthy IP found"| throttle
    select_ip -->|"No healthy IP"| route_deny
    throttle -->|"Under warm-up cap"| route_warm
    throttle -->|"Fully warmed"| route_ok
    quarantine -->|"Tenant quarantined"| route_deny

    style reputation_engine fill:#1a1a2e,stroke:#e94560,color:#fff
    style select_ip fill:#0f3460,stroke:#e94560,color:#fff
    style route_ok fill:#2d6a4f,stroke:#1b4332,color:#fff
    style route_deny fill:#9b2226,stroke:#641220,color:#fff
    style route_warm fill:#e76f51,stroke:#9b2226,color:#fff
```

---

## 9. Nine-Layer Validation Pipeline

Every inbound API request passes through a strict 9-layer pipeline before reaching the Cascade Engine. Each layer can reject the request with a specific error code. The pipeline is designed to fail fast: cheap checks run first, expensive checks run last.

```mermaid
flowchart TD
    req(["Inbound API Request"]) --> L0

    L0["<b>Layer 0: Idempotency</b><br/>Check Idempotency-Key header.<br/>If seen before, return cached response."]
    L0 -->|"New request"| L1
    L0 -->|"Duplicate"| R0["409 Conflict<br/>Return cached response"]

    L1["<b>Layer 1: Authentication</b><br/>Verify API key signature.<br/>Resolve tenant_id from key."]
    L1 -->|"Valid"| L2
    L1 -->|"Invalid"| R1["401 Unauthorized"]

    L2["<b>Layer 2: Tenant Active</b><br/>Check tenant status in registry.<br/>Verify not suspended/deleted."]
    L2 -->|"Active"| L3
    L2 -->|"Inactive"| R2["403 Forbidden<br/>Tenant suspended"]

    L3["<b>Layer 3: Rate Limit</b><br/>Token bucket per tenant per endpoint.<br/>Check burst and sustained limits."]
    L3 -->|"Within limit"| L4
    L3 -->|"Exceeded"| R3["429 Too Many Requests<br/>Retry-After header"]

    L4["<b>Layer 4: Balance & Quota</b><br/><i>CRITICAL LAYER</i><br/>Verify prepaid balance or<br/>monthly quota not exhausted."]
    L4 -->|"Sufficient"| L5
    L4 -->|"Exhausted"| R4["402 Payment Required"]

    L5["<b>Layer 5: Schema Validation</b><br/>Validate JSON body against<br/>channel-specific schemas."]
    L5 -->|"Valid"| L6
    L5 -->|"Invalid"| R5["422 Unprocessable Entity<br/>Validation errors"]

    L6["<b>Layer 6: Compliance</b><br/>GDPR consent verification.<br/>TCPA opt-in for SMS.<br/>Suppression list check."]
    L6 -->|"Compliant"| L7
    L6 -->|"Non-compliant"| R6["451 Unavailable for<br/>Legal Reasons"]

    L7["<b>Layer 7: Pre-flight Content Scan</b><br/>Spam heuristics, phishing URL check,<br/>prohibited content patterns."]
    L7 -->|"Clean"| L8
    L7 -->|"Flagged"| R7["400 Bad Request<br/>Content policy violation"]

    L8["<b>Layer 8: Audit Log</b><br/>Write immutable audit entry.<br/>tenant_id, timestamp, action,<br/>request fingerprint."]
    L8 --> L9

    L9["<b>Layer 9: Route to Cascade</b><br/>Publish validated message<br/>to NATS JetStream."]
    L9 --> accepted(["202 Accepted<br/>{message_id}"])

    style L0 fill:#264653,stroke:#2a9d8f,color:#fff
    style L1 fill:#264653,stroke:#2a9d8f,color:#fff
    style L2 fill:#264653,stroke:#2a9d8f,color:#fff
    style L3 fill:#264653,stroke:#2a9d8f,color:#fff
    style L4 fill:#6a040f,stroke:#e5383b,color:#fff
    style L5 fill:#264653,stroke:#2a9d8f,color:#fff
    style L6 fill:#264653,stroke:#2a9d8f,color:#fff
    style L7 fill:#264653,stroke:#2a9d8f,color:#fff
    style L8 fill:#264653,stroke:#2a9d8f,color:#fff
    style L9 fill:#2d6a4f,stroke:#52b788,color:#fff

    style R0 fill:#6c757d,stroke:#495057,color:#fff
    style R1 fill:#9b2226,stroke:#641220,color:#fff
    style R2 fill:#9b2226,stroke:#641220,color:#fff
    style R3 fill:#e76f51,stroke:#9b2226,color:#fff
    style R4 fill:#9b2226,stroke:#641220,color:#fff
    style R5 fill:#9b2226,stroke:#641220,color:#fff
    style R6 fill:#9b2226,stroke:#641220,color:#fff
    style R7 fill:#9b2226,stroke:#641220,color:#fff
    style accepted fill:#2d6a4f,stroke:#52b788,color:#fff
```

---

## 10. Deployment Architecture

Hermes runs as Docker Compose in development and Kubernetes in production. The production layout uses a dedicated namespace for platform services, per-tenant namespaces for Enterprise-tier isolation, and auto-scaled adapter pods.

### Development (Docker Compose)

```mermaid
flowchart TB
    subgraph docker_compose["Docker Compose (dev)"]
        direction TB

        subgraph app_services["Application Services"]
            gw_dev["api-gateway<br/><i>:8080</i>"]
            tr_dev["tenant-registry<br/><i>:8081</i>"]
            ce_dev["cascade-engine<br/><i>:8082</i>"]
            re_dev["reputation-engine<br/><i>:8083</i>"]
            bm_dev["billing-metering<br/><i>:8084</i>"]
            car_dev["adapter-registry<br/><i>:8085</i>"]
        end

        subgraph adapters_dev["Channel Adapters"]
            email_dev["adapter-email<br/><i>:9001</i>"]
            sms_dev["adapter-sms<br/><i>:9002</i>"]
            push_dev["adapter-push<br/><i>:9003</i>"]
        end

        subgraph infra_dev["Infrastructure"]
            nats_dev["nats<br/><i>:4222 (client)</i><br/><i>:8222 (monitor)</i>"]
            supa_dev["supabase-db<br/><i>:5432</i>"]
            supa_studio["supabase-studio<br/><i>:3000</i>"]
            prom_dev["prometheus<br/><i>:9090</i>"]
            grafana_dev["grafana<br/><i>:3001</i>"]
            jaeger_dev["jaeger<br/><i>:16686</i>"]
        end
    end

    gw_dev --> nats_dev
    tr_dev --> nats_dev
    ce_dev --> nats_dev
    re_dev --> nats_dev
    bm_dev --> nats_dev
    car_dev --> nats_dev

    ce_dev --> car_dev
    car_dev --> email_dev
    car_dev --> sms_dev
    car_dev --> push_dev

    tr_dev --> supa_dev
    ce_dev --> supa_dev
    bm_dev --> supa_dev
    re_dev --> supa_dev

    prom_dev -.->|"scrape"| gw_dev
    prom_dev -.->|"scrape"| ce_dev
    prom_dev -.->|"scrape"| re_dev
    grafana_dev --> prom_dev

    style docker_compose fill:#0d1117,stroke:#30363d,color:#c9d1d9
    style app_services fill:#161b22,stroke:#1f6feb,color:#c9d1d9
    style adapters_dev fill:#161b22,stroke:#3fb950,color:#c9d1d9
    style infra_dev fill:#161b22,stroke:#f78166,color:#c9d1d9
```

### Production (Kubernetes)

```mermaid
flowchart TB
    subgraph k8s_cluster["Kubernetes Cluster"]
        direction TB

        subgraph ingress_ns["Namespace: ingress"]
            lb["Cloud Load Balancer"]
            ing["Ingress Controller<br/><i>nginx / envoy</i>"]
        end

        subgraph hermes_ns["Namespace: hermes-platform"]
            direction TB
            gw_k8s["api-gateway<br/><i>Deployment (3 replicas)</i>"]
            tr_k8s["tenant-registry<br/><i>Deployment (2 replicas)</i>"]
            ce_k8s["cascade-engine<br/><i>Deployment (5 replicas)</i><br/><i>HPA: CPU + queue depth</i>"]
            re_k8s["reputation-engine<br/><i>Deployment (2 replicas)</i>"]
            bm_k8s["billing-metering<br/><i>Deployment (2 replicas)</i>"]
            car_k8s["adapter-registry<br/><i>Deployment (2 replicas)</i>"]
            tp_k8s["tenant-provisioner<br/><i>Deployment (1 replica)</i>"]
        end

        subgraph adapters_ns["Namespace: hermes-adapters"]
            email_k8s["adapter-email<br/><i>Deployment (HPA 3-20)</i>"]
            sms_k8s["adapter-sms<br/><i>Deployment (HPA 2-10)</i>"]
            whatsapp_k8s["adapter-whatsapp<br/><i>Deployment (HPA 2-10)</i>"]
            telegram_k8s["adapter-telegram<br/><i>Deployment (HPA 1-5)</i>"]
            push_k8s["adapter-push<br/><i>Deployment (HPA 2-10)</i>"]
        end

        subgraph nats_ns["Namespace: nats"]
            nats_k8s["NATS JetStream Cluster<br/><i>StatefulSet (3 replicas)</i><br/><i>PVC: 100Gi per node</i>"]
        end

        subgraph obs_ns["Namespace: observability"]
            otel["OTel Collector<br/><i>DaemonSet</i>"]
            prom_k8s["Prometheus<br/><i>StatefulSet</i>"]
            grafana_k8s["Grafana"]
            tempo["Tempo<br/><i>Distributed traces</i>"]
            loki["Loki<br/><i>Log aggregation</i>"]
        end

        subgraph ent_ns["Namespace: tenant-{id} (Enterprise)"]
            ent_workers["Dedicated Workers<br/><i>Isolated pod pool</i>"]
            ent_adapters["Custom Adapters<br/><i>Tenant-provided images</i>"]
            ent_supa["Supabase Proxy<br/><i>-> On-premise instance</i>"]
        end
    end

    subgraph external_k8s["External Services"]
        supa_shared["Supabase Cloud<br/><i>Starter + Business tiers</i>"]
        supa_ent["On-Premise Supabase<br/><i>Enterprise tier</i>"]
        workos_k8s["WorkOS"]
    end

    lb --> ing
    ing --> gw_k8s
    gw_k8s --> nats_k8s
    ce_k8s --> nats_k8s
    tr_k8s --> nats_k8s
    re_k8s --> nats_k8s
    bm_k8s --> nats_k8s
    car_k8s --> nats_k8s

    car_k8s --> email_k8s
    car_k8s --> sms_k8s
    car_k8s --> whatsapp_k8s
    car_k8s --> telegram_k8s
    car_k8s --> push_k8s
    car_k8s -.->|"Enterprise"| ent_adapters

    tr_k8s --> supa_shared
    ce_k8s --> supa_shared
    bm_k8s --> supa_shared
    ent_workers --> ent_supa
    ent_supa --> supa_ent
    gw_k8s --> workos_k8s

    otel -.->|"collect"| hermes_ns
    otel -.->|"collect"| adapters_ns
    otel --> tempo
    otel --> loki
    prom_k8s -.->|"scrape"| hermes_ns
    prom_k8s -.->|"scrape"| adapters_ns
    grafana_k8s --> prom_k8s
    grafana_k8s --> tempo
    grafana_k8s --> loki

    style k8s_cluster fill:#0d1117,stroke:#30363d,color:#c9d1d9
    style hermes_ns fill:#161b22,stroke:#1f6feb,color:#c9d1d9
    style adapters_ns fill:#161b22,stroke:#3fb950,color:#c9d1d9
    style nats_ns fill:#161b22,stroke:#da3633,color:#c9d1d9
    style obs_ns fill:#161b22,stroke:#d29922,color:#c9d1d9
    style ent_ns fill:#161b22,stroke:#f78166,color:#c9d1d9
    style ingress_ns fill:#161b22,stroke:#8b949e,color:#c9d1d9
```

---

## 11. Protocol Gateway Layer

The Protocol Gateway normalizes three distinct inbound protocols into a single unified internal message format. REST requests arrive as JSON, SMPP traffic arrives as binary PDUs (Protocol Data Units), and SMTP traffic arrives as MIME-encoded messages. The gateway handles protocol-specific parsing and credential validation before producing a canonical message structure that the rest of the pipeline consumes identically regardless of origin.

```mermaid
flowchart TB
    subgraph inbound_protocols["Inbound Protocols"]
        rest["REST API Client<br/><i>JSON over HTTPS</i>"]
        smpp_in["SMPP Client<br/><i>Binary PDU over TCP</i><br/><i>Enterprise integration</i>"]
        smtp_in["SMTP Client<br/><i>MIME messages over TCP</i><br/><i>Legacy email relay</i>"]
    end

    subgraph protocol_gateway["Protocol Gateway"]
        direction TB

        subgraph parsers["Protocol Parsers"]
            rest_parser["REST JSON Decoder<br/><i>net/http + Chi router</i><br/><i>JSON schema validation</i>"]
            smpp_parser["SMPP PDU Decoder<br/><i>gosmpp library</i><br/><i>submit_sm parsing,<br/>TLV extraction,<br/>charset decode (GSM7/UCS2)</i>"]
            smtp_parser["SMTP MIME Parser<br/><i>go-smtp server</i><br/><i>MIME multipart parse,<br/>attachment extraction,<br/>header normalization</i>"]
        end

        subgraph auth_layer_pgw["Credential Validation"]
            rest_auth["API Key<br/>Verification"]
            smpp_auth["SMPP Bind<br/>Credentials<br/><i>system_id + password</i>"]
            smtp_auth["SMTP AUTH<br/><i>PLAIN / LOGIN</i>"]
        end

        normalizer["Message Normalizer<br/><i>Maps protocol-specific fields<br/>to unified internal format</i>"]

        unified["Unified Message Format<br/><i>tenant_id, channel, recipients,<br/>content, metadata, priority,<br/>cascade_config, origin_protocol</i>"]
    end

    subgraph downstream["Downstream"]
        validation["Validation Pipeline<br/><i>9 layers (L0-L9)</i>"]
    end

    rest --> rest_parser
    smpp_in --> smpp_parser
    smtp_in --> smtp_parser

    rest_parser --> rest_auth
    smpp_parser --> smpp_auth
    smtp_parser --> smtp_auth

    rest_auth --> normalizer
    smpp_auth --> normalizer
    smtp_auth --> normalizer

    normalizer --> unified
    unified --> validation

    style protocol_gateway fill:#1a1a2e,stroke:#2a9d8f,color:#fff
    style parsers fill:#264653,stroke:#2a9d8f,color:#fff
    style auth_layer_pgw fill:#264653,stroke:#2a9d8f,color:#fff
    style normalizer fill:#2d6a4f,stroke:#1b4332,color:#fff
    style unified fill:#0f3460,stroke:#e94560,color:#fff
    style rest fill:#1d3557,stroke:#0d1b2a,color:#fff
    style smpp_in fill:#6a040f,stroke:#e5383b,color:#fff
    style smtp_in fill:#3a0ca3,stroke:#240046,color:#fff
    style validation fill:#2d6a4f,stroke:#52b788,color:#fff
```

---

## 12. Smart Routing Engine

The Smart Router sits between the Cascade Engine and the delivery infrastructure. When the Cascade Engine determines which channel to use, the Smart Router decides the optimal delivery path: direct carrier (cheapest), aggregator (medium cost), or retail provider API (highest cost, highest reliability). For SMS, it performs HLR/MNP lookups to identify the subscriber's home carrier. For email, it evaluates MTA pool health. For WhatsApp, it checks BSP availability.

```mermaid
flowchart TB
    entry(["Message from<br/>Cascade Engine"])

    channel_check{"Determine<br/>Channel Type"}

    subgraph sms_routing["SMS Routing Path"]
        direction TB
        hlr["HLR/MNP Lookup<br/><i>Identify carrier &<br/>porting status</i>"]
        carrier_id{"Carrier<br/>Identified?"}
        direct_check{"Direct Route<br/>Available?"}
        capacity_check{"Carrier<br/>Capacity OK?"}
        lcr["Least-Cost Routing<br/><i>Compare: direct vs<br/>aggregator vs retail</i>"]

        sms_direct["DIRECT<br/><i>Own SMPP to carrier</i><br/>Cost: lowest"]
        sms_aggregator["AGGREGATOR<br/><i>Messente / Infobip</i><br/>Cost: medium"]
        sms_fallback["FALLBACK<br/><i>Twilio API</i><br/>Cost: highest"]
    end

    subgraph email_routing["Email Routing Path"]
        direction TB
        mta_health["Check MTA Pool<br/>Health & Capacity"]
        mta_select{"Pool<br/>Available?"}
        email_type{"Message<br/>Type?"}

        email_transact["OWN MTA Pool A<br/><i>Transactional IPs</i><br/>Cost: lowest"]
        email_market["OWN MTA Pool B<br/><i>Marketing IPs</i><br/>Cost: lowest"]
        email_fallback["FALLBACK<br/><i>SES / SendGrid</i><br/>Cost: medium"]
    end

    subgraph whatsapp_routing["WhatsApp Routing Path"]
        direction TB
        bsp_check{"BSP<br/>Available?"}
        wa_direct["DIRECT<br/><i>Meta Cloud API (BSP)</i><br/>Cost: lowest"]
        wa_fallback["FALLBACK<br/><i>Twilio WhatsApp API</i><br/>Cost: higher"]
    end

    entry --> channel_check

    channel_check -->|"SMS"| hlr
    channel_check -->|"Email"| mta_health
    channel_check -->|"WhatsApp"| bsp_check

    hlr --> carrier_id
    carrier_id -->|"Yes"| direct_check
    carrier_id -->|"No (unknown)"| sms_aggregator
    direct_check -->|"Yes"| capacity_check
    direct_check -->|"No"| sms_aggregator
    capacity_check -->|"Yes"| lcr
    capacity_check -->|"No (congested)"| sms_aggregator
    lcr -->|"Direct cheapest"| sms_direct
    lcr -->|"Aggregator cheaper"| sms_aggregator
    sms_aggregator -->|"Aggregator down"| sms_fallback

    mta_health --> mta_select
    mta_select -->|"Yes"| email_type
    mta_select -->|"No (all pools degraded)"| email_fallback
    email_type -->|"Transactional"| email_transact
    email_type -->|"Marketing"| email_market

    bsp_check -->|"Yes"| wa_direct
    bsp_check -->|"No"| wa_fallback

    style channel_check fill:#0f3460,stroke:#e94560,color:#fff
    style lcr fill:#e76f51,stroke:#9b2226,color:#fff
    style sms_direct fill:#2d6a4f,stroke:#1b4332,color:#fff
    style sms_aggregator fill:#e76f51,stroke:#9b2226,color:#fff
    style sms_fallback fill:#9b2226,stroke:#641220,color:#fff
    style email_transact fill:#2d6a4f,stroke:#1b4332,color:#fff
    style email_market fill:#2d6a4f,stroke:#1b4332,color:#fff
    style email_fallback fill:#e76f51,stroke:#9b2226,color:#fff
    style wa_direct fill:#2d6a4f,stroke:#1b4332,color:#fff
    style wa_fallback fill:#e76f51,stroke:#9b2226,color:#fff
    style hlr fill:#264653,stroke:#2a9d8f,color:#fff
```

---

## 13. Carrier-Direct SMS Infrastructure

Detailed view of the SMPP infrastructure that enables direct carrier connections. The SMPP Connection Pool Manager maintains persistent transceiver binds to each carrier, manages sliding window flow control for throughput optimization, and handles delivery receipt (DLR) processing. When a carrier connection degrades, traffic automatically fails over to the aggregator path.

```mermaid
flowchart TB
    subgraph inbound["From Smart Router"]
        route_decision(["Route: DIRECT SMS<br/><i>carrier identified</i>"])
    end

    subgraph smpp_infra["SMPP Connection Pool Manager"]
        direction TB

        pool_mgr["Connection Pool Manager<br/><i>Health monitoring, bind recovery,<br/>load balancing across connections</i>"]

        subgraph carrier_connections["Carrier SMPP Connections"]
            direction LR

            subgraph telia["Telia DK"]
                telia_conn["Transceiver Bind<br/><i>Window: 50</i><br/><i>TPS limit: 100</i><br/><i>2 connections</i>"]
                telia_health["Health: OK<br/><i>Latency: 12ms</i><br/><i>DLR rate: 98.7%</i>"]
            end

            subgraph tdc["TDC"]
                tdc_conn["Transceiver Bind<br/><i>Window: 30</i><br/><i>TPS limit: 80</i><br/><i>2 connections</i>"]
                tdc_health["Health: OK<br/><i>Latency: 15ms</i><br/><i>DLR rate: 97.2%</i>"]
            end

            subgraph three_dk["3 DK"]
                three_conn["Transceiver Bind<br/><i>Window: 20</i><br/><i>TPS limit: 50</i><br/><i>1 connection</i>"]
                three_health["Health: DEGRADED<br/><i>Latency: 180ms</i><br/><i>DLR rate: 91.5%</i>"]
            end

            subgraph telenor["Telenor DK"]
                telenor_conn["Transmitter Bind<br/><i>Window: 25</i><br/><i>TPS limit: 60</i><br/><i>1 connection</i>"]
                telenor_health["Health: OK<br/><i>Latency: 18ms</i><br/><i>DLR rate: 96.8%</i>"]
            end
        end

        subgraph window_mgmt["Sliding Window Flow Control"]
            window["Window Manager<br/><i>Track in-flight PDUs per connection</i><br/><i>Back-pressure when window full</i><br/><i>Retry on timeout (enquire_link)</i>"]
        end

        subgraph dlr_processing["DLR Processing"]
            dlr_handler["DLR Handler<br/><i>Parse deliver_sm receipts</i><br/><i>Map message_id to internal ID</i><br/><i>Update cascade state</i>"]
        end
    end

    subgraph fallback_path["Fallback Path"]
        aggregator_fb["Aggregator Fallback<br/><i>Messente / Infobip</i><br/><i>Activated when carrier<br/>connection is DOWN or<br/>error rate > threshold</i>"]
    end

    subgraph outbound["Carrier Networks"]
        telia_net(["Telia DK Network"])
        tdc_net(["TDC Network"])
        three_net(["3 DK Network"])
        telenor_net(["Telenor DK Network"])
    end

    route_decision --> pool_mgr
    pool_mgr --> telia_conn
    pool_mgr --> tdc_conn
    pool_mgr --> three_conn
    pool_mgr --> telenor_conn
    pool_mgr --> window

    telia_conn -->|"submit_sm"| telia_net
    tdc_conn -->|"submit_sm"| tdc_net
    three_conn -->|"submit_sm"| three_net
    telenor_conn -->|"submit_sm"| telenor_net

    telia_net -->|"deliver_sm (DLR)"| dlr_handler
    tdc_net -->|"deliver_sm (DLR)"| dlr_handler
    three_net -->|"deliver_sm (DLR)"| dlr_handler
    telenor_net -->|"deliver_sm (DLR)"| dlr_handler

    three_conn -.->|"Connection degraded"| aggregator_fb

    style smpp_infra fill:#1a1a2e,stroke:#e5383b,color:#fff
    style pool_mgr fill:#6a040f,stroke:#e5383b,color:#fff
    style telia fill:#264653,stroke:#2a9d8f,color:#fff
    style tdc fill:#264653,stroke:#2a9d8f,color:#fff
    style three_dk fill:#9b2226,stroke:#641220,color:#fff
    style telenor fill:#264653,stroke:#2a9d8f,color:#fff
    style window_mgmt fill:#0f3460,stroke:#e94560,color:#fff
    style dlr_processing fill:#2d6a4f,stroke:#1b4332,color:#fff
    style aggregator_fb fill:#e76f51,stroke:#9b2226,color:#fff
```

---

## 14. Email MTA Infrastructure

The MTA infrastructure separates transactional and marketing email traffic into isolated IP pools to protect sender reputation. Each pool manages its own IP warming schedules, DKIM signing keys (per-tenant), and SPF/DMARC alignment. ISP feedback loops feed back into the Reputation Engine. When pool capacity is exceeded or health degrades, traffic overflows to SES or SendGrid as elastic burst capacity.

```mermaid
flowchart TB
    subgraph inbound_email["From Smart Router"]
        route_email(["Route: OWN MTA<br/><i>email delivery</i>"])
    end

    subgraph mta_infra["MTA Infrastructure"]
        direction TB

        mta_orchestrator["MTA Manager<br/><i>Pool selection, health monitoring,<br/>capacity planning, overflow routing</i>"]

        subgraph pool_a["MTA Pool A -- Transactional"]
            mta_a1["MTA Instance A1<br/><i>Postfix/Haraka</i>"]
            mta_a2["MTA Instance A2<br/><i>Postfix/Haraka</i>"]
            ip_a["Dedicated IPs<br/><i>198.51.100.10-15</i><br/><i>Fully warmed</i><br/><i>Reputation: 98/100</i>"]
        end

        subgraph pool_b["MTA Pool B -- Marketing"]
            mta_b1["MTA Instance B1<br/><i>Postfix/Haraka</i>"]
            mta_b2["MTA Instance B2<br/><i>Postfix/Haraka</i>"]
            ip_b["Dedicated IPs<br/><i>198.51.100.20-30</i><br/><i>Warming: week 6/8</i><br/><i>Reputation: 91/100</i>"]
        end

        subgraph ip_management["IP Pool Manager"]
            dkim["DKIM Signing<br/><i>Per-tenant keys<br/>(2048-bit RSA)</i>"]
            spf["SPF Alignment<br/><i>Include records for<br/>all active IPs</i>"]
            dmarc["DMARC Enforcement<br/><i>p=reject for tenants<br/>with mature setup</i>"]
            warming_sched["Warming Schedules<br/><i>ISP-specific volume<br/>ramp plans</i>"]
            fbl_proc["ISP Feedback Loops<br/><i>Gmail Postmaster, Yahoo CFL,<br/>Microsoft SNDS/JMRP</i>"]
        end
    end

    subgraph fallback_email["Elastic Burst / Fallback"]
        ses["Amazon SES<br/><i>API-based sending</i><br/><i>Burst capacity</i>"]
        sendgrid["SendGrid<br/><i>API-based sending</i><br/><i>Secondary burst</i>"]
    end

    subgraph recipients_dest["Recipient ISPs"]
        gmail(["Gmail"])
        outlook(["Outlook"])
        yahoo(["Yahoo"])
        other_isp(["Other ISPs"])
    end

    route_email --> mta_orchestrator
    mta_orchestrator -->|"Transactional"| mta_a1
    mta_orchestrator -->|"Transactional"| mta_a2
    mta_orchestrator -->|"Marketing"| mta_b1
    mta_orchestrator -->|"Marketing"| mta_b2
    mta_orchestrator -->|"Overflow / degraded"| ses
    mta_orchestrator -->|"Overflow / degraded"| sendgrid

    mta_a1 --> dkim
    mta_a2 --> dkim
    mta_b1 --> dkim
    mta_b2 --> dkim

    dkim --> gmail
    dkim --> outlook
    dkim --> yahoo
    dkim --> other_isp

    gmail -->|"FBL reports"| fbl_proc
    outlook -->|"SNDS/JMRP"| fbl_proc
    yahoo -->|"CFL reports"| fbl_proc

    style mta_infra fill:#1a1a2e,stroke:#e5383b,color:#fff
    style mta_orchestrator fill:#6a040f,stroke:#e5383b,color:#fff
    style pool_a fill:#2d6a4f,stroke:#1b4332,color:#fff
    style pool_b fill:#e76f51,stroke:#9b2226,color:#fff
    style ip_management fill:#264653,stroke:#2a9d8f,color:#fff
    style ses fill:#3a0ca3,stroke:#240046,color:#fff
    style sendgrid fill:#3a0ca3,stroke:#240046,color:#fff
```

---

## 15. Inbound SMPP Enterprise Integration

End-to-end sequence showing an enterprise client connected via SMPP. The client maintains a persistent transceiver bind to the Hermes SMPP Server, sends messages as submit_sm PDUs, and receives delivery receipts as deliver_sm PDUs on the same bind. This enables high-throughput, low-latency integration without HTTP overhead.

```mermaid
sequenceDiagram
    autonumber
    participant EC as Enterprise Client<br/>(SMPP Transceiver)
    participant SS as Hermes SMPP Server<br/>(gosmpp)
    participant PGW as Protocol Gateway
    participant VP as Validation Pipeline
    participant NATS as NATS JetStream
    participant CE as Cascade Engine
    participant SR as Smart Router
    participant HLR as HLR/MNP Provider
    participant SCM as SMPP Connection Manager
    participant CRR as Carrier (e.g. Telia DK)

    Note over EC,SS: Connection Establishment
    EC->>SS: bind_transceiver (system_id, password)
    SS->>SS: Validate credentials against tenant registry
    SS-->>EC: bind_transceiver_resp (status: OK)

    Note over EC,SS: Message Submission
    EC->>SS: submit_sm (dest_addr: +4512345678,<br/>short_message: "Your OTP is 847291")
    SS->>SS: Decode PDU, extract fields
    SS-->>EC: submit_sm_resp (message_id: HRM-00001)

    SS->>PGW: Normalized message (unified format)
    PGW->>VP: Run 9-layer validation

    Note over VP: L0-L8: Idempotency, Auth, Tenant,<br/>Rate limit, Balance, Schema,<br/>Compliance, Content, Audit

    VP->>NATS: Publish validated message
    NATS->>CE: Deliver to Cascade Engine
    CE->>CE: Create cascade state machine

    CE->>SR: Route decision request (SMS, +4512345678)
    SR->>HLR: Lookup +4512345678
    HLR-->>SR: Carrier: Telia DK, IMSI: 238010xxxxxxxxx
    SR->>SR: Direct route available, cost: 0.02 DKK
    SR-->>CE: Route: DIRECT via Telia DK SMPP

    CE->>SCM: Dispatch to Telia DK
    SCM->>CRR: submit_sm (+4512345678, "Your OTP is 847291")
    CRR-->>SCM: submit_sm_resp (carrier_msg_id: TEL-99887)

    Note over CE: State: Channel1_Waiting

    CRR--)SCM: deliver_sm (DLR: DELIVRD, id: TEL-99887)
    SCM->>NATS: Publish DLR event
    NATS->>CE: Delivery receipt
    CE->>CE: State -> Channel1_Delivered (terminal)

    Note over SS,EC: Forward DLR to Enterprise Client
    SS->>EC: deliver_sm (DLR: DELIVRD, id: HRM-00001)
    EC-->>SS: deliver_sm_resp (status: OK)
```

---

## 16. End-to-End Carrier-Direct Message Flow

Complete sequence showing a REST API message taking the carrier-direct path. A client sends an SMS to a Danish mobile number via the REST API. The message traverses the full validation pipeline, the Cascade Engine invokes the Smart Router which performs an HLR lookup, determines the cheapest direct route, and dispatches via SMPP to the carrier. The delivery receipt flows back to confirm delivery and triggers the client's webhook.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client Application
    participant GW as API Gateway
    participant VP as Validation Pipeline
    participant NATS as NATS JetStream
    participant CE as Cascade Engine
    participant SR as Smart Router
    participant HLR as HLR/MNP Provider
    participant SCM as SMPP Connection Manager
    participant TDK as Telia DK (SMPP)
    participant WH as Client Webhook

    C->>GW: POST /v1/messages<br/>{to: "+4523456789", channel: "sms",<br/>body: "Your order #1234 has shipped"}
    GW->>VP: Run 9-layer validation

    Note over VP: L0: Idempotency check -- new request
    Note over VP: L1: Auth -- API key valid, tenant: acme-corp
    Note over VP: L2: Tenant active -- OK
    Note over VP: L3: Rate limit -- 42/100 per second, OK
    Note over VP: L4: Balance -- 12,450 DKK remaining, OK
    Note over VP: L5: Schema -- valid SMS payload
    Note over VP: L6: Compliance -- recipient not suppressed
    Note over VP: L7: Content -- clean, no phishing URLs
    Note over VP: L8: Audit log written
    Note over VP: L9: Route to Cascade

    VP-->>GW: Validation passed
    GW-->>C: 202 Accepted {message_id: "msg_8f3a2b"}
    GW->>NATS: Publish validated message

    NATS->>CE: Deliver message
    CE->>CE: Create cascade state (primary: SMS)
    CE->>SR: Route request (SMS, +4523456789, DK)

    SR->>HLR: Lookup +4523456789
    HLR-->>SR: Carrier: Telia DK, MCC-MNC: 238-01

    Note over SR: Route comparison:<br/>Direct (Telia SMPP): 0.02 DKK<br/>Aggregator (Messente): 0.08 DKK<br/>Fallback (Twilio): 0.15 DKK

    SR-->>CE: Route: DIRECT via Telia DK SMPP (0.02 DKK)

    CE->>SCM: Dispatch via Telia DK
    SCM->>SCM: Acquire connection from pool<br/>(window: 38/50 in-flight)
    SCM->>TDK: submit_sm (dest: +4523456789)
    TDK-->>SCM: submit_sm_resp (msg_id: TEL-55443)

    Note over CE: State: Channel1_Waiting<br/>Timer: 60s timeout

    TDK--)SCM: deliver_sm (DLR: DELIVRD,<br/>id: TEL-55443, done_date: 20260211T143022)
    SCM->>NATS: Publish delivery event
    NATS->>CE: DLR received -- DELIVERED

    CE->>CE: State -> Channel1_Delivered (terminal)
    CE->>NATS: Publish final outcome: DELIVERED

    NATS->>CE: Trigger webhook dispatch
    CE->>WH: POST /webhooks/hermes<br/>{message_id: "msg_8f3a2b",<br/>status: "delivered",<br/>carrier: "telia_dk",<br/>cost: 0.02,<br/>timestamp: "2026-02-11T14:30:22Z"}
```

---

## 17. Route Selection Decision Matrix

Overview of how the Smart Router selects the optimal delivery path for each channel type. The decision follows a tiered preference: direct infrastructure first (lowest cost), then aggregator partners (medium cost), then retail provider APIs (highest cost, highest reliability guarantee). Each tier has specific availability checks before traffic is routed.

```mermaid
flowchart TB
    entry(["Inbound Message<br/>from Cascade Engine"])

    entry --> channel_split{"Channel<br/>Type?"}

    %% SMS Path
    channel_split -->|"SMS"| sms_region{"Recipient<br/>Region?"}

    sms_region -->|"Danish Mobile<br/>(+45)"| sms_hlr["HLR/MNP Lookup"]
    sms_region -->|"International"| sms_intl_agg

    sms_hlr --> sms_carrier{"Home<br/>Carrier?"}
    sms_carrier -->|"Telia DK"| sms_telia["Direct SMPP: Telia<br/><b>0.02 DKK/msg</b>"]
    sms_carrier -->|"TDC"| sms_tdc["Direct SMPP: TDC<br/><b>0.025 DKK/msg</b>"]
    sms_carrier -->|"3 DK"| sms_three["Direct SMPP: 3 DK<br/><b>0.022 DKK/msg</b>"]
    sms_carrier -->|"Telenor"| sms_telenor["Direct SMPP: Telenor<br/><b>0.023 DKK/msg</b>"]
    sms_carrier -->|"Unknown/Ported"| sms_intl_agg

    sms_intl_agg["Aggregator Route<br/><i>Messente / Infobip</i><br/><b>0.05-0.12 DKK/msg</b>"]
    sms_intl_agg -->|"Aggregator down"| sms_twilio

    sms_telia -->|"Connection down"| sms_intl_agg
    sms_tdc -->|"Connection down"| sms_intl_agg
    sms_three -->|"Connection down"| sms_intl_agg
    sms_telenor -->|"Connection down"| sms_intl_agg

    sms_twilio["Fallback: Twilio API<br/><b>0.12-0.20 DKK/msg</b><br/><i>Highest reliability</i>"]

    %% Email Path
    channel_split -->|"Email"| email_type{"Message<br/>Type?"}

    email_type -->|"Transactional"| email_pool_a["Own MTA Pool A<br/><i>Dedicated transactional IPs</i><br/><b>~0.001 DKK/msg</b>"]
    email_type -->|"Marketing"| email_pool_b["Own MTA Pool B<br/><i>Separate marketing IPs</i><br/><b>~0.001 DKK/msg</b>"]

    email_pool_a -->|"Pool degraded /<br/>burst overflow"| email_ses
    email_pool_b -->|"Pool degraded /<br/>burst overflow"| email_ses

    email_ses["Elastic Burst: SES / SendGrid<br/><b>~0.005-0.01 DKK/msg</b>"]

    %% WhatsApp Path
    channel_split -->|"WhatsApp"| wa_bsp{"BSP<br/>Available?"}

    wa_bsp -->|"Yes"| wa_direct["Direct: Meta Cloud API (BSP)<br/><b>Template-based pricing</b>"]
    wa_bsp -->|"No"| wa_twilio["Fallback: Twilio WhatsApp<br/><b>Higher per-message cost</b>"]

    %% Push / Telegram (unchanged)
    channel_split -->|"Push"| push_fcm["Firebase CM<br/><i>HTTP v1 API</i><br/><b>Free</b>"]
    channel_split -->|"Telegram"| tg_bot["Telegram Bot API<br/><b>Free</b>"]

    style channel_split fill:#0f3460,stroke:#e94560,color:#fff
    style sms_region fill:#264653,stroke:#2a9d8f,color:#fff
    style sms_carrier fill:#264653,stroke:#2a9d8f,color:#fff

    style sms_telia fill:#2d6a4f,stroke:#1b4332,color:#fff
    style sms_tdc fill:#2d6a4f,stroke:#1b4332,color:#fff
    style sms_three fill:#2d6a4f,stroke:#1b4332,color:#fff
    style sms_telenor fill:#2d6a4f,stroke:#1b4332,color:#fff
    style sms_intl_agg fill:#e76f51,stroke:#9b2226,color:#fff
    style sms_twilio fill:#9b2226,stroke:#641220,color:#fff

    style email_pool_a fill:#2d6a4f,stroke:#1b4332,color:#fff
    style email_pool_b fill:#2d6a4f,stroke:#1b4332,color:#fff
    style email_ses fill:#e76f51,stroke:#9b2226,color:#fff

    style wa_direct fill:#2d6a4f,stroke:#1b4332,color:#fff
    style wa_twilio fill:#e76f51,stroke:#9b2226,color:#fff

    style push_fcm fill:#264653,stroke:#2a9d8f,color:#fff
    style tg_bot fill:#264653,stroke:#2a9d8f,color:#fff
```

---

## Diagram Index

| # | Diagram | Type | Key Concepts |
|---|---------|------|-------------|
| 1 | System Overview | C4 Context | External actors, platform boundary, carrier-direct + provider paths |
| 2 | Main Cluster Architecture | Component (Flowchart) | Protocol Gateway, Smart Router, SMPP/MTA managers, NATS connectivity |
| 3 | Tenant Tier Model | Flowchart | Starter / Business / Enterprise isolation levels |
| 4 | Message Flow | Sequence | Full message lifecycle, validation through delivery |
| 5 | Cascade Engine States | State Diagram | Multi-channel fallback state machine |
| 6 | Tenant Onboarding | Sequence | Sign-up through provisioning to dashboard |
| 7 | Adapter Registration | Sequence | Ed25519 auth, heartbeat protocol, graceful shutdown |
| 8 | IP Reputation Engine | Flowchart | Scoring, warming, blacklist, routing decisions |
| 9 | Validation Pipeline | Flowchart | 9 layers with rejection paths |
| 10 | Deployment Architecture | Flowchart | Docker Compose (dev) and Kubernetes (prod) |
| 11 | Protocol Gateway Layer | Flowchart | REST/SMPP/SMTP inbound, protocol normalization, unified format |
| 12 | Smart Routing Engine | Flowchart | HLR/MNP lookup, least-cost routing, direct vs aggregator vs fallback |
| 13 | Carrier-Direct SMS Infrastructure | Flowchart | SMPP connection pools, window management, DLR handling, carrier binds |
| 14 | Email MTA Infrastructure | Flowchart | MTA pools, IP warming, DKIM/SPF/DMARC, ISP feedback loops |
| 15 | Inbound SMPP Enterprise Integration | Sequence | Enterprise SMPP bind, submit_sm/deliver_sm, DLR forwarding |
| 16 | End-to-End Carrier-Direct Flow | Sequence | REST to carrier-direct SMS, full validation, cost comparison, DLR |
| 17 | Route Selection Decision Matrix | Flowchart | Per-channel routing tiers, cost comparison, fallback chains |
