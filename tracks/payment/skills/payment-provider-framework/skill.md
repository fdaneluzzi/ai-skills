---
name: payment-provider-framework
description: >
  Apply when building a VTEX IO payment connector using the @vtex/payment-provider SDK (Payment Provider
  Framework / PPF). Covers the real TypeScript API of the package: abstract methods authorize/cancel/settle/refund,
  correct type names (AuthorizationRequest, CancellationRequest, etc.), TypeScript 3.x compatibility resolutions,
  PaymentProviderService configuration, card data access via isCardAuthorization/isTokenizedCard, Maybe<T> semantics,
  and complete response shapes. This skill is REQUIRED for any VTEX IO app that extends PaymentProvider — do NOT use
  payment-provider-protocol for VTEX IO apps.
metadata:
  track: payment
  tags:
    - payment-provider-framework
    - ppf
    - vtex-io
    - payment-connector
    - typescript-sdk
    - vtex-payment-provider
  globs:
    - "**/node/service.ts"
    - "**/node/connector.ts"
    - "**/node/index.ts"
    - "**/node/package.json"
    - "**/manifest.json"
  version: "1.0"
  purpose: >
    Implement a VTEX IO payment connector using @vtex/payment-provider that compiles on the
    first attempt — covering the exact SDK API, TypeScript compatibility fixes, and correct response shapes
  applies_to:
    - building a VTEX IO app that extends PaymentProvider
    - implementing authorize/cancel/settle/refund/inbound methods
    - debugging TypeScript compilation errors in a PPF connector
    - configuring node/package.json for the PPF SDK
  excludes:
    - non-VTEX-IO connectors using plain Express (use payment-provider-protocol)
    - idempotency logic (see payment-idempotency)
    - async callback flows for non-IO connectors (see payment-async-flow)
  decision_scope:
    - real method names vs PPP documentation names
    - correct type names exported by @vtex/payment-provider
    - TypeScript 3.x vs 4.x compatibility
    - Maybe<T> semantics (undefined, not null)
    - card data access patterns
  vtex_docs_verified: "2026-03-20"
---

# PPF SDK — VTEX IO Payment Connector

## When this skill applies

Use this skill when:
- Building a VTEX IO app that extends `PaymentProvider` from `@vtex/payment-provider`
- You have a `node/service.ts` that uses `PaymentProviderService`
- Debugging TypeScript compilation errors in a PPF connector (builder-hub errors)
- Writing the `node/package.json` for a PPF VTEX IO app

Do **not** use this skill for:
- Non-VTEX IO connectors (plain Node/Express middleware) — use [`payment-provider-protocol`](../payment-provider-protocol/skill.md)
- Idempotency logic — use [`payment-idempotency`](../payment-idempotency/skill.md)
- PCI compliance details beyond card access patterns — use [`payment-pci-security`](../payment-pci-security/skill.md)

## Decision rules

- **PPP vs PPF**: If the connector lives in a VTEX IO app and uses `@vtex/payment-provider`, it is PPF. Use this skill.
- **Method names**: PPF method names differ from PPP documentation names. Always use the SDK names (see table below).
- **TypeScript version**: The builder-hub uses TypeScript **3.9.7** (fixed). Always pin `"typescript": "3.9.7"` and add `resolutions` to fix type incompatibilities.
- **`vendor` field**: In `manifest.json`, `vendor` MUST match the output of `vtex whoami`. Never use a placeholder.
- **`Maybe<T>`**: In `@vtex/payment-provider`, `Maybe<T> = T | undefined`. Never assign `null` to a `Maybe` field.
- **Card data access**: Card data is always in `request.card`, not in the top-level request. Use the provided type guards.

## Hard constraints

### Constraint: Use the real SDK method names — not the PPP documentation names

The `PaymentProvider` abstract class exposes methods named `authorize`, `cancel`, `settle`, `refund`, and optionally `inbound`. These are **not** named `createPayment`, `cancelPayment`, `settlePayment`, or `refundPayment` as in the PPP public documentation.

**Why this matters**
The public PPP documentation describes the HTTP protocol, not the TypeScript SDK. Using PPP names causes `Method 'createPayment' not implemented` errors that prevent `vtex link` from succeeding.

**Detection**
If the connector class implements methods named `createPayment`, `cancelPayment`, `settlePayment`, or `refundPayment`, STOP and rename them.

**Method name mapping**

| PPP Documentation Name | Real PPF SDK Method Name | Request Type              | Response Type              |
|------------------------|--------------------------|---------------------------|----------------------------|
| Create Payment         | `authorize`              | `AuthorizationRequest`    | `AuthorizationResponse`    |
| Cancel Payment         | `cancel`                 | `CancellationRequest`     | `CancellationResponse`     |
| Capture/Settle Payment | `settle`                 | `SettlementRequest`       | `SettlementResponse`       |
| Refund Payment         | `refund`                 | `RefundRequest`           | `RefundResponse`           |
| Inbound Request        | `inbound` (optional)     | `InboundRequest`          | `InboundResponse`          |

**Correct**
```typescript
import {
  PaymentProvider,
  AuthorizationRequest,
  AuthorizationResponse,
  CancellationRequest,
  CancellationResponse,
  SettlementRequest,
  SettlementResponse,
  RefundRequest,
  RefundResponse,
  InboundRequest,
  InboundResponse,
} from "@vtex/payment-provider"

export default class MyConnector extends PaymentProvider {
  async authorize(request: AuthorizationRequest): Promise<AuthorizationResponse> {
    // implement
  }

  async cancel(request: CancellationRequest): Promise<CancellationResponse> {
    // implement
  }

  async settle(request: SettlementRequest): Promise<SettlementResponse> {
    // implement
  }

  async refund(request: RefundRequest): Promise<RefundResponse> {
    // implement
  }

  async inbound(request: InboundRequest): Promise<InboundResponse> {
    // implement (optional but recommended)
  }
}
```

**Wrong**
```typescript
// WRONG: These method names do not exist in the SDK
// "Method 'createPayment' not implemented" error at runtime
export default class MyConnector extends PaymentProvider {
  async createPayment(request: any): Promise<any> { ... }
  async cancelPayment(request: any): Promise<any> { ... }
  async settlePayment(request: any): Promise<any> { ... }
  async refundPayment(request: any): Promise<any> { ... }
}
```

---

### Constraint: Use the real type names from @vtex/payment-provider

The `@vtex/payment-provider` package exports specific TypeScript types. Using names from the PPP documentation (like `CreatePaymentRequest`) causes `Module has no exported member` compilation errors.

**Why this matters**
Builder-hub compiles TypeScript strictly. Any reference to a non-existent export is a hard compile error that blocks `vtex link`.

**Detection**
If the code imports `CreatePaymentRequest`, `CancelPaymentRequest`, or any other type not listed below, STOP and replace with the correct export name.

**All exported types — complete reference**

```typescript
// ── Requests ──────────────────────────────────────────────────────────────────
// AuthorizationRequest is a union type:
// CreditCardAuthorization | DebitCardAuthorization | BankInvoiceAuthorization | ...
import {
  AuthorizationRequest,   // union of all authorization variants
  CancellationRequest,    // { paymentId, requestId, transactionId, authorizationId, tid? }
  SettlementRequest,      // { paymentId, requestId, transactionId, value, authorizationId, tid? }
  RefundRequest,          // { paymentId, requestId, transactionId, value, settleId, authorizationId? }
  InboundRequest,         // { paymentId, requestId, transactionId, authorizationId, tid, requestData }
} from "@vtex/payment-provider"

// ── Responses ─────────────────────────────────────────────────────────────────
import {
  AuthorizationResponse,  // union: CreditCardAuthorized | FailedAuthorization | PendingAuthorization | ...
  CancellationResponse,   // { paymentId, cancellationId, code, message }  ← NO requestId!
  SettlementResponse,     // { paymentId, requestId, settleId, value, code, message }
  RefundResponse,         // { paymentId, requestId, refundId, value, code, message }
  InboundResponse,        // { paymentId, requestId, code, message, responseData: { statusCode, contentType, content } }
} from "@vtex/payment-provider"

// ── Concrete response types ────────────────────────────────────────────────────
import {
  CreditCardAuthorized,   // approved card: status='approved', delayToAutoSettle, delayToAutoSettleAfterAntifraud
  FailedAuthorization,    // denied: status='denied', tid, acquirer, authorizationId, delayToCancel, code, message
} from "@vtex/payment-provider"

// ── Card types ────────────────────────────────────────────────────────────────
import {
  Card,           // raw card (PCI): { number, holder, expiration, csc, document }
  TokenizedCard,  // VTEX tokens: { numberToken, holderToken, cscToken, bin, numberLength, expiration }
  CardAuthorization, // the whole authorization request when payment method is card
} from "@vtex/payment-provider"

// ── Type guards (use these for narrowing) ─────────────────────────────────────
import {
  isCardAuthorization, // (request: AuthorizationRequest) => request is CardAuthorization
  isTokenizedCard,     // (card: Card | TokenizedCard) => card is TokenizedCard
} from "@vtex/payment-provider"

// ── Service infrastructure ────────────────────────────────────────────────────
import {
  PaymentProvider,         // abstract class to extend
  PaymentProviderService,  // service factory
  SecureExternalClient,    // extend for Secure Proxy HTTP calls (not ExternalClient)
} from "@vtex/payment-provider"
```

---

### Constraint: CancellationResponse does NOT have a requestId field

`CancellationResponse` contains `{ paymentId, cancellationId, code, message }`. There is **no** `requestId` field. Only `SettlementResponse` and `RefundResponse` include `requestId`.

**Why this matters**
Adding `requestId` to `CancellationResponse` causes a TypeScript compilation error: `Object literal may only specify known properties`. This is a frequent mistake because the PPP HTTP response for cancellation does include `requestId`.

**Detection**
If the `cancel()` method returns an object with a `requestId` field, STOP and remove it.

**Correct**
```typescript
async cancel(request: CancellationRequest): Promise<CancellationResponse> {
  const result = await this.stripe.paymentIntents.cancel(request.authorizationId)

  return {
    paymentId: request.paymentId,
    cancellationId: result.id,
    code: "cancel-success",
    message: "Cancelled successfully",
    // ← NO requestId here
  }
}
```

**Wrong**
```typescript
async cancel(request: CancellationRequest): Promise<CancellationResponse> {
  return {
    paymentId: request.paymentId,
    cancellationId: "xxx",
    code: "ok",
    message: "Cancelled",
    requestId: request.requestId,  // ← COMPILE ERROR: property does not exist on CancellationResponse
  }
}
```

---

### Constraint: SettlementResponse and RefundResponse require a `value` field

Both `SettlementResponse` and `RefundResponse` have `value: number` as a required field. Omitting it causes a TypeScript compilation error.

**Why this matters**
`value` is typed as `number` (not optional). The TypeScript compiler rejects the object literal if it is absent.

**Correct**
```typescript
async settle(request: SettlementRequest): Promise<SettlementResponse> {
  const result = await this.stripe.paymentIntents.capture(request.authorizationId)

  return {
    paymentId: request.paymentId,
    requestId: request.requestId,
    settleId: result.id,
    value: request.value,  // ← required!
    code: "capture-success",
    message: "Captured",
  }
}

async refund(request: RefundRequest): Promise<RefundResponse> {
  const result = await this.stripe.refunds.create({ payment_intent: request.authorizationId })

  return {
    paymentId: request.paymentId,
    requestId: request.requestId,
    refundId: result.id,
    value: request.value,  // ← required!
    code: "refund-success",
    message: "Refunded",
  }
}
```

---

### Constraint: Maybe<T> accepts undefined, never null

In `@vtex/payment-provider`, `Maybe<T>` is defined as `T | undefined`. It does **not** include `null`. Assigning `null` to any `Maybe<T>` field causes `Type 'null' is not assignable to type 'Maybe<T>'` because `@vtex/tsconfig` sets `strictNullChecks: true`.

**Why this matters**
The package's `tsconfig` enables `strictNullChecks`. `null` and `undefined` are distinct types. `Maybe<boolean>` means `boolean | undefined` — `null` is rejected by the compiler.

**Detection**
If the code assigns `null` to `isNewTokenization`, `generatedCardToken`, or any other `Maybe<T>` field, STOP and replace with `undefined`.

**Correct**
```typescript
const approved: CreditCardAuthorized = {
  paymentId: request.paymentId,
  status: "approved",
  authorizationId: "pi_xxx",
  tid: "pi_xxx",
  nsu: "pi_xxx",
  acquirer: "Stripe",
  code: "requires_capture",
  message: "Payment authorized",
  delayToAutoSettle: 21600,
  delayToAutoSettleAfterAntifraud: 1800,
  delayToCancel: 300,
  isNewTokenization: undefined,    // ← Maybe<boolean> — use undefined, not null
  generatedCardToken: undefined,   // ← Maybe<CreditCardToken> — use undefined, not null
}
```

**Wrong**
```typescript
const approved: CreditCardAuthorized = {
  // ...
  isNewTokenization: null,    // ← COMPILE ERROR: null not assignable to Maybe<boolean>
  generatedCardToken: null,   // ← COMPILE ERROR: null not assignable to Maybe<CreditCardToken>
}
```

---

### Constraint: Access card data through request.card with type guards

Card data (tokens or raw) lives in `request.card` (typed as `Card | TokenizedCard`), not at the top level of the request. Accessing `request.numberToken` directly causes `Property 'numberToken' does not exist on type 'CardAuthorization'`.

**Why this matters**
`AuthorizationRequest` is a union type. For card payments it resolves to `CardAuthorization`, which has a `.card` field typed as `Card | TokenizedCard`. Token properties (`numberToken`, `holderToken`, `cscToken`) exist only on `TokenizedCard`. Without type narrowing, the compiler rejects any property access.

**Detection**
If the code accesses `request.numberToken`, `request.cscToken`, or `request.card.numberToken` without first calling `isCardAuthorization()` and `isTokenizedCard()`, STOP and add proper type narrowing.

**Correct pattern for card data access**
```typescript
import { isCardAuthorization, isTokenizedCard, FailedAuthorization } from "@vtex/payment-provider"

async authorize(request: AuthorizationRequest): Promise<AuthorizationResponse> {
  // Step 1: Check that this is a card payment
  if (!isCardAuthorization(request)) {
    // Non-card payment method (e.g. BankInvoice) — handle separately or deny
    const denied: FailedAuthorization = {
      paymentId: request.paymentId,
      status: "denied",
      tid: null,
      acquirer: "MyPSP",
      authorizationId: null,
      delayToCancel: 300,
      delayToAutoSettle: null,
      delayToAutoSettleAfterAntifraud: null,
      code: "UNSUPPORTED_PAYMENT_METHOD",
      message: "Only card payments are supported",
    }
    return denied
  }

  // Step 2: Narrow card type — Secure Proxy provides TokenizedCard
  if (!isTokenizedCard(request.card)) {
    // Raw card data (PCI-certified connectors only) — typically deny for VTEX IO apps
    const denied: FailedAuthorization = {
      paymentId: request.paymentId,
      status: "denied",
      tid: null,
      acquirer: "MyPSP",
      authorizationId: null,
      delayToCancel: 300,
      delayToAutoSettle: null,
      delayToAutoSettleAfterAntifraud: null,
      code: "RAW_CARD_NOT_SUPPORTED",
      message: "Raw card data not supported; use Secure Proxy",
    }
    return denied
  }

  // Step 3: Now card is typed as TokenizedCard — access tokens safely
  const card = request.card  // TokenizedCard
  const {
    numberToken,          // ← from card, not from request
    holderToken,
    cscToken,
    bin,
    numberLength,
    expiration,           // { month: string, year: string }
  } = card

  // secureProxyUrl is on the request, not on the card
  const { paymentId, secureProxyUrl } = request

  // Use secureProxyUrl to call the acquirer
  const result = await this.callAcquirerViaProxy(secureProxyUrl, {
    numberToken,
    holderToken,
    cscToken,
    expMonth: expiration.month,
    expYear: expiration.year,
    amount: request.value,
  })

  // ...
}
```

**Wrong**
```typescript
async authorize(request: AuthorizationRequest): Promise<AuthorizationResponse> {
  // WRONG: numberToken does not exist directly on AuthorizationRequest
  const { numberToken, cscToken } = request  // ← COMPILE ERROR

  // ALSO WRONG: accessing card.numberToken without narrowing
  const { numberToken } = request.card  // ← COMPILE ERROR: Card | TokenizedCard has no numberToken
}
```

---

### Constraint: PaymentProviderService must receive clients as { implementation, options }

`PaymentProviderService` requires the `clients` field to be an object with `implementation` and `options` keys. Passing `Clients` directly causes a TypeScript error.

**Why this matters**
The `ServiceConfig.clients` type requires `{ implementation: ClientsConfig, options: Record<string, InstanceOptions> }`. Passing a class directly as `clients` does not match this shape.

**Correct**
```typescript
// node/service.ts
import { PaymentProviderService } from "@vtex/payment-provider"
import MyConnector from "./connector"
import { Clients } from "./clients"

export default new PaymentProviderService({
  connector: MyConnector,
  clients: {
    implementation: Clients,
    options: {},   // ← required even if empty
  },
})
```

**Wrong**
```typescript
export default new PaymentProviderService({
  connector: MyConnector,
  clients: Clients,  // ← COMPILE ERROR: does not match ServiceConfig.clients shape
})
```

---

### Constraint: Fix TypeScript 3.x incompatibility with resolutions in node/package.json

The VTEX builder-hub uses TypeScript **3.9.7**. `@vtex/api@6.50.x` depends on `@types/express-serve-static-core >= 4.17.21` and `@types/koa >= 2.13.x`, which use TypeScript 4.x syntax (template literal types, `import type X = require()`). These cause **parse errors** in TS 3.x that `skipLibCheck` cannot suppress.

**Why this matters**
These are parser-level failures — the TS 3.x parser does not understand the syntax in those `.d.ts` files. `skipLibCheck` only skips type-checking, not parsing. The build fails before any type checking occurs.

**The fix — add `resolutions` to `node/package.json`**

Yarn `resolutions` override the installed version of a dependency at any nesting level, including transitive dependencies.

**Correct `node/package.json`**
```json
{
  "name": "node",
  "version": "1.0.0",
  "main": "index.js",
  "license": "UNLICENSED",
  "dependencies": {
    "@vtex/payment-provider": "1.x"
  },
  "devDependencies": {
    "@types/node": "^12.0.0",
    "@vtex/api": "6.x",
    "@vtex/tsconfig": "0.x",
    "typescript": "3.9.7"
  },
  "resolutions": {
    "@types/express-serve-static-core": "4.17.20",
    "@types/koa": "2.11.6"
  }
}
```

**Why these specific versions**

| Package | Max Compatible Version | Reason |
|---------|----------------------|--------|
| `@types/express-serve-static-core` | `4.17.20` | `4.17.21` introduced template literal types (TS 4.1 feature) |
| `@types/koa` | `2.11.6` | Later versions use `import type X = require()` (TS 4.x syntax) |

**Wrong**
```json
{
  "devDependencies": {
    "typescript": "4.x"
  }
}
```
`"4.x"` is not valid semver for npm/yarn and causes `yarn install` to fail. Use `"3.9.7"` (exact pin).

**After adding resolutions, run `yarn install` in the `node/` folder before `vtex link`.**

---

### Constraint: vendor in manifest.json must match vtex whoami

The `vendor` field in `manifest.json` must be the exact account name returned by `vtex whoami`. The builder-hub rejects any app whose vendor does not match the logged-in account.

**Why this matters**
The builder-hub validates that the publishing account owns the vendor namespace. A placeholder like `"myvendor"` or `"acme"` causes an immediate build rejection.

**Detection**
If `manifest.json` contains `"vendor": "myvendor"` or any placeholder, STOP. Run `vtex whoami` and replace the vendor with the actual account name.

**Correct**
```json
{
  "vendor": "myaccount",
  "name": "connector-mypsp",
  "version": "0.0.1",
  "title": "MyPSP Payment Connector",
  "description": "VTEX IO payment connector for MyPSP",
  "builders": {
    "node": "6.x",
    "docs": "0.x"
  }
}
```

---

## Preferred pattern — Complete PPF connector skeleton

```typescript
// node/connector.ts
import {
  PaymentProvider,
  AuthorizationRequest,
  AuthorizationResponse,
  CancellationRequest,
  CancellationResponse,
  SettlementRequest,
  SettlementResponse,
  RefundRequest,
  RefundResponse,
  InboundRequest,
  InboundResponse,
  CreditCardAuthorized,
  FailedAuthorization,
  isCardAuthorization,
  isTokenizedCard,
} from "@vtex/payment-provider"

export default class MyPspConnector extends PaymentProvider {
  async authorize(request: AuthorizationRequest): Promise<AuthorizationResponse> {
    if (!isCardAuthorization(request)) {
      const denied: FailedAuthorization = {
        paymentId: request.paymentId,
        status: "denied",
        tid: null,
        acquirer: "MyPSP",
        authorizationId: null,
        delayToCancel: 300,
        delayToAutoSettle: null,
        delayToAutoSettleAfterAntifraud: null,
        code: "UNSUPPORTED",
        message: "Only card payments supported",
      }
      return denied
    }

    if (!isTokenizedCard(request.card)) {
      const denied: FailedAuthorization = {
        paymentId: request.paymentId,
        status: "denied",
        tid: null,
        acquirer: "MyPSP",
        authorizationId: null,
        delayToCancel: 300,
        delayToAutoSettle: null,
        delayToAutoSettleAfterAntifraud: null,
        code: "RAW_CARD_REJECTED",
        message: "Raw card not supported; Secure Proxy required",
      }
      return denied
    }

    const { paymentId, value, secureProxyUrl } = request
    const { numberToken, holderToken, cscToken, expiration } = request.card

    // Call acquirer via Secure Proxy
    const result = await this.acquirerClients.myPsp.authorize(secureProxyUrl, {
      amount: Math.round(value * 100), // convert to cents
      card: { numberToken, holderToken, cscToken, expiration },
      idempotencyKey: `authorize-${paymentId}`,
    })

    if (result.status === "requires_capture" || result.status === "succeeded") {
      const approved: CreditCardAuthorized = {
        paymentId,
        status: "approved",
        authorizationId: result.id,
        tid: result.id,
        nsu: result.id,
        acquirer: "MyPSP",
        code: result.status,
        message: "Payment authorized",
        delayToAutoSettle: 21600,               // 6 hours
        delayToAutoSettleAfterAntifraud: 1800,  // 30 minutes
        delayToCancel: 300,
        isNewTokenization: undefined,            // Maybe<boolean> — use undefined, not null
        generatedCardToken: undefined,           // Maybe<CreditCardToken> — use undefined, not null
      }
      return approved
    }

    const denied: FailedAuthorization = {
      paymentId,
      status: "denied",
      tid: null,
      acquirer: "MyPSP",
      authorizationId: null,
      delayToCancel: 300,
      delayToAutoSettle: null,
      delayToAutoSettleAfterAntifraud: null,
      code: result.last_payment_error?.code ?? "DECLINED",
      message: result.last_payment_error?.message ?? "Payment declined",
    }
    return denied
  }

  async cancel(request: CancellationRequest): Promise<CancellationResponse> {
    const result = await this.acquirerClients.myPsp.cancel(request.authorizationId)

    return {
      paymentId: request.paymentId,
      cancellationId: result.id,
      code: "cancel-success",
      message: "Cancelled",
      // ← NO requestId — CancellationResponse does not have this field
    }
  }

  async settle(request: SettlementRequest): Promise<SettlementResponse> {
    const result = await this.acquirerClients.myPsp.capture(request.authorizationId, {
      amount: Math.round(request.value * 100),
    })

    return {
      paymentId: request.paymentId,
      requestId: request.requestId,
      settleId: result.id,
      value: request.value,  // ← required!
      code: "capture-success",
      message: "Captured",
    }
  }

  async refund(request: RefundRequest): Promise<RefundResponse> {
    const result = await this.acquirerClients.myPsp.refund(request.authorizationId, {
      amount: Math.round(request.value * 100),
    })

    return {
      paymentId: request.paymentId,
      requestId: request.requestId,
      refundId: result.id,
      value: request.value,  // ← required!
      code: "refund-success",
      message: "Refunded",
    }
  }

  async inbound(request: InboundRequest): Promise<InboundResponse> {
    return {
      paymentId: request.paymentId,
      requestId: request.requestId,
      code: "received",
      message: "Webhook received",
      responseData: {                   // ← field is responseData, NOT body
        statusCode: 200,
        contentType: "application/json",
        content: JSON.stringify({ received: true }),
      },
    }
  }
}
```

```typescript
// node/service.ts
import { PaymentProviderService } from "@vtex/payment-provider"
import MyPspConnector from "./connector"
import { Clients } from "./clients"

export default new PaymentProviderService({
  connector: MyPspConnector,
  clients: {
    implementation: Clients,
    options: {},  // ← required even if empty
  },
})
```

## Pre-link checklist — run before every `vtex link`

```
[ ] vtex whoami → value matches vendor in manifest.json exactly
[ ] node/package.json has resolutions: { "@types/express-serve-static-core": "4.17.20", "@types/koa": "2.11.6" }
[ ] typescript version in devDependencies is a pinned value like "3.9.7" (not a range like "4.x")
[ ] yarn install ran in node/ after adding resolutions
[ ] Connector implements: authorize, cancel, settle, refund (not createPayment/cancelPayment/...)
[ ] authorize() uses isCardAuthorization() + isTokenizedCard() for narrowing
[ ] secureProxyUrl comes from request (CardAuthorization), not from the card
[ ] Card tokens (numberToken, etc.) come from request.card (TokenizedCard), not from request directly
[ ] CreditCardAuthorized uses isNewTokenization: undefined (not null)
[ ] CancellationResponse does NOT include requestId
[ ] SettlementResponse includes value: number
[ ] RefundResponse includes value: number
[ ] InboundResponse uses responseData: { statusCode, contentType, content } (not body)
[ ] PaymentProviderService receives clients: { implementation: Clients, options: {} }
```

## Common failure modes

- **Wrong method names** — Implementing `createPayment` / `cancelPayment` / `settlePayment` / `refundPayment` instead of `authorize` / `cancel` / `settle` / `refund`. Results in `Method 'createPayment' not implemented` at runtime.
- **Wrong type names** — Importing `CreatePaymentRequest` or `CancelPaymentResponse` from `@vtex/payment-provider`. These do not exist. Results in `Module has no exported member` compile error.
- **null in Maybe<T> fields** — Assigning `null` to `isNewTokenization` or `generatedCardToken`. Results in `Type 'null' is not assignable to type 'Maybe<boolean>'`. Use `undefined`.
- **requestId in CancellationResponse** — Adding `requestId` to the cancel response. Results in `Object literal may only specify known properties`.
- **Missing value in SettlementResponse / RefundResponse** — Omitting `value: number` from settle or refund responses. Results in a compile error.
- **InboundResponse using body** — Returning `body: "..."` instead of `responseData: { statusCode, contentType, content }`.
- **TypeScript version range** — Using `"typescript": "4.x"` in package.json. `4.x` is not valid semver for yarn. Use `"3.9.7"`.
- **Missing resolutions** — Not adding `resolutions` to `node/package.json`. Results in template literal type parse errors from `@types/express-serve-static-core >= 4.17.21`.
- **Wrong vendor** — Keeping a placeholder vendor in `manifest.json`. Builder-hub rejects it immediately.
- **clients passed as class** — Passing `clients: Clients` instead of `clients: { implementation: Clients, options: {} }` to `PaymentProviderService`. Results in a TypeScript shape mismatch error.

## Build sequence

```bash
# 1. Verify account name — this is your vendor
vtex whoami

# 2. Install dependencies WITH resolutions applied
cd node && yarn install && cd ..

# 3. Link the app
vtex link

# 4. Configure PSP credentials (after successful link)
vtex settings set {vendor}.{app-name} stripeSecretKey sk_test_...

# 5. Verify manifest endpoint
curl https://dev--{account}.myvtex.com/_v/{vendor}.{app-name}/v0/manifest

# 6. Run the homologation test suite
vtex install vtex.payment-provider-test-suite
# Open: https://dev--{account}.myvtex.com/admin/payment-provider-test-suite
```

## Related skills

- [`payment-provider-protocol`](../payment-provider-protocol/skill.md) — PPP HTTP protocol for non-VTEX IO connectors (plain Express/Node middleware)
- [`payment-pci-security`](../payment-pci-security/skill.md) — Secure Proxy, PCI compliance, card data rules
- [`payment-idempotency`](../payment-idempotency/skill.md) — `paymentId`/`requestId` idempotency and state machine
- [`payment-async-flow`](../payment-async-flow/skill.md) — Async payment methods and callback handling

## Reference

- [Payment Provider Framework](https://developers.vtex.com/docs/guides/payments-integration-payment-provider-framework) — VTEX IO SDK overview, including setup, Secure Proxy requirement, and app structure
- [Payment Provider Framework (GitHub example)](https://github.com/vtex-apps/payment-provider-example) — Official boilerplate with correct method names and types
- [Secure Proxy](https://developers.vtex.com/docs/guides/payments-integration-secure-proxy) — Required for all VTEX IO connectors; covers `secureProxyUrl`, token forwarding, and `X-PROVIDER-Forward-To`
- [Payment Provider Protocol API Reference](https://developers.vtex.com/docs/api-reference/payment-provider-protocol) — HTTP-level spec (useful for understanding the protocol, not the SDK names)
