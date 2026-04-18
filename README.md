# cognitive-meter

Signed receipts for AI API calls. Know exactly what you're spending and prove it.

This repository is the public spec, reference implementation, and verification tooling. The receipt format is language-neutral — receipts issued by the Python reference implementation validate against verifiers in any language.

Licensed Apache 2.0. Verification will always be free.

---

## What it does

You wrap an AI API call. You get a signed receipt that proves:

- how many tokens were used
- which model ran
- how the price was computed
- that nobody tampered with the numbers

If someone says "you owe $47," you don't have to trust them. You verify the receipt.

## Quick start (Python)

```python
from cognitive_meter import (
    AIMeter, PricingModel, build_receipt, sign_receipt,
    generate_signing_keys, verify_receipt,
)

meter = AIMeter()
pricing = PricingModel(per_1k_input=0.003, per_1k_output=0.015, per_tool_call=0.002)
private_key, public_key = generate_signing_keys()

# After any AI call, extract the meter reading
reading = meter.extract(response, provider="anthropic", latency_ms=842)

# Build and sign a receipt
receipt = build_receipt(reading, resource="POST /v1/chat", pricing_model=pricing)
receipt = sign_receipt(receipt, private_key)

# Verify it (anyone can do this with the public key)
assert verify_receipt(receipt, public_key)
```

## Three ways to measure

**Wrap mode** — the library times and measures the call:

```python
meter = AIMeter()
result, reading = meter.measure_anthropic_sync(client, model="claude-sonnet-4-20250514", messages=msgs)
```

**Extract mode** — you already made the call:

```python
reading = meter.extract(response, provider="anthropic", latency_ms=842)
# Works with: anthropic, openai (+ any compatible API), or "auto" to detect
```

**OTel mode** — read from OpenTelemetry spans:

```python
reading = meter.from_otel_span(span)
# Reads gen_ai.usage.* attributes automatically
```

Each receipt records which mode produced the measurement, so an auditor can evaluate trust per receipt.

## Verify in any language

The receipt format is language-neutral. Verification is three steps in any language: canonicalize → hash → check signature.

**JavaScript / Node.js:**

```javascript
import { webcrypto } from 'crypto';

async function verifyReceipt(receipt, publicKeyBytes) {
  // 1. Extract signable content (everything except "signature")
  const signable = Object.fromEntries(
    Object.entries(receipt).filter(([k]) => k !== 'signature')
  );

  // 2. JCS canonicalize (RFC 8785 — use a proper library in production)
  const canonical = JSON.stringify(signable, Object.keys(signable).sort(), 0);

  // 3. Verify Ed25519 signature
  const key = await webcrypto.subtle.importKey(
    'raw', publicKeyBytes, { name: 'Ed25519' }, false, ['verify']
  );
  const sig = Uint8Array.from(
    atob(receipt.signature.sig.replace(/-/g,'+').replace(/_/g,'/')),
    c => c.charCodeAt(0)
  );
  return webcrypto.subtle.verify(
    'Ed25519', key, sig, new TextEncoder().encode(canonical)
  );
}
```

**Key resolution.** The receipt's `signature.verify_keys_url` points to the issuer's public-key endpoint. Fetch it, match by `key_id`, and verify. The CLI does this automatically:

```bash
# Fetches the key from verify_keys_url in the receipt
cognitive-meter verify receipt.json

# Or use a local key
cognitive-meter verify receipt.json --key keys/cognitive-meter.pub
```

## CLI

```bash
cognitive-meter keygen --dir keys/          # Generate signing keys
cognitive-meter verify receipt.json         # Verify (fetches key from receipt)
cognitive-meter inspect receipt.json        # Pretty-print a receipt
```

## Pricing

Set rates in familiar "per 1K tokens" units:

```python
pricing = PricingModel(
    per_request=0.001,        # flat fee
    per_1k_input=0.003,       # per 1,000 input tokens
    per_1k_output=0.015,      # per 1,000 output tokens
    per_tool_call=0.002,      # per tool / function call
)
```

Prices are computed as exact decimals (never floats), rounded once at the end to 8 decimal places. The receipt contains meter data, pricing-model ID, and computed price, so anyone can independently recompute and verify the charge.

## Chained receipts

```python
from cognitive_meter import ReceiptBuilder

builder = ReceiptBuilder(pricing_model=pricing, signing_key_bytes=private_key)

# Each receipt chains to the previous one
r1 = builder.issue(reading1, resource="POST /v1/chat")
r2 = builder.issue(reading2, resource="POST /v1/chat")
# r2.hash_chain.prev_receipt_hash == r1.hash_chain.receipt_hash
```

## Supported providers

| Provider | Wrap mode | Extract mode |
|----------|-----------|--------------|
| Anthropic (Claude) | ✓ async + sync | ✓ |
| OpenAI (GPT) | ✓ async + sync | ✓ |
| Groq, Together, OpenRouter | via OpenAI mode | ✓ |
| Gemini, Llama, any API | — | ✓ (generic) |
| OTel spans | — | ✓ (from_otel_span) |

## What a receipt looks like

See [`examples/sample-receipt.json`](./examples/sample-receipt.json) for a full example. Canonical schema reference: [`docs/RECEIPT_SCHEMA.md`](./docs/RECEIPT_SCHEMA.md).

```json
{
  "receipt_version": "0.1",
  "receipt_id": "rcpt_a1b2c3d4e5f6...",
  "issued_at": "2026-03-02T22:00:00Z",
  "resource": { "method": "POST", "url": "/v1/chat" },
  "meter_schema": "otel.genai@0.1",
  "meter": {
    "tokens": { "input": 500, "output": 1200 },
    "latency_ms": 842,
    "source": "server_observed",
    "model": "claude-sonnet-4-20250514",
    "tool_calls": 2
  },
  "pricing": {
    "pricing_model_id": "flat+token@v0",
    "currency": "USD",
    "actual": "0.02050000"
  },
  "signature": {
    "alg": "ed25519",
    "key_id": "default",
    "sig": "base64url-encoded-signature..."
  },
  "hash_chain": {
    "prev_receipt_hash": "sha256:0000...0000",
    "receipt_hash": "sha256:a1b2c3..."
  }
}
```

## Compatibility pledge

- **Receipts are stable.** A receipt issued today will validate against every future version of the verifier.
- **Verification is always free.** `cognitive-meter verify` will never be paywalled.
- **Schema versions are maintained indefinitely.** New versions are added; old versions are never removed.

## Format neutrality

Schema identifiers (`otel.genai@0.1`, `flat+token@v0`, `ed25519`) contain no brand or product names. A verifier written in Go, Rust, or JavaScript produces identical results — only the tooling differs. If the tooling name ever changes, your receipts do not.

## Related work

- [wesumn-public](https://github.com/Stray-South/wesumn-public) — the architecture and design notes for the platform that uses this receipt format at scale for AI agent governance.

## License

Apache 2.0 — see [LICENSE](./LICENSE).
