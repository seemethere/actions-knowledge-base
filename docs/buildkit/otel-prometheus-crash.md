# BuildKit v0.27.1 Crashes with OTEL Prometheus Exporter

## Context
Deploying BuildKit with `OTEL_METRICS_EXPORTER=prometheus` and `OTEL_EXPORTER_PROMETHEUS_PORT=9090` environment variables causes an immediate crash in BuildKit v0.27.1.

## Finding
BuildKit v0.27.1 does **NOT** support `OTEL_METRICS_EXPORTER=prometheus`. Setting this env var causes the buildkitd process to exit immediately with:

```
buildkitd: unsupported opentelemetry exporter prometheus
```

This manifests as:
- `CrashLoopBackOff` on both `buildkitd-arm64` and `buildkitd-amd64` pods
- `exceeded its progress deadline` errors during `helm upgrade --install` or `kubectl rollout status`
- The error is only visible in pod logs (`kubectl logs <pod> -n buildkit`), not in events

### What works
- BuildKit supports `OTEL_METRICS_EXPORTER=otlp` (push-based OTLP export)
- BuildKit does NOT support pull-based Prometheus metrics exposition natively
- To get Prometheus metrics from BuildKit, you'd need an OTLP-to-Prometheus bridge (e.g., OpenTelemetry Collector with a Prometheus exporter)

### Fix applied
Removed both env vars and the `containerPort: 9090` (metrics) from the deployment template in `modules/buildkit/scripts/python/generate_buildkit.py`. Also removed the buildkit PodMonitor from `modules/monitoring/` since there's no metrics endpoint to scrape.

## Source
Discovered via debugging CrashLoopBackOff in production. Confirmed by reading BuildKit source code — the `prometheus` exporter type is not in the supported list.
