# Payment Connector Development

Best practices for developing VTEX payment connectors, including PPP endpoints, PPF SDK (VTEX IO), idempotency, PCI compliance, and integration patterns. This track covers everything needed to build production-ready payment connectors that integrate with the VTEX Payment Gateway.

## Overview

VTEX offers two ways to build a payment connector:

- **PPP (Payment Provider Protocol)** — Any HTTP server (Node, Express, etc.) implementing the VTEX payment protocol. Use the `payment-provider-protocol` skill.
- **PPF (Payment Provider Framework)** — A VTEX IO app that uses the `@vtex/payment-provider` TypeScript SDK. Uses a class-based abstract API with different method and type names than PPP. **Use the `payment-provider-framework` skill for all VTEX IO connectors.**

> ⚠️ **PPF method names differ from PPP documentation names.** The public VTEX docs describe the HTTP protocol (`createPayment`, `cancelPayment`, etc.), but the `@vtex/payment-provider` SDK uses `authorize`, `cancel`, `settle`, `refund`. Using PPP names in a PPF app causes compile errors.

## Skills

| Skill | Description | Link |
|-------|-------------|------|
| **PPF SDK — VTEX IO Connector** | Build a VTEX IO payment connector using `@vtex/payment-provider`. Covers real method names (authorize/cancel/settle/refund), correct type names, TypeScript 3.x resolutions, card access patterns, and complete response shapes. **Use this for any VTEX IO connector.** | [skills/payment-provider-framework/skill.md](skills/payment-provider-framework/skill.md) |
| **PPP Endpoint Implementation** | Implement all nine required PPP endpoints for non-VTEX IO connectors: Manifest, Create Payment, Cancel, Capture/Settle, Refund, Inbound Request, Create Auth Token, Provider Auth Redirect, and Get Credentials. **Use this for plain HTTP middleware connectors only.** | [skills/payment-provider-protocol/skill.md](skills/payment-provider-protocol/skill.md) |
| **Idempotency & Duplicate Prevention** | Use `paymentId` and `requestId` as idempotency keys, implement a payment state machine, and handle Gateway retries that can occur for up to 7 days on `undefined` status payments. | [skills/payment-idempotency/skill.md](skills/payment-idempotency/skill.md) |
| **Asynchronous Payment Flows & Callbacks** | Handle asynchronous payment methods (Boleto, Pix, bank transfers) by returning `undefined` status, managing `callbackUrl` notifications, and validating `X-VTEX-signature` headers. | [skills/payment-async-flow/skill.md](skills/payment-async-flow/skill.md) |
| **PCI Compliance & Secure Proxy** | Understand PCI DSS requirements, use `secureProxyUrl` for card tokenization, protect sensitive data, and route acquirer calls through the Secure Proxy. | [skills/payment-pci-security/skill.md](skills/payment-pci-security/skill.md) |

## Recommended Learning Order

### For VTEX IO connectors (PPF)
1. **Start with PPF SDK** — Learn the `@vtex/payment-provider` SDK API: real method names, type names, TypeScript compatibility, and response shapes.
2. **Learn Idempotency** — Understand how to prevent duplicate charges and handle retries correctly.
3. **Implement PCI Security** — Ensure your connector uses Secure Proxy correctly for card payments.
4. **Add Async Flow Support** — Handle asynchronous payment methods like Boleto and Pix.

### For non-VTEX IO connectors (PPP)
1. **Start with PPP Endpoints** — Understand the complete protocol structure and all nine endpoints first.
2. **Learn Idempotency** — Understand how to prevent duplicate charges and handle retries correctly.
3. **Add Async Flow Support** — Learn how to handle asynchronous payment methods and callbacks.
4. **Implement PCI Security** — Ensure your connector meets PCI compliance requirements and uses Secure Proxy correctly.

## Key Constraints Summary

### PPF (VTEX IO) specific
- **Use SDK method names, not PPP names** — `authorize` not `createPayment`, `cancel` not `cancelPayment`, `settle` not `settlePayment`, `refund` not `refundPayment`.
- **`vendor` in manifest.json must match `vtex whoami`** — Never use a placeholder.
- **Pin TypeScript to `"3.9.7"` and add `resolutions`** — `@types/express-serve-static-core: "4.17.20"` and `@types/koa: "2.11.6"` prevent TS 3.x parse errors.
- **`Maybe<T> = T | undefined`** — Never assign `null` to a `Maybe` field.
- **`CancellationResponse` has no `requestId`** — Only `SettlementResponse` and `RefundResponse` have it.
- **`SettlementResponse` and `RefundResponse` require `value: number`** — Never omit this field.
- **Card data is in `request.card`** — Use `isCardAuthorization()` + `isTokenizedCard()` to narrow types before accessing tokens.
- **`PaymentProviderService` requires `clients: { implementation, options: {} }`** — Do not pass the class directly.

### General (PPP and PPF)
- **All nine PPP endpoints are required for homologation** — Manifest, Create Payment, Cancel, Capture/Settle, Refund, Inbound Request, Create Auth Token, Provider Auth Redirect, Get Credentials.
- **`paymentId` is the idempotency key for Create Payment** — If the Gateway retries with the same `paymentId`, return the exact same response without creating a new transaction.
- **`requestId` is the idempotency key for Cancel/Capture/Refund** — Each `requestId` represents a single logical operation. Duplicate requests must return the cached result.
- **Never store, log, or transmit raw card data** — Use Secure Proxy tokenization. Raw card data is a PCI violation and a data breach risk.
- **Validate `X-VTEX-signature` on all callback URLs** — Unsigned callbacks are a security vulnerability. Always verify the signature before processing.
- **Return `undefined` status for asynchronous payments** — Do not return `approved` or `denied` until the acquirer confirms. Premature status changes break the payment flow.
- **Respect the 7-day retry window** — The Gateway retries `undefined` payments for up to 7 days. Your connector must handle retries gracefully.

## Related Tracks

- **For marketplace order payments**, see [Track 4: Marketplace Integration](../marketplace/index.md) — Understand how payment connectors integrate with marketplace order flows.
- **For VTEX IO payment apps**, see [Track 3: Custom VTEX IO Apps](../vtex-io/index.md) — Build VTEX IO apps that use payment connectors.
