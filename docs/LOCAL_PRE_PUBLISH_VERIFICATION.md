# Local pre-publish verification

Record of everything the agent verified locally before handing the
TestPyPI sequence to the human. No artifact was uploaded or published.

## 1. Reproducible builds

Two separate clean checkouts (`git worktree` + fresh `git clone`),
each `python3 -m build`, at 2026-09-26T07:5xZ:

| File | SHA-256 (both builds identical) |
|---|---|
| `tesera-0.2.0-py3-none-any.whl` | `f9f66576228b9c15b5629ac2b18dd2cded04a66df0a456b1098c1d4c1866ecd1` |
| `tesera-0.2.0.tar.gz` | `1bbbb5d54f605bca8ab72da29cad8f00b0ff14ccab1b7f14b4bded3a240e843d` |

A third build from the working checkout at 2026-09-26T07:59:49Z
produced byte-identical SHAs. File lists compared equal (wheel: 35
files; `METADATA`+`RECORD` differ only where intended).

`python3 -m twine check dist/*`: PASSED, zero warnings, both files.

Note: the sdist hash changed since the closeout-2 build because the
sdist vendors `docs/` (the PUBLISHING.md rewrite is intentionally
included). The wheel hash is unaffected (it ships only `src/tesera`).

## 2. Local wheel install + README example

Fresh venv, `pip install dist/tesera-0.2.0-py3-none-any.whl`
(pulled `cryptography` from PyPI as declared):

```
Successfully installed cffi-2.1.1 cryptography-50.0.1 pycparser-3.0 tesera-0.2.0
```

`pip show tesera`: `Name: tesera, Version: 0.2.0`.

README 30-second example (verbatim, `TESERA_EVIDENCE_HOME` isolated):

```
{'id': 're_123', 'amount_cents': 1999}
```

`tesera verify`:

```
OK  /tmp/tesera-py-home/journal.jsonl
    2 events, signatures and hash chain intact
    note: tail truncation is detectable only with a checkpoint witness
```

## 3. Local npm tarball install + TypeScript example

`pnpm --dir ts build` clean. `npm pack --dry-run`: 32 files
(`dist/src` js+d.ts, `verifiers/verify.mjs`, `README.md`, `LICENSE`,
`package.json`), no tests/scripts/maps/raw-ts leaks. `npm pack`:

```
tesera-0.2.0.tgz
SHA-256: efe351359f2e453e30ff2ea4b52bf81994b2f8456b6ea4f0b544a0077ad0481b
```

Fresh project, `npm install /path/to/ts/tesera-0.2.0.tgz`: 0
vulnerabilities. README TypeScript example (verbatim):

```
{ id: 're_123', orderId: 'order-1', amountCents: 1999 }
```

Packaged verifier:

```
OK  ./journal.jsonl
    2 events verified (independent node verifier)
```

Environment note: `pnpm` invoked inside `/tmp` git worktrees resolves
via corepack to pnpm 11.22.0, which crashes on Node 20.19.5
(`ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING`) before reading the
lockfile. This is a container quirk, not a source defect: the same
commands succeed in the working checkout (standalone pnpm 10.12.1).
The authoritative TS build above is from the working checkout, which
was clean (`git status` empty apart from the intended doc edits).

## 4. Cross-verification (both directions)

Python journal (2 events, from the local wheel) checked with the
dependency-free Node verifier:

```
OK  /tmp/tesera-py-home/journal.jsonl
    2 events verified (independent node verifier)
```

TypeScript journal (2 events, from the local tarball) checked with
the Python verifier:

```
OK  /tmp/tesera-npm-proj/journal.jsonl
    2 events, signatures and hash chain intact
    note: tail truncation is detectable only with a checkpoint witness
```

Both directions pass. The evidence format is the contract; both SDKs
agree.

## 5. What the agent did NOT do

- No upload to TestPyPI.
- No publish to PyPI or npm.
- No workflow, evidence-format, or crypto changes.
- No git history rewrite. No new dependencies.

Timestamps are UTC. SHAs above are the tested baseline: the human's
Step 8 rebuild must produce identical SHAs before real publish.
