
# Project Documentation

Central documentation index for the E-Commerce project.

This documentation covers the core modules, business workflows, database architecture, and implementation decisions.

## Modules

### 1. Orders

Order creation, lifecycle management, and payment synchronization.

- [Orders Overview](./orders/README.md)
- [Order Create Flow](./orders/order-create-flow.md)
- [Order Payment Lifecycle](./orders/order-payment-lifecycle.md)
- [Order Decisions](./orders/order-decisions.md)

### 2. Payments

Payment processing, checkout integration, transaction management, and refunds.

- [Payments Overview](./payments/README.md)
- [Payment Create Flow](./payments/payment-create-flow.md)
- [Payment Lifecycle](./payments/payment-lifecycle.md)
- [Payment Decisions](./payments/payment-decisions.md)

### 3. Products

Product attributes configuration, variants, and inventory structure.

- [Product Attributes](./product/product-attributes.md)
- [Product Attributes Examples](./product/product-attributes-examples.md)

---

## Documentation Structure

```text
docs/
├── README.md
├── orders/
│   ├── order-create-flow.md
│   ├── order-decisions.md
│   ├── order-payment-lifecycle.md
│   └── README.md
├── payments/
│   ├── payment-create-flow.md
│   ├── payment-decisions.md
│   ├── payment-lifecycle.md
│   └── README.md
└── product/
    ├── product-attributes-examples.md
    └── product-attributes.md
```


