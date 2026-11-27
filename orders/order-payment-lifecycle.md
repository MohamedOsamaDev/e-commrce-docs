
# Order & Payment Lifecycle

## 1. Independent Lifecycles

Order, Payment, and Checkout Session are separate entities with independent responsibilities.

### Order Statuses

| Status | Description |
|---|---|
| `pending` | Order created and awaiting completion. |
| `confirmed` | Order confirmed. |
| `processing` | Order preparation started. |
| `shipped` | Order shipped. |
| `delivered` | Order delivered. |
| `cancelled` | Order cancelled. |
| `failed` | Internal order processing failure. |

### Payment Statuses

| Status | Description |
|---|---|
| `pending` | Awaiting payment. |
| `processing` | Payment processing. |
| `paid` | Payment successfully completed. |
| `failed` | Payment failed. |
| `cancelled` | Payment cancelled. |
| `expired` | Payment session expired. |
| `refunded` | Payment refunded. |

### Checkout Session Statuses

Proposed statuses:

| Status | Description |
|---|---|
| `active` | Checkout is active. |
| `completed` | Checkout successfully completed. |
| `cancelled` | Checkout cancelled. |
| `expired` | Checkout expired. |

---

## 2. Online Payment Flow

```text
Create Order
    |
Create Checkout Session
    |
Reserve Stock
    |
Create Provider Payment Session
    |
Redirect User
    |
Provider Processes Payment
    |
Verify Payment Result
    |
    +----------------------+
    |                      |
  Success                Failure
    |                      |
Payment: paid          Payment: failed
    |                      |
Order: confirmed       Order: cancelled
    |                      |
Commit Stock           Release Stock
    |                      |
Checkout: completed    Checkout: cancelled
```

The payment provider's verified server-side result is the source of truth.

Frontend redirects must never independently determine payment success or failure.

---

## 3. Successful Payment

When the provider confirms successful payment:

1. Verify the payment notification.
2. Verify the provider transaction reference.
3. Verify the expected amount and currency.
4. Ensure the notification has not already been processed.
5. Update Payment to `paid`.
6. Update Order to `confirmed`.
7. Commit reserved stock.
8. Complete the Checkout Session.

All state changes must be safely coordinated.

---

## 4. Failed Payment

When payment failure is confirmed:

1. Update Payment to `failed`.
2. Update Order to `cancelled`.
3. Release reserved stock.
4. Cancel the Checkout Session.

Stock release must be idempotent.

A failed payment must not leave an active checkout session.

---

## 5. Payment Cancellation

When the user cancels the payment flow:

- Do not assume that the payment has failed based solely on frontend navigation.
- Verify the actual provider payment state.
- If unpaid and cancellation is confirmed, cancel the order.
- Release the reservation.
- Close the checkout session.

---

## 6. Checkout Expiration

Every checkout session has an expiration timestamp.

When the session expires:

1. Identify the expired checkout.
2. Verify the actual payment state.
3. If payment is confirmed, complete the successful payment flow.
4. If payment is confirmed unpaid, cancel the order.
5. Release reserved stock when safe.
6. Close the checkout session.

A background job must handle expired reservations.

---

## 7. Payment Timeout & Uncertain Results

A provider timeout does not necessarily mean payment failure.

The system must distinguish between:

- Confirmed payment success.
- Confirmed payment failure.
- Pending provider processing.
- Unknown payment result.

For unknown results:

- Do not immediately mark payment as failed.
- Do not blindly release stock.
- Reconcile the payment with the provider.
- Resolve the final state before applying stock changes.

This is especially important when the provider may have received payment but the application did not receive its response.

---

## 8. Payment Retry Policy

A new payment attempt must create a new Order.

The system must never reuse a cancelled order.

### Retry Flow

```text
Previous Payment Failed
        |
Previous Order Cancelled
        |
Release Previous Reservation
        |
Close Previous Checkout
        |
User Starts New Checkout
        |
Run Full Validation Again
        |
Create New Order
        |
Create New Checkout Session
        |
Reserve Stock Again
        |
Start New Payment Attempt
```

### Retry Rules

- Old Order remains `cancelled`.
- Old Payment remains `failed`.
- Old Checkout Session remains `cancelled`.
- A new Order is created.
- A new Payment Record is created.
- Stock availability is checked again.
- Stock is reserved again according to current availability.

---

## 9. Webhook Requirements

Payment webhooks must:

- Verify provider signatures.
- Validate transaction references.
- Validate payment amount and currency.
- Handle duplicate notifications.
- Handle out-of-order notifications safely.
- Reject invalid state transitions.
- Process events idempotently.

Never trust frontend payment callbacks as final payment confirmation.
