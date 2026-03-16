# kube-prometheus-stack CRD Ordering Problem

## Context
Deploying the OSDC monitoring module failed with `resource mapping not found for kind "ServiceMonitor"` and `ensure CRDs are installed first` errors.

## Finding

### The problem
The OSDC justfile's generic `deploy-module` recipe runs three phases in order:
1. **Terraform** (if `terraform/main.tf` exists)
2. **Kubernetes** (if `kubernetes/kustomization.yaml` exists) — `kubectl apply -k`
3. **Deploy script** (if `deploy.sh` exists)

For the monitoring module, Phase 2 tries to apply ServiceMonitor/PodMonitor resources, but the CRDs for `monitoring.coreos.com/v1` don't exist yet — they're installed by the kube-prometheus-stack Helm chart in Phase 3 (`deploy.sh`).

### The fix
Split kubernetes resources into two kustomizations:

**`kubernetes/kustomization.yaml`** (applied by justfile Phase 2 — CRD-free):
- `namespace.yaml`
- `dcgm-exporter/daemonset.yaml` (DaemonSet + Service — standard k8s resources)

**`kubernetes/monitors/kustomization.yaml`** (applied by `deploy.sh` AFTER Helm install):
- All ServiceMonitors (`servicemonitors/*.yaml`)
- All PodMonitors (`podmonitors/*.yaml`)
- `dcgm-exporter/servicemonitor.yaml`

The monitors kustomization references parent-relative paths (`../servicemonitors/arc-controller.yaml`, etc.) so the actual YAML files stay in their original locations.

### Deploy order (monitoring module)
1. Justfile applies `kubernetes/kustomization.yaml` → namespace + DCGM DaemonSet
2. `deploy.sh` installs kube-prometheus-stack via Helm → CRDs are created
3. `deploy.sh` applies `kubernetes/monitors/` → ServiceMonitors + PodMonitors
4. `deploy.sh` optionally installs Grafana Alloy

### General lesson
Any OSDC module that uses CRD-dependent resources AND installs the CRD provider via `deploy.sh` will hit this ordering problem. The pattern is: keep CRD-free resources in the main `kubernetes/kustomization.yaml` and CRD-dependent resources in a sub-kustomization applied by `deploy.sh`.

## Source
Discovered during first-time monitoring deployment to a fresh cluster (arc-staging).
