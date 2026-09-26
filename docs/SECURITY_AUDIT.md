# Tesera security audit (pre-publication)

> One section per phase. Every finding: severity, file:line, disposition
> (fixed / accepted / deferred with issue link). No high or critical
> finding remains unresolved at publication.

## Phase 1 — Account and publishing security — PASS 2026-09-26

Tools: zizmor 1.30.1 (offline mode; online-only audits unavailable and
noted), manual permission review. `pull_request_target`: absent in all
workflows. Unquoted input interpolation into shell: none found.

### Findings (all fixed)

| # | Severity | Rule | Location | Disposition |
|---|---|---|---|---|
| 1 | high | unpinned-uses (21×) | all `uses:` in `ci.yml`, `publish-python.yml`, `publish-npm.yml` | Fixed: every action pinned by commit SHA with the prior tag kept as a trailing comment (e.g. `actions/checkout@11d5960… # v4`). SHAs resolved via the actions' repos on 2026-09-26. |
| 2 | low | artipacked (8×) | every `actions/checkout` step | Fixed: `persist-credentials: false` on all 8 checkout steps (CI has no push/authenticated step, so nothing needs the token). |
| 3 | medium | excessive-permissions (7×) | `ci.yml` (no block at all → broad default) | Fixed: workflow-level `permissions: contents: read`. Checkout is the only permission consumer. |
| 4 | low | cache-poisoning (2×) | `publish-npm.yml:29` (`setup-node`), `publish-python.yml:26` (`setup-uv`) | Fixed: caches disabled in release jobs (`package-manager-cache: false`, `enable-cache: false`). Release builds are hermetic; CI keeps caches (zizmor does not flag non-publishing workflows). |

Final state: `zizmor .github/workflows/` reports **0 findings**.

### Token permissions

| Workflow | Declared | Required | Verdict |
|---|---|---|---|
| `ci.yml` (all jobs) | `contents: read` (workflow level) | `contents: read` (checkout only) | pass |
| `publish-python.yml` build | inherits `contents: read` | `contents: read` | pass |
| `publish-python.yml` publish | `contents: read` + `id-token: write` | exactly those (OIDC to PyPI) | pass |
| `publish-npm.yml` publish | `contents: read` + `id-token: write` | exactly those (OIDC to npm) | pass |

No `write-all`, no broad scopes, no long-lived tokens in any workflow.

### Human tasks (blocking publication, not this phase)

MFA (hardware key or TOTP, not SMS) on PyPI + npm, dedicated emails,
unique 20+ char passwords in a manager, and Trusted Publishing pointed
at this repo + the two workflow filenames. The human has NOT yet
confirmed these — publication remains blocked on that confirmation
regardless of audit progress.
