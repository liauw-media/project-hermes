# Hetzner Infrastructure Research for Project Hermes

> **Date:** 2026-02-16
> **Status:** RESEARCH
> **Context:** RKE2 Kubernetes cluster for multi-tenant message broker
> **Target:** 22M messages/day at scale, 1K msg/sec for MVP

---

## Table of Contents

1. [Hetzner Cloud Server Pricing](#1-hetzner-cloud-server-pricing)
2. [Hetzner Dedicated Server Pricing](#2-hetzner-dedicated-server-pricing)
3. [Hetzner Cloud Locations](#3-hetzner-cloud-locations)
4. [Additional Infrastructure Costs](#4-additional-infrastructure-costs)
5. [Hetzner Managed Kubernetes](#5-hetzner-managed-kubernetes)
6. [RKE2 on Hetzner: Compatibility and Gotchas](#6-rke2-on-hetzner-compatibility-and-gotchas)
7. [Community Projects and Terraform Modules](#7-community-projects-and-terraform-modules)
8. [Project Hermes Resource Requirements](#8-project-hermes-resource-requirements)
9. [Recommended Cluster Layouts](#9-recommended-cluster-layouts)
10. [Pricing Comparison: Hetzner vs AWS/GCP](#10-pricing-comparison-hetzner-vs-awsgcp)
11. [Recommendations](#11-recommendations)

---

## 1. Hetzner Cloud Server Pricing

All prices in EUR/month, excluding VAT. Prices are for EU locations (Germany/Finland -- cheapest regions).

### Shared vCPU -- CX Series (Intel/AMD x86)

| Model | vCPU | RAM | Storage | Traffic | EUR/mo |
|-------|------|-----|---------|---------|--------|
| CX23  | 2    | 4 GB  | 40 GB  | 20 TB | €3.49  |
| CX33  | 4    | 8 GB  | 80 GB  | 20 TB | €5.49  |
| CX43  | 8    | 16 GB | 160 GB | 20 TB | €9.49  |
| CX53  | 16   | 32 GB | 320 GB | 20 TB | €17.49 |

### Shared vCPU -- CAX Series (ARM Ampere Altra)

| Model | vCPU | RAM | Storage | Traffic | EUR/mo |
|-------|------|-----|---------|---------|--------|
| CAX11 | 2    | 4 GB  | 40 GB  | 20 TB | €3.79  |
| CAX21 | 4    | 8 GB  | 80 GB  | 20 TB | €6.49  |
| CAX31 | 8    | 16 GB | 160 GB | 20 TB | €12.49 |
| CAX41 | 16   | 32 GB | 320 GB | 20 TB | €24.49 |

> **Note on ARM (CAX):** Go compiles natively to ARM64. All Hermes services (Go), PostgreSQL, NATS, Redis, and Temporal support ARM. Prometheus and Grafana also have ARM builds. CAX is viable for this stack, but RKE2 ARM support should be verified. CAX offers ~30% worse price/performance than CX for compute but is competitive for memory-bound workloads.

### Dedicated vCPU -- CPX Series (AMD, General Purpose)

| Model | vCPU | RAM | Storage | Traffic | EUR/mo |
|-------|------|-----|---------|---------|--------|
| CPX11 | 2    | 2 GB  | 40 GB  | 1 TB  | €4.99  |
| CPX21 | 3    | 4 GB  | 80 GB  | 2 TB  | €9.49  |
| CPX31 | 4    | 8 GB  | 160 GB | 3 TB  | €16.49 |
| CPX41 | 8    | 16 GB | 240 GB | 4 TB  | €30.49 |
| CPX51 | 16   | 32 GB | 360 GB | 5 TB  | €60.49 |

### Dedicated vCPU -- CPX2 Series (AMD, Storage-Optimized)

| Model | vCPU | RAM | Storage | Traffic | EUR/mo |
|-------|------|-----|---------|---------|--------|
| CPX12 | 1    | 2 GB  | 40 GB  | 0.5 TB | €6.49  |
| CPX22 | 2    | 4 GB  | 80 GB  | 1 TB   | €6.49  |
| CPX32 | 4    | 8 GB  | 160 GB | 2 TB   | €10.99 |
| CPX42 | 8    | 16 GB | 320 GB | 3 TB   | €19.99 |
| CPX52 | 12   | 24 GB | 480 GB | 4 TB   | €28.49 |
| CPX62 | 16   | 32 GB | 640 GB | 5 TB   | €38.99 |

### Dedicated vCPU -- CCX Series (AMD EPYC/Intel Xeon, High Performance)

| Model | vCPU | RAM | Storage | Traffic | EUR/mo |
|-------|------|-----|---------|---------|--------|
| CCX13 | 2    | 8 GB   | 80 GB  | 1 TB | €12.49  |
| CCX23 | 4    | 16 GB  | 160 GB | 2 TB | €24.49  |
| CCX33 | 8    | 32 GB  | 240 GB | 3 TB | €48.49  |
| CCX43 | 16   | 64 GB  | 360 GB | 4 TB | €96.49  |
| CCX53 | 32   | 128 GB | 600 GB | 6 TB | €192.49 |
| CCX63 | 48   | 192 GB | 960 GB | 8 TB | €288.49 |

> **CCX vs CPX:** CCX provides guaranteed dedicated CPU cores with no noisy-neighbor effects. Critical for latency-sensitive workloads (Temporal, PostgreSQL). ~2.5x the cost of shared CX but consistent performance. For production databases, CCX is strongly recommended.

---

## 2. Hetzner Dedicated Server Pricing

Bare-metal servers. All prices in EUR/month, excluding VAT. Located in Germany (Falkenstein) or Finland (Helsinki). Setup fees apply (typically €0-€79 one-time).

| Model | CPU | Cores/Threads | RAM | Storage | Traffic | EUR/mo |
|-------|-----|---------------|-----|---------|---------|--------|
| AX42 | AMD Ryzen 7 PRO 8700GE | 8c/16t | 64 GB DDR5 ECC | 2x 512 GB NVMe | Unlimited | €49 |
| AX52 | AMD Ryzen 7 7700 | 8c/16t | 64 GB DDR5 | 2x 1 TB NVMe | Unlimited | €64 |
| AX102 | AMD EPYC (16 cores) | 16c/32t | 128 GB DDR5 ECC | 2x 1.92 TB NVMe | Unlimited | ~€128 |
| AX162 | AMD EPYC 9454P | 48c/96t | 128-256 GB DDR5 ECC | 2x 1.92+ TB NVMe | Unlimited | €199 |

> **Dedicated vs Cloud trade-offs:**
> - Dedicated: Better raw performance, unlimited traffic, ECC RAM, NVMe storage, predictable cost. No API-driven scaling, no snapshots, 1-24h provisioning time.
> - Cloud: API-driven, instant provisioning, snapshots, load balancers, floating IPs, block storage volumes. Shared vCPU (except CCX), limited traffic (though 20 TB is generous), higher per-resource cost.

---

## 3. Hetzner Cloud Locations

### EU Datacenters

| Location | City | Identifier | Notes |
|----------|------|------------|-------|
| **Nuremberg, Germany** | Nuremberg | `nbg1` | Primary DE datacenter |
| **Falkenstein, Germany** | Falkenstein (Vogtland) | `fsn1` | Hetzner HQ area, dedicated servers |
| **Helsinki, Finland** | Tuusula (near Helsinki) | `hel1` | Nordic location, cold climate (low cooling costs) |

### Other Locations (Non-EU relevance)

| Location | Identifier | Notes |
|----------|------------|-------|
| Ashburn, USA | `ash` | US East Coast |
| Hillsboro, USA | `hil` | US West Coast |
| Singapore | `sin` | Asia-Pacific |

### Recommendation for Project Hermes

**Primary: Falkenstein (`fsn1`) or Nuremberg (`nbg1`)** -- both in Germany, which satisfies GDPR data residency. Hetzner's own datacenter parks with lowest latency to each other. Dedicated servers are primarily in Falkenstein.

**Secondary/DR: Helsinki (`hel1`)** -- Finland, also EU/GDPR compliant. Good for geographic redundancy. Connected to German DCs via Hetzner's own fiber.

> **Inter-datacenter latency:** Nuremberg <-> Falkenstein: ~2ms. Germany <-> Helsinki: ~20-30ms (over Hetzner backbone). All within acceptable range for async replication.

---

## 4. Additional Infrastructure Costs

### Networking and Services

| Service | Cost | Notes |
|---------|------|-------|
| **Load Balancer LB11** | €5.39/mo | 5 targets, 25 connections/s |
| **Load Balancer LB21** | €16.40/mo | 25 targets, 250 connections/s |
| **Load Balancer LB31** | €32.90/mo | 75 targets, 500 connections/s |
| **Floating IPv4** | ~€3.57/mo | Per IP address |
| **Primary IPv4** | Included | One per server |
| **Private Network** | **Free** | VLAN between servers in same DC |
| **Firewall** | **Free** | Cloud firewall rules |
| **DDoS Protection** | **Free** | Basic L3/L4 protection |

### Storage

| Service | Cost | Notes |
|---------|------|-------|
| **Block Storage Volumes** | €0.044/GB/mo | ~€4.40/100GB, SSD, 3x replicated |
| **Snapshots** | €0.011/GB/mo | Point-in-time server snapshots |
| **Automatic Backups** | 20% of instance price | Up to 7 retained |

### Traffic

| Region | Included | Overage |
|--------|----------|---------|
| Germany/Finland (Cloud) | 20 TB/mo (most plans) | €1/TB |
| Dedicated Servers | Unlimited | Included |
| Internal traffic | **Free** | Between servers in same DC |

---

## 5. Hetzner Managed Kubernetes

Hetzner does **not** offer a native managed Kubernetes service directly. However, there are third-party managed Kubernetes options that run on Hetzner infrastructure:

### Option A: Self-Managed (RKE2/K3s)

- **Control plane cost:** €0 (runs on your own nodes)
- **You manage:** etcd, API server, scheduler, controller-manager, upgrades
- **Best for:** Teams with Kubernetes experience who want full control

### Option B: Syself Managed Kubernetes on Hetzner

- Third-party managed Kubernetes using Cluster API
- Enterprise-grade with automated upgrades and monitoring
- Pricing available on request (not publicly listed)
- Source: [syself.com/hetzner](https://syself.com/hetzner)

### Option C: Cloudfleet Managed Kubernetes on Hetzner

- Managed K8s control plane on Hetzner infrastructure
- Automated node management
- Source: [cloudfleet.ai](https://cloudfleet.ai/lp/managed-hetzner-kubernetes/)

### Option D: Control Plane (controlplane.com)

- Multi-cloud managed Kubernetes with Hetzner as a provider
- Source: [docs.controlplane.com/mk8s/hetzner](https://docs.controlplane.com/mk8s/hetzner)

### Recommendation

**Self-managed RKE2 is the right choice for Project Hermes.** The cost savings are substantial (no managed K8s premium), and the team needs to understand the infrastructure deeply for a production message broker. RKE2 provides a good balance of security defaults (CIS hardened) and operational simplicity.

---

## 6. RKE2 on Hetzner: Compatibility and Gotchas

### Compatibility Status

RKE2 runs well on Hetzner Cloud with standard Linux distributions (Ubuntu 22.04/24.04, openSUSE MicroOS, Rocky Linux). There are no fundamental incompatibilities. Multiple community projects and production deployments confirm this.

### Known Issues and Workarounds

| Issue | Severity | Workaround |
|-------|----------|------------|
| **TLS/x509 certificate error after CCM install** | HIGH | Must set `node-ip` in RKE2 config to the private network IP before installing Hetzner Cloud Controller Manager. Without this, the CCM changes node addresses and the kubelet certificate becomes invalid. |
| **Cilium + Hetzner CNI + Rancher UI** | MEDIUM | Container log access via Rancher UI fails with TLS verification errors when using Cilium as CNI with Hetzner networking. Workaround: use Canal (default RKE2 CNI) or configure Cilium to use the private network correctly. |
| **No native Load Balancer integration** | LOW | Hetzner Cloud Controller Manager (hcloud-ccm) provides `LoadBalancer` service type support. Must be installed as a DaemonSet/Deployment. Not as seamless as AWS ALB Ingress Controller but functional. |
| **No native CSI driver** | LOW | Hetzner CSI driver (hcloud-csi) provides persistent volume support via Hetzner Block Storage. Works well but volumes are limited to single-node attachment (ReadWriteOnce). |
| **Floating IP failover** | LOW | Floating IPs require the hcloud-ccm or a custom controller for automatic failover. Not as instant as cloud-native VIP solutions. |
| **Primary IP deletion deprecation** | INFO | From May 2026, you must unassign Primary IPs before deleting them. Plan automation accordingly. |
| **DHE key exchange removal** | INFO | From March 2026, Hetzner Load Balancers drop DHE (finite-field Diffie-Hellman). ECDHE continues to work. Ensure TLS clients support ECDHE. |

### Critical Configuration for RKE2 on Hetzner

```yaml
# /etc/rancher/rke2/config.yaml on each node
node-ip: "<PRIVATE_NETWORK_IP>"  # CRITICAL: Use private IP, not public
tls-san:
  - "<PUBLIC_IP>"
  - "<PRIVATE_IP>"
  - "<LOAD_BALANCER_IP>"
cloud-provider-name: external     # Required for hcloud-ccm
disable-cloud-controller: true    # Let external CCM handle it
```

### Required Hetzner Integrations

1. **hcloud-ccm** (Cloud Controller Manager): Provides node lifecycle management, LoadBalancer service type, and node metadata.
2. **hcloud-csi** (Container Storage Interface): Provides persistent volumes backed by Hetzner Block Storage.
3. **Hetzner Firewall**: Configure via Terraform/API to allow RKE2 ports (6443, 9345, 10250, 2379-2380, etcd peer ports).

---

## 7. Community Projects and Terraform Modules

### terraform-hcloud-rke2 (wenzel-felix)

- **Repository:** [github.com/wenzel-felix/terraform-hcloud-rke2](https://github.com/wenzel-felix/terraform-hcloud-rke2)
- **Terraform Registry:** [registry.terraform.io/modules/wenzel-felix/rke2/hcloud](https://registry.terraform.io/modules/wenzel-felix/rke2/hcloud/latest)
- **Version:** 0.6.1
- **Features:**
  - HA RKE2 cluster across multiple Hetzner zones
  - Integrated hcloud-ccm and hcloud-csi
  - Firewall configuration
  - Optional Cloudflare DNS integration
  - Small, maintainable codebase
- **Status:** Actively maintained, purpose-built for RKE2 on Hetzner
- **Recommendation:** **Primary choice for Project Hermes.** Direct RKE2 support, clean module design, small dependency footprint.

### terraform-hcloud-kube-hetzner (kube-hetzner)

- **Repository:** [github.com/kube-hetzner/terraform-hcloud-kube-hetzner](https://github.com/kube-hetzner/terraform-hcloud-kube-hetzner)
- **Features:**
  - Highly optimized K3s cluster on openSUSE MicroOS
  - Auto-upgrades for both OS and Kubernetes
  - Large community, well-tested
  - RKE2 support discussed but not yet merged (active PR)
- **Status:** K3s-focused. RKE2 support is in development.
- **Recommendation:** Good fallback if K3s is acceptable. The community is larger, but K3s is not RKE2.

### Hetzner-cloud-Rancher-RKE2 (Amirtheahmed)

- **Repository:** [github.com/Amirtheahmed/Hetzner-cloud-Rancher-RKE2](https://github.com/Amirtheahmed/Hetzner-cloud-Rancher-RKE2)
- **Features:** End-to-end deployment of Rancher + RKE2 using Ansible and Terraform
- **Status:** Reference implementation, less actively maintained

### terraform-hcloud-rke2 (granito-source)

- **Repository:** [github.com/granito-source/terraform-hcloud-rke2](https://github.com/granito-source/terraform-hcloud-rke2)
- **Features:** Alternative RKE2 Terraform module for Hetzner
- **Status:** Smaller community

### clusterworks-dev/terraform-hcloud-rke2

- **Repository:** [github.com/clusterworks-dev/terraform-hcloud-rke2](https://github.com/clusterworks-dev/terraform-hcloud-rke2)
- **Features:** Another community RKE2 module
- **Status:** Less documentation available

---

## 8. Project Hermes Resource Requirements

Based on the implementation plan's infrastructure costs section and standard production requirements for each component:

### Per-Component Resource Estimates

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Storage | Replicas (MVP) | Replicas (Prod) |
|-----------|-------------|-----------|----------------|--------------|---------|----------------|-----------------|
| **Hermes API** | 500m | 2000m | 512 Mi | 2 Gi | - | 2 | 3 |
| **Hermes Cascade Worker** | 500m | 2000m | 512 Mi | 2 Gi | - | 2 | 3 |
| **PostgreSQL 16** | 1000m | 4000m | 2 Gi | 8 Gi | 100 GB | 1 | 2 (primary+replica) |
| **NATS JetStream** | 500m | 2000m | 1 Gi | 4 Gi | 50 GB | 1 | 3 (cluster) |
| **Redis 7** | 250m | 1000m | 512 Mi | 2 Gi | 10 GB | 1 | 2 (primary+replica) |
| **Temporal Frontend** | 500m | 1500m | 1 Gi | 4 Gi | - | 1 | 2 |
| **Temporal History** | 1000m | 4000m | 2 Gi | 4 Gi | - | 1 | 3 |
| **Temporal Matching** | 500m | 1000m | 512 Mi | 2 Gi | - | 1 | 2 |
| **Temporal Worker** | 200m | 500m | 256 Mi | 1 Gi | - | 1 | 2 |
| **Prometheus** | 500m | 2000m | 2 Gi | 4 Gi | 100 GB | 1 | 1 |
| **Grafana** | 250m | 500m | 256 Mi | 512 Mi | 10 GB | 1 | 1 |

### Aggregate Resource Requirements

| Phase | Total CPU Request | Total Memory Request | Total Storage |
|-------|-------------------|----------------------|---------------|
| **MVP (minimal replicas)** | ~6.2 cores | ~11.5 Gi | ~270 GB |
| **Production v1 (HA)** | ~14 cores | ~28 Gi | ~520 GB |
| **Production at scale** | ~24+ cores | ~48+ Gi | ~1 TB+ |

> **Note:** These estimates include ~20% headroom for Kubernetes system pods (kube-system, monitoring agents, CNI, CSI, CCM). RKE2 system components typically consume 1-2 CPU cores and 2-4 GB RAM.

---

## 9. Recommended Cluster Layouts

### Phase 1: MVP / Staging (~€60-100/month)

**Goal:** Functional development and demo environment. Not HA. Single control plane node.

| Role | Server Type | Specs | Count | EUR/mo | Purpose |
|------|-------------|-------|-------|--------|---------|
| Control Plane + Worker | CX43 | 8 vCPU, 16 GB, 160 GB | 1 | €9.49 | RKE2 server + all workloads |
| Worker | CX33 | 4 vCPU, 8 GB, 80 GB | 1 | €5.49 | Overflow workloads |
| Block Storage (PG) | Volume | 100 GB | 1 | €4.40 | PostgreSQL data |
| Block Storage (NATS/Prom) | Volume | 100 GB | 1 | €4.40 | NATS streams + Prometheus |
| Load Balancer | LB11 | Basic | 1 | €5.39 | API ingress |
| **Total** | | | | **~€29.17** | |

> This is the absolute minimum. For a more comfortable staging environment, use CX53 (16 vCPU, 32 GB) as the primary node for ~€17.49, bringing total to ~€37.17.

**Alternative with CCX (recommended for stability):**

| Role | Server Type | Specs | Count | EUR/mo | Purpose |
|------|-------------|-------|-------|--------|---------|
| Control Plane + Worker | CCX23 | 4 dedicated vCPU, 16 GB, 160 GB | 1 | €24.49 | RKE2 server + core workloads |
| Worker | CX33 | 4 vCPU, 8 GB, 80 GB | 1 | €5.49 | Hermes services + monitoring |
| Block Storage | Volume | 200 GB | 1 | €8.80 | All persistent data |
| Load Balancer | LB11 | Basic | 1 | €5.39 | API ingress |
| **Total** | | | | **~€44.17** | |

---

### Phase 2: Production v1 (~€150-250/month)

**Goal:** HA control plane. Separate database workloads. Handles 1K msg/sec MVP target comfortably.

| Role | Server Type | Specs | Count | EUR/mo Each | EUR/mo Total | Purpose |
|------|-------------|-------|-------|-------------|--------------|---------|
| Control Plane | CX33 | 4 vCPU, 8 GB | 3 | €5.49 | €16.47 | HA etcd + RKE2 server (tainted, no workloads) |
| Worker (Stateless) | CPX31 | 4 dedicated vCPU, 8 GB | 2 | €16.49 | €32.98 | Hermes API, Cascade Workers, Temporal |
| Worker (Stateful) | CCX23 | 4 dedicated vCPU, 16 GB | 1 | €24.49 | €24.49 | PostgreSQL, Redis, NATS |
| Worker (Monitoring) | CX33 | 4 vCPU, 8 GB | 1 | €5.49 | €5.49 | Prometheus + Grafana |
| Block Storage (PG) | Volume | 200 GB | 1 | €8.80 | €8.80 | PostgreSQL data |
| Block Storage (NATS) | Volume | 100 GB | 1 | €4.40 | €4.40 | NATS JetStream |
| Block Storage (Prom) | Volume | 200 GB | 1 | €8.80 | €8.80 | Prometheus TSDB |
| Load Balancer | LB21 | Medium | 1 | €16.40 | €16.40 | API ingress + TLS termination |
| Floating IP | IPv4 | - | 1 | €3.57 | €3.57 | Stable ingress IP |
| **Total** | | | | | **~€121.40** | |

**Why this layout:**

- 3x control plane nodes for etcd quorum (tolerates 1 node failure)
- Dedicated vCPU (CPX) for stateless workers ensures consistent latency for Temporal and Hermes
- CCX for stateful workload node: PostgreSQL and Redis benefit from guaranteed CPU and higher memory
- Shared vCPU (CX) acceptable for monitoring (bursty, not latency-critical)
- All in same datacenter with private networking (free)

---

### Phase 3: Production at Scale (~€400-600/month)

**Goal:** Handle 22M messages/day. Full HA for all components. Horizontal scaling capacity.

| Role | Server Type | Specs | Count | EUR/mo Each | EUR/mo Total | Purpose |
|------|-------------|-------|-------|-------------|--------------|---------|
| Control Plane | CCX13 | 2 dedicated vCPU, 8 GB | 3 | €12.49 | €37.47 | HA etcd + RKE2 server |
| Worker (App) | CCX23 | 4 dedicated vCPU, 16 GB | 3 | €24.49 | €73.47 | Hermes API (3), Cascade Workers (3) |
| Worker (Temporal) | CCX23 | 4 dedicated vCPU, 16 GB | 2 | €24.49 | €48.98 | Temporal services (frontend, history, matching, worker) |
| Worker (Data) | CCX33 | 8 dedicated vCPU, 32 GB | 2 | €48.49 | €96.98 | PostgreSQL primary + replica, Redis, NATS cluster |
| Worker (Monitoring) | CX43 | 8 vCPU, 16 GB | 1 | €9.49 | €9.49 | Prometheus + Grafana + alerting |
| Block Storage (PG) | Volume | 500 GB | 2 | €22.00 | €44.00 | PostgreSQL primary + replica |
| Block Storage (NATS) | Volume | 200 GB | 3 | €8.80 | €26.40 | NATS JetStream cluster |
| Block Storage (Prom) | Volume | 500 GB | 1 | €22.00 | €22.00 | Prometheus long-term storage |
| Load Balancer | LB31 | Large | 1 | €32.90 | €32.90 | API ingress |
| Floating IPs | IPv4 | - | 2 | €3.57 | €7.14 | Ingress + DB failover |
| **Total** | | | | | **~€398.83** | |

---

### Alternative: Dedicated Server Approach (Production at Scale)

For maximum cost efficiency at scale, a hybrid approach using Hetzner dedicated servers:

| Role | Server Type | Specs | Count | EUR/mo Each | EUR/mo Total | Purpose |
|------|-------------|-------|-------|-------------|--------------|---------|
| All-in-one K8s Node | AX42 | 8c/16t Ryzen 7, 64 GB DDR5, 2x512 GB NVMe | 3 | €49 | €147 | Control plane + workers |
| Block Storage (extra) | Volume | 500 GB | 2 | €22.00 | €44.00 | Additional persistent storage |
| Load Balancer | LB31 | Large | 1 | €32.90 | €32.90 | API ingress |
| **Total** | | | | | **~€223.90** | |

> **Trade-off:** 3x AX42 gives you 24 real cores, 192 GB RAM, and 3 TB NVMe for €147/month. The equivalent in cloud CCX nodes would cost €400+. However, you lose cloud-native features (instant scaling, snapshots, API-driven management). Good for stable, predictable workloads. Provisioning takes hours, not seconds.

---

## 10. Pricing Comparison: Hetzner vs AWS/GCP

### Equivalent Cluster: Production v1 (HA K8s, ~16 vCPU, ~48 GB RAM total)

| Component | Hetzner | AWS | GCP |
|-----------|---------|-----|-----|
| **Control Plane (3 nodes)** | 3x CX33 = **€16.47** | EKS control plane = **$73/mo** | GKE Autopilot = **$73/mo** |
| **Worker nodes (3x 4vCPU/16GB)** | 3x CCX23 = **€73.47** | 3x m6i.xlarge = **~$432/mo** | 3x e2-standard-4 = **~$365/mo** |
| **1x DB node (4vCPU/16GB)** | 1x CCX23 = **€24.49** | 1x RDS db.m6g.xlarge = **~$265/mo** | 1x Cloud SQL = **~$245/mo** |
| **Load Balancer** | LB21 = **€16.40** | ALB = **~$22/mo** + LCU | Cloud LB = **~$18/mo** + data |
| **Block Storage (500GB)** | **€22.00** | 500GB gp3 = **~$40/mo** | 500GB SSD = **~$85/mo** |
| **Traffic (5 TB/mo)** | **€0** (included) | **~$450/mo** | **~$400/mo** |
| **Monitoring** | Self-hosted = **€5.49** | CloudWatch = **~$50-100/mo** | Cloud Monitoring = **~$50-100/mo** |
| | | | |
| **TOTAL** | **~€158/mo (~$170)** | **~$1,332-1,382/mo** | **~$1,236-1,336/mo** |

### Cost Savings Summary

| Provider | Monthly Cost | Annual Cost | Savings vs AWS |
|----------|-------------|-------------|----------------|
| **Hetzner** | ~€158 (~$170) | ~€1,900 (~$2,040) | **87% savings** |
| **AWS** | ~$1,350 | ~$16,200 | Baseline |
| **GCP** | ~$1,280 | ~$15,360 | 5% savings vs AWS |

> **The traffic difference is massive.** Hetzner includes 20 TB/month per node. AWS charges ~$0.09/GB for egress. For a message broker processing millions of webhooks, API calls, and provider callbacks, 5+ TB/month of traffic is realistic. That alone is $450/month on AWS vs $0 on Hetzner.

### Managed Database Comparison (PostgreSQL)

| Provider | Config | Monthly Cost |
|----------|--------|-------------|
| Hetzner (self-managed on CCX23) | 4 vCPU, 16 GB, 200 GB volume | €33.29 |
| AWS RDS (db.m6g.xlarge) | 4 vCPU, 16 GB, 200 GB gp3 | ~$310/mo |
| GCP Cloud SQL | 4 vCPU, 16 GB, 200 GB SSD | ~$290/mo |

> Self-managed PostgreSQL on Hetzner is 9x cheaper than AWS RDS. The trade-off is operational overhead: you manage backups, replication, and failover yourself. For a startup, this is acceptable.

---

## 11. Recommendations

### Recommended Approach: Cloud-First with Dedicated Migration Path

**Phase 1 (MVP/Staging) -- Start with Hetzner Cloud:**
- Use `terraform-hcloud-rke2` (wenzel-felix) module for cluster provisioning
- CX/CCX instances for flexibility and rapid iteration
- Single datacenter: Falkenstein (`fsn1`) or Nuremberg (`nbg1`)
- **Budget: ~€30-45/month**

**Phase 2 (Production v1) -- Expand Cloud:**
- Add HA control plane (3 nodes)
- Separate stateful and stateless workloads
- CCX for databases and Temporal (guaranteed CPU)
- Add proper monitoring and alerting
- **Budget: ~€120-160/month**

**Phase 3 (Production at Scale) -- Consider Hybrid:**
- Evaluate dedicated servers (AX42) for stable workload nodes
- Keep Hetzner Cloud for burst capacity and management tooling
- Or scale with more CCX cloud nodes
- **Budget: ~€225-400/month** (depending on cloud vs dedicated)

### Key Decisions

| Decision | Recommendation | Rationale |
|----------|---------------|-----------|
| **Cloud vs Dedicated** | Start Cloud, consider Dedicated at scale | Cloud gives flexibility for MVP; dedicated gives cost efficiency at stable load |
| **Datacenter** | Falkenstein (`fsn1`) | Closest to dedicated server availability if you migrate later. Germany = GDPR. |
| **Instance type for DBs** | CCX (dedicated vCPU) | PostgreSQL and Redis need predictable I/O and CPU. No noisy neighbors. |
| **Instance type for app** | CX or CPX for MVP, CCX for production | Shared vCPU fine for development; dedicated for production latency SLAs. |
| **ARM (CAX) vs x86** | x86 for now | RKE2 ARM support is less battle-tested. All components support ARM, but not worth the risk for MVP. Revisit at scale for cost optimization. |
| **K3s vs RKE2** | RKE2 | CIS-hardened by default, closer to upstream Kubernetes, FIPS 140-2 option. Better for a security-conscious message broker. |
| **Terraform module** | wenzel-felix/rke2/hcloud | Purpose-built for RKE2, actively maintained, clean codebase, Terraform Registry published. |
| **Load Balancer** | Hetzner LB + nginx-ingress inside cluster | Hetzner LB for external traffic, nginx-ingress or Traefik for K8s routing. |

### What You Get vs AWS for the Same Budget

With ~€400/month on Hetzner (equivalent to 1 month of a small AWS EKS setup):

- 3x HA control plane nodes
- 8x dedicated-vCPU worker nodes
- 14+ dedicated vCPU cores for workloads
- 64+ GB dedicated RAM
- 1.5 TB+ NVMe SSD storage
- 20+ TB traffic included
- Full RKE2 cluster with Prometheus, Grafana, all Hermes services

On AWS, €400/month ($430) gets you roughly:
- EKS control plane ($73)
- 2x t3.large worker nodes (2 vCPU, 8 GB each) (~$130)
- 1x small RDS ($120)
- Remaining ~$100 for storage and traffic

The difference is stark: **7-8x more compute resources on Hetzner for the same price.**

---

## Sources

- [Hetzner Cloud Pricing](https://www.hetzner.com/cloud/)
- [Hetzner Cloud Pricing Calculator (Feb 2026)](https://costgoat.com/pricing/hetzner)
- [Hetzner Dedicated Server Matrix](https://www.hetzner.com/dedicated-rootserver/matrix-ax/)
- [Hetzner Server Comparison 2025 (Achromatic)](https://www.achromatic.dev/blog/hetzner-server-comparison)
- [Hetzner Cloud Cost-Optimized Plans (Bitdoze)](https://www.bitdoze.com/hetzner-cloud-cost-optimized-plans/)
- [terraform-hcloud-rke2 (wenzel-felix)](https://github.com/wenzel-felix/terraform-hcloud-rke2)
- [terraform-hcloud-kube-hetzner (kube-hetzner)](https://github.com/kube-hetzner/terraform-hcloud-kube-hetzner)
- [RKE2 Hetzner Cloud Provider Setup (support.tools)](https://support.tools/training/rke2/hetzner-cloud-provider-rke2/)
- [Hetzner Cloud API Changelog](https://docs.hetzner.cloud/changelog)
- [Kubernetes: Hetzner vs Civo vs DigitalOcean 2025 (sanj.dev)](https://sanj.dev/post/kubernetes-hetzner-cloud-2025)
- [Syself Managed Kubernetes on Hetzner](https://syself.com/hetzner)
- [Hetzner Cloud Datacenter Locations](https://www.datacenters.com/providers/hetzner/data-center-locations)
- [AWS vs Hetzner Comparison (getdeploying.com)](https://getdeploying.com/aws-vs-hetzner)
- [Saving Costs: AWS to Hetzner Migration (lasoft.org)](https://lasoft.org/blog/saving-costs-on-your-server-infrastructure-why-lasoft-moved-projects-from-aws-to-hetzner/)
- [RKE2 Cilium Hetzner Issue #5163](https://github.com/rancher/rke2/issues/5163)
- [Hetzner-cloud-Rancher-RKE2 (Amirtheahmed)](https://github.com/Amirtheahmed/Hetzner-cloud-Rancher-RKE2)
- [Hetzner Datacenter Locations (hostingrevelations.com)](https://hostingrevelations.com/hetzner-datacenter-locations/)
- [Temporal Deployment Guide (piotrmucha.blog)](https://piotrmucha.blog/2025/09/12/temporal-deployment/)
