# PodPerfect — Technical Design Document

---

## Core Workflow

```
Cluster(s)
   │
   ├── Prometheus
   │         │
   │    [Data Collector]
   │         └── analyzes up to 90d of historical CPU/memory data (uses all available if < 90d)
   │         │
   │    [Time-Series Store]
   │         └── stores raw metrics per container (sidecars have their own requests/limits)
   │         │
   │    [Analysis Engine]
   │         ├── p99 CPU usage
   │         ├── p99 Memory usage
   │         ├── CPU throttling %
   │         ├── OOMKill history
   │         ├── burst vs sustained pattern detection
   │         └── workload type awareness (Deployment / StatefulSet / DaemonSet / CronJob)
   │         │
   │    [Recommendation Engine]
   │         ├── suggested requests  = p99 usage
   │         ├── suggested limits    = requests × 2x (configurable per workload)
   │         └── confidence score    = f(data window length, variance)
   │         │
   │    [Cost Estimator]
   │         ├── current cost    = (current_cpu_request / node_cpu) × node_monthly_price × replicas
   │         ├── projected cost  = (suggested_cpu_request / node_cpu) × node_monthly_price × replicas
   │         └── projected savings = (Δcpu / node_cpu + Δmem / node_mem) × node_price × replicas
   │         │
   │    [Dashboard]
   │         ├── displays recommended requests/limits — user applies manually
   │         └── Savings Unlocked ✨ — hero metric showing total cost saved from all done recommendations
```

---

## Data Model

```
Cluster
  └── Namespace
        └── Workload (Deployment / StatefulSet / DaemonSet / CronJob)
              └── Pod
                    └── Container
                          └── MetricSample[]
                                ├── timestamp
                                ├── cpu_cores (float64)
                                └── memory_bytes (int64)

Recommendation
  ├── id
  ├── namespace
  ├── workload_name
  ├── workload_kind
  ├── container_name
  ├── analysis_window_days        (actual days of data available, max 90)
  ├── current_cpu_request / current_cpu_limit
  ├── current_mem_request / current_mem_limit
  ├── suggested_cpu_request / suggested_cpu_limit
  ├── suggested_mem_request / suggested_mem_limit
  ├── p99_cpu
  ├── p99_mem
  ├── cpu_throttle_pct
  ├── oomkill_count_7d
  ├── confidence_score            (0.0 – 1.0)
  ├── projected_monthly_savings_usd
  └── status                      (open | accepted | declined | done)
```

---

## Recommendation Logic

### requests
```
cpu_request = p99_cpu
mem_request = p99_mem
```

### limits
```
cpu_limit = cpu_request × limit_multiplier   (default: 2x, configurable per workload)
mem_limit = mem_request × limit_multiplier   (default: 2x, configurable per workload)
```

### Confidence Score
| Score | Condition |
|---|---|
| 🟢 High (0.8 – 1.0) | 60–90d of data, low variance |
| 🟡 Medium (0.5 – 0.8) | 30–60d of data |
| 🔴 Low (0.0 – 0.5) | < 30d or high variance |

### Skip Threshold
If delta < 10% from current values — show ✅ **"Current values are optimal"** instead of a recommendation.
