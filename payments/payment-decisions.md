# Payment Architecture Decisions

This document defines the approved architectural and business decisions for the Payment Module.

These decisions are authoritative. Older conflicting decisions must not be used.

## 1. Payment and Order Separation

**Decision:** Payment is stored in an independent MongoDB collection.

Order and Payment represent different business responsibilities and maintain independent lifecycles.

- Order represents the customer's purchase and fulfillment.
- Payment represents the financial operation associated with that Order.
- Payment owns its financial status and transaction information.
- Order must not duplicate Payment status or payment method.

Each Order has exactly one Payment.

The relationship is enforced through a unique index:

```js
{ order: 1 }
```

Payment references its Order and User.

## 2. Financial Snapshot

**Decision:** Payment stores its own amount and currency.

```js
{
  amount: Number,
  currency: String,
}
```

These values are copied from the Order during Payment creation.

They represent the financial snapshot of the original purchase and must not be modified after creation.

The following fields are immutable:

- `order`
- `user`
- `method`
- `provider`
- `amount`
- `currency`

All financial updates must go through controlled backend services.

## 3. Payment Methods

The system supports two payment methods:

- COD (Cash on Delivery)
- Online Payment

COD and Online Payment share the Payment model but have different initialization and collection workflows.

COD does not initialize an online checkout session.

COD Payment remains unpaid until cash is actually collected.

## 4. Active Online Payment Restriction

**Decision:** A user can have only one active Online Payment at a time, regardless of Provider.

For example:

An active Paymob Payment prevents the same user from creating another Online Payment through Stripe.

The restriction applies at the user level, not at the Provider level.

The check must be concurrency-safe.

A sequential lookup followed by Payment creation is insufficient because concurrent checkout requests could both pass the check.

The implementation must guarantee that concurrent requests cannot create multiple active Online Payments for the same user.

COD Payments are not subject to this restriction.

## 5. COD Daily Limit

**Decision:** COD Daily Limit is checked before creating the Order.

The limit is evaluated using:

- The current user.
- The store's local calendar day.
- Non-cancelled Orders only.

The store timezone is obtained from:

`general.timezone`

The daily limit must be enforced through an atomic counter.

The implementation must guarantee exactly-once increment and decrement behavior.

### Counter Rules

- Increment the counter when a qualifying COD Order is created.
- Cancelled Orders must not count toward the daily limit.
- Cancelling an Order must release its original day's counter.
- If an Order is cancelled after midnight, its counter must still be decremented for the original Order day.
- Duplicate cancellation or retry operations must not decrement the counter more than once.

The counter must remain consistent with the qualifying Orders.

## 6. Checkout Information

**Decision:** Checkout information is stored inside the Payment document.

There is no separate persisted Checkout Session collection or entity.

The current checkout information includes:

```js
{
  providerSessionId: String,
  checkoutUrl: String,
  expiresAt: Date,
}
```

Checkout initialization is performed only for Online Payments.

A new checkout initialization updates the current checkout information.

The Payment schema does not maintain an embedded attempts array.

Previous checkout attempts are not retained in the current Payment document.

## 7. Provider-Agnostic Payment Core

**Decision:** The Payment Core must remain independent of Provider-specific implementation details.

The Payment record identifies its Provider through:

```js
provider: String
```

The Core handles internal Payment operations and must not contain:

- Provider-specific API endpoints.
- Provider authentication mechanisms.
- Provider request or response formats.
- Provider-specific checkout implementations.
- Raw Provider webhook parsing.
- Provider signature verification.
- Provider-specific error handling.

All external Provider integration belongs to the Connector.

## 8. Connector Responsibilities

**Decision:** The Connector owns the complete external integration with its Provider.

The Connector is the integration bridge between an external payment Provider and the internal Payment system.

It is responsible for:

- Provider API requests.
- Provider authentication.
- Request and response transformation.
- Checkout session creation.
- Checkout session cancellation.
- Refund requests.
- Webhook reception and parsing.
- Webhook signature verification.
- Event normalization.
- Provider-specific error and timeout handling.

Each Provider has its own integration implementation.

The Connector converts Provider-specific data into the internal contract expected by the Payment Core.

The Payment Core must not need to understand Provider-specific behavior.

## 9. Checkout Failure and External Compensation

**Decision:** Checkout failure uses:

- Transactional Rollback.
- Synchronous External Compensation.

Order creation, Payment creation, and their database changes must be transactionally consistent.

However, MongoDB transactions cannot roll back operations already completed by an external Provider.

If a Provider checkout session has been created and a subsequent operation causes the database transaction to fail, the corresponding Connector must synchronously attempt to cancel the external session.

The operation order must ensure that external resources are compensated when the internal transaction cannot be committed.

### Compensation Failure

External cancellation is not guaranteed to succeed.

If cancellation fails or its result is uncertain:

- The system must not assume that the external session was cancelled.
- The system must not assume that the external Payment is safely unpaid.
- The case must remain available for Admin Reconciliation.

Compensation errors must not be silently ignored.

## 10. Compensation Without a Queue

**Decision:** Compensation does not use a background queue.

The approved approach relies on:

- Correct operation ordering.
- Synchronous external cancellation through the Connector.
- Refund processing inside the verified Webhook workflow when required.

A queue-based compensation mechanism is not part of the approved architecture.

Cases that cannot be resolved synchronously must be handled through Admin Reconciliation.

## 11. Webhook Ownership and Normalization

**Decision:** Provider-specific webhook integration belongs to the Connector.

The Connector is responsible for:

1. Receiving the Provider webhook.
2. Parsing the Provider payload.
3. Verifying the Provider signature.
4. Validating Provider-specific data.
5. Normalizing the event.
6. Passing the normalized event to the Payment Core.

The Payment Core must not parse raw Provider webhook payloads or verify Provider-specific signatures.

The internal event contract must contain the information required for Payment processing, such as:

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

This represents the internal normalized contract, not a Provider webhook payload.

## 12. Webhook Idempotency

**Decision:** Webhook event history is not embedded inside the Payment schema.

Webhook idempotency and event deduplication must be handled through durable backend infrastructure outside the Payment document.

The implementation must prevent:

- Duplicate event processing.
- Duplicate financial operations.
- Invalid Payment status transitions caused by repeated events.
- Incorrect results caused by out-of-order event delivery.

## 13. Late Successful Webhooks

**Decision:** A verified successful Payment event must not be ignored merely because the original checkout operation failed.

A Provider may report successful payment after the internal Order and Payment creation transaction has rolled back.

If the external Payment succeeded but the corresponding Order can no longer be fulfilled, the system must initiate a refund through the Connector as part of the Webhook workflow.

The system must not proceed with Order fulfillment when the original Order is no longer valid.

### Durable Correlation Requirement

The system must be able to identify the original Order and Payment when a late webhook arrives, even if the original database transaction rolled back and the Payment document no longer exists.

The Payment document cannot be the only source of correlation because a rolled-back document is not available for lookup.

A durable correlation and recovery mechanism is therefore required.

**Implementation decision pending:** The persistence mechanism and exact correlation design have not yet been approved.

No additional collection, model, or storage structure is implicitly approved by this requirement.

## 14. Refund Management

**Decision:** Refund information belongs to the Payment record.

Refunds are embedded inside Payment because they are financial operations associated with a specific Payment.

The system supports:

- Full refunds.
- Partial refunds.

Refund amounts must be validated by backend services.

Only successful refund operations contribute to the refunded amount.

```js
refundedAmount =
  refunds
    .filter((refund) => refund.status === "succeeded")
    .reduce((total, refund) => total + refund.amount, 0);
```

Remaining refundable amount:

```js
remainingRefundableAmount = Math.max(
  0,
  amount - refundedAmount
);
```

These calculations do not replace backend validation.

All external refund requests must go through the corresponding Connector.

## 15. Manual Admin Reconciliation

**Decision:** Reconciliation is performed through a dedicated Admin Endpoint.

There is no automatic reconciliation queue.

The reconciliation workflow must:

- Inspect the internal Order and Payment state.
- Retrieve external Payment information through the Connector when necessary.
- Allow only authorized administrative actions.
- Record reconciliation actions in an audit log.
- Preserve financial integrity.

Admins must not be able to arbitrarily mark Payments as paid or fabricate Provider transaction results.

Financial state must be reconciled against verified Provider information.

## 16. Controlled Financial Updates

**Decision:** Payment is a financial record, not a normal editable admin resource.

### Immutable Fields

- `order`
- `user`
- `method`
- `provider`
- `amount`
- `currency`

### Controlled Fields

- `status`
- Checkout information.
- Transaction information.
- Refund information.
- `paidAt`
- `refundedAt`

All updates must go through authorized backend services.

Direct arbitrary updates to financial fields are not allowed.

## 17. Admin Permissions and Audit

Admins can:

- View Payment information.
- View transaction information.
- View refund history.
- Perform authorized reconciliation actions.
- Request authorized refund operations through dedicated actions.

Admins cannot:

- Directly change Payment amounts.
- Change Payment currency.
- Arbitrarily mark Payments as paid.
- Modify Provider transaction references.
- Directly edit refund results.

Financial and reconciliation operations must be recorded in an audit log.

---

## Final Approved Decisions

| Area | Decision |
|---|---|
| Payment Storage | Separate MongoDB collection |
| Order Relationship | One Payment per Order |
| Payment Lifecycle | Independent from Order lifecycle |
| Financial Snapshot | Immutable amount and currency |
| Payment Methods | COD and Online Payment |
| Active Online Payment | One per user, regardless of Provider |
| COD Daily Limit | Before Order creation, non-cancelled Orders only, per user/store-local day, atomic counter |
| Checkout Information | Embedded inside Payment |
| Payment Attempts | No embedded attempts array |
| Payment Core | Provider-agnostic |
| Connector | Owns complete external Provider integration |
| Checkout Failure | Transactional Rollback + synchronous external compensation |
| Compensation | No queue |
| Webhook | Connector verifies and normalizes Provider events |
| Webhook Idempotency | Durable deduplication outside Payment |
| Late Successful Webhook | Refund if the original Order cannot be fulfilled |
| Refunds | Embedded inside Payment |
| Reconciliation | Manual Admin Endpoint |
| Financial Updates | Controlled backend services |
| Audit | Required for financial and reconciliation operations |

## Pending Technical Design

The following technical requirement still needs a concrete design before implementation:

- Durable correlation for late webhooks when the original Order/Payment transaction has rolled back.

This pending design does not change or invalidate the approved business decisions above.