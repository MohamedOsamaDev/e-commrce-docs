# Payment Lifecycle

## 1. Overview

Payment has an independent lifecycle from Order.

Each module owns its own business state:

- **Payment:** Represents the financial state of the transaction.
- **Order:** Represents the purchase and fulfillment state.

Payment status is owned by the Payment Module.

Order status must not duplicate Payment status.

Changes that require coordination between the two modules must happen through controlled backend services.

A Payment becoming `paid` does not mean that Order fulfillment is complete.

## 2. Payment Statuses

The Payment lifecycle uses the following statuses:

| Status | Description |
|---|---|
| `pending` | Payment created but not completed |
| `processing` | Payment operation is being processed |
| `paid` | Payment successfully completed |
| `failed` | Payment operation failed |
| `cancelled` | Payment cancelled |
| `expired` | Checkout session expired |
| `partially_refunded` | Part of the successful payment amount was refunded |
| `refunded` | The entire successful payment amount was refunded |

Payment status must only be changed through authorized backend services.

## 3. Online Payment Lifecycle

Online Payment starts with a pending Payment record.

The checkout session is initialized through the corresponding Connector.

```text
Create Payment
      |
      v
   pending
      |
      v
Initialize Checkout
      |
      v
   processing
      |
      +-------------------+
      |         |         |
      v         v         v
    paid      failed   cancelled
      |
      +-------------------+
      |
      v
 Refund Processing
      |
      +----------------------+
      |                      |
      v                      v
partially_refunded       refunded
```

The diagram describes the general lifecycle. Actual status transitions must be validated against the current Payment state and the verified provider event.

### Payment Confirmation

Payment confirmation must come from a verified server-side Provider result.

A browser redirect or frontend callback alone must never be considered proof of payment.

The Provider result reaches the Payment Core through the Connector's normalized event contract.

## 4. COD Lifecycle

COD does not initialize an online checkout session.

Its lifecycle is:

```text
Create COD Payment
        |
        v
      pending
        |
        v
 Order Fulfillment
        |
        v
 Cash Collection
        |
        v
       paid
```

COD Payment remains `pending` until the money is actually collected.

Creating an Order, dispatching it, or marking it delivered must not automatically mark its Payment as `paid`.

The transition to `paid` requires confirmation that the cash was collected.

## 5. Checkout Management

Checkout information is stored directly inside the Payment document.

There is no separate persisted Checkout Session collection or entity.

The current checkout information includes:

- `providerSessionId`
- `checkoutUrl`
- `expiresAt`

### Checkout Initialization

For Online Payments:

1. Payment is created.
2. Checkout initialization is requested.
3. The corresponding Connector creates the external Provider session.
4. The Connector returns normalized checkout information.
5. The Payment service stores the current checkout information.

COD Payments skip this process.

### Checkout Updates

When a new checkout session is generated, the current checkout information is updated.

The Payment document does not maintain an embedded attempts array.

Previous checkout attempts are not retained in the current Payment schema.

### Checkout Expiration

When the Provider checkout session expires, the corresponding Payment may transition to `expired` according to the validated Provider result or trusted expiration handling.

A local expiration timestamp alone must not be treated as proof that no successful payment occurred externally.

## 6. Transaction Management

Transaction information is stored directly inside Payment.

Relevant fields include:

- `providerPaymentId`
- `transactionId`

These fields are populated or updated through verified Provider results normalized by the Connector.

The Payment Core must validate:

- The Payment identity.
- The associated Provider.
- The expected amount.
- The expected currency.
- The current Payment status.
- Whether the incoming event represents a valid transition.

Provider-specific response formats must not be handled inside the Payment Core.

## 7. Refund Lifecycle

Refund operations are stored inside the embedded `refunds[]` array.

A refund is a financial operation associated with its parent Payment.

### Refund Statuses

| Status | Description |
|---|---|
| `pending` | Refund requested |
| `processing` | Refund is being processed |
| `succeeded` | Refund successfully completed |
| `failed` | Refund failed |
| `cancelled` | Refund cancelled |

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
Create Refund Operation
      |
      v
Process Through Connector
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

The Connector handles the external Provider refund request and Provider-specific response normalization.

The Payment Core handles internal refund validation and state updates.

### Refund Rules

- Full refunds are supported.
- Partial refunds are supported.
- Only successful refunds count toward `refundedAmount`.
- Total successful refunds must never exceed the original Payment amount.
- Failed or cancelled refunds do not reduce the remaining refundable amount.
- Refund operations must be protected against concurrent requests.
- External refund results must be verified before updating the financial state.

### Refund Calculations

`refundedAmount` represents the total amount of successful refunds.

```js
refundedAmount =
  refunds
    .filter((refund) => refund.status === "succeeded")
    .reduce((total, refund) => total + refund.amount, 0);
```

`remainingRefundableAmount` represents the amount that can still be refunded.

```js
remainingRefundableAmount = Math.max(
  0,
  amount - refundedAmount
);
```

These calculations do not replace backend validation.

### Payment Status After Refund

The Payment status is recalculated according to successful refund operations:

- No successful refunds: `paid`.
- Successful refunds totaling less than the original amount: `partially_refunded`.
- Successful refunds totaling the original amount: `refunded`.

A pending, processing, failed, or cancelled refund must not be counted as a successful refund.

## 8. Webhook Processing

Provider-specific webhook processing belongs to the Connector.

The Connector is responsible for:

1. Receiving the Provider webhook.
2. Parsing the Provider payload.
3. Verifying the Provider signature.
4. Validating Provider-specific data.
5. Normalizing the event.
6. Passing the normalized event to the Payment Core.

The Payment Core must never process raw Provider webhook payloads.

### Normalized Event Processing

The Payment Core receives an internal event containing the information required to process the financial operation.

For example:

```js
{
  eventId: String,
  paymentId: ObjectId,
  provider: String,
  status: String,
  amount: Number,
  currency: String,
  providerPaymentId: String,
  transactionId: String,
  failureCode: String,
  failureMessage: String,
}
```

The Core must:

1. Validate the normalized event.
2. Resolve the related Payment.
3. Validate Provider, amount, and currency.
4. Check durable event deduplication.
5. Validate the expected Payment state transition.
6. Apply the Payment update.
7. Synchronize the Order when required.

### Webhook Idempotency

Webhook event history is not stored inside Payment.

Durable event deduplication must be implemented outside the Payment schema.

Processing the same event more than once must not cause:

- Duplicate Payment updates.
- Duplicate refunds.
- Duplicate financial operations.
- Invalid status transitions.

Out-of-order events must be handled without allowing an older event to overwrite a newer valid financial state.

## 9. Late Successful Webhooks

A successful Provider payment event may arrive after the original checkout operation has failed.

For example:

```text
Create Order
     |
     v
Create Payment
     |
     v
Create Provider Session
     |
     v
Database Transaction Fails
     |
     v
Synchronous Compensation
     |
     v
Provider Reports Successful Payment
     |
     v
Late Webhook Received
```

The system must not ignore a verified successful payment merely because the original internal transaction failed.

### Required Handling

When a verified successful event arrives:

1. Identify the original payment and Order.
2. Verify the actual Provider payment result.
3. Determine whether the original Order still exists and can be fulfilled.
4. If fulfillment is valid, process the payment according to the normal lifecycle.
5. If the Order no longer exists or cannot be fulfilled, initiate a refund through the Connector.

The system must not create or fulfill an Order merely because an external payment succeeded.

### Durable Correlation Requirement

A rolled-back MongoDB transaction may remove both the original Order and Payment records.

Therefore, the Webhook workflow cannot rely exclusively on those records to identify a late successful payment.

A durable correlation and recovery mechanism is required to identify the original transaction and determine the appropriate compensation.

**Implementation detail pending:** The exact persistence mechanism and correlation design have not yet been approved.

No additional model, collection, or storage structure is defined by this document.

## 10. Checkout Failure and Compensation

Checkout initialization may involve an external Provider operation and internal database operations.

MongoDB rollback cannot reverse an external Provider operation.

The system therefore uses:

- Transactional Rollback for internal database operations.
- Synchronous External Compensation through the Connector.

### Compensation Flow

```text
Start Database Transaction
          |
          v
     Create Order
          |
          v
    Create Payment
          |
          v
 Create Provider Session
          |
          v
 Internal Operation Fails?
          |
      Yes |
          v
 Rollback Database Transaction
          |
          v
 Synchronous Connector Cancellation
          |
          v
 Cancellation Confirmed?
       /       \
     Yes        No / Unknown
      |              |
      v              v
   Resolved    Admin Reconciliation
```

The system does not use a background compensation queue.

If external cancellation fails or its result is uncertain, the system must preserve the case for manual reconciliation.

It must not assume that the Provider session was successfully cancelled.

## 11. Payment and Order Synchronization

Payment and Order maintain independent status fields.

Payment status changes may require corresponding Order business actions.

### When Payment Becomes Paid

- The Payment Module confirms the financial state.
- The Order Module applies the appropriate payment-related business transition.
- Order fulfillment continues according to Order lifecycle rules.

### When Payment Becomes Failed, Cancelled, or Expired

The Order Module applies the appropriate business rules.

A Payment status change must not directly overwrite unrelated Order lifecycle information.

### When Payment Becomes Refunded

The Order Module applies the appropriate business rules.

A refund does not automatically mean that the Order must be cancelled.

The business outcome depends on the Order's current state and the reason for the refund.

## 12. Financial Integrity

All financial updates must be performed through authorized backend services.

The following rules must always be enforced:

- Payment amount and currency must remain consistent with the original financial snapshot.
- Provider confirmation must be verified server-side.
- Refund totals must never exceed the original Payment amount.
- Duplicate webhook events must not produce duplicate financial operations.
- Invalid or outdated status transitions must be rejected.
- Concurrent refund requests must not exceed the remaining refundable amount.
- External operation failures must not be treated as successful operations.
- Uncertain external results must remain available for reconciliation.

Payment is a financial record and must not be treated as a normal editable resource.

## 13. Admin Reconciliation

Reconciliation is performed through a dedicated Admin Endpoint.

There is no automatic reconciliation queue.

The Admin Reconciliation workflow must:

1. Inspect the internal Payment and Order state.
2. Retrieve the external Provider state through the Connector when necessary.
3. Compare internal and external financial information.
4. Allow only authorized reconciliation actions.
5. Record the performed actions in an audit log.

Admins must not be able to arbitrarily mark Payments as paid or fabricate successful refund results.

All financial corrections must be based on verified information.

---

## Lifecycle Rules Summary

| Area | Rule |
|---|---|
| Payment Ownership | Payment Module owns Payment status |
| Order Ownership | Order Module owns Order status |
| Online Payment | Requires Provider checkout initialization |
| COD | No online checkout initialization |
| COD Collection | Remains pending until cash is collected |
| Payment Confirmation | Verified server-side Provider result |
| Checkout Storage | Embedded inside Payment |
| Checkout Attempts | No embedded attempts history |
| Transaction Data | Stored inside Payment |
| Webhook Verification | Connector responsibility |
| Webhook Normalization | Connector responsibility |
| Webhook Processing | Payment Core responsibility |
| Webhook Deduplication | Durable infrastructure outside Payment |
| Refund Storage | Embedded inside Payment |
| Refund Execution | Through Connector |
| Late Successful Webhook | Refund if original Order cannot be fulfilled |
| Checkout Compensation | Synchronous Connector cancellation |
| Compensation Queue | Not used |
| Reconciliation | Manual Admin Endpoint |
| Financial Integrity | Controlled backend services and audit logging |

## Pending Technical Design

The following requirement must be resolved before implementing late webhook recovery:

- Durable correlation for successful Provider events when the original Order/Payment transaction has rolled back.

This requirement does not change the approved Payment lifecycle or compensation decisions.