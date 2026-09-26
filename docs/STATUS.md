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
