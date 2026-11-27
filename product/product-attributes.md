
# Product Attributes System

## Overview

The Product Attributes System provides a flexible structure for managing product variations and inventory.

A product can have two optional variation dimensions:

- **Colors:** Determines whether the product has multiple color variants.
- **Sizes:** Determines whether the product has multiple size variants.

These options are controlled by the product's `attributes.config` object:

```js
{
  hasColors: false,
  hasSizes: false
}
```

The combination of these two boolean properties determines the structure of `attributes` and the schema used to validate and manage its data.

There are four possible configurations.

---

## 1. Simple Product

**Configuration:**

```js
{
  hasColors: false,
  hasSizes: false
}
```

**Attributes structure:**

```js
{
  stock: Number,
  images: [Media]
}
```

**Description:**

A product without color or size variations.

- Stock is managed directly at the product level.
- Images are managed directly at the product level.
- A single stock value represents the entire product inventory.

---

## 2. Size Variants

**Configuration:**

```js
{
  hasColors: false,
  hasSizes: true
}
```

**Attributes structure:**

```js
{
  images: [Media],
  sizes: [
    {
      size: SizeReference,
      stock: Number
    }
  ]
}
```

**Description:**

A product available in multiple sizes without color variations.

- Each size references a record from the Size collection.
- Every size has its own independent stock value.
- Product images are managed at the root of `attributes`, shared across all sizes.
- Stock is managed separately for each size.

**Example:**

A T-shirt available in S, M, L, and XL, with independent inventory for each size.

---

## 3. Color Variants

**Configuration:**

```js
{
  hasColors: true,
  hasSizes: false
}
```

**Attributes structure:**

```js
{
  colors: [
    {
      color: ColorReference,
      stock: Number,
      images: [Media]
    }
  ]
}
```

**Description:**

A product available in multiple colors without size variations.

- Each color references a record from the Color collection.
- Every color has its own independent stock value.
- Each color has its own images.
- Stock and images are managed independently for each color.

**Example:**

A watch available in black, silver, and gold, with separate inventory and images for each color.

---

## 4. Color & Size Variants

**Configuration:**

```js
{
  hasColors: true,
  hasSizes: true
}
```

**Attributes structure:**

```js
{
  colors: [
    {
      color: ColorReference,
      images: [Media],
      sizes: [
        {
          size: SizeReference,
          stock: Number
        }
      ]
    }
  ]
}
```

**Description:**

A product available in multiple colors, with different sizes for each color.

- Each color references a record from the Color collection.
- Each color has its own images.
- Every color contains its own available sizes.
- Each size references a record from the Size collection.
- Stock is managed independently for every color-size combination.

**Example:**

A pair of shoes available in black and white, with sizes 38–44 for each color.

Each combination has its own stock value.

---

## Configuration Reference

| Configuration | Colors | Sizes | Stock Location | Images Location |
|---|---|---|---|---|
| Simple Product | No | No | Root | Root |
| Size Variants | No | Yes | Each size | Root |
| Color Variants | Yes | No | Each color | Each color |
| Color & Size Variants | Yes | Yes | Each size within its color | Each color |

---

## Schema Resolution

The `getProductAttributesConfig` function determines the appropriate schema using the two configuration properties.

```js
const getProductAttributesConfig = (config) => {
  const { hasColors = false, hasSizes = false } = config || {};

  const key = hasColors
    ? hasSizes
      ? "Case_4"
      : "Case_3"
    : hasSizes
      ? "Case_2"
      : "Case_1";

  const cases = {
    Case_1,
    Case_2,
    Case_3,
    Case_4,
  };

  return {
    key,
    fields: cases[key] || [],
  };
};
```

The resolver returns:

- `key`: The matching schema identifier.
- `fields`: The schema fields associated with the selected configuration.

### Important Rules

1. `hasColors` and `hasSizes` determine the attributes structure.
2. Stock must be managed at the appropriate variation level.
3. Images belong to the product root when colors are disabled.
4. When colors are enabled, images belong to individual color entries.
5. When both colors and sizes are enabled, sizes are nested inside each color.
6. Size and color values are references to their respective collections, not duplicated objects.
7. The schema must remain consistent with the selected product configuration.

---

## Architecture

The attributes system is built using reusable schema definitions:

- `stock`: Shared numeric inventory field.
- `images`: Shared media field.
- `size`: Size collection relation.
- `sizes`: Array of sizes with individual stock.
- `color`: Color collection relation.
- `colors`: Reusable array of colors with configurable nested fields.

The four schemas reuse these definitions to avoid duplication and maintain a consistent data structure.

