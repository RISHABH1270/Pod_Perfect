# PodPerfect — Product Requirements Document (PRD)

---

## Problem Statement

While working on the DevOps side, I noticed that services running in production clusters invariably have their Kubernetes `requests` and `limits` set based on guesswork or worst-case assumptions — rather than actual usage data.

This leads to two expensive problems that end up costing companies significantly:

- **Over-provisioning** → wasted cloud spend (the most common case in production)
- **Under-provisioning** → OOMKills, CPU throttling, pod restarts, CrashLoopBackOff, and pod instability

The goal: **data-driven rightsizing using actual 99th percentile usage — where `requests` are set to the p99 usage value and `limits` are set to 2x the p99 usage by default (configurable, but 2x is the recommended baseline) — to eliminate waste and reduce cloud costs.**

---

## Target Users

- **DevOps / Platform Engineers** — managing production and development K8s clusters
- **SREs** — responsible for reliability and cost efficiency
- **Engineering Managers** — looking for cost visibility across teams

---

## Features

### 1. Data Collection
- Pull metrics from **Prometheus** — up to 90 days of historical data
- Collect metrics per container (sidecars like Istio, Fluentd each have their own requests/limits)
- Separate metric collection for **init containers**

### 2. Analysis Engine
- Compute **p99** CPU and memory per container
- Detect workload usage patterns: **Bursty**, **Steady**, **Diurnal**
- Flag **CPU throttling %** and **OOMKill history**
- Track **replica count over time** for auto-scaled workloads

### 3. Recommendation Engine
- `requests` = p99 usage, `limits` = 2x p99 (configurable per workload)
- Confidence score: 🟢 High / 🟡 Medium / 🔴 Low — based on data availability
- If delta < 10% from current values — show ✅ **"Current values are optimal"**
- Override per workload — e.g. a specific service wants 3x limits instead of the global 2x default
- See [TECHNICAL_DESIGN.md](TECHNICAL_DESIGN.md) for full recommendation logic

### 4. Cost Estimation
- Support AWS (EKS), GCP (GKE), Azure (AKS) node pricing
- Show per-workload: current monthly cost → projected monthly cost → **monthly savings $**
- Aggregate savings view at cluster level

### 5. Dashboard
- PodPerfect is recommendation-only — it analyzes your cluster and suggests improvements, your team decides what to apply
- Recommended `requests` and `limits` values are displayed clearly on the dashboard
- Your team reviews the suggestions and applies them manually to their manifests

### 6. Recommendation Summary
- Dashboard shows a quick count of recommendations by status:
  - 🟡 **Open** — pending review
  - 🟢 **Accepted** — team has acknowledged and will apply
  - ✅ **Done** — team has applied the changes

### 7. Savings Unlocked ✨
- Hero metric on the main dashboard — shows **total cost saved so far**
- Calculated by summing `projected_monthly_savings_usd` of all recommendations marked as `done`
- Designed to motivate teams — the more recommendations actioned, the higher the number grows
- Shows: total savings all time, total savings this month, number of workloads optimised

---

## Phased Delivery

| Phase | Scope |
|---|---|
| **v0.1** | Single cluster, Prometheus, p99 analysis, recommendation dashboard |
| **v0.2** | Cost estimation (AWS first), confidence scoring UI |
| **v0.3** | GCP + Azure pricing, OOMKill detection, CPU throttling analysis |
| **v1.0** | Full Helm chart, weekly reports |
