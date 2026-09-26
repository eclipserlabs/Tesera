# Release checklist

Every item must pass before tagging `py@*` or `ts@*`. If any item
fails, the release waits. No exceptions for cryptographic code.

## Tests and vectors

- [ ] `uv run --frozen pytest -q` passes (coverage gate in `pyproject.toml`).
- [ ] `pnpm --dir ts check && pnpm --dir ts test` pass (tsc + all vitest suites).
- [ ] Cross-language vectors green both directions
      (`tests/test_vectors.py`, `tests/test_ts_interop.py`,
      `ts/test/vectors.test.ts`).
- [ ] No test was weakened or skipped to pass (check the diff, not just green).

## Dependencies

- [ ] `pip-audit --strict` clean for high/critical (see weekly scan issues).
- [ ] `pnpm --dir ts audit --audit-level=high` clean for high/critical.
- [ ] `uv.lock` / `ts/pnpm-lock.yaml` committed and current (`--frozen`
      installs succeed from a clean checkout).

## Docs and version

- [ ] `CHANGELOG.md` has an entry for this release (newest first).
- [ ] `pyproject.toml` / `ts/package.json` versions bumped together and
      equal; the tag matches (`py@<version>`, `ts@<version>`).
- [ ] README examples still run as written (re-run in a fresh venv if
      any example changed).
- [ ] `docs/SECURITY_AUDIT.md` has no unresolved high/critical finding.

## Publish and provenance

- [ ] Tag pushed; the publish workflow ran green.
- [ ] PyPI/npm show the new version with signed provenance/attestations.
- [ ] Fresh-environment install verified (`pip install tesera==<v>`,
      `npm install tesera@<v>`) and recorded in
      `docs/PUBLICATION_VERIFIED.md`.
