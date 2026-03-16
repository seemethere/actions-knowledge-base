# Grafana Alloy for Metrics Push to Grafana Cloud

## Context
Needed to push OSDC cluster metrics to Grafana Cloud without coupling Prometheus to the cloud endpoint. Chose Grafana Alloy over Prometheus `remote_write` for full decoupling.

## Finding

### Architecture
Alloy runs independently alongside Prometheus. It discovers the same ServiceMonitor/PodMonitor CRDs and scrapes targets itself, then pushes to Grafana Cloud via `prometheus.remote_write`. Prometheus remains purely in-cluster — no `remote_write` config needed.

### Helm chart
- Chart: `grafana/alloy` from `https://grafana.github.io/helm-charts`
- Controller type: `deployment` (not DaemonSet — Alloy doesn't need to run on every node)
- Replicas: 2 (for HA)

### Clustering for deduplication
With 2+ replicas, Alloy will scrape the same targets from each replica, causing duplicate metrics. Fix: enable clustering in each component:

```river
prometheus.operator.servicemonitors "default" {
  forward_to = [prometheus.remote_write.grafana_cloud.receiver]
  clustering {
    enabled = true
  }
}
```

Clustering uses a hash ring to distribute scrape targets across replicas — each target is scraped by exactly one replica.

### Secret-gated deployment
The Alloy install is double-gated in `deploy.sh`:
1. `kubectl get secret grafana-cloud-credentials -n monitoring` must succeed
2. `monitoring.grafana_cloud_url` in `clusters.yaml` must be non-empty

This allows clusters to opt-in by creating the secret and setting the URL.

### Secret format
```bash
kubectl create secret generic grafana-cloud-credentials \
  -n monitoring \
  --from-literal=username='<GRAFANA_CLOUD_INSTANCE_ID>' \
  --from-literal=password='<GRAFANA_CLOUD_API_KEY>'
```

### Env var injection
Per-cluster values (credentials, URL, cluster name) are injected via a temporary Helm override file rather than fragile `--set` array indexing:

```yaml
alloy:
  extraEnv:
    - name: GCLOUD_RW_API_USER
      valueFrom:
        secretKeyRef:
          name: grafana-cloud-credentials
          key: username
    - name: GCLOUD_RW_API_KEY
      valueFrom:
        secretKeyRef:
          name: grafana-cloud-credentials
          key: password
    - name: GCLOUD_RW_URL
      value: "https://prometheus-prod-36-prod-us-west-0.grafana.net/api/prom/push"
    - name: CLUSTER_NAME
      value: "my-cluster"
```

### Alloy config (River syntax)
Alloy uses HCL-like "River" config, not YAML. Key components:
- `prometheus.operator.servicemonitors` — auto-discovers ServiceMonitor CRDs
- `prometheus.operator.podmonitors` — auto-discovers PodMonitor CRDs
- `prometheus.remote_write` — pushes to Grafana Cloud Mimir endpoint

Environment variables are accessed via `env("VAR_NAME")` in River config.

### Node placement
Alloy pods run on base-infrastructure nodes (same as Prometheus):
```yaml
nodeSelector:
  role: base-infrastructure
tolerations:
  - key: CriticalAddonsOnly
    operator: Equal
    value: "true"
    effect: NoSchedule
```

## Source
Grafana Alloy docs, Helm chart source, and experimentation during OSDC monitoring setup.
