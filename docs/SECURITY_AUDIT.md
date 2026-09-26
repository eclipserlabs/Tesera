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

## Phase 3 — Supply chain and secrets — PASS 2026-09-26

Tools: gitleaks 8.30.1 (full history, `--all`), pip-audit 2.10.1,
pnpm audit, registry probes. One human decision taken mid-phase
(allowlist for fixtures, approved in writing).

### Findings

| # | Severity | Location | Finding | Disposition |
|---|---|---|---|---|
| 1 | low (false positive ×4) | git history: `src/interceptor/identity.py@cf49a8f` (identifier `mEd25519PrivateKey`), `tests/test_redaction_homoglyphs.py@e9dea77` (`msk-live-HOMOGLYPH-TEST-…`), `tests/test_maturity.py@3bb6c2c` (`sk-live-abcdefgh…` inside a redaction test), `tests/test_redaction_property.py@29ed62f` (`msk-live-PROPERTY-TEST-SECRET-…`) | Gitleaks `generic-api-key` hits on synthetic fixtures that exist to exercise the redactor (plus one code identifier). Verified each in context; none is a credential. | Human-approved allowlist in `.gitleaks.toml` (exact regexes + paths). Re-scan: 345 commits, **no leaks found**. CI should run gitleaks with this config. |
| 2 | medium (accepted) | `ts/`: vitest chain, GHSA-82fw-gwwq-j7x9 (esbuild dev-server), 2 moderate findings | Dev-dependency only (`vitest`); fix requires a vitest 3→4 major bump. The repo never runs a dev server (`vitest run` only). | Accepted risk with reasoning; Dependabot (phase 6) will surface it weekly. Below the high/critical bar. |

### Confirmations (all pass)

* Python deps (`pip-audit`, frozen venv pins incl. `cryptography`
  50.0.1): **no known vulnerabilities**.
* Typosquat: `tesera` free on PyPI and npm; neighbors (`tesara`,
  `tesero`, `teserra`, `teser4`, `teserq`) all unclaimed. PyPI
  `tessera` (double-s) is an unrelated Graphite dashboard (0.10.0) —
  discoverability note, not a collision: different name, different
  domain, long-established.
* Lockfiles: `uv.lock` + `ts/pnpm-lock.yaml` committed with integrity
  hashes; CI installs with `uv sync --frozen` /
  `pnpm install --frozen-lockfile`.
* Install scripts: no `preinstall`/`postinstall`/`install` in
  `package.json` (only `prepack`, which copies two reviewed files at
  pack time — not install time); no `setup.py`, no `cmdclass`.

## Phase 4 — Build and provenance — PASS 2026-09-26

### Findings

None. One accepted-by-design item below.

### Confirmations

* Reproducibility: wheel + sdist built twice from a clean worktree
  (`main@5347f91`), 60s+ apart — identical file lists and SHA-256 per
  file (35 wheel files, 51 sdist files). REPRODUCIBLE.
* Artifact contents: wheel has no tests/fixtures/env/git/pycache;
  sdist additionally carries only docs + `.gitignore` (tracked build
  input, benign — no secrets, no code). npm tarball: 32 files
  (`dist/src` js+d.ts, `verifiers/verify.mjs`, README, LICENSE,
  package.json); no tests/scripts/maps/raw-ts.
* Provenance: `publish-python.yml` uses `pypa/gh-action-pypi-publish`
  (attestations by default) with `id-token: write`; `publish-npm.yml`
  runs `npm publish --provenance --access public` with `id-token:
  write`. No long-lived tokens anywhere.
* `files` allowlist verified via `pnpm pack` output (see above).
* Versions: `pyproject.toml` 0.2.0, `ts/package.json` 0.2.0,
  `CHANGELOG.md` 0.2.0 — consistent.
* `0.0.0+unknown` in source checkouts: accepted by design (CHANGELOG
  documents it: a checkout without installed metadata reports the
  fallback instead of masquerading as a release). Installed packages
  report the real version; the release workflows build before publish,
  so published artifacts always carry 0.2.0.

## Phase 5 — Documentation and transparency — PASS 2026-09-26

### SECURITY.md (rewritten)

The prior file was accurate but pre-rename (`interceptor` ×3) and
missing three required sections. Now contains: private reporting
channel (advisory flow + email, no public issues), 48h acknowledgment,
30-day critical-fix timeline with announcement fallback, post-fix
advisory + reporter credit, no bounty promise, scope table (unchanged),
key handling (unchanged), latest-release-only support, and the
primitives note (audited libraries, correct usage is our job). Plain
English, no legal language.

### THREAT_MODEL.md accuracy review — pass

No stale names (renamed in the publication pass). Every load-bearing
claim sourced: decision-before-execution + fsync (engine + tests),
contract/input-hash semantics, per-event signatures, chain linkage,
tail-truncation gap with `test_truncated_tail_is_not_detectable`
(`tests/test_evidence.py:89`) pinning it, checkpoint/countersign
residuals, rotation bounds, approval-server phishing note, redaction
limits, idempotency scope (`DuplicateActionError` blocks, never replays).
No claim removed, none corrected — the model matches the code.

### README review — pass

* "What it is not": 5 bullets, each accurate, each starting with Not,
  ending at `docs/THREAT_MODEL.md`. Verified: (a) no exactly-once claim
  — crash-between-effect-and-outcome stated; (b) success ≠ external
  action — stated; (c) no tail-detection claim — detection never claimed.
* Pitch: "gates consequential function calls on approval and records
  signed evidence of each call" — no overclaim (denials record too).
