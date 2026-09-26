# Publishing (first release)

> Manual steps for the account owner. Test on TestPyPI and local
> tarballs FIRST (Steps 1-7). Real publish is Step 8 only. CI handles
> every release AFTER these one-time steps. Tags: `py@0.2.0` and
> `ts@0.2.0` (matching the `0.2.0` version in `pyproject.toml` /
> `ts/package.json`; the `0.2.0` tree is what ships, so the tags name
> it — not `0.1.0`).

## Prerequisites

- Create a TestPyPI account at
  https://test.pypi.org/account/register/ (separate from PyPI;
  credentials do not carry over).
- Enable MFA on TestPyPI (hardware key or TOTP, not SMS).
- Generate a TestPyPI API token (account-scoped, because the project
  does not exist yet). Username for upload is `__token__`.
- Create a PyPI account at https://pypi.org/account/register/ if
  needed. Enable MFA (hardware key or TOTP, not SMS).
- Confirm npm MFA and account access (`npm profile enable-2fa` —
  auth-and-writes level).
- Confirm Trusted Publishing pending publisher at
  `pypi.org/manage/account/publishing/`: owner `sheringfords`,
  repository `tesera`, workflow `publish-python.yml`, environment
  `pypi`. If the GitHub UI suggests `eclipserlabs`, ignore the
  suggestion — the canonical owner is `sheringfords` (the old name
  redirects, but Trusted Publishing must point at the canonical
  repo path).

## Step 1 — Build once

From a clean worktree:

```sh
git worktree add /tmp/tesera-clean main
cd /tmp/tesera-clean
python3 -m build
python3 -m twine check dist/*
shasum -a 256 dist/*
```

`twine check dist/*` must pass with zero warnings. Record the SHA-256
of each file in `dist/` — you will compare against it in Step 8.

If this fails: do not continue. Fix the metadata (`pyproject.toml`)
until `twine check` is clean, then rebuild.

## Step 2 — Upload to TestPyPI

TestPyPI is a distinct registry with a distinct token. Do not use your
real PyPI token here.

```sh
cd /tmp/tesera-clean
python3 -m twine upload --repository testpypi dist/*
```

When prompted: username is `__token__`, password is the TestPyPI
account-scoped token from Prerequisites.

Confirm the package page renders at
https://test.pypi.org/project/tesera/.

If this fails: check the token scope (must be account-scoped for a
first upload) and that you used `--repository testpypi`, not the real
index.

## Step 3 — Install from TestPyPI in a fresh venv

```sh
python3 -m venv /tmp/tesera-test
source /tmp/tesera-test/bin/activate
pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ tesera==0.2.0
```

`--extra-index-url https://pypi.org/simple/` is required because
TestPyPI does not mirror PyPI dependencies: `tesera` depends on
`cryptography>=42.0`, which lives only on real PyPI. The first
`--index-url` points pip at TestPyPI for `tesera` itself; the extra
index lets pip resolve its dependencies from real PyPI.

If this fails: confirm the version on the TestPyPI page matches
`0.2.0` exactly, and that the venv is fresh (no cached `tesera`).

## Step 4 — Run the README example from the TestPyPI install

Copy the 30-second example from `README.md` verbatim into a file and
run it against the TestPyPI-installed package:

```sh
source /tmp/tesera-test/bin/activate
cat > /tmp/tesera-test-readme.py <<'EOF'
from tesera import guard
from tesera.policy import AllowListProvider

allow_refunds = AllowListProvider(["billing.refund"])


@guard(action="billing.refund", risk="high", approval_provider=allow_refunds)
def refund(order_id: str, amount_cents: int) -> dict:
    return {"id": "re_123", "amount_cents": amount_cents}


print(refund("order-1", 1999))
EOF
python3 /tmp/tesera-test-readme.py
```

Expected output:

```
{'id': 're_123', 'amount_cents': 1999}
```

Then verify the produced journal:

```sh
tesera verify
```

Expected output (paths may differ):

```
OK  ~/.tesera/journal.jsonl
    2 events, signatures and hash chain intact
```

If the output differs: stop. The README example is the contract;
fix the package, rebuild, re-upload to TestPyPI, and repeat Steps
1-4.

## Step 5 — npm dry-run

```sh
pnpm --dir ts build
cd ts && npm pack --dry-run
```

Inspect the file list. Confirm only `dist/src` js+d.ts files,
`verifiers/verify.mjs`, `README.md`, `LICENSE`, and `package.json`
(32 files total). No tests, scripts, maps, or raw `.ts` sources.

Then build the local tarball:

```sh
cd ts && npm pack
```

This creates `ts/tesera-0.2.0.tgz` locally. Nothing is published.

If the file list is wrong: fix `ts/package.json` `files`, rebuild,
and re-run `--dry-run` until clean.

## Step 6 — Install the local npm tarball

```sh
mkdir -p /tmp/tesera-npm-test && cd /tmp/tesera-npm-test && npm init -y
npm install /tmp/tesera-clean/ts/tesera-0.2.0.tgz
```

Run the README TypeScript example against the installed package.
Copy `ts/README.md` 30-second example verbatim:

```sh
cat > /tmp/tesera-npm-test/run.mjs <<'EOF'
import { writeFileSync } from "node:fs";
import { Effect } from "effect";
import { Guard, Identity, Journal, Policy } from "tesera";

const program = Effect.gen(function* () {
  const journal = Journal.makeFileJournal("./journal.jsonl");
  const identity = yield* Identity.generateIdentity();
  writeFileSync("./tesera.pub.pem", identity.publicKeyPem);
  const budget = yield* Policy.BudgetProvider.make(1000, false);

  const refund = Guard.guard({
    action: "billing.refund",
    journal,
    approve: (request) => budget.decide(request),
    identity,
  })((orderId, amountCents) => ({ id: "re_123", orderId, amountCents }));

  return yield* refund("order-1", 1999);
});

Effect.runPromise(program).then(
  (result) => console.log(result),
  (error) => { console.error(String(error)); process.exit(1); },
);
EOF
cd /tmp/tesera-npm-test && node run.mjs
```

Expected output:

```
{ id: 're_123', orderId: 'order-1', amountCents: 1999 }
```

Then run the packaged verifier:

```sh
cd /tmp/tesera-npm-test && node ./node_modules/tesera/verifiers/verify.mjs --journal ./journal.jsonl --public-key ./tesera.pub.pem
```

Expected output:

```
OK  ./journal.jsonl
    2 events verified (independent node verifier)
```

If this fails: fix `ts/` packaging, rebuild, re-pack, and repeat.

## Step 7 — Cleanup

- If TestPyPI reveals a metadata issue, fix it, rebuild, and
  re-upload. TestPyPI allows re-upload of the same version if you
  delete the release first (release page → Settings → Delete), or you
  can bump to `0.2.1` and iterate.
- If npm dry-run reveals an issue, fix it and rebuild.
- Once clean, delete the TestPyPI release and the local tarball
  before the real publish. The real publish must be from a clean
  build:

```sh
rm /tmp/tesera-clean/ts/tesera-0.2.0.tgz
rm -rf /tmp/tesera-test /tmp/tesera-npm-test /tmp/tesera-test-readme.py
```

## Step 8 — Real publish (only after Steps 1-7 are clean)

Rebuild from a clean worktree. Record the new SHA-256:

```sh
git worktree add /tmp/tesera-real main
cd /tmp/tesera-real
python3 -m build
python3 -m twine check dist/*
shasum -a 256 dist/*
```

Confirm the SHA-256 matches the tested build from Step 1. If it
differs, do not publish — investigate first (source, tool, or
environment drift).

Publish Python (real PyPI, real account-scoped token):

```sh
cd /tmp/tesera-real
python3 -m twine upload dist/*
```

Confirm at https://pypi.org/project/tesera/.

Publish TypeScript (real npm):

```sh
cd /tmp/tesera-real/ts && npm publish --access public
```

Confirm at https://www.npmjs.com/package/tesera.

Report the published versions back to the agent:

```
Published. Python <version>, TypeScript <version>.
```

## After publish

1. Configure Trusted Publishing for the package on both registries
   (PyPI: `pypi.org/manage/account/publishing/`; npm: package Settings
   → Trusted Publisher), pointing at repo `sheringfords/tesera` and the
   two workflow files — so the NEXT release publishes from a tag.
   If the GitHub UI suggests `eclipserlabs`, ignore the suggestion —
   the canonical owner is `sheringfords`.
2. Do NOT push `py@*`/`ts@*` tags for this release: the release
   workflows trigger on tags and would fail against the just-published
   version. Tag-driven publishing starts with the next version bump.
3. Report to the agent: "Published. Python \<version\>, TypeScript
   \<version\>." The agent will then run post-publish verification
   (fresh installs, README example, both verifiers, provenance) and
   write `docs/PUBLICATION_VERIFIED.md`.

## If a name is taken

Stop. Do not silently pick an alternative. The availability check in
`docs/NAMING_DECISION.md` was done pre-rename; re-check, propose, and
get written confirmation before touching a package name.
