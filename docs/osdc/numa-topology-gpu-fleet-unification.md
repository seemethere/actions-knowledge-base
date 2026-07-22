# NUMA Topology and GPU Fleet Unification

## Context
OSDC unified GPU sub-fleets (g5-1gpu, g5-4gpu, g5-8gpu) into single fleets per GPU family (g5), relying on Kubernetes nvidia.com/gpu resource scheduling instead of fleet-level taint isolation. During design review, we investigated how NUMA topology interacts with mixed GPU packing.

## Finding
All Karpenter-managed nodes (GPU and CPU) use `topologyManagerPolicy: restricted` with `topologyManagerScope: container` and `prefer-closest-numa-nodes: true`. This is set by the NodePool generator's default at `modules/nodepools/scripts/python/generate_nodepools.py` line 129.

With `restricted` policy, the kubelet's topology manager rejects pod admission when GPU resources cannot be NUMA-aligned. On multi-GPU nodes (e.g., g5.48xlarge with 8 GPUs across 4 NUMA domains, 2 GPUs per NUMA), partial GPU allocation from scattered 1-GPU jobs can fragment the NUMA layout, making subsequent multi-GPU requests impossible to NUMA-align even when raw GPU count says there's room.

### The livelock scenario
1. Four 1-GPU jobs each take 1 GPU from different NUMA domains (GPUs 0, 2, 4, 6)
2. A 4-GPU job arrives. Node reports 4 GPUs free (1, 3, 5, 7) — one per NUMA domain
3. Kubelet topology manager cannot NUMA-align 4 scattered GPUs → TopologyAffinityError
4. Pod fails (terminal), controller creates replacement
5. Scheduler sees same node still has 4 "free" GPUs → binds again → fails again
6. Karpenter never provisions a new node because its IsProvisionable() only triggers on pods with PodScheduled=False/Reason=Unschedulable. The scheduler DID bind the pod, so Karpenter ignores it.

### Source code evidence
- Karpenter `IsProvisionable()`: `repos/karpenter/pkg/utils/pod/scheduling.go` lines 94-106 — requires `FailedToSchedule()` which checks for `PodReasonUnschedulable`
- Karpenter scheduling simulation has zero NUMA awareness — the word "NUMA" doesn't appear in the Karpenter source
- NVIDIA device plugin reports NUMA affinity per GPU: `repos/k8s-device-plugin/internal/rm/nvml_devices.go` line 127

### Why CPU unified fleets don't have this problem
CPU cores are plentiful per NUMA domain (dozens), so CPU requests are almost always NUMA-satisfiable regardless of fragmentation. GPUs are scarce (2 per NUMA on g5.48xlarge), so fragmentation hits quickly.

## Decision
Proceed with unified GPU fleets. Monitor for TopologyAffinityError events post-deployment. If observed, switch affected NodePools to `topologyManagerPolicy: best-effort` (one-line YAML change per def file). `best-effort` still prefers NUMA alignment but falls back to cross-NUMA allocation instead of rejecting.

### Mitigation options if NUMA livelock occurs
1. **best-effort policy** — eliminates livelock, pods get cross-NUMA GPUs when alignment impossible. Simplest fix.
2. **Topology-Aware Scheduling plugin** (NodeResourceTopology) — makes kube-scheduler NUMA-aware so it won't bind to nodes where alignment fails. Turns failure into proper Unschedulable condition Karpenter can react to. Adds operational complexity.
3. **Tiered merge** — combine 1+4 GPU fleets, keep 8-GPU separate. Reduces fragmentation surface.

## Source
Discovered via source code analysis of Karpenter, kubelet topology manager, and NVIDIA device plugin during GPU fleet unification design review (April 2026).
