# Plan: Add 4Mica Guarantee Payments (Keeping Aligned Escrow as Default)

## Goals
- Preserve the existing prepaid ETH escrow flow (BatcherPaymentService) as the default when no payment method is specified.
- Add a 4Mica guarantee-based path where a BLS certificate from 4Mica is accepted as payment (no on-chain deposit in Aligned for that path).
- Rely on the 4Mica sdk for tabs/guarantees storage; do not persist certificates in Aligned.

## References (checked)
- 4Mica SDK (`rust-sdk-4mica`): Recipient-side helpers verify BLS certs (`RecipientClient::verify_payment_guarantee`) and can remunerate if a user defaults (`RecipientClient::remunerate`). `verify_payment_guarantee` checks the BLS signature + optional domain and returns `PaymentGuaranteeClaims { domain, user_address, recipient_address, tab_id, req_id, amount, total_amount, asset_address, timestamp, version }`. Business rules (recipient/asset/amount/TTL) are not enforced in the helper and must be validated by the caller.
- 4Mica core: Guarantees = BLS certs over `PaymentGuaranteeClaims`; `req_id` is assigned sequentially per tab during issuance and `total_amount` accumulates all guarantees on that tab. Issuance validates the signature scheme, asset match, and that the claims timestamp sits inside the tab window (tab TTL defaults to 24h). There is no extra “settlement window” delay before remuneration; a recipient can call `remunerate` as soon as a certificate exists.
- 4Mica payment gateway (`x402.4mica.xyz`): HTTP API exposes `/supported`, `/verify`, `/settle`, `/tabs`, `/health`. `/settle` posts to core `/core/guarantees`, then re-verifies the returned certificate (signature + optional domain) before replying; it enforces scheme/network/payTo/asset and requires `amount == maxAmountRequired`. `/tabs` forwards to `/core/payment-tabs`, returns a hex `tabId`, and stamps `startTimestamp` as “now” with `ttlSeconds` coming from the request (no tab lookup endpoints are exposed).

## Payment Options UX
- CLI/SDK/API add `--payment-method` (or equivalent) with options:
  - `aligned-deposit` (default/backward compatible): unchanged escrow path.
  - `4mica-credit`: submitter must supply a 4Mica BLS certificate blob from the gateway.
  - `x402-exact` / `other`: reserved for future.
- If no method is provided, run the current escrow path.

## Aligned Flow (no contract changes)
- Escrow path: unchanged balance checks and `createNewTask`.
- 4Mica path:
  - On submission, require a BLS certificate payload (claims+signature) from the 4Mica gateway.
  - Validate off-chain with `rust-sdk-4mica`:
    - `RecipientClient::verify_payment_guarantee(cert)` for signature/domain/operator key.
    - Business checks: recipient = configured Aligned payee, asset/network match expectations, amount equals the required per-proof fee (gateway enforces equality), timestamp/TTL still valid. (TODO: add a helper that pulls tab TTL + settlement policy from core and validates claims/timestamps/amount/recipient in one place)
    - Consider `total_amount` to cap cumulative credit granted per tab. (TODO)
  - If valid, accept the proof into the batch; skip escrow balance deduction. Aggregator gas funding stays as today (batcher wallet).


## Aggregator Behavior
- Remuneration is callable immediately after a certificate is issued; define the policy (e.g., wait until tab TTL elapses) before slashing collateral. (TODO)
- If a user defaults after that policy threshold, ops (or a future job) should fetch the cert (from gateway/core or the submission payload) and call `RecipientClient::remunerate(cert)` to claim collateral. This is an exceptional/default-recovery path.

## 4Mica Gateway Expectations
- Provide to clients/operators:
  - `GET /supported`, `POST /verify` (preflight), `POST /settle` (returns `{certificate: {claims, signature}}`), `POST /tabs` (open/reuse tab), `GET /health`.
  - `POST /settle` enforces scheme/network/payTo/asset/amount equality and re-validates the certificate against the operator public key (and optional configured domain).
  - `POST /tabs` currently returns `{tabId, userAddress, recipientAddress, assetAddress, startTimestamp, ttlSeconds}` derived locally (start time = now, ttlSeconds = request or 0). No GET/tab lookup endpoints exist yet. (TODO: add tab retrieval/lookup if Aligned needs to refresh TTL or certificates)
- Enforce scheme/network/payTo/asset/amount. 


## 4Mica Adapter in Aligned
- New module in the batcher service that uses `rust-sdk-4mica`:
  - Verify certificates during submission (signature, domain, recipient, asset, amount, TTL policy, `total_amount` cap).
  - Optionally fetch certs on-demand from gateway when initiating remuneration (only on default). (TODO: decide storage vs. fetch policy)

## Docs & Config
- Add CLI/SDK doc snippet: `aligned submit ... --payment-method 4mica-credit --guarantee path/to/cert.json`.
- Env/config for adapter: 4Mica core/gateway URL, expected domain, operator key (fetched via SDK), Aligned payee address, scheme/network.
- Note: default behavior remains escrow; 4Mica is opt-in.

## Rollout Notes
- Start with an allowlist for `4mica-credit` if desired; otherwise keep open.
- Metrics/logging around: payment-method chosen, cert validation failures, and any default-triggered remuneration attempts.
