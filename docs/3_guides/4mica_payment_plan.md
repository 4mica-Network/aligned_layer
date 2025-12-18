# Plan: Add 4Mica Guarantee Payments (Keeping Aligned Escrow as Default)

## Goals
- Preserve the existing prepaid ETH escrow flow (BatcherPaymentService) as the default when no payment method is specified.
- Add a 4Mica guarantee-based path where a BLS certificate from 4Mica is accepted as payment (no on-chain deposit in Aligned for that path).
- Rely on the 4Mica sdk for tabs/guarantees storage; do not persist certificates in Aligned.

## References (checked)
- 4Mica SDK (`rust-sdk-4mica`): Recipient-side helpers verify BLS certs (`RecipientClient::verify_payment_guarantee`) and can remunerate if a user defaults (`RecipientClient::remunerate`). Guarantees carry `claims` + `signature`, include domain, recipient, asset, amount, tab_id, req_id, timestamp.
- 4Mica core : Guarantees = BLS certs over `PaymentGuaranteeClaims`; tabs have TTL then a 7-day settlement window before remuneration is allowed.
- 4Mica payment gateway (`x402.4mica.xyz`): HTTP API exposes `/supported`, `/verify`, `/settle`, `/tabs`; `/settle` issues a BLS certificate (claims+signature) after validating the payment header against requirements; `/tabs` opens/reuses tabs via core; gateway enforces scheme/network/payTo/asset/amount.

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
    - Business checks: recipient = configured Aligned payee, amount ≥ required per-proof fee (and expected asset/network), timestamp/TTL still valid. (TODO: add these to sdk for checking)
  - If valid, accept the proof into the batch; skip escrow balance deduction. Aggregator gas funding stays as today (batcher wallet).


## Aggregator Behavior
- If a user defaults after the tab TTL + 7-day settlement window should ops (or a future job) fetch the cert from the 4Mica gateway/core and call `RecipientClient::remunerate(cert)` to claim collateral. This is an exceptional/default-recovery path.

## 4Mica Gateway Expectations
- Provide to clients/operators:
  - `POST /tabs` to open/reuse tabs (user+recipient+asset+TTL).
  - `GET /tabs/{id}` and `GET /tabs?user=...&settled=false` for unsettled tabs.
  - `POST /settle` (or `/guarantees` if added) to issue BLS certificates; response includes `{certificate: {claims, signature}, tabId, amount, ttl}`.
  - Optional `/verify` for preflight of headers.
- Enforce scheme/network/payTo/asset/amount/TTL. 


## 4Mica Adapter in Aligned
- New module in the batcher service that uses `rust-sdk-4mica`:
  - Verify certificates during submission (signature, domain, recipient, amount, TTL).
  - Optionally fetch certs on-demand from gateway when initiating remuneration (only on default).

## Docs & Config
- Add CLI/SDK doc snippet: `aligned submit ... --payment-method 4mica-credit --guarantee path/to/cert.json`.
- Env/config for adapter: 4Mica core/gateway URL, expected domain, operator key (fetched via SDK), Aligned payee address, scheme/network.
- Note: default behavior remains escrow; 4Mica is opt-in.

## Rollout Notes
- Start with an allowlist for `4mica-credit` if desired; otherwise keep open.
- Metrics/logging around: payment-method chosen, cert validation failures, and any default-triggered remuneration attempts.
