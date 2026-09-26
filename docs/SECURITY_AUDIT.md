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

## Phase 2 — Code and cryptographic review — PASS 2026-09-26

Scope: primitives, randomness, comparisons, key handling, input
validation, redaction path, crypto dependency versions. No refactors;
one minimal fix. Human review of medium+ findings: none above low.

### Findings

| # | Severity | Location | Finding | Disposition |
|---|---|---|---|---|
| 1 | low | `ts/src/Lock.ts:145` | Lock owner token mixed `Math.random()` into pid+time. File is 0600 and same-uid attackers have easier paths (direct journal rewrite), so impact is hygiene-level. | Fixed: token is now `` `${process.pid}:${randomUUID()}` `` (CSPRNG). `===` comparison kept deliberately with an in-code reason (token is stored in the lockfile; not a secret from observers). TS suite still 56/56. |
| 2 | low | journal readers (`src/tesera/verification.py`, `journal.py`) | No per-line length cap: a multi-GB single line in a local journal could exhaust memory. Journal files are operator-owned local files (self-DoS only, no remote vector). | Accepted risk: documented here. A remote journal is never parsed (ingestion is a separate, private service). |
| 3 | info | `src/tesera/identity.py` | Private key material is not zeroized after use. The `cryptography` library owns the memory; neither Python nor the API exposes wiping. | Accepted risk: standard for the ecosystem; keys live 0600 on disk regardless. |

### Confirmations (all pass)

* Primitives: Ed25519 sign/verify and SHA-256 go exclusively through
  `cryptography` (Python: `identity.py:34,309`, `verification.py:34`) and
  Node `crypto` (`generateKeyPairSync("ed25519")`, `timingSafeEqual` in
  `Witness.ts:41`). Nothing hand-rolled. 64-byte sig / 32-byte key
  enforced by the libraries; `verify_signature` maps `InvalidSignature`
  to `False` (`identity.py:302-310`).
* Randomness: `secrets.token_urlsafe` (approval/decision tokens),
  `uuid4` (event ids), `randomUUID` (TS event ids, witness tokens, lock
  tokens after fix). No `random` module, no `Math.random` remains
  (verified by grep).
* Comparisons: bearer/decision tokens via `secrets.compare_digest`
  (`approve_server.py:147,176,222`); witness checkpoint via
  `timingSafeEqual` (`Witness.ts:41`); signatures verified by the
  libraries (constant-time internally). No `==` on secrets remains.
* Key management: keys generated via lib CSPRNG, stored 0600
  (`_write_restricted`, `identity.py:528`), never logged/printed
  (CLI prints audit payloads only — no key material), rotation keeps old
  events verifiable (`trusted_keys/` + in-chain `rotation` events).
* Input validation: no `eval`/`exec`/`os.system`/`subprocess`/`pickle`/
  shell anywhere in `src/`; malformed journals yield typed errors
  (`malformed_json`, `malformed_event`); CLI bounds (`--limit`,
  `--max-events`, 1000-char notes).
* Redaction: fused redact+canonicalize traversal (`canonical.py:1-26`
  design note); `engine.py:530` hashes the canonicalized (redacted)
  arguments; value patterns + homoglyph folding covered by property tests.
* Deps: `cryptography` installed 50.0.1 (`>=42.0` floor in
  `pyproject.toml`); TS crypto is Node builtins (no third-party crypto
  dep). CVE status checked in phase 3.
