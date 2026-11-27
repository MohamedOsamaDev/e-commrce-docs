
# Payment Lifecycle

## Overview

Payment has an independent lifecycle from Order.

Payment status represents the financial state of the transaction, while Order status represents the fulfillment state.

## Payment Statuses

| Status | Description |
|---|---|
| pending | Payment created but not completed |
| processing | Payment is being processed |
| paid | Payment successfully completed |
| failed | Payment attempt failed |
| cancelled | Payment cancelled |
| expired | Checkout session expired |
| partially_refunded | Part of the payment refunded |
| refunded | Fully refunded |

## Online Payment Lifecycle

```text
pending
   |
   v
processing
   |
   |----> paid
   |
   |----> failed
   |
   |----> cancelled
   |
   |----> expired
```

Payment confirmation must come from a verified provider response or webhook.

A browser redirect alone must never be considered proof of payment.

## COD Lifecycle

```text
pending
   |
   v
Order Processing
   |
   v
Payment Collection
   |
   v
paid
```

COD payment remains pending until the money is actually collected.

## Checkout Management

Checkout information is stored directly inside Payment.

Fields:

- providerSessionId
- checkoutUrl
- expiresAt

Checkout information can be updated when a new checkout session is generated.

The current schema does not maintain historical payment attempts.

## Transaction Management

Transaction information is stored directly inside Payment.

Fields:

- providerPaymentId
- transactionId

These fields are populated or updated through verified provider responses.

## Refund Lifecycle

Refund operations are stored in the embedded `refunds[]` array.

### Refund Statuses

| Status | Description |
|---|---|
| pending | Refund requested |
| processing | Refund being processed |
| succeeded | Refund successfully completed |
| failed | Refund failed |
| cancelled | Refund cancelled |

### Refund Flow

```text
Payment: paid
      |
      v
Request Refund
      |
      v
Validate Refund Amount
      |
      v
Process Through Provider
      |
      v
Verify Refund Result
      |
      v
Update Refund Status
      |
      v
Recalculate Payment Status
```

### Refund Rules

- Full refunds are supported.
- Partial refunds are supported.
- Only successful refunds count toward refundedAmount.
- Total successful refunds must never exceed the original payment amount.
- Failed or cancelled refunds do not reduce the remaining refundable amount.

### Refund Virtuals

`refundedAmount`:

Total amount of successful refunds.

`remainingRefundableAmount`:

Original payment amount minus successful refunds.

## Payment and Order Synchronization

Payment and Order have separate status fields.

When Payment becomes paid:
- Update Order payment-related business state.
- Continue the appropriate Order lifecycle.

When Payment becomes refunded:
- Apply the appropriate Order business rules.
- Do not automatically assume that every refund requires Order cancellation.

## Webhook Processing

Webhook handling must:

1. Verify the provider signature.
2. Validate the event.
3. Verify the related Payment.
4. Prevent duplicate event processing.
5. Validate the expected state transition.
6. Update Payment.
7. Synchronize the Order when necessary.

Webhook idempotency must be implemented using durable event deduplication outside the current Payment schema.

## Financial Integrity

All financial updates must be performed through backend services.

Refund amount validation and status updates must be protected against concurrent requests using atomic operations or MongoDB transactions.
