# GitHub Actions: retrying flaky setup steps (retry-by-duplication)

## Context
OSDC deploy / drain / undrain workflows had flaky "setup" steps — `actions/checkout`,
install `just`, install `mise`, `uv sync`, configure AWS credentials, setup QEMU — that
failed intermittently. We wanted per-step retries. GitHub Actions has **no native per-step
`retry:` keyword** (as of 2026), so we evaluated the workarounds: third-party retry actions,
hand-rolled bash loops, and retry-by-duplication.

## Finding

### Third-party retry actions don't fit a security-sensitive path
- **`nick-fields/retry`** wraps a shell `command:` ONLY — it cannot wrap a `uses:` step, so
  it's useless for `checkout`, `setup-just`, `mise-action`, etc.
- **`Wandalen/wretry.action`** CAN wrap a `uses:` action, but two blockers:
  - (a) It supports only **node/docker** actions. On a **composite** action (e.g.
    `extractions/setup-just` is `using: composite`) it hard-fails with
    `Runner "composite" does not implemented`.
  - (b) Even SHA-pinned to its v3.8.0 composite commit, that composite internally delegates
    to a **mutable tag** `Wandalen/wretry.action@v3.8.0_js_action`, so the JS that actually
    runs is **not pinned** and can be repointed upstream (tj-actions-class supply-chain risk).
- Unacceptable on a path that runs with **production AWS deploy credentials + OIDC
  `id-token: write`** and **gates the merge queue**. No unpinnable third-party code allowed.

### Retry-by-duplication (chosen pattern)
Step A gets an `id` + `continue-on-error: true`; an identical step B runs with
`if: ${{ steps.A.outcome == 'failure' }}` and **no** `continue-on-error`.
- Use **`outcome`**, not `conclusion` (which `continue-on-error` masks to `success`), and not
  `failure()` (inside a composite it checks the composite's own status, which
  `continue-on-error` already reset).
- Works for **both `run:` and `uses:` steps** and **inside composite actions** — the
  actions/runner schema permits `continue-on-error` + `if` on composite steps.
- **Fail-closed:** if both attempts fail, the job fails.
- Trade-offs: duplicated YAML (mitigate by keeping to **1 retry**); no built-in backoff (add
  a gated `sleep` step before B if needed).

### Don't double-wrap steps that already retry
- **`actions/checkout`** retries git ops internally (3 attempts, 10–20s backoff).
- **`aws-actions/configure-aws-credentials`** retries the STS assume-role internally
  (`retry-max-attempts` default 12, on by default) — wrapping it would multiply STS calls.

### OSDC decision
A local composite action **`.github/actions/osdc-setup`** installs `just` + `mise` + `uv sync`
with retry-by-duplication (1 retry each); **QEMU** is retried the same way inline in the deploy
job. No third-party retry action is used, so no unpinnable code runs on the deploy / merge-queue
path.

## Source
Source-code review of the pinned actions at their exact SHAs: `Wandalen/wretry.action`
`Retry.js` runner branch + its composite `action.yml` referencing `@v3.8.0_js_action`;
`jdx/mise-action` `action.yml` v2.4.4; `extractions/setup-just` is `using: composite`. Plus the
GitHub Actions runner metadata schema / composite-action docs (`continue-on-error` + `if`
permitted on composite steps). Discovered 2026.
