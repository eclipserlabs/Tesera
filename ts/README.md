# Tesera (TypeScript)

Signed, hash-chained evidence for consequential function calls — the
Effect/TypeScript sibling of the Python `tesera` package. Same evidence
format v1, same guarantees, enforced where TypeScript agents run.

## Install

```sh
npm install tesera
```

Requires Node 20+.

## 30-second example

```ts
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
  })((orderId: string, amountCents: number) => ({ id: "re_123", orderId, amountCents }));

  return yield* refund("order-1", 1999);
});

Effect.runPromise(program).then(
  (result) => console.log(result),
  (error) => { console.error(String(error)); process.exit(1); },
);
```

Output:

```
{ id: 're_123', orderId: 'order-1', amountCents: 1999 }
```

Each call writes a signed `decision` before execution and a signed
`outcome` after, chained into `./journal.jsonl`. The generated
`tesera.pub.pem` is the verifying key — keep it somewhere the journal
cannot reach.

## Verify evidence offline

```sh
node ./node_modules/tesera/verifiers/node/verify.mjs --journal ./journal.jsonl --public-key ./tesera.pub.pem
```

Output:

```
OK  ./journal.jsonl
    2 events verified (independent node verifier)
```

The verifier is dependency-free (Node builtins only). No account, no
server, no network.

## What it is not

- Not a workflow engine. Use Temporal, Airflow, or Cadence for orchestration.
- Not a replacement for Stripe idempotency or provider-level guarantees.
- Not a guarantee of exactly-once execution. A crash between the side
  effect and the outcome record still needs a human to check the provider.
- Not an LLM in the loop. Verification is deterministic, always.
- Not proof the external world changed. A `succeeded` outcome means the
  function returned — not that the payment processor acted.

The full boundary is in `docs/THREAT_MODEL.md`. Read it before relying on
the evidence for anything that matters.

## Evidence format

The journal is newline-delimited JSON, specified in
`docs/EVIDENCE_FORMAT_v1.md`. Any language can verify this format: the
format is the contract, not the library. Committed cross-language vectors
in `verifiers/vectors/v1/` are checked by both implementations, and the
Python package verifies TypeScript-written journals and vice versa.

## Status

Works today: approval-gated guard with signed decision/outcome evidence,
file journals, offline verification, budgets/rate-limit/quorum/declarative
policy providers, checkpoints with witness files, idempotency, dry runs,
receipts, and an HTTP witness client. Python remains the reference
implementation; capabilities land there first and are ported where
TypeScript agents need them.

Does not exist yet: a public hosted witness service (private, not
accepting external traffic).

Planned (5 max): first npm release; a hosted witness for tail-truncation
detection; a Stripe reconciliation adapter; a public verify binary; policy
parity for new Python providers.

## License

MIT. See `LICENSE`.
