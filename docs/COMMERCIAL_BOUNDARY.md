# Tessera — Commercial Boundary (Phase 0)

> Normative for scoping. If in doubt, the constraint wins over the roadmap: hard constraints in the task brief override any reading of this file.

## 1. Rule

**Free library:** everything that produces, verifies, or audits evidence *locally and offline*, with no account, no network, and no Tessera-operated infrastructure.

**Paid service (Tessera Cloud):** everything that requires *independent custody* — a second machine, a second key, a second operator — plus the workflows that make ambiguous outcomes resolvable and billable.

A customer can always verify exported evidence without Tessera Cloud. That portability is the product's trust anchor, not a feature to be metered.

## 2. Free library (MIT — Python `interceptor` + TS `interceptor-effect` + Node verifier)

* Approval gates: terminal/prompt, policy providers (budget, rate-limit, spending, quorum, attested, TTL, timeout, `AllOf`/`AnyOf`, glob/JSON rules, `policy-test`), LAN approval server, `ToolGateway`, `wrap_tools`.
* Evidence production: `@guard` / Effect guard, signed hash-chained JSONL journal (all 7 event types), redaction + canonicalization, idempotency keys, dry runs, receipts, contracts + `describe_tool`/`as_openai_tool`/`mcp_tool`.
* Local verification and audit: `verify`, `verify-chain`, `audit` (+ streaming + triage filters), `inspect`, `stats`, `export` (JSON/HTML packs), `resolve` (signed operator attestation), checkpoint/countersign/witness-file/archive/rotation/keygen/key-rotate CLI, Node stdlib verifier, committed cross-language vectors.
* Offline reconciliation helper: `reconcile_journal` + `StripeRefundFetcher` (caller-injected client; library performs no I/O, takes no network dependency).
* Self-run operations: systemd units, witness cron/prune recipes, DEPLOYMENT/PERFORMANCE/THREAT_MODEL docs, benchmarks.
* Evidence format v1 spec and versioning rules. Format changes bump the version with a migration path; v1 journals verify forever with the pinned verifier.

No seat, event, storage, or reconciliation-attempt metering in the library. No phone-home, no license check, no hosted key.

## 3. Paid service (Tessera Cloud — does not exist yet; scope for Phases 2–4)

* **Authenticated journal ingestion** with per-record independent signature verification; invalid signatures rejected, never stored as valid. Idempotency keys for ingestion + explicit acknowledgments; a record is never silently dropped.
* **Tenant isolation:** every record, key, and query scoped to a tenant; cross-tenant access proven absent by integration test.
* **Externally stored checkpoints:** periodic hash + object-storage segments with retention controls, independently verifiable without Tessera Cloud.
* **Reconciliation worker (Stripe first):** scheduled polling of UNKNOWN operations using provider-documented guarantees only; automatic retry only under a provider idempotency contract or independently established non-execution evidence.
* **Unresolved-operation queue + operator dashboard:** provider evidence, attempt history, human resolution cryptographically bound to the operation and immutable thereafter; roles (operator/reviewer/admin), per-action attribution, alerts (entered-UNKNOWN, resolution recorded, repeated reconciliation failure).
* **Export + public verify CLI:** full journal export in the versioned format, verifiable without the service.
* **Metering/billing/entitlements:** operations ingested/month, evidence storage, reconciliation attempts, dashboard seats; Stripe Billing; backup/restore/incident/key-management runbook.

Explicitly **not** in the paid service: a workflow engine, a payment processor/CPQ, a GRC platform, an LLM judge in the correctness path (never), a generic adapter framework (one Stripe adapter, done well; next adapter only on paying-customer demand), customer-facing analytics before verifiability + reconciliation work.

## 4. Boundary examples (no ambiguity)

| Question | Answer |
|---|---|
| Sign + verify a refund journal on my laptop, no account? | Free, offline, forever. |
| Second-key countersignature on my own witness box? | Free (CLI + docs). |
| Tessera holds a checkpoint key off-host so tail truncation is detectable? | Paid (independent custody). |
| `audit` says `needs_reconciliation`; I check Stripe myself and `resolve`? | Free (local attestation). |
| A worker polls Stripe on a schedule and pages my operator queue? | Paid (reconciliation + alerts). |
| Export my journal and verify it after cancelling? | Free, guaranteed. Verification never requires the service. |
| RBAC, seats, retention entitlements, metering? | Paid. |
| New adapter for provider X? | Only after a paying customer demands it; otherwise not built. |

## 5. Stop condition (restated)

If three design-partner conversations all conclude orchestrator + idempotency + database records already solve the problem, **stop commercial development**: publish the library, document findings, keep the evidence-format spec as a public good. No subsidized SaaS.
