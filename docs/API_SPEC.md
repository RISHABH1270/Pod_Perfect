# PodPerfect — API Specification

> Base URL: `/api/v1`

---

## Recommendations

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/recommendations` | List all recommendations |
| `GET` | `/recommendations/summary` | Count of recommendations by status |
| `GET` | `/recommendations/:id` | Get a single recommendation |
| `POST` | `/recommendations/:id/accept` | Mark recommendation as accepted |
| `POST` | `/recommendations/:id/decline` | Mark recommendation as declined |
| `POST` | `/recommendations/:id/done` | Mark recommendation as done |

---

### `GET /recommendations`

**Query params**
| Param | Type | Description |
|---|---|---|
| `namespace` | string | Filter by namespace |
| `status` | string | `open`, `accepted` 🟢, `declined` 🔴, `done` ✅ |
| `sort_by` | string | `savings` (default), `confidence` |

**Response**
```json
[
  {
    "id": "rec-uuid",
    "namespace": "default",
    "workload_name": "api-server",
    "workload_kind": "Deployment",
    "container_name": "api",
    "current_cpu_request": "500m",
    "current_cpu_limit": "1000m",
    "current_mem_request": "512Mi",
    "current_mem_limit": "1024Mi",
    "suggested_cpu_request": "120m",
    "suggested_cpu_limit": "240m",
    "suggested_mem_request": "200Mi",
    "suggested_mem_limit": "400Mi",
    "p99_cpu": "110m",
    "p99_mem": "185Mi",
    "confidence_score": 0.92,
    "projected_monthly_savings_usd": 48.50,
    "status": "open"  // open | accepted 🟢 | declined 🔴 | done ✅
  }
]
```

---

### `GET /recommendations/summary`

**Response**
```json
{
  "open": 18,
  "accepted": 5,
  "declined": 2,
  "done": 24
}
```

---

### `GET /recommendations/:id`

**Response**
```json
{
  "id": "rec-uuid",
  "namespace": "default",
  "workload_name": "api-server",
  "workload_kind": "Deployment",
  "container_name": "api",
  "current_cpu_request": "500m",
  "current_cpu_limit": "1000m",
  "current_mem_request": "512Mi",
  "current_mem_limit": "1024Mi",
  "suggested_cpu_request": "120m",
  "suggested_cpu_limit": "240m",
  "suggested_mem_request": "200Mi",
  "suggested_mem_limit": "400Mi",
  "p99_cpu": "110m",
  "p99_mem": "185Mi",
  "confidence_score": 0.92,
  "projected_monthly_savings_usd": 48.50,
  "status": "open"
}
```

---

### `POST /recommendations/:id/accept`

**Response**
```json
{
  "id": "rec-uuid",
  "status": "accepted",
  "message": "Recommendation marked as accepted"
}
```

---

### `POST /recommendations/:id/decline`

**Response**
```json
{
  "id": "rec-uuid",
  "status": "declined",
  "message": "Recommendation marked as declined"
}
```

---

### `POST /recommendations/:id/done`

**Response**
```json
{
  "id": "rec-uuid",
  "status": "done",
  "message": "Recommendation marked as done"
}
```

---

## Savings Unlocked

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/savings-unlocked` | Total savings from all done recommendations |

---

### `GET /savings-unlocked`

**Response**
```json
{
  "total_saved_all_time_usd": 3200.00,
  "total_saved_this_month_usd": 480.00,
  "workloads_optimised": 24
}
```

> Calculated by summing `projected_monthly_savings_usd` of all recommendations with `status: done`.

---

## Cost Summary

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/cost-summary` | Cluster-level cost and savings overview |

---

### `GET /cost-summary`

**Response**
```json
{
  "current_monthly_cost_usd": 4200.00,
  "projected_monthly_cost_usd": 2800.00,
  "potential_savings_usd": 1400.00
}
```
