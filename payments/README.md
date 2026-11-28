# Payment Module

## 1. Overview

The Payment Module manages financial transactions associated with customer Orders.

It is responsible for:

- Payment creation and lifecycle management.
- Payment status management.
- Checkout initialization coordination.
- Transaction information management.
- Full and partial refund management.
- Webhook event processing.
- Financial data integrity.
- Payment and Order status synchronization.

The Payment Module is designed to remain independent of specific payment Providers.

External Provider integration is handled through dedicated Connectors.

## 2. Architecture

Payment is stored in a separate MongoDB collection and linked to Order through an ObjectId reference.

### Entity Relationship

```text
Order
  |
  | payment: ObjectId
  |
  v
Payment
  |
  |-- Payment Information
  |-- Checkout Information
  |-- Transaction Information
  |-- Refunds[]
```

### Relationship Rules

- One Order has exactly one Payment.
- One Payment belongs to one Order.
- Payment references its Order and User.
- Refunds are embedded inside Payment.
- Checkout information is embedded inside Payment.
- Payment and Order maintain independent lifecycles.

A unique index on the Payment `order` field enforces the one-to-one relationship.

## 3. Payment Components

The Payment architecture separates internal financial operations from external Provider integration.

```text
                  Payment Module
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
     Payment Core   Checkout Core  Webhook Core
          |             |             |
          +-------------+-------------+
                        |
                        v
                    Connector
                        |
             +----------+----------+
             |          |          |
             v          v          v
           Paymob     Stripe     Other Providers
```

### Payment Core

Responsible for internal Payment operations:

- Creating Payment records.
- Retrieving active Online Payments.
- Managing Payment status.
- Validating financial information.
- Processing normalized payment events.
- Managing internal Payment state transitions.

### Checkout Core

Responsible for coordinating checkout initialization and retrieving current checkout information.

It works with the corresponding Connector to create external checkout sessions.

Checkout information is stored inside the Payment document.

### Webhook Core

Responsible for processing normalized Provider events.

It handles:

- Payment identification.
- Event validation.
- Payment amount and currency validation.
- Event deduplication.
- Payment status transitions.
- Required Order synchronization.

It does not process raw Provider webhook payloads.

### Connector

The Connector owns the complete external integration with its Provider.

It is responsible for:

- Provider API requests.
- Authentication.
- Request and response transformation.
- Checkout session creation and cancellation.
- Refund requests.
- Webhook reception and parsing.
- Signature verification.
- Event normalization.
- Provider-specific error and timeout handling.

The Connector acts as the bridge between an external Provider and the internal Payment Core.

The Payment Core must not contain Provider-specific implementation logic.

## 4. Payment Responsibilities

### Payment Information

| Field | Description |
|---|---|
| `order` | Related Order reference |
| `user` | Payment owner |
| `method` | Payment method |
| `provider` | Selected Provider |
| `amount` | Original payment amount |
| `currency` | Payment currency |
| `status` | Current financial status |

### Checkout Information

| Field | Description |
|---|---|
| `providerSessionId` | External Provider session identifier |
| `checkoutUrl` | Customer checkout URL |
| `expiresAt` | Checkout expiration date |

Checkout information is stored directly inside Payment.

There is no separate persisted Checkout Session entity.

### Transaction Information

| Field | Description |
|---|---|
| `providerPaymentId` | External Provider payment identifier |
| `transactionId` | Transaction reference |

### Refund Information

Refund operations are embedded inside Payment through `refunds[]`.

Payment also maintains relevant financial timestamps:

- `paidAt`
- `refundedAt`

## 5. Payment Methods

The system supports two payment methods.

### COD

Cash on Delivery:

- Does not require an external Provider.
- Does not initialize an online checkout session.
- Remains pending until cash is collected.

### Online Payment

Online Payment:

- Requires a configured Provider.
- Requires checkout initialization through the corresponding Connector.
- Uses verified server-side Provider results to confirm payment.
- Is subject to the active Online Payment restriction at the user level, regardless of Provider.

## 6. Payment Lifecycle

Payment has its own lifecycle, independent of Order.

Supported Payment statuses:

| Status | Description |
|---|---|
| `pending` | Payment created but not completed |
| `processing` | Payment operation is being processed |
| `paid` | Payment successfully completed |
| `failed` | Payment operation failed |
| `cancelled` | Payment cancelled |
| `expired` | Checkout session expired |
| `partially_refunded` | Part of the payment refunded |
| `refunded` | Fully refunded |

Payment status is owned by the Payment Module.

Order status represents the purchase and fulfillment lifecycle and must not duplicate Payment status.

See [Payment Lifecycle](./payment-lifecycle.md) for status transitions and detailed workflows.

## 7. Financial Integrity

The Payment Module follows these principles:

1. Payment is an independent financial record.
2. Each Order has exactly one Payment.
3. Amount and currency preserve the original financial snapshot.
4. Financial updates must go through controlled backend services.
5. Provider integration must remain isolated inside Connectors.
6. Webhook processing must be idempotent.
7. Refund totals must never exceed the original Payment amount.
8. External financial results must be verified before updating internal state.
9. Concurrent financial operations must be protected against inconsistent updates.

## 8. Checkout Failure and Compensation

Checkout initialization may involve both MongoDB operations and external Provider operations.

MongoDB transactions cannot roll back external Provider operations.

The approved failure strategy is:

- Transactional rollback for internal database operations.
- Synchronous external compensation through the Connector.
- No background compensation queue.

If external cancellation fails or its result is uncertain, the case must remain available for Admin Reconciliation.

A verified successful payment received after the original checkout transaction has rolled back must be handled through the late webhook recovery workflow.

## 9. Webhook Processing

Provider-specific webhook integration belongs to the Connector.

The Connector:

- Receives Provider webhooks.
- Parses and verifies Provider payloads.
- Verifies signatures.
- Normalizes Provider events.

The Webhook Core:

- Receives normalized events.
- Validates Payment information.
- Applies event deduplication.
- Validates status transitions.
- Updates Payment.
- Coordinates required Order synchronization.

Webhook event history is not embedded inside Payment.

Durable event deduplication must be implemented outside the Payment document.

## 10. Refund Management

Refunds belong to their original Payment.

The system supports:

- Full refunds.
- Partial refunds.
- Multiple refund operations, subject to the remaining refundable amount.

The Payment Core manages internal refund validation and state updates.

The corresponding Connector executes external Provider refund requests.

Only successful refunds contribute to the refunded amount.

See [Payment Lifecycle](./payment-lifecycle.md) for refund statuses and calculations.

## 11. Reconciliation

Reconciliation is performed through a dedicated Admin Endpoint.

There is no automatic reconciliation queue.

The reconciliation workflow uses the Connector to inspect external Provider information when necessary.

Administrative reconciliation must:

- Require authorization.
- Preserve financial integrity.
- Verify external financial information.
- Record actions in an audit log.

Admins must not arbitrarily mark Payments as paid or fabricate successful refund results.

## 12. Related Documentation

| Document | Responsibility |
|---|---|
| [Payment Create Flow](./payment-create-flow.md) | Payment creation and checkout initialization workflow |
| [Payment Lifecycle](./payment-lifecycle.md) | Payment statuses, transactions, refunds, webhooks, and compensation |
| [Payment Decisions](./payment-decisions.md) | Approved Payment architecture and business decisions |