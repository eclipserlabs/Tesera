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
