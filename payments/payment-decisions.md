
# Payment Architecture Decisions

## 1. Separate Payment Collection

**Decision:** Payment uses an independent MongoDB collection.

**Reason:**
- Payment has its own lifecycle.
- Financial records require controlled updates.
- Provider integration should remain independent of Order.
- Refund management should not increase Order schema complexity.

Order stores only:

```js
payment: {
  type: ObjectId,
  ref: "payment",
  default: null,
}
```

## 2. One Payment Per Order

**Decision:** Each Order has exactly one Payment record.

Enforced through a unique index on:

```js
{ order: 1 }
```

This prevents multiple Payment records for the same Order.

## 3. No Payment Attempts Array

**Decision:** Payment does not maintain an embedded attempts array.

**Reason:**
- Keep the schema simple.
- Store only the current checkout and transaction information.
- Avoid unnecessary embedded history.

A new checkout attempt updates the current checkout information.

Previous attempts are not retained in the current schema.

## 4. Financial Snapshot

**Decision:** Payment stores its own amount and currency.

Fields:

```js
amount: Number,
currency: String,
```

Values are copied from the Order during Payment creation.

**Reason:**
- Preserve the original financial snapshot.
- Maintain payment reconciliation.
- Provide the exact amount and currency required by payment providers.
- Avoid depending on mutable Order data.

These fields must not be directly editable after creation.

## 5. Provider-Agnostic Design

**Decision:** Payment schema must not depend on a specific provider.

The provider is identified through:

```js
provider: String
```

Provider-specific integration logic belongs in the service layer.

## 6. Embedded Refunds

**Decision:** Refund operations are embedded inside Payment.

**Reason:**
- Refunds belong to a specific Payment.
- Refund history is naturally associated with its parent record.
- Full and partial refunds are supported.

Refund amounts must be validated by backend services.

## 7. No Webhook Events in Payment Schema

**Decision:** Webhook event history is not embedded inside Payment.

Webhook idempotency and event deduplication are handled separately through durable backend infrastructure.

## 8. Controlled Financial Updates

**Decision:** Payment is treated as a financial record, not a normal editable admin resource.

### Immutable Fields

- order
- user
- method
- provider
- amount
- currency

### Controlled Fields

- status
- checkout information
- transaction information
- refunds
- paidAt
- refundedAt

All updates must go through authorized backend services.

## 9. Admin Permissions

Admins can:
- View Payment information.
- View transaction information.
- View refund history.
- Request authorized refunds through dedicated actions.

Admins cannot:
- Directly change payment amounts.
- Change payment currency.
- Manually mark arbitrary payments as paid.
- Modify provider transaction references.
- Directly edit refund results.

All financial operations should be recorded in an audit log.

## 10. Payment Status Ownership

Payment status is owned by the Payment Module.

Order must not maintain duplicated payment status or payment method fields.

Payment and Order status synchronization must happen through controlled services.

## 11. Refund Calculation

Refunded amount is calculated from successful refund operations only.

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

---

## Final Architecture

- Separate Payment Collection.
- One Payment per Order.
- No Attempts History.
- Financial Snapshot.
- Provider-Agnostic Integration.
- Embedded Refunds.
- Independent Payment Lifecycle.
- Controlled Financial Updates.
- Durable Webhook Idempotency.
- Audit Logging.
