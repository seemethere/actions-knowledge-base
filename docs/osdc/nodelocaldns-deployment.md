# NodeLocal DNSCache deployment in OSDC

## Why this exists

The IPv4 cutover plan (`INCREASE_IPV4.md` in the OSDC repo) scales the cluster from ~7,500 concurrent runner pods today toward ~15,000 at burst. At ~5 DNS QPS/pod that is ~75k QPS aimed at a 2-replica cluster CoreDNS — far over what those replicas can answer. NodeLocal DNSCache (NLD) absorbs ~70-95% of that load locally on every node, dropping cluster CoreDNS to ~10-15k QPS where a small replica count survives.

NLD is a cutover prerequisite. It must run on the existing fleet for ≥3 days (preferably 7) of stable operation, including a planned mid-soak DaemonSet rollout, before the IPv4 cutover PRs are allowed to schedule.

## Architecture: iptables-mode (NOT IPVS-mode)

NLD launches with `-localip 169.254.20.10,${KUBE_DNS_CLUSTER_IP}`. The binary creates a dummy `nodelocaldns` interface, binds BOTH the link-local address and the cluster's kube-dns Service ClusterIP, then inserts NOTRACK rules in iptables' `raw` table for traffic destined to either address.

Pods continue resolving via the kube-dns Service ClusterIP — kubelet's `--cluster-dns` is deliberately UNCHANGED. NLD intercepts those queries via iptables.

The reason: if the NLD pod dies, traffic to the kube-dns ClusterIP falls through to cluster CoreDNS via the unchanged kube-dns Service Endpoints. With IPVS-mode (kubelet pointing at the link-local IP), an NLD pod failure causes total per-node DNS outage with no fallback. Iptables-mode trades a small interception layer for graceful degradation.

## Key design decisions

- **No `preStop` hook.** Bare `iptables -F` flushes the entire shared `filter` table and breaks kube-proxy / aws-node / cache-enforcer rules cluster-wide. Targeted flush (`iptables -t raw -F NODELOCAL_DNS`) is fragile because chain names depend on NLD version, and the binary already re-asserts rules on startup. The binary's in-process teardown handles graceful SIGTERM cleanup. `terminationGracePeriodSeconds: 30` (NOT the spec's original 5s) gives the cleanup loop time to run.
- **No memory limit, no CPU limit.** Strong upstream consensus. OOMKill orphans iptables → cluster-wide DNS degradation until pod restart. `priorityClassName: system-node-critical` already prevents eviction. CPU throttling = DNS latency. Set requests only (`cpu: 25m`, `memory: 100Mi`).
- **Image pinned to `1.26.8`** (`registry.k8s.io/dns/k8s-dns-node-cache:1.26.8`, includes CVE-2025-22870 fix). Not pre-mirrored — flows lazily through the Harbor proxy cache. The cache-enforcer DaemonSet rejects direct egress to `registry.k8s.io`; the registry-mirror-config DaemonSet writes containerd `hosts.toml` rewriting pulls to Harbor. Cold-pull race accepted (see Operational gotchas).
- **Explicit tolerations list, NOT `[{operator: Exists}]`.** `Exists` would tolerate `karpenter.sh/unschedulable` (blocks Karpenter consolidation) and other system taints. The list (`CriticalAddonsOnly`, `nvidia.com/gpu`, `node-fleet`, `instance-type`, `git-cache-not-ready`) is a superset of known node taints. If a smoke test fails because a base node carries a taint not on this list, add the taint.
- **`maxUnavailable: 1%`.** At 700 nodes that's 7 nodes cycling at a time. 10% would be 70 nodes simultaneously falling through to cluster CoreDNS during a rollout — a 70-node QPS spike on top of normal load. On staging (~10 nodes) `1%` rounds down to 0 which Kubernetes coerces to 1; behavior is correct on both clusters.
- **Liveness probe tuned for 75s.** `initialDelaySeconds: 60, periodSeconds: 15, timeoutSeconds: 5, failureThreshold: 5`. Default (3 × 10s × 1s = 30s) is too tight for ConfigMap reload windows + dummy-interface setup latency. Probe target is `169.254.20.10:8080/health` — proves both the dummy interface AND the binary are up. The `health 169.254.20.10:8080` directive is declared ONLY in the `cluster.local:53` block (matches upstream `nodelocaldns.yaml`). Per CoreDNS plugin/health docs the listener is a singleton: declaring the same address in a second server block risks "address already in use" at startup. Implication: if the `cluster.local` block fails to load (e.g., PILLAR-substitution race / Corefile parse error), the `:8080` endpoint won't bind, kubelet's liveness probe fails, the pod restarts, and `deploy.sh` re-runs substitution. This is the desired behavior — a broken Corefile parse is a serious problem worth restarting on.

## Dynamic kube-dns ClusterIP substitution

The cluster service CIDR is dynamic per cluster (default-CIDR EKS = `10.100.0.0/16`, kube-dns at `10.100.0.10`, but operators sometimes override `--service-cidr`). The kube-dns ClusterIP must be baked into both the Corefile (`bind 169.254.20.10 <ip>`) and the DaemonSet args (`-localip 169.254.20.10,<ip>`).

Solution: `base/kubernetes/nodelocaldns/deploy.sh` renders the kustomization, resolves `kube-dns` Service ClusterIP from the live cluster via `kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIP}'`, then sed-substitutes `__KUBE_DNS_CLUSTER_IP__` into the rendered manifest before applying.

This is why `nodelocaldns/` is NOT included in `base/kubernetes/kustomization.yaml`'s `resources:` list — a literal `__KUBE_DNS_CLUSTER_IP__` placeholder in a static manifest would fail kubeconform on any field typed as IP. Pattern matches the `image-cache-janitor` precedent (also has its own `deploy.sh`, also excluded from the parent kustomization). A comment in the parent kustomization documents the exclusion.

## Service-before-DaemonSet ordering

The NLD binary reads `KUBE_DNS_UPSTREAM_SERVICE_HOST` to discover the `kube-dns-upstream` Service ClusterIP. Kubelet only injects Service env vars into pods for Services that exist at pod-create time. If the DaemonSet is created before the Service, NLD pods come up with no `KUBE_DNS_UPSTREAM_SERVICE_HOST` set and fail their setup loop.

`deploy.sh` applies Services first (`kube-dns-upstream` and `node-local-dns-metrics`), waits briefly, then applies the rest (ServiceAccount, ConfigMap, DaemonSet). On first deploy where the Services were freshly created, `kubectl rollout restart ds node-local-dns` is issued as a safety net to force pods to come up with the env var injected.

A precondition check in deploy.sh fails loudly if `kube-dns-upstream` already exists with a wrong selector — guards against silent overwrite of a pre-existing install.

## Two metrics ports: `:9253` and `:9353`

There are TWO metrics endpoints and they are commonly conflated.

- **`:9253`** — CoreDNS `prometheus` plugin, configured per-server-block in the Corefile. Exposes per-zone request/response metrics from the CoreDNS plugin chain.
- **`:9353`** — the NLD binary's own metrics endpoint (default `-metrics-listen-address`). This is where `coredns_nodecache_setup_errors_total` lives — the metric that fires when iptables NOTRACK rule installation fails. The metric is NOT named `nodelocaldns_setup_errors_total`; this is a frequent docs error (the spec for this PR had the wrong name and the alert was corrected during implementation).

Upstream's reference `nodelocaldns.yaml` only exposes `:9253`, leaving the binary's setup-errors metric orphan. OSDC exposes BOTH ports as separate container ports (`metrics`/9253, `metrics-binary`/9353) and scrapes BOTH via two `podMetricsEndpoints` in the same PodMonitor.

## Operational gotchas

- **NLD pod failure is NOT a per-node DNS outage.** Traffic to the kube-dns ClusterIP falls through to cluster CoreDNS via the unchanged kube-dns Service Endpoints. The fallback is transparent to pods.
- **Brief startup race on fresh Karpenter nodes.** ~10-30s window before NLD's iptables NOTRACK rules are programmed. Pods scheduled onto the new node during that window resolve via cluster CoreDNS directly. Iptables-mode fallthrough handles this transparently — no startup taint required.
- **Cold-pull race on the FIRST pull on a brand-new node.** ~30-60s of `ImagePullBackOff` until the registry-mirror-config DaemonSet writes containerd `hosts.toml` and Harbor lazy-pulls the image. NLD is `system-node-critical`, the race resolves in seconds-to-minute. Accepted trade-off versus building new pre-mirror infrastructure.
- **iptables-nft vs legacy backend.** AL2023 uses nft; NLD's bundled iptables binary claims auto-detection. No known industry issues at scale, but worth watching during the soak window.
- **`force_tcp` to upstream multiplies persistent TCP connections.** Each node holds 1 TCP conn per upstream IP across all server blocks (cluster.local / in-addr / ip6 share one). 700 nodes × 2 cluster CoreDNS replicas = ~350 conns per replica. Within CoreDNS' default `max_concurrent: 1000` but tight. PR 13a (separate, independent) raises CoreDNS to 6 replicas and resolves the concern.

## Soak signal

NLD must run ≥3 days (preferably 7) on the existing fleet, including a planned mid-soak DaemonSet rollout, with no sustained DNS errors and no `coredns_nodecache_setup_errors_total` increases, before the IPv4 cutover PR sequence in `INCREASE_IPV4.md` is allowed to schedule. The mid-soak rollout proves that `maxUnavailable: 1%` does not produce a measurable cluster CoreDNS QPS spike during pod cycling.

## Verification

Confirm NLD is actually answering queries:

- DaemonSet rolled out everywhere: `kubectl get ds -n kube-system node-local-dns` (DESIRED == CURRENT == READY == AVAILABLE).
- Setup errors metric clean: query `coredns_nodecache_setup_errors_total` in Mimir; expect 0 across the fleet.
- Restart counts low: `kubectl get pods -n kube-system -l k8s-app=node-local-dns --no-headers | awk '{print $4}' | sort | uniq -c` — expect almost all 0.
- End-to-end probe from any pod with hostNetwork (or from a debug pod): `dig @169.254.20.10 kubernetes.default.svc.cluster.local +short` — expect a non-empty Service ClusterIP.
- Cluster CoreDNS QPS dropped: compare `coredns_dns_requests_total` rate before vs after — expect ~70-95% reduction.

## Files (in OSDC repo)

- `base/kubernetes/nodelocaldns/` — kustomization, manifests (serviceaccount, daemonset, configmap, upstream-service, metrics-service), and `deploy.sh`
- `base/kubernetes/kustomization.yaml` — comment documents why `nodelocaldns/` is excluded
- `justfile` — `deploy-base` recipe invokes `nodelocaldns/deploy.sh`
- `modules/monitoring/kubernetes/monitors/podmonitors/nodelocaldns.yaml` — PodMonitor with two endpoints (`:9253` and `:9353`)
- `modules/monitoring/kubernetes/alerts/nodelocaldns-alerts.yaml` — `NodeLocalDNSSetupErrors`, `NodeLocalDNSPodRestarting`, `NodeLocalDNSDaemonSetDegraded`
- `base/kubernetes/tests/smoke/test_base_kubernetes.py` — DaemonSet health, headless metrics service, upstream service selector

## Related docs

- `INCREASE_IPV4.md` (in OSDC repo) — broader cutover plan that NLD is a prerequisite for
- (sibling) `numa-topology-gpu-fleet-unification.md` — separate single-fleet decision

## Source

Synthesized from PR 13b spec (`PR13b.md`) and the implementation plan (Phase 1-3 research). Upstream NLD source pinned in `repos/dns/` at v1.26.8.
