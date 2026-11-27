
# Order Create Flow — Decisions & Requirements

This document records approved business decisions and unresolved questions.

## 1. Approved Decisions

| Area | Decision |
|---|---|
| Product Source | Frontend Products Only |
| Product Pricing | Backend Database |
| Online Payment Failure | Cancel Order and Release Stock |
| Payment Retry | Create a Completely New Order |
| Reservation Duration | Configurable per Payment Provider |
| COD Reservation | Controlled by Store Settings |
| Active Checkout Protection | Check Pending Orders and Active Checkout Sessions |
| Payment Verification | Server-side Provider Verification |
| Stock Updates | Atomic Operations |

---

## 2. Product Request Contract

The frontend sends:

- Product ID.
- Selected Variant ID or attributes.
- Requested Quantity.
- Selected Payment Method.

The frontend must not determine:

- Product price.
- Discount amount.
- Shipping cost.
- Tax amount.
- Final order total.
- Available stock.

These values must be calculated or validated by the backend.

---

## 3. Active Checkout Rules

A user cannot start another checkout while having:

- An existing Pending Order.
- An existing Active Checkout Session.

Both conditions must be checked.

The system must prevent concurrent checkout creation, not just sequential requests.

---

## 4. Stock Management Requirements

### Required Behavior

- Support temporary stock reservations.
- Support configurable reservation expiration.
- Support COD-specific stock policies.
- Prevent overselling.
- Support automatic reservation expiration.
- Guarantee exactly-once stock release at the business-operation level.

### Technical Requirements

- Atomic stock updates.
- Conditional database updates.
- Idempotent reservation operations.
- Safe concurrency handling.
- Transactional consistency or reliable compensation.

---

## 5. Payment Requirements

- Support multiple payment providers.
- Keep Order and Payment states independent.
- Verify successful payments through trusted provider notifications.
- Handle failed, cancelled, expired, and uncertain payment states.
- Support payment reconciliation.
- Never reuse cancelled orders.

---

## 6. Technical Requirements

| Requirement | Purpose |
|---|---|
| Database Transactions | Maintain data consistency. |
| Atomic Updates | Prevent stock conflicts. |
| Idempotency | Prevent duplicate operations. |
| Payment Webhooks | Verify payment results. |
| Background Jobs | Process expired reservations. |
| Race Condition Protection | Protect concurrent requests. |
| Payment Reconciliation | Resolve uncertain transactions. |
| Order Snapshot | Preserve purchase-time data. |
| Rate Limiting | Prevent order creation spam. |

---

## 7. Open Questions — Pending Approval

These decisions have not been finalized.

### 7.1 Daily Ordering Limit

- Does the limit count all created orders?
- Do cancelled orders count?
- Should the limit be based on created orders or confirmed orders?

### 7.2 COD Confirmation

- Should COD orders be confirmed immediately?
- Should they require administrative approval?

### 7.3 Payment Timeout

What happens when the reservation expires while the provider payment result is still unknown?

The system must not release stock until the payment state is safely resolved.

### 7.4 Stock Reservation Configuration

Should stock reservation settings be configurable:

- Per payment method?
- Per payment provider?
- Both?

### 7.5 Shipping & Discounts

Which pricing components must be snapshotted at order creation?

Expected components:

- Product prices.
- Discounts.
- Taxes.
- Shipping costs.
- Final payable amount.

---

## 8. Implementation Rule

Do not implement unresolved business decisions using arbitrary defaults.

Finalize the relevant decision before implementing the affected business logic.

Approved decisions must remain consistent across:

- Backend services.
- Database models.
- Payment integrations.
- Stock management.
- Admin settings.
- Frontend checkout.
