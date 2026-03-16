# PodPerfect — Runbook

---

## Installation

### Prerequisites
- Kubernetes cluster (1.21+)
- Prometheus running in the cluster
- Helm 3+

### Install via Helm
```bash
helm repo add podperfect https://charts.podperfect.io
helm repo update

helm install podperfect podperfect/podperfectchart \
  --namespace podperfect \
  --create-namespace \
  --set prometheus.url=http://prometheus-server.monitoring.svc.cluster.local
```

### Access the Dashboard
The UI is exposed via a LoadBalancer service — your cloud provider (AWS/GCP/Azure) automatically assigns an external IP. No domain or ingress needed.

```bash
kubectl get svc podperfect-ui -n podperfect
```

Open: `http://<EXTERNAL-IP>:3000`

### Verify Installation
```bash
kubectl get pods -n podperfect
```

Expected output:
```
NAME                              READY   STATUS    RESTARTS   AGE
podperfect-api-xxx                1/1     Running   0          1m
podperfect-collector-xxx          1/1     Running   0          1m
podperfect-ui-xxx                 1/1     Running   0          1m
podperfect-timescaledb-xxx        1/1     Running   0          1m
podperfect-redis-xxx              1/1     Running   0          1m
```

---

## Upgrade

```bash
helm repo update
helm upgrade podperfect podperfect/podperfect -n podperfect
```

---

## Configuration

| Key | Default | Description |
|---|---|---|
| `prometheus.url` | required | Internal Prometheus service URL |
| `analysis.windowDays` | `90` | Max days of historical data to analyze |
| `recommendation.limitMultiplier` | `2` | Limits = requests × multiplier |
| `recommendation.skipThresholdPct` | `10` | Skip recommendations with < 10% delta |
| `auth.oidc.enabled` | `false` | Enable OIDC SSO |

---

## Troubleshooting

### Collector not scraping Prometheus
```bash
kubectl logs -n podperfect deploy/podperfect-collector
```
- Verify `prometheus.url` is reachable from inside the cluster
- Check Prometheus is running: `kubectl get svc -n monitoring`

### No recommendations appearing
- Ensure at least 1d of data has been collected
- Check TimescaleDB has data: `kubectl exec -n podperfect <timescaledb-pod> -- psql -U podperfect -c "SELECT count(*) FROM metric_samples;"`
- Confidence score may be 🔴 Low if < 30d of data — recommendations still show but are flagged

### UI not loading
```bash
kubectl logs -n podperfect deploy/podperfect-ui
kubectl logs -n podperfect deploy/podperfect-api
```

---

## Uninstall

```bash
helm uninstall podperfect -n podperfect
kubectl delete namespace podperfect
```
