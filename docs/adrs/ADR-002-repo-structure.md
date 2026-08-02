# ADR-002: Multi-Repository Code Organization and Governance

* **Status:** Accepted
* **Date:** 2026-08-02
* **Authors:** SSJ Bespoke Joinery & Engineering

---

## 1. Context

### Background
As the lab expands across OS configuration, Python microservices, containerization, network overlay management, and Kubernetes orchestration, storing all assets in a single monolithic repository ("monorepo") or mixing infrastructure automation with application source code introduces significant operational friction:

1. **Tangled CI/CD Pipelines:** A change to a Python telemetry agent trigger would unnecessarily run Kubernetes manifest linters or Linux server hardening tests.
2. **Ambiguous Ownership Boundaries:** In professional platform engineering, infrastructure management (OS/Networking), application development (Python APIs), and platform delivery (Kubernetes/GitOps) represent distinct domains with different lifecycles and testing requirements.
3. **Configuration Leakage:** Storing host systemd units, network routing rules, application code, and K8s manifests in one directory makes secret sanitization, repository public/private visibility management, and portfolio demonstration confusing.

### Problem Statement
We require a clear, scalable repository architecture that:
* Decouples system provisioning from application code and deployment delivery.
* Defines explicit ownership rules for every script, configuration file, container definition, and deployment manifest.
* Directly maps software artifacts to their target deployment runtime (Cloud, Raspberry Pi, Proxmox).

---

## 2. Decision

We decide to adopt a **3-Repository Model** representing the three core disciplines of platform engineering: **Infrastructure**, **Application**, and **Platform/Delivery**.

```text
[github.com/your-username/](https://github.com/your-username/)
├── 1. lab-infrastructure/    # OS, Networking, Hardening, Bash, Tailscale, systemd
├── 2. lab-telemetry-app/     # Python agents, FastAPI backend, Dockerfiles, CLI tools
└── 3. lab-k8s-platform/      # Declarative Kubernetes YAML, Helm values, Drone CI pipelines