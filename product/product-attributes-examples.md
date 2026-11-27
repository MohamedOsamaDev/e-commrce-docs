
# Product Attributes — Data Examples

This document demonstrates the expected `attributes` data structure for each product configuration.

All examples assume that the product configuration is stored separately under `attributes.config`.

---

## 1. Simple Product

**Configuration:**

```js
{
  hasColors: false,
  hasSizes: false
}
```

**Data:**

```js
{
  attributes: {
    stock: 150,
    images: [
      "media-id-1",
      "media-id-2"
    ]
  }
}
```

**Inventory:**

| Field | Value |
|---|---:|
| Total Stock | 150 |

Stock is managed directly at the product level.

---

## 2. Size Variants

**Configuration:**

```js
{
  hasColors: false,
  hasSizes: true
}
```

**Data:**

```js
{
  attributes: {
    images: [
      "media-id-1",
      "media-id-2"
    ],
    sizes: [
      {
        size: "size-s-id",
        stock: 20
      },
      {
        size: "size-m-id",
        stock: 35
      },
      {
        size: "size-l-id",
        stock: 40
      },
      {
        size: "size-xl-id",
        stock: 15
      }
    ]
  }
}
```

**Inventory:**

| Size | Stock |
|---|---:|
| S | 20 |
| M | 35 |
| L | 40 |
| XL | 15 |
| **Total** | **110** |

Images are shared across all sizes.

---

## 3. Color Variants

**Configuration:**

```js
{
  hasColors: true,
  hasSizes: false
}
```

**Data:**

```js
{
  attributes: {
    colors: [
      {
        color: "black-color-id",
        stock: 30,
        images: [
          "black-image-1",
          "black-image-2"
        ]
      },
      {
        color: "white-color-id",
        stock: 25,
        images: [
          "white-image-1",
          "white-image-2"
        ]
      },
      {
        color: "blue-color-id",
        stock: 15,
        images: [
          "blue-image-1",
          "blue-image-2"
        ]
      }
    ]
  }
}
```

**Inventory:**

| Color | Stock |
|---|---:|
| Black | 30 |
| White | 25 |
| Blue | 15 |
| **Total** | **70** |

Each color maintains its own inventory and images.

---

## 4. Color & Size Variants

**Configuration:**

```js
{
  hasColors: true,
  hasSizes: true
}
```

**Data:**

```js
{
  attributes: {
    colors: [
      {
        color: "black-color-id",
        images: [
          "black-image-1",
          "black-image-2"
        ],
        sizes: [
          {
            size: "size-s-id",
            stock: 10
          },
          {
            size: "size-m-id",
            stock: 20
          },
          {
            size: "size-l-id",
            stock: 15
          }
        ]
      },
      {
        color: "white-color-id",
        images: [
          "white-image-1",
          "white-image-2"
        ],
        sizes: [
          {
            size: "size-s-id",
            stock: 12
          },
          {
            size: "size-m-id",
            stock: 18
          },
          {
            size: "size-l-id",
            stock: 8
          }
        ]
      }
    ]
  }
}
```

**Inventory:**

| Color | Size | Stock |
|---|---|---:|
| Black | S | 10 |
| Black | M | 20 |
| Black | L | 15 |
| White | S | 12 |
| White | M | 18 |
| White | L | 8 |
| **Total** | | **83** |

Each color has its own size options, and every color-size combination has independent inventory.

Images are managed at the color level and shared across that color's sizes.

---

## Inventory Management Rules

| Product Type | Inventory Management |
|---|---|
| Simple Product | One stock value |
| Size Variants | One stock value per size |
| Color Variants | One stock value per color |
| Color & Size Variants | One stock value per color-size combination |

### Important Considerations

- Stock values must be non-negative numbers.
- Each variation maintains its own inventory.
- Total product inventory can be calculated by summing the stock values at the deepest variation level.
- Images should not be duplicated unnecessarily between sizes.
- Color and size relations should reference existing records in their respective collections.
- A variation's availability should be determined by its own stock value.

---

## Data Structure Summary

```text
Product
└── attributes
    ├── config
    │   ├── hasColors
    │   └── hasSizes
    │
    └── [Dynamic Structure]
        │
        ├── Simple Product
        │   ├── stock
        │   └── images[]
        │
        ├── Size Variants
        │   ├── images[]
        │   └── sizes[]
        │       ├── size
        │       └── stock
        │
        ├── Color Variants
        │   └── colors[]
        │       ├── color
        │       ├── stock
        │       └── images[]
        │
        └── Color & Size Variants
            └── colors[]
                ├── color
                ├── images[]
                └── sizes[]
                    ├── size
                    └── stock
```
