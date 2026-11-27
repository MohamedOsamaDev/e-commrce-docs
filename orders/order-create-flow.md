# Order Create Flow

## 1. Entry Point

The process starts when an authenticated user clicks **Create Order**.

The frontend submits the selected products, variants, quantities, and payment method.

### Request Payload

```json
{
  "items": [
    {
      "productId": "product_id",
      "variantId": "variant_id",
      "quantity": 2
    }
  ],
  "paymentMethod": "stripe"
}
```

The frontend is responsible for sending product selections only.

The backend must independently retrieve and calculate all trusted product and pricing information.

---

## 2. Validation Pipeline

All required validations must pass before the order creation process proceeds.

### Step 1: Request Validation

- Validate request structure.
- Ensure the products array is not empty.
- Validate product IDs and variant IDs.
- Ensure quantities are positive integers.
- Reject duplicated or conflicting items.

### Step 2: User Validation

- Verify authentication.
- Verify that the user is eligible to place orders.
- Apply user-level rate limiting.

### Step 3: Active Order & Checkout Validation

Prevent multiple concurrent checkout attempts.

Check:

- Existing Pending Orders.
- Existing Active Checkout Sessions.

The user must not have another active checkout flow.

### Step 4: Product & Variant Validation

For every submitted item:

- Verify that the product exists.
- Verify that the product is active.
- Verify that the selected variant exists.
- Verify that the selected variant is valid and available.
- Validate selected attributes against the product configuration.
- Retrieve the current product price from the database.

Never trust frontend-submitted prices.

### Step 5: Stock Availability

- Verify requested quantities.
- Check the current `Product.stock`.
- Ensure sufficient stock for every item.
- Account for stock already deducted by previous reservations.
- Protect stock availability checks against concurrent requests.

Stock deduction must use atomic conditional updates to prevent overselling.

### Step 6: Ordering Limits

- Enforce daily order creation limits.
- Apply anti-spam protection.
- Prevent repeated order creation attempts.

### Step 7: Payment Validation

- Verify the selected payment method.
- Ensure the payment method is enabled.
- Verify the associated provider is active.
- Validate supported currency.
- Validate provider-specific payment requirements.

---

## 3. Order Creation

After successful validation:

1. Calculate product prices on the backend.
2. Calculate discounts.
3. Calculate taxes and shipping costs, when applicable.
4. Calculate the final order total.
5. Create the Order.
6. Create the Checkout Session.
7. Create the Payment Record.

### Initial States

```text
Order.status       = pending
Payment.status     = pending
Checkout.status    = active
```

The order must preserve a snapshot of product information and pricing at the time of purchase.

The snapshot should not depend on future changes to the original product.

---

## 4. Payment Branching

After creating the order, the flow branches according to the selected payment method.

### COD

1. Read COD stock reservation settings.
2. Apply the configured stock policy.
3. Process the order according to COD confirmation rules.
4. Keep payment pending until payment collection.

### Online Payment

1. Read provider configuration.
2. Determine reservation expiration duration.
3. Reserve stock by atomically deducting the requested quantities from `Product.stock`.
4. Create the provider payment session.
5. Save the provider reference.
6. Redirect the user to the provider.

---

## 5. Stock Reservation Rules

The system uses a single stock field on each product or product variant.

**Do not introduce `availableStock` or `reservedStock`.**

### Stock Structure

```js
{
  stock: 10
}
```

### Reservation Mechanism

Stock reservation works by temporarily deducting the requested quantity directly from `Product.stock`.

Example:

Initial stock:

```js
Product.stock = 10;
```

After reserving 3 units:

```js
Product.stock = 7;
```

The reserved quantity is tracked through the associated Order items:

```js
Order.items = [
  {
    productId: "product_id",
    variantId: "variant_id",
    quantity: 3
  }
];
```

The system must also maintain an explicit reservation state to distinguish between reserved, committed, and released stock.

### Reservation States

| State | Description |
|---|---|
| `reserved` | Stock deducted temporarily and awaiting payment or confirmation. |
| `committed` | Stock deduction finalized. |
| `released` | Previously reserved stock restored. |

### Reservation Lifecycle

**Successful Payment:**

- Order becomes `confirmed`.
- Stock remains deducted.
- Reservation becomes `committed`.
- No additional stock deduction is performed.

**Failed Payment / Cancellation / Expiration:**

- Order becomes `cancelled`.
- Restore the successfully reserved quantity to `Product.stock`.
- Reservation becomes `released`.

Example:

```js
Product.stock += orderItem.quantity;
```

### Reservation Requirements

- Every reservation must have an expiration timestamp (`expiresAt`).
- Stock deduction must be atomic.
- Stock must never become negative.
- A reservation must never be released more than once.
- Only successfully deducted quantities can be restored.
- A committed reservation must never be released.
- Expired reservations must be released automatically when payment is confirmed unpaid.
- Cancelled checkouts must release their reservations.
- Successful payments must commit reserved stock without additional deduction.
- Reservation operations must be idempotent.

### COD

COD reservation behavior is controlled by store settings.

If reservation is enabled, deduct stock during order creation and restore it if the order is cancelled.

If reservation is disabled, stock must be atomically deducted at the configured confirmation stage.

Both configurations must prevent overselling.

---

## 6. Validation Failure Handling

If any validation fails:

- Do not create an order.
- Do not create a payment record.
- Do not reserve stock.
- Return a meaningful error response.

If an internal failure occurs after stock reservation or partial order creation, the system must safely roll back or compensate for the completed operations.

Stock restoration must only apply to quantities that were successfully deducted.

---

## 7. Concurrency Protection

The system must prevent:

- Multiple checkouts for the same user.
- Overselling the same product variant.
- Duplicate order creation.
- Duplicate stock reservation.
- Duplicate stock release.
- Conflicting payment state transitions.

Use database transactions where supported, atomic conditional updates, and idempotency mechanisms.

Stock reservation and restoration must be protected against concurrent operations.

Order item quantities alone must never be treated as proof that stock was reserved. The reservation state must be verified before committing or releasing stock.