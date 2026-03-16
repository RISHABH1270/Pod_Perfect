# PodPerfect — Architecture

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Backend API | **Go** | Native `client-go`, low overhead, ideal for K8s tooling |
| Frontend | **Next.js + TypeScript** | SSR for fast dashboard load, built on React |
| Charts | **Recharts / Tremor** | Time-series visualizations |
| Time-series DB | **TimescaleDB** | Postgres extension — native SQL percentile queries, easy self-hosting |
| Cache | **Redis** | Cache computed recommendations, avoid re-processing on each load |
| Deployment | **Helm chart** | Deployed into the user's cluster via Helm |
| Metrics source | **Prometheus** | Industry standard — most production clusters already have it |

---

## Deployment Diagram

```
┌─────────────────────────────────────────────────┐
│                  User's K8s Cluster              │
│                                                   │
│   ┌──────────────┐     ┌────────────────────┐    │
│   │  PodPerfect  │────▶│    Prometheus      │    │
│   │  Collector   │     │  (existing)        │    │
│   └──────┬───────┘     └────────────────────┘    │
│          │                                        │
│          ▼                                        │
│   ┌──────────────┐     ┌────────────────────┐    │
│   │  TimescaleDB │     │   Redis Cache      │    │
│   └──────┬───────┘     └────────────────────┘    │
│          │                                        │
│   ┌──────▼───────┐                               │
│   │  PodPerfect  │◀──── K8s API (read-only)      │
│   │  API (Go)    │                               │
│   └──────┬───────┘                               │
│          │                                        │
│   ┌──────▼───────┐                               │
│   │  Next.js UI  │◀──── Browser                  │
│   └──────────────┘                               │
└─────────────────────────────────────────────────┘
```

All components run inside the user's cluster via a single `helm install` — **no metrics leave their infrastructure.**

---

## Component Responsibilities

| Component | Responsibility |
|---|---|
| **Collector** | Scrapes Prometheus, stores raw metrics into TimescaleDB |
| **TimescaleDB** | Stores time-series metric samples, runs p99 queries |
| **Redis** | Caches computed recommendations to avoid re-processing on each page load |
| **API (Go)** | Serves REST API, runs recommendation engine, reads from K8s API |
| **Next.js UI** | Dashboard — displays recommendations and cost savings |
| **Prometheus** | Existing cluster monitoring — source of all metrics |
