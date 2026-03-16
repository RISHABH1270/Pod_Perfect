# PodPerfect — Architecture Decision Records (ADR)

---

## ADR-001: Deployment Model — Fully In-Cluster

**Decision:** PodPerfect runs entirely inside the user's K8s cluster via Helm.

| Option | Pros | Cons |
|---|---|---|
| **Option 1: Fully In-Cluster** ✅ | No data egress, secure, self-contained, works with internal Prometheus DNS, familiar pattern (same as Grafana, Argo CD) | Consumes some cluster resources |
| **Option 2: Hybrid (Agent in-cluster, Backend external)** | Backend managed separately, easier to update | Data leaves the cluster — enterprises won't accept this |
| **Option 3: External with port-forward** | No in-cluster footprint | Only works for local dev/testing, not production |

**Rationale:** Enterprises will not send metrics to an external system. All data stays inside their infrastructure.

---

## ADR-002: Metrics Source — Prometheus over metrics-server

**Decision:** PodPerfect uses Prometheus as the sole metrics source.

| | metrics-server | Prometheus |
|---|---|---|
| **Retention** | ~15 minutes | Days, weeks, months (configurable) |
| **Resolution** | 60s scrape | 15s scrape (default) |
| **Data for p99** | Impossible — too short | Yes — 90d easily |
| **Per-container metrics** | Yes | Yes |
| **CPU throttling metric** | No | Yes (`container_cpu_throttled_seconds_total`) |
| **OOMKill metric** | No | Yes (`kube_pod_container_status_last_terminated_reason`) |
| **Purpose** | Real-time `kubectl top` / HPA only | Full observability, alerting, long-term analysis |

**Rationale:** metrics-server was never designed for historical analysis — it only powers `kubectl top` and HPA. Prometheus was built exactly for what PodPerfect needs: scrape, store, and query metrics over time.

---

## ADR-003: Why p99 for `requests`

**Decision:** `requests` are set to the p99 of observed CPU and memory usage.

`requests` tell the K8s scheduler the minimum a pod needs to run normally. The percentile choice matters:

| Percentile | Problem |
|---|---|
| p50 | Too low — pod starves 50% of the time |
| p90 | Leaves 10% of traffic starving |
| p95 | Better, but spikes can still cause throttling |
| **p99** | **Covers almost all real traffic — sweet spot** ✅ |
| p100 (max) | Over-provisions — defeats the purpose |

**Rationale:** p99 covers real-world load without padding for extreme outliers (one-time batch job, deploy spike, debug session that ran once).

---

## ADR-004: Why 2x for `limits`

**Decision:** `limits` are set to 2x the p99 usage by default (configurable).

`limits` define when a pod gets killed or throttled instead of starving the node:

| Multiplier | Risk |
|---|---|
| 1x (requests = limits) | Any spike → OOMKill or CPU throttle immediately |
| 1.5x | Still tight for bursty workloads |
| **2x** | **Handles spikes and rare bursts — recommended default** ✅ |
| 3x | Wastes capacity, back to the original problem |
| Unlimited | Dangerous for memory; acceptable for CPU |

**Rationale:** 2x absorbs p99→p100 spikes without over-allocating. It is the right balance between safety and cost efficiency.

---

## ADR-006: Backend Language — Go

**Decision:** PodPerfect backend is written in Go.

| Language | Pros | Cons |
|---|---|---|
| **Go** ✅ | Native `client-go` K8s library, low memory footprint, fast static binary, great concurrency for metrics processing, same language as K8s ecosystem | Verbose error handling |
| **Python** | Fast to write, great stats libraries (numpy, pandas) | High memory usage, slow startup, not native to K8s ecosystem |
| **Java** | Mature, strong typing, good K8s client | Heavy JVM overhead — bad for in-cluster deployment, slow startup |
| **C++** | Fastest raw performance | Overkill, no K8s ecosystem, extremely slow to build |
| **Rust** | Fast, low memory, safe | Steep learning curve, small K8s ecosystem |

**Rationale:** Go wins for three specific reasons:
1. **`client-go`** — the official K8s client library is Go. Every other language has a community-maintained wrapper that lags behind
2. **In-cluster footprint** — Go compiles to a single static binary, tiny Docker image (~10MB). Python/Java images are 200MB–500MB
3. **Same language as the ecosystem** — Prometheus, Helm, Argo CD, kubectl are all Go. Easier to contribute, debug, and integrate

---

## ADR-007: Frontend Framework — Next.js + TypeScript

**Decision:** PodPerfect frontend is built with Next.js + TypeScript.

| Framework | Pros | Cons |
|---|---|---|
| **Next.js + TypeScript** ✅ | Server-Side Rendering (SSR) for fast initial load, built on React so same ecosystem, production-ready out of the box | Slightly more complex than plain React |
| **React** | Simpler setup, pure client-side | No SSR — dashboard loads slower on first hit, blank screen until JS loads |
| **Angular** | Full framework, strong typing | Heavy, overkill for a dashboard, slow to develop |

**SSR (Server-Side Rendering):** The page HTML is generated on the server before being sent to the browser — meaning the dashboard is visible immediately on load, without waiting for JavaScript to execute. This matters for dashboards with large tables of workloads and recommendations.

**Rationale:** Next.js wins because:
1. **SSR** — dashboards with hundreds of workloads/recommendations load faster. Pure React shows a blank screen until JS loads
2. **It IS React** — Next.js is React with SSR on top. All React libraries (Recharts, Tremor, Tailwind) work as-is, no ecosystem tradeoff

---

## ADR-008: Database — TimescaleDB

**Decision:** PodPerfect uses TimescaleDB as the time-series database.

| Database | Pros | Cons |
|---|---|---|
| **TimescaleDB** ✅ | Built on Postgres, native SQL percentile queries (`percentile_cont`), hypertables for time-series, automatic data retention policies | Requires Postgres knowledge |
| **Postgres** | Mature, reliable, SQL | Not optimized for time-series — slow on large metric queries, no built-in retention |
| **MySQL** | Simple, widely used | No native time-series support, weak analytics queries |
| **MongoDB** | Flexible schema | No native percentile queries, not designed for time-series, expensive aggregations |
| **Cassandra** | Excellent write throughput, scales horizontally | Complex to operate, no SQL percentile functions, overkill for single-cluster deployment |

**Rationale:** TimescaleDB wins for three reasons:
1. **It IS Postgres** — TimescaleDB is just a Postgres extension. Same SQL, same tooling, same backups. Zero extra learning curve if you know Postgres
2. **Built for exactly this** — time-series metrics with retention policies. `SELECT percentile_cont(0.99) WITHIN GROUP (ORDER BY cpu_cores)` works natively and is fast
3. **Cassandra/Mongo are overkill** — those are for distributed, multi-node deployments at massive scale. PodPerfect runs in a single cluster — that complexity is not justified

---

## ADR-009: UI Access — LoadBalancer Service

**Decision:** PodPerfect UI is exposed via a Kubernetes LoadBalancer service.

| Option | Pros | Cons |
|---|---|---|
| **LoadBalancer** ✅ | Cloud provider auto-assigns external IP, works on EKS/GKE/AKS out of the box, no domain or ingress needed | Requires cloud provider support |
| **port-forward** | Zero setup | Poor UX — requires kubectl access on every session, not suitable for a team |
| **Ingress** | Integrates with existing ingress controller | Requires ingress controller already running, more configuration |
| **NodePort** | Simple, works on-prem | Exposes a port on every node, less clean |

**Rationale:** port-forward is a developer shortcut — not suitable for a product used by a team. LoadBalancer gives a stable external IP (`http://<EXTERNAL-IP>:3000`) that anyone on the team can access without kubectl. No domain or certificate required.
