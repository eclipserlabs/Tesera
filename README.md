# Tesera

Signed, hash-chained evidence for consequential Python function calls.
Approval-gated. Locally verifiable. No network required.

## Install

```sh
pip install tesera
```

Requires Python 3.10+.

## 30-second example

```python
from tesera import guard


@guard(action="billing.refund", risk="high", approval="never")
def refund(order_id: str, amount_cents: int) -> dict:
    return {"id": "re_123", "amount_cents": amount_cents}


print(refund("order-1", 1999))
```

Output:

```
{'id': 're_123', 'amount_cents': 1999}
```

Every call writes two signed records to `~/.tesera/journal.jsonl`: a
`decision` (approved before execution) and an `outcome` (what the function
returned). `approval="never"` skips the human prompt and records that choice
explicitly; point the decorator at an approval provider when a human must
say yes (see `src/tesera/policy.py` providers and `docs/DEPLOYMENT.md`).

## Verify evidence offline

```sh
tesera verify
```

Output:

```
OK  /Users/you/.tesera/journal.jsonl
    2 events, signatures and hash chain intact
    note: tail truncation is detectable only with a checkpoint witness
```

Verification needs only the journal and the public key. No account, no
server, no network. `tesera audit` goes further: it pairs decisions with
outcomes and flags anything ambiguous (`needs_reconciliation`) instead of
guessing.

## What it is not

- Not a workflow engine. Use Temporal, Airflow, or Cadence for orchestration.
- Not a replacement for Stripe idempotency or provider-level guarantees.
- Not a guarantee of exactly-once execution. A crash between the side
  effect and the outcome record still needs a human to check the provider.
- Not an LLM in the loop. Verification is deterministic, always.
- Not proof the external world changed. A `succeeded` outcome means the
  function returned — not that the payment processor acted. Receipts
  (provider reference IDs transcribed into the outcome) make later
  reconciliation possible; they do not verify the provider did anything.

The full boundary is in `docs/THREAT_MODEL.md`. Read it before relying on
the evidence for anything that matters.

## Evidence format

The journal is newline-delimited JSON, specified in
`docs/EVIDENCE_FORMAT_v1.md`. Any language can verify this format: the
format is the contract, not the library. A dependency-free Node verifier
lives in `verifiers/node/verify.mjs`, and committed cross-language vectors
in `verifiers/vectors/v1/` are checked by both the Python and TypeScript
implementations.

## Status

Works today: approval gates (terminal, policy, quorum, LAN server),
signed hash-chained journals, offline verify/audit/inspect/export,
idempotency keys, dry runs, receipts, checkpoints with witness files,
second-key countersignatures, key rotation, archive rotation, and a
TypeScript sibling (`ts/`, package `tesera`) implementing the same format.

Does not exist yet: a public hosted witness service. The
`tessera-cloud` backend (ingestion, reconciliation, operator queue) is
private and not accepting external traffic.

Planned (5 max): first PyPI release; first npm release; a hosted witness
for tail-truncation detection; a Stripe reconciliation adapter; a public
`tesera verify` binary.

## License

MIT. See `LICENSE`.
