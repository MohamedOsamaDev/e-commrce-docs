
# Order Management

## Overview

This documentation defines the architecture and business rules for the Order Creation and Checkout lifecycle.

The goal is to provide a reliable order processing system that handles:

- Product and variant validation.
- Stock availability and reservation.
- Checkout session management.
- COD and online payments.
- Payment provider integrations.
- Order and payment state transitions.
- Concurrent checkout attempts.
- Payment failures and retries.

## Documentation

| Document | Description |
|---|---|
| [Order Create Flow](./order-create-flow.md) | Complete order creation process and validation pipeline. |
| [Order Payment Lifecycle](./order-payment-lifecycle.md) | Payment processing, stock reservation, expiration, and retry handling. |
| [Order Decisions](./order-decisions.md) | Approved business rules, technical requirements, and unresolved decisions. |

## Core Principles

1. The backend is the source of truth for all order data.
2. Frontend-submitted prices and totals must never be trusted.
3. Stock operations must be atomic.
4. Order and Payment have independent lifecycles.
5. Checkout sessions must have controlled expiration.
6. Payment success must be verified server-side.
7. All critical operations must be idempotent.
8. A cancelled order must never be reused for a new payment attempt.

## Supported Payment Methods

### Cash on Delivery (COD)

Stock reservation behavior is controlled by store settings.

### Online Payments

Payment providers are configurable and may include:

- Stripe
- Fawry
- Other supported providers

Reservation duration is configurable per payment provider.

## Important

This documentation represents the currently approved planning decisions.

Unresolved business rules are explicitly documented and must be finalized before implementation.
