# Tesera

[![CI](https://github.com/eclipserlabs/tesera/actions/workflows/ci.yml/badge.svg)](https://github.com/eclipserlabs/tesera/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/github/license/eclipserlabs/tesera)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue)](https://www.python.org/)
[![Node 20+](https://img.shields.io/badge/node-20%2B-green)](https://nodejs.org/)

Tesera is a Python library that gates consequential function calls on approval and records signed evidence of each call.

The package is `tesera` (single 's'), not to be confused with `tessera` (double 's'), an unrelated Graphite dashboard on PyPI.

## The problem this solves

A refund function runs, the network times out, and the process exits
before writing a result. The money may have moved. The logs say the call
started. Nobody can say whether it finished. Retrying risks charging
twice; not retrying risks never refunding — and the argument about what
happened starts from zero.

Structured logging and tracing record what the process printed, not what
it did. They live in the same trust domain as the code that acted, and
nothing stops later edits. When two systems disagree about a payment, a
log line settles nothing.

Orchestrators answer a different question: they track workflow state, not
whether an external side effect happened. The gap is a record, made
before and after the call, that survives the argument about what happened.

Tesera closes that gap by requiring approval before consequential calls run and writing hash-chained, Ed25519-signed evidence of the decision and the outcome.

## Install

```sh
pip install tesera
```

Requires Python 3.10+.

## Quick example

```python
from tesera import guard
from tesera.policy import AllowListProvider

allow_refunds = AllowListProvider(["billing.refund"])


@guard(action="billing.refund", risk="high", approval_provider=allow_refunds)
def refund(order_id: str, amount_cents: int) -> dict:
    return {"id": "re_123", "amount_cents": amount_cents}


print(refund("order-1", 1999))
```

Output:

```
{'id': 're_123', 'amount_cents': 1999}
```

Two things to notice. First, the refund ran only because the allow-list
approved it: the journal's decision record says `allowed` with the reason
`allowed by policy predicate`, written before the function executed.
Second, the outcome record captures what the function returned, chained
to that decision — so approval and result are one checkable unit, not two
separate logs.

## Verify the evidence

Commit a checkpoint witness, then verify against it:

```sh
tesera checkpoint
tesera verify --checkpoint ~/.tesera/journal.jsonl.checkpoint
```

Output:

```
OK  ~/.tesera/journal.jsonl
    3 events, signatures and hash chain intact
    checkpoint witness applied; truncation before it is detected
    witness age: 0s
```

Verification needs only the journal and the public key with no account, server, or network.

## What it is not

- Not a workflow engine. Use Temporal, Airflow, or Cadence for orchestration.
- Not a replacement for Stripe idempotency or provider-level guarantees.
- Not a guarantee of exactly-once execution. A crash between the side effect and the outcome record still needs a human to check the provider.
- Not an LLM in the loop. Verification is deterministic, always.
- Not proof the external world changed. A recorded success means the function returned, not that the payment processor acted.

The full boundary is in `docs/THREAT_MODEL.md`.

## Evidence format

The format is the contract, not the library: the journal is specified in `docs/EVIDENCE_FORMAT.md`, and any implementation in any language can produce or verify it. A dependency-free Node verifier lives in `verifiers/node/verify.mjs`, committed cross-language vectors in `verifiers/vectors/v1/` are checked by both implementations, and the TypeScript sibling is published as `tesera` on npm (see `ts/README.md`).

## Status

Works today: approval-gated `@guard` with composable policy providers; signed, hash-chained journals; offline `verify`, `audit`, `inspect`, and `export`; idempotency keys; dry runs; provider receipts; checkpoints, countersignatures, key rotation, and archive rotation.

Does not exist yet: the `tesera` 0.2.0 release on PyPI; the `tesera` 0.2.0 release on npm; a public hosted witness service (the cloud backend is private and not accepting traffic).

Planned: publish `tesera` 0.2.0 to PyPI. Publish `tesera` 0.2.0 to npm. Ship a hosted witness for tail-truncation detection. Ship a Stripe reconciliation adapter. Ship a standalone `verify` binary.

Tesera is MIT-licensed (see License).

## Where to go next

- `docs/THREAT_MODEL.md` — what the evidence does and does not establish.
- `docs/EVIDENCE_FORMAT.md` — the versioned journal spec both SDKs implement.
- `docs/DEPLOYMENT.md` — witness schedules, rotation, and reconciliation recipes.
- `ts/README.md` — the TypeScript sibling for Effect-based agents.
- `CHANGELOG.md` — every notable change, newest first.

## License

MIT — see [LICENSE](LICENSE).
