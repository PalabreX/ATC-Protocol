# ATC Protocol — Agent Trust Certificate

An open, implementation-agnostic spec for portable, cryptographically
signed proof of an AI agent's standing — its trust score, verified
identity, and transaction history — that any counterparty can verify
independently, without calling back to the issuer.

Live reference implementation: https://getrunpay.com/agent-trust-protocol.md
Built and maintained by [run.pay](https://getrunpay.com).

## Why

Payment protocols (x402, AP2, MPP) solve *how* an agent pays.
They don't solve *who do you trust enough to pay*. ATC is the layer
underneath: a short-lived, signed certificate any agent can export
and any counterparty can verify — portable across marketplaces,
not locked to one platform's API.

## What this is

A portable, cryptographically signed proof of an AI agent's verified trust score — designed to be checked by ANYONE, not just run.pay. Any platform can verify a certificate with nothing but the issuer's public key. Any platform meeting the same guarantees below could issue its own compatible certificates.

## The guarantee that matters

A certificate is only meaningful if the underlying score reflects real, independently-verifiable activity — not self-reported claims. run.pay's own implementation computes scores from real transaction history and cryptographic audit-chain verification. Any issuer adopting this protocol should hold to the same principle: **a score a subject can inflate by simply claiming success is not a trust score, it's a self-review.**

## Certificate format (JSON)

```json
{
  "protocol": "agent-trust-certificate",
  "protocol_version": "1.0",
  "issuer": "run.pay",
  "agent_id": "agt_example",
  "score": 850,
  "label": "EXCELLENT",
  "total_actions": 142,
  "successful_actions": 138,
  "audit_integrity": 1.0,
  "member_since": "2026-01-15T00:00:00.000Z",
  "issued_at": "2026-08-07T00:00:00.000Z",
  "expires_at": "2026-08-08T00:00:00.000Z",
  "verify_key_url": "https://.../.well-known/runpay-trust-key.json",
  "spec_url": "https://getrunpay.com/agent-trust-protocol.md"
}
```

## Required fields for any conforming issuer

- `protocol`: must be exactly `"agent-trust-certificate"`
- `protocol_version`: semver-style string, this document describes `"1.0"`
- `issuer`: a human-readable name AND a `verify_key_url` resolving to that issuer's public key
- `score`: numeric, 0-1000 scale, computed from real verifiable activity (see guarantee above)
- `issued_at` / `expires_at`: ISO 8601. **Certificates SHOULD expire within 24-48h** — a portable proof with no expiry can circulate long after it stops reflecting reality.

## Verification procedure (works without contacting the issuer)

1. Fetch `verify_key_url` — a JSON document with a `public_key_pem` field (RSA, SHA-256, PKCS1 padding).
2. Serialize the certificate object to JSON exactly as received (do not re-order or reformat keys).
3. Verify the signature against that exact byte sequence using the public key.
4. Check `expires_at` has not passed.

Node.js example:
```js
const valid = crypto.verify('sha256', Buffer.from(JSON.stringify(certificate)),
  { key: publicKeyPem, padding: crypto.constants.RSA_PKCS1_PADDING },
  Buffer.from(signature, 'base64'));
```

## Versioning policy

Backwards-incompatible changes bump the major version (`2.0`). Additive, optional fields may appear in minor updates without a version bump. A verifier SHOULD reject a certificate whose `protocol_version` major number it doesn't recognize, rather than guessing at a possibly-changed format.

## Reference implementation (run.pay)

- Issue: `POST /api/agents/trust-certificate` (requires the agent's wallet_secret)
- Verify (convenience): `POST /api/verify-trust-certificate`
- Public key: `GET https://getrunpay.com/.well-known/runpay-trust-key.json`
- Discovery: `GET https://getrunpay.com/.well-known/agent-trust-protocol.json`
- Human-readable trust leaderboard: [getrunpay.com/trust.html](https://getrunpay.com/trust.html)

This is v1.0 of a protocol run.pay is publishing openly — other platforms
are welcome to verify run.pay-issued certificates, or implement their own
compatible issuer against this spec.

## Status

v1.0 — implemented and live in production, open for feedback and
adoption by other agent marketplaces.

## License

MIT — see [LICENSE](./LICENSE).
