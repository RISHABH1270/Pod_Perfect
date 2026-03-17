# PodPerfect

<h3><span style="color:#00b894">Stop guessing.</span> <span style="color:#0984e3">Start rightsizing.</span></h3>

PodPerfect is an open-source, self-hosted Kubernetes resource optimizer. It analyzes historical CPU and memory usage at the **99th percentile** and recommends precise `requests` and `limits` — cutting cluster costs without sacrificing stability.

---

## The Problem

While working on the DevOps side, I noticed that services running in production clusters invariably have their Kubernetes `requests` and `limits` set based on guesswork or worst-case assumptions — rather than actual usage data.

This leads to two expensive problems that end up costing companies significantly:

- **Over-provisioning** → wasted cloud spend (the most common case in production)
- **Under-provisioning** → OOMKills, CPU throttling, pod restarts, CrashLoopBackOff, and pod instability

---

## How It Works

```
Prometheus ──▶ Collector ──▶ TimescaleDB ──▶ Analysis Engine ──▶ Recommendations ──▶ Apply
```

1. **Collects** up to 90 days of CPU and memory metrics from Prometheus
2. **Analyzes** p99 usage per container across all workloads
3. **Recommends** — `requests` = p99 usage, `limits` = 2x p99 (configurable)
4. **Displays** recommended `requests` and `limits` on the dashboard — your team applies the changes

---

## Key Features

- **p99-based recommendations** — data-driven, not guesswork
- **Recommendation-only** — PodPerfect analyzes your cluster and suggests improvements, your team decides what to apply
- **Cost estimation** — see exact monthly savings (AWS, GCP, Azure)
- **Confidence scoring** — 🟢 High / 🟡 Medium / 🔴 Low based on data availability
- **Dashboard recommendations** — suggested `requests` and `limits` values shown clearly, your team applies them manually
- **Self-hosted** — fully in-cluster via Helm, no data leaves your infrastructure

---

## Tech Stack

| Layer | Choice |
|---|---|
| Backend | Go |
| Frontend | Next.js + TypeScript |
| Database | TimescaleDB |
| Cache | Redis |
| Metrics | Prometheus |
| Deployment | Helm |

---

## Installation

```bash
helm repo add podperfect https://charts.podperfect.io
helm repo update

helm install podperfect podperfect/podperfectchart \
  --namespace podperfect \
  --create-namespace \
  --set prometheus.url=http://prometheus-server.monitoring.svc.cluster.local
```

Then get the external IP assigned by your cloud provider:

```bash
kubectl get svc podperfect-ui -n podperfect
```

Open: `http://<EXTERNAL-IP>:3000`

---


## Documentation

| Doc | Description |
|---|---|
| [PRD](docs/PRD.md) | Problem statement, features, phased delivery |
| [Technical Design](docs/TECHNICAL_DESIGN.md) | Core workflow, data model, recommendation logic |
| [Architecture](docs/ARCHITECTURE.md) | Tech stack, deployment diagram, component breakdown |
| [API Spec](docs/API_SPEC.md) | REST endpoints and schemas |
| [ADR](docs/ADR.md) | Architecture decisions and tradeoffs |
| [Runbook](docs/RUNBOOK.md) | Install, configure, troubleshoot |

---
