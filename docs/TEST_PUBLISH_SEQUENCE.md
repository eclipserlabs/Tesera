# Test publish sequence (human script)

Single numbered list. Run in order, verbatim, in under 20 minutes.
Each command is prefixed with the account/registry it uses.
Nothing here publishes to the real PyPI or npm.

Prerequisite: TestPyPI account + MFA + account-scoped token ready
(see `docs/PRE_PUBLISH_HUMAN_TASKS.md`). The agent has already run
every command below that can run without your credentials; only the
authenticated uploads need you.

## 1. [local] Build once and check

```sh
git worktree add /tmp/tesera-clean main
cd /tmp/tesera-clean
python3 -m build
python3 -m twine check dist/*
shasum -a 256 dist/*
```

Expected: `twine check` PASSED, zero warnings, for both files.
Record both SHA-256 values.

If this fails: stop, report the `twine check` output to the agent.
Do not continue. Fix is metadata, not credentials.

## 2. [TestPyPI: `__token__` + TestPyPI token] Upload

```sh
cd /tmp/tesera-clean
python3 -m twine upload --repository testpypi dist/*
```

Username: `__token__`. Password: your TestPyPI account-scoped token
(not your PyPI token — TestPyPI is a separate registry).

Expected: upload completes, package page renders at
https://test.pypi.org/project/tesera/.

If this fails: check (a) token is TestPyPI (not PyPI), (b) scope is
account-scoped (project does not exist yet), (c) you typed
`--repository testpypi`. Report HTTP errors to the agent verbatim.

## 3. [TestPyPI + PyPI (deps)] Install in a fresh venv

```sh
python3 -m venv /tmp/tesera-test
source /tmp/tesera-test/bin/activate
pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ tesera==0.2.0
```

Expected: `Successfully installed ... tesera-0.2.0` (plus
`cryptography` from real PyPI — TestPyPI does not mirror
dependencies, hence `--extra-index-url`).

If this fails: confirm the TestPyPI page shows `0.2.0`, and the venv
is fresh. Report pip errors verbatim.

## 4. [TestPyPI install] README example + verify

```sh
source /tmp/tesera-test/bin/activate
export TESERA_EVIDENCE_HOME=/tmp/tesera-test-home
mkdir -p /tmp/tesera-test-home
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
tesera verify
```

Expected stdout: `{'id': 're_123', 'amount_cents': 1999}`.
Expected verify:

```
OK  /tmp/tesera-test-home/journal.jsonl
    2 events, signatures and hash chain intact
```

If this fails: stop. Report both outputs to the agent. The README
example is the contract — no publish until it matches.

## 5. [local, npm account not needed] npm dry-run + tarball

```sh
pnpm --dir ts build
cd ts && npm pack --dry-run
cd ts && npm pack
```

(Adjust `cd` if you built from `/tmp/tesera-clean`: the tarball lands
in the `ts/` directory you packed from.)

Expected: 32 files (`dist/src` js+d.ts, `verifiers/verify.mjs`,
`README.md`, `LICENSE`, `package.json`). Nothing published.

If the file list is wrong: stop, report the list to the agent.

## 6. [local tarball] Install + TypeScript example + verifier

```sh
mkdir -p /tmp/tesera-npm-test && cd /tmp/tesera-npm-test && npm init -y
npm install /tmp/tesera-clean/ts/tesera-0.2.0.tgz
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
node ./node_modules/tesera/verifiers/verify.mjs --journal ./journal.jsonl --public-key ./tesera.pub.pem
```

(If your tarball path differs from `/tmp/tesera-clean/ts/`, substitute
the actual path in the `npm install` line.)

Expected run output: `{ id: 're_123', orderId: 'order-1',
amountCents: 1999 }`.
Expected verifier output:

```
OK  ./journal.jsonl
    2 events verified (independent node verifier)
```

If this fails: stop, report to the agent.

## 7. [local] Cleanup

```sh
rm -f /tmp/tesera-clean/ts/tesera-0.2.0.tgz
rm -rf /tmp/tesera-test /tmp/tesera-npm-test /tmp/tesera-test-readme.py /tmp/tesera-test-home
```

Then decide:

- Metadata issue found → fix, rebuild, re-upload to TestPyPI
  (delete the TestPyPI release first, or bump to `0.2.1`). Repeat
  Steps 1-4.
- npm issue found → fix, rebuild, repeat Steps 5-6.
- All clean → delete the TestPyPI release (release page → Settings
  → Delete) and proceed to Step 8 of `docs/PUBLISHING.md` (real
  publish from a clean rebuild, with SHA-256 comparison).

## Ready-for-real-publish checklist

- [ ] `twine check` passed with zero warnings on the tested build.
- [ ] TestPyPI page rendered correctly.
- [ ] Fresh-venv install from TestPyPI succeeded.
- [ ] README Python example printed the documented output.
- [ ] `tesera verify` reported 2 events intact.
- [ ] `npm pack --dry-run` showed only the 32 intended files.
- [ ] Local tarball installed and ran the TypeScript example.
- [ ] Packaged Node verifier reported 2 events verified.
- [ ] SHAs of the tested build recorded for Step 8 comparison.
- [ ] TestPyPI release deleted; local tarball and test venvs removed.

Report to the agent: which boxes are checked, the two SHA-256 values
from Step 1, and either "ready for real publish" or the exact failure
output.
