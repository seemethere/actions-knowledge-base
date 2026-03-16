# kube-prometheus-stack Admission Webhook Fails on Tainted Clusters

## Context
First-time install of kube-prometheus-stack on an OSDC cluster fails with `timed out waiting for the condition` during the Helm pre-install phase.

## Finding

### Symptom
```
Error: failed pre-install: 1 error occurred:
    * timed out waiting for the condition
```

The `kube-prometheus-stack-admission-create` job stays in `Pending` state. Pod events show:
```
0/11 nodes are available: 11 node(s) had untolerated taint(s)
```

### Root cause
The kube-prometheus-stack Helm chart creates a pre-install hook job (`kube-prometheus-stack-admission-create`) that sets up the admission webhook. This job doesn't have any tolerations by default. On OSDC clusters, ALL nodes are tainted:
- Base infrastructure: `CriticalAddonsOnly=true:NoSchedule`
- Runner nodes: `instance-type=<type>:NoSchedule`
- GPU nodes: `nvidia.com/gpu=true:NoSchedule`
- BuildKit nodes: `instance-type=<type>:NoSchedule`

So the job pod can't schedule anywhere.

### Fix
Add tolerations and nodeSelector to the admission webhook patch jobs in the Helm values:

```yaml
prometheusOperator:
  tolerations:
    - key: CriticalAddonsOnly
      operator: Equal
      value: "true"
      effect: NoSchedule
  nodeSelector:
    role: base-infrastructure
  admissionWebhooks:
    patch:
      tolerations:
        - key: CriticalAddonsOnly
          operator: Equal
          value: "true"
          effect: NoSchedule
      nodeSelector:
        role: base-infrastructure
```

### Recovery from stuck state
If the install already timed out, the failed Helm release and the stuck job must be cleaned up before retrying:

```bash
helm delete kube-prometheus-stack -n monitoring
kubectl delete job kube-prometheus-stack-admission-create -n monitoring
# Then redeploy
just deploy-module <cluster> monitoring
```

### General lesson
On fully-tainted OSDC clusters, EVERY component — including Helm hook jobs — needs explicit tolerations and nodeSelector for base-infrastructure nodes. The kube-prometheus-stack chart has many sub-components, each with its own toleration path. All must be set:

- `prometheus.prometheusSpec.tolerations`
- `grafana.tolerations`
- `alertmanager.alertmanagerSpec.tolerations`
- `nodeExporter.tolerations` (use `operator: Exists` to tolerate all taints)
- `kube-state-metrics.tolerations`
- `prometheusOperator.tolerations`
- `prometheusOperator.admissionWebhooks.patch.tolerations`

## Source
Discovered during first-time monitoring deployment to arc-staging cluster. Pod events showed taint-related scheduling failure.
