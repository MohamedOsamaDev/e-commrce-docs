
# Payment Create Flow

## Overview

Payment creation takes place during Order creation.

The Payment record receives its initial financial information from the Order.

## Creation Flow

```text
Customer Checkout
       |
       v
Validate Order Data
       |
       v
Calculate Order Prices
       |
       v
Create Order
       |
       v
Create Payment
       |
       v
Link Payment to Order
       |
       v
Continue Payment Flow
```

## Payment Initialization

The Payment record must receive:

| Field | Source |
|---|---|
| order | Created Order._id |
| user | Order.user |
| method | Selected payment method |
| provider | Selected payment provider |
| amount | Order.overviewPrices.finalTotal |
| currency | Order.currency |
| status | pending |

## COD Flow

For Cash on Delivery:

1. Create Order.
2. Create Payment with method `cod`.
3. Set initial Payment status to `pending`.
4. Link Payment to Order.
5. Continue Order processing.
6. Mark Payment as `paid` when payment is collected.

## Online Payment Flow

For Online Payment:

1. Create Order.
2. Create Payment with method `online`.
3. Set the selected provider.
4. Initialize the Payment record.
5. Create a checkout session through the provider.
6. Store the provider session information.
7. Return the checkout URL to the customer.
8. Wait for verified payment confirmation.

## Order-Payment Relationship

After Payment creation:

```js
order.payment = payment._id;
```

The Order stores only the Payment reference.

Payment details remain inside the Payment collection.

## Data Integrity

Order creation and Payment creation must be handled consistently.

Use a MongoDB transaction when creating both records and linking them.

If either operation fails, the transaction must roll back.

## Important Rules

- Never accept payment amount directly from the client.
- Always derive amount from the calculated Order total.
- Never allow currency mismatch between Order and Payment.
- Never create multiple Payment records for the same Order.
- Do not expose provider secrets to the client.
