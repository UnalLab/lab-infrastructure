# ADR-001: Network Architecture and Access Boundaries

* **Status:** Accepted
* **Date:** 2026-08-02
* **Authors:** AHMED ERVA UNAL

---

## 1. Context

### Environment & Topography
The home lab infrastructure spans three distinct physical and logical locations with varying availability characteristics:
1. **Cloud Core (`cloud-core-01`):** An always-available public cloud server acting as the ingress edge and public service host.
2. **Home Edge (`home-edge-01`):** An always-available Raspberry Pi residing behind a residential NAT on the Virgin home network (`192.168.0.0/24`).
3. **Lab Platform (`platform-01`):** A Proxmox hypervisor hosting pfSense (`10.10.10.0/24` transit, `10.0.1.0/24` MGMT, `10.0.10.0/24` CORP) and K3s virtual machines. This environment is intentionally temporary and operates only when booted into lab mode.

### The Problem Statement
As the lab expands across cloud, edge, and virtualized local networks, multiple traffic types compete for access routes:
* **Public Users:** Require low-latency HTTPS access to the portfolio website and public health endpoints without reaching internal infrastructure.
* **Infrastructure Workloads:** Require stable, encrypted machine-to-machine paths across NAT boundaries for metrics scraping, log shipping, database queries, and CI/CD pipeline deployments.
* **Human Administrators:** Require authenticated, identity-checked access to management tools (e.g., Grafana, Gitea, Drone, Kubernetes API, SSH) without exposing management ports directly to the public internet.

Exposing administrative ports to the public internet creates severe security vulnerabilities. Conversely, forcing all traffic through a single remote access mechanism (such as a single VPN or tunnel) creates routing collisions, single points of failure, ambiguous security perimeters, and circular recovery dependencies during outages.

### Architectural Risks Without Explicit Boundaries
If no access boundaries are enforced, the environment risks:
* Accidentally exposing sensitive management interfaces or raw database sockets to the public internet.
* Constructing overlapping IP subnets or competing overlay routes across Cloudflare and Tailscale.
* Establishing single points of failure where an outage in the identity provider or overlay network locks out all recovery mechanisms.
* Excessive over-privilege where monitoring agents or CI runners possess full administrative cluster rights.

---

## 2. Decision

We decision to establish a **strict separation of network access responsibilities** governed by traffic type, trust model, and operational intent. No single technology will handle all connectivity. Instead, each technology owns a specific layer, an isolated failure domain, and a explicit architectural purpose.

### Core Architectural Principles

1. **Private by Default:** All internal services, administrative web UI panels, database ports, and management APIs remain strictly private. Public access must be explicitly justified and routed through the designated public ingress layer.
2. **Outbound-First Connectivity:** Edge and home systems must initiate outbound tunnels to public edges or overlay networks. No inbound port-forwarding rules will be configured on the home router.
3. **Separation of Transport and Identity:** Network reachability (Layer 3/4 transport) does not grant application authorization (Layer 7 identity). Traffic transport and identity authorization are decoupled.
4. **Independent Emergency Break-Glass Path:** Every management layer must maintain a documented, independent recovery path that does not depend on the primary access technology itself.

### Layer Ownership Mapping

The initial technology implementation maps to access layers as follows:

* **Public Ingress Layer (Cloudflare Tunnel):** Manages anonymous public HTTP/HTTPS traffic from internet visitors to public web frontends and APIs.
* **Infrastructure Mesh Layer (Tailscale):** Manages private machine-to-machine communications, metrics collection, automated deployments, and subnet routing across cloud, edge, and lab nodes.
* **Identity-Aware Access Layer (Octelium):** Manages authenticated, human administrator and workload access to sensitive internal web applications, databases, and control planes.
* **Local Virtual Network Layer (pfSense):** Controls firewall policies, inter-VLAN routing, and network segmentation within the local Proxmox environment.
* **Break-Glass Recovery Layer (Direct Console / Tailscale SSH):** Provides out-of-band administrative access via cloud provider web consoles, physical Proxmox consoles, and direct Tailscale SSH if upper layers fail.

---

## 3. Separation of Responsibilities Matrix

| Traffic / Service | Source | Destination | Primary Transport Owner | Authorisation Owner | Exposure Level | Emergency Recovery Path |
|---|---|---|---|---|---|---|
| **Public Portfolio & Status** | Internet Visitors | Portfolio Frontend | Cloudflare Tunnel | Public / App Auth | Public | Private Tailscale / Cloud Console |
| **Telemetry Ingestion** | Local Agents | Cloud FastAPI Endpoint | Tailscale | API Token / mTLS | Private Workload | Direct SSH via Tailscale |
| **Prometheus Metrics** | Cloud Prometheus | Node Exporters / cAdvisor | Tailscale | Network ACL + Exporter IP | Private Workload | Local SSH / Direct Inspection |
| **Human Grafana Access** | Administrator | Grafana UI | Octelium | Octelium Identity + Grafana RBAC | Private Human | Tailscale SSH / Direct Tunnel |
| **Git Operations & CI** | Drone Runner / Admin | Gitea / K3s API | Tailscale | SSH Key / K8s RBAC / Octelium | Private Workload | Direct Proxmox Console |
| **Database Traffic** | FastAPI Backend | PostgreSQL | Tailscale | Database User Credentials | Private Workload | Local psql via SSH |
| **SSH Node Admin** | Administrator | Linux Host Nodes | Octelium (Normal) | SSH Key / Identity | Private Human | Tailscale SSH / Cloud Console |
| **Lab Routing & Segment** | Proxmox VMs | pfSense Interfaces | pfSense | pfSense Firewall Policy | Local Network | Proxmox Virtual Console |
| **Media & Storage (Jellyfin)**| Home LAN | Raspberry Pi / Shield | Local LAN / Tailscale | Media Server Auth | Local / Private | Physical LAN Access |

### Ownership Rule
*Every distinct traffic flow MUST have exactly ONE primary transport owner. Alternate paths are permitted exclusively as documented emergency recovery mechanisms and must never operate as secondary active routes during normal operations.*

---

## 4. Forbidden Configurations and Guardrails

To prevent architectural degradation, the following rules are strictly enforced:

### A. Public Exposure Guardrails
* **MUST NOT** expose Proxmox VE, pfSense WebGUI, Docker Sockets, Kubernetes API, Prometheus, or database ports directly to the public internet.
* **MUST NOT** configure public port-forwarding rules on the home residential gateway.
* **MUST NOT** publish an administrative interface to the web to bypass configuring private connectivity.

### B. Routing & Subnet Guardrails
* **MUST NOT** advertise the same CIDR block through two competing overlay systems (e.g., Cloudflare Private Network and Tailscale Subnet Router simultaneously).
* **MUST NOT** route general management traffic over the Proxmox WAN transit network (`10.10.10.0/24`).
* **MUST NOT** permit uncontrolled routing between the local `CORP` network and `MGMT` network without explicit pfSense stateful firewall inspection.

### C. Identity & Least Privilege Guardrails
* **MUST NOT** grant administrative cluster permissions to automated monitoring agents or CI/CD runners.
* **MUST NOT** embed infrastructure credentials, private IP addresses, or overlay tokens in public web application source code or Git repositories.
* **MUST NOT** use personal administrative credentials for automated service accounts.

### D. Recovery Guardrails
* **MUST NOT** execute maintenance upgrades on primary and break-glass access channels at the same time.
* **MUST NOT** remove a legacy access path before validating the replacement in an isolated test run.

---

## 5. Consequences

### Positive
* **Operational Clarity:** Clear boundaries eliminate guesswork regarding which system handles specific traffic types.
* **Reduced Attack Surface:** Zero inbound firewall ports on the home gateway and private management interfaces shield the lab from automated public scanning.
* **Fault Isolation:** Failure of the public ingress tunnel (Cloudflare) does not interrupt private monitoring, telemetry collection, or administrative capabilities.
* **Professional Standards:** Demonstrates separation of concerns, Layer 3 vs. Layer 7 boundary management, and zero-trust engineering principles.

### Negative & Trade-offs
* **Increased System Complexity:** Managing three distinct access tools increases configuration overhead compared to a single monolithic VPN.
* **Operational Maintenance:** Requires maintaining multiple agents (`cloudflared`, `tailscaled`, `octelium-gateway`), token rotation lifecycles, and policy code.
* **Resource Constraints:** Identity-aware access software (Octelium) requires dedicated Kubernetes resources, which must be accounted for on low-resource nodes.

---

## 6. Assumptions

1. The cloud server (`cloud-core-01`) remains operational 24/7 to accept public web traffic and ingest edge telemetry.
2. The Raspberry Pi (`home-edge-01`) remains powered on continuously to act as a home edge probe and subnet router.
3. The Proxmox environment (`platform-01`) is intentionally intermittent and may be shut down without triggering false-positive system outage alerts on public status pages.

---

## 7. Non-Goals

This ADR explicitly **does not** cover:
* Specific installation steps, package commands, or Docker Compose syntax for tools.
* The internal application code structure of the FastAPI backend or telemetry agents.
* The specific storage replication or backup schedules for virtual machines.

---

## 8. Future Decisions

Subsequent architecture decisions will build upon this foundation:
* **ADR-002:** Public Web Ingress via Cloudflare Tunnel
* **ADR-003:** Infrastructure Mesh Configuration via Tailscale
* **ADR-004:** Identity-Aware Access Control via Octelium
* **ADR-005:** Telemetry Agent Data Contract & Ingestion Architecture
* **ADR-006:** K3s Kubernetes Topology & Deployment Pipeline

---

## 9. Review Triggers

This ADR must be reopened and formally re-evaluated if:
1. A new physical site or secondary cloud provider is added to the infrastructure.
2. Tailscale, Cloudflare Tunnel, or Octelium is fully replaced by an alternative software solution.
3. Overlapping private IP space is introduced that disrupts existing subnet routing tables.
4. An emergency lockout occurs due to a failure in the break-glass recovery hierarchy.

---

## 10. Validation Criteria

This architecture is considered valid and operational when:
* [ ] Public HTTPS requests to `portfolio.yourdomain.com` resolve cleanly through Cloudflare Tunnel without exposing port 80/443 on the cloud firewall.
* [ ] Prometheus on `cloud-core-01` successfully scrapes Node Exporters on `home-edge-01` and `platform-01` exclusively over Tailscale IPs.
* [ ] Simulating a complete outage of the identity gateway leaves direct Tailscale SSH operational for emergency recovery.
* [ ] Shutting down the local Proxmox server transitions the public status page to `Planned Maintenance` without causing cascading failure alerts on the cloud server.