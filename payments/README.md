
# Payment Module

## Overview

The Payment Module manages financial transactions associated with customer orders.

It is responsible for:
- Payment creation and lifecycle management.
- Payment provider integration.
- Checkout session management.
- Transaction tracking.
- Full and partial refunds.
- Payment status synchronization.
- Financial data integrity.

## Architecture

Payment is stored in a separate MongoDB collection and linked to Order through an ObjectId reference.

### Relationship

- One Order has one Payment.
- One Payment belongs to one Order.
- Payment references Order and User.
- Refunds are embedded inside Payment.

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
  |
  v
Payment Provider
```

## Payment Responsibilities

### Payment Information
- method
- provider
- amount
- currency
- status

### Checkout Information
- providerSessionId
- checkoutUrl
- expiresAt

### Transaction Information
- providerPaymentId
- transactionId

### Refund Information
- refunds[]
- paidAt
- refundedAt

## Payment Methods

- COD (Cash on Delivery)
- Online Payment

## Core Principles

1. Payment is an independent financial record.
2. Each Order has one Payment record.
3. Financial snapshots must remain immutable.
4. Payment updates must go through controlled backend services.
5. Payment providers must remain interchangeable.
6. Refunds must be tracked independently.
7. Webhook processing must be idempotent.

## Related Documentation

- [Payment Create Flow](./payment-create-flow.md)
- [Payment Lifecycle](./payment-lifecycle.md)
- [Payment Decisions](./payment-decisions.md)
