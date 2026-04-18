hljsSECURITY-POLICY-FILE
# Security Policy

`cognitive-meter` defines a receipt format intended to provide cryptographically verifiable usage records for AI API calls. Flaws in the schema, signature scheme, or canonicalization would compromise the trust model.

## Reporting a vulnerability

Please report privately if you identify:

- An issue with the signature scheme (Ed25519 usage, key rotation, canonicalization inputs).
- An attack on the hash-chain integrity (reordering, insertion, deletion detection).
- A way receipts could verify successfully under the spec while materially misrepresenting the underlying operation.
- A spec ambiguity that could lead to incompatible implementations believing they interoperate.

How to report:

- **Email** — `ljfreeman83@gmail.com` with `[SECURITY: cognitive-meter]` in the subject.
- **Response time** — acknowledgment within two business days, triage within five.

Do not open public GitHub issues for security topics.

## What's in scope

- Schema design and invariants described in `docs/RECEIPT_SCHEMA.md`.
- Reference implementation, when published in this repo.
- Verification examples in the README.

Out of scope — welcome as normal issues:

- General feedback on naming, ergonomics, or performance.
- Feature requests.

## Disclosure

Fixes to the spec get a version bump per the versioning policy in `docs/RECEIPT_SCHEMA.md`. Reporters are credited on release notes unless they request otherwise.
