# Receipt Schema — `otel.genai@0.1`

This document describes the receipt format used by `cognitive-meter`. The schema is language-neutral; identifiers contain no brand names.

## Top-level fields

| Field | Type | Required | Description |
|---|---|---|---|
| `receipt_version` | string | yes | Schema version. Currently `"0.1"`. |
| `receipt_id` | string | yes | Stable unique identifier. Prefix `rcpt_` followed by a URL-safe base64 random token. |
| `issued_at` | string (RFC 3339) | yes | UTC timestamp when the receipt was signed. |
| `resource` | object | yes | See [Resource](#resource). |
| `meter_schema` | string | yes | Meter-schema identifier. Current value: `"otel.genai@0.1"`. |
| `meter` | object | yes | See [Meter](#meter). |
| `pricing` | object | yes | See [Pricing](#pricing). |
| `signature` | object | yes | See [Signature](#signature). |
| `hash_chain` | object | yes | See [Hash chain](#hash-chain). |

## Resource

Describes the operation the receipt attests to.

| Field | Type | Required | Description |
|---|---|---|---|
| `method` | string | yes | HTTP method when applicable (e.g., `"POST"`). |
| `url` | string | yes | Resource path or logical identifier (e.g., `"/v1/chat"`). |

## Meter

The usage reading for the operation.

| Field | Type | Required | Description |
|---|---|---|---|
| `tokens.input` | integer | yes | Input tokens consumed. |
| `tokens.output` | integer | yes | Output tokens produced. |
| `latency_ms` | integer | no | End-to-end latency in milliseconds when observed. |
| `source` | string enum | yes | `"server_observed"`, `"provider_reported"`, or `"otel_span"`. Indicates how the measurement was captured. |
| `model` | string | yes | Model identifier as reported by the provider. |
| `tool_calls` | integer | no | Number of tool / function calls invoked in this operation. |

## Pricing

How the charge was computed.

| Field | Type | Required | Description |
|---|---|---|---|
| `pricing_model_id` | string | yes | Identifier of the pricing model used. Current default: `"flat+token@v0"`. |
| `currency` | string (ISO 4217) | yes | Three-letter currency code. |
| `actual` | string (decimal) | yes | Computed price as an exact decimal string with 8 fractional digits. |

Prices are computed as exact decimals and rounded once at the end. Floats are never used.

## Signature

Ed25519 signature over the canonicalized receipt content (everything except the `signature` field itself).

| Field | Type | Required | Description |
|---|---|---|---|
| `alg` | string | yes | Always `"ed25519"` in v0.1. |
| `key_id` | string | yes | Identifier of the signing key. Supports key rotation. |
| `sig` | string | yes | URL-safe base64 of the raw Ed25519 signature. |
| `verify_keys_url` | string | no | Endpoint from which public keys for this issuer can be fetched. Strongly recommended. |

Canonicalization uses RFC 8785 JCS (JSON Canonicalization Scheme) so signatures are reproducible across implementations.

## Hash chain

Makes insertion, deletion, and reordering of receipts detectable.

| Field | Type | Required | Description |
|---|---|---|---|
| `prev_receipt_hash` | string | yes | `sha256:` followed by the hex digest of the previous receipt for this resource. Sentinel for the first receipt: `sha256:0000...0000`. |
| `receipt_hash` | string | yes | `sha256:` followed by the hex digest of this receipt (computed over the canonicalized content, same as the signature input). |

## Reserved values

The following strings are reserved and should not be reused by alternate implementations of the schema:

- Meter schema IDs starting with `otel.genai@`
- Pricing-model IDs starting with `flat+token@`
- Signature algorithms: `ed25519`

## Versioning

- Schema versions are maintained indefinitely. New versions are added; old versions are never removed.
- A receipt issued under version `X` is always verifiable by a verifier that supports version `X`, regardless of how much newer the verifier is.
- Breaking changes get a new major version; additive changes get a new minor version.

## Reference implementation

- Python — this repository.
- Verification examples in JavaScript — see [`../README.md`](../README.md).

External implementations in other languages are encouraged. Interop is tested against the reference.
