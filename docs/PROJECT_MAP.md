# Tessera — Project Map (Phase 0)

> Mapping only. No refactors, no fixes, no features. Generated 2026-09-25 from a clean checkout (`0ade674`).
> Rename note: working directory is `tessera`; code, package, and docs still say `interceptor` / `interceptor-effect`. Nothing has been renamed yet.

## 1. Full file tree (condensed)

```
pyproject.toml  uv.lock  CHANGELOG.md  LICENSE (MIT, (c) 2026 Wira Mahendra)
README.md  SECURITY.md  .python-version  .github/workflows/ci.yml
src/interceptor/ (29 modules):
  __init__.py  approval.py  approve_server.py  archive.py  audit.py
  canonical.py  checkpoint.py  cli.py  contracts.py  cosign.py
  engine.py  errors.py  export.py  gateway.py  guard.py
  identity.py  journal.py  observer.py  policy.py  privacy.py
  providers.py  reconcile.py  redaction.py  resolve.py  schemas.py
  shipping.py  verification.py  witness.py  wrap_tool.py  py.typed
tests/ (30 files): conftest.py helpers.py
  test_audit.py test_audit_streaming.py test_checkpoint.py test_cli.py
  test_deploy_units.py test_evidence.py test_guard.py test_hardening.py
  test_idem_index.py test_interceptor_features.py test_key_rotation.py
  test_maturity.py test_next_batch.py test_node_verifier.py test_privacy.py
  test_redaction_homoglyphs.py test_redaction_property.py test_redaction_traversal.py
  test_review_batch.py test_round3.py test_schemas.py test_ts_interop.py
  test_vectors.py test_witness_freshness.py test_witness_prune.py
  test_witness_reconcile.py test_wrap_tool.py
ts/ (package `interceptor-effect` 0.2.0, Effect, Node 20):
  src/: Canonical.ts Checkpoint.ts Custody.ts Guard.ts Idempotency.ts
       Identity.ts Journal.ts Lock.ts Policy.ts Redaction.ts Schemas.ts
       Verify.ts Witness.ts index.ts
  test/: custody idempotency lock policy prune redaction roundtrip vectors witness (9 files)
  scripts/emit-sample.js  package.json  pnpm-lock.yaml  tsconfig.json  README.md
verifiers/node/verify.mjs (dependency-free Node verifier, stdlib only)
verifiers/vectors/v1/*.json (9 committed vectors) + generate.py
docs/: API_STABILITY.md DEPLOYMENT.md EVIDENCE_FORMAT.md PERFORMANCE.md REVIEW_GUIDE.md THREAT_MODEL.md
deploy/systemd/: witness/verify/prune .service + .timer units + README.md
benchmarks/: bench.py bench_idempotent.py  .benchmarks/ (local output)
dist/ (build artifact, not source)
```

No `services/`, `apps/dashboard`, `simulator/`, `tools/verify-evidence`, or wire-protocol docs exist. No Tessera Cloud code exists.

## 2. Current capabilities (what works today)

* **Approval-gated guard (Python + TS):** `@guard` / `Guard.guard` denies by default, prompts on TTY, raises off-TTY, supports `approval="never"`, policy providers, quorum, TTL cache, timeout, budget/rate-limit/spending/attested/rule-file. Denials never execute.
* **Signed hash-chained evidence (local file):** JSONL journal, Ed25519, `key_id = ed25519:<16 hex of SHA-256(pubkey)>`, per-event `event_hash = SHA-256(canonical(unsigned_payload))`, signature over raw digest bytes. `verify` + `audit` CLI offline. Key rotation via in-chain `rotation` events + `trusted_keys/` dir.
* **Redaction fused into canonicalization:** single traversal; name match (case/accent/homoglyph-folded) + value-pattern match (`sk-live/test`, `ghp_/gho_`, `xoxb-`, `AKIA`, PEM, JWT + custom regex). Secrets never in input hash, summary, prompt, or output hash.
* **Idempotency + dry-run:** `idempotency_key` (name/literal/callable) blocks second completed execution with `DuplicateActionError` (subclass of `ActionDenied`); retries after failure allowed; in-process exact index + file-journal cross-process resolution; `dry_run=True` writes decision only, audited as `dry_run`.
* **Receipts + offline reconciliation:** `receipt_from` extracts provider IDs (canonicalized/redacted/bounded, dropped on error); `reconcile_journal` compares receipts against an injected fetcher (`matched/mismatched/provider_unknown/fetch_error`), never mutates journal, refuses to run on unverified journals.
* **Checkpoints / witnesses / countersigning / archive:** `checkpoint` + `witness` (ship off-host) + `witness-audit` + `witness-prune`, `countersign` second-key, `archive --keep N` with custody links, `verify-chain`, `WitnessFreshnessProvider` gate, TS HTTP loopback witness with signed checkpoints and coverage checks.
* **Operator tooling:** `resolve` (signed attestation `confirmed_completed/not_completed`), `export` (JSON/HTML packs), `stats`, `inspect` (shareability classifier), `policy-test`, `ToolGateway`, `wrap_tools`, `describe_tool`/`as_openai_tool`/`mcp_tool`, encrypted PEMs, `CallbackSigningIdentity` (TPM/HSM/KMS seam), systemd units, S3 ObjectLock recipe in DEPLOYMENT.md.

## 3. Evidence format spec (existing v1 — reference, not frozen copy)

Full contract: `docs/EVIDENCE_FORMAT.md` (v1, `schema_version: "1"`). Summary:

* **Encoding:** canonical JSON (UTF-8, sorted keys, `,`/`:` separators, no whitespace, reject NaN/Inf, unescaped UTF-8). Value canonicalization table covers None/bool/int/str/float/list/tuple/namedtuple/dataclass/str-keyed dict; non-string-keyed maps and exotic types become `<unsupported:…>` placeholders; cycles / depth>64 are errors.
* **Envelope (every event):** `schema_version event_type event_id action_id action_name contract_hash timestamp_utc key_id previous_event_hash event_hash signature`. `action_*`/`contract_hash` absent on `checkpoint`/`countersignature`; `rotation`/`archive` also carry no action fields.
* **Event types (7):** `decision` (allowed/denied + risk/mode/input_hash/summary/retention/idempotency/dry_run/duplicate_of/reason/approved_by/spend), `outcome` (succeeded/failed + decision link/type/output-hash/error/receipt), `checkpoint` (count+head), `resolution` (decision link + completed/not_completed + note ≤1000ch), `countersignature` (checkpoint ref + count/head match), `rotation` (prior/successor key ids + fingerprint), `archive` (first line, prior_count/prior_head/archived_path).
* **Hash/sign:** `event_hash = SHA-256(canonical(payload minus event_hash/signature))`; `signature = b64(Ed25519_sign(raw_digest))`. Contract hash over `{schema_version action_name module qualified_name risk approval_mode execution_mode parameter_descriptors code_fingerprint}`.
* **Verify:** chain link + hash + per-event signature + key_id match; checkpoint adds `count` coverage rule; verifiers must refuse unknown `schema_version`. Any added field changes `event_hash` by design.
* **Audit (post-verify):** unique event_ids, outcome→earlier-allowed-decision, identity match, no double outcomes, enum checks, duplicate `(action,idempotency_key)` detection; statuses `denied/succeeded/failed/dry_run/resolved_completed/resolved_not_completed/needs_reconciliation`; `needs_reconciliation` + `failed` exit non-zero. Prefix truncation is **undetectable without an external witness** (see THREAT_MODEL.md).

### Python vs TypeScript compatibility

* Committed vectors (`verifiers/vectors/v1/`, 9 journals) are checked by **both** Python (`test_vectors.py`), Node stdlib verifier (`test_node_verifier.py`), and TS Effect verifier (`vectors.test.ts`) — validity, counts, failure codes must match. Reverse direction (TS-written → Python verify/audit/inspect) is proven by `test_ts_interop.py`.
* **No known schema divergence.** Deliberate TS gaps (per `ts/README.md`): `parameterNames` must be supplied for positional secret redaction by name; exotic Unicode beyond the ported lookalike table is best-effort on both sides; no hosted witness (loopback/LAN only — pair with TLS/auth/storage yourself). Python is the reference implementation; freshness gates and retention are already ported.
* **Missing for Tessera v1:** a byte-identical cross-language conformance test for *equivalent operations* (same action/args → identical canonical bytes) and a frozen `EVIDENCE_FORMAT_v1.md` compatibility contract. Current vectors prove mutual *verification*, not byte-identical *production*.

## 4. Test status (reproduced 2026-09-25)

| Suite | Result |
|---|---|
| Python `pytest` (`.venv/bin/python -m pytest -q`) | **428 passed**, coverage **86.66%** (gate 85% in `pyproject.toml`) |
| TS `vitest` (`ts/`, Node 20) | **56 passed / 9 files** (incl. 10-fiber exactly-once race, 4-process appends, vector conformance) |
| Vectors subset (`test_ts_interop + test_vectors + test_node_verifier`) | **24 passed** |
| `ruff check .` | clean |
| `mypy` (local `.venv`) | **broken harness, not code**: `.venv/bin/mypy` shebang points at deleted `/Users/wira/Documents/guardrail-evidence/.venv/bin/python`. CI runs `uv run --frozen mypy` (strict) — re-run there. |
| `uv run pytest` (fresh sync path) | fails at conftest import (`No module named 'interceptor'`) — use `.venv/bin/python -m pytest` or `uv run --frozen pytest` after `uv sync --frozen` as CI does. |

No flaky tests observed in a single run. Property-based tests exist for redaction (`test_redaction_property.py`, Hypothesis) and evidence chain tampering/replay/truncation via vectors + `test_evidence.py`/`test_hardening.py`. No dedicated refund-failure simulator; no adversarial tenant/replay/forge suite for a service (nothing to attack yet).

## 5. Integration status

| Integration | Status |
|---|---|
| OpenAI function calling (`as_openai_tool`, `describe_tool`) | **Works.** Schema derives from the hashed contract; fidelity matrix in `test_schemas.py`. |
| MCP (`mcp_tool`) | **Works.** Registers guarded callable, guard stays on. |
| LangChain / generic agents (`wrap_tools`, `ToolGateway`) | **Works generically.** No LangChain-native adapter; config errors surface as `ToolWrapError`. |
| Stripe refunds (`StripeRefundFetcher` + `reconcile_journal`) | **Shipped, duck-typed, offline-tested with fakes.** Caller injects `stripe` module; `stripe` is never a library dependency (no-socket suite guarantee holds). Compares `processor/refund_id/status` + optional `extra_fields`. No live-Stripe test, no scheduled worker, no UNKNOWN-state polling. |
| Anthropic / other providers | **None.** No adapter, no stub. |
| Approval surfaces (terminal, policy file, HTTP LAN server, quorum, budgets) | **Works** (see §2). `ApprovalServer` has constant-time tokens + single-use decision tokens. |
| External witness shipping (`FanoutJournalStore`, `witness_journal`, TS HTTP witness) | **Works locally.** No independent hosted custodian, no multi-tenant ingestion, no replay-safe server ingestion. |

## 6. TODO / FIXME / HACK scan

`grep -rn TODO|FIXME|HACK|XXX|BUG src/ ts/src/` → **zero matches**. Debt is tracked in CHANGELOG review batches and docs, not inline markers.

## 7. Package metadata / branding defects (prioritized)

1. **Repo rename not applied (P0).** `pyproject.toml` urls still `github.com/wiramahendra/guardrail-evidence`; README badges/clone still `github.com/rapture-fx/interceptor`; package `interceptor` collides with an unrelated PyPI name (README tells users to install from source). Nothing says `tessera` yet. Decide: new PyPI name + repo URL + `docs/` links in one commit.
2. **Stale local toolchain path (P1).** `.venv/bin/mypy` shebang references deleted `guardrail-evidence/.venv`; `uv run pytest` without frozen sync mis-resolves imports. CI (`uv sync --frozen` → `uv run --frozen …`) is correct; local `.venv` needs recreation after the rename.
3. **Version source vs checkout (P1).** Source checkouts report `0.0.0+unknown`; release is `0.2.0`. Documented in CHANGELOG but still surprises `pip install -e .` users and any version-gated wire protocol later.
4. **TS positional-secret caveat (P1).** `parameterNames` must be supplied or positional secrets miss name redaction. Documented in `ts/README.md`; one missed call-site = disclosure. Needs a conformance test that fails closed.
5. **Trusted-key set is local state, not authentication (P2).** Rotation bounds future forgery but the `trusted_keys/` dir authenticates nothing by itself; out-of-band `--public-key` pinning is required. Correct per THREAT_MODEL, but every future ingestion endpoint must re-state it or a tenant will misconfigure.
6. **`archive --keep` deletion destroys evidence (P2).** Custody chain with a missing predecessor proves nothing about the gap. Documented; needs a retention entitlement before any paid tier.
7. **Tail truncation without a witness is undetectable (P2, by design).** The exact gap Tessera Cloud must close. Today only file copies + countersignatures + loopback witness mitigate it.

## 8. Top 5 defects → Top 5 quick wins (for PR comment)

**Defects:** (1) obsolete repo URLs + PyPI name collision; (2) stale `.venv` mypy/python shim paths; (3) TS positional-redaction requires manual `parameterNames`; (4) trusted-key set easily mistaken for authentication; (5) tail-truncation defense is manual (no hosted custodian).

**Quick wins:** (1) fix `pyproject.toml` Project-URLs + README badges/clone to the Tessera repo in one branding commit; (2) recreate `.venv` via `uv sync --frozen` and pin the documented `make baseline` path; (3) add a TS↔Python byte-identical vector for one `billing.refund` call (closes the v1 conformance hole); (4) freeze `docs/EVIDENCE_FORMAT_v1.md` from the current spec with versioning rules; (5) publish the refund-failure simulator matrix (success/provider-failure/timeout/crash/retry/delayed-visibility/lost-ack) with a ground-truth log before any service code.

## 9. Gaps blocking Phase 1–4 (do not build yet — record only)

* No `EVIDENCE_FORMAT_v1.md` frozen contract; no `make baseline`; no refund-failure simulator; no cross-language byte-identical conformance dir; no `DESIGN_PARTNERS.md`; no wire protocol; no ingestion/tenant/checkpoint/object-storage/upload-queue/health/export service; no state machine doc (`NOT_DISPATCHED…RESOLVED`); no resolution queue/dashboard/RBAC/alerts; no billing/metering/runbook/verify-CLI/reference-app.
