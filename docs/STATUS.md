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
