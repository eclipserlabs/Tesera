# Security

`tesera` writes signed, hash-chained evidence for consequential calls.
Its trust model matters more than its code does, so read
[`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md) before reporting or fixing.

The cryptographic primitives come from well-audited libraries (Python
`cryptography`, Node builtins). This library's job is to use them
correctly: one traversal for redact-and-canonicalize, signatures over
canonical bytes, verification that recomputes everything.

## Reporting a vulnerability

Do **not** open a public GitHub issue for a vulnerability that could let an
attacker forge, edit, or disclose evidence.

Report it privately instead:

- If you have write access to the repository, use GitHub's private security
  advisory flow.
- Otherwise, email the maintainers via the contact address listed on the
  project page, and include "tesera" in the subject line.

Please include:

- the affected version(s);
- the threat-model assumption you believe is violated (or why it is not);
- a minimal reproduction, ideally with a patched journal you can verify
  against;
- whether the issue is a forgery, an integrity break, a disclosure, or a
  denial of service.

We acknowledge within 48 hours. Critical issues (forgery or disclosure
without the key) get a fix release within 30 days; anything slower is
announced. After the fix, we publish a security advisory and credit the
reporter unless they ask otherwise. There is no bug bounty.

## If a vulnerability is reported

One maintainer triages: reproduce against the latest release, assess
against `docs/THREAT_MODEL.md` (which assumption breaks?), and set
severity. A fix ships as a patch release with a `CHANGELOG.md` entry
and a GitHub security advisory; users are notified through the advisory
and the release notes. Pin exact versions downstream and upgrade: a
compromise of the signing path is total and retroactive, so there is no
"safely ignore" for a real finding. Reports that restate documented
limits (tail truncation without a witness, stolen keys, low-entropy
hash recovery) are closed with a pointer to the threat model.

## What is and is not in scope

In scope:

- a path by which evidence can be produced, reordered, or edited without the
  signing key, or verified as authentic when it is not;
- a route for named secrets past the fused redaction/canonicalization
  traversal into the journal, a summary, a prompt, or an error message;
- a fail-open path (an approval or observer failure that silently proceeds);
- an authentication or permission weakness in key handling.

Explicitly out of scope, because the threat model states them as limits:

- detecting deletion of the journal **tail** — the documented gap that needs an
  external witness;
- a compromised process or a stolen signing key (both are trust assumptions);
- recoverability of *non-redacted* low-entropy values from `input_hash`.

## Key handling

- The private key is created mode `0600` and the guard refuses to sign with a
  key that is readable by other users (POSIX).
- Rotation (`key-rotate`) keeps the outgoing public key in the `trusted_keys/`
  directory inside the evidence home, so evidence signed before the rotation
  stays verifiable. The trusted set is local operator state, not a signature:
  it protects integrity across key changes, it does not authenticate a key.
  Pin the public key out-of-band (e.g. `--public-key`) for real authentication.
- Never paste a private key or a journal into an issue.
- A journal that failed `tesera verify` is evidence of tampering or
  corruption; do not discard it, keep it for analysis.

## Supported versions

Only the latest release is patched. Pin exact versions in anything that signs
or verifies evidence; a compromise of the signing path is total and
retroactive.
