# OSDC Module Deploy Phases and Ordering

## Context
Understanding the exact sequence of operations when deploying an OSDC module is critical to avoid CRD ordering issues and other dependency problems.

## Finding

### Justfile `deploy-module` recipe (Phase ordering)
The justfile's `deploy-module` recipe runs three phases sequentially:

1. **Phase 1: Terraform** — if `$MODULE_DIR/terraform/main.tf` exists, runs `tofu init` + `tofu plan` + `tofu apply`
2. **Phase 2: Kubernetes** — if `$MODULE_DIR/kubernetes/kustomization.yaml` exists, runs `kubectl apply -k $MODULE_DIR/kubernetes/`
3. **Phase 3: Deploy script** — if `$MODULE_DIR/deploy.sh` exists and is executable, runs it with `(cluster-id, cluster-name, region)` args

### Key implications

- Phase 2 runs BEFORE Phase 3. If a module's Helm chart (installed in `deploy.sh`) provides CRDs that the kubernetes resources depend on, the kubernetes resources will fail to apply.
- **Workaround**: Split kubernetes resources. Put CRD-free resources in `kubernetes/kustomization.yaml` (auto-applied by justfile). Put CRD-dependent resources in a sub-kustomization and apply them from `deploy.sh` after the Helm install.
- The justfile recipe is generic — it doesn't know about module-specific ordering requirements. Module authors must design around this.

### Environment variables available in deploy.sh
- `$1` = cluster-id (e.g., `arc-staging`)
- `$2` = cluster-name (e.g., `pytorch-arc-staging`)
- `$3` = region (e.g., `us-west-2`)
- `$OSDC_ROOT` = consumer's osdc/ directory
- `$OSDC_UPSTREAM` = upstream's osdc/ directory
- `$CLUSTERS_YAML` = path to clusters.yaml

### Module resolution
The justfile checks `$ROOT/modules/$MODULE` first (consumer), then `$UPSTREAM/modules/$MODULE` (upstream). Local modules override upstream modules with the same name.

### Shebang recipe gotcha
The `deploy-module` recipe uses `#!/usr/bin/env bash` (shebang style), which means it runs as a standalone script, NOT through the configured `mise exec` shell. However, `deploy.sh` scripts source `scripts/mise-activate.sh` to get mise tools on PATH.

## Source
Read from `osdc/justfile` lines 209-288 during debugging of monitoring CRD ordering failure.
