# Karpenter v1.12.0 — #9142 is bounded; stale NodeClaims are a separate NodeRepair-OFF bug

## Context
OSDC runs Karpenter **v1.12.0** (`osdc/modules/karpenter/deploy.sh:57`) with only the
`spotToSpotConsolidation` feature gate, no `NodeRepair`, no `registrationTTL`, no `expireAfter`,
and no NodePool `spec.limits`. While debugging runner queueing we needed to know precisely
whether upstream issue aws/karpenter-provider-aws **#9142** (a stuck NodeClaim blocking ALL
provisioning) could leave a NodeClaim stuck for days, and whether the fix is only in v1.13.0+.
Prior write-ups conflated #9142 with a long-lived (~4.5-day) `Ready=Unknown` NodeClaim. Verified
by reading pinned source: core `kubernetes-sigs/karpenter` v1.12.0 & v1.13.0,
`aws/karpenter-provider-aws` v1.12.0 & v1.13.0, and `awslabs/operatorpkg`@34e9a18.

## Finding

**#9142 is real on v1.12.0 but its provisioning block is time-BOUNDED and self-clearing —
it does NOT explain a NodeClaim stuck for days.** That is a different bug. Two distinct
mechanisms:

1. **#9142 (bounded, cluster-wide, recurring).** A launch failure classified as a `CreateError`
   (specifically `InstanceTypeFilteringFailed` — post-simulation `Truncate` finds no instance
   type satisfies minValues/family-diversity) leaves the NodeClaim alive with empty providerID
   and `Launched=Unknown` instead of deleting it:
   - provider `pkg/providers/instance/instance.go:305` (v1.12.0):
     `cloudprovider.NewCreateError(..., "InstanceTypeFilteringFailed", ...)`
   - core `pkg/controllers/nodeclaim/lifecycle/launch.go:107-113` sets `Launched=Unknown`,
     returns the error, does NOT delete (contrast: the ICE branch `launch.go:80-92` DOES
     `kubeClient.Delete`).
   - Any empty-providerID claim makes `cluster.Synced()` return false — core
     `pkg/controllers/state/cluster.go:158` and `:187` — and the provisioner refuses to schedule
     while unsynced — core `pkg/controllers/provisioning/provisioner.go:132-134`. So one stuck
     CreateError claim blocks ALL provisioning cluster-wide.
   - **But it self-clears:** liveness has `launchTimeout = time.Minute * 5` (core
     `pkg/controllers/nodeclaim/lifecycle/liveness.go:54`); for any claim with `Launched != True`
     it deletes the claim after 5 min (`liveness.go:84-99`) → `Synced()` recovers. Retries do
     NOT reset the clock: operatorpkg `ConditionSet.Set` preserves `LastTransitionTime` when the
     status is unchanged (`status/condition_set.go:122-125`). The #9142 issue text itself says
     the block lasts "until the retry succeeds or the 15-minute registration TTL expires."
   - **Fingerprint:** recurring ~5-min cluster-wide stalls (`cluster is waiting on sync` /
     `could not schedule pod`) correlated with a transient `InstanceTypeFilteringFailed`
     CreateError NodeClaim that has an EMPTY providerID — NOT a single days-long claim.
   - **Fix is v1.13.0-only (PR #9170, merged 2026-05-18, "Fixes #9142"):** the same line becomes
     provider `instance.go:328` `cloudprovider.NewInsufficientCapacityError(...)`, so the launch
     controller takes the delete-immediately branch. NOT backported to v1.12.x (verified by
     diffing the two clones).

2. **Stale `Ready=Unknown` claim (unbounded — the real days-long hang).** A claim that reached
   `Registered=True` (node joined) and then went unhealthy is invisible to every liveness TTL:
   liveness returns early on `registered.IsTrue()` (core `liveness.go:76-78`) and never checks
   `Initialized`. The ONLY built-in that reclaims it is the node-health/NodeRepair controller,
   which is **conditionally registered only when `FeatureGates.NodeRepair` is true** — core
   `pkg/controllers/controllers.go:155-156`. With NodeRepair OFF (OSDC's case) the controller is
   not even instantiated and there is no `expireAfter` fallback, so a `Ready=Unknown` node
   persists indefinitely. **Crucially this claim has a POPULATED providerID**, so it does NOT
   flip `Synced()` false and does NOT block cluster-wide provisioning — it merely wastes that
   node/GPU. (If instead providerID is empty, it's the #9142 case above.)

**One `kubectl describe nodeclaim <name>` disambiguates:** empty providerID + `Launched=Unknown`
+ `InstanceTypeFilteringFailed`/CreateError ⇒ #9142 (should self-clear in 5 min; a persistent
one means liveness/reconcile is itself wedged). Populated providerID + `Registered=True` +
`Ready=Unknown` ⇒ NodeRepair-OFF stale node (wasted, not a global block).

**Related durable facts confirmed from the same source:**
- ICE offering cache TTL = `3 * time.Minute`, keyed `<capacityType>:<instanceType>:<zone>` —
  provider `pkg/cache/cache.go:29-31`, `pkg/cache/unavailableofferings.go:53,77`. A NodePool
  pinned to a single instance type has NO in-pool fallback on ICE.
- `karpenter_nodepools_limit` reads empty/0 when `Spec.Limits == nil`, and `Limits.ExceededBy`
  returns nil (never throttles) — core `pkg/apis/v1/nodepool.go:175-177`,
  `pkg/controllers/metrics/nodepool/controller.go:142-146`. A stuck claim cannot throttle via
  spec.limits when no limits are set.
- Two DISTINCT log strings: Karpenter's internal provisioning simulation logs
  `"could not schedule pod"` (core `provisioning/scheduling/scheduler.go:257`, reason
  `"no instance type has enough resources"` at `scheduling/nodeclaim.go:357`) — NO EC2 call —
  vs EC2 launch ICE `"failed launching nodeclaim"` (core `nodeclaim/lifecycle/launch.go:84,96`).
  Do not read the former as an AWS stockout.

## Source
Discovered via source reading + cross-version diff of pinned clones (core v1.12.0 vs v1.13.0;
aws-provider v1.12.0 vs v1.13.0; operatorpkg@34e9a18). Issue
https://github.com/aws/karpenter-provider-aws/issues/9142 ; fix PR
https://github.com/aws/karpenter-provider-aws/pull/9170 (merged 2026-05-18, v1.13.0+ only).
KB gap noted: `actions-knowledge-base/sync.py:88` pins core at v1.10.0 and has NO
`aws/karpenter-provider-aws` entry — bump core to v1.12.0 and add the aws provider so future
agents read the deployed provider source directly.
