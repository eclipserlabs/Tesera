# Tesera publication — status

## Commit 1 — Python rename PASSED 2026-09-26

* Renamed: `src/interceptor/` → `src/tesera/` (git mv), 63 files'
  `interceptor`/`Interceptor`/`INTERCEPTOR` references → tesera/Tesera/TESERA,
  `tests/test_interceptor_features.py` → `test_tesera_features.py`, 6 systemd
  units → `tesera-*`, `pyproject.toml` (name, script `tesera`, cov, ruff,
  mypy paths, eclipserlabs/tesera URLs), `uv.lock` (member tesera 0.2.0),
  `docs/` prose, `deploy/`, `benchmarks/`, `.github/`, `TeseraError`
  (replaces `InterceptorError`, no alias), evidence home `~/.tesera`, env
  `TESERA_*`. CHANGELOG rename entry (old spellings preserved there only).
* Proof of clean: `git grep -i interceptor -- src tests docs pyproject.toml
  deploy benchmarks .github` is empty except 3 frozen vector-key seed lines
  in `verifiers/vectors/generate.py` (deliberate: changing them would alter
  committed vector keys; documented in CHANGELOG). `guardrail-evidence`
  appears only in CHANGELOG.
* Tests: 428 passed before → 428 passed after (86.68% cover). `import
  tesera` works (`tesera.__version__` reports `0.0.0+unknown` from source
  checkout, `0.2.0` installed). `import interceptor` correctly fails.
  Vectors: `test_vectors` + node + interop 24 passed; `tesera verify`
  accepts the committed basic vector journal offline.
* Untouched: `ts/` (commit 2), root `README.md` (commit 3 rewrite),
  evidence format, wire protocol, all logic.
* Human next: nothing until commits 2–4 are done; publication (commit 5)
  remains a manual human step.

## Commit 2 — TypeScript rename PASSED 2026-09-26

* Renamed: `ts/package.json` name `interceptor-effect` → `tesera` (+
  description, + repository `eclipserlabs/tesera`),
  `INTERCEPTOR_APPROVER` → `TESERA_APPROVER` (src + test unset-marker),
  matching Python's `TESERA_*` prefix. No logic changes; no import
  rewrites needed (relative imports throughout).
* Proof of clean: `grep -rni interceptor ts/src ts/test ts/scripts
  ts/package.json ts/tsconfig.json` empty. pnpm-lock untouched (no name
  reference). `ts/README.md` deferred to commit 3.
* Tests: 56 passed before → 56 passed after (`pnpm install` frozen,
  `pnpm check` clean, `pnpm test` 9 files). `pnpm build` emits package
  `tesera`. Cross-language: Python vectors verify in TS (suite) and
  TS-emitted journals verify in Python (`test_ts_interop` + vectors, 19
  passed on rebuilt dist).
* Human next: commit 3 (READMEs), then commit 4 (publication prep).

## Commit 3 — READMEs PASSED 2026-09-26

* Rewrote: root `README.md`, `ts/README.md` (plain pitch, install,
  copy-paste 30-second example, offline verify, unsoftened "not" section,
  format-as-contract, honest status ≤5-bullet plan, MIT). No badges, no
  compliance claims.
* Proof of run: Python example + `tesera verify` executed in a FRESH venv
  (`uv venv`, `uv pip install /repo` → 0.2.0 installed, example prints,
  `OK 2 events`). TS example + node verifier executed against the built
  package (prints result, `OK 2 events verified`). Cloud `make
  submodule-init && make up && make migrate && make run` re-verified with
  `healthz`/`readyz` 200 (plus a `make up` poll bugfix: HTTP 000 no longer
  counts as "answered").
* Findings for commit 4 (publication prep): (1) `ts/package.json`
  `exports` points at `./dist/index.js` but tsc emits
  `./dist/src/index.js` — must be fixed before publish or `import
  "tesera"` fails; (2) TS README's verify command needs
  `verifiers/node/verify.mjs` inside the published tarball — the `files`
  allowlist must include it (or the command must change).
* Human next: commit 4 (workflows, packaging, PUBLISHING.md). No publish yet.

## Commit 4 — publication prep PASSED 2026-09-26

* Files: `pyproject.toml` (+authors), `.github/workflows/publish-{python,npm}.yml`
  (tag-triggered `py@*`/`ts@*`, OIDC Trusted Publishing, no stored tokens),
  `ts/package.json` (name `tesera`, repository, `publishConfig.public`,
  `files` allowlist, `exports` fix, `prepack` staging of LICENSE +
  verifier), `ts/README.md` (verify path matches packaged layout),
  `.gitignore` (prepack outputs), `docs/PUBLISHING.md` (exact manual steps).
* Validation: workflows YAML-parsed + structure-asserted (no `act`
  available); `uv build` → wheel+sdist, `twine check` PASSED; wheel
  metadata (name/version/author/MIT/`>=3.10`) verified; fresh wheel
  install works (`tesera 0.2.0` + CLI). `pnpm pack` → 32 files, no
  tests/scripts/maps/raw-ts leaks; fresh tarball install runs the README
  example and the packaged verifier path end-to-end.
* Deviations: tags `py@0.2.0`/`ts@0.2.0` (not `0.1.0`): the tree IS 0.2.0,
  tagging it 0.1.0 would mislabel the code. Human may still choose 0.1.0
  at publish time by bumping the version fields first (one line each).
* Findings fixed here: TS `exports` pointed at a nonexistent
  `./dist/index.js` (tsc emits `./dist/src/index.js`) — `import "tesera"`
  would have failed on install day.
* Env note: after the folder rename (`tessera` → `tesera`), the dev
  `.venv` editable hook broke (`ModuleNotFoundError`); `uv sync` fixed
  it. If your shell shows the same, re-run `uv sync`.
* Suites: 428 Python + 56 TS pass. Human next: commit 5 manual publish
  (`twine upload`, `npm publish --access public`, Trusted Publishing
  setup), then report versions back.

## README polish — badges + cross-links PASSED 2026-09-26

* Added: CI + MIT + Python 3.10+ + Node 20+ badges (root), CI + MIT +
  Node 20+ (TS). All badges point at things that exist (workflow file,
  LICENSE, documented versions). No PyPI/npm/coverage badges — nothing
  to point at until publish.
* Added: TS cross-link, table of contents with verified anchors, fixed
  stale `tessera-cloud` reference.
* Code blocks untouched (still the executed ones from commit 3).

## Security audit phase 1 — PASS 2026-09-26

zizmor clean (0 findings, was 21 unpinned + 8 artipacked + 7 permissions + 2 cache). All actions SHA-pinned, least-privilege permissions, no caches in release jobs. Report: docs/SECURITY_AUDIT.md.
Note: org renamed eclipserlabs → sheringfords (remote updated locally).
Stale eclipserlabs URLs remain in pyproject/README/package.json — queued for the docs phase (redirects still resolve; not a security finding).
Human next: MFA + Trusted Publishing confirmation (blocks publish).

## Closeout commit 1 — eight items PASSED 2026-09-26

C1 zizmor online 0 findings. C2 pip-audit==2.10.1 pinned. C3 advisories
+ secret scanning recorded as human tasks (API state unconfirmable).
C4 real email + best-effort note. C5 two attestation steps added. C6
one-sentence disambiguation. C7 first-public note + header fix. C8 no
private links; remaining links resolve.
New: docs/PRE_PUBLISH_HUMAN_TASKS.md (20-minute checklist).
Human next: run the checklist, publish manually, report versions.

## Closeout commit 2 — final build verification PASSED 2026-09-26

* Fresh clone (`/tmp/tesera-final`) rebuilt: wheel same 35 files (only
  METADATA+RECORD differ, solely from intended README edits since the
  phase-4 build); sdist file delta is new docs only; sdist root README
  byte-identical to current. No leaks in any artifact (npm tarball: 32
  files, clean, rebuilt after the audit's Lock.ts change).
* Versions: pyproject 0.2.0, package.json 0.2.0, CHANGELOG 0.2.0. No
  `py@`/`ts@` tags exist yet — created by the human at publish time.
* README example vs fresh wheel install: exact output
  (`{'id': 're_123', 'amount_cents': 1999}`). Fresh journal verified by
  both `tesera verify --checkpoint` (3 events intact) and the Node
  verifier (3 events verified).
* Human next: commit 3 handoff docs, then manual publish.

## Closeout commit 3 — handoff PASSED 2026-09-26

PUBLISHING.md: MFA-confirm-first ordering, exact PyPI publishing URL,
sheringfords org (was eclipserlabs), twine-check step, After-publish
section (TP setup, no-tag-push rule with rationale, report-back line).
PRE_PUBLISH_HUMAN_TASKS.md verified complete (6 security + 3 publish
items, sheringfords paths). `twine check` passes on a fresh build.
No npm re-pack needed (ts/ untouched since commit 4's verified pack).
Human next: run PRE_PUBLISH_HUMAN_TASKS.md, publish, report versions.
Commit 4 (post-publish verification) waits on that report — stopping here.
