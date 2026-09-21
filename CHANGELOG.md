# Changelog

## 0.2.0

### Fixed

- `accounts.balances()` was attaching a body to a GET, which `fetch` rejects outright at runtime. The decryption proof now travels as a `decryption_proof` query parameter instead.
- The version reported in the `User-Agent` and the version in `package.json` had drifted apart. Both now read `0.2.0`.

### Added

- Transient failures are retried: dropped connections, `408`, `429`, and any `5xx`, with the wait doubling each round and `Retry-After` honoured when present. Governed by the new `maxRetries` option, which defaults to `2`.
- `transfers.create` now sends an `Idempotency-Key` on every attempt, generated when you do not supply one, which is what makes retrying a transfer safe. Your own key goes in the new options argument.
- Every method takes a trailing `RequestConfig` carrying an `AbortSignal` and a `timeoutMs` for that call alone. Aborting drops the in-flight request along with any retry still queued.
- `webhooks.constructEvent` proves a delivery and returns the typed, camelCased payload in a single step, raising the new `MansaFiWebhookVerificationError` when the signature or the body does not hold up.
- `WebhookEvent` gained `transfer.initiated` and `agent.policy.updated`, bringing it in line with the documented catalog.
- `webhooks.deliveries` is typed as `WebhookDeliveryList` rather than `unknown`.

### Changed

- Internal modules were reorganised: `http.ts` is now `transport.ts` (exporting `Transport`), `case.ts` is now `wire-format.ts` (exporting `toWire` and `fromWire`), and `webhook-signature.ts` is now `signature.ts`. None of these were part of the public surface, and every published export is unchanged.

## 0.1.0

- First release, covering accounts, confidential transfers, agent accounts and their spend policies, and webhook management with signature verification.
